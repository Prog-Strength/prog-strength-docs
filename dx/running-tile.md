---
type: dx
status: selected
surface: running-tile
idioms:
  - stacked-week
  - week-log
  - pace-band
  - effort-heart
  - vertical-gain
  - load-ramp
references:
  - Strava
  - Garmin Connect
  - Whoop
  - TrainingPeaks
  - Robinhood
  - Oura
scope: in-system
variant_count: 6
repos:
  - prog-strength-web
  - prog-strength-docs
---

# DX: Running Tile

**Status**: Selected · **Last updated**: 2026-08-03

> **Selection** (2026-08-03): `load-ramp` is the **default** — it inherits the
> `running` id and its rendering. `week-log`, `effort-heart`, and
> `vertical-gain` ship as opt-in catalog tiles (`running_log`,
> `running_effort`, `running_vertical`). `stacked-week` and `pace-band` are
> **dropped**. Implemented by
> [`sows/running-tile-family.md`](../sows/running-tile-family.md).

> A DX (Design Exploration) is the platform's **divergent** work type. Unlike a
> SOW it does not converge on one correct implementation — it produces N
> differentiated visual variants of a single frontend surface, side by side on
> one comparison route, awaiting a human pick. It **never merges** and ships no
> production code; the chosen direction feeds a downstream SOW that builds it for
> real.

## Context

The dashboard shipped from [`dx/dashboard`](./dashboard.md) as a grid of
per-domain mini-cards, each wearing the same furniture: a `BigNum` headline, a
min/max-normalized `Spark` line, a `MetaRow` of facts. Running was the tile that
furniture was designed around, and it is now the tile that furniture fails
hardest — because **five other cards on the grid wear it too.** Walking, Cycling,
and Hiking are literal copies (their source files each open with the comment
*"Mirrors RunningCard's grammar"*), and Lifting differs only in which noun sits
under the numeral. A user with an ordinary dashboard sees the same green squiggle
five times and learns nothing from any of them.

[`dx/steps-tile`](./steps-tile.md) was the first admission that the grammar
doesn't fit every metric — it shipped `micro-bars-goal-line`, and Steps now draws
goal-relative bars. [`dx/recovery-tile`](./recovery-tile.md) went further and
shipped **five** tiles from one exploration. Running is the next one, and its
failure is different in kind from Recovery's. Recovery's sparkline was
*misinforming*; Running's sparkline is **redundant**. It is not wrong — 8 weekly
distance totals really is what it draws — it is simply the least interesting true
thing available about a week of running, rendered identically to four neighbors.

Concretely, today's tile prints `21.3 mi this week`, a normalized 8-week line, and
`runs 4 · pace 10:06 · last 12.7 mi`. What it does not say, though the server
already knows all of it:

- **Which four runs.** A 12.7-mile long run and three shakeouts is a completely
  different week from four identical 5-milers, and the tile renders them the same.
- **How hard.** Every run carries `avg_heart_rate_bpm` and `max_heart_rate_bpm`.
  Neither has ever appeared on the dashboard.
- **How much climbing.** Every outdoor run carries `elevation_gain_meters`,
  `loss`, `high`, and `low`. Hiking's tile shows gain; Running's does not.
- **Whether this week is normal.** `delta_pct_vs_prior_week` is computed, shipped
  in the payload, and then **dropped on the floor by the card** — it appears in
  `RunningView` and is rendered nowhere.

**The unlock is that none of this costs a query.** `buildRunningSection` already
receives the **53-week hydrated run list** (`handler.go` — one unified
`ListInRange` read that feeds the running tile and the streak), and
`activityColumns` already selects `avg_heart_rate_bpm`, `max_heart_rate_bpm`,
`total_calories`, `elevation_gain/loss/high/low`, `avg_pace_sec_per_km`,
`best_pace_sec_per_km`, and `environment`. A year of fully-populated runs is
sitting in memory in `buildRunning`, and the section serializes six fields of it.
**The data gap here is serialization, not retrieval** — which is why this DX can
propose an ambitious payload without proposing an expensive one.

**This exploration ends in a keep-several selection, and one of the kept variants
replaces the tile in the screenshot.** [`sows/customizable-dashboard-tiles.md`](../sows/customizable-dashboard-tiles.md)
made the grid user-composed, and `dx/recovery-tile` proved the pattern —
`hrv_balance`, `morning_vitals`, `recovery_trend`, and `recovery_log` are live
TileIDs in both catalog mirrors today. So the gate is **not** "which one variant
wins." It is:

1. **Which variant becomes the default `running` tile** — the generic
   `BigNum + Spark + MetaRow` card is being **retired**, not kept alongside. One
   variant has to be good enough to be what every user sees on day one.
2. **Which of the rest ship as separate, opt-in catalog tiles** — so a runner who
   cares about climbing can pin *Vertical Gain* and a runner who doesn't never
   sees it.

See *Composition* below — the retirement is what makes the default-candidate
constraint binding, and it is a hard constraint on the spread, not a nice-to-have.

`scope: in-system`: the foundation is decided (see
[`../design-system.md`](../design-system.md) **v0.4**, oura-calm) — soft
near-black ramp, the single **periwinkle** accent (`#9aa6d6`, app-chrome and
meaning only), **Manrope** with tight numeric tracking, tabular figures, 14px
hairline panels, desaturated status colors, and the **run sage** discipline
triplet. Variants do **not** re-litigate palette, accent, or type. They diverge on
**layout, structure, density, and composition** inside a mini-card.

## The surface

The **Running mini-card** — `RunningCard` in
`app/(app)/dashboard/_components/tile-renderer.tsx` — one cell of the responsive
`grid-cols-1 sm:grid-cols-2 lg:grid-cols-3` `CardGrid`. A variant owns everything
inside the `MiniCard` shell — the uppercase title, the `p-4` padding, and the
whole-card `next/link` into `/activities?view=running` are chrome it keeps
functional — and composes the card body.

**The space budget is the hard constraint.** Today the body is `BigNum` (~32px) +
`Spark` (`h-7` = 28px) + `MetaRow` (~16px) with `gap-3` gutters — roughly
**150–170px** of content. Treat **~180px total** as the ceiling. Because two or
three of these may sit on one grid, a variant that only works at 260px is not a
candidate; it is a component for `/activities?view=running`.

**Neighbors.** The tile sits beside up to fourteen others under one grid. Three
are worth studying before designing: **Walking / Cycling / Hiking** are the
copies this exploration is trying to stop looking like — whatever wins must be
visibly *not* their grammar; **Steps** already broke from the sparkline
(`StepsGoalBars`), so the grid's furniture is provably not uniform; and the
**recovery family** is the direct structural precedent for shipping several
tiles off one section.

### The data — today, and what this DX proposes

**What ships today** (`RunningSection` in `internal/dashboard/dto.go` →
`RunningView` in `prog-strength-web/lib/dashboard.ts`):

```ts
type RunningView = {
  currentWeek: { distance: string; runCount: number; deltaPct: number | null };
  pace: string;                    // "m:ss" in display unit, or "—"
  latestRun: { name, distance, durationSeconds, startTime } | null;
  spark: SparkSeries;              // 8 weekly distances, zero-filled
  unit: DistanceUnit;
};
```

That supports exactly the card in the screenshot and nothing in this document.
**Unlike `dx/recovery-tile`, the payload work has not landed yet** — it is
deliberately folded into the downstream build SOW rather than run as a separate
round, because every field below is a projection of data `buildRunning` already
holds. **The variants here are mockups driven by static fixtures**; the shape
below is the contract the downstream SOW implements, and it is specified here so
the mockups and the eventual payload agree.

**Proposed `RunningSection` extension.** Every field is derivable from the
`runs []activity.Activity` slice and `activity.Metrics` that `buildRunning`
already receives — **no new repository method, no new query, no new table**:

```go
type RunningSection struct {
	CurrentWeek           RunningCurrentWeek `json:"current_week"`
	Baseline              *RunningBaseline   `json:"baseline"`
	RecentAvgPaceSecPerKm *float64           `json:"recent_avg_pace_sec_per_km"`
	LatestRun             *LatestRun         `json:"latest_run"`
	WeekRuns              []RunningWeekRun   `json:"week_runs"`   // this local week, oldest→newest
	WeeklyLoad            []RunningWeekPoint `json:"weekly_load"` // 8 buckets, oldest→newest
	WeeklyDistanceSpark   []float64          `json:"weekly_distance_spark"` // LEGACY
}

type RunningCurrentWeek struct {
	DistanceMeters      float64  `json:"distance_meters"`
	RunCount            int      `json:"run_count"`
	DeltaPctVsPriorWeek *float64 `json:"delta_pct_vs_prior_week"`
	DurationSeconds     int      `json:"duration_seconds"`
	AvgPaceSecPerKm     *float64 `json:"avg_pace_sec_per_km"`   // week aggregate: Σduration / Σkm
	AvgHeartRateBpm     *int     `json:"avg_heart_rate_bpm"`    // DURATION-WEIGHTED over HR-bearing runs
	ElevationGainMeters *float64 `json:"elevation_gain_meters"` // nil when NO run carried altitude
	HeartRateRuns       int      `json:"heart_rate_runs"`       // how many of RunCount contributed
	ElevationRuns       int      `json:"elevation_runs"`
	LongestRunMeters    float64  `json:"longest_run_meters"`
	DaysRun             int      `json:"days_run"`              // distinct local dates, 0–7
}

type RunningWeekRun struct {
	ActivityID          string    `json:"activity_id"`
	Name                *string   `json:"name"`
	StartTime           time.Time `json:"start_time"`
	LocalDate           string    `json:"local_date"`  // YYYY-MM-DD in the user's tz
	DistanceMeters      float64   `json:"distance_meters"`
	DurationSeconds     int       `json:"duration_seconds"`
	AvgPaceSecPerKm     *float64  `json:"avg_pace_sec_per_km"`
	AvgHeartRateBpm     *int      `json:"avg_heart_rate_bpm"`
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

// Trailing 4-week average EXCLUDING the current week — what "normal" means
// for this athlete. nil until there is at least one prior week with a run.
type RunningBaseline struct {
	WindowWeeks         int      `json:"window_weeks"` // 4
	Weeks               int      `json:"weeks"`        // weeks with ≥1 run behind it
	DistanceMeters      *float64 `json:"distance_meters"`
	DurationSeconds     *int     `json:"duration_seconds"`
	AvgPaceSecPerKm     *float64 `json:"avg_pace_sec_per_km"`
	AvgHeartRateBpm     *int     `json:"avg_heart_rate_bpm"`
	ElevationGainMeters *float64 `json:"elevation_gain_meters"`
	RunsPerWeek         *float64 `json:"runs_per_week"`
}
```

Six things about this shape that variants must respect:

1. **`weekly_load` supersedes `weekly_distance_spark`, which is not honest
   enough for anything but a squiggle.** `spark` is a bare `[]float64` with no
   week anchoring, so a chart drawn from it cannot label an axis or tell a
   zero-distance week from a missing one. `weekly_load` carries `week_start` and
   duration and count per bucket. **Use `weekly_load`.** `spark` stays only
   because the shipped card reads it, and dies with that card.
2. **Nil elevation is not zero elevation.** `ElevationGainMeters` is `*float64`
   precisely because a treadmill run and a pancake-flat loop are different facts.
   A variant that renders `nil` as `0 ft` is asserting the user ran on flat
   ground when the truth is the source carried no altitude. Use `ElevationRuns`
   to say *"3 of 4 runs"* rather than silently averaging over the gap.
3. **Same for heart rate.** `AvgHeartRateBpm` is nullable on every run — manual
   entries and many imports carry none. `HeartRateRuns` exists so a variant can
   be honest about coverage instead of quietly under-reporting the week.
4. **Never recompute a server figure.** `AvgPaceSecPerKm`, `Baseline.*`, and
   `DeltaPctVsPriorWeek` are computed server-side over defined windows. A variant
   that means-of-means the per-run paces will disagree with the deep page and
   with itself — a duration-weighted week pace and a naive average of four run
   paces are different numbers, and the long run is exactly what separates them.
5. **`recent_avg_pace_sec_per_km` and `current_week.avg_pace_sec_per_km` are
   different figures and must be labelled differently.** The first is a **30-day**
   aggregate (that is the `10:06` in the screenshot); the second is this week's.
   They will visibly disagree in any week that is faster or slower than the month.
   A variant heroing pace must say which one it is showing.
6. **Pace is stored per kilometre; distance and elevation are stored in metres.**
   Display conversion goes through the existing `formatPaceValue` /
   `formatDistanceValue` / `formatElevationValue(meters, unit)` helpers and the
   user's `DistanceUnit`. Fixtures are metric; **every mockup must be checked in
   both `mi` and `km`** — a pace clock and a 4-digit foot count have different
   widths than their metric equivalents and this card has no room to spare.

**Out of scope for this round:** personal records. `GetUserRunningBestEfforts`
exists and powers `/personal-records`, but it is a *separate query*, and adding it
would forfeit the "no new reads" property that makes this cheap. No idiom below
depends on it. If a PR-flavoured tile is wanted later it is its own DX.

### Color logic — the traps on this surface

Running has **one** hue and it is not up for negotiation:

| Token | Value |
| --- | --- |
| `--discipline-run-bg` | `#16241f` |
| `--discipline-run-fg` | `#9cc7b8` |
| `--discipline-run-dot` | `#7fae9e` |

- **Periwinkle is not a running colour.** `--accent` `#9aa6d6` is app chrome and
  selection/"today" meaning. An activity must never read as selection. (Note the
  design system's own standing contradiction: `--discipline-lift-dot` *is*
  `#9aa6d6`. That is a documented exception for lifting, not licence to reach for
  the accent here.)
- **Do not borrow hike clay for the elevation variant.** Elevation is
  clay-coloured on the *Hiking* surfaces because **activity type owns activity
  colour** — a running tile that draws its climbing in `#b08e77` will read as a
  hike on the grid. `vertical-gain` renders elevation in run sage. This is the
  single most likely colour mistake in the spread.
- **There is no run tonal ramp.** Lifting has `--discipline-lift-1..4` for its
  volume heatmap; running has only the bg/fg/dot triplet. A variant needing
  gradation (`stacked-week`'s segments) derives it by **varying alpha on
  `--discipline-run-dot`**, not by inventing `--discipline-run-1..4`.
- **`--zone-1..5` belongs to heart rate and to exactly one variant.** The
  cool→warm five-tone scale (`#6b7280` → `#cc8077`) is the HR-zone palette used on
  the run detail page. Only `effort-heart` may spend it; a second variant painting
  five-tone anything turns the grid into confetti.
- **Status colour is for the ramp, and it is not an alarm.** A big week is a
  *choice*, not a failure. `load-ramp` may use warning `#d6b87f` for an aggressive
  ramp; **nothing on this tile is ever danger red.** Running more than usual is
  not an emergency, and the app does not scold.
- **Pick one thing to carry colour per variant.** Inside 180px, a card that paints
  both the pace deltas and the effort dots reads as noise.

### States every variant must render

This is where a lazy tile design falls apart. The mockups must show all of these:

- **An ordinary week** — 3–5 runs, one longer than the rest. The common case, and
  the one every variant will look fine in. Not sufficient on its own.
- **Zero runs this week** — **the single most important state, and the one the
  current tile handles worst.** It is Monday morning for everyone, and it is the
  entire week for anyone tapering, injured, or busy. Today the card prints
  `0.0 mi this week` over a sparkline that flatlines into the gutter. Every
  variant must degrade to something true and useful — *"last run Saturday · 12.7
  mi"*, the 4-week baseline, days since — **without promoting last week's numbers
  into this week's slot.** A variant that is an empty frame here has failed the
  brief, and the variant that becomes the default fails it hardest.
- **One run this week** — the sparse case. Any variant that draws a distribution,
  a band, or a spread has to survive n = 1 without looking broken or implying a
  trend from a single point.
- **Indoor / treadmill runs** (`environment: "indoor"`) — no elevation, frequently
  no GPS. A mixed week must not average a treadmill's absent elevation into the
  total as zero, and `vertical-gain` needs a real answer for the user who runs
  exclusively indoors (it may be the variant that person never pins — but it must
  not render a lie).
- **Runs with no heart rate** — very common on manual and older imports. `4 runs ·
  2 with HR` is honest; a duration-weighted average silently computed over half
  the week is not.
- **A brand-new runner** — one run ever, `baseline` nil, `delta_pct_vs_prior_week`
  nil, `weekly_load` almost entirely zeros. **Binding on the default candidate**,
  because this is a new user's first impression of the product's flagship tile.
  No `NaN`, no `+Infinity%`, no chart frame around nothing.
- **A very long single run** — a 12.7-mile long run beside three 3-milers is the
  realistic shape of a training week, and it is what breaks naive scaling: any
  proportional bar, segment, or dot sized linearly will render the shakeouts as
  slivers. Decide the scaling deliberately and show it.
- **Both breakpoints** — full-width single-column on mobile, one-third-width on
  desktop — **and both units**, `mi` and `km`.

### Representative fixture

The headline fixture, in the proposed shape. Week of Mon 2026-07-27, user's unit
`mi`, `America/New_York`. It deliberately mixes an outdoor easy run, an **indoor
treadmill run with no elevation**, a run with **no heart rate**, and a long run —
so three of the awkward states are visible in the default view:

```ts
const running = {
  currentWeek: {
    distanceMeters: 34278,        // 21.30 mi
    runCount: 4,
    deltaPctVsPriorWeek: 10.9,
    durationSeconds: 12977,       // 3:36:17
    avgPaceSecPerKm: 378.6,       // 10:09 /mi  ← this week
    avgHeartRateBpm: 153,         // duration-weighted over the 3 HR-bearing runs
    elevationGainMeters: 274,     // 899 ft, from the 3 outdoor runs
    heartRateRuns: 3,
    elevationRuns: 3,
    longestRunMeters: 20438,      // 12.70 mi
    daysRun: 4,
  },
  recentAvgPaceSecPerKm: 376.6,   // 10:06 /mi  ← 30-day, the screenshot's figure
  latestRun: {
    name: "Saturday long run",
    distanceMeters: 20438, durationSeconds: 7784,
    startTime: "2026-08-01T11:02:00Z",
  },
  weekRuns: [
    { localDate: "2026-07-27", name: "Easy shakeout", distanceMeters: 5633,
      durationSeconds: 2128, avgPaceSecPerKm: 377.8, avgHeartRateBpm: 148,
      elevationGainMeters: 38, environment: "outdoor" },
    { localDate: "2026-07-28", name: null, distanceMeters: 4184,
      durationSeconds: 1490, avgPaceSecPerKm: 356.1, avgHeartRateBpm: 152,
      elevationGainMeters: null, environment: "indoor" },   // treadmill
    { localDate: "2026-07-30", name: "Lunch run", distanceMeters: 4023,
      durationSeconds: 1575, avgPaceSecPerKm: 391.5, avgHeartRateBpm: null,
      elevationGainMeters: 22, environment: "outdoor" },    // no HR
    { localDate: "2026-08-01", name: "Saturday long run", distanceMeters: 20438,
      durationSeconds: 7784, avgPaceSecPerKm: 380.9, avgHeartRateBpm: 156,
      elevationGainMeters: 214, environment: "outdoor" },
  ],
  weeklyLoad: [
    // …8 buckets, oldest→newest, ending with the current week…
    { weekStart: "2026-06-29", distanceMeters: 24140, durationSeconds:  9180, runCount: 3, elevationGainMeters: 156 },
    { weekStart: "2026-07-06", distanceMeters: 29772, durationSeconds: 11310, runCount: 4, elevationGainMeters: 203 },
    { weekStart: "2026-07-13", distanceMeters:     0, durationSeconds:     0, runCount: 0, elevationGainMeters: null }, // down week
    { weekStart: "2026-07-20", distanceMeters: 30900, durationSeconds: 11760, runCount: 4, elevationGainMeters: 231 },
    { weekStart: "2026-07-27", distanceMeters: 34278, durationSeconds: 12977, runCount: 4, elevationGainMeters: 274 },
  ],
  baseline: {
    windowWeeks: 4, weeks: 3,
    distanceMeters: 27358,        // 17.0 mi/wk
    durationSeconds: 10440,
    avgPaceSecPerKm: 381.2,       // 10:13 /mi
    avgHeartRateBpm: 150,
    elevationGainMeters: 198,
    runsPerWeek: 3.75,
  },
  unit: "mi",
};
```

Read as: 21.3 miles across 4 runs on 4 days, **+25% over the 4-week average of
17.0** — a real ramp, driven almost entirely by one 12.7-mile long run that is
59% of the week's distance and 60% of its time. The week ran slightly *faster*
than baseline (10:09 vs 10:13) at a slightly *higher* heart rate (153 vs 150) —
which is the kind of two-figure read the current tile cannot express at all.
Derived figures a variant may show: **+6.9 mi vs the 4-week average**,
**+25.3% load**, **899 ft climbed**, **4 of 7 days**, **long run 12.7 mi (59% of
the week)**, **−4 s/mi vs baseline pace**, **+3 bpm vs baseline HR**.

**Also build**: a **zero-runs** fixture (empty `weekRuns`, `currentWeek` all
zero/nil, `latestRun` five days old, `baseline` intact); a **first-run-ever**
fixture (one run, `baseline` nil, `deltaPctVsPriorWeek` nil, `weeklyLoad` zeros);
an **indoor-only** fixture (three treadmill runs, every `elevationGainMeters`
nil); and the same headline fixture with `unit: "km"`. These are mockups —
**static fixtures that look real are preferred**; do not wire variants to live
activity services.

### Composition — one replaces the tile, the rest are opt-in

The generic `BigNum + Spark + MetaRow` running card is being **retired**. That
makes the spread's constraints sharper than `dx/recovery-tile`'s were:

- **Exactly one variant must be a credible default.** It has to work for a
  first-time user with one imported run, for a week with zero runs, and for
  someone who has never looked at a training metric in their life. Two of the six
  below are drafted as default candidates and are marked as such; the other four
  are explicitly *enthusiast* tiles that assume the user opted in.
- **No two variants may hero the same figure.** Six distinct heroes are assigned
  below — the runs, the distance, the pace, the heart rate, the climbing, the
  ramp — and that assignment is binding. Two variants both printing `21.3` in
  32px type means keeping both prints the same number twice on one grid.
- **Each variant carries its own title and tray description** — not six cards
  titled `RUNNING`. Proposed titles are given per idiom, and the comparison route
  must render each with its own title so pairings can actually be judged.
- **The comparison route must include at least two pair-in-grid mocks** — the
  default candidate beside one enthusiast tile, in a real `CardGrid`, with
  Walking and Steps as neighbours. The whole premise of this DX is that the grid
  looks repetitive; **judging these one at a time in isolation is exactly the
  mistake it exists to correct.** If the winning pair still reads as two green
  squiggles next to Walking's green squiggle, nothing has been fixed.
- **The retirement is cheap, and that is deliberate.** The winning default
  **keeps the existing `TileID` `"running"`** and simply changes what renders
  under it — no stored-layout migration, no `defaultLayout` change, no catalog
  churn, no user's dashboard silently losing a tile. Only the *rendering* is
  retired.
- **A kept enthusiast variant costs**: one `TileId` in both mirrors
  (`lib/dashboard-tiles.ts` and the Go `Catalog` in `internal/dashboard/tiles.go`),
  a catalog entry, a `TileCard` case, and a contract-test update. Proposed ids:
  `running_log`, `running_pace`, `running_effort`, `running_vertical`,
  `running_load`.
- **One implementation note for the downstream SOW**: every running-family tile
  reads the **one** shared `running` section, exactly as the recovery family reads
  one `recovery` section. `buildRunningSection` is currently gated on
  `enabled[TileRunning]`, so it must be re-gated on *any* running-family tile
  being enabled — or a user who pins only *Training Load* gets a nil section.
  Follow the `recoveryFamily` loop in `handler.go`, including its rule that the
  section is emitted under the family key (`"running"`) regardless of which member
  tiles are in the layout.

## Idioms

Six compositions of the same near-black / run-sage / Manrope mini-card, each
heroing a **different** figure and leaning on a different reference. They diverge
along **type scale** (one giant numeral vs even tabular rows vs no numeral at
all), **color logic** (which single dimension carries sage, status, or the zone
scale), and **spacing rhythm** (bar-dominant vs dense ledger vs rail vs
silhouette). All six fit ~180px.

- **stacked-week** — *Proposed title: `Running`. **Default candidate.*** **Heroes
  total distance — but decomposes it.** Keeps the figure users already read
  (`21.3 mi this week`) and replaces the sparkline with a single full-width
  **segmented bar**, one segment per run, widths proportional to distance, so the
  headline visibly resolves into *"one long run and three short ones."* A hairline
  ghost mark on the bar sits at the 4-week baseline, turning `+25%` into a spatial
  fact rather than a percentage nobody reads. The meta row spends its third slot
  on `vs 4-wk avg` instead of the redundant `last`. This is the minimal honest
  replacement: same question as today's card, an actually informative answer.
  Type scale: one 32px numeral over a 20px bar over a small meta row — the same
  vertical rhythm as today, so it does not disturb the grid. Color logic: **sage
  carries everything**, segments stepped by alpha on `--discipline-run-dot`
  (longest run most opaque), baseline mark in faint ink; no status colour at all.
  Spacing: airy, three stacked bands, generous gutters. Borrowing **Strava**'s
  weekly-total header with the segment decomposition it never does. → *"how much,
  and what shape was it?"* — and the only variant that a user who never opts into
  anything still benefits from.

- **week-log** — *Proposed title: `Runs This Week`.* **Heroes the runs
  themselves.** No headline numeral at all — a compact stack of this week's runs
  as dated rows: `Sat · 12.7 mi · 10:13 · 156`, `Thu · 2.5 mi · 10:30 · —`, under
  a quiet `21.3 mi · 4 runs · 3:36` header. Rows are the hierarchy; the week total
  is a caption. A treadmill run carries a small indoor glyph rather than a blank
  elevation column, and a run with no HR shows an em-dash in that column only —
  the row still says everything else it knows. When a week has more than four
  runs, the oldest collapse into a `+2 earlier` line rather than the card growing.
  Type scale: small functional tabular rows, `Geist_Mono` permitted for the
  figures, **no numeral above 14px** — the flattest type on the grid, and the
  strongest possible contrast with today's card. Color logic: near-monochrome ink;
  **sage spent entirely on the per-row pace figure**, brighter when that run beat
  the baseline pace, faint when it did not. Spacing: dense blotter, tight vertical
  rhythm, hairline row separation, near-zero gutters. Borrowing **Strava**'s
  activity-feed row and **Robinhood**'s list density. → *"just show me what I
  actually ran"* — the most literal answer to the brief, and the tile for someone
  who thinks in sessions rather than totals.

- **pace-band** — *Proposed title: `Running Pace`.* **Heroes pace, and puts it
  against the athlete's own normal.** A mid-size pace clock (`10:09 /mi`) with the
  30-day figure beside it as a faint reference, over a **horizontal pace band**:
  the 4-week baseline pace as a centre line, one tick per run positioned by that
  run's pace, faster to the left. Four ticks clustered tightly is a controlled
  week; three easy and one hard is a polarized one — and both are legible in a
  glance without reading a single number. This is the only variant that answers
  *"was this week fast for me?"*, and the per-user centre line is what makes the
  question answerable at all: 10:09 means nothing in the abstract, and everything
  against your own 10:13. Type scale: a medium pace clock as the only prominent
  figure, everything else at caption size — restrained, no giant numeral. Color
  logic: the **band is sage**, ticks are neutral ink except the fastest run of the
  week; nothing else is coloured. Spacing: chart-dominant and horizontally
  centred, the band running nearly full-bleed within the padding. Borrowing
  **Oura**'s "your normal range" band, applied to pace instead of HRV. → *"was
  this week fast, for me?"*

- **effort-heart** — *Proposed title: `Run Effort`.* **Heroes heart rate.** The
  week's duration-weighted average bpm as the figure, with `+3 vs 4-wk` beside it,
  over a **zone rail**: each run a dot on a horizontal bpm axis, coloured by the
  `--zone-1..5` five-tone scale and sized by duration, so a week's intensity
  distribution reads as a spatial pattern. Coverage is stated, never hidden —
  `3 of 4 runs` sits in the caption whenever a run lacks HR. **The honest-cost
  note that shapes this variant**: real time-in-zone is computed per-activity from
  trackpoints at read time (`attachHeartRateZones`), and pulling that for a week
  of runs on every dashboard render is a cost this tile will not pay. This variant
  classifies each run by its **average** HR against the same max-HR reference —
  one cheap read, no trackpoint scan — and its language must reflect that
  ("mostly zone 2 runs"), never claim minutes-in-zone it did not compute. Type
  scale: a medium bpm numeral over a rail; the dots do the work. Color logic:
  **the only variant that spends `--zone-1..5`**, and it spends it nowhere else —
  no sage on this card at all, which is precisely what makes it not look like its
  siblings. Spacing: rail-dominant, one horizontal axis with generous vertical
  breathing room above and below. Borrowing **Garmin Connect**'s zone rail and
  **Whoop**'s single-number intensity framing. → *"how hard did I actually go?"*

- **vertical-gain** — *Proposed title: `Vertical Gain`.* **Heroes climbing.** Total
  feet (or metres) gained this week as the figure, over a **stepped silhouette** —
  one column per run, height by that run's gain — with the week's biggest climb
  called out (`most: 702 ft · Sat`). Elevation is the dimension the running tile
  has never shown despite carrying gain, loss, high, and low on every outdoor run,
  and it is the one that most changes what a mileage number means: 21 miles with
  900 feet is a different week from 21 flat miles. Its awkward states are the
  whole test — a treadmill run contributes **no column and no zero**, it
  contributes a gap, and the caption says `3 of 4 runs` rather than pretending the
  fourth was flat; the indoor-only runner sees a stated *"no outdoor runs this
  week"*, not an empty chart. Type scale: a large figure with a small unit suffix
  over a short silhouette — closer to today's proportions than any other variant,
  deliberately, so it pairs cleanly beneath one. Color logic: **sage fill on the
  silhouette, hike clay explicitly forbidden** (see the traps above); gaps are
  hairline outlines, not filled columns. Spacing: figure over a full-width
  landform, no columns of text, most horizontal of the six. Borrowing **Strava**'s
  elevation profile, reduced to a per-run summary. → *"how much climbing was in
  that?"*

- **load-ramp** — *Proposed title: `Training Load`. **Default candidate.***
  **Heroes the direction, and deliberately demotes this week's total.** One large
  signed figure — `+25%` — for the current week's **time on feet** against the
  4-week baseline, with the plain-language read beneath (*building · 3:36 vs
  2:54 avg*). Under it, an 8-bucket micro-rail of weekly load bars with the
  baseline as a ghost line through them, so the down week three weeks back is
  visible as a deliberate dip rather than a hole. It heroes **duration, not
  distance**, on purpose: an hour is an hour whether it was fast or slow, and it
  is the figure that makes this variant orthogonal to `stacked-week` rather than a
  restatement of it. It is the second default candidate because it is the only
  variant whose headline is still meaningful in a week with **zero runs** —
  `−100% · resting, 5 days since your last run` is a true and useful sentence
  where every other treatment goes quiet. Type scale: **the strongest size
  contrast in the spread** — one big delta numeral over a tiny rail, nothing in
  between. Color logic: **status carries the delta and nothing else** — success
  `#86b39f` for a steady build, warning `#d6b87f` above roughly +10%/week, muted
  for a down week; **never danger red**, and the rail stays neutral ink. Spacing:
  one figure stacked over a single full-width rail, no columns. Borrowing
  **TrainingPeaks**' acute-vs-chronic ramp framing and **Robinhood**'s delta-figure
  headline. → *"am I building or backsliding?"*

## References

In-system, so what to take from each is **structural** — composition, density,
and data legibility, not their palettes or type:

- **Strava** — take two things: the **activity-feed row** (date, distance, pace,
  HR as one scannable line) driving `week-log`, and the **elevation profile**
  reduced to a per-run summary driving `vertical-gain`. Take neither its orange
  nor its chart chrome. It is also the product whose weekly-total header
  `stacked-week` is deliberately improving on.
- **Garmin Connect** — take the **heart-rate zone rail**: intensity as position
  along a coloured axis rather than a number to interpret. Drives `effort-heart`,
  and it is the reason that variant may spend `--zone-1..5`.
- **Whoop** — take the **single-number intensity framing** and its users' training
  in reading one figure against a personal baseline. Reinforces `effort-heart`
  and `load-ramp`; take none of its ring chrome, which the recovery family already
  owns on this grid.
- **TrainingPeaks** — take the **acute-vs-chronic ramp**: this week's load against
  a trailing average, with the ramp rate itself as the headline. Drives
  `load-ramp`. Take its *concept*, not its density — it is a desktop analytics
  product and this is a 180px card.
- **Oura** — take the **"your normal range" band**: a personal baseline drawn as a
  zone the value sits inside or outside. Drives `pace-band`; the same precedent
  that drove `balance-band` in [`dx/recovery-tile`](./recovery-tile.md), applied
  to a performance metric rather than a physiological one.
- **Robinhood** — take the **delta-figure headline** and the **inline per-row
  delta** that turns a short list into a glanceable trend. Drives `load-ramp`'s
  headline and `week-log`'s rows.

## Selection criteria

A note-to-self for the pick, not a rubric the worker optimizes against. When I
compare these I am trying to decide:

- **Which one can be the default?** This is the binding question, because the
  generic card is going away. Would I be happy if this were the *only* running
  tile a new user ever saw — on their first run, on a zero-run week, before they
  know the product has a tray at all?
- **Does the grid stop looking repetitive?** Put the winner beside Walking and
  Hiking in the pair mocks. If it still reads as a third green squiggle, the
  exploration failed regardless of how good the card is on its own.
- **Which two or three compose?** Do the pair mocks read as two facts or as one
  fact printed twice? The heroes are assigned to be distinct — but distinct on
  paper and distinct at 180px on a real grid are different claims.
- **Does "this week vs my normal" come through?** The baseline is the product,
  not decoration. A variant showing this week's figures without anything to
  measure them against has regressed to the current tile with extra steps — which
  is exactly the failure `delta_pct_vs_prior_week` already suffers, shipped in the
  payload and rendered nowhere.
- **What does it say on a zero-run week?** I will see this state constantly —
  every Monday, and every week I skip. If a variant is an empty frame or a
  flatlined chart, it fails the way the current card fails, and I already know
  that annoys me.
- **Does the long run break it?** A 12.7-mile run beside three 3-milers is what a
  real week looks like. Any variant whose scaling turns the short runs into
  slivers has picked the wrong scale.
- **Is it honest about missing data** — treadmill runs with no elevation, imports
  with no HR — without making the card mostly caveats? "3 of 4 runs" is honest;
  four em-dashes is not a design.
- **Does it stay a calm dashboard tile** — within ~180px, quiet enough to sit
  beside Steps and Bodyweight without shouting, and legible at one-third width in
  both `mi` and `km`?
- **Does it still read as Prog Strength v0.4** — near-black, run sage as the
  discipline hue, periwinkle nowhere, no invented tonal ramp, no borrowed hike
  clay — and does it preview the deep `/activities?view=running` page it links
  into, rather than looking like Strava's widget dropped into my grid?

---

> **Lifecycle.** `status:` is editorial — the owner is the dispatch gate. It moves
> `draft` → `exploring` (worker running) → `awaiting_selection` (draft PR open,
> owner deciding) → `selected` / `abandoned`. The worker sets `awaiting_selection`
> on the `dx/running-tile` branch as it opens the PR; the owner sets the terminal
> value when they close it.
>
> **Handoff.** This DX ends at *chosen variants* — plural is expected, and one of
> them is designated the default — not merged code. I open the draft
> `[DX — DO NOT MERGE]` PR, compare the variants on the preview deploy
> (`/design-explore/running-tile`, flag-gated behind
> `NEXT_PUBLIC_ENABLE_DESIGN_EXPLORE`, driven across the ordinary-week /
> zero-runs / first-run-ever / indoor-only fixtures, in **both units**, **and the
> pair-in-grid mocks**), pick the default plus the enthusiast tiles worth keeping,
> tick their boxes, set `status: selected` (noting the winning idioms and which is
> the default), and **close the PR — never merge it.** Then I open a SOW:
> *"implement the `<chosen-idiom>` running tiles from `dx/running-tile`,
> production-quality, conforming to the design system"* — which, unlike the
> recovery family's SOW, **also ships the `RunningSection` payload extension
> specified above** (no new queries; `buildRunning` already holds the data),
> re-points `TileRunning`'s rendering at the winning default, adds one `TileId`
> per kept enthusiast tile to both catalog mirrors with a `TileCard` case each,
> re-gates `buildRunningSection` on any running-family tile being enabled, and
> retires `weekly_distance_spark` along with the card that read it. The mockup
> code is never promoted as-is.
