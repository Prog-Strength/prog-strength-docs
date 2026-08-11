# Implementation Plan — HRV Balance Tile Redesign

Derived from [`sows/hrv-balance-tile-redesign.md`](../sows/hrv-balance-tile-redesign.md).
Single repo: `prog-strength-web`. Branch `feat/hrv-balance-tile-redesign`.

The SOW builds the `garmin-status-stack` idiom selected in
[`dx/hrv-balance-tile.md`](../dx/hrv-balance-tile.md) for real: `HrvBalanceCard`'s
body becomes four hairline-separated registers over a chart of discrete nightly
marks and a band polygon that drifts, drawn at measured pixel width.

## Ordering rationale

Bottom-up, so every later task lands on a tested foundation and the risky global
change is isolated in its own commit:

1. The `vitest.setup.ts` DOM stubs land **first and alone** — a global
   `getBoundingClientRect` override can perturb unrelated layout-adjacent tests,
   and the SOW explicitly asks that this be found out separately from the redesign.
2. `lib/use-measured-width.ts` — the hook the stubs exist to make testable.
3. `recovery/shared.ts` — the drift formatters and the `statusWord` signature
   change, source-compatible with all four existing callers.
4. `recovery/hrv-chart.ts` — the pure chart machinery, including the gauge-scale
   correction. Most of the value is here and none of it needs React.
5. `recovery/fixtures.ts` — drifting-band views for the states the tile must render.
6. `recovery/balance-band.tsx` — the tile rewrite, which is the only task that can
   fail visually and lands last, on top of everything it needs.

## Task 1 — `vitest.setup.ts`: DOM stubs for measured-width components

**Files**: `vitest.setup.ts`

jsdom implements neither `ResizeObserver` nor layout, so a component that measures
itself renders no chart at all under test. Add both stubs globally per the SOW:

- `vi.stubGlobal("ResizeObserver", ResizeObserverStub)` — `observe` / `unobserve` /
  `disconnect` no-ops.
- `HTMLElement.prototype.getBoundingClientRect` returning a plausible
  260×62 rect (`configurable: true`).

**Verify**: `npm run test` over the whole suite. Any test that goes red here is a
pre-existing dependence on jsdom's all-zero rects and must be understood, not
papered over. Report the delta from the 171-files / 1633-tests baseline.

**Commit**: `test(dashboard): stub ResizeObserver and layout for measured-width components`

## Task 2 — `lib/use-measured-width.ts`

**Files**: `lib/use-measured-width.ts` (new), `lib/use-measured-width.test.tsx` (new)

Port the hook from the DX mockup's `_util.ts` as written: a `RefCallback` that
measures on attach plus a `ResizeObserver` in `useLayoutEffect` that updates on
resize, ignoring sub-0.5px jitter. Returns `[ref, width]`, width `0` until measured.

The header comment must keep the *why*: the shipped tile's
`preserveAspectRatio="none"` scales every `<circle>` into an ellipse, and
`vectorEffect` rescues a stroke but nothing rescues a scaled circle. Measuring the
container so `viewBox` equals rendered size removes the problem at the root.

**Tests**: width is 0 before attach; becomes the measured width once a node
attaches; `disconnect` is called on unmount; the hook is safe when
`ResizeObserver` is undefined.

**Commit**: `feat(lib): add useMeasuredWidth for charts drawn in real pixel units`

## Task 3 — `recovery/shared.ts`: the drift formatters

**Files**: `app/(app)/dashboard/_components/recovery/shared.ts`,
`app/(app)/dashboard/_components/recovery/shared.test.ts`

Additive, plus one signature change that is source-compatible with all four
existing callers:

- `statusWord(status, hasReading = true)` — `unknown` splits on its cause:
  *Calibrating* with a reading, *No reading yet* without one. The default keeps
  the other recovery tiles' behaviour byte-for-byte.
- `driftGlyph(direction)` — `▲ ▼ ▬ ·`.
- `driftColor(direction)` — rising `--success`, falling `--warning`, steady and
  unknown `--muted`. Deliberately **not** `hrvStatusColor`: this is about the
  range moving, not about last night, and a steady month should read calm.
- `driftTag(drift, unit = " ms")` — `▲ +6 ms · 4w`, or `drift not yet known` when
  `deltaMs` is null or the direction is `unknown`. Reuses the file's existing
  `signed`; the mockup's private copy is not carried over.

**Tests**: `statusWord("unknown", false)` is *No reading yet* and
`statusWord("unknown")` is still *Calibrating*; `driftTag` formats rising,
falling, steady and unknown; `driftColor` maps steady and unknown to muted.
Existing `shared.test.ts` assertions must pass unmodified.

**Commit**: `feat(dashboard): add baseline-drift formatters to the recovery tiles`

## Task 4 — `recovery/hrv-chart.ts`: the pure chart machinery

**Files**: `app/(app)/dashboard/_components/recovery/hrv-chart.ts` (new),
`app/(app)/dashboard/_components/recovery/hrv-chart.test.ts` (new)

Pure, no React. Ported from the mockup's `_util.ts` with the gauge correction;
the mockup's `niceTicks` and `shortDate` are dropped — they belonged to
`instrument-plot`, which was not selected.

- **`prepareHrvChart(view): HrvChart | null`** — the single guard. Returns `null`
  (meaning *render the calibrating state*) when `days` / `baseline` / `hrv` /
  `baselineTrend` is missing, when `days` is empty, when `baseline.hrvAvg` is
  null, or when either scalar band bound is null. The returned object carries the
  **narrowed** figures (`hrvAvg`, `balancedLow`, `balancedHigh` as `number`) so
  the component needs no `as number` cast and no `!`-assert, plus `days`,
  `shortAvg`, `drift`, `today` (the last charted day) and `domain`.
- **`bandRuns(days)`** — consecutive runs of days that have a band, as
  `{ i, d }[][]`, **preserving original indices** so a polygon stays registered
  with the marks. A polygon breaks at a null baseline rather than closing across it.
- **`scaler(domain, top, height)`** — linear ms→pixel for a top-down SVG box.
- **`gaugeTickPct(shortAvg, hrvAvg, balancedHigh)`** — the correction. Positions
  the 7-day average on a ±2-band-width gauge in **band half-widths read off the
  emitted bounds**, not off `hrv_std_dev`, so the tick and the 25%/75% bound
  labels agree by construction whatever `balanced_z` and the SD floor are.
  Clamped to 0–100; null for a null `shortAvg` or a degenerate band.

`domain` spans every non-null mark, every per-day band bound, and the scalar
bounds, with ±5 ms headroom.

**Tests** (this is where most of the value is):

- `gaugeTickPct` returns exactly `25` at `shortAvg === balancedLow` and exactly
  `75` at `shortAvg === balancedHigh` **for a band whose half-width is not the raw
  SD** (`hrvAvg: 88`, `balancedHigh: 108`, `hrvStdDev: 5`). This is the regression
  test for the correction and must fail against the mockup's formula.
- `gaugeTickPct` clamps to `0` / `100` beyond ±2 half-widths; null for a null
  `shortAvg` and for a degenerate band (`balancedHigh === hrvAvg`).
- `bandRuns` splits at a null-baseline day, returns one run for a fully banded
  series, `[]` for a wholly unbanded one, and preserves original indices —
  asserted explicitly.
- `prepareHrvChart` returns null for each missing block and for a null `hrvAvg`;
  returns a populated object whose `today` is the last day for a full fixture.
- The domain spans every mark and every band bound with headroom, including a
  mark outside the band at both ends.

**Commit**: `feat(dashboard): add pure hrv chart machinery with a bounds-derived gauge scale`

## Task 5 — `recovery/fixtures.ts`: drifting-band views

**Files**: `app/(app)/dashboard/_components/recovery/fixtures.ts`,
`app/(app)/dashboard/_components/recovery/fixtures.test.ts`

The existing four views deliberately leave interior days' band fields null —
correct when written, because a day's own trailing band could not be known
without recomputing 30-day means. The redesign needs a band that *moves*, and the
DX's instruction for exactly this case is to **hand-author** the numbers so the
band drifts the way the fixture claims. Strictly additive; the four existing views
keep their day series and gain only `baselineTrend`.

Add one generator and the views it builds:

- `driftingDays({ hrv, fromAvg, toAvg, halfWidth, bandFrom })` — a 31-day window
  whose baseline walks linearly from `fromAvg` to `toAvg`, each day carrying the
  band as it stood that morning (`baselineAvg ± halfWidth`) and a `zScore` /
  `status` classified against **its own** band. Days before `bandFrom` carry a
  null band and `status: "unknown"`. A null HRV keeps the band and drops the z and
  the status, mirroring the engine's contract for a missing morning.
- `risingView()` — baseline 84.8 → 91.2 (`deltaMs: 6.4`, `direction: "rising"`,
  `overDays: 28`), SD 12.6, today 94 ms and balanced. The headline case. Rising is
  legitimate here: `0.35 × 12.6 = 4.4 ms` and the move is 6.4.
- `fallingView()` — baseline 99.3 → 91.2 (`deltaMs: −8.1`, `direction: "falling"`),
  today 94 ms still balanced against the *lowered* band.
- `steadyDriftView()` — the threshold fixture the SOW's discussion turns on: SD
  20.1, `deltaMs: 6.4`, `direction: "steady"`, because `0.35 × 20.1 = 7.0 > 6.4`.
- `partialBandView()` — the five oldest charted days carry a null band and
  `status: "unknown"`; the rest are populated.
- `bandGapView()` — an interior day with a null baseline, so the polygon must break.
- `suppressedDriftView()` — a steady baseline with a suppressed morning (today
  74 ms) and a `shortAvg` of 77.0, which puts the gauge tick in the lower quarter.

Every view derives its `hrv` block from the generated last day, so the
`fixtures.test.ts` agreement invariant holds by construction rather than by two
hand-copied number sets. Extend that test's `VIEWS` table with the new views.

**Commit**: `test(dashboard): add drifting-band recovery fixtures`

## Task 6 — `recovery/balance-band.tsx`: the tile

**Files**: `app/(app)/dashboard/_components/recovery/balance-band.tsx`,
`app/(app)/dashboard/_components/recovery/balance-band.test.tsx`

Keep the `MiniCard` shell, the title, the `href`, the catalog entry, and the
`Calibrating` progress body **verbatim**. Replace the body with four
hairline-separated registers, per the SOW's markup:

1. **Verdict** — status dot + `statusWord(today.status, today.hrv !== null)` on the
   left, `driftTag(drift)` in `driftColor(drift.direction)` on the right. The status
   is read off `days[last]`, **not** `hrv.status`, so the dot can never disagree
   with the last mark.
2. **Stable figure** — `Math.round(shortAvg)` at 28px with a `7D AVG` caption; an
   em-dash only when `shortAvg` is null.
3. **Gauge** — warning quarter / success half / accent quarter at 0.28 alpha, a
   tick at `gaugeTickPct(shortAvg, hrvAvg, balancedHigh)`, the bounds printed at
   25% and 75%.
4. **Chart** — `h-[62px]` reserved unconditionally so hydration causes no layout
   shift; the SVG renders only once measured, with `width === viewBox` so no
   `<circle>` is ever scaled. `CHART_H = 62`, `DOT_R = 2.6`, `R_MAX = DOT_R + 1`.
   Both axes are inset by `R_MAX` — `x(i) = R_MAX + (i / (n−1)) × (width − 2·R_MAX)`
   and `scaler(domain, R_MAX, CHART_H − 2·R_MAX)` — so neither today's mark nor a
   domain-extreme mark is drawn half outside the box. Band polygons use the same
   `x` and `y` and stay registered with the marks. Balanced marks are `--muted`,
   not `--success`; the band is `--surface-3` with a `--border` hairline.

A run of a single banded day has no area, so the component renders a polygon only
for runs of two or more days; `bandRuns` itself stays index-preserving and total.

Rewrite the file-header comment: the four registers, why the big figure is the
7-day average and not last night, and why balanced marks are muted.

**Tests** — one per state:

- rising renders `▲`, `+6`, `4w`; falling renders `▼` and a `−` delta (asserted
  on rendered text, not colour).
- a `steady` fixture with `deltaMs: 6.4` **still renders the magnitude** (`+6`,
  not a bare "steady").
- no-reading-yet renders *No reading yet* **and** the 7-day figure, with the gauge
  tick still present and no `—` anywhere in the card.
- suppressed: the status dot and today's mark carry the same colour token.
- partial-band: one polygon that visibly starts part-way across (its leftmost x is
  well right of the inset), and marks still render for the unbanded days.
- band gap: an interior null baseline splits the band into two polygons where the
  fully banded fixture renders one.
- mark count equals the number of non-null `hrv` days; an interior HRV gap renders
  one fewer mark while the polygon count is unchanged.
- `shortAvg` of `84.9143` renders `85` and the raw string is absent.
- calibrating renders the n-of-14 progress body and **no `<svg>`**.

**Regression**: `dashboard/page.test.tsx` and `tile-renderer.test.tsx` must pass
unmodified. If either needs editing, the catalog or the empty state has been
disturbed and that is out of scope.

**Commit**: `feat(dashboard): rebuild the hrv balance tile as a garmin status stack`

## Known deviation from the SOW's test list

The SOW asks that "the partial-band fixture renders fewer `<polygon>` elements
than a fully banded one". A band that is merely absent at the *start* of the
series is still one unbroken run, so both fixtures render exactly one polygon and
the assertion cannot hold as written. The property it is reaching for is split in
two and both halves are pinned: the partial-band polygon **starts part-way
across** (asserted on its leftmost x), and a **band gap in the interior** splits
the polygon in two. This is strictly stronger than a count comparison that would
have passed vacuously.

## Out of scope (restating the SOW's non-goals)

No catalog, tray, `TileId`, `href`, or grid change. No change to the other four
recovery tiles. No `/recovery` page change. No API change. No range switcher. No
config change. No `baseline_drift_z` retune. No promotion of `app/design-explore/**`.
No new design-system token and no `design-system.md` edit.

## Gate before pushing

`npm run lint` → `npm run format:check` → `npm run typecheck` → `npm run test` →
`npm run build`, all green locally, plus commitlint on every commit subject. The
Husky `pre-commit` and `commit-msg` hooks stay armed; never `--no-verify`.
