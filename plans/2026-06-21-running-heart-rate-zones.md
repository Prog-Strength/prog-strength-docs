# Running Heart Rate Zones Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Compute a five-zone, percent-of-max-HR breakdown for every running activity on the backend, surface it as an additive `heart_rate_zones` block on `GET /activities/{id}`, and render a zones widget on the web running detail page.

**Architecture:** A new pure Go package `internal/hrzones` owns all zone logic (zone model, reference-max-HR estimation ladder, classification, time-in-zone accumulation) with no DB/HTTP/`activity` coupling. The activity repository computes the historical `hrzones.Stats` a user needs; the activity handler estimates the reference, computes the result, and attaches it as a DTO. Tunables live in a new `[hr_zones]` section of `config.toml`. The web client adds a hand-rolled (no charting dep) stacked-bar widget that reads the new response block.

**Tech Stack:** Go 1.25 (chi, sqlite3, BurntSushi/toml), Next.js 16 / React 19 / TypeScript / Tailwind v4, Vitest.

---

## File Structure

### prog-strength-api
- **Create** `internal/hrzones/hrzones.go` — `Config`, `Stats`, `Confidence`, `Reference`, `Zone`, `Result`, `Trackpoint`, `Engine`, `New`, `EstimateReference`, `Compute`, and the internal helpers (clamp, percentile, classify, zone-bpm bounds).
- **Create** `internal/hrzones/doc.go` — package doc: zone model, estimation ladder, confidence states, documented ingest-cache growth path.
- **Create** `internal/hrzones/hrzones_test.go` — table-driven engine tests.
- **Modify** `config.toml` (repo root) — add `[hr_zones]` section.
- **Modify** `internal/config/config.go` — `HRZonesConfig` struct on `Config`, `fileConfig` `[hr_zones]` block, mapping in `Load`.
- **Modify** `internal/config/config_test.go` — assert `[hr_zones]` parses.
- **Modify** `internal/activity/repository.go` — add `RecentHRStats` to the `Repository` interface.
- **Modify** `internal/activity/sqlite_repository.go` — implement `RecentHRStats`.
- **Modify** `internal/activity/sqlite_repository_test.go` — `RecentHRStats` tests.
- **Modify** `internal/activity/handler.go` — `hrEngine` field + `SetHRZonesEngine`, `heartRateZonesDTO`, attach in `toActivityDTO` path.
- **Modify** `internal/activity/handler_test.go` — assert the block (happy path) and its omission (no-HR).
- **Modify** `internal/server/server.go` — construct the engine from config and wire it into the activity handler.

### prog-strength-web
- **Modify** `lib/api.ts` — `HeartRateZone` + `HeartRateZones` types; `heart_rate_zones?` on `RunningSession`.
- **Modify** `app/globals.css` — `--zone-1..5` periwinkle ramp tokens (+ Tailwind aliases if the file defines them for other tokens).
- **Modify** `lib/format.ts` — `formatPercent` helper (if not already present).
- **Create** `app/(app)/running/[id]/_components/HeartRateZones.tsx` — the widget.
- **Create** `app/(app)/running/[id]/_components/HeartRateZones.test.tsx` — component tests.
- **Modify** `app/(app)/running/[id]/page.tsx` — render `<HeartRateZones>` under the stats band.

### prog-strength-docs
- **Create** `notes/hrzones-engine.md` — companion note describing the engine as the home for HR-zone estimation logic and its evolution.
- **Modify** `sows/running-heart-rate-zones.md` — status flip (handled in the workflow's final step, not a coding task).

---

## Shared Reference: the engine API (authoritative signatures)

Every task below uses these exact names. Do not rename.

```go
package hrzones

type Config struct {
    PopulationDefaultMaxHR int
    CalibratedRunThreshold int
    RecencyWindowDays      int
    MinReferenceBpm        int
    MaxReferenceBpm        int
    ZoneUpperBounds        []float64 // ascending interior bounds, e.g. [0.60,0.70,0.80,0.90]
    ZoneNames              []string  // len == len(ZoneUpperBounds)+1
}

type Stats struct {
    RecentHRSamplesP99 *int
    HistoryRunCount    int
    CurrentRunP99      *int
}

type Confidence string

const (
    ConfidenceEstimated   Confidence = "estimated"
    ConfidenceCalibrating Confidence = "calibrating"
    ConfidenceCalibrated  Confidence = "calibrated"
)

type Reference struct {
    MaxHRBpm   int
    Source     string // "population_default" | "current_run" | "p99_recent_runs"
    Confidence Confidence
}

type Trackpoint struct {
    ElapsedSeconds int
    HeartRateBpm   *int
}

type Zone struct {
    Number      int
    Name        string
    LowerPct    float64
    UpperPct    float64
    MinBpm      int
    MaxBpm      int
    TimeSeconds int
    TimePct     float64
}

type Result struct {
    Model          string // "percent_max_hr"
    Reference      Reference
    TotalHRSeconds int
    Zones          []Zone
    Calibrating    bool
}

type Engine struct { cfg Config }

func New(cfg Config) *Engine
func (e *Engine) EstimateReference(s Stats) Reference
func (e *Engine) Compute(ref Reference, tps []Trackpoint) (Result, bool)

// P99 returns the 99th-percentile of samples (nearest-rank), nil if empty.
func P99(samples []int) *int
```

Response JSON shape (authoritative — match field names exactly):

```json
"heart_rate_zones": {
  "model": "percent_max_hr",
  "max_hr_reference_bpm": 191,
  "reference_source": "p99_recent_runs",
  "reference_confidence": "calibrated",
  "calibrating": false,
  "total_hr_seconds": 3170,
  "zones": [
    { "zone": 1, "name": "Recovery", "lower_pct": 0.0, "upper_pct": 0.60, "min_bpm": 0, "max_bpm": 114, "time_seconds": 240, "time_pct": 0.076 }
  ]
}
```

---

## Task 1: hrzones types, zone model, classification & Compute

**Files:**
- Create: `internal/hrzones/hrzones.go`
- Create: `internal/hrzones/hrzones_test.go`

This task builds the pure engine *except* the estimation ladder (Task 2). It includes `New`, the `Compute` method, `P99`, and the internal classification/zone-bounds/clamp helpers.

**Semantics to implement:**

- `New(cfg Config) *Engine` stores cfg.
- **Zone bounds:** five zones from four interior `ZoneUpperBounds`. Zone i (0-indexed) has `LowerPct = bounds[i-1]` (0.0 for zone 1) and `UpperPct = bounds[i]` (1.0 for the last zone). Lower inclusive, upper exclusive — except the top zone's upper is the open-ended ceiling.
- **Per-zone bpm range** from a reference `maxHR`: `MinBpm = round(LowerPct * maxHR)`, `MaxBpm = round(UpperPct * maxHR) - 1` for all zones except the top zone, whose `MaxBpm = maxHR`. Zone 1 `MinBpm = 0`. Use `int(math.Round(...))`. (Worked example, maxHR=191, bounds [.6,.7,.8,.9]: Z1 0–114, Z2 115–133, Z3 134–152, Z4 153–171, Z5 172–191 — matches the SOW table.)
- **Classification of a bpm value:** find the lowest zone whose `UpperPct*maxHR` (exclusive) the value is under; the top zone catches everything `>=` the last interior bound. Classify by comparing `bpm` against `UpperPct*maxHR` using the *fractional* threshold (compare `float64(bpm) < UpperPct*float64(maxHR)`), NOT the rounded `MaxBpm`, so the boundary math is done once and consistently. The top zone is the fallthrough.
- **Compute:** iterate consecutive trackpoint pairs `(i-1, i)`. For each pair where *both* endpoints have non-nil `HeartRateBpm`: `dt = tps[i].ElapsedSeconds - tps[i-1].ElapsedSeconds`; classify by the **mean** of the two endpoints' bpm (`mean = (a+b)/2.0`, a float, classified against fractional thresholds); add `dt` to that zone's `TimeSeconds` and to `TotalHRSeconds`. Skip pairs where either endpoint lacks HR, and skip non-positive `dt` defensively. After accumulation, set each zone's `TimePct = TimeSeconds / TotalHRSeconds` (float; 0 when `TotalHRSeconds == 0`). Populate every zone's static fields (Number, Name, LowerPct, UpperPct, MinBpm, MaxBpm) from the reference. `Result.Model = "percent_max_hr"`, `Result.Calibrating = ref.Confidence != ConfidenceCalibrated`.
- **ok=false:** return `(Result{}, false)` when `TotalHRSeconds == 0` (no usable HR interval). Otherwise `(result, true)`.
- `P99(samples []int) *int`: nearest-rank percentile. Sort a copy ascending; `rank = ceil(0.99 * n)`; return `&sorted[rank-1]`; nil if `len==0`. (For n=1 returns that sample; this is the spike-resistant statistic the repo/handler feed in.)

- [ ] **Step 1: Write failing tests** in `internal/hrzones/hrzones_test.go` covering:
  - `TestZoneBpmBounds`: maxHR=191, default bounds/names → the exact SOW table ranges.
  - `TestClassifyBoundaries`: a bpm exactly at a boundary (e.g. mean=114.6→Z2 at maxHR 191; mean exactly `0.70*191=133.7`) lands in the upper (exclusive-lower-of-next) zone; values just below/above each interior boundary land correctly; `>= 0.90*max` → Z5; very low → Z1.
  - `TestComputeTimeInZone`: trackpoints with mixed 9s/10s `dt`, a null-HR pair (skipped), verify per-zone `TimeSeconds`, `TotalHRSeconds` == sum of classified dts, and `sum(TimePct)` ≈ 1.0 (within 1e-9).
  - `TestComputeNoHR`: all-nil HR (or fewer than 2 HR-bearing points) → `ok == false`.
  - `TestP99`: `[100..200]`-style slice → expected nearest-rank value; single sample → that sample; empty → nil; an injected lone 220 among 150s does not become the p99.

- [ ] **Step 2: Run tests, verify they fail** — `cd /workspace/prog-strength-api && go test ./internal/hrzones/` → FAIL (undefined symbols).

- [ ] **Step 3: Implement** `internal/hrzones/hrzones.go` with the types from the Shared Reference and the semantics above. Keep it pure (imports: `math`, `sort` only). Short `// why` comments on the p99 choice and the boundary (fractional-threshold) decision.

- [ ] **Step 4: Run tests, verify pass** — `go test ./internal/hrzones/` → PASS.

- [ ] **Step 5: Commit** — `git add internal/hrzones && git commit -m "feat(hrzones): zone model, classification and time-in-zone engine"`

## Task 2: hrzones EstimateReference ladder

**Files:**
- Modify: `internal/hrzones/hrzones.go`
- Modify: `internal/hrzones/hrzones_test.go`

Implement `EstimateReference(s Stats) Reference` exactly per the SOW ladder, then clamp:

1. **Calibrated** — `s.HistoryRunCount >= cfg.CalibratedRunThreshold && s.RecentHRSamplesP99 != nil`: `MaxHRBpm = *s.RecentHRSamplesP99`, `Source = "p99_recent_runs"`, `Confidence = ConfidenceCalibrated`.
2. **Calibrating** — `s.HistoryRunCount > 0 && s.RecentHRSamplesP99 != nil` (and not calibrated): `MaxHRBpm = max(*s.RecentHRSamplesP99, currentOr0)`, `Source = "p99_recent_runs"`, `Confidence = ConfidenceCalibrating`, where `currentOr0 = *s.CurrentRunP99` if non-nil else `0`.
3. **Estimated (cold start)** — otherwise: start from `cfg.PopulationDefaultMaxHR`, `Source = "population_default"`. If `s.CurrentRunP99 != nil && *s.CurrentRunP99 > PopulationDefaultMaxHR`: `MaxHRBpm = *s.CurrentRunP99`, `Source = "current_run"`. `Confidence = ConfidenceEstimated`.

Then clamp `MaxHRBpm` to `[cfg.MinReferenceBpm, cfg.MaxReferenceBpm]` (clamp the final value regardless of branch). Confidence and Source are not changed by clamping.

- [ ] **Step 1: Write failing tests** — `TestEstimateReference` table cases:
  - cold start, no data (`Stats{}`) → default 190, `population_default`, `estimated`.
  - cold start, `CurrentRunP99=200` > default → 200, `current_run`, `estimated`.
  - cold start, `CurrentRunP99=150` < default → 190, `population_default`, `estimated`.
  - calibrating, `HistoryRunCount=2, RecentHRSamplesP99=180, CurrentRunP99=195` → 195, `p99_recent_runs`, `calibrating`.
  - calibrated, `HistoryRunCount=5, RecentHRSamplesP99=191` → 191, `p99_recent_runs`, `calibrated`.
  - clamp ceiling: cold start `CurrentRunP99=250` → clamped to `MaxReferenceBpm` (230).
  - clamp floor: calibrated `RecentHRSamplesP99=80` → clamped to `MinReferenceBpm` (100).
  - spike rejection sanity: a p99 fed value of 191 (not 220) when the recent set contained a lone 220 — verify the reference is the p99, not the max (this is exercised by feeding `RecentHRSamplesP99` = P99 of a sample set with an injected 220).

- [ ] **Step 2: Run tests, verify fail** — `go test ./internal/hrzones/ -run TestEstimateReference` → FAIL.

- [ ] **Step 3: Implement** `EstimateReference` + a small `clamp(v, lo, hi int) int` helper.

- [ ] **Step 4: Run tests, verify pass** — `go test ./internal/hrzones/` → PASS.

- [ ] **Step 5: Commit** — `git commit -am "feat(hrzones): cold-start to calibrated reference estimation ladder"`

## Task 3: hrzones doc.go

**Files:**
- Create: `internal/hrzones/doc.go`

- [ ] **Step 1: Write** a `// Package hrzones ...` doc comment in `doc.go` (no code, just `package hrzones` + the doc block) covering: the percent-of-max zone model and why five open-ended-at-both-ends zones; the estimation ladder (estimated → calibrating → calibrated) and the p99/population-default rationale; the confidence states and how `Calibrating` drives the widget; and the documented **growth path** — persisting a per-run robust HR statistic (the run's p99) to a nullable `activities` column at ingest so `RecentHRStats` becomes a cheap column aggregate, an optimization that touches only the repository because of the engine boundary. No emoji, `why` not `what`.

- [ ] **Step 2: Verify it builds & vets** — `go vet ./internal/hrzones/` → clean; `go build ./...` → clean.

- [ ] **Step 3: Commit** — `git add internal/hrzones/doc.go && git commit -m "docs(hrzones): package doc for zone model, ladder and growth path"`

## Task 4: `[hr_zones]` config section

**Files:**
- Modify: `config.toml`
- Modify: `internal/config/config.go`
- Modify: `internal/config/config_test.go`

Add to `config.toml` (root):
```toml
[hr_zones]
population_default_max_hr = 190
calibrated_run_threshold  = 5
recency_window_days       = 90
min_reference_bpm         = 100
max_reference_bpm         = 230
zone_upper_bounds         = [0.60, 0.70, 0.80, 0.90]
zone_names                = ["Recovery", "Aerobic", "Tempo", "Threshold", "VO2max"]
```

In `internal/config/config.go`:
- Add an exported `HRZonesConfig` struct mirroring the toml (fields: `PopulationDefaultMaxHR, CalibratedRunThreshold, RecencyWindowDays, MinReferenceBpm, MaxReferenceBpm int`; `ZoneUpperBounds []float64`; `ZoneNames []string`).
- Add `HRZones HRZonesConfig` to `Config`.
- Add a `HRZones struct { ... \`toml:"..."\` }` block to `fileConfig` with `toml:"hr_zones"` and matching field tags (`population_default_max_hr`, etc.; `zone_upper_bounds`, `zone_names`).
- Map it in `Load`'s `cfg := Config{...}` literal: `HRZones: HRZonesConfig{ PopulationDefaultMaxHR: fc.HRZones.PopulationDefaultMaxHR, ... }`.
- These are non-secret literals (ints/floats/strings), so **no** `interpolate`/`applyEnvOverrides`/`applyDefaults` entries are needed (follow how the numeric `vectormemory` knobs are handled — they aren't interpolated). Do not add env overrides.

- [ ] **Step 1: Write a failing test** in `config_test.go` — load the embedded default (mirror the existing test that calls `Load(...)` with a minimal valid toml, or extend the existing default-load test) and assert `cfg.HRZones.PopulationDefaultMaxHR == 190`, `cfg.HRZones.CalibratedRunThreshold == 5`, `len(cfg.HRZones.ZoneUpperBounds) == 4`, `cfg.HRZones.ZoneNames[4] == "VO2max"`. (Inspect the existing test helpers first; reuse the same toml-building/`Load` pattern they use.)

- [ ] **Step 2: Run, verify fail** — `go test ./internal/config/` → FAIL.

- [ ] **Step 3: Implement** the struct, fileConfig block, mapping, and config.toml section.

- [ ] **Step 4: Run, verify pass** — `go test ./internal/config/` → PASS.

- [ ] **Step 5: Commit** — `git add config.toml internal/config && git commit -m "feat(config): add [hr_zones] engine tunables"`

## Task 5: `RecentHRStats` repository method

**Files:**
- Modify: `internal/activity/repository.go`
- Modify: `internal/activity/sqlite_repository.go`
- Modify: `internal/activity/sqlite_repository_test.go`

Add to the `Repository` interface and implement on `*SQLiteRepository`:
```go
RecentHRStats(ctx context.Context, userID string, window time.Duration, excludeActivityID string) (hrzones.Stats, error)
```
(`internal/activity` may import `internal/hrzones` — the dependency direction is activity→hrzones, never the reverse.)

Behavior — over the user's **non-deleted running** activities whose `start_time >= now-window` and `id != excludeActivityID`:
- `HistoryRunCount` = number of those activities that have **at least one HR-bearing trackpoint**.
- `RecentHRSamplesP99` = `hrzones.P99` over **all** HR samples across those activities' trackpoints (nil if none).
- Leave `CurrentRunP99` nil — the handler fills it from the viewed run.

Implementation approach (one query is cleanest):
```sql
SELECT t.heart_rate_bpm
FROM activity_trackpoints t
JOIN activities a ON a.id = t.activity_id
WHERE a.user_id = ? AND a.activity_type = ? AND a.deleted_at IS NULL
  AND a.start_time >= ? AND a.id != ? AND t.heart_rate_bpm IS NOT NULL
```
Collect samples into `[]int` for the p99. For `HistoryRunCount`, run a second small query (or `SELECT COUNT(DISTINCT t.activity_id)` with the same WHERE) so the count reflects only HR-bearing runs. Use `r.now()` for "now" (the repo already holds a `now func() time.Time`). Pass `ActivityRunning` and the computed `since := r.now().Add(-window)`.

- [ ] **Step 1: Write failing tests** in `sqlite_repository_test.go` — `TestRecentHRStats`:
  - Seed (via `repo.Create`) three running activities for `u1`: A (in window, HR-bearing trackpoints), B (in window, HR-bearing), C (outside the window — `start_time` older than `window`); plus one deleted in-window running activity D with HR; plus the "current" activity X (in window, HR-bearing) which is passed as `excludeActivityID`. Optionally a non-running activity to confirm it's excluded.
  - Call `RecentHRStats(ctx, "u1", 90*24*time.Hour, X.ID)`.
  - Assert `HistoryRunCount == 2` (A and B; C out of window, D deleted, X excluded), `RecentHRSamplesP99 != nil` and equals `hrzones.P99` of A+B's HR samples, `CurrentRunP99 == nil`.
  - A second case with zero qualifying runs → `HistoryRunCount == 0`, `RecentHRSamplesP99 == nil`.

  Build trackpoints with explicit `HeartRateBpm` via the existing `ptrInt` helper. Use `mustTime` for start times; for the out-of-window run set a start_time well before `r.now()-window` (note the repo's `now` is `time.Now` in tests unless overridden — set C's start_time to e.g. `time.Now().Add(-200*24*time.Hour)`).

- [ ] **Step 2: Run, verify fail** — `go test ./internal/activity/ -run TestRecentHRStats` → FAIL.

- [ ] **Step 3: Implement** the interface method + SQLite impl.

- [ ] **Step 4: Run, verify pass** — `go test ./internal/activity/` → PASS (whole package, to catch the interface addition breaking any mock — if a test mock implements `Repository`, add the method to it).

- [ ] **Step 5: Commit** — `git commit -am "feat(activity): RecentHRStats for HR-zone reference estimation"`

## Task 6: attach `heart_rate_zones` to the activity response

**Files:**
- Modify: `internal/activity/handler.go`
- Modify: `internal/server/server.go`
- Modify: `internal/activity/handler_test.go`

**Handler wiring (mirror the existing `SetPublisher`/`SetPlanMatcher` optional-setter pattern):**
- Add field `hrEngine *hrzones.Engine` to `Handler` and method `func (h *Handler) SetHRZonesEngine(e *hrzones.Engine, window time.Duration) { h.hrEngine = e; h.hrWindow = window }` (store the recency window too; derive it from config days). Add `hrWindow time.Duration` field.
- Guard: if `h.hrEngine == nil`, skip the block entirely (keeps existing handler tests that don't set it working, and keeps the field optional).

**DTO:**
```go
type heartRateZoneDTO struct {
    Zone        int     `json:"zone"`
    Name        string  `json:"name"`
    LowerPct    float64 `json:"lower_pct"`
    UpperPct    float64 `json:"upper_pct"`
    MinBpm      int     `json:"min_bpm"`
    MaxBpm      int     `json:"max_bpm"`
    TimeSeconds int     `json:"time_seconds"`
    TimePct     float64 `json:"time_pct"`
}
type heartRateZonesDTO struct {
    Model              string             `json:"model"`
    MaxHRReferenceBpm  int                `json:"max_hr_reference_bpm"`
    ReferenceSource    string             `json:"reference_source"`
    ReferenceConfidence string            `json:"reference_confidence"`
    Calibrating        bool               `json:"calibrating"`
    TotalHRSeconds     int                `json:"total_hr_seconds"`
    Zones              []heartRateZoneDTO `json:"zones"`
}
```
Add `HeartRateZones *heartRateZonesDTO \`json:"heart_rate_zones,omitempty"\`` to `activityDTO`.

**Assembly:** `toActivityDTO` currently has no DB access and no `ctx`/handler receiver. Keep `toActivityDTO` pure and instead assemble the zones in the `get` handler (it has `ctx`, `userID`, the loaded `*Activity` with trackpoints, and `h`). After building the base DTO, when `h.hrEngine != nil && a.ActivityType == ActivityRunning`:
1. Build `[]hrzones.Trackpoint` from `a.Trackpoints` (`ElapsedSeconds`, `HeartRateBpm`).
2. `stats, err := h.repo.RecentHRStats(ctx, userID, h.hrWindow, a.ID)` — on error, `ServerError`. Set `stats.CurrentRunP99 = hrzones.P99(currentRunHRSamples)` where `currentRunHRSamples` is the non-nil `HeartRateBpm` values of this run.
3. `ref := h.hrEngine.EstimateReference(stats)`; `res, ok := h.hrEngine.Compute(ref, tps)`.
4. If `ok`, map `res` → `heartRateZonesDTO` and set `dto.HeartRateZones`. If `!ok`, leave it nil (omitempty omits it).

Refactor: change `get` to build the DTO via `toActivityDTO(*a, true)` then attach the zones, and write the response. Do **not** thread the engine through `toActivityDTO`'s signature used by other callers (List etc.) — only the single `get` path gets zones. (Confirm `toActivityDTO` is also used by import/list responses and leave those unchanged.)

**Server wiring** (`internal/server/server.go`, near the existing `activityHandler := activity.NewHandler(activityRepo)` / `SetPublisher` block):
```go
hrEngine := hrzones.New(hrzones.Config{
    PopulationDefaultMaxHR: cfg.HRZones.PopulationDefaultMaxHR,
    CalibratedRunThreshold: cfg.HRZones.CalibratedRunThreshold,
    RecencyWindowDays:      cfg.HRZones.RecencyWindowDays,
    MinReferenceBpm:        cfg.HRZones.MinReferenceBpm,
    MaxReferenceBpm:        cfg.HRZones.MaxReferenceBpm,
    ZoneUpperBounds:        cfg.HRZones.ZoneUpperBounds,
    ZoneNames:              cfg.HRZones.ZoneNames,
})
activityHandler.SetHRZonesEngine(hrEngine, time.Duration(cfg.HRZones.RecencyWindowDays)*24*time.Hour)
```
(Use the `cfg` variable already in scope in server setup; confirm its name.)

- [ ] **Step 1: Write failing tests** in `handler_test.go`:
  - Extend the import-happy-path fixture flow (or add `TestGetActivity_HeartRateZones`): import `typical_5k.tcx`, then `GET /activities/{id}` through the handler with an engine set via `SetHRZonesEngine` (construct `hrzones.New` with the default config inline in the test). Decode and assert: `heart_rate_zones` present, `model == "percent_max_hr"`, `len(zones) == 5`, `sum(time_pct)` ≈ 1.0, `reference_confidence` is one of the valid values (cold-start single run → `"estimated"`, `calibrating == false` only when calibrated — for a single imported run expect `"estimated"` and `calibrating == true`).
  - `TestGetActivity_NoHR_OmitsBlock`: import/get a fixture with no HR trackpoints (create a minimal no-HR activity via the repo directly, or a no-HR TCX fixture if one exists; otherwise insert via `repo.Create` with HR-less trackpoints and call the `get` handler) → assert `heart_rate_zones` key absent from the JSON.
  - The handler test helper `newTestHandler` must call `SetHRZonesEngine` (add a variant or set it in the test) so the path is exercised; existing tests that use a handler without the engine should still pass (engine nil → no block, no panic).

- [ ] **Step 2: Run, verify fail** — `go test ./internal/activity/ -run HeartRateZones` → FAIL.

- [ ] **Step 3: Implement** the DTO, handler wiring, and server wiring.

- [ ] **Step 4: Run, verify pass** — `go test ./...` → PASS (full module; the server package must still compile and run).

- [ ] **Step 5: Commit** — `git commit -am "feat(activity): add heart_rate_zones block to activity response"`

## Task 7: web — API types

**Files:**
- Modify: `lib/api.ts`

Add near `RunningSession`:
```ts
export type HeartRateZone = {
  zone: number;
  name: string;
  lower_pct: number;
  upper_pct: number;
  min_bpm: number;
  max_bpm: number;
  time_seconds: number;
  time_pct: number;
};
export type HeartRateZones = {
  model: string;
  max_hr_reference_bpm: number;
  reference_source: string;
  reference_confidence: "estimated" | "calibrating" | "calibrated";
  calibrating: boolean;
  total_hr_seconds: number;
  zones: HeartRateZone[];
};
```
Add to `RunningSession`: `heart_rate_zones?: HeartRateZones;` (optional — absent when the run has no HR).

- [ ] **Step 1: Edit** the types. No behavior, so no new test; verify with `npm run typecheck`.
- [ ] **Step 2: Run** `npm run typecheck` → clean.
- [ ] **Step 3: Commit** — `git commit -am "feat(running): heart_rate_zones API types"`

## Task 8: web — zone tokens + format helper

**Files:**
- Modify: `app/globals.css`
- Modify: `lib/format.ts` (+ `lib/format.test.ts` if it exists)

**Tokens** — add a periwinkle single-hue ramp (light→saturated Z1→Z5, anchored on the `--accent` periwinkle `#9aa6d6`), placed alongside the existing `--discipline-*` tokens. Match how neighboring tokens are declared (CSS custom properties under `:root`/`@theme`; if the file declares `--color-*` Tailwind aliases for other ramps like `--discipline-lift-1..4`, add parallel `--color-zone-1..5` aliases):
```css
--zone-1: #444a66;
--zone-2: #5a6188;
--zone-3: #717aa8;
--zone-4: #8893c5;
--zone-5: #9aa6d6; /* = --accent */
```
(Inspect the existing `--discipline-lift-1..4` block and mirror its exact declaration style — `:root` var + any `@theme`/`--color-*` alias — so the new tokens are usable the same way.)

**Format** — add to `lib/format.ts` if not present:
```ts
/** Format a 0..1 fraction as a whole-number percent, em-dash for non-finite. */
export function formatPercent(fraction: number): string {
  if (!Number.isFinite(fraction)) return "—";
  return `${Math.round(fraction * 100)}%`;
}
```

- [ ] **Step 1: Write/extend test** — if `lib/format.test.ts` exists, add `formatPercent(0.076) === "8%"`, `formatPercent(0.5) === "50%"`, `formatPercent(NaN) === "—"`. Run → fail.
- [ ] **Step 2: Implement** the helper and tokens.
- [ ] **Step 3: Run** `npm run test -- format` (or the file) → pass; `npm run lint` clean.
- [ ] **Step 4: Commit** — `git commit -am "feat(running): zone color ramp tokens and formatPercent"`

## Task 9: web — HeartRateZones widget

**Files:**
- Create: `app/(app)/running/[id]/_components/HeartRateZones.tsx`
- Create: `app/(app)/running/[id]/_components/HeartRateZones.test.tsx`

A presentational, prop-driven component matching the `PaceStrip.tsx` idiom (card wrapper `rounded-[var(--radius-card)] border border-[var(--border)] bg-[var(--surface)] px-4 py-3`, faint uppercase header, tokens only — no charting dep, no hard-coded hex).

```tsx
import type { HeartRateZones as HRZones } from "@/lib/api";
import { formatDuration } from "@/lib/format";

const ZONE_VARS = ["--zone-1", "--zone-2", "--zone-3", "--zone-4", "--zone-5"];

export function HeartRateZones({ zones }: { zones: HRZones | null | undefined }) {
  if (!zones || zones.zones.length === 0) return null;
  const calibrating = zones.reference_confidence !== "calibrated";
  // ... stacked bar (flex row of segments, width: `${time_pct * 100}%`,
  //     background: var(--zone-N) by zone.zone) + legend rows
  //     (swatch, name, `${min_bpm}–${max_bpm} bpm`, formatDuration(time_seconds),
  //      `${Math.round(time_pct*100)}%`) + a calibrating banner when calibrating.
}
```

Requirements:
- **Stacked bar:** one segment per zone, `width: time_pct*100%`, fill `var(--zone-{zone.zone})`. Give a zone with `time_pct === 0` no segment (or width 0). Round the bar corners; full-width.
- **Legend:** per zone — a color swatch (same `--zone-N`), the `name`, the bpm range `min_bpm–max_bpm`, the time via `formatDuration(time_seconds)`, and the percent (`Math.round(time_pct*100)%`).
- **Calibrating banner:** when `reference_confidence` is `"estimated"` or `"calibrating"`, render a subtle banner with copy exactly: *"Calibrating — zones will sharpen as Prog Strength learns your heart rate."* Use `--muted`/`--accent-soft` tokens (a quiet info treatment), not an alarming color. When `"calibrated"`, no banner.
- **Absent data:** when `zones` is null/undefined, render nothing (`return null`) — the page passes `session.heart_rate_zones`.
- Header label e.g. `Heart rate zones` in the `text-[10px] uppercase tracking-wider text-[var(--faint)]` style.

- [ ] **Step 1: Write failing tests** `HeartRateZones.test.tsx` (Vitest + Testing Library, jsdom): 
  - renders 5 legend rows with names and bpm ranges given a calibrated fixture; no calibrating banner.
  - renders the calibrating banner (assert the copy) when `reference_confidence: "calibrating"`.
  - renders nothing (`container.firstChild` null) when `zones={null}`.
  Build a fixture object matching `HeartRateZones` type.
- [ ] **Step 2: Run, verify fail** — `npm run test -- HeartRateZones` → FAIL (module not found).
- [ ] **Step 3: Implement** the component.
- [ ] **Step 4: Run** `npm run test -- HeartRateZones` → PASS; `npm run lint && npm run typecheck` clean.
- [ ] **Step 5: Commit** — `git commit -am "feat(running): heart-rate zones widget"`

## Task 10: web — render the widget on the detail page

**Files:**
- Modify: `app/(app)/running/[id]/page.tsx`

- [ ] **Step 1: Edit** — import `HeartRateZones` and render it under the existing stats band. Place it after `<PaceStrip ... />` (or directly under `RunHeaderBand`, within the same `max-w-3xl` column), passing `zones={session?.heart_rate_zones}`. Since it returns null when absent, no extra guard is needed (but ensure `session` is non-null at that point — it's already guarded by the page's loading state; pass `session.heart_rate_zones` where `session` is known non-null, else `session?.heart_rate_zones`).
- [ ] **Step 2: Extend the page test** `page.test.tsx` — add `heart_rate_zones` to the `runningSession` fixture and assert the widget header/legend renders; for a fixture without `heart_rate_zones`, assert the widget is absent. Run `npm run test -- "running/\[id\]/page"` → pass (adjust the existing fixture so other assertions still hold).
- [ ] **Step 3: Full gate** — `npm run typecheck && npm run lint && npm run test && npm run build` → all clean.
- [ ] **Step 4: Commit** — `git commit -am "feat(running): show heart-rate zones on the detail page"`

## Task 11: docs — engine companion note

**Files:**
- Create: `notes/hrzones-engine.md` (in prog-strength-docs; if a different conventional dir exists, e.g. `notes/` vs `architecture/`, inspect the repo and place it where similar notes live — otherwise `notes/`).

- [ ] **Step 1: Write** a short note describing `internal/hrzones` as the single home for heart-rate-zone estimation logic: the percent-of-max model, the estimation ladder and confidence states, and the intended evolution — additional models (`Karvonen`/`LTHR`) via the `model` field, a future user-supplied max-HR override slotting into `EstimateReference`, and ingest-time caching of a per-run p99. Link back to `sows/running-heart-rate-zones.md`. (This commit lands on the `feat/running-heart-rate-zones` branch in prog-strength-docs, together with the SOW status flip handled by the workflow.)
- [ ] **Step 2: Commit** — `git add notes/hrzones-engine.md && git commit -m "docs: note the hrzones engine as home for HR-zone logic"`

---

## Self-Review Notes (coverage check)

- Goals → Tasks: `heart_rate_zones` block (T6), five-zone time-in-zone (T1), reference estimation w/ confidence (T2), single engine package (T1–T3), web widget + calibrating state (T9–T10), config tunables (T4). Non-goals respected: no user max-HR setting/migration, percent-of-max only (`Model` field present for future), running-only wiring, read-time compute (growth path documented in T3), config boundaries (no per-user editing).
- Type consistency: engine names (`EstimateReference`, `Compute`, `P99`, `Stats`, `Reference`, `Result`, `Zone`, `Confidence`) are used identically in T1–T6; DTO JSON keys match the SOW response shape; web types mirror the JSON.
- Edge cases: no-HR run → `Compute` ok=false → block omitted (T6); cold start single run → `estimated`/`calibrating:true` (T2/T6); spike rejection via p99 (T1/T2); clamp floor/ceiling (T2).
