---
type: dx
status: awaiting_selection
surface: workout-detail
idioms:
  - session-recap
  - pr-celebration
  - exercise-ledger
  - training-dashboard
  - session-timeline
references:
  - The Athletic
  - Apple Fitness
  - Strong
  - Whoop
  - Strava
scope: in-system
variant_count: 5
repos:
  - prog-strength-web
  - prog-strength-docs
---

# DX: Workout Detail

**Status**: Awaiting selection · **Last updated**: 2026-06-19

> A DX (Design Exploration) is the platform's **divergent** work type. Unlike a
> SOW it does not converge on one correct implementation — it produces N
> differentiated visual variants of a single frontend surface, side by side on
> one comparison route, awaiting a human pick. It **never merges** and ships no
> production code; the chosen direction feeds a downstream SOW that builds it for
> real.

## Context

The **Workout Detail** page (`/workouts/[id]`,
`prog-strength-web/app/(app)/workouts/[id]/page.tsx`) is the one-stop record of a
single logged session — *"what did I do, and how did it go?"* It's reached from the
Workouts list, the Personal Records `View workout →` links, the calendar, the
agent/chat, and anywhere a workout is referenced by id. The **functionality is
solid and not in question**: the workout loads, the PR break events render, the
summary tiles and muscle analytics compute, the plan-linkage resolves, and the
edit affordances (Edit details, Delete, per-exercise pencil, + Add exercise) all
work.

What's unsatisfying is that the page that should make a lifter **relive a great
session** reads like a generic admin dashboard — a row of equal stat tiles, two
analytics boxes, and a flat list of set bullets — and the screenshot is a *great*
session (four PRs, a 305 lb bench double) that the composition under-celebrates.
Five things specifically undercut it:

- **The PR moment is muted and disconnected.** Four new PRs — the most exciting
  thing that can happen in this app — render as a flat amber box at the top, while
  the sets that *achieved* them sit ~1000px down in the exercise list, where the
  PR'd `305 × 2` bench set looks **identical** to the `185 × 10` warm-up above it.
  The celebration is a list item; the achievement is never tied to where it
  happened.
- **The two muscle analytics are redundant.** **Body Parts** (a six-axis radar) and
  **Sets per muscle** (horizontal bars) plot the *same* `setsByCategory` numbers
  twice, side by side, eating a full panel of vertical space for one fact (set
  distribution). Worse, at **single-workout grain** the radar is degenerate — only
  3 of 6 axes have data (Back 13, Chest 12, Shoulders 4; Arms/Legs/Core 0), so it
  draws a lopsided sliver that reads as "broken chart," not insight.
- **The exercise list is a flat data dump.** Each exercise is a stack of identical
  bullets — `10 reps × 1 set @ 185 lbs`, `… @ 225 lbs`, `… @ 275 lbs`, `… @ 305
  lbs` — with no expression of the session's **structure**: the warm-up ramp vs the
  working/top set, the per-exercise volume or top set, or which row was the PR. The
  shape of how the lift was actually worked up is invisible.
- **The human voice is buried.** The lifter wrote a real reflection — *"I had to
  skip my usual abs superset because I was late for work… I hit 305 lbs for 2 reps
  on bench press. Nice!!!"* — but it's plain body text wedged between the
  "+ Add exercise" link and the first exercise, with no emphasis. The note that
  gives the session its story is the least prominent thing on screen.
- **The visual language is stale.** This surface **predates the v0.4 oura-calm
  re-tone**: the radar and bars are hardcoded raw blue/zinc (`#3b82f6`, `#27272a`,
  `#a1a1aa`), the PR banner is Tailwind `amber-*` rather than the `--warning` token,
  and the superset block leans on the periwinkle accent as a structural tint. The
  page reads like the old slate/blue dashboard the rest of the app has moved past.

This DX explores the **entire Workout Detail page** afresh — header, PR moment,
summary, muscle analytics, the note, and the exercise list — to find a direction
that treats the surface as **the record of one training session, with a story, a
structure, and (when it happened) a celebration**, before committing engineering
effort.

`scope: in-system`: the foundation is decided (see
[`../design-system.md`](../design-system.md) **v0.4**, oura-calm) and is **not**
re-litigated here — soft near-black ramp, the single **periwinkle** accent
(`#9aa6d6`, app-chrome only), the **lift discipline tone** (steel-blue
`--discipline-lift-*`: `#1a2a3c` / `#8cbce8` / `#5598d8`), **Manrope** with tight
numeric tracking and no display face, 14px panel radius, hairline borders.
Critically, **this surface has not yet been migrated to those tokens** — so
conforming the bespoke charts and the PR banner to v0.4 is itself part of the brief.
Variants diverge on **layout, structure, density, and composition** — what the
page's hero is, how the PR moment is told, how the redundant analytics collapse,
how the exercise list expresses set structure, and where the note lives — never on
palette, accent, or type.

## The surface

The Workout Detail page (a variant owns the whole view and may reorganize, demote,
merge, or recompose any of it, but must account for all of it):

- **Header** — a `← Workouts` back-link, the workout **name** (`Week 2 Workout 1 -
  Chest & Back`, falling back to the date when the name is auto-generated), a
  `date · duration` subtitle (`May 11, 2026 · 1h 20m`), and two top-right actions:
  **Edit details** (opens the workout-level field editor) and **Delete**
  (destructive, confirm-then-redirect). **These edit affordances must survive any
  recomposition** — unlike the running/PR detail surfaces, this page is *editable*.
- **Plan-linkage banner** — present only when this session fulfilled a planned
  workout (`getPlannedWorkoutBySession(token, id, "workout")` → non-null): the
  shared `CompletesPlanBanner` with an Unlink action. Absent for an unplanned
  session (the common case).
- **PR banner** — present only when `personal_records_set.length > 0`: a 🏆 header
  (`{n} new personal records!`) and one line per break — exercise name · `weight ×
  reps` · `(up from {previous_weight})`. The "up from" clause is omitted on a
  first-ever logged set (`previous_weight === null`, e.g. the Dumbbell Tripod Row).
- **Summary tiles** — a row of compact stats derived from the workout:
  **Volume** (`23,685 lbs`, Σ reps × weight in the predominant unit), **Sets**
  (`25`), **Exercises** (`6`), **Duration** (`1h 20m`, **omitted when no
  `ended_at`**), **PRs** (`4`, **omitted when zero**). The row stays honest — it
  drops tiles rather than padding with zeros.
- **Muscle analytics** — today two side-by-side views of the **same**
  `setsByCategory` aggregation over six fixed categories (Chest, Back, Shoulders,
  Arms, Legs, Core): a **Body Parts** radar and a **Sets per muscle** bar list
  (`Back 13 · Chest 12 · Shoulders 4 · Arms/Legs/Core 0`). Set-count only (no unit).
- **Exercises** — a `+ Add exercise` action, the workout **note** (free text,
  `whitespace-pre-wrap`), then the exercise list rendered by the shared
  `WorkoutDetailsBody`: group-numbered exercises (a **superset** is one numbered
  block with a left-accent border and its sub-exercises inside), each with a
  **muscle-group pill** and its sets as `{reps} reps × 1 set @ {weight} lbs`
  bullets, a per-exercise **edit pencil**, and a **Total volume** footer.

The data shape (`prog-strength-web/lib/api.ts`), one `Workout` from
`getWorkout(token, id)`:

```ts
type Workout = {
  id: string;
  name?: string;                       // "Week 2 Workout 1 - Chest & Back"; auto "Workout - <date>" ⇒ show date
  performed_at: string;                // RFC3339
  ended_at?: string | null;            // null ⇒ no duration tile
  notes?: string;                      // the free-text reflection
  exercises: WorkoutExercise[];
  personal_records_set: PersonalRecordEvent[];  // always present; [] when none
  // ...user_id, created_at, updated_at
};

type WorkoutExercise = {
  exercise_id: string;                 // slug → catalog (name + muscle_groups)
  order: number;
  superset_group?: number | null;      // shared value ⇒ same superset; null ⇒ standalone
  sets: WorkoutSet[];
  notes?: string;
};

type WorkoutSet = { reps: number; weight: number; unit: "lb" | "kg" };

type PersonalRecordEvent = {
  id: string;
  exercise_id: string;
  weight: number; reps: number; unit: "lb" | "kg";
  previous_weight: number | null;      // null ⇒ first-ever set (no "up from")
  previous_reps: number | null;
  previous_unit: "lb" | "kg" | null;
  achieved_at: string;
};

// Catalog (listExercises()): id → { name, muscle_groups[], equipment[] }.
// muscle_groups feed setsByCategory → the six body-part categories.
```

**Data notes that shape the idioms — read before designing.**

- **PRs aren't linked to sets in the payload.** A `PersonalRecordEvent` carries the
  PR `weight × reps` but **not a pointer to the `WorkoutSet`** that set it. A variant
  that wants to mark the PR row inline (🏆 on the `305 × 2` bench set) must **match
  it client-side** — find the exercise's set whose `weight`/`reps` equal the event's
  — not expect a server-provided link. (Tie-break / honest "couldn't match"
  behavior is the variant's to design.)
- **Set structure is latent, not labeled.** There's no "warm-up" flag — the ramp
  (`185 → 225 → 275 → 305`) is just an ascending sequence of sets. A variant that
  wants to distinguish warm-ups from working/top sets reconstructs that from the
  set sequence itself (e.g. the top set, or sets within X% of it), and must degrade
  for a flat/even set scheme that has no clear ramp.
- **`setsByCategory` is shared by both analytics**, so the radar and bars can never
  disagree — collapsing them to one visualization loses no information.

**Shared-component constraint.** The exercise list is the shared
`WorkoutDetailsBody`, **also rendered by the Workouts list inline-expand and the
Calendar read-only modal**. A variant that rebuilds the set/exercise presentation
should either introduce a **detail-page-specific** rendering (leaving the shared
body for the other two surfaces) or note that changing the shared body has
blast radius — the SOW will decide, but the variant should flag which it assumes.

**Visual states the variants must handle** (render them all in the mockup):

- **Zero-PR workout.** Most sessions set no PR — the celebration must **vanish
  gracefully** and the page can't lean entirely on the trophy moment; it still has
  to feel complete as a plain session record.
- **The degenerate single-workout radar.** Only 3 of 6 categories have sets — a
  six-axis radar of one session is a lopsided sliver. A variant must **not** render
  a near-empty chart as if it's insight; collapse, reshape, or drop it.
- **Warm-up ramp vs working sets.** The `185 → 305` bench progression vs a flat
  `3 × 5` — the structure idioms must make the ramp legible **without** inventing a
  warm-up flag the data doesn't have, and degrade when there's no clear top set.
- **Supersets.** `superset_group` groups alternating-set exercises into one block
  (the note even mentions a *skipped* abs superset). The grouping must read as one
  unit, not dissolve into standalone exercises.
- **No note / no `ended_at`.** A session with no reflection (the narrative idiom
  must degrade), and an open/never-finalized session with no duration (the tile and
  any time-based framing must vanish cleanly).
- **Long / dense session.** 6 exercises here, but up to ~12 with many sets — the
  exercise rendering and any analytics must survive both sparse and dense.
- **Dumbbell convention.** `weight` is **per dumbbell** (the `80 lb` Tripod Row and
  `90 lb` Incline DB Bench are *per hand*); a variant may make this explicit but
  must not imply a combined load. Unit is `lb` or `kg` per set.
- **Planned-workout linkage present vs absent** — the ✓ banner must integrate, not
  fight, the PR banner when both are present, and disappear when unplanned.
- **Editable surface.** Edit details, Delete, the per-exercise pencil, and
  + Add exercise must remain **present and obvious** after any recomposition.

**Representative fixture** (mirror the screenshot — a four-PR chest & back day):

- **Workout**: `Week 2 Workout 1 - Chest & Back`, May 11 2026, **1h 20m**.
  **Volume** `23,685 lbs`, **25 sets**, **6 exercises**, **4 PRs**.
- **PR events**: Barbell Bench Press `305 × 2` (up from 245); Barbell Bent Over Row
  `205 × 3` (up from 185); Incline Dumbbell Bench Press `90 × 7` (up from 85);
  Dumbbell Tripod Row `80 × 6` (**first-ever — no "up from"**).
- **Muscle distribution**: Back 13 · Chest 12 · Shoulders 4 · Arms 0 · Legs 0 ·
  Core 0 (the degenerate-radar case).
- **Note**: *"I had to skip my usual abs superset because I was late for work. Maybe
  I can do some abs during lunch to make up for it. Other than that, it was a great
  workout! I hit 305 lbs. for 2 reps on bench press. Nice!!!"*
- **Bench sets (the ramp to a top-set PR)**: `185 × 10 · 225 × 10 · 275 × 6 · 305 ×
  2` — design the variant so the `305 × 2` reads as the **top set / PR**, not a
  fourth identical bullet.
- Include a **second fixture** for the degrade paths: a **zero-PR, no-note,
  no-`ended_at`** session with a flat set scheme and a superset, so every variant is
  checked away from the celebratory happy path.

These are mockups: **static fixtures that look real are preferred** — do not wire
the variants to the live workout service. The crafted fixture must carry the PR
ramp, the redundant/degenerate muscle data, the note, and a superset, because the
idioms have nothing to show against a featureless session.

## Idioms

Five genuinely distinct **compositions** of the *same* oura-calm v0.4 surface.
Because `scope: in-system`, none re-decides palette, accent, or type — they diverge
along **type scale** (one editorial headline vs uniform tabular set rows vs a hero
stat), **color logic** (how the lift discipline steel-blue, the `--warning`
PR/celebration tone, and the `--success` "up from" delta separate meaning from
neutral chrome — never the periwinkle accent as a card tint), and **spacing rhythm**
(airy recap ↔ dense ledger ↔ chronological flow). Each makes a **different element
the hero** and leans on a different reference.

- **session-recap** — Tells the session as **a short story**. The **note is
  promoted to the lead** (a real headline + the reflection), the PRs ride as a
  celebratory subhead (*"4 PRs — including 305 × 2 on bench"*), and the numbers and
  exercises support beneath. Editorial type contrast within Manrope; airy
  reading-document spacing; the muscle analytics collapse to a single quiet strip.
  Color calm and near-monochrome, `--warning` reserved for the PR mention.
  Degrades to a plain session summary when there's no note. → answers *the human
  voice is buried* and *the celebration is muted*. (The Athletic recap / Oura day
  card.)

- **pr-celebration** — The **PRs are the page**. The trophy moment is promoted to
  the hero: each PR a confident block with the lift, the `weight × reps`, and the
  **`up from` delta large** (`245 → 305, +60`), each **tied back to the set that
  set it**. The summary, analytics, and full exercise list demote beneath. Mid
  type scale; `--warning` carries the celebration budget, `--success` the deltas,
  the lift steel-blue for the discipline. **Degrades cleanly to a calm session
  summary when there are zero PRs** (the common case) — the variant must prove it
  doesn't collapse without trophies. → answers *the PR moment is muted and
  disconnected*. (Apple Fitness award moment / Strava achievements.)

- **exercise-ledger** — The **exercise list is the hero**, rebuilt so each lift
  expresses its **set structure**: the warm-up ramp distinguished from the working/
  top set, a per-exercise top-set + volume readout, and the **PR set marked inline**
  (🏆 on the `305 × 2` row, matched client-side). Supersets read as one block.
  Uniform fine tabular set rows; hierarchy from alignment; the redundant radar+bars
  collapse to one compact muscle strip in a demoted header. Color-as-state only —
  steel-blue on the working set, `--warning` on the PR row. → answers *the exercise
  list is a flat data dump* and *the PR is disconnected from its set*. (Strong /
  Hevy workout log.)

- **training-dashboard** — Leans **into** the dashboard, but makes it *earn it*.
  The redundant two-chart muscle panel collapses to **one** consolidated
  visualization (volume- or set-by-muscle, the degenerate radar killed), promoted
  as the hero alongside the stat tiles; the exercise list demotes to a dense
  scannable table. Small functional type, very tight rhythm, status-encoded color.
  The power-user *"what did this session train"* answer, and the one that scales
  best to a 12-exercise day. → directly fixes *the redundant/degenerate analytics*.
  (Whoop overview grid / a training dashboard.)

- **session-timeline** — The session **as it happened in time**. Exercises (and
  supersets) lay out on a vertical **time spine**, the skipped-abs note dropped
  inline where it belongs in the flow, sets reading set-by-set, and the **PR breaks
  lighting up at the moment they occurred**. The `1h 20m` duration becomes
  structural — the page answers *when*, not just *what*. Medium feed type; color
  driven by event (a PR is a warm beat in the timeline), calm otherwise. Degrades
  gracefully when there's no `ended_at` to anchor a true timeline (falls back to
  ordered flow). → makes the **session's chronology** the story and surfaces the
  note in context. (Strava lap/segment timeline / a workout replay.)

## References

In-system, so "what to take" from each is **structural** — composition, encoding,
and information design, not their palettes or type:

- **The Athletic** (race/game recap) — take its **editorial recap voice**: a
  headline and a sentence that *say what happened* before the numbers. Drives
  `session-recap`.
- **Apple Fitness** (award / close-your-rings moment) — take its **celebratory hero
  moment**: one achievement promoted large and unmissable. Drives `pr-celebration`.
- **Strong / Hevy** (workout log) — take their **per-exercise set structure**:
  working sets vs warm-ups, top set, inline PR markers in a dense legible log.
  Drives `exercise-ledger`.
- **Whoop** (overview grid) — take its **single consolidated overview** that makes
  one dense panel do the analytical work. Drives `training-dashboard`.
- **Strava** (activity timeline / laps) — take its **chronological flow**, the
  session read along time with events marked where they happened. Drives
  `session-timeline`.

## Selection criteria

A note-to-self for the pick, not a rubric the worker optimizes against. When I
compare these I'm trying to decide:

- **Does a great session finally feel great?** Four PRs and a 305 bench should make
  me want to relive it — not read a dashboard.
- **Is the PR moment tied to where it happened**, so the achievement and the set
  that earned it aren't 1000px apart?
- **Did the redundant/degenerate muscle analytics get fixed** — one honest
  visualization, not the same numbers twice and a near-empty radar?
- **Does the exercise list express the session's structure** — the warm-up ramp,
  the top set — instead of identical bullets?
- **Does the human note get a real home** without making a no-note session feel
  broken?
- **Does it degrade** to a zero-PR, no-note, no-duration session, to supersets, and
  to a 12-exercise day without looking half-built?
- **Do the edit affordances survive** — Edit details, Delete, per-exercise pencil,
  + Add exercise still present and obvious?
- **Does it finally read as v0.4** — calm near-black, steel-blue discipline tone,
  `--warning` spent only on the celebration, periwinkle as app-chrome — rather than
  the stale blue/zinc/amber dashboard it is today?

---

> **Lifecycle.** `status:` is editorial — the owner is the dispatch gate. It moves
> `draft` → `exploring` (worker running) → `awaiting_selection` (draft PR open,
> owner deciding) → `selected` / `abandoned`. The worker sets `awaiting_selection`
> on the `dx/workout-detail` branch as it opens the PR; the owner sets the terminal
> value when they close it.
>
> **Handoff.** This DX ends at a *chosen variant*, not merged code. I open the draft
> `[DX — DO NOT MERGE]` PR, compare the variants on the preview deploy
> (`/design-explore/workout-detail`, flag-gated behind
> `NEXT_PUBLIC_ENABLE_DESIGN_EXPLORE`), pick one, tick its box, set `status:
> selected` (noting the winning idiom), and **close the PR — never merge it.** Then
> I open a SOW: *"implement workout-detail per the `<chosen-idiom>` variant from
> `dx/workout-detail`, production-quality, conforming to the design system."* The
> mockup code is never promoted as-is.
