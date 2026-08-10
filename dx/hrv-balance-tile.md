---
type: dx
status: draft
surface: hrv-balance-tile
idioms:
  - garmin-status-stack
  - drift-field
  - dual-window
  - z-lane
  - instrument-plot
references:
  - Garmin Connect
  - Whoop
  - Oura
  - Bloomberg Terminal
  - Robinhood
scope: in-system
variant_count: 5
repos:
  - prog-strength-web
  - prog-strength-docs
---

# DX: HRV Balance Tile

**Status**: Draft · **Last updated**: 2026-08-09

> A DX (Design Exploration) is the platform's **divergent** work type. Unlike a
> SOW it does not converge on one correct implementation — it produces N
> differentiated visual variants of a single frontend surface, side by side on
> one comparison route, awaiting a human pick. It **never merges** and ships no
> production code; the chosen direction feeds a downstream SOW that builds it for
> real.

> **This DX has a prerequisite.** Three of the five idioms cannot be built for
> real against the payload that ships today — the API has no per-day baseline.
> The refactor is specified in *The API refactor* below and lands as its **own
> SOW before** the downstream implementation SOW. Within this DX the new fields
> are **mocked with static fixtures**, exactly like every other DX figure. Do not
> add `prog-strength-api` to `repos:` — a DX does not touch the backend.

## Context

`hrv_balance` ("HRV Balance") shipped from [`dx/recovery-tile`](./recovery-tile.md)
as one of five recovery tiles, built by
[`sows/recovery-tile-family.md`](../sows/recovery-tile-family.md). Its brief was
the right one — *make the per-user threshold visible rather than asserted* — and
the band-behind-the-series idea survives. The execution has three problems, and
the third is structural.

- **It prints `77.39185 ms`.** The headline renders `todayVal` raw
  (`balance-band.tsx:74`) while the caption two elements below rounds the same
  family of figures. Whoop sends RMSSD as a float; the tile shows all of it. A
  five-decimal heart-rate-variability reading on a calm dashboard is the single
  most conspicuous thing on the grid, and it makes the whole card read as
  unfinished. **The house convention is integer milliseconds** — this is a bug
  the downstream SOW fixes, not a taste question.
- **A polyline is the wrong mark for thirty nightly readings.** HRV is a noisy
  daily *sample*, not a continuous quantity that was passing through the values
  between Tuesday and Wednesday. Connecting the dots draws slopes that do not
  exist and turns an ordinary spread into a seismograph — the shipped tile's
  chart looks alarming on a completely normal month. Garmin and Whoop both plot
  HRV as **discrete points**, and the shipped tile's own gap-splitting logic is
  an admission that the line is a lie whenever a night is missing. The fix is to
  stop drawing the line at all.
- **The band is flat by construction, so it cannot answer the question the tile
  exists to answer.** The API emits *one* `balanced_low` / `balanced_high` pair —
  today's baseline ± 1 SD — so the zone is a rectangle stretched across a month
  of history. That rectangle asserts that the athlete's normal range was
  identical thirty days ago and today. It was not, and **whether it is moving is
  the most useful thing the metric has to say.** A baseline that has climbed
  6 ms over four weeks is a training adaptation; one that has fallen 6 ms is a
  reason to look at sleep and load. The shipped tile is structurally blind to
  both, and no amount of layout work fixes that from the client side.

Garmin solves the third problem directly: its HRV Status card draws a **grey
band that drifts** behind the daily dots, so the window visibly rises or falls
across the four weeks, and each dot is coloured by that day's standing against
*that day's* band. That is the target behaviour. This exploration redesigns the
tile around discrete marks and a moving baseline, and specifies the API work
that makes the moving baseline real rather than interpolated in the browser.

**The selection gate here is single-winner.** Unlike
[`dx/recovery-tile`](./recovery-tile.md), which was explicitly a search for two
or three complementary tiles, this DX rebuilds one existing tile. The catalog is
unchanged: no new `TileId`, no new tray entry, no `TileCard` case. The grid
already carries five recovery tiles (`recovery`, `hrv_balance`, `morning_vitals`,
`recovery_trend`, `recovery_log`); a sixth would dilute the tray, and four of
those five are unaffected by this work.

`scope: in-system`: the foundation is decided (see
[`../design-system.md`](../design-system.md) **v0.4**) — soft near-black ramp,
the single **periwinkle** accent (`#9aa6d6`, app-chrome and meaning only),
**Manrope** with tight numeric tracking, tabular figures, 14px hairline panels,
desaturated status colours. Variants do **not** re-litigate palette, accent, or
type. They diverge on **layout, structure, density, and composition**.

## The surface

`HrvBalanceCard` in
`prog-strength-web/app/(app)/dashboard/_components/recovery/balance-band.tsx`,
one cell of the `grid-cols-1 sm:grid-cols-2 lg:grid-cols-3` `TileGrid`. A variant
owns everything inside the `MiniCard` shell — the uppercase `HRV BALANCE` title,
the `p-4` padding, and the whole-card `next/link` into `/recovery` are chrome it
keeps functional — and composes the card body.

**Space budget: ~240px of card body**, up from the ~180px ceiling the
recovery-tile DX imposed. A scatter whose band visibly drifts needs more than the
92px of chart the shipped tile allots; Garmin's HRV Status card stacks four
registers to do the same job. The grid has **no span support** — `TileGrid` is
uniform thirds with `gap-3` and auto rows — so a taller HRV tile makes its whole
row taller. That is acceptable (the Weather tile already sets a tall row) but it
is a real cost, and a variant that needs 300px is not a candidate.

**Neighbours.** The tile sits in a user-composed grid beside up to fifteen
others, including the four sibling recovery tiles. It must not print the same
figure they do at the same weight: `recovery_trend` already heroes
`▼ 10%` / *falling this week* for `shortAvg` vs `hrvAvg`, and `morning_vitals`
already prints today's HRV in a three-column stat cell. **This tile's job is the
distribution and its drift** — where today sits inside a range that is itself
moving. It is not a second trend tile.

### The two signals, and why they are different

The tile carries **two** trend-shaped facts, and conflating them is the main way
a variant goes wrong:

1. **Today vs my baseline** — `zScore`, `status`, and the `trend` field
   (`shortAvg` vs `hrvAvg`, i.e. this week against this month). Already shipped.
   Answers *"was last night normal for me?"*
2. **My baseline vs my past baseline** — is the *range itself* climbing or
   sinking over the month? **Not currently expressible.** Answers *"is my normal
   changing?"*

Garmin's card shows both: the dots-against-the-band give (1), the visible slope
of the band gives (2). The existing `trend` field is emphatically **not** (2) —
its windows overlap by design (see `internal/recoverytrend/doc.go`), so it
measures today's neighbourhood against the month it sits inside. A variant that
labels `trend: "falling"` as "your baseline is falling" is stating something the
payload does not say.

### The API refactor

The moving band and the per-dot colouring are **server figures**, not client
arithmetic. The standing rule holds — *never recompute a server figure* — so the
downstream work begins with a `prog-strength-api` SOW that extends
`internal/recoverytrend`. It is small and additive.

**Rolling pass.** `Compute` gains a companion that walks the chart window and,
for each day *i*, computes the baseline over the trailing `baseline_window_days`
ending at day *i−1* — the same exclude-today rule the scalar path already uses,
applied per day. The read path's fetch widens from 31 days to
`chart_window_days + baseline_window_days` = **60 days**. The compute is
30 × 30 ≈ 900 float operations per request; no new tables, no new queries beyond
the wider date range on the existing `whoop_recovery` read.

**Wire additions** (all additive, all nullable, no existing field changes):

```jsonc
"days": [
  {
    "date": "2026-08-09",
    "hrv_rmssd_milli": 77.4,
    // NEW — this day's own band, null until it has min_baseline_days behind it
    "baseline_avg":  88.2,
    "balanced_low":  68.1,
    "balanced_high": 108.3,
    "z_score":       -0.53,
    "status":        "balanced"
  }
],
// NEW — the baseline against its own past. NOT the same as hrv.trend.
"baseline_trend": {
  "direction": "rising",   // rising | falling | steady | unknown
  "delta_ms":  6.4,        // baseline now − baseline over_days ago
  "from_avg":  81.8,       // the baseline it moved from
  "over_days": 28
}
```

**Config additions** — non-secret tuning knobs, so `config.toml` under
`[recovery]`, never env vars:

```toml
chart_window_days   = 30    # days of daily marks the tile draws
baseline_drift_days = 28    # how far back the baseline is compared against
baseline_drift_z    = 0.35  # |delta| must exceed this many SDs to read rising/falling
```

`baseline_drift_z` keeps the drift verdict SD-relative like every other threshold
in the engine, so a wide-spread athlete does not get a "rising" verdict from
noise a narrow-spread athlete would not.

**Two invariants the SOW must test**, because they are exactly what will drift
apart later:

- `days[last].status == hrv.status`, and `days[last].balanced_low/high` equal the
  scalar `hrv.balanced_low/high`. The scalar block stays the authority for
  "today"; the series must agree with it, never quietly disagree by an epsilon.
- `baseline_trend.direction` is derived only from baselines, never from
  `shortAvg`. It is a different question from `hrv.trend` and the two are allowed
  to point opposite ways — a rising baseline with a suppressed morning is a real
  and interesting state, not a bug.

**This introduces a render state that does not exist today: a band that starts
part-way across the chart.** A user with 40 nights of history has a baseline for
the last 26 chart days and none for the first 4, so the band begins mid-card and
the earliest dots have no `status`. Every variant must handle it — see *States*.

### The data a variant renders

`RecoveryView` in `prog-strength-web/lib/dashboard.ts`, with the fields this DX
adds marked. The three existing optionals (`days?`, `baseline?`, `hrv?`) still
require a guard — guard once at the top and render the calibrating state, never
`!`-assert.

```ts
type RecoveryDayPoint = {
  date: string;                  // YYYY-MM-DD, local
  restingHr: number | null;
  recoveryScore: number | null;
  hrv: number | null;            // ms
  // NEW — this day's own band. null until that day has 14 samples behind it.
  baselineAvg: number | null;
  balancedLow: number | null;
  balancedHigh: number | null;
  zScore: number | null;
  status: "balanced" | "elevated" | "suppressed" | "unknown";
};

// NEW — the baseline against its own past. Distinct from RecoveryHrvView.trend.
type RecoveryBaselineTrendView = {
  direction: "rising" | "falling" | "steady" | "unknown";
  deltaMs: number | null;
  fromAvg: number | null;
  overDays: number;
};
```

Unchanged and still binding: `days` is the honest date-aligned series and `spark`
is not (it omits missing days and destroys alignment — never draw from it);
server figures are displayed as received; the band is **this athlete's own**
baseline ± their own SD, never a population "normal HRV" range.

### Marks, not lines — the technical constraints

Two of these are the difference between a good mockup and a broken one.

- **`preserveAspectRatio="none"` must go.** The shipped SVG
  (`balance-band.tsx:93`) uses a 260×92 viewBox stretched non-uniformly to the
  card width. `vectorEffect="non-scaling-stroke"` rescues the polyline, but
  **nothing rescues a `<circle>`** — it is scaled into an ellipse. With one
  3.5px dot this is invisible; with thirty large dots it is the first thing you
  see, and it will look like a rendering bug. Use `meet` with a fixed aspect,
  measure the container, or place the marks as positioned HTML.
- **Dot pitch is a design axis, and the desktop third is the tight case.** At a
  one-third cell the chart is ~260–300px wide. Thirty marks is a **~9px pitch** —
  workable at r≈3.5 with a 1px surface-coloured ring to separate touching
  neighbours, but nowhere near Garmin's generous ~10px dots. **Fourteen marks
  gives an ~18px pitch** and genuinely large Garmin-scale points at the cost of
  half the history. Variants should not all make the same choice; the spread
  below deliberately splits.
- **No interactive controls.** The whole card is a `next/link`. A Whoop-style
  `6m/3m/1m/2w/1w/1d` range switcher cannot live here without swallowing the
  navigation — range selection belongs on `/recovery`, and a variant that mocks
  a switcher is mocking something the tile cannot have.
- **Never interpolate a gap.** With marks this is free — a missing night is
  simply an absent dot. Do not draw a placeholder at zero, and do not close the
  band across the gap either: the band is defined on days with a baseline, and
  its polygon should break where the baseline is null.

### Representative fixture

The headline fixture is the state the shipped tile cannot express: **a balanced
morning inside a baseline that has been climbing.** `days` is abridged for
legibility — **build it out to all 30 entries**, oldest→newest, ending today,
drawing HRV from a distribution whose mean walks from ~82 to ~88 across the
window with SD ≈ 20, and including **at least one interior all-null day** so the
gap case is visible by default.

```ts
const recovery: RecoveryView = {
  restingToday: 59,
  recoveryScore: 61,
  hrvToday: 77.4,                 // NOTE the float — render as 77 ms
  spark: [58, 59, 57, 59],        // legacy, gap-omitting — do not draw from this
  days: [
    // …extend to 30 entries, oldest→newest, ending today…
    { date: "2026-08-03", restingHr: 57, recoveryScore: 72, hrv: 66.2,
      baselineAvg: 85.9, balancedLow: 66.4, balancedHigh: 105.4, zScore: -1.01, status: "suppressed" },
    { date: "2026-08-04", restingHr: 56, recoveryScore: 74, hrv: 87.1,
      baselineAvg: 86.2, balancedLow: 66.6, balancedHigh: 105.8, zScore: 0.05, status: "balanced" },
    { date: "2026-08-05", restingHr: 55, recoveryScore: 81, hrv: 100.6,
      baselineAvg: 86.6, balancedLow: 66.9, balancedHigh: 106.3, zScore: 0.71, status: "balanced" },
    { date: "2026-08-06", restingHr: null, recoveryScore: null, hrv: null,
      baselineAvg: 87.0, balancedLow: 67.2, balancedHigh: 106.8, zScore: null, status: "unknown" }, // gap
    { date: "2026-08-07", restingHr: 58, recoveryScore: 66, hrv: 78.1,
      baselineAvg: 87.4, balancedLow: 67.5, balancedHigh: 107.3, zScore: -0.47, status: "balanced" },
    { date: "2026-08-08", restingHr: 58, recoveryScore: 69, hrv: 83.2,
      baselineAvg: 87.8, balancedLow: 67.8, balancedHigh: 107.8, zScore: -0.23, status: "balanced" },
    { date: "2026-08-09", restingHr: 59, recoveryScore: 61, hrv: 77.4,
      baselineAvg: 88.2, balancedLow: 68.1, balancedHigh: 108.3, zScore: -0.53, status: "balanced" }, // today
  ],
  baseline: {
    windowDays: 30,
    restingHrAvg: 57.2, restingHrDays: 28,
    hrvAvg: 88.2, hrvStdDev: 20.1, hrvDays: 28,
    recoveryScoreAvg: 70.4, recoveryScoreDays: 28,
  },
  hrv: {
    status: "balanced",
    balancedLow: 68.1, balancedHigh: 108.3,
    zScore: -0.53,
    trend: "steady",
    shortAvg: 84.9,
  },
  baselineTrend: {
    direction: "rising",
    deltaMs: 6.4,
    fromAvg: 81.8,
    overDays: 28,
  },
};
```

Read as: last night's 77 ms sits half a standard deviation below an 88 ms
baseline — a perfectly ordinary morning, comfortably inside the 68–108 ms band —
while the baseline itself has climbed **6 ms over four weeks**, from 82. The
shipped tile renders this as `77.39185 ms Balanced` over a flat rectangle and
says nothing at all about the 6 ms. That gap is the whole exploration.

**Also build these fixtures** — the comparison route must be drivable across all
of them:

- **`falling`** — the mirror case, and the one with product weight. Baseline
  down from 94.3 to 86.2 over 28 days (`direction: "falling"`, `deltaMs: −8.1`) with
  today at 79 ms reading `balanced` against the *lowered* band. A tile that shows
  "Balanced" in green while the athlete's normal quietly sinks is the failure
  mode this redesign exists to prevent; every variant must be judged on it.
- **`suppressed`** — today 61 ms, `zScore: −1.36`, well under the band, with a
  steady baseline. The dramatic state. Must read as *true and slightly
  concerning*, never as an alarm.
- **`partial-band`** — 40 nights of history: `days[0..3]` carry
  `baselineAvg: null` and `status: "unknown"`, the rest are populated. The band
  begins part-way across the chart and the earliest dots are uncoloured. **New
  with this refactor and easy to render as a broken chart** — get it right.
- **`calibrating`** — `hrvDays: 9`, every average, bound, and z null, both
  `status` and `direction` `unknown`. No band anywhere, no chart frame around
  nothing, no `NaN`. Every new Whoop user lives here for two weeks.
- **`no-reading-yet`** — today's metrics null, the window and the baseline
  intact. The state every user is in each morning until the webhook lands. The
  29 prior dots, the band, and the drift are all still true and still printable;
  **a variant that degrades to em-dashes here has failed the brief.**

These are mockups — **static fixtures that look real are preferred**. Do not wire
variants to live recovery services, and do not implement the rolling baseline in
the browser to generate them; hand-author the numbers so the band moves the way
the fixture claims.

## Idioms

Five compositions of the same near-black / periwinkle / Manrope mini-card. All
five plot **discrete marks** and all five show a **moving baseline** — that is the
shared brief, not a divergence axis — so they are forced apart on **type scale**
(one 28px figure vs no headline at all vs 10px mono furniture everywhere),
**colour logic** (which element carries status: the dots, the band, or a single
word), and **spacing rhythm** (stacked registers vs full-bleed vs split panels vs
charted margins). Two draw 30 marks at ~9px pitch, two draw 14 at ~18px, and one
abandons the ms axis entirely.

- **garmin-status-stack** — **Heroes the verdict, in four registers.** The most
  literal translation of Garmin's HRV Status card into the system: a status dot
  and word on the first line; the **7-day average** in ms as the big figure with
  a small `7d avg` caption beneath it (Garmin heroes the stable figure, not last
  night's noise — a deliberate departure from the shipped tile, and worth testing
  precisely because it disagrees with it); a slim **distribution gauge** — a
  segmented horizontal bar marking this athlete's own −2/−1/+1/+2 SD zones with a
  tick showing where the 7-day average falls; and, at the bottom, a compact 30-day
  scatter with the drifting band, ~60px tall. Type scale: one ~28px figure over
  three registers that never exceed 13px. Colour logic: **the dots and the gauge
  carry status** — in-band marks in muted ink, out-of-band in warning or accent,
  the gauge's zones in desaturated success/warning; the band itself stays neutral.
  Garmin's saturated red/orange/green must be **re-toned to the system's
  desaturated statuses** — importing their traffic light is the obvious trap here.
  Spacing: four hairline-separated horizontal registers, tight and even.
  Borrowing **Garmin Connect**'s HRV Status card structurally, register for
  register. → *"give me Garmin's card, in our language."*

- **drift-field** — **Heroes the chart, and prints almost no type at all.** No
  headline figure, no status word, no caption row: the body is one edge-to-edge
  chart from padding to padding. Thirty marks at ~9px pitch over a band drawn as
  a genuine drifting polygon — an upper and a lower path, not a rectangle — with
  the baseline as a faint spline through its middle. Exactly **two** pieces of
  text, both anchored into the plot: a small tag beside today's mark reading
  `77 ms`, and a tag at the band's right edge reading `▲ +6 ms · 4w`. The drift
  is not described anywhere; you read it off the slope. Type scale: the most
  extreme suppression in the spread — two 11px mono tags and the `MiniCard`
  title, nothing else. Colour logic: the **band is neutral** (`--surface-2` at low
  alpha), the marks carry everything — muted when inside, status-coloured when
  outside, today filled solid with a surface-coloured ring. Spacing: full-bleed,
  zero internal gutters, no registers; the chart *is* the card. Borrowing
  **Oura**'s normal-range zone and **Whoop**'s full-width chart, with the
  annotation discipline of a good print figure. → *"show me the shape, don't
  narrate it."*

- **dual-window** — **Heroes the step between two windows.** Splits the body into
  two side-by-side panels, `PREV 2W` and `LAST 2W`, each drawing 14 marks at a
  generous ~18px pitch — Garmin-scale dots — over **its own band segment drawn at
  its own level**. It spends the last 28 of the window's 30 days; the two oldest
  are dropped rather than making the panels uneven. Because the two bands sit at different heights, the drift is a
  literal visible offset between the panels rather than a slope you have to
  estimate. Between them, one figure: `82 → 88 ms ▲ +6`. This is the only variant
  where the two-week comparison is structural rather than annotated, and the only
  one that gets large dots *and* four weeks of history. Type scale: one ~22px
  delta figure and two 10px uppercase panel labels; no big ms figure anywhere.
  Colour logic: **the bands carry the story** — the later panel's fill tinted
  toward success when rising and warning when falling, the marks left near-mono
  so they do not compete. Spacing: two symmetric columns with a hairline gutter,
  equal weight, nothing full-width but the delta. Borrowing **Garmin Connect**'s
  4-week chart, which already reads as two banded panels, and **Robinhood**'s
  `from → to` delta framing. → *"how did this fortnight compare to the last?"*

- **z-lane** — **Heroes the deviation, and abandons the ms axis.** Plots each
  day's `zScore` against a **fixed** ±1 SD corridor with a zero line through the
  centre, so the corridor never moves and the marks do all the moving. Its
  argument: once the baseline drifts, two dots at the same *height in ms* mean
  different things, and only the detrended view makes a Tuesday in July directly
  comparable to a Tuesday in August. The baseline's own movement is then shown
  **separately and honestly** — a 16px strip beneath the lane carrying the
  baseline as a thin line on its own scale, labelled `baseline 82 → 88 ms`. A
  status word sits top-left as the only large type. Type scale: one ~18px word,
  10px labels, no numeral above 13px. Colour logic: an **intensity ramp on the
  marks** — muted near zero, deepening to warning as |z| grows below, to accent
  above — with the corridor nearly invisible; the single status word is the only
  other coloured thing. Spacing: one dominant lane over one thin subordinate
  strip, strict horizontal registers, no columns. Borrowing **Whoop**'s
  deviation-from-normal framing. → *"how unusual was each night, on one scale?"*
  **The deliberate outlier in this spread** — see *Selection criteria* for the
  risk it is being tested against.

- **instrument-plot** — **Heroes the chart furniture.** A real miniature chart
  rather than a sparkline: a labelled y-axis in milliseconds (three or four
  ticks), dated x-ticks at the week boundaries, faint gridlines, the drifting
  band behind, and thirty hollow marks over it — the Whoop web HRV chart shrunk
  to tile scale and re-toned. A small right-aligned annotation block gives
  `baseline 88 ms` and `▲ +6 · 4w`. This is the only variant that **spends space
  on axes**, and the argument for it is that the shipped tile's chart is
  unreadable precisely because it has no scale: with a labelled axis, 77 against
  a 68–108 band is a fact you can check, not a position you have to trust.
  Type scale: the densest on the grid — 10px `Geist_Mono` labels everywhere and
  **no headline figure at all**. Colour logic: near-monochrome — gridlines at
  ~0.06 alpha, band fill barely above the surface, marks hollow in muted ink, and
  **only today's mark filled in the status colour**. One coloured pixel on the
  whole card. Spacing: honest chart margins — a ~24px left axis gutter and ~14px
  bottom tick row — the opposite of `drift-field`'s full bleed. Borrowing the
  **Whoop** web HRV chart's labelled axes and hollow points, with **Bloomberg
  Terminal**'s conviction that furniture is information. → *"give me the real
  chart, small."*

## References

In-system, so what to take from each is **structural** — composition, mark
design, and data legibility, not their palettes or type:

- **Garmin Connect** — the north star for this DX. Take three things
  specifically: the **drifting grey baseline band** behind discrete dots (the
  entire reason this exploration exists); the **per-dot colouring against that
  day's own band**, including a distinct marker for a day that falls far outside
  it; and the **four-register card structure** — status word, figure, range
  gauge, chart. Do **not** take its saturated red/orange/green, and do not take
  its light theme. Drives `garmin-status-stack` and `dual-window`.
- **Whoop** — take the **labelled-axis chart with hollow daily points** from the
  web app's HRV view (y-axis in ms, weekday ticks, one point per night, no
  connecting emphasis), and its framing of a reading as a **deviation from your
  own normal** rather than an absolute. Drives `instrument-plot` and `z-lane`.
  Do **not** take its range switcher — the tile cannot have controls.
- **Oura** — take the **"your normal range" zone**: a personal band a day sits
  inside or outside, rendered as territory rather than as two numbers to compare
  mentally. Drives `drift-field`.
- **Robinhood** — take the **`from → to` delta framing** that turns two figures
  into a movement. Drives `dual-window`'s headline.
- **Bloomberg Terminal** — take the conviction that **axis furniture is
  information, not clutter**, at small sizes. Drives `instrument-plot`'s density.

## Selection criteria

A note-to-self for the pick, not a rubric the worker optimises against. When I
compare these I am trying to decide:

- **Can I see the baseline move?** This is the whole point. Not "is the drift
  labelled somewhere" — can I *see* it, in the shape, on the balanced fixture
  where the movement is 6 ms and undramatic? If I have to read a tag to know,
  the chart is not doing the work.
- **Does the falling fixture change how I feel about the card?** A sinking
  baseline under a green "Balanced" morning is the case the shipped tile hides.
  If a variant renders `falling` and `rising` almost identically, it has solved
  the visual brief and missed the product one.
- **Do the marks read as nights?** Discrete, countable, one per morning — with a
  missing night visibly missing. If a variant's dots merge into something
  line-shaped at a one-third-width cell, the ~9px pitch has beaten it and it
  should have drawn fourteen.
- **Is it still obviously *my* range?** The band is my baseline and my spread.
  The moment it reads as a generic healthy-HRV zone, the redesign has cost me the
  one thing the shipped tile got right.
- **Does it stay distinct from `recovery_trend` and `morning_vitals`?** They may
  be on the same grid. If this tile's biggest figure is a trend delta, I own two
  trend tiles.
- **`z-lane` specifically: does abandoning milliseconds cost more than the
  comparability buys?** It is in the spread because the detrended view is the
  analytically honest one, and against the brief's own words — *clearly establish
  and show the user baseline* — it is the most at risk. I want to see whether the
  drift strip is enough to keep the baseline concrete, or whether a card with no
  ms on it stops feeling like it is about me. If it fails, it should fail here
  rather than in my head.
- **`instrument-plot` specifically: is it legible or merely dense?** 10px axis
  labels on a one-third cell is either the most useful card on the grid or
  unreadable. There is no middle outcome and I will know on sight.
- **What does it say at 7am?** Before the webhook lands there is no today mark.
  Twenty-nine dots, a band, and a drift are still true. A variant that looks
  broken without today's point fails the same way the deep page's hero did.
- **Does the partial band look intentional?** A band that starts a third of the
  way across should read as *"your range wasn't established yet"*, not as a
  clipping bug.
- **Is it calm?** It sits next to Steps and Blood Pressure. Thirty coloured dots
  can very easily become the loudest thing on the dashboard; the balanced fixture
  should be nearly monochrome.
- **Does it still read as Prog Strength v0.4** — near-black, periwinkle as meaning
  not chrome, desaturated status, no imported Garmin traffic light — and does it
  preview the `/recovery` page it links into?

---

> **Lifecycle.** `status:` is editorial — the owner is the dispatch gate. It moves
> `draft` → `exploring` (worker running) → `awaiting_selection` (draft PR open,
> owner deciding) → `selected` / `abandoned`. The worker sets `awaiting_selection`
> on the `dx/hrv-balance-tile` branch as it opens the PR; the owner sets the
> terminal value when they close it.
>
> **Handoff.** This DX ends at *one chosen variant*, not merged code. I open the
> draft `[DX — DO NOT MERGE]` PR, compare the variants on the preview deploy
> (`/design-explore/hrv-balance-tile`, flag-gated behind
> `NEXT_PUBLIC_ENABLE_DESIGN_EXPLORE`, driven across the **rising / falling /
> suppressed / partial-band / calibrating / no-reading-yet** fixtures), tick the
> winner's box, set `status: selected`, and **close the PR — never merge it.**
>
> Then **two** SOWs, in order:
>
> 1. **`prog-strength-api`** — the rolling per-day baseline and `baseline_trend`
>    specified in *The API refactor*, with the two agreement invariants under
>    test, plus the `RecoveryDayPoint` / `RecoveryBaselineTrendView` adapter
>    fields in `prog-strength-web/lib/dashboard.ts`. Ships alone; the tile keeps
>    rendering as-is against it.
> 2. **`prog-strength-web`** — *"rebuild the `hrv_balance` tile per the
>    `<chosen-idiom>` variant from `dx/hrv-balance-tile`, production-quality,
>    conforming to the design system"* — replacing `HrvBalanceCard`'s body, fixing
>    the unrounded-millisecond headline, and dropping the non-uniform
>    `preserveAspectRatio`. No catalog change: same `TileId`, same title, same
>    tray entry, same `href`. The mockup code is never promoted as-is.
