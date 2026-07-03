---
type: dx
status: selected
selected_idiom: micro-bars-goal-line
surface: steps-tile
idioms:
  - micro-bars-goal-line
  - goal-hit-columns
  - dual-baseline-editorial
  - progress-meter
  - daily-ledger
references:
  - Apple Fitness
  - Gentler Streak
  - The Athletic
  - GitHub
  - Robinhood
scope: in-system
variant_count: 5
repos:
  - prog-strength-web
  - prog-strength-docs
---

# DX: Steps Tile

**Status**: Selected (`micro-bars-goal-line`) · **Last updated**: 2026-07-03

> **Selected:** `micro-bars-goal-line` (DX comparison PR
> Prog-Strength/prog-strength-web#106, closed un-merged). Built for real by
> [`sows/steps-tile-micro-bars-goal-line.md`](../sows/steps-tile-micro-bars-goal-line.md).

> A DX (Design Exploration) is the platform's **divergent** work type. Unlike a
> SOW it does not converge on one correct implementation — it produces N
> differentiated visual variants of a single frontend surface, side by side on
> one comparison route, awaiting a human pick. It **never merges** and ships no
> production code; the chosen direction feeds a downstream SOW that builds it for
> real.

## Context

The Dashboard shipped from the [`dx/dashboard`](./dashboard.md) exploration —
the `command-center` direction was selected and built by
[`sows/dashboard-command-center.md`](../sows/dashboard-command-center.md): a dense
KPI strip over a 6-up grid of per-domain mini-cards, each a glance that links
into its deep page. That exploration set the *whole grid's* tone at once; it gave
every tile the same furniture — a `BigNum` headline, a normalized `Spark`
line, and a `MetaRow` of facts. Good enough as a first pass, but the Steps tile
is the one where that generic furniture actively underserves the metric.

This DX narrows the lens the way [`dx/steps-view`](./steps-view.md) narrowed from
[`dx/activities-page`](./activities-page.md) — down to **just the dashboard Steps
tile** — and explores it properly. It matters because **daily step count against
a goal is a habit metric a Prog Strength user checks constantly**, and the tile's
current sparkline answers the wrong question. The `Spark` is min/max-normalized,
so it shows "the line goes up-ish" but hides the only things a step-counter
cares about: *did I hit my goal, on which days, and how does today compare to my
average?* Concretely, today the tile:

- **never draws the goal** — it's a number in the meta row, not a line the days
  are read against, so the sparkline can trend up while every day sat under goal;
- **never draws the average** — same problem, it's text-only;
- **loses per-day structure** — normalization renders a steady week and a
  one-huge-Saturday week nearly identically; and
- **lies about unlogged days** — a missing day arrives as `0` and the line dips
  to the floor, reading as "barely walked" rather than "didn't log."

The owner's intent for this exploration: make the tile read like a **daily log /
per-day bar graph with the goal and average clearly shown**, while staying
honest that this is **one tile in a 6-up grid** — it cannot grow tall or loud.
The spread below deliberately does *not* converge on "the same bar chart five
times"; each idiom heroes a different element (today's bars, hit-consistency, the
average, goal-progress, the daily log) so the pick is a real choice, not a
coin-flip between accent colors.

`scope: in-system`: the foundation is decided (see
[`../design-system.md`](../design-system.md) **v0.4**, oura-calm) — soft
near-black ramp, the single **periwinkle** accent (`#9aa6d6`, app-chrome only),
**Manrope** with tight numeric tracking (`-0.03em`), tabular figures, 14px
hairline panels, desaturated success/danger. Variants do **not** re-litigate
palette, accent, or type. They diverge on **layout, structure, density, and
composition** inside a single mini-card. The tile sits **beside the five other
dashboard tiles** (Running, Lifting, Nutrition, Bodyweight, Streak) under one
grid, so the winner has to still read as one of them — not a louder tile that
breaks the grid's calm.

## The surface

The **single Steps mini-card** on `app/(app)/dashboard/page.tsx` — the
`StepsCard` component, one cell of the responsive
`grid-cols-1 sm:grid-cols-2 lg:grid-cols-3` `CardGrid`. A variant owns everything
inside the card's `MiniCard` shell (the uppercase `STEPS` title, the `p-4`
padding, and the whole-card `next/link` into `/activities?view=steps` are chrome
it keeps functional), and composes the card body. It must **not** grow the card's
footprint meaningfully or add nested interactive controls that fight the card's
click target.

**The space budget is the hard constraint.** Today the body is `BigNum` (~32px) +
`Spark` (`h-7` = 28px) + `MetaRow` (~16px) with `gap-3` gutters — roughly
**150–170px** of content. Treat **~180px total** as the ceiling and the current
tile as the reference; every variant is judged at true tile width against its
five neighbors, not in isolation.

The data is exactly what the dashboard summary already returns — **no backend
change for this round.** From `prog-strength-web/lib/dashboard.ts`, the tile's
view-model is:

```ts
type StepsView = {
  avg: number;        // server-computed average — a window figure, NOT the
                      //   mean of `spark`; display as-is, never recompute it
  today: number;      // today's step count
  goal: number | null;// the daily target; null ⇒ no goal set
  spark: number[];    // 7 daily counts, oldest → newest, ENDING TODAY
};
```

sourced from `DashboardSteps` (`lib/api.ts`): `{ avg, today, goal, daily_spark }`,
where `daily_spark` is **7 daily counts, oldest→newest** — i.e. exactly a week of
daily bars, the last of which is today. Everything a variant wants to *add* — a
per-day goal delta, an at/over-goal count, a "days hit this week" ratio, an
inline mini-bar, an attainment %/arc — is derived **client-side** from those four
fields. The weekday label for each spark value is derivable (`today − (6 − i)`
days); no new field needed.

**Color logic (in-system) — the key constraint, inherited from
[`dx/steps-view`](./steps-view.md).** Steps has **no `--discipline-*` hue** (the
system enumerates run green-teal and lift steel-blue only). So a step bar has no
activity color of its own to wear — variants resolve goal state using **the
periwinkle accent and the desaturated status colors the system already ships**,
never a new hue:

- **at / over goal** reads as a **win** — desaturated success green (`#86b39f`,
  `CHART_STEPS_MET`) or the accent, used as the *meaningful* color;
- **under goal** stays **muted slate** (`CHART_STEPS_UNDER` `#5b6168`) — never
  alarming a normal low-step day with danger red;
- the **accent earns its place on meaning** (today, the hero figure, goal
  attainment), not as generic chrome on every bar.

This mirrors the shipped deep Steps page, whose goal-relative bars use exactly
these hexes (`prog-strength-web/lib/chart-colors.ts`).

**Visual states every variant must render in the mockup** (this is where lazy
tile designs fall apart):

- **Goal set, today over it** — the happy screenshot case (today 16k vs 10k
  goal). Over-goal should feel like a win against the goal reference.
- **A sub-goal day in the week** — some of the seven days fall short; over vs
  under must read differently, not just by height.
- **A big outlier day** — one 16k day next to an 8–9k day; the mini-chart can't
  let one tall bar flatten the rest into indistinguishable stubs (the current
  `Spark`'s exact failure is the opposite — it *over*-normalizes).
- **Today vs average vs goal** — the three numbers the owner wants legible at
  once; a variant must place all three without clutter at tile scale.
- **Unlogged / zero day** — `daily_spark` cannot distinguish a logged `0` from an
  unlogged day (both are `0`); render `0` honestly as a floor bar and **do not**
  invent a heuristic to guess "unlogged" (see *Known data gap* below).
- **No goal set** (`goal: null`) — every goal-relative treatment (the line, the
  over/under coloring, the % / arc, the hit count) degrades to a neutral
  "just the steps" view with a `set a goal` affordance — never `NaN%` or a goal
  line at zero.
- **Brand-new / sparse** — a user with only today's count (no meaningful week or
  average) still gets an intentional tile, and the fully-empty case falls back to
  the grid's shared `MiniCardEmpty` CTA ("Log your steps to start tracking"),
  matching its five neighbors.
- **Both breakpoints** — the tile is full-width single-column on mobile and
  one-third-width on desktop; the composition must survive both.

**Representative fixture** (mirror the owner's screenshot — a "this week" tile,
goal 10,000, today riding well over it, a couple of strong days and a couple of
misses):

```ts
const steps: StepsView = {
  today: 16000,
  avg: 13100,          // server window figure (≈ shown "13.1k")
  goal: 10000,
  //         6d ago → today (oldest → newest); Sun & the 3rd day miss goal
  spark: [11200, 14800, 9400, 13000, 15100, 8700, 16000],
};
```

Derived examples a variant can show: **today +6,000 vs goal** (`160%`),
**5 of 7 days at-or-over goal**, **today vs avg +2.9k**, per-day deltas
(`+4.8k` best day, `−1.3k` on the Sunday miss). Note the seven spark values
average to ~12.6k, which is **deliberately not** the server `avg` of 13.1k — the
two are computed over different windows; a variant must show `avg` as the
authoritative server number and never silently recompute it from the week.
Also build a **no-goal** fixture (`goal: null`) and a **today-only / empty**
fixture — the happy path is easy; the degraded states are where a tile breaks.
These are mockups: **static fixtures that look real are preferred** — do not wire
the variants to live steps services.

## Idioms

Five genuinely distinct compositions of the *same* near-black / periwinkle /
Manrope mini-card. None re-decides palette, accent, or type — they diverge along
**type scale** (one giant tabular figure vs even Manrope weight vs tiny mono
rows), **color logic** (how the accent and success/muted status colors separate
goal-meaning from neutral chrome, given steps has no discipline hue), and
**spacing rhythm** (a single hero + chart vs a gridded row of boxes vs a dense
ledger). Each makes a **different element the hero** and leans on a different
reference. All five fit the ~180px tile budget.

- **micro-bars-goal-line** — Heroes **today against a per-day bar week**. Keeps
  the `BigNum` "16k today" headline, then replaces the sparkline with **seven
  slim daily bars** drawn **goal-relative**: a dashed **goal line** across the
  chart is the dominant reference, over-goal bars crest it in success green,
  under-goal bars sit muted below, today's bar carries the accent. A faint
  secondary line marks the **average**. The `MetaRow` keeps `avg · goal` as
  numbers. Type scale: one big today figure over tiny bars. Color logic:
  success = cleared goal, muted = under, accent = today. Spacing: the current
  tile's rhythm, chart swapped 1:1 for the sparkline. This is the **most literal
  answer to the brief** and a faithful shrink of the deep page's own
  goal-relative bars. Borrowing **Apple Fitness / Gentler Streak** goal-relative
  day treatment. → "did I beat my goal today, and how did the week go?"

- **goal-hit-columns** — Heroes **consistency**, in the shipped `StreakCard`'s
  own idiom (the sibling tile two cells over). Drops the y-axis entirely: seven
  small **columns under `M T W T F S S`**, each **colored by outcome** —
  success-filled when the day hit goal, muted when under, faint-outline when
  unlogged — and the meta line becomes a **`5/7 hit`** attainment ratio beside
  `avg · goal`. Optionally the columns scale slightly by attainment, but color is
  the primary signal. Type scale: modest; hierarchy from the row of boxes and the
  ratio, not one giant number. Color logic: green = hit, muted = miss, faint =
  gap. Spacing: tight, gridded, streak-card-consistent. Trades magnitude for a
  dead-simple "did I or didn't I" read and adds a fact the tile can't show today.
  Borrowing the **GitHub contribution row** and **Gentler Streak**. → the most
  glanceable "am I consistent?" tile, and the most on-grid with its neighbor.

- **dual-baseline-editorial** — Heroes **the average**. Promotes the
  average-daily-steps figure to the **oversized headline** (huge tabular Manrope,
  `13.1k` over a small "avg / day · this week" caption), demoting **today** to a
  small delta chip (`today 16k ▲`). Beneath, faint per-day bars carry **both the
  goal and the average drawn as reference lines**, so "am I above my norm, and is
  my norm above goal?" is something you *see*. Type scale: dramatic big/small
  contrast — the one number that matters huge, everything else small. Color
  logic: near-monochrome ink-on-dark, the **accent reserved for the average
  baseline**, success only on the goal line/crest. Spacing: airier within the
  tile, editorial. The most literal take on "**goal and average clearly
  displayed**." Borrowing **The Athletic**'s headline-figure framing. → makes the
  headline health metric *feel* like the headline.

- **progress-meter** — Heroes **goal attainment as a shape**. A compact
  **arc / ring or a horizontal progress meter** leads the tile — today's (or the
  week's) **% of goal** rendered as a fillable shape with the value in it
  (`16k`, arc filled past 100% cresting to success) — a tile-scale shrink of the
  deep page's selected `goal-ring-hero`. A **7-bar sparkbar** sits beneath as
  supporting texture; `avg · goal` stays as captions. Type scale: the meter's
  center/edge number is the size anchor; everything else small. Color logic:
  accent **fills the meter's progress**, flipping to success once at/over goal.
  Spacing: centered, compact around the shape. Makes the deep page's chosen idiom
  legible on the dashboard, so the tile previews the page it links into. Borrowing
  **Apple Fitness** rings, shrunk to tile scale. → the most visceral "how close
  am I to goal" answer.

- **daily-ledger** — Heroes **the log itself** — "daily log count" read
  literally. Drops the chart for a compact **stack of the ~3–4 most-recent day
  rows**: `Thu 16.0k ▲ · Wed 8.7k ▼ · Tue 15.1k ▲`, each row a weekday, the
  count, a **tiny inline mini-bar**, and a **goal-delta sign** (accent/success
  ▲ on a beat, muted ▼ on a miss), under a quiet `avg 13.1k · goal 10k` header.
  Type scale: small functional tabular/mono rows, the column does the work, no
  giant figure. Color logic: accent/success spent **entirely on the delta sign
  and inline bar**, rows otherwise neutral. Spacing: dense blotter, tight vertical
  rhythm. The "daily log count" phrasing taken at its word — a mini-ledger that a
  power user would actually read. Borrowing **Robinhood**'s sparkline list rows.
  → for the user who wants the last few days as discrete, dated facts.

## References

In-system, so "what to take" from each is **structural** — composition, density,
and data legibility, not their palettes or type:

- **Apple Fitness** (activity rings) — take the **goal-as-shape**: an abstract
  percent rendered as a fillable arc/ring you read instantly, with the value in
  it. Drives `progress-meter`; the crest-past-goal reading reinforces
  `micro-bars-goal-line`.
- **Gentler Streak** (goal/effort attainment) — take its **goal-relative day
  treatment** and gentle "on / off target" framing. Reinforces
  `micro-bars-goal-line` and `goal-hit-columns`.
- **The Athletic** (sports data journalism) — take its **authored
  headline-figure layout**: one oversized number with a small caption, the chart
  demoted to a captioned figure. Drives `dual-baseline-editorial`.
- **GitHub** (contribution graph) — take the **row of outcome-shaded day cells**
  where consistency and gaps read spatially, compressed to a single week. Drives
  `goal-hit-columns`.
- **Robinhood** (sparkline list rows) — take the **inline per-row mini-bar +
  delta** that turns a short list into a glanceable trend. Drives `daily-ledger`.

## Selection criteria

A note-to-self for the pick, not a rubric the worker optimizes against. When I
compare these I'm trying to decide:

- **Does it read as a daily log / per-day view at a glance** — can I see the
  shape of my week (which days I moved, which I didn't) the way the sparkline
  never let me?
- **Are the goal and the average genuinely legible** — drawn or clearly shown,
  not buried as two numbers in a meta row — without the tile getting busy?
- **Does goal attainment come through** — over vs under, today vs the line — and
  does the mini-chart survive one big 16k day without flattening the rest?
- **Does it stay a calm dashboard tile** — within the ~180px budget, quiet enough
  to sit beside Running/Lifting/Nutrition/Bodyweight/Streak without shouting, and
  ideally sharing a language with the `StreakCard` beside it?
- **Do the degraded states hold** — no goal set, an unlogged/`0` day, a brand-new
  user with only today — with no `NaN%`, no goal line at zero, no fake 0-step bar
  implying data that isn't there?
- **Does today-vs-average-vs-goal read without me doing the math** — is the
  "am I above my norm, is my norm above goal" story told, not just listed?
- **Does it still read as Prog Strength v0.4** — near-black, periwinkle as
  meaning not chrome, success/muted for goal state, no invented steps hue — and
  does it preview the deep Steps page it links into rather than looking like a
  different app's widget?

---

> **Lifecycle.** `status:` is editorial — the owner is the dispatch gate. It moves
> `draft` → `exploring` (worker running) → `awaiting_selection` (draft PR open,
> owner deciding) → `selected` / `abandoned`. The worker sets `awaiting_selection`
> on the `dx/steps-tile` branch as it opens the PR; the owner sets the terminal
> value when they close it.
>
> **Handoff.** This DX ends at a *chosen variant*, not merged code. I open the
> draft `[DX — DO NOT MERGE]` PR, compare the variants on the preview deploy
> (`/design-explore/steps-tile`, flag-gated behind
> `NEXT_PUBLIC_ENABLE_DESIGN_EXPLORE`, driven across the full / no-goal / empty
> fixtures), pick one, tick its box, set `status: selected` (noting the winning
> idiom), and **close the PR — never merge it.** Then I open a SOW: *"implement
> the dashboard Steps tile per the `<chosen-idiom>` variant from `dx/steps-tile`,
> production-quality, conforming to the design system"* — replacing the current
> `Spark`-based `StepsCard` body. The mockup code is never promoted as-is.
