---
type: dx
status: awaiting_selection
surface: recovery-log-tile
idioms:
  - score-gutter
  - week-columns
  - band-rail-and-recent
  - blotter-lines
  - baseline-lanes
references:
  - Whoop
  - Garmin Connect
  - Oura
  - Linear
  - Bloomberg Terminal
scope: in-system
variant_count: 5
repos:
  - prog-strength-web
  - prog-strength-docs
---

# DX: Recovery Log Tile

**Status**: Awaiting selection · **Last updated**: 2026-08-11

> A DX (Design Exploration) is the platform's **divergent** work type. Unlike a
> SOW it does not converge on one correct implementation — it produces N
> differentiated visual variants of a single frontend surface, side by side on
> one comparison route, awaiting a human pick. It **never merges** and ships no
> production code; the chosen direction feeds a downstream SOW that builds it for
> real.

> **No prerequisite.** Unlike [`dx/hrv-balance-tile`](./hrv-balance-tile.md),
> every field these variants need is already on the wire. The per-day baseline
> payload shipped with
> [`sows/recovery-baseline-drift-payload.md`](../sows/recovery-baseline-drift-payload.md)
> on 2026-08-10, so `days[i]` now carries its own `baselineAvg`, `zScore`, and
> `status`. This DX is layout and legibility only — do **not** add
> `prog-strength-api` to `repos:`.

## Context

`recovery_log` ("Recovery Log") is the fourth and quietest of the recovery
tiles. Its brief is the one none of its siblings covers: **the other three all
describe this morning, and this one is the only place a *past* morning is
legible.** `recovery` gives a verdict sentence about today, `morning_vitals`
gives today's three dials, `hrv_balance` gives one metric's month-long
distribution. The log is where you go to answer *"was Sunday actually as bad as
it felt, and has this been building?"* — three metrics, dated, side by side.

That brief is right and the tile mostly fails it.

- **It shows four days, and one of them is usually empty.** `ROWS = 4` with the
  no-reading row for today means a typical morning renders **three** readable
  days. Three days is not a log — it is a recent-items list. You cannot see a
  pattern in three points, and the pattern is the only reason to put history on
  a dashboard tile at all. A week is the smallest window in which "Sunday was
  bad" becomes "the back half of last week was bad."
- **It prints `96.10188 ms`, and then wraps.** `LedgerRow` renders `${day.hrv}
  ms` — the raw Whoop RMSSD float — into a `w-16` cell, so a five-decimal
  reading breaks onto a second line and the row grows to two lines tall. Three
  wrapped rows is why the tile currently reads as unfinished; it is the same
  unrounded-millisecond bug [`dx/hrv-balance-tile`](./hrv-balance-tile.md) named
  on `balance-band.tsx`, surviving in the one tile that SOW did not touch.
  **The house convention is integer milliseconds.** This is a bug the downstream
  SOW fixes, not a taste question.
- **It leads with the wrong number.** The row reads HRV → resting HR → score,
  so the widest, noisiest, least interpretable figure sits closest to the date
  and the one number a user actually recognises — the recovery score — is a bare
  two-digit integer stranded at the right margin with no unit and no label. Whoop
  trained every one of its users to read that score first. The row should read
  **Recovery → resting HR → HRV**, in that order, hero to footnote.
- **It re-derives a server figure, and gets it subtly wrong.** The delta glyph
  computes `day.hrv − baseline.hrvAvg` against a hardcoded ±3 ms threshold —
  client arithmetic against a *server* figure, which the house rule forbids, and
  worse, it judges **every** day against **today's** baseline. Sunday is scored
  against a yardstick that did not exist on Sunday. Since 2026-08-10 each day
  carries its own `status`; the glyph should read it.
- **It spends its only colour on its least important column.** The component's
  own docstring says so: *"The ONLY color on a row is the HRV delta glyph."* So
  the tile is monochrome except for a small triangle attached to the figure that
  matters least, and the recovery score — the one field with a universally
  understood colour language — is rendered in plain ink.

A better log fits a week, reads Recovery-first, prints integers, and uses colour
where colour already means something. This exploration is about how much
structure that takes.

`scope: in-system`: the foundation is decided (see
[`../design-system.md`](../design-system.md) **v0.4**) — soft near-black ramp,
the single **periwinkle** accent (`#9aa6d6`, app-chrome and meaning only),
**Manrope** with tight numeric tracking, tabular figures, 14px hairline panels,
desaturated status colours. Variants do **not** re-litigate palette, accent, or
type. They diverge on **layout, structure, density, and composition**.

## The surface

`MorningLedgerCard` in
`prog-strength-web/app/(app)/dashboard/_components/recovery/morning-ledger.tsx`,
one cell of the `grid-cols-1 sm:grid-cols-2 lg:grid-cols-3` `TileGrid`. A
variant owns everything inside the `MiniCard` shell — the uppercase
`RECOVERY LOG` title, the `p-4` padding, and the whole-card `next/link` into
`/recovery` are chrome it keeps functional — and composes the card body.

**Space budget: ~180px of card body today, ~260px ceiling — and density is a
deliberate divergence axis.** `TileGrid` has **no span support** (uniform thirds,
`gap-3`, auto rows), so a taller Recovery Log makes its entire dashboard row
taller, including whatever unrelated tiles the user has placed beside it. That
is a real cost and it is not obviously worth paying. So the spread splits on it:
two variants spend the height on comfortable, scannable rows; three fit a week or
more inside roughly today's footprint by abandoning one-row-per-day. A variant
that needs more than 260px is not a candidate.

**Neighbours.** Four recovery tiles share `data.recovery` and may all be on the
grid at once (`recovery_trend` was folded into `hrv_balance`, which is now a
two-view tile). The log must not restate what they already say:

| Tile | Already prints | So the log must not |
| --- | --- | --- |
| `recovery` (`readiness-verdict.tsx`) | Today's verdict sentence, one coloured status word, three contributor rows with baseline-delta chips | Hero a verdict sentence, or become a second today-card |
| `morning_vitals` (`three-dial-vitals.tsx`) | Today's Score / Rest HR / HRV as three dials | Make today's row so dominant that the history is garnish |
| `hrv_balance` (`hrv-tile.tsx`, two views) | 31 nights of HRV, the drifting band, and the `driftTag` (`▲ +6 ms · 4w`) | **Print the drift tag** — it is already on the grid, once |

**The log's distinct job is the cross-metric row.** It is the only surface where
one morning's score, resting HR, and HRV sit on a single line and can be read
against each other — *the score was low and the resting HR was up* is a sentence
only this tile can make. Every variant must keep all three metrics per day. A
variant that heroes one metric across a week has built a fourth trend tile.

### The three metrics, and their reading order

**Fixed brief, not a divergence axis: Recovery → resting HR → HRV**, in that
order of prominence and in that left-to-right (or top-to-bottom) order.

1. **Recovery score** (`recoveryScore`, 0–100) — the hero. A Whoop-normalised
   composite every Whoop user reads first, and the only field on the tile with an
   established colour vocabulary.
2. **Resting HR** (`restingHr`, bpm) — the corroborating figure. Two or three
   digits, never wraps, low drama.
3. **HRV** (`hrv`, ms) — the specialist's number, and the one `hrv_balance`
   already explores in depth. Here it is context, not headline. **Integer
   milliseconds, always.**

### Colour: one system per row

Whoop's recovery bands are the colour language, re-toned to the system:

| Band | Score | Word | Token |
| --- | --- | --- | --- |
| Green | ≥ 67 | `Recovered` | `--success` `#86b39f` |
| Yellow | 34–66 | `Adequate` | `--warning` `#d6b87f` |
| Red | ≤ 33 | `Low` | `--danger` `#c79292` |

Three things about this, all of which the variants must respect.

**It is a labelling, not a recomputation.** The house rule is *never recompute a
server figure* — an average, a band bound, a z-score. Whoop's recovery score is a
fixed, published 0–100 scale with fixed, published cut points; naming which third
of that scale a number falls in is display formatting, the same class of
operation as `weekday()`. It introduces no statistics. Single-source it as
`recoveryBand()` / `recoveryBandWord()` / `recoveryBandColor()` in
`recovery/shared.ts` beside `hrvStatusColor` and `statusWord`, for exactly the
reason that file already gives: three hand-rolled copies of a threshold switch is
how `52` ends up green on one tile.

**Red is `--danger`, and this is the one place it is allowed.** `shared.ts`'s
colour contract is explicit that a suppressed HRV morning reads `--warning`,
never danger red, *"a low-HRV morning is information, not an emergency."* That
still holds — for HRV. A sub-33 recovery score is Whoop's own red, shown to the
user in Whoop's own app, and softening it to yellow means the log disagrees with
the device it came from. It gets `--danger`, at **text weight, never as a filled
block**: a red word or numeral, not a red row. Whether a week with two red
mornings still reads calm on the grid is a selection criterion, not a
foregone conclusion.

**Exactly one colour system per row.** This is the resolution to the tension
`readiness-verdict.tsx` names in its docstring — it refuses to colour the score
because it *already* colours the verdict word, and two traffic lights on one card
is confusion. The log has no verdict word, so its colour budget is unspent and
the recovery band takes it. In exchange, **HRV's `balanced`/`suppressed`/
`elevated` status must be demoted** — a glyph, an opacity, a hairline, or dropped
entirely, but never a second saturated colour beside the score. Where a variant
does paint HRV status, it uses the existing `hrvStatusColor` / `nightOpacity`
pair from `shared.ts`, never a private palette.

### The data a variant renders

`RecoveryView` in `prog-strength-web/lib/dashboard.ts`. The optionals (`days?`,
`baseline?`) still require a guard — guard once at the top and render the
calibrating state, never `!`-assert.

```ts
type RecoveryDayPoint = {
  date: string;                  // YYYY-MM-DD, local
  restingHr: number | null;      // bpm
  recoveryScore: number | null;  // 0–100
  hrv: number | null;            // ms — Whoop float, RENDER AS INTEGER
  baselineAvg: number | null;    // that day's OWN trailing baseline
  balancedLow: number | null;
  balancedHigh: number | null;
  zScore: number | null;
  status: "balanced" | "elevated" | "suppressed" | "unknown";
};
```

`days` is 31 entries, oldest→newest, **date-aligned with nulls preserved** —
every calendar date is present whether or not Whoop had a reading. The tile
slices the tail. `spark` omits missing days and destroys alignment; never draw
from it. Server figures are displayed as received.

**The 30-day baseline header stays.** `baseline.recoveryScoreAvg` /
`restingHrAvg` / `hrvAvg` is what makes a row mean anything — 52 is a bad morning
only because 58 is normal *for this person* — and it is the tile's cheapest
piece of information per pixel. Variants may re-site it (a header strip, a
pinned footer row, a column of tick marks, a "BASE" row inside the table) but
none may drop it, and none may recompute it. Baseline order follows the metric
order: **score · bpm · ms**, which is a change from today's `ms · bpm · score`.

### Representative fixture

The default state is the screenshot that prompted this DX: **a bad weekend, a
strong Monday, and no reading yet today.** Nothing dramatic, nothing broken —
this is what the tile looks like most mornings, and it is what the variants are
judged on. Build `days` out to all 31 entries oldest→newest so the wider-window
variants have real history behind them; the eight below are the ones on screen.

```ts
const recovery: RecoveryView = {
  restingToday: null,            // no reading yet today — the 7am state
  recoveryScore: null,
  hrvToday: null,
  spark: [61, 51, 54, 57, 59, 47],   // legacy, gap-omitting — do not draw from this
  days: [
    // …extend to 31 entries, oldest→newest, ending today…
    { date: "2026-08-04", restingHr: 61, recoveryScore: 41, hrv: 61.0,
      baselineAvg: 87.4, balancedLow: 67.3, balancedHigh: 107.5, zScore: -1.31, status: "suppressed" },
    { date: "2026-08-05", restingHr: 51, recoveryScore: 78, hrv: 112.44031,
      baselineAvg: 87.6, balancedLow: 67.5, balancedHigh: 107.7, zScore: 1.24, status: "elevated" },
    { date: "2026-08-06", restingHr: null, recoveryScore: null, hrv: null,
      baselineAvg: 87.8, balancedLow: 67.7, balancedHigh: 107.9, zScore: null, status: "unknown" }, // strap off
    { date: "2026-08-07", restingHr: 54, recoveryScore: 66, hrv: 90.5127,
      baselineAvg: 87.9, balancedLow: 67.8, balancedHigh: 108.0, zScore: 0.13, status: "balanced" },
    { date: "2026-08-08", restingHr: 57, recoveryScore: 35, hrv: 83.242966,
      baselineAvg: 87.9, balancedLow: 67.8, balancedHigh: 108.0, zScore: -0.23, status: "balanced" },
    { date: "2026-08-09", restingHr: 59, recoveryScore: 29, hrv: 77.39185,
      baselineAvg: 88.0, balancedLow: 67.9, balancedHigh: 108.1, zScore: -0.53, status: "balanced" },
    { date: "2026-08-10", restingHr: 47, recoveryScore: 52, hrv: 96.10188,
      baselineAvg: 88.2, balancedLow: 68.1, balancedHigh: 108.3, zScore: 0.39, status: "balanced" },
    { date: "2026-08-11", restingHr: null, recoveryScore: null, hrv: null,
      baselineAvg: 88.2, balancedLow: 68.1, balancedHigh: 108.3, zScore: null, status: "unknown" }, // today
  ],
  baseline: {
    windowDays: 30,
    restingHrAvg: 53.4, restingHrDays: 28,
    hrvAvg: 88.2, hrvStdDev: 20.1, hrvDays: 27,
    recoveryScoreAvg: 57.6, recoveryScoreDays: 28,
  },
  hrv: { status: "unknown", balancedLow: 68.1, balancedHigh: 108.3,
         zScore: null, trend: "steady", shortAvg: 86.7 },
  baselineTrend: { direction: "steady", deltaMs: 1.2, fromAvg: 87.0, overDays: 28 },
};
```

Read as: **Sunday's 29 is the story.** A red morning, resting HR six beats above
a 53 baseline, HRV ordinary — the classic under-recovered-but-not-sick signature,
and Saturday's 35 says it was two days, not one. Monday's 52 with a 47 bpm
resting HR is the rebound. The shipped tile renders exactly this as three
double-height rows of `77.39185 ms` with a small yellow triangle and the 29
whispered at the right edge, and a reader gets none of it. That gap is the whole
exploration.

Note the deliberately ugly floats (`112.44031`, `83.242966`, `96.10188`) — they
are real Whoop values, they are in the fixture on purpose, and a variant that
wraps or overflows on them has failed before it is compared.

**Also build these fixtures** — the comparison route must be drivable across all
of them:

- **`full-week`** — today has landed (score 71, 49 bpm, 94 ms, `balanced`) and
  no day in the window is missing. The clean case, and the one where a
  seven-row variant looks its best. Every variant should be *good* here; the
  others are where they separate.
- **`red-morning`** — today at score 22, 63 bpm, 58 ms `suppressed`, following
  two yellows. The dramatic state and the `--danger` stress test. Must read as
  *true and worth noticing*, never as an alarm on a dashboard that also shows
  Steps and Weather.
- **`sparse`** — only 3 of the last 8 days have readings (travel, strap on the
  charger). **The most important new fixture in this DX**: going from 4 rows to 7+
  quadruples the chance of a mostly-empty tile, and a week of *no reading* rows
  is a much worse failure than three tight ones. A variant must make absence look
  like absence, not like breakage.
- **`calibrating`** — `hrvDays: 9`, every baseline average null, every day
  `status: "unknown"`. Readings exist and print; the baseline header reads
  `9 of 14 nights` and no comparison is drawn anywhere. No `NaN`, no empty frame,
  no colour on the HRV column. Every new Whoop user lives here for two weeks.
- **`partial-band`** — 40 nights of history, so the oldest days in a 14-day
  window carry `baselineAvg: null` / `status: "unknown"` while the recent ones do
  not. Only bites the wider-window variants; they must not render the
  uncalibrated days as a different *kind* of row.

These are mockups — **static fixtures that look real are preferred**. Do not wire
variants to live recovery services.

## Idioms

Five compositions of the same near-black / periwinkle / Manrope mini-card. All
five read **Recovery → resting HR → HRV**, print **integer milliseconds**, keep
the **30-day baseline**, show **at least seven days**, and paint the **recovery
band** as the row's single colour system — that is the shared brief, not a
divergence axis. They are forced apart on **type scale** (one 20px numeral per
row vs a 7-across grid of 15px figures vs nothing above 11px), **colour logic**
(band as territory vs band as fill vs band as a single word vs band as mark
height), and **spacing rhythm** (tall hairline-separated rows vs a dense column
grid vs two stacked registers vs unruled text lines vs three lanes). Two spend
height on ~7 comfortable rows; three fit 7–14 days inside roughly today's ~180px.

- **score-gutter** — **Heroes the score, one row per day, and lets colour be
  territory.** The most direct evolution of the shipped tile: seven rows, but
  each row leads with a **3px full-height rule in the day's band colour** flush
  to the card's left padding edge, then the weekday, then the score as the row's
  largest figure (~20px, tight tracking) with its band word beneath it at 10px,
  then `bpm` and `ms` as small mono at the right. The gutter rules stack into a
  readable vertical stripe of the week — you see the bad weekend as a block of
  warm colour before you read a single number. Type scale: one ~20px numeral per
  row over 10px furniture, the widest intra-row range in the spread. Colour
  logic: the **gutter carries the band**, saturated and continuous; the score
  numeral stays neutral ink so the tile does not become seven coloured numbers;
  HRV status is a 1px underline on the ms figure at `nightOpacity`, nothing more.
  Spacing: seven ~30px rows on hairlines, generous and even — **this is the
  ~260px variant** and the one that most obviously costs the grid row height.
  Borrowing **Linear**'s coloured status rails, which make a list scannable
  before it is read, and **Whoop**'s band vocabulary. → *"the shipped tile, done
  properly."*

- **week-columns** — **Transposes the table: days across, metrics down.** Seven
  day-columns (`W T F S S M T` on the default fixture, oldest→newest left to
  right, today rightmost), three metric rows down — score, bpm, ms — with the
  metric names as a 10px left gutter and the 30-day baseline pinned as a **`BASE`
  column** at the far right, hairline-separated from the week so it reads as the
  yardstick rather than an eighth day. Fits a full week in **today's ~180px** because there is no
  per-row weekday label, no per-row padding, and each figure is printed once.
  The score cell carries a **soft band-tinted fill** (~13% alpha, the macro-tint
  convention); the bpm and ms cells are unfilled. Type scale: flat and dense —
  15px tabular score figures, 12px bpm/ms, 10px uppercase column and row labels,
  no hero. Colour logic: **band as a filled cell**, so the week's shape is a row
  of seven tinted blocks; a missing day is an empty cell with a faint dash, not a
  grey block. Spacing: a strict 9-column `grid` — label gutter, seven days,
  `BASE` — with `gap-1` and no row rules;
  alignment does the separating. Borrowing **Oura**'s weekly readiness strip and
  **Garmin Connect**'s dense weekly summary tables. → *"show me the week as a
  grid."* The tightest fit and the one most at risk of reading as a spreadsheet.

- **band-rail-and-recent** — **Two registers: fourteen days at a glance, three in
  detail.** The top ~40px is a **14-day rail** of vertical bars, one per day,
  height proportional to the recovery score and fill in the band colour, with a
  faint horizontal tick across it at `baseline.recoveryScoreAvg` — so a fortnight
  of recovery reads as a shape against its own normal, and the bad weekend is a
  visible notch. Beneath it, on hairlines, the **three most recent mornings** as
  full detail rows (weekday · score + word · bpm · ms), which is where the
  cross-metric reading actually happens. The baseline's bpm and ms figures live in
  a single 10px line between the registers. Its argument: seven identical rows
  give a week you must read row by row, while a rail gives a *fortnight* you
  read in one glance and detail only where detail is used. Type scale: no
  numeral above 13px anywhere — the rail is the emphasis, not type. Colour logic:
  **the rail carries the band** across fourteen bars, the detail rows carry only
  a small band-coloured word beside each score, and the baseline tick is neutral.
  Spacing: two unequal stacked registers separated by a hairline, ~40px over
  ~110px. Borrowing **Whoop**'s recovery-history bar strip and **Garmin
  Connect**'s summary-over-detail card structure. → *"the shape of the fortnight,
  then this morning."* The only variant that shows more than a week.

- **blotter-lines** — **Abandons columns entirely: one sentence per morning.**
  Each day is a single unruled line of text in one size —
  `Mon · Recovered 72 · 47 bpm · 91 ms ▲` — set in 12px with the weekday in
  `--faint`, the band word in the band colour, and everything else in muted ink,
  packed at tight leading with no row padding and no dividers. Nine or ten days
  fit in ~180px because a text line costs a line, not a row. The band word does
  double duty as the label the shipped tile lacks and as the only colour, and the
  lines left-align into a ragged column that reads top to bottom like a log file —
  which is what the tile is called. A missing morning is `Thu · no reading` in
  `--faint`, one line, occupying exactly the same space as a real one, which is
  why this idiom survives the `sparse` fixture better than any row-based variant.
  Type scale: **one size for everything** — the flattest in the spread, and the
  deliberate opposite of `score-gutter`. Colour logic: **exactly one coloured word
  per line**, HRV status reduced to a `nightOpacity`-weighted glyph at the end of
  the line. Spacing: no grid, no rules, no columns — text leading is the entire
  rhythm. Borrowing **Bloomberg Terminal**'s conviction that a dense monospaced
  log is *more* readable than a table when every line has the same shape, and
  **Linear**'s activity-feed line density. → *"read it like a log."* The risk is
  that unaligned figures are harder to compare down the column than across the
  line — see *Selection criteria*.

- **baseline-lanes** — **Transposes against the baseline instead of against the
  date.** Three stacked ~26px lanes, one per metric in brief order, each with the
  30-day baseline drawn as a **fixed centre tick** and fourteen day-marks placed
  by their deviation from it — score above/below its average, resting HR
  above/below (inverted, so "better" is always the same direction), HRV
  above/below. The lanes are labelled at 10px with the baseline's own value
  (`RECOVERY · base 58`), so the yardstick is structural rather than a header
  line, and today's mark is filled while the rest are hollow. Its argument: every
  other variant prints numbers you must mentally subtract from a baseline printed
  somewhere else; this one does the subtraction in space, and a fortnight where
  every metric sits below its tick is instantly visible in a way that a column of
  integers never is. Cost: **the exact values are gone** — you get position, not
  figures, on a tile whose current job is to print figures. Type scale: 10px
  labels only, no numerals in the plot at all. Colour logic: **the band colours
  the mark**, so the recovery lane is a row of coloured dots and the bpm/ms lanes
  are near-monochrome — colour appears in one lane out of three. Spacing: three
  equal horizontal lanes with hairlines between, no columns, no rows. Borrowing
  **Garmin Connect**'s deviation-from-baseline plotting and **Oura**'s
  contributor lanes. → *"where did each metric sit, relative to normal?"*
  **The deliberate outlier in this spread** — see *Selection criteria* for the
  risk it is being tested against.

## References

In-system, so what to take from each is **structural** — composition, density,
and data legibility, not their palettes or type:

- **Whoop** — take two things and only two: the **recovery band vocabulary**
  (green/yellow/red at 67 and 33, and the fact that users already know it) and
  the **recovery-history bar strip** that turns a fortnight of scores into a
  shape. Do **not** take its saturated traffic light — re-tone to `--success` /
  `--warning` / `--danger` — and do not take its ring. Drives the colour system
  throughout and `band-rail-and-recent`.
- **Garmin Connect** — take the **dense weekly summary table** (days across,
  metrics down, a baseline column at the edge) and its **deviation-from-baseline
  plotting**, where a metric's position against a fixed tick is the datum. Drives
  `week-columns` and `baseline-lanes`.
- **Oura** — take the **weekly readiness strip**, seven tinted cells that read as
  a week before they read as numbers, and its **contributor lanes** stacked one
  per metric. Drives `week-columns` and `baseline-lanes`.
- **Linear** — take the **coloured status rail** on list rows, which makes a list
  scannable at a glance without colouring the content, and its **activity-feed
  line density**. Drives `score-gutter` and `blotter-lines`.
- **Bloomberg Terminal** — take the conviction that a **dense, uniform log line**
  beats a table when every entry has the same shape. Drives `blotter-lines`.

## Selection criteria

A note-to-self for the pick, not a rubric the worker optimises against. When I
compare these I am trying to decide:

- **Can I read Sunday off it in under a second?** The default fixture has one
  story in it — a red weekend with an elevated resting HR, then a rebound. If I
  have to scan three columns and subtract a header to find it, the variant has
  built a table, not a log.
- **Does Recovery actually lead?** Not "is it leftmost" — is it the thing my eye
  lands on first? A variant can put the score first and still let a wide `96 ms`
  pull the eye, which is exactly the failure I am fixing.
- **Does a week of colour stay calm?** It sits next to Steps, Weather, and Blood
  Pressure. Seven coloured scores is very easily the loudest thing on the
  dashboard. The `full-week` fixture — mostly greens and yellows, nothing wrong —
  should look almost quiet, and `red-morning` should be the only one that pulls.
- **Is the red honest?** A 29 is a real red in Whoop and I want to see it as one.
  But if a `--danger` word makes me feel like the app is telling me I am ill, the
  colour is too strong and it belongs at yellow after all. I will know on sight,
  and only on the `red-morning` fixture.
- **Does `sparse` look intentional?** Three readings in eight days is my actual
  life some weeks. If a variant renders that as five *no reading* rows and reads
  as a broken tile, going wider has cost me more than it bought. This is the
  fixture I expect to eliminate a variant on.
- **Is the 30-day baseline still doing work?** It is the reason a 52 means
  anything. If a variant relegates it to a header I stop reading, the comparison
  is happening in my head instead of on the card — and if a variant makes it
  structural (a `BASE` column, a centre tick), does that actually help or does it
  just make the card harder to parse?
- **Is the height worth it?** `score-gutter` costs the whole grid row ~80px, and
  the three dense variants cost nothing. That is a big price for legibility and I
  should make myself feel it — with the tile placed beside two short tiles in the
  comparison, not alone.
- **Does it stay distinct from `recovery`, `morning_vitals`, and `hrv_balance`?**
  All four may be on one grid. If the log's biggest element is today, I own two
  today-cards. If its biggest element is one metric over time, I own two
  `hrv_balance`es. Its job is the *dated cross-metric row*.
- **`blotter-lines` specifically: does losing column alignment cost more than the
  density buys?** Ten days is more than any other variant fits at this height,
  and unaligned figures are genuinely harder to compare down a column. It is in
  the spread because a log file is a real and good idiom for a thing literally
  named a log. If it fails, it should fail here rather than in my head.
- **`baseline-lanes` specifically: can I live without the numbers?** It is the
  most analytically honest view — everything measured against its own normal —
  and it is the most at risk against the tile's own brief, which is to *print
  three readings per day*. I want to see whether the lane labels keep it concrete
  or whether a log with no figures on it stops being a log.
- **Does it still read as Prog Strength v0.4** — near-black, periwinkle as
  meaning not chrome, desaturated status, no imported Whoop traffic light — and
  does it preview the `/recovery` page it links into?

---

> **Lifecycle.** `status:` is editorial — the owner is the dispatch gate. It moves
> `draft` → `exploring` (worker running) → `awaiting_selection` (draft PR open,
> owner deciding) → `selected` / `abandoned`. The worker sets `awaiting_selection`
> on the `dx/recovery-log-tile` branch as it opens the PR; the owner sets the
> terminal value when they close it.
>
> **Handoff.** This DX ends at *one chosen variant*, not merged code. I open the
> draft `[DX — DO NOT MERGE]` PR, compare the variants on the preview deploy
> (`/design-explore/recovery-log-tile`, flag-gated behind
> `NEXT_PUBLIC_ENABLE_DESIGN_EXPLORE`, driven across the **default / full-week /
> red-morning / sparse / calibrating / partial-band** fixtures), tick the
> winner's box, set `status: selected`, and **close the PR — never merge it.**
>
> Then one SOW against `prog-strength-web` — *"rebuild the `recovery_log` tile
> per the `<chosen-idiom>` variant from `dx/recovery-log-tile`,
> production-quality, conforming to the design system"* — replacing
> `MorningLedgerCard`'s body. No catalog change: same `TileId`, same title, same
> tray entry, same `href` (the tile's description in `lib/dashboard-tiles.ts`,
> *"Your last few mornings as dated readings,"* wants updating to say a week).
> That SOW also carries the four defects named in *Context*, which are fixes
> rather than taste and are binding whichever variant wins:
>
> 1. **Integer milliseconds.** `${day.hrv} ms` must round. This is the wrap.
> 2. **Per-day status from the payload.** Delete the `day.hrv − baseline.hrvAvg`
>    ±3 ms client arithmetic and read `day.status`, which is that day's own
>    verdict against that day's own band.
> 3. **`ROWS = 4` → at least seven days**, per the chosen variant.
> 4. **No fixed-width value cells** (`w-16` / `w-14` / `w-6`) that cannot hold a
>    three-digit figure plus its unit — the proximate cause of the wrap.
>
> The mockup code is never promoted as-is.
