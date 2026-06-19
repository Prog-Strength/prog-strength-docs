---
type: dx
status: awaiting_selection
surface: workout-detail-refinements
idioms:
  - muscle-body-map
  - per-exercise-volume-bars
  - body-parts-radar
  - per-muscle-set-bars
  - quiet-inline-strip
references:
  - Strava
  - Hevy
  - Strong
  - Whoop
  - TrainingPeaks
scope: in-system
variant_count: 5
repos:
  - prog-strength-web
  - prog-strength-docs
---

# DX: Workout Detail — Refinements (a Graphic for the Session + Expandable Sets)

**Status**: Awaiting selection · **Last updated**: 2026-06-19

> A DX (Design Exploration) is the platform's **divergent** work type. Unlike a
> SOW it does not converge on one correct implementation — it produces N
> differentiated visual variants of a single frontend surface, side by side on
> one comparison route, awaiting a human pick. It **never merges** and ships no
> production code; the chosen direction feeds a downstream SOW that builds it for
> real.
>
> This is a **refinement DX**: the Workout Detail page already shipped its
> `session-recap` redesign (from [`dx/workout-detail.md`](./workout-detail.md)) and
> **reads well** — it is *not* being re-explored wholesale. This DX adds two things
> to that good page, and the **variants diverge on the open question: which graphic
> the page should carry.**

## Context

The Workout Detail page (`/workouts/[id]`,
`prog-strength-web/app/(app)/workouts/[id]/page.tsx`) shipped the **session-recap**
layout and is in good shape: a centered editorial column — a date kicker + a PR
headline (`A new PR — 335 lb × 4 on High Bar Back Squat`), the lifter's note as
prose, a quiet `Volume · Sets · Exercises · Duration · PRs` stat line, a **What it
trained** text strip (`Legs 27 · Core 5`), and **The Work**: one scannable line per
exercise (`Barbell High Bar Back Squat — 5 × · top 335 lb`). Two things are missing:

- **The page is entirely type — it has no graphic.** Every other surface in the app
  now carries a visual (the steps ring, the running splits strip, the PR sparks), but
  the workout page — arguably the richest single record in the product — shows no
  chart at all. The **What it trained** strip *states* the muscle work (`Legs 27 ·
  Core 5`) but doesn't *show* it. The session deserves a graphical signal, and the
  right graphic can **replace that text strip** by answering "what did this train?"
  visually. **Which graphic is the open question this DX explores** — see Idioms.
- **You can't see the exact sets.** The collapsed top-set readout (set count +
  reconstructed top set) is a **good default** worth keeping, but there's **no way to
  expand an exercise and see the actual sets** (the `135 → 225 → 295 → 315 → 335`
  ramp, each `reps × weight`). This is a **shared addition across all variants** —
  not the thing being compared.

**The constraint that rules out the obvious answer — read before designing.** The
tempting graphic is a **body-parts radar**, and it was in fact *deliberately removed*
from this page. At **single-workout grain** the muscle data is sparse: across six
fixed categories (Chest · Back · Shoulders · Arms · Legs · Core), one session usually
lights only two or three — the screenshot's leg day is **Legs 27 · Core 5, four axes
at zero**. A naive radar draws a lopsided sliver that reads as "broken chart," and
that's why it was cut for the text strip. So the design problem isn't "make a radar" —
it's **"what graphic genuinely suits a single, often-focused workout?"** Some
candidates (a muscle body-map, per-exercise volume bars) sidestep the sparsity
entirely; the radar is included as one option, but a variant must prove its graphic
reads as **intentional, not broken, on the focused leg day.**

`scope: in-system`: the foundation is decided (design-system **v0.4**, oura-calm) —
soft near-black ramp, periwinkle accent (app-chrome only), the **lift discipline
hue** steel-blue (`--discipline-lift-*`), Manrope with tight numeric tracking, 14px
hairline panels, desaturated success/warning. Variants do **not** re-decide palette,
accent, or type, and **keep the session-recap structure intact** — they diverge only
on **which graphic replaces the "What it trained" strip** (and how it's composed,
placed, and degraded). **No API change** — the exact sets are already in the workout
payload and the muscle tally is computed client-side.

## What every variant shares

So the comparison isolates *the graphic*, all five variants carry the **same two
non-divergent pieces**:

- **Expandable sets.** The `RecapExerciseList` keeps its collapsed top-set readout as
  the **default**, and each exercise gains an **expand affordance** revealing its
  actual sets (`set · reps × weight`, the top set / PR row marked in `--warning`,
  client-matched). Use one sensible, consistent pattern across variants (an
  inline per-exercise accordion is the obvious default) — this is an *addition*, not
  the axis of comparison. It must work within superset blocks and keep bodyweight /
  per-dumbbell readouts.
- **The session-recap structure** — Lead, Reflection, Stat line, The Work — and
  **every edit affordance** (per-group pencil, `+ Add exercise`, Edit details,
  Delete) are preserved unchanged.

The **one thing that differs between variants is the graphic** that replaces the
*What it trained* strip.

## The surface

- **Header** — `← Workouts`, subtitle (`W5 D2 - Legs · June 19, 2026 · 55m`),
  **Edit details** + **Delete**. *(Unchanged.)*
- **Lead** — date kicker + PR headline + `--warning` PR subhead. *(Unchanged.)*
- **Reflection** — the note as prose, or a quiet placeholder. *(Unchanged.)*
- **Stat line** — `Volume · Sets · Exercises · Duration · PRs`, honest tile-dropping.
  *(Unchanged — though a variant may place its graphic beside it.)*
- **What it trained** — **the slot the graphic replaces.** Today a text strip of the
  populated categories largest-first: `Legs 27 · Core 5`.
- **The Work** — `RecapExerciseList`, gaining the shared **expand-to-sets**.

The data (`prog-strength-web/lib/api.ts`) — everything needed is already loaded:

```ts
type WorkoutExercise = {
  exercise_id: string;          // → catalog name + muscle_groups
  order: number;
  superset_group?: number | null;
  sets: WorkoutSet[];           // THE EXACT SETS — already in the payload
};
type WorkoutSet = { reps: number; weight: number; unit: "lb" | "kg" };
type PersonalRecordEvent = {    // workout.personal_records_set[]
  exercise_id: string; weight: number; reps: number; unit: "lb" | "kg";
  // ...no pointer to the WorkoutSet that set it (see data note)
};
```

The graphics draw on two derivations, both client-side and already available:

- **Per-muscle / body-part work** — `setsByCategory(workouts, exercises)`
  (`lib/muscle-set-distribution.ts`) over the fixed six-axis `CATEGORIES`, the same
  source the current strip uses (feeds the radar, the per-muscle bars, the body-map).
  Finer-grained `exercise.muscle_groups` (from the catalog) is what a body-map would
  shade per region.
- **Per-exercise volume** — `workoutVolume` / a per-exercise `Σ reps × weight`
  (feeds the volume bars).

**Data notes that shape the variants.**

- **PRs aren't linked to sets.** A `PersonalRecordEvent` carries the PR's `weight ×
  reps` but **no pointer to the `WorkoutSet`** — marking the PR set in the expanded
  list (🏆 on `335 × 4`) is a **client-side match** by weight/reps; design the honest
  tie / "couldn't match" behavior.
- **No warm-up flag.** The ramp is just ascending sets; the top set is reconstructed
  (`topSetIndex`). Distinguishing warm-ups in the expanded view is reconstruction.
- **Bodyweight & per-dumbbell sets.** Non-positive weight → read reps, not a load;
  per-dumbbell lifts append the clarifier (the recap list already does both) — the
  expanded rows and any volume graphic must respect this.
- **Component boundary (low blast radius).** The list is the detail-page-specific
  `RecapExerciseList`; the shared `WorkoutDetailsBody` (Workouts list, Calendar,
  Timeline) is **out of scope**. A `MuscleGroupRadarChart` exists but is **pre-v0.4
  (raw `#3b82f6`/zinc hex)** and used elsewhere — so variants build a **v0.4-conforming**
  graphic, not adopt the old chart as-is.

**Visual states the variants must handle** (render them all in the mockup):

- **The focused, sparse session (the crux).** The leg day is **Legs + Core only** —
  the graphic must read as **intentional emphasis**, never broken/empty. This is the
  test that removed the radar; each variant must visibly pass it.
- **A multi-category session.** A chest-&-back day hitting 4–5 categories / many
  exercises — the graphic's "full" case, proven alongside the sparse one.
- **Uncategorizable session.** No mappable muscle groups → the graphic degrades (drop
  or a calm empty state) like the current strip, never an empty frame.
- **Expanded sets** — the shared accordion across collapsed default ↔ per-set detail,
  including supersets, bodyweight, per-dumbbell, and the client-matched PR row.
- **Long / dense vs short sessions** — the graphic and the expanded list survive a
  10-exercise day and a 3-exercise one.
- **Edit affordances intact**; **both breakpoints** (graphic + expand work on mobile
  and desktop).

**Representative fixture** (mirror the screenshot — the sparse case is the point):

- **Workout**: `W5 D2 - Legs`, June 19 2026, **55m**. **Volume 31,265 lbs · 17 sets ·
  4 exercises · 1 PR.** Headline `A new PR — 335 lb × 4 on High Bar Back Squat`.
- **What it trained**: **Legs 27 · Core 5** (two of six axes — the hard case).
- **The Work** (real sets to expand):
  - **Barbell High Bar Back Squat** — `135 × 8 · 225 × 5 · 295 × 5 · 315 × 4 · 335 ×
    4` (`335 × 4` is the top set **and the PR**).
  - **Machine Lying Leg Curl** (top 145), **Barbell Calf Raise** (top 275), **Leg
    Extension** (top 165) — 4 sets each.
- **Plus a second fixture**: a multi-category chest-&-back day (4–5 categories,
  several PRs, a superset) — so each graphic is proven both sparse and full.

These are mockups: **static fixtures that look real are preferred** — no live wiring.

## Idioms

Five **different graphics** for the session, each replacing the *What it trained*
strip and each paired with the shared expand-to-sets. They diverge on **what the
graphic encodes and how it handles the focused leg day**; none re-decides palette,
accent, or type, and all keep the recap structure.

- **muscle-body-map** — A **front/back anatomical silhouette** with the worked
  regions **shaded by set volume** (a steel-blue intensity ramp). The most spatial,
  intuitive answer to "what did this train" — and it **degrades beautifully**: a
  focused leg day simply glows at the quads/hamstrings/calves + core, which reads as
  obviously intentional rather than as missing data. Sits where the strip was (or
  beside the stat line), captioned with the top categories. → the most "polished
  graphic," and the one that turns the sparsity from a liability into a feature.
  (Strava's muscle heatmap.)

- **per-exercise-volume-bars** — A compact **horizontal bar per exercise**, sized by
  that exercise's **volume** (`Σ reps × weight`), largest first — "where the work
  went" this session (Squat ≫ Calf Raise ≫ …). **Zero degeneracy**: every exercise
  has a bar, so it reads the same on a 4-lift or 10-lift day and never looks empty or
  lopsided. The lift discipline hue on the bars; the PR exercise marked. → the
  honest, always-populated answer that doubles as a glance at session balance.
  (Hevy / Strong session breakdowns.)

- **body-parts-radar** — The **six-axis radar** of `setsByCategory`, the flashy
  "shape of the session" take — included so it can be judged head-to-head. To survive
  the focused day it must be built to read as emphasis: the full six-axis frame + a
  faint baseline ring drawn always, the area filled in steel-blue, **axis labels
  carrying the counts** so even a two-axis shape reads as "this session was Legs &
  Core," not a broken sliver. → the most distinctive shape, carrying the most risk on
  sparse data. (Whoop's single emphasis visual.)

- **per-muscle-set-bars** — Horizontal **bars per muscle category** (`Legs 27 · Core
  5` as sized bars, largest first), the literal visual upgrade of today's text strip.
  Honest and calm; degrades by simply **showing fewer bars** (a focused day shows two,
  which looks fine). Steel-blue fill, counts at the end of each bar. → the smallest
  leap from the current strip — same data, now seen. (TrainingPeaks splits-style
  bars.)

- **quiet-inline-strip** — The lightest touch, honoring the calm reading page: a
  **small inline intensity strip** — the categories as a single thin segmented
  bar (each segment a category, width = its share of sets, steel-blue by intensity),
  with the counts as a caption — that stays *within* the editorial column rather than
  becoming a chart block. Replaces the strip with a graphic that barely raises the
  page's volume. → the conservative, in-character option for "a graphic, but keep it
  quiet." (The session-recap's own restraint.)

## References

In-system, so "what to take" is **structural** — composition and data legibility,
not palette or type:

- **Strava** (workout muscle heatmap) — take its **anatomical body-map** shading
  worked regions, which reads naturally even when few regions are hit. Drives
  `muscle-body-map`.
- **Hevy / Strong** (workout logs) — take their **per-exercise session breakdown**
  (volume/effort per movement) and the **expandable set detail** every variant
  shares. Drives `per-exercise-volume-bars`.
- **Whoop** (single emphasis visual) — take the **one promoted shape** that makes a
  distribution legible at a glance. Drives `body-parts-radar`.
- **TrainingPeaks** (training analysis) — take its **per-category / per-segment bars**
  for an honest, scannable distribution. Drives `per-muscle-set-bars`.
- **The shipped session-recap** itself — take its **restraint**: add signal without
  raising the page's volume. Drives `quiet-inline-strip`.

## Forward-compatibility — lift TCX heart rate & calories (a later SOW)

A follow-up is planned: let users **upload a Garmin TCX for a lifting session** to
attach **heart-rate-over-time and calories burned** to the workout (plus avg / max
HR, and the like). Unlike the running TCX path it carries **no pace, distance, or
elevation** — just an HR trackpoint series, total calories, and duration — so it's a
subset of the running importer, and the workout page would gain an **HR-over-time
trace** and a couple of **HR / calorie stat tiles**.

**This DX does not build that.** The data doesn't exist yet and the upload + parse +
storage is a backend change; mocking HR/calories in the comparison route would muddy
the graphic comparison over fabricated data, and would pile a second/third axis onto
what should be a clean "which graphic?" pick. It is deferred to its **own follow-up
SOW** — **API**: a lift-TCX upload/parse that stores an HR trackpoint series +
calories on the workout (no pace/distance/elevation); **web**: the HR-over-time trace
(which can reuse the running detail's existing `HeartRateChart` pattern) plus calorie
/ avg-HR / max-HR tiles.

**What this DX *should* do is leave it a home.** Each variant should compose its
graphic + stat region so that, when HR and calories arrive, they **slot in without a
layout redo**: the stat line has room to grow `Avg HR · Max HR · Calories` tiles
(rendered only when a TCX is attached, dropped otherwise — exactly the existing
honest tile-dropping), and the graphic area can sit beside or above a future
full-width **HR-over-time trace** rather than fighting it for space. Treat HR &
calories as **reserved space, not rendered content** — a variant that boxes itself in
(a stat line that can't grow, no room for a second visual) is a worse pick even if
its muscle graphic is the nicest.

## Selection criteria

A note-to-self for the pick, not a rubric the worker optimizes against. When I
compare these I'm trying to decide:

- **Which graphic reads as polished and intentional on the focused leg day** (Legs +
  Core only) — not broken, not empty — *and* still shines on a balanced day?
- **Does it answer "what it trained"** well enough to **replace the text strip**, and
  degrade to nothing on an uncategorizable session?
- **Does it feel worth the space** — a genuine glance-value graphic, not decoration —
  on the app's richest single record?
- **(Shared) Can I expand an exercise to the exact sets**, with the calm collapsed
  readout kept as the default and the PR set marked?
- **Does the layout leave a natural home** for the planned lift-TCX data — a stat
  line that can grow `Avg HR · Max HR · Calories` tiles and room for an HR-over-time
  trace — so the future feature drops in without a redo?
- **Do supersets, bodyweight, and per-dumbbell sets** survive the expanded rendering?
- **Are the edit affordances** (pencil, + Add exercise, Edit details, Delete) still
  present and obvious?
- **Does it stay the calm v0.4 recap** — lift steel-blue for the graphic, `--warning`
  only on the PR, periwinkle as app-chrome — rather than turning a good reading page
  into a busy dashboard?

---

> **Lifecycle.** `status:` is editorial — the owner is the dispatch gate. It moves
> `draft` → `exploring` (worker running) → `awaiting_selection` (draft PR open, owner
> deciding) → `selected` / `abandoned`. The worker sets `awaiting_selection` on the
> `dx/workout-detail-refinements` branch as it opens the PR; the owner sets the
> terminal value when they close it.
>
> **Handoff.** This DX ends at a *chosen variant*, not merged code. I open the draft
> `[DX — DO NOT MERGE]` PR, compare the variants on the preview deploy
> (`/design-explore/workout-detail-refinements`, flag-gated behind
> `NEXT_PUBLIC_ENABLE_DESIGN_EXPLORE`, across the sparse leg-day and the multi-category
> fixtures), pick one, tick its box, set `status: selected` (noting the winning
> graphic), and **close the PR — never merge it.** Then I open a SOW: *"add a
> body-parts graphic (the `<chosen-idiom>` treatment) and expandable per-exercise sets
> to the workout detail page per `dx/workout-detail-refinements`, production-quality,
> conforming to the design system."* The mockup code is never promoted as-is.
