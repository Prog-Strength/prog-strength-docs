# Implementation Plan: Progress Page Modernization

**SOW:** `sows/progress-page-modernization.md`
**Date:** 2026-06-10
**Repos:** `prog-strength-api`, `prog-strength-web`, `prog-strength-docs`

## Overview

Rework the Progress page around its actual job — *am I getting stronger?* — in
three layers: API correctness fixes (per-exercise trends, aggregates, baseline
labeling), movement-pattern filter consolidation (12 muscle chips → 5 pattern
chips), and a chart rework (per-exercise lines instead of a cross-exercise
scatter+regression). Strength-only, read-only. No DB migrations.

## Module facts (verified)

- Go module: `github.com/jwallace145/progressive-overload-fitness-tracker`, Go 1.25.
- Progression compute lives in `internal/workout/muscle_group_progression.go`;
  `Trendline` + `normalizedRegressionLine` shape in `internal/workout/progression.go`.
- Handler `progression` in `internal/workout/handler.go`, mounted `GET /workouts/progression`.
- Error helpers: `internal/httpresp` — `Error(w, status, msg)` and
  `ErrorWithCode(w, status, msg, code)`.
- Muscle groups in `internal/exercise/muscle_group.go` (11 constants, switch-based `Valid()`,
  no `AllMuscleGroups()` yet).
- Exercise repo `ListOptions{ MuscleGroup MuscleGroup; Equipment Equipment }`;
  memory + sqlite implementations filter by single group.
- Web: `app/(app)/progress/page.tsx` (~1060 lines, all inline components, raw
  `useEffect`+`Promise.all` fetch — NOT TanStack Query yet). `lib/api.ts`
  `listProgression(token, muscleGroup, since?, until?)`.
- Web tests: vitest + @testing-library/react + jsdom. No MSW — fetch is stubbed
  and modules are mocked via `vi.mock`. QueryClientProvider already exists
  (`app/providers.tsx`).
- Web design tokens in `app/globals.css`: `--surface --border --muted --foreground
  --accent --accent-fg --danger --warning`. Reference pattern: `app/(app)/personal-records/`
  uses `_components/`, TanStack Query, `useSearchParams`, `router.replace`.

## Task breakdown

### API track (prog-strength-api)

**API-1: Movement-pattern type + exercise repo multi-group filter.**
- New `internal/exercise/movement_pattern.go`: `MovementPattern` string type;
  `push/pull/legs/core/all` constants; `AllMuscleGroups()` helper (returns all 11
  in catalog order); `MovementPatternMuscleGroups` map; `MovementPattern.Valid()`;
  `MovementPattern.MuscleGroups()` resolver.
- Extend `exercise.ListOptions` with `MuscleGroups []MuscleGroup` (keep single
  `MuscleGroup` for back-compat). Update memory + sqlite repos to filter by the
  slice (`IN (...)`) when set, falling back to single-group when only that is set.
- New `internal/exercise/movement_pattern_test.go`: table-driven resolution,
  `MovementAll` covers every catalog muscle, `Valid()` rejects unknowns.
- Reviews: spec + code-quality.

**API-2: Progression compute rework.**
- In `muscle_group_progression.go`: add `PerExerciseTrend`, `Aggregate`, `Filter`
  structs. Extend `MuscleGroupProgression` with `Filter`, `BaselineModel`,
  `PerExerciseTrends`, `Aggregate`; drop top-level `Trendline`.
- Per-exercise least-squares slope in months (x = months since `since`), reported
  in percentage points/month (×100). `null` slope+trendline when `session_count < 3`
  or degenerate. `min_sessions_threshold = 3`, `progressingSlopeThreshold = 0.25`.
- Aggregate: `lifts_tracked` (sessions ≥ 3), `lifts_progressing` (slope > 0.25),
  `median_slope_per_month` (median over qualifying; null if none),
  `min_sessions_threshold`.
- `ComputeMuscleGroupProgression` signature gains the resolved filter info so it can
  populate the `filter` block (movement_pattern OR muscle_group + muscle_groups_included).
- Update existing tests; add the SOW's new test cases.
- Reviews: spec + code-quality.

**API-3: Handler param routing.**
- `progression` handler: accept `movement_pattern` and `muscle_group`. Exactly-one
  rule: neither → 400 `missing_filter`; both → 400 `conflicting_filters`. Resolve
  movement_pattern → groups, query `ListOptions{MuscleGroups: groups}`. Pass filter
  context into compute. Legacy `muscle_group` path unchanged.
- Handler tests: push, all, missing, conflicting, legacy muscle_group.
- Reviews: spec + code-quality.

### Web track (prog-strength-web)

**WEB-1: api.ts types + listProgression.**
- Add `movement_pattern` option to `listProgression` (accept an options object or
  extend signature). New types: `PerExerciseTrend`, `ProgressionAggregate`,
  `ProgressionFilter`; extend `MuscleGroupProgression` (filter, baseline_model,
  per_exercise_trends, aggregate); drop top-level `trendline`.
- Extend `lib/api.test.ts`.
- Reviews: spec + code-quality.

**WEB-2: Progress page redesign into `_components/`.**
- Extract `page.tsx` shell + `_components/` (MovementPatternFilter, StatCards,
  ProgressChart, EstimatesTable, SetsTable, TablesSection). URL state
  `?pattern=&days=` via `useSearchParams` + `router.replace`. TanStack Query keyed
  `["progression", pattern, days]` and `["workouts", since, until]`, staleTime 60s.
- Five movement-pattern chips (neutral accent). Three stat cards (Lifts progressing,
  Strength trend, Best session) with tone selectors matching backend thresholds.
- Chart: one `<Line>` per exercise, dots, no dashed trendline, reference line at 1.0
  with updated label; legend direction indicators + slope readouts. Below-chart
  caption from `muscle_groups_included`. Empty/loading/error/all-below-threshold states.
- Table: rename "% Baseline" → "% of current capability" + caption.
- Tests: StatCards, ProgressChart, MovementPatternFilter, page integration.
- Reviews: spec + code-quality.

### Docs track (prog-strength-docs)

**DOCS-1:** Flip SOW frontmatter `status: shipped`, body `**Status**: Shipped`,
`**Last updated**: 2026-06-10`. Commit. (Done after impl PRs so the docs PR body
can reference them.)

## Verification gates

- API: `go build ./...`, `go vet ./...`, `gofmt -l`, `go test ./...` all green.
- Web: `npm run typecheck`, `npm run lint`, `npm test`, `npm run build` all green.

## Rollout / merge order

1. `prog-strength-api` (read-side; new load-bearing fields).
2. `prog-strength-web` (consumes new fields).
3. `prog-strength-docs` (status flip — the ship signal).
</content>
</invoke>
