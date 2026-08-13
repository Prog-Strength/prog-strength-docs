# Recovery Page Refresh — Implementation Plan

SOW: [`sows/recovery-page-refresh.md`](../sows/recovery-page-refresh.md)
DX: [`dx/recovery-page-refresh.md`](../dx/recovery-page-refresh.md) — `aligned-deck`, selected
Repos: `prog-strength-web`, `prog-strength-docs`
Branch: `feat/recovery-page-refresh`

## Shape of the work

`app/(app)/recovery/page.tsx` is rebuilt as the `aligned-deck` composition. One
fetch of `GET /recovery/history` (shipped by
[`sows/recovery-history-endpoint.md`](../sows/recovery-history-endpoint.md),
`status: shipped`) plus the existing `GET /me/whoop/connection` feeds five
registers: header + window pills, the window strip, the three-row deck bound to
one x-domain and one crosshair, the ledger paginated over all stored history,
and the settings backlink.

Everything lands in `prog-strength-web`. `prog-strength-docs` gets the status
flip only. No API change: the endpoint already serves the exact `RecoverySection`
shape `adaptRecovery` speaks.

The DX mockup on the closed `dx/recovery-page-refresh` branch
(`app/design-explore/recovery-page-refresh/**`) is the transcription source for
geometry and copy, and stays where it is — nothing is promoted from that branch.
The five corrections in the SOW are the substance of this work beyond
transcription.

## Decisions taken here (the SOW left them open or under-specified)

1. **Open Question 1 — window control.** Ship the three pills `30d · 90d · 1y`.
   The SOW's lean, and the Proposed Solution already states three.
2. **Open Question 2 — crosshair persistence.** Implement the lean: **pin on
   pointer-down, clear on `Escape` or a second pointer-down on the same
   morning.** This is not decoration — with pointer events, a touch drag ends in
   `pointerleave`, so an unpinned crosshair would clear the moment a finger
   lifts and the ledger's selected row could never be read on a phone. Keyboard
   selection pins for the same reason.
3. **Open Questions 3 & 4** — keep the client-computed window means labelled
   `window avg` (correction 2); no rank strip. Both the SOW's leans.
4. **`prepareHrvChart` is called ONCE per render, in the deck**, and the chart
   object is passed down. `hrv-panel.tsx` takes a prepared `HrvChart`, not a
   `RecoveryView` — the mockup's panel guarded internally, which would mean two
   calls (one for the panel, one for the deck's shared date list) and therefore
   two chances to disagree about the axis correction 1 exists to fix. The deck
   owns the guard and renders `HrvCalibrating` itself, exactly as `hrv-tile.tsx`
   owns its single guard.
5. **The deck's shared day list IS `chart.series`**, not a re-slice by date.
   `series` is a tail slice of the days handed to `prepareHrvChart`, so the same
   `RecoveryDayPoint` objects carry `recoveryScore` and `restingHr` for the other
   two rows. Equality of the three plotted date lists then holds *by
   construction* rather than by a lookup that could drift.
6. **Mobile is driven by `matchMedia`, not by a measured width.** The house
   pattern (`components/date-tile-strip.tsx`, `app/(app)/bodyweight/page.tsx`)
   is a `useSyncExternalStore` subscription to `(max-width: 639px)` — Tailwind's
   `sm:` breakpoint — so the JS-chosen plot heights and the CSS-chosen grid
   collapse switch on the same pixel. Both existing copies carry a comment
   saying a third consumer should extract a shared hook; this is that consumer,
   so the plumbing moves to `lib/use-is-mobile.ts`. The two pre-existing inline
   copies are **left alone** — folding them in is an unrelated diff and this SOW
   is scoped to `/recovery`.
7. **`lib/recovery.ts` is not deleted.** Deleting the three old components leaves
   it with no production consumer, but the recovery tiles' `shared.ts` already
   records that reconciling the two `recoveryBand` vocabularies is "the owner's
   call which module wins". Removing it here would pre-empt that decision inside
   a page PR. Noted in the PR body.
8. **`getRecoveryHistory` returns `DashboardRecovery | null`**, and the page
   adapts it. `lib/api.ts` returns wire types; adapting is `lib/dashboard.ts`'s
   job, and `adaptRecovery` becoming exported is the SOW's one-word change.
9. **The "zero rows" gate is "no recorded mornings"**, not `days.length === 0`.
   A connected user with no history gets a window of materialized null slots
   from the endpoint, not an empty array, so gating on the array length would
   render an empty deck instead of `FirstNightState`.

## Shared context every task needs

- **House rule, standing**: nothing recomputes a server figure. The page's only
  arithmetic is (a) means over the *rendered window*, labelled `window avg`,
  (b) ranking a score into Whoop's fixed thirds, (c) `restingHr − baseline.restingHrAvg`,
  and (d) value→pixel mapping.
- **Tokens** (all existing, `app/globals.css`): `--surface`, `--surface-2`,
  `--surface-3`, `--border`, `--border-strong`, `--foreground`, `--muted`,
  `--faint`, `--accent`, `--accent-fg`, `--accent-soft`, `--accent-line`,
  `--success`, `--warning`, `--danger`, `--radius-card`, `--radius-pill`. No new
  tokens, no raw hex.
- **Type scale**: 10px uppercase labels, 11–12px ledger figures, 16px crosshair
  readouts, 24px window figures, 28px only inside the HRV panel's optional
  figure register. Nothing larger. No hero number.
- **Colour budget**: band colour in the score columns and the ledger pill only;
  resting HR monochrome plus `--warning` above the average; `--accent` only on
  the crosshair rule, the active window pill, the selected ledger row, the jump
  link and the backlink.
- **Rhythm**: deck rows `py-3` separated by hairlines with no gaps, page `gap-4`,
  ledger rows a strict 30px.
- **Imported, never restated**: `prepareHrvChart`, `bandRuns`, `rollingRuns`,
  `weekBand`, `scaler`, `gaugePct`, `ROLLING_WINDOW_NIGHTS` from
  `app/(app)/dashboard/_components/recovery/hrv-chart`; `recoveryBand`,
  `recoveryBandColor`, `recoveryBandWord`, `hrvStatusColor`, `statusWord`,
  `driftTag`, `driftColor`, `round`, `MIN_BASELINE_DAYS` from
  `app/(app)/dashboard/_components/recovery/shared`. Neither file is refactored;
  every existing recovery test must pass unmodified.
- **Every component carries a header comment in the house voice** — read
  `morning-ledger.tsx` or `balance-band.tsx` for the register.

## File layout when this lands

```
app/(app)/recovery/
  page.tsx                      # fetch, gates, window state, crosshair state
  page.test.tsx
  fixtures.ts                   # test + state fixtures (the DX's own, unkind)
  _components/
    shared.ts                   # dates, windowing, paging, scales, correction 1
    shared.test.ts
    window-strip.tsx            # register 2
    window-strip.test.tsx
    deck.tsx                    # register 3 — rows, crosshair, readouts
    deck.test.tsx
    hrv-panel.tsx               # the shipped chart at page scale
    hrv-panel.test.tsx
    score-plot.tsx              # banded columns
    score-plot.test.tsx
    rhr-plot.tsx                # diverging bars
    rhr-plot.test.tsx
    ledger.tsx                  # register 4 — paginated, all history
    ledger.test.tsx
lib/use-is-mobile.ts            # extracted matchMedia plumbing
```

Deleted: `_components/recovery-hero.tsx`, `recovery-trends.tsx`,
`recovery-log.tsx` and their three test files.

---

## Task 1 — the read path

**Files**: `lib/api.ts`, `lib/dashboard.ts`, `lib/api.test.ts`

1. In `lib/api.ts`, beside `listWhoopRecovery`, add:

```ts
export async function getRecoveryHistory(
  token: string,
  opts: { timezone: string; since?: string; until?: string },
): Promise<DashboardRecovery | null>;
```

   `GET {config.apiUrl}/recovery/history?timezone=…[&since=…][&until=…]` with
   the bearer header, unwrapped as `{ recovery: DashboardRecovery | null }` with
   `{ recovery: null }` as the empty fallback, returning `body.recovery ?? null`.
   Doc comment: same shape as `/dashboard/summary`'s `sections.recovery`, over
   the caller's local-date window; `timezone` is **required** here (unlike
   `/whoop/recovery`, which accepts and ignores it) because the endpoint
   materializes dates; omitting `since` asks for all stored history and the
   server clamps. `listWhoopRecovery` is left exactly as it is.

2. In `lib/dashboard.ts`, add `export` to `function adaptRecovery`. Nothing else
   changes; extend its doc comment with one sentence naming `/recovery/history`
   as the second consumer and why a second adapter would be a second place for
   the band fields to be mis-mapped.

3. `lib/api.test.ts`: a test for the happy path (URL, params, bearer header,
   unwrapping `data.recovery`), one for a payload with no `recovery` key → null,
   one for a non-2xx → throws with the server's `error` text.

**Done when** `npm run typecheck` and the two touched test files pass.

## Task 2 — `_components/shared.ts`, and the correction-1 contract

**Files**: `app/(app)/recovery/_components/shared.ts`, `shared.test.ts`,
`lib/use-is-mobile.ts`

The page's own module. It **imports from** the recovery tiles' `shared.ts`; it
never re-exports it.

Exports:

- `LEDGER_PAGE_SIZE = 20` — the bodyweight readings table's precedent, named in
  the comment.
- Dates, parsed as local parts so `new Date(iso)`'s UTC shift never bites:
  `parseISO`, `toISO`, `addDays`, `longDate` (`"Wed 12 Aug"`), `shortDate`
  (`"12 Aug"`), `monthLabel`.
- `isMissingNight(d)` — all three metrics null.
- `ledgerRows(days)` — newest first, missing nights dropped. Comment: the charts
  keep the date-aligned slot because that is what makes the absence visible; the
  record has nothing to record.
- `pageCount(total, per?)`, `pageOf(rows, page, per?)`.
- `tail(days, n)`, `windowView(view, days)`, `mean(values)`.
- `bandCounts(days)` → `{ green, yellow, red }`, delegating the thresholds to
  `recoveryBand` from the tiles' `shared.ts` — the thirds are defined once in
  the codebase.
- `PLOT_INSET = 4.6` **with the comment the SOW requires**: it is the HRV
  chart's own `R_MAX` (the radius of the largest mark it draws), and the columns
  are centred on the HRV chart's positions rather than on their own slots
  because a deck whose whole argument is one axis cannot have three x-mappings.
  It looks arbitrary and is not.
- `xAt(i, n, width)` — the one x-mapping all three plots share.
- `WINDOWS`: `30d`/`90d`/`1y` → `{ key, days, label }`, and `WindowKey`.
- `LEAD_IN = ROLLING_WINDOW_NIGHTS - 1` — imported, never the literal 6.
- `prepareDeck(view, windowLength): { chart: HrvChart | null; days: RecoveryDayPoint[] }`
  — **the load-bearing function.**

```ts
// The HRV chart consumes ROLLING_WINDOW_NIGHTS - 1 mornings before it can plot
// its first point, so hand it that many EXTRA mornings and its `series` comes
// out equal to the visible window — one point per column, by construction.
const hrvView = windowView(view, tail(view.days ?? [], windowLength + LEAD_IN));
const chart = prepareHrvChart(hrvView);
```

  Then: when `chart` is non-null **and `chart.series` is non-empty**, the deck's
  days are `chart.series` itself (the same day objects, so the three rows cannot
  drift); otherwise — calibrating, or a series so sparse no rolling mean forms —
  the days are `tail(days, windowLength)` and the HRV row renders its own state.
  The header comment states correction 1 in full: the lead-in exists to make the
  axis true, and a future reader trimming it would silently re-introduce the
  six-day skew. Short history is covered by the same branch: a user with 40
  stored mornings asking for 90 days gets a shorter `series`, and all three rows
  narrow together rather than one narrowing alone.

`lib/use-is-mobile.ts`: the `useSyncExternalStore` plumbing lifted verbatim from
`components/date-tile-strip.tsx` (module-scope `MOBILE_QUERY`, `subscribe`,
`getSnapshot`, `getServerSnapshot` returning `false`), exported as
`useIsMobile()`. Header comment records that the two pre-existing inline copies
stay put and folding them in is a follow-up.

**Tests** (`shared.test.ts`), against fixtures from Task 3 where a view is
needed — this task and Task 3 land together:

- `longDate`/`shortDate` do not shift across a timezone boundary.
- `ledgerRows` drops a wholly-missing morning, keeps a partial-null one, and is
  newest-first.
- `pageOf`/`pageCount` over 425 rows → 22 pages, page 22 holds the remainder.
- `bandCounts` matches a hand-counted fixture window.
- **The correction-1 contract**: for each of the three windows,
  `chart.series.length === deckDays.length`, and `deckDays[0].date` /
  `deckDays.at(-1).date` equal `chart.series` at both ends.
- With history shorter than `window + LEAD_IN`, `deckDays` narrows to
  `chart.series` and is still one contiguous tail.
- With a calibrating view (`prepareHrvChart` → null), `chart` is null and
  `deckDays` spans the requested window.

## Task 3 — `fixtures.ts`

**File**: `app/(app)/recovery/fixtures.ts`

Port the DX's generator (`app/design-explore/recovery-page-refresh/_fixtures.ts`
on the closed `dx/recovery-page-refresh` branch) into the page's own module,
importing `addDays` / `isMissingNight` / `mean` from `_components/shared.ts`
instead of the DX's `_shared`. Test-only; never imported by production code —
say so in the header, as the recovery tiles' `fixtures.ts` does.

The fixture stays deliberately unkind, and these values are asserted by later
tasks' tests, so they are contract:

- Today is **Wed 12 Aug 2026 — score 13 · 69 bpm · 45 ms**.
- Tue 11 Aug `78 · 50 · 106`; Mon 10 Aug `52 · 47 · 96`; Sun 9 Aug null score.
- **No row at all for Fri 7 Aug**, plus longer strap-off stretches in Nov, Mar.
- Three score-but-no-HRV mornings.
- **Five consecutive below-band HRV nights from 21 Jul.**
- A rough week in February and a green stretch in June, so band colour has both
  directions to show.
- **428 days of history** — the ledger really pages, and a 1y window is full.
- The generator plays the server: per-day bands, trailing baselines, spread,
  z-scores, the 7-day mean and the baseline drift are computed **once, here**,
  and read as received by every component. That is what keeps the standing rule
  true of a fixture with no server behind it.

Exported states, following the DX: `default`, `noReadingYet` (today's slot
present and empty), `calibrating` (nine mornings), `sparseYear` (a year of
alignment with only the last 34 mornings recorded). Plus `recordedMornings(view)`.

A `fixtures.test.ts` is **not** required — but the fixture's headline claims
(today's triple, the missing Friday, the suppressed run's length, the history
depth) are asserted in `shared.test.ts` so a future edit to the generator cannot
silently invalidate every other test's premise.

## Task 4 — `hrv-panel.tsx`

**Files**: `_components/hrv-panel.tsx`, `hrv-panel.test.tsx`

`HrvPanel({ chart, height })` — takes a **prepared `HrvChart`** (decision 4) and
draws the shipped tile's grammar at page scale: the `bandRuns` × `weekBand`
ribbon, the `rollingRuns` curve, the circle / triangle / diamond `Mark`s from
`rolling`, `hrvStatusColor` for status, the verdict register (`statusWord` on
`chart.today.status`, `driftTag` + `driftColor` right-aligned), and a **dated
x-axis**: month ticks read off the drawn series (`date.slice(8) === "01"`) plus
the window's two ends via `shortDate`.

Also exports `HrvCalibrating({ nights })` — the tile's honest progress state at
page width.

The two invariants the SOW requires, **as comments in the file**:

1. The window must end on the last stored morning, because `rolling`'s final
   entry is overwritten with the server's `hrv.shortAvg`; a window ending
   earlier would terminate the curve on a figure describing a different day. All
   three page windows are tails ending today. A future "brush a past stretch"
   feature would break this quietly.
2. The verdict register describes **last night, not the window** — so it is
   labelled `last night`, and a 1y chart does not appear to be summarised by a
   one-night verdict.

And the header records **why the tile's component was not extracted**: the
tile's registers (the swipe, the gauge, the 28px figure) are tile furniture; a
component serving both grows two layout modes and a props matrix, and the
shipped tile takes a regression risk it gains nothing from. The divergence risk
this leaves is bounded to *geometry*, because every figure still comes from one
object.

`role="img"` + `aria-label` on the SVG for the non-interactive read.

**Tests**: nothing is recomputed — the rendered 7-day figure (where shown) is
`chart.shortAvg` and the final curve point is the same value; marks carry the
statuses from `nights`/`rolling`; the drift tag is `driftTag(chart.drift)`
verbatim; the axis prints the first and last drawn dates and a month tick;
`HrvCalibrating` prints `n of MIN_BASELINE_DAYS`.

## Task 5 — `score-plot.tsx`

**Files**: `_components/score-plot.tsx`, `score-plot.test.tsx`

One `<rect>` per morning at `xAt(i, n, width) - colW/2`, `colW = max(1, step-1)`
where `step` is the shared x-step, `fill={recoveryBandColor(recoveryBand(score))}`
at 0.85 opacity, height proportional to the score over a **fixed 0–100 domain**
— fixed because the bands are fixed and a rescaled chart would move the
hairlines. Dashed zone hairlines at **33 and 67** in `--border-strong`, labelled
at the right edge in 9px tabular numerals.

**A null score draws nothing** — no zero-height stub, no baseline tick. A gap is
a gap, and this is the one place a column chart can quietly lie. Say so in a
comment.

Measures its own width (`useMeasuredWidth`); `role="img"` + `aria-label`.

**Tests**: a null score renders no `<rect>` for that morning; a 13 fills
`--danger` and a 78 `--success`; the hairlines sit at 33 and 67 of the fixed
domain regardless of the data's range; a 365-day window still renders a ≥1px
column per morning.

## Task 6 — `rhr-plot.tsx`

**Files**: `_components/rhr-plot.tsx`, `rhr-plot.test.tsx`

`deviation = restingHr − avg` where `avg` is `baseline.restingHrAvg` — the
server's trailing average, and the plot's zero line. Above → `--warning` at
0.85; below → `--muted` at 0.7. Resting HR has no canonical band and the plot
must not invent one: only the "worse than my normal" direction takes colour, and
it takes the desaturated warning token, never `--danger`. Zero line in
`--border-strong`. Half-height scale `max(4, max|deviation|)` bpm so a flat
month is not magnified into a mountain range.

Caption, per **correction 2**, and this exact shape matters:
`baseline {windowDays}d avg {round(avg)} bpm · excl. today`.

`RhrCalibrating` (or an inline branch, the component's call) prints the honest
sentence when `restingHrAvg` is null: the diverging baseline appears once
`MIN_BASELINE_DAYS` mornings are recorded, and nothing is drawn against a number
that does not exist. The deck skips the crosshair on this row in that state.

**Tests**: bars diverge on the correct side of the server average; a null
`restingHrAvg` renders the sentence and zero `<rect>`s; a flat month does not
exceed the ±4 bpm floor scale; the caption carries `excl. today`.

## Task 7 — `window-strip.tsx`

**Files**: `_components/window-strip.tsx`, `window-strip.test.tsx`

Four cells at 24px figures, 4-up on desktop and 2-up on mobile: score, resting
HR, HRV, and a band-mix bar with its counts (`18 · 9 · 3`) from `bandCounts`.

**Correction 2 is this component's whole point.** Every cell is a client mean
over the *rendered window* and says so:

| Cell | Label | Sub-line |
| --- | --- | --- |
| Score | `Score · window avg` | `{n} nights` |
| Resting HR | `Resting HR · window avg` | `{n} nights` |
| HRV | `HRV · window avg` | `{n} nights` |

Neither figure is ever compared against `baseline.recoveryScoreAvg`,
`baseline.restingHrAvg` or `baseline.hrvAvg` anywhere on the page. The strip is
the only place a client-computed mean appears, and the comment says why Fixed
Decision 8 survives: these means have no server counterpart being re-derived —
they describe a window only the client knows about.

Calibration gate, per the SOW: a cell prints its mean only once the window holds
`MIN_BASELINE_DAYS` readings **of that metric**, and prints
`{present} of {MIN_BASELINE_DAYS} nights` otherwise. `MIN_BASELINE_DAYS` is
imported from the tiles' `shared.ts`, and a comment notes that borrowing the HRV
baseline floor as a general "enough nights to say something" threshold is a
decision rather than an accident.

**Tests**: cells print `window avg`; under `MIN_BASELINE_DAYS` readings a cell
prints its count and not a mean; band-mix counts match the fixture; the strip's
resting-HR label and `rhr-plot`'s caption are distinguishable (the correction-2
regression test — assert both strings, in one test, so the two can never
converge again).

## Task 8 — `ledger.tsx`

**Files**: `_components/ledger.tsx`, `ledger.test.tsx`

Rows are **all stored history** via `ledgerRows`, newest first, `LEDGER_PAGE_SIZE`
to a page, opening on **page 1** (correction 3 — the mockup's `useState(4)` was
a fixture staging device for the DX's mid-state requirement, not behaviour).

Columns: date (`longDate`, 12px; `shortDate` on mobile) · score (a band-tinted
pill, `color-mix(in srgb, {band} 14%, transparent)` background) · bpm · ms (12px
tabular numerals, `--muted`; 11px on mobile). **Row height is a strict 30px** —
the variant's uniform rhythm is one of its three distinguishing properties, which
is why mobile keeps four columns at a smaller size rather than wrapping to two
lines. The selected row takes `--accent-soft` with a 2px `--accent-line` rule at
its left edge.

Header prints `{n} mornings` from the count of **real rows**, never of
materialized date slots — an endpoint window wider than the user's history must
not inflate it.

**The jump link**: when the crosshair selects a morning that is not on the
current page, a quiet accent line appears — `12 Aug is on page 3 →` — and
clicking it pages there. **Selecting a morning must never silently re-page the
ledger underneath the user's hands**; page state is owned here (or by the page),
and the selection only ever *offers* to move it.

Pager: `Page {current} of {total}` with Prev/Next buttons, disabled at the ends,
in the bodyweight table's register.

**Tests**: opens on page 1; 425 rows page to 22; missing nights absent while
partial-null mornings are present with em-dashes; the jump link appears only
when the selection is off-page and pages there when clicked; selecting an
on-page morning does not change the page; row height is uniform.

## Task 9 — `deck.tsx`

**Files**: `_components/deck.tsx`, `deck.test.tsx`

One bordered instrument: a header strip naming the deck and printing the
selected morning's long date, then three rows divided by hairlines with no gaps.
Each row is `[132px, 1fr]` on desktop — metric name, 16px readout, unit, and a
`when` line — and **collapses to a single column on mobile**, the header becoming
a compact row *above* the plot rather than a gutter beside it. Plot heights:
HRV 170 → 130, score 120 → 90, RHR 100 → 72, switched by `useIsMobile()`.

`prepareDeck(view, windowLength)` is called once here. `chart.series` is the one
date list; `HrvPanel` gets `chart`, `ScorePlot` and `RhrPlot` get `days`.
When `chart` is null the HRV row renders `HrvCalibrating` and the other two rows
span the requested window — the crosshair then binds two rows, which is honest:
there is no third series to bind. The RHR row's crosshair is likewise suppressed
when `baseline.restingHrAvg` is null.

**Correction 4 — pointer, keyboard, touch:**

- **Pointer, not mouse.** `onPointerMove` / `onPointerDown` / `onPointerLeave` on
  the deck body, with `touch-action: pan-y` so vertical page scrolling still
  works. Pin semantics per decision 2.
- **Keyboard.** The deck is a focusable `role="group"` with an accessible name.
  `←`/`→` step one morning (starting from the last when nothing is selected),
  `Home`/`End` jump to the ends, `Escape` clears. Arrow keys `preventDefault` so
  the page does not scroll under the selection.
- **The readouts sit in an `aria-live="polite"` region** so a screen-reader user
  hears the morning they moved to. Each plot keeps its own `role="img"` +
  `aria-label` for the non-interactive read.
- **The gutter is measured, not assumed.** No `GUTTER`/`PLOT_PAD` pixel
  constants. A ref on the plot column; the index comes from that element's own
  `getBoundingClientRect()` at pointer time, using the same `PLOT_INSET` the
  plots draw with:
  `idx = round(((clientX - rect.left - PLOT_INSET) / (rect.width - 2*PLOT_INSET)) * (n - 1))`,
  clamped. Reading the rect in the handler rather than snapshotting a width into
  state is what keeps it true across a resize and a breakpoint change — the two
  cases the mockup's constants silently stopped being true at.

Readouts fall back to the latest morning that *has* a value when nothing is
selected, labelled `latest · 11 Aug`, never `today` (Fixed Decision 5).

The crosshair rule is `--accent-line`, 1px, positioned with the same inset the
plots use so it lands exactly on the morning at either end.

**Header comment states correction 1 explicitly** — the lead-in exists to make
the axis true, and trimming it re-introduces the six-day skew.

**Tests**:

- **Alignment**: for each window, HRV series length = score column count = RHR
  bar count, and first/last plotted dates match across rows. With history shorter
  than `window + LEAD_IN`, all three narrow to the same date list. With
  `prepareHrvChart` → null, score and RHR still render over the requested window
  and the HRV row shows the calibrating state.
- **Crosshair**: an index derived from a stubbed bounding box lands on the
  expected date; `←`/`→`/`Home`/`End`/`Escape` behave; readouts move with it;
  switching from 1y to 30d while a morning is selected **clamps** rather than
  reading past the end of the array.

## Task 10 — `page.tsx`, and the deletions

**Files**: `app/(app)/recovery/page.tsx`, `page.test.tsx`; delete
`_components/recovery-hero.tsx`, `recovery-trends.tsx`, `recovery-log.tsx` and
their three tests.

Composition, top to bottom: header (`Recovery`, the count of stored mornings,
the `30d · 90d · 1y` pills) → `WindowStrip` → `Deck` → `Ledger` → the quiet
`Manage Whoop connection →` link to `/settings?tab=integrations` in accent link
register.

The window control uses the existing `components/segmented-toggle.tsx` —
`SegmentedToggle` is already the design system's decided full-pill segmented
control (accent fill + `--accent-fg` on the active segment, `--muted`
brightening to `--foreground` otherwise). The mockup hand-rolled one because a
DX variant is throwaway; production reuses the component.

**Fetch — one `useEffect`, two requests in parallel, and no refetch on window
change:**

```ts
Promise.all([
  getWhoopConnection(token),
  getRecoveryHistory(token, { timezone }),   // no since → all stored history
]);
```

Window switching re-slices `view.days` in memory. This is the SOW's contract and
`page.test.tsx` asserts it: one fetch per load, zero on window change.

Crosshair state lives on the page (the deck reads and the ledger highlights from
it), ledger page state lives with the ledger's owner — whichever placement keeps
"selecting a morning never silently re-pages the ledger" true.

**States**, all of which must render:

| State | Treatment |
| --- | --- |
| Loading | Deck-shaped skeleton — rows at the deck's own geometry, not a centred spinner, and no layout shift when data lands. |
| Not connected / revoked | Existing `ConnectState` connect variant, **kept verbatim**. |
| Connection errored | Existing `ConnectState` reconnect variant, **kept verbatim**. |
| Connected, zero recorded mornings | Existing `FirstNightState`, **kept verbatim**. |
| Fetch error | The current page's inline `--danger` banner, **kept verbatim**, plus a retry. `401` → `clearToken()` + `router.replace("/login")`, via the existing `handleAuthError` pattern. |
| Today hasn't landed | The page barely flinches — readouts fall back to the latest morning with a value, labelled `latest · 11 Aug`; the ledger simply has no row for today. |
| Partial-null day | Em-dash in the ledger; a gap in that metric's plot only; the other two rows still draw that morning. |
| Missing night | Absent from the ledger; a gap in all three plots; the column slot still exists so the axis does not compress. |
| Calibrating | HRV row calibrating, RHR row's no-average sentence, score columns still draw (Whoop's bands need no baseline), window cells print honest counts. No confident number beside a calibrating panel. |

`recharts` must no longer be imported by anything under `app/(app)/recovery/`
— grep to prove it.

**Tests** (`page.test.tsx`): each render gate; one fetch per load and none on
window change; `401` clears the token and routes to `/login`; the fetch-error
banner and its retry; the deleted components' tests are **removed, not left
orphaned**.

## Task 11 — the gate

Run, in this order, and fix anything that fails in the code rather than around
it:

```bash
npm run lint
npm run format:check     # prettier; `npm run format` to fix
npm run typecheck
npm run test
npm run build
```

Then confirm by inspection:

- No new `react-hooks` lint warnings (the repo's `purity` /
  `set-state-in-effect` / `immutability` rules are warnings; don't add to them).
- No `//nolint`-equivalent suppressions, no skipped tests, no `--no-verify`.
- Every existing recovery test passes **unmodified** — the tiles' `hrv-chart.ts`
  and `shared.ts` were imported, not refactored.
- `git status` shows the three deleted components and their three tests gone.

Commits are Conventional Commits; the Husky `pre-commit` (lint-staged + full
typecheck) and `commit-msg` (commitlint) hooks run and are never bypassed.

## Rollout

Web-only, behind no flag, no migration and no API change — the endpoint shipped
ahead of this. Merge and deploy; the page swaps on the next build.

Hand-test after rollout:

- `/recovery` opens on the **90d** window with the deck drawn and the ledger on
  **page 1**, newest morning first.
- Drag across any of the three rows: all three readouts change together and name
  the same morning; the ledger highlights that morning when it is on the current
  page and otherwise offers `… is on page N →`.
- Tab to the deck and press `←`/`→`/`Home`/`End`/`Escape` — the crosshair steps,
  jumps and clears without the page scrolling.
- On a phone: drag the deck horizontally (the crosshair scrubs) and swipe
  vertically (the page scrolls); the ledger is reachable and still four columns
  at 30px rows.
- Switch `30d → 90d → 1y`: the network panel shows **no new request**.
- Disconnect Whoop in settings and reload — the connect CTA renders, unchanged.
