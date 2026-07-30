# Hiking Activity Type Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Register hiking as a fifth endurance-shaped activity type end-to-end — TCX ingest with an explicit `activity_type` override, a widened shared endurance elevation shape (loss/high/low), a Hiking tab and `/hiking/[id]` detail page on web, a clay discipline hue, and MCP docstrings — with no switch statements touched outside descriptors/declared repository seams.

**Architecture:** Hiking rides the existing unified activity model. One migration adds `activity_hike_details` and three nullable elevation columns to the four existing detail tables. The Go registry gains one `NewEnduranceDescriptor(ActivityHiking, …)` registration; the shared endurance store/summarizer serve it unchanged except a hiking-only three-chip card. TCX ingest gains an optional `activity_type` form field. The web read side computes six tiles client-side from the unified list projection (no new endpoint).

**Tech Stack:** Go 1.25 (chi, mattn/go-sqlite3, golangci-lint v2.12.2), Next.js/React/TypeScript (vitest, eslint, prettier), Python FastMCP (ruff, pytest).

---

## Affected repos & branches

All work lands on a branch named `feat/hiking-activity-type` in each repo:
- `prog-strength-api` (Go) — Tasks 1–2
- `prog-strength-web` (TS/React) — Tasks 3–6
- `prog-strength-mcp` (Python) — Task 7
- `prog-strength-docs` — Task 8 (design-system + SOW status flip + this plan)

## Local CI gates (run before pushing each repo)

- **api:** `cd /workspace/prog-strength-api && go build ./... && go run github.com/golangci/golangci-lint/v2/cmd/golangci-lint@v2.12.2 run && go vet ./... && go mod tidy && git diff --exit-code go.mod go.sum && go test ./...`
- **web:** `cd /workspace/prog-strength-web && npm run lint && npm run format:check && npm run typecheck && npm run test && npm run build`
- **mcp:** `cd /workspace/prog-strength-mcp && ruff check src tests && pytest tests/`

Never bypass a hook, never add `//nolint`, never weaken a test. If a check fails, fix the code.

---

## Task 1: API — endurance elevation shape, hiking type, repository seams, summarizer

**Files:**
- Create: `internal/db/migrations/044_activity_hike_elevation.sql`
- Modify: `internal/activity/activity_type.go`
- Modify: `internal/activity/model.go` (Activity struct)
- Modify: `internal/activity/endurance_descriptor.go` (EnduranceDetails, enduranceLabel, enduranceSummarize)
- Modify: `internal/activity/endurance_detail_store.go` (Load/Save column lists)
- Modify: `internal/activity/sqlite_repository.go` (joins, detailCoalesce, activityColumns, scanActivity, detailTable, Create INSERT)
- Modify: `internal/activity/tcx_summarizer.go` (elevationGain → elevationProfile)
- Modify: `internal/activity/handler.go` (activityDTO + toActivityDTO elevation fields)
- Modify: `internal/server/server.go` (register hiking descriptor)
- Tests: `internal/activity/tcx_summarizer_test.go`, `internal/activity/endurance_descriptor_test.go`, `internal/activity/sqlite_repository_test.go` (or a new `hiking_test.go`)

### Migration `044_activity_hike_elevation.sql`

Mirror `activity_run_details` (from 042) plus the elevation triple, then ALTER the other four tables. Nullable ADD COLUMN with no default is O(1) in SQLite.

```sql
-- migrations/044_activity_hike_elevation.sql
-- Hiking is the fifth endurance-shaped type. It reuses the shared endurance
-- detail shape, so this migration only (a) creates activity_hike_details and
-- (b) widens the shared shape with the elevation triple (loss/high/low) on
-- every existing detail table. No CHECK on activities.activity_type — the Go
-- registry owns the type enum (see migration 042). The elevation columns are
-- nullable with no default: a NULL means "no altitude in the source track",
-- distinct from a genuinely flat 0.

CREATE TABLE activity_hike_details (
    activity_id TEXT PRIMARY KEY REFERENCES activities(id) ON DELETE CASCADE,
    distance_meters REAL NOT NULL,
    raw_distance_meters REAL NOT NULL DEFAULT 0,
    avg_pace_sec_per_km REAL,
    best_pace_sec_per_km REAL,
    elevation_gain_meters REAL,
    elevation_loss_meters REAL,
    elevation_high_meters REAL,
    elevation_low_meters REAL,
    environment TEXT NOT NULL DEFAULT 'outdoor' CHECK(environment IN ('outdoor','indoor')),
    route_geojson TEXT
);

ALTER TABLE activity_run_details ADD COLUMN elevation_loss_meters REAL;
ALTER TABLE activity_run_details ADD COLUMN elevation_high_meters REAL;
ALTER TABLE activity_run_details ADD COLUMN elevation_low_meters REAL;

ALTER TABLE activity_walk_details ADD COLUMN elevation_loss_meters REAL;
ALTER TABLE activity_walk_details ADD COLUMN elevation_high_meters REAL;
ALTER TABLE activity_walk_details ADD COLUMN elevation_low_meters REAL;

ALTER TABLE activity_cycle_details ADD COLUMN elevation_loss_meters REAL;
ALTER TABLE activity_cycle_details ADD COLUMN elevation_high_meters REAL;
ALTER TABLE activity_cycle_details ADD COLUMN elevation_low_meters REAL;

ALTER TABLE activity_other_details ADD COLUMN elevation_loss_meters REAL;
ALTER TABLE activity_other_details ADD COLUMN elevation_high_meters REAL;
ALTER TABLE activity_other_details ADD COLUMN elevation_low_meters REAL;
```

> Confirm the migration runner picks up bare-numbered files in `internal/db/migrations/` (042/043 are there). Follow how they are embedded/loaded (`go:embed`) — no code change should be needed, but verify a fresh `dbtest.New` applies 044.

### `activity_type.go`

Add the constant and include it in `Valid()`:

```go
	ActivityHiking  ActivityType = "hiking"
```
```go
func (t ActivityType) Valid() bool {
	switch t {
	case ActivityRunning, ActivityWalking, ActivityCycling, ActivityHiking, ActivityOther, ActivityStrengthTraining:
		return true
	}
	return false
}
```
Do NOT add hiking to `normalizeActivityType` — TCX `<Sport>` has no hiking value; hiking arrives only via the explicit override (Task 2).

### `model.go` — Activity struct

After `ElevationGainMeters *float64` add:
```go
	ElevationLossMeters *float64
	ElevationHighMeters *float64
	ElevationLowMeters  *float64
```

### `endurance_descriptor.go`

`EnduranceDetails` gains three fields (omitempty, matching ElevationGainMeters):
```go
	ElevationLossMeters *float64 `json:"elevation_loss_meters,omitempty"`
	ElevationHighMeters *float64 `json:"elevation_high_meters,omitempty"`
	ElevationLowMeters  *float64 `json:"elevation_low_meters,omitempty"`
```
`enduranceLabel` gains a hiking case returning `"Hike"`.

`enduranceSummarize` — hiking is the one deviation from the shared two-chip card. When the type is `ActivityHiking` AND `ElevationGainMeters` is non-nil, render three chips `distance · gain · duration`; when gain is nil, fall through to the standard two-chip render. Gain string format: whole feet with thousands separator + ` ft ↑` (e.g. `3,450 ft ↑`). Use the existing feet conversion/formatting helper if one exists in the package; otherwise convert meters→feet (`* 3.28084`), round, and format with a thousands separator. Example target card: `6.7 mi · 3,450 ft ↑ · 5:12:40`. Keep the zero-distance guard (duration-only) unchanged.

Implementation sketch (adapt to existing helpers/formatters in the file):
```go
func enduranceSummarize(t ActivityType) func(a Activity, details any) Summary {
	label := enduranceLabel(t)
	return func(a Activity, details any) Summary {
		title := label
		if a.Name != nil && strings.TrimSpace(*a.Name) != "" {
			title = *a.Name
		}
		meters := a.DistanceMeters
		var gain *float64
		if d, ok := details.(*EnduranceDetails); ok && d != nil {
			meters = d.DistanceMeters
			gain = d.ElevationGainMeters
		}
		if gain == nil {
			gain = a.ElevationGainMeters
		}
		duration := FormatDuration(float64(a.DurationSeconds))
		if meters <= 0 {
			return Summary{Title: title, Subtitle: duration, Metrics: []string{duration}}
		}
		distance := FormatMiles(meters)
		if t == ActivityHiking && gain != nil {
			g := formatGainFeet(*gain) // "3,450 ft ↑"
			return Summary{
				Title:    title,
				Subtitle: distance + " · " + g + " · " + duration,
				Metrics:  []string{distance, g, duration},
			}
		}
		return Summary{
			Title:    title,
			Subtitle: distance + " · " + duration,
			Metrics:  []string{distance, duration},
		}
	}
}
```
Add a small `formatGainFeet(meters float64) string` helper (whole feet, thousands separator, ` ft ↑`). Check `summary.go`/`FormatMiles` for an existing feet formatter before adding a new one; reuse if present.

### `endurance_detail_store.go`

Extend the `Load` SELECT and scan, and the `Save` INSERT…ON CONFLICT column list + excluded set, to include `elevation_loss_meters`, `elevation_high_meters`, `elevation_low_meters` (all `*float64` → `sql.NullFloat64`-style scan is not needed since existing code scans `*float64` directly via the struct pointer fields; follow the existing pattern which scans `&d.ElevationGainMeters` directly). Place the three new columns immediately after `elevation_gain_meters` in every list.

### `sqlite_repository.go`

1. `activityJoins`: add `LEFT JOIN activity_hike_details hd ON hd.activity_id = a.id`.
2. `detailCoalesce`: include the `hd` alias:
```go
	expr := "rd." + col + ", wd." + col + ", cd." + col + ", od." + col + ", hd." + col
```
3. `activityColumns`: after the existing `raw_distance_meters` coalesce (keep it last of the current set OR append — pick append for a clean scan order), append three projections:
```go
	`,
	` + detailCoalesce("elevation_loss_meters", "") + `,
	` + detailCoalesce("elevation_high_meters", "") + `,
	` + detailCoalesce("elevation_low_meters", "")
```
4. `scanActivity`: add three `sql.NullFloat64` locals and scan them in the SAME position they were appended in `activityColumns` (i.e. append to the end of the `s.Scan(...)` arg list), then map Valid→`*float64` onto the new Activity fields.
5. `detailTable`: add `case ActivityHiking: return "activity_hike_details"`.
6. `Create` detail INSERT: the shared INSERT writes to whatever `detailTable(a.ActivityType)` returns; extend its column list and value list to include `elevation_loss_meters, elevation_high_meters, elevation_low_meters` with `a.ElevationLossMeters, a.ElevationHighMeters, a.ElevationLowMeters`. All five detail tables now have these columns, so the shared INSERT is safe for every endurance type.

> The elevation triple must be scanned in exactly the same order in every query that uses `activityColumns` — it is one constant, so appending once covers `List`, `ListInRange`, `Get`, `GetBySourceActivityID`, `SummariesByIDs`, `readActivitySummary`.

### `tcx_summarizer.go`

Rename `elevationGain` → `elevationProfile` returning four `*float64` (gain, loss, high, low) in one pass. Skip points with no altitude; only advance the previous-altitude cursor on points that have altitude (a gap does not manufacture a cliff). All four nil when no trackpoint carried altitude; a flat-but-present track returns gain=0, loss=0, high=low=that altitude.

```go
// elevationProfile derives total ascent, total descent (stored positive),
// high point, and low point from consecutive trackpoint altitudes in one
// pass. Points with no altitude are skipped and the previous-altitude cursor
// only advances on a point that has one, so a gap in the altitude stream does
// not manufacture a cliff. All four are nil when NO trackpoint carried
// altitude — distinct from a genuinely flat route, which returns 0/0/alt/alt.
func elevationProfile(tps []parsedTrackpoint) (gain, loss, high, low *float64) {
	var prev *float64
	var g, l float64
	var hi, lo float64
	seen := false
	for _, tp := range tps {
		if tp.AltitudeMeters == nil {
			continue
		}
		alt := *tp.AltitudeMeters
		if !seen {
			hi, lo = alt, alt
			seen = true
		} else {
			if d := alt - *prev; d > 0 {
				g += d
			} else {
				l += -d
			}
			if alt > hi {
				hi = alt
			}
			if alt < lo {
				lo = alt
			}
		}
		prev = tp.AltitudeMeters
	}
	if !seen {
		return nil, nil, nil, nil
	}
	return &g, &l, &hi, &lo
}
```
In `summarize`, replace `ElevationGainMeters: elevationGain(tps)` with a call that captures all four and assigns them to `a.ElevationGainMeters`, `a.ElevationLossMeters`, `a.ElevationHighMeters`, `a.ElevationLowMeters`. This runs for every endurance type — Denver runs pick up loss/high/low for free.

### `handler.go` — DTO

Add to `activityDTO` after `ElevationGainMeters` (pointers WITHOUT omitempty, present-as-null convention):
```go
	ElevationLossMeters *float64 `json:"elevation_loss_meters"`
	ElevationHighMeters *float64 `json:"elevation_high_meters"`
	ElevationLowMeters  *float64 `json:"elevation_low_meters"`
```
Map them in `toActivityDTO`.

### `server.go` — registration

Alongside the four existing endurance registrations (around line 543-548), add:
```go
		activity.NewEnduranceDescriptor(activity.ActivityHiking, activity.NewSQLiteEnduranceDetailStore(database, activity.ActivityHiking)),
```
Note the real signature is `NewSQLiteEnduranceDetailStore(db, ActivityType)` (the SOW's `"hike"` string is shorthand — pass `activity.ActivityHiking`; `detailTable` maps it to `activity_hike_details`). `NewRegistry` panics on a duplicate type, so a wiring mistake fails boot loudly. `MountRoutes` stays nil.

### Tests (TDD — write first, watch fail, implement, watch pass)

Add to the relevant `_test.go` files (or a new `internal/activity/hiking_test.go` for the round-trip):
- `elevationProfile`: (a) normal ascent+descent → gain/loss/high/low correct; (b) no altitude anywhere → all four nil; (c) altitude gap (some points nil in the middle) → no phantom cliff (gain/loss ignore the gap); (d) flat route (constant altitude present) → gain 0, loss 0, high==low==const (non-nil).
- `enduranceSummarize(ActivityHiking)`: with gain → `Metrics` has 3 chips and subtitle `distance · gain · duration`; with nil gain → 2-chip standard card.
- Hiking round-trip: build an `Activity{ActivityType: ActivityHiking, DistanceMeters>0, ElevationGain/Loss/High/Low set}`, `repo.Create`, then `List`/`Get`, assert the elevation triple survives and the detail row lands in `activity_hike_details`; `SoftDelete` and confirm gone (CASCADE covered by FK).
- The existing `contract_test.go` MUST keep passing untouched — do not edit it. If it fails, a switch outside the declared seams was touched; fix that instead.

**Gate:** run the full api gate (build, golangci-lint v2.12.2, vet, mod tidy no drift, test). Commit `feat: register hiking endurance type with elevation profile`.

---

## Task 2: API — `POST /activities/tcx` optional `activity_type` override

**Files:**
- Modify: `internal/activity/ingest.go` (IngestTCX signature)
- Modify: `internal/activity/handler.go` (uploadTCX form field + validation; update the IngestTCX call)
- Tests: `internal/activity/handler_test.go` (or wherever uploadTCX is tested)

### `ingest.go`

Add a `typeOverride ActivityType` parameter (empty = derive from TCX):
```go
func IngestTCX(ctx context.Context, repo Repository, userID string, source IngestSource, r io.Reader, typeOverride ActivityType) (Activity, error) {
	...
	actType := typeOverride
	if actType == "" {
		actType = normalizeActivityType(parsed.Sport, source)
	}
	a := summarize(parsed, actType)
	...
}
```

### `handler.go` — uploadTCX

After `r.ParseMultipartForm`, read the optional field and validate BEFORE calling IngestTCX:
```go
	var override ActivityType
	if raw := strings.TrimSpace(r.FormValue("activity_type")); raw != "" {
		t := ActivityType(raw)
		if h.registry != nil {
			if _, err := h.registry.Lookup(t); err != nil {
				// Unknown type: 422 listing the valid set (Lookup's message).
				httpresp.Error(w, http.StatusUnprocessableEntity, err.Error())
				return
			}
		}
		// Registered but non-endurance (e.g. strength_training): the endurance
		// summarizer would write a garbage detail row. Strength ingest has its
		// own path. detailTable(t) == "" identifies base-only / non-endurance.
		if detailTable(t) == "" {
			httpresp.ErrorWithCode(w, http.StatusBadRequest,
				"activity_type "+raw+" cannot be ingested from a TCX file", "invalid_activity_type")
			return
		}
		override = t
	}
```
Pass `override` as the new IngestTCX argument. When the field is absent, `override == ""` and behavior is byte-for-byte unchanged. Verify `strings` and `httpresp` are imported (strings likely already is; add if not). Confirm the exact `httpresp` helper names against the file (`Error`, `ErrorWithCode` are used elsewhere in the handler).

The success branch's timeline-publish / plan-match stays gated on `a.ActivityType == ActivityRunning` — a hike is stored and listed but publishes no feed post (Non-Goal). No change there.

### Tests
Table-driven over the multipart upload, using a small valid TCX fixture (reuse an existing testdata TCX):
- absent field → today's behavior (type derived from `<Sport>`; a Running fixture stays running).
- `activity_type=hiking` → stored as hiking, 201, elevation profile present.
- `activity_type=kickboxing` (unregistered) → 422, body lists valid types.
- `activity_type=strength_training` (registered, non-endurance) → 400, code `invalid_activity_type`.

The handler test env must wire a registry that includes hiking (mirror server.go's registrations). **Gate + commit** `feat: accept activity_type override on POST /activities/tcx`.

---

## Task 3: Web — api client, hiking-stats, elevation formatter, design tokens

**Files:**
- Modify: `lib/api.ts`
- Create: `lib/hiking-stats.ts` + `lib/hiking-stats.test.ts`
- Modify: `lib/distance-unit-context.tsx` (+ test)
- Modify: `app/globals.css`
- Modify: `lib/activity-colors.ts`
- Modify: `components/calendar/derivations.ts`

### `lib/api.ts`
- `ActivityType` union gains `"hiking"`.
- The activity/session type(s) that carry `elevation_gain_meters` gain `elevation_loss_meters: number | null`, `elevation_high_meters: number | null`, `elevation_low_meters: number | null` (match the existing nullable convention).
- Add `listHikingSessions(token, opts)` wrapping `listActivities(token, { ...opts, type: "hiking" })`, mirroring `listRunningSessions`.
- Rename `importRunningTcx(token, file)` → `importActivityTcx(token, file, activityType?)`. When `activityType` is provided, append it as the `activity_type` multipart form field. Return type stays the unified session type (rename only; do NOT migrate the `RunningSession` alias). Update the single call site (UploadTCXModal — done in Task 4). Keep behavior identical when `activityType` is omitted.

### `lib/hiking-stats.ts` (pure, mirrors `lib/activities-overview-stats.ts`)
Export a pure function computing the six tiles over a windowed list of hiking sessions. Nulls are SKIPPED, never coerced to 0.
```
totalDistanceMeters = Σ distance_meters
totalGainMeters     = Σ elevation_gain_meters      (skip null)
highPointMeters     = max(elevation_high_meters)   (skip null; null when none)
lowPointMeters      = min(elevation_low_meters)    (skip null; null when none)
avgPaceSecPerKm     = Σ duration_seconds / (Σ distance_meters / 1000)   (null if total distance 0)
gainPerMileMeters   = totalGainMeters / (totalDistanceMeters / METERS_PER_MILE)  (null if no gain/distance)
```
Return a typed object with these six values where high/low/gain-derived tiles are `number | null` so the view renders an em-dash for null. Average pace is total-time-over-total-distance (weighted), NOT a mean of per-hike paces — matches running's `RecentAvgPaceSecPerKm` policy.

Tests (`lib/hiking-stats.test.ts`): weighted-pace formula; null-skipping for each of gain/high/low; an all-null window → high/low/gain null (em-dash); an empty window → sensible zeros/nulls.

### `lib/distance-unit-context.tsx`
Add a pure `formatElevationValue(meters: number | null, unit: DistanceUnit): string` next to `formatDistanceValue`/`formatPaceValue`: whole feet with a thousands separator under `mi` (`meters * 3.28084`, rounded, `toLocaleString`), whole meters under `km`, em-dash (`—`) for null. Expose a `formatElevation(meters)` binding on the context value (like `formatDistance`). Add a unit test for feet/meters/null.

### `app/globals.css`
Add the clay hike triple next to the other `--discipline-*` vars, and any `@theme inline`/Tailwind alias the existing discipline vars have:
```css
  --discipline-hike-bg: #2a201c;
  --discipline-hike-fg: #c9a690;
  --discipline-hike-dot: #b08e77;
```

### `lib/activity-colors.ts`
Widen `MappedDiscipline` to `"run" | "lift" | "hike"`; add `hike` entries to `ACTIVITY_COLORS` (dot/bg/fg → the three hike vars) and `ACTIVITY_RING` (`focus-visible:ring-[var(--discipline-hike-dot)]`). Ensure `activityColors(type)`/the resolver maps a hiking activity to `hike`.

### `components/calendar/derivations.ts`
Add `"hike"` to the `Discipline` union and map hiking activities to `"hike"` in whatever `disciplineOf`/derivation switch classifies types (running→run, strength→lift). This is the one intentional discipline-classification edit; the activity-colors resolver keys off it.

**Gate:** web gate. Commit `feat: hiking api client, stats, elevation formatter, clay discipline hue`.

---

## Task 4: Web — shared component extraction + activities shell + upload modal

**Files:**
- Move: `ElevationChart`, `RunRouteMap`, `HeartRateRecap`, `SectionKicker`, `NotesEditor` from `app/(app)/running/[id]/_components/` and `app/(app)/running/_components/` → `components/activity-detail/`, updating every import (running detail page, RunningView, etc.). Only move the components the hiking detail page will reuse; leave running-only components (SplitsSpine, PaceRecap, CalibrateDistanceModal, TreadmillBadge) where they are.
- Modify: `app/(app)/activities/page.tsx`
- Modify: `app/(app)/running/_components/UploadTCXModal.tsx` (or its current path)

### Extraction
Move each reused component file into `components/activity-detail/<Name>.tsx`, keep the component API identical, and fix all imports across route groups so nothing reaches sideways into `app/(app)/running/...`. Run the web gate after the move to confirm running still compiles and its tests pass BEFORE adding hiking consumers.

### Activities shell (`app/(app)/activities/page.tsx`)
- `View` union gains `"hiking"`; the `?view=` parse accepts `"hiking"`; `setView` already generic.
- Toolbar: add a Hiking entry with a two-peak mountain glyph in the existing 16px / 1.75-stroke inline-SVG house style (match the sibling icons).
- Upload TCX button condition widens from `view === "running"` to `view === "running" || view === "hiking"`.
- Pass the active view down to the upload modal so it can preselect the sport, and render `<HikingView … />` when `view === "hiking"` (HikingView built in Task 5 — for this task, wire the branch; a stub import is acceptable only if Task 5 lands in the same branch before the gate — prefer ordering Task 5 before enabling the branch, or add the branch here and let Task 5 create the component. Coordinate so the branch and component land together and the build stays green.)

### UploadTCXModal
Add a small pill row (Run / Hike / Walk / Ride) matching the timeframe pills in the header, defaulting to the tab it was opened from (via a new `defaultSport`/`view` prop). The chosen value is always sent explicitly as the `activity_type` arg to `importActivityTcx` (even when the default applies). Update the call from `importRunningTcx` → `importActivityTcx(token, file, sport)`.

**Gate:** web gate. Commit `feat: extract shared activity-detail components; hiking tab + sport selector`.

---

## Task 5: Web — HikingView + hike history list

**Files:**
- Create: `components/activities/hiking-view.tsx`
- Create: hike history list component (mirror `RunHistoryList`, e.g. `app/(app)/hiking/_components/HikeHistoryList.tsx` or a `components/activities/` sibling — follow where RunHistoryList lives)
- Create: `components/activities/hiking-view.test.tsx`

Follow `RunningView`'s structure — `refetch` deriving `[since, until)` from `days`, the same 401 handling, the same error surface — with two simplifications: ONE fetch (`listHikingSessions`, no metrics endpoint) and NO analytics card. Render a `grid-cols-2 md:grid-cols-3` of six `StatTile`s computed via `lib/hiking-stats.ts`, above the hike history list. Tile labels: DISTANCE, VERTICAL GAIN, HIGH POINT, LOW POINT, AVG PACE, GAIN / MI — the last two labels swap to `AVG PACE`/`/KM` and `GAIN / KM` under the `km` unit (i.e. `GAIN / MI` ↔ `GAIN / KM`, and the pace tile sub-unit follows unit). Elevation values render via `formatElevation`; distance/pace via the existing context helpers. Null tiles render an em-dash. There is no window-level loss tile (loss is detail-page only).

Test (`hiking-view.test.tsx`) mirrors `running-view.test.tsx`: mock `listHikingSessions`, render, assert the six tiles and the history list appear, and that an all-null-elevation window shows em-dashes.

**Gate:** web gate. Commit `feat: hiking tab view with six timeframe tiles`.

---

## Task 6: Web — `/hiking/[id]` detail page + elevation cleanup

**Files:**
- Create: `app/(app)/hiking/[id]/page.tsx` (+ any hiking-only sub-components)
- Modify: `components/calendar/run-digest.tsx` (~line 38)
- Modify: `components/activities/overview/instruments.tsx` (~line 342)
- Modify: `app/(app)/running/[id]/page.tsx` (~line 349)

### `/hiking/[id]`
New detail page on the activity session-recap grammar (design-system.md § Activity session-recap), reusing the extracted `ElevationChart`, `RunRouteMap`, `HeartRateRecap`, `SectionKicker`, `NotesEditor` from `components/activity-detail/`. Header stat tiles lead with distance, vertical gain, and duration. The elevation profile is the page centerpiece (not a supporting chart). Deliberately absent: `SplitsSpine`, `PaceRecap`, `CalibrateDistanceModal`. Fetch the single activity via the existing get-activity client call (the same one the running detail page uses), typed as the unified session. Elevation renders via `formatElevation`. Use the hike discipline hue (`--discipline-hike-*`) for section kickers per the grammar.

### Elevation cleanup (adopt the shared helper)
Replace the three open-coded elevation formatters with `formatElevation`/`formatElevationValue`:
- `components/calendar/run-digest.tsx:~38` — inline `FEET_PER_METER` conversion.
- `components/activities/overview/instruments.tsx:~342` — inline `FEET_PER_METER` conversion.
- `app/(app)/running/[id]/page.tsx:~349` — hardcoded `"m"` regardless of unit (the bug: same run shows meters on detail, feet on calendar). After this, all three honor the user's unit. Remove now-dead inline `FEET_PER_METER` constants. Nothing else is refactored.

**Gate:** web gate. Commit `feat: hiking detail page; unify elevation formatting`.

---

## Task 7: MCP — hiking in tool docstrings

**Files:**
- Modify: `src/prog_strength_mcp/activities.py`
- Modify: `README.md`

Add `'hiking'` to the enumerated activity-type lists so the agent knows hiking exists:
- module docstring registered-types list: `running, walking, cycling, hiking, other, and strength_training`.
- `log_activity` `activity_type` Field description registered set: add `'hiking'` after `'cycling'`.
- `log_activity` docstring endurance-types example: `('running', 'walking', 'cycling', 'hiking')`.
- `list_activities` docstring filter list: add `'hiking'`.
- `README.md` tools table `log_activity` row: `(running, walking, cycling, hiking, other, strength_training)`.

No logic changes. **Gate:** `ruff check src tests && pytest tests/`. Commit `docs: list hiking activity type in MCP tool docstrings`.

---

## Task 8: Docs — design-system v0.4.4, SOW status flip, PRs

**Files:**
- Modify: `design-system.md`
- Modify: `sows/hiking-activity-type.md`
- (this plan is already in `plans/`)

### `design-system.md`
- Bump `**Status**: v0.4.3` → `v0.4.4` and `**Last updated**` to `2026-07-30`.
- Add a **hike** row to the Activity tonal hues table:
  `| **hike** (clay) | \`#2a201c\` | \`#c9a690\` | \`#b08e77\` |`
- Add a v0.4.4 changelog entry: added the **hike** discipline hue (desaturated clay), registered in `lib/activity-colors.ts`; first new discipline hue since v0.3; no re-tone of the v0.4 foundation; provenance `sows/hiking-activity-type.md`.
- Do NOT correct the run/lift drift or the lift/accent collision — those are Open Questions in the SOW, not in-scope Goals.

### SOW status flip (`sows/hiking-activity-type.md`)
- Frontmatter `status: draft` → `status: shipped`.
- Body header `**Status**: Draft` → `Shipped`.
- Body header `**Last updated**: 2026-07-29` → `2026-07-30`.

Commit `docs: mark hiking-activity-type as shipped`.

### PRs
Push every modified repo's `feat/hiking-activity-type` branch (only after its local gate is green) and open a PR against `main` in each, plus the docs PR. The docs PR body uses the required template from the task brief (Shipped header, SOW link, Implementation PRs bullets in `repos:` order, Deployment order, Verification after rollout, status-flip footer).

---

## Self-review checklist (run before dispatching Task 1)
- Spec coverage: every SOW Goal maps to a task (type reg → T1; override → T2; elevation one-pass → T1; hiking tab six tiles → T5; `/hiking/[id]` → T6; formatElevation → T3/T6; clay hue → T3/T8; MCP docstrings → T7). ✅
- Non-goals respected: no timeline publish, no mobile, no best-efforts, no plan-match, no overview inclusion, no manual hike form, no calibration. ✅
- Contract test untouched and must stay green (T1). ✅
- Type consistency: `ActivityHiking = "hiking"`; detail table `activity_hike_details`; JSON keys `elevation_loss_meters`/`elevation_high_meters`/`elevation_low_meters` consistent across Go DTO, EnduranceDetails, and TS types. ✅
