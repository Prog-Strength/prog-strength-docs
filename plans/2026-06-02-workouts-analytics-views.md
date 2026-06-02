# Implementation Plan: Workouts Analytics Views

**SOW**: [`sows/workouts-analytics-views.md`](../sows/workouts-analytics-views.md) ·
**Status**: ready_for_implementation · **Created**: 2026-06-02

Adds a Volume area chart and a Body Parts radar chart as sibling views to the
existing Time Lifting chart on `/workouts`, behind a three-icon view switcher
(Time Lifting default). Each workout card gains a "Total Volume" pill, the
analytics header gains a "PRs" count, and the API exposes the authed user's
`weight_unit` via a new `GET /me`.

Repos: `prog-strength-api`, `prog-strength-web`, `prog-strength-docs`.

---

## Findings from repo reconnaissance

These shape the plan and resolve the SOW's open questions:

1. **No `lucide-react`** (`prog-strength-web/package.json`). Icons in this repo
   are hand-rolled inline `<svg>` components (see `ChevronIcon`, `PencilIcon`,
   `TrophyIcon`, `TrashIcon` in `app/(app)/workouts/page.tsx`). The three
   view-switcher icons (clock / dumbbell / radar) will be inline SVGs matching
   that idiom. **Do not add an icon dependency** (SOW Open Question 1).

2. **No web test runner.** `package.json` has no `test` script and no test
   files exist anywhere in the repo. The established verification convention in
   merged web PRs (#1, #2) is `npm run typecheck` + `npm run lint` + a manual
   test plan in the PR body. Standing up a test framework (vitest/jest +
   config) is a new dependency and a scope expansion the SOW doesn't authorize
   ("do NOT add a new dependency"). **Decision:** web changes are verified by
   `npm run typecheck`, `npm run lint`, and `npm run build`, plus a documented
   manual test plan — mirroring the repo's actual convention over the SOW's
   per-component unit-test list, exactly as the SOW's "this is not the Next.js
   you know / follow in-repo patterns" directive instructs. The Go handler
   test *is* written — `go test` is the API repo's established convention
   (`internal/.../*_test.go` everywhere).

3. **Next.js 16.2.6, React 19.2.4, recharts 3.8.1.** All recharts radar
   exports (`RadarChart`, `Radar`, `PolarGrid`, `PolarAngleAxis`,
   `PolarRadiusAxis`, `Tooltip`) are present. The existing page is a
   `"use client"` component using `useEffect`/`useState` direct-fetch — the
   canonical in-repo pattern; new code mirrors it. `node_modules/next/dist/docs/`
   exists and was consulted; nothing in our surface (client components, charts,
   client-side fetch) hits a deprecation. (The docs' embedded `unstable_instant`
   / instant-navigation hint is about navigation perf and is **not** relevant
   to this work.)

4. **API envelope.** `lib/api.ts` wraps every call through `unwrap<T>(resp,
   empty)` which reads `body.data`. The SOW's illustrative `getMe` uses a raw
   `fetch`/`envelope.data`; the plan uses the in-repo `unwrap` helper instead,
   matching `getMacroGoals`.

5. **`user.User` already has correct JSON tags** (`id`, `email`,
   `display_name`, `weight_unit`, `created_at`, `updated_at`; `DeletedAt` is
   `json:"-"`). The handler can return the struct directly — no DTO needed —
   and the shape matches the web `User` type. Note `display_name` is a
   non-pointer `string` server-side (always present); the web type marks it
   optional, which is a safe superset.

6. **11 muscle-group enum values** in `internal/exercise/muscle_group.go`:
   `chest, back, shoulders, biceps, triceps, forearms, core, quads,
   hamstrings, glutes, calves` — exactly matching the SOW's `MAP`.

---

## Task breakdown

Each task is implemented by a subagent, then spec-reviewed and
code-quality-reviewed before the next dependent task starts. Tasks within a
phase that don't depend on each other run in parallel.

### Phase 1 — leaf modules (parallel)

**T-API — `GET /me` (prog-strength-api)**
- New `internal/user/handler.go`: `Handler{repo Repository}`, `NewHandler`,
  `Mount(r)` registering `r.Get("/me", h.getMe)`, and `getMe` reading
  `auth.UserIDFrom`, calling `repo.GetByID`, mapping `ErrNotFound`→404, other
  errors→`ServerError`, success→`httpresp.OK(w, "got user", u)`.
- Wire in `internal/server/server.go` inside the existing
  `r.Group(func(r){ r.Use(auth.RequireUser(jwtSecret)); ... })` block:
  `user.NewHandler(userRepo).Mount(r)`.
- `internal/user/handler_test.go`: httptest-driven, injecting the user id via
  `auth.WithUserID(req.Context(), id)` against a seeded `MemoryRepository`.
  Cases: 200 with payload for an authed user; 404 when the repo has no such
  user (`ErrNotFound`).
- Gate: `go build ./...` and `go test ./...` green.

**T-LIB — web lib modules (prog-strength-web)**
- `lib/api.ts`: add `User` type (per SOW) and `getMe(token)` using the existing
  `unwrap` helper (`return unwrap<User>(resp, ...)` is awkward without a sane
  empty — instead follow the `getWorkout` pattern: `unwrap<User | null>(resp,
  null)` then throw if null). Match surrounding style.
- `lib/workout-volume.ts`: `setsVolume` / `workoutVolume` + private `convert`
  using `LB_PER_KG = 2.20462`, exactly as the SOW specifies.
- `lib/muscle-categories.ts`: `CATEGORIES` tuple, `MuscleCategory` type,
  `categorize(muscleGroup)`, exactly as the SOW specifies.
- Gate: `npx tsc --noEmit` green.

**T-DOC — this plan** (already underway).

### Phase 2 — chart components (parallel; depend on T-LIB)

**T-VOL — `components/workout-volume-chart.tsx`**
- Same `AreaChart` structure, props contract, weekly Monday-bucket aggregation,
  numeric X-axis, `CHART_HEIGHT = 200`, loading/empty/truncated copy as
  `workout-duration-chart.tsx`. Per-bucket value = sum of `workoutVolume(w,
  displayUnit)` for completed workouts in that week. Y-axis tick + tooltip
  format the display unit with thousands separators. Empty weeks render zero.
  Emits **only the chart area + its own summary fields** (no card chrome) so it
  drops into the wrapper — see Phase 3 contract note below.

**T-RADAR — `components/muscle-group-radar-chart.tsx`**
- `RadarChart` with the six `CATEGORIES` axes. Build `Record<MuscleCategory,
  number>` zero-initialized; for each workout → each `WorkoutExercise`, look up
  the catalog `Exercise` by `exercise_id`, and for each `muscle_group` call
  `categorize()` and add `e.sets.length` to that category. Unknown groups
  dropped. Recharts block per SOW. Loading state (`workouts === null`) and
  all-zero empty state with the SOW's recommended copy.

### Phase 3 — wrapper + duration refactor (sequential; coupled)

**T-WRAP — refactor `workout-duration-chart.tsx` + add
`components/workouts-analytics.tsx`**
- Move the card chrome (outer `<section>` border/padding + the Total Time /
  Sessions summary header) **out** of `WorkoutDurationChart` and **into**
  `WorkoutsAnalytics`. `WorkoutDurationChart` keeps its drawing innards
  (AreaChart + aggregation + truncation note) and renders just the chart area;
  its prop contract stays compatible with the wrapper.
- `WorkoutsAnalytics` props per SOW (`workouts`, `exercises`, `displayUnit`,
  `days`, `truncated`, `fetchLimit`). State `view: "time" | "volume" |
  "muscle"` (default `"time"`). Renders: shared summary header (Total Time,
  Sessions, and PRs **only when `prCount > 0`**), a row of three icon buttons
  (active = accent, inactive = muted), and a `switch (view)` selecting the
  active child chart. Passes each child **only the props it needs** — the radar
  receives `{ workouts, exercises }`, no `displayUnit` leak.
- Total Time / Sessions computation moves into the wrapper (lift the existing
  `summarize` logic or compute the two scalars the header needs). The
  Monday-week aggregation needed by each chart stays inside that chart.

### Phase 4 — page integration (depends on all)

**T-PAGE — `app/(app)/workouts/page.tsx` + volume pill**
- Add `user` state + a mount `useEffect` calling `getMe(token)` (mirror the
  catalog fetch's error handling; 401 → clearToken + redirect like the
  workouts fetch). `const displayUnit = user?.weight_unit ?? "lb"`.
- Replace `<WorkoutDurationChart .../>` with `<WorkoutsAnalytics workouts=...
  exercises={exercises} displayUnit={displayUnit} days=... truncated=...
  fetchLimit={FETCH_LIMIT} />`.
- Add the Total Volume pill to `WorkoutRow`'s metadata line, styled to match
  existing inline metadata (the muted `text-xs` summary line / pill idiom),
  using `workoutVolume(w, displayUnit).toLocaleString(undefined, {
  maximumFractionDigits: 0 })`. `WorkoutRow` needs `displayUnit` passed in.
- Gate: `npm run typecheck`, `npm run lint`, `npm run build` all green.

### Phase 5 — PRs

Push each branch (`feat/workouts-analytics-views`) and open one PR per modified
repo against `main`, following each repo's PR conventions:
- API/docs: conventional-commit title (`feat(api): ...`, `docs: ...`), Summary +
  Test plan with `go test ./...`.
- Web: `feat(workouts): ...` title, Summary bullets + Test Plan (typecheck /
  lint / build + manual checks). Reference the SOW and this plan.

---

## Risks / watch-items

- **Header lift coupling (T-WRAP).** The duration chart's `summarize` currently
  produces `totalMinutes`, `sessionCount`, `openWorkouts`. Moving the header up
  means the wrapper needs those scalars while the chart still needs its weekly
  buckets. Keep the chart's internal aggregation; compute the header scalars in
  the wrapper from `workouts` to avoid a fragile prop hand-off. Preserve the
  "+ N open" affordance.
- **Prop-leak test intent (SOW).** The radar must not receive `displayUnit`.
  Enforced structurally by the wrapper's call site.
- **Volume on mixed units.** `convert` only fires when `s.unit !== displayUnit`;
  bodyweight (`weight 0`) contributes 0. Covered by the util's design.
- **`personal_records_set` is always present** (non-optional array) per
  `lib/api.ts`, so `prCount` reduce needs no null guard beyond the `workouts?`
  itself.
