---
status: draft
repos:
  - prog-strength-web
  - prog-strength-docs
---

# Recovery Page Refresh — `aligned-deck`

**Status**: Draft · **Last updated**: 2026-08-12

## Introduction

[`dx/recovery-page-refresh`](../dx/recovery-page-refresh.md) explored four
compositions of `/recovery` and **`aligned-deck` was selected** (DX comparison PR
Prog-Strength/prog-strength-web#167, closed un-merged). This SOW builds it for
real.

The page it replaces was written when `/recovery` was the only place Whoop data
was properly rendered. Five recovery tiles have shipped since — `recovery`,
`morning_vitals`, `hrv_balance`, `recovery_log`, `resting_hr` — and between them
they answer *how am I this morning?* at a glance, above the fold, without a
click. A page whose hero is a bigger version of the score ring one scroll up is
answering a question that has already been answered.

So the page's job changed, and the DX records the new one as a fixed decision:
**history and depth**. What a tile cannot hold — long windows, the full ledger,
patterns across weeks — is what the page is for. Three specific defects follow
from the old brief and all three are in scope here:

- **The three charts are the same chart three times.** Recovery score, resting
  HR and HRV render as identical white polylines with a dashed average. They are
  visually interchangeable, which is a strange thing to do to three metrics with
  completely different meanings and baselines — especially now that the HRV tile
  draws its metric with a drifting band, shape-changing per-night marks and a
  gauge, legibly, in a third of the space.
- **The day log shows three rows.** Unpaginated and effectively truncated. A user
  with fourteen months of ingested mornings can reach none of it.
- **The colour story is unused.** Recovery is *the* three-band metric and Whoop
  trained every user to read it at a glance. The page spends band colour on the
  log chips and nowhere else.

`aligned-deck`'s argument is that the real question on a history page is *what
happened to all three at once* — and that three panels with independent x-axes
make that comparison impossible. It keeps three full-width panels, gives each the
right **form** for its metric, and binds them to **one time axis and one
crosshair**: drag anywhere and all three read out the same morning.

What changes for the user: a `/recovery` that opens on a stretch of time rather
than on this morning; three charts that no longer pretend to be the same chart; a
recovery score whose colour finally says something; and a ledger they can page
back to March in.

## Proposed Solution

A rebuilt `app/(app)/recovery/page.tsx` in five registers, top to bottom:

1. **Header + window control** — the title, the count of stored mornings, and a
   `30d · 90d · 1y` pill group. Not the old `7d · 30d · 90d`: a page about
   history starts at a month and reaches a year.
2. **The window strip** — the hero, and it is *window orientation*, never today.
   Four quiet cells at 24px: score average, resting-HR average, HRV average, and
   a band-mix bar with its day counts (`18 · 9 · 3`). Per Fixed Decision 5 a
   window figure is never an em-dash where a number could be, and never a stale
   number wearing today's label.
3. **The deck** — one bordered instrument, three rows divided by hairlines, no
   gaps: **HRV** (the shipped tile chart at page scale), **Recovery** (a banded
   column chart in Whoop's canonical thirds), **Resting HR** (diverging bars
   around the server's trailing average). One shared x-domain, one accent
   crosshair, three readouts that move together.
4. **The ledger** — paginated over **all stored history**, 20 rows a page, dense
   tabular numerals, the crosshair's morning highlighted, and a jump link when
   the selected morning is on another page.
5. **The backlink** — a quiet `Manage Whoop connection →` in accent link
   register.

The data comes from **one fetch** of `GET /recovery/history` — the endpoint
[`sows/recovery-history-endpoint.md`](./recovery-history-endpoint.md) ships
first — plus the existing `GET /me/whoop/connection` for the render gates.
Switching windows re-slices what is already in memory; it does not re-fetch. The
`RecoveryView` the page holds is the same one the dashboard tiles hold, mapped by
the same `adaptRecovery`, so the HRV chart's machinery imports and runs unchanged.

The DX mockup is **not promoted**. `app/design-explore/**` is throwaway and stays
on its closed PR. Five corrections to it are specified below and are the
substance of this work beyond transcription — the first of them is a defect in
the variant's own central claim.

## Goals and Non-Goals

### Goals

- `/recovery` renders the `aligned-deck` composition, production-quality,
  conforming to [`design-system.md`](../design-system.md) v0.4 (`scope:
  in-system`, inherited from the DX).
- **One x-domain, actually.** The HRV plot, the score columns and the resting-HR
  bars are drawn over the **same list of dates**, so the crosshair lands on the
  same morning in all three rows at both ends of the window. See correction 1 —
  the mockup does not achieve this.
- **The HRV panel is the shipped tile's chart**, rendered from `prepareHrvChart`
  and the shared `hrv-chart.ts` primitives. Nothing about the ribbon, the rolling
  curve, the shape-changing marks or the gauge is redesigned, and **nothing
  recomputes a figure `prepareHrvChart` already returns** (Fixed Decisions 2 and
  8).
- **Score is a banded column chart** — one column per morning in its canonical
  Whoop band, zone hairlines at 33 and 67, drawn from the shared
  `recoveryBand`/`recoveryBandColor` helpers the tiles already use.
- **Resting HR is diverging bars** around `baseline.restingHrAvg` — the server's
  trailing average as the zero line. No band is claimed, and when the average is
  null the row says so rather than drawing against a number that does not exist.
- **The ledger pages over all stored history**, not the selected range, opening
  on page 1, with a jump link to the selected morning's page.
- Every DX visual state renders correctly: **today hasn't landed, each band
  present, partial-null days, missing nights, calibrating, deep ledger,
  sparse long window** — plus loading, fetch error, not connected, revoked,
  errored, and zero rows.
- The deck is operable **without a mouse and on a phone**: pointer events, not
  mouse events; keyboard focus and arrow-key stepping; a mobile layout that does
  not assume a 132px gutter.
- No client arithmetic invents a baseline, a band, or a verdict. The page's only
  maths is means over the **rendered window** (labelled as such, and visibly
  distinct from the server baseline it sits beside — correction 2), ranking a
  value into an already-computed band, and mapping values onto pixels.
- The old `recovery-hero.tsx` / `recovery-trends.tsx` / `recovery-log.tsx` are
  deleted along with their tests, and `recharts` is no longer imported by this
  page.

### Non-Goals

- **No API change.** `GET /recovery/history` is
  [its own SOW](./recovery-history-endpoint.md) and ships first; this SOW assumes
  it. `prog-strength-api` is deliberately absent from `repos:`.
- **No resting-HR band** (the DX's prerequisite P2). `aligned-deck` does not
  carry it, and picking `aligned-deck` was the product's answer to
  `resting-hr-tile`'s Open Question 3: resting HR is answered with the average
  that exists.
- **No dashboard change.** The five recovery tiles are untouched. `hrv-chart.ts`
  and `recovery/shared.ts` are **imported, not refactored**; if either needs a
  new export it is additive and every existing recovery test passes unmodified.
- **No new tile, no catalog change, no default-layout change.**
- **No mobile-app change.** `prog-strength-mobile` has no recovery page.
- **No changes to the render gates.** Connect / reconnect / first-night are fine
  and stay as written, per the DX.
- **No brushing, no zoom, no URL-encoded window state.** The window control is
  three pills; `season-rail`'s scrub was a different variant and lost.
- **No sleep, no strain, no workout overlay** on the deck.
- **No promotion of the DX branch code.** `app/design-explore/**` stays where it
  is.

## Implementation Details

### File layout

```
app/(app)/recovery/
  page.tsx                       # fetch, gates, window state, crosshair state
  _components/
    window-strip.tsx             # register 2
    deck.tsx                     # register 3 — rows, crosshair, readouts
    hrv-panel.tsx                # the shipped chart at page scale
    score-plot.tsx               # banded columns
    rhr-plot.tsx                 # diverging bars
    ledger.tsx                   # register 4 — paginated, all history
    shared.ts                    # dates, windowing, paging, scales
  fixtures.ts                    # test + state fixtures
```

`shared.ts` is the page's own; it does **not** re-export the recovery tiles'
`shared.ts`, which it imports from. Deleted: `recovery-hero.tsx`,
`recovery-trends.tsx`, `recovery-log.tsx` and their tests.

### Correction 1 — the deck's axis does not actually align

**This is the load-bearing correction.** The variant's whole argument is one
x-axis, and the mockup breaks it by six days.

`prepareHrvChart` returns `series` — *"the tail of `days` beginning at the oldest
one with a complete rolling window behind it"* — which is `days.length − 6` on a
full payload, because a 7-night rolling mean cannot be formed over the first six
charted mornings. The HRV plot maps `x(i)` across `series.length`. The score and
resting-HR plots map `xAt(i, n)` across `windowDays.length`. Same inset, same box,
**different point counts** — so the leftmost HRV point sits above a score column
six mornings newer than it. The error is largest at the left edge and tapers to
zero at the right: on a 30-day window that is a fifth of the plot width, and the
crosshair is lying about which morning it is reading.

The fix is lead-in, not clipping:

```ts
// The HRV chart consumes ROLLING_WINDOW_NIGHTS - 1 mornings before it can plot
// its first point, so hand it that many EXTRA mornings and its `series` comes
// out equal to the visible window — one point per column, by construction.
const LEAD_IN = ROLLING_WINDOW_NIGHTS - 1; // 6, imported, never a literal
const hrvView = windowView(view, tail(days, windowLength + LEAD_IN));
const chart = prepareHrvChart(hrvView);
```

Then the deck derives its shared column list from what the chart actually drew:

- When `chart` is non-null, the deck's dates are `chart.series.map(d => d.date)`,
  and the score and resting-HR plots are sliced to exactly those dates. This
  covers the short-history case too: a user with 40 stored mornings asking for
  90 days gets a `series` shorter than the requested window, and all three rows
  narrow together rather than one of them narrowing alone.
- When `chart` is `null` (calibrating), the HRV row renders the calibrating state
  and the other two rows span the requested window. The crosshair then binds two
  rows, which is honest — there is no third series to bind.

`PLOT_INSET` stays `4.6` and stays a shared constant imported by all three plots,
because it is the HRV chart's own `R_MAX` (the radius of the largest mark it
draws) and the columns are centred on the HRV chart's positions rather than on
their own slots. A comment says so at the constant, because it looks arbitrary
and is not.

**A test pins this**: for each of the three windows, `chart.series.length ===
scoreColumns.length === rhrBars.length`, and the first and last plotted dates are
equal across the three rows.

### Correction 2 — two averages on one screen that can disagree

The window strip prints `Resting HR avg` — a mean over the **rendered window,
including today**. The resting-HR plot's caption prints `30d avg 54 bpm` — the
server's `baseline.restingHrAvg`, a trailing 30-day mean that **excludes today**,
and which is also the plot's zero line. At the 30d window these are labelled
almost identically, computed differently, and will print different numbers. A
user who notices reads it as a bug, and they are right to.

They are genuinely different figures and the page keeps both — the bars must
diverge from the server's baseline, and the strip must describe the window the
user chose. So the labels stop pretending otherwise:

| Register | Label | Figure |
| --- | --- | --- |
| Window strip cell | `Resting HR · window avg` with `{n} nights` beneath | client mean over the rendered window |
| RHR plot caption | `baseline {win}d avg {v} bpm · excl. today` | `baseline.restingHrAvg`, `baseline.windowDays` |

The same rule applies to the score and HRV cells: they are window means, they say
`window avg`, and neither is ever compared against `baseline.recoveryScoreAvg` or
`baseline.hrvAvg` anywhere on the page. The window strip is the only place a
client-computed mean appears, and it is the reason Fixed Decision 8 survives —
these means have no server counterpart being re-derived; they describe a window
only the client knows about.

The strip's calibration gate stays as the mockup has it: a cell prints its mean
only once the window holds `MIN_BASELINE_DAYS` readings of that metric, and prints
`{present} of {MIN_BASELINE_DAYS} nights` otherwise. That constant is the HRV
baseline floor being borrowed as a general "enough nights to say something"
threshold; it is imported from `recovery/shared.ts` rather than restated, and the
borrowing is noted in a comment so it is a decision rather than an accident.

### Correction 3 — the ledger opens on page 1

The mockup initialises `pageState` to `4` to satisfy the DX's requirement that
variants render a **mid-state** page rather than page one. That is a fixture
staging device, not behaviour. Production opens on page 1 — the newest mornings —
and the mid-state is exercised by tests and by the user clicking `Next`.

The jump affordance stays and is the reason the mid-state matters at all: when
the crosshair selects a morning that is not on the current page, a quiet accent
line appears — `12 Aug is on page 3 →` — and clicking it pages there. Selecting a
morning must never silently re-page the ledger underneath the user's hands.

### Correction 4 — pointer, keyboard, touch

The mockup drives the crosshair from `onMouseMove` on the deck container, with
the plot's left edge computed as `rect.left + GUTTER`. Three problems, all real:

- **Touch does nothing.** Use pointer events (`onPointerMove` / `onPointerDown` /
  `onPointerLeave`) so a finger drag scrubs the deck. `touch-action: pan-y` on
  the deck so vertical page scrolling still works.
- **Keyboard does nothing.** The deck is a focusable `role="group"` with an
  accessible name; `←`/`→` step the crosshair one morning, `Home`/`End` jump to
  the ends, `Escape` clears it. The three readouts sit in an `aria-live="polite"`
  region so a screen-reader user hears the morning they moved to, and each plot
  keeps its `role="img"` + `aria-label` summary for the non-interactive read.
- **The gutter constant is assumed, not measured.** `GUTTER` (132) and
  `PLOT_PAD` (16) are pixel constants that must equal the CSS grid's first column
  and the row's right padding, and they silently stop being true at the mobile
  breakpoint. Measure the plot area instead: one `useMeasuredWidth` ref on the
  plot column, and derive the index from that element's own bounding box.

### Correction 5 — the page has no mobile layout, and no loading or error state

**Mobile** (`< 640px`): the deck's `[132px, 1fr]` grid collapses to a single
column — the metric name, readout and unit become a compact header row *above*
each plot rather than a left gutter beside it. The window strip goes 4-up → 2-up
(the mockup already does this). The ledger drops its `bpm` and `ms` columns to a
second line per row, or keeps four columns at 11px with the date shortened to
`12 Aug`; the pager stays. Plot heights shrink proportionally (HRV 170 → 130,
score 120 → 90, RHR 100 → 72). Nothing is hidden — this is the history surface,
and the ledger in particular must be reachable on a phone.

**Loading**: skeleton rows matching the deck's geometry, not a centred spinner
and not a layout that jumps when data lands.

**Fetch error**: the existing inline `--danger` banner from the current page,
kept verbatim, with a retry. `401` still clears the token and routes to `/login`.

### The HRV panel — promoted machinery, not a promoted component

Fixed Decision 2 says the tile's chart is carried forward unredesigned. There are
two ways to honour that and the choice matters:

- **Extract a shared component from `hrv-tile.tsx`** parameterised by height —
  rejected. The tile's registers (the swipe between balance and trend views, the
  gauge, the 28px figure, the verdict line) are tile furniture; the deck wants a
  plot with a date axis and a one-line verdict. A component serving both grows
  two layout modes and a props matrix, and the shipped tile — a tile the owner
  likes, on the dashboard — takes the regression risk of a refactor it gains
  nothing from.
- **Render from the shared machinery** — chosen. `hrv-chart.ts` is already pure,
  React-free and exhaustively tested, and its own header names this page as its
  intended second consumer. The page's `hrv-panel.tsx` imports `prepareHrvChart`,
  `bandRuns`, `rollingRuns`, `weekBand`, `scaler`, `gaugePct` and the colour
  helpers, and draws the same grammar at page scale: the band polygon from
  `bandRuns`, the rolling curve from `rollingRuns`, the circle / triangle /
  diamond marks from `nights`, `hrvStatusColor` for status, `driftTag` +
  `driftColor` for the drift line.

The divergence risk this leaves is bounded to *geometry*, because every figure
still comes from one object. What the page adds over the tile is exactly two
things the DX names: **more room** and **a dated x-axis** (month ticks read off
the drawn series, plus the window's ends).

Two invariants the panel must hold, both consequences of `prepareHrvChart`'s
contract:

1. **The window must end on the last stored morning.** `rolling`'s final entry is
   overwritten with the server's `hrv.shortAvg`, so a window ending earlier would
   terminate the curve on a figure describing a different day. All three of the
   page's windows are tails ending today, so this holds — and it is stated in a
   comment so a future "brush a past stretch" feature does not quietly break it.
2. **The verdict register describes last night, not the window.** With the 1y
   window selected, the status word and the drift tag still describe the most
   recent morning. They are labelled `last night` so a year-wide chart does not
   appear to be summarised by a one-night verdict.

### The score plot

One `<rect>` per morning, `fill={recoveryBandColor(recoveryBand(score))}` at 0.85
opacity, height proportional to the score over a fixed 0–100 domain — fixed
because the bands are fixed and a rescaled score chart would move the hairlines.
Zone hairlines at **33 and 67**, dashed, in `--border-strong`, labelled at the
right edge in 9px tabular numerals.

A **null score draws nothing** — no zero-height stub, no baseline tick. A gap is
a gap. This is the DX's partial-null and missing-night requirement, and it is the
one place a column chart can quietly lie.

Column width is `max(1, step − 1)` where `step` is the shared x-step, so a
365-day window renders 1px columns with no gap and a 30-day window renders fat
ones — the same plot at three densities, no branching.

`recoveryBand`, `recoveryBandColor` and `recoveryBandWord` are imported from
`app/(app)/dashboard/_components/recovery/shared.ts`. The thresholds are not
restated here: one definition of Whoop's thirds in the codebase, and it is
already the one the ledger's pill and the `recovery_log` tile use.

### The resting-HR plot

`deviation = restingHr − baseline.restingHrAvg`, mapped onto a pixel. That
subtraction is the page's only arithmetic on a server figure and it is a
comparison, not a re-derivation — the average it diverges from is the server's,
printed on the plot.

- Above the average → `--warning` at 0.85; below → `--muted` at 0.7. Resting HR
  has no canonical band and the plot must not invent one: only the "worse than my
  normal" direction takes colour, and it takes the desaturated warning token, not
  `--danger`.
- The zero line is `--border-strong`, captioned per correction 2.
- The half-height scale is `max(4, max|deviation|)` bpm so a flat month is not
  magnified into a mountain range.
- **When `baseline.restingHrAvg` is null** (calibrating) no bars are drawn and
  the row prints the honest sentence the mockup has: the diverging baseline
  appears once `MIN_BASELINE_DAYS` mornings are recorded. The crosshair skips
  this row.

### The ledger

Rows are **all stored history**, newest first, 20 to a page (`LEDGER_PAGE_SIZE`,
the bodyweight readings table's precedent). Mornings Whoop never recorded are
**dropped from the ledger** while remaining gaps in the charts — the charts keep
the date-aligned slot because that is what makes the absence visible; the record
has nothing to record. A morning with *some* metrics prints em-dashes for the
missing ones and is kept.

Columns: date (`Wed 12 Aug`, 12px) · score (a band-tinted pill, `color-mix` at
14% on the band colour) · bpm · ms (12px tabular numerals, `--muted`). Row height
is a strict 30px — the variant's uniform rhythm is one of its three distinguishing
properties and must survive implementation. The selected row takes
`--accent-soft` with a 2px `--accent-line` rule at its left edge.

The header prints `{n} mornings` from the count of **real rows**, not of
materialized date slots — an endpoint window wider than the user's history must
not inflate it.

### Data fetching

One `useEffect`, two requests in parallel, no refetch on window change:

```ts
Promise.all([
  getWhoopConnection(token),
  getRecoveryHistory(token, { timezone }),   // no since → all stored history
])
```

- `getRecoveryHistory` is a new function in `lib/api.ts` beside
  `listWhoopRecovery`, following the house pattern (bearer token, `config.apiUrl`,
  `data.recovery` unwrapped, error text on non-2xx).
- `adaptRecovery` in `lib/dashboard.ts` becomes **exported** — a one-word change
  — so the page maps the wire payload with the same adapter the dashboard uses. A
  second adapter would be a second place for the band fields to be mis-mapped.
- `listWhoopRecovery` stays in `lib/api.ts`: it is still the client for
  `GET /whoop/recovery` and other callers may use it. If the page is its only
  caller after this lands, deleting it is a follow-up, not this SOW's business.
- Window switching slices `view.days` in memory. Three windows over one payload
  is one request per page load, which is what the DX's prerequisite P3 assumes.

### The states

| State | Treatment |
| --- | --- |
| **Loading** | Deck-shaped skeleton; no layout shift when data lands. |
| **Not connected / revoked** | Existing `ConnectState` connect variant, unchanged. |
| **Connection errored** | Existing `ConnectState` reconnect variant, unchanged. |
| **Connected, zero rows** | Existing `FirstNightState`, unchanged. |
| **Fetch error** | Inline `--danger` banner with retry; `401` → clear token, `/login`. |
| **Today hasn't landed** | The page barely flinches — its subject is the window. Readouts fall back to the latest morning that *has* a value and label it `latest · 11 Aug`, never `today`. The ledger simply has no row for today. |
| **Partial-null day** | Em-dash in the ledger; a **gap** in the plot for that metric only; the other two rows still draw that morning. |
| **Missing night** | Absent from the ledger; a gap in all three plots; the column slot still exists so the axis does not compress. |
| **Calibrating** | HRV row renders the calibrating state (`prepareHrvChart` → `null`); RHR row renders its no-average sentence; score columns still draw, because Whoop's bands need no baseline. Window cells print honest counts. **No confident number appears anywhere beside a calibrating panel.** |
| **Deep ledger** | 425 mornings → 22 pages; pager and jump link exercised mid-state in tests. |
| **Sparse long window** | 1y selected with three months stored: all three rows narrow together to `chart.series` (correction 1); the strip's counts say how many nights are actually behind each figure. |

### Design system

`scope: in-system`, inherited from the DX. No new tokens, no new hues, no new type
face. What the implementation must preserve, because these are the three
properties that made `aligned-deck` a distinct variant rather than a layout:

- **Uniform small type.** 10px uppercase labels, 11–12px ledger figures, 16px
  crosshair readouts, 24px window figures. Nothing larger — there is deliberately
  no hero number on this page.
- **Confined colour.** Band colour appears in the score columns and the ledger
  pill and essentially nowhere else. Resting HR is monochrome plus `--warning`
  above average. The periwinkle accent is spent only on the crosshair rule, the
  active window pill, the selected ledger row, the jump link and the backlink.
- **Strict rhythm.** The three deck rows are one instrument separated by hairlines
  only — `py-3` per row, `gap-4` page-wide, 30px ledger rows, no editorial
  whitespace anywhere.

Tokens used, all existing: `--surface`, `--surface-2`, `--border`,
`--border-strong`, `--foreground`, `--muted`, `--faint`, `--accent`,
`--accent-fg`, `--accent-soft`, `--accent-line`, `--success`, `--warning`,
`--danger`, `--radius-card`, `--radius-pill`.

### Testing

Vitest + React Testing Library, per component, in the house style:

**Alignment (`deck.test.tsx`)** — the correction-1 contract:

- For each window, the HRV series length equals the score column count equals the
  RHR bar count, and the first/last plotted dates match across rows.
- With history shorter than `window + LEAD_IN`, all three rows narrow to the same
  date list.
- With `prepareHrvChart` returning null, the score and RHR rows still render over
  the requested window and the HRV row shows the calibrating state.

**Crosshair (`deck.test.tsx`)** — index from a stubbed bounding box lands on the
expected date; `←`/`→`/`Home`/`End`/`Escape` behave; readouts and the ledger
highlight move together; switching from 1y to 30d while hovering clamps rather
than reading past the end of the array.

**Score plot** — a null score renders no `<rect>`; a 13 is `--danger` and a 78 is
`--success`; hairlines sit at 33 and 67 of a fixed 0–100 domain; a 1px column at
the 1y window still renders.

**RHR plot** — bars diverge on the correct side of the server average; a null
`restingHrAvg` renders the sentence and no bars; a flat month does not exceed the
±4bpm floor scale.

**Window strip** — cells print `window avg` labels; under `MIN_BASELINE_DAYS`
readings a cell prints its count, not a mean; band-mix counts match the fixture;
the strip's resting-HR figure and the plot caption are labelled distinctly
(correction 2).

**Ledger** — opens on page 1; 425 rows page to 22; missing nights are absent while
partial-null mornings are present with em-dashes; the jump link appears only when
the selection is off-page and pages there when clicked; row height is uniform.

**HRV panel** — nothing is recomputed: the rendered 7-day figure is
`chart.shortAvg` and the final curve point is the same value; marks carry the
statuses from `nights`; the drift tag comes from `driftTag`.

**Page (`page.test.tsx`)** — each render gate; one fetch per load and none on
window change; `401` clears the token and routes to `/login`; the deleted
components' tests are removed rather than left orphaned.

**Fixtures (`fixtures.ts`)** — the DX's own, which are deliberately unkind and
should stay that way: today is **13 on 69 bpm and 45 ms** (Wed 12 Aug 2026), Tue
11 Aug is 78 · 50 · 106, Mon 10 Aug is 52 · 47 · 96, Sun 9 Aug has a null score,
**Fri 7 Aug has no row at all**, there are **five consecutive below-band HRV
nights in late July**, and history runs **14 months** so the ledger really pages.
The 30-day window averages score 61 · 54 bpm · 89 ms with the HRV baseline having
drifted **+6 ms over 4 weeks**. A fixture full of green days proves nothing.

### Documentation

- Each component carries a header comment in the house voice, and `deck.tsx`'s
  states correction 1 explicitly — the lead-in exists to make the axis true, and
  a future reader trimming it would silently re-introduce the six-day skew.
- `hrv-panel.tsx` records why the tile's component was not extracted, so the
  duplication reads as a decision.
- This SOW's `status` moves to `shipped` on merge; `dx/recovery-page-refresh.md`
  is already `selected`.

## Open Questions

1. **Should the window control be `30d · 90d · 1y` or `30d · 90d · 1y · All`?**
   The fetch already returns all stored history, so `All` is free and is the
   honest maximum for a page whose job is history; against it, a 3-year window at
   1px per column is a texture rather than a chart, and the ledger is the better
   surface for that depth. **Lean: ship the three, add `All` if the ledger's
   pager turns out to be doing all the deep-history work.**
2. **Should the crosshair persist on click rather than following the pointer?**
   Today it clears on `pointerleave`, so the selected ledger row cannot be read
   while the pointer is over the ledger. A click-to-pin would fix that at the cost
   of a mode. **Lean: pin on click, clear on `Escape` or a second click** — the
   jump link already implies a selection that outlives the hover.
3. **Should the window strip's figures come from the server instead?** The API
   could emit per-window aggregates and remove the only client-computed means on
   the page. Against it: the three windows are sliced client-side from one
   payload, so server aggregates would mean either three sets of figures on every
   response or a refetch per window. **Lean: keep the client means, labelled
   `window avg`** — they describe a window only the client knows about, which is
   the exact carve-out Fixed Decision 8 leaves open.
4. **Does `resting_hr`'s rank strip belong on this page?** `metric-focus` argued
   it should be promoted to page scale, and that variant lost — but the argument
   was about the *whole page's* structure, not about whether a rank belongs
   anywhere on it. **Lean: not now.** `aligned-deck`'s thesis is that the three
   metrics share one axis; a rank strip has no time axis and would be the one
   object on the deck that the crosshair cannot touch.
