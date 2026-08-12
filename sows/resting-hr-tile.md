---
status: draft
repos:
  - prog-strength-api
  - prog-strength-web
  - prog-strength-docs
---

# Resting HR Tile

**Status**: Draft · **Last updated**: 2026-08-12

## Introduction

[`dx/resting-hr-tile`](../dx/resting-hr-tile.md) explored five compositions of a
**new** `resting_hr` mini-card and **`sorted-strip` was selected** (DX comparison
PR Prog-Strength/prog-strength-web#164, closed un-merged). This SOW builds it for
real.

Three tiles on the dashboard already print resting heart rate. `recovery` prints
today's as a contributor row with a `vs 30d` chip; `morning_vitals` prints
today's as one of three cells with the same chip; `recovery_log` prints the
30-day average in its seam and three recent mornings in its detail rows. All
three answer *"what is my resting HR this morning, and is it above or below
normal?"* and the longest resting-HR history anywhere in the product is **three
days**.

That is the gap. A resting heart rate that has climbed from 48 to 56 and stayed
there for five days is the single most legible early signal a wearable produces
— illness incubating, a training block that has tipped from productive to
grinding, sleep debt accumulating — and a sustained shift is invisible in any
window three days wide. Sunday's 59 reads as one bad night on `recovery_log`;
across a month it might be the fourth day of a climb, and nothing in the product
can currently tell the user which.

The selected idiom answers a sharper version of that question than "plot the
month" would. **`sorted-strip` ranks the morning instead of dating it**: the last
thirty mornings are sorted low to high as a strip of thin ticks, today's tick is
filled and labelled, the athlete's own 30-day average is a second dashed tick,
and a caption reads `4th lowest of your last 30`. Its argument is that a user
does not actually know whether 50 is good *for them* — a 50 is excellent for one
athlete and elevated for another — and no amount of dated history answers that as
directly as showing them where today falls in their own month. It is the same
reasoning behind `hrv_balance`'s distribution gauge, applied to the metric where
an absolute number is least interpretable.

It was also the deliberate outlier in the spread, the one variant that does not
primarily answer "over time", and it won on sight. Two properties earned it:
**it is the only variant whose main graphic survives `calibrating` intact** — a
rank needs no baseline, only a distribution — and it stays visibly distinct from
`hrv_balance` beside it, because it draws no line.

What changes for the user: a fifth recovery tile that answers *"is this a good
morning, for me?"* — with the actual numbers on it, and without the tile ever
rendering a judgement the server never made.

## Proposed Solution

A new `RestingRankCard` in the recovery tile family, rendered from the same
shared `recovery` section its four siblings read, inside the standard `MiniCard`
shell. Four registers, top to bottom:

1. **The hero** — today's bpm at 20px, with its rounded value repeated at 9px
   directly over its own tick on the strip below.
2. **The strip** — the last thirty mornings as thin vertical ticks, sorted
   ascending left to right, evenly spaced by *rank* rather than by magnitude.
   Today's tick is filled and full-height; the 30-day average is a dashed tick
   inserted at the boundary it belongs to; the extremes are printed at the ends
   as `47 lowest` / `59 highest`.
3. **The caption** — `4th lowest of your last 30`, the single line that makes the
   whole card make sense, carrying the calibrating progress as a suffix when
   there is no average yet.
4. **Recent mornings** — the last three in date order with their values, so
   chronology is demoted but not discarded.

Unlike both recovery DXs before it, this one **adds a catalog entry**, and that
is where this SOW's scope diverges from what the DX predicted — see
*The catalog change* below. The tile is added to the catalog in **both** repos
and ships **tray-only**, not in the default layout.

The mockup is not promoted. It is reimplemented against the real `MiniCard` and
the family's existing `recovery/shared.ts`, which already exports `round` (the
mockup's `bpm`) and `weekday` (the mockup's `shortDay`); only `ordinal` is
genuinely new. Four corrections to the mockup are specified below and are the
substance of this work beyond transcription:

- **Ranking runs on integers**, so two mornings the card both prints as `50`
  cannot be ranked apart.
- **Colour is gated on the 30-day average, not on the upper third** — the one
  place the DX contradicts itself, resolved in favour of the card not
  disagreeing with the average tick it draws.
- **Labels are anchored and de-conflicted** so the avg label can neither leave
  the card nor collide with an endpoint label.
- **The recent rows read newest-first**, matching `recovery_log`, which may be
  sitting directly beside this tile.

Nothing recomputes a server figure. The card's only arithmetic is counting how
many of the athlete's own mornings sat below today's, which is **position** — the
thing the DX licenses explicitly — and never a threshold classifying a bpm value.

## Goals and Non-Goals

### Goals

- A new `resting_hr` tile renders the `sorted-strip` composition,
  production-quality, conforming to [`design-system.md`](../design-system.md)
  v0.4.
- The catalog gains `resting_hr` in **both** the Go source of truth
  (`internal/dashboard/tiles.go`) and its TypeScript mirror
  (`lib/dashboard-tiles.ts`), in the same position, so the contract tests on
  both sides pass and a user can actually add the tile.
- `resting_hr` joins the **recovery family** in `handler.go`, so a layout whose
  only recovery-family tile is `resting_hr` still gets a `recovery` section
  built.
- The tile ships **tray-only**: it is not in the default layout, pinned by a Go
  test mirroring `TestSummary_DefaultLayoutHasNoSleepTile`.
- **bpm renders as an integer everywhere**, and the *ranking* runs on the same
  rounded values, so the strip's order and the printed digits never disagree.
- The calibrating state gates on **`restingHrAvg` / `restingHrDays`** and on
  nothing else — never `hrvDays`.
- Colour is spent on exactly one thing: today being **above** the athlete's own
  30-day average, at the rounded delta. No `--danger`, no green for a low
  morning, no client-side classification, no client-side re-averaging.
- The strip is drawn from `days[]` — never from `spark`, which omits missing
  mornings.
- All six DX states render correctly at a fixed height: **default, creeping-up,
  flat-month, no-reading-yet, sparse, calibrating** — plus the not-connected
  state and both breakpoints.
- The card's body height is **constant across all six states** and stays inside
  the DX's ~260px whole-card ceiling.
- `ordinal` and `MIN_BASELINE_DAYS` are single-sourced into `recovery/shared.ts`
  rather than kept as private copies, following the `recoveryBand*` precedent.

### Non-Goals

- **No API data change.** Every figure this tile draws already ships.
  `RecoveryView`, `RecoveryDayPoint`, `RecoveryBaselineView`, and the recovery
  section builder are untouched. The API change in this SOW is **catalog and
  routing only** — see below.
- **No resting-HR band, z-score, status, or trend.** Adding an RHR equivalent of
  `recoverytrend`'s HRV machinery is a reasonable follow-up and is explicitly
  not in scope; this tile is built inside the constraint that no such verdict
  exists. See Open Question 3.
- **No change to the other four recovery tiles' behaviour.** `shared.ts` changes
  are additive; the `MIN_BASELINE_DAYS` move is a constant relocation with no
  behaviour change, and every existing recovery test must pass unmodified.
- **No `/recovery` deep-page change.** The tile links into the page; promoting
  the strip onto it is a follow-up.
- **No default-layout change.** The default stays
  `[running, lifting, steps, nutrition, bodyweight, (recovery), streak]`.
- **No retirement of any existing tile.** `morning_vitals` and `recovery` keep
  printing today's resting HR with a `vs 30d` chip. Whether the family is one
  tile too wide is a question for after this ships — see Open Question 2.
- **No mobile change.** `prog-strength-mobile` carries no tile catalog.
- **No interactivity.** The whole card is a `next/link`; no range switcher, no
  scrub, no tooltip. Controls belong on the deep page.
- **No promotion of the DX branch code.** `app/design-explore/**` is throwaway
  and stays on its closed PR.

## Implementation Details

### The catalog change, and why this SOW touches `prog-strength-api`

The DX ticket says, in a blockquote near the top: *"Do **not** add
`prog-strength-api` to `repos:`"*. **That instruction is wrong, and this SOW
deliberately overrides it.** It is right about the thing it was reasoning about —
no new field, no new computation, no payload change — and wrong about the
catalog, which is server-owned.

The tile catalog is not a client list. `internal/dashboard/tiles.go` is the
source of truth and `lib/dashboard-tiles.ts` says so in its own header comment
(*"The Go `Catalog` and this `TILE_CATALOG` must stay identical in id set and
order"*). Two mechanisms enforce it, and both would defeat a web-only change:

1. **The layout write path rejects the id outright.** `layout_handler.go`
   validates every incoming tile id against `ValidTileID` and returns
   `400 unknown tile id resting_hr; valid ids: …`. A user who added the tile
   from the tray would get an error toast and the tile would never persist.
2. **The read path filters it.** `Layout.Normalize` drops any id failing
   `ValidTileID`, so even a layout blob written by some other route would be
   silently repaired away on the next read.

And a third would fail in CI before either could be observed: `tiles_test.go`
asserts `Catalog`'s exact contents and order, and `lib/dashboard-tiles.test.ts`
asserts `TILE_CATALOG.length === 20` plus the same id list. Changing one side
alone breaks that side's own test suite.

So the API changes, minimally and with no payload consequence:

| File | Change |
| --- | --- |
| `internal/dashboard/tiles.go` | New `TileRestingHR TileID = "resting_hr"` constant; added to `Catalog` **immediately after `TileRecoveryLog`**, before `TileSleep`. |
| `internal/dashboard/handler.go` | `resting_hr` added to the `recoveryFamily` slice. |
| `internal/dashboard/tiles_test.go` | Both the `all` list and the `TestCatalog_Order` `want` list gain the id in the same position; `TestValidTileID` gains a `resting_hr` case. |
| `internal/dashboard/summary_layout_test.go` | A new `TestSummary_DefaultLayoutHasNoRestingHrTile`, mirroring the sleep one; plus a test that a layout of `[resting_hr]` alone yields a `recovery` section key. |

**The `recoveryFamily` addition is the one that is easy to miss and would ship a
broken tile.** The recovery section is built once, when *any* family tile is
enabled, and emitted under the single `recovery` key:

```go
recoveryFamily := []TileID{
    TileRecovery, TileHRVBalance, TileMorningVitals, TileRecoveryLog, TileRestingHR,
}
```

Without the last entry, a user whose only recovery tile is `resting_hr` gets a
response with no `recovery` section, `data.recovery.present` is false, and the
tile renders the connect CTA forever — to a user who is already connected. There
is no type error and no test failure to catch it, which is exactly why it is
called out here.

**Position in the catalog is load-bearing** because catalog order fixes the
add-tile tray order. `resting_hr` sits at the end of the recovery run so the tray
reads `recovery, hrv_balance, morning_vitals, recovery_log, resting_hr, sleep` —
the family stays contiguous and `sleep` keeps its own position after it.

On the web side:

| File | Change |
| --- | --- |
| `lib/dashboard-tiles.ts` | `"resting_hr"` added to the `TileId` union and a `TILE_CATALOG` entry in the same position. |
| `lib/dashboard-tiles.test.ts` | Count `20 → 21`; the order list and the `ALL_TILE_IDS` exhaustiveness record both gain the id. |
| `app/(app)/dashboard/_components/tile-renderer.tsx` | New `case "resting_hr"`, following the family pattern exactly. |

The catalog entry:

```ts
{
  id: "resting_hr",
  title: "Resting HR",
  href: "/recovery",
  description: "Where this morning's resting heart rate ranks in your own month.",
}
```

The title is `Resting HR` in sentence-plus-initialism case; `MiniCardTitle`
uppercases it for display, so it renders as `RESTING HR` without the catalog
storing a shouted string (the same way every other entry is stored).

The renderer case:

```tsx
case "resting_hr":
  return data.recovery.present ? (
    <RestingRankCard section={data.recovery} href={href} />
  ) : (
    <RecoveryConnectCard title="Resting HR" href={href} />
  );
```

`tile-renderer.tsx`'s `never` default is the compile-time guard the SOW relies
on: adding the id to the union without adding this case is a type error.

**Default layout: tray-only.** `defaultLayout` in `layout_resolve.go` stays
`[running, lifting, steps, nutrition, bodyweight, (recovery if Whoop), streak]`.
Three tiles already print today's resting HR; a fourth arriving unbidden on
everyone's dashboard is a bigger product decision than this SOW's remit, and
`sleep` set the precedent for shipping a catalog tile opt-in. The Go test pinning
it is part of this work so the choice is deliberate rather than incidental.

### File layout (web)

| File | Responsibility |
| --- | --- |
| `app/(app)/dashboard/_components/recovery/resting-rank.tsx` | **New.** The tile: the four registers, the guard, the states. |
| `app/(app)/dashboard/_components/recovery/resting-rank.ts` | **New.** Pure ranking and geometry — `rankOf`, `stripTicks`, `avgInsertPct`, label anchoring. No React. |
| `app/(app)/dashboard/_components/recovery/shared.ts` | **Extended.** `ordinal` and `MIN_BASELINE_DAYS` join the existing formatters. |
| `app/(app)/dashboard/_components/recovery/morning-ledger.tsx` | **Touched.** Its private `MIN_BASELINE_DAYS` becomes an import. |
| `app/(app)/dashboard/_components/recovery/hrv-tile.tsx` | **Touched.** The literal `14` in `Calibrating` becomes the same import. |
| `app/(app)/dashboard/_components/recovery/fixtures.ts` | **Extended.** A fourth generation of fixtures for the six RHR states. |

The pure module is separated for the same reason `hrv-chart.ts` was: the ranking
is where every correction below lives, it is where the tests earn their keep, and
it is the natural import if the strip is ever promoted to `/recovery`.

The component name is `RestingRankCard`, not `SortedStripCard`. The family names
its files after what the card *says* (`readiness-verdict`, `morning-ledger`,
`balance-band`), not after the idiom that produced it; `sorted-strip` is the DX's
vocabulary and should not outlive the DX.

### Register 1 — the hero

```tsx
<p className="tabular-nums tracking-[-0.03em] text-[20px] leading-[22px] text-[var(--foreground)]">
  {round(todayValue)}
  <span className="ml-1 text-[10px] tracking-normal text-[var(--muted)]">bpm</span>
</p>
```

`round` is `shared.ts`'s existing formatter — `null` becomes `—`, and a float
becomes an integer. **The DX's fixture puts `49.6` on the 11th on purpose: a card
that prints `49.6 bpm` has failed before it is compared.** `resting_heart_rate`
is a `*float64` in the API's DTO and nothing on the wire guarantees it arrives
whole.

The hero is deliberately modest at 20px. `morning_vitals` and `recovery` both
print today's resting HR already; if the biggest element on this card were
today's number, the dashboard would own a third today-card. The card's job is the
**month**, and the 20px hero is sized so the strip beneath it is what the eye
lands on.

### Register 2 — the strip

Thirty mornings, sorted ascending, evenly spaced by rank. The strip encodes
**order**, not magnitude; magnitude lives in the printed extremes. That is the
Robinhood 52-week-range idiom taken structurally, and it is what makes
`flat-month` legible without a chart: the ends read `48 lowest` / `50 highest`,
and a two-beat range is instantly readable as *nothing is happening*, with no
auto-scale to lie about it.

#### Rank runs on integers — correction 1

The mockup computes `rank` from raw floats:

```ts
// mockup — NOT what ships
const rank = values.filter((v) => v < todayValue).length + 1;
```

With the DX's own default fixture that ranks `49.6` strictly below `50` while the
card prints both as `50`. The user sees two ticks it calls fifty, sitting at
different ranks, and a caption whose ordinal counts a difference the card never
shows them.

This is the same lesson the `flat-month` fixture taught the DX about colour —
recorded in its `isAbove` docstring as *"a difference the card does not print is a
difference the card does not colour"* — applied to order instead of ink. **Round
first, then sort and rank:**

```ts
/**
 * The athlete's last `window` mornings as INTEGER bpm, sorted ascending.
 *
 * Rounding before sorting is not cosmetic. Every figure on this card prints as
 * an integer, so ranking on the raw floats would order two mornings the card
 * both calls `50` — and then print an ordinal that counts a difference the user
 * cannot see. Round first and the strip's order, the endpoint labels, and the
 * caption all describe the same numbers.
 *
 * Nulls are dropped, not zero-filled: a strap-off morning is an absent reading,
 * and `n` reports how many mornings are actually behind the rank.
 */
export function sortedMornings(days: RecoveryDayPoint[], window: number): number[] {
  return days
    .slice(-window)
    .map((d) => d.restingHr)
    .filter((v): v is number => v !== null)
    .map((v) => Math.round(v))
    .sort((a, b) => a - b);
}

/**
 * Today's rank among them, 1-based, with ties sharing the LOWER rank — two
 * identical 48s are both "1st lowest" rather than arbitrarily ordered. Null when
 * there is no reading yet today.
 */
export function rankOf(sorted: number[], today: number | null): number | null {
  if (today === null) return null;
  const t = Math.round(today);
  return sorted.filter((v) => v < t).length + 1;
}
```

The window is the **last 30 mornings including today** (`days` carries 31
entries, so the oldest is excluded). `n` is the count of non-null readings in it,
which is what the caption reports — on `sparse`, `of your last 24` is the honest
number and it is visibly not thirty.

#### Tick geometry

```ts
/** Tick centres, evenly spaced by rank. */
export const tickPct = (i: number, n: number): number => ((i + 0.5) / n) * 100;

/**
 * Where the 30-day average inserts. Deliberately at the BOUNDARY between two
 * ticks — `k / n`, exactly halfway between tick k−1 and tick k — because the
 * average is not one of the athlete's mornings and must not appear to be one.
 * Computed against the raw average, not a rounded one: the tick is a position,
 * and 53.4 genuinely sits above every 53.
 */
export function avgInsertPct(sorted: number[], avg: number | null): number | null {
  if (avg === null) return null;
  return (sorted.filter((v) => v < avg).length / sorted.length) * 100;
}
```

The average tick is **dashed**, via a `repeating-linear-gradient` in `--muted`,
for the same reason: it is a statistic, not a morning. This is the mockup's
decision and it is right; keep it and keep the comment explaining it.

Note that the average's own window (30 days trailing, *excluding* today) is not
the same set as the strip's thirty (including today, excluding the oldest). That
is not a defect and must not be "fixed" by re-averaging the strip's values, which
is the forbidden operation. The tick claims a position, not membership, and the
dash is what says so.

#### Label anchoring and de-confliction — correction 3

The mockup positions three labels with `left: {pct}%` and
`transform: translateX(-50%)`. Today's label leaves the card whenever today is
near an extreme (`tickPct(0)` is 1.7%, so a centred label sits mostly outside the
panel), and the `avg` label collides with `lowest` or `highest` whenever the
recent month sits mostly on one side of the baseline — which is precisely what a
sustained climb looks like, i.e. the fixture the tile exists for.

**Today's label switches anchor rather than clamping**, so it stays over its own
tick:

```ts
/** left / centre / right anchoring, so a label near an edge stays on the card
 *  AND stays adjacent to the tick it names. */
export function labelAnchor(pct: number): { left: string; transform: string } {
  if (pct < 6) return { left: "0%", transform: "none" };
  if (pct > 94) return { left: "100%", transform: "translateX(-100%)" };
  return { left: `${pct}%`, transform: "translateX(-50%)" };
}
```

**The avg label yields to the card edges and wins against the endpoints.** It is
the more informative figure, and the extreme values remain visible as the
outermost ticks even without their captions:

```ts
export const AVG_LABEL_MIN_PCT = 18;
export const AVG_LABEL_MAX_PCT = 82;
/** Clearance either side of the avg label before an endpoint label is dropped. */
export const ENDPOINT_CLEARANCE_PCT = 16;

export function endpointRow(avgPct: number | null) {
  if (avgPct === null) return { avgLabelPct: null, showLowest: true, showHighest: true };
  const avgLabelPct = Math.min(AVG_LABEL_MAX_PCT, Math.max(AVG_LABEL_MIN_PCT, avgPct));
  return {
    avgLabelPct,
    showLowest: avgLabelPct >= AVG_LABEL_MIN_PCT + ENDPOINT_CLEARANCE_PCT,   // ≥ 34
    showHighest: avgLabelPct <= AVG_LABEL_MAX_PCT - ENDPOINT_CLEARANCE_PCT,  // ≤ 66
  };
}
```

The avg **tick** always draws at its true `avgPct`; only its label is clamped, so
the geometry never lies even when the caption is nudged. On the DX's own
fixtures `avgPct` lands mid-strip (`default` ~50%, `creeping-up` ~57%,
`flat-month` ~35%) and nothing is suppressed — this machinery exists for the
athlete the fixtures do not cover, which is why it is unit-tested rather than
eyeballed.

#### The strip's accessibility

The ticks are decoration; the meaning is in the caption and the recent rows,
which are real text. Following `morning-ledger`'s `railLabel` precedent, the tick
container takes `role="img"` with a composed label and every tick is
`aria-hidden`:

> *"Today's resting heart rate of 50 bpm is the 4th lowest of your last 30
> mornings, which ranged from 47 to 59 bpm, against a 30-day average of 53."*

with the two degenerate forms — no reading today, and no average yet — spelled
out rather than interpolated into a sentence with a `—` in it.

### Register 3 — the caption, and its fixed height

```
4th lowest of your last 30
```

`ordinal` moves into `shared.ts` (see *Shared helpers*). When `restingHrAvg` is
null the caption gains the calibrating progress — `no avg yet, 9 of 14 mornings`
— and **that suffix goes on its own line, always reserved.** The mockup appends
it inline, which wraps at a one-third desktop cell and grows the card by a line
exactly when the user's dashboard is newest. Reserve the second line
unconditionally (`min-h` of two lines' leading) so the card's height is identical
in all six states; an empty reserved line is invisible and a 12px jump on
calibration is not.

This is the same class of defect as the `hrv_balance` chart's unreserved height:
a layout shift on a tile that shares its row with unrelated tiles moves the whole
row.

### Register 4 — recent mornings — correction 4

Three rows, `Today / Tue / Mon`, each printing `50 bpm` or `no reading`, above a
`--border` hairline. **Newest first**, which is a change from the mockup's
oldest-first ordering.

The argument is adjacency, not preference: `recovery_log` prints the same three
mornings in the same 10px register with `days.slice(-3).reverse()` — today at the
top — and the two tiles can sit side by side in the same recovery section. Two
neighbouring cards listing the same three days in opposite orders is the kind of
disagreement the DX's own selection criteria call out. The strip is sorted by
value, not by date, so it supplies no counter-argument from within the card.

An absent morning prints `no reading` in `--faint` rather than an em-dash — the
row has space for the word, and `sparse` should read as intentional.

### Colour: the one place the DX contradicts itself — correction 2

The DX states a colour contract and calls it binding on all five variants:

| Situation | Treatment |
| --- | --- |
| A reading **above** the 30-day average | `--warning` |
| A reading **at or below** the average | Neutral ink |
| An absent morning | `--surface-2` |

Its `sorted-strip` idiom description then says something different: *"`--warning`
appears only if today's tick falls in the upper third of the athlete's own
month"*, which the mockup implements as `rank > (n * 2) / 3`. The two rules
disagree in a real band — a morning above the average but not in the top third
is warm under the contract and neutral under the mockup.

**This SOW takes the contract: colour is gated on being above the 30-day
average**, at the rounded delta, via the family's existing rounding discipline.
Three reasons, in order of weight:

1. **The card draws the average tick.** Under the upper-third rule, today's tick
   can sit visibly to the *right* of the dashed average tick and still be painted
   neutral ink. That is the card contradicting its own graphic, and it is the
   worst failure mode available to a tile whose entire claim is that position is
   the meaning.
2. **The dashboard must not disagree with itself.** `recovery` and
   `morning_vitals` print `vs 30d` for the same morning, from the same section,
   possibly in the same row. A morning they mark as up should not be neutral
   here.
3. **Rounding ties ink to digits.** `isAbove` is defined on the *printed* delta —
   a 49 against a 48.9 average rounds to `0` and takes no colour — which is the
   rule the `flat-month` fixture forced on the DX and which the upper-third rule
   cannot express, because it never computes a delta at all.

The upper-third rule's one genuine advantage is that it survives `calibrating`,
where there is no average to be above. That is not worth the contradiction:
**when `restingHrAvg` is null, the card spends no colour at all.** Calibrating
means the server has not established a normal, and a tile that paints a morning
warm anyway has invented the verdict the DX spends a whole section forbidding.
The rank caption still prints, which is the variant's selling point and is
untouched by this.

Concretely, the whole colour budget:

```ts
/**
 * The one question colour is allowed to ask: is this morning above the
 * athlete's own 30-day mean? A DIRECTION, never a verdict — `+1` is coloured
 * exactly like `+9`, because both mean the one thing this card may say.
 *
 * Tested on the ROUNDED departure. Every figure here prints as an integer, so a
 * difference the card does not print is a difference the card does not colour.
 */
function isAbove(value: number | null, avg: number | null): boolean {
  if (value === null || avg === null) return false;
  return Math.round(value - avg) > 0;
}
```

`--warning` reaches exactly two elements: today's tick on the strip, and the
ordinal phrase in the caption. Every other tick is `--border-strong`, the
average tick is `--muted`, the recent rows are `--foreground` / `--faint`. There
is **no `--danger` anywhere on this card** — red is licensed in this family for
one thing only, a sub-33 Whoop recovery score, which is Whoop's own red in
Whoop's own app; a resting HR has no published threshold and inventing one is the
classification the DX forbids. And **a low morning is not painted green**, for
the reason `balance-band` already argues: most mornings are ordinary, and
painting them all `--success` makes a healthy month the loudest card on a grid
that also carries Steps and Weather.

### Polarity: lower is better, without a legend

Every other tile on the dashboard means up-is-good. This one inverts it, and the
DX made "do I read *lower is better* without being told?" a selection criterion.

`sorted-strip` answers it **spatially, and once**: the strip is sorted ascending,
so **left is better**, and that is stated by the endpoint labels — `47 lowest` on
the left, `59 highest` on the right — and then true everywhere on the card. There
is no axis to misread and no arrow whose direction has to be decoded.

Two things follow and are binding on the implementation:

- **The sort direction is not a preference.** Reversing it to descending inverts
  the card's only statement of polarity while leaving both labels technically
  correct, which is the quietest possible way to break it. Pin it with a test
  asserting `sorted[0] <= sorted[n-1]` and that the left-hand label reads
  `lowest`.
- **The caption says `lowest`, never `best`.** *Lowest* is a fact about the
  number; *best* is a judgement the server never made. The card carries the
  polarity through arrangement and vocabulary, not through a verdict.

### The states

Every one renders correctly and gets a test.

- **default** — an ordinary week with one bump: 30-day average 53, Sunday 59,
  Monday 47, today 50, and `49.6` on the 11th. Today ranks mid-strip, the card
  spends **no colour**, and it should read calm. The float prints as `50` and its
  tick sits alongside the other 50.
- **creeping-up** — the fixture the tile exists for. Three weeks flat around 48,
  then 54/56/57/58, average still 49. Today (58) is the **highest** tick: it sits
  at the far right of the strip, hard against `58 highest`, well right of the
  dashed average tick, warm, and the caption reads `30th lowest of your last 30`.
  This variant answers `creeping-up` by *position*, which is the whole reason it
  is a candidate — the recent rows beneath, reading `58 / 57 / 56`, are what
  supply the "and it has been climbing" that a rank alone cannot.
- **flat-month** — 48–50 all month. The endpoints read `48 lowest` / `50 highest`
  and there is no chart to auto-scale, so a healthy month is boring by
  construction. This is the variant's structural advantage and the test should
  assert the endpoint labels, since they are the whole mechanism.
- **no-reading-yet** — today is null. The hero prints `—`, no tick is filled, the
  caption reads `No reading yet today`, and the strip and its average tick still
  draw from the remaining mornings. **Yesterday is never promoted into today** —
  the recent rows carry it explicitly one register below, which is where it
  belongs. This is the variant's weakest state and it is accepted knowingly; see
  Open Question 1.
- **sparse** — three readings in the last eight days. `n` drops to the true count
  of readings in the window and the caption says `of your last <n>` rather than
  claiming thirty; tick pitch widens accordingly, and the recent rows print
  `no reading` where the strap was off. Gaps must read as gaps.
- **calibrating** — `restingHrAvg: null`, `restingHrDays: 9`. **The strip renders
  intact** — a rank needs no baseline, only a distribution — with no average
  tick, no average label, no colour, and the caption's reserved second line
  reading `no avg yet, 9 of 14 mornings`. No `NaN`, no empty frame, no borrowed
  `hrvDays`.
- **no mornings at all** — `days` present but every `restingHr` null. Guard
  before dividing by `n`: render a one-line muted body, not a zero-width strip.
- **not connected** — `data.recovery.present` is false and
  `RecoveryConnectCard title="Resting HR"` renders. Handled in the renderer, not
  the card.
- **both breakpoints** — full-width single column on mobile and one-third on
  desktop. Thirty 1px ticks at a ~230px one-third cell is a ~7.7px pitch, which
  is the density the DX judged; it is not to be "improved" by thinning the
  window.

### Shared helpers

Two things move into `recovery/shared.ts`, following the `recoveryBand*`
precedent the DX handoff names explicitly.

```ts
/**
 * How many mornings the server needs behind a metric before it emits an average
 * (`internal/recoverytrend`'s `MinBaselineDays`). A client-side copy of a server
 * constant, single-sourced here because it was already written out three times
 * across this family and a fourth copy is how the four tiles start disagreeing
 * about what "calibrating" means.
 */
export const MIN_BASELINE_DAYS = 14;

/** 1 → "1st", 4 → "4th", 11 → "11th". For the rank caption. */
export function ordinal(n: number): string { … }
```

`morning-ledger.tsx` drops its private `const MIN_BASELINE_DAYS = 14` for the
import, and `hrv-tile.tsx`'s `Calibrating` interpolates it in place of the two
literal `14`s. Both are constant-substitutions with no behaviour change and both
files' existing tests must pass **unmodified** — if either needs editing,
something other than the constant moved.

The mockup's `bpm` and `shortDay` are **not** ported: `shared.ts` already exports
`round` and `weekday` with identical behaviour, and a second copy under a new
name is exactly what this section exists to prevent. The mockup's `delta`,
`signed`, `warmTint`, `weekdayIndex`, `dayInitial`, `addDays`, and `shortDate`
belong to variants that were not selected and are dropped.

### Testing

**`recovery/resting-rank.test.ts`** — the pure module, where the corrections
live:

- `sortedMornings` rounds before sorting: a series containing `49.6` and `50`
  yields two `50`s, and **this test must fail against the mockup's formula**.
- `sortedMornings` drops nulls rather than zero-filling, and returns `[]` for an
  all-null window.
- `rankOf` gives tied values the same rank: `[47, 48, 48, 48, 50]` with today
  `48` returns `2`, not `3` or `4`.
- `rankOf` returns `1` for the lowest, `n` for the highest, and `null` for a null
  reading; a `49.6` today ranks identically to a `50` today.
- `avgInsertPct` places the average at the boundary — for `[47,48,49,50]` with
  avg `48.5` it returns exactly `50`, not a tick centre — and returns `null` for
  a null average.
- `endpointRow` clamps the label to `[18, 82]` while the caller's tick keeps its
  true position; suppresses `lowest` below 34% and `highest` above 66%; and
  suppresses neither at 50%.
- `labelAnchor` returns a left anchor below 6%, a right anchor above 94%, and a
  centred one between.

**`recovery/resting-rank.test.tsx`** — one test per state above. Specifically:

- The default fixture renders **no `--warning`** anywhere and the caption's
  ordinal is in `--foreground`. (Assert on the rendered token string, which is
  how the family's existing tests assert colour.)
- The creeping-up fixture renders `30th lowest of your last 30`, colours today's
  tick, and renders `58` as the `highest` endpoint label.
- **A morning above the average but *below* the upper third is coloured** — the
  regression test for correction 2, which must fail against the mockup's
  `upperThird` rule.
- The calibrating fixture renders the strip (a non-zero tick count), renders **no
  average tick**, spends no colour, and prints `9 of 14`.
- The calibrating fixture and the default fixture render the **same card
  height** — assert the caption block's reserved class rather than a measured
  pixel, since jsdom has no layout.
- `49.6` renders as `50` and the string `49.6` is absent from the card.
- The no-reading-yet fixture prints `—` in the hero and `No reading yet today`,
  renders **no filled tick**, and does **not** print yesterday's value anywhere
  outside the recent-rows register.
- The sparse fixture's caption reports the true `n`, not 30, and renders
  `no reading` rows.
- The recent rows are newest-first: the first row reads `Today`.
- `sortedMornings` output is ascending in the rendered DOM order and the
  left-hand endpoint label reads `lowest` — the polarity pin.
- The all-null-mornings fixture renders the muted fallback and no strip.

**`recovery/fixtures.ts`** — a fourth generation, `restingDays` plus six views,
built to the DX's own series so the tile and the DX preview cannot disagree.
Each fixture's `restingHrAvg` must be the **true mean of its own thirty pre-today
readings**, as the DX's fixture file required, so a card that prints the baseline
and a card that positions a tick against it stay consistent. Add a fixture test
asserting that property directly — it is cheap and it is the thing most likely to
rot when a series is edited.

**`lib/dashboard-tiles.test.ts`** and **`tile-renderer.test.tsx`** — updated for
the new id, and `dashboard/page.test.tsx` must pass unmodified.

**Go** — `tiles_test.go` updated for the new constant and order;
`summary_layout_test.go` gains the default-layout pin and the
`[resting_hr]`-alone-yields-a-`recovery`-section test; `layout_handler_test.go`
gains a case proving `resting_hr` now validates on write, which is the test that
would have caught the whole API-scope question.

### Design system

`scope: in-system`, inherited from the DX. Every colour is a v0.4 token —
`--foreground`, `--muted`, `--faint`, `--border`, `--border-strong`,
`--surface-2`, `--warning` — and no raw hex appears in the component. The mockup
carried a `warmTint` helper with a literal `rgba(214, 184, 127, α)`; it belonged
to the fill-based variants and is **not** ported. Type is Manrope with
`tabular-nums` on every figure. The panel, radius, and padding are `MiniCard`'s
and are not overridden.

Three conformance notes, because they are where this could go wrong:

- **Robinhood's palette is not imported.** What is taken is structural — one
  strip, the current position marked against the period's extremes, almost no
  ink. Not its green.
- **The periwinkle accent does not appear on this card.** `--accent` carries
  meaning elsewhere in the family (`elevated` HRV); there is no equivalent state
  here and none is invented.
- **Recovery still has no `--discipline-*` hue**, and this tile does not start
  one.

No new tokens. No `design-system.md` change.

### Documentation

- The component's file header explains the four registers, why ranking is
  position and not classification, why the average tick is dashed, and why the
  colour gate is the average rather than the upper third. It should be readable
  by someone who never saw the DX.
- `resting-rank.ts`'s header explains the integer-first ranking with the
  "a difference the card does not print" argument, since that is the correction
  most likely to be undone by a well-meaning simplification.
- Set [`dx/resting-hr-tile.md`](../dx/resting-hr-tile.md) to `status: selected`
  with `selected_idioms: [sorted-strip]`, and add the selection note at the top
  pointing at PR #164 and at this SOW, following the pattern in
  [`dx/recovery-log-tile.md`](../dx/recovery-log-tile.md).
- **Amend the DX's "no prerequisite" blockquote** to record that the catalog is
  server-owned and that a new tile therefore always touches
  `prog-strength-api`, even when it needs no new data. The next new-tile DX will
  otherwise repeat the same wrong instruction.

## Open Questions

1. **What should the hero print at 7am, before the morning webhook lands?** As
   specified it prints `—`, because there is no morning to rank yet. That is
   honest, and the three recent rows carry the last actual reading directly
   beneath it — but it is also the state an early riser sees every single day,
   and a 20px em-dash is a weak thing to open a dashboard on. Options: leave it;
   hero the 30-day average with a `30D AVG` caption while the strip's dashed tick
   stays where it is (a server figure, labelled as what it is — not yesterday
   promoted); hero the last actual reading with its date (explicitly forbidden by
   the DX and listed only for completeness). **Tentative lean: leave it, and
   judge from real use.** The card's proposition is *"is this a good morning, for
   me?"* and at 7am the honest answer is that there is no morning yet; a hero that
   silently changes what it means is how a user learns to distrust a figure. But
   this is the state most likely to send it back for a second pass, and the
   average-hero option is cheap if it does.

2. **Is the recovery family now one tile too wide?** Four of the five recovery
   tiles print today's resting HR, and this SOW adds the fifth without retiring
   anything. `morning_vitals` in particular is three cells of *today*, one of
   which this card now covers with a month of context. Options: leave all five
   and let the tray sort it out; open a DX for `morning_vitals` now that its
   resting-HR cell is redundant; retire `morning_vitals` outright. **Tentative
   lean: leave it and watch.** Both tiles are opt-in, so nothing is forced on
   anyone, and the question is genuinely answered by which one the owner keeps on
   their own dashboard after a fortnight — which is evidence this SOW can produce
   and cannot anticipate.

3. **Should the server compute a resting-HR band?** The DX's sharpest observation
   is that this is the first recovery tile that must carry its meaning almost
   entirely without colour, and it left open whether that reads as restraint or
   as an unfinished card. If it reads as unfinished, the fix is an API SOW adding
   an RHR standard deviation, band, and status to `internal/recoverytrend` — the
   machinery already exists for HRV and is metric-agnostic in shape — **not** a
   client-side threshold, which stays forbidden either way. Options: ship this
   and judge; add the RHR band first and rebuild the card against it; never add
   one, on the grounds that a resting HR has no published clinical threshold and
   a personal band would be inventing one with more ceremony. **Tentative lean:
   ship this and judge.** The rank caption is a genuine substitute for a status
   word — *"4th lowest of your last 30"* is a stronger statement than most
   verdicts — and this variant was selected partly to find out whether that is
   enough. If it is, the band is work that never needs doing.
