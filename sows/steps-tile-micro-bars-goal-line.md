---
type: sow
status: shipped
repos:
  - prog-strength-web
  - prog-strength-docs
---

# Dashboard Steps Tile — Micro-Bars + Goal Line

**Status**: Shipped · **Last updated**: 2026-07-03

> Frontend SOW. It implements a chosen DX variant and therefore inherits that DX's
> `scope` — here **`in-system`**. The visual foundation (design-system **v0.4**,
> oura-calm) is already decided, so this SOW **conforms** to it; it does **not**
> re-tone the system or touch shared tokens. One repo does the work
> (`prog-strength-web`); `prog-strength-docs` only flips the DX to `selected` and
> marks this SOW shipped.

## Introduction

The **Steps mini-card** on the dashboard (`StepsCard` in
`app/(app)/dashboard/page.tsx`) is one tile in the 6-up grid that shipped from the
`command-center` exploration (`sows/dashboard-command-center.md`). It carries the
grid's generic furniture: a `BigNum` "16k today" headline over a **`Spark`** line
(`_components/spark.tsx`) and a `MetaRow` of `avg · goal`. The
Design Exploration `dx/steps-tile.md` (PR Prog-Strength/prog-strength-web#106)
called out why that furniture actively underserves this particular metric:

- **The sparkline answers the wrong question.** `Spark` is **min/max-normalized**,
  so it shows "the line goes up-ish" but hides the only things a step-counter
  cares about — *did I hit my goal, on which days, and how does today compare to
  my average?* The line can trend up while every day sat under goal.
- **The goal is never drawn.** It's a number in the meta row, not a reference the
  days are read against.
- **The average is never drawn.** Same problem — text-only.
- **Per-day structure is lost.** Normalization renders a steady week and a
  one-huge-Saturday week nearly identically; one 16k day flattens the rest.

The DX explored the tile across **five `in-system` variants** (all sharing v0.4:
soft near-black ramp, the single periwinkle accent `#9aa6d6`, Manrope with tabular
figures, desaturated success/muted), diverging only on **layout, structure,
density, and composition** — not palette, accent, or type. The owner selected
**`micro-bars-goal-line`** (references: **Apple Fitness / Gentler Streak**
goal-relative day treatment): the tile keeps the big **"16k today"** `BigNum`
headline, then replaces the sparkline with **seven slim daily bars drawn
goal-relative** — a dominant dashed **goal line** across the chart, a fainter
**average** line behind it, over-goal bars cresting the goal line in **success**,
under-goal bars sitting **muted** below, and **today's** bar carrying the
**accent**. The `MetaRow` keeps `avg · goal` as numbers. It is the **most literal
answer to the brief** and a faithful tile-scale shrink of the deep Steps page's
own goal-relative bars, so the tile previews the page it links into.

This SOW reimplements that variant **production-quality** against the real tile and
data, replacing the `Spark`-based `StepsCard` body. Because `scope: in-system`,
there is **no palette, accent, type, or design-system change** — it conforms to
v0.4. There is **no backend change**: every value the composition needs is already
in the tile's `StepsView` (see below); **no new endpoint and no new fetch are
assumed**.

## Proposed Solution

The dashboard's steps section is already fully shaped for this. `StepsCard` renders
a `Section<StepsView>` (`lib/dashboard.ts`), and `StepsView` carries exactly the
four values the variant heroes:

```ts
type StepsView = {
  avg: number;          // average daily steps  → the average reference line
  today: number;        // today's count        → the BigNum + today's accented bar
  goal: number | null;  // null = no goal set   → the dashed goal line (or its absence)
  spark: number[];      // 7 daily counts, oldest→newest → the seven bars
};
```

`spark` here is **already the seven raw daily counts** (from the API's
`DashboardSteps.daily_spark`, sanitized non-finite→0 by `adaptSteps`) — it is the
`Spark` component that discards their magnitude by min/max-normalizing. So the work
is **entirely presentational**: a new small chart replacing `<Spark>` inside
`StepsCard` only, driven by the `spark` / `avg` / `goal` the card already holds.

The chart is a compact, hand-rolled `StepsGoalBars` (SVG/divs, mirroring the
mockup) that draws, within the tile's chart band:

1. **Seven daily bars, linearly scaled** against a ceiling of `max(goal, maxDay)`
   (with a little headroom) — **not** min/max-normalized. This is load-bearing: it
   keeps one 16k day from flattening the rest, makes a logged **0** an honest floor
   bar rather than a mid-height artifact, and lets over-goal bars visibly **crest**
   the goal line.
2. **A dominant dashed goal line** across the band at the goal's scaled height —
   the primary reference the bars are read against.
3. **A fainter average line** behind the bars at `avg`'s scaled height — so
   "am I above my norm, and is my norm above goal?" is something you *see*.
4. **Color as goal-state, not decoration** (steps has no discipline hue): each bar
   is **success** when it clears the goal, **muted** when under; **today's** bar is
   the **accent** (`--accent`). Lines: goal in success, average in the accent/quiet
   ink — final token assignment confirmed at review against v0.4 so meaning reads
   without inventing a steps hue.

The `BigNum` "16k today" headline and the `MetaRow` (`avg · goal`) are kept — the
chart swaps **1:1** for the sparkline, preserving the tile's existing rhythm and
its ~180px height budget. The `present: false` empty state (`MiniCardEmpty` CTA
"Log your steps to start tracking") is untouched. The five sibling tiles (Running,
Lifting, Nutrition, Bodyweight, Streak) are **not touched** — the winner must still
read as one of the grid.

The deep Steps page's `lib/steps-stats.ts` (`buildAxis` / `deriveStats`) is the
**reference** for the goal-relative semantics (per-day `logged` / `hit` / `delta`,
goal-relative and null-safe with no goal); it is **not** re-fetched or re-run on the
dashboard — the tile derives its bar heights and colors from the seven `spark`
counts + `avg` + `goal` it already has.

## Goals and Non-Goals

### Goals

- **Add a small, tested `StepsGoalBars` chart component** (dashboard-local, e.g.
  `app/(app)/dashboard/_components/steps-goal-bars.tsx`) that renders, from
  `{ spark, avg, goal }`:
  - **Seven linearly-scaled daily bars** against a `max(goal, maxDay)` ceiling with
    headroom — a 16k day does not flatten the week, a logged **0** is an honest
    floor bar, and over-goal bars **crest** the goal line.
  - **A dominant dashed goal line** and **a fainter average line**, both drawn (not
    listed), positioned at their scaled heights.
  - **Goal-state coloring**: over-goal bars **success**, under-goal **muted**,
    **today** (last bar) the **accent** — using the shared v0.4 tokens
    (`--success`, `--muted`, `--accent`), **never raw hex or a new hue**.
- **Rewire `StepsCard`** (`app/(app)/dashboard/page.tsx`) to render `StepsGoalBars`
  in place of `<Spark points={v.spark} …>`, keeping the `BigNum` today headline, the
  `MetaRow` (`avg · goal` — `goal` still shows "set a goal" when null), the
  `MiniCard` chrome/href, and the `present: false` `MiniCardEmpty` state.
- **Handle every degraded state the DX enumerated**, with no `NaN`, no goal line at
  zero, no fake bar implying data that isn't there:
  - **No goal set** (`goal === null`) — **no goal line**; bars scale to `maxDay`;
    coloring falls back to neutral/accent-today (no over/under split with no goal to
    split on); meta shows "set a goal". No `NaN%`, no zero-height goal line.
  - **A `0` day** — renders as an **honest floor bar**, not a mid-height artifact.
  - **A brand-new user with only today** / **fewer than seven days of history** —
    the fixed seven-slot axis holds; absent days are quiet floor/empty slots, the
    band never looks broken at `n = 1`.
  - **An all-zero week** — flat floor, no crest, no divide-by-zero.
  - **Today under vs over goal** — today's bar is the accent in both cases; its
    crest/sit-below still reads against the goal line.
- **Stay a calm dashboard tile** — within the ~180px budget, quiet enough to sit
  beside Running/Lifting/Nutrition/Bodyweight/Streak, sharing the grid's language
  (and ideally the `StreakCard`'s) rather than shouting.
- **Tests** — priority on the chart:
  - `StepsGoalBars` — bar heights linearly scaled (a 16k day does not flatten a 6k
    day); goal line present and positioned when `goal` set, **absent** when null;
    over-goal bar success / under-goal muted / today accent; **no goal** →
    neutral+today, no goal line; **all-zero** and **single-day** fixtures render
    without `NaN` / broken frame; a **logged 0** is a floor bar.
  - `StepsCard` (extend `page.test.tsx`) — renders the bars + `BigNum` + `avg · goal`
    when present; the `MiniCardEmpty` CTA when `present: false`; "set a goal" when
    `goal` null.
  - **CI green** (lint/format/typecheck/test/build).

### Non-Goals

- **Any backend / API / SDK change.** The composition is derived entirely from the
  `StepsView` (`{ avg, today, goal, spark }`) the tile already receives.
  `DashboardSteps.daily_spark` already carries the seven daily counts; **no new
  endpoint, no payload change, no new fetch.** `prog-strength-api` and
  `prog-strength-sdk` are **not** in `repos:`. (One nullable-`spark` enrichment is
  noted in Open Questions as an explicit *follow-up*, out of scope here.)
- **Any design-system or shared-token change.** `scope: in-system` against v0.4 —
  conform only; no token/accent/type edit, no `design-system.md` change, **no
  invented steps hue**.
- **Touching the other five dashboard tiles**, the shared `Spark` / `BigNum` /
  `MiniCard` / `MetaRow` primitives (they keep serving the other tiles unchanged),
  or the dashboard's data layer / loaders.
- **The deep Steps page** (`/activities?view=steps`) and its `steps-stats.ts` — out
  of scope and unchanged; this SOW only swaps the dashboard tile's chart body.
- **Promoting the DX mockup code.** The `dx/steps-tile` variant sources
  (`app/design-explore/steps-tile/_variants/micro-bars-goal-line.tsx` + `_shell` /
  `_fixtures`) are the **visual spec, not code to copy**; the
  `design-explore/steps-tile` route stays gated
  (`NEXT_PUBLIC_ENABLE_DESIGN_EXPLORE`) and never ships. The other four variants are
  not built; the DX PR is **closed, never merged**.

## Implementation Details

### Chart component (`app/(app)/dashboard/_components/steps-goal-bars.tsx`)

A pure, presentational component taking `{ spark: number[]; avg: number; goal:
number | null }` (the values already on `StepsView`). It computes a single scale —
`ceiling = maxHeadroom × max(goal ?? 0, ...spark)` — and maps every bar, the goal
line, and the average line through it, so bars, goal, and average share one axis and
over-goal bars crest the goal line. Bars are div/SVG rects; the goal line is a dashed
rule; the average line a fainter rule behind the bars. Color is assigned per bar from
goal-state (`steps >= goal` → success, else muted; last index → accent for today),
falling back to neutral-plus-accent-today when `goal` is null. Non-finite inputs are
already sanitized upstream by `adaptSteps`; the component still guards an empty/short
`spark` by rendering the fixed seven-slot axis with quiet empty slots. It reads the
v0.4 CSS vars directly (`--success`, `--muted`, `--accent`, hairline/`--surface-*`),
matching how the other tiles' charts are toned.

### Tile rewire (`app/(app)/dashboard/page.tsx`)

In `StepsCard`, replace the `<Spark points={v.spark} className="… text-[var(--accent-2)]" />`
line with `<StepsGoalBars spark={v.spark} avg={v.avg} goal={v.goal} />`, keeping the
surrounding `MiniCard`, the `BigNum` today headline, and the `MetaRow` (`avg · goal`,
`goal` → "set a goal" when null) exactly as they are. The `present: false` branch
(`MiniCardEmpty` CTA) is unchanged. No other card in the file is touched.

### Tests (`prog-strength-web`)

- **`steps-goal-bars.test.tsx`** — the scaling and goal-state assertions above over
  representative fixtures (a mixed week with over/under/zero days and a goal; a
  no-goal week; an all-zero week; a single-day/new-user week).
- **`page.test.tsx`** — extend the existing dashboard test so the Steps tile
  asserts the bars + `BigNum` + `avg · goal` present-state, the `MiniCardEmpty`
  empty-state, and the "set a goal" no-goal meta.

## Rollout

1. **`prog-strength-docs`** — flip `dx/steps-tile.md` to `status: selected` (winning
   idiom `micro-bars-goal-line`); mark this SOW `shipped` on merge.
2. **`prog-strength-web`** — the `StepsGoalBars` component + `StepsCard` rewire +
   tests, in one PR. Vercel preview to verify the tile against a normal week, a week
   with a huge day, a week with a logged 0, a no-goal user, a new user with only
   today, and both breakpoints.

### Verification after rollout

- The dashboard **Steps tile** reads as a **per-day goal-relative bar week** — you
  can see the shape of the week (which days cleared goal, which sat under) the way
  the old sparkline never let you.
- The **goal is drawn** (dashed line) and the **average is drawn** (fainter line) —
  not buried as two numbers; over-goal bars **crest** the goal line in success,
  under-goal bars sit muted, **today** is the accent, and the "16k today" `BigNum`
  still leads.
- **One big 16k day no longer flattens the rest** (linear, not min/max
  normalization); a logged **0** is an honest floor bar.
- **Degrades honestly**: no goal → no goal line, bars scale to the max day, meta
  reads "set a goal", **no `NaN`**; a new user with only today and a sub-seven-day
  history hold the seven-slot axis without a broken frame; an all-zero week is a flat
  floor.
- It **sits beside Running, Lifting, Nutrition, Bodyweight, and Streak** as the same
  v0.4 surface, within the tile budget; the shared `Spark`/`BigNum`/`MiniCard`
  primitives still serve the other tiles unchanged; `design-system.md` unchanged; the
  `design-explore/steps-tile` route stays gated / 404 in production; **no DX mockup
  code shipped**; the DX PR is closed (never merged).

## Open Questions

1. **Unlogged day vs a logged `0`.** `StepsView.spark` (from
   `DashboardSteps.daily_spark`) is a `number[]` with **no nullability**, so an
   unlogged day and a logged-0 day both arrive as `0` — the tile cannot distinguish
   "didn't log" from "walked zero." `micro-bars-goal-line`'s own framing renders a
   `0` as an honest **floor bar**, and the current `Spark` already floors unlogged
   days to 0, so this is **not a regression** and the tile ships correct against
   today's payload. **Lean:** accept the server's current 0-semantics for v1 — do
   **not** enrich the payload in this SOW. If the product later wants true "no data"
   honesty (an empty slot, distinct from a zero-step floor), that is a **follow-up**
   that changes `daily_spark` to `(number | null)[]` end-to-end and **would** add
   `prog-strength-api` (+ the SDK) to its `repos:`. Flag at review; keep it out of
   scope here.
2. **Chart implementation: hand-rolled SVG/divs vs recharts.** The seven-bar +
   two-line composition is small and needs no tooltips/axis; the mockup hand-rolls
   it, and the other mini-card charts favor the lightweight primitive over recharts.
   **Lean:** **hand-roll** `StepsGoalBars` (SVG/divs) rather than pull recharts into a
   ~180px tile. Confirm against how the sibling tiles' charts are built.
3. **Average vs goal line collision.** When `avg` sits near `goal` (a
   consistently-on-target user), the two reference lines can overlap and muddy. **Lean:**
   keep the goal line dominant (dashed, forward) and the average quiet (thin, behind
   the bars); if they coincide, the goal line wins the pixel. Confirm the treatment
   reads at tile scale.
4. **Bar-scale ceiling.** Scaling to exactly `max(goal, maxDay)` can pin the tallest
   bar to the frame top and hide the crest-over-goal reading. **Lean:** add a small
   fixed **headroom** (e.g. ×1.1) above `max(goal, maxDay)` so over-goal bars visibly
   crest the goal line and the top bar isn't clipped. Confirm the headroom factor at
   review.
