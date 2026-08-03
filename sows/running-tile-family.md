---
type: sow
status: shipped
repos:
  - prog-strength-api
  - prog-strength-web
  - prog-strength-docs
---

# Running Tile Family — Four Dashboard Tiles and the `RunningSection` Payload from `dx/running-tile`

**Status**: Shipped · **Last updated**: 2026-08-03

> Frontend SOW with a substantial API companion. It implements the chosen DX variants
> and therefore inherits that DX's `scope` — here **`in-system`**. The visual
> foundation (design-system **v0.4**, oura-calm) is already decided, so this SOW
> **conforms** to it; it does **not** re-tone the system or touch shared tokens.
> Unlike [`sows/recovery-tile-family.md`](./recovery-tile-family.md), whose payload
> had already landed in a prior SOW, **this SOW also ships the data** — the
> `RunningSection` extension the DX specified. `prog-strength-api` does the payload,
> catalog, and section-gating work, `prog-strength-web` does the rendering work, and
> `prog-strength-docs` only flips the DX to `selected` and marks this SOW shipped.

## Introduction

The **Running mini-card** on the dashboard (`RunningCard` in
`app/(app)/dashboard/_components/tile-renderer.tsx`) prints `21.3 mi this week`, a
min/max-normalized 8-week sparkline, and `runs 4 · pace 10:06 · last 12.7 mi`. The
Design Exploration [`dx/running-tile.md`](../dx/running-tile.md)
(PR Prog-Strength/prog-strength-web#146) named the failure, and it is **different in
kind** from the recovery tile's. Recovery's sparkline was *misinforming*. Running's
sparkline is **redundant**:

- **Four other cards wear the identical grammar.** Walking, Cycling, and Hiking are
  literal copies — their sources each open with the comment *"Mirrors RunningCard's
  grammar"* — and Lifting differs only in the noun under the numeral. A user with an
  ordinary dashboard sees the same green squiggle five times and learns nothing from
  any of them.
- **Eight weekly distance totals is the least interesting true thing available.** It
  is not wrong; it is simply not worth a chart. A 12.7-mile long run and three
  shakeouts renders identically to four flat 5-milers.
- **The tile already receives a figure it throws away.** `delta_pct_vs_prior_week` is
  computed server-side, serialized into the payload, adapted into `RunningView` — and
  rendered nowhere.
- **A year of rich run data is sitting in memory and six fields of it are
  serialized.** `buildRunning` receives the full 53-week hydrated run list from the
  handler's one unified `ListInRange` read, and `activityColumns` already selects
  `avg_heart_rate_bpm`, `max_heart_rate_bpm`, `elevation_gain/loss/high/low`,
  `avg_pace_sec_per_km`, and `environment`. **The gap is serialization, not
  retrieval** — which is what makes an ambitious payload a cheap one.

The DX ran a **keep-several** gate, because
[`sows/customizable-dashboard-tiles.md`](./customizable-dashboard-tiles.md) made the
grid user-composed and `dx/recovery-tile` proved the pattern. But unlike the recovery
family, this gate had a **binding default constraint**: the generic
`BigNum + Spark + MetaRow` running card is being **retired**, not kept alongside, so
one variant had to be good enough to be what every user sees on day one.

**The owner selected four of six idioms** — `load-ramp` as the **default**, with
`week-log`, `effort-heart`, and `vertical-gain` as opt-in catalog tiles.
`stacked-week` and `pace-band` are **dropped**. The catalog goes **15 → 18 tiles**.

This SOW reimplements the four **production-quality** against the real tile and real
data, and ships the `RunningSection` extension they read. Because `scope: in-system`,
there is **no palette, accent, type, or design-system change**. There is **no new
query, no new repository method, and no new table** — every field below is a
projection of data `buildRunning` already holds, with exactly one exception
(`effort-heart`'s max-HR reference) that is called out and gated.

## Proposed Solution

### The shipped `running` tile is rewritten, not preserved

`load-ramp` inherits the `running` id. Rather than mint a fifth id and leave the
generic card addable forever, **the `running` id keeps its id and its catalog slot and
its body becomes `load-ramp`.** Its title changes `Running` → **`Training Load`** and
its tray description is rewritten to match — a card whose hero figure is `+25%`
misdescribes itself as `Running`, and every other domain tile on the grid still owns
the domain noun. Its `href` stays `/activities?view=running`. The other three ship as
new ids:

| Tile id | Variant | Title | Heroes |
| --- | --- | --- | --- |
| `running` *(rewritten)* | `load-ramp` | `Training Load` | the direction (signed load delta) |
| `running_log` *(new)* | `week-log` | `Runs This Week` | the runs themselves |
| `running_effort` *(new)* | `effort-heart` | `Run Effort` | heart rate |
| `running_vertical` *(new)* | `vertical-gain` | `Vertical Gain` | climbing |

**This is deliberately migration-free.** Every stored layout containing `"running"`
keeps working and silently renders the ramp card; no layout row is rewritten, no
`defaultLayout` edit, no read-path filter drops a tile a user asked for, and every
existing user is upgraded off the redundant sparkline without action. Only the
*rendering* is retired.

**No two tiles hero the same figure.** The assignment above was binding on the DX
spread and stays binding here: four running tiles on one grid never print the same
number twice. `load-ramp` heroes **duration**, not distance, on purpose — an hour is
an hour whether it was fast or slow, and it is what keeps the default orthogonal to
the three enthusiast tiles rather than a restatement of any of them.

### All four read one section

Every tile reads the **same** `running` section — there is no per-tile payload. The
three new ids get **no section of their own**; the API keeps emitting exactly one
`running` key and the web adapter keeps reading `summary.running`. `buildRunningSection`
is currently `if enabled[TileRunning]`, so a user who added only *Vertical Gain* would
get a nil section and a blank tile. It is re-gated on **any** running-family tile being
enabled, following the `recoveryFamily` loop already in `handler.go` — including its
rule that the section is emitted under the family key regardless of which member tiles
are on the layout.

This makes the response's section-key set intentionally *not* a subset of `layout` —
a layout of `[running_vertical]` yields a `running` section and no `running_vertical`
key. That property is already established by the recovery family and already handled
by the web adapter; it gains a second family here and a handler test to pin it.

### The payload extension ships in this SOW

The DX deliberately folded the data work into this build rather than running it as a
separate round, because **every new field is derivable from the `runs []activity.Activity`
slice and `activity.Metrics` that `buildRunning` already receives.** The contract, from
the DX verbatim:

```go
type RunningSection struct {
	CurrentWeek           RunningCurrentWeek `json:"current_week"`
	Baseline              *RunningBaseline   `json:"baseline"`
	RecentAvgPaceSecPerKm *float64           `json:"recent_avg_pace_sec_per_km"`
	LatestRun             *LatestRun         `json:"latest_run"`
	WeekRuns              []RunningWeekRun   `json:"week_runs"`   // this local week, oldest→newest
	WeeklyLoad            []RunningWeekPoint `json:"weekly_load"` // 8 buckets, oldest→newest
}
```

`WeeklyDistanceSpark` **dies with the card that read it**, exactly as the DX specified —
but on an expand/contract schedule, not in one step. `adaptRunning` calls
`running.weekly_distance_spark.map(...)` with **no null tolerance** (`lib/dashboard.ts:270`),
so removing the field while the deployed web build still reads it throws a `TypeError`
rather than degrading. The field therefore **survives step 1 unchanged**, stops being read
in step 2, and is deleted in a small step-3 API PR. See Rollout. It is untouched on
`EnduranceSection` throughout (Walking, Cycling, Hiking still draw sparklines and still
need it).

Six properties of this shape are **binding on every tile**, carried over from the DX:

1. **`weekly_load` supersedes the spark.** A bare `[]float64` with no week anchoring
   cannot label an axis or tell a zero-distance week from a missing one.
   `weekly_load` carries `week_start`, duration, and count per bucket.
2. **Nil elevation is not zero elevation.** `ElevationGainMeters` is `*float64`
   precisely because a treadmill run and a pancake-flat loop are different facts. A
   tile rendering `nil` as `0 ft` asserts the user ran on flat ground when the truth
   is the source carried no altitude. Use `ElevationRuns` to say *"3 of 4 runs"*.
3. **Same for heart rate.** `AvgHeartRateBpm` is nullable on every run.
   `HeartRateRuns` exists so a tile can be honest about coverage instead of quietly
   under-reporting the week.
4. **Never recompute a server figure.** `AvgPaceSecPerKm`, `Baseline.*`, and
   `DeltaPctVsPriorWeek` are computed server-side over defined windows. A tile that
   means-of-means the per-run paces will disagree with the deep page and with itself
   — a duration-weighted week pace and a naive average of four run paces are different
   numbers, and the long run is exactly what separates them. **The only client-side
   arithmetic permitted is a signed delta of two server figures** (`load-ramp`'s
   duration ramp, `effort-heart`'s `+3 vs 4-wk`), never a re-averaged series.
5. **`recent_avg_pace_sec_per_km` and `current_week.avg_pace_sec_per_km` are different
   figures and must be labelled differently.** The first is a **30-day** aggregate
   (the `10:06` on today's card); the second is this week's. They visibly disagree in
   any week faster or slower than the month.
6. **Pace is stored per kilometre; distance and elevation are stored in metres.**
   Display conversion goes through the existing `formatPaceValue` /
   `formatDistanceValue` / `formatElevationValue(meters, unit)` helpers and the user's
   `DistanceUnit`. **Every tile is checked in both `mi` and `km`** — a pace clock and
   a 4-digit foot count have different widths than their metric equivalents and this
   card has no room to spare.

### Color logic — the traps, restated as a per-tile contract

Running has **one** hue: `--discipline-run-bg` `#16241f` / `--discipline-run-fg`
`#9cc7b8` / `--discipline-run-dot` `#7fae9e`. Per-tile assignment (binding, from the
DX):

| Tile | What carries color | What stays neutral |
| --- | --- | --- |
| `Training Load` | the **delta figure only** — success / warning / muted | the rail, the caption, everything else |
| `Runs This Week` | the **per-row pace figure** in sage, brighter when that run beat baseline | dates, distances, HR, header |
| `Run Effort` | the **zone dots** (`--zone-1..5`) — and **no sage at all** | the bpm numeral, the delta, the caption |
| `Vertical Gain` | the **silhouette fill** in sage | figure, caption, gap outlines |

Four rules that are the most likely way this gets built wrong:

- **Periwinkle is not a running colour.** `--accent` `#9aa6d6` is app chrome and
  selection/"today" meaning. An activity must never read as selection. The design
  system's `--discipline-lift-dot: #9aa6d6` is a documented exception for lifting, not
  licence to reach for the accent here.
- **Hike clay is forbidden on `Vertical Gain`.** Elevation is clay-coloured on the
  *Hiking* surfaces because **activity type owns activity colour**. A running tile
  drawing its climbing in `#b08e77` reads as a hike on the grid. Sage fill, always.
  This is the single most likely colour mistake in the set.
- **There is no run tonal ramp.** Lifting has `--discipline-lift-1..4`; running has
  only the bg/fg/dot triplet. Gradation is derived by **varying alpha on
  `--discipline-run-dot`**, never by inventing `--discipline-run-1..4`.
- **Nothing on this family is ever danger red.** A big week is a *choice*, not a
  failure. `Training Load` may use warning `#d6b87f` for an aggressive ramp; running
  more than usual is not an emergency, and the app does not scold.

`--zone-1..5` belongs to heart rate and to **exactly one tile**. A second tile
painting five-tone anything turns the grid into confetti.

## Goals and Non-Goals

### Goals

**`prog-strength-api`**

- **Extend `RunningSection`** with `Baseline`, `WeekRuns`, `WeeklyLoad`, and the eight
  new `RunningCurrentWeek` fields, then **retire `WeeklyDistanceSpark`** from it in a
  follow-up contract PR once web has stopped reading it (only from it —
  `EnduranceSection` keeps its own). All new computation lands in
  `internal/dashboard/running.go` as pure functions over the already-fetched run
  slice, testable with no DB and no clock.
- **No new query, no new repository method, no new table.** `buildRunning` keeps its
  existing inputs (`activity.Metrics` + the 53-week `endurance` slice + `now`/`loc`).
  The **one** exception is `effort-heart`'s max-HR reference — see below.
- **Wire the heart-rate-zone engine into the dashboard handler** via a
  `SetHRZonesEngine(engine, window)` setter mirroring `activity.Handler`'s, and
  populate per-run zone indices **only when the `running_effort` tile is enabled**.
  A nil engine or a failed reference read degrades to nil zone fields — never a 500,
  never a fabricated zone.
- **Add three `TileID` constants** — `TileRunningLog` (`running_log`),
  `TileRunningEffort` (`running_effort`), `TileRunningVertical` (`running_vertical`)
  — placed in `Catalog` immediately after `TileRunning`, in that order. `Catalog`
  order fixes the web tray order and must stay byte-identical to the TS mirror.
- **Re-gate `buildRunningSection`** on *any* running-family tile being enabled,
  building and emitting the section **exactly once** under the `"running"` key
  regardless of how many family tiles are on the layout.
- **Leave `defaultLayout` unchanged** — a user still gets exactly one running tile
  (`TileRunning`) by default; the other three are opt-in from the tray.
- **Update `tiles_test.go`** (both the `all` and `want` lists), add handler tests
  asserting a layout of `[running_vertical]` alone yields a populated `running`
  section and **no** `running_vertical` key, three family tiles yield one `running`
  section, and no family tile yields **no** `running` key.
- **Cover the builder's arithmetic in `running_test.go`** — the duration-weighted HR,
  the aggregate week pace, the baseline window and its denominator, nil-vs-zero
  elevation, DST week boundaries, and the empty/first-run cases.
- **Go CI green** (build, vet, test).

**`prog-strength-web`**

- **Mirror the payload** — `lib/api.ts`'s `DashboardRunning` type and
  `lib/dashboard.ts`'s `RunningView` + `adaptRunning`, including dropping
  `spark`/`weekly_distance_spark` from the running path only. Unit conversion stays in
  the adapter through the existing helpers; **no tile does raw metre arithmetic**.
- **Add the three `TileId`s and their catalog entries** in `lib/dashboard-tiles.ts`,
  in the same position and order as the Go mirror, each `href:
  "/activities?view=running"`, each with its own one-line tray description, and
  **rewrite the `running` entry's title and description** — `Running` /
  `"Weekly running distance and pace."` no longer describes the card. Proposed copy:
  - `running` — `Training Load` — *"Whether you're building or backing off, against your 4-week normal."*
  - `running_log` — `Runs This Week` — *"Every run this week as a dated row."*
  - `running_effort` — `Run Effort` — *"How hard your runs were, by average heart rate."*
  - `running_vertical` — `Vertical Gain` — *"How much climbing was in this week's runs."*
- **Build four production tile components** under
  `app/(app)/dashboard/_components/running/`, each owning the `MiniCard` body (the
  uppercase title, the `p-4` padding, and the whole-card link into
  `/activities?view=running` stay chrome each keeps functional) and each exporting a
  **titled empty variant** for the no-running-data case.
- **Single-source shared formatting and status mapping** in one small tested module
  (`running/shared.ts`): `loadStatus`, `loadWord`, `signedPct`, `paceDelta`,
  `weekdayLabel`, `coverageCaption`, `zoneToken`. Four hand-rolled copies of the
  status switch is exactly how a `+25%` week ends up red.
- **Rewire `TileCard`** (`tile-renderer.tsx`) with four cases, each threading
  `data.running` and each falling to its own empty variant when `!data.running.present`.
  `RunningCard` is deleted from `tile-renderer.tsx`. The `never` default keeps its
  compile-time exhaustiveness guarantee.
- **Handle every state the DX enumerated, on all four**, with no `NaN`, no
  `+Infinity%`, no chart frame around nothing, and **no degrading to em-dashes**:
  - **An ordinary week** — 3–5 runs, one longer than the rest.
  - **Zero runs this week** — the state the current card handles worst, seen every
    Monday. Each tile says something true from `baseline` and `latestRun` (*"resting ·
    5 days since your last run"*) **without promoting last week's numbers into this
    week's slot**.
  - **One run this week** — n = 1 must not imply a distribution or a trend.
  - **A brand-new runner** — one run ever, `baseline` nil, `deltaPctVsPriorWeek` nil,
    `weeklyLoad` almost entirely zeros. **Binding on `Training Load`**, because this is
    a new user's first impression of the flagship tile.
  - **Indoor / treadmill runs** — a treadmill contributes **no column and no zero** to
    `Vertical Gain`; the indoor-only runner sees a stated *"no outdoor runs this week"*,
    not an empty chart.
  - **Runs with no heart rate** — `4 runs · 2 with HR` is honest; a duration-weighted
    average silently computed over half the week is not.
  - **A very long single run** — a 12.7-mile run beside three 3-milers is what a real
    week looks like. Scaling is chosen deliberately so the shakeouts are not slivers.
  - **Both breakpoints and both units** — full-width single-column on mobile,
    one-third-width on desktop, within the **~180px** content budget, in `mi` and `km`.
- **Tests** — priority on the state matrix and the color contract, one file per tile
  plus `shared.test.ts`, `tile-renderer.test.tsx`, `lib/dashboard.test.ts` (adapter,
  including the removed spark), and `lib/dashboard-tiles.test.ts` (count 15 → 18,
  order, `ALL_TILE_IDS` exhaustiveness).
- **CI green** (lint/format/typecheck/test/build).

**`prog-strength-docs`**

- Flip [`dx/running-tile.md`](../dx/running-tile.md) to `status: selected`, noting
  `load-ramp` as the **default** and `week-log` / `effort-heart` / `vertical-gain` as
  the kept enthusiast tiles, and `stacked-week` / `pace-band` as dropped; mark this SOW
  `shipped` on merge.

### Non-Goals

- **Personal records.** `GetUserRunningBestEfforts` exists and powers
  `/personal-records`, but it is a *separate query*, and adding it would forfeit the
  no-new-reads property that makes this cheap. No kept idiom depends on it. A
  PR-flavoured tile is its own DX.
- **Real time-in-zone.** `Run Effort` classifies each run by its **average** HR
  against the max-HR reference — one cheap read, no trackpoint scan. Actual
  time-in-zone is computed per-activity from trackpoints at read time
  (`attachHeartRateZones`) and pulling it for a week of runs on every dashboard render
  is a cost this tile will not pay. **The tile's language must reflect that**
  (*"mostly zone 2 runs"*) and must never claim minutes-in-zone it did not compute.
- **Any design-system or shared-token change.** `scope: in-system` against v0.4 —
  conform only. No token/accent/type edit, no `design-system.md` change, no invented
  run tonal ramp, and no hike clay on the elevation tile.
- **Any layout migration or stored-layout rewrite.** The `running` id is preserved
  precisely so none is needed. No SQL, no backfill, no `CHECK` constraint (there isn't
  one — migration 049; the write path validates and the read path filters).
- **Changing the default layout.** A user still gets one running tile; `defaultLayout`'s
  composition is untouched.
- **`stacked-week` and `pace-band`.** Not selected, not built, not held open. If either
  is wanted later it re-enters through a fresh selection against the closed DX PR.
- **The Walking / Cycling / Hiking / Lifting tiles.** They keep `BigNum + Spark +
  MetaRow` and keep `EnduranceSection.weekly_distance_spark`. Making the *rest* of the
  grid stop repeating itself is downstream work this SOW deliberately does not start.
- **The shared `Spark` / `BigNum` / `MiniCard` / `MetaRow` primitives.** Unmodified;
  the running tiles simply stop using `Spark` and `BigNum`.
- **The deep `/activities?view=running` page.** Out of scope and unchanged. These tiles
  link into it; they do not restyle it.
- **`prog-strength-mobile`.** It carries no tile-catalog mirror, does not consume the
  dashboard layout, and does not read `weekly_distance_spark` (verified). Not in `repos:`.
- **Promoting the DX mockup code.** `app/design-explore/running-tile/**` is the
  **visual spec, not code to copy** — the throwaway shell, the fixture file, and the
  `_components/*` sources stay on the unmerged DX branch. The `design-explore` route
  stays gated behind `NEXT_PUBLIC_ENABLE_DESIGN_EXPLORE` and never ships. **The DX PR
  is closed, never merged.**

## Implementation Details

### API — DTO (`internal/dashboard/dto.go`)

The **end state**, after the step-3 contract PR. Step 1 adds every field below *and
leaves `WeeklyDistanceSpark []float64` in place*; only step 3 removes it.

```go
type RunningSection struct {
	CurrentWeek           RunningCurrentWeek `json:"current_week"`
	Baseline              *RunningBaseline   `json:"baseline"`
	RecentAvgPaceSecPerKm *float64           `json:"recent_avg_pace_sec_per_km"`
	LatestRun             *LatestRun         `json:"latest_run"`
	WeekRuns              []RunningWeekRun   `json:"week_runs"`
	WeeklyLoad            []RunningWeekPoint `json:"weekly_load"`
}

type RunningCurrentWeek struct {
	DistanceMeters      float64  `json:"distance_meters"`
	RunCount            int      `json:"run_count"`
	DeltaPctVsPriorWeek *float64 `json:"delta_pct_vs_prior_week"`
	DurationSeconds     int      `json:"duration_seconds"`
	AvgPaceSecPerKm     *float64 `json:"avg_pace_sec_per_km"`
	AvgHeartRateBpm     *int     `json:"avg_heart_rate_bpm"`
	ElevationGainMeters *float64 `json:"elevation_gain_meters"`
	HeartRateRuns       int      `json:"heart_rate_runs"`
	ElevationRuns       int      `json:"elevation_runs"`
	LongestRunMeters    float64  `json:"longest_run_meters"`
	DaysRun             int      `json:"days_run"`
}

type RunningWeekRun struct {
	ActivityID          string    `json:"activity_id"`
	Name                *string   `json:"name"`
	StartTime           time.Time `json:"start_time"`
	LocalDate           string    `json:"local_date"` // YYYY-MM-DD in the user's tz
	DistanceMeters      float64   `json:"distance_meters"`
	DurationSeconds     int       `json:"duration_seconds"`
	AvgPaceSecPerKm     *float64  `json:"avg_pace_sec_per_km"`
	AvgHeartRateBpm     *int      `json:"avg_heart_rate_bpm"`
	HeartRateZone       *int      `json:"heart_rate_zone"` // 1..5, nil unless Run Effort is enabled
	ElevationGainMeters *float64  `json:"elevation_gain_meters"`
	Environment         string    `json:"environment"` // outdoor | indoor
}

type RunningWeekPoint struct {
	WeekStart           string   `json:"week_start"` // YYYY-MM-DD, local Monday
	DistanceMeters      float64  `json:"distance_meters"`
	DurationSeconds     int      `json:"duration_seconds"`
	RunCount            int      `json:"run_count"`
	ElevationGainMeters *float64 `json:"elevation_gain_meters"`
}

// Trailing 4-week average EXCLUDING the current week — what "normal" means for
// this athlete. nil until there is at least one prior week with a run.
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

`HeartRateZone` is the one addition to the DX's shape, and it exists to keep the
five-zone thresholds **single-sourced in Go**. The alternative — shipping a raw
`max_hr_reference_bpm` and mirroring `ZoneUpperBounds` into TypeScript — duplicates a
config-driven boundary set across the language boundary, which is exactly the drift
`lib/dashboard-tiles.ts`'s mirror test exists to prevent elsewhere. One integer per
run is cheaper and cannot disagree with the run detail page.

### API — the builder (`internal/dashboard/running.go`)

`buildRunning` keeps its signature and its nil-on-no-running-data behaviour. Its body
grows four pure helpers over the run slice, all bucketed with the existing
`localWeekStart` / `weeklyBucketStarts` helpers so DST correctness comes for free.

**Current-week aggregates.** `weekRuns` is every `ActivityRunning` row whose
`localWeekStart(StartTime, loc)` equals the current week's, sorted **ascending** by
`StartTime`.

```
DurationSeconds     = Σ r.DurationSeconds
AvgPaceSecPerKm     = Σduration / (Σdistance / 1000)   ; nil when Σdistance == 0
AvgHeartRateBpm     = round( Σ(dur_i × hr_i) / Σ(dur_i) ) over runs with hr != nil
                                                        ; nil when none carry HR
HeartRateRuns       = count(hr != nil)
ElevationGainMeters = Σ gain over runs with gain != nil ; nil when NONE carry gain
ElevationRuns       = count(gain != nil)
LongestRunMeters    = max(distance)
DaysRun             = |distinct LocalDate|
```

The HR average is **duration-weighted, not run-averaged**: a 65-minute long run and a
20-minute shakeout are not equal evidence about the week's effort, and the naive mean
would let the shakeout move the figure as much as the long run.
`ElevationGainMeters` sums only over gain-bearing runs and is `nil` — never `0` — when
none carry it; the indoor-only week must be distinguishable from a flat week.

**`DistanceMeters` and `RunCount` continue to come from `metrics`, unchanged.** They
are the two figures already on screen today and the ones the deep page prints; leaving
their provenance alone means this SOW cannot silently move a number a user is looking
at. See Open Question 1.

**Baseline.** The four buckets *before* the current week:
`weeklyBucketStarts(now, loc, 5)[0:4]`.

```
Weeks = count of those four buckets holding >= 1 run
if Weeks == 0 → Baseline is nil

DistanceMeters      = Σdistance(window)                 / Weeks
DurationSeconds     = Σduration(window)                 / Weeks
RunsPerWeek         = count(runs in window)             / Weeks
ElevationGainMeters = Σ gain over gain-bearing runs     / Weeks ; nil when none
AvgPaceSecPerKm     = Σduration(window) / (Σdistance(window) / 1000)  ; aggregate, not
                                                                       a mean of means
AvgHeartRateBpm     = round( Σ(dur_i × hr_i) / Σ(dur_i) ) over HR-bearing runs in window
```

The denominator is `Weeks` — **weeks that actually held a run** — not a flat 4. A
runner three weeks into the product would otherwise have their baseline diluted by
weeks of pre-signup zeros, and every subsequent week would read as an enormous build.
The trade-off is that a deliberate down week does not drag the baseline down; see Open
Question 2. Pace and HR are **aggregates over the window's runs**, not averages of
weekly averages — the same method `RecentAvgPaceSecPerKm` uses, so the two figures are
comparable.

**Weekly load.** Eight buckets from `weeklyBucketStarts(now, loc, 8)`, oldest→newest,
each carrying `week_start` as `YYYY-MM-DD` in `loc`, summed distance, duration, and run
count, and `ElevationGainMeters` summed over gain-bearing runs in that bucket (nil when
none). A bucket with no runs is a real zero and is rendered as one — that is the
distinction `weekly_distance_spark` could not make.

**Zone classification.** When the engine is wired *and* `running_effort` is enabled,
the handler reads `RecentHRStats(ctx, userID, hrWindow, "")` once, derives
`ref := engine.EstimateReference(stats)`, and sets `HeartRateZone = engine.ZoneForBPM(ref, *avgHR) + 1`
for each run carrying HR (the engine is 0-indexed; the wire field is 1-indexed to match
the zone tokens). Runs without HR keep `nil`. A nil engine, a failed read, or a
disabled tile leaves **every** `HeartRateZone` nil, and `Run Effort` renders its dots in
neutral ink with the bpm figures intact — degraded, not broken.

### API — handler (`internal/dashboard/handler.go`)

The gating mirrors the recovery family exactly:

```go
// Every running-family tile reads the ONE shared "running" section, so it is
// built once when ANY family tile is enabled and emitted only under the
// "running" key — the same section-key/layout divergence the recovery family
// established below.
runningFamily := []TileID{
	TileRunning, TileRunningLog, TileRunningEffort, TileRunningVertical,
}
anyRunning := false
for _, id := range runningFamily {
	if enabled[id] {
		anyRunning = true
		break
	}
}
if anyRunning {
	out[string(TileRunning)] = h.buildRunningSection(
		ctx, r, userID, endurance, now, loc, enabled[TileRunningEffort],
	)
}
```

`buildRunningSection` gains the `withZones bool` parameter and performs the
`RecentHRStats` read through the existing `defer1` wrapper, so a reference-read failure
yields nil zones rather than a 500 — the same resilience principle as every other
section read. `SetHRZonesEngine(e *hrzones.Engine, window time.Duration)` is added to
`dashboard.Handler` as a setter mirroring `activity.Handler`'s (nil-guarded, safe never
to call), and `internal/server/server.go` calls it on the dashboard handler with the
same `hrEngine` and window it already passes to the activity handler. A setter rather
than a constructor arg keeps `NewHandler`'s twelve-parameter signature and every
existing test untouched.

### API — catalog (`internal/dashboard/tiles.go`)

```go
TileRunning         TileID = "running"
TileRunningLog      TileID = "running_log"
TileRunningEffort   TileID = "running_effort"
TileRunningVertical TileID = "running_vertical"
TileWalking         TileID = "walking"
```

…and the same three appended to `Catalog` in that position, so the family reads as a
contiguous group in the tray exactly as the recovery ids do. `ValidTileID` and
`catalogSet` derive from `Catalog` and need no edit, so the layout write-path validation
picks the new ids up for free.

### Web — payload mirror (`lib/api.ts`, `lib/dashboard.ts`)

`DashboardRunning` gains the new blocks and loses `weekly_distance_spark`.
`adaptRunning` converts every metre and every pace **once, in the adapter**, through
`formatDistanceValue` / `formatPaceValue` / `formatElevationValue` and
`distanceToDisplay`, so no tile does raw arithmetic on a metre:

```ts
export type RunningView = {
  currentWeek: {
    distance: string; runCount: number; deltaPct: number | null;
    durationSeconds: number; pace: string; avgHeartRate: number | null;
    elevation: string | null; heartRateRuns: number; elevationRuns: number;
    longestRun: string; daysRun: number;
  };
  pace: string;                    // 30-day aggregate — labelled "30d", never "this week"
  baseline: RunningBaselineView | null;
  latestRun: { name, distance, durationSeconds, startTime } | null;
  weekRuns: RunningWeekRunView[];  // oldest→newest
  weeklyLoad: RunningWeekPointView[];
  unit: DistanceUnit;
};
```

`spark` is removed from `RunningView`. `EnduranceView`'s spark and `adaptEndurance` are
untouched. `baseline` and the nullable per-run fields are typed optional/nullable and
**guarded once at the top of each tile** — never `!`-asserted.

### Web — the four tiles (`app/(app)/dashboard/_components/running/`)

- **`shared.ts`** — the load-status switch, the house status words, signed-percentage
  and signed-pace formatting with a unicode minus, the coverage caption
  (`"3 of 4 runs"`), local-date weekday parsing with no timezone drift, and the zone
  index → token map. Pure, no React, tested directly. Reads the v0.4 CSS vars by name
  (`--success`, `--warning`, `--muted`, `--discipline-run-dot`, `--zone-1..5`) and
  **never a raw hex**.

- **`load-ramp.tsx`** (`Training Load`, id `running`) — one large signed figure for
  this week's **time on feet** against the 4-week baseline, the plain-language read
  beneath (`building · 3:36 vs 2:54 avg`), over a tiny 8-bucket neutral rail of weekly
  load with the baseline as a ghost line through it. **The strongest size contrast in
  the family** — one big numeral over a small rail, nothing in between.

  ```
  loadDeltaPct = (currentWeek.durationSeconds − baseline.durationSeconds)
                 / baseline.durationSeconds × 100
  ```

  A signed delta of two server figures — the permitted client computation. Status
  bands and the words that go with them:

  | Condition | Token | Word |
  | --- | --- | --- |
  | `runCount == 0` | `--muted` | `resting` |
  | `delta >= +10%` | `--warning` | `ramping` |
  | `0 <= delta < +10%` | `--success` | `building` |
  | `delta < 0` | `--muted` | `easing` |

  **Never `--danger`.** `baseline == null` (first week ever) prints the week's own
  time on feet with `first week` beneath and **no percentage** — no `NaN`, no
  `+Infinity%`. `baseline.durationSeconds == 0` is treated as no baseline.
  The zero-run week — the state that made this the default — renders
  `−100% · resting · 5 days since your last run` from `baseline` and `latestRun`, which
  is true, useful, and the reason this tile earns the default slot where every other
  treatment goes quiet.

- **`week-log.tsx`** (`Runs This Week`, id `running_log`) — **no numeral above 14px.**
  A quiet caption header (`21.3 mi · 4 runs · 3:36`) over dated tabular rows,
  hairline-divided, `Geist_Mono` permitted for the figures:
  `Sat · 12.7 mi · 10:13 · 156`. An indoor run carries a small glyph rather than a
  blank column; a run with no HR shows an em-dash **in that column only** — the row
  still says everything else it knows. More than four runs collapses the oldest into a
  `+2 earlier` line rather than growing the card. Sage is spent **entirely on the
  per-row pace**, brighter when that run beat `baseline.avgPaceSecPerKm`, faint when it
  did not. Zero runs renders the last run's row under a `no runs this week` caption,
  clearly dated so last week is never mistaken for this one.

- **`effort-heart.tsx`** (`Run Effort`, id `running_effort`) — the week's
  duration-weighted bpm as a medium numeral with `+3 vs 4-wk` beside it (a signed delta
  of two server figures), over a horizontal rail of one dot per run, positioned by bpm,
  **coloured by `HeartRateZone` and sized by duration**. **No sage on this card at all**
  — which is precisely what stops it looking like its siblings. Coverage is stated,
  never hidden: `3 of 4 runs` sits in the caption whenever `heartRateRuns < runCount`.
  The copy says *"mostly zone 2 runs"* and **never** claims minutes-in-zone. Zero
  HR-bearing runs renders the run count and a `no heart-rate data this week` line, not
  an empty rail.

- **`vertical-gain.tsx`** (`Vertical Gain`, id `running_vertical`) — a large figure
  with a small unit suffix (`899 ft`) over a full-width stepped silhouette, one column
  per run, height by that run's gain, with the week's biggest climb called out
  (`most: 702 ft · Sat`). **The most horizontal composition of the four**, deliberately,
  so it pairs cleanly beneath another tile. A run with nil gain contributes a **hairline
  outline gap, not a zero column**, and the caption says `3 of 4 runs`. The indoor-only
  week renders the stated sentence *"no outdoor runs this week"* — never an empty chart
  frame, never `0 ft`. Sage fill; **hike clay is forbidden**.

**Scaling, for every chart in the family.** The long run is what breaks naive scaling —
a 12.7-mile run beside three 3-milers renders the shakeouts as slivers under linear
sizing. Each tile picks its scale deliberately and states it in a comment: the load
rail scales to the max of `weeklyLoad` ∪ `baseline` so the ghost line is always on
canvas; the effort rail's dot sizing is clamped to a legible minimum; the vertical
silhouette scales to the week's max gain with a floor so a 20-foot run is still a
visible step.

### Web — tile renderer (`tile-renderer.tsx`)

Four cases replacing the one, each with the same present/empty shape, and `RunningCard`
deleted:

```tsx
case "running_vertical":
  return data.running.present
    ? <VerticalGainCard section={data.running} href={href} />
    : <RunningEmptyCard title="Vertical Gain" href={href} />;
```

`RunningEmptyCard` generalizes today's inline empty body to take the tile's `title`, so
all four share one empty card with four different headings and the existing
`"Import a run to start tracking"` CTA. No other case in the switch is touched, and the
`never` default still makes a missing case a **type error**.

### Tests

**API**

- `running_test.go` — the duration-weighted HR against a hand-computed expectation;
  the aggregate week pace differing from the mean of per-run paces on the headline
  fixture (the long run is what separates them); baseline `Weeks` counting only weeks
  with runs and the denominator following it; `ElevationGainMeters` **nil, not zero**,
  when every run is indoor; `DaysRun` counting distinct local dates across a two-runs-
  one-day week; `weekly_load` zero-vs-missing on the down week; a DST-crossing week
  boundary; `baseline == nil` for a first-run-ever user; nil section when there is no
  running data at all.
- `handler_test.go` / `summary_layout_test.go` — the three gating assertions above, plus
  `HeartRateZone` nil when `running_effort` is absent from the layout and populated when
  present, and nil (not a 500) when the reference read fails.
- `tiles_test.go` — `all` and `want` lists updated; `TestCatalog_Order` pins the family
  contiguous after `running`.

**Web**

- `running/shared.test.ts` — the load-status mapping asserted explicitly, including
  **`+25%` → `--warning` and never `--danger`** and **a down week → `--muted`, never
  `--danger`**; `signedPct` producing `+25%` / `−100%` / `±0%` with a unicode minus;
  `weekdayLabel` parsing a local date with no timezone drift; `zoneToken(3)` →
  `--zone-3`.
- One test file per tile, each driven across the **four fixture states** (ordinary week
  / zero runs / first run ever / indoor only) **in both units**, asserting the tile's own
  hero figure is present and that no state renders a bare `—` as its whole body.
  Specifically: `load-ramp` renders a true sentence at zero runs and no percentage with
  a nil baseline; `week-log` shows an em-dash only in the HR column of a no-HR row and
  collapses a five-run week; `effort-heart` states coverage whenever
  `heartRateRuns < runCount` and renders neutral dots when zones are nil;
  `vertical-gain` renders a gap outline (not a zero column) for a treadmill run and the
  stated sentence for an indoor-only week.
- `tile-renderer.test.tsx` — all four ids render their card when `running.present`, their
  titled empty CTA when not.
- `lib/dashboard.test.ts` — `adaptRunning` maps the new blocks, converts units once, and
  **no longer reads `weekly_distance_spark`**; the endurance adapter still does.
- `lib/dashboard-tiles.test.ts` — count `15 → 18`, the order array, and the
  `ALL_TILE_IDS` exhaustiveness record all updated.

## Rollout

Expand/contract across three PRs, because the payload field being retired is one the
deployed web build reads without a guard.

1. **`prog-strength-api` (expand)** — the DTO extension, the builder work, the engine
   setter and server wiring, the three `TileID`s, the `Catalog` insertion, the family
   re-gate, and the tests, in one PR. **Purely additive: `WeeklyDistanceSpark` stays.**
   **Merges and deploys first** — the web tray must not offer a tile the API's write path
   will reject as an invalid id, and the web tiles have nothing to read until the payload
   ships.
2. **`prog-strength-web`** — the payload mirror, the catalog mirror, the four tile
   components, the shared module, the renderer rewire, and the tests, in one PR.
   `adaptRunning` **stops reading `weekly_distance_spark`** and drops `spark` from
   `RunningView`; `adaptEndurance` is untouched. Vercel preview to verify each tile
   against an ordinary week, a zero-run Monday, a first-run-ever account, an indoor-only
   week, and a week with a 12.7-mile long run — at both breakpoints, in **both units**,
   and with **two and three running tiles on one grid** beside Walking and Steps.
3. **`prog-strength-api` (contract)** — a small PR deleting `WeeklyDistanceSpark` from
   `RunningSection` and the `weeklyDistanceSpark` function from `running.go`, with the
   fixture updates that follow. Lands only after step 2 is deployed. **`weeklyBucketStarts`
   and the `sparkWeeks` constant stay.** `sparkWeeks` is *declared* in `running.go` but
   read by `endurance_tiles.go:24` and `lifting.go:52` — deleting it alongside the
   function it was named for breaks the Walking/Cycling/Hiking and Lifting sparklines at
   compile time. `weeklyLoad`'s eight buckets use it too.
4. **`prog-strength-docs`** — flip `dx/running-tile.md` to `status: selected` (default
   `load-ramp`; kept `week-log`, `effort-heart`, `vertical-gain`; dropped `stacked-week`,
   `pace-band`), mark this SOW `shipped`.

### Verification after rollout

- **The grid stops looking repetitive.** `Training Load` beside Walking and Hiking reads
  as a *different kind of card*, not a third green squiggle. This is the whole premise of
  the DX; if it fails here, nothing was fixed.
- **The add-tile tray offers four running tiles** with four distinct titles and four
  distinct descriptions, contiguous at the top of the catalog. Adding each one persists,
  survives reload, and renders its own card.
- **A user who adds only *Vertical Gain*** — with no `running` tile on their layout —
  gets a **populated silhouette**, not a blank card. (This is the bug the re-gate exists
  to prevent; check it explicitly.)
- **Existing users were upgraded silently**: a stored layout containing `running` now
  renders the ramp card, with no layout edit and no lost tile.
- **No two running tiles on one grid print the same figure as their hero** — the pair
  reads as two facts, not one fact twice.
- **The zero-run week says something true on all four.** Every Monday morning, and every
  skipped week. No empty frame, no flatlined chart, and **last week's numbers are never
  promoted into this week's slot**.
- **A brand-new runner's first dashboard is intact** — one run, no baseline, no `NaN`, no
  `+Infinity%`, no chart frame around nothing.
- **Missing data is stated, not faked.** A treadmill run is a gap and not a `0 ft`
  column; a no-HR run is `3 of 4 runs` and not a silently under-reported average.
- **This week vs normal comes through.** The baseline is visible on every tile that has
  one; no tile shows a today-figure with nothing to measure it against, which is exactly
  the failure `delta_pct_vs_prior_week` has been suffering.
- **The two pace figures never get confused** — the 30-day aggregate and this week's are
  labelled distinctly wherever both appear.
- **The long run doesn't break the scaling** — a 12.7-mile run beside three 3-milers
  leaves the shakeouts legible on every chart.
- **The colour contract holds**: the ramp is the only status colour and is **never
  danger red**; `--zone-1..5` appears on `Run Effort` and nowhere else; `Vertical Gain`
  is sage and not hike clay; periwinkle appears nowhere in the family.
- **They stay calm dashboard tiles** — each within ~180px, legible at one-third width in
  `mi` and `km`, still reading as v0.4, and previewing the `/activities?view=running`
  page they link into rather than looking like Strava's widget dropped into the grid.
- `design-system.md` unchanged; the `design-explore/running-tile` route stays gated and
  404s in production; **no DX mockup code shipped**; the DX PR is **closed, never merged**.

## Open Questions

1. **Two provenances for the current week's distance.** `CurrentWeek.DistanceMeters`
   and `RunCount` come from `activity.Metrics` (an all-time SQL scan bucketed in Go by
   `computeMetrics`), while every new week field is derived from the handler's 53-week
   `endurance` slice. Both see the same rows for the current week, so they should agree
   by construction — but they are two code paths and could drift. **Lean:** leave the two
   existing figures on `metrics` (moving them risks silently changing a number already on
   screen) and add a builder test asserting `Σ weekRuns.DistanceMeters == CurrentWeek.DistanceMeters`
   and `len(weekRuns) == RunCount` on a fixture both paths see, so drift fails loudly.
   Revisit consolidating onto one path as a separate cleanup.
2. **The baseline denominator, and the deliberate down week.** Dividing by weeks-with-runs
   rather than a flat 4 protects the new user, but it means a planned recovery week does
   not lower the baseline — so the week *after* a down week reads as a smaller build than
   it feels. **Lean:** ship weeks-with-runs (protecting the new user matters more, and
   `Weeks` is on the wire so the tile can caption `4-wk avg · 3 weeks` honestly). If down
   weeks turn out to be common enough to distort the ramp, switch the denominator to a
   flat `WindowWeeks` once the user has ≥4 weeks of history and keep weeks-with-runs
   below that.
3. **`Run Effort`'s reference read.** It is the family's only new read, and it is gated on
   the tile being enabled — so a user who pins it pays one extra query per dashboard load.
   **Lean:** ship it gated as specified; `RecentHRStats` is a covered scan over one user's
   running rows and the dashboard already performs several such reads. If it shows up in
   latency, cache the reference per user rather than dropping the zones.
4. **Whether the `weekly_distance_spark` contract PR is worth a third deploy.** The
   expand/contract schedule above costs an extra API PR to delete one dead field.
   **Lean:** keep it — the alternative (removing it in step 1) is a hard `TypeError` in
   the deployed web build, since `adaptRunning` maps the array with no null guard
   (`lib/dashboard.ts:270`), and a one-line optional-typing patch shipped ahead of step 1
   costs a deploy anyway while leaving the field dead in the payload indefinitely. If the
   third PR is judged not worth it, the field can simply be **left in place** and deleted
   with a future payload cleanup — nothing reads it after step 2, and it costs eight
   floats on the wire.
5. **Tray density — four of eighteen tiles are now running, on top of five recovery
   tiles.** Half the catalog now sits under two domains. **Lean:** ship the tray **flat**
   in catalog order (both families are contiguous, so they already read as groups) and
   revisit grouping or section headings as a separate tray-UX change, exactly as
   [`recovery-tile-family`](./recovery-tile-family.md) Open Question 1 decided. Do **not**
   grow this SOW into a tray redesign.
6. **`Training Load` as the tile's name in the tray.** It describes the card honestly, but
   it is the only tile in the catalog not named for its domain — a user scanning for
   "running" finds four tiles and none of them called `Running`. **Lean:** ship
   `Training Load` (the DX's proposed title, and the card genuinely is not a running
   summary), and lean on the three sibling titles all starting from run vocabulary plus
   the shared `/activities?view=running` deep link. Confirm at preview that the tray still
   reads as a running group; if it doesn't, `Running Load` is the compromise.
7. **`week-log`'s four-row ceiling at one-third width.** The `+2 earlier` collapse keeps a
   heavy week inside ~180px, but a user running six days a week sees two-thirds of their
   week collapsed into a count. **Lean:** keep the four-row ceiling — the tile links into
   the full list, and growing the card breaks the grid's rhythm, which is the thing this
   DX exists to protect. Confirm at preview against a six-run week.
