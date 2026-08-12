---
type: sow
status: ready_for_implementation
repos:
  - prog-strength-web
  - prog-strength-docs
---

# Recovery Log Tile Redesign

**Status**: Ready for implementation · **Last updated**: 2026-08-11

## Introduction

[`dx/recovery-log-tile`](../dx/recovery-log-tile.md) explored five redesigns of
the `recovery_log` tile and **`band-rail-and-recent` was selected** (DX
comparison PR Prog-Strength/prog-strength-web#161, closed un-merged). This SOW
builds it for real.

The shipped tile fails the one brief no sibling covers — making a *past* morning
legible as a dated, cross-metric row. It renders four rows, one of which is
usually today's empty one, so a typical morning shows **three** readable days;
three points is a recent-items list, not a log, and a pattern is the only reason
history earns space on a dashboard tile. It prints `96.10188 ms` into a `w-16`
cell, so every HRV reading wraps and the rows grow to double height. It reads
HRV → resting HR → score, putting the widest and least interpretable figure
closest to the date and stranding the recovery score — the one number a Whoop
user recognises — unlabelled at the right margin. It re-derives a server figure
with client arithmetic (`day.hrv − baseline.hrvAvg`, ±3 ms) and judges *every*
day against *today's* baseline, so Sunday is scored with a yardstick that did
not exist on Sunday. And it spends its only colour on the least important
column, by its own docstring's admission.

What changes for the user: the tile answers *"has this been building?"* — the
question it is named for and currently cannot answer. A fortnight of recovery
becomes a shape read in one glance, with the bad weekend visible as a notch
before a single figure is read, and the three most recent mornings stay
available in full so the cross-metric reading (*the score was low and the
resting HR was up*) still happens on the card.

The chosen idiom takes **Whoop**'s recovery-history bar strip for the overview
register and **Garmin Connect**'s summary-over-detail card structure for the
composition. Its deliberate disagreement with the other four variants is worth
restating because it is the decision everything else follows from: **it does not
give every day a row.** Fourteen days appear as bars; three appear as rows. The
argument is that seven identical rows give you a week you must read row by row,
while a rail gives you a fortnight you read at a glance and detail only where
detail is used — and that this is what makes the tile survive the `sparse`
fixture, where a row-per-day variant degrades into a column of *no reading*.

## Proposed Solution

`MorningLedgerCard` keeps its `MiniCard` shell, its `RECOVERY LOG` title, its
`href`, and its catalog entry, and its body is replaced with two unequal
registers separated by a seam:

1. **The rail** — fourteen days as vertical bars, height proportional to the
   recovery score, filled in that day's band colour, with a dashed neutral tick
   across the rail at `baseline.recoveryScoreAvg`. ~40px.
2. **The seam** — a single 10px line carrying the 30-day baseline as
   `score · bpm · ms`, on a hairline, doubling as the register divider.
3. **The detail rows** — the three most recent mornings, hairline-separated:
   weekday · score · band word · `bpm · ms`. Nothing on the card is larger than
   13px; the rail is the emphasis, not type. ~77px.

The mockup is not promoted. Three helpers graduate into
`recovery/shared.ts` — `recoveryBand`, `recoveryBandWord`, `recoveryBandColor`
— which is where the DX ticket says they belong, beside `hrvStatusColor` and
`statusWord`, for the reason that file already gives: three hand-rolled copies
of a threshold switch is how `52` ends up green on one tile. Everything else in
the mockup's `_shared.ts` either duplicates a production helper (`shortDay` is
`weekday`; `int` and `ms` are the same function twice) or belongs to a variant
that was not selected (`recoveryBandTint`, `nightMark`), and is not carried
over.

**Five corrections to the mockup are specified below and are the substance of
this work beyond transcription.** In order of consequence: the calibrating gate
reads one metric's average and prints a different metric's day count; a partial
morning renders a blank score beside the words *No reading*; an absent day on
the rail is a 2px stub that is a pixel away from the shortest real bar, which is
exactly the "a gap must not read as a catastrophic score" failure the mockup's
own comment says it is avoiding; the entire rail register is `aria-hidden` with
no text alternative; and the bar-height and tick maps must be pinned to each
other or the tick silently desynchronises the first time someone rescales the
rail to the data range.

Nothing recomputes a server figure. Naming which third of Whoop's fixed,
published 0–100 scale a score falls in is display formatting — the same class of
operation as `weekday()` — and the only other arithmetic is mapping an
already-computed score onto a pixel height, which is what a bar chart is.

## Goals and Non-Goals

### Goals

- `MorningLedgerCard` renders the `band-rail-and-recent` two-register
  composition, production-quality, conforming to
  [`design-system.md`](../design-system.md) v0.4.
- **Integer milliseconds everywhere.** `${day.hrv} ms` rounds. This is the wrap,
  and it is a bug, not a taste question. (DX binding defect 1.) **The recovery
  score rounds too** — `RecoveryScore` is a `*float64` on the wire, so nothing
  guarantees it arrives integral, and the mockup prints it raw. Every figure on
  this card goes through one formatter.
- **No client arithmetic against a server figure.** The `day.hrv −
  baseline.hrvAvg` ±3 ms delta glyph is deleted outright — this idiom spends its
  colour budget on the recovery band and shows no HRV status at all, which the
  ticket's colour contract permits explicitly ("a glyph, an opacity, a hairline,
  or dropped entirely"). (DX binding defect 2.)
- **`ROWS = 4` → fourteen days on the rail, three in detail.** (DX binding
  defect 3.)
- **No fixed-width value cells.** `w-16` / `w-14` / `w-6` are gone; the figure
  group is `ml-auto` and sizes to its content, so a three-digit figure plus its
  unit cannot wrap. (DX binding defect 4.)
- The row reads **Recovery → resting HR → HRV**, hero to footnote, and the
  baseline follows the same order: `score · bpm · ms` (a change from today's
  `ms · bpm · score`).
- The recovery band is the card's **single colour system**: the rail's fill and
  one small word per detail row. `--danger` appears at text and mark weight
  only, never as a filled block.
- The 30-day baseline survives as both a **figure** (the seam) and a
  **structure** (the rail's tick).
- An **absent morning reads as an absence** — a ghost slot on the rail and a
  *no reading* row in the detail register — and cannot be confused with a low
  score.
- The rail carries a **text alternative**; it is not silently dropped for
  assistive tech.
- All six DX fixtures render correctly: **default · full-week · red-morning ·
  sparse · calibrating · partial-band** — plus a partial morning and both
  breakpoints.
- The catalog is unchanged but for the tray copy, which currently claims "your
  last few mornings" and is now wrong.

### Non-Goals

- **No API change.** Every figure this tile draws already ships;
  [`sows/recovery-baseline-drift-payload.md`](./recovery-baseline-drift-payload.md)
  landed 2026-08-10 and `days[i]` carries its own `baselineAvg`, `zScore`, and
  `status`. Do not add `prog-strength-api` to `repos:`.
- **No catalog or layout change beyond the description.** Same `TileId`, same
  title, same `href`, no `tile-renderer.tsx` change, no grid-span work.
- **No change to the other four recovery tiles.** `shared.ts` changes are
  strictly additive — three new exports, no existing signature touched.
- **No `/recovery` deep-page change.** Promoting the rail to the page is a
  reasonable follow-up and is explicitly not in scope.
- **Do not add resting HR or HRV to the rail.** The ticket's "every variant must
  keep all three metrics per day" is satisfied by the detail register; this
  idiom's whole argument is that fourteen days of one metric plus three days of
  three metrics beats seven days of three. Adding lanes rebuilds `baseline-lanes`,
  which was not selected.
- **No interactive controls.** No hover tooltip on a bar, no range switcher. The
  whole card is a `next/link`; interaction belongs on the deep page. Restated
  because a bar strip invites a tooltip more than any tile so far.
- **No promotion of the DX branch code.** `app/design-explore/**` is throwaway
  and stays on its closed PR.

## Implementation Details

### File layout

| File | Responsibility |
| --- | --- |
| `app/(app)/dashboard/_components/recovery/morning-ledger.tsx` | **Rewritten.** The two registers, the seam, the guard, the calibrating state. |
| `app/(app)/dashboard/_components/recovery/shared.ts` | **Extended.** The three `recoveryBand*` helpers and one rounding formatter. Additive only. |
| `app/(app)/dashboard/_components/recovery/fixtures.ts` | **Extended.** Score-bearing fixtures; see *Testing*. Existing exports unchanged. |
| `app/(app)/dashboard/_components/recovery/morning-ledger.test.tsx` | **Rewritten.** Every existing assertion describes behaviour this SOW deletes. |
| `lib/dashboard-tiles.ts` | One-line tray description change. |

The tile stays a single component file. The rail is ~30 lines and is not
reusable until `/recovery` wants it, at which point extracting it is a
three-minute change against a tested component — extracting it now would be
speculative.

### Register 1 — the rail

Fourteen days, `items-end`, `flex-1` bars with a 2px gutter, over a dashed tick
at the 30-day recovery-score average.

```tsx
const RAIL_DAYS = 14;
const RAIL_H = 40;

/** Score (0–100) → bar height in px. The tick uses this same map — see below. */
const railY = (score: number) => (score / 100) * RAIL_H;
```

**The tick and the bars must share one map.** The mockup already does this
correctly (`(score / 100) * RAIL_H` for the bar, `(recoveryScoreAvg / 100) *
RAIL_H` for the tick) and it is load-bearing: the rail's whole claim is that a
fortnight reads *against its own normal*, which is true only while a bar at the
baseline's value terminates exactly at the tick. The obvious future
"improvement" — rescaling the rail to the observed data range so a flat week
uses the full height — breaks that silently, because the tick would still be
drawn on the 0–100 scale. Extracting `railY` and pinning it under test is the
guard against that, and the reason it is a named function rather than an inline
expression.

The 0–100 scale is also the right one on its own terms: the recovery score is a
percentage with meaningful absolute values, so a 29 should look short in
absolute terms and not merely short relative to a bad fortnight.

```tsx
<div
  role="img"
  aria-label={railLabel(rail, baseline.recoveryScoreAvg)}
  className="relative flex items-end gap-[2px]"
  style={{ height: RAIL_H }}
>
  {baseline.recoveryScoreAvg !== null && (
    <div
      aria-hidden="true"
      className="pointer-events-none absolute inset-x-0 border-t border-dashed border-[var(--border-strong)]"
      style={{ bottom: railY(baseline.recoveryScoreAvg) }}
    />
  )}
  {rail.map((d) => (
    <RailBar key={d.date} day={d} />
  ))}
</div>
```

**Correction — the tick's contrast.** `--border-strong` is
`rgba(255,255,255,0.1)`, which is a hairline chosen to read against `--surface`.
It crosses fourteen band-coloured bars, and over `--success` at full strength it
will be close to invisible — which loses the register's entire "against its own
normal" argument at exactly the moment the fortnight is good. Draw the tick
**above** the bars (it already is, by DOM order) and verify it on the
`full-week` fixture specifically, where every bar is coloured and tall. If it
does not read there, step it up to a `--muted` dashed line at ~0.5 opacity
rather than leaving it at the hairline token; a structural datum is not chrome.

**Correction — an absent morning must differ in kind, not in height.** The
mockup draws a missing day as a 2px `--surface-2` stub at the bottom of the
rail. `--surface-2` (`#191c21`) against the card's `--surface` (`#15171b`) is a
four-value difference, so at 2px it is very nearly invisible; and where it *is*
visible it sits one pixel below the shortest real bar the mockup permits
(`Math.max(3, …)`), so a gap and a catastrophic score render as the same mark.
That is precisely the confusion the mockup's own comment says it is avoiding,
and it is the failure mode the `sparse` fixture exists to catch.

Draw the absent slot as a **full-rail-height ghost column** instead:

```tsx
function RailBar({ day }: { day: RecoveryDayPoint }) {
  const score = day.recoveryScore;

  // An absent morning is a ghost slot running the full height of the rail — an
  // empty position in a rack, not a very short bar. Differing in KIND rather
  // than in height is what keeps a strap-off day from reading as a score of 2,
  // and it is what makes the `sparse` fixture look intentional rather than
  // broken.
  if (score === null) {
    return (
      <div
        aria-hidden="true"
        className="h-full flex-1 rounded-[1px] bg-[var(--surface-2)] opacity-60"
      />
    );
  }

  return (
    <div
      aria-hidden="true"
      className="flex-1 rounded-[1px]"
      style={{
        height: Math.max(2, railY(score)),
        background: recoveryBandColor(recoveryBand(score)),
      }}
    />
  );
}
```

The 2px floor now only bites below a score of 5, where it can never approach a
plausible baseline tick, and it exists so a genuine near-zero morning still
renders a mark.

**Correction — the rail needs a text alternative.** Every bar and the tick are
`aria-hidden`, so as written the first register is entirely absent from the
accessibility tree with nothing standing in for it. Give the container
`role="img"` and a label, the same treatment `balance-band.tsx` gives its SVG:

```ts
/** e.g. "Fourteen days of recovery score, 11 with readings, against a 30-day average of 58." */
function railLabel(rail: RecoveryDayPoint[], avg: number | null): string { … }
```

Per-bar `aria-hidden` stays — fourteen unlabelled divs in the tree is worse than
one labelled group.

**Windowing.** `rail = days.slice(-RAIL_DAYS)`. `days` is 31 entries,
date-aligned oldest→newest with nulls preserved, so the slice is a true calendar
fortnight and the bar pitch is a day. **Never draw from `spark`** — it omits
missing days and destroys the alignment the rail's shape depends on.

At a one-third desktop cell the card's inner width is ~228px; fourteen bars with
thirteen 2px gutters leaves ~14.4px per bar. At the mobile full-width
breakpoint it is ~19px. Both are comfortable.

### The seam — the baseline, and a calibrating gate that is currently wrong

```tsx
<div className="flex items-baseline justify-between border-t border-[var(--border)] pt-1.5 text-[10px] uppercase tracking-[0.08em] text-[var(--faint)]">
  <span>{calibrating ? "calibrating" : "30-day base"}</span>
  <span className="tabular-nums tracking-[-0.01em] normal-case">
    {calibrating
      ? `${baseline.recoveryScoreDays} of ${MIN_BASELINE_DAYS} mornings`
      : `${round(baseline.recoveryScoreAvg)} · ${round(baseline.restingHrAvg)} bpm · ${round(baseline.hrvAvg)} ms`}
  </span>
</div>
```

**Correction — the gate and its progress line are about two different metrics.**
The mockup computes `calibrating = baseline.recoveryScoreAvg === null` and then,
inside that branch, prints `${baseline.hrvDays} of 14 nights`. Those are
independent quantities. `internal/recoverytrend` collects a separate sample per
metric and emits each average only once that metric has `MinBaselineDays` (14)
non-null readings behind it, reporting `restingHrDays`, `hrvDays`, and
`recoveryScoreDays` separately — a Whoop user who wore the strap without
sleeping in it, or who had a run of nights where the recovery algorithm
declined to score, can easily sit at 20 score-days and 10 HRV-nights, or the
reverse. As written the tile can say *"10 of 14 nights"* while the rail it is
labelling is fully calibrated, or claim to be calibrated while printing a
progress figure that has nothing to do with anything on the card.

Gate on the metric the register actually draws:

- `calibrating` is `baseline.recoveryScoreAvg === null`. This is the correct
  gate — it is the tick's own input, and without it the rail has no normal to be
  read against. (Note this differs from the shipped tile, which gates on
  `hrvAvg`; that was right for a tile whose only coloured element was an HRV
  delta and is wrong for this one.)
- The progress line counts `recoveryScoreDays`, and says **mornings**, not
  nights — a recovery score is a morning's verdict, and calling it a night
  invites confusion with `hrv_balance`'s n-of-14, which genuinely is nights.
- The three baseline figures each guard independently. `restingHrAvg` or
  `hrvAvg` may be null while `recoveryScoreAvg` is not; each renders `—` on its
  own and the seam keeps its shape. Do **not** collapse the whole line when one
  average is missing.

`MIN_BASELINE_DAYS = 14` is a client-side copy of a server config value, which
the shipped tile also hardcodes. It stays hardcoded here — see Open Question 2 —
but it becomes a named constant with a comment saying where the real value
lives, rather than a literal in a template string.

### Register 2 — the detail rows

The three most recent calendar days, newest first, on hairlines. Nothing above
13px.

```tsx
<div className="-mt-0.5 flex flex-col divide-y divide-[var(--border)]">
  {recent.map((d, i) => (
    <DetailRow key={d.date} day={d} isToday={i === 0} />
  ))}
</div>
```

The row is `weekday` (or `Today`) at 10px in `--faint`, the score at 13px in
`--foreground` with tight tracking and tabular figures — **through `round()`,
not raw**, since `recovery_score` is a nullable float on the wire and a
`52.4` would wrap this row exactly the way `96.10188` wraps today's — the band word at 10px
uppercase in the band colour, and then `ml-auto` a mono 11px group of
`{bpm} bpm · {ms} ms` with the units themselves in `--faint`. The figure group
has no fixed width and cannot wrap — this is DX binding defect 4, and it is why
`96.10188` becoming `96` is necessary but not sufficient.

**Correction — a partial morning renders a blank score.** The mockup's
`isMissing` requires all three readings to be null, and then the row body prints
`{day.recoveryScore}` unguarded. A morning with a resting HR and an HRV but no
recovery score — independently nullable in `RecoveryDayPoint`, and a state Whoop
does produce — therefore renders an empty gap where the score should be,
followed by the band word for the `"none"` band, which is the string **"No
reading"**, followed by two perfectly good figures. The row reads
`Sun  No reading  54 bpm · 90 ms`, which is false.

Handle the score's absence at the field level rather than the row level:

- The row renders as *no reading* only when **all three** are null — unchanged,
  and correct: that is a morning the webhook never delivered.
- Otherwise the score prints `—` when it is null, and the **band word is omitted
  entirely** rather than printing "No reading" beside two live figures. A missing
  score has no band; the honest rendering is silence, not a label.
- `restingHr` and `hrv` already round-or-dash through the shared formatter.

**Today's row.** `isToday` is positional (`i === 0`), which is correct by the
payload's contract that `days` ends on the local today. The 7am state — today
present in `days` with all three fields null — therefore renders `Today · no
reading`, which is the true and useful answer to *"has my morning landed?"* and
must not be suppressed.

### Shared helpers

**`recovery/shared.ts`** — additive only. No existing export changes signature,
so the other four recovery tiles are untouched by construction.

```ts
export type RecoveryBand = "green" | "yellow" | "red" | "none";

/**
 * Which third of Whoop's published 0–100 recovery scale a score falls in, at
 * Whoop's own published cut points (67 and 33). This is a LABELLING, not a
 * recomputation: the house rule forbids re-deriving a server figure — an
 * average, a band bound, a z-score — and naming a third of a fixed scale
 * introduces no statistics. Single-sourced here for the reason this file
 * already exists: three hand-rolled copies of a threshold switch is how `52`
 * ends up green on one tile and yellow on another.
 */
export function recoveryBand(score: number | null): RecoveryBand { … }

/** The word Whoop trained every one of its users to read beside the score. */
export function recoveryBandWord(band: RecoveryBand): string { … }

/**
 * Re-toned to v0.4, never Whoop's saturated traffic light. Red really is
 * `--danger`, and the recovery SCORE is the one place in this family that is
 * allowed: this file's contract that a suppressed HRV morning reads `--warning`
 * ("information, not an emergency") still holds for HRV, but a sub-33 recovery
 * score is Whoop's own red shown in Whoop's own app, and softening it means the
 * log disagrees with the device the number came from. Text and mark weight
 * only — a red word or a red bar, never a red row.
 */
export function recoveryBandColor(band: RecoveryBand): string { … }
```

Plus one formatter, because `int` and `ms` in the mockup are the same function
written twice:

```ts
/** A server figure rounded for display, or `—`. Never re-derived, only rounded. */
export function round(v: number | null): string {
  return v === null ? "—" : String(Math.round(v));
}
```

`weekday` already exists and is used as-is; the mockup's `shortDay` is a
duplicate of it and is not carried over. `recoveryBandTint` and `nightMark` are
not carried over either — the tint belonged to `week-columns` and `nightMark`
adapts a day into the `NightMark` that `nightColor`/`nightOpacity` take, which
this variant does not use because it paints no HRV status.

### States

Every one of these renders correctly and each gets a test:

- **Default** — the DX's headline fixture: a red weekend, a strong Monday, no
  reading yet today. Sunday's 29 must be findable in under a second, as a short
  `--danger` bar in a rail that is otherwise warm, and it must be legible in the
  detail rows without scanning columns.
- **`full-week`** — every day present, today landed. Mostly greens and yellows,
  and the whole card should read **almost quiet** beside Steps and Weather. This
  is also the fixture the baseline tick's contrast is judged on.
- **`red-morning`** — today at 22. `--danger` at text and mark weight, one short
  red bar and one red word. True and worth noticing; never an alarm.
- **`sparse`** — 3 readings in 8 days. The rail shows ghost slots and the detail
  rows show *no reading*. **This is the fixture the idiom was selected on** and
  it must look intentional. Assert that a ghost slot and a low-score bar render
  as structurally different marks, not merely different heights.
- **`calibrating`** — every average null, every day `unknown`. Readings still
  print in the detail rows and the bars still draw (the score exists; only the
  *normal* does not); the seam reads `n of 14 mornings` and **no tick is
  drawn**. No `NaN`, no empty frame.
- **`partial-band`** — the oldest days carry `baselineAvg: null` /
  `status: "unknown"`. This variant reads no per-day band field, so those days
  are ordinary days here; assert that explicitly, because it is the one place
  the tile is *supposed* to be indifferent to a field the sibling tiles care
  about deeply.
- **Partial morning** — score null, `restingHr` and `hrv` present. `—` for the
  score, **no band word**, both figures printed, a ghost slot on the rail.
- **No reading yet today** — the 7am state. `Today · no reading` as the first
  detail row, ghost slot as the rail's last position, everything else intact.
- **Legacy payload** — `days` or `baseline` absent. The existing *"Log is
  calibrating."* body, preserved verbatim.
- **Both breakpoints** — one-third desktop (~14.4px bar pitch) and full-width
  mobile (~19px).

### Testing

**`recovery/fixtures.ts`** — extended additively. Existing exports are consumed
by four other tiles' tests and must not change.

The gap is that `makeDays` derives `recoveryScore` as `48 + (i % 30)`, a
sawtooth spanning 48–77, so it produces yellows and greens and **never a red**
— the fixtures cannot currently express the default fixture's story at all. Add:

- `bandedDays(scores)` — a day series taking explicit recovery scores so a test
  can state the band sequence it is asserting on, rather than reading it out of
  a modulus.
- `recoveryLogView()` — the DX's default fixture, 31 days ending in a red
  weekend (35, 29), a rebound Monday (52), and a null today. The deliberately
  ugly floats (`112.44031`, `83.242966`, `96.10188`) are carried over verbatim;
  they are real Whoop values and a variant that wraps on them has failed.
- `sparseView()` — 3 of the last 8 days present.
- `partialMorningView()` — a day with `recoveryScore: null` and both other
  readings present.

**`recovery/shared.test.ts`** — additive:

- `recoveryBand` at the cut points exactly: `67` is green, `66` is yellow, `34`
  is yellow, `33` is red, `0` is red, `100` is green, `null` is none. The
  boundaries are the whole point of single-sourcing this.
- `recoveryBandColor` maps red to `var(--danger)` — pinning the one deliberate
  exception to this file's own no-danger-red contract, so a future reader does
  not "fix" it to warning.
- `round(96.10188)` is `"96"`, `round(null)` is `"—"`.
- The existing suite passes unmodified.

**`recovery/morning-ledger.test.tsx`** — **rewritten.** Every current assertion
describes behaviour this SOW deletes: the header's `ms · bpm · score` order, the
four-row window, the `▼` delta glyph and its `--warning` colour. Do not adapt
them; write the file against the new brief.

- The seam prints the baselines in **score · bpm · ms** order, rounded.
- **`96.10188` never appears in the DOM and `96` does.** The regression test for
  the wrap, and the one that must survive every future refactor. A fixture with
  a non-integral `recoveryScore` (`52.4`) renders `52` for the same reason.
- **No `▼`/`▲`/`▬` glyph appears anywhere**, and no element carries
  `hrvStatusColor`'s tokens — the HRV delta is gone, not relocated.
- The rail renders 14 positions for a full fixture; the count of band-coloured
  bars equals the count of non-null `recoveryScore` days in the window, and the
  remainder are ghost slots.
- **A ghost slot is full-rail-height and a low-score bar is not.** This is the
  `sparse` correction; assert on the rendered height relationship, not on the
  colour, so the test fails if someone reverts to a 2px stub.
- `railY` is exported and pinned: a score equal to `recoveryScoreAvg` produces a
  bar height equal to the tick's `bottom`. This is the tick-desynchronisation
  guard.
- The tick is absent on the calibrating fixture and present on the default one.
- The rail container carries `role="img"` and a non-empty label naming the
  window length and the baseline.
- Detail rows are newest-first with `Today` leading, and there are exactly
  three.
- **The partial-morning row renders `—` for the score and does *not* render the
  string `No reading`** while printing both other figures. The correction's
  regression test.
- The all-null morning renders one *no reading* row, and the 7am fixture renders
  it for `Today`.
- **Calibrating counts mornings, not nights**: the fixture sets
  `recoveryScoreDays: 9` and `hrvDays: 22`, and the tile prints `9`, not `22`.
  This fails against the mockup and is the correction's regression test.
- Each of the three baseline figures dashes independently: a fixture with
  `hrvAvg: null` and a live `recoveryScoreAvg` prints the score and `— ms`.
- The legacy fixture still renders *"Log is calibrating."*

**Regression** — `dashboard/page.test.tsx`, `tile-renderer.test.tsx`, and the
other four recovery tiles' suites must pass unmodified. If any needs editing,
either the catalog or `shared.ts` has been disturbed and that is out of scope.

### Design system

`scope: in-system`, inherited from the DX. Every colour is a v0.4 token —
`--foreground`, `--muted`, `--faint`, `--surface-2`, `--border`,
`--border-strong`, `--success`, `--warning`, `--danger` — and no raw hex appears
in the component. Type is Manrope with `tabular-nums` on every figure and
`font-mono` on the `bpm · ms` group, matching the tile's neighbours. The panel,
radius, and padding are `MiniCard`'s and are not overridden.

Two conformance notes, because they are where this could go wrong. **Whoop's
saturated traffic light is not imported** — the bars are `--success` /
`--warning` / `--danger` as re-toned, and a fortnight of them must read as a
quiet card. And **the rail is not a chart**: no axis, no gridline, no label on a
bar. It is a shape with one reference tick.

The card body lands at ~154px (40 rail + 8 + ~21 seam + 8 + ~77 detail) against
the shipped tile's ~180px and the ticket's 260px ceiling, so this redesign
**costs the dashboard row no height** — it takes slightly less. That is a
property worth keeping; `TileGrid` has no span support, so any growth here makes
every tile beside it taller.

### Documentation

- Rewrite the component's file-header comment. The current one describes the
  four-row ledger and its HRV delta glyph, and would otherwise become the most
  misleading text in the file. It should say what the two registers are, why the
  rail shows fourteen days while the detail shows three, why an absent morning is
  a full-height ghost rather than a short bar, and why the tile paints no HRV
  status.
- `lib/dashboard-tiles.ts`: the `recovery_log` description becomes something
  like *"A fortnight of recovery, and your last three mornings in full."*
  ("Your last few mornings as dated readings" is now wrong in both halves.)
- Set [`dx/recovery-log-tile.md`](../dx/recovery-log-tile.md) to
  `status: selected` with `selected_idioms: [band-rail-and-recent]`, and add the
  selection note at the top pointing at PR #161 and at this SOW, following the
  pattern in [`dx/hrv-balance-tile.md`](../dx/hrv-balance-tile.md).

## Open Questions

1. **What should the detail register do when all three recent days are empty?**
   On the `sparse` fixture it is reachable: three *no reading* rows under a rail
   that clearly shows readings a week ago, which is the tile's worst state and
   the one the DX's selection criteria single out. Options: keep the last three
   calendar days; show the last three days *with readings*, labelled by weekday;
   grow the register until three readings are found. **Tentative lean: keep the
   last three calendar days.** The two registers would otherwise disagree about
   what window they cover — the rail's rightmost position and the detail's top
   row would be different days — and the weekday labels make the honest version
   readable anyway (*Today · no reading* over *Mon · 52 · Adequate*). The rail
   is already carrying the "there is history here" message, which is the job the
   detail rows would be borrowed for. Worth revisiting only if it reads as a
   broken tile on the real preview.

2. **Should `MIN_BASELINE_DAYS = 14` stop being a client-side literal?** It is
   `config.Recovery.MinBaselineDays` on the server, the tile hardcodes it, and
   `hrv_balance` hardcodes it too — so retuning it silently makes two tiles lie
   about their own progress. Options: leave it and accept the duplication; emit
   the threshold on the baseline payload beside the day counts; drop the
   denominator and print *"9 mornings so far."* **Tentative lean: leave it in
   this SOW** — it is a pre-existing duplication in a second tile, fixing it
   properly is an API change this SOW is explicitly scoped out of, and doing it
   here would mean shipping a payload field for a cosmetic denominator. It is a
   good candidate for a small follow-up that fixes both tiles at once.

3. **Should `recoveryBand*` now colour the score on `recovery` and
   `morning_vitals` too?** They print the same figure in neutral ink, and once
   the band vocabulary is single-sourced the inconsistency becomes visible: the
   same 29 is red on the log and grey on the two tiles beside it. But
   `readiness-verdict.tsx` refuses to colour its score *on purpose*, because it
   already spends its colour on the verdict word and two traffic lights on one
   card is confusion. **Tentative lean: leave both alone and treat this as its
   own question.** The log's colour budget was unspent, which is exactly why the
   ticket gave the band to this tile; the other two have already spent theirs,
   and reopening that is a decision about the recovery family as a whole rather
   than a footnote to one tile's redesign.
