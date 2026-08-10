---
status: draft
repos:
  - prog-strength-web
  - prog-strength-docs
---

# HRV Balance Tile Redesign

**Status**: Draft · **Last updated**: 2026-08-10

## Introduction

[`dx/hrv-balance-tile`](../dx/hrv-balance-tile.md) explored five redesigns of
the `hrv_balance` tile and **`garmin-status-stack` was selected**
(DX comparison PR Prog-Strength/prog-strength-web#157, closed un-merged). This
SOW builds it for real.

The shipped tile still draws what the DX indicted: a polyline through thirty
nightly readings, which invents slopes between mornings that were never passed
through and turns an ordinary month into a seismograph, and behind it a single
flat rectangle asserting that the athlete's normal range was identical thirty
days ago and this morning. That second claim is now not merely unhelpful but
**demonstrably contradicted by the payload the tile is reading**:
[`sows/recovery-baseline-drift-payload.md`](./recovery-baseline-drift-payload.md)
shipped, so every day in `days[]` carries the band as it stood on that day's own
morning, and `baseline_trend` reports whether the range itself is climbing or
sinking. The tile renders none of it.

What changes for the user: the card answers two questions instead of one. *"Was
last night normal for me?"* it already answered. *"Is my normal changing?"* — the
question that separates a training block that is working from one that is
grinding the athlete down — becomes a thing they can see, in the slope of a band
that actually moves, without reading a number or doing arithmetic.

The chosen idiom is Garmin Connect's HRV Status card taken **structurally,
register for register** — verdict word, stable figure, distribution gauge,
chart — and re-toned into design-system v0.4. Its one deliberate disagreement
with the shipped tile is worth restating because it is the most consequential
decision here: **the big figure is the 7-day average, not last night.** One
night of RMSSD is noise; Garmin heroes the stable number for that reason. It
also means the tile still prints a headline figure at 7am, before the morning
webhook lands — the state the DX made a pass/fail criterion.

## Proposed Solution

`HrvBalanceCard` keeps its `MiniCard` shell, its title, its `href`, and its
catalog entry, and its body is replaced with four hairline-separated registers:

1. **The verdict** — a status dot and word on the left, the baseline-drift tag
   on the right (`▲ +6 ms · 4w`).
2. **The stable figure** — `short_avg` at 28px with a `7D AVG` caption.
3. **The distribution gauge** — a segmented bar spanning ±2 band-widths of the
   athlete's own scale, with a tick where the 7-day average falls and the band
   bounds labelled beneath it.
4. **The chart** — thirty-one nights as discrete marks over a band polygon that
   drifts, drawn in real pixel units.

The mockup is not promoted. Three pieces of it graduate into production with
changes: a `useMeasuredWidth` hook (new, `lib/`), the drift formatters (into the
recovery tiles' existing `shared.ts`), and the chart geometry helpers (a new
`hrv-chart.ts` beside the tile). Everything else is rewritten against the real
`MiniCard`.

Two corrections to the mockup are specified below and are the substance of this
work beyond transcription. **The gauge is rescaled off the emitted bounds rather
than off `hrv_std_dev`**, so the tick and the labels under it cannot disagree
with the band the server drew. And **the chart insets its marks by their own
radius**, because the mockup — like the shipped tile — places the first and last
dot exactly on the box edge, so today's mark is drawn half outside the SVG.

Nothing recomputes a server figure. The only arithmetic is mapping an
already-computed millisecond value onto a pixel, which is what a chart is.

## Goals and Non-Goals

### Goals

- `HrvBalanceCard` renders the `garmin-status-stack` four-register composition,
  production-quality, conforming to [`design-system.md`](../design-system.md)
  v0.4.
- HRV history renders as **discrete marks, one per night**, never a polyline. A
  missing night is an absent mark, not an interpolation and not a zero.
- The band is a **polygon that drifts**, drawn per day from `days[].balanced_low`
  / `balanced_high`, breaking wherever the baseline is null.
- Each mark is coloured by **its own day's** `status`, not against today's band.
- `baseline_trend` is surfaced in register 1, showing the magnitude whenever
  `delta_ms` is non-null — including when the direction reads `steady`.
- The gauge's tick position and its printed bounds are both derived from
  `balanced_low` / `balanced_high` / `hrv_avg`, so they agree by construction
  under any `balanced_z` or SD floor.
- No `<circle>` is ever non-uniformly scaled; the SVG's viewBox equals its
  rendered pixel size.
- All six DX states render correctly: **rising, falling, suppressed,
  partial-band, calibrating, no-reading-yet** — plus the interior-gap case and
  both breakpoints.
- The catalog is unchanged: same `TileId`, title, tray description, and `href`.
- `vitest.setup.ts` gains a `ResizeObserver` stub so measured-width components
  are testable, here and for every chart after this one.

### Non-Goals

- **No catalog or layout change.** No new `TileId`, no tray copy change, no
  change to `tile-renderer.tsx` beyond what the component's own props require,
  no grid-span work.
- **No change to the other four recovery tiles.** `recovery`, `morning_vitals`,
  `recovery_trend`, and `recovery_log` are untouched. `shared.ts` changes are
  strictly additive or signature-compatible — see *Shared helpers*.
- **No `/recovery` deep-page change.** Promoting this chart to the page is a
  reasonable follow-up and is explicitly not in scope.
- **No API change.** Every figure this tile draws already ships. If a number is
  not in the payload, the tile does not show it.
- **No range switcher.** The whole card is a `next/link`; interactive controls
  belong on the deep page. Restated because it is the most likely thing to creep
  in from the Whoop reference.
- **No config change.** In particular this SOW does **not** retune
  `baseline_drift_z` — see *The drift verdict* and Open Question 1.
- **No promotion of the DX branch code.** `app/design-explore/**` is throwaway
  and stays on its closed PR.

## Implementation Details

### File layout

| File | Responsibility |
| --- | --- |
| `app/(app)/dashboard/_components/recovery/balance-band.tsx` | **Rewritten.** The tile: the four registers, the guard, the calibrating state. |
| `app/(app)/dashboard/_components/recovery/hrv-chart.ts` | **New.** Chart machinery only — `prepareHrvChart`, `bandRuns`, `scaler`. Pure, no React. |
| `app/(app)/dashboard/_components/recovery/shared.ts` | **Extended.** Drift formatters join the existing status/trend formatters; `statusWord` gains an optional parameter. |
| `lib/use-measured-width.ts` | **New.** The measurement hook. General-purpose, not recovery-specific. |
| `vitest.setup.ts` | **Extended.** `ResizeObserver` stub. |

The tile file stays a single component file because the registers are not
independently reusable — they are one composition, and splitting four
twenty-line blocks across four files would make the layout harder to read, not
easier. The chart machinery is separated because it is pure, heavily tested, and
the natural import for the `/recovery` page when that follow-up happens.

### Register 1 — the verdict

A status dot and the house status word on the left; the drift tag right-aligned.

```tsx
<div className="flex items-center justify-between gap-2 pb-2.5">
  <div className="flex items-center gap-2">
    <span
      className="h-[7px] w-[7px] shrink-0 rounded-full"
      style={{ backgroundColor: hrvStatusColor(today.status) }}
    />
    <span className="text-[13px] font-medium text-[var(--foreground)]">
      {statusWord(today.status, today.hrv !== null)}
    </span>
  </div>
  <span
    className="font-mono text-[11px] tabular-nums"
    style={{ color: driftColor(drift.direction) }}
  >
    {driftTag(drift)}
  </span>
</div>
```

The status shown is `days[last].status`, **not** `hrv.status`. They are equal by
the invariant SOW 1 pinned under test, and reading it off the same object the
marks are coloured from means the dot can never disagree with the last mark on
the chart.

`statusWord(status, hasReading)` splits `unknown` on its cause: with a reading it
is *Calibrating*, without one it is *No reading yet*. The distinction matters
because those are different situations and the shipped tile conflates them.

### Register 2 — the stable figure

```tsx
<div className="py-2.5">
  <div className="flex items-baseline gap-1">
    <span className="text-[28px] font-semibold leading-none tracking-[-0.03em] tabular-nums text-[var(--foreground)]">
      {hrv.shortAvg !== null ? Math.round(hrv.shortAvg) : "—"}
    </span>
    <span className="text-xs font-medium text-[var(--muted)]">ms</span>
  </div>
  <div className="mt-1 text-[10px] font-semibold uppercase tracking-wide text-[var(--faint)]">
    7d avg
  </div>
</div>
```

`short_avg` is null below `min_trend_days` (4 readings in the trailing 7). The
em-dash is correct there and is reachable only in a state where register 3's
tick is also absent — a user with a band but fewer than four readings this week.
It is not the 7am state: `short_avg` is computed over a window *including* today
but does not require today, so it survives a missing morning intact. That is the
property that makes this idiom the one that still prints a headline before the
webhook lands.

### Register 3 — the distribution gauge, and the scale correction

The gauge is a fixed ±2-band-width scale: a warning quarter, a success half, an
accent quarter, with a tick marking where the 7-day average falls and the band
bounds printed at the 25% and 75% boundaries.

**The mockup positions the tick with `(shortAvg − hrvAvg) / hrvStdDev` and this
SOW does not.** Two things are wrong with it. The server's band is
`hrv_avg ± balanced_z × max(sd, min_std_dev_ms)`, so it uses the *floored*
deviation and a multiplier the client never receives; positioning by raw
`hrv_std_dev` silently assumes `balanced_z == 1` and no floor. For every ordinary
athlete the two agree, which is precisely what makes it a bad bug — it would
surface only for a user with a near-flat history, or the first time
`balanced_z` is retuned, as a tick sitting somewhere the printed bounds say it
should not be.

Derive the scale from the bounds themselves, in units of band half-widths:

```ts
/**
 * Position the 7-day average on a ±2-band-width gauge, in band half-widths.
 * Reading the half-width off the EMITTED bounds — rather than off hrv_std_dev —
 * makes the tick and the 25%/75% bound labels agree by construction, whatever
 * balanced_z and the SD floor happen to be. Null when there is no 7-day mean or
 * the band is degenerate.
 */
export function gaugeTickPct(
  shortAvg: number | null,
  hrvAvg: number,
  balancedHigh: number,
): number | null {
  const half = balancedHigh - hrvAvg;
  if (shortAvg === null || half <= 0) return null;
  const u = (shortAvg - hrvAvg) / half;
  return Math.min(100, Math.max(0, ((u + 2) / 4) * 100));
}
```

At `u === −1` this returns exactly 25%, where `balanced_low` is printed, and at
`u === +1` exactly 75%. The clamp keeps a wildly atypical week pinned to an end
of the bar rather than escaping it.

```tsx
<div className="py-2.5">
  <div className="relative">
    <div className="flex h-[8px] w-full gap-[2px] overflow-hidden">
      <div className="h-full w-1/4 rounded-l-[2px]" style={{ backgroundColor: "var(--warning)", opacity: 0.28 }} />
      <div className="h-full w-1/2" style={{ backgroundColor: "var(--success)", opacity: 0.28 }} />
      <div className="h-full w-1/4 rounded-r-[2px]" style={{ backgroundColor: "var(--accent)", opacity: 0.28 }} />
    </div>
    {tickPct !== null && (
      <span
        className="absolute top-[-3px] h-[14px] w-[2px] rounded-full bg-[var(--foreground)]"
        style={{ left: `${tickPct}%`, transform: "translateX(-1px)" }}
      />
    )}
  </div>
  <div className="relative mt-1.5 h-[11px]">
    <span className="absolute font-mono text-[9px] tabular-nums text-[var(--faint)]"
          style={{ left: "25%", transform: "translateX(-50%)" }}>
      {Math.round(hrv.balancedLow as number)}
    </span>
    <span className="absolute font-mono text-[9px] tabular-nums text-[var(--faint)]"
          style={{ left: "75%", transform: "translateX(-50%)" }}>
      {Math.round(hrv.balancedHigh as number)}
    </span>
  </div>
</div>
```

The zone colours follow the house contract, not Garmin's: the low quarter is
**warning** (a suppressed morning is information, never danger red), the high
quarter is **accent** (unusual, not "extra good"), the balanced half is
**success**, all at 0.28 alpha so the bar reads as territory rather than as a
second traffic light competing with the dot above it.

### Register 4 — the chart

Thirty-one nights as marks over a drifting band, drawn at measured pixel width so
`viewBox` equals rendered size and no circle is ever scaled into an ellipse.

```tsx
<div ref={ref} className="h-[62px] pt-2.5">
  {width > 0 && (
    <svg width={width} height={CHART_H} viewBox={`0 0 ${width} ${CHART_H}`} className="block"
         role="img" aria-label="Thirty-one nights of HRV against a baseline band that drifts">
      {bandRuns(days).map((run, k) => {
        const top = run.map(({ i, d }) => `${x(i)},${y(d.balancedHigh as number)}`);
        const bottom = run.slice().reverse().map(({ i, d }) => `${x(i)},${y(d.balancedLow as number)}`);
        return (
          <polygon key={k} points={[...top, ...bottom].join(" ")}
                   fill="var(--surface-3)" stroke="rgba(255,255,255,0.05)" strokeWidth={1} />
        );
      })}
      {days.map((d, i) =>
        d.hrv === null ? null : (
          <circle key={d.date} cx={x(i)} cy={y(d.hrv)}
                  r={i === days.length - 1 ? DOT_R + 1 : DOT_R}
                  fill={d.status === "balanced" ? "var(--muted)" : hrvStatusColor(d.status)}
                  fillOpacity={d.status === "unknown" ? 0.45 : 1}
                  stroke="var(--surface)" strokeWidth={1} />
        ),
      )}
    </svg>
  )}
</div>
```

Four things about this that are load-bearing:

- **Balanced marks are `--muted`, not `--success`.** Roughly two-thirds of
  nights are balanced; painting them green would make the ordinary month the
  loudest card on the grid. Colour is spent only where a night departs from
  normal. The band, likewise, is `--surface-3` — neutral territory, carrying no
  status.
- **`--surface-3` is the raised-2 neutral**, one step above the card's
  `--surface`. On a partial-band fixture the polygon visibly begins part-way
  across, which is the honest rendering of "your range wasn't established yet."
- **The wrapper reserves `h-[62px]` unconditionally.** The SVG renders only once
  measured, so without a reserved height the card would grow 62px on hydration —
  a layout shift on every dashboard load, on the tallest tile in the grid.
- **Marks are inset by their own radius** — see below.

**The inset.** The mockup uses `x(i) = (i / (n−1)) × width`, which puts the first
mark's centre at `x = 0` and the last at `x = width`, so both are drawn half
outside the SVG. On the shipped tile this is already visible: today's dot is
clipped against the right edge. Inset by the largest radius drawn, which is
today's:

```ts
const R_MAX = DOT_R + 1;
const x = (i: number) => R_MAX + (i / Math.max(1, days.length - 1)) * (width - R_MAX * 2);
```

The vertical scaler takes the same treatment — `scaler(domain, R_MAX, CHART_H − R_MAX * 2)` —
so a mark at the domain's extreme is not clipped against the top or bottom
either. The band polygon uses the same `x` and `y`, so it stays registered with
the marks.

At a one-third desktop cell (~260px of chart) with 31 marks the pitch is ~8.4px
against `DOT_R = 2.6` (5.2px diameter), leaving ~3px of gap — tight but
countable, which is what the 1px `--surface` ring on each mark protects. This is
the density the DX judged and selected; it is not to be "improved" by thinning
the series.

### The drift verdict, and a threshold worth naming

The DX comparison PR flagged a contradiction and it deserves a decision on the
record rather than a silent one in code.

`baseline_drift_z = 0.35` against this athlete's SD of ~20.1 ms requires a drift
of **> 7.0 ms** to read `rising`. The DX's headline fixture paired
`deltaMs: 6.4` with `direction: "rising"`, which the shipped engine would
classify `steady`. **The fixture was wrong; the threshold is not**, and this SOW
changes neither the engine nor the config.

The threshold is defensible on its own terms. The standard error of a 30-day mean
is `sd/√30 ≈ 0.183 × sd`, and the two baselines being differenced overlap by only
two days, so the drift estimate's own noise is about `0.26 × sd`. A `0.35 × sd`
threshold is therefore ~1.35 standard errors of the quantity being measured —
constant across athletes, by construction, which is exactly the property that
made an SD-relative threshold the right choice over a fixed millisecond count.
Loosening it to catch a 6.4 ms move on a 20 ms spread would be tuning the
statistic to one fixture.

What follows for the tile:

- **Print the magnitude whenever `delta_ms` is non-null, including when the
  direction is `steady`.** `▬ +6 ms · 4w` is a true and useful thing to show; a
  bare "steady" throws away the number the user came for. `driftTag` already
  behaves this way and must keep doing so.
- **`steady` and `unknown` are muted**, `rising` is success, `falling` is
  warning. A steady verdict should read as calm, not as an absence.
- **The chart is the fallback, and this is why the idiom survives the
  threshold.** Even when the verdict word is `steady`, the band polygon visibly
  slopes across the card. The user sees the drift whether or not the engine is
  willing to name it — which is the DX's own primary selection criterion ("can I
  *see* it, in the shape") satisfied independently of the classifier.
- The tile **never** substitutes its own judgement for the server's. No
  client-side re-thresholding, no "well, 6.4 is nearly 7."

Whether 0.35 is right in practice is now an empirical question with a live
answer, and the tile makes the raw delta visible so it can be judged from real
use. See Open Question 1.

### Shared helpers

**`recovery/shared.ts`** — additive, plus one signature change that is
source-compatible with all four existing callers:

```ts
/** The house word for a status. `unknown` splits on WHY it is unknown: with a
 *  reading it is calibrating, without one the morning webhook simply has not
 *  landed. Defaults to the pre-existing behaviour so the other recovery tiles
 *  are unaffected. */
export function statusWord(status: RecoveryHrvStatus, hasReading = true): string {
  switch (status) {
    case "suppressed": return "Suppressed";
    case "elevated":   return "Elevated";
    case "balanced":   return "Balanced";
    default:           return hasReading ? "Calibrating" : "No reading yet";
  }
}

/** The glyph for a baseline-drift direction. */
export function driftGlyph(direction: RecoveryTrendDirection): string { … }

/** The colour a DRIFT direction carries — about the RANGE moving, not about
 *  last night, so it is deliberately not hrvStatusColor. Steady and unknown
 *  stay muted so an ordinary month reads calm. */
export function driftColor(direction: RecoveryTrendDirection): string {
  switch (direction) {
    case "rising":  return "var(--success)";
    case "falling": return "var(--warning)";
    default:        return "var(--muted)";
  }
}

/** `▲ +6 ms · 4w`, or a shrug when the history is too short to say. */
export function driftTag(drift: RecoveryBaselineTrendView, unit = " ms"): string {
  if (drift.deltaMs === null || drift.direction === "unknown") return "drift not yet known";
  return `${driftGlyph(drift.direction)} ${signed(drift.deltaMs)}${unit} · ${Math.round(drift.overDays / 7)}w`;
}
```

`driftTag` reuses the file's existing `signed`; the mockup's private copy is not
carried over. `hrvStatusColor` and `trendLabel` are unchanged.

**`recovery/hrv-chart.ts`** — pure, no React, the chart's own machinery:
`prepareHrvChart(view)` (the single guard, returning `null` to mean *render the
calibrating state*), `bandRuns(days)` (consecutive runs of banded days, so a
polygon breaks at a null baseline rather than closing across it), `scaler`, and
`gaugeTickPct`. Ported from the mockup's `_util.ts` with the gauge correction
above and the mockup's `niceTicks` and `shortDate` dropped — they belonged to
`instrument-plot`, which was not selected.

**`lib/use-measured-width.ts`** — the hook, ported as written. Its doc comment
should keep the explanation of *why* it exists: the shipped tile's
`preserveAspectRatio="none"` turns every circle into an ellipse, and measuring
the container removes that at the root rather than working around it.

### States

Every one of these must render correctly and each gets a test:

- **Balanced + rising** — the headline case. Ordinary night, climbing range.
  Should be nearly monochrome: a muted dot row, a neutral band, and one small
  success-coloured drift tag.
- **Balanced + falling** — the case the shipped tile hides. Must be visibly
  different from rising, in the band's slope and in the tag's colour.
- **Suppressed** — warning dot, warning-coloured recent marks, tick in the lower
  quarter of the gauge. True and slightly concerning; never alarming.
- **Partial-band** — the oldest charted days carry `balancedLow: null` and
  `status: "unknown"`. The polygon starts part-way across, those marks render at
  0.45 opacity in muted, and nothing looks clipped.
- **Calibrating** (`hrvAvg` null) — `prepareHrvChart` returns null and the
  existing n-of-14 progress body renders. Preserved verbatim from the shipped
  tile; no chart frame around nothing.
- **No reading yet today** — `days[last].hrv` is null. The word reads *No
  reading yet*, the 28px figure still prints the 7-day average, the gauge still
  has its tick, and the chart simply has no final mark. **No em-dashes beyond
  the absent mark.**
- **Interior gap** — a null night mid-series is an absent mark; the band
  continues across it, because the band is defined by the baseline (which
  exists) and not by the reading (which does not).
- **Not connected** — the section is absent, `tile-renderer.tsx` renders the
  existing empty CTA. Unchanged; the rewrite must not break it.
- **Both breakpoints** — full-width single column on mobile (~330px of chart,
  ~10.6px pitch) and one-third on desktop (~260px, ~8.4px).

### Testing

**`vitest.setup.ts`** — jsdom implements neither `ResizeObserver` nor layout, so
a measured-width component currently renders no chart at all under test. Stub it
globally, which unblocks every future chart test too:

```ts
class ResizeObserverStub {
  observe() {}
  unobserve() {}
  disconnect() {}
}
vi.stubGlobal("ResizeObserver", ResizeObserverStub);

// jsdom reports every element as 0×0. Give elements a plausible content width so
// components that measure themselves render their charts under test.
Object.defineProperty(HTMLElement.prototype, "getBoundingClientRect", {
  configurable: true,
  value() {
    return { width: 260, height: 62, top: 0, left: 0, bottom: 62, right: 260, x: 0, y: 0, toJSON() {} };
  },
});
```

Run the full web suite after this change and before touching the tile — a global
DOM stub can perturb unrelated snapshot or layout-adjacent tests, and finding
that out separately from the redesign is worth one extra commit.

**`recovery/hrv-chart.test.ts`** — the pure machinery, where most of the value
is:

- `gaugeTickPct` returns exactly `25` when `shortAvg === balancedLow` and exactly
  `75` when `shortAvg === balancedHigh`, **for a band whose half-width is not the
  raw SD** (e.g. `hrvAvg: 88`, `balancedHigh: 108`, `hrvStdDev: 5`). This is the
  regression test for the correction and it must fail against the mockup's
  formula.
- `gaugeTickPct` clamps to `0`/`100` beyond ±2 half-widths, and returns null for
  a null `shortAvg` or a degenerate band (`balancedHigh === hrvAvg`).
- `bandRuns` splits at a null-baseline day, returns one run for a fully banded
  series, `[]` for a wholly unbanded one, and preserves original indices — the
  indices are what keep the polygon registered with the marks, so assert them
  explicitly.
- `prepareHrvChart` returns null for a missing `days`/`baseline`/`hrv`/
  `baselineTrend`, and null when `hrvAvg` is null; returns a populated object
  with `today` equal to the last day for a full fixture.
- The domain spans every mark and every band bound with headroom, including the
  case where a mark sits outside the band on both ends.

**`recovery/balance-band.test.tsx`** — one test per state above. Specifically:

- The rising fixture renders `▲`, `+6`, and `4w` in the drift tag; the falling
  fixture renders `▼` and a `−` delta. Assert on rendered text, not colour.
- **A `steady` fixture with `deltaMs: 6.4` still renders the magnitude** — the
  card shows `+6`, not a bare "steady". This is the behaviour the threshold
  discussion above turns on and it should be pinned so nobody "simplifies" it
  away.
- The no-reading-yet fixture renders *No reading yet* **and** the 7-day figure,
  and the card contains no `—` other than in an explicitly asserted position.
- The partial-band fixture renders fewer `<polygon>` elements than a fully
  banded one, and renders marks for the unbanded days.
- The suppressed fixture's status dot and today's mark use the same colour
  token — pinning the "the dot cannot disagree with the last mark" property.
- Mark count equals the number of non-null `hrv` days, and a fixture with an
  interior gap renders one fewer mark than a gapless one while its band polygon
  count is unchanged.
- `Math.round` on the headline: a `shortAvg` of `84.9143` renders `85` and the
  raw string is absent. Carries forward the guarantee SOW 1 established for the
  old headline.
- The calibrating fixture renders the n-of-14 progress body and **no `<svg>`**.

**`recovery/shared.test.ts`** — `statusWord("unknown", false)` is *No reading
yet* and `statusWord("unknown")` is still *Calibrating*, proving the default
keeps the other tiles' behaviour. `driftTag` formats a rising, a falling, a
steady, and an unknown drift; `driftColor` maps steady and unknown to muted.

**Regression** — `dashboard/page.test.tsx` and `tile-renderer.test.tsx` must pass
unmodified. If either needs editing, the catalog or the empty state has been
disturbed and that is out of scope.

### Design system

`scope: in-system`, inherited from the DX. Every colour is a v0.4 token —
`--foreground`, `--muted`, `--faint`, `--surface`, `--surface-3`, `--border`,
`--success`, `--warning`, `--accent` — and no raw hex appears in the component.
Type is Manrope with `tabular-nums` on every figure and `font-mono` on the drift
tag and the gauge labels, matching the tile's neighbours. The panel, radius, and
padding are `MiniCard`'s and are not overridden.

Two conformance notes worth stating because they are where this could go wrong:
**Garmin's saturated red/orange/green is not imported** — the gauge and the marks
use the desaturated statuses, and `suppressed` is warning rather than danger.
And **recovery still has no `--discipline-*` hue**; none is invented here.

No new tokens. No `design-system.md` change.

### Documentation

- Rewrite the component's file-header comment. The current one describes the
  band-behind-a-polyline design and would otherwise become the most misleading
  text in the file. It should say what the four registers are, why the big
  figure is the 7-day average and not last night, and why balanced marks are
  muted.
- `hrv-chart.ts` and `lib/use-measured-width.ts` each get a header explaining
  their reason for existing — for the hook, the ellipse problem specifically.
- Set [`dx/hrv-balance-tile.md`](../dx/hrv-balance-tile.md) to
  `status: selected` with `selected_idioms: [garmin-status-stack]`, and add the
  selection note at the top pointing at PR #157 and at this SOW, following the
  pattern in [`dx/recovery-tile.md`](../dx/recovery-tile.md).

## Open Questions

1. **Is `baseline_drift_z = 0.35` right in practice?** As shipped, an athlete
   with a 20 ms spread needs a 7 ms four-week move to see anything but `steady`,
   so the verdict word may be `steady` most of the year while the band visibly
   slopes. Options: leave it and judge from real use; lower it to ~0.25 (a 5 ms
   move at this spread); drop the word entirely and show only the delta.
   **Tentative lean: leave it, and revisit after a month of real data.** The
   threshold is ~1.35 standard errors of the drift estimate, which is a
   principled place to sit, and this tile is the instrument that will answer the
   question — it prints the raw delta regardless of the verdict, so the gap
   between "what the number did" and "what the engine called it" is visible on
   the dashboard every day. Retuning before that evidence exists would be fitting
   the statistic to one fixture.

2. **Should the gauge tick show today rather than the 7-day average?** Garmin
   marks the stable figure, which is what this implements, and register 1's dot
   already carries today. Options: keep the 7-day tick; add a second fainter tick
   for today; switch to today. **Tentative lean: keep it.** Two ticks on an 8px
   bar at a one-third cell is more density than the bar can carry, and the
   variant's whole argument is that the stable figure is the honest one. Worth
   revisiting only if the gauge reads as disconnected from the dot above it in
   real use.

3. **Should this chart be promoted to `/recovery`?** The deep page still renders
   the older treatment, so the tile would preview something the page does not
   show — the inverse of the "does it preview the page it links into" criterion
   the DX applied. Options: a follow-up SOW; fold into this one; leave the page
   alone indefinitely. **Tentative lean: a follow-up SOW.** `hrv-chart.ts` is
   deliberately factored so the page can import it unchanged, and the page has
   its own layout questions (a full-width chart can afford axes, a range
   selector, and the mark sizes `dual-window` wanted) that deserve their own
   exploration rather than being decided as a footnote here.
