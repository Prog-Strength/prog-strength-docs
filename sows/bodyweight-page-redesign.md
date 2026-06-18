---
type: sow
status: draft
repos:
  - prog-strength-web
  - prog-strength-docs
---

# Bodyweight Page Redesign — Trend-Band-Analyst

**Status**: Draft · **Last updated**: 2026-06-17

## Introduction

`/bodyweight` is where a Prog Strength user watches the one number that tells
them whether a cut or a bulk is working. The **functionality is solid** — the
page (`app/(app)/bodyweight/page.tsx`) already supports multi-per-day logging,
range tabs (30/60/90/All), a goal line, and a log table with full edit/delete —
but the **composition buries the trend**. Today the chart plots raw readings as
loud amber dots that compete with a plain daily-mean line for attention, the
four flat stat tiles (Average/Goal/Min/Max) give nothing a hero, and the trend
line is just an arithmetic per-day mean — there's no smoothing, so daily
morning/evening noise reads as the subject rather than the trajectory.

The Design Exploration `dx/bodyweight-page.md` (PR
Prog-Strength/prog-strength-web#72) explored that surface across five in-system
variants on a representative 30-day, multi-per-day, above-goal fixture (the
states where lazy designs fall apart: heavy same-day spread, no-goal degrade,
sparse vs dense ranges). The owner selected **`trend-band-analyst`** (drawing on
**TrendWeight / Libra**): the **trend wins, decisively** — a smoothed **EWMA
curve in the violet accent** becomes the unmistakable subject, thick and
confident, wrapped in a faint **±spread band**, while the raw readings drop to
small, low-contrast slate dots in the background. A **rate-of-change** readout
("↓ −0.4 lb/wk") and a big condensed-display **"Trend now"** value lead above an
enlarged chart that owns the page; the four stat tiles compress into a single
thin **caption strip** beneath it.

This is a **web-only redesign** (plus docs bookkeeping). It rebuilds the
`/bodyweight` view against the **existing** data layer — `GET /bodyweight`,
`GET /me/bodyweight-goal`, the client-side daily-average grouping, unit handling
(`convertWeight`, `displayUnit`), range filtering, the full reading CRUD
(create/edit/delete modals + mobile action sheet), and pagination — restyling
the composition and **adding a client-side smoothing computation**, not changing
the plumbing. **There is no backend change.** The SOW inherits the DX's
`scope: in-system`: dark slate ramp, the single **violet** accent, Nunito, and
the Oswald-style condensed display face the system reserves for big stat
numerals, per [`../design-system.md`](../design-system.md). It also **retires the
legacy chart palette** (the old `#3b82f6` blue / amber / green — the blue being
exactly the "legacy blue the violet accent replaces") onto the system, and in
doing so **fixes the tooltip timestamp bug** the DX flagged (`Daily avg :
1780150948286 lb`).

## Proposed Solution

One shippable change: rebuild the `/bodyweight` chart-and-stats region into the
trend-band-analyst composition, leaving the log table and all modals
functionally intact. The variant on the DX branch
(`app/design-explore/bodyweight-page/_variants/trend-band-analyst.tsx` +
`_data.ts` + `_chart.ts`) is the **visual spec**, not code to promote — it's
reimplemented production-quality against the real route, wired to real readings
instead of static fixtures, and the `design-explore` route stays gated and
untouched.

The composition, top to bottom:

- **Range row** — the real `30 days · 60 days · 90 days · All` tabs (existing
  `range` state and from-now filtering), restyled to quiet pills since the chart
  is the headline, with a `"{n} readings · ending {date}"` caption on the right.
- **Trend hero readout** — a `"TREND NOW"` overline; the **current smoothed
  trend value** in the condensed display face (`var(--font-display)`, ~`text-5xl`)
  with a muted unit; and a **rate-of-change pill** (`↓`/`↑` + signed `lb/wk`,
  `var(--accent-soft)` fill, accent text). The arrow + sign carry direction — not
  color alone. An inline legend names the four marks: **EWMA trend**, **±spread
  band**, **readings**, **goal {n}**.
- **The chart (the hero)** — a Recharts `ComposedChart` (~300px mobile / 400px
  desktop) layering, back to front: a faint **±spread band** (`Area`, violet
  vertical gradient, no stroke), the **raw readings** (`Scatter`, muted slate
  `#6b7280` at 0.5 opacity — context, not headline), the **EWMA trend** (`Line`,
  violet `#8b7cf6`, `strokeWidth 3.5`, no dots — the subject), and the **goal**
  (`ReferenceLine`, quiet slate dashed, only when a goal is set).
- **Caption strip** — the four tiles compressed into one wrap-friendly row of
  label/value pairs: **Average**, **Range Δ** (accented), **Low** (`value ·
  date`), **High** (`value · date`), **To goal**, and a right-aligned footnote
  `"{n} days · trend smoothed (EWMA α0.25)"`.
- **Log section (unchanged behavior)** — the existing `Log` toolbar + goal
  affordance, the Date/Time/Weight table with desktop pencil/trash and the mobile
  action sheet, pagination at 20, and the Log/Goal/Edit/Delete modals all stay,
  conformed visually but functionally identical. The DX idiom redesigns the
  trend hero, not the weigh-in history.

The substantive new logic is the **smoothing layer**: a TrendWeight-style
exponentially-weighted moving average (α 0.25) over the daily means, plus a
rolling ±band and a weekly rate of change — all computed client-side from the
readings the page already fetches.

## Goals and Non-Goals

### Goals

- **Rebuild the `/bodyweight` chart-and-stats region** into the
  trend-band-analyst composition: quiet range tabs → trend hero readout →
  enlarged EWMA chart with ±band and demoted raw dots → caption strip.
- **Add a client-side smoothing computation** (`ewmaTrend`, α 0.25) over the
  existing per-day means: seed at the first day's mean, `s = α·avg + (1−α)·s`,
  with an EW-variance ±band floored at 0.6 lb plus half the intra-day spread —
  ported faithfully from the DX `_data.ts`. The smoothed curve, not the raw daily
  mean, becomes the trend line.
- **Add a weekly rate-of-change readout** computed over a **true ~7-day window**
  (production hardening over the mockup's fixed index-8 shortcut): take the trend
  point nearest 7 days before the latest and normalize `(Δtrend / Δdays)·7`;
  hide the rate gracefully when there's under ~a week of data.
- **Make the trend beat the noise**: violet EWMA curve as the only saturated
  thing on screen, raw readings demoted to muted-slate low-opacity dots inside a
  faint violet ±band — the core thing the current chart fails at.
- **Reuse the existing data layer unchanged** — `getBodyweightLog` (`GET
  /bodyweight`), `getBodyweightGoal` (`GET /me/bodyweight-goal`), the client-side
  daily-average grouping, `convertWeight`/`displayUnit` unit handling, and the
  from-now range filtering. No new endpoints, no SDK changes.
- **Preserve the full reading CRUD and log table**: the Date/Time/Weight table,
  desktop pencil/trash, the mobile `BodyweightActionSheet`, pagination (20/page),
  and the Log/Goal/Edit/Delete modals — behavior, validation, and optimistic
  updates intact inside the conformed visual.
- **Retire the legacy chart palette onto the design system**: the violet accent
  owns the trend, raw readings become muted slate, the goal becomes a quiet
  neutral dashed marker. The hardcoded `#3b82f6` (legacy blue) / `#fcd34d`
  (amber) / `#10b981` (green) are removed. Because Recharts writes stroke/fill as
  SVG attributes where CSS `var(--token)` does **not** resolve, the system tokens
  are mirrored as literal hex/rgba constants in one small colocated module (the
  same values as `globals.css`, not a new palette), with a comment pinning them
  to the design-system tokens.
- **Fix the tooltip timestamp bug** as part of the rebuild: the new tooltip
  formats the hovered **weight** (`182.4 lb`, via the existing rounding helper)
  labeled "Trend"/"Reading", labels the day via a day-long formatter, and
  **excludes the band series** from the tooltip — the noon timestamp (`t`) is
  never passed as a value, so `Daily avg : 1780150948286 lb` cannot recur.
- **Conform to the design system** ([`../design-system.md`](../design-system.md)):
  dark slate ramp, the single violet accent, Nunito + the condensed display face
  for the hero numeral, rounded cards, full-pill chips — via the CSS-variable
  tokens for page chrome and the mirrored literals for chart SVG.
- **Intentional at every state**: the representative day (multi-per-day spread,
  above goal, roughly flat trend); a **clearly trending-down** range; the
  **no-goal-set** degrade (no goal line, no "To goal", "Set goal weight" stays);
  a **sparse early-user** range and a **single-reading** day (no misleading
  rate); the **empty** state; and **narrow/mobile** widths where the hero, chart,
  and caption strip reflow without breaking.
- Keep web CI green (lint/format/typecheck/test/build), adding unit tests for
  `ewmaTrend` and the rate computation and a regression test that the tooltip
  renders a weight, never a raw timestamp.

### Non-Goals

- **Any backend change.** Both endpoints already exist; `prog-strength-api` is
  not in `repos:`. The trend, band, and rate are derived **client-side** from
  loaded readings, exactly as the daily mean is today. No schema/route/SDK work.
- **Redesigning the weigh-in log table or the modals.** They're conformed
  visually but keep their internals — columns, edit/delete, pagination, the
  mobile action sheet, and the Log/Goal/Edit/Delete flows with their validation.
  (Re-treating the history as a journal feed was a *different* DX idiom,
  `weigh-in-journal`, deliberately not selected.)
- **Promoting the DX mockup code.** `trend-band-analyst.tsx` + `_data.ts` +
  `_chart.ts` are the visual spec, not code to copy; the
  `design-explore/bodyweight-page` route stays gated
  (`designExplore` / `NEXT_PUBLIC_DESIGN_EXPLORE`), untouched, and never ships to
  production. In particular the mockup's UTC-pinned static fixtures are replaced
  by the page's real local-day grouping of live `measured_at` timestamps.
- **Changing the daily-average semantics or the timezone behavior.** Readings
  stay grouped by **local** calendar day (as today); the EWMA runs over those
  means. No switch to a server-side aggregate.
- **A configurable smoothing factor.** α is fixed at 0.25 in code (a reviewable
  constant, like the mockup), not a user setting.
- **Reworking units, goal-setting, or adding new metrics** (body-fat, measurements,
  etc.) — out of scope.

## Implementation Details

### Smoothing layer (new, client-side)

Add a small pure module (e.g. `lib/bodyweight-trend.ts`) with unit-tested
functions, fed the same per-day means the chart already builds (readings grouped
by local calendar day, converted to `displayUnit`, meaned, timestamped at local
noon — keep the existing grouping in `page.tsx`):

- `ewmaTrend(dayPoints, alpha = 0.25): { t, trend, lo, hi }[]` — ported from the
  DX `_data.ts`: `s` seeded to `dayPoints[0].avg`; per point `s = α·avg +
  (1−α)·s`; EW variance `v = α·dev² + (1−α)·v` of `dev = avg − s`; `band =
  max(√v, 0.6) + spread/2` where `spread` is the day's intra-day max−min (0 for a
  single reading); emit `lo = s − band`, `hi = s + band`. The chart consumes
  `band: [lo, hi]` as a tuple for the `Area`.
- `trendSummary(trend): { current, ratePerWeek, hasRate }` — `current` is the
  last trend value; `ratePerWeek` is time-normalized over a true ~7-day window
  (nearest point ≥7 days before the last; `(last.trend − ref.trend) / Δdays ·
  7`); `hasRate` is false when the range spans under ~a week or has a single
  point, so the hero can hide the pill instead of showing a noisy/zero rate.

EWMA runs over the **filtered range** (so the curve matches the selected tab);
note in code that the first few points of a short range are less "warmed up" —
acceptable and consistent with the range semantics (see Open Questions).

### Chart rebuild

Replace the current `ComposedChart` block (and the four-tile grid) in
`page.tsx`. Keep `ResponsiveContainer`, the forced-to-include-goal Y domain
(`[floor(min(min,goal))−2, ceil(max+2)]`), and the mobile/desktop axis-width and
tick-format switches that already exist. New layering and tokens per the variant:
violet gradient `Area` band → muted-slate `Scatter` raw readings (opacity 0.5) →
violet `Line` trend (3.5px, dotless) → quiet-slate dashed goal `ReferenceLine`
(conditional on a goal). Tooltip: day-long label, value via the existing
`formatNumber`, band series suppressed.

### Chart color tokens

One colocated constants module mirroring the design-system hex (violet accent
`#8b7cf6`, accent band `rgba(139,124,246,~0.16–0.22)`, raw-dot slate `#6b7280`,
goal slate `#9aa1ad`, grid `rgba(255,255,255,0.06)`, axis text `#9aa1ad`, tooltip
surface `#1e2128`), with a comment that these must track `globals.css` /
`design-system.md` and are mirrored only because SVG attributes can't read CSS
vars. No `#3b82f6` / `#fcd34d` / `#10b981` anywhere after this change.

### Preserved surface

Data fetching, `displayUnit` (from the most recent entry) + `convertWeight`,
range filtering from now, the derived Average/Δ/Min+date/Max+date/To-go stats
(now rendered in the caption strip), the goal affordance, the log table with
edit/delete + pagination + mobile action sheet, and all four modals — unchanged
in behavior.

## Testing

- **`lib/bodyweight-trend.ts` unit tests**: EWMA seeding and α weighting against
  a known series; band floor (0.6) and intra-day-spread contribution; single-day
  input (trend = that mean, minimal band); `trendSummary` rate over a clean
  7-day window, over a **sparse/gapped** window (time-normalized, not index-8),
  and the `hasRate=false` fallbacks (single point, <1 week).
- **Component tests** (`page.tsx` region): hero shows the current trend value and
  the correct arrow/sign for a down- vs up-trending fixture; the rate pill is
  hidden when `hasRate` is false; caption strip renders Average/Δ/Low/High/To-go;
  the **no-goal** fixture omits the goal line, the "goal" legend, and the "To
  goal" caption while keeping "Set goal weight"; empty and single-reading states
  render without a broken chart.
- **Tooltip regression test**: hovering the trend/readings yields a `"… lb"`
  weight string, and asserts the formatter never returns a raw millisecond
  timestamp (guards the DX-flagged bug) — and that the band series contributes no
  tooltip row.
- **Responsive check**: the hero, chart, and caption strip reflow at mobile width
  without overflow.
- Web CI green: lint/format/typecheck/test/build.

## Rollout

One **web PR** rebuilds the `/bodyweight` chart-and-stats region + adds
`lib/bodyweight-trend.ts` + tests; mergeable on its own, no backend dependency.
Plus **docs bookkeeping**: this SOW, and setting `dx/bodyweight-page.md` to
`status: selected` (trend-band-analyst) with PR #72 closed (never merged).

## Open Questions

1. **Rate-of-change window.** Recommend the time-normalized ~7-day window over
   the mockup's fixed 8-points-back (more honest with gaps/multi-per-day). Confirm
   the "under ~a week of data → hide the pill" threshold (7 days proposed).
2. **Rate pill color.** Recommend keeping the single-accent treatment (arrow +
   sign convey direction) for in-system restraint, rather than tinting the pill by
   trending-toward vs away-from goal — the latter reintroduces a second semantic
   color the system avoids. Easy to switch if you'd rather the away-from-goal case
   shout.
3. **EWMA over filtered range vs all history.** Recommend computing over the
   **filtered** range so the curve matches the selected tab; the trade-off is that
   the earliest points of a short range are less warmed-up. Alternative: always
   smooth over full history and slice to the range (steadier early curve, but the
   visible curve depends on data outside the window). Filtered is simpler and
   matches the mockup.
4. **α value.** 0.25 (gentle) is the mockup's choice and reads the trajectory
   well on the fixture; revisit only if real data looks too laggy or too jumpy —
   a one-constant change with the tests to confirm.
