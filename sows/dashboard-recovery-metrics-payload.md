---
status: shipped
repos:
  - prog-strength-api
  - prog-strength-web
  - prog-strength-docs
---

# Dashboard Recovery Metrics Payload

**Status**: Shipped · **Last updated**: 2026-08-02

## Introduction

The dashboard's Recovery tile is the only surface on the grid whose data is
*richer than what it shows*. Whoop gives **Prog Strength** three daily numbers —
recovery score, resting heart rate, and HRV (RMSSD) — and the tile renders one
and a half of them: a resting-HR headline, a normalized sparkline of recent
resting HR, and the recovery score as a footnote. **HRV never reaches the
browser at all.** It is fetched, stored, serialized into
`GET /dashboard/summary`, and then dropped on the floor by the web adapter
(`adaptRecovery` in `lib/dashboard.ts` reads `today.resting_heart_rate` and
`today.recovery_score` and ignores `today.hrv_rmssd_milli`).

That matters because HRV is the metric a recovery surface exists to tell a story
about, and a single day's HRV is close to meaningless on its own. Whether 88 ms
is a good morning or a warning sign depends entirely on what *this athlete's*
HRV usually is — one person's balanced day is another's worst week. The reading
only becomes information when it is placed against the user's own baseline and
their own day-to-day variability. Neither exists in the payload today: there is
no HRV history, no baseline for any metric, and no measure of spread.

The same gap makes the resting-HR headline weaker than it looks. "51 bpm" is a
number; "51 bpm against your 30-day average of 54" is a fact about the athlete.
The tile currently shows the first and leaves the second to the user's memory.

This SOW does none of the tile's design work. It closes the data gap underneath
it: extend the recovery section of `GET /dashboard/summary` with a date-aligned
daily history, per-metric baselines computed from the user's own record, and an
HRV balance assessment whose thresholds are derived from that user's own
variability rather than a population constant. Once it ships, a design
exploration of the Recovery tile — and the tiles that come out of it — can draw
a trend, place today against a baseline, and label a day *balanced*,
*elevated*, or *suppressed* using numbers the server computed once and every
client reads identically.

## Proposed Solution

The recovery section grows three additive fields — `days`, `baseline`, and
`hrv` — while `today` and `resting_hr_spark` stay byte-for-byte as they are, so
the shipped `RecoveryCard` and any other consumer keep working untouched.

`days` is the honest per-day history the current sparkline cannot be: the
trailing local-date window, **oldest→newest, with missing days present and their
metrics null**. (Today's `resting_hr_spark` silently *omits* days without a
reading, which shortens the array and destroys date alignment — a per-day trend
drawn from it would misplace every point.) `baseline` carries the trailing
averages — resting HR, HRV, recovery score — plus the HRV standard deviation and
the sample count behind each figure, so a client can render an honest
"calibrating" state instead of a confident number backed by four days of data.
`hrv` is the interpretation: today's balance status, the numeric bounds of the
balanced band so a chart can *draw* it, the z-score behind the verdict, and a
near-term trend direction.

The math lives in a new self-contained package, **`internal/recoverytrend`**,
built the way [`internal/hrzones`](./running-heart-rate-zones.md) is: pure
functions over a `Config` struct, no `internal/config` import, no clock, no
database. Its tunables — window lengths, minimum sample sizes, the balance
threshold — are non-secret and belong in a version-controlled `[recovery]` block
of `config.toml` alongside `[hr_zones]`, not in the environment.

The thresholds that separate *balanced* from *unbalanced* are **derived from the
user, not configured by them**. There is no new setting, no new persisted
preference, and no settings-page work: the band is the user's own HRV baseline
plus or minus a multiple of their own standard deviation, so an athlete whose
HRV swings 25 ms night to night gets a wide band and a metronomic athlete gets a
narrow one, automatically, from day fifteen onward.

## Goals and Non-Goals

### Goals

- Add `days`, `baseline`, and `hrv` to the recovery section of
  `GET /dashboard/summary`, leaving `today` and `resting_hr_spark` unchanged.
- Emit a date-aligned daily history in which a day with no Whoop reading is
  present with null metrics, never omitted and never zero-filled.
- Compute trailing baselines for resting HR, HRV, and recovery score over a
  window that **excludes today**, reporting the sample count behind each.
- Classify today's HRV as `balanced` / `elevated` / `suppressed` / `unknown`
  against per-user thresholds derived from that user's own HRV standard
  deviation, and emit the band bounds so a client can draw them.
- Report a near-term HRV trend (`rising` / `falling` / `steady` / `unknown`)
  comparing the recent mean against the baseline.
- House every tunable in a new `[recovery]` block in `config.toml` — public
  literals, no env vars, no secrets.
- Mirror the new payload in `prog-strength-web`'s `lib/api.ts` types and pass it
  through `adaptRecovery` into `RecoveryView`, including today's HRV, which the
  adapter drops today.
- Degrade honestly at every sample size: a user with three days of Whoop data
  gets nulls and `unknown`, never a fabricated baseline or a `NaN` band.

### Non-Goals

- **No tile or page design.** The shipped `RecoveryCard` renders exactly as it
  does now. Consuming the new fields is the job of the downstream Recovery tile
  DX and the SOW it feeds.
- **No new Whoop sync.** Nothing new is fetched from Whoop and no migration is
  written; every field here is derived on read from
  `user_whoop_recovery` rows that already exist.
- **No day-average heart rate.** Whoop's cycle-level average HR is not synced or
  stored (migration `040_user_whoop_recovery.sql` persists score, resting HR,
  and HRV only). "Average heart rate" in this SOW means the **trailing average
  resting HR**. Syncing cycle-level HR is a separate SOW.
- **No user-configurable thresholds.** Thresholds vary per user because they are
  computed from that user's data — not because the user sets them. No settings
  surface, no new preference column.
- **No changes to `/whoop/recovery`.** The deep recovery page's endpoint is
  untouched; this is dashboard-summary work.
- **No mobile work.** `prog-strength-mobile` does not consume the recovery
  section.
- **No agent or MCP exposure.** The training snapshot does not gain HRV balance
  here (see Open Questions).

## Implementation Details

### The trend engine: `internal/recoverytrend`

A new package with no dependencies on the rest of the codebase beyond the
`whooprecovery` entry type. It exposes a `Config`, a constructor, and one entry
point that turns a window of daily entries into the derived block:

```go
// Config groups the recovery-trend tunables. All are non-secret public
// literals sourced from the [recovery] block of config.toml.
type Config struct {
    BaselineWindowDays int     // trailing days, EXCLUDING today, behind every average
    MinBaselineDays    int     // samples required before an average is emitted
    TrendWindowDays    int     // recent days, INCLUDING today, behind the trend
    MinTrendDays       int     // samples required in that window before a trend is emitted
    BalancedZ          float64 // |z| within this many SDs of baseline reads as balanced
    TrendZ             float64 // recent mean must sit this many SDs off baseline to read rising/falling
    MinStdDevMs        float64 // SD floor, so a near-flat history cannot divide by ~0
}

type Engine struct{ cfg Config }

func New(cfg Config) *Engine

// Compute derives the baseline and HRV blocks from a date-ascending window of
// daily rows whose FINAL element is today. Pure: no clock, no DB, no I/O.
func (e *Engine) Compute(days []Day) (Baseline, HRV)
```

`Day`, `Baseline`, and `HRV` are the engine's **own** value types, not the
dashboard DTOs — `Day` is a `Date string` plus three `*float64` metrics, and the
two results carry the same fields as their DTO counterparts under Go-native
names. Keeping them distinct is what stops the package from coupling to the
summary payload: the dashboard package maps `whooprecovery.Entry` →
`recoverytrend.Day` on the way in and `recoverytrend.Baseline`/`HRV` →
`RecoveryBaseline`/`RecoveryHRV` on the way out, so a future change to the wire
format is a mapping edit rather than an engine edit.

Placing the math here rather than in `internal/dashboard` keeps it table-testable
in isolation and leaves it reusable by the deep recovery page or the agent later
without dragging the whole summary handler along.

### Data Model

**No schema change.** No migration, no new table, no new column. Every field in
this SOW is derived on read from existing `user_whoop_recovery` rows
(`recovery_score`, `resting_heart_rate`, `hrv_rmssd_milli`, one row per
`(user_id, date)`).

### Read Path

The only change to the fetch is the width of the window.
`buildRecoverySection` currently asks the repository for the trailing
`recoverySparkDays` (7) local days:

```go
sinceStr := now.In(loc).AddDate(0, 0, -(recoverySparkDays - 1)).Format("2006-01-02")
untilStr := now.In(loc).Format("2006-01-02")
```

It now asks for `baseline_window_days` days *before* today through today
inclusive — 31 local dates at the default:

```go
sinceStr := now.In(loc).AddDate(0, 0, -cfg.BaselineWindowDays).Format("2006-01-02")
untilStr := now.In(loc).Format("2006-01-02")
```

This is the same single indexed `ListRange` call
(`idx_user_whoop_recovery_user_date` covers `(user_id, date DESC)`), returning
at most 31 rows instead of at most 7. Everything else in the read path is
unchanged: the connected-Whoop gate still decides presence, and a failed
recovery read still degrades to a nil section rather than a 500.

`buildWhoop` gains the engine as a parameter and stays pure — `now` and `loc`
are still passed in, so the local-day window remains deterministic across
timezones and DST. It continues to build `today` and the 7-day
`resting_hr_spark` exactly as it does now, and additionally materializes the
full date-aligned `days` slice and hands it to `Engine.Compute`.

`recoverySparkDays` stays at 7 and keeps its current, gap-omitting semantics.
It is deliberately not "fixed" — the shipped card depends on it, and `days` is
the correct replacement for anything new.

### Algorithms

Let the window be the date-ascending slice `days`, whose final element is
today's local date. Every entry is present even when Whoop has no row for it;
absent metrics are null and are skipped, never treated as zero.

**Baseline sample.** All entries *except the last* — the
`baseline_window_days` local dates preceding today. Today is excluded so that a
day is measured against history rather than against a window it is a member of.
With a 30-day window one included day would shift its own baseline by ~3% of its
own deviation; small, but the framing "today vs your baseline" should mean what
it says. The same exclusion applies to the resting-HR and recovery-score
averages so a single sentence covers all three.

**Averages.** Per metric, the arithmetic mean over the non-null values in the
baseline sample. A metric with fewer than `min_baseline_days` non-null values
emits a **null average** and reports its real sample count, so a client can show
"calibrating, 9 of 14 days" instead of a confident number backed by nine
readings. Counts are always emitted, whether or not the average is.

**Spread.** Population standard deviation of the non-null HRV values in the
baseline sample (divide by `n`, not `n−1`). The sample is the athlete's actual
recorded history rather than a draw from a larger population, and at n≈30 the
two differ by under 2% — well below the resolution of the verdict it feeds.

```
hrv_avg      = mean(hrv values in baseline sample)
hrv_std_dev  = sqrt( mean( (x − hrv_avg)^2 ) )
sd_effective = max(hrv_std_dev, min_std_dev_ms)
```

**The balanced band.** `sd_effective` — not the raw SD — drives both the band
bounds and the z-score, so the emitted bounds and the emitted status can never
disagree:

```
balanced_low  = hrv_avg − balanced_z × sd_effective
balanced_high = hrv_avg + balanced_z × sd_effective
z_score       = (hrv_today − hrv_avg) / sd_effective
```

**Status.**

```
unknown     if hrv_avg is null (insufficient history) or hrv_today is null
balanced    if |z_score| ≤ balanced_z
elevated    if  z_score >  balanced_z
suppressed  if  z_score < −balanced_z
```

The status is **directional**: HRV well above baseline is a materially different
signal from HRV well below it, and collapsing both into "unbalanced" throws away
information a client cannot recover. A tile that wants a two-state chip can
merge `elevated` and `suppressed` at render time; the reverse is impossible.

At `balanced_z = 1.0` roughly two-thirds of a normally-distributed athlete's own
days land inside the band, which is the intent — *balanced* should describe a
typical morning, not a rare one.

**Trend.** The mean HRV over the last `trend_window_days` entries (including
today), compared against the same baseline:

```
short_avg = mean(non-null hrv in the last trend_window_days entries)
delta     = short_avg − hrv_avg
trend = unknown  if hrv_avg is null or the window holds < min_trend_days samples
        rising   if delta >  trend_z × sd_effective
        falling  if delta < −trend_z × sd_effective
        steady   otherwise
```

Note the windows **overlap**: the trailing 7 days sit inside the 30-day baseline
(all but today). The considered alternative was comparing the recent 7 days
against the *non-overlapping* prior 23 — statistically cleaner, since overlap
pulls the baseline toward the recent mean and dampens the measured delta. It was
rejected for two reasons. First, overlap dampens the signal but cannot invert it:
a genuinely falling week still reads as falling, just less dramatically, and the
`trend_z` threshold is calibrated against the dampened scale anyway. Second, it
would make the product tell two different stories with one word — the deep
recovery page's language is already "*X% below your 30-day baseline*"
([`dx/recovery-page.md`](../dx/recovery-page.md)), and a trend measured against a
23-day segment nobody named would not agree with the number printed beside it.
One baseline, one story.

**Degenerate and sparse cases.**

- *No rows at all* (connected user, nothing synced): `days` is a full window of
  null-metric entries, every average null with zero counts, status and trend
  `unknown`. The section still renders — the user is connected.
- *Today has no reading yet*: baselines and trend still compute and are still
  emitted; status is `unknown`, `z_score` null. Yesterday is never promoted into
  today — an absent morning webhook must not read as stale readiness.
- *Partial nulls*: a day with a score but no HRV contributes to the score
  average and not the HRV one. Counts are per metric, not per day.
- *Near-zero SD*: `min_std_dev_ms` floors the divisor, so an athlete with an
  improbably flat history gets a narrow-but-finite band rather than an infinite
  z-score.
- *Fewer than `min_baseline_days` HRV samples*: no HRV average, therefore no
  band, no z-score, status and trend `unknown`. The counts still ship so a
  client can render progress toward a usable baseline.

### Config

A new `[recovery]` block in `config.toml`, mapped through `Load` exactly the way
`[hr_zones]` is (a `RecoveryConfig` struct on `Config`, a matching `toml:"recovery"`
struct on the file shape, field-by-field assignment). Public literals: no
`${VAR}` interpolation, no env override, no secrets.

```toml
[recovery]
# Tunables for the dashboard recovery trend engine (internal/recoverytrend).
# Public literals — not secrets, not env-overridden. Thresholds are derived
# PER USER from their own HRV spread; these knobs shape that derivation, they
# are not population constants a user's numbers are compared against.
#
# baseline_window_days: trailing local days behind every average. EXCLUDES
#   today, so a day is measured against history, not a window it is inside.
# min_baseline_days: non-null samples required before an average is emitted at
#   all. Below it the payload reports the count and a null average — an honest
#   "calibrating" state rather than a confident number backed by a few nights.
# trend_window_days / min_trend_days: the near-term window (INCLUDING today)
#   whose mean is compared against the baseline, and its minimum sample count.
# balanced_z: half-width of the balanced band in the user's own standard
#   deviations. 1.0 puts ~two-thirds of a typical athlete's own days inside it
#   — "balanced" should describe an ordinary morning, not a rare one.
# trend_z: how far the recent mean must sit off baseline to read rising or
#   falling. Below balanced_z because a sustained 7-day shift is a stronger
#   signal than one morning's reading.
# min_std_dev_ms: floor on the standard deviation used as the divisor, so a
#   near-flat history cannot produce an unbounded z-score.
baseline_window_days = 30
min_baseline_days    = 14
trend_window_days    = 7
min_trend_days       = 4
balanced_z           = 1.0
trend_z              = 0.5
min_std_dev_ms       = 1.0
```

`server.go` builds the engine from `cfg.Recovery` beside the existing
`hrzones.New(...)` call and passes it into `dashboard.NewHandler` as a trailing
parameter. `NewHandler`'s positional argument list is already long; the engine is
mandatory rather than optional, so it goes in the constructor instead of a
`SetX` setter — a handler that could be constructed without it would serve a
half-built payload.

### API Surface

`GET /dashboard/summary` — the `recovery` section only. Additive: `today` and
`resting_hr_spark` are unchanged in shape, semantics, and content.

```go
type RecoverySection struct {
    Today          *RecoveryDay  `json:"today"`            // unchanged; nil when no row today
    RestingHRSpark []float64     `json:"resting_hr_spark"` // unchanged; 7d, gap-omitting
    Days           []RecoveryDay `json:"days"`             // NEW: full window, date-aligned, oldest→newest
    Baseline       RecoveryBaseline `json:"baseline"`      // NEW: value, always present
    HRV            RecoveryHRV      `json:"hrv"`           // NEW: value, always present
}

// RecoveryBaseline carries the trailing averages and the spread behind the HRV
// band. Averages are null until their metric has min_baseline_days samples; the
// *Days counts are always populated so a client can render calibration progress.
type RecoveryBaseline struct {
    WindowDays       int      `json:"window_days"`
    RestingHRAvg     *float64 `json:"resting_hr_avg"`
    RestingHRDays    int      `json:"resting_hr_days"`
    HRVAvg           *float64 `json:"hrv_avg"`
    HRVStdDev        *float64 `json:"hrv_std_dev"`
    HRVDays          int      `json:"hrv_days"`
    RecoveryScoreAvg *float64 `json:"recovery_score_avg"`
    RecoveryScoreDays int     `json:"recovery_score_days"`
}

// RecoveryHRV is today's HRV read against the user's own baseline. Bounds and
// z-score are derived from the same floored standard deviation, so they always
// agree with Status.
type RecoveryHRV struct {
    Status       string   `json:"status"`        // balanced | elevated | suppressed | unknown
    BalancedLow  *float64 `json:"balanced_low"`
    BalancedHigh *float64 `json:"balanced_high"`
    ZScore       *float64 `json:"z_score"`
    Trend        string   `json:"trend"`         // rising | falling | steady | unknown
    ShortAvg     *float64 `json:"short_avg"`
}
```

`Baseline` and `HRV` are struct **values**, not pointers, for the reason
`StreakSection` is: they are always meaningful. An athlete with no history has a
baseline of "nothing yet, here are the counts", which is a fact worth
serializing — clients then read one shape instead of branching on null.

`Days` reuses the existing `RecoveryDay` type, so a history entry and today's
entry are literally the same shape. At the defaults the slice holds **31
entries**: the 30 baseline dates plus today. The payload is self-describing —
every average in `baseline` is computed over `days[:len(days)-1]`, which a client
can verify.

Example (abridged to five days for legibility):

```jsonc
"recovery": {
  "today": { "date": "2026-08-01", "resting_heart_rate": 51, "recovery_score": 58, "hrv_rmssd_milli": 74 },
  // Note the gap-omitting legacy semantics: 07-30 has no reading and is simply
  // absent here — four values for five days. `days` below is the honest version.
  "resting_hr_spark": [53, 52, 51, 51],
  "days": [
    { "date": "2026-07-28", "resting_heart_rate": 53, "recovery_score": 71, "hrv_rmssd_milli": 92 },
    { "date": "2026-07-29", "resting_heart_rate": 52, "recovery_score": 74, "hrv_rmssd_milli": 95 },
    { "date": "2026-07-30", "resting_heart_rate": null, "recovery_score": null, "hrv_rmssd_milli": null },
    { "date": "2026-07-31", "resting_heart_rate": 51, "recovery_score": 66, "hrv_rmssd_milli": 81 },
    { "date": "2026-08-01", "resting_heart_rate": 51, "recovery_score": 58, "hrv_rmssd_milli": 74 }
  ],
  "baseline": {
    "window_days": 30, "resting_hr_avg": 52.4, "resting_hr_days": 27,
    "hrv_avg": 91.2, "hrv_std_dev": 12.6, "hrv_days": 26,
    "recovery_score_avg": 68.1, "recovery_score_days": 27
  },
  "hrv": {
    "status": "suppressed", "balanced_low": 78.6, "balanced_high": 103.8,
    "z_score": -1.37, "trend": "falling", "short_avg": 82.3
  }
}
```

Read as: today's 74 ms sits 1.37 of this athlete's own standard deviations below
their 91.2 ms baseline — outside their balanced band of 78.6–103.8 — and the
week is trending down, not just this morning.

### Web type mirror (`prog-strength-web`)

`lib/api.ts` — `DashboardRecovery` gains `days`, `baseline`, and `hrv`, mirroring
the Go DTO field-for-field in snake_case.

`lib/dashboard.ts` — `RecoveryView` gains the camelCase view-model, **including
today's HRV, which `adaptRecovery` currently discards**:

```ts
export type RecoveryHrvStatus = "balanced" | "elevated" | "suppressed" | "unknown";
export type RecoveryTrendDirection = "rising" | "falling" | "steady" | "unknown";

export type RecoveryDayPoint = {
  date: string;                 // YYYY-MM-DD, local
  restingHr: number | null;
  recoveryScore: number | null;
  hrv: number | null;           // ms
};

export type RecoveryView = {
  restingToday: number | null;
  recoveryScore: number | null;
  hrvToday: number | null;      // NEW — was fetched and dropped
  spark: number[];              // unchanged
  days: RecoveryDayPoint[];     // NEW — date-aligned, nulls preserved
  baseline: RecoveryBaselineView; // NEW
  hrv: RecoveryHrvView;         // NEW
};
```

`adaptRecovery` passes these through. Two adapter rules matter:

- **Never `sanitizeSpark` the day series.** `sanitizeSpark` maps non-finite
  values to `0`, which is right for a sparkline and catastrophic here — it would
  convert "no reading" into "a resting heart rate of zero". Nulls in `days` stay
  null; only `spark` keeps its existing sanitization.
- **Never recompute a server figure.** Averages, band bounds, and z-score are
  displayed as received, per the house convention that a server-computed average
  is a window figure a client must not re-derive from a partial series.

An unknown `status` or `trend` string from a future server maps to `"unknown"`
rather than throwing.

`RecoveryCard` and `RecoveryCardEmpty` are **not modified**. They keep reading
`restingToday`, `recoveryScore`, and `spark`, and render identically.

### Backfill or Migration

**None.** No schema change and nothing to backfill: every new field is derived
on read from rows the Whoop sync already writes. A user with 30 days of history
gets full baselines on the first request after deploy; a user with three days
gets `unknown` until their fifteenth.

**Recoverability** is therefore trivial — a bad derivation is a code fix and a
redeploy, with no persisted state to repair.

**Scale boundary:** the change takes the per-request recovery read from ≤7 rows
to ≤31, on an index that already covers `(user_id, date DESC)`, for one user per
request. The derivation is a handful of single passes over ≤31 values. This stays
comfortably cheap well past any realistic dashboard load; it would need
revisiting only if the window grew to a year-scale history or the section moved
into a multi-user aggregate.

### Testing

- **`internal/recoverytrend`** — table-driven unit tests, the bulk of the
  coverage: full window; exactly `min_baseline_days` samples and one below it;
  all-null HRV; today null with history present; zero-variance history hitting
  the `min_std_dev_ms` floor; each of `balanced` / `elevated` / `suppressed` /
  `unknown`; each of `rising` / `falling` / `steady` / `unknown`; and a boundary
  case at exactly `|z| == balanced_z`, which must read `balanced` (the
  comparison is inclusive).
- **`internal/dashboard/whoop_test.go`** — extend the existing fixtures: `days`
  is the full window with gaps preserved and ordered oldest→newest; the final
  entry is today's local date; `resting_hr_spark` and `today` are byte-identical
  to their pre-change values for the same input (an explicit regression guard on
  the compatibility promise).
- **`internal/dashboard/handler_test.go`** — the widened `ListRange` window is
  requested with the right `since`/`until` local dates; a failed recovery read
  still yields a nil section, not a 500.
- **`internal/config`** — the `[recovery]` block parses and maps into
  `RecoveryConfig`.
- **Contract test** — the recovery section's JSON keys, asserted against the web
  mirror the way the tile catalog is.
- **`prog-strength-web`** — `lib/dashboard.test.ts` covers `adaptRecovery`:
  today's HRV surfaces; nulls in `days` survive as null (not `0`); an unknown
  status/trend string degrades to `"unknown"`; a missing section still yields the
  empty marker. Existing `RecoveryCard` tests must pass unchanged.

### Documentation

- `config.toml` carries the commented `[recovery]` block above as its own
  documentation, matching the density of `[hr_zones]` and `[vectormemory]`.
- `internal/recoverytrend/doc.go` explains the baseline/band/trend model and why
  today is excluded from the baseline, mirroring `internal/hrzones/doc.go`.
- This SOW's status moves to Shipped when the PRs merge.

## Open Questions

1. **Should the agent see HRV balance?** The training snapshot
   (`internal/snapshot`) composes every domain for the agent but carries no
   recovery data at all. Options: extend it in this SOW, extend it in a
   follow-up, or leave the agent recovery-blind. *Lean:* follow-up. The agent's
   snapshot has its own shape and consumers, and bundling it here widens the
   blast radius of a payload change whose only current consumer is the
   dashboard.
2. **When does `resting_hr_spark` retire?** It becomes fully derivable from
   `days` the moment this ships, but the live `RecoveryCard` reads it. Options:
   drop it now and update the card, or keep both until the tile work lands.
   *Lean:* keep both. Removing it here would force tile changes into a SOW that
   deliberately ships none; the downstream tile SOW can delete it once nothing
   reads it.
3. **Mean and SD, or median and MAD?** A single travel night or a fever can pull
   both the baseline and the spread, briefly widening the band and making a
   genuinely suppressed morning read as balanced. A median/MAD formulation is
   materially more robust to that. *Lean:* mean and SD for v1 — it is what the
   z-score language and the "X below your baseline" phrasing assume, and it is
   easier to explain in the UI. Revisit if real data shows outlier nights
   distorting the band.
4. **Calendar days or last-N-with-data?** The window is 30 *calendar* days, so
   an athlete who wore the strap 12 of the last 30 nights gets a 12-sample
   baseline and, below `min_baseline_days`, none at all. The alternative — the
   last 30 days *that have readings* — always yields a full sample but silently
   compares today against a window that may stretch back months. *Lean:*
   calendar days. A baseline should describe a recent period, and "not enough
   recent data" is a true statement worth showing.
