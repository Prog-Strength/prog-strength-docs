---
type: dx
status: awaiting_selection
surface: activities-page
idioms:
  - editorial-data-journalism
  - whoop-recovery-dense
  - brutalist-terminal
  - strava-vibrant
  - oura-calm-minimal
references:
  - The Athletic
  - Whoop
  - Bloomberg Terminal
  - Strava
  - Oura
scope: greenfield
variant_count: 5
repos:
  - prog-strength-web
  - prog-strength-docs
---

# DX: Activities Page

**Status**: Awaiting selection · **Last updated**: 2026-06-18

> A DX (Design Exploration) is the platform's **divergent** work type. Unlike a
> SOW it does not converge on one correct implementation — it produces N
> differentiated visual variants of a single frontend surface, side by side on
> one comparison route, awaiting a human pick. It **never merges** and ships no
> production code; the chosen direction feeds a downstream SOW that builds it for
> real.

## Context

The Activities page (`/activities`) is the analytics home of Prog Strength — the
surface a user opens to answer "how's my training going?" across all three things
the app tracks: **lifting**, **running**, and **steps**. It's a period-scoped
dashboard (`Last 7 days · Last 30 days · Last 90 days · All`) with four tabs —
**Overview**, **Workouts**, **Running**, **Steps** — each its own stat row, trend
chart, and history list.

The **functionality is solid and not in question here**: the aggregates compute,
the charts plot, the per-week/per-month grouping works, and the per-tab actions
(Start live workout, Upload TCX, Log steps) all do their jobs. What this DX
questions is the **visual direction itself**. The current page is competent and
on-brand, but it reads like a generic dark-mode admin dashboard — rows of
identical glassy stat cards, two area charts that look the same whether they're
showing hours or miles, and three tabs that feel like three separate mini-apps
rather than one coherent picture of a training week. It doesn't yet have a point
of view, and for the page that's supposed to make a lifter *feel* their progress,
neutral competence is the thing to beat.

This DX exists to find that point of view before committing engineering effort.
**`scope: greenfield`**: unlike most of our surfaces, the variants here are *free
to leave the current system* — they may re-decide the palette, the type family
and scale, the accent, and the chart language. Only the **Fixed Points** hold:
the **"P" brand mark** and the **dark theme**. The goal is genuine divergence —
five legitimately different answers to "what should training data feel like" —
not five re-skins of the same dashboard. No production activities code changes
here; the chosen direction feeds a downstream SOW.

## The surface

The full tabbed `/activities` page. **Every variant renders all four tabs** at
full fidelity — the tabs share one idiom, and seeing lifting, running, and steps
all wearing it is the whole point (the surface's hardest job is making three
different data domains feel like one product). A persistent **period filter**
(`7 / 30 / 90 / All`, default `30`) and the **tab bar** are page chrome a variant
may restyle or relocate but must keep functional-looking.

What lives on each tab (a variant should account for all of it, though it may
reorganize, demote, or fold chrome together in service of its idiom):

**Overview** — the cross-domain summary.
- A **stat row**: `Total time`, `Total sessions`, `Workouts`, `Runs`.
- A **Weekly Activity chart**: the signature element — a **dual-series** area/line
  plot of **Lifting** and **Running** hours per week across the period, with a
  hover tooltip (`Jun 1 – 7 · Lifting 4h 48m · Running 37m`). Its hardest job is
  keeping two overlaid series legible.
- A second **stat row**: `Total volume`, `Total mileage`, `PRs`, `Avg session`.
- A **steps summary**: `Avg daily steps` and `Steps goal` (% of goal).

**Workouts** ("lifting") — `▷ Start live workout` action.
- A **summary card**: `Total time · Sessions · PRs`, a **metric toggle**
  (time / volume / per-muscle), and a trend **area chart** over the period.
- **Per-week groups** (`This week · Jun 15 – 21`, `Last week · Jun 8 – 14`, …),
  each with a `Total time` + `Activities` mini-header and a list of **session
  cards**: name, a 🏆 badge when the session set a PR, a meta line
  (`Yesterday, 4:45 PM · 1h 30m · 6 exercises · 29,090 lb Total Volume`), a
  truncated **note**, an expand chevron, and edit/delete affordances.

**Running** — `⬆ Upload TCX` action.
- A **KPI row**: `This week` (with `+18% vs last week` delta), `This month · N runs`,
  `Avg pace (30D)`, `All time · N runs`.
- A **summary card**: `Total distance · Total time · Runs`, a **distance/time
  toggle**, and a trend **area chart**.
- **Per-month → per-week groups** (`June 2026 · 11.7 mi · 1h 55m · 3 runs`, then
  `This week`, `Last week`, …) with **run cards**: name, datetime, and a metric
  rail (`Distance · Time · Pace · Avg HR`).

**Steps** — `✎ Log steps` action.
- A **KPI row**: `Avg daily steps`, `Total steps`, `Best day` (with its date),
  `Goal` (% of goal).
- A **Daily steps bar chart** with a **dashed goal line** at the target;
  above/below-goal bars should read differently.
- An editable **log table**: `Date · Steps` rows with edit/delete.

The data shapes (from `prog-strength-web/lib/api.ts`):

```ts
// Lifting. Total Volume and duration are derived, not stored:
// volume = Σ(set.reps × set.weight); duration = ended_at − performed_at.
type WorkoutSet = { reps: number; weight: number; unit: "lb" | "kg" };
type WorkoutExercise = { exercise_id: string; order: number; sets: WorkoutSet[]; notes?: string };
type Workout = {
  id: string; name?: string;
  performed_at: string; ended_at?: string | null; // RFC3339
  notes?: string;
  exercises: WorkoutExercise[];
  personal_records_set: PersonalRecordEvent[];     // non-empty ⇒ 🏆 badge
};

// Running. Stored in SI; the UI converts to mi / min·per·mi / bpm.
type RunningSession = {
  id: string; name: string | null; start_time: string; // RFC3339
  distance_meters: number; duration_seconds: number;
  avg_pace_sec_per_km: number | null; avg_heart_rate_bpm: number | null;
};
type RunningMetrics = {
  current_week:  { distance_meters: number; run_count: number; delta_pct_vs_prior_week: number | null };
  current_month: { distance_meters: number; run_count: number };
  recent_avg_pace_sec_per_km: number | null;
  all_time:      { distance_meters: number; run_count: number };
};

// Steps.
type StepsEntry = { id: string; date: string /* YYYY-MM-DD */; steps: number };
type StepsGoal  = { goal: number };
```

Overview's numbers are **cross-domain rollups** of the above over the active
period — total time and session count sum lifting + running; total volume comes
from the workouts, total mileage from the runs, steps from the step entries.

**Visual states the variants must handle** (this is where lazy designs fall
apart, so render them all in the mockup):

- **A zero/rest week.** Both Overview series dip to **0h** around `May 25` — the
  dual chart, and the empty week groups beneath each tab, must read as
  *intentional rest*, not broken or missing data.
- **Two overlaid series staying legible.** Lifting and Running cross and diverge
  on the same axes (hours); whatever color/encoding language a variant invents has
  to keep both readable, including where they overlap.
- **Over vs under goal (steps).** `Jun 14` is **over** (11,000 vs 10,000 goal),
  `Jun 16/17` are **under** (5,500 / 6,500). Over-goal bars should feel like a win;
  the dashed goal line is the reference.
- **A delta that can be positive or negative.** Running's `+18% vs last week` is
  green and up; the same tile must also handle a *down* week (a negative delta) and
  a *no-prior-week* null (hide it).
- **PR badges and long names + notes.** Sessions like `W5 D1 - Chest & Back` and
  `Recovery Week Lift 2 - Shoulders & Arms` carry 🏆 badges and free-text notes
  that **truncate** ("…Not sure why though to be honest…"); names and notes can't
  assume short strings.
- **Mixed density across the period filter.** `7 days` may show a single sparse
  week; `All` shows many weeks/months. The same layout has to survive both — a
  near-empty list and a long scrolling history.
- **The tab handoff.** A variant is judged on whether Overview → Workouts →
  Running → Steps feels like **one surface** — consistent stat-row, chart, and
  list language — rather than three dashboards sharing a tab bar.

**Representative fixture** (the `Last 30 days` view from the screenshots — use
something like it so variants render realistically: a real training month with a
rest-week trough, PRs, an over- and an under-goal step day):

- **Overview** — Total time **14h 1m** · 13 sessions · 8 workouts · 5 runs;
  Total volume **150,875 lb** · Total mileage **22.3 mi** · **7 PRs** · Avg
  session 1h 5m; Avg daily steps **8,250** · Steps goal **83% of 10,000**. Weekly
  chart peaks the week of `Jun 1 – 7` (Lifting 4h 48m, Running 37m) and bottoms
  near `May 25` (both ~0h).
- **Workouts** — Total time **10h 32m** · 8 sessions · 7 PRs.
  *This week (Jun 15 – 21)* — 1h 30m · 1 activity: **W5 D1 - Chest & Back** 🏆 ·
  Yesterday, 4:45 PM · 1h 30m · 6 exercises · **29,090 lb** · note "Bench
  exercises felt great but my bent over row was struggling today…".
  *Last week (Jun 8 – 14)* — 2h 32m · 2 activities: **Recovery Week Lift 2 -
  Shoulders & Arms** 🏆 · Saturday, 12:23 PM · 1h 17m · 6 exercises · **17,875 lb**.
- **Running** — This week **4.2 mi** (+18% vs last week) · This month 11.7 mi · 3
  runs · Avg pace **9:22 /mi** (30D) · All time **116.3 mi · 25 runs**. Card
  rollup: Total distance 22.3 mi · Total time 3h 29m · 5 runs.
  *June 2026 · 11.7 mi · 1h 55m · 3 runs* → *This week (Jun 15 – 21)* — 4.2 mi ·
  43m · 1 run: **W7 D1 - Easy Run** · Mon, Jun 15, 2026 · 1:39 PM · 4.2 mi ·
  42:51 · **10:07 /mi** · **153 bpm**.
- **Steps** — Avg 8,250 · Total 33,000 · Best day **11,000 (Jun 14)** · Goal
  10,000 (83%). Daily bars: Sun Jun 14 **11,000** (over), Mon Jun 15 10,000 (at),
  Tue Jun 16 5,500 (under), Wed Jun 17 6,500 (under).

These are mockups: **static fixtures that look real are preferred** — do not wire
the variants to live activity services.

## Idioms

Five genuinely distinct answers to "what should training data feel like." Because
`scope: greenfield`, each idiom is **free to bring its own palette, type family
and scale, accent, and chart language** — they diverge along **type scale**,
**color logic**, and **spacing rhythm**, and here also along the *overall visual
world* each inhabits. The only constraints are the **Fixed Points**: the **"P"**
mark and the **dark theme** survive in every variant. Each leans hard on a
different reference and makes a different element the hero.

- **editorial-data-journalism** — Training as a **data story**. Oversized display
  numerals (a strong serif or a confident grotesk) sit in an airy, column-based
  layout with real whitespace and captioned charts, like a sports-analytics
  feature in **The Athletic / FiveThirtyEight**. Type scale: dramatic — huge
  headline figures, small set-in captions and labels. Color logic: near-monochrome
  ink-on-dark with **one restrained accent** reserved for the single number that
  matters most on each tab; the dual chart uses one solid + one outlined series
  rather than two competing fills. Spacing rhythm: generous, editorial, the page
  breathes. → makes the numbers feel *authored* and significant.

- **whoop-recovery-dense** — Training as a **readiness readout**. A single signal
  color on near-black charcoal, **ring- and gauge-forward**, with tight
  metric-label-value triplets packed into a high-density grid — the **Whoop**
  today screen applied to a lifter. Type scale: compact and uniform; hierarchy
  from the gauges and the one signal color, not from size. Color logic: one
  continuous accent (a recovery-style gradient) carries every metric — goal
  attainment, PR streaks, the week's load — against an otherwise colorless field.
  Spacing rhythm: tight, gridded, maximally informative. → the data-dense,
  glanceable, "where am I right now" end of the spread.

- **brutalist-terminal** — Training as a **ledger**. Monospace tabular numerals,
  visible hairline grid, **alignment as the only hierarchy** — a Bloomberg-style
  terminal where the figures line up in columns and nothing is decorated. Type
  scale: uniform mono, no display sizing; the grid does the work. Color logic:
  a **terminal palette** (amber/green on black), color used only as *state*
  (up/down, hit/missed, PR) the way a trading screen colors ticks — never
  decoration. Spacing rhythm: dense rows, hairline dividers, no rounded cards. →
  the power-user, information-maximal, anti-dashboard extreme.

- **strava-vibrant** — Training as **energy**. Full-bleed gradient hero stats,
  bold rounded cards, and **vibrant dual-tone charts** that look alive — the
  kinetic, celebratory language of **Strava / Nike Run Club**. Type scale: punchy
  bold sans, big confident numbers, strong contrast. Color logic: **energetic and
  multi-accent** — each domain (lift / run / steps) gets its own vivid hue, the
  Weekly Activity chart leans into the two-tone fill, PRs and goal-hits get
  celebratory color. Spacing rhythm: rhythmic, card-forward, comfortable. → the
  loudest, most motivating, most "share it" direction.

- **oura-calm-minimal** — Training as **a calm instrument**. Desaturated,
  low-contrast palette, fine precise type with tight tracking, soft gradients, and
  lots of negative space — the quiet, premium restraint of **Oura / Linear**. Type
  scale: small, exact, even; nothing shouts. Color logic: muted and structural —
  thin lines and soft fills, a single desaturated accent, the dual chart rendered
  as two delicate lines rather than filled areas. Spacing rhythm: generous and
  perfectly even, a consistent quiet grid. → the understated, premium, "let the
  data sit still" end — the deliberate opposite of strava-vibrant.

## References

Greenfield, so "what to take" includes the **palette, type, and chart language**,
not only structure:

- **The Athletic** (sports data journalism) — take its **authored data-story
  layout**: oversized figures, captioned charts, column whitespace, one accent
  used sparingly. Drives `editorial-data-journalism`.
- **Whoop** (today / recovery screen) — take its **single-signal-color gauge
  density** and the first-half-second "here's where you are" readout. Drives
  `whoop-recovery-dense`.
- **Bloomberg Terminal** (market screen) — take its **monospace tabular grid**,
  alignment-as-hierarchy, and **color-as-state-only** discipline. Drives
  `brutalist-terminal`.
- **Strava / Nike Run Club** (activity feed) — take its **vibrant multi-accent
  energy**, gradient heroes, and two-tone activity charts. Drives `strava-vibrant`.
- **Oura / Linear** (calm instrument / minimal product) — take its **desaturated
  low-contrast restraint**, fine type, and delicate line charts over filled areas.
  Drives `oura-calm-minimal`.

## Selection criteria

A note-to-self for the pick, not a rubric the worker optimizes against. When I
compare these I'm trying to decide:

- **Does it make a training week feel like *progress*, not admin?** This is the
  page that should make me want to keep going. Neutral competence is what I'm
  trying to beat — I want the one with an actual point of view.
- **Does the dual Lifting/Running chart stay legible** — both series readable
  where they cross and where one goes to zero — and does the chart language feel
  *designed*, not default?
- **Do the three domains feel like one product?** Lifting, running, and steps
  wearing the same idiom should cohere — Overview should feel like the same surface
  as Steps, not a different app per tab.
- **Is goal/PR state unmistakable?** The over-goal step day, the `+18%` delta
  (and its negative twin), the 🏆 PR badge — the wins should *land* at a glance.
- **Does it survive both sparse and dense?** A single rest week on `7 days` and a
  long scroll on `All` should both look intentional.
- **Does it still read as Prog Strength** — the "P" and the dark theme anchor it —
  while genuinely *leaving* the generic-dashboard look I'm trying to escape?

---

> **Lifecycle.** `status:` is editorial — the owner is the dispatch gate. It moves
> `draft` → `exploring` (worker running) → `awaiting_selection` (draft PR open,
> owner deciding) → `selected` / `abandoned`. The worker sets `awaiting_selection`
> on the `dx/activities-page` branch as it opens the PR; the owner sets the
> terminal value when they close it.
>
> **Handoff.** This DX ends at a *chosen variant*, not merged code. I open the
> draft `[DX — DO NOT MERGE]` PR, compare the variants on the preview deploy, pick
> one, tick its box, set `status: selected` (noting the winning idiom), and
> **close the PR — never merge it.** Then I open a SOW: *"implement activities-page
> per the `<chosen-idiom>` variant from `dx/activities-page`, production-quality."*
