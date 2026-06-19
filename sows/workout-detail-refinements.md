---
type: sow
status: ready_for_implementation
repos:
  - prog-strength-web
  - prog-strength-docs
---

# Workout Detail — Muscle Body-Map & Expandable Sets

**Status**: Ready for implementation · **Last updated**: 2026-06-19

## Introduction

The Workout Detail page (`/workouts/[id]`) shipped its `session-recap` editorial
layout and reads well: a date kicker, a PR headline, the lifter's note as prose, a
quiet `Volume · Sets · Exercises · Duration · PRs` stat line, a **What it trained**
text strip (`Legs 27 · Core 5`), and **The work** — one scannable line per exercise.
The page is in good shape and is **not** being re-explored wholesale.

Two gaps remain, and the design exploration in
[`dx/workout-detail-refinements.md`](../dx/workout-detail-refinements.md) (PR
[#93](https://github.com/Prog-Strength/prog-strength-web/pull/93), now closed)
converged the open question to a single direction:

1. **The page is entirely type — it carries no graphic.** Every other surface in the
   app now shows a visual (the steps ring, the running splits strip, the PR sparks),
   but the richest single record in the product shows no chart. The **What it
   trained** strip *states* the muscle work but doesn't *show* it.
2. **You can't see the exact sets.** The collapsed top-set readout is a good default,
   but there's no way to expand an exercise and see its actual `135 → 225 → 295 → 315
   → 335` ramp.

The DX produced five graphics for the session and the selected direction is
**`muscle-body-map`** — a front/back anatomical silhouette with the worked regions
shaded by set volume. It was chosen because it **degrades beautifully**: a focused
leg day simply *glows* at the quads/hamstrings/calves + core, which reads as
intentional emphasis rather than the "broken sliver" that a sparse radar produces.
This SOW reimplements that variant production-quality (the DX branch shipped no
production code) and adds the shared **expand-to-sets** affordance.

The constraint that ruled out the obvious radar still holds: at single-workout grain
the muscle data is sparse (the representative leg day is `Legs 27 · Core 5`, four of
six categories at zero). The body-map sidesteps sparsity entirely — an unworked
region is simply not shaded, which on a silhouette reads as obviously deliberate.

`scope: in-system`. The design-system **v0.4** (oura-calm) foundation is decided and
**not re-opened**: near-black ramp, periwinkle accent (app-chrome only), the **lift
discipline hue** (`--discipline-lift-*`) for the graphic, Manrope, 14px hairline
panels. No palette, accent, or type is re-decided here. **No API change** — the exact
sets are already in the workout payload and the muscle tally is computed client-side.

## Proposed Solution

The session-recap structure (Lead, Reflection, Stat line, The work) and every edit
affordance (per-group pencil, `+ Add exercise`, Edit details, Delete) are preserved.
Two things change.

**The graphic.** A new `MuscleBodyMap` component renders a front + back anatomical
silhouette with worked regions shaded by set volume on a steel-blue intensity ramp.
It is built on the **`react-muscle-highlighter`** library (v1.2.0, MIT, React 18/19
peer, zero runtime deps, typed `Slug` union, published Jan 2026) rather than
hand-rolled SVG — the anatomy is intense to author and the library models intensity
as a first-class concept. The library renders one `<Body side="front" />` and one
`<Body side="back" />`; each datum is `{ slug, intensity }` where `intensity` is a
1-based index into a `colors` ramp (`colors[intensity - 1]`), falling back to
`defaultFill` for unworked parts. We retone it to v0.4 entirely through props
(`colors`, `defaultFill`, `defaultStroke`, `border`) — no CSS polygon overrides.

The map **replaces the "What it trained" text strip** and is placed as a **hero
beside the stat line** on desktop (the variant's signature), stacked above the stat
line on mobile. It keeps the populated categories as its caption (`Legs 27 · Core 5`)
so the exact counts the strip carried aren't lost.

The map is fed by a new client-side derivation, `setsByMuscleGroup`, a sibling to the
existing `setsByCategory` in `lib/muscle-set-distribution.ts`. It tallies sets per the
catalog's fine-grained `muscle_groups` (11 values), which a body-map can shade per
region — finer than the six broad `CATEGORIES`, which collapse all leg regions into
one blob. The component maps each `muscle_group` to the library's slug taxonomy,
buckets per-region set counts into four intensity tiers relative to the session's
busiest region (so the worked regions always glow), and passes the v0.4 steel-blue
ramp.

**Expandable sets.** `RecapExerciseList` keeps its collapsed top-set readout as the
default and gains a **chevron toggle** per row — a target distinct from the
hover-revealed edit pencil. Expanding reveals the exercise's actual sets (`set · reps
× weight`), with the PR set marked 🏆 / `--warning` by a client-side weight×reps match
(the `PersonalRecordEvent` carries no pointer to the set that broke it). Bodyweight
(non-positive weight → reps), per-dumbbell (append the clarifier), and superset blocks
are all respected, mirroring the collapsed readout's existing rules.

**Degrade.** An uncategorizable session (no mappable muscle groups) drops the map
entirely — no empty frame — exactly as the current strip drops.

Scope is `prog-strength-web` (component, derivation, page, list, dependency) plus
`prog-strength-docs` (the lift intensity ramp added to `design-system.md`, and this
SOW). The shared `WorkoutDetailsBody` (Workouts list, Calendar, Timeline) and the
pre-v0.4 `MuscleGroupRadarChart` are **untouched** — low blast radius.

## Goals and Non-Goals

### Goals

- Add `react-muscle-highlighter` (v1.2.0) to `prog-strength-web` dependencies.
- Add a four-stop **lift intensity ramp** to `design-system.md` and `app/globals.css`
  as `--discipline-lift-1` … `--discipline-lift-4` (dim → bright steel-blue), the
  canonical encoding of graded muscle intensity.
- Add `setsByMuscleGroup(workout, exercises)` to `lib/muscle-set-distribution.ts`:
  tally sets crediting each of the 11 catalog `muscle_groups` for a single workout,
  with a `hasData` flag, structurally parallel to the existing `setsByCategory`.
- Add a `MuscleBodyMap` component (`app/(app)/workouts/[id]/_components/muscle-body-map.tsx`):
  front + back `<Body>` instances, our `muscle_group` → library `Slug` mapping,
  set-volume → four-tier intensity bucketing, v0.4 ramp + near-black `defaultFill`,
  top-categories caption. Renders `null` when there is no mappable data.
- Integrate the map into `page.tsx` as a hero beside the stat line (desktop) / stacked
  above it (mobile), replacing the `WhatItTrained` strip slot.
- Add a per-row **expand-to-sets** chevron to `RecapExerciseList`: collapsed top-set
  default preserved, expansion reveals `reps × weight` per set, PR set client-matched
  and marked, bodyweight / per-dumbbell / superset readouts respected, edit pencil
  preserved as a distinct affordance.
- Render correctly across all visual states: the sparse focused session (the crux),
  a multi-category session, an uncategorizable session, expanded sets, long vs short
  sessions, and both mobile and desktop breakpoints.

### Non-Goals

- **No API change.** Sets, muscle groups, and PR events are already in the payload.
- **No new agent tooling.**
- The shared `WorkoutDetailsBody` and the pre-v0.4 `MuscleGroupRadarChart` are not
  modified, retoned, or removed by this SOW.
- The other four DX variants (`per-exercise-volume-bars`, `body-parts-radar`,
  `per-muscle-set-bars`, `quiet-inline-strip`) are not built.
- **Lift-TCX heart-rate / calories tiles** (`Avg HR · Max HR · Calories`) are out of
  scope — a later SOW. The stat line is left with room for them but they are not added.
- Warm-up-set distinction in the expanded view (no warm-up flag exists; the ramp is
  reconstructed) is out of scope.
- A user gender preference for the figure is out of scope; the figure uses the
  library's default (`gender="male"`).

## Implementation Details

### Dependency (`prog-strength-web`)

Add `react-muscle-highlighter@^1.2.0`. It is a client-only SVG component; the detail
page is already `"use client"`, and `MuscleBodyMap` carries the `"use client"`
directive, so there is no SSR concern. Verify the lockfile and that the typed default
export `Body` plus `ExtendedBodyPart` / `Slug` import cleanly.

### Design system — lift intensity ramp (`prog-strength-docs`, `prog-strength-web`)

The lift hue today is a single triplet (`--discipline-lift-bg #1b1f2e`,
`--discipline-lift-fg #aab4dd`, `--discipline-lift-dot #9aa6d6`) with no graded ramp.
Add a four-stop ramp interpolating the lift hue from dim (near the unworked body) to
bright (`--discipline-lift-fg`), so "harder-worked = more saturated steel-blue":

```css
--discipline-lift-1: #39405a; /* tier 1 — lightly worked  */
--discipline-lift-2: #5a6493; /* tier 2                    */
--discipline-lift-3: #7d88c2; /* tier 3                    */
--discipline-lift-4: #aab4dd; /* tier 4 — hardest worked (= --discipline-lift-fg) */
```

Document the ramp in `design-system.md` under the discipline-hue section as the
canonical encoding of graded intensity (reusable by any future per-intensity graphic),
and mirror the four custom properties (and their `--color-discipline-lift-*` aliases)
in `app/globals.css`. Exact stop values are finalized against the v0.4 palette during
the design-system edit (see Open Questions).

### Data — `setsByMuscleGroup` (`prog-strength-web/lib/muscle-set-distribution.ts`)

Add a sibling to `setsByCategory` that tallies at the fine `muscle_groups` grain
instead of collapsing to the six `CATEGORIES`:

```ts
export type MuscleGroupSetCount = { muscleGroup: string; value: number };

export function setsByMuscleGroup(
  workout: Workout,
  exercises: Exercise[],
): { data: MuscleGroupSetCount[]; hasData: boolean };
```

For each `WorkoutExercise`, resolve the catalog entry and credit **each** of its
`muscle_groups` with the exercise's set count (a compound lift contributes to every
group it hits — same crediting rule as `setsByCategory`). `hasData` is false when no
exercise resolves to any mappable group. The 11 keys are the catalog `muscle_groups`
values already enumerated in `lib/muscle-categories.ts`'s `MAP`: `chest`, `back`,
`shoulders`, `biceps`, `triceps`, `forearms`, `core`, `quads`, `hamstrings`,
`glutes`, `calves`. This is the single source of truth feeding `MuscleBodyMap`.

### Component — `MuscleBodyMap` (`prog-strength-web/app/(app)/workouts/[id]/_components/muscle-body-map.tsx`)

A `"use client"` component, page-scoped (low blast radius), taking `{ workout,
exercises }`.

**Muscle-group → library slug mapping.** A local constant maps each of our 11
`muscle_groups` to one or more `react-muscle-highlighter` `Slug` values. Where one
group spans several anatomical regions, every mapped slug inherits that group's
intensity tier:

```ts
const GROUP_TO_SLUGS: Record<string, Slug[]> = {
  chest:      ["chest"],
  back:       ["trapezius", "upper-back", "lower-back"],
  shoulders:  ["deltoids"],
  biceps:     ["biceps"],
  triceps:    ["triceps"],
  forearms:   ["forearm"],
  core:       ["abs", "obliques"],
  quads:      ["quadriceps"],
  hamstrings: ["hamstring"],
  glutes:     ["gluteal"],
  calves:     ["calves"],
};
```

A test asserts every key of `lib/muscle-categories.ts`'s `MAP` has an entry here, so
a future catalog muscle can't silently fall off the figure.

**Intensity tiering.** Call `setsByMuscleGroup`; if `!hasData`, render `null`. Otherwise
bucket each group's set count into one of four tiers **relative to the session's
busiest region** (`tier = ceil(value / maxValue * 4)`, clamped to 1–4), so the
worked regions of even a sparse, focused session reach the bright end of the ramp and
glow. (Relative-to-session tiering is intentional — a focused leg day should read as
emphasis, not as a uniformly dim "light day"; see Open Questions for the fixed-threshold
alternative considered.)

**Building `data` and rendering.** Expand each group to its slugs, each carrying the
group's tier as `intensity`, into a single `ExtendedBodyPart[]`. Render two `<Body>`
instances, `side="front"` and `side="back"`, both fed the same `data` (the library
draws only the slugs present on each side). Props for v0.4:

- `colors={[ "#39405a", "#5a6493", "#7d88c2", "#aab4dd" ]}` — the lift intensity ramp,
  passed as **literal hex** mirroring the `--discipline-lift-1..4` tokens (see the
  `var()` verification note in Open Questions).
- `defaultFill` — a desaturated near-black slate (≈ `--surface-2`) so the unworked
  silhouette is visible against the page without competing with the worked regions.
- `border="none"`, `defaultStroke="none"` — flat, in keeping with the calm page.
- `scale` tuned so front + back sit comfortably beside the stat line.

Caption beneath the figure: the populated **categories** largest-first (reuse
`populatedCategories` from `lib/workout-recap.ts`) — i.e. `Legs 27 · Core 5` — so the
strip's exact counts survive in the new graphic.

### Page integration (`prog-strength-web/app/(app)/workouts/[id]/page.tsx`)

Replace the `<WhatItTrained ... />` call (and remove that local component, or repurpose
it as the caption inside `MuscleBodyMap`) with the body-map, laid out as a hero beside
the stat line. Wrap the existing `StatLine` and the new `MuscleBodyMap` in a responsive
container: a single column on mobile (map stacked **above** the stat line, matching the
variant), a two-column row on `md+` with the map to the side. The stat line's honest
tile-dropping is unchanged. When `MuscleBodyMap` returns `null` (uncategorizable
session), the stat line falls back to full width — no empty frame.

### Expand-to-sets (`prog-strength-web/app/(app)/workouts/[id]/_components/recap-exercise-list.tsx`)

Add per-exercise expansion while keeping the collapsed top-set readout as the default:

- Local expanded-state keyed per exercise (within a render group). A **chevron button**
  on each exercise line toggles it — placed distinctly from the hover-revealed edit
  pencil so the two affordances never collide (the pencil keeps editing; the chevron
  reveals sets). Wire `aria-expanded` and keyboard activation.
- Expanded content: one row per `WorkoutSet` in order, `set # · reps × weight`
  formatted via `fmtWeight`. Non-positive weight reads `reps reps` (bodyweight);
  per-dumbbell exercises append the existing " per dumbbell" clarifier (reuse the
  `isPerDumbbell` rule already local to this file).
- **PR row.** For each set, mark it 🏆 in `--warning` when it matches an entry in
  `workout.personal_records_set` for this `exercise_id` by `weight` **and** `reps`
  (and `unit`). Because the event carries no pointer to the originating set, this is a
  client match: if multiple sets tie the PR weight×reps, mark the first (the top set
  via `topSetIndex` takes precedence); if none match (unit mismatch or reconstruction
  gap), mark nothing rather than guessing — the honest "couldn't match" behavior.
- Superset blocks keep their single block tag and grouping (`groupExercises` unchanged);
  each exercise within a superset expands independently.

### Visual states to verify (all in tests and on the preview)

- **Sparse focused session (the crux)** — `W5 D2 - Legs`, `Legs 27 · Core 5`: the
  figure glows at quads/hamstrings/calves/glutes + core, four other regions unshaded,
  reading as intentional.
- **Multi-category session** — a chest-&-back day hitting 4–5 categories, several PRs,
  a superset: the figure's "full" case.
- **Uncategorizable session** — no mappable groups → map drops, stat line full width.
- **Expanded sets** — collapsed default ↔ per-set detail, including superset,
  bodyweight, per-dumbbell, and the client-matched PR row.
- **Long vs short** — a 10-exercise day and a 3-exercise day both survive.
- **Both breakpoints** — map beside the stat line on desktop, stacked on mobile; edit
  affordances intact throughout.

### Tests

- `lib/muscle-set-distribution.test.ts` — extend with `setsByMuscleGroup`: per-group
  crediting, a compound lift hitting multiple groups, `hasData` false on an
  uncategorizable workout, stable ordering.
- `muscle-body-map.test.tsx` — worked slugs receive the expected intensity tier and
  unworked slugs fall to `defaultFill`; a multi-slug group (`back` → three slugs) all
  share one tier; relative tiering puts the busiest region at tier 4; `null` render on
  no data; caption shows populated categories largest-first. A guard test asserts the
  `GROUP_TO_SLUGS` mapping covers every `muscle-categories` `MAP` key.
- `recap-exercise-list.test.tsx` — extend: chevron toggles expansion; expanded rows
  show each set as `reps × weight`; bodyweight reads reps; per-dumbbell suffix present;
  PR set marked, no-match marks nothing, tie resolves to the top set; superset grouping
  preserved; edit pencil still fires `onEditGroup`.
- `page.test.tsx` — extend: the body-map renders beside the stat line and the old
  text strip is gone; an uncategorizable workout drops the map and the stat line spans
  full width.

## Rollout

Pure `prog-strength-web` frontend plus a `design-system.md` doc edit — no migration,
no backend change, no feature flag. Ships through the normal Vercel preview → main →
production flow. The design-system ramp edit lands in `prog-strength-docs` alongside
this SOW.

### Verification after rollout

On the Vercel preview, walk the workout detail page across the representative fixtures
(sparse leg day, multi-category day, uncategorizable session) at both breakpoints:
confirm the figure glows on the worked regions and reads as intentional on the sparse
day, the caption carries the category counts, every exercise expands to its true sets
with the PR row marked, and all edit affordances (pencil, + Add exercise, Edit details,
Delete) still work.

## Open Questions

- **Exact ramp hex stops.** The four `--discipline-lift-1..4` values above are a
  starting interpolation of the lift hue; finalize them against the v0.4 palette during
  the `design-system.md` edit so the dim end stays distinct from `defaultFill` and the
  bright end matches `--discipline-lift-fg`.
- **`colors` via `var()` vs literal hex.** The library applies ramp entries as SVG
  `fill` values. SVG `fill` accepts `var(--…)`, but to avoid a resolution surprise the
  plan passes literal hex mirroring the tokens. During implementation, verify whether
  `var(--discipline-lift-*)` strings resolve cleanly when passed through the library; if
  so, prefer them to keep a single source of truth.
- **Intensity tiering — relative vs fixed.** The plan tiers relative to the session's
  busiest region so focused days glow. The alternative (fixed cross-session thresholds,
  e.g. 1–2 / 3–5 / 6–9 / 10+ sets) gives comparability across sessions but leaves a
  light day uniformly dim. Confirm relative tiering reads right on the preview; the
  threshold function is isolated and trivially swappable.
- **Hide non-muscle slugs?** The library's figure includes `head` / `hair` / `hands` /
  `feet` regions we never shade. Decide during the design-system check whether to
  `hiddenParts`-hide them for a cleaner muscle-only figure or keep the full silhouette
  (Strava keeps it) — a visual call, not a structural one.
