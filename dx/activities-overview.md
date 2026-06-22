---
type: dx
status: selected
surface: activities-overview
idioms:
  - headline-digest
  - discipline-columns
  - command-strip-dense
  - consistency-heatmap
  - instrument-panel
references:
  - The Athletic
  - Garmin Connect
  - Whoop
  - Oura
  - Strava
scope: in-system
variant_count: 5
repos:
  - prog-strength-web
  - prog-strength-docs
---

# DX: Activities Overview

**Status**: Selected (`instrument-panel`) · **Last updated**: 2026-06-22

> A DX (Design Exploration) is the platform's **divergent** work type. Unlike a
> SOW it does not converge on one correct implementation — it produces N
> differentiated visual variants of a single frontend surface, side by side on
> one comparison route, awaiting a human pick. It **never merges** and ships no
> production code; the chosen direction feeds a downstream SOW that builds it for
> real.

## Context

The Activities page (`/activities`) is the analytics home of Prog Strength, and
its **Overview** tab is the first thing a user lands on — the cross-domain answer
to "how's my training going?" *before* they drill into Workouts, Running, or
Steps. The page-wide visual language was settled by the earlier
[`dx/activities-page`](activities-page.md) (selected `oura-calm-minimal`) and
shipped in [`sows/activities-page-redesign`](../sows/activities-page-redesign.md).
**That language is not in question here.** What this DX questions is the
**Overview tab's information architecture** — *what it surfaces and how it
composes* — which the earlier exploration only sketched as one stat-row-chart-
stat-row template among four tabs.

Today the Overview underdelivers for a page that is meant to summarize *all*
activity. It's three near-identical rows of stat tiles bracketing a single
weekly line chart. Several of the tiles are weak for this surface: **Total
volume** and **PRs** are lifting-detail metrics that already lead the Workouts
tab and mean nothing next to a run; **Steps goal %** is a flat percentage that
hides the actual trend. Running — half the user's sessions — gets a single
`Runs` count and a `Total mileage` tile, with **no pace, no effort, no trend**.
And the page shows *no sense of consistency*: nothing answers "did I show up
this month?", which is the emotional core of an activity overview. The result
reads like a totals receipt, not a picture of a training block.

This DX exists to find a richer, more visual Overview before committing
engineering effort. **`scope: in-system`**: every variant **uses the
design-system tokens** — the soft near-black ramp, the periwinkle app accent, the
`run`/`lift` discipline hues, Manrope with tight numeric tracking, and the calm
1px-line chart language. Variants **do not re-skin**; they diverge on
**composition, density, which metrics lead, and what visuals appear**. No
production Overview code changes here; the chosen direction feeds a downstream
SOW.

## The surface

The **Overview tab only** (`/activities?view=overview` →
`components/activities/activities-overview-view.tsx`). The persistent **period
filter** (`7 / 30 / 90 / All`, default `30`) and the **tab bar** are page chrome
a variant may restyle or relocate but must keep functional-looking; the other
three tabs are out of scope and unchanged. The Overview owns one pair of
client-side fetches (workouts + running sessions + steps + steps goal) and
computes every number as a rollup over the active window — no new endpoints are
assumed.

**Decided content edits (constraints every variant inherits — not up for
re-exploration):**

- **Remove `Total volume`** — a lifting-only figure that leads the Workouts tab;
  it doesn't belong on a cross-domain summary.
- **Remove `PRs`** — also lifting-only and already badged on the Workouts tab.
- **Remove `Steps goal %`** — a flat percentage; replace its job with an actual
  steps *trend* (a sparkline / mini-bars) wherever steps appear.
- **Add `Avg run pace`** — the running dimension the Overview is missing.

Beyond those, the variants are asked to **explore richer running and consistency
data** that the API already exposes but the Overview ignores:

- **Pace & distance trends** — `avg pace` over the window plus a weekly **mileage
  trend**, so running has visual presence beyond one count.
- **Discipline time-split** — share of *time* across **lifting vs running** (a
  donut, a split bar, or a stacked series), answering "where did my effort go".
- **Consistency / streak** — a **calendar heatmap of days active** (any modality)
  with a "trained N of M days" / current-streak readout. This is the strongest
  "feel your progress" move and the page's biggest current gap.
- **Heart rate & effort** — **avg/max HR** (on runs with TCX enrichment) and
  **elevation gain**, adding an intensity dimension the page lacks today.

The data shapes (from `prog-strength-web/lib/api.ts` — all already fetched by the
Overview; durations and rollups are derived, not stored):

```ts
// Lifting. duration = ended_at − performed_at (completed sessions only).
type Workout = {
  id: string; name?: string;
  performed_at: string; ended_at?: string | null;   // RFC3339; null ⇒ in-progress
  exercises: WorkoutExercise[];
  personal_records_set: PersonalRecordEvent[];
};

// Running. Stored in SI; the UI converts to mi / min·per·mi / bpm / ft.
// avg_pace and HR are null on manually-logged runs (no TCX) — variants
// must render "— /mi" / "— bpm" without breaking, not assume a number.
type RunningSession = {
  id: string; name: string | null; start_time: string;   // RFC3339
  distance_meters: number; duration_seconds: number;
  avg_pace_sec_per_km: number | null; best_pace_sec_per_km: number | null;
  avg_heart_rate_bpm: number | null;  max_heart_rate_bpm: number | null;
  elevation_gain_meters: number | null; total_calories: number | null;
};

// Steps. Date-keyed; days with no entry are gaps, never zero-step days.
type StepsEntry = { id: string; date: string /* YYYY-MM-DD */; steps: number };
type StepsGoal  = { goal: number };
```

The new metrics are all derivable client-side over the fetched window: **avg
pace** = Σ`distance` / Σ`duration` across runs (or the mean of non-null
`avg_pace_sec_per_km`); **discipline time-split** = Σ lifting duration vs Σ
running duration; **days active** = count of distinct calendar days with any
workout, run, or step entry; **streak** = the longest / current run of
consecutive active days; **avg HR** / **elevation** = means / sums over runs that
report them.

**Visual states the variants must handle** (render them all in the mockup — this
is where lazy compositions fall apart):

- **A zero/rest week.** The window includes a week with no activity (the trough
  on the right of the current chart). Trends must read as *intentional rest* —
  gaps in the heatmap, a dip to zero in the line — not as broken/missing data.
- **A single-modality user.** Some windows are lifting-only or running-only. A
  discipline split that's 100/0, a missing pace tile (no runs), or an absent
  lifting trend must each degrade gracefully, not leave a hole.
- **Null pace / null HR.** Manually-logged runs report no pace or HR. `Avg run
  pace` and any HR/effort element must show "—" cleanly when the window's runs
  lack the data, and weight the average only over runs that have it.
- **No steps history.** Steps are optional; when the window has no step entries
  the steps trend/tile is hidden entirely (today's view already gates on this).
- **Sparse vs dense window.** `7 days` may be one sparse week; `All` spans many
  months. The same composition has to survive a near-empty heatmap and a long,
  dense one — and the trends must not look broken at n=1.
- **Truncation cap.** Workouts fetch caps at 100; when hit, the existing
  truncation note must still surface so older data's absence is explained.

**Representative fixture** (a `Last 90 days` block resembling the screenshots —
use something like it so variants render realistically: a real training quarter
with a rest-week trough, a mix of lifting and running, and a few pace-less manual
runs):

- **Headline** — Total time **46h 7m** · **47 sessions** (21 workouts · 26 runs)
  · Avg session **59m**.
- **Running** — Total mileage **121.5 mi** · Avg pace **9:14 /mi** · Avg HR
  **151 bpm** · Elevation **4,120 ft** · Best pace 7:48 /mi. Weekly mileage
  trends up over the block with a dip the rest week; 3 of the 26 runs are manual
  (pace/HR null).
- **Lifting vs running split** — ~**27h lifting · 19h running** (≈59% / 41% of
  active time).
- **Consistency** — **41 of 90 days active** · current streak **3 days** ·
  longest streak **9 days**; the heatmap shows clustered active weeks and a
  visible rest week near `May 25`.
- **Steps** — Avg daily steps **9,000**; a 90-day sparkline trending flat-to-up,
  a couple of gap days where nothing was logged. (No goal-% tile.)
- **Weekly activity** — the existing dual **Lifting / Running** minutes trend,
  peaking the week of `Jun 1 – 7` and bottoming near `May 25` (both ~0h).

These are mockups: **static fixtures that look real are preferred** — do not wire
the variants to live activity services.

## Idioms

Because `scope: in-system`, the five variants share **one palette, one type
family, one accent, and one chart language** (the design system's). They diverge
on **composition** — *what is the hero, which metrics lead, what new visuals
appear, and how dense the rhythm is* — along the template's three axes (**type
scale**, **color logic**, **spacing rhythm**) as expressed *within* the system.
Each leans on a different reference and promotes a different element.

- **headline-digest** — The Overview as an **authored summary**. A single
  oversized lead readout — total time or a coaching-voice "you trained 41 of 90
  days" line — anchors an airy, column-based layout, with the supporting metrics
  and a couple of captioned charts set small beneath it, like a training-block
  feature in **The Athletic**. Type scale: dramatic *within Manrope* — one huge
  figure, small set-in captions and labels. Color logic: near-monochrome
  ink-on-near-black with the **periwinkle accent reserved for the one number that
  matters most**; discipline hues only on the split. Spacing rhythm: generous,
  editorial, the page breathes. → makes the block feel *narrated* and
  significant; consistency leads.

- **discipline-columns** — The Overview as **parity across disciplines**. Drops
  the undifferentiated stat rows for **side-by-side Lifting / Running / Steps
  columns** (sections on mobile), each a self-contained card with its own lead
  metric, supporting figures, and a small sparkline — lifting time, running
  pace + mileage + HR, steps trend — in the **Garmin Connect "In Focus"** spirit.
  The discipline tonal hues (`lift` steel-blue, `run` green-teal) carry each
  column; a discipline **time-split bar** bridges the top. Type scale: even,
  per-column hierarchy. Color logic: **discipline-hue-led**, accent only for
  cross-cutting chrome. Spacing rhythm: balanced three-up grid. → answers "how's
  each domain" with equal weight; running finally gets real estate.

- **command-strip-dense** — The Overview as a **glanceable console**. A compact
  **KPI strip** of many small metric-label-value triplets across the top — time,
  sessions, pace, avg HR, days active, streak, elevation — then the weekly chart,
  then a dense secondary grid, in the **Whoop** today-screen idiom applied to a
  lifter. Type scale: compact and uniform; hierarchy from **position and density,
  not size**. Color logic: accent + status tints only, sparingly, against an
  otherwise colorless field. Spacing rhythm: tight, gridded, maximally
  informative. → the data-dense, "everything at a glance" end of the spread.

- **consistency-heatmap** — The Overview as a **show-up record**. Makes the
  **calendar heatmap of days active** the hero — a GitHub-contribution-style grid
  over the window, cells tinted by the day's dominant discipline hue, with a
  prominent "**41 of 90 days · 3-day streak**" readout — and demotes the totals to
  a quiet supporting strip beneath, in the calm **Oura** trends idiom. Type scale:
  small, exact; the grid does the talking. Color logic: structural — the lift
  steel-blue and run green-teal tint the heatmap by intensity; everything else is
  muted. Spacing rhythm: generous and even around a single dominant visual. → the
  "**did I show up?**" emotional core, made the centerpiece.

- **instrument-panel** — The Overview as a **panel of trends**. A minimal KPI row
  over a **grid of small-multiple charts** — weekly activity (dual line), a pace
  sparkline, a mileage bar trend, the discipline time-split donut, a steps
  mini-bar — each a small framed instrument, in the **Strava** "your stats" /
  multi-chart spirit. Type scale: small labels, the charts are the hero. Color
  logic: discipline hues per chart, one accent for emphasis. Spacing rhythm: a
  tight, even chart grid. → the most **visual**, trend-forward direction — the
  page becomes graphs first, numbers second.

## References

In-system, so "what to take" is **structure, density, and composition** — not
palette/type, which are fixed:

- **The Athletic** (sports data feature) — take its **authored lead-figure
  layout**: one dominant number, captioned charts, column whitespace, accent used
  once. Drives `headline-digest`.
- **Garmin Connect** ("In Focus" / per-activity cards) — take its **per-discipline
  parity**: each domain its own self-contained card with a lead metric and a
  sparkline. Drives `discipline-columns`.
- **Whoop** (today screen) — take its **dense KPI strip** of metric-label-value
  triplets and first-half-second glanceability. Drives `command-strip-dense`.
- **Oura** (trends / readiness) — take its **calm single-hero-visual restraint**
  and the contribution-grid consistency readout. Drives `consistency-heatmap`.
- **Strava** ("your stats" / activity trends) — take its **multi-panel grid of
  small charts**, graphs-first over numbers. Drives `instrument-panel`.

## Selection criteria

A note-to-self for the pick, not a rubric the worker optimizes against. When I
compare these I'm trying to decide:

- **Does it actually feel like an activity *overview*** — all modalities, not a
  lifting receipt with running bolted on? Running and consistency should carry
  real weight, the way lifting already does.
- **Does it make me *feel* the training block?** Consistency / showing up is the
  emotional core; the variant that makes "41 of 90 days" land is doing the job a
  totals row can't.
- **Is running finally first-class?** Pace, mileage trend, and effort (HR /
  elevation) should read at a glance — not hide behind a single count.
- **Do the new visuals earn their space**, or is it charts for charts' sake? More
  graphs is the ask, but each should answer a real question, and the page must
  still survive a sparse `7 days` and a dense `All`.
- **Does it degrade honestly** on the hard states — a rest week, a single-modality
  window, null pace/HR, no steps — without leaving holes?
- **Does it still read as one surface with the other three tabs?** It inherits the
  shipped language; Overview shouldn't drift from Workouts / Running / Steps.

---

> **Lifecycle.** `status:` is editorial — the owner is the dispatch gate. It moves
> `draft` → `exploring` (worker running) → `awaiting_selection` (draft PR open,
> owner deciding) → `selected` / `abandoned`. The worker sets `awaiting_selection`
> on the `dx/activities-overview` branch as it opens the PR; the owner sets the
> terminal value when they close it.
>
> **Handoff.** This DX ends at a *chosen variant*, not merged code. I open the
> draft `[DX — DO NOT MERGE]` PR, compare the variants on the preview deploy, pick
> one, tick its box, set `status: selected` (noting the winning idiom), and
> **close the PR — never merge it.** Then I open a SOW: *"implement
> activities-overview per the `<chosen-idiom>` variant from
> `dx/activities-overview`, production-quality, conforming to the design system."*
