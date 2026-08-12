---
type: dx
status: selected
selected_idioms:
  - sorted-strip
surface: resting-hr-tile
idioms:
  - delta-ledger
  - departure-area
  - month-grid
  - true-scale-marks
  - sorted-strip
references:
  - Garmin Connect
  - Oura
  - Whoop
  - Apple Health
  - Bloomberg Terminal
  - Robinhood
scope: in-system
variant_count: 5
repos:
  - prog-strength-web
  - prog-strength-docs
---

# DX: Resting HR Tile

**Status**: Selected · **Last updated**: 2026-08-12

> **Selected:** `sorted-strip` — the last thirty mornings sorted low to high as a
> strip of thin ticks, today's filled and labelled, the 30-day average a second
> dashed tick, under the caption `4th lowest of your last 30` (DX comparison PR
> Prog-Strength/prog-strength-web#164, closed un-merged). The deliberate outlier
> in the spread, and the only variant that does not primarily answer "over time"
> — it won on the two things a ranking buys that a chart cannot: it is the only
> variant whose main graphic **survives `calibrating` intact** (a rank needs no
> baseline, only a distribution), and it stays visibly distinct from
> `hrv_balance` beside it because it draws no line. `flat-month` is boring by
> construction here: there is no axis to auto-scale, so a 48–50 month reads as
> `48 lowest` / `50 highest` and nothing else. Built for real by
> [`sows/resting-hr-tile.md`](../sows/resting-hr-tile.md), which corrects four
> defects found by reading the mockup against the shipped code — integer-first
> ranking, the colour gate below, label de-confliction, and newest-first recent
> rows.

> A DX (Design Exploration) is the platform's **divergent** work type. Unlike a
> SOW it does not converge on one correct implementation — it produces N
> differentiated visual variants of a single frontend surface, side by side on
> one comparison route, awaiting a human pick. It **never merges** and ships no
> production code; the chosen direction feeds a downstream SOW that builds it for
> real.

> **This is a NEW tile, not a redesign.** Unlike
> [`dx/recovery-log-tile`](./recovery-log-tile.md) and
> [`dx/hrv-balance-tile`](./hrv-balance-tile.md), there is no shipped surface to
> indict — `resting_hr` does not exist in `lib/dashboard-tiles.ts` yet. The
> downstream SOW therefore **adds a catalog entry**, which both previous recovery
> DXs explicitly did not.

> **No prerequisite.** Every field these variants need already ships. Do **not**
> add `prog-strength-api` to `repos:` — but read *The verdict this tile does not
> have* before assuming that means the tile can say whatever the others say. It
> cannot, and that constraint is the most interesting thing about this surface.
>
> **Correction, recorded after selection.** The first sentence is right and the
> second is wrong, and the distinction matters for every new-tile DX after this
> one. No new *data* is needed — but the tile **catalog is server-owned**.
> `internal/dashboard/tiles.go` is the source of truth and
> `lib/dashboard-tiles.ts` is its mirror; `layout_handler.go` rejects an unknown
> id on write with a `400`, `Layout.Normalize` drops it on read, and the contract
> tests on both sides assert the id set and order. A web-only catalog entry
> therefore cannot be added by a user at all. `sows/resting-hr-tile.md` carries
> `prog-strength-api` in its `repos:` for exactly this, and for one thing with no
> test behind it: `resting_hr` must also join `handler.go`'s `recoveryFamily`
> slice, or a layout whose only recovery tile is this one gets no `recovery`
> section built and the tile shows a connect CTA to an already-connected user.
> **A DX that adds a tile always touches the API, even when it needs no new
> field.**

## Context

The recovery section currently carries four tiles, and **three of them print
resting heart rate.** `recovery` prints today's as a contributor row with a
`vs 30d` chip. `morning_vitals` prints today's as one of three equal cells, with
the same chip. `recovery_log` prints the 30-day average in its seam (`53 bpm`)
and three recent mornings' values in its detail rows (`50 bpm`, `47 bpm`,
`59 bpm`).

So the obvious objection to a fifth tile is that resting HR is the
best-covered metric on the dashboard and does not need its own card. That
objection is wrong, and the reason it is wrong is the whole brief:

**All three of those answer "what is my resting HR *this morning*, and is it
above or below normal?" None of them answers "what has my resting heart rate
been *doing*, and what were the actual numbers?"** The longest RHR history
anywhere in the product is three days. `hrv_balance` gives a month of one
metric with its own baseline; resting HR — the metric with the strongest claim
to a month-long read, because it is the one that moves slowly and means
something when it shifts — gets three rows in a corner of another tile.

That matters more for this metric than for the other two. A resting heart rate
that has climbed from 48 to 56 and stayed there for five days is the single
most legible early signal a wearable produces: illness incubating, a training
block that has tipped from productive to grinding, alcohol, poor sleep debt
accumulating. It is a *sustained shift*, and a sustained shift is invisible in
any window three days wide. Sunday's 59 in the fixture below reads as one bad
night on `recovery_log`. Across a month it might be the fourth day of a climb,
and nothing in the product can currently tell you which.

What a good tile unlocks: the user can answer *"is my resting heart rate
creeping up?"* — and read the actual numbers, which for this metric they can,
because a resting HR is always two digits and never needs rounding drama or a
unit that wraps.

`scope: in-system`: the foundation is decided (see
[`../design-system.md`](../design-system.md) **v0.4**) — soft near-black ramp,
the single **periwinkle** accent (`#9aa6d6`, app-chrome and meaning only),
**Manrope** with tight numeric tracking, tabular figures, 14px hairline panels,
desaturated status colours. Variants do **not** re-litigate palette, accent, or
type. They diverge on **layout, structure, density, and composition**.

## The surface

A new `MiniCard` tile, one cell of the `grid-cols-1 sm:grid-cols-2
lg:grid-cols-3` `TileGrid`, rendered from the same `RecoveryView` its four
siblings read. Proposed identity for the downstream SOW: `TileId` `resting_hr`,
title **`RESTING HR`**, `href` `/recovery` — the family's shared deep page.

A variant owns everything inside the `MiniCard` shell; the uppercase title, the
`p-4` padding, and the whole-card `next/link` are chrome it keeps functional.

**Space budget: ~180px of card body, ~260px ceiling.** `TileGrid` has no span
support (uniform thirds, `gap-3`, auto rows), so a taller tile makes its entire
dashboard row taller, including unrelated tiles beside it. The shipped
`recovery_log` lands at ~154px and `hrv_balance` is the tallest card on the
grid; a new tile that pushes past 260px is not a candidate.

### The data a variant renders

`RecoveryView` in `prog-strength-web/lib/dashboard.ts`. The optionals (`days?`,
`baseline?`) require a guard — guard once at the top and render the calibrating
state, never `!`-assert.

```ts
type RecoveryDayPoint = {
  date: string;                  // YYYY-MM-DD, local
  restingHr: number | null;      // bpm — a FLOAT on the wire; render as integer
  recoveryScore: number | null;
  hrv: number | null;
  baselineAvg: number | null;    // HRV's baseline. NOT resting HR's.
  balancedLow: number | null;    // HRV's band.
  balancedHigh: number | null;   // HRV's band.
  zScore: number | null;         // HRV's z.
  status: "balanced" | "elevated" | "suppressed" | "unknown";  // HRV's status.
};
```

`days` is 31 entries, oldest→newest, **date-aligned with nulls preserved** —
every calendar date is present whether or not Whoop had a reading. **Never draw
from `spark`** (`resting_hr_spark` on the wire): it omits missing days and
destroys the alignment every variant here depends on. It is legacy and it is a
trap.

The only two other fields that exist for this metric:

```ts
baseline.restingHrAvg: number | null;   // 30-day trailing mean, EXCLUDING today
baseline.restingHrDays: number;         // sample size behind it
```

**`restingHrAvg` is gated independently of the other metrics.**
`internal/recoverytrend` collects a separate sample per metric and emits each
average only once *that metric* has `MinBaselineDays` (14) non-null readings
behind it, reporting `restingHrDays`, `hrvDays`, and `recoveryScoreDays`
separately. A user can easily have a resting-HR baseline while HRV is still
calibrating, or the reverse. **Gate this tile's calibrating state on
`restingHrAvg` / `restingHrDays` and on nothing else** — reading `hrvDays` here
is the exact bug the `recovery_log` SOW had to correct in its mockup.

**Render bpm as an integer.** `resting_heart_rate` is a `*float64` in the API's
DTO, so nothing guarantees it arrives whole. This is the same lesson
`96.10188 ms` taught on `recovery_log`, arriving early enough to design around.

### The verdict this tile does not have

**This is the central constraint of the surface and every variant must be built
inside it.**

Resting heart rate has **no band, no z-score, no status, and no trend** on the
wire. `recoverytrend.Compute` derives all of that for HRV only — it computes an
RHR *mean* and stops. There is no `restingHrStdDev`, no `restingHrStatus`, no
short-window average.

What follows, precisely:

- **Allowed: a signed delta of a value against a server baseline.** This is
  established house practice, not a loophole — `readiness-verdict.tsx` and
  `three-dial-vitals.tsx` both compute `value − baseline` and print
  `vs 30d ±n`, and both docstrings say so explicitly. `48 bpm · −5 vs 30d` is a
  true statement built from two server figures.
- **Allowed: position.** Placing a mark by its distance from the baseline is
  mapping an already-computed value onto a pixel, which is what a chart is.
- **Forbidden: re-averaging a series.** No client-side 7-day RHR average. The
  server emits `short_avg` for HRV and not for RHR, and computing one here is
  the "never re-average" rule broken outright. A variant that wants to hero a
  stable recent figure **cannot**, and must find another hero.
- **Forbidden: classification.** No client-side threshold deciding that +4 bpm
  is "elevated" and +2 is "normal". That is the `day.hrv − baseline.hrvAvg`
  ±3 ms bug `dx/recovery-log-tile` indicted, and there is no `day.status` to
  fall back on here because RHR has none.

So **this tile has no status vocabulary.** `recovery_log` has Whoop's
green/yellow/red. `hrv_balance` has balanced/elevated/suppressed. This one has
nothing, and it is the first recovery tile that must carry its meaning almost
entirely without colour. That is a genuinely unusual brief and it is the most
interesting thing to explore here: *how much can a tile say when it is not
allowed to render a judgement?*

### Colour: a fixed contract, not a divergence axis

Because there is no server verdict, the colour budget is tiny and spending it
needs a rule. All five variants use this one:

| Situation | Treatment |
| --- | --- |
| A reading **above** the 30-day average | `--warning`, at text or mark weight |
| A reading **at or below** the average | Neutral ink — `--foreground` / `--muted` |
| An absent morning | `--surface-2`, as an absence |

> **Correction, recorded after selection.** This table and the `sorted-strip`
> idiom below disagree: the idiom says `--warning` appears "only if today's tick
> falls in the upper third of the athlete's own month", which is a different rule
> and leaves a morning that is above the average but outside the top third
> uncoloured. `sows/resting-hr-tile.md` resolves it in favour of **this table**,
> chiefly because the selected card draws the average as a tick — so under the
> upper-third rule today's mark could sit visibly right of it and still be
> neutral ink, which is the card contradicting its own graphic. When
> `restingHrAvg` is null the card spends no colour at all.

Three things about it, all binding.

**A low morning is not painted green.** This is deliberate and it is the same
argument `balance-band.tsx` makes for drawing balanced nights in `--muted`: most
mornings are ordinary, and painting every below-average day `--success` makes a
normal month the loudest card on a grid that also shows Steps and Weather.
Colour is spent only where something departs *upward*, because upward is the
direction that means something.

**`--danger` is not available here.** `dx/recovery-log-tile` established that
red is licensed for exactly one thing — a sub-33 Whoop recovery score, which is
Whoop's own red shown in Whoop's own app. A resting HR has no such published
threshold, and inventing one client-side is the forbidden classification above.
The warmest this tile ever gets is `--warning`.

**Colour marks a direction, never a verdict.** `--warning` on a `+6` means
*this is above your average*, which is a fact. It does not mean *this is bad*,
which would be a judgement the server never made.

### Lower is better — the polarity problem

Every other tile on this dashboard means **up is good**: recovery score, HRV,
steps, sleep hours, streak. Resting HR inverts it. A rising line is bad news,
and a user who has learned the dashboard's visual grammar in four other tiles
will read a climbing chart as improvement for at least a beat.

There are only three honest answers and **the variants are deliberately split
across all of them**, because this is the sharpest decision in the spread and it
should be made on sight rather than in the abstract:

1. **Keep the raw bpm axis** (up = more beats = worse) and carry the polarity in
   colour and wording alone. Honest to the numbers, at odds with the grid.
2. **Invert the axis** so better is up. Honest to the grid, and quietly a lie
   about a quantity everyone reads as a number that goes up and down.
3. **Have no axis at all** — a calendar, a table, a sorted strip — so the
   question never arises and direction lives entirely in the type.

No variant is allowed to duck it silently. Whichever it picks, the tile must
make "lower is better" legible without a legend.

### The states a variant must handle

- **default** — the ordinary morning, matching the shipped dashboard exactly so
  the two tiles cannot disagree: 30-day average **53 bpm**, Sunday **59**,
  Monday **47**, today **50**. Nothing dramatic. This is what the tile looks
  like most days and it should read calm.
- **creeping-up** — **the fixture this tile exists for.** Three weeks flat
  around 48, then 54, 56, 57, 58 over the last five days, average still reading
  49 because the climb has not yet dragged a 30-day mean. A variant that cannot
  make this obvious in one glance has failed the brief, whatever else it does
  well. It is also the fixture where the polarity decision bites hardest.
- **flat-month** — an athlete whose resting HR is 48–50 for the entire window.
  **The auto-scale trap, and the analogue of the `96.10188` wrap.** A chart
  fitted to its own data range turns 2 bpm of noise into a mountain range and
  tells the user something is happening when nothing is. Any variant that plots
  must state its axis policy and survive this fixture looking boring.
- **no-reading-yet** — today is null, the 7am state before the morning webhook
  lands. Every recovery tile handles it; this one must too, and must not promote
  yesterday into today.
- **sparse** — strap off for a stretch, only 3 readings in the last 8 days.
  Gaps must read as gaps, not as breakage — the lesson `sparse` taught on
  `dx/recovery-log-tile`, where it decided the selection.
- **calibrating** — `restingHrAvg: null`, `restingHrDays: 9`. Readings exist and
  print; there is no average, so **there is no deviation to draw**. This is the
  fixture that most endangers the deviation-based variants, the same way
  `calibrating` exposed `baseline-lanes` in the previous DX, and it is worth
  seeing before picking one. No `NaN`, no empty frame, honest n-of-14 progress.

### Representative fixture

The default state, built to agree with the shipped `recovery_log` in the
screenshot that prompted this DX. Build `days` out to all 31 entries
oldest→newest; the eight below are the recent ones.

```ts
const recovery: RecoveryView = {
  restingToday: 50,
  recoveryScore: 78,
  hrvToday: 106,
  spark: [51, 54, 57, 59, 47, 50],   // legacy, gap-omitting — DO NOT DRAW FROM THIS
  days: [
    // …extend to 31 entries, oldest→newest, ending today…
    { date: "2026-08-05", restingHr: 51, recoveryScore: 78, hrv: 112.44031, /* …HRV band fields… */ },
    { date: "2026-08-06", restingHr: null, recoveryScore: null, hrv: null,  /* strap off */ },
    { date: "2026-08-07", restingHr: 54, recoveryScore: 66, hrv: 90.5127 },
    { date: "2026-08-08", restingHr: 57, recoveryScore: 35, hrv: 83.242966 },
    { date: "2026-08-09", restingHr: 59, recoveryScore: 29, hrv: 77.39185 },   // Sunday
    { date: "2026-08-10", restingHr: 47, recoveryScore: 52, hrv: 96.10188 },   // Monday
    { date: "2026-08-11", restingHr: 49.6, recoveryScore: 61, hrv: 92.4 },     // note the FLOAT
    { date: "2026-08-12", restingHr: 50, recoveryScore: 78, hrv: 106.2 },      // today
  ],
  baseline: {
    windowDays: 30,
    restingHrAvg: 53.4, restingHrDays: 28,
    hrvAvg: 88.2, hrvStdDev: 20.1, hrvDays: 27,
    recoveryScoreAvg: 57.6, recoveryScoreDays: 28,
  },
  // …hrv, baselineTrend as the other tiles receive them…
};
```

Read as: **an ordinary week with one bump.** Sunday's 59 is six beats over a 53
average and Monday's 47 is the rebound; today's 50 is unremarkable. Nothing here
is alarming and the tile should not make it look alarming — that is what the
`creeping-up` fixture is for, and the contrast between the two is most of the
selection.

Note `49.6` on the 11th. It is there on purpose: a variant that prints `49.6 bpm`
has failed before it is compared.

These are mockups — **static fixtures that look real are preferred**. Do not
wire variants to live recovery services.

## Idioms

Five compositions of the same near-black / periwinkle / Manrope mini-card. All
five print the **30-day baseline**, show **actual bpm values** rather than only
a shape, render **integer bpm**, gate calibrating on **`restingHrAvg`**, and
follow the **colour contract** above — that is the shared brief, not a
divergence axis. They are forced apart on three things: **whether the history is
printed or plotted**, **which axis the deviation lives on** (raw bpm, deviation
from baseline, or none at all), and **how they answer the polarity problem**.

- **delta-ledger** — **Prints everything, plots nothing.** Ten dated rows, one
  per morning newest-first, each reading `Sun · 59 bpm · +6` — the actual value
  at 13px in neutral ink, the signed delta against the 30-day average beside it
  at 11px, warm when positive and muted when not, under a pinned header carrying
  the average itself (`30-DAY BASE · 53 bpm · 28 days`). No chart, no rail, no
  mark of any kind: the tile is a table and the deltas are the only colour on
  it. Its argument is that resting HR is the one metric in this family whose
  numbers are genuinely readable — always two digits, never wrapping, no unit
  drama — so plotting it throws away the precision the user asked for. Type
  scale: 13px figures over 11px deltas over 10px dates, a narrow range and no
  hero. Colour logic: **the delta's sign, and nothing else** — a column of warm
  `+n`s down the right edge *is* the creeping-up signal, with no chart needed.
  Spacing rhythm: ten rows on hairlines, tight, ~15px each. **Polarity: answer 1
  by wording** — the delta column is captioned `vs 30d` and a rise is the only
  coloured thing, so up-is-worse is learned from the colour rather than from a
  direction. Borrowing **Bloomberg Terminal**'s conviction that a dense uniform
  dated row beats a chart when every entry has the same shape, and **Apple
  Health**'s "Show More Data" tables of dated values. → *"just show me the
  numbers."* The risk: it is `recovery_log`'s detail register with more rows and
  one metric, and it may read as a spreadsheet.

- **departure-area** — **Plots the gap, not the value.** The 30-day average is
  drawn as a horizontal rule straight through the middle of the card, and the
  filled area between the rule and each day's reading is the entire chart —
  warm fill above the rule, neutral fill below, nothing plotted in absolute
  terms at all. Thirty-one days of *departure*. Above it, one 28px figure for
  today with `vs 30d +N` beneath it, and the average printed at the rule's left
  end so the zero line is labelled. Its argument: the absolute value of a
  resting heart rate is close to meaningless (a 50 is excellent for one athlete
  and elevated for another) while the departure from that athlete's own normal
  is the whole signal, so draw the signal and let the figures be a caption.
  Type scale: **the widest range in the spread** — one 28px numeral over a chart
  whose only other type is 9px. Colour logic: **fill, above the rule only**, so
  a good month is a card with almost no colour on it and `creeping-up` is a
  warm wedge on the right. Spacing rhythm: two registers, a ~54px headline over
  a ~90px chart, separated by a hairline. **Polarity: answer 1** — raw bpm axis,
  up is worse, and the warm fill above the rule is what teaches it. Borrowing
  **Garmin Connect**'s resting-HR card, which plots against a shaded personal
  range rather than an absolute scale, and **Whoop**'s recovery-history strip.
  → *"how far from normal, and for how long?"*

- **month-grid** — **Twenty-eight days as a calendar, with every number in it.**
  A 7-wide × 4-tall grid of day cells, weeks running down and weekdays running
  across, each cell printing that morning's bpm at 10px over a background tinted
  by its distance from the 30-day average — warm tint above, untinted at or
  below, `--surface-2` for a morning with no reading. Column heads are single
  weekday initials at 9px; the average sits in a one-line footer. Four weeks fit
  in ~140px because a calendar cell costs a line of type and no row furniture.
  Its argument: a month of resting HR has *weekly* structure that a linear
  series hides — weekends run high, the day after a hard session runs high — and
  a calendar is the only layout in which that pattern is visible at all. It is
  also the densest possible answer to "actual values over time": every value,
  every date, one screenful. Type scale: **dead flat** — 10px figures, 9px
  column heads, nothing else, the deliberate opposite of `departure-area`.
  Colour logic: **tint as territory**, at the ~13% macro-tint alpha, so the
  month reads as a field of warm and cool blocks before a digit is read.
  Spacing rhythm: a strict 7-column grid with `gap-[2px]` and no rules —
  alignment does all the separating. **Polarity: answer 3** — there is no axis,
  so the question never arises and warmth alone carries direction. Borrowing
  **Oura**'s calendar-style month views and **Garmin Connect**'s dense summary
  tables. → *"the whole month, every number, at once."* The risk: 28 cells of
  10px type at a one-third desktop cell is genuinely small.

- **true-scale-marks** — **Refuses to amplify.** Thirty-one marks on a raw bpm
  axis whose bounds are **fixed to a plausible human range rather than fitted to
  the data** — roughly 40–70 bpm, with the athlete's 30-day average drawn as a
  labelled rule and the axis endpoints printed at 9px — and, deliberately, the
  **axis is inverted so lower sits higher**, making better-is-up true on this
  card the way it is true on every other card in the grid. On the `flat-month`
  fixture this variant renders an almost perfectly straight line across the
  middle of the card, which is the honest picture and which every auto-scaled
  chart in this spread will render as drama. On `creeping-up` the line visibly
  descends toward the floor. Its argument is a direct rebuttal of
  `departure-area`: a chart that rescales itself to its own noise is a chart that
  cannot tell you *nothing is happening*, and for a metric this stable that is
  the most common true answer. Type scale: modest and even — a 20px figure for
  today, 9px axis labels, nothing between. Colour logic: **near-monochrome**;
  the line and marks are `--muted` throughout and `--warning` is spent only on
  marks sitting above the average rule. Spacing rhythm: a single full-bleed
  plot register with a fixed vertical scale and two hairline axis labels — one
  register, not two. **Polarity: answer 2, alone in the spread** — the inverted
  axis is the thing being tested, and whether it reads as intuitive or as a lie
  is exactly what the comparison is for. Borrowing **Oura**'s resting-heart-rate
  graph with its lowest-point marker and **Apple Health**'s fixed-range daily
  plots. → *"how much does this actually move?"*

- **sorted-strip** — **Ranks the morning instead of dating it.** The last 30
  mornings are sorted **low to high, left to right**, as a strip of thin
  vertical ticks; today's tick is filled and labelled with its value, the 30-day
  average is a second labelled tick, and a caption reads
  `4th lowest of your last 30`. Beneath the strip, the three most recent
  mornings appear in date order with their values, so chronology is not lost
  entirely. Its argument: a user does not actually know whether 50 is good for
  them, and no amount of dated history answers that as directly as showing them
  where today falls among their own month — the same reasoning behind
  `hrv_balance`'s distribution gauge, applied to the metric where an absolute
  number is least interpretable. Type scale: one 20px figure for today over a
  10px caption and 10px recent rows. Colour logic: **position is the meaning** —
  the strip is monochrome, and `--warning` appears only if today's tick falls in
  the upper third of the athlete's own month. Spacing rhythm: a ~28px strip over
  a hairline over three ~18px rows. **Polarity: answer 3, spatially** — sorted
  ascending means *left is better*, stated once in the caption and true
  everywhere on the card. Borrowing **Robinhood**'s 52-week range bar with its
  position marker, and **Whoop**'s habit of telling you where a reading sits in
  your own distribution. → *"is this a good morning, for me?"* **The deliberate
  outlier in this spread** — it is the only variant that does not primarily
  answer "over time," which is what the brief asked for, and it is here to test
  whether ranking is a better answer to the underlying question than chronology
  is. See *Selection criteria*.

## References

In-system, so what to take from each is **structural** — composition, density,
and data legibility, not their palettes or type:

- **Garmin Connect** — take its **Resting Heart Rate card**, which plots the
  recent series against a shaded band of the athlete's own normal rather than an
  absolute scale, and prints the range endpoints so the axis is never a mystery.
  Do not take its saturated red/orange. Drives `departure-area`, and its dense
  summary tables drive `month-grid`.
- **Oura** — take its **resting-heart-rate graph** (a fixed, calm vertical scale
  with the night's lowest point marked, so an ordinary month looks ordinary) and
  its **calendar-style month views**, where a metric is read as a field of days
  rather than a line. Drives `true-scale-marks` and `month-grid`.
- **Whoop** — take the **history strip** already borrowed by `recovery_log`, and
  its habit of siting a reading **within the user's own distribution** rather
  than against a population norm. Do not take its traffic light — there is no
  verdict to colour here. Drives `departure-area` and `sorted-strip`.
- **Apple Health** — take its **"Show More Data" table** of dated values, which
  is the most literal and most underrated answer to "show me the actual
  numbers," and its **fixed-range daily plots**. Drives `delta-ledger` and
  `true-scale-marks`.
- **Bloomberg Terminal** — take the conviction that a **dense, uniform dated
  row** beats a chart when every entry has the same shape. Drives
  `delta-ledger`.
- **Robinhood** — take the **52-week range bar**: a single strip with the
  current position marked against the period's extremes, which communicates
  "where does this sit" with almost no ink. Drives `sorted-strip`.

## Selection criteria

A note-to-self for the pick, not a rubric the worker optimises against. When I
compare these I am trying to decide:

- **Can I see the climb on `creeping-up` in under a second?** This is the
  fixture the tile exists for. If I have to read four numbers and subtract a
  header to notice that the last five mornings are all above my average, the
  variant has built a reference table and not an instrument.
- **Does `flat-month` look boring?** This is the one I expect to eliminate a
  variant on. An athlete whose resting HR sat between 48 and 50 all month should
  open the dashboard and feel reassured. If a variant renders that as a jagged
  line filling the card, it will cry wolf every day of a healthy month, and no
  amount of good behaviour on `creeping-up` makes up for that.
- **Do I read "lower is better" without being told?** Three variants keep the
  raw direction, one inverts the axis, one has no axis. I genuinely do not know
  which is right and I will only know on sight. Specifically: does
  `true-scale-marks`' inverted axis feel intuitive, or does it feel like the
  chart is lying to me about which way a number went?
- **Are the actual numbers actually there?** This was the explicit ask. A
  variant that renders a beautiful shape and prints two figures has not
  delivered it, however good the shape is.
- **Does it stay distinct from the four tiles already on the grid?** All five
  recovery tiles may be up at once. `morning_vitals` and `recovery` already
  print today's resting HR with a `vs 30d` chip; if the biggest element here is
  *today's number*, I own a third today-card and should not ship this at all.
  Its job is the **month**.
- **Does it stay distinct from `hrv_balance`?** This is structurally the same
  proposition — one metric, long window, its own baseline — and the honest risk
  is that I end up with two charts that look like siblings and mean different
  things. The variants that do *not* draw a line (`delta-ledger`, `month-grid`,
  `sorted-strip`) are in the spread partly to test whether avoiding that
  resemblance is worth what it costs.
- **Does a tile with no verdict still feel useful?** Every other recovery tile
  tells the user something. This one, by construction, can only show them
  something. I want to see whether that reads as restraint or as an unfinished
  card — and if it reads as unfinished, that is an argument for a follow-up API
  SOW adding an RHR band, not an argument for inventing a threshold on the
  client.
- **Does `sparse` look intentional?** Three readings in eight days is a real
  week for me. A calendar with gaps and a chart with gaps fail very differently
  and I should see both.
- **`month-grid` specifically: is 10px type in a 28-cell grid legible at a
  one-third desktop cell,** or only on the mobile full-width breakpoint? It is
  the most information-dense answer to the brief and it may simply not fit. The
  comparison must show it at true tile width, not alone.
- **`sorted-strip` specifically: is "4th lowest of your last 30" a better answer
  than a month of dates?** It is not what I asked for, which is why it is in the
  spread. If ranking turns out to be the thing I actually want to know each
  morning, I would rather find that out here than after shipping a chart.
- **Does it still read as Prog Strength v0.4** — near-black, periwinkle as
  meaning not chrome, one desaturated warm accent and no red, calm beside Steps
  and Weather — and does it preview the `/recovery` page it links into?

---

> **Lifecycle.** `status:` is editorial — the owner is the dispatch gate. It
> moves `draft` → `exploring` (worker running) → `awaiting_selection` (draft PR
> open, owner deciding) → `selected` / `abandoned`. The worker sets
> `awaiting_selection` on the `dx/resting-hr-tile` branch as it opens the PR;
> the owner sets the terminal value when they close it.
>
> **Handoff.** This DX ends at *one chosen variant*, not merged code. I open the
> draft `[DX — DO NOT MERGE]` PR, compare the variants on the preview deploy
> (`/design-explore/resting-hr-tile`, flag-gated behind
> `NEXT_PUBLIC_ENABLE_DESIGN_EXPLORE`, driven across the **default /
> creeping-up / flat-month / no-reading-yet / sparse / calibrating** fixtures),
> tick the winner's box, set `status: selected`, and **close the PR — never
> merge it.**
>
> Then one SOW against `prog-strength-web` — *"build the `resting_hr` tile per
> the `<chosen-idiom>` variant from `dx/resting-hr-tile`, production-quality,
> conforming to the design system."* Unlike the two recovery DXs before it, that
> SOW **does** touch the catalog: a new `TileId` `resting_hr`, a new
> `TILE_CATALOG` entry (title, `href: "/recovery"`, tray description), a
> `tile-renderer.tsx` case, and a default-layout decision — whether the tile
> ships in the default recovery section or is opt-in from the tray. It should
> also single-source any shared helper it needs into
> `recovery/shared.ts` rather than keeping a private copy, following the
> `recoveryBand*` precedent.
>
> Two things that SOW must carry whichever variant wins:
>
> 1. **Integer bpm.** `resting_heart_rate` is a `*float64`; `49.6` must render
>    as `50`.
> 2. **Calibrating gates on `restingHrAvg` / `restingHrDays`** — never on
>    `hrvDays`, which is a different sample with a different size.
>
> The mockup code is never promoted as-is.
