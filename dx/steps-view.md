---
type: dx
status: selected
surface: steps-view
idioms:
  - editorial-average
  - goal-ring-hero
  - streak-momentum
  - sparkline-ledger
  - calendar-heat
references:
  - The Athletic
  - Apple Fitness
  - Gentler Streak
  - Whoop
  - Robinhood
  - GitHub
scope: in-system
variant_count: 5
repos:
  - prog-strength-web
  - prog-strength-docs
---

# DX: Steps View

**Status**: Selected (`goal-ring-hero`) · **Last updated**: 2026-06-19

> **Selected:** `goal-ring-hero` (DX comparison PR
> Prog-Strength/prog-strength-web#87, closed un-merged). Built for real by
> [`sows/steps-view-goal-ring.md`](../sows/steps-view-goal-ring.md).

> A DX (Design Exploration) is the platform's **divergent** work type. Unlike a
> SOW it does not converge on one correct implementation — it produces N
> differentiated visual variants of a single frontend surface, side by side on
> one comparison route, awaiting a human pick. It **never merges** and ships no
> production code; the chosen direction feeds a downstream SOW that builds it for
> real.

## Context

The Steps tab of the Activities page (`/activities`) shipped from the
[`dx/activities-page`](./activities-page.md) exploration — the `oura-calm-minimal`
direction was selected, built by
[`sows/activities-page-redesign.md`](../sows/activities-page-redesign.md), and
promoted into the design system as v0.4. That exploration set the *whole page's*
tone across four tabs at once; it did not give the Steps tab the attention its own
content deserves. So Steps shipped with the page's generic furniture: four
identical glassy stat tiles, a plain bar chart, and a `Date · Steps` log that
reads like a spreadsheet export — the same database-grid complaint the bodyweight
page named and the [`dx/bodyweight-readings-table`](./bodyweight-readings-table.md)
DX narrowed in to fix.

This DX narrows the lens the same way, to **just the Steps tab**, and explores it
properly. It matters now because **average daily steps is one of the headline
health metrics a Prog Strength user checks** — for many users it's the single
number that says whether their non-training activity is trending the right way —
and right now it's a small label in a row of four look-alike tiles, with no more
weight than `Total steps`. The chart plots the days but says little about the one
thing a step-counter cares about: *am I hitting my goal, and how often?* And the
log is a grid of rows. A steps view should answer **"how's my daily movement, and
am I consistent?"** at a glance; today it lists numbers and leaves the user to do
the reading.

`scope: in-system`: the foundation is decided and was reaffirmed by the page
redesign that *created* v0.4 — soft near-black ramp, the single desaturated
**periwinkle** accent (`#9aa6d6`), **Manrope** with tight numeric tracking and
**no display face**, 14px hairline panels, tabular figures. Variants do **not**
re-litigate palette, accent, or type. They diverge on **layout, structure,
density, and composition** — how the average metric is hero'd, how the chart is
reframed around the goal, and what the log row becomes. The Steps tab sits beside
the already-shipped Workouts and Running tabs under one tab bar, so the winner has
to still read as the same surface as its neighbors.

## The surface

The full Steps tab of `/activities`, everything below the period filter
(`Last 7 / 30 / 90 days · All`, default `30`) and the tab bar — both are page
chrome a variant may restyle but must keep functional-looking. A variant owns this
whole region and may reorganize, demote, or fold its parts together, but must
account for all of it:

- **KPI row** — four stat tiles today: **`Avg daily steps`** (`9,400`),
  **`Total steps`** (`47,000`), **`Best day`** (`14,000`, with its date `Jun 18`),
  **`Goal`** (`10,000`, with `94% of 10,000` as the avg-vs-goal attainment). Making
  **`Avg daily steps` the hero of this region** — rather than one of four equals —
  is a central job of this DX. A variant may collapse, re-rank, or absorb the other
  three tiles in service of that.
- **Daily steps chart** — a vertical **bar per day** across the period, with a
  **dashed goal line** at the target. Over-goal and below-goal days must read
  differently. The chart's job is not just to plot the days but to make
  **goal attainment** legible — today the bars and the goal line barely
  differentiate a 14,000 day from a 6,500 one beyond height.
- **Log region** — a `✎ Log steps` action (pencil affordance, opens the
  create/edit modal) on the left and a goal affordance on the right
  (`◎ Goal: 10,000`), sitting above an editable **table**: **`Date · Steps`** rows,
  sorted **most-recent-first**, each with **edit + delete** icon buttons. This is
  the "reads spreadsheet-y" region; de-gridding it without losing
  fast-to-scan / quick-to-add is the table half of the DX.

The data shape (from `prog-strength-web/lib/api.ts`):

```ts
type StepsEntry = {
  id: string;
  date: string;  // YYYY-MM-DD — a calendar day, no time-of-day
  steps: number;
};

type StepsGoal = {
  goal: number;  // the user's daily step target, e.g. 10,000
};
```

Endpoints: `GET /activities/steps` (scoped by the active period) → `StepsEntry[]`,
`GET /me/steps-goal` → `StepsGoal`. Everything a variant wants to *add* — the
**average**, the **best day**, **% of goal**, a **per-day goal delta**, a
**goal-hit count / rate**, a **logged-or-hit streak**, an **inline sparkline**, a
**calendar heat cell** — is derived **client-side** from the flat
`{ date, steps }` list plus the single `goal`; the API returns only daily totals.
One entry per calendar day (no multiple-readings-per-day complication, unlike
bodyweight) — so the design weight goes entirely into goal-attainment and
consistency, not into intra-day grouping.

**Color logic (in-system) — the key constraint for this surface.** Steps has **no
`--discipline-*` hue**; the system enumerates run (green-teal) and lift
(steel-blue) only, with mobility/core reserved. So unlike a workout or a run, a
step bar has no activity color of its own to wear. Variants resolve goal state
using **the periwinkle accent and the desaturated status colors** the system
already ships — **never a new hue**:

- **Over / at goal** reads as a **win** — the desaturated success green
  (`#86b39f`) or the periwinkle accent, used as the *meaningful* color.
- **Under goal** stays **muted slate** (or, where a variant wants to mark a clear
  miss, the restrained danger `#c79292` — sparingly, never alarming a normal
  low-step day).
- The accent earns its place on **meaning** — goal attainment, the active
  selection, the hero number — **not as generic chrome** on every bar or row.

This mirrors how the shipped activities and bodyweight surfaces reconciled their
charts to the accent: structural color, not decoration.

**Visual states the variants must handle** (render them all in the mockup — this
is where lazy chart-and-table designs fall apart):

- **Over vs under goal.** Some days clear `10,000` (`Jun 14` 11,000, `Jun 18`
  14,000), some fall short (`Jun 16` 5,500, `Jun 17` 6,500). Over-goal should feel
  like a win; the dashed goal line is the reference the whole chart reads against.
- **A big day that compresses the rest.** `Jun 18` at **14,000** is ~2.5× the
  low days; the y-axis can't let one tall bar flatten the others into
  indistinguishable stubs (the current chart's weakness).
- **The average vs the goal.** Avg `9,400` sits **just under** the `10,000` goal
  (`94%`) — the near-miss is the interesting story, and the hero treatment must
  make "almost there" legible, not round it away.
- **A missing / unlogged day.** Days with no entry are gaps, not zeros — the chart
  must distinguish *didn't walk* from *didn't log*, and the table/heatmap must not
  imply a 0-step day where there's simply no data.
- **Mixed density across the period filter.** `7 days` is a handful of bars/rows;
  `90 days` / `All` is a long scroll or a wide chart — the same layout (and the
  heatmap, the sparkline rows, the streak ribbon) has to survive both sparse and
  dense.
- **Empty / first entry.** Zero entries ("Tap Log steps to add today") and the
  one-entry case where there's no average-vs-trend story yet — the region should
  still feel like a place to start, with no broken stat or empty chart frame.
- **No goal set.** The goal affordance degrades to `Set step goal`, and every
  goal-relative treatment (the ring %, the over/under coloring, the goal line, the
  hit-streak) must degrade gracefully to a neutral "just the steps" view rather
  than a broken `NaN%` or a goal line at zero.
- **Both breakpoints.** Desktop (the `<table>` and full-width chart) and mobile
  (the chart reflowed, the log as tappable rows/cards) — a variant defines the row
  and chart treatment for both.

**Representative fixture** (mirror the screenshot — a `Last 30 days` view, goal
10,000, average riding just under it, a couple of strong days and a couple of
misses). Recent entries, newest first:

| date          | steps  |
| ------------- | ------ |
| Thu, Jun 18   | 14,000 |
| Wed, Jun 17   |  6,500 |
| Tue, Jun 16   |  5,500 |
| Mon, Jun 15   | 10,000 |
| Sun, Jun 14   | 11,000 |

Derived examples a variant can show: **avg 9,400** (`94%` of the 10,000 goal),
**total 47,000**, **best day 14,000 (Jun 18)**, **3 of 5 days at-or-over goal**
(Jun 18, 15, 14), a **2-day hit streak** broken by Jun 16/17, and per-day deltas
(`Jun 18 +4,000`, `Jun 17 −3,500`). Extend the fixture across ~30 days for the
chart/heatmap so the period view reads realistically. These are mockups:
**static fixtures that look real are preferred** — do not wire the variants to live
steps services.

## Idioms

Five genuinely distinct compositions of the *same* near-black / periwinkle /
Manrope surface. None re-decides palette, accent, or type — they diverge along
**type scale** (how hard they lean on big tabular figures vs even Manrope weight
for hierarchy), **color logic** (how the periwinkle accent and the success/danger
status colors separate goal-meaning from neutral chrome, given steps has no
discipline hue), and **spacing rhythm** (airy editorial ↔ dense blotter ↔ gridded
calendar). Each makes a **different element the hero** and leans on a different
reference.

- **editorial-average** — Heroes **the average itself**. The avg-daily-steps figure
  becomes the page's **oversized headline** (huge tabular Manrope, the period as a
  small set-in caption — `9,400` over "avg / day · last 30 days"), with the
  goal/best/total demoted to small captioned figures beside or beneath it. The
  chart is a **captioned figure** with the **average drawn as a baseline** read
  against the dashed goal line, so "94% of goal" is something you *see*. The log
  sits below as a quiet ledger. Type scale: dramatic big/small contrast (the one
  number that matters, huge; everything else small). Color logic: near-monochrome
  ink-on-dark, the **single accent reserved for the hero average** and its baseline.
  Spacing rhythm: generous, editorial, the page breathes. Borrowing **The
  Athletic** sports-feature framing. → makes the headline health metric *feel*
  headline.

- **goal-ring-hero** — Heroes **goal attainment**. A **progress ring / arc** leads
  the region (`94%` filling toward `10,000`, average in its center), turning the
  abstract percent into a shape you read in a glance. The chart bars are drawn
  **goal-relative** — each day's fill measured against the goal line as the
  dominant reference, over-goal bars cresting it in the accent/success, under-goal
  falling muted below — and each **log row carries a tiny per-day ring or goal-%**.
  Type scale: the ring's center number is the size anchor; rows stay small. Color
  logic: the accent **fills the ring's progress**; success-green marks the days
  (and the overall) that clear goal. Spacing rhythm: centered, generous around the
  ring. Borrowing **Apple Fitness** rings and **Gentler Streak**. → the most
  visceral "did I hit it" answer.

- **streak-momentum** — Heroes **consistency**. Leans into the design system's
  first-class **streak Voice** ("you hit your goal N of M days," encouraging, never
  scolding). The KPI row is reframed around **current goal-hit streak** and
  **days-hit-this-period** alongside the average; the chart carries a **hit-streak
  ribbon** — consecutive over-goal days joined into a highlighted run, the misses
  visibly breaking it; the log **groups runs of hits/misses** rather than listing
  isolated rows, each row a clear **hit ✓ / miss** against goal. Type scale:
  compact, hierarchy from the ribbon and the streak count, not from one giant
  number. Color logic: success-green = hit, muted slate = miss, accent on the
  active streak. Spacing rhythm: tight, momentum-forward. Borrowing **Whoop**'s
  consistency framing and **Gentler Streak**. → makes *showing up* the story.

- **sparkline-ledger** — Heroes **the log**, the spreadsheet de-gridded into a
  power-user history that *doubles as a chart*. Each row keeps the day and the
  number but gains an **inline mini-bar/sparkline** and a **goal-delta column**
  (`▲ 4,000` accent on a beat, `▼ 3,500` muted on a miss), with **hairline week
  grouping** and a small per-week summary (week avg, days hit). Because the rows
  now *show* motion, the big chart can shrink to a header strip or fold away. Type
  scale: small functional tabular figures, the column does the work. Color logic:
  accent/success spent **entirely on the delta sign and the inline spark**, rows
  otherwise neutral. Spacing rhythm: dense blotter, tight vertical rhythm.
  Borrowing **Robinhood**'s sparkline rows and **Linear**'s earned table density.
  → the answer that fixes *spreadsheet-y* head-on: still a log, but one a power user
  would want.

- **calendar-heat** — Heroes a **calendar grid**. The bar chart becomes a **month
  heatmap** — each day a cell **shaded by goal attainment** (a periwinkle/success
  ramp: pale for a low day, saturated for an over-goal day, an empty outline for an
  unlogged gap) — turning "last 30 days" into a glanceable **habit grid** where
  streaks and gaps are *spatial*. The average and goal sit as a compact header; the
  `Date · Steps` table becomes the **detail view** beneath the grid (tap a cell →
  the row). Type scale: uniform grid labels, one calm header figure. Color logic:
  a **single-hue attainment ramp** (accent or success, intensity = % of goal),
  gaps as hairline-outline cells. Spacing rhythm: even, gridded, calendar-regular.
  Borrowing the **GitHub contribution graph** and **Cardiogram**'s day calendar. →
  reframes steps as a *habit you can see the shape of*, the way a contribution
  graph reads consistency at a glance.

## References

In-system, so "what to take" from each is **structural** — composition, density,
and data legibility, not their palettes or type:

- **The Athletic** (sports data journalism) — take its **authored headline-figure
  layout**: one oversized number with a small caption, the chart as a captioned
  figure. Drives `editorial-average`.
- **Apple Fitness** (activity rings) — take the **goal-as-ring**: an abstract
  percent rendered as a fillable shape you read instantly, with the value in the
  center. Drives `goal-ring-hero`.
- **Gentler Streak** (goal/effort attainment) — take its **goal-relative day
  treatment** and gentle "you're on / off target" framing. Reinforces
  `goal-ring-hero` and `streak-momentum`.
- **Whoop** (consistency / streak readout) — take its **streak-and-consistency
  framing**, the run of days as the headline rather than a single value. Drives
  `streak-momentum`.
- **Robinhood / Linear** (sparkline list rows / dense issue lists) — take the
  **inline per-row sparkline + delta** and **earned table density** that turn a
  list into a glanceable trend. Drives `sparkline-ledger`.
- **GitHub contribution graph / Cardiogram** (day-cell heatmaps) — take the
  **calendar of intensity-shaded cells** where consistency and gaps read
  spatially. Drives `calendar-heat`.

## Selection criteria

A note-to-self for the pick, not a rubric the worker optimizes against. When I
compare these I'm trying to decide:

- **Does average daily steps finally read as the headline** it is — the first
  number my eye lands on, not one of four equal tiles — without the other stats
  (best, total, goal) becoming useless?
- **Does the chart make goal attainment obvious** — over vs under, the near-miss
  average sitting just under 10,000 — and does it survive the one big day (14,000)
  without flattening the rest?
- **Does the log finally stop feeling like a spreadsheet** while staying *fast to
  scan* and *quick to add to*? The `Log steps` action can't get buried for the sake
  of a prettier list.
- **Does consistency come through** — that I hit 3 of these 5 days, that there's a
  streak and a break — given that's what a step-counter actually cares about?
- **Does the no-goal / unlogged-day / empty case degrade gracefully** — no `NaN%`
  ring, no goal line at zero, no 0-step bar where there's just no data?
- **Does it hold up at both breakpoints** and across the period filter — a sparse
  `7 days` and a long `90 days` / `All` both looking intentional?
- **Does it still read as Prog Strength** — native to the v0.4 near-black /
  periwinkle / Manrope system, accent and status colors spent on goal-meaning, no
  invented steps hue — and **sit right beside the shipped Workouts and Running
  tabs**, rather than looking like a different app's dashboard?

---

> **Lifecycle.** `status:` is editorial — the owner is the dispatch gate. It moves
> `draft` → `exploring` (worker running) → `awaiting_selection` (draft PR open,
> owner deciding) → `selected` / `abandoned`. The worker sets `awaiting_selection`
> on the `dx/steps-view` branch as it opens the PR; the owner sets the terminal
> value when they close it.
>
> **Handoff.** This DX ends at a *chosen variant*, not merged code. I open the
> draft `[DX — DO NOT MERGE]` PR, compare the variants on the preview deploy, pick
> one, tick its box, set `status: selected` (noting the winning idiom), and
> **close the PR — never merge it.** Then I open a SOW: *"implement the steps view
> per the `<chosen-idiom>` variant from `dx/steps-view`, production-quality,
> conforming to the design system."*
