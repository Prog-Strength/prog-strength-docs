---
status: ready_for_implementation
repos:
  - prog-strength-api
  - prog-strength-web
  - prog-strength-docs
---

# Workouts Analytics Views

**Status**: Proposed · **Last updated**: 2026-06-02

## Introduction

The `/workouts` page in **Prog Strength**'s web app today shows a single chart at the top — Time Lifting, a weekly area chart of total training time across the active timeframe. It answers one question well ("am I going to the gym?") but leaves two other axes a lifter cares about untouched: *am I lifting more?* and *am I training my whole body, not just my favorite muscles?*

Volume (sum of `reps × weight` across every set) is the single most diagnostic number a lifter has about whether they're progressing. Muscle-group distribution is the next most diagnostic — chest/back imbalance and skipped legs are programming failures that compound over months and are nearly invisible without a visual. Neither metric is hard to compute from the workouts data the page already loads; both are valuable enough that not showing them feels like a gap.

This SOW adds a Volume area chart and a Body Parts radar chart as sibling views to the existing Time Lifting chart, behind a three-icon view switcher with Time Lifting as the default. Each workout card in the list below also gets a small "Total Volume" badge so the user can correlate single-session totals with the Volume chart's trend. The summary header gains a small "PRs" count alongside the existing Total Time and Sessions stats, since `workouts[].personal_records_set` is already populated and surfacing the count is a one-line free win.

After this work ships, opening `/workouts` shows Time Lifting by default; clicking the dumbbell icon swaps in a Volume trend over the same window; clicking the radar icon shows the six-category muscle-group distribution. The list below remains exactly as it is structurally, just with a volume pill on each row.

## Proposed Solution

A new `WorkoutsAnalytics` component owns the top-of-page card. It receives the same `workouts` array the existing chart receives plus the catalog (`exercises`) and the user's preferred weight unit; it renders a shared summary header (Total Time, Sessions, PRs), a row of three icon buttons, and one of three sibling chart components depending on the active view. The three charts are:

- `WorkoutDurationChart` — the existing chart, lightly trimmed so its internal card-chrome (header, summary stats) moves up into `WorkoutsAnalytics`. The drawing logic stays put.
- `WorkoutVolumeChart` — new. Same recharts `AreaChart` structure as duration, weekly buckets matching the existing X-axis exactly. Y-axis is total volume in the user's preferred unit (lb or kg). Empty weeks render as zero so the line stays continuous over gaps.
- `MuscleGroupRadarChart` — new. Recharts `RadarChart` with six axes (Chest, Back, Shoulders, Arms, Legs, Core). Each axis value is the count of sets touching that category over the active timeframe. Compound exercises that hit multiple categories credit each one.

Two new shared utility modules live under `lib/`:

- `lib/workout-volume.ts` — `workoutVolume(workout, displayUnit)` and `setsVolume(sets, displayUnit)`. Single source of truth for the conversion math; consumed by both the volume chart and the per-card badge.
- `lib/muscle-categories.ts` — the 11-to-6 mapping and a `categorize(muscleGroup)` helper. Consumed by the radar chart.

The user's preferred weight unit comes from a new `GET /me` endpoint on the API — the field exists on the `User` struct already but isn't currently exposed via HTTP (only nested resources like `/me/headline-exercises` and `/me/macro-goals` are). The endpoint is small, returns the authed user object, and is the right home for any future "user-scoped read" the frontend needs.

Each workout card in the existing list gains an inline "Total Volume" pill formatted like `12,450 lb · Total Volume` (or `kg`), positioned alongside the existing "X sets across Y exercises" summary. The expandable detail panel stays unchanged.

The summary header at the top of the analytics card gains a third stat block: `PRs` count = `workouts.reduce((sum, w) => sum + w.personal_records_set.length, 0)`. Renders only when `> 0`.

### Critical implementer note

The web repo's `AGENTS.md` and `CLAUDE.md` are explicit: **"This is NOT the Next.js you know"**. The version in use has breaking changes from training-data Next.js — APIs, conventions, and file structure may all differ. Before writing any code in `prog-strength-web`, the implementer must consult the relevant guide in `node_modules/next/dist/docs/` and heed any deprecation notices. This applies to component patterns, data-fetching hooks, route handlers, and anything else that historically had a "standard" Next.js form.

## Goals and Non-Goals

### Goals

- Add a `GET /me` endpoint on `prog-strength-api` returning the authed user object (including `weight_unit`). Mounted under the existing JWT-gated router group; uses the `auth.RequireUser` middleware that already protects `/me/headline-exercises`.
- Add a `User` type and `getMe(token)` function to `prog-strength-web/lib/api.ts` mirroring the new endpoint.
- Add `lib/workout-volume.ts` with `workoutVolume(workout, displayUnit)` and `setsVolume(sets, displayUnit)`. Sets in the non-display unit get converted using the constant `2.20462` (lb per kg). Returns a plain `number`.
- Add `lib/muscle-categories.ts` exporting the ordered tuple `["Chest", "Back", "Shoulders", "Arms", "Legs", "Core"]` and a `categorize(muscleGroup)` helper that maps each of the 11 catalog enum values to its category (or returns `null` for unknown values, which the radar then drops).
- Add `components/workout-volume-chart.tsx` — a recharts `AreaChart` with weekly buckets matching the existing duration chart's X-axis logic exactly. Reads from a `workouts: Workout[] | null`, `days: number | null`, `displayUnit: "lb" | "kg"`, `truncated: boolean`, `fetchLimit: number` props contract that mirrors `WorkoutDurationChart`'s.
- Add `components/muscle-group-radar-chart.tsx` — a recharts `RadarChart` rendering set counts per category. Reads `workouts: Workout[] | null`, `exercises: Exercise[]`, props contract that follows the same pattern.
- Add `components/workouts-analytics.tsx` — wrapper component that owns the view-switcher state, renders the shared summary header (Total Time, Sessions, PRs when > 0), the three icon buttons, and the active chart. The three existing chart components live as children; the wrapper passes only the props each one needs.
- Refactor `components/workout-duration-chart.tsx` so its outer card chrome (border, padding, summary header) moves into `WorkoutsAnalytics`. The chart-drawing innards stay. Function signature stays compatible with the new wrapper's prop contract.
- Wire `WorkoutsAnalytics` into `app/(app)/workouts/page.tsx`: replace the existing `<WorkoutDurationChart />` slot, fetch the user once on mount via `getMe`, pass `userWeightUnit` to the wrapper. The existing `workouts` and `exercises` state stays as the single source of truth.
- Add the Total Volume pill to each workout card in the existing list. Formatted with thousands separators: `12,450 lb`. Uses `workoutVolume(w, displayUnit)` from the new util.
- Add the `PRs` summary stat to the `WorkoutsAnalytics` header. Renders the third stat block only when the count is greater than zero.
- Choose three icons consistent with the project's icon library (likely `lucide-react` — the implementer should grep `package.json` and existing components to confirm). Suggested mapping: clock for Time Lifting, dumbbell or trending-up for Volume, target or radar for Body Parts.

### Non-Goals

- **No new aggregate metrics beyond the three views + PR count.** Workout-frequency / streak charts, estimated-1RM-of-headline-lifts trends, average-intensity calculations, and top-exercise-by-frequency lists are all explicitly out of scope.
- **No URL persistence of the active view.** Refreshing the page returns to Time Lifting. Matches the existing timeframe-pill behavior on this page, which also resets on refresh. Persisting via search params is a clean follow-up if the user finds themselves wanting it.
- **No mobile-app changes.** This SOW is web-only. The mobile app's workouts view can adopt the same shape in a follow-up.
- **No backend volume aggregation endpoint.** Volume is computed client-side from the workouts array the page already loads. Server-side aggregation is appropriate only if/when single-fetch payload sizes become a real perf problem; at the existing 100-workout fetch cap that's not the case.
- **No bidirectional unit conversion in the workout-volume util.** It accepts a `displayUnit` and converts sets only when their unit differs. There's no `lb → kg` user-side toggle on `/workouts`; the conversion direction is always "toward the user's preference."
- **No new exercise-catalog data.** The 11-to-6 muscle-group categorization happens on the frontend. The catalog stays as-is.
- **No animation of view-switcher transitions.** Clicking an icon swaps the chart with no crossfade. Animation can land later if it adds value.
- **No CSV export of volume or muscle-group totals.** Reading the chart is enough at v1.
- **No "compare two windows" overlay.** Each view shows one window. Time-series comparison is a separate, larger UX undertaking.
- **No user preference for radar metric** (set-count vs volume). v1 ships set-count only.
- **No tooltip enhancements on the radar.** Just default recharts hover behavior.
- **No reordering or hiding of the volume badge per user preference.** Always shown.

## Implementation Details

### API change

`prog-strength-api/internal/user/handler.go` (new file):

```go
package user

import (
    "errors"
    "net/http"

    "github.com/go-chi/chi/v5"
    "github.com/jwallace145/progressive-overload-fitness-tracker/internal/auth"
    "github.com/jwallace145/progressive-overload-fitness-tracker/internal/httpresp"
)

type Handler struct {
    repo Repository
}

func NewHandler(repo Repository) *Handler { return &Handler{repo: repo} }

// Mount wires GET /me. The router is the JWT-gated group; the
// handler reads the user id from request context (auth.UserIDFrom).
func (h *Handler) Mount(r chi.Router) {
    r.Get("/me", h.getMe)
}

func (h *Handler) getMe(w http.ResponseWriter, r *http.Request) {
    userID, ok := auth.UserIDFrom(r.Context())
    if !ok {
        httpresp.ServerError(w, r.Context(), "missing user in context",
            errors.New("auth middleware not applied"))
        return
    }
    u, err := h.repo.GetByID(r.Context(), userID)
    if err != nil {
        if errors.Is(err, ErrNotFound) {
            httpresp.Error(w, http.StatusNotFound, "user not found")
            return
        }
        httpresp.ServerError(w, r.Context(), "get user", err)
        return
    }
    httpresp.OK(w, "got user", u)
}
```

Mount via `user.NewHandler(userRepo).Mount(r)` inside the existing `r.Group(func(r chi.Router) { r.Use(auth.RequireUser(jwtSecret)) ... })` block in `internal/server/server.go`. Handler test follows the same pattern as the existing workout/nutrition handler tests.

### Web `lib/api.ts` additions

Add the `User` type matching the API's JSON shape:

```ts
export type User = {
  id: string;
  email: string;
  display_name?: string;
  weight_unit: "lb" | "kg";
  created_at: string;
  updated_at: string;
};

export async function getMe(token: string): Promise<User> {
  const resp = await fetch(`${config.apiUrl}/me`, {
    headers: { Authorization: `Bearer ${token}` },
  });
  if (!resp.ok) throw new Error(`getMe: ${resp.status}`);
  const envelope = await resp.json();
  return envelope.data as User;
}
```

The exact envelope shape and config import should follow the pattern of the existing `listWorkouts` / `getMacroGoals` calls in the same file.

### `lib/workout-volume.ts`

```ts
import type { Workout, WorkoutSet } from "@/lib/api";

const LB_PER_KG = 2.20462;

function convert(weight: number, from: "lb" | "kg", to: "lb" | "kg"): number {
  if (from === to) return weight;
  if (from === "kg" && to === "lb") return weight * LB_PER_KG;
  return weight / LB_PER_KG;
}

export function setsVolume(
  sets: WorkoutSet[],
  displayUnit: "lb" | "kg",
): number {
  return sets.reduce(
    (total, s) => total + s.reps * convert(s.weight, s.unit, displayUnit),
    0,
  );
}

export function workoutVolume(
  workout: Workout,
  displayUnit: "lb" | "kg",
): number {
  return workout.exercises.reduce(
    (total, e) => total + setsVolume(e.sets, displayUnit),
    0,
  );
}
```

Bodyweight sets (where `weight === 0`) contribute zero — correct behavior; volume isn't a meaningful metric for pull-ups in this context.

### `lib/muscle-categories.ts`

```ts
export const CATEGORIES = [
  "Chest",
  "Back",
  "Shoulders",
  "Arms",
  "Legs",
  "Core",
] as const;

export type MuscleCategory = (typeof CATEGORIES)[number];

const MAP: Record<string, MuscleCategory> = {
  chest: "Chest",
  back: "Back",
  shoulders: "Shoulders",
  biceps: "Arms",
  triceps: "Arms",
  forearms: "Arms",
  core: "Core",
  quads: "Legs",
  hamstrings: "Legs",
  glutes: "Legs",
  calves: "Legs",
};

export function categorize(muscleGroup: string): MuscleCategory | null {
  return MAP[muscleGroup] ?? null;
}
```

### Volume chart component contract

```tsx
export function WorkoutVolumeChart({
  workouts,
  days,
  displayUnit,
  truncated,
  fetchLimit,
}: {
  workouts: Workout[] | null;
  days: number | null;
  displayUnit: "lb" | "kg";
  truncated: boolean;
  fetchLimit: number;
}): JSX.Element
```

Internally mirrors `WorkoutDurationChart`'s weekly-bucket aggregation; the per-bucket value is `workouts.filter(w => inThisWeek(w)).reduce(workoutVolume sum)` instead of total minutes. The Y-axis label is the display unit. Chart height 200 matching the existing chart. Empty-state and loading-state copy align with the existing chart's wording.

### Radar chart component contract

```tsx
export function MuscleGroupRadarChart({
  workouts,
  exercises,
}: {
  workouts: Workout[] | null;
  exercises: Exercise[];
}): JSX.Element
```

The component builds a `Record<MuscleCategory, number>` initialized to `{ Chest: 0, Back: 0, ... }`. For each workout, for each `WorkoutExercise`, look up the exercise in the catalog by `exercise_id`; for each `muscle_group` on the exercise, call `categorize()` and add `e.sets.length` to that category's tally. Unknown muscle groups (categorize returns null) are silently dropped — they don't contribute. The radar data is the categories array mapped to `{ category, value }` rows.

Recharts:
```tsx
<RadarChart data={data}>
  <PolarGrid />
  <PolarAngleAxis dataKey="category" />
  <PolarRadiusAxis />
  <Radar dataKey="value" stroke="#3b82f6" fill="#3b82f6" fillOpacity={0.32} />
  <Tooltip />
</RadarChart>
```

Loading + empty states follow the existing chart's conventions.

### `WorkoutsAnalytics` wrapper

```tsx
export function WorkoutsAnalytics({
  workouts,
  exercises,
  displayUnit,
  days,
  truncated,
  fetchLimit,
}: {
  workouts: Workout[] | null;
  exercises: Exercise[];
  displayUnit: "lb" | "kg";
  days: number | null;
  truncated: boolean;
  fetchLimit: number;
}): JSX.Element {
  const [view, setView] = useState<"time" | "volume" | "muscle">("time");
  // ...summary stats (total time, sessions, PR count)
  // ...icon button row
  // ...switch (view) { case "time": ... case "volume": ... case "muscle": ... }
}
```

Owns the shared summary header. The three child charts no longer render their own card chrome — they render only the chart area + chart-specific summary fields if any.

PR count: `workouts?.reduce((s, w) => s + w.personal_records_set.length, 0) ?? 0`. Only render the stat block when `prCount > 0` to avoid showing `0 PRs` as a default visual fact.

### Workout card volume badge

In the workout-list row (currently in `app/(app)/workouts/page.tsx`'s render output — implementer should locate the exact JSX), add a pill alongside the existing "X sets across Y exercises" summary line:

```tsx
<span className="rounded-full bg-[var(--surface)] px-2 py-0.5 text-xs tabular-nums text-[var(--muted)]">
  {workoutVolume(w, displayUnit).toLocaleString(undefined, { maximumFractionDigits: 0 })} {displayUnit} · Total Volume
</span>
```

Exact styling should follow the existing inline-metadata pills on the row (the implementer should match the visual treatment of any existing badge — set count, date stamp, etc.).

### Page integration

`app/(app)/workouts/page.tsx`:

- Add `user` to state: `const [user, setUser] = useState<User | null>(null);`
- Add a `useEffect` on mount that calls `getMe(token).then(setUser).catch(...)`. Mirror the existing exercise-catalog fetch's error handling.
- Compute `const displayUnit = user?.weight_unit ?? "lb";` — falls back to lb during the brief loading window.
- Replace the existing `<WorkoutDurationChart workouts={...} days={...} truncated={...} fetchLimit={...} />` with `<WorkoutsAnalytics workouts={workouts} exercises={exercises} displayUnit={displayUnit} days={days} truncated={truncated} fetchLimit={FETCH_LIMIT} />`.
- In the workout-list row JSX, add the volume pill described above.

### Tests

Per-module:

- `lib/workout-volume.ts`: unit tests covering (a) all-lb pure case, (b) all-kg → lb conversion, (c) all-lb → kg conversion, (d) mixed-unit set list within one workout, (e) bodyweight (weight=0) set yielding zero contribution, (f) empty exercises array → 0.
- `lib/muscle-categories.ts`: unit tests covering (a) each of the 11 catalog enums maps to its expected category, (b) an unknown string returns null.
- `components/workout-volume-chart.tsx`: a snapshot test plus a behavioral test that the rendered weekly value equals the manually-summed volume for a fixture of 3 workouts in different weeks.
- `components/muscle-group-radar-chart.tsx`: behavioral test that two workouts hitting chest+back accumulate the expected set counts on each axis; behavioral test that exercises with unknown muscle groups don't contribute.
- `components/workouts-analytics.tsx`: behavioral test that clicking each icon swaps the visible chart component; test that the PRs stat block renders only when count > 0; test that the radar chart only receives the props it needs (no leak of displayUnit etc).
- API: handler test for `GET /me` returning 200 with the user payload for an authed request and 404 if the user repo returns `ErrNotFound`.

The repo's test framework is the existing one used in `components/*-test.tsx` / `lib/*-test.ts` files; the implementer should locate and follow that pattern. If no test files exist for these areas, defer to the project's `package.json` scripts for the canonical command.

### Layout / spacing

The summary header (Total Time | Sessions | PRs) goes inside the existing card border at the top of `WorkoutsAnalytics`. The three icon buttons sit below the summary, above the active chart, in a row centered or left-aligned matching the timeframe-pill row's alignment lower on the page. Active icon gets the accent color (`var(--accent)` or whatever the project uses); inactive icons render with `var(--muted)`. Spacing follows the existing component's `mt-3` / `mt-4` cadence.

### Critical reminder for the implementer

**This is not the Next.js you know.** Before writing any code in `prog-strength-web`:

1. Read the relevant guide in `node_modules/next/dist/docs/`.
2. Heed any deprecation notices the compiler prints.
3. When in doubt, look at the existing components and pages in this repo for the canonical patterns — those are known to work in this Next.js version. Do not introduce patterns from your training data that contradict what you see in-repo.

This is the single most important constraint on this SOW. A patch that "works in standard Next.js" but breaks the project's actual conventions wastes a worker run.

## Open Questions

1. **Icon library.** `lucide-react` is the strong default in projects of this shape; the implementer should grep `package.json` and confirm before importing. If the project uses a different icon set, swap accordingly. Recommendation: use whatever icon library is already a dependency; do NOT add a new dependency just for these three icons.

2. **Exact pill styling for the volume badge.** The implementer should match the visual treatment of any existing inline metadata pills on the workout row rather than introduce a new visual idiom. If no such pill exists today, match the timeframe-pill button styling at the top of the page.

3. **Empty / loading states inside the radar.** When `workouts === null` (loading) or the array yields all-zero categories (no exercises hit any category in the window), the radar renders an empty-state message in line with the existing duration chart's empty-state. The recommended copy: `"No completed workouts in this window."` (matching duration chart) for the loading-equivalent case; `"Workouts in this window don't have categorizable muscle groups."` for the all-zero case. Implementer can fine-tune.
