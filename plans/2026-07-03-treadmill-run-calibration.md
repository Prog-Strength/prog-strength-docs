# Treadmill Run Calibration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Tag indoor/treadmill runs, let users calibrate their distance, and recompute every distance-derived field in one transaction so the detail page, list views, and dashboard mileage stay internally consistent — while excluding treadmill runs from best-efforts/PRs.

**Architecture:** Two general columns (`environment`, `raw_distance_meters`) added to the existing `activities` table. Ingest detects `<Position>` presence to default running activities to indoor/outdoor. A new `POST /activities/{id}/calibrate` applies a uniform scale factor `f = new/current` across the summary and stored trackpoints in one DB transaction. `PATCH /activities/{id}` gains an `environment` override that maintains best-effort rows (delete on outdoor→indoor; regenerate from the archived S3 TCX on indoor→outdoor). Web adds a treadmill badge + calibrate modal + environment toggle; MCP adds two thin forwarder tools; the agent gets prompt guidance.

**Tech Stack:** Go 1.25 (chi, SQLite/go-sqlite3, aws-sdk S3), Next.js 16 / React 19 / TypeScript / Tailwind v4 / Vitest, Python 3.12 (FastMCP, httpx, pytest) for MCP, Python 3.12 (FastAPI) for the agent.

**Cross-repo dependency order:** `prog-strength-api` is the contract and must land first. `prog-strength-web`, `prog-strength-mcp`, and `prog-strength-agent` consume it and are independent of each other. `prog-strength-docs` flips the SOW status.

---

## Repo: prog-strength-api

Environment/build note for the implementer: this box is Amazon Linux without the apt `libsqlite3-dev` the CI installs; the sqlite-vec cgo header has already been provisioned via `go env -w CGO_CFLAGS="-I/home/developer/localinclude"`, so `go build/test ./...` already work. Do not change that. The CI-pinned linter is golangci-lint **v2.12.2**; run it with `go run github.com/golangci/golangci-lint/v2/cmd/golangci-lint@v2.12.2 run ./...`. Match existing style: comments explain *why*; one type/concept per file in domain packages; use the `httpresp` envelope; `errors.Is` for sentinels; TDD.

### Task A1: TCX parser detects `<Position>` presence

**Files:**
- Modify: `internal/activity/tcx_parser.go`
- Test: `internal/activity/tcx_parser_test.go`

Detect whether any trackpoint carries a `<Position>` element (lat/lon). Do **not** store lat/lon — only a boolean used later for environment defaulting.

- [ ] **Step 1: Add the `HasPosition` field to `parsedTCX`.** In `tcx_parser.go`, add to the `parsedTCX` struct (near `hasCalories`):

```go
	// HasPosition is true iff at least one trackpoint carried a <Position>
	// element (GPS lat/lon). Used only at ingest to default a running
	// activity's environment (no position anywhere => indoor/treadmill). We
	// deliberately do not store the coordinates — route geometry is a
	// separate SOW; this is a presence bit, nothing more.
	HasPosition bool
```

- [ ] **Step 2: Add the XML position type and wire it into `xmlTrackpoint`.** Add a new struct and a field:

```go
type xmlTrackpoint struct {
	Time           string        `xml:"Time"`
	DistanceMeters *float64      `xml:"DistanceMeters"`
	Position       *xmlPosition  `xml:"Position"`
	HeartRate      *xmlHeartRate `xml:"HeartRateBpm"`
	AltitudeMeters *float64      `xml:"AltitudeMeters"`
}

// xmlPosition models <Position><LatitudeDegrees>..</LatitudeDegrees>
// <LongitudeDegrees>..</LongitudeDegrees></Position>. We only need to know
// the element was present, so the fields are parsed but never read beyond a
// nil check on the pointer.
type xmlPosition struct {
	LatitudeDegrees  *float64 `xml:"LatitudeDegrees"`
	LongitudeDegrees *float64 `xml:"LongitudeDegrees"`
}
```

- [ ] **Step 3: Set `HasPosition` in the parse loop.** Inside the `for _, tp := range lap.Trackpoints` loop in `parseTCX`, after computing `out`, add:

```go
			if tp.Position != nil {
				p.HasPosition = true
			}
```

- [ ] **Step 4: Add parser tests.** In `tcx_parser_test.go`, add tests: (a) a running TCX whose trackpoints include a `<Position>` parses with `HasPosition == true`; (b) a running TCX with no `<Position>` anywhere parses with `HasPosition == false`. Use small inline TCX byte slices (mirror the existing inline-XML test style in that file). Run `go test ./internal/activity/ -run TestParse -v`.

- [ ] **Step 5: Commit.** `git add internal/activity/tcx_parser.go internal/activity/tcx_parser_test.go && git commit -m "feat(activity): detect trackpoint Position presence in tcx parser"`

### Task A2: Migration + model + ingest defaulting + repository/DTO plumbing

This task adds the two columns, the `Environment` type, wires `summarize`/ingest to set them (and skip best-efforts for indoor running), threads the fields through the SQLite repository and DTO, and keeps all existing tests green by making the running fixtures carry `<Position>`.

**Files:**
- Create: `internal/db/migrations/036_activity_environment.sql`
- Create: `internal/activity/environment.go`
- Modify: `internal/activity/model.go` (Activity fields)
- Modify: `internal/activity/tcx_summarizer.go` (`summarize`)
- Modify: `internal/activity/sqlite_repository.go` (`activityColumns`, `Create` INSERT, `scanActivity`)
- Modify: `internal/activity/handler.go` (`activityDTO`, `toActivityDTO`)
- Modify fixtures: `internal/activity/testdata/typical_5k.tcx`, `internal/activity/testdata/intervals.tcx`
- Create fixture: `internal/activity/testdata/treadmill_5k.tcx`
- Test: `internal/activity/tcx_summarizer_test.go`, `internal/activity/sqlite_repository_test.go`, `internal/activity/handler_test.go`

- [ ] **Step 1: Write the migration.** Create `internal/db/migrations/036_activity_environment.sql`:

```sql
-- migrations/036_activity_environment.sql
-- Treadmill Run Calibration: add two general, cross-cutting columns to
-- activities.
--   environment          'outdoor' | 'indoor' — whether the activity was
--                        recorded with GPS (outdoor) or without (indoor /
--                        treadmill). General to any distance-based sport,
--                        not running-only.
--   raw_distance_meters  the distance as originally ingested, never mutated
--                        by calibration — enables provenance ("3.10 -> 3.00
--                        mi") and reset-to-original without an S3 read.
-- See prog-strength-docs/sows/treadmill-run-calibration.md.
--
-- SQLite ALTER TABLE ADD COLUMN accepts a column-level CHECK and a constant
-- DEFAULT, so both columns are added in place — no table rebuild. Every
-- existing row defaults environment='outdoor'. raw_distance_meters is added
-- NOT NULL with a constant 0 default (SQLite requires a non-null constant
-- default when adding a NOT NULL column to a populated table) and then
-- backfilled from the current distance_meters so provenance is exact.

ALTER TABLE activities
    ADD COLUMN environment TEXT NOT NULL DEFAULT 'outdoor'
        CHECK (environment IN ('outdoor', 'indoor'));

ALTER TABLE activities
    ADD COLUMN raw_distance_meters REAL NOT NULL DEFAULT 0;

UPDATE activities SET raw_distance_meters = distance_meters;
```

- [ ] **Step 2: Add the `Environment` type.** Create `internal/activity/environment.go`:

```go
package activity

// Environment distinguishes a GPS-recorded outdoor activity from an
// indoor/treadmill activity recorded without position. It is a general
// cross-cutting attribute — it applies to any distance-based activity type,
// not running-only — even though only running defaults to indoor at ingest.
type Environment string

const (
	EnvironmentOutdoor Environment = "outdoor"
	EnvironmentIndoor  Environment = "indoor"
)

// Valid reports whether e is a known member. Used to reject a bad value on
// the PATCH environment-override path before it reaches the DB CHECK.
func (e Environment) Valid() bool {
	switch e {
	case EnvironmentOutdoor, EnvironmentIndoor:
		return true
	}
	return false
}
```

- [ ] **Step 3: Add the Activity fields.** In `model.go`, add to the `Activity` struct (after `DistanceMeters`):

```go
	DistanceMeters      float64
	// RawDistanceMeters is the distance as originally ingested. Set equal to
	// DistanceMeters at ingest and never touched by calibration, so
	// "calibrated" is derivable as RawDistanceMeters != DistanceMeters and a
	// reset is a calibrate back to this value.
	RawDistanceMeters float64
	// Environment is 'outdoor' (GPS) or 'indoor' (treadmill/no-position).
	// Defaulted at ingest for running activities from Position presence;
	// user-overridable via PATCH.
	Environment Environment
```

- [ ] **Step 4: Set the fields in `summarize`.** In `tcx_summarizer.go`, in `summarize`, set `RawDistanceMeters` and `Environment`, and gate the best-effort sweep on outdoor. Replace the current running block:

```go
	a := Activity{
		SourceActivityID:    p.ActivityID,
		ActivityType:        actType,
		StartTime:           first.Time,
		Name:                p.Notes,
		DistanceMeters:      distance,
		RawDistanceMeters:   distance,
		Environment:         EnvironmentOutdoor,
		DurationSeconds:     duration,
		AvgHeartRateBpm:     avgHeartRate(tps),
		MaxHeartRateBpm:     maxHeartRate(tps),
		ElevationGainMeters: elevationGain(tps),
	}

	if actType == ActivityRunning {
		// A running activity with no <Position> anywhere was recorded without
		// GPS — treadmill/indoor. Non-running sports keep the outdoor default;
		// only running is auto-tagged here.
		if !p.HasPosition {
			a.Environment = EnvironmentIndoor
		}
		if distance > 0 {
			avg := float64(duration) / (distance / 1000)
			a.AvgPaceSecPerKm = &avg
		}
		a.BestPaceSecPerKm = bestPace(tps)
		// Best efforts (running PRs) are an outdoor-only surface. Indoor runs
		// are excluded by design: no activity_best_efforts rows are written,
		// so they never appear in PRs or feed max-effort estimates.
		if a.Environment == EnvironmentOutdoor {
			a.BestEfforts = bestEfforts(tps, StandardDistances)
		}
	}
```

(Leave the `if p.hasCalories {...}` and `a.Trackpoints = downsample(...)` tail unchanged.)

- [ ] **Step 5: Thread through the SQLite repository.** In `sqlite_repository.go`:

Extend `activityColumns` (append the two columns at the end so scan order is stable):

```go
const activityColumns = `
	id, user_id, activity_type, ingest_source, source_activity_id,
	start_time, name, distance_meters, duration_seconds,
	avg_pace_sec_per_km, best_pace_sec_per_km,
	avg_heart_rate_bpm, max_heart_rate_bpm, total_calories, elevation_gain_meters,
	tcx_s3_key, created_at, deleted_at, environment, raw_distance_meters`
```

In `Create`, add the two columns + placeholders + args to the INSERT:

```go
	if _, err := tx.ExecContext(ctx, `
		INSERT INTO activities (
			id, user_id, activity_type, ingest_source, source_activity_id,
			start_time, name, distance_meters, duration_seconds,
			avg_pace_sec_per_km, best_pace_sec_per_km,
			avg_heart_rate_bpm, max_heart_rate_bpm, total_calories, elevation_gain_meters,
			tcx_s3_key, created_at, environment, raw_distance_meters
		) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
	`, a.ID, a.UserID, a.ActivityType, a.IngestSource, a.SourceActivityID,
		a.StartTime, a.Name, a.DistanceMeters, a.DurationSeconds,
		a.AvgPaceSecPerKm, a.BestPaceSecPerKm,
		a.AvgHeartRateBpm, a.MaxHeartRateBpm, a.TotalCalories, a.ElevationGainMeters,
		a.TCXS3Key, a.CreatedAt, a.Environment, a.RawDistanceMeters); err != nil {
```

In `scanActivity`, add the two destinations at the end of the `Scan(...)` (matching the new `activityColumns` order — `Environment` is a named string type scanned directly like `ActivityType`; `RawDistanceMeters` is a non-null float):

```go
		&act.TCXS3Key, &act.CreatedAt, &deletedAt, &act.Environment, &act.RawDistanceMeters,
```

- [ ] **Step 6: Expose on the DTO.** In `handler.go`, add to `activityDTO` (after `DistanceMeters`):

```go
	DistanceMeters      float64         `json:"distance_meters"`
	RawDistanceMeters   float64         `json:"raw_distance_meters"`
	Environment         Environment     `json:"environment"`
```

and set them in `toActivityDTO`:

```go
		DistanceMeters:      a.DistanceMeters,
		RawDistanceMeters:   a.RawDistanceMeters,
		Environment:         a.Environment,
```

- [ ] **Step 7: Keep running fixtures outdoor.** The existing running fixtures carry no `<Position>`, so without this step they would reclassify as indoor and drop their best efforts, breaking many tests. Add a `<Position>` element to **every** `<Trackpoint>` in `testdata/typical_5k.tcx` and `testdata/intervals.tcx` (insert directly after each `<AltitudeMeters>...</AltitudeMeters>` line — or after `<DistanceMeters>` for intervals if it has no altitude). A scripted edit is fine, e.g.:

```bash
cd internal/activity/testdata
python3 - <<'PY'
import re
for f in ("typical_5k.tcx", "intervals.tcx"):
    s = open(f).read()
    # Insert a Position block after each closing AltitudeMeters tag; fall back
    # to after DistanceMeters when a trackpoint has no altitude.
    def add_pos(m):
        return m.group(0) + "\n          <Position><LatitudeDegrees>40.0</LatitudeDegrees><LongitudeDegrees>-105.0</LongitudeDegrees></Position>"
    if "<AltitudeMeters>" in s:
        s = re.sub(r"<AltitudeMeters>[^<]*</AltitudeMeters>", add_pos, s)
    else:
        s = re.sub(r"<DistanceMeters>[^<]*</DistanceMeters>", add_pos, s)
    open(f, "w").write(s)
PY
```

Verify each now has Position on trackpoints: `grep -c "<Position>" typical_5k.tcx intervals.tcx` (should be > 0 and equal to the trackpoint count).

- [ ] **Step 8: Add the indoor fixture.** Create `testdata/treadmill_5k.tcx`: a running TCX with **no `<Position>`**, enough cumulative distance to cover at least the 1-mile best-effort target (so best-effort deletion/regeneration is testable in A4), HR values, and a `<Calories>` element. The simplest reliable way is to copy `typical_5k.tcx` **before** the Step-7 edit (a pristine no-position running file) — i.e. generate it from the original content. If the original is already edited, reconstruct a compact ~600-point running track by hand or with a small generator; it must parse and validate (>=1 non-zero-distance trackpoint) and total >= 1609.344 m. Keep `Sport="Running"`.

- [ ] **Step 9: Tests.** Add/adjust:
  - `tcx_summarizer_test.go`: a running `parsedTCX` with `HasPosition=false` → `summarize` yields `Environment == EnvironmentIndoor` and `len(BestEfforts) == 0`; with `HasPosition=true` → `Environment == EnvironmentOutdoor` and best efforts present. Both set `RawDistanceMeters == DistanceMeters`.
  - `sqlite_repository_test.go` / `handler_test.go`: after uploading `typical_5k.tcx`, the returned DTO has `environment == "outdoor"` and `raw_distance_meters == distance_meters`; after uploading `treadmill_5k.tcx`, `environment == "indoor"` and (via repo) zero `activity_best_efforts` rows. Confirm existing typical_5k assertions still pass.

- [ ] **Step 10: Verify + commit.** `go test ./internal/activity/... && go run github.com/golangci/golangci-lint/v2/cmd/golangci-lint@v2.12.2 run ./internal/activity/...`. Then `git add -A && git commit -m "feat(activity): add environment + raw_distance_meters, default indoor from position"`.

### Task A3: Calibration endpoint

**Files:**
- Modify: `internal/activity/repository.go` (add `Calibrate` to the interface)
- Modify: `internal/activity/sqlite_repository.go` (implement `Calibrate`)
- Modify: `internal/activity/handler.go` (route + handler + shared detail-DTO helper)
- Test: `internal/activity/handler_test.go`, `internal/activity/sqlite_repository_test.go`

- [ ] **Step 1: Add `Calibrate` to the `Repository` interface** in `repository.go`:

```go
	// Calibrate rescales an activity to newDistanceMeters in one transaction:
	// it reads the current distance in-tx, computes the uniform factor
	// f = newDistanceMeters / current, updates the summary distance and paces
	// (avg pace recomputed from the corrected distance; best pace scaled by
	// 1/f), and multiplies every stored trackpoint's cumulative distance by f
	// (dividing its non-null pace by f). raw_distance_meters is never touched.
	// Returns the updated activity WITH its rescaled trackpoints. Returns
	// ErrNotFound when the activity is missing, soft-deleted, or not owned.
	// The caller (handler) is responsible for the indoor-only and
	// scale-bounds guards before invoking this.
	Calibrate(ctx context.Context, userID, id string, newDistanceMeters float64) (*Activity, error)
```

- [ ] **Step 2: Implement `Calibrate`** in `sqlite_repository.go`:

```go
func (r *SQLiteRepository) Calibrate(ctx context.Context, userID, activityID string, newDistanceMeters float64) (*Activity, error) {
	tx, err := r.db.BeginTx(ctx, nil)
	if err != nil {
		return nil, err
	}
	defer tx.Rollback()

	var (
		curDistance float64
		duration    int
	)
	row := tx.QueryRowContext(ctx, `
		SELECT distance_meters, duration_seconds
		FROM activities
		WHERE id = ? AND user_id = ? AND deleted_at IS NULL
	`, activityID, userID)
	if err := row.Scan(&curDistance, &duration); err != nil {
		if errors.Is(err, sql.ErrNoRows) {
			return nil, ErrNotFound
		}
		return nil, err
	}
	if curDistance <= 0 {
		// Nothing to scale from; the handler also guards this, but a zero here
		// would divide by zero below.
		return nil, fmt.Errorf("activity: cannot calibrate zero-distance activity %s", activityID)
	}
	f := newDistanceMeters / curDistance

	// avg pace is recomputed from the corrected distance; best pace scales
	// inversely with the distance factor (a longer real distance => faster
	// pace). NULL best pace stays NULL under SQL arithmetic.
	avgPace := float64(duration) / (newDistanceMeters / 1000)
	if _, err := tx.ExecContext(ctx, `
		UPDATE activities
		SET distance_meters = ?,
		    avg_pace_sec_per_km = ?,
		    best_pace_sec_per_km = best_pace_sec_per_km / ?
		WHERE id = ? AND user_id = ?
	`, newDistanceMeters, avgPace, f, activityID, userID); err != nil {
		return nil, err
	}

	// Rescale every trackpoint's cumulative distance and (non-null) pace. A
	// NULL pace_sec_per_km divided by f is NULL, so stationary-filtered points
	// keep their null.
	if _, err := tx.ExecContext(ctx, `
		UPDATE activity_trackpoints
		SET distance_meters = distance_meters * ?,
		    pace_sec_per_km = pace_sec_per_km / ?
		WHERE activity_id = ?
	`, f, f, activityID); err != nil {
		return nil, err
	}

	if err := tx.Commit(); err != nil {
		return nil, err
	}
	return r.Get(ctx, userID, activityID)
}
```

Add the `var _ Repository = (*SQLiteRepository)(nil)` assertion already present will now enforce the new method — make sure it compiles. (`MemoryArchiver` is unaffected; only the SQLite repo implements `Repository`.)

- [ ] **Step 3: Extract a shared detail-DTO builder in `handler.go`.** The GET detail path builds the DTO with trackpoints + optional HR zones. Extract that into a helper so the calibrate response is byte-for-byte the same shape. Add:

```go
// buildDetailDTO renders the full single-activity detail shape: the base DTO
// with trackpoints plus, for running activities with usable HR when the
// engine is wired, the heart_rate_zones block. Shared by the detail GET and
// the calibrate response so both return an identical shape.
func (h *Handler) buildDetailDTO(ctx context.Context, userID string, a Activity) (activityDTO, error) {
	dto := toActivityDTO(a, true)
	if h.hrEngine == nil || a.ActivityType != ActivityRunning {
		return dto, nil
	}
	tps := make([]hrzones.Trackpoint, 0, len(a.Trackpoints))
	currentRunHRSamples := make([]int, 0, len(a.Trackpoints))
	for _, tp := range a.Trackpoints {
		tps = append(tps, hrzones.Trackpoint{ElapsedSeconds: tp.ElapsedSeconds, HeartRateBpm: tp.HeartRateBpm})
		if tp.HeartRateBpm != nil {
			currentRunHRSamples = append(currentRunHRSamples, *tp.HeartRateBpm)
		}
	}
	stats, err := h.repo.RecentHRStats(ctx, userID, h.hrWindow, a.ID)
	if err != nil {
		return activityDTO{}, err
	}
	stats.CurrentRunP99 = hrzones.P99(currentRunHRSamples)
	ref := h.hrEngine.EstimateReference(stats)
	if res, ok := h.hrEngine.Compute(ref, tps); ok {
		zones := make([]heartRateZoneDTO, 0, len(res.Zones))
		for _, z := range res.Zones {
			zones = append(zones, heartRateZoneDTO{
				Zone: z.Number, Name: z.Name, LowerPct: z.LowerPct, UpperPct: z.UpperPct,
				MinBpm: z.MinBpm, MaxBpm: z.MaxBpm, TimeSeconds: z.TimeSeconds, TimePct: z.TimePct,
			})
		}
		dto.HeartRateZones = &heartRateZonesDTO{
			Model: res.Model, MaxHRReferenceBpm: res.Reference.MaxHRBpm,
			ReferenceSource: res.Reference.Source, ReferenceConfidence: string(res.Reference.Confidence),
			Calibrating: res.Calibrating, TotalHRSeconds: res.TotalHRSeconds, Zones: zones,
		}
	}
	return dto, nil
}
```

Then refactor the existing `get` handler to use it: after fetching `a`, replace the inline DTO+HR-zones block with:

```go
	dto, err := h.buildDetailDTO(r.Context(), userID, *a)
	if err != nil {
		httpresp.ServerError(w, r.Context(), "build activity detail", err)
		return
	}
	httpresp.OK(w, "fetched activity", dto)
```

- [ ] **Step 4: Add calibration bounds constants + the handler** in `handler.go`:

```go
// Calibration rejects a correction factor outside these bounds to catch a
// unit-entry mistake (miles typed as meters, etc.). Documented in the SOW;
// tune if a real calibration legitimately exceeds them.
const (
	calibrationMinFactor = 0.5
	calibrationMaxFactor = 2.0
)
```

```go
func (h *Handler) calibrate(w http.ResponseWriter, r *http.Request) {
	userID, ok := auth.UserIDFrom(r.Context())
	if !ok {
		httpresp.ServerError(w, r.Context(), "missing user in context", errors.New("auth middleware not applied"))
		return
	}
	activityID := chi.URLParam(r, "id")
	if activityID == "" {
		httpresp.Error(w, http.StatusBadRequest, "activity id is required")
		return
	}
	var req struct {
		DistanceMeters float64 `json:"distance_meters"`
	}
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		httpresp.Error(w, http.StatusBadRequest, "invalid request body")
		return
	}

	a, err := h.repo.Get(r.Context(), userID, activityID)
	if err != nil {
		if errors.Is(err, ErrNotFound) {
			httpresp.ErrorWithCode(w, http.StatusNotFound, "activity not found", "not_found")
			return
		}
		httpresp.ServerError(w, r.Context(), "get activity", err)
		return
	}
	if a.ActivityType != ActivityRunning {
		httpresp.ErrorWithCode(w, http.StatusBadRequest, "only running activities can be calibrated", "not_a_running_activity")
		return
	}
	if a.Environment != EnvironmentIndoor {
		httpresp.ErrorWithCode(w, http.StatusBadRequest, "tag the run as indoor before calibrating its distance", "outdoor_run_not_calibratable")
		return
	}
	if req.DistanceMeters <= 0 {
		httpresp.ErrorWithCode(w, http.StatusBadRequest, "distance_meters must be greater than zero", "invalid_calibration_distance")
		return
	}
	if a.DistanceMeters <= 0 {
		httpresp.ErrorWithCode(w, http.StatusBadRequest, "activity has no distance to calibrate from", "invalid_calibration_distance")
		return
	}
	f := req.DistanceMeters / a.DistanceMeters
	if f < calibrationMinFactor || f > calibrationMaxFactor {
		httpresp.ErrorWithCode(w, http.StatusBadRequest, "calibrated distance implies an implausible correction (must be between 0.5x and 2x the current distance)", "calibration_out_of_range")
		return
	}

	updated, err := h.repo.Calibrate(r.Context(), userID, activityID, req.DistanceMeters)
	if err != nil {
		if errors.Is(err, ErrNotFound) {
			httpresp.ErrorWithCode(w, http.StatusNotFound, "activity not found", "not_found")
			return
		}
		httpresp.ServerError(w, r.Context(), "calibrate activity", err)
		return
	}
	dto, err := h.buildDetailDTO(r.Context(), userID, *updated)
	if err != nil {
		httpresp.ServerError(w, r.Context(), "build activity detail", err)
		return
	}
	httpresp.OK(w, "calibrated activity", dto)
}
```

- [ ] **Step 5: Register the route.** In `Mount`, inside the `/activities` route group, add (after the PATCH line):

```go
		r.Post("/{id}/calibrate", h.calibrate)
```

- [ ] **Step 6: Tests** in `handler_test.go` (multipart-upload + JSON pattern already established there) and `sqlite_repository_test.go`:
  - Upload `treadmill_5k.tcx` (indoor). POST `/activities/{id}/calibrate` with a `distance_meters` ~5% off current. Assert 200; `distance_meters` equals the requested value; `raw_distance_meters` unchanged (== original); the returned `trackpoints` last cumulative distance ≈ `distance_meters` (within float tolerance); `avg_pace_sec_per_km` == `duration/(new/1000)`.
  - Calibrating an **outdoor** run (upload `typical_5k.tcx`) → 400 with code `outdoor_run_not_calibratable`.
  - `distance_meters <= 0` → 400 `invalid_calibration_distance`.
  - A distance implying `f` outside [0.5, 2.0] → 400 `calibration_out_of_range`.
  - Unknown id → 404.
  - Repo-level `Calibrate` test: header distance and the sum/last of trackpoint distances stay consistent (uniform scale).

- [ ] **Step 7: Verify + commit.** `go test ./internal/activity/... && golangci-lint run ./internal/activity/...`. Then `git commit -am "feat(activity): add POST /activities/{id}/calibrate for indoor runs"`.

### Task A4: PATCH environment override + best-effort maintenance

**Files:**
- Modify: `internal/activity/repository.go` (add `ChangeEnvironment` to the interface)
- Modify: `internal/activity/sqlite_repository.go` (implement `ChangeEnvironment`)
- Modify: `internal/activity/handler.go` (extend the PATCH handler)
- Test: `internal/activity/handler_test.go`, `internal/activity/sqlite_repository_test.go`

- [ ] **Step 1: Add `ChangeEnvironment` to the interface** in `repository.go`:

```go
	// ChangeEnvironment sets a live activity's environment and maintains its
	// best-effort rows for the transition:
	//   outdoor -> indoor: delete this activity's activity_best_efforts rows
	//     (indoor runs are excluded from PR surfaces). distance_meters,
	//     including any prior calibration, is preserved.
	//   indoor -> outdoor: regenerate best-efforts by fetching the archived
	//     TCX, scaling each raw cumulative distance by the effective factor
	//     distance_meters/raw_distance_meters, sweeping at full resolution,
	//     and inserting the rows.
	// A same-environment call is a no-op that returns the current row.
	// Best-effort maintenance only applies to running activities. Returns
	// ErrNotFound when missing/soft-deleted/not owned; returns the updated
	// activity summary (no trackpoints).
	ChangeEnvironment(ctx context.Context, userID, id string, newEnv Environment) (*Activity, error)
```

- [ ] **Step 2: Implement `ChangeEnvironment`** in `sqlite_repository.go`. The S3 fetch/parse happens *outside* the DB transaction (mirroring `BackfillActivityBestEfforts`); the environment flip and best-effort rows are committed together:

```go
func (r *SQLiteRepository) ChangeEnvironment(ctx context.Context, userID, activityID string, newEnv Environment) (*Activity, error) {
	var (
		curEnv      Environment
		rawDistance float64
		distance    float64
		s3Key       string
		actType     ActivityType
	)
	row := r.db.QueryRowContext(ctx, `
		SELECT environment, raw_distance_meters, distance_meters, tcx_s3_key, activity_type
		FROM activities
		WHERE id = ? AND user_id = ? AND deleted_at IS NULL
	`, activityID, userID)
	if err := row.Scan(&curEnv, &rawDistance, &distance, &s3Key, &actType); err != nil {
		if errors.Is(err, sql.ErrNoRows) {
			return nil, ErrNotFound
		}
		return nil, err
	}
	if curEnv == newEnv {
		return r.readActivitySummary(ctx, userID, activityID)
	}

	// For indoor -> outdoor on a running activity, regenerate best efforts from
	// the archived raw TCX BEFORE opening the write transaction, so a slow S3
	// fetch never holds a DB write lock and a fetch/parse failure aborts the
	// whole change (rather than leaving an outdoor run with no PR rows).
	var regenerated []ActivityBestEffort
	if actType == ActivityRunning && newEnv == EnvironmentOutdoor {
		body, err := r.archiver.Get(ctx, s3Key)
		if err != nil {
			return nil, fmt.Errorf("change environment: fetch tcx: %w", err)
		}
		parsed, err := parseTCX(body)
		if err != nil {
			return nil, fmt.Errorf("change environment: parse tcx: %w", err)
		}
		if err := validate(parsed); err != nil {
			return nil, fmt.Errorf("change environment: validate tcx: %w", err)
		}
		// Apply the effective calibration factor to the raw cumulative
		// distances so regenerated efforts match the (possibly calibrated)
		// current distance. Never-calibrated runs have distance == raw => 1.0.
		scale := 1.0
		if rawDistance > 0 {
			scale = distance / rawDistance
		}
		scaled := make([]parsedTrackpoint, len(parsed.Trackpoints))
		copy(scaled, parsed.Trackpoints)
		for i := range scaled {
			scaled[i].DistanceMeters *= scale
		}
		regenerated = bestEfforts(scaled, StandardDistances)
	}

	tx, err := r.db.BeginTx(ctx, nil)
	if err != nil {
		return nil, err
	}
	defer tx.Rollback()

	if _, err := tx.ExecContext(ctx, `
		UPDATE activities SET environment = ?
		WHERE id = ? AND user_id = ? AND deleted_at IS NULL
	`, newEnv, activityID, userID); err != nil {
		return nil, err
	}

	if actType == ActivityRunning {
		switch newEnv {
		case EnvironmentIndoor:
			if _, err := tx.ExecContext(ctx, `
				DELETE FROM activity_best_efforts WHERE activity_id = ?
			`, activityID); err != nil {
				return nil, err
			}
		case EnvironmentOutdoor:
			// Replace any existing rows, then insert the freshly swept set.
			if _, err := tx.ExecContext(ctx, `
				DELETE FROM activity_best_efforts WHERE activity_id = ?
			`, activityID); err != nil {
				return nil, err
			}
			if err := insertBestEffortsTx(ctx, tx, activityID, regenerated); err != nil {
				return nil, err
			}
		}
	}

	if err := tx.Commit(); err != nil {
		return nil, err
	}
	return r.readActivitySummary(ctx, userID, activityID)
}

// readActivitySummary re-reads a row (without trackpoints) after a mutation,
// mirroring Rename's return shape. It does not filter deleted_at because the
// caller has just confirmed/updated a live row.
func (r *SQLiteRepository) readActivitySummary(ctx context.Context, userID, activityID string) (*Activity, error) {
	row := r.db.QueryRowContext(ctx, `
		SELECT `+activityColumns+`
		FROM activities
		WHERE id = ? AND user_id = ?
	`, activityID, userID)
	return scanActivity(row)
}
```

- [ ] **Step 3: Extend the PATCH handler** in `handler.go`. Rename the `rename` handler to `patch` (update the `Mount` reference `r.Patch("/{id}", h.patch)`), and replace its body to accept optional `name` and/or `environment`:

```go
func (h *Handler) patch(w http.ResponseWriter, r *http.Request) {
	userID, ok := auth.UserIDFrom(r.Context())
	if !ok {
		httpresp.ServerError(w, r.Context(), "missing user in context", errors.New("auth middleware not applied"))
		return
	}
	activityID := chi.URLParam(r, "id")
	if activityID == "" {
		httpresp.Error(w, http.StatusBadRequest, "activity id is required")
		return
	}
	var req struct {
		Name        *string `json:"name"`
		Environment *string `json:"environment"`
	}
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		httpresp.Error(w, http.StatusBadRequest, "invalid request body")
		return
	}
	if req.Name == nil && req.Environment == nil {
		httpresp.Error(w, http.StatusBadRequest, "name or environment is required")
		return
	}

	var updated *Activity
	if req.Name != nil {
		name := strings.TrimSpace(*req.Name)
		if name == "" {
			httpresp.Error(w, http.StatusBadRequest, "name is required")
			return
		}
		if len(name) > 200 {
			httpresp.Error(w, http.StatusBadRequest, "name is too long")
			return
		}
		a, err := h.repo.Rename(r.Context(), userID, activityID, name)
		if err != nil {
			if errors.Is(err, ErrNotFound) {
				httpresp.ErrorWithCode(w, http.StatusNotFound, "activity not found", "not_found")
				return
			}
			httpresp.ServerError(w, r.Context(), "rename activity", err)
			return
		}
		updated = a
	}
	if req.Environment != nil {
		env := Environment(*req.Environment)
		if !env.Valid() {
			httpresp.ErrorWithCode(w, http.StatusBadRequest, "environment must be 'outdoor' or 'indoor'", "invalid_environment")
			return
		}
		a, err := h.repo.ChangeEnvironment(r.Context(), userID, activityID, env)
		if err != nil {
			if errors.Is(err, ErrNotFound) {
				httpresp.ErrorWithCode(w, http.StatusNotFound, "activity not found", "not_found")
				return
			}
			httpresp.ServerError(w, r.Context(), "change environment", err)
			return
		}
		updated = a
	}
	httpresp.OK(w, "updated activity", toActivityDTO(*updated, false))
}
```

- [ ] **Step 4: Tests** in `handler_test.go` and `sqlite_repository_test.go`:
  - Upload `typical_5k.tcx` (outdoor, has best efforts). PATCH `{ "environment": "indoor" }` → 200, DTO `environment == "indoor"`; assert its `activity_best_efforts` rows are now deleted (query the repo / `GetUserRunningBestEfforts` no longer includes it).
  - Then PATCH `{ "environment": "outdoor" }` back → 200; assert best-effort rows are regenerated (present again). With the archived TCX intact (MemoryArchiver keeps it), regeneration should reproduce the original efforts.
  - PATCH `{ "environment": "indoor" }` then calibrate then PATCH `{ "environment": "outdoor" }` → regenerated efforts reflect the calibrated distance (scale = distance/raw). (Assert the row set is non-empty and the distances differ from raw when calibrated.)
  - PATCH `{ "name": "X", "environment": "indoor" }` (both) → 200, both applied.
  - PATCH `{ "environment": "sideways" }` → 400 `invalid_environment`.
  - PATCH `{}` (neither) → 400.
  - Existing rename tests (`{ "name": ... }`) still pass.

- [ ] **Step 5: Verify + commit.** `go test ./internal/activity/... && golangci-lint run ./internal/activity/...`. Then `git commit -am "feat(activity): extend PATCH with environment override + best-effort maintenance"`.

### Task A5: Full-module gate

- [ ] Run the complete CI gate locally from the repo root and fix anything:
```
go build ./...
go vet ./...
go mod tidy   # must produce no go.mod/go.sum diff
go test ./...
go run github.com/golangci/golangci-lint/v2/cmd/golangci-lint@v2.12.2 run ./...
```
- [ ] Commit any tidy drift only if legitimately needed (none expected — no new deps).

---

## Repo: prog-strength-web

Design-system conformance: use existing CSS tokens in `app/globals.css` (`--surface-2`, `--border`, `--muted`, `--faint`, `--foreground`, `--danger`, `--accent`, `--discipline-run-*`). The treadmill badge encodes an **attribute/status via shape + label**, not a new activity hue — render it as a neutral metadata chip (surface-2 fill, hairline `--border`, `--muted`/`--faint` text) with a small treadmill glyph + the word **"Treadmill"**. Do **not** invent a new color token. Match the form-control specs (rounded field, uppercase faint labels, accent focus ring) for the modal. Run `npm run typecheck && npm run lint && npm run test` (and `npm run build` since routes/components change) before declaring done. Reuse the existing 401-redirect, toast, and modal patterns.

### Task W1: API client — types + calibrate/patch wrappers

**Files:**
- Modify: `lib/api.ts`
- Test: `lib/api.test.ts`

- [ ] **Step 1: Extend `RunningSession`** (near line 1676) with the two fields:

```typescript
  distance_meters: number;
  raw_distance_meters: number;
  environment: "outdoor" | "indoor";
```

- [ ] **Step 2: Add `calibrateRunningSession`** following the `renameRunningSession` shape (uses `unwrap`, POSTs to `/activities/{id}/calibrate`):

```typescript
export async function calibrateRunningSession(
  token: string,
  id: string,
  distanceMeters: number,
): Promise<RunningSession> {
  const resp = await fetch(`${config.apiUrl}/activities/${encodeURIComponent(id)}/calibrate`, {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      Authorization: `Bearer ${token}`,
    },
    body: JSON.stringify({ distance_meters: distanceMeters }),
  });
  const updated = await unwrap<RunningSession | null>(resp, null);
  if (!updated) {
    throw new Error("API did not return the calibrated activity");
  }
  return updated;
}
```

- [ ] **Step 3: Add `setRunningSessionEnvironment`** (PATCH with `environment`, mirroring `renameRunningSession`):

```typescript
export async function setRunningSessionEnvironment(
  token: string,
  id: string,
  environment: "outdoor" | "indoor",
): Promise<RunningSession> {
  const resp = await fetch(`${config.apiUrl}/activities/${encodeURIComponent(id)}`, {
    method: "PATCH",
    headers: {
      "Content-Type": "application/json",
      Authorization: `Bearer ${token}`,
    },
    body: JSON.stringify({ environment }),
  });
  const updated = await unwrap<RunningSession | null>(resp, null);
  if (!updated) {
    throw new Error("API did not return the updated activity");
  }
  return updated;
}
```

- [ ] **Step 4: Tests** in `lib/api.test.ts` using the `mockFetchOk`/`vi.stubGlobal("fetch", ...)` pattern: assert `calibrateRunningSession` POSTs to `/activities/{id}/calibrate` with body `{ distance_meters }` and bearer header, and returns the unwrapped session; same for `setRunningSessionEnvironment` (PATCH, body `{ environment }`). Include a case where the API returns an updated session carrying `environment` and `raw_distance_meters`.

- [ ] **Step 5: Commit.** `git commit -am "feat(running): add calibrate + environment api wrappers and session fields"`

### Task W2: Treadmill badge + run-list glyph

**Files:**
- Create: `app/(app)/running/_components/TreadmillBadge.tsx`
- Modify: `app/(app)/running/[id]/_components/RunHeaderBand.tsx` (or the detail page that renders it) to show the badge for indoor running
- Modify: `app/(app)/running/_components/RunListRow.tsx` (glyph on indoor running rows)
- Test: co-located `*.test.tsx`

- [ ] **Step 1: Create `TreadmillBadge`** — a small neutral chip (rounded-full, `--surface-2` bg, hairline `--border`, `--muted` text) with an inline treadmill glyph (an emoji `🏃` is off; use a simple inline SVG or the text label). Keep it purely presentational:

```tsx
export function TreadmillBadge({ className = "" }: { className?: string }) {
  return (
    <span
      className={`inline-flex items-center gap-1 rounded-full border border-[var(--border)] bg-[var(--surface-2)] px-2 py-0.5 text-[11px] font-medium text-[var(--muted)] ${className}`}
      title="Recorded indoors (treadmill) — distance is user-calibrated and excluded from PRs"
    >
      <svg width="11" height="11" viewBox="0 0 24 24" fill="none" aria-hidden="true">
        <path d="M4 18h14M6 18l2-8 6 1 3 4" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round" />
        <circle cx="14" cy="5" r="1.6" fill="currentColor" />
      </svg>
      Treadmill
    </span>
  );
}
```

- [ ] **Step 2: Render it on the detail header** when `session.environment === "indoor" && session.activity_type === "running"`. Place it in/near `RunHeaderBand` (pass a prop or render adjacent to the title). Keep the run discipline hue on the activity itself; the badge stays neutral.

- [ ] **Step 3: Render a compact glyph on `RunListRow`** when `session.environment === "indoor" && session.activity_type === "running"` — the same small SVG (no label needed in the dense row), placed next to the run title. Reuse the `TreadmillBadge` (or a `glyphOnly` variant) so there's one source of truth.

- [ ] **Step 4: Tests** — render `RunListRow`/header with an indoor running session → the treadmill glyph/label appears; with an outdoor session → it does not.

- [ ] **Step 5: Commit.** `git commit -am "feat(running): treadmill badge on run detail + list glyph"`

### Task W3: Calibrate distance modal + provenance + environment toggle

**Files:**
- Create: `app/(app)/running/[id]/_components/CalibrateDistanceModal.tsx`
- Modify: `app/(app)/running/[id]/page.tsx` (wire the action, toggle, provenance line, and state replacement)
- Test: co-located `*.test.tsx`

- [ ] **Step 1: Build `CalibrateDistanceModal`** matching the existing modal pattern (`components/exercise-edit-modal.tsx`) and form-control tokens. Props: `{ session, onClose, onCalibrated }`. Pre-fill a numeric input with the current distance in the user's unit (`formatDistanceValue(session.distance_meters, unit)`), an uppercase faint label ("Corrected distance"), accent focus ring, a live pace preview computed from `duration_seconds` and the entered distance (convert unit → meters → `formatPaceValue`), inline validation (must parse to a number > 0), and a primary accent "Calibrate" button + secondary "Cancel". On submit, call `calibrateRunningSession(token, session.id, meters)`; on success call `onCalibrated(updatedSession)` and toast success; on error show the error message (surface `outdoor_run_not_calibratable` etc. from the thrown message) and toast error.

  Unit → meters: `meters = value * (unit === "mi" ? METERS_PER_MILE : METERS_PER_KM)`. Reuse the constants from `lib/distance-unit-context.tsx` (export them if not already exported, or recompute the two constants locally with a comment).

- [ ] **Step 2: Add the "Calibrate distance" action** on the detail page, shown only for `environment === "indoor" && activity_type === "running"`. Opening it renders the modal. On `onCalibrated(updated)`, replace the whole session state with the API response **including its trackpoints** (do not preserve stale trackpoints):

```tsx
onCalibrated={(updated) => {
  setSession(updated);
  setCalibrateOpen(false);
}}
```

Because the splits/pace-strip derive from `session.trackpoints` via `deriveRunningActivity`, replacing `session` wholesale keeps the header and splits consistent.

- [ ] **Step 3: Provenance line** — when `session.raw_distance_meters !== session.distance_meters`, show a small `--muted` line near the header: `Calibrated from {formatDistance(raw)} {unitLabel} · Reset`, where **Reset** is a button that calls `calibrateRunningSession(token, id, session.raw_distance_meters)` and replaces session on success (reset = calibrate back to raw).

- [ ] **Step 4: Environment toggle** near the badge — a segmented Outdoor/Indoor control (design-system segmented toggle: `--surface` track, active segment `--accent` fill + `--accent-fg`). Switching calls `setRunningSessionEnvironment(token, id, next)`. Because switching **to** outdoor can add the run back into PR surfaces and switching **to** indoor removes it, show a lightweight confirm (a window.confirm or an inline confirm consistent with existing patterns) noting PR membership will change, then on success replace `session` (refetch via `getRunningSession` to pick up any server-side changes, or use the PATCH response summary and keep existing trackpoints since environment change does not rescale trackpoints).

- [ ] **Step 5: Tests** — modal: invalid (0 / empty / negative) shows validation and does not call the API; a valid value calls `calibrateRunningSession` with the correct meters (unit-converted) and, on the mocked rescaled response, the parent replaces session so the rendered header distance equals the sum of split distances derived from the mocked trackpoints (assert header/splits consistency). Environment toggle: calls `setRunningSessionEnvironment` with the target value. Badge visibility keyed on environment.

- [ ] **Step 6: Verify + commit.** `npm run typecheck && npm run lint && npm run test && npm run build`. Then `git commit -am "feat(running): calibrate-distance modal, provenance reset, environment toggle"`.

---

## Repo: prog-strength-mcp

Transparent forwarder — no business logic, no signing key. Add the two client methods and two tools; ensure run-read tools surface the new fields (they forward API JSON verbatim, so `environment`/`raw_distance_meters` flow through automatically once the API returns them — confirm and add a test asserting they pass through). Line length 100. Test with `uv run pytest` and lint with `uv run ruff check src tests`.

### Task M1: API client methods + tools + tests

**Files:**
- Modify: `src/prog_strength_mcp/api_client.py`
- Modify: `src/prog_strength_mcp/running.py`
- Test: `tests/test_running_tools.py`

- [ ] **Step 1: Add client methods** to `APIClient` (mirror the existing POST/PUT unwrap+`_raise_for_status` pattern):

```python
async def calibrate_run_distance(
    self, auth_header: str, activity_id: str, *, distance_meters: float
) -> dict[str, Any]:
    """POST /activities/{id}/calibrate. Rescales an indoor run's distance and
    all distance-derived fields server-side; returns the updated activity."""
    resp = await self._client.post(
        f"/activities/{activity_id}/calibrate",
        json={"distance_meters": distance_meters},
        headers={"Authorization": auth_header},
    )
    _raise_for_status(resp)
    data = resp.json().get("data")
    return data if isinstance(data, dict) else {}


async def set_run_environment(
    self, auth_header: str, activity_id: str, *, environment: str
) -> dict[str, Any]:
    """PATCH /activities/{id} with an environment override
    ('outdoor' | 'indoor'). Returns the updated activity summary."""
    resp = await self._client.patch(
        f"/activities/{activity_id}",
        json={"environment": environment},
        headers={"Authorization": auth_header},
    )
    _raise_for_status(resp)
    data = resp.json().get("data")
    return data if isinstance(data, dict) else {}
```

(Confirm `self._client` supports `.patch`; httpx.AsyncClient does.)

- [ ] **Step 2: Add the two tools** inside `running.py`'s `register()` (close over `api`, guard auth, map `APIError`):

```python
@mcp.tool
async def set_run_environment(
    activity_id: Annotated[str, Field(description="The running activity ID to retag.")],
    environment: Annotated[
        Literal["outdoor", "indoor"],
        Field(description="'indoor' for treadmill runs (no GPS), 'outdoor' for GPS runs."),
    ],
) -> dict[str, Any]:
    """Tag a run as indoor (treadmill) or outdoor.

    Indoor/treadmill runs are recorded without GPS; their distance comes from
    the watch footpod and is user-calibrated. Tagging a run 'indoor' removes
    it from running PRs / best-efforts; 'outdoor' restores it. You must tag a
    run 'indoor' before its distance can be calibrated with
    calibrate_run_distance. Returns the updated activity.
    """
    auth = _auth_header_or_raise()
    try:
        return await api.set_run_environment(auth, activity_id, environment=environment)
    except APIError as e:
        raise RuntimeError(f"API error ({e.status_code}): {e.message}") from e


@mcp.tool
async def calibrate_run_distance(
    activity_id: Annotated[str, Field(description="The indoor running activity ID to calibrate.")],
    distance_meters: Annotated[
        float, Field(gt=0, description="The corrected total distance in meters.")
    ],
) -> dict[str, Any]:
    """Calibrate the distance of an indoor/treadmill run.

    Treadmill footpod distance is often wrong; set the corrected total
    distance (e.g. from the treadmill console) and the server rescales pace
    and every trackpoint by a uniform factor in one step. Only allowed on runs
    already tagged 'indoor' (use set_run_environment first if needed) — the
    API rejects calibration of an outdoor run. Returns the updated activity
    with rescaled trackpoints.
    """
    auth = _auth_header_or_raise()
    try:
        return await api.calibrate_run_distance(auth, activity_id, distance_meters=distance_meters)
    except APIError as e:
        raise RuntimeError(f"API error ({e.status_code}): {e.message}") from e
```

Ensure `Literal` and `Annotated`/`Field` are imported at the top of `running.py` (mirror `workouts.py`).

- [ ] **Step 3: Tests** in `tests/test_running_tools.py` using `respx`: `calibrate_run_distance` POSTs to `/activities/{id}/calibrate` with body `{"distance_meters": ...}` and forwards auth, unwraps `data`; `set_run_environment` PATCHes `/activities/{id}` with `{"environment": "indoor"}`; both map a non-2xx to `RuntimeError`; both raise on missing auth before any HTTP (the `_ExplodingAPI` guard pattern). Add a pass-through test: a run-read tool response containing `environment` and `raw_distance_meters` returns them verbatim to the caller.

- [ ] **Step 4: Verify + commit.** `uv run pytest && uv run ruff check src tests`. Then `git commit -am "feat: add calibrate_run_distance and set_run_environment tools"`.

---

## Repo: prog-strength-agent

Prompt-only change (no new intent, no routing change). Line length 100. Test with `uv run pytest` and lint with `uv run ruff check src tests evals`.

### Task AG1: Tool guidance for treadmill calibration

**Files:**
- Modify: `src/prog_strength_agent/prompt.py` (`SYSTEM_PROMPT`)
- Test: `tests/test_prompt.py`

- [ ] **Step 1: Add a guidance block** to `SYSTEM_PROMPT` under "## Conventions you must follow" (alongside the other domain rules). Keep lines ≤100 chars, use backslash continuations like the surrounding text, and reference the MCP tool names in backticks:

```
**Treadmill (indoor) runs.** Runs recorded on a treadmill have no GPS and \
their footpod distance is often wrong. Tag such a run indoor with \
`set_run_environment(activity_id, "indoor")`, then fix its distance with \
`calibrate_run_distance(activity_id, distance_meters)` — the server rescales \
pace and splits from a single corrected distance, so never try to edit pace \
or trackpoints yourself. Only indoor runs can be calibrated: if the user \
wants to correct an outdoor run's distance, tag it indoor first. Treadmill \
runs stay in weekly mileage and streaks but are excluded from running PRs / \
best-efforts and max-effort estimates.
```

- [ ] **Step 2: Add prompt tests** in `tests/test_prompt.py` mirroring existing assertions: assert `SYSTEM_PROMPT` contains `set_run_environment`, `calibrate_run_distance`, and phrasing about indoor/treadmill runs being excluded from PRs; and assert the guidance survives `build_chat_system_prompt(...)` composition (the composed `out` still contains "treadmill" or "indoor" and "calibrate").

- [ ] **Step 3: Verify + commit.** `uv run pytest && uv run ruff check src tests evals`. Then `git commit -am "feat(agent): document treadmill tag + distance calibration tools"`.

---

## Repo: prog-strength-docs

Handled by the controller after all implementation PRs are open: flip `sows/treadmill-run-calibration.md` frontmatter `status: shipped`, body `**Status**: Shipped`, `**Last updated**: 2026-07-03`, commit `docs: mark treadmill-run-calibration as shipped`, and open the docs PR with the required template body linking every implementation PR.

---

## Self-Review (controller checklist)

- **Spec coverage:** migration 036 (env + raw_distance) ✓; parser Position/has_position ✓; ingest defaulting + skip best-efforts for indoor ✓; POST calibrate (indoor-only, bounds, one-tx rescale, full detail return) ✓; PATCH environment (delete on outdoor→indoor, regenerate via S3 on indoor→outdoor, keep calibrated distance) ✓; DTO fields on GET/list ✓; web fields + wrappers + badge + modal + provenance + toggle + list glyph ✓; MCP two tools + field pass-through ✓; agent guidance ✓; tests each repo ✓.
- **Non-goals respected:** no lat/lon storage; no mobile; no automated backfill; no route map; duration/HR/calories/elevation untouched by calibrate; no timeline retraction; outdoor not directly calibratable (indoor-first enforced).
- **Type consistency:** `Environment`/`EnvironmentIndoor`/`EnvironmentOutdoor`, `RawDistanceMeters`, `Calibrate`, `ChangeEnvironment`, `buildDetailDTO`, `readActivitySummary`, `calibrateRunningSession`, `setRunningSessionEnvironment`, `calibrate_run_distance`, `set_run_environment` used consistently across tasks.
- **Open Questions:** Q1 keep calibrated distance on indoor→outdoor → lean (a) implemented (scale = distance/raw). Q4 bounds [0.5, 2.0] implemented. Q2/Q3 (timeline staleness, calendar glyph) out of scope, unchanged.
