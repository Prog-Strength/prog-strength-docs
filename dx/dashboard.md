---
type: dx
status: selected
surface: dashboard
idioms:
  - focus-grid
  - hero-summary
  - discipline-columns
  - feed-digest
  - command-center
references:
  - Garmin Connect
  - Strava
  - Oura
  - Whoop
  - Linear
scope: in-system
variant_count: 5
repos:
  - prog-strength-web
  - prog-strength-docs
---

# DX: Dashboard

**Status**: Selected (`command-center`) · **Last updated**: 2026-06-19

> A DX (Design Exploration) is the platform's **divergent** work type. Unlike a
> SOW it does not converge on one correct implementation — it produces N
> differentiated visual variants of a single frontend surface, side by side on
> one comparison route, awaiting a human pick. It **never merges** and ships no
> production code; the chosen direction feeds a downstream SOW that builds it for
> real.

## Context

Prog Strength has **no top-level home**. After login the app drops the user
straight into `/chat`; everything else — training, running, steps, nutrition,
bodyweight, progress, records — lives behind its own sidebar tab, each a deep,
single-domain surface. There is **no one place that answers "how am I doing across
everything?"** the moment a user opens the app. Every comparable product has one:
Garmin's **In Focus** card grid, Strava's **dashboard** with its profile/streak
rail and activity feed. Prog Strength's executive overview is the gap.

This DX explores a **brand-new Dashboard** — the surface a user **lands on by
default at login**, a new sidebar selection, and a glanceable roll-up of the
top-level metrics across **running, lifting, steps, nutrition, bodyweight**, plus
cross-cutting signals like the **activity streak**. It is explicitly a **first
iteration**: a confident, well-composed v1 overview that surfaces the headline
number per domain and **links into** the existing deep pages for the detail — not a
reimplementation of them. It also carries one new interaction: a **top-level chat
bar** so a user can, on login, immediately ask the agent whatever
lifting/running/nutrition question is on their mind, and be handed into the chat
view to continue.

**An honesty note that bounds this whole DX — read it before designing.** The
reference dashboards lead with metrics Prog Strength **does not have**: Garmin's
*Training Readiness*, *HRV Status*, *Sleep*, *Stress*, *Training Status* and
Strava's *relative effort* all come from **wearable biometrics Prog Strength never
collects**. So Garmin and Strava are **compositional references only** — take their
*executive card grid*, *per-discipline week-summary cards*, *streak rail*, *muscle
heatmap*, and *feed rhythm* — **not their metric set.** A variant that mocks a
"readiness ring" or a "sleep score" is inventing data and fails this DX. Everything
on the dashboard must trace to a real Prog Strength metric (see **The metrics**).

`scope: in-system`: the foundation is decided (see
[`../design-system.md`](../design-system.md) **v0.4**, oura-calm) — soft near-black
ramp, the single **periwinkle** accent (`#9aa6d6`, app-chrome only), the discipline
hues (run green-teal `--discipline-run-*`, lift steel-blue `--discipline-lift-*`),
**Manrope** with tight numeric tracking, 14px hairline panels, desaturated
success/danger. Variants do **not** re-litigate palette, accent, or type — they
diverge on **layout, structure, density, and composition**: what the page's hero
is, how the per-domain metrics are arranged and ranked, how the streak and the
chat bar are placed, and how a glance answers "how am I doing?" Because this is the
default landing surface, it must read as the **calm front door** to the app, not a
busy console — and it must sit beside the existing tabs as the same v0.4 product.

## The surface

A **new** authenticated route `app/(app)/dashboard/page.tsx`, a **new sidebar
entry**, and the **default post-login landing**. The following are **fixed
structural facts** (not what the variants explore — they explore the *visual
composition* of the overview and the placement of the chat bar):

- **New sidebar selection** — a `Dashboard` item is added to the nav
  (`components/sidebar.tsx`, the inline-SVG `NAV` array), positioned first as the
  primary overview.
- **Default landing** — authenticated users land on `/dashboard` instead of
  `/chat` (the redirect lives in three places today: `app/page.tsx`,
  `app/login/page.tsx`, `app/auth/callback/page.tsx`).
- **Top-level chat bar** — a prompt input on the dashboard; on submit it routes to
  the existing chat view via **`/chat?prompt=<encoded>`**, where the chat page
  pre-fills its composer with the prompt (the chat page already resumes via
  `?session=`, creates a session lazily on first send, and streams from the agent —
  so the handoff is a one-line addition: read `?prompt=` into the input). The
  dashboard **does not** call the agent itself; it hands off and the chat session
  continues there. Variants explore **where and how** this bar sits (a hero
  prompt, a persistent command bar, a card), not the handoff plumbing.

What a variant **owns and composes** (the metric overview), per domain — each
should also be a **link into** its deep page:

- **Activity streak** — a cross-cutting "you've been active N days/weeks" signal,
  the kind of habit cue Strava heroes. **Derived client-side** (no endpoint).
- **Running** — this-week distance · run count · **Δ% vs last week** · recent avg
  pace (one fetch), with all-time / best-effort as optional supporting figures.
- **Lifting** — this-week volume · sessions · sets · **PRs this week**, with
  headline estimated-1RM and the muscle-group set distribution (the "muscle map")
  as optional supporting visuals.
- **Steps** — average vs goal (attainment %), today's count.
- **Nutrition** — today's calories + macros **vs goals** (adherence).
- **Bodyweight** — current weight + **weekly trend rate**, vs goal. *(A metric
  lifters and runners both track — first-class on the dashboard.)*
- **Upcoming / plan** — the next planned workout and recent plan adherence
  (optional for v1).

The data layer (`prog-strength-web/lib/api.ts`) — **no single dashboard endpoint
exists**; the page **fans out parallel fetches** (the precedent is
`components/activities/activities-overview-view.tsx`). Each card should render
**independently as its fetch resolves**, so one slow domain never blocks the page:

```ts
// Running — server-bucketed week/month in the user's timezone.
getRunningMetrics(token, timezone) → {
  current_week: { distance_meters, run_count, delta_pct_vs_prior_week | null },
  current_month: { distance_meters, run_count },
  recent_avg_pace_sec_per_km | null,
  all_time: { distance_meters, run_count },
}
listRunningBestEfforts(token) → RunningBestEffort[]   // optional supporting

// Lifting — windowed fetch + client aggregation (no summary endpoint).
listWorkouts(token, { since, until }) → { items: Workout[], total, has_more }
//   volume via workoutVolume(workout, unit); sessions = items.length;
//   PRs this week = Σ items[].personal_records_set.length
listPersonalRecords(token) → PersonalRecord[]          // headline est-1RM
setsByCategory(workouts, exercises)                    // the muscle map (client lib)

// Steps — client aggregation over the window.
listSteps(token, { since, until }) → { steps: StepsEntry[] }
getStepsGoal(token) → { goal }                          // 0/unset ⇒ no goal

// Nutrition — per-day totals vs goals.
getDailyMacros(token, { timezone, startDate, endDate }) → DailyMacros[]  // {date,calories,protein_g,carbs_g,fat_g}
getMacroGoals(token) → { calories, protein_g, carbs_g, fat_g }           // unset ⇒ no goal

// Bodyweight — latest + trend (client helpers in lib/bodyweight-trend.ts).
listBodyweight(token, { since, until }) → BodyweightEntry[]
getBodyweightGoal(token) → { weight, unit }            // unset ⇒ no goal
//   trendSummary(ewmaTrend(buildDayPoints(...))) → { current, ratePerWeek }

// Plan — optional.
listPlannedWorkouts(token, { since, until }) → PlannedWorkout[]

// Streak — DERIVED client-side: union of activity dates across
//   workouts (performed_at), runs (start_time), steps (date),
//   de-duped by local calendar day, walk back from today.
```

**Timezone** is real here: running metrics and nutrition take the IANA timezone
(from `getMe().timezone`); the date-windowed lists must be built in the user's
local day, not UTC (the app's `internal/daterange` convention).

**Visual states the variants must handle** (render them all in the mockup — a
landing page lives or dies on these):

- **The brand-new / sparse user.** No runs, no lifts, no steps, no nutrition, no
  bodyweight — the **first thing a new user ever sees**. Every card needs an
  **inviting empty state** ("Log your first run", "Set a step goal") — never a
  broken `0` or `NaN%`. This is the make-or-break state for a default landing page.
- **Partial data.** Lifts but no runs; steps but no goal; nutrition logged but no
  macro goal set — **each card degrades independently**, the page still feels
  whole.
- **No goals set.** Steps / macro / bodyweight goals null → attainment treatments
  degrade to a "set a goal" affordance, not a goal line at zero.
- **Zero vs active streak.** A 0-day streak (don't shame it) and a long active
  streak both read intentionally; define what counts as an "active day."
- **Loading.** Many parallel fetches — **per-card skeletons**, not one page-blocking
  spinner; a slow domain shows a skeleton while its neighbors render.
- **The chat bar as the primary CTA.** On login it should be **obvious and
  inviting** — the clearest "ask me anything" affordance — without turning the
  calm overview into a chatbot landing page.
- **Both breakpoints.** Multi-column compositions must reflow to a clean single
  column on mobile; the chat bar stays reachable.
- **No biometrics.** Reiterated: no readiness/HRV/sleep/stress mockups — only real
  Prog Strength metrics.

**Representative fixture** (an established multi-sport user, "this week"):

- **Streak**: 25 weeks active (Strava-style), 3 active days this week.
- **Running**: this week **13.18 mi** across 3 runs, **+9% vs last week**, recent
  avg pace `10:06 /mi`; latest run *Lunch Run · 5.25 mi · 53:04*.
- **Lifting**: this week **2:43 total**, 3 sessions, 21 sets, **4 PRs** (the
  `Week 2 Workout 1` chest & back day); headline est-1RM bench `326.9`.
- **Steps**: avg **9,400 / 10,000** goal (94%), today 14,000.
- **Nutrition**: today **1,840 / 2,100 kcal**, protein `150 / 180 g` — under on
  protein.
- **Bodyweight**: **182.4 lb**, trending **−0.3 lb/wk** toward a 178 goal.
- **Plus a second, brand-new-user fixture** (everything empty) so every variant is
  proven on the day-one state, and a **partial** fixture (lifts + steps only) — the
  happy path is the easy one; the empty and partial states are where a dashboard
  falls apart.

These are mockups: **static fixtures that look real are preferred** — do not wire
the variants to live services. Build all three fixtures (full / empty / partial),
because a dashboard's whole job is to compose gracefully across exactly that range.

## Idioms

Five genuinely distinct **compositions** of the *same* v0.4 surface, each making a
**different element the hero**, placing the **chat bar** differently, and leaning on
a different reference. None re-decides palette, accent, or type — they diverge on
**layout** (mosaic ↔ columns ↔ feed ↔ dense strip), **what's promoted** (a single
status line vs per-domain parity vs raw density), and **spacing rhythm** (airy front
door ↔ tight command center).

- **focus-grid** — Garmin's **In Focus** answer. A responsive **grid of equal-weight
  metric cards** — one per domain (Running this week, Lifting this week, Steps,
  Nutrition, Bodyweight, Streak) — each a self-contained glance (headline figure, a
  tiny trend, a link in), with the **chat bar as a prominent hero strip across the
  top**. No single domain wins; the **mosaic** is the hero, and it's the most
  legible "everything at once." Each card carries its own empty state so a sparse
  user sees an inviting board, not gaps. Spacing: generous, card-forward. → the
  safest, most familiar executive overview. (Garmin Connect *In Focus*.)

- **hero-summary** — An **executive top-line**. One consolidated hero leads — the
  **activity streak + a "this week" headline** (the single "how am I doing" read,
  Oura/Whoop-style) with the **chat bar woven into the hero** — then the per-domain
  metrics sit beneath as a calmer secondary tier. Big type up top, demoted detail
  below. The hero is a *status*, not a grid. Degrades when there's little data by
  leaning the hero on the streak + the chat invitation. → the calmest front door;
  answers the one question before showing the six numbers. (Oura / Whoop overview.)

- **discipline-columns** — **Parity across disciplines.** The page organizes into
  **columns/sections by discipline** — a Running column, a Lifting column, a
  Health column (steps · nutrition · bodyweight) — each with its headline metric and
  a mini-trend, the **chat bar spanning the top** above all columns. Hierarchy from
  **side-by-side structure**, not one hero number; the runner-who-also-lifts sees
  both worlds at once. Reflows to stacked sections on mobile. → for the multi-sport
  user who wants each discipline to have its own home on one screen. (Strava's
  multi-pane dashboard.)

- **feed-digest** — A **prioritized vertical digest**, mobile-first. The **chat bar
  leads**, then a single-column, **relevance-ordered stack**: streak → today's
  activity → this week's running → this week's lifting → nutrition today →
  bodyweight trend — each a compact row/card that *says what matters* and links in.
  Editorial rhythm, most-important-first, reads top-to-bottom like a morning
  briefing. The empty state becomes a friendly "here's how to start" checklist. →
  the "open the app, scroll once, you're caught up" answer; best on phones.
  (Strava feed / The Athletic digest.)

- **command-center** — The **dense power-user console**. A compact **KPI strip**
  across the top puts every domain's headline number in one scannable row; the
  **chat bar is a command bar**; beneath, tight **mini-cards with sparklines** pack
  maximum signal into minimum space. Small functional type, very tight rhythm,
  status-encoded color. The deliberate-density opposite of feed-digest, and the one
  that shows the most without scrolling. Must prove it stays *calm* and on-brand at
  this density rather than reading as a trading terminal. → for the user who wants
  every number at a glance. (Linear's earned density / a trading dashboard.)

## References

In-system, so "what to take" from each is **structural** — composition, density,
and information design, **not** their palettes, type, **or their biometric metric
set**:

- **Garmin Connect** (*In Focus*) — take its **executive card grid** and
  **per-discipline week-summary cards** (distance/time with a mini week chart);
  leave its readiness/HRV/sleep metrics behind. Drives `focus-grid`.
- **Strava** (dashboard) — take its **profile/streak rail**, the **streak ribbon**,
  the **muscle heatmap**, and the **activity-feed rhythm**; leave relative effort.
  Drives `discipline-columns` and `feed-digest`.
- **Oura / Whoop** (daily overview) — take the **single consolidated "how am I
  doing" hero** that leads with one status before the detail. Drives `hero-summary`.
- **Linear** (dense product surfaces) — take its **earned density**: tabular
  alignment, hairline grouping, single-accent restraint at high information density.
  Drives `command-center`.
- **The Athletic** (digest) — take its **prioritized, most-important-first** feed
  rhythm. Reinforces `feed-digest`.

## Selection criteria

A note-to-self for the pick, not a rubric the worker optimizes against. When I
compare these I'm trying to decide:

- **Does it answer "how am I doing across everything?" in one glance** — running,
  lifting, steps, nutrition, bodyweight, and the streak — without making me hunt?
- **Is it the right front door?** It's the *default landing surface* — does it feel
  like a calm, inviting home, or a busy console I'd want to skip past?
- **Is the chat bar an obvious, inviting first move** — "ask the agent anything" —
  and does the hand-off into the chat view read as natural?
- **Does the brand-new / sparse user get an inviting page**, not a wall of `0`s and
  empty frames — given that's the first thing they ever see?
- **Does every metric trace to real Prog Strength data** — no invented
  readiness/HRV/sleep — and does each card link into its deep page?
- **Does it hold up across full / partial / empty data and both breakpoints**, with
  per-card loading so one slow domain doesn't block the page?
- **Does it still read as Prog Strength v0.4** — near-black, periwinkle as
  app-chrome, the run/lift discipline hues used with discipline — and sit beside the
  existing tabs as the same product, not a different app's dashboard?

---

> **Lifecycle.** `status:` is editorial — the owner is the dispatch gate. It moves
> `draft` → `exploring` (worker running) → `awaiting_selection` (draft PR open,
> owner deciding) → `selected` / `abandoned`. The worker sets `awaiting_selection`
> on the `dx/dashboard` branch as it opens the PR; the owner sets the terminal value
> when they close it.
>
> **Handoff.** This DX ends at a *chosen variant*, not merged code. I open the draft
> `[DX — DO NOT MERGE]` PR, compare the variants on the preview deploy
> (`/design-explore/dashboard`, flag-gated behind `NEXT_PUBLIC_ENABLE_DESIGN_EXPLORE`,
> driven across the full / partial / empty fixtures), pick one, tick its box, set
> `status: selected` (noting the winning idiom), and **close the PR — never merge
> it.** Then I open a SOW: *"implement the dashboard per the `<chosen-idiom>` variant
> from `dx/dashboard`, production-quality, conforming to the design system"* — which
> also wires the new route, the sidebar entry, the default-landing redirect, the
> parallel data fan-out, and the `/chat?prompt=` hand-off. The mockup code is never
> promoted as-is.
