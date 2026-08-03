# Running Tile Family Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship the four running tiles chosen from `dx/running-tile` — the `running` tile rewritten as `load-ramp` ("Training Load"), plus three new catalog ids (`running_log`, `running_effort`, `running_vertical`) — **and** the `RunningSection` payload extension they read, with the API's running section re-gated on any family tile.

**Architecture:** `prog-strength-api` extends `RunningSection` (baseline, week runs, weekly load, eight new current-week fields) as pure functions over the already-fetched 53-week run slice — no new query, no new repository method, no new table, except one gated `RecentHRStats` read for the Run Effort tile's max-HR reference. It adds three `TileID`s to `Catalog` immediately after `TileRunning` and re-gates `buildRunningSection` on any running-family tile, still emitting exactly one `running` section key. **`WeeklyDistanceSpark` stays in this PR** (expand step of expand/contract — the deployed web build maps it with no null guard). `prog-strength-web` mirrors the payload and catalog, builds four production tile components under `app/(app)/dashboard/_components/running/` sharing one tested formatting/status module, and rewires `TileCard` with four cases (deleting `RunningCard`). `prog-strength-docs` flips the DX to `selected` and the SOW to `shipped`.

**Tech Stack:** Go 1.25 / chi / SQLite (API); Next.js 16 App Router, React 19, TypeScript, Tailwind v4, Vitest + Testing Library (web).

**Source spec:** `/workspace/prog-strength-docs/sows/running-tile-family.md` — read it before starting; its payload contract, per-tile color table, and state matrix are binding. The DX (`/workspace/prog-strength-docs/dx/running-tile.md`) is background; the SOW supersedes it where they differ. The DX mockup code is on a never-merged branch and is **not** available or needed — this plan is the spec.

**Binding constraints (from the SOW; apply everywhere):**

1. **All four tiles read the ONE shared `running` section.** No per-tile payload, no `running_log`/`running_effort`/`running_vertical` response keys, ever.
2. **Never recompute a server figure.** The only client arithmetic permitted is a signed delta of two server figures (`load-ramp`'s duration ramp, `effort-heart`'s `+3 vs 4-wk`), never a re-averaged series.
3. **Nil is not zero.** `ElevationGainMeters` nil ≠ 0 (treadmill ≠ pancake-flat); HR nil runs are stated via `HeartRateRuns`/`ElevationRuns` coverage counts, never silently averaged over.
4. **The two pace figures are different figures.** `recent_avg_pace_sec_per_km` is a 30-day aggregate; `current_week.avg_pace_sec_per_km` is this week's. Label them distinctly wherever both appear.
5. **Color contract (web):** running's hue is the sage triplet (`--discipline-run-bg/fg/dot`). Periwinkle `--accent` never appears in the family. Hike clay is forbidden on Vertical Gain. There is no run tonal ramp — gradation is alpha on `--discipline-run-dot` only. `--zone-1..5` appears on Run Effort and nowhere else. Nothing in the family is ever `--danger`. CSS vars by name, never raw hex.
6. **Unit conversion happens once, in the adapter,** through `formatDistanceValue` / `formatPaceValue` / `formatElevationValue` / `distanceToDisplay`. No tile does raw metre→unit conversion.
7. **`WeeklyDistanceSpark` survives the API PR unchanged** (`EnduranceSection`'s copy is untouched forever). The web adapter stops reading it. The API delete is a separate future contract PR — NOT part of this plan.
8. **`defaultLayout` unchanged** — a user still gets exactly one running tile by default.
9. Never bypass hooks (`--no-verify`), never add `//nolint`, never silence gosec, never skip/weaken a test to get green.

**Branches:** create `feat/running-tile-family` from `main` in each of the three repos. Conventional-commit messages, lowercase subjects (both repos enforce this; API PR titles too).

---

## Repo 1: prog-strength-api (`/workspace/prog-strength-api`)

Run all commands from `/workspace/prog-strength-api`. Test with `go test ./internal/dashboard/`.

### Task 1: Catalog — three new TileIDs

**Files:**
- Modify: `internal/dashboard/tiles.go`
- Test: `internal/dashboard/tiles_test.go`

- [ ] **Step 1: Create the branch**

```bash
cd /workspace/prog-strength-api && git checkout -b feat/running-tile-family
```

- [ ] **Step 2: Update the catalog tests to expect the three new ids (failing first)**

In `internal/dashboard/tiles_test.go`, replace both the `all` list (in `TestCatalog_EveryConstantAppearsExactlyOnce`) and the `want` list (in `TestCatalog_Order`) with this 18-id slice — the family contiguous immediately after `TileRunning`, in this exact order:

```go
	all := []TileID{
		TileRunning, TileRunningLog, TileRunningEffort, TileRunningVertical,
		TileWalking, TileCycling, TileHiking, TileLifting,
		TileSteps, TileNutrition, TileBodyweight, TileBloodPressure,
		TileRecovery, TileHRVBalance, TileMorningVitals, TileRecoveryTrend, TileRecoveryLog,
		TileStreak,
	}
```

(same slice named `want` in `TestCatalog_Order`). Also extend `TestValidTileID`:

```go
	if !ValidTileID("running_log") {
		t.Error("running_log should be valid")
	}
	if !ValidTileID("running_effort") {
		t.Error("running_effort should be valid")
	}
	if !ValidTileID("running_vertical") {
		t.Error("running_vertical should be valid")
	}
```

- [ ] **Step 3: Run to verify it fails**

Run: `go test ./internal/dashboard/ -run TestCatalog -v`
Expected: compile error — `undefined: TileRunningLog` etc.

- [ ] **Step 4: Add the constants and Catalog entries**

In `internal/dashboard/tiles.go`, the const block gains three ids directly after `TileRunning`:

```go
const (
	TileRunning         TileID = "running"
	TileRunningLog      TileID = "running_log"
	TileRunningEffort   TileID = "running_effort"
	TileRunningVertical TileID = "running_vertical"
	TileWalking         TileID = "walking"
	TileCycling         TileID = "cycling"
	TileHiking          TileID = "hiking"
	TileLifting         TileID = "lifting"
	TileSteps           TileID = "steps"
	TileNutrition       TileID = "nutrition"
	TileBodyweight      TileID = "bodyweight"
	TileBloodPressure   TileID = "blood_pressure"
	TileRecovery        TileID = "recovery"
	TileHRVBalance      TileID = "hrv_balance"
	TileMorningVitals   TileID = "morning_vitals"
	TileRecoveryTrend   TileID = "recovery_trend"
	TileRecoveryLog     TileID = "recovery_log"
	TileStreak          TileID = "streak"
)
```

and `Catalog` becomes (order fixes the web tray; `ValidTileID`/`catalogSet` derive from it and need no edit):

```go
var Catalog = []TileID{
	TileRunning, TileRunningLog, TileRunningEffort, TileRunningVertical,
	TileWalking, TileCycling, TileHiking, TileLifting,
	TileSteps, TileNutrition, TileBodyweight, TileBloodPressure,
	TileRecovery, TileHRVBalance, TileMorningVitals, TileRecoveryTrend, TileRecoveryLog,
	TileStreak,
}
```

- [ ] **Step 5: Run to verify it passes**

Run: `go test ./internal/dashboard/ -run 'TestCatalog|TestValidTileID' -v`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add internal/dashboard/tiles.go internal/dashboard/tiles_test.go
git commit -m "feat(dashboard): add running_log, running_effort, running_vertical tile ids"
```

### Task 2: DTO extension + builder helpers (`dto.go`, `running.go`)

**Files:**
- Modify: `internal/dashboard/dto.go` (extend `RunningSection`/`RunningCurrentWeek`, add three types)
- Modify: `internal/dashboard/running.go` (pure helpers over the run slice)
- Test: `internal/dashboard/running_test.go`

**Context:** `buildRunning(metrics, runs, now, loc)` keeps its exact signature and its nil-on-no-running-data behaviour. All new computation is pure functions over the already-fetched slice, bucketed with the existing `localWeekStart` / `weeklyBucketStarts` / `weekIndex` helpers (`buckets.go`, `running.go`) so DST correctness comes for free. `CurrentWeek.DistanceMeters` and `RunCount` **continue to come from `metrics` unchanged** (SOW Open Question 1 — moving them risks silently changing an on-screen number). `HeartRateZone` on week runs is always nil from the builder; the handler fills it (Task 3). `WeeklyDistanceSpark` and `weeklyDistanceSpark()` stay untouched.

- [ ] **Step 1: Extend the DTO types**

In `internal/dashboard/dto.go`, replace the existing `RunningSection` and `RunningCurrentWeek` and add the three new types (keep `LatestRun` as is):

```go
// RunningSection is the ONE shared payload every running-family tile reads
// (running / running_log / running_effort / running_vertical). nil at the
// Summary level when the user has no running activity at all.
type RunningSection struct {
	CurrentWeek RunningCurrentWeek `json:"current_week"`
	// Baseline is the trailing 4-week average EXCLUDING the current week —
	// what "normal" means for this athlete. nil until at least one prior
	// week holds a run.
	Baseline *RunningBaseline `json:"baseline"`
	// RecentAvgPaceSecPerKm is a 30-DAY aggregate — a different figure from
	// CurrentWeek.AvgPaceSecPerKm and labelled differently by every tile.
	RecentAvgPaceSecPerKm *float64 `json:"recent_avg_pace_sec_per_km"`
	LatestRun             *LatestRun `json:"latest_run"`
	// WeekRuns is this local week's runs, oldest→newest.
	WeekRuns []RunningWeekRun `json:"week_runs"`
	// WeeklyLoad is 8 week-anchored buckets, oldest→newest. A bucket with no
	// runs is a real zero — the distinction the bare spark could not make.
	WeeklyLoad []RunningWeekPoint `json:"weekly_load"`
	// WeeklyDistanceSpark is the legacy series the retired card read. It
	// survives this (expand) step because the deployed web build maps it
	// with no null guard; a follow-up contract PR deletes it once web has
	// stopped reading it.
	WeeklyDistanceSpark []float64 `json:"weekly_distance_spark"`
}

type RunningCurrentWeek struct {
	DistanceMeters      float64  `json:"distance_meters"`
	RunCount            int      `json:"run_count"`
	DeltaPctVsPriorWeek *float64 `json:"delta_pct_vs_prior_week"`
	DurationSeconds     int      `json:"duration_seconds"`
	// AvgPaceSecPerKm is the week AGGREGATE (Σduration / Σkm), not a mean of
	// per-run paces — the long run is exactly what separates the two.
	AvgPaceSecPerKm *float64 `json:"avg_pace_sec_per_km"`
	// AvgHeartRateBpm is duration-weighted over HR-bearing runs; nil when
	// none carry HR. HeartRateRuns says how many contributed.
	AvgHeartRateBpm *int `json:"avg_heart_rate_bpm"`
	// ElevationGainMeters sums gain over gain-bearing runs and is nil —
	// never 0 — when none carry it: an indoor-only week must stay
	// distinguishable from a flat week.
	ElevationGainMeters *float64 `json:"elevation_gain_meters"`
	HeartRateRuns       int      `json:"heart_rate_runs"`
	ElevationRuns       int      `json:"elevation_runs"`
	LongestRunMeters    float64  `json:"longest_run_meters"`
	// DaysRun counts distinct local dates run, 0–7.
	DaysRun int `json:"days_run"`
}

// RunningWeekRun is one of this week's runs, projected for the tiles.
type RunningWeekRun struct {
	ActivityID      string    `json:"activity_id"`
	Name            *string   `json:"name"`
	StartTime       time.Time `json:"start_time"`
	LocalDate       string    `json:"local_date"` // YYYY-MM-DD in the user's tz
	DistanceMeters  float64   `json:"distance_meters"`
	DurationSeconds int       `json:"duration_seconds"`
	AvgPaceSecPerKm *float64  `json:"avg_pace_sec_per_km"`
	AvgHeartRateBpm *int      `json:"avg_heart_rate_bpm"`
	// HeartRateZone is 1..5, nil unless the Run Effort tile is enabled (the
	// handler classifies against the max-HR reference; zone thresholds stay
	// single-sourced in Go rather than mirrored into TypeScript).
	HeartRateZone       *int     `json:"heart_rate_zone"`
	ElevationGainMeters *float64 `json:"elevation_gain_meters"`
	Environment         string   `json:"environment"` // outdoor | indoor
}

// RunningWeekPoint is one weekly bucket of the load rail.
type RunningWeekPoint struct {
	WeekStart           string   `json:"week_start"` // YYYY-MM-DD, local Monday
	DistanceMeters      float64  `json:"distance_meters"`
	DurationSeconds     int      `json:"duration_seconds"`
	RunCount            int      `json:"run_count"`
	ElevationGainMeters *float64 `json:"elevation_gain_meters"`
}

// RunningBaseline is the trailing 4-week average EXCLUDING the current week.
// The denominator is Weeks — weeks that actually held a run — not a flat 4,
// so a runner three weeks into the product isn't diluted by pre-signup zeros
// (SOW Open Question 2). Pace and HR are aggregates over the window's runs,
// the same method RecentAvgPaceSecPerKm uses, so the figures are comparable.
type RunningBaseline struct {
	WindowWeeks         int      `json:"window_weeks"` // 4
	Weeks               int      `json:"weeks"`        // weeks with >=1 run behind it
	DistanceMeters      *float64 `json:"distance_meters"`
	DurationSeconds     *int     `json:"duration_seconds"`
	AvgPaceSecPerKm     *float64 `json:"avg_pace_sec_per_km"`
	AvgHeartRateBpm     *int     `json:"avg_heart_rate_bpm"`
	ElevationGainMeters *float64 `json:"elevation_gain_meters"`
	RunsPerWeek         *float64 `json:"runs_per_week"`
}
```

- [ ] **Step 2: Write the failing builder tests**

Append to `internal/dashboard/running_test.go`. First extend the `run` fixture helper family with a detailed variant (keep the existing `run` helper untouched — existing tests use it):

```go
func ptrI(i int) *int { return &i }

// detailedRun builds a running activity with the optional columns the new
// builder helpers read. hr/gain nil-able via pointers; env "" defaults outdoor.
func detailedRun(name string, dist float64, dur int, start time.Time, pace *float64, hr *int, gain *float64, env string) activity.Activity {
	a := run(name, dist, dur, start)
	a.AvgPaceSecPerKm = pace
	a.AvgHeartRateBpm = hr
	a.ElevationGainMeters = gain
	if env == "" {
		env = "outdoor"
	}
	a.Environment = activity.Environment(env)
	return a
}
```

Then the tests. The headline fixture mirrors the DX: week of Mon 2026-07-27 in America/New_York, now = Sat 2026-08-01 18:00 local. It deliberately mixes an outdoor easy run, an indoor treadmill run with no elevation, a run with no heart rate, and a long run:

```go
// headlineWeek returns the DX's representative week: 4 runs, one indoor (no
// elevation), one with no HR, one 20.4km long run. now is Sat of that week.
func headlineWeek(t *testing.T) ([]activity.Activity, time.Time, *time.Location) {
	t.Helper()
	ny := mustLoad(t, "America/New_York")
	now := time.Date(2026, 8, 1, 18, 0, 0, 0, ny)
	runs := []activity.Activity{
		detailedRun("Easy shakeout", 5633, 2128, time.Date(2026, 7, 27, 7, 0, 0, 0, ny), ptrF(377.8), ptrI(148), ptrF(38), "outdoor"),
		detailedRun("", 4184, 1490, time.Date(2026, 7, 28, 6, 30, 0, 0, ny), ptrF(356.1), ptrI(152), nil, "indoor"),
		detailedRun("Lunch run", 4023, 1575, time.Date(2026, 7, 30, 12, 15, 0, 0, ny), ptrF(391.5), nil, ptrF(22), "outdoor"),
		detailedRun("Saturday long run", 20438, 7784, time.Date(2026, 8, 1, 7, 2, 0, 0, ny), ptrF(380.9), ptrI(156), ptrF(214), "outdoor"),
	}
	return runs, now, ny
}

func TestBuildRunning_CurrentWeekAggregates(t *testing.T) {
	runs, now, ny := headlineWeek(t)
	m := activity.Metrics{AllTime: activity.PeriodStat{RunCount: len(runs)}}

	got := buildRunning(m, runs, now, ny)
	if got == nil {
		t.Fatal("expected section")
	}
	cw := got.CurrentWeek

	if cw.DurationSeconds != 12977 {
		t.Errorf("duration = %d, want 12977", cw.DurationSeconds)
	}
	// Aggregate pace: 12977 / 34.278 km = 378.58 s/km. The naive mean of the
	// four per-run paces is 376.575 — the long run is what separates them.
	if cw.AvgPaceSecPerKm == nil || math.Abs(*cw.AvgPaceSecPerKm-378.58) > 0.05 {
		t.Errorf("avg pace = %v, want ~378.58 (aggregate, not mean of means)", cw.AvgPaceSecPerKm)
	}
	// Duration-weighted HR over the 3 HR-bearing runs:
	// (2128*148 + 1490*152 + 7784*156) / (2128+1490+7784) = 153.98 → 154.
	if cw.AvgHeartRateBpm == nil || *cw.AvgHeartRateBpm != 154 {
		t.Errorf("avg hr = %v, want 154 (duration-weighted)", cw.AvgHeartRateBpm)
	}
	if cw.HeartRateRuns != 3 {
		t.Errorf("hr runs = %d, want 3", cw.HeartRateRuns)
	}
	if cw.ElevationGainMeters == nil || *cw.ElevationGainMeters != 274 {
		t.Errorf("elevation = %v, want 274 (indoor run contributes nothing)", cw.ElevationGainMeters)
	}
	if cw.ElevationRuns != 3 {
		t.Errorf("elevation runs = %d, want 3", cw.ElevationRuns)
	}
	if cw.LongestRunMeters != 20438 {
		t.Errorf("longest = %v, want 20438", cw.LongestRunMeters)
	}
	if cw.DaysRun != 4 {
		t.Errorf("days run = %d, want 4", cw.DaysRun)
	}
}

func TestBuildRunning_WeekRuns_SortedRowsWithZoneNil(t *testing.T) {
	runs, now, ny := headlineWeek(t)
	// Shuffle input order to prove the builder sorts ascending by StartTime.
	shuffled := []activity.Activity{runs[3], runs[0], runs[2], runs[1]}
	m := activity.Metrics{AllTime: activity.PeriodStat{RunCount: 4}}

	got := buildRunning(m, shuffled, now, ny)
	if len(got.WeekRuns) != 4 {
		t.Fatalf("week runs = %d, want 4", len(got.WeekRuns))
	}
	wantDates := []string{"2026-07-27", "2026-07-28", "2026-07-30", "2026-08-01"}
	for i, wr := range got.WeekRuns {
		if wr.LocalDate != wantDates[i] {
			t.Errorf("week run %d local date = %q, want %q", i, wr.LocalDate, wantDates[i])
		}
		if wr.HeartRateZone != nil {
			t.Errorf("week run %d zone = %v, want nil from the builder (handler fills it)", i, wr.HeartRateZone)
		}
	}
	if got.WeekRuns[1].Environment != "indoor" {
		t.Errorf("run 1 environment = %q, want indoor", got.WeekRuns[1].Environment)
	}
	if got.WeekRuns[1].ElevationGainMeters != nil {
		t.Errorf("indoor run elevation = %v, want nil (not zero)", got.WeekRuns[1].ElevationGainMeters)
	}
	if got.WeekRuns[2].AvgHeartRateBpm != nil {
		t.Errorf("no-HR run bpm = %v, want nil", got.WeekRuns[2].AvgHeartRateBpm)
	}
}

func TestBuildRunning_IndoorOnlyWeek_ElevationNilNotZero(t *testing.T) {
	ny := mustLoad(t, "America/New_York")
	now := time.Date(2026, 8, 1, 18, 0, 0, 0, ny)
	runs := []activity.Activity{
		detailedRun("", 4000, 1400, time.Date(2026, 7, 28, 6, 0, 0, 0, ny), ptrF(350), ptrI(150), nil, "indoor"),
		detailedRun("", 5000, 1800, time.Date(2026, 7, 30, 6, 0, 0, 0, ny), ptrF(360), ptrI(148), nil, "indoor"),
	}
	m := activity.Metrics{AllTime: activity.PeriodStat{RunCount: 2}}

	got := buildRunning(m, runs, now, ny)
	if got.CurrentWeek.ElevationGainMeters != nil {
		t.Errorf("elevation = %v, want nil for an indoor-only week", got.CurrentWeek.ElevationGainMeters)
	}
	if got.CurrentWeek.ElevationRuns != 0 {
		t.Errorf("elevation runs = %d, want 0", got.CurrentWeek.ElevationRuns)
	}
}

func TestBuildRunning_TwoRunsOneDay_DaysRunCountsDistinctDates(t *testing.T) {
	ny := mustLoad(t, "America/New_York")
	now := time.Date(2026, 8, 1, 18, 0, 0, 0, ny)
	runs := []activity.Activity{
		detailedRun("am", 5000, 1800, time.Date(2026, 7, 28, 6, 0, 0, 0, ny), nil, nil, nil, ""),
		detailedRun("pm", 3000, 1100, time.Date(2026, 7, 28, 18, 0, 0, 0, ny), nil, nil, nil, ""),
		detailedRun("", 4000, 1500, time.Date(2026, 7, 30, 6, 0, 0, 0, ny), nil, nil, nil, ""),
	}
	m := activity.Metrics{AllTime: activity.PeriodStat{RunCount: 3}}

	got := buildRunning(m, runs, now, ny)
	if got.CurrentWeek.DaysRun != 2 {
		t.Errorf("days run = %d, want 2 (two runs share a local date)", got.CurrentWeek.DaysRun)
	}
}

func TestBuildRunning_Baseline_WeeksWithRunsDenominator(t *testing.T) {
	ny := mustLoad(t, "America/New_York")
	now := time.Date(2026, 8, 1, 18, 0, 0, 0, ny) // current week Mon 07-27
	runs := []activity.Activity{
		// Window week 07-20: two runs.
		detailedRun("", 5000, 1800, time.Date(2026, 7, 21, 7, 0, 0, 0, ny), ptrF(360), ptrI(150), ptrF(50), ""),
		detailedRun("", 10000, 3600, time.Date(2026, 7, 23, 7, 0, 0, 0, ny), ptrF(360), ptrI(160), nil, ""),
		// Window week 07-06: one run.
		detailedRun("", 6000, 2400, time.Date(2026, 7, 8, 7, 0, 0, 0, ny), ptrF(400), nil, ptrF(30), ""),
		// Weeks 06-29 and 07-13: no runs. Older than the window: excluded.
		detailedRun("", 9000, 3000, time.Date(2026, 6, 24, 7, 0, 0, 0, ny), nil, nil, nil, ""),
		// Current week: excluded from the baseline.
		detailedRun("", 8000, 2900, time.Date(2026, 7, 29, 7, 0, 0, 0, ny), nil, ptrI(170), ptrF(500), ""),
	}
	m := activity.Metrics{AllTime: activity.PeriodStat{RunCount: len(runs)}}

	b := buildRunning(m, runs, now, ny).Baseline
	if b == nil {
		t.Fatal("baseline nil, want present")
	}
	if b.WindowWeeks != 4 || b.Weeks != 2 {
		t.Fatalf("window/weeks = %d/%d, want 4/2 (only weeks holding a run count)", b.WindowWeeks, b.Weeks)
	}
	// 21000m over 2 run-holding weeks.
	if b.DistanceMeters == nil || *b.DistanceMeters != 10500 {
		t.Errorf("distance = %v, want 10500", b.DistanceMeters)
	}
	// 7800s over 2 weeks.
	if b.DurationSeconds == nil || *b.DurationSeconds != 3900 {
		t.Errorf("duration = %v, want 3900", b.DurationSeconds)
	}
	// 3 runs over 2 weeks.
	if b.RunsPerWeek == nil || *b.RunsPerWeek != 1.5 {
		t.Errorf("runs/week = %v, want 1.5", b.RunsPerWeek)
	}
	// Aggregate pace over the window's runs: 7800 / 21 km = 371.43.
	if b.AvgPaceSecPerKm == nil || math.Abs(*b.AvgPaceSecPerKm-371.43) > 0.05 {
		t.Errorf("pace = %v, want ~371.43 (aggregate, not a mean of weekly means)", b.AvgPaceSecPerKm)
	}
	// Duration-weighted HR over the window's HR-bearing runs:
	// (1800*150 + 3600*160) / 5400 = 156.67 → 157.
	if b.AvgHeartRateBpm == nil || *b.AvgHeartRateBpm != 157 {
		t.Errorf("hr = %v, want 157", b.AvgHeartRateBpm)
	}
	// (50 + 30) gain over 2 weeks.
	if b.ElevationGainMeters == nil || *b.ElevationGainMeters != 40 {
		t.Errorf("elevation = %v, want 40", b.ElevationGainMeters)
	}
}

func TestBuildRunning_Baseline_NilForFirstEverRunner(t *testing.T) {
	ny := mustLoad(t, "America/New_York")
	now := time.Date(2026, 8, 1, 18, 0, 0, 0, ny)
	runs := []activity.Activity{
		detailedRun("First run", 3000, 1200, time.Date(2026, 7, 30, 7, 0, 0, 0, ny), nil, nil, nil, ""),
	}
	m := activity.Metrics{AllTime: activity.PeriodStat{RunCount: 1}}

	got := buildRunning(m, runs, now, ny)
	if got.Baseline != nil {
		t.Errorf("baseline = %+v, want nil for a first-ever runner", got.Baseline)
	}
}

func TestBuildRunning_WeeklyLoad_ZeroWeekIsRealZero(t *testing.T) {
	runs, now, ny := headlineWeek(t)
	// One prior-week run so a populated bucket exists beside zero buckets.
	runs = append(runs, detailedRun("", 30900, 11760, time.Date(2026, 7, 22, 7, 0, 0, 0, ny), nil, nil, ptrF(231), ""))
	m := activity.Metrics{AllTime: activity.PeriodStat{RunCount: len(runs)}}

	load := buildRunning(m, runs, now, ny).WeeklyLoad
	if len(load) != sparkWeeks {
		t.Fatalf("weekly load = %d buckets, want %d", len(load), sparkWeeks)
	}
	// Oldest→newest, ending with the current week.
	if load[len(load)-1].WeekStart != "2026-07-27" {
		t.Errorf("last bucket = %q, want 2026-07-27", load[len(load)-1].WeekStart)
	}
	last := load[len(load)-1]
	if last.RunCount != 4 || last.DurationSeconds != 12977 || last.DistanceMeters != 34278 {
		t.Errorf("current bucket = %+v, want 4 runs / 12977s / 34278m", last)
	}
	prior := load[len(load)-2]
	if prior.WeekStart != "2026-07-20" || prior.RunCount != 1 || prior.ElevationGainMeters == nil || *prior.ElevationGainMeters != 231 {
		t.Errorf("prior bucket = %+v, want 1 run week 2026-07-20 with 231m gain", prior)
	}
	empty := load[len(load)-3]
	if empty.RunCount != 0 || empty.DistanceMeters != 0 || empty.DurationSeconds != 0 {
		t.Errorf("empty bucket = %+v, want a real zero", empty)
	}
	if empty.ElevationGainMeters != nil {
		t.Errorf("empty bucket elevation = %v, want nil", empty.ElevationGainMeters)
	}
}

func TestBuildRunning_DSTWeekBoundary(t *testing.T) {
	denver := mustLoad(t, "America/Denver")
	// The US spring-forward week: Sun 2026-03-08 02:00 MST → 03:00 MDT.
	// A run late Sunday local (23:00, after the shift) must land in the week
	// of Mon 2026-03-02, and the current week Mon 2026-03-09 must hold its
	// own run — the AddDate-based bucketing keeps this correct where raw
	// 168-hour arithmetic would drift an hour.
	now := time.Date(2026, 3, 11, 13, 0, 0, 0, denver)
	runs := []activity.Activity{
		detailedRun("dst sunday", 5000, 1800, time.Date(2026, 3, 8, 23, 0, 0, 0, denver), nil, nil, nil, ""),
		detailedRun("this week", 4000, 1500, time.Date(2026, 3, 10, 7, 0, 0, 0, denver), nil, nil, nil, ""),
	}
	m := activity.Metrics{AllTime: activity.PeriodStat{RunCount: 2}}

	got := buildRunning(m, runs, now, denver)
	if len(got.WeekRuns) != 1 || got.WeekRuns[0].LocalDate != "2026-03-10" {
		t.Fatalf("week runs = %+v, want only the 2026-03-10 run", got.WeekRuns)
	}
	load := got.WeeklyLoad
	prior := load[len(load)-2]
	if prior.WeekStart != "2026-03-02" || prior.RunCount != 1 {
		t.Errorf("prior bucket = %+v, want the DST-Sunday run in week 2026-03-02", prior)
	}
}
```

Add `"math"` to the test file's imports.

- [ ] **Step 3: Run to verify the new tests fail**

Run: `go test ./internal/dashboard/ -run TestBuildRunning -v`
Expected: compile errors (`unknown field Baseline`, `undefined: detailedRun`… the DTO half compiles after Step 1; the builder fields are zero so aggregate assertions fail).

- [ ] **Step 4: Implement the builder helpers**

In `internal/dashboard/running.go`, add `"math"` and `"sort"` to imports, and:

Replace `buildRunning` with:

```go
// baselineWindowWeeks is the trailing window behind RunningBaseline.
const baselineWindowWeeks = 4

// buildRunning assembles the running section from already-fetched metrics and
// the user's run activities. It is pure: now and loc are passed in (no
// time.Now, no DB) so the local-week bucketing is deterministic and testable
// across timezones and DST. Returns nil when there is no running activity at
// all. CurrentWeek's DistanceMeters/RunCount stay metrics-sourced (the two
// figures already on screen); every new field derives from the run slice.
// HeartRateZone is left nil here — the handler classifies when Run Effort is
// enabled.
func buildRunning(metrics activity.Metrics, runs []activity.Activity, now time.Time, loc *time.Location) *RunningSection {
	if len(runs) == 0 && metrics.AllTime.RunCount == 0 {
		return nil
	}
	if loc == nil {
		loc = time.UTC
	}

	week := currentWeekRuns(runs, now, loc)
	return &RunningSection{
		CurrentWeek:           currentWeekStats(metrics, week, loc),
		Baseline:              buildBaseline(runs, now, loc),
		RecentAvgPaceSecPerKm: metrics.RecentAvgPaceSecPerKm,
		LatestRun:             latestRun(runs),
		WeekRuns:              weekRunRows(week, loc),
		WeeklyLoad:            weeklyLoad(runs, now, loc),
		WeeklyDistanceSpark:   weeklyDistanceSpark(runs, now, loc),
	}
}
```

Then append the helpers (keep `latestRun`, `weeklyDistanceSpark`, `weekIndex` untouched):

```go
// currentWeekRuns filters to the running rows of the current local week,
// sorted ascending by start time (the order the week-log tile renders).
func currentWeekRuns(runs []activity.Activity, now time.Time, loc *time.Location) []activity.Activity {
	current := localWeekStart(now, loc)
	var week []activity.Activity
	for i := range runs {
		if runs[i].ActivityType != activity.ActivityRunning {
			continue
		}
		if localWeekStart(runs[i].StartTime, loc).Equal(current) {
			week = append(week, runs[i])
		}
	}
	sort.Slice(week, func(i, j int) bool { return week[i].StartTime.Before(week[j].StartTime) })
	return week
}

// currentWeekStats aggregates the current week. DistanceMeters and RunCount
// come from metrics, unchanged — they are the two figures already on screen
// and the ones the deep page prints; leaving their provenance alone means
// this section cannot silently move a number a user is looking at (the
// handler test pins that the two code paths agree on a shared fixture).
func currentWeekStats(metrics activity.Metrics, week []activity.Activity, loc *time.Location) RunningCurrentWeek {
	cw := RunningCurrentWeek{
		DistanceMeters:      metrics.CurrentWeek.DistanceMeters,
		RunCount:            metrics.CurrentWeek.RunCount,
		DeltaPctVsPriorWeek: metrics.DeltaPctVsPriorWeek,
	}
	var dist float64
	days := make(map[string]bool, len(week))
	for i := range week {
		r := &week[i]
		cw.DurationSeconds += r.DurationSeconds
		dist += r.DistanceMeters
		if r.DistanceMeters > cw.LongestRunMeters {
			cw.LongestRunMeters = r.DistanceMeters
		}
		days[r.StartTime.In(loc).Format("2006-01-02")] = true
	}
	cw.DaysRun = len(days)
	cw.AvgPaceSecPerKm = aggregatePace(cw.DurationSeconds, dist)
	cw.AvgHeartRateBpm = durationWeightedHR(week)
	cw.HeartRateRuns = countHeartRateRuns(week)
	cw.ElevationGainMeters, cw.ElevationRuns = sumElevation(week)
	return cw
}

// aggregatePace is Σduration / Σkm — the same method the 30-day figure uses,
// so week and baseline paces are comparable. nil when there is no distance.
func aggregatePace(durationSeconds int, distanceMeters float64) *float64 {
	if distanceMeters <= 0 {
		return nil
	}
	p := float64(durationSeconds) / (distanceMeters / 1000)
	return &p
}

// durationWeightedHR weights each HR-bearing run's average by its duration —
// a 65-minute long run and a 20-minute shakeout are not equal evidence about
// the week's effort. nil when no run carries both HR and a positive duration.
func durationWeightedHR(runs []activity.Activity) *int {
	var num, den float64
	for i := range runs {
		if runs[i].AvgHeartRateBpm == nil || runs[i].DurationSeconds <= 0 {
			continue
		}
		num += float64(runs[i].DurationSeconds) * float64(*runs[i].AvgHeartRateBpm)
		den += float64(runs[i].DurationSeconds)
	}
	if den == 0 {
		return nil
	}
	v := int(math.Round(num / den))
	return &v
}

func countHeartRateRuns(runs []activity.Activity) int {
	n := 0
	for i := range runs {
		if runs[i].AvgHeartRateBpm != nil {
			n++
		}
	}
	return n
}

// sumElevation sums gain over gain-bearing runs. nil — never 0 — when none
// carry gain: a treadmill week and a flat week are different facts.
func sumElevation(runs []activity.Activity) (*float64, int) {
	var sum float64
	n := 0
	for i := range runs {
		if runs[i].ElevationGainMeters == nil {
			continue
		}
		sum += *runs[i].ElevationGainMeters
		n++
	}
	if n == 0 {
		return nil, 0
	}
	return &sum, n
}

// weekRunRows projects the (already sorted) current-week runs onto the wire
// shape. HeartRateZone stays nil; the handler fills it when Run Effort is on.
func weekRunRows(week []activity.Activity, loc *time.Location) []RunningWeekRun {
	rows := make([]RunningWeekRun, 0, len(week))
	for i := range week {
		r := &week[i]
		rows = append(rows, RunningWeekRun{
			ActivityID:          r.ID,
			Name:                r.Name,
			StartTime:           r.StartTime,
			LocalDate:           r.StartTime.In(loc).Format("2006-01-02"),
			DistanceMeters:      r.DistanceMeters,
			DurationSeconds:     r.DurationSeconds,
			AvgPaceSecPerKm:     r.AvgPaceSecPerKm,
			AvgHeartRateBpm:     r.AvgHeartRateBpm,
			ElevationGainMeters: r.ElevationGainMeters,
			Environment:         string(r.Environment),
		})
	}
	return rows
}

// buildBaseline averages the four buckets BEFORE the current week over the
// weeks that actually held a run (see RunningBaseline). nil when none did.
func buildBaseline(runs []activity.Activity, now time.Time, loc *time.Location) *RunningBaseline {
	starts := weeklyBucketStarts(now, loc, baselineWindowWeeks+1)[:baselineWindowWeeks]
	var window []activity.Activity
	weekHasRun := make([]bool, len(starts))
	for i := range runs {
		if runs[i].ActivityType != activity.ActivityRunning {
			continue
		}
		idx := weekIndex(starts, localWeekStart(runs[i].StartTime, loc))
		if idx < 0 {
			continue
		}
		window = append(window, runs[i])
		weekHasRun[idx] = true
	}
	weeks := 0
	for _, has := range weekHasRun {
		if has {
			weeks++
		}
	}
	if weeks == 0 {
		return nil
	}

	var dist float64
	var dur int
	for i := range window {
		dist += window[i].DistanceMeters
		dur += window[i].DurationSeconds
	}
	b := &RunningBaseline{WindowWeeks: baselineWindowWeeks, Weeks: weeks}
	avgDist := dist / float64(weeks)
	b.DistanceMeters = &avgDist
	avgDur := dur / weeks
	b.DurationSeconds = &avgDur
	rpw := float64(len(window)) / float64(weeks)
	b.RunsPerWeek = &rpw
	b.AvgPaceSecPerKm = aggregatePace(dur, dist)
	b.AvgHeartRateBpm = durationWeightedHR(window)
	if gain, n := sumElevation(window); n > 0 {
		avgGain := *gain / float64(weeks)
		b.ElevationGainMeters = &avgGain
	}
	return b
}

// weeklyLoad buckets runs into the last sparkWeeks local weeks,
// oldest→newest. A bucket with no runs is a real zero (count 0, sums 0, nil
// elevation) — the distinction weekly_distance_spark could not make.
func weeklyLoad(runs []activity.Activity, now time.Time, loc *time.Location) []RunningWeekPoint {
	starts := weeklyBucketStarts(now, loc, sparkWeeks)
	points := make([]RunningWeekPoint, len(starts))
	for i := range starts {
		points[i] = RunningWeekPoint{WeekStart: starts[i].Format("2006-01-02")}
	}
	for i := range runs {
		if runs[i].ActivityType != activity.ActivityRunning {
			continue
		}
		idx := weekIndex(starts, localWeekStart(runs[i].StartTime, loc))
		if idx < 0 {
			continue
		}
		p := &points[idx]
		p.DistanceMeters += runs[i].DistanceMeters
		p.DurationSeconds += runs[i].DurationSeconds
		p.RunCount++
		if g := runs[i].ElevationGainMeters; g != nil {
			if p.ElevationGainMeters == nil {
				var zero float64
				p.ElevationGainMeters = &zero
			}
			*p.ElevationGainMeters += *g
		}
	}
	return points
}
```

- [ ] **Step 5: Run the package tests**

Run: `go test ./internal/dashboard/ -v -run TestBuildRunning`
Expected: PASS (all new tests plus the four pre-existing `TestBuildRunning_*` tests — they assert only pass-through/latest/spark and must keep passing).

Run: `go test ./internal/dashboard/`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add internal/dashboard/dto.go internal/dashboard/running.go internal/dashboard/running_test.go
git commit -m "feat(dashboard): extend running section with baseline, week runs, and weekly load"
```

### Task 3: Handler — family re-gate, HR-zone wiring, server wiring

**Files:**
- Modify: `internal/dashboard/handler.go`
- Modify: `internal/server/server.go` (dashboard handler wiring, ~line 710)
- Test: `internal/dashboard/handler_test.go`, `internal/dashboard/summary_layout_test.go`

**Context:** the gating mirrors the `recoveryFamily` loop already in `summary` (handler.go ~line 236). `buildRunningSection` gains a `withZones bool` parameter. The engine is injected via a setter mirroring `activity.Handler.SetHRZonesEngine` (`internal/activity/handler.go:179`) so `NewHandler`'s signature and every existing test stay untouched. A failed reference read must yield nil zones, never a fabricated population-default zone — so the `defer1` wrapper returns a `*hrzones.Reference` whose zero value (nil) means "skip zones".

- [ ] **Step 1: Write the failing gating + zone tests**

Append to `internal/dashboard/summary_layout_test.go`:

```go
// TestSummary_RunningFamilyTileAlone_YieldsRunningSection pins the family
// re-gate: a layout containing ONLY running_vertical must still produce a
// populated "running" section — and no "running_vertical" key. Second family
// after recovery where the section-key set is deliberately not a subset of
// layout.
func TestSummary_RunningFamilyTileAlone_YieldsRunningSection(t *testing.T) {
	r, rp, userID := newTestEnv(t)
	seedRun(t, rp, userID, testNow.Add(-24*time.Hour), 5000)

	if err := rp.layout.Upsert(context.Background(), userID, []TileID{TileRunningVertical}); err != nil {
		t.Fatalf("layout upsert: %v", err)
	}

	layout, data := dataEnvelope(t, r, userID, "?timezone=UTC")

	if !equalStrs(layout, []string{"running_vertical"}) {
		t.Errorf("layout = %v, want [running_vertical]", layout)
	}
	assertKeysPresent(t, data, "running")
	assertKeysAbsent(t, data, "running_vertical")
	if string(data["running"]) == "null" {
		t.Error("running = null, want a populated section (user has a run)")
	}
}

// TestSummary_MultipleRunningFamilyTiles_OneRunningSection asserts the
// section is built and emitted exactly once under "running" no matter how
// many family tiles are on the layout.
func TestSummary_MultipleRunningFamilyTiles_OneRunningSection(t *testing.T) {
	r, rp, userID := newTestEnv(t)
	seedRun(t, rp, userID, testNow.Add(-24*time.Hour), 5000)

	ids := []TileID{TileRunning, TileRunningLog, TileRunningEffort}
	if err := rp.layout.Upsert(context.Background(), userID, ids); err != nil {
		t.Fatalf("layout upsert: %v", err)
	}

	layout, data := dataEnvelope(t, r, userID, "?timezone=UTC")

	if !equalStrs(layout, []string{"running", "running_log", "running_effort"}) {
		t.Errorf("layout = %v, want [running running_log running_effort]", layout)
	}
	assertKeysPresent(t, data, "running")
	assertKeysAbsent(t, data, "running_log", "running_effort", "running_vertical")
}

// TestSummary_NoRunningFamilyTile_NoRunningSection asserts the inverse: with
// no running-family tile enabled, the "running" key is absent even though the
// user has runs.
func TestSummary_NoRunningFamilyTile_NoRunningSection(t *testing.T) {
	r, rp, userID := newTestEnv(t)
	seedRun(t, rp, userID, testNow.Add(-24*time.Hour), 5000)

	if err := rp.layout.Upsert(context.Background(), userID, []TileID{TileLifting, TileStreak}); err != nil {
		t.Fatalf("layout upsert: %v", err)
	}

	_, data := dataEnvelope(t, r, userID, "?timezone=UTC")

	assertKeysAbsent(t, data, "running", "running_log", "running_effort", "running_vertical")
}
```

Append to `internal/dashboard/handler_test.go` (add `"github.com/jwallace145/progressive-overload-fitness-tracker/internal/hrzones"` to its imports):

```go
// testHRZonesEngine mirrors the production config shape with round numbers:
// population default 190 puts a 150-bpm run in zone 3 (bounds 114/133/152/171).
func testHRZonesEngine() *hrzones.Engine {
	return hrzones.New(hrzones.Config{
		PopulationDefaultMaxHR: 190,
		CalibratedRunThreshold: 8,
		RecencyWindowDays:      90,
		MinReferenceBpm:        120,
		MaxReferenceBpm:        220,
		ZoneUpperBounds:        []float64{0.60, 0.70, 0.80, 0.90},
		ZoneNames:              []string{"Zone 1", "Zone 2", "Zone 3", "Zone 4", "Zone 5"},
	})
}

// seedRunWithHR seeds a current-week run carrying an average HR.
func seedRunWithHR(t *testing.T, rp *repos, userID string, start time.Time, distanceMeters float64, hr int) {
	t.Helper()
	bpm := hr
	a := &activity.Activity{
		UserID:           userID,
		ActivityType:     activity.ActivityRunning,
		IngestSource:     activity.IngestManualTCX,
		SourceActivityID: start.Format("20060102T150405") + "hr",
		StartTime:        start,
		DistanceMeters:   distanceMeters,
		DurationSeconds:  1800,
		AvgHeartRateBpm:  &bpm,
	}
	if err := rp.activity.Create(context.Background(), a, []byte("<tcx/>")); err != nil {
		t.Fatalf("seed run with hr: %v", err)
	}
}

func runningSectionFrom(t *testing.T, data map[string]json.RawMessage) RunningSection {
	t.Helper()
	var section RunningSection
	if err := json.Unmarshal(data["running"], &section); err != nil {
		t.Fatalf("decode running section: %v; raw=%s", err, data["running"])
	}
	return section
}

// TestSummary_HeartRateZones_OnlyWhenEffortEnabled: zones are the family's
// one extra read, gated on the Run Effort tile. Without it every
// HeartRateZone is nil; with it an HR-bearing run classifies against the
// reference (150 bpm at the 190 population default → zone 3).
func TestSummary_HeartRateZones_OnlyWhenEffortEnabled(t *testing.T) {
	r, rp, userID := newTestEnv(t)
	seedRunWithHR(t, rp, userID, testNow.Add(-24*time.Hour), 5000, 150)

	if err := rp.layout.Upsert(context.Background(), userID, []TileID{TileRunning}); err != nil {
		t.Fatalf("layout upsert: %v", err)
	}
	_, data := dataEnvelope(t, r, userID, "?timezone=UTC")
	section := runningSectionFrom(t, data)
	if len(section.WeekRuns) != 1 {
		t.Fatalf("week runs = %d, want 1", len(section.WeekRuns))
	}
	if section.WeekRuns[0].HeartRateZone != nil {
		t.Errorf("zone = %v, want nil when running_effort is not enabled", section.WeekRuns[0].HeartRateZone)
	}

	if err := rp.layout.Upsert(context.Background(), userID, []TileID{TileRunningEffort}); err != nil {
		t.Fatalf("layout upsert: %v", err)
	}
	_, data = dataEnvelope(t, r, userID, "?timezone=UTC")
	section = runningSectionFrom(t, data)
	if len(section.WeekRuns) != 1 || section.WeekRuns[0].HeartRateZone == nil {
		t.Fatalf("zone nil, want populated when running_effort is enabled; runs=%+v", section.WeekRuns)
	}
	if *section.WeekRuns[0].HeartRateZone != 3 {
		t.Errorf("zone = %d, want 3 (150 bpm at a 190 reference)", *section.WeekRuns[0].HeartRateZone)
	}
}

// errHRStatsRepo fails only RecentHRStats, proving a reference-read failure
// degrades to nil zones — never a 500, never a fabricated population-default
// zone.
type errHRStatsRepo struct {
	activity.Repository
}

func (errHRStatsRepo) RecentHRStats(ctx context.Context, userID string, window time.Duration, excludeActivityID string) (hrzones.Stats, error) {
	return hrzones.Stats{}, errors.New("hr stats boom")
}

func TestSummary_HeartRateZones_ReferenceReadFailure_DegradesToNil(t *testing.T) {
	_, rp, userID := newTestEnv(t)
	seedRunWithHR(t, rp, userID, testNow.Add(-24*time.Hour), 5000, 150)
	if err := rp.layout.Upsert(context.Background(), userID, []TileID{TileRunningEffort}); err != nil {
		t.Fatalf("layout upsert: %v", err)
	}

	h := NewHandler(errHRStatsRepo{rp.activity}, rp.workout, rp.exercise, rp.steps, rp.nutrition, rp.bodyweight, rp.bloodPressure, rp.user, rp.whoopConn, rp.whoopRec, rp.layout, testRecoveryEngine())
	h.now = func() time.Time { return testNow }
	h.SetHRZonesEngine(testHRZonesEngine(), 90*24*time.Hour)
	r2 := chi.NewRouter()
	h.Mount(r2)

	_, data := dataEnvelope(t, r2, userID, "?timezone=UTC")
	section := runningSectionFrom(t, data)
	if len(section.WeekRuns) != 1 {
		t.Fatalf("week runs = %d, want 1 (section must survive the failed read)", len(section.WeekRuns))
	}
	if section.WeekRuns[0].HeartRateZone != nil {
		t.Errorf("zone = %v, want nil after a failed reference read", section.WeekRuns[0].HeartRateZone)
	}
}

// TestSummary_RunningMetricsAgreeWithWeekRuns pins SOW Open Question 1: the
// metrics path (SQL scan) and the slice path (week_runs) see the same rows
// for the current week, so their figures must agree — drift fails loudly.
func TestSummary_RunningMetricsAgreeWithWeekRuns(t *testing.T) {
	r, rp, userID := newTestEnv(t)
	seedRun(t, rp, userID, testNow.Add(-24*time.Hour), 5000)
	seedRun(t, rp, userID, testNow.Add(-48*time.Hour), 8000)

	if err := rp.layout.Upsert(context.Background(), userID, []TileID{TileRunning}); err != nil {
		t.Fatalf("layout upsert: %v", err)
	}
	_, data := dataEnvelope(t, r, userID, "?timezone=UTC")
	section := runningSectionFrom(t, data)

	var sum float64
	for _, wr := range section.WeekRuns {
		sum += wr.DistanceMeters
	}
	if sum != section.CurrentWeek.DistanceMeters {
		t.Errorf("Σ week_runs distance = %v, metrics distance = %v — the two provenances drifted", sum, section.CurrentWeek.DistanceMeters)
	}
	if len(section.WeekRuns) != section.CurrentWeek.RunCount {
		t.Errorf("len(week_runs) = %d, metrics run_count = %d", len(section.WeekRuns), section.CurrentWeek.RunCount)
	}
}
```

Note: `testNow` is Wed 2026-06-17 13:00 UTC, so `-24h`/`-48h` are Tue/Mon of the same UTC week — both in the current week.

- [ ] **Step 2: Run to verify they fail**

Run: `go test ./internal/dashboard/ -run 'TestSummary_Running|TestSummary_HeartRate|TestSummary_NoRunningFamily|TestSummary_MultipleRunning' -v`
Expected: FAIL / compile error (`h.SetHRZonesEngine` undefined; family gating absent).

- [ ] **Step 3: Implement handler changes**

In `internal/dashboard/handler.go`:

(a) Add the import `"github.com/jwallace145/progressive-overload-fitness-tracker/internal/hrzones"`.

(b) Add fields to `Handler` (after `recovery`):

```go
	// hrEngine classifies each week run's average HR into a zone for the
	// Run Effort tile. Optional: injected post-construction via
	// SetHRZonesEngine (mirroring activity.Handler); nil leaves every
	// HeartRateZone nil.
	hrEngine *hrzones.Engine
	// hrWindow is the recency window RecentHRStats summarizes for the
	// reference max-HR estimate.
	hrWindow time.Duration
```

(c) Add the setter after `NewHandler`:

```go
// SetHRZonesEngine wires the heart-rate-zone engine (and its recency window)
// in so week runs carry a heart_rate_zone when the Run Effort tile is
// enabled. Called from server wiring after construction, mirroring
// activity.Handler's setter; a setter rather than a constructor arg keeps
// NewHandler's signature and every existing test untouched. Safe to never
// call — buildRunningSection nil-guards and leaves zones nil.
func (h *Handler) SetHRZonesEngine(e *hrzones.Engine, window time.Duration) {
	h.hrEngine = e
	h.hrWindow = window
}
```

(d) In `summary`, replace

```go
	if enabled[TileRunning] {
		out[string(TileRunning)] = h.buildRunningSection(ctx, r, userID, endurance, now, loc)
	}
```

with:

```go
	// Every running-family tile reads the ONE shared "running" section, so it
	// is built once when ANY family tile is enabled and emitted only under the
	// "running" key — the same section-key/layout divergence the recovery
	// family established below.
	runningFamily := []TileID{
		TileRunning, TileRunningLog, TileRunningEffort, TileRunningVertical,
	}
	for _, id := range runningFamily {
		if enabled[id] {
			out[string(TileRunning)] = h.buildRunningSection(
				ctx, r, userID, endurance, now, loc, enabled[TileRunningEffort],
			)
			break
		}
	}
```

(e) Replace `buildRunningSection` with:

```go
// buildRunningSection fetches the running metrics and assembles the section
// from them plus the already-fetched 53-week run list. withZones additionally
// classifies each HR-bearing week run against the user's max-HR reference —
// only when the Run Effort tile is enabled, because the reference is the
// family's one extra read. A nil engine or a failed reference read leaves
// every zone nil (degraded, never a 500, never a fabricated zone): the defer1
// wrapper yields a nil *Reference on error, which skips classification.
func (h *Handler) buildRunningSection(ctx context.Context, r *http.Request, userID string, runs []activity.Activity, now time.Time, loc *time.Location, withZones bool) *RunningSection {
	metrics := defer1(ctx, r, "running metrics", func() (activity.Metrics, error) {
		return h.activityRepo.RunningMetrics(ctx, userID, now, loc)
	})
	section := buildRunning(metrics, runs, now, loc)
	if section == nil || !withZones || h.hrEngine == nil {
		return section
	}

	ref := defer1(ctx, r, "running hr reference", func() (*hrzones.Reference, error) {
		stats, err := h.activityRepo.RecentHRStats(ctx, userID, h.hrWindow, "")
		if err != nil {
			return nil, err
		}
		estimated := h.hrEngine.EstimateReference(stats)
		return &estimated, nil
	})
	if ref == nil {
		return section
	}
	for i := range section.WeekRuns {
		if bpm := section.WeekRuns[i].AvgHeartRateBpm; bpm != nil {
			// The engine is 0-indexed; the wire field is 1-indexed to match
			// the web's --zone-1..5 tokens.
			zone := h.hrEngine.ZoneForBPM(*ref, *bpm) + 1
			section.WeekRuns[i].HeartRateZone = &zone
		}
	}
	return section
}
```

(f) Wire the test env: in `handler_test.go`'s `newTestEnv`, after `h.now = func() time.Time { return testNow }`, add:

```go
	h.SetHRZonesEngine(testHRZonesEngine(), 90*24*time.Hour)
```

- [ ] **Step 4: Wire the server**

In `internal/server/server.go` (~line 710), replace the one-line dashboard construction:

```go
		dashboard.NewHandler(activityRepo, workoutRepo, exerciseRepo, stepsRepo, nutritionRepo, bodyweightRepo, bloodPressureRepo, userRepo, whoopConnRepo, whoopRecoveryRepo, dashboard.NewSQLiteLayoutRepository(database), recoveryEngine).Mount(r)
```

with:

```go
		dashboardHandler := dashboard.NewHandler(activityRepo, workoutRepo, exerciseRepo, stepsRepo, nutritionRepo, bodyweightRepo, bloodPressureRepo, userRepo, whoopConnRepo, whoopRecoveryRepo, dashboard.NewSQLiteLayoutRepository(database), recoveryEngine)
		// Same engine and window as the activity handler: the Run Effort tile
		// classifies week runs against the same max-HR reference the run
		// detail page uses, so the two surfaces can never disagree.
		dashboardHandler.SetHRZonesEngine(hrEngine, time.Duration(cfg.HRZones.RecencyWindowDays)*24*time.Hour)
		dashboardHandler.Mount(r)
```

(keep the existing explanatory comment above it).

- [ ] **Step 5: Run the tests**

Run: `go test ./internal/dashboard/ && go build ./... && go vet ./...`
Expected: PASS. All pre-existing tests (default layout, stored layout, recovery family, streak) must still pass — the default layout still contains only `TileRunning` and yields the `running` key exactly as before.

- [ ] **Step 6: Full-repo check**

Run: `go test ./...`
Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add internal/dashboard/handler.go internal/dashboard/handler_test.go internal/dashboard/summary_layout_test.go internal/server/server.go
git commit -m "feat(dashboard): gate running section on any family tile and wire hr zones"
```

---

## Repo 2: prog-strength-web (`/workspace/prog-strength-web`)

Run all commands from `/workspace/prog-strength-web`. `npm ci` has been run. Test with `npm run test -- <file>`; the Husky pre-commit hook runs lint-staged + a full typecheck — commits fail if types are broken, so tasks are ordered to keep every commit type-clean. **The catalog mirror and the `TileCard` cases land in the same task (Task 8)** — adding a `TileId` without a renderer case is a compile error by design.

### Task 4: Payload mirror — `lib/api.ts` + `lib/dashboard.ts`

**Files:**
- Modify: `lib/api.ts` (`DashboardRunning` + three new types, ~line 4526)
- Modify: `lib/dashboard.ts` (`RunningView`, `adaptRunning`)
- Test: `lib/dashboard.test.ts`

- [ ] **Step 1: Create the branch**

```bash
cd /workspace/prog-strength-web && git checkout -b feat/running-tile-family
```

- [ ] **Step 2: Replace the `DashboardRunning` wire type**

In `lib/api.ts`, replace the existing `DashboardRunning` type and its doc comment with:

```ts
/**
 * The dashboard's running section — the ONE shared payload every
 * running-family tile reads (running / running_log / running_effort /
 * running_vertical). `current_week` carries this week's aggregates
 * (`avg_pace_sec_per_km` is the WEEK aggregate; `avg_heart_rate_bpm` is
 * duration-weighted over `heart_rate_runs` HR-bearing runs;
 * `elevation_gain_meters` is null — not zero — when no run carried
 * altitude). `baseline` is the trailing 4-week average excluding the
 * current week, null until a prior week holds a run. `week_runs` is this
 * local week oldest→newest; `heart_rate_zone` (1..5) is null unless the
 * Run Effort tile is enabled. `weekly_load` is 8 week-anchored buckets
 * oldest→newest — a zero bucket is a real zero week.
 * `recent_avg_pace_sec_per_km` is a 30-DAY aggregate, a different figure
 * from `current_week.avg_pace_sec_per_km`. The server still emits the
 * legacy `weekly_distance_spark`; this client no longer reads it (it dies
 * with the retired card in a follow-up API contract PR).
 */
export type DashboardRunningWeekRun = {
  activity_id: string;
  name: string | null;
  start_time: string;
  local_date: string; // YYYY-MM-DD in the user's tz
  distance_meters: number;
  duration_seconds: number;
  avg_pace_sec_per_km: number | null;
  avg_heart_rate_bpm: number | null;
  heart_rate_zone: number | null; // 1..5
  elevation_gain_meters: number | null;
  environment: "outdoor" | "indoor";
};

export type DashboardRunningWeekPoint = {
  week_start: string; // YYYY-MM-DD, local Monday
  distance_meters: number;
  duration_seconds: number;
  run_count: number;
  elevation_gain_meters: number | null;
};

export type DashboardRunningBaseline = {
  window_weeks: number;
  weeks: number;
  distance_meters: number | null;
  duration_seconds: number | null;
  avg_pace_sec_per_km: number | null;
  avg_heart_rate_bpm: number | null;
  elevation_gain_meters: number | null;
  runs_per_week: number | null;
};

export type DashboardRunning = {
  current_week: {
    distance_meters: number;
    run_count: number;
    delta_pct_vs_prior_week: number | null;
    duration_seconds: number;
    avg_pace_sec_per_km: number | null;
    avg_heart_rate_bpm: number | null;
    elevation_gain_meters: number | null;
    heart_rate_runs: number;
    elevation_runs: number;
    longest_run_meters: number;
    days_run: number;
  };
  baseline: DashboardRunningBaseline | null;
  recent_avg_pace_sec_per_km: number | null;
  latest_run: {
    name: string | null;
    distance_meters: number;
    duration_seconds: number;
    start_time: string;
  } | null;
  week_runs: DashboardRunningWeekRun[];
  weekly_load: DashboardRunningWeekPoint[];
};
```

(`DashboardWalking`'s own `weekly_distance_spark` is untouched.)

- [ ] **Step 3: Replace `RunningView` and `adaptRunning` in `lib/dashboard.ts`**

Replace the `RunningView` type with (and export the three sub-view types):

```ts
/** One of this week's runs, display-shaped. Raw sec/km and metres are kept
 * alongside the display strings ONLY for unit-free comparisons and chart
 * proportions (ratios of two server figures) — never for re-conversion. */
export type RunningWeekRunView = {
  activityId: string;
  name: string | null;
  localDate: string; // YYYY-MM-DD
  startTime: string;
  distance: string; // display-unit numeric string, no suffix
  durationSeconds: number;
  pace: string; // "m:ss" in display unit, or "—"
  paceSecPerKm: number | null;
  avgHeartRate: number | null;
  heartRateZone: number | null; // 1..5, null unless Run Effort enabled
  elevation: string | null; // display string with suffix; null = source carried none
  elevationGainMeters: number | null;
  indoor: boolean;
};

export type RunningWeekPointView = {
  weekStart: string; // YYYY-MM-DD, local Monday
  distance: number; // display-unit numeric, for charting
  durationSeconds: number;
  runCount: number;
};

/** Trailing 4-week averages excluding the current week; null until a prior
 * week holds a run. `weeks` is the denominator actually used (weeks with a
 * run), so tiles can caption "4-wk avg · 3 weeks" honestly. */
export type RunningBaselineView = {
  windowWeeks: number;
  weeks: number;
  distance: string | null;
  durationSeconds: number | null;
  pace: string | null;
  paceSecPerKm: number | null;
  avgHeartRate: number | null;
  elevation: string | null;
  runsPerWeek: number | null;
};

/** Display view-model for the running-family tiles. */
export type RunningView = {
  currentWeek: {
    distance: string;
    runCount: number;
    deltaPct: number | null;
    durationSeconds: number;
    pace: string; // THIS WEEK's aggregate — label distinctly from `pace` below
    avgHeartRate: number | null; // duration-weighted
    elevation: string | null; // null = no run carried altitude (≠ "0 ft")
    heartRateRuns: number;
    elevationRuns: number;
    longestRun: string;
    daysRun: number;
  };
  pace: string; // 30-DAY aggregate — labelled "30d", never "this week"
  baseline: RunningBaselineView | null;
  latestRun: {
    name: string | null;
    distance: string;
    durationSeconds: number;
    startTime: string;
  } | null;
  weekRuns: RunningWeekRunView[]; // oldest→newest
  weeklyLoad: RunningWeekPointView[]; // 8 buckets, oldest→newest
  unit: DistanceUnit;
};
```

Add `formatElevationValue` to the import from `@/lib/distance-unit-context`, and import the new wire types from `@/lib/api`. Replace `adaptRunning` with:

```ts
/** Elevation display, preserving null (missing ≠ zero — the tiles branch on it). */
function elevationOrNull(meters: number | null, unit: DistanceUnit): string | null {
  return meters === null ? null : formatElevationValue(meters, unit);
}

function adaptRunningWeekRun(r: DashboardRunningWeekRun, unit: DistanceUnit): RunningWeekRunView {
  return {
    activityId: r.activity_id,
    name: r.name,
    localDate: r.local_date,
    startTime: r.start_time,
    distance: formatDistanceValue(r.distance_meters, unit),
    durationSeconds: r.duration_seconds,
    pace: formatPaceValue(r.avg_pace_sec_per_km, unit),
    paceSecPerKm: r.avg_pace_sec_per_km,
    avgHeartRate: r.avg_heart_rate_bpm,
    heartRateZone: r.heart_rate_zone,
    elevation: elevationOrNull(r.elevation_gain_meters, unit),
    elevationGainMeters: r.elevation_gain_meters,
    indoor: r.environment === "indoor",
  };
}

function adaptRunningBaseline(b: DashboardRunningBaseline, unit: DistanceUnit): RunningBaselineView {
  return {
    windowWeeks: b.window_weeks,
    weeks: b.weeks,
    distance: b.distance_meters === null ? null : formatDistanceValue(b.distance_meters, unit),
    durationSeconds: b.duration_seconds,
    pace: b.avg_pace_sec_per_km === null ? null : formatPaceValue(b.avg_pace_sec_per_km, unit),
    paceSecPerKm: b.avg_pace_sec_per_km,
    avgHeartRate: b.avg_heart_rate_bpm,
    elevation: elevationOrNull(b.elevation_gain_meters, unit),
    runsPerWeek: b.runs_per_week,
  };
}

function adaptRunning(running: DashboardRunning, unit: DistanceUnit): RunningView {
  return {
    currentWeek: {
      distance: formatDistanceValue(running.current_week.distance_meters, unit),
      runCount: running.current_week.run_count,
      deltaPct: running.current_week.delta_pct_vs_prior_week,
      durationSeconds: running.current_week.duration_seconds,
      pace: formatPaceValue(running.current_week.avg_pace_sec_per_km, unit),
      avgHeartRate: running.current_week.avg_heart_rate_bpm,
      elevation: elevationOrNull(running.current_week.elevation_gain_meters, unit),
      heartRateRuns: running.current_week.heart_rate_runs,
      elevationRuns: running.current_week.elevation_runs,
      longestRun: formatDistanceValue(running.current_week.longest_run_meters, unit),
      daysRun: running.current_week.days_run,
    },
    pace: formatPaceValue(running.recent_avg_pace_sec_per_km, unit),
    baseline: running.baseline ? adaptRunningBaseline(running.baseline, unit) : null,
    latestRun: running.latest_run
      ? {
          name: running.latest_run.name,
          distance: formatDistanceValue(running.latest_run.distance_meters, unit),
          durationSeconds: running.latest_run.duration_seconds,
          startTime: running.latest_run.start_time,
        }
      : null,
    weekRuns: running.week_runs.map((r) => adaptRunningWeekRun(r, unit)),
    weeklyLoad: running.weekly_load.map((p) => ({
      weekStart: p.week_start,
      distance: distanceToDisplay(p.distance_meters, unit),
      durationSeconds: p.duration_seconds,
      runCount: p.run_count,
    })),
    unit,
  };
}
```

`spark` is gone from `RunningView`; `EnduranceView`'s spark and `adaptEndurance` are untouched.

- [ ] **Step 4: Update `lib/dashboard.test.ts`**

The fixture's `running` block and every `running:` payload in the file must move to the new shape. Replace the `fullSummary.running` block with:

```ts
  running: {
    current_week: {
      distance_meters: 34278,
      run_count: 4,
      delta_pct_vs_prior_week: 10.9,
      duration_seconds: 12977,
      avg_pace_sec_per_km: 378.6,
      avg_heart_rate_bpm: 153,
      elevation_gain_meters: 274,
      heart_rate_runs: 3,
      elevation_runs: 3,
      longest_run_meters: 20438,
      days_run: 4,
    },
    baseline: {
      window_weeks: 4,
      weeks: 3,
      distance_meters: 27358,
      duration_seconds: 10440,
      avg_pace_sec_per_km: 381.2,
      avg_heart_rate_bpm: 150,
      elevation_gain_meters: 198,
      runs_per_week: 3.75,
    },
    recent_avg_pace_sec_per_km: 376.5,
    latest_run: {
      name: "Saturday long run",
      distance_meters: 20438,
      duration_seconds: 7784,
      start_time: "2026-08-01T11:02:00Z",
    },
    week_runs: [
      {
        activity_id: "a1",
        name: "Easy shakeout",
        start_time: "2026-07-27T11:00:00Z",
        local_date: "2026-07-27",
        distance_meters: 5633,
        duration_seconds: 2128,
        avg_pace_sec_per_km: 377.8,
        avg_heart_rate_bpm: 148,
        heart_rate_zone: 2,
        elevation_gain_meters: 38,
        environment: "outdoor",
      },
      {
        activity_id: "a2",
        name: null,
        start_time: "2026-07-28T10:30:00Z",
        local_date: "2026-07-28",
        distance_meters: 4184,
        duration_seconds: 1490,
        avg_pace_sec_per_km: 356.1,
        avg_heart_rate_bpm: 152,
        heart_rate_zone: null,
        elevation_gain_meters: null,
        environment: "indoor",
      },
    ],
    weekly_load: [
      {
        week_start: "2026-07-20",
        distance_meters: 30900,
        duration_seconds: 11760,
        run_count: 4,
        elevation_gain_meters: 231,
      },
      {
        week_start: "2026-07-27",
        distance_meters: 34278,
        duration_seconds: 12977,
        run_count: 4,
        elevation_gain_meters: null,
      },
    ],
  },
```

Update the running assertions in `"adaptDashboard — full payload"`:

```ts
  it("converts running distances/pace to miles for a mi profile", () => {
    const data = adaptDashboard(fullSummary, profile({ distance_unit: "mi" }));
    expect(data.running.present).toBe(true);
    if (!data.running.present) throw new Error("running absent");

    // 34278 m / 1609.344 = 21.3 mi
    expect(data.running.currentWeek.distance).toBe("21.3");
    expect(data.running.currentWeek.runCount).toBe(4);
    expect(data.running.currentWeek.deltaPct).toBe(10.9);
    // Week aggregate: 378.6 s/km → 10:09 per mile. 30-day: 376.5 → 10:06.
    expect(data.running.currentWeek.pace).toBe("10:09");
    expect(data.running.pace).toBe("10:06");
    // 274 m → 899 ft, with coverage counts passed through.
    expect(data.running.currentWeek.elevation).toBe("899 ft");
    expect(data.running.currentWeek.heartRateRuns).toBe(3);
    expect(data.running.currentWeek.elevationRuns).toBe(3);
    expect(data.running.currentWeek.longestRun).toBe("12.7");
    expect(data.running.currentWeek.daysRun).toBe(4);
    // Baseline converted once, raw sec/km retained for comparisons.
    expect(data.running.baseline?.distance).toBe("17.0");
    expect(data.running.baseline?.pace).toBe("10:13");
    expect(data.running.baseline?.paceSecPerKm).toBe(381.2);
    expect(data.running.baseline?.weeks).toBe(3);
    // Week runs oldest→newest; nulls preserved (never coerced to 0).
    expect(data.running.weekRuns).toHaveLength(2);
    expect(data.running.weekRuns[0].distance).toBe("3.5");
    expect(data.running.weekRuns[0].heartRateZone).toBe(2);
    expect(data.running.weekRuns[1].indoor).toBe(true);
    expect(data.running.weekRuns[1].elevation).toBeNull();
    expect(data.running.weekRuns[1].avgHeartRate).toBe(152);
    // Weekly load in display units for charting.
    expect(data.running.weeklyLoad).toHaveLength(2);
    expect(data.running.weeklyLoad[0].distance).toBeCloseTo(19.2, 1);
    expect(data.running.weeklyLoad[1].runCount).toBe(4);
    expect(data.running.unit).toBe("mi");
  });

  it("no longer reads weekly_distance_spark from the running section", () => {
    // The deployed API still emits the legacy field during expand/contract;
    // the adapter must ignore it (and must not throw on its absence either).
    const withLegacy = {
      ...fullSummary,
      running: {
        ...fullSummary.running!,
        weekly_distance_spark: [1, 2, 3],
      } as unknown as NonNullable<typeof fullSummary.running>,
    };
    const data = adaptDashboard(withLegacy, profile({ distance_unit: "mi" }));
    if (!data.running.present) throw new Error("running absent");
    expect("spark" in data.running).toBe(false);
  });

  it("converts running to km for a km profile", () => {
    const data = adaptDashboard(fullSummary, profile({ distance_unit: "km" }));
    if (!data.running.present) throw new Error("running absent");
    expect(data.running.currentWeek.distance).toBe("34.3");
    expect(data.running.currentWeek.pace).toBe("6:19");
    expect(data.running.currentWeek.elevation).toBe("274 m");
    expect(data.running.weekRuns[0].distance).toBe("5.6");
  });
```

Fix every other `running:` payload in the file (the `"null goals / deltas"` and `"sanitization"` and `"layout-aware"` describes) to the minimal new shape — where an old fixture was

```ts
running: { current_week: {...3 fields}, recent_avg_pace_sec_per_km: X, latest_run: null, weekly_distance_spark: [...] }
```

use this minimal builder added near the top of the file and reuse it:

```ts
function minimalRunning(overrides: Partial<DashboardRunning> = {}): DashboardRunning {
  return {
    current_week: {
      distance_meters: 5000,
      run_count: 1,
      delta_pct_vs_prior_week: null,
      duration_seconds: 1800,
      avg_pace_sec_per_km: 360,
      avg_heart_rate_bpm: null,
      elevation_gain_meters: null,
      heart_rate_runs: 0,
      elevation_runs: 0,
      longest_run_meters: 5000,
      days_run: 1,
    },
    baseline: null,
    recent_avg_pace_sec_per_km: null,
    latest_run: null,
    week_runs: [],
    weekly_load: [],
    ...overrides,
  };
}
```

(import `DashboardRunning` type into the test). The walking/cycling/hiking fixtures keep `weekly_distance_spark` — the endurance adapter still reads it; keep the existing endurance spark assertions passing. Delete only running-spark assertions (e.g. the sanitization test's running-spark case — repoint that test at the walking spark).

- [ ] **Step 5: Run the tests**

Run: `npm run test -- lib/dashboard.test.ts && npm run typecheck`
Expected: dashboard tests PASS. Typecheck FAILS in exactly one place: `tile-renderer.tsx`'s `RunningCard` reads `v.spark`/`v.pace` etc. **Fix the minimal bridge now** (the full rewrite is Task 8): in `tile-renderer.tsx`, change `RunningCard`'s body to render from the new view without `spark` — replace the `<Spark …/>` line with nothing and the MetaRow `pace` item stays (`v.pace` still exists). i.e.:

```tsx
  const v: RunningView = section;
  return (
    <MiniCard title="Running" href={href}>
      <BigNum value={v.currentWeek.distance} suffix={`${v.unit} this week`} />
      <MetaRow
        items={[
          { label: "runs", value: compact(v.currentWeek.runCount) },
          { label: "pace", value: v.pace },
          { label: "last", value: v.latestRun ? `${v.latestRun.distance} ${v.unit}` : null },
        ]}
      />
    </MiniCard>
  );
```

This bridge is deleted in Task 8; it exists only so this commit typechecks.

Run: `npm run typecheck && npm run test`
Expected: PASS (if `tile-renderer.test.tsx` or others assert on the running spark, update those assertions minimally).

- [ ] **Step 6: Commit**

```bash
git add lib/api.ts lib/dashboard.ts lib/dashboard.test.ts "app/(app)/dashboard/_components/tile-renderer.tsx"
git commit -m "feat(dashboard): mirror the extended running section payload"
```

### Task 5: `running/` shared module, fixtures, empty card

**Files:**
- Create: `app/(app)/dashboard/_components/running/shared.ts`
- Create: `app/(app)/dashboard/_components/running/shared.test.ts`
- Create: `app/(app)/dashboard/_components/running/fixtures.ts`
- Create: `app/(app)/dashboard/_components/running/empty-card.tsx`

- [ ] **Step 1: Write the failing shared tests**

`app/(app)/dashboard/_components/running/shared.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import {
  loadStatus,
  loadStatusColor,
  signedPct,
  signed,
  paceDelta,
  weekdayLabel,
  coverageCaption,
  zoneToken,
} from "./shared";

describe("loadStatus", () => {
  it("maps a zero-run week to resting regardless of delta", () => {
    expect(loadStatus(-100, 0)).toBe("resting");
  });
  it("maps +25% to ramping — and its color is warning, never danger", () => {
    expect(loadStatus(25, 4)).toBe("ramping");
    expect(loadStatusColor("ramping")).toBe("var(--warning)");
    expect(loadStatusColor("ramping")).not.toBe("var(--danger)");
  });
  it("maps a steady build to building/success", () => {
    expect(loadStatus(0, 3)).toBe("building");
    expect(loadStatus(9.9, 3)).toBe("building");
    expect(loadStatusColor("building")).toBe("var(--success)");
  });
  it("maps a down week to easing/muted — never danger", () => {
    expect(loadStatus(-30, 2)).toBe("easing");
    expect(loadStatusColor("easing")).toBe("var(--muted)");
    expect(loadStatusColor("easing")).not.toBe("var(--danger)");
  });
  it("maps resting to muted", () => {
    expect(loadStatusColor("resting")).toBe("var(--muted)");
  });
});

describe("signedPct / signed", () => {
  it("formats with a unicode minus", () => {
    expect(signedPct(25.3)).toBe("+25%");
    expect(signedPct(-100)).toBe("−100%");
    expect(signedPct(0)).toBe("±0%");
    expect(signed(3)).toBe("+3");
    expect(signed(-17)).toBe("−17");
    expect(signed(0)).toBe("±0");
  });
});

describe("paceDelta", () => {
  it("converts the sec/km delta into the display unit's seconds", () => {
    // 378.6 vs 381.2 s/km is −2.6 s/km ≈ −4 s/mi.
    expect(paceDelta(378.6, 381.2, "mi")).toBe("−4s");
    expect(paceDelta(378.6, 381.2, "km")).toBe("−3s");
    expect(paceDelta(381.2, 381.2, "mi")).toBe("±0s");
    expect(paceDelta(390, 381.2, "km")).toBe("+9s");
  });
});

describe("weekdayLabel", () => {
  it("parses a local date with no timezone drift", () => {
    // 2026-08-01 is a Saturday everywhere — a UTC-parse off-by-one would say Fri.
    expect(weekdayLabel("2026-08-01")).toBe("Sat");
    expect(weekdayLabel("2026-07-27")).toBe("Mon");
  });
  it("degrades malformed input to an em-dash", () => {
    expect(weekdayLabel("not-a-date")).toBe("—");
  });
});

describe("coverageCaption", () => {
  it("states coverage honestly", () => {
    expect(coverageCaption(3, 4)).toBe("3 of 4 runs");
    expect(coverageCaption(0, 1)).toBe("0 of 1 run");
  });
});

describe("zoneToken", () => {
  it("maps 1..5 to the zone scale and everything else to neutral ink", () => {
    expect(zoneToken(3)).toBe("var(--zone-3)");
    expect(zoneToken(1)).toBe("var(--zone-1)");
    expect(zoneToken(5)).toBe("var(--zone-5)");
    expect(zoneToken(null)).toBe("var(--muted)");
    expect(zoneToken(0)).toBe("var(--muted)");
    expect(zoneToken(6)).toBe("var(--muted)");
  });
});
```

- [ ] **Step 2: Run to verify it fails**

Run: `npm run test -- "app/(app)/dashboard/_components/running/shared.test.ts"`
Expected: FAIL (module not found).

- [ ] **Step 3: Implement `shared.ts`**

```ts
/**
 * Shared formatting and status→token mapping for the four running tiles.
 *
 * Single-sourced on purpose: four hand-rolled copies of the status switch is
 * exactly how a +25% week ends up red. The mapping is the SOW's color
 * contract — ramping reads as warning (a big week is a CHOICE, not a
 * failure: nothing in this family is ever danger red), building as success,
 * resting/easing as muted. Pure functions, no React; reads the v0.4 CSS vars
 * by name, never a raw hex.
 */

import type { DistanceUnit } from "@/lib/distance-unit-context";

const KM_PER_MILE = 1.609344;

/** The ramp band above which a build reads as warning (per week, %). */
export const RAMP_WARNING_PCT = 10;

export type LoadStatus = "resting" | "ramping" | "building" | "easing";

/** SOW status bands. A zero-run week is resting no matter what the delta says. */
export function loadStatus(deltaPct: number, runCount: number): LoadStatus {
  if (runCount === 0) return "resting";
  if (deltaPct >= RAMP_WARNING_PCT) return "ramping";
  if (deltaPct >= 0) return "building";
  return "easing";
}

/** Status → CSS var. Never `--danger`: running more than usual is not an emergency. */
export function loadStatusColor(status: LoadStatus): string {
  switch (status) {
    case "ramping":
      return "var(--warning)";
    case "building":
      return "var(--success)";
    default:
      return "var(--muted)"; // resting, easing
  }
}

/** Signed integer with a unicode minus: +3 / −17 / ±0. */
export function signed(n: number): string {
  const r = Math.round(n);
  if (r === 0) return "±0";
  return r > 0 ? `+${r}` : `−${Math.abs(r)}`;
}

/** Signed percentage: +25% / −100% / ±0%. */
export function signedPct(pct: number): string {
  return `${signed(pct)}%`;
}

/**
 * Signed pace delta vs a baseline, converted to seconds in the display unit
 * (s/mi under "mi", s/km under "km") — a signed difference of two server
 * figures, the only pace arithmetic a tile may do.
 */
export function paceDelta(
  secPerKm: number,
  baselineSecPerKm: number,
  unit: DistanceUnit,
): string {
  const factor = unit === "mi" ? KM_PER_MILE : 1;
  return `${signed((secPerKm - baselineSecPerKm) * factor)}s`;
}

/** "2026-08-01" → "Sat" — parsed as local date parts, no timezone drift. */
export function weekdayLabel(iso: string): string {
  const [y, m, d] = iso.split("-").map(Number);
  if (!y || !m || !d) return "—";
  return new Date(y, m - 1, d).toLocaleDateString("en-US", { weekday: "short" });
}

/** Honest coverage caption: "3 of 4 runs". */
export function coverageCaption(n: number, total: number): string {
  return `${n} of ${total} ${total === 1 ? "run" : "runs"}`;
}

/**
 * Zone index (1..5, as the API ships it) → its CSS var. Anything else —
 * including null when Run Effort's engine is unwired — reads neutral ink,
 * never a guessed zone.
 */
export function zoneToken(zone: number | null): string {
  if (zone === null || zone < 1 || zone > 5) return "var(--muted)";
  return `var(--zone-${zone})`;
}
```

- [ ] **Step 4: Run to verify it passes**

Run: `npm run test -- "app/(app)/dashboard/_components/running/shared.test.ts"`
Expected: PASS

- [ ] **Step 5: Add the empty card and the fixtures**

`app/(app)/dashboard/_components/running/empty-card.tsx`:

```tsx
/**
 * RunningEmptyCard — the shared `present: false` body for every
 * running-family tile. One empty grammar, four different headings (each tile
 * passes its catalog title), over the existing import-a-run CTA — the same
 * generalization RecoveryConnectCard made for its family.
 */

import { MiniCard, MiniCardEmpty } from "../mini-card";

export function RunningEmptyCard({ title, href }: { title: string; href: string }) {
  return (
    <MiniCard title={title} href={href}>
      <MiniCardEmpty cta="Import a run to start tracking" />
    </MiniCard>
  );
}
```

`app/(app)/dashboard/_components/running/fixtures.ts` — view-level fixtures for the four SOW states, parameterized by unit (display strings are authored per unit; the adapter's own conversion is covered by `lib/dashboard.test.ts`):

```ts
/**
 * RunningView fixtures for the four SOW states, in both units. View-level on
 * purpose (like recovery/fixtures.ts): component tests assert rendering, not
 * conversion — lib/dashboard.test.ts owns the adapter. The ordinary week
 * mirrors the DX headline fixture: 4 runs, one indoor treadmill (no
 * elevation), one with no HR, one 12.7 mi long run — three awkward states
 * visible in the default view.
 */

import type { DistanceUnit } from "@/lib/distance-unit-context";
import type { RunningView, RunningWeekPointView, RunningWeekRunView } from "@/lib/dashboard";

function load(unit: DistanceUnit, weeks: [string, number, number, number][]): RunningWeekPointView[] {
  const div = unit === "mi" ? 1609.344 : 1000;
  return weeks.map(([weekStart, meters, durationSeconds, runCount]) => ({
    weekStart,
    distance: meters / div,
    durationSeconds,
    runCount,
  }));
}

const WEEKS: [string, number, number, number][] = [
  ["2026-06-08", 24140, 9180, 3],
  ["2026-06-15", 27359, 10230, 4],
  ["2026-06-22", 21726, 8220, 3],
  ["2026-06-29", 24140, 9180, 3],
  ["2026-07-06", 29772, 11310, 4],
  ["2026-07-13", 0, 0, 0], // down week — a real zero
  ["2026-07-20", 30900, 11760, 4],
  ["2026-07-27", 34278, 12977, 4],
];

function weekRuns(unit: DistanceUnit): RunningWeekRunView[] {
  const mi = unit === "mi";
  return [
    {
      activityId: "a1",
      name: "Easy shakeout",
      localDate: "2026-07-27",
      startTime: "2026-07-27T11:00:00Z",
      distance: mi ? "3.5" : "5.6",
      durationSeconds: 2128,
      pace: mi ? "10:08" : "6:18",
      paceSecPerKm: 377.8,
      avgHeartRate: 148,
      heartRateZone: 2,
      elevation: mi ? "125 ft" : "38 m",
      elevationGainMeters: 38,
      indoor: false,
    },
    {
      activityId: "a2",
      name: null,
      localDate: "2026-07-28",
      startTime: "2026-07-28T10:30:00Z",
      distance: mi ? "2.6" : "4.2",
      durationSeconds: 1490,
      pace: mi ? "9:33" : "5:56",
      paceSecPerKm: 356.1,
      avgHeartRate: 152,
      heartRateZone: 3,
      elevation: null, // treadmill: the source carried no altitude
      elevationGainMeters: null,
      indoor: true,
    },
    {
      activityId: "a3",
      name: "Lunch run",
      localDate: "2026-07-30",
      startTime: "2026-07-30T16:15:00Z",
      distance: mi ? "2.5" : "4.0",
      durationSeconds: 1575,
      pace: mi ? "10:30" : "6:32",
      paceSecPerKm: 391.5,
      avgHeartRate: null, // manual import: no HR
      heartRateZone: null,
      elevation: mi ? "72 ft" : "22 m",
      elevationGainMeters: 22,
      indoor: false,
    },
    {
      activityId: "a4",
      name: "Saturday long run",
      localDate: "2026-08-01",
      startTime: "2026-08-01T11:02:00Z",
      distance: mi ? "12.7" : "20.4",
      durationSeconds: 7784,
      pace: mi ? "10:13" : "6:21",
      paceSecPerKm: 380.9,
      avgHeartRate: 156,
      heartRateZone: 3,
      elevation: mi ? "702 ft" : "214 m",
      elevationGainMeters: 214,
      indoor: false,
    },
  ];
}

/** An ordinary training week: 21.3 mi over 4 runs, +24% time on feet. */
export function ordinaryWeek(unit: DistanceUnit = "mi"): RunningView {
  const mi = unit === "mi";
  return {
    currentWeek: {
      distance: mi ? "21.3" : "34.3",
      runCount: 4,
      deltaPct: 10.9,
      durationSeconds: 12977,
      pace: mi ? "10:09" : "6:19",
      avgHeartRate: 153,
      elevation: mi ? "899 ft" : "274 m",
      heartRateRuns: 3,
      elevationRuns: 3,
      longestRun: mi ? "12.7" : "20.4",
      daysRun: 4,
    },
    pace: mi ? "10:06" : "6:17",
    baseline: {
      windowWeeks: 4,
      weeks: 3,
      distance: mi ? "17.0" : "27.4",
      durationSeconds: 10440,
      pace: mi ? "10:13" : "6:21",
      paceSecPerKm: 381.2,
      avgHeartRate: 150,
      elevation: mi ? "650 ft" : "198 m",
      runsPerWeek: 3.75,
    },
    latestRun: {
      name: "Saturday long run",
      distance: mi ? "12.7" : "20.4",
      durationSeconds: 7784,
      startTime: "2026-08-01T11:02:00Z",
    },
    weekRuns: weekRuns(unit),
    weeklyLoad: load(unit, WEEKS),
    unit,
  };
}

/** Monday morning / a skipped week: zero runs, baseline intact, last run 5 days old. */
export function zeroRuns(unit: DistanceUnit = "mi"): RunningView {
  const base = ordinaryWeek(unit);
  return {
    ...base,
    currentWeek: {
      distance: "0.0",
      runCount: 0,
      deltaPct: -100,
      durationSeconds: 0,
      pace: "—",
      avgHeartRate: null,
      elevation: null,
      heartRateRuns: 0,
      elevationRuns: 0,
      longestRun: "0.0",
      daysRun: 0,
    },
    latestRun: {
      name: "Saturday long run",
      distance: unit === "mi" ? "12.7" : "20.4",
      durationSeconds: 7784,
      startTime: "2026-08-01T11:02:00Z",
    },
    weekRuns: [],
    weeklyLoad: load(unit, [...WEEKS.slice(1), ["2026-08-03", 0, 0, 0]]),
  };
}

/** A brand-new runner: one run ever, no baseline, no delta, load almost all zeros. */
export function firstRunEver(unit: DistanceUnit = "mi"): RunningView {
  const mi = unit === "mi";
  return {
    currentWeek: {
      distance: mi ? "3.5" : "5.6",
      runCount: 1,
      deltaPct: null,
      durationSeconds: 2128,
      pace: mi ? "10:08" : "6:18",
      avgHeartRate: 148,
      elevation: mi ? "125 ft" : "38 m",
      heartRateRuns: 1,
      elevationRuns: 1,
      longestRun: mi ? "3.5" : "5.6",
      daysRun: 1,
    },
    pace: mi ? "10:08" : "6:18",
    baseline: null,
    latestRun: {
      name: "First run",
      distance: mi ? "3.5" : "5.6",
      durationSeconds: 2128,
      startTime: "2026-07-27T11:00:00Z",
    },
    weekRuns: [weekRuns(unit)[0]],
    weeklyLoad: load(unit, [
      ["2026-06-08", 0, 0, 0],
      ["2026-06-15", 0, 0, 0],
      ["2026-06-22", 0, 0, 0],
      ["2026-06-29", 0, 0, 0],
      ["2026-07-06", 0, 0, 0],
      ["2026-07-13", 0, 0, 0],
      ["2026-07-20", 0, 0, 0],
      ["2026-07-27", 5633, 2128, 1],
    ]),
    unit,
  };
}

/** An indoor-only week: three treadmill runs, every elevation null, HR intact. */
export function indoorOnly(unit: DistanceUnit = "mi"): RunningView {
  const base = ordinaryWeek(unit);
  const treadmill = weekRuns(unit)[1];
  const runs: RunningWeekRunView[] = [
    { ...treadmill, activityId: "t1", localDate: "2026-07-27", heartRateZone: 2 },
    { ...treadmill, activityId: "t2", localDate: "2026-07-29", avgHeartRate: 149 },
    { ...treadmill, activityId: "t3", localDate: "2026-07-31", avgHeartRate: 155 },
  ];
  return {
    ...base,
    currentWeek: {
      ...base.currentWeek,
      runCount: 3,
      elevation: null, // nil — not "0 ft" — no run carried altitude
      elevationRuns: 0,
      heartRateRuns: 3,
      daysRun: 3,
    },
    weekRuns: runs,
  };
}
```

- [ ] **Step 6: Typecheck and commit**

Run: `npm run typecheck && npm run test -- "app/(app)/dashboard/_components/running/shared.test.ts"`
Expected: PASS

```bash
git add "app/(app)/dashboard/_components/running/"
git commit -m "feat(dashboard): running family shared module, fixtures, and empty card"
```

### Task 6: `load-ramp.tsx` — Training Load (id `running`)

**Files:**
- Create: `app/(app)/dashboard/_components/running/load-ramp.tsx`
- Test: `app/(app)/dashboard/_components/running/load-ramp.test.tsx`

**Binding color/behavior:** the delta figure is the ONLY thing carrying status color (success/warning/muted — never danger). The rail and caption stay neutral. Hero is **time on feet** (duration), not distance. `loadDeltaPct = (currentWeek.durationSeconds − baseline.durationSeconds) / baseline.durationSeconds × 100` — a signed delta of two server figures. `baseline == null` or `baseline.durationSeconds` null/0 → "first week": print the week's own time on feet, no percentage, no NaN, no `+Infinity%`. Zero-run week: `−100% · resting`, plus the last run stated (dated via its own timestamp — not "N days since", which would need an impure `Date.now()` in render). The rail scales to max(weeklyLoad ∪ baseline) so the ghost baseline line is always on canvas.

- [ ] **Step 1: Write the failing tests**

`load-ramp.test.tsx`:

```tsx
/// <reference types="vitest/globals" />

import { render, screen } from "@testing-library/react";
import { LoadRampCard } from "./load-ramp";
import { firstRunEver, indoorOnly, ordinaryWeek, zeroRuns } from "./fixtures";

const UNITS = ["mi", "km"] as const;

describe("LoadRampCard", () => {
  it.each(UNITS)("heroes the signed duration delta on an ordinary week (%s)", (unit) => {
    render(<LoadRampCard section={ordinaryWeek(unit)} href="/x" />);
    // (12977 − 10440) / 10440 = +24.3% → +24%, a ramp → warning word.
    expect(screen.getByText("+24%")).toBeInTheDocument();
    expect(screen.getByText(/ramping/)).toBeInTheDocument();
    // Plain-language read: this week vs the 4-week average, as durations.
    expect(screen.getByText(/3:36:17/)).toBeInTheDocument();
    expect(screen.getByText(/2:54:00/)).toBeInTheDocument();
  });

  it("colors the ramp with the warning token, never danger", () => {
    render(<LoadRampCard section={ordinaryWeek("mi")} href="/x" />);
    const hero = screen.getByText("+24%");
    expect(hero).toHaveStyle({ color: "var(--warning)" });
  });

  it.each(UNITS)("says something true on a zero-run week (%s)", (unit) => {
    render(<LoadRampCard section={zeroRuns(unit)} href="/x" />);
    expect(screen.getByText("−100%")).toBeInTheDocument();
    expect(screen.getByText(/resting/)).toBeInTheDocument();
    // The last run is stated and dated — not promoted into this week's slot.
    expect(screen.getByText(/last run/i)).toBeInTheDocument();
    expect(screen.queryByText("NaN")).not.toBeInTheDocument();
  });

  it.each(UNITS)("renders a first week with no percentage and no NaN (%s)", (unit) => {
    const { container } = render(<LoadRampCard section={firstRunEver(unit)} href="/x" />);
    expect(screen.getByText(/first week/)).toBeInTheDocument();
    // The hero is the week's own time on feet.
    expect(screen.getByText("35:28")).toBeInTheDocument();
    expect(container.textContent).not.toMatch(/%/);
    expect(container.textContent).not.toMatch(/NaN|Infinity/);
  });

  it("still renders a full body for an indoor-only week", () => {
    const { container } = render(<LoadRampCard section={indoorOnly("mi")} href="/x" />);
    expect(container.textContent).not.toBe("—");
    expect(screen.getByText(/vs/)).toBeInTheDocument();
  });

  it("renders the 8-bucket rail with the baseline ghost line", () => {
    render(<LoadRampCard section={ordinaryWeek("mi")} href="/x" />);
    expect(screen.getByRole("img", { name: /eight weeks/i })).toBeInTheDocument();
  });
});
```

- [ ] **Step 2: Run to verify it fails**

Run: `npm run test -- "app/(app)/dashboard/_components/running/load-ramp.test.tsx"`
Expected: FAIL (module not found)

- [ ] **Step 3: Implement**

`load-ramp.tsx`:

```tsx
/**
 * LoadRampCard — the `running` tile, rewritten ("Training Load").
 *
 * Heroes the DIRECTION and deliberately demotes this week's total: one large
 * signed figure for this week's time on feet against the 4-week baseline — a
 * signed delta of two server figures, the only client arithmetic the SOW
 * permits — with the plain-language read beneath and an 8-bucket neutral
 * rail of weekly load with the baseline as a ghost line through it. The
 * strongest size contrast in the family: one big numeral over a small rail.
 *
 * Status carries the delta and NOTHING else — success for a steady build,
 * warning above +10%/week (a big week is a choice, not a failure: never
 * danger red), muted for resting/easing; the rail stays neutral ink. A nil
 * baseline (first week ever) prints the week's own time on feet with "first
 * week" beneath and no percentage — no NaN, no +Infinity%. The zero-run week
 * renders "−100% · resting" with the last run stated and dated, never
 * promoted into this week's slot.
 */

import type { RunningView } from "@/lib/dashboard";
import { formatDuration } from "@/lib/format";
import { MiniCard } from "../mini-card";
import { loadStatus, loadStatusColor, signedPct } from "./shared";

const TITLE = "Training Load";

export function LoadRampCard({ section, href }: { section: RunningView; href: string }) {
  const { currentWeek, baseline, weeklyLoad, latestRun, unit } = section;

  // A zero-duration baseline cannot anchor a ramp — treated as no baseline.
  const baselineDuration =
    baseline?.durationSeconds != null && baseline.durationSeconds > 0
      ? baseline.durationSeconds
      : null;
  const deltaPct =
    baselineDuration === null
      ? null
      : ((currentWeek.durationSeconds - baselineDuration) / baselineDuration) * 100;
  const status = loadStatus(deltaPct ?? 0, currentWeek.runCount);
  const heroColor = deltaPct === null ? "var(--foreground)" : loadStatusColor(status);

  // Scale to max(weeklyLoad ∪ baseline) so the ghost line is always on canvas.
  const railMax = Math.max(...weeklyLoad.map((p) => p.durationSeconds), baselineDuration ?? 0, 1);

  return (
    <MiniCard title={TITLE} href={href}>
      <div className="flex items-baseline gap-2">
        <span
          className="font-mono text-3xl font-semibold tracking-tight tabular-nums"
          style={{ color: heroColor }}
        >
          {deltaPct === null ? formatDuration(currentWeek.durationSeconds) : signedPct(deltaPct)}
        </span>
      </div>
      <p className="-mt-1 text-xs text-[var(--muted)]">{caption()}</p>

      <div
        className="relative mt-1 flex h-8 items-end gap-[3px]"
        role="img"
        aria-label="Eight weeks of running load with your four-week average"
      >
        {weeklyLoad.map((p) => (
          <span
            key={p.weekStart}
            className="flex-1 rounded-[1px]"
            style={
              p.durationSeconds > 0
                ? {
                    // Floored so a light week is still a visible bar, not a sliver.
                    height: `${Math.max(10, Math.round((p.durationSeconds / railMax) * 100))}%`,
                    backgroundColor: "var(--faint)",
                  }
                : // A zero week is a REAL zero: a 2px stub, not a missing bar.
                  { height: "2px", backgroundColor: "var(--surface-2)" }
            }
          />
        ))}
        {baselineDuration !== null && (
          <span
            aria-hidden="true"
            className="absolute inset-x-0 border-t border-dashed border-[var(--muted)]"
            style={{ bottom: `${Math.min(98, Math.round((baselineDuration / railMax) * 100))}%` }}
          />
        )}
      </div>
    </MiniCard>
  );

  function caption(): string {
    if (currentWeek.runCount === 0) {
      const last = latestRun
        ? ` · last run ${latestRun.distance} ${unit}, ${runDay(latestRun.startTime)}`
        : "";
      return `resting${last}`;
    }
    if (deltaPct === null || baselineDuration === null) {
      return "first week";
    }
    return `${status} · ${formatDuration(currentWeek.durationSeconds)} vs ${formatDuration(baselineDuration)} avg`;
  }
}

/** Short dated label for the latest run ("Aug 1") — pure function of props. */
function runDay(startTime: string): string {
  const d = new Date(startTime);
  if (Number.isNaN(d.getTime())) return "—";
  return d.toLocaleDateString("en-US", { month: "short", day: "numeric" });
}
```

- [ ] **Step 4: Run to verify it passes**

Run: `npm run test -- "app/(app)/dashboard/_components/running/load-ramp.test.tsx" && npm run typecheck`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add "app/(app)/dashboard/_components/running/load-ramp.tsx" "app/(app)/dashboard/_components/running/load-ramp.test.tsx"
git commit -m "feat(dashboard): training load tile (load-ramp)"
```

### Task 7: `week-log.tsx` — Runs This Week (id `running_log`)

**Files:**
- Create: `app/(app)/dashboard/_components/running/week-log.tsx`
- Test: `app/(app)/dashboard/_components/running/week-log.test.tsx`

**Binding:** NO numeral above 14px — the flattest type on the grid; a quiet caption header over dated tabular rows (`font-mono` permitted). Sage is spent **entirely on the per-row pace figure** — `--discipline-run-fg` (brighter) when that run beat `baseline.paceSecPerKm`, `--discipline-run-dot` when it did not, plain when no baseline. An indoor run carries a small glyph, not a blank column; a no-HR run shows an em-dash in the HR column only. More than four runs collapses the oldest into `+N earlier`. Zero runs renders the last run's row under a `no runs this week` caption, dated so last week is never mistaken for this one.

- [ ] **Step 1: Write the failing tests**

`week-log.test.tsx`:

```tsx
/// <reference types="vitest/globals" />

import { render, screen } from "@testing-library/react";
import type { RunningView } from "@/lib/dashboard";
import { WeekLogCard } from "./week-log";
import { firstRunEver, ordinaryWeek, zeroRuns } from "./fixtures";

const UNITS = ["mi", "km"] as const;

describe("WeekLogCard", () => {
  it.each(UNITS)("renders each run as a dated row under a quiet header (%s)", (unit) => {
    render(<WeekLogCard section={ordinaryWeek(unit)} href="/x" />);
    // Header caption: total · runs · time.
    const total = unit === "mi" ? "21.3" : "34.3";
    expect(screen.getByText(new RegExp(`${total} ${unit} · 4 runs · 3:36:17`))).toBeInTheDocument();
    // Rows are dated by weekday.
    expect(screen.getByText("Sat")).toBeInTheDocument();
    expect(screen.getByText("Mon")).toBeInTheDocument();
    // The long run's figures are on its row.
    expect(screen.getByText(unit === "mi" ? "12.7" : "20.4")).toBeInTheDocument();
  });

  it("shows an em-dash only in the HR column of a no-HR run", () => {
    render(<WeekLogCard section={ordinaryWeek("mi")} href="/x" />);
    // Exactly one of the four runs (Lunch run) lacks HR.
    expect(screen.getAllByText("—")).toHaveLength(1);
    // Its other columns still speak: the pace is present.
    expect(screen.getByText("10:30")).toBeInTheDocument();
  });

  it("marks the indoor run with a glyph instead of a blank column", () => {
    render(<WeekLogCard section={ordinaryWeek("mi")} href="/x" />);
    expect(screen.getByLabelText("indoor run")).toBeInTheDocument();
  });

  it("collapses a five-run week to four rows plus an earlier line", () => {
    const five: RunningView = ordinaryWeek("mi");
    five.weekRuns = [
      { ...five.weekRuns[0], activityId: "a0", localDate: "2026-07-26" },
      ...five.weekRuns,
    ];
    render(<WeekLogCard section={five} href="/x" />);
    expect(screen.getByText("+1 earlier")).toBeInTheDocument();
  });

  it("brightens the pace of runs that beat the baseline", () => {
    render(<WeekLogCard section={ordinaryWeek("mi")} href="/x" />);
    // Treadmill run 356.1 beat the 381.2 baseline → bright sage.
    expect(screen.getByText("9:33")).toHaveStyle({ color: "var(--discipline-run-fg)" });
    // The 391.5 lunch run did not → dim sage.
    expect(screen.getByText("10:30")).toHaveStyle({ color: "var(--discipline-run-dot)" });
  });

  it.each(UNITS)("dates the last run clearly on a zero-run week (%s)", (unit) => {
    render(<WeekLogCard section={zeroRuns(unit)} href="/x" />);
    expect(screen.getByText(/no runs this week/)).toBeInTheDocument();
    // The last run is dated (short date, not a bare weekday), so last week
    // is never mistaken for this one.
    expect(screen.getByText(/Aug 1/)).toBeInTheDocument();
  });

  it("renders a single-row week without implying more", () => {
    const { container } = render(<WeekLogCard section={firstRunEver("mi")} href="/x" />);
    expect(screen.getByText("Mon")).toBeInTheDocument();
    expect(container.textContent).not.toMatch(/earlier/);
  });
});
```

- [ ] **Step 2: Run to verify it fails**

Run: `npm run test -- "app/(app)/dashboard/_components/running/week-log.test.tsx"`
Expected: FAIL

- [ ] **Step 3: Implement**

`week-log.tsx`:

```tsx
/**
 * WeekLogCard — the `running_log` tile ("Runs This Week").
 *
 * Heroes the RUNS THEMSELVES: no numeral above 14px — the flattest type on
 * the grid and the strongest possible contrast with the old BigNum card. A
 * quiet caption header (total · runs · time) over dated tabular rows,
 * hairline-divided, mono figures. Sage is spent ENTIRELY on the per-row pace
 * — brighter when that run beat the 4-week baseline pace (a comparison of
 * two server figures), faint when it did not. An indoor run carries a glyph
 * rather than a blank column; a run with no HR shows an em-dash in that
 * column only. More than four runs collapses the oldest into "+N earlier"
 * rather than growing the card. Zero runs renders the last run's row under a
 * "no runs this week" caption, short-dated so last week is never mistaken
 * for this one.
 */

import type { RunningView, RunningWeekRunView } from "@/lib/dashboard";
import { formatDuration } from "@/lib/format";
import { MiniCard } from "../mini-card";
import { weekdayLabel } from "./shared";

const TITLE = "Runs This Week";
// Row ceiling keeping a heavy week inside the ~180px budget (SOW OQ 7).
const MAX_ROWS = 4;

export function WeekLogCard({ section, href }: { section: RunningView; href: string }) {
  const { currentWeek, weekRuns, baseline, latestRun, unit } = section;

  if (weekRuns.length === 0) {
    return (
      <MiniCard title={TITLE} href={href}>
        <p className="text-xs text-[var(--muted)]">no runs this week</p>
        {latestRun ? (
          <div className="flex items-baseline gap-2 border-t border-[var(--border)] pt-1.5 font-mono text-xs tabular-nums">
            <span className="text-[var(--muted)]">{shortDate(latestRun.startTime)}</span>
            <span className="flex-1 text-[var(--foreground)]">
              {latestRun.distance} {unit}
            </span>
            <span className="text-[10px] text-[var(--faint)]">last run</span>
          </div>
        ) : (
          <p className="text-xs text-[var(--faint)]">Log a run to fill this in.</p>
        )}
      </MiniCard>
    );
  }

  const newestFirst = [...weekRuns].reverse();
  const visible = newestFirst.slice(0, MAX_ROWS);
  const earlier = newestFirst.length - visible.length;

  return (
    <MiniCard title={TITLE} href={href}>
      <p className="text-xs text-[var(--muted)]">
        {currentWeek.distance} {unit} · {currentWeek.runCount}{" "}
        {currentWeek.runCount === 1 ? "run" : "runs"} ·{" "}
        {formatDuration(currentWeek.durationSeconds)}
      </p>
      <div className="flex flex-col">
        {visible.map((r) => (
          <Row key={r.activityId} run={r} unit={unit} baselinePace={baseline?.paceSecPerKm ?? null} />
        ))}
        {earlier > 0 && (
          <p className="pt-1 font-mono text-[10px] text-[var(--faint)]">+{earlier} earlier</p>
        )}
      </div>
    </MiniCard>
  );
}

function Row({
  run,
  unit,
  baselinePace,
}: {
  run: RunningWeekRunView;
  unit: string;
  baselinePace: number | null;
}) {
  return (
    <div className="flex items-baseline gap-2 border-t border-[var(--border)] py-1 font-mono text-xs tabular-nums first:border-t-0">
      <span className="w-8 shrink-0 text-[var(--muted)]">{weekdayLabel(run.localDate)}</span>
      <span className="flex-1 text-[var(--foreground)]">
        {run.distance} {unit}
      </span>
      {run.indoor && (
        <span aria-label="indoor run" title="indoor" className="text-[10px] text-[var(--faint)]">
          ⌂
        </span>
      )}
      <span style={{ color: paceColor(run, baselinePace) }}>{run.pace}</span>
      <span className="w-7 shrink-0 text-right text-[var(--muted)]">{run.avgHeartRate ?? "—"}</span>
    </div>
  );
}

/** Sage lives here and only here: brighter when the run beat the baseline. */
function paceColor(run: RunningWeekRunView, baselinePace: number | null): string {
  if (run.paceSecPerKm === null) return "var(--discipline-run-dot)";
  if (baselinePace !== null && run.paceSecPerKm < baselinePace) {
    return "var(--discipline-run-fg)";
  }
  return "var(--discipline-run-dot)";
}

/** Short local date ("Aug 1") for the zero-week last-run row. */
function shortDate(startTime: string): string {
  const d = new Date(startTime);
  if (Number.isNaN(d.getTime())) return "—";
  return d.toLocaleDateString("en-US", { month: "short", day: "numeric" });
}
```

- [ ] **Step 4: Run to verify it passes**

Run: `npm run test -- "app/(app)/dashboard/_components/running/week-log.test.tsx" && npm run typecheck`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add "app/(app)/dashboard/_components/running/week-log.tsx" "app/(app)/dashboard/_components/running/week-log.test.tsx"
git commit -m "feat(dashboard): runs this week tile (week-log)"
```

### Task 8: `effort-heart.tsx` — Run Effort (id `running_effort`)

**Files:**
- Create: `app/(app)/dashboard/_components/running/effort-heart.tsx`
- Test: `app/(app)/dashboard/_components/running/effort-heart.test.tsx`

**Binding:** the ONLY tile that spends `--zone-1..5`, and **no sage at all**. Hero: the week's duration-weighted bpm (medium numeral) with `+3 vs 4-wk` beside it (signed delta of two server figures). Below: a horizontal rail, one dot per HR-bearing run, positioned by bpm, colored by `zoneToken(heartRateZone)` (neutral ink when zones are nil — degraded, not broken), sized by duration with a clamped legible minimum. Coverage stated whenever `heartRateRuns < runCount`. Language says "mostly zone N runs" — never minutes-in-zone. Zero HR-bearing runs → run count + "no heart-rate data this week", not an empty rail.

- [ ] **Step 1: Write the failing tests**

`effort-heart.test.tsx`:

```tsx
/// <reference types="vitest/globals" />

import { render, screen } from "@testing-library/react";
import type { RunningView } from "@/lib/dashboard";
import { EffortHeartCard } from "./effort-heart";
import { firstRunEver, ordinaryWeek, zeroRuns } from "./fixtures";

const UNITS = ["mi", "km"] as const;

describe("EffortHeartCard", () => {
  it.each(UNITS)("heroes the duration-weighted bpm with the baseline delta (%s)", (unit) => {
    render(<EffortHeartCard section={ordinaryWeek(unit)} href="/x" />);
    expect(screen.getByText("153")).toBeInTheDocument();
    // 153 vs baseline 150 — a signed delta of two server figures.
    expect(screen.getByText(/\+3 vs 4-wk/)).toBeInTheDocument();
  });

  it("states coverage when not every run carried HR", () => {
    render(<EffortHeartCard section={ordinaryWeek("mi")} href="/x" />);
    expect(screen.getByText(/3 of 4 runs/)).toBeInTheDocument();
  });

  it("renders one dot per HR-bearing run, colored by zone", () => {
    render(<EffortHeartCard section={ordinaryWeek("mi")} href="/x" />);
    const dots = screen.getAllByTestId("effort-dot");
    expect(dots).toHaveLength(3);
    expect(dots[0]).toHaveStyle({ backgroundColor: "var(--zone-2)" });
    expect(dots[1]).toHaveStyle({ backgroundColor: "var(--zone-3)" });
  });

  it("says which zone the week mostly was — never minutes-in-zone", () => {
    const { container } = render(<EffortHeartCard section={ordinaryWeek("mi")} href="/x" />);
    expect(screen.getByText(/mostly zone 3 runs/)).toBeInTheDocument();
    expect(container.textContent).not.toMatch(/min(ute)?s? in zone/i);
  });

  it("renders neutral dots when zones are nil (engine unwired)", () => {
    const section: RunningView = ordinaryWeek("mi");
    section.weekRuns = section.weekRuns.map((r) => ({ ...r, heartRateZone: null }));
    render(<EffortHeartCard section={section} href="/x" />);
    for (const dot of screen.getAllByTestId("effort-dot")) {
      expect(dot).toHaveStyle({ backgroundColor: "var(--muted)" });
    }
    // Without zones the card must not claim a zone in words either.
    expect(screen.queryByText(/mostly zone/)).not.toBeInTheDocument();
  });

  it("handles a week with no HR-bearing runs honestly", () => {
    const section: RunningView = ordinaryWeek("mi");
    section.weekRuns = section.weekRuns.map((r) => ({
      ...r,
      avgHeartRate: null,
      heartRateZone: null,
    }));
    section.currentWeek = { ...section.currentWeek, avgHeartRate: null, heartRateRuns: 0 };
    render(<EffortHeartCard section={section} href="/x" />);
    expect(screen.getByText(/4 runs/)).toBeInTheDocument();
    expect(screen.getByText(/no heart-rate data this week/)).toBeInTheDocument();
    expect(screen.queryAllByTestId("effort-dot")).toHaveLength(0);
  });

  it.each(UNITS)("says something true on a zero-run week (%s)", (unit) => {
    const { container } = render(<EffortHeartCard section={zeroRuns(unit)} href="/x" />);
    expect(screen.getByText(/no runs this week/)).toBeInTheDocument();
    expect(container.textContent).not.toBe("—");
  });

  it("survives n = 1 without implying a distribution", () => {
    render(<EffortHeartCard section={firstRunEver("mi")} href="/x" />);
    expect(screen.getByText("148")).toBeInTheDocument();
    expect(screen.getAllByTestId("effort-dot")).toHaveLength(1);
    // No baseline → no delta claim; one run → no "mostly zone" claim either.
    expect(screen.queryByText(/vs 4-wk/)).not.toBeInTheDocument();
    expect(screen.queryByText(/mostly zone/)).not.toBeInTheDocument();
  });
});
```

- [ ] **Step 2: Run to verify it fails**

Run: `npm run test -- "app/(app)/dashboard/_components/running/effort-heart.test.tsx"`
Expected: FAIL

- [ ] **Step 3: Implement**

`effort-heart.tsx`:

```tsx
/**
 * EffortHeartCard — the `running_effort` tile ("Run Effort").
 *
 * Heroes HEART RATE: the week's duration-weighted bpm as a medium numeral
 * with "+3 vs 4-wk" beside it (a signed delta of two server figures), over a
 * horizontal rail of one dot per HR-bearing run — positioned by bpm, colored
 * by the server-classified HeartRateZone (the ONLY surface in the family
 * spending --zone-1..5; no sage on this card at all, which is what stops it
 * looking like its siblings), and sized by duration with a clamped legible
 * minimum so the long run reads bigger without turning shakeouts into dust.
 *
 * Honesty rules: coverage is stated ("3 of 4 runs") whenever a run lacks HR;
 * the copy says "mostly zone N runs" and NEVER claims minutes-in-zone (the
 * classification is per-run average HR, not a trackpoint scan); nil zones
 * (engine unwired / reference read failed) degrade to neutral-ink dots with
 * the bpm figures intact.
 */

import type { RunningView } from "@/lib/dashboard";
import { MiniCard } from "../mini-card";
import { coverageCaption, signed, zoneToken } from "./shared";

const TITLE = "Run Effort";
const DOT_MIN_PX = 8;
const DOT_MAX_PX = 16;

export function EffortHeartCard({ section, href }: { section: RunningView; href: string }) {
  const { currentWeek, weekRuns, baseline } = section;

  if (currentWeek.runCount === 0) {
    return (
      <MiniCard title={TITLE} href={href}>
        <p className="text-sm text-[var(--muted)]">no runs this week</p>
        {baseline?.avgHeartRate != null && (
          <p className="text-[11px] text-[var(--faint)]">
            4-wk average <span className="font-mono tabular-nums">{baseline.avgHeartRate}</span> bpm
          </p>
        )}
      </MiniCard>
    );
  }

  const hrRuns = weekRuns.filter((r) => r.avgHeartRate !== null);
  if (currentWeek.avgHeartRate === null || hrRuns.length === 0) {
    return (
      <MiniCard title={TITLE} href={href}>
        <p className="text-sm text-[var(--muted)]">
          {currentWeek.runCount} {currentWeek.runCount === 1 ? "run" : "runs"}
        </p>
        <p className="text-[11px] text-[var(--faint)]">no heart-rate data this week</p>
      </MiniCard>
    );
  }

  const delta =
    baseline?.avgHeartRate != null ? currentWeek.avgHeartRate - baseline.avgHeartRate : null;

  const bpms = hrRuns.map((r) => r.avgHeartRate as number);
  const minBpm = Math.min(...bpms);
  const bpmSpan = Math.max(1, Math.max(...bpms) - minBpm);
  const maxDur = Math.max(...hrRuns.map((r) => r.durationSeconds), 1);

  const zones = hrRuns
    .map((r) => r.heartRateZone)
    .filter((z): z is number => z !== null);
  // "mostly" needs at least two classified runs — n = 1 must not imply a
  // distribution (SOW state matrix).
  const modalZone = zones.length > 1 ? mode(zones) : null;

  return (
    <MiniCard title={TITLE} href={href}>
      <div className="flex items-baseline gap-2">
        <span className="font-mono text-2xl font-semibold tracking-tight tabular-nums text-[var(--foreground)]">
          {currentWeek.avgHeartRate}
        </span>
        <span className="text-xs text-[var(--muted)]">bpm avg</span>
        {delta !== null && (
          <span className="font-mono text-sm tabular-nums text-[var(--muted)]">
            {signed(delta)} vs 4-wk
          </span>
        )}
      </div>

      {/* One dot per HR-bearing run on a bpm axis; a single run sits centered. */}
      <div
        className="relative mt-2 h-5 border-b border-[var(--border)]"
        role="img"
        aria-label="This week's runs by average heart rate"
      >
        {hrRuns.map((r) => {
          const bpm = r.avgHeartRate as number;
          const left = hrRuns.length === 1 ? 50 : 8 + ((bpm - minBpm) / bpmSpan) * 84;
          const size =
            DOT_MIN_PX + Math.round((r.durationSeconds / maxDur) * (DOT_MAX_PX - DOT_MIN_PX));
          return (
            <span
              key={r.activityId}
              data-testid="effort-dot"
              className="absolute bottom-0 translate-x-[-50%] rounded-full"
              style={{
                left: `${left}%`,
                width: `${size}px`,
                height: `${size}px`,
                backgroundColor: zoneToken(r.heartRateZone),
              }}
            />
          );
        })}
      </div>

      <p className="text-[11px] text-[var(--faint)]">
        {modalZone !== null && <>mostly zone {modalZone} runs · </>}
        {currentWeek.heartRateRuns < currentWeek.runCount
          ? coverageCaption(currentWeek.heartRateRuns, currentWeek.runCount)
          : `${currentWeek.runCount} ${currentWeek.runCount === 1 ? "run" : "runs"}`}
      </p>
    </MiniCard>
  );
}

/** Most frequent value; ties resolve to the highest zone seen among the tied. */
function mode(xs: number[]): number {
  const counts = new Map<number, number>();
  for (const x of xs) counts.set(x, (counts.get(x) ?? 0) + 1);
  let best = xs[0];
  let bestCount = 0;
  for (const [value, count] of counts) {
    if (count > bestCount || (count === bestCount && value > best)) {
      best = value;
      bestCount = count;
    }
  }
  return best;
}
```

- [ ] **Step 4: Run to verify it passes**

Run: `npm run test -- "app/(app)/dashboard/_components/running/effort-heart.test.tsx" && npm run typecheck`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add "app/(app)/dashboard/_components/running/effort-heart.tsx" "app/(app)/dashboard/_components/running/effort-heart.test.tsx"
git commit -m "feat(dashboard): run effort tile (effort-heart)"
```

### Task 9: `vertical-gain.tsx` — Vertical Gain (id `running_vertical`)

**Files:**
- Create: `app/(app)/dashboard/_components/running/vertical-gain.tsx`
- Test: `app/(app)/dashboard/_components/running/vertical-gain.test.tsx`

**Binding:** sage fill; **hike clay (`#b08e77` / `--discipline-hike-*`) is forbidden** — the single most likely color mistake. A large figure with a small unit suffix over a full-width stepped silhouette, one column per run, height by gain, scaled to the week's max gain with a floor so a small climb is still a visible step. A nil-gain run contributes a **hairline outline gap, not a zero column**; the caption says `3 of 4 runs`. The indoor-only week renders the stated sentence "no outdoor runs this week" — never an empty chart frame, never `0 ft`. Biggest climb called out (`most: 702 ft · Sat`).

- [ ] **Step 1: Write the failing tests**

`vertical-gain.test.tsx`:

```tsx
/// <reference types="vitest/globals" />

import { render, screen } from "@testing-library/react";
import { VerticalGainCard } from "./vertical-gain";
import { firstRunEver, indoorOnly, ordinaryWeek, zeroRuns } from "./fixtures";

const UNITS = ["mi", "km"] as const;

describe("VerticalGainCard", () => {
  it.each(UNITS)("heroes the week's climb with the biggest climb called out (%s)", (unit) => {
    render(<VerticalGainCard section={ordinaryWeek(unit)} href="/x" />);
    // The hero figure and its unit suffix render as separate spans.
    expect(screen.getByText(unit === "mi" ? "899" : "274")).toBeInTheDocument();
    const most = unit === "mi" ? "702 ft" : "214 m";
    expect(screen.getByText(new RegExp(`most: ${most} · Sat`))).toBeInTheDocument();
  });

  it("renders a gap outline — not a zero column — for the treadmill run", () => {
    render(<VerticalGainCard section={ordinaryWeek("mi")} href="/x" />);
    expect(screen.getAllByTestId("gain-column")).toHaveLength(3);
    expect(screen.getAllByTestId("gain-gap")).toHaveLength(1);
    // Coverage is stated.
    expect(screen.getByText(/3 of 4 runs/)).toBeInTheDocument();
  });

  it("fills columns with sage, never hike clay", () => {
    render(<VerticalGainCard section={ordinaryWeek("mi")} href="/x" />);
    for (const col of screen.getAllByTestId("gain-column")) {
      expect(col).toHaveStyle({ backgroundColor: "var(--discipline-run-dot)" });
    }
  });

  it.each(UNITS)("states the indoor-only week in words, not an empty chart (%s)", (unit) => {
    const { container } = render(<VerticalGainCard section={indoorOnly(unit)} href="/x" />);
    expect(screen.getByText(/no outdoor runs this week/)).toBeInTheDocument();
    expect(container.textContent).not.toMatch(/0 ft|0 m/);
    expect(screen.queryAllByTestId("gain-column")).toHaveLength(0);
  });

  it.each(UNITS)("says something true on a zero-run week (%s)", (unit) => {
    const { container } = render(<VerticalGainCard section={zeroRuns(unit)} href="/x" />);
    expect(screen.getByText(/no runs this week/)).toBeInTheDocument();
    expect(container.textContent).not.toBe("—");
  });

  it("keeps a lone small climb a visible step (floor, not a sliver)", () => {
    render(<VerticalGainCard section={firstRunEver("mi")} href="/x" />);
    const cols = screen.getAllByTestId("gain-column");
    expect(cols).toHaveLength(1);
  });
});
```

- [ ] **Step 2: Run to verify it fails**

Run: `npm run test -- "app/(app)/dashboard/_components/running/vertical-gain.test.tsx"`
Expected: FAIL

- [ ] **Step 3: Implement**

`vertical-gain.tsx`:

```tsx
/**
 * VerticalGainCard — the `running_vertical` tile ("Vertical Gain").
 *
 * Heroes CLIMBING: total gain as a large figure with a small unit suffix
 * over a full-width stepped silhouette — one column per outdoor run, height
 * by that run's gain — with the week's biggest climb called out
 * ("most: 702 ft · Sat"). The most horizontal composition of the four, so it
 * pairs cleanly beneath another tile.
 *
 * Color: SAGE fill (--discipline-run-dot). Hike clay is explicitly forbidden
 * — elevation is clay on the Hiking surfaces because activity type owns
 * activity color; a running tile drawing its climbing in clay would read as
 * a hike on the grid.
 *
 * Honesty: a nil-gain run (treadmill) contributes a hairline OUTLINE GAP,
 * never a zero column — nil elevation is not zero elevation — and the
 * caption states coverage ("3 of 4 runs"). The indoor-only week renders the
 * stated sentence "no outdoor runs this week", never an empty chart frame or
 * a fabricated "0 ft". Columns scale to the week's max gain with a floor so
 * a small climb is still a visible step beside a 700 ft one.
 */

import type { RunningView } from "@/lib/dashboard";
import { MiniCard } from "../mini-card";
import { coverageCaption, weekdayLabel } from "./shared";

const TITLE = "Vertical Gain";
// Floor (% of rail height) so a 20 ft run is a visible step, not a sliver.
const COLUMN_FLOOR_PCT = 14;

export function VerticalGainCard({ section, href }: { section: RunningView; href: string }) {
  const { currentWeek, weekRuns } = section;

  if (currentWeek.runCount === 0) {
    return (
      <MiniCard title={TITLE} href={href}>
        <p className="text-sm text-[var(--muted)]">no runs this week</p>
      </MiniCard>
    );
  }

  const gainRuns = weekRuns.filter((r) => r.elevationGainMeters !== null);
  if (currentWeek.elevation === null || gainRuns.length === 0) {
    return (
      <MiniCard title={TITLE} href={href}>
        <p className="text-sm text-[var(--muted)]">no outdoor runs this week</p>
        <p className="text-[11px] text-[var(--faint)]">
          {currentWeek.runCount} {currentWeek.runCount === 1 ? "run" : "runs"}, all without altitude
          data
        </p>
      </MiniCard>
    );
  }

  const [value, suffix] = splitUnit(currentWeek.elevation);
  const maxGain = Math.max(...gainRuns.map((r) => r.elevationGainMeters as number), 1);
  const biggest = gainRuns.reduce((a, b) =>
    (b.elevationGainMeters as number) > (a.elevationGainMeters as number) ? b : a,
  );

  return (
    <MiniCard title={TITLE} href={href}>
      <div className="flex items-baseline gap-1">
        <span className="font-mono text-3xl font-semibold tracking-tight tabular-nums text-[var(--foreground)]">
          {value}
        </span>
        <span className="text-sm text-[var(--muted)]">{suffix}</span>
      </div>

      {/* The stepped silhouette: gain-bearing runs are sage columns scaled to
          the week's max; nil-gain runs are hairline outline gaps. */}
      <div
        className="flex h-9 items-end gap-[3px]"
        role="img"
        aria-label="Elevation gained per run this week"
      >
        {weekRuns.map((r) =>
          r.elevationGainMeters === null ? (
            <span
              key={r.activityId}
              data-testid="gain-gap"
              aria-label="indoor run, no altitude data"
              className="h-2 flex-1 rounded-[1px] border border-[var(--border)]"
            />
          ) : (
            <span
              key={r.activityId}
              data-testid="gain-column"
              className="flex-1 rounded-[1px]"
              style={{
                height: `${Math.max(COLUMN_FLOOR_PCT, Math.round((r.elevationGainMeters / maxGain) * 100))}%`,
                backgroundColor: "var(--discipline-run-dot)",
              }}
            />
          ),
        )}
      </div>

      <p className="text-[11px] text-[var(--faint)]">
        most: {biggest.elevation} · {weekdayLabel(biggest.localDate)}
        {currentWeek.elevationRuns < currentWeek.runCount && (
          <> · {coverageCaption(currentWeek.elevationRuns, currentWeek.runCount)}</>
        )}
      </p>
    </MiniCard>
  );
}

/** "899 ft" → ["899", "ft"] so the unit renders as a small suffix. */
function splitUnit(display: string): [string, string] {
  const i = display.lastIndexOf(" ");
  if (i < 0) return [display, ""];
  return [display.slice(0, i), display.slice(i + 1)];
}
```

- [ ] **Step 4: Run to verify it passes**

Run: `npm run test -- "app/(app)/dashboard/_components/running/vertical-gain.test.tsx" && npm run typecheck`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add "app/(app)/dashboard/_components/running/vertical-gain.tsx" "app/(app)/dashboard/_components/running/vertical-gain.test.tsx"
git commit -m "feat(dashboard): vertical gain tile (vertical-gain)"
```

### Task 10: Catalog mirror + `TileCard` rewire (one commit — the exhaustive switch demands it)

**Files:**
- Modify: `lib/dashboard-tiles.ts` (3 new ids + entries; rewrite the `running` entry)
- Modify: `app/(app)/dashboard/_components/tile-renderer.tsx` (delete `RunningCard`, four cases)
- Test: `lib/dashboard-tiles.test.ts`, `app/(app)/dashboard/_components/tile-renderer.test.tsx`

- [ ] **Step 1: Update the catalog tests (failing first)**

In `lib/dashboard-tiles.test.ts`:
- `expect(TILE_CATALOG.length).toBe(15)` → `toBe(18)` (test name "has exactly 15 tiles" → "has exactly 18 tiles").
- The id-order array gains `"running_log", "running_effort", "running_vertical"` immediately after `"running"` (mirror the Go `Catalog` byte-for-byte).
- The `ALL_TILE_IDS: Record<TileId, true>` map gains the three ids.
- Add title assertions:

```ts
  test("the running entry was rewritten for the ramp card", () => {
    expect(tileEntry("running").title).toBe("Training Load");
    expect(tileEntry("running").href).toBe("/activities?view=running");
    expect(tileEntry("running_log").title).toBe("Runs This Week");
    expect(tileEntry("running_effort").title).toBe("Run Effort");
    expect(tileEntry("running_vertical").title).toBe("Vertical Gain");
    // The whole family deep-links into the running view.
    for (const id of ["running_log", "running_effort", "running_vertical"] as const) {
      expect(tileEntry(id).href).toBe("/activities?view=running");
    }
  });
```

- [ ] **Step 2: Update the renderer tests**

In `app/(app)/dashboard/_components/tile-renderer.test.tsx`:
- The first test ("renders the running card for id 'running'") now expects the heading `Training Load` (empty state via `RunningEmptyCard`).
- Add family coverage (import `ordinaryWeek` from `./running/fixtures`):

```tsx
  const RUNNING_FAMILY: [TileId, string][] = [
    ["running", "Training Load"],
    ["running_log", "Runs This Week"],
    ["running_effort", "Run Effort"],
    ["running_vertical", "Vertical Gain"],
  ];

  it.each(RUNNING_FAMILY)("renders the %s card from the shared running section", (id, title) => {
    render(
      <TileCard id={id} data={fixture({ running: { present: true, ...ordinaryWeek("mi") } })} />,
    );
    expect(screen.getByRole("heading", { name: title })).toBeInTheDocument();
    expect(screen.queryByText("Import a run to start tracking")).not.toBeInTheDocument();
  });

  it.each(RUNNING_FAMILY)("renders the titled empty CTA for %s when running is absent", (id, title) => {
    render(<TileCard id={id} data={fixture()} />);
    expect(screen.getByRole("heading", { name: title })).toBeInTheDocument();
    expect(screen.getByText("Import a run to start tracking")).toBeInTheDocument();
  });
```

- [ ] **Step 3: Run to verify they fail**

Run: `npm run test -- lib/dashboard-tiles.test.ts "app/(app)/dashboard/_components/tile-renderer.test.tsx"`
Expected: FAIL

- [ ] **Step 4: Implement the catalog entries**

In `lib/dashboard-tiles.ts`, extend the `TileId` union after `"running"`:

```ts
export type TileId =
  | "running"
  | "running_log"
  | "running_effort"
  | "running_vertical"
  | "walking"
  ...
```

and replace the `running` catalog entry + insert the three new ones directly after it (SOW-proposed copy, verbatim):

```ts
  {
    id: "running",
    title: "Training Load",
    href: "/activities?view=running",
    description: "Whether you're building or backing off, against your 4-week normal.",
  },
  {
    id: "running_log",
    title: "Runs This Week",
    href: "/activities?view=running",
    description: "Every run this week as a dated row.",
  },
  {
    id: "running_effort",
    title: "Run Effort",
    href: "/activities?view=running",
    description: "How hard your runs were, by average heart rate.",
  },
  {
    id: "running_vertical",
    title: "Vertical Gain",
    href: "/activities?view=running",
    description: "How much climbing was in this week's runs.",
  },
```

- [ ] **Step 5: Rewire `TileCard` and delete `RunningCard`**

In `tile-renderer.tsx`:
- Delete the `RunningCard` component (including the Task 4 bridge) and its now-unused imports if any (`BigNum`/`Spark`/`MetaRow` stay — other cards use them). Update the file-header comment's mention of the six original cards accordingly (running now lives under `running/`).
- Add imports:

```tsx
import { LoadRampCard } from "./running/load-ramp";
import { WeekLogCard } from "./running/week-log";
import { EffortHeartCard } from "./running/effort-heart";
import { VerticalGainCard } from "./running/vertical-gain";
import { RunningEmptyCard } from "./running/empty-card";
```

- Replace the `case "running":` and add the three new cases (mirroring the recovery-family pattern; the `never` default keeps compile-time exhaustiveness):

```tsx
    case "running":
      return data.running.present ? (
        <LoadRampCard section={data.running} href={href} />
      ) : (
        <RunningEmptyCard title="Training Load" href={href} />
      );
    case "running_log":
      return data.running.present ? (
        <WeekLogCard section={data.running} href={href} />
      ) : (
        <RunningEmptyCard title="Runs This Week" href={href} />
      );
    case "running_effort":
      return data.running.present ? (
        <EffortHeartCard section={data.running} href={href} />
      ) : (
        <RunningEmptyCard title="Run Effort" href={href} />
      );
    case "running_vertical":
      return data.running.present ? (
        <VerticalGainCard section={data.running} href={href} />
      ) : (
        <RunningEmptyCard title="Vertical Gain" href={href} />
      );
```

- [ ] **Step 6: Full gate**

Run: `npm run typecheck && npm run lint && npm run test && npm run build`
Expected: PASS. If any other test in the repo referenced the old `Running` title or `RunningCard` export, fix it to the new grammar (search: `grep -rn "RunningCard\|\"Running\"" app/ lib/ --include="*.test.*"`).

- [ ] **Step 7: Commit**

```bash
git add lib/dashboard-tiles.ts lib/dashboard-tiles.test.ts "app/(app)/dashboard/_components/tile-renderer.tsx" "app/(app)/dashboard/_components/tile-renderer.test.tsx"
git commit -m "feat(dashboard): running family catalog entries and tile renderer rewire"
```

---

## Repo 3: prog-strength-docs (`/workspace/prog-strength-docs`)

### Task 11: Flip the DX to selected and the SOW to shipped

**Files:**
- Modify: `dx/running-tile.md`
- Modify: `sows/running-tile-family.md`

- [ ] **Step 1: Create the branch** (if not already created for this plan file)

```bash
cd /workspace/prog-strength-docs && git checkout feat/running-tile-family 2>/dev/null || git checkout -b feat/running-tile-family
```

- [ ] **Step 2: Flip the DX**

In `dx/running-tile.md`: frontmatter `status: draft` → `status: selected`; body line `**Status**: Draft · **Last updated**: 2026-08-02` → `**Status**: Selected · **Last updated**: 2026-08-03`, and directly below it add:

```markdown
> **Selection** (2026-08-03): `load-ramp` is the **default** — it inherits the
> `running` id and its rendering. `week-log`, `effort-heart`, and
> `vertical-gain` ship as opt-in catalog tiles (`running_log`,
> `running_effort`, `running_vertical`). `stacked-week` and `pace-band` are
> **dropped**. Implemented by
> [`sows/running-tile-family.md`](../sows/running-tile-family.md).
```

Commit:

```bash
git add dx/running-tile.md
git commit -m "docs: flip dx/running-tile to selected"
```

- [ ] **Step 3: Flip the SOW**

In `sows/running-tile-family.md`: frontmatter `status: draft` → `status: shipped`; body `**Status**: Draft · **Last updated**: 2026-08-02` → `**Status**: Shipped · **Last updated**: 2026-08-03`.

Commit:

```bash
git add sows/running-tile-family.md
git commit -m "docs: mark running-tile-family as shipped"
```

---

## Final verification (before any push)

Per repo, the full CI-mirroring gate must be green locally:

**prog-strength-api** (`/workspace/prog-strength-api`):

```bash
go build ./... && go vet ./...
go run github.com/golangci/golangci-lint/v2/cmd/golangci-lint@v2.12.2 run --timeout=5m
go mod tidy && git diff --exit-code go.mod go.sum
go test ./...
```

**prog-strength-web** (`/workspace/prog-strength-web`):

```bash
npm run typecheck && npm run lint && npm run test && npm run format:check && npm run build
```

(`npm run format` first if format:check fails, then re-stage.)

Fix anything that fails — never with `//nolint`, rule disables, or skipped tests. Then push each `feat/running-tile-family` branch and open PRs (docs PR body follows the operator template; API/web PR bodies follow each repo's recent merged-PR format; PR titles are conventional commits with lowercase subjects).
