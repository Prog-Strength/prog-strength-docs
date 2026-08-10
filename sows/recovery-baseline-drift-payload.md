---
status: shipped
repos:
  - prog-strength-api
  - prog-strength-web
  - prog-strength-docs
---

# Recovery Baseline Drift Payload

**Status**: Shipped · **Last updated**: 2026-08-10

## Introduction

[`sows/dashboard-recovery-metrics-payload.md`](./dashboard-recovery-metrics-payload.md)
gave the dashboard a per-user HRV baseline, and the `hrv_balance` tile draws it:
a filled zone behind thirty nights, labelled *your balanced range 68–108 ms*.
The band is honest and it is the user's own. It is also **frozen**.

The API emits exactly one pair of bounds — today's baseline ± 1 of today's
standard deviations — and the tile has no choice but to stretch that single
rectangle across the whole month of history behind it. The picture that produces
asserts something false: that this athlete's normal range thirty days ago was
identical to their normal range this morning. It never is. A baseline that has
climbed 6 ms over four weeks is an adaptation showing up in the data; one that
has sunk 8 ms is a reason to look at sleep, load, or illness. **Neither is
expressible in the payload today**, so neither reaches the user.

That absence is the more expensive half of the metric. "Was last night normal
for me?" is answered well already. "Is my normal changing?" — the question that
distinguishes a training block that is working from one that is grinding the
athlete down — is not answered at all, and cannot be answered by any amount of
client work, because the client is not permitted to recompute a server figure
and could not do it correctly from a 31-day window regardless.

There is a second, quieter gap. Every day in the series is currently
uninterpreted: `days[]` carries three raw metrics and nothing else, so a client
that wants to colour a night by whether it was in-range must either re-derive
the whole band itself or colour every night against *today's* band — measuring
July against an August baseline. Garmin does not do this; its HRV Status chart
colours each dot against the band as it stood on that dot's own morning.

This SOW closes both gaps in `internal/recoverytrend`. It adds a rolling pass
that gives **every day in the series its own trailing baseline**, and a
`baseline_trend` block that reports the baseline against **its own past**. It
does no tile design work: the redesign is
[`dx/hrv-balance-tile`](../dx/hrv-balance-tile.md), and the tile that ships from
it is a separate SOW. What changes for the user when this alone ships is one
visible thing — the tile stops printing `77.39185 ms` — plus the data a
drifting-band tile needs to exist at all.

## Proposed Solution

`internal/recoverytrend` grows one method beside `Compute`. Where `Compute`
answers *"how does today stand against the trailing window?"*, `ComputeSeries`
answers that question **once per day across the whole series**, walking the
window and computing each day's baseline from the thirty days that preceded
*it* — the same exclude-the-day-itself rule, applied thirty-one times. It
returns one result per charted day plus a `BaselineTrend`: today's baseline
against the baseline as it stood twenty-eight days earlier.

The engine stays pure. It gains no clock, no database, and no I/O; the two
entry points share their classification and band arithmetic through extracted
helpers, which is what makes the agreement invariant below structural rather
than aspirational.

The read path widens by one number. To give the *oldest* charted day a full
thirty-day trailing sample, the handler must fetch thirty days of lead-in before
the window it already fetches: **61 local dates instead of 31**, on the same
single indexed `ListRange` call. `buildWhoop` materialises that 61-date window,
hands the whole thing to `ComputeSeries`, and hands its **last 31 dates** —
byte-for-byte the window it builds today — to `Compute`. The scalar block is
therefore computed from exactly the same sample as before: `hrv_avg`,
`balanced_low`, `balanced_high`, `z_score`, `status`, and `trend` do not move by
an epsilon.

On the wire everything is additive. `days[]` entries gain five keys, the section
gains a `baseline_trend` object, and `today`, `resting_hr_spark`, `baseline`,
and `hrv` are untouched. Every shipped consumer — the five recovery tiles, the
`/recovery` page — keeps working with no change.

The one deliberate scope addition is a two-line fix in `prog-strength-web`:
`HrvBalanceCard` renders `todayVal` raw, so the shipped tile prints HRV to five
decimal places. It is the most conspicuous defect on the dashboard, this SOW is
already in that file's type dependencies, and the redesign that would otherwise
fix it is two dispatches away. It gets rounded now.

## Goals and Non-Goals

### Goals

- Every entry in `days[]` carries its own trailing baseline: `baseline_avg`,
  `balanced_low`, `balanced_high`, `z_score`, and `status`, each null (status
  `unknown`) until that day has `min_baseline_days` samples behind it.
- A `baseline_trend` block reports the baseline against its own past —
  `direction`, `delta_ms`, `from_avg`, `over_days` — derived **only** from
  baselines, never from `short_avg`.
- The scalar `baseline` and `hrv` blocks are bit-identical to what ships today
  for the same input data, proven by a regression test.
- `days[last]` agrees exactly with the scalar `hrv` block — same status, same
  bounds — enforced by test, not by convention.
- The two new tunables live in `config.toml` under `[recovery]` as non-secret
  literals; no new environment variable, no new GitHub secret.
- The web type mirror (`lib/api.ts`, `lib/dashboard.ts`) carries the new fields
  through the adapter as pass-through server figures.
- `HrvBalanceCard` prints integer milliseconds.
- The recovery read stays one indexed query per request.

### Non-Goals

- **No tile redesign.** The scatter, the drifting band, and the per-dot
  colouring are [`dx/hrv-balance-tile`](../dx/hrv-balance-tile.md) and the SOW
  that follows it. This work only makes them possible. `HrvBalanceCard` keeps
  its polyline and its flat band until then; the only change to it is rounding.
- **No `/recovery` deep-page change.** The page reads the scalar blocks and
  keeps doing so.
- **No new endpoint.** Everything lands inside the existing
  `GET /dashboard/summary` recovery section.
- **No mobile change.** `prog-strength-mobile` does not render the recovery
  tile; the additive payload is inert there.
- **No backfill.** Every figure is derived at read time from
  `whoop_recovery` rows that already exist. There is no new stored state.
- **No rolling baselines for resting HR or recovery score.** HRV is the metric
  with a band; the other two have averages and that is enough. Adding them would
  triple the payload growth for no surface that consumes it.
- **No configurable chart window.** The series length stays
  `baseline_window_days + 1`, exactly as `days[]` is sized today. A second
  window knob would let the series and the baseline window disagree, and nothing
  needs that.

## Implementation Details

### The trend engine: `internal/recoverytrend`

Three extractions and one new method. The extractions come first because they
are what guarantee the two entry points can never disagree.

**1. Extract the band arithmetic and the classification.** `Compute` currently
inlines both. Pull them out so `ComputeSeries` cannot drift:

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

`Compute`'s body changes to call them and keeps every other line:

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

**2. Two new value types**, alongside `Day`, `Baseline`, and `HRV`:

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

**3. `ComputeSeries`.** It takes the *wide* window — `BaselineWindowDays` of
lead-in followed by the charted days — and returns one `DayResult` per charted
day plus the drift:

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

`Config` gains the two fields, and `doc.go` gains a `# The series and the drift`
section describing the rolling window and why the drift question is distinct
from `HRV.Trend`.

```go
	BaselineDriftDays int     // how far back the baseline is compared against
	BaselineDriftZ    float64 // |delta| must exceed this many SDs to read rising/falling
```

### Algorithms

**Per-day baseline.** For charted day *i*, over input window `days`:

```
sample_i      = { days[j].HRV : i − baseline_window_days ≤ j < i, HRV ≠ nil }
baseline_i    = mean(sample_i)                     when |sample_i| ≥ min_baseline_days
sd_eff_i      = max(stddev_pop(sample_i), min_std_dev_ms)
balanced_low  = baseline_i − balanced_z × sd_eff_i
balanced_high = baseline_i + balanced_z × sd_eff_i
z_i           = (days[i].HRV − baseline_i) / sd_eff_i    when days[i].HRV ≠ nil
```

The upper bound is exclusive (`j < i`) — day *i* is never a member of the sample
it is measured against. This is the same rule `Compute` already applies to today
and the reason the input needs lead-in: without it, the oldest charted days
would be measured against a truncated sample and the band's left edge would show
a spurious wobble that is an artefact of the window, not the athlete.

**Baseline drift.** Both endpoints are baselines, never readings:

```
delta     = baseline[last] − baseline[last − baseline_drift_days]
direction = rising   when delta >  baseline_drift_z × sd_eff_last
            falling  when delta < −baseline_drift_z × sd_eff_last
            steady   otherwise
```

Two choices worth recording. **The threshold is SD-relative, not a fixed
millisecond count or a percentage**, because every other threshold in this
engine is: a 6 ms move is a real signal for an athlete whose spread is 8 ms and
noise for one whose spread is 25 ms, and a fixed threshold would hand the
narrow-spread athlete a verdict the wide-spread athlete could never earn. A
percentage was considered and rejected for the same reason — it scales with the
*level* of the baseline, which is not what determines whether a move is
meaningful.

**`baseline_drift_days` must be strictly less than the series length**
(`baseline_window_days`, since the series is that plus one). At the defaults,
28 < 30. Overshooting is not a crash: `drift` returns `unknown`, which is the
correct honest answer for "I cannot see that far back". It is documented in
`config.toml` and covered by a test rather than enforced by a loader validation
— the config block has no validator today and this SOW does not add the first
one.

**Why `lastSDEff`.** The threshold uses the *most recent* day's effective SD, so
the verdict is scaled to the athlete's spread as it stands now. Using the older
endpoint's SD would scale the judgement to a spread they have since grown out
of; averaging the two would make the threshold depend on a quantity neither
endpoint reports. When no charted day ever reached `min_baseline_days`,
`lastSDEff` is zero and the direction is `unknown` — the same state the bounds
are already in.

### Read Path

`internal/dashboard/handler.go`, `buildRecoverySection` — the fetch widens to
carry the lead-in. Nothing else about the function changes:

```go
	// TWO baseline windows plus today: baseline_window_days of lead-in so the
	// OLDEST charted day still has a full trailing sample, then the charted
	// window itself (61 dates at the defaults). Same single indexed ListRange
	// call as before — idx_user_whoop_recovery_user_date covers (user_id,
	// date DESC) — the read just returns ≤61 rows instead of ≤31.
	win := h.recovery.BaselineWindowDays()
	sinceStr := now.In(loc).AddDate(0, 0, -2*win).Format("2006-01-02")
```

`internal/dashboard/whoop.go`, `buildWhoop` — the window materialisation moves
up and widens, and the two engine calls are fed from it. The existing
`section.Days` loop and the separate `engineDays` copy loop collapse into one
pass, which is a simplification the widening pays for:

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

	section.BaselineTrend = RecoveryBaselineTrend{
		Direction: drift.Direction,
		DeltaMs:   drift.DeltaMs,
		FromAvg:   drift.FromAvg,
		OverDays:  drift.OverDays,
	}
```

The `Today` row and the `RestingHRSpark` loop above are untouched, and the
`Baseline`/`HRV` assignment blocks below are untouched.

**Cost.** One indexed range scan returning ≤61 rows instead of ≤31 — the same
query, a wider bound. The rolling pass is 31 × 30 ≈ 930 float operations per
request, which is beneath measurement next to the eleven other section builds.
The recovery section's JSON grows by five keys × 31 days plus one small object,
roughly 2 KB uncompressed.

### Data Model

No schema change. Every figure is derived at read time from `whoop_recovery`
rows that already exist, and no new state is stored. The wire types in
`internal/dashboard/dto.go` change as follows.

**New — `RecoveryDayPoint`.** `Days` switches from `[]RecoveryDay` to
`[]RecoveryDayPoint`. The embedded struct carries no JSON tag, so Go flattens
its fields into the same object and the four existing keys stay exactly where
they are — `Today` keeps its own `RecoveryDay` type and its wire shape is
byte-for-byte unchanged.

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

// RecoveryBaselineTrend is the baseline against its own past. NOT the same
// question as RecoveryHRV.Trend, which compares the recent mean against the
// window it sits inside; these two may point opposite ways.
type RecoveryBaselineTrend struct {
	Direction string   `json:"direction"` // rising | falling | steady | unknown
	DeltaMs   *float64 `json:"delta_ms"`
	FromAvg   *float64 `json:"from_avg"`
	OverDays  int      `json:"over_days"`
}
```

`RecoverySection` gains one field and re-types one:

```go
	Days          []RecoveryDayPoint    `json:"days"`
	BaselineTrend RecoveryBaselineTrend `json:"baseline_trend"` // always present
}
```

`baseline_trend` is a value, not a pointer — like `Baseline` and `HRV`, it is
always present and expresses "nothing to say" as `direction: "unknown"` rather
than as an absent key.

### API Surface

`GET /dashboard/summary` — unchanged path, method, and auth. The `recovery`
section is additive:

```jsonc
"recovery": {
  "today": { /* unchanged */ },
  "resting_hr_spark": [ /* unchanged */ ],
  "days": [
    {
      "date": "2026-08-09",
      "resting_heart_rate": 59,
      "recovery_score": 61,
      "hrv_rmssd_milli": 77.4,
      // NEW — this day's own band; null until it has min_baseline_days behind it
      "baseline_avg":  88.2,
      "balanced_low":  68.1,
      "balanced_high": 108.3,
      "z_score":       -0.53,
      "status":        "balanced"
    }
  ],
  "baseline": { /* unchanged */ },
  "hrv":      { /* unchanged */ },
  // NEW
  "baseline_trend": {
    "direction": "rising",
    "delta_ms":  6.4,
    "from_avg":  81.8,
    "over_days": 28
  }
}
```

No consumer is required to read the new keys, and none breaks by ignoring them.

### Config

`config.toml`, appended to the existing `[recovery]` block. Non-secret public
literals — no `${VAR}` interpolation, no env override, no GitHub secret:

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
baseline_drift_days = 28
baseline_drift_z    = 0.35
```

Threaded through in three places, following the existing pattern exactly:
`internal/config/config.go` — the `RecoveryConfig` struct (line ~268), the
`toml:"recovery"` file struct (line ~465), and the `Recovery: RecoveryConfig{…}`
mapping (line ~666) — then `internal/server/server.go` (line ~823) where
`recoverytrend.New` is constructed. `internal/dashboard/whoop_test.go`'s
`testRecoveryEngine` and `internal/activity/contract_test.go`'s engine literal
both need the two new fields so they keep matching production defaults.

### Web type mirror (`prog-strength-web`)

**`lib/api.ts`** — `DashboardRecoveryDay` is the raw wire type for both `today`
and `days[]`. Since only `days[]` gains fields, add a separate type rather than
widening the shared one:

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

and on `DashboardRecovery`: `days: DashboardRecoveryDayPoint[]` plus
`baseline_trend: DashboardRecoveryBaselineTrend`.

**`lib/dashboard.ts`** — `RecoveryDayPoint` gains the five fields,
`RecoveryBaselineTrendView` is new, `RecoveryView` gains
`baselineTrend?: RecoveryBaselineTrendView`, and `adaptRecovery` passes
everything through. It reuses the existing `recoveryStatus` and `recoveryTrend`
narrowers rather than adding new ones — the vocabularies are identical:

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
    baselineTrend: {
      direction: recoveryTrend(recovery.baseline_trend.direction),
      deltaMs: recovery.baseline_trend.delta_ms,
      fromAvg: recovery.baseline_trend.from_avg,
      overDays: recovery.baseline_trend.over_days,
    },
```

`baselineTrend` is typed optional to match its `days?`/`baseline?`/`hrv?`
neighbours, so consumers keep guarding rather than `!`-asserting. Nothing
renders it yet.

### The rounding fix

`app/(app)/dashboard/_components/recovery/balance-band.tsx:74` renders
`{todayVal}` — a raw Whoop RMSSD float, hence `77.39185 ms` on the shipped tile.
The caption below it already rounds. One change, matching the caption:

```tsx
              {Math.round(todayVal)}
```

Its sibling tiles are correct already and need no change: `morning-ledger.tsx`
and `three-dial-vitals.tsx` round their HRV figures, and `trend-rail.tsx` uses
`signedUnit`, which rounds. A test asserts the tile renders `77` and not
`77.39185` for a fixture carrying the float.

### Backfill or Migration

**None, by construction.** Every figure is derived at read time from
`whoop_recovery` rows already in the database — the same rows the scalar
baseline reads today, over a wider date bound. There is no derived table, no new
column, and nothing to populate.

**Recoverability** is therefore trivial: a bad deploy is a rollback, and the
next request recomputes from the source rows. A user whose history does not
reach back 61 days is not a failure case — the missing dates materialise with
null metrics, the oldest charted days report a null baseline and
`status: "unknown"`, and `baseline_trend.direction` is `unknown` until the
series has both endpoints. This is the `partial-band` state the DX enumerates,
and it resolves itself as the Whoop webhook accumulates mornings.

**Scale boundary.** The read is bounded by 61 rows per user per request, so
nothing here scales with total history. The strategy would need revisiting only
if `baseline_window_days` grew past roughly a year, at which point the read
(≤731 rows) and the O(n²) rolling pass (~133k operations) would justify caching
the series or computing baselines incrementally. Neither is close.

### Testing

**`internal/recoverytrend/recoverytrend_test.go`** — all new tests build their
window with the existing helpers so they read like their neighbours:

- `TestComputeSeries_LengthAndLeadIn` — a 61-day window returns exactly 31
  results whose dates are the last 31 input dates, in order.
- `TestComputeSeries_PerDayBaselineExcludesTheDayItself` — a window where one
  day is a large outlier: that day's own `BaselineAvg` is unaffected by it,
  while the *next* day's baseline moves. This is the rule that makes the whole
  thing honest and it is worth a dedicated test.
- `TestComputeSeries_AgreesWithComputeOnToday` — **the invariant.** Build the
  61-day window, call `ComputeSeries(days)` and `Compute(days[30:])`, and assert
  `series[len−1].BaselineAvg == *baseline.HRVAvg`, `.BalancedLow`,
  `.BalancedHigh`, and `.ZScore` equal the scalar block's, and
  `series[len−1].Status == hrv.Status`. Exact float equality, not epsilon — they
  are the same arithmetic over the same sample and must be bit-identical.
- `TestComputeSeries_ShortHistoryLeavesEarlyDaysUnknown` — a window whose first
  charted days have fewer than `min_baseline_days` non-null samples behind them:
  those results carry nil bounds and `StatusUnknown`, and the later ones are
  populated. Asserts the band-starts-part-way-across state.
- `TestComputeSeries_NullDayKeepsBandDropsZ` — a charted day with nil HRV but a
  full trailing sample: `BaselineAvg`/bounds populated, `ZScore` nil, `Status`
  unknown. A missing morning must not erase the band that morning sat in.
- `TestDrift_RisingFallingSteady` — three windows whose baselines climb, sink,
  and hold; asserts direction, and that `DeltaMs`/`FromAvg` are populated and
  `OverDays == 28` in all three.
- `TestDrift_UnknownWhenSeriesTooShort` — `BaselineDriftDays` set to 40 against
  a 31-day series: direction `unknown`, `DeltaMs` and `FromAvg` nil,
  `OverDays == 40`. The documented misconfiguration degrades honestly.
- `TestDrift_UnknownWhenBaselineAbsent` — an all-null-HRV window: direction
  `unknown`, no panic, no division by a zero SD.
- `TestDrift_IsNotShortAvg` — a window where the 7-day mean is falling while the
  baseline is rising; asserts `hrv.Trend == falling` **and**
  `drift.Direction == rising` from the same input. Pins the semantic difference
  the whole SOW rests on.
- `TestCompute_UnchangedAfterExtraction` — the existing `Compute` tests all
  still pass unmodified, which is the regression guard on the `band`/`classify`
  extraction. No new test needed; the requirement is that **no existing
  assertion in this file is edited**.

**`internal/dashboard/whoop_test.go`:**

- `TestBuildWhoop_ScalarBlocksUnchangedByLeadIn` — the strongest regression
  test in the set. Run `buildWhoop` over a fixture, record `Baseline` and `HRV`,
  then add rows *older than the charted window* and run again; every scalar
  field must be identical. This is what proves the widened fetch did not move
  the numbers the shipped tiles print.
- `TestBuildWhoop_DaysCarryPerDayBands` — `len(section.Days) == 31` (unchanged),
  each entry's `Date` is unchanged, and the last entry's band equals
  `section.HRV`'s bounds and status.
- `TestBuildWhoop_LeadInNotSerialized` — dates older than `win` days before
  today never appear in `section.Days`, even when rows exist for them.
- `TestBuildWhoop_BaselineTrendPresentWhenNoEntries` — no rows at all:
  `BaselineTrend.Direction == "unknown"`, `OverDays == 28`, and the section is
  still non-nil. The connected-but-empty user must not get a missing key.
- `TestRecoverySection_JSONKeys` — extend the existing key assertions with
  `baseline_trend` and its four keys, and with the five new `days[]` keys.
  Explicitly assert `today` still has **exactly** its original four keys — the
  embedded-struct approach is correct but it is precisely the kind of thing that
  silently leaks fields if someone later moves the embed.

**`internal/config/config_test.go`** — extend the existing default-values
assertions (lines ~618 and ~733) with `BaselineDriftDays == 28` and
`BaselineDriftZ == 0.35`, matching the pattern of the seven knobs already there.

**`prog-strength-web`:**

- `lib/dashboard.test.ts` — `adaptRecovery` maps the five new day fields and the
  `baseline_trend` block, and an unrecognised `status`/`direction` string
  narrows to `"unknown"` through the existing narrowers rather than leaking
  through.
- `app/(app)/dashboard/_components/recovery/balance-band.test.tsx` — a fixture
  with `hrv: 77.39185` renders `77` and the string `77.39185` is absent from the
  card. The existing tests in this file must keep passing untouched.

### Documentation

- `internal/recoverytrend/doc.go` gains **`# The series and the drift`**,
  covering the rolling window, the lead-in requirement, the exclude-the-day
  rule, and — at length — why `BaselineTrend` and `HRV.Trend` answer different
  questions and may disagree. The package doc is the only place that distinction
  is written down for the next reader.
- `internal/dashboard/dto.go` doc comments on both new types, as drafted above.
- The `[recovery]` comment block in `config.toml`, as drafted above, including
  the `baseline_drift_days < baseline_window_days` constraint.
- [`dx/hrv-balance-tile.md`](../dx/hrv-balance-tile.md)'s *The API refactor*
  section is the design record for this work; no update needed unless the shape
  below changes during review.

## Open Questions

1. **Should `baseline_trend` also cover resting HR?** A drifting resting-HR
   baseline is arguably as informative as a drifting HRV baseline, and the
   averages are already computed. Options: add a parallel block now; add it when
   a surface asks for it; never. **Tentative lean: when a surface asks.** No tile
   in [`dx/hrv-balance-tile`](../dx/hrv-balance-tile.md) consumes it, and a
   payload field with no reader is a field that rots. Listed as a non-goal above
   on that basis, and cheap to add later since `ComputeSeries` would only need a
   second metric selector.

2. **Should the config loader validate `baseline_drift_days < baseline_window_days`?**
   Today the misconfiguration degrades to `direction: "unknown"` forever, which
   is honest but silent — nobody would notice for months. Options: hard-fail at
   boot; log a warning at engine construction; leave it to the documented
   constraint and the test. **Tentative lean: leave it.** The `[recovery]` block
   has no validator at all today and adding the first one for this knob sets an
   inconsistent precedent — either every recovery tunable gets bounds-checked or
   none does, and that is its own small SOW.

3. **Should the API round HRV to integer milliseconds on the wire?** It would
   have prevented the `77.39185` bug at the source and every consumer wants
   integers. Options: round server-side; keep floats and round at each display
   site. **Tentative lean: keep floats.** Precision belongs to the data and
   formatting belongs to the client — the baseline arithmetic genuinely wants
   the decimals, and rounding on the wire would make `hrv_avg` and the band
   bounds disagree with a client that rounds differently. The display fix above
   is the right layer.
