# Recovery Baseline Drift Payload Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Give every charted day in the dashboard recovery payload its own trailing HRV baseline, and add a `baseline_trend` block reporting the baseline against its own past — without moving a single existing scalar figure.

**Architecture:** `internal/recoverytrend` grows a second pure entry point, `ComputeSeries`, beside `Compute`. Both share extracted `band`/`classify` helpers so they can never disagree. The dashboard read path widens its single indexed `ListRange` from 31 to 61 local dates so the *oldest* charted day still has a full 30-day trailing sample, materialises that wide window once, feeds the whole thing to `ComputeSeries` and its last 31 dates to `Compute`. The wire change is purely additive: `days[]` entries gain five keys via an embedded struct, and the section gains one always-present `baseline_trend` object. `prog-strength-web` mirrors the types through its adapter as pass-through server figures and renders none of them yet — plus one two-line rounding fix on `HrvBalanceCard`.

**Tech Stack:** Go 1.25 (chi, `go-toml`, stdlib `testing`), Next.js 16 / React 19 / TypeScript / Vitest.

**Source SOW:** [`sows/recovery-baseline-drift-payload.md`](../sows/recovery-baseline-drift-payload.md)

---

## File Structure

### `prog-strength-api`

| File | Responsibility | Change |
| --- | --- | --- |
| `internal/recoverytrend/recoverytrend.go` | The pure engine | Extract `band`/`classify`; add `Config.BaselineDriftDays`/`BaselineDriftZ`, `DayResult`, `BaselineTrend`, `ComputeSeries`, `drift` |
| `internal/recoverytrend/doc.go` | Package narrative | New `# The series and the drift` section |
| `internal/recoverytrend/recoverytrend_test.go` | Engine unit tests | Nine new tests; **no existing assertion edited** |
| `config.toml` | Non-secret tunables | `baseline_drift_days` / `baseline_drift_z` in `[recovery]` |
| `internal/config/config.go` | Config load + defaults | Two fields in three places |
| `internal/config/config_test.go` | Default assertions | Two new assertions in each of two blocks |
| `internal/server/server.go` | Engine construction | Two fields threaded into `recoverytrend.New` |
| `internal/dashboard/dto.go` | Wire types | `RecoveryDayPoint`, `RecoveryBaselineTrend`; `RecoverySection.Days` re-typed, `BaselineTrend` added |
| `internal/dashboard/handler.go` | Read path | `sinceStr` widens to `-2*win` |
| `internal/dashboard/whoop.go` | Section builder | One-pass wide-window materialisation; two engine calls; per-day band zip |
| `internal/dashboard/whoop_test.go` | Builder tests | `testRecoveryEngine` gains two fields; four new tests; JSON-key test extended |
| `internal/activity/contract_test.go` | Cross-package contract | Engine literal gains two fields |

### `prog-strength-web`

| File | Responsibility | Change |
| --- | --- | --- |
| `lib/api.ts` | Raw wire types | `DashboardRecoveryDayPoint`, `DashboardRecoveryBaselineTrend`; `DashboardRecovery.days` re-typed, `baseline_trend` added |
| `lib/dashboard.ts` | View-model adapter | `RecoveryDayPoint` gains five fields; `RecoveryBaselineTrendView` new; `RecoveryView.baselineTrend?`; `adaptRecovery` passes through |
| `lib/dashboard.test.ts` | Adapter tests | Fixture builder gains `baseline_trend`; two new tests |
| `app/(app)/dashboard/page.test.tsx` | Page smoke test | Recovery payload fixture gains `baseline_trend` |
| `app/(app)/dashboard/_components/recovery/fixtures.ts` | Tile fixtures | `makeDays` emits the five new fields |
| `app/(app)/dashboard/_components/recovery/balance-band.tsx` | The `hrv_balance` tile | `{todayVal}` → `{Math.round(todayVal)}` |
| `app/(app)/dashboard/_components/recovery/balance-band.test.tsx` | Tile tests | One new rounding test |

### Design system

No new visual tokens. The only frontend change is `Math.round` on an already-styled number; `/workspace/prog-strength-docs/design-system.md` needs nothing from this work.

---

## Local check gate (run before every commit / push)

**`prog-strength-api`** (from the repo root):

```bash
go build ./...
go vet ./...
go mod tidy && git diff --exit-code go.mod go.sum
go run github.com/golangci/golangci-lint/v2/cmd/golangci-lint@v2.12.2 run
go run github.com/golangci/golangci-lint/v2/cmd/golangci-lint@v2.12.2 fmt --diff
go test ./...
```

`v2.12.2` is the version pinned in `.github/workflows/ci.yml`. Never pass `--no-verify`, never add a `//nolint` directive, never silence a `gosec` finding, never skip a test.

**`prog-strength-web`**:

```bash
npm run lint
npm run typecheck
npm run format:check
npm run test
```

---

## Task 1: Extract `band` and `classify` in `recoverytrend`

Pure refactor. `Compute`'s output must not move by a bit. The guard is that **every existing test in `internal/recoverytrend/recoverytrend_test.go` passes unmodified** — do not edit a single existing assertion.

**Files:**
- Modify: `internal/recoverytrend/recoverytrend.go:117-139`

- [ ] **Step 1: Run the existing tests and record the baseline**

Run: `cd /workspace/prog-strength-api && go test ./internal/recoverytrend/ -v 2>&1 | tail -40`
Expected: all PASS. This is the regression guard for the whole task.

- [ ] **Step 2: Add the two helpers**

Insert immediately after the `Compute` method (before the `collect` helper) in `internal/recoverytrend/recoverytrend.go`:

```go
// band returns the balanced bounds and the effective (floored) SD for a
// baseline mean and spread. The floored SD is returned because the caller
// needs the SAME divisor for the z-score — a band and a z computed from
// different divisors could disagree about which side of the bound a day is on.
func (e *Engine) band(avg, sd float64) (low, high, sdEff float64) {
	sdEff = math.Max(sd, e.cfg.MinStdDevMs)
	return avg - e.cfg.BalancedZ*sdEff, avg + e.cfg.BalancedZ*sdEff, sdEff
}

// classify maps a z-score to a status. Inclusive at the boundary: |z| within
// BalancedZ reads balanced, above reads elevated, below reads suppressed.
func (e *Engine) classify(z float64) string {
	switch {
	case math.Abs(z) <= e.cfg.BalancedZ:
		return StatusBalanced
	case z > e.cfg.BalancedZ:
		return StatusElevated
	default:
		return StatusSuppressed
	}
}
```

- [ ] **Step 3: Rewrite `Compute`'s band/status block to call them**

Replace lines 117–139 of `internal/recoverytrend/recoverytrend.go` — currently:

```go
	avg := mean(hrv)
	b.HRVAvg = &avg
	sd := stdDevPop(hrv, avg)
	b.HRVStdDev = &sd
	sdEff := math.Max(sd, e.cfg.MinStdDevMs)

	low := avg - e.cfg.BalancedZ*sdEff
	high := avg + e.cfg.BalancedZ*sdEff
	h.BalancedLow = &low
	h.BalancedHigh = &high

	if today.HRV != nil {
		z := (*today.HRV - avg) / sdEff
		h.ZScore = &z
		switch {
		case math.Abs(z) <= e.cfg.BalancedZ:
			h.Status = StatusBalanced
		case z > e.cfg.BalancedZ:
			h.Status = StatusElevated
		default:
			h.Status = StatusSuppressed
		}
	}
```

with:

```go
	avg := mean(hrv)
	b.HRVAvg = &avg
	sd := stdDevPop(hrv, avg)
	b.HRVStdDev = &sd
	low, high, sdEff := e.band(avg, sd)
	h.BalancedLow = &low
	h.BalancedHigh = &high

	if today.HRV != nil {
		z := (*today.HRV - avg) / sdEff
		h.ZScore = &z
		h.Status = e.classify(z)
	}
```

Everything below (the `trendHRV` block, which still uses `sdEff`) stays exactly as it is.

- [ ] **Step 4: Verify the refactor changed nothing**

Run: `cd /workspace/prog-strength-api && go test ./internal/recoverytrend/ ./internal/dashboard/ && go vet ./internal/recoverytrend/`
Expected: PASS, with `git diff internal/recoverytrend/recoverytrend_test.go` empty.

- [ ] **Step 5: Run the lint gate**

Run:
```bash
cd /workspace/prog-strength-api
go run github.com/golangci/golangci-lint/v2/cmd/golangci-lint@v2.12.2 run ./internal/recoverytrend/
```
Expected: `0 issues.`

- [ ] **Step 6: Commit**

```bash
cd /workspace/prog-strength-api
git add internal/recoverytrend/recoverytrend.go
git commit -m "refactor(recoverytrend): extract band and classify helpers"
```

---

## Task 2: Thread `baseline_drift_days` / `baseline_drift_z` through config

Adds the two tunables everywhere they are read, with the shipped defaults `28` and `0.35`. Nothing consumes them yet — Task 3 does.

**Files:**
- Modify: `internal/recoverytrend/recoverytrend.go:7-15` (the `Config` struct)
- Modify: `config.toml:376-403` (the `[recovery]` block)
- Modify: `internal/config/config.go:258-276`, `:465-473`, `:666-674`
- Modify: `internal/server/server.go:823-831`
- Modify: `internal/config/config_test.go:~618`, `:~745`
- Modify: `internal/recoverytrend/recoverytrend_test.go:9-20` (`defaultCfg`)
- Modify: `internal/dashboard/whoop_test.go:17-27` (`testRecoveryEngine`)
- Modify: `internal/activity/contract_test.go:161-164`

- [ ] **Step 1: Add the fields to `recoverytrend.Config`**

In `internal/recoverytrend/recoverytrend.go`, append to the `Config` struct after `MinStdDevMs`:

```go
	BaselineDriftDays int     // how far back the baseline is compared against
	BaselineDriftZ    float64 // |delta| must exceed this many SDs to read rising/falling
```

- [ ] **Step 2: Add the config-loader assertions FIRST (they must fail)**

In `internal/config/config_test.go`, find the two existing recovery default blocks (one asserting a `RecoveryConfig` literal around line 618, one asserting `cfg.Recovery.*` around line 733). Add to the struct literal, after `MinStdDevMs: 1.0,`:

```go
			BaselineDriftDays:  28,
			BaselineDriftZ:     0.35,
```

and after the `MinStdDevMs` assertion block:

```go
	if cfg.Recovery.BaselineDriftDays != 28 {
		t.Errorf("BaselineDriftDays = %d, want 28", cfg.Recovery.BaselineDriftDays)
	}
	if cfg.Recovery.BaselineDriftZ != 0.35 {
		t.Errorf("BaselineDriftZ = %v, want 0.35", cfg.Recovery.BaselineDriftZ)
	}
```

Match the surrounding gofmt alignment exactly — the struct literal's field names are column-aligned, so adding a longer name may re-align the whole block. `golangci-lint fmt` decides; run it.

- [ ] **Step 3: Run the config tests to verify they fail**

Run: `cd /workspace/prog-strength-api && go test ./internal/config/ 2>&1 | tail -20`
Expected: FAIL — `unknown field BaselineDriftDays in struct literal` (compile error), which counts as red.

- [ ] **Step 4: Add the two knobs to `config.toml`**

In `config.toml`, inside the `[recovery]` block: append the following comment lines directly after the existing `# min_std_dev_ms: …` comment paragraph, and the two assignments directly after `min_std_dev_ms       = 1.0`:

```toml
# baseline_drift_days: how far back today's baseline is compared against to
#   decide whether the athlete's normal range is rising or falling. 28 gives a
#   four-week read, matching the window a training block is judged over. MUST be
#   strictly less than baseline_window_days — the series only reaches back that
#   far, and a larger value yields direction = "unknown" (honest, but useless).
# baseline_drift_z: how far the baseline must have moved, in the athlete's OWN
#   current standard deviations, to read rising or falling rather than steady.
#   Below trend_z because a baseline shift is a slower, more considered signal
#   than a week's mean moving: by the time a 30-day average has shifted a third
#   of an SD, something real has changed.
baseline_drift_days  = 28
baseline_drift_z     = 0.35
```

Keep the existing `=` column alignment of the block — the existing keys align at column 22 (`baseline_window_days = 30`), so `baseline_drift_days` and `baseline_drift_z` pad to the same column.

- [ ] **Step 5: Thread the fields through `internal/config/config.go`**

Three edits.

(a) In the `RecoveryConfig` struct (around line 268), after `MinStdDevMs        float64`:

```go
	BaselineDriftDays  int
	BaselineDriftZ     float64
```

Also extend the doc comment above `RecoveryConfig`, appending to its final sentence:

```go
// balanced-band half-width in the user's own SDs; TrendZ is the rising/falling
// threshold; MinStdDevMs floors the SD divisor. BaselineDriftDays/BaselineDriftZ
// bound the rolling-baseline drift read: how far back the baseline is compared
// against, and how far it must have moved (in the user's own current SDs) to
// read rising or falling. BaselineDriftDays MUST be strictly less than
// BaselineWindowDays or the drift is permanently "unknown".
```

(b) In the `toml:"recovery"` file struct (around line 465), after `MinStdDevMs        float64 \`toml:"min_std_dev_ms"\``:

```go
		BaselineDriftDays  int     `toml:"baseline_drift_days"`
		BaselineDriftZ     float64 `toml:"baseline_drift_z"`
```

(c) In the `Recovery: RecoveryConfig{…}` mapping (around line 666), after `MinStdDevMs:        fc.Recovery.MinStdDevMs,`:

```go
			BaselineDriftDays:  fc.Recovery.BaselineDriftDays,
			BaselineDriftZ:     fc.Recovery.BaselineDriftZ,
```

- [ ] **Step 6: Thread the fields into the engine construction**

In `internal/server/server.go`, inside the `recoverytrend.New(recoverytrend.Config{…})` literal (around line 823), after `MinStdDevMs:        cfg.Recovery.MinStdDevMs,`:

```go
			BaselineDriftDays:  cfg.Recovery.BaselineDriftDays,
			BaselineDriftZ:     cfg.Recovery.BaselineDriftZ,
```

- [ ] **Step 7: Update the three test engine literals so they keep matching production defaults**

In `internal/recoverytrend/recoverytrend_test.go`, `defaultCfg()` — after `MinStdDevMs:        1.0,`:

```go
		BaselineDriftDays:  28,
		BaselineDriftZ:     0.35,
```

In `internal/dashboard/whoop_test.go`, `testRecoveryEngine()` — after `MinStdDevMs:        1.0,`:

```go
		BaselineDriftDays:  28,
		BaselineDriftZ:     0.35,
```

In `internal/activity/contract_test.go` (around line 161), change:

```go
		recoverytrend.New(recoverytrend.Config{
			BaselineWindowDays: 30, MinBaselineDays: 14, TrendWindowDays: 7,
			MinTrendDays: 4, BalancedZ: 1.0, TrendZ: 0.5, MinStdDevMs: 1.0,
		}))
```

to:

```go
		recoverytrend.New(recoverytrend.Config{
			BaselineWindowDays: 30, MinBaselineDays: 14, TrendWindowDays: 7,
			MinTrendDays: 4, BalancedZ: 1.0, TrendZ: 0.5, MinStdDevMs: 1.0,
			BaselineDriftDays: 28, BaselineDriftZ: 0.35,
		}))
```

- [ ] **Step 8: Run the full gate**

Run:
```bash
cd /workspace/prog-strength-api
go build ./... && go vet ./... && go test ./...
go run github.com/golangci/golangci-lint/v2/cmd/golangci-lint@v2.12.2 fmt --diff
go run github.com/golangci/golangci-lint/v2/cmd/golangci-lint@v2.12.2 run
```
Expected: all tests PASS (`internal/config` in particular), `fmt --diff` prints nothing, `run` reports `0 issues.`

- [ ] **Step 9: Commit**

```bash
cd /workspace/prog-strength-api
git add config.toml internal/config internal/server internal/recoverytrend internal/dashboard/whoop_test.go internal/activity/contract_test.go
git commit -m "feat(config): add baseline_drift_days and baseline_drift_z recovery tunables"
```

---

## Task 3: `ComputeSeries`, `DayResult`, `BaselineTrend`, `drift`

The heart of the SOW. TDD: all nine tests first, then the implementation.

**Files:**
- Modify: `internal/recoverytrend/recoverytrend.go`
- Modify: `internal/recoverytrend/recoverytrend_test.go`

### Test helpers this task needs

The file already has `defaultCfg()`, `p(f float64) *float64`, `approx(a *float64, want float64) bool`, `window(sampleHRV []*float64, todayHRV *float64) []Day`, and `flat(n int, v float64) []*float64`. `window` builds `len(sample)+1` days — reuse it where the shape fits, and add **one** new helper for the wide input:

```go
// wide builds a date-ascending window of len(hrv) days named d000, d001, …, so
// a test can assert on the exact dates ComputeSeries echoes back. Resting HR
// and recovery score are left nil — ComputeSeries only reads HRV.
func wide(hrv []*float64) []Day {
	days := make([]Day, len(hrv))
	for i, v := range hrv {
		days[i] = Day{Date: fmt.Sprintf("d%03d", i), HRV: v}
	}
	return days
}

// ramp returns n HRV pointers rising linearly from start by step per day.
func ramp(n int, start, step float64) []*float64 {
	out := make([]*float64, n)
	for i := range out {
		out[i] = p(start + step*float64(i))
	}
	return out
}
```

`wide` needs `"fmt"` added to the test file's import block.

- [ ] **Step 1: Write the failing tests**

Append to `internal/recoverytrend/recoverytrend_test.go` (after adding the helpers above and the `fmt` import):

```go
func TestComputeSeries_LengthAndLeadIn(t *testing.T) {
	e := New(defaultCfg())
	days := wide(flat(61, 80))
	series, _ := e.ComputeSeries(days)
	if len(series) != 31 {
		t.Fatalf("len(series) = %d, want 31", len(series))
	}
	for i, r := range series {
		if want := days[30+i].Date; r.Date != want {
			t.Errorf("series[%d].Date = %q, want %q", i, r.Date, want)
		}
	}
}

func TestComputeSeries_PerDayBaselineExcludesTheDayItself(t *testing.T) {
	e := New(defaultCfg())
	hrv := flat(61, 80)
	// One large outlier at charted index 10 (input index 40).
	hrv[40] = p(200)
	series, _ := e.ComputeSeries(wide(hrv))

	// The outlier's OWN baseline is the 30 flat days behind it — untouched.
	if !approx(series[10].BaselineAvg, 80) {
		t.Errorf("outlier day's own baseline = %v, want 80 (day excluded from its own sample)", series[10].BaselineAvg)
	}
	// The NEXT day's baseline includes it: (29*80 + 200)/30 = 84.
	if !approx(series[11].BaselineAvg, 84) {
		t.Errorf("next day's baseline = %v, want 84 (outlier now in sample)", series[11].BaselineAvg)
	}
}

func TestComputeSeries_AgreesWithComputeOnToday(t *testing.T) {
	e := New(defaultCfg())
	days := wide(ramp(61, 70, 0.5))
	series, _ := e.ComputeSeries(days)
	baseline, hrv := e.Compute(days[30:])

	last := series[len(series)-1]
	if last.BaselineAvg == nil || baseline.HRVAvg == nil || *last.BaselineAvg != *baseline.HRVAvg {
		t.Errorf("BaselineAvg = %v, want scalar HRVAvg %v (exact)", last.BaselineAvg, baseline.HRVAvg)
	}
	if last.BalancedLow == nil || hrv.BalancedLow == nil || *last.BalancedLow != *hrv.BalancedLow {
		t.Errorf("BalancedLow = %v, want %v (exact)", last.BalancedLow, hrv.BalancedLow)
	}
	if last.BalancedHigh == nil || hrv.BalancedHigh == nil || *last.BalancedHigh != *hrv.BalancedHigh {
		t.Errorf("BalancedHigh = %v, want %v (exact)", last.BalancedHigh, hrv.BalancedHigh)
	}
	if last.ZScore == nil || hrv.ZScore == nil || *last.ZScore != *hrv.ZScore {
		t.Errorf("ZScore = %v, want %v (exact)", last.ZScore, hrv.ZScore)
	}
	if last.Status != hrv.Status {
		t.Errorf("Status = %q, want %q", last.Status, hrv.Status)
	}
}

func TestComputeSeries_ShortHistoryLeavesEarlyDaysUnknown(t *testing.T) {
	e := New(defaultCfg())
	hrv := make([]*float64, 61)
	// Only the last 25 input days have readings, so the first charted days have
	// fewer than min_baseline_days (14) behind them.
	for i := 36; i < 61; i++ {
		hrv[i] = p(80)
	}
	series, _ := e.ComputeSeries(wide(hrv))

	// Charted index 0 is input index 30: its sample is days[0:30] — all nil.
	if series[0].BaselineAvg != nil || series[0].BalancedLow != nil || series[0].BalancedHigh != nil {
		t.Errorf("series[0] should carry no band, got %+v", series[0])
	}
	if series[0].Status != StatusUnknown {
		t.Errorf("series[0].Status = %q, want %q", series[0].Status, StatusUnknown)
	}
	// Charted index 20 is input index 50: its sample days[20:50] holds 14
	// readings (indices 36..49) — exactly min_baseline_days.
	if !approx(series[20].BaselineAvg, 80) {
		t.Errorf("series[20].BaselineAvg = %v, want 80", series[20].BaselineAvg)
	}
	if series[20].Status != StatusBalanced {
		t.Errorf("series[20].Status = %q, want %q", series[20].Status, StatusBalanced)
	}
}

func TestComputeSeries_NullDayKeepsBandDropsZ(t *testing.T) {
	e := New(defaultCfg())
	hrv := flat(61, 80)
	hrv[45] = nil // charted index 15, a missing morning with a full history
	series, _ := e.ComputeSeries(wide(hrv))

	r := series[15]
	if !approx(r.BaselineAvg, 80) {
		t.Errorf("BaselineAvg = %v, want 80 — a missing morning must not erase the band", r.BaselineAvg)
	}
	if r.BalancedLow == nil || r.BalancedHigh == nil {
		t.Errorf("bounds should survive a null reading, got %+v", r)
	}
	if r.ZScore != nil {
		t.Errorf("ZScore = %v, want nil", r.ZScore)
	}
	if r.Status != StatusUnknown {
		t.Errorf("Status = %q, want %q", r.Status, StatusUnknown)
	}
}

func TestDrift_RisingFallingSteady(t *testing.T) {
	e := New(defaultCfg())
	cases := []struct {
		name string
		hrv  []*float64
		want string
	}{
		{"rising", ramp(61, 70, 0.5), TrendRising},
		{"falling", ramp(61, 100, -0.5), TrendFalling},
		{"steady", flat(61, 80), TrendSteady},
	}
	for _, tc := range cases {
		t.Run(tc.name, func(t *testing.T) {
			_, bt := e.ComputeSeries(wide(tc.hrv))
			if bt.Direction != tc.want {
				t.Errorf("Direction = %q, want %q", bt.Direction, tc.want)
			}
			if bt.DeltaMs == nil {
				t.Error("DeltaMs = nil, want a value")
			}
			if bt.FromAvg == nil {
				t.Error("FromAvg = nil, want a value")
			}
			if bt.OverDays != 28 {
				t.Errorf("OverDays = %d, want 28", bt.OverDays)
			}
		})
	}
}

func TestDrift_UnknownWhenSeriesTooShort(t *testing.T) {
	cfg := defaultCfg()
	cfg.BaselineDriftDays = 40 // the documented misconfiguration: > the series
	e := New(cfg)
	_, bt := e.ComputeSeries(wide(flat(61, 80)))
	if bt.Direction != TrendUnknown {
		t.Errorf("Direction = %q, want %q", bt.Direction, TrendUnknown)
	}
	if bt.DeltaMs != nil || bt.FromAvg != nil {
		t.Errorf("DeltaMs/FromAvg = %v/%v, want nil/nil", bt.DeltaMs, bt.FromAvg)
	}
	if bt.OverDays != 40 {
		t.Errorf("OverDays = %d, want 40", bt.OverDays)
	}
}

func TestDrift_UnknownWhenBaselineAbsent(t *testing.T) {
	e := New(defaultCfg())
	series, bt := e.ComputeSeries(wide(make([]*float64, 61)))
	if len(series) != 31 {
		t.Fatalf("len(series) = %d, want 31", len(series))
	}
	if bt.Direction != TrendUnknown {
		t.Errorf("Direction = %q, want %q", bt.Direction, TrendUnknown)
	}
	if bt.DeltaMs != nil || bt.FromAvg != nil {
		t.Errorf("DeltaMs/FromAvg = %v/%v, want nil/nil", bt.DeltaMs, bt.FromAvg)
	}
	if bt.OverDays != 28 {
		t.Errorf("OverDays = %d, want 28", bt.OverDays)
	}
}

func TestDrift_IsNotShortAvg(t *testing.T) {
	e := New(defaultCfg())
	// Baseline climbing all month (0..53 rise 70→96.5), then the last week
	// collapses to 75. The 30-day baseline is still well above where it stood
	// four weeks ago, while the 7-day mean sits well below it.
	hrv := ramp(61, 70, 0.5)
	for i := 54; i < 61; i++ {
		hrv[i] = p(75)
	}
	days := wide(hrv)

	_, hrvBlock := e.Compute(days[30:])
	_, bt := e.ComputeSeries(days)

	if hrvBlock.Trend != TrendFalling {
		t.Errorf("HRV.Trend = %q, want %q (7-day mean below baseline)", hrvBlock.Trend, TrendFalling)
	}
	if bt.Direction != TrendRising {
		t.Errorf("BaselineTrend.Direction = %q, want %q (baseline above its own past)", bt.Direction, TrendRising)
	}
}
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `cd /workspace/prog-strength-api && go test ./internal/recoverytrend/ 2>&1 | tail -20`
Expected: FAIL — `e.ComputeSeries undefined (type *Engine has no field or method ComputeSeries)`.

- [ ] **Step 3: Add the two value types**

In `internal/recoverytrend/recoverytrend.go`, after the `HRV` struct (line ~68) and before `type Engine struct`:

```go
// DayResult is one charted day read against ITS OWN trailing baseline — the
// window of BaselineWindowDays ending the day before it. Every field is nil
// (Status unknown) until that day has MinBaselineDays samples behind it, which
// is why the band on a chart legitimately begins part-way across.
type DayResult struct {
	Date         string
	BaselineAvg  *float64
	BalancedLow  *float64
	BalancedHigh *float64
	ZScore       *float64
	Status       string
}

// BaselineTrend is the baseline measured against its own past: today's
// baseline minus the baseline as it stood BaselineDriftDays earlier. This is a
// DIFFERENT question from HRV.Trend, which compares the recent mean against the
// window it sits inside. The two may legitimately point opposite ways — a
// rising baseline under a suppressed morning is a real state, not a bug.
type BaselineTrend struct {
	Direction string   // rising | falling | steady | unknown
	DeltaMs   *float64 // now − then; nil when either baseline is absent
	FromAvg   *float64 // the baseline it moved from
	OverDays  int      // always populated, even when Direction is unknown
}
```

- [ ] **Step 4: Add `ComputeSeries` and `drift`**

In `internal/recoverytrend/recoverytrend.go`, after `Compute` and its extracted `band`/`classify` helpers, before `collect`:

```go
// ComputeSeries derives, for every charted day, that day's own trailing
// baseline and the standing of that day's reading against it, plus the drift of
// the baseline against its own past.
//
// days must be date-ascending and must carry BaselineWindowDays days of LEAD-IN
// before the first charted day, so the oldest charted day has a full trailing
// sample. The returned slice covers days[BaselineWindowDays:] in the same order
// — 31 entries at the default 30-day window fed a 61-day input.
//
// Pure: no clock, no DB, no I/O. Absent days must be present in the input with
// nil metrics, exactly as Compute requires.
func (e *Engine) ComputeSeries(days []Day) ([]DayResult, BaselineTrend) {
	lead := e.cfg.BaselineWindowDays
	if lead > len(days) {
		lead = len(days)
	}

	series := make([]DayResult, 0, len(days)-lead)
	var lastSDEff float64
	for i := lead; i < len(days); i++ {
		// Trailing window ending the day BEFORE i — the same exclude-the-day-
		// itself rule Compute applies to today, applied to every day.
		from := i - e.cfg.BaselineWindowDays
		if from < 0 {
			from = 0
		}
		sample := collect(days[from:i], func(d Day) *float64 { return d.HRV })

		r := DayResult{Date: days[i].Date, Status: StatusUnknown}
		if len(sample) >= e.cfg.MinBaselineDays {
			avg := mean(sample)
			sd := stdDevPop(sample, avg)
			low, high, sdEff := e.band(avg, sd)
			lastSDEff = sdEff
			r.BaselineAvg, r.BalancedLow, r.BalancedHigh = &avg, &low, &high
			if v := days[i].HRV; v != nil {
				z := (*v - avg) / sdEff
				r.ZScore = &z
				r.Status = e.classify(z)
			}
		}
		series = append(series, r)
	}

	return series, e.drift(series, lastSDEff)
}

// drift compares the newest baseline in the series against the baseline
// BaselineDriftDays earlier. Unknown when the series is too short to reach
// back that far, when either endpoint has no baseline yet, or when no day in
// the series produced a spread to threshold against.
func (e *Engine) drift(series []DayResult, sdEff float64) BaselineTrend {
	bt := BaselineTrend{Direction: TrendUnknown, OverDays: e.cfg.BaselineDriftDays}
	back := len(series) - 1 - e.cfg.BaselineDriftDays
	if back < 0 || sdEff <= 0 {
		return bt
	}
	now, then := series[len(series)-1].BaselineAvg, series[back].BaselineAvg
	if now == nil || then == nil {
		return bt
	}
	d := *now - *then
	bt.DeltaMs, bt.FromAvg = &d, then
	switch {
	case d > e.cfg.BaselineDriftZ*sdEff:
		bt.Direction = TrendRising
	case d < -e.cfg.BaselineDriftZ*sdEff:
		bt.Direction = TrendFalling
	default:
		bt.Direction = TrendSteady
	}
	return bt
}
```

Note the guard on `back < 0`: it is also what makes an empty series safe, because `len(series) - 1 - BaselineDriftDays` is negative for any non-negative `BaselineDriftDays`.

- [ ] **Step 5: Run the tests to verify they pass**

Run: `cd /workspace/prog-strength-api && go test ./internal/recoverytrend/ -v 2>&1 | tail -50`
Expected: every test PASS, including all pre-existing `TestCompute_*` tests.

- [ ] **Step 6: Run the lint gate**

Run:
```bash
cd /workspace/prog-strength-api
go vet ./internal/recoverytrend/
go run github.com/golangci/golangci-lint/v2/cmd/golangci-lint@v2.12.2 fmt --diff
go run github.com/golangci/golangci-lint/v2/cmd/golangci-lint@v2.12.2 run
```
Expected: no diff, `0 issues.`

- [ ] **Step 7: Commit**

```bash
cd /workspace/prog-strength-api
git add internal/recoverytrend
git commit -m "feat(recoverytrend): add ComputeSeries and baseline drift"
```

---

## Task 4: Document the series and the drift in `doc.go`

**Files:**
- Modify: `internal/recoverytrend/doc.go`

- [ ] **Step 1: Append the new section**

In `internal/recoverytrend/doc.go`, insert after the `# The trend` section and before the closing `package recoverytrend` line:

```go
// # The series and the drift
//
// Compute answers one question about one day. ComputeSeries answers it once per
// charted day: for every day in the series it derives that day's OWN trailing
// baseline from the BaselineWindowDays that preceded it, applying the same
// exclude-the-day-itself rule Compute applies to today. A client can therefore
// colour each night against the band as it stood on that night's morning rather
// than against today's band, which would measure July against an August
// baseline.
//
// This is why the input is WIDE. ComputeSeries takes BaselineWindowDays of
// lead-in ahead of the first charted day — 61 dates for a 31-day series at the
// default window — so the OLDEST charted day still has a full trailing sample.
// Without the lead-in the left edge of the band would wobble as the sample
// truncated, an artefact of the window rather than anything about the athlete.
// The lead-in is input only; the returned slice covers days[BaselineWindowDays:]
// and nothing older is ever serialized.
//
// Both entry points share band and classify, so the bounds and status the
// series reports for its last day are bit-identical to the scalar HRV block
// computed from the same sample. That agreement is structural, not a
// convention, and it is pinned by test.
//
// BaselineTrend is a DIFFERENT question from HRV.Trend, and the distinction is
// the point of the whole series. HRV.Trend compares the recent TrendWindowDays
// mean against the baseline it sits inside: "is this week off my normal?"
// BaselineTrend compares the baseline against the baseline as it stood
// BaselineDriftDays ago: "is my normal itself moving?" A climbing baseline is
// adaptation; a sinking one is a reason to look at sleep, load, or illness. The
// two can legitimately point opposite ways — a rising baseline under a
// suppressed morning is a real state, not a bug, and a client that renders both
// must not try to reconcile them.
//
// The drift threshold is SD-relative, like every other threshold here: a 6 ms
// move is signal for an athlete whose spread is 8 ms and noise for one whose
// spread is 25 ms. It uses the MOST RECENT day's effective SD, so the verdict is
// scaled to the spread the athlete has now rather than one they have grown out
// of. Direction is unknown when the series cannot reach back BaselineDriftDays,
// when either endpoint has no baseline yet, or when no charted day ever reached
// MinBaselineDays — the same honest silence the bounds already use.
```

- [ ] **Step 2: Verify the package doc renders and still builds**

Run: `cd /workspace/prog-strength-api && go build ./internal/recoverytrend/ && go doc ./internal/recoverytrend 2>&1 | head -20`
Expected: builds; `go doc` prints the package synopsis.

- [ ] **Step 3: Run the gate**

Run:
```bash
cd /workspace/prog-strength-api
go run github.com/golangci/golangci-lint/v2/cmd/golangci-lint@v2.12.2 fmt --diff
go run github.com/golangci/golangci-lint/v2/cmd/golangci-lint@v2.12.2 run
go test ./internal/recoverytrend/
```
Expected: no diff, `0 issues.`, PASS.

- [ ] **Step 4: Commit**

```bash
cd /workspace/prog-strength-api
git add internal/recoverytrend/doc.go
git commit -m "docs(recoverytrend): document the rolling series and baseline drift"
```

---

## Task 5: Wire types in `internal/dashboard/dto.go`

Types only — `whoop.go` still populates the old shape, so this task must leave the tree compiling. `RecoveryDayPoint` embeds `RecoveryDay`, which means the existing `Days` construction in `whoop.go` needs a mechanical adjustment in the same commit to keep `go build` green; Task 6 then rewrites that block properly.

**Files:**
- Modify: `internal/dashboard/dto.go:61-80`
- Modify: `internal/dashboard/whoop.go:66-79` (mechanical only)

- [ ] **Step 1: Re-type `RecoverySection` and add the two types**

In `internal/dashboard/dto.go`, replace the `RecoverySection` struct's doc comment and body:

```go
// RecoverySection is the Whoop recovery tile. nil at the Summary level unless a
// connected Whoop connection exists. Today and RestingHRSpark are unchanged in
// shape, semantics, and content; Days, Baseline, HRV, and BaselineTrend are
// additive.
type RecoverySection struct {
	Today          *RecoveryDay          `json:"today"`            // nil when no row today
	RestingHRSpark []float64             `json:"resting_hr_spark"` // trailing 7 days resting HR (oldest→newest), missing days omitted
	Days           []RecoveryDayPoint    `json:"days"`             // full window, date-aligned oldest→newest, missing days present with null metrics
	Baseline       RecoveryBaseline      `json:"baseline"`         // trailing averages + HRV spread; always present
	HRV            RecoveryHRV           `json:"hrv"`              // today's HRV vs the user's own baseline; always present
	BaselineTrend  RecoveryBaselineTrend `json:"baseline_trend"`   // the baseline vs its own past; always present
}
```

Then, immediately after the existing `RecoveryDay` struct, add:

```go
// RecoveryDayPoint is one charted day: the day's raw metrics (embedded, so the
// existing four keys serialize unchanged) plus the band as it stood on THAT
// day. The band fields are null and Status is "unknown" until the day has
// min_baseline_days behind it, so a client's band legitimately begins part-way
// across the series for a user with short history.
type RecoveryDayPoint struct {
	RecoveryDay
	BaselineAvg  *float64 `json:"baseline_avg"`
	BalancedLow  *float64 `json:"balanced_low"`
	BalancedHigh *float64 `json:"balanced_high"`
	ZScore       *float64 `json:"z_score"`
	Status       string   `json:"status"` // balanced | elevated | suppressed | unknown
}
```

and, after the `RecoveryHRV` struct, add:

```go
// RecoveryBaselineTrend is the baseline against its own past. NOT the same
// question as RecoveryHRV.Trend, which compares the recent mean against the
// window it sits inside; these two may point opposite ways. Always present —
// "nothing to say" is Direction "unknown", never an absent key.
type RecoveryBaselineTrend struct {
	Direction string   `json:"direction"` // rising | falling | steady | unknown
	DeltaMs   *float64 `json:"delta_ms"`
	FromAvg   *float64 `json:"from_avg"`
	OverDays  int      `json:"over_days"`
}
```

- [ ] **Step 2: Keep `whoop.go` compiling (mechanical, superseded by Task 6)**

In `internal/dashboard/whoop.go`, change the `Days` materialisation so the element type matches. Replace:

```go
	section.Days = make([]RecoveryDay, 0, win+1)
	for i := win; i >= 0; i-- {
		day := time.Date(y, mo, d-i, 0, 0, 0, 0, loc)
		ds := day.Format("2006-01-02")
		rd := RecoveryDay{Date: ds}
		if e, ok := byDate[ds]; ok {
			rd.RestingHeartRate = e.RestingHeartRate
			rd.RecoveryScore = e.RecoveryScore
			rd.HRVRmssdMilli = e.HRVRmssdMilli
		}
		section.Days = append(section.Days, rd)
	}
```

with:

```go
	section.Days = make([]RecoveryDayPoint, 0, win+1)
	for i := win; i >= 0; i-- {
		day := time.Date(y, mo, d-i, 0, 0, 0, 0, loc)
		ds := day.Format("2006-01-02")
		rd := RecoveryDayPoint{RecoveryDay: RecoveryDay{Date: ds}, Status: recoverytrend.StatusUnknown}
		if e, ok := byDate[ds]; ok {
			rd.RestingHeartRate = e.RestingHeartRate
			rd.RecoveryScore = e.RecoveryScore
			rd.HRVRmssdMilli = e.HRVRmssdMilli
		}
		section.Days = append(section.Days, rd)
	}
```

and in the `engineDays` copy loop below it, `rd.Date` etc. still resolve through the embed, so that loop is unchanged.

Also set the new section field so the always-present contract holds even at this intermediate step — after the `section.HRV = …` assignment:

```go
	section.BaselineTrend = RecoveryBaselineTrend{Direction: recoverytrend.TrendUnknown, OverDays: 0}
```

Task 6 replaces both of these with the real thing.

- [ ] **Step 3: Verify the tree builds and existing tests pass**

Run: `cd /workspace/prog-strength-api && go build ./... && go test ./internal/dashboard/ 2>&1 | tail -30`
Expected: builds. Some `internal/dashboard` tests may fail if they assert the exact JSON key set of the section — if `TestRecoverySection_JSONKeys` (or similar) fails because `baseline_trend`/the five day keys are now present, that is expected; extend those assertions in Task 6, not here. Everything else must pass.

- [ ] **Step 4: Run the gate and commit**

```bash
cd /workspace/prog-strength-api
go vet ./internal/dashboard/
go run github.com/golangci/golangci-lint/v2/cmd/golangci-lint@v2.12.2 fmt --diff
go run github.com/golangci/golangci-lint/v2/cmd/golangci-lint@v2.12.2 run
git add internal/dashboard/dto.go internal/dashboard/whoop.go
git commit -m "feat(dashboard): add RecoveryDayPoint and RecoveryBaselineTrend wire types"
```

If `go test ./internal/dashboard/` fails only on the JSON-key assertion described above, commit anyway and fix it in Task 6 — but say so explicitly in your report. If anything else fails, stop and fix it here.

---

## Task 6: Widen the read path and wire the series into `buildWhoop`

**Files:**
- Modify: `internal/dashboard/handler.go:437-448`
- Modify: `internal/dashboard/whoop.go:26-110`
- Modify: `internal/dashboard/whoop_test.go`

- [ ] **Step 1: Write the failing tests**

Append to `internal/dashboard/whoop_test.go` (matching the file's existing fixture style — read the top of the file first and reuse whatever entry-building helper it already has; if it builds `[]whooprecovery.Entry` literals inline, do the same):

```go
// entriesEndingAt builds n consecutive daily whoop entries ending on the local
// date of now, oldest first, with a constant HRV.
func entriesEndingAt(now time.Time, loc *time.Location, n int, hrv float64) []whooprecovery.Entry {
	local := now.In(loc)
	y, mo, d := local.Date()
	out := make([]whooprecovery.Entry, 0, n)
	for i := n - 1; i >= 0; i-- {
		day := time.Date(y, mo, d-i, 0, 0, 0, 0, loc)
		v := hrv
		rhr := 52.0
		score := 65.0
		out = append(out, whooprecovery.Entry{
			Date:             day.Format("2006-01-02"),
			RestingHeartRate: &rhr,
			RecoveryScore:    &score,
			HRVRmssdMilli:    &v,
		})
	}
	return out
}

func TestBuildWhoop_ScalarBlocksUnchangedByLeadIn(t *testing.T) {
	denver := mustLoad(t, "America/Denver")
	now := time.Date(2026, 8, 9, 12, 0, 0, 0, denver)

	// A charted window's worth of data and nothing older.
	narrow := entriesEndingAt(now, denver, 31, 80)
	before := buildWhoop(narrow, testRecoveryEngine(), now, denver)

	// Now add 30 days of much higher readings STRICTLY OLDER than the charted
	// window — the lead-in the widened fetch will now return.
	local := now.In(denver)
	y, mo, d := local.Date()
	wide := make([]whooprecovery.Entry, 0, 61)
	for i := 60; i >= 31; i-- {
		day := time.Date(y, mo, d-i, 0, 0, 0, 0, denver)
		v := 200.0
		wide = append(wide, whooprecovery.Entry{Date: day.Format("2006-01-02"), HRVRmssdMilli: &v})
	}
	wide = append(wide, narrow...)
	after := buildWhoop(wide, testRecoveryEngine(), now, denver)

	if !reflect.DeepEqual(before.Baseline, after.Baseline) {
		t.Errorf("Baseline moved with lead-in:\n before = %+v\n after  = %+v", before.Baseline, after.Baseline)
	}
	if !reflect.DeepEqual(before.HRV, after.HRV) {
		t.Errorf("HRV moved with lead-in:\n before = %+v\n after  = %+v", before.HRV, after.HRV)
	}
}

func TestBuildWhoop_DaysCarryPerDayBands(t *testing.T) {
	denver := mustLoad(t, "America/Denver")
	now := time.Date(2026, 8, 9, 12, 0, 0, 0, denver)
	got := buildWhoop(entriesEndingAt(now, denver, 61, 80), testRecoveryEngine(), now, denver)

	if len(got.Days) != 31 {
		t.Fatalf("len(Days) = %d, want 31", len(got.Days))
	}
	last := got.Days[len(got.Days)-1]
	if last.Date != now.In(denver).Format("2006-01-02") {
		t.Errorf("last Date = %q, want today", last.Date)
	}
	if last.Status != got.HRV.Status {
		t.Errorf("days[last].Status = %q, want HRV.Status %q", last.Status, got.HRV.Status)
	}
	if last.BalancedLow == nil || got.HRV.BalancedLow == nil || *last.BalancedLow != *got.HRV.BalancedLow {
		t.Errorf("days[last].BalancedLow = %v, want %v", last.BalancedLow, got.HRV.BalancedLow)
	}
	if last.BalancedHigh == nil || got.HRV.BalancedHigh == nil || *last.BalancedHigh != *got.HRV.BalancedHigh {
		t.Errorf("days[last].BalancedHigh = %v, want %v", last.BalancedHigh, got.HRV.BalancedHigh)
	}
	if last.BaselineAvg == nil || got.Baseline.HRVAvg == nil || *last.BaselineAvg != *got.Baseline.HRVAvg {
		t.Errorf("days[last].BaselineAvg = %v, want %v", last.BaselineAvg, got.Baseline.HRVAvg)
	}
}

func TestBuildWhoop_LeadInNotSerialized(t *testing.T) {
	denver := mustLoad(t, "America/Denver")
	now := time.Date(2026, 8, 9, 12, 0, 0, 0, denver)
	got := buildWhoop(entriesEndingAt(now, denver, 61, 80), testRecoveryEngine(), now, denver)

	local := now.In(denver)
	y, mo, d := local.Date()
	oldest := time.Date(y, mo, d-30, 0, 0, 0, 0, denver).Format("2006-01-02")
	if got.Days[0].Date != oldest {
		t.Errorf("Days[0].Date = %q, want %q — lead-in must not be serialized", got.Days[0].Date, oldest)
	}
	for _, dp := range got.Days {
		if dp.Date < oldest {
			t.Errorf("Days carries lead-in date %q, older than %q", dp.Date, oldest)
		}
	}
}

func TestBuildWhoop_BaselineTrendPresentWhenNoEntries(t *testing.T) {
	denver := mustLoad(t, "America/Denver")
	now := time.Date(2026, 8, 9, 12, 0, 0, 0, denver)
	got := buildWhoop(nil, testRecoveryEngine(), now, denver)
	if got == nil {
		t.Fatal("connected user should always get a section")
		return
	}
	if got.BaselineTrend.Direction != recoverytrend.TrendUnknown {
		t.Errorf("Direction = %q, want %q", got.BaselineTrend.Direction, recoverytrend.TrendUnknown)
	}
	if got.BaselineTrend.OverDays != 28 {
		t.Errorf("OverDays = %d, want 28", got.BaselineTrend.OverDays)
	}
	if got.BaselineTrend.DeltaMs != nil || got.BaselineTrend.FromAvg != nil {
		t.Errorf("DeltaMs/FromAvg = %v/%v, want nil/nil", got.BaselineTrend.DeltaMs, got.BaselineTrend.FromAvg)
	}
}
```

Then find the existing JSON-key test for the recovery section (search the `internal/dashboard` package for the test that marshals a `RecoverySection` and asserts its key set — `grep -rn "resting_hr_spark" internal/dashboard/*_test.go`). Extend it so that:

- the section's key set additionally contains `baseline_trend`;
- `baseline_trend`'s object has exactly `direction`, `delta_ms`, `from_avg`, `over_days`;
- each `days[]` entry has exactly the original four keys **plus** `baseline_avg`, `balanced_low`, `balanced_high`, `z_score`, `status`;
- `today` has **exactly** its original four keys — `date`, `resting_heart_rate`, `recovery_score`, `hrv_rmssd_milli` — and nothing else. Assert the exact set, not a subset: the embedded-struct approach is right, but this is precisely the thing that silently leaks fields if someone later moves the embed.

If no such test exists in the package, add one named `TestRecoverySection_JSONKeys` that marshals a section built by `buildWhoop` over `entriesEndingAt(now, denver, 61, 80)` and asserts all four bullets above via `map[string]json.RawMessage`.

Note the new imports the test file needs: `recoverytrend` (for the `TrendUnknown` constant), and `encoding/json` if you add the key test. `reflect`, `time`, `testing`, and `whooprecovery` are already imported.

- [ ] **Step 2: Run the tests to verify they fail**

Run: `cd /workspace/prog-strength-api && go test ./internal/dashboard/ -run 'TestBuildWhoop|TestRecoverySection' 2>&1 | tail -30`
Expected: FAIL — `len(Days) = 31, want 31` will pass but `days[last].Status = "unknown"`, `BaselineAvg = <nil>`, and `OverDays = 0` will fail, because Task 5 only stubbed the fields.

- [ ] **Step 3: Widen the fetch in `handler.go`**

In `internal/dashboard/handler.go`, inside `buildRecoverySection`, replace the comment and `sinceStr` line:

```go
	// baseline_window_days local dates BEFORE today through today inclusive — the
	// full window the trend engine needs (31 dates at the default). Same single
	// indexed ListRange call as before (idx_user_whoop_recovery_user_date covers
	// (user_id, date DESC)); the read just returns ≤31 rows instead of ≤7.
	win := h.recovery.BaselineWindowDays()
	sinceStr := now.In(loc).AddDate(0, 0, -win).Format("2006-01-02")
```

with:

```go
	// TWO baseline windows plus today: baseline_window_days of lead-in so the
	// OLDEST charted day still has a full trailing sample, then the charted
	// window itself (61 dates at the defaults). Same single indexed ListRange
	// call as before — idx_user_whoop_recovery_user_date covers (user_id,
	// date DESC) — the read just returns ≤61 rows instead of ≤31.
	win := h.recovery.BaselineWindowDays()
	sinceStr := now.In(loc).AddDate(0, 0, -2*win).Format("2006-01-02")
```

Also update the `buildRecoverySection` doc comment: change "it fetches the trailing baseline_window_days+1 local days of recovery" to "it fetches 2×baseline_window_days+1 local days of recovery — a window of lead-in ahead of the charted window, so every charted day has a full trailing baseline".

- [ ] **Step 4: Rewrite the `buildWhoop` window block**

In `internal/dashboard/whoop.go`, replace everything from the `// The full date-aligned window:` comment through the `section.HRV = …` assignment (i.e. the block Task 5 mechanically patched, the `engineDays` copy loop, the `eng.Compute` call, and the two assignment blocks) with:

```go
	// Materialize the WIDE window: 2*win + 1 local dates ending today,
	// oldest→newest, every date present, missing days carrying null metrics
	// (never omitted, never zero-filled). The first `win` dates are lead-in for
	// the rolling baseline and are NOT serialized.
	win := eng.BaselineWindowDays()
	total := 2*win + 1
	engineDays := make([]recoverytrend.Day, 0, total)
	for i := total - 1; i >= 0; i-- {
		day := time.Date(y, mo, d-i, 0, 0, 0, 0, loc)
		ds := day.Format("2006-01-02")
		ed := recoverytrend.Day{Date: ds}
		if e, ok := byDate[ds]; ok {
			ed.RestingHR = e.RestingHeartRate
			ed.RecoveryScore = e.RecoveryScore
			ed.HRV = e.HRVRmssdMilli
		}
		engineDays = append(engineDays, ed)
	}

	// The charted window is the last win+1 dates — byte-for-byte the window
	// this function built before the lead-in was added.
	charted := engineDays[win:]
	series, drift := eng.ComputeSeries(engineDays)
	baseline, hrv := eng.Compute(charted)

	// Zip the charted metrics with their per-day bands. Index-aligned by
	// construction: ComputeSeries returns exactly one result per charted day,
	// in the same order.
	section.Days = make([]RecoveryDayPoint, len(charted))
	for i, ed := range charted {
		section.Days[i] = RecoveryDayPoint{
			RecoveryDay: RecoveryDay{
				Date:             ed.Date,
				RestingHeartRate: ed.RestingHR,
				RecoveryScore:    ed.RecoveryScore,
				HRVRmssdMilli:    ed.HRV,
			},
			BaselineAvg:  series[i].BaselineAvg,
			BalancedLow:  series[i].BalancedLow,
			BalancedHigh: series[i].BalancedHigh,
			ZScore:       series[i].ZScore,
			Status:       series[i].Status,
		}
	}

	section.Baseline = RecoveryBaseline{
		WindowDays:        baseline.WindowDays,
		RestingHRAvg:      baseline.RestingHRAvg,
		RestingHRDays:     baseline.RestingHRDays,
		HRVAvg:            baseline.HRVAvg,
		HRVStdDev:         baseline.HRVStdDev,
		HRVDays:           baseline.HRVDays,
		RecoveryScoreAvg:  baseline.RecoveryScoreAvg,
		RecoveryScoreDays: baseline.RecoveryScoreDays,
	}
	section.HRV = RecoveryHRV{
		Status:       hrv.Status,
		BalancedLow:  hrv.BalancedLow,
		BalancedHigh: hrv.BalancedHigh,
		ZScore:       hrv.ZScore,
		Trend:        hrv.Trend,
		ShortAvg:     hrv.ShortAvg,
	}
	section.BaselineTrend = RecoveryBaselineTrend{
		Direction: drift.Direction,
		DeltaMs:   drift.DeltaMs,
		FromAvg:   drift.FromAvg,
		OverDays:  drift.OverDays,
	}

	return section
```

The `Today` row and the `RestingHRSpark` loop above are untouched.

Update `buildWhoop`'s doc comment's last paragraph to read:

```go
// Today and RestingHRSpark are built exactly as before; Days is the honest
// date-aligned history (missing days present with null metrics) with each day's
// own trailing band attached, and the engine derives Baseline, HRV, and
// BaselineTrend from a window that carries baseline_window_days of lead-in
// ahead of the charted days.
```

- [ ] **Step 5: Run the tests to verify they pass**

Run: `cd /workspace/prog-strength-api && go test ./internal/dashboard/ -v 2>&1 | tail -60`
Expected: all PASS, including every pre-existing `TestBuildWhoop_*`.

- [ ] **Step 6: Run the full gate**

Run:
```bash
cd /workspace/prog-strength-api
go build ./... && go vet ./...
go mod tidy && git diff --exit-code go.mod go.sum
go run github.com/golangci/golangci-lint/v2/cmd/golangci-lint@v2.12.2 fmt --diff
go run github.com/golangci/golangci-lint/v2/cmd/golangci-lint@v2.12.2 run
go test ./...
```
Expected: everything green, no `go.mod`/`go.sum` drift.

- [ ] **Step 7: Commit**

```bash
cd /workspace/prog-strength-api
git add internal/dashboard
git commit -m "feat(dashboard): emit per-day recovery bands and baseline_trend"
```

---

## Task 7: Mirror the wire types in `prog-strength-web`'s `lib/api.ts`

**Files:**
- Modify: `lib/api.ts:4705-4765`

- [ ] **Step 1: Add the two types and re-type `DashboardRecovery`**

In `/workspace/prog-strength-web/lib/api.ts`, after the `DashboardRecoveryHrv` type (around line 4747) and before `DashboardRecovery`, add:

```ts
/**
 * One charted day of the recovery series: the day's raw metrics plus the band
 * as it stood on that day. The band fields are null and `status` is "unknown"
 * until the day has `min_baseline_days` behind it.
 */
export type DashboardRecoveryDayPoint = DashboardRecoveryDay & {
  baseline_avg: number | null;
  balanced_low: number | null;
  balanced_high: number | null;
  z_score: number | null;
  status: string;
};

/**
 * The baseline against its own past. Distinct from `DashboardRecoveryHrv.trend`,
 * which compares the recent mean against the window it sits inside; the two may
 * point opposite ways.
 */
export type DashboardRecoveryBaselineTrend = {
  direction: string;
  delta_ms: number | null;
  from_avg: number | null;
  over_days: number;
};
```

Then in `DashboardRecovery`, change `days: DashboardRecoveryDay[];` to `days: DashboardRecoveryDayPoint[];` and add after `hrv: DashboardRecoveryHrv;`:

```ts
  baseline_trend: DashboardRecoveryBaselineTrend;
```

Leave `today: DashboardRecoveryDay | null;` alone — only `days[]` gains fields, which is why this is a separate type rather than a widening of the shared one.

- [ ] **Step 2: Typecheck**

Run: `cd /workspace/prog-strength-web && npm run typecheck 2>&1 | tail -30`
Expected: FAIL on `lib/dashboard.test.ts` and `app/(app)/dashboard/page.test.tsx` — their `DashboardRecovery` fixtures are now missing `baseline_trend`. Tasks 8 fixes both. `lib/dashboard.ts` itself should still typecheck, since `days.map` reads only the four original keys.

- [ ] **Step 3: Commit**

```bash
cd /workspace/prog-strength-web
git add lib/api.ts
git commit -m "feat(api): mirror recovery day-point and baseline-trend wire types"
```

If the repo's husky pre-commit hook rejects the commit because the typecheck is red at this intermediate point, do NOT pass `--no-verify`. Instead fold Task 7 and Task 8 into a single commit: complete Task 8's edits first, then run this commit with both files staged. Report which path you took.

---

## Task 8: Pass the new fields through `lib/dashboard.ts`

**Files:**
- Modify: `lib/dashboard.ts:218-270`, `:533-568`
- Modify: `lib/dashboard.test.ts:~375-450`
- Modify: `app/(app)/dashboard/page.test.tsx:~240-265`
- Modify: `app/(app)/dashboard/_components/recovery/fixtures.ts:57-65`

- [ ] **Step 1: Write the failing tests**

In `/workspace/prog-strength-web/lib/dashboard.test.ts`, first make the fixture builder compile: add `baseline_trend` to the `recoveryBlock` defaults. Near `UNKNOWN_HRV`, add:

```ts
const UNKNOWN_BASELINE_TREND = {
  direction: "unknown",
  delta_ms: null,
  from_avg: null,
  over_days: 28,
};
```

and inside `recoveryBlock`'s returned object, after `hrv: UNKNOWN_HRV,`:

```ts
    baseline_trend: UNKNOWN_BASELINE_TREND,
```

Then add two tests inside the existing `describe("adaptDashboard — recovery (Whoop)", …)` block:

```ts
  it("passes per-day bands and the baseline trend straight through", () => {
    const data = adaptDashboard(
      {
        ...fullSummary,
        recovery: recoveryBlock({
          days: [
            {
              date: "2026-08-09",
              resting_heart_rate: 59,
              recovery_score: 61,
              hrv_rmssd_milli: 77.4,
              baseline_avg: 88.2,
              balanced_low: 68.1,
              balanced_high: 108.3,
              z_score: -0.53,
              status: "balanced",
            },
          ],
          baseline_trend: {
            direction: "rising",
            delta_ms: 6.4,
            from_avg: 81.8,
            over_days: 28,
          },
        }),
      },
      profile(),
    );
    const recovery = data.sections.recovery;
    if (!recovery.present) throw new Error("recovery section should be present");
    expect(recovery.days?.[0]).toEqual({
      date: "2026-08-09",
      restingHr: 59,
      recoveryScore: 61,
      hrv: 77.4,
      baselineAvg: 88.2,
      balancedLow: 68.1,
      balancedHigh: 108.3,
      zScore: -0.53,
      status: "balanced",
    });
    expect(recovery.baselineTrend).toEqual({
      direction: "rising",
      deltaMs: 6.4,
      fromAvg: 81.8,
      overDays: 28,
    });
  });

  it("narrows unrecognised day status and drift direction to unknown", () => {
    const data = adaptDashboard(
      {
        ...fullSummary,
        recovery: recoveryBlock({
          days: [
            {
              date: "2026-08-09",
              resting_heart_rate: null,
              recovery_score: null,
              hrv_rmssd_milli: null,
              baseline_avg: null,
              balanced_low: null,
              balanced_high: null,
              z_score: null,
              status: "sideways",
            },
          ],
          baseline_trend: {
            direction: "sideways",
            delta_ms: null,
            from_avg: null,
            over_days: 28,
          },
        }),
      },
      profile(),
    );
    const recovery = data.sections.recovery;
    if (!recovery.present) throw new Error("recovery section should be present");
    expect(recovery.days?.[0].status).toBe("unknown");
    expect(recovery.baselineTrend?.direction).toBe("unknown");
  });
```

Match the file's existing idiom for reaching into `data.sections.recovery` — read the neighbouring tests first and mirror exactly how they narrow the `Section<RecoveryView>` union (the `if (!recovery.present) throw` shape above is a placeholder for whatever the file already does).

In `/workspace/prog-strength-web/app/(app)/dashboard/page.test.tsx`, add `baseline_trend` to the recovery payload fixture so the page test compiles, directly after its `hrv: { … }` block:

```ts
        baseline_trend: {
          direction: "unknown",
          delta_ms: null,
          from_avg: null,
          over_days: 28,
        },
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `cd /workspace/prog-strength-web && npx vitest run lib/dashboard.test.ts 2>&1 | tail -30`
Expected: FAIL — the adapter does not emit `baselineAvg`/`baselineTrend` yet.

- [ ] **Step 3: Extend the view-model types**

In `/workspace/prog-strength-web/lib/dashboard.ts`, replace the `RecoveryDayPoint` type:

```ts
/**
 * A single date-aligned day in the recovery history; nulls preserved as null.
 * The band fields are that day's OWN trailing band — server figures, passed
 * through untouched — and are null (`status` "unknown") until the day has
 * enough history behind it.
 */
export type RecoveryDayPoint = {
  date: string;
  restingHr: number | null;
  recoveryScore: number | null;
  hrv: number | null;
  baselineAvg: number | null;
  balancedLow: number | null;
  balancedHigh: number | null;
  zScore: number | null;
  status: RecoveryHrvStatus;
};
```

(keep whatever the existing four fields' exact names and order are — `date`, `restingHr`, `recoveryScore`, `hrv` — and append the five new ones.)

After `RecoveryHrvView`, add:

```ts
/**
 * The baseline against its own past. Distinct from `RecoveryHrvView.trend`,
 * which compares the recent mean against the window it sits inside; the two may
 * point opposite ways. Nothing renders this yet.
 */
export type RecoveryBaselineTrendView = {
  direction: RecoveryTrendDirection;
  deltaMs: number | null;
  fromAvg: number | null;
  overDays: number;
};
```

On `RecoveryView`, after `hrv?: RecoveryHrvView;`:

```ts
  baselineTrend?: RecoveryBaselineTrendView;
```

Optional, matching its `days?`/`baseline?`/`hrv?` neighbours, so consumers keep guarding rather than `!`-asserting.

Also add `DashboardRecoveryDayPoint` / `DashboardRecoveryBaselineTrend` to the `import type { … } from "@/lib/api"` block **only if** the adapter references them by name; the code below does not, so no import change is needed.

- [ ] **Step 4: Extend `adaptRecovery`**

In `adaptRecovery`, replace the `days:` mapping and add `baselineTrend` after the `hrv: { … }` block:

```ts
    days: recovery.days.map((d) => ({
      date: d.date,
      restingHr: d.resting_heart_rate,
      recoveryScore: d.recovery_score,
      hrv: d.hrv_rmssd_milli,
      // Server figures, passed through — never re-derived client-side.
      baselineAvg: d.baseline_avg,
      balancedLow: d.balanced_low,
      balancedHigh: d.balanced_high,
      zScore: d.z_score,
      status: recoveryStatus(d.status),
    })),
```

```ts
    baselineTrend: {
      direction: recoveryTrend(recovery.baseline_trend.direction),
      deltaMs: recovery.baseline_trend.delta_ms,
      fromAvg: recovery.baseline_trend.from_avg,
      overDays: recovery.baseline_trend.over_days,
    },
```

It reuses the existing `recoveryStatus` and `recoveryTrend` narrowers rather than adding new ones — the vocabularies are identical.

- [ ] **Step 5: Update the tile fixtures so they satisfy the widened type**

In `/workspace/prog-strength-web/app/(app)/dashboard/_components/recovery/fixtures.ts`, `makeDays` now has to emit the five new fields. Replace it with:

```ts
/**
 * Build the 31-day window from an HRV series; a null HRV ⇒ a fully-null day.
 * The per-day band fields are left null / "unknown": no shipped tile renders
 * them yet, and a fabricated per-day band would be fixture fiction.
 */
export function makeDays(hrv: (number | null)[] = HRV_SERIES): RecoveryDayPoint[] {
  return hrv.map((v, i) => ({
    date: isoDate(i),
    hrv: v,
    restingHr: v === null ? null : 49 + (i % 6),
    recoveryScore: v === null ? null : 48 + (i % 30),
    baselineAvg: null,
    balancedLow: null,
    balancedHigh: null,
    zScore: null,
    status: "unknown",
  }));
}
```

The three fixture views that overwrite `days[days.length - 1]` with an object literal (`suppressedView`, `balancedView`, and any sibling doing the same) also need the five fields. For each such literal, e.g.

```ts
  days[days.length - 1] = { date: FIXTURE_TODAY, hrv: 74, restingHr: 51, recoveryScore: 58 };
```

extend it to:

```ts
  days[days.length - 1] = {
    date: FIXTURE_TODAY,
    hrv: 74,
    restingHr: 51,
    recoveryScore: 58,
    baselineAvg: null,
    balancedLow: null,
    balancedHigh: null,
    zScore: null,
    status: "unknown",
  };
```

Keep each literal's own `hrv`/`restingHr`/`recoveryScore` values. **Do not add `baselineTrend` to the fixture views** — it is optional and nothing renders it.

- [ ] **Step 6: Run the tests to verify they pass**

Run:
```bash
cd /workspace/prog-strength-web
npm run typecheck
npm run test
```
Expected: typecheck clean, all vitest suites PASS.

- [ ] **Step 7: Run lint and format, then commit**

```bash
cd /workspace/prog-strength-web
npm run lint
npm run format:check   # if this fails, run `npm run format` and re-check
git add lib app
git commit -m "feat(dashboard): carry per-day recovery bands and baseline trend through the adapter"
```

---

## Task 9: Round the HRV headline on `HrvBalanceCard`

**Files:**
- Modify: `app/(app)/dashboard/_components/recovery/balance-band.tsx:74`
- Modify: `app/(app)/dashboard/_components/recovery/balance-band.test.tsx`

- [ ] **Step 1: Write the failing test**

Append to `/workspace/prog-strength-web/app/(app)/dashboard/_components/recovery/balance-band.test.tsx`, inside the existing top-level `describe` (read the file first and mirror its render helper and import list exactly — it already imports `HrvBalanceCard` and the fixture builders):

```ts
  it("renders integer milliseconds, never the raw float", () => {
    const view = balancedView();
    view.hrvToday = 77.39185;
    view.days = view.days!.map((d, i) =>
      i === view.days!.length - 1 ? { ...d, hrv: 77.39185 } : d,
    );

    render(<HrvBalanceCard section={view} href="/recovery" />);

    expect(screen.getByText("77")).toBeInTheDocument();
    expect(screen.queryByText(/77\.39185/)).toBeNull();
  });
```

Adjust the render call and matchers to whatever idiom the file already uses (some suites in this repo use a `renderCard` helper or `container.textContent` assertions rather than `screen.getByText`). The two things that must be asserted either way: the string `77` appears as the headline number, and the string `77.39185` appears nowhere in the card.

- [ ] **Step 2: Run the test to verify it fails**

Run: `cd /workspace/prog-strength-web && npx vitest run "app/(app)/dashboard/_components/recovery/balance-band.test.tsx" 2>&1 | tail -25`
Expected: FAIL — the card renders `77.39185`.

- [ ] **Step 3: Round it**

In `/workspace/prog-strength-web/app/(app)/dashboard/_components/recovery/balance-band.tsx`, inside the headline `<span className="font-mono text-xl …">`, change:

```tsx
              {todayVal}
```

to:

```tsx
              {Math.round(todayVal)}
```

That is the whole change. Do not touch the caption (it already rounds), the polyline, the band rect, or the sibling tiles — `morning-ledger.tsx`, `three-dial-vitals.tsx`, and `trend-rail.tsx` round their HRV figures already.

- [ ] **Step 4: Run the tests to verify they pass**

Run: `cd /workspace/prog-strength-web && npm run test 2>&1 | tail -25`
Expected: all PASS, including the pre-existing tests in `balance-band.test.tsx`, which must not be edited.

- [ ] **Step 5: Run the full web gate**

Run:
```bash
cd /workspace/prog-strength-web
npm run lint && npm run typecheck && npm run format:check && npm run test
```
Expected: all green.

- [ ] **Step 6: Commit**

```bash
cd /workspace/prog-strength-web
git add "app/(app)/dashboard/_components/recovery"
git commit -m "fix(dashboard): round HRV to integer milliseconds on the balance tile"
```

---

## Final verification (run before opening any PR)

- [ ] **`prog-strength-api` full gate**

```bash
cd /workspace/prog-strength-api
go build ./...
go vet ./...
go mod tidy && git diff --exit-code go.mod go.sum
go run github.com/golangci/golangci-lint/v2/cmd/golangci-lint@v2.12.2 fmt --diff
go run github.com/golangci/golangci-lint/v2/cmd/golangci-lint@v2.12.2 run
go test ./...
```

- [ ] **`prog-strength-web` full gate**

```bash
cd /workspace/prog-strength-web
npm run lint && npm run typecheck && npm run format:check && npm run test
```

- [ ] **Goal audit against the SOW**

- Every `days[]` entry carries `baseline_avg`, `balanced_low`, `balanced_high`, `z_score`, `status` — Task 6.
- `baseline_trend` reports `direction`, `delta_ms`, `from_avg`, `over_days` from baselines only — Task 3.
- Scalar `baseline`/`hrv` bit-identical — `TestBuildWhoop_ScalarBlocksUnchangedByLeadIn`.
- `days[last]` agrees with the scalar `hrv` block — `TestComputeSeries_AgreesWithComputeOnToday` and `TestBuildWhoop_DaysCarryPerDayBands`.
- Two tunables in `config.toml` `[recovery]`, no env var, no secret — Task 2.
- Web type mirror carries the fields through as pass-through — Tasks 7–8.
- `HrvBalanceCard` prints integer milliseconds — Task 9.
- Recovery read stays one indexed query — `handler.go` still makes exactly one `ListRange` call.
- Non-goals respected: no tile redesign, no `/recovery` change, no new endpoint, no mobile change, no backfill, no rolling baselines for RHR/recovery score, no configurable chart window.
