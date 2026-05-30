# Workout List — Batched Hydration

**Status**: Draft · **Last updated**: 2026-05-30

## Introduction

Today, `GET /workouts` issues one SQL statement for the workout list, plus one statement per workout to load that workout's exercises, plus one statement per exercise to load that exercise's sets. For a representative single-user page — 50 workouts × 5 exercises × 8 sets — that's about **300 SQL statements** to satisfy a single HTTP request. The same shape exists on `GET /workouts/{id}` (proportionally smaller, since it's one workout) and on `GET /exercises` (one query per exercise to fetch muscle-group + equipment join rows).

Functionally it works at single-user scale. The smell only becomes a problem when the API is under any amount of concurrent load — which it just was, when the mobile app's tab-focus pattern stacked five concurrent `/workouts` calls and the nested-query pattern deadlocked the SQL connection pool. The drain-rows fix that shipped in `v0.28.2` and `v0.28.3` patched the deadlock specifically, but it left the underlying N+1 pattern in place. Each call still does ~300 statements; we just no longer hold connections across them.

This SOW turns the N+1 pattern into **three batched queries** regardless of how many workouts, exercises, or sets are involved. After this work ships, the per-request SQL statement count for `/workouts` is bounded — not proportional to data size — and the same applies to `/workouts/{id}` and `/exercises`. The benchmark that lands alongside the code is the documented proof.

## Proposed Solution

Replace the per-row fan-out with **batched parent → children loads**. For each list endpoint:

1. Run the parent query, drain it into a slice (already true after the deadlock fix).
2. Collect the parent IDs.
3. Run **one** batched query for all children of those parents, using an `IN (?, ?, …)` clause built from the ID slice.
4. Repeat for the grandchildren layer when applicable (sets → workout_exercises).
5. Assemble the tree in Go using `map[parentID][]child` lookups — `O(N)` once you have all the rows.

The result shape returned to the handler is identical to today's. No public API change, no migration, no DTO changes. The work is contained inside the SQLite repositories.

The naive concern with `IN (?, ?, …)` is SQLite's `SQLITE_MAX_VARIABLE_NUMBER` limit (default 32,766 since SQLite 3.32, 999 before that). For a page of at most 100 workouts × at most ~50 exercises = 5,000 workout_exercises, we're nowhere near the cap. The `LIMIT` on the workouts query is the bound; downstream IN-clauses are derived from that bounded set and stay small by construction.

## Goals and Non-Goals

### Goals

- `GET /workouts` issues at most three SQL statements (workouts, all workout_exercises, all sets) regardless of page size. Same for `GET /workouts/{id}` (with `IN (?)` degenerating to a single-ID lookup).
- `GET /exercises` issues at most three SQL statements (exercises, all muscle_groups, all equipment).
- Response shape and DTOs are byte-identical to the current implementation. Existing API integration tests pass without modification.
- A new benchmark in `internal/workout/` (and one in `internal/exercise/`) measures statement count + wall time before/after on a seed-data fixture. The benchmark's pre/post numbers are pasted into the merging PR description and into this SOW's `## Resolutions` section before flipping `Status` to `Shipped`.
- Connection-pool pressure under concurrent load drops in proportion. A k6 / vegeta load test of `GET /workouts` at 10 concurrent virtual users for 30s shows median statement count per request drop from ~301 to 3 with no errors.
- The drain-rows discipline introduced in `v0.28.2` / `v0.28.3` stays. The new helpers still avoid holding rows open across nested calls, even though there's nothing nested left to call.

### Non-Goals

- **Schema changes.** No new tables, no denormalization, no JSON-blob columns. The relational shape is fine; only the query strategy changes.
- **Caching layers.** No in-process LRU, no Redis. Catalog data could plausibly be cached because it's admin-curated and basically static, but that's a different SOW; this work is purely about query batching.
- **A query builder or ORM.** Hand-rolled SQL stays. The codebase has consciously avoided ORMs (per `AGENTS.md`) and adding one for three list endpoints would be a much bigger bet than this work warrants.
- **Pagination of the nested layers.** A request for "50 workouts with all their exercises" still gets all the exercises. The outer `LIMIT` is what bounds the response size; sub-limits would be a future concern if any workout ever holds an extreme number of exercises (none does today).
- **Touching repositories that don't have the N+1 pattern.** `nutrition`, `bodyweight`, `user`, and `telemetry` are already single-table queries; this work doesn't visit them.
- **Single-mega-JOIN approach** (one SQL statement returning a Cartesian product of parents × children × grandchildren, deduped in Go). Considered and rejected — see the Algorithms section for why.

## Implementation Details

### Inventory

Three callers have the N+1 pattern today:

| Function | File | Layers |
| --- | --- | --- |
| `SQLiteRepository.ListByUser` | `internal/workout/sqlite_repository.go` | workouts → workout_exercises → sets |
| `SQLiteRepository.GetByID` | `internal/workout/sqlite_repository.go` | one workout → workout_exercises → sets |
| `SQLiteRepository.ListAll` | `internal/exercise/sqlite_repository.go` | exercises → exercise_muscle_groups + exercise_equipment |

`GetByID` is included even though it's a single-workout lookup — the children layers still N+1, and the helper extracted from `ListByUser` covers it for free.

### Algorithms

The batched-load shape, in pseudocode:

```
parents = SELECT * FROM workouts WHERE user_id = ? ... LIMIT ? OFFSET ?
parentIDs = [p.ID for p in parents]
children = SELECT * FROM workout_exercises WHERE workout_id IN (parentIDs)
childIDs = [c.ID for c in children]
grandchildren = SELECT * FROM sets WHERE workout_exercise_id IN (childIDs)

childrenByParent = group children by workout_id
grandchildrenByChild = group grandchildren by workout_exercise_id

for each child: child.Sets = grandchildrenByChild[child.ID]
for each parent: parent.Exercises = childrenByParent[parent.ID]
```

Two helpers fall out, both private to the workout package:

- `loadWorkoutExercisesByWorkoutIDs(ctx, []string) (map[string][]WorkoutExercise, error)` — one batched query, returns rows already grouped by `workout_id`. Sets are already attached when this returns.
- `loadSetsByExerciseIDs(ctx, []int64) (map[int64][]Set, error)` — one batched query, returns rows grouped by `workout_exercise_id`.

Equivalent helpers in the exercise package handle the muscle-group and equipment many-to-many tables.

#### Why batched IN-clauses, not one mega-JOIN

A single `LEFT JOIN` from workouts through workout_exercises through sets returns a Cartesian product: one row per (workout, exercise, set) triple. The Go assembly code has to deduplicate by tracking which parent/child boundaries it has already seen. That's tricky to get right, harder to read, and the row count from SQLite is `O(workouts × exercises × sets)` — the same order as the N+1 statement count, just shifted from "many small queries" to "one large network of rows." For our data shape (~300 sets per page), the Cartesian product is ~300 rows whether it's one JOIN or three batched queries; the batched approach reads better and tests easier, so we take it.

#### Ordering inside groups

The current code orders `workout_exercises` by `exercise_order ASC` and `sets` by `set_order ASC`. The batched queries keep that ordering inside the SQL (`ORDER BY workout_id ASC, exercise_order ASC` for the exercises query; analogous for sets). The Go assembly preserves the order rows arrive in by appending to a slice, so the public API's "exercises in their authored order, sets in their performed order" invariant is maintained.

### Testing

Three layers of tests, all under `internal/workout/` and `internal/exercise/`:

1. **Result-parity tests** — seed a fixture covering: multiple workouts per user, multiple exercises per workout, multiple sets per exercise, zero-exercise workouts, zero-set workout-exercises, duplicate exercise IDs across different workouts. Call both the old and new repository implementations against the same fixture; assert `reflect.DeepEqual` on the returned slices. The old code lives on a `legacy_*` branch in the same file until the parity tests are green, then gets deleted.
2. **Statement-count benchmark** — a `BenchmarkListByUser_StatementCount` test wraps the underlying `*sql.DB` in a counting driver shim (or counts via a query hook) and asserts the count is exactly 3 regardless of fixture size. Same shape benchmark in the exercise package.
3. **Concurrency soak** — a `TestListByUser_NoPoolDeadlock` test fires 100 concurrent `ListByUser` calls against a `MaxOpenConns=2` pool. Today's code (even with the drain-rows fix) would handle this because no nesting holds connections; the test pins the property so a future regression to nested-query style is caught at CI time.

The benchmark's measured wall-time delta on a seed fixture of 50 workouts × 5 exercises × 8 sets goes into the SOW's `## Resolutions` block when the work ships. Expected order of magnitude: ~10ms → ~1ms, dominated by the saved per-statement SQLite overhead.

### Rollout

Single commit, single deploy. The change is internal to the SQLite repositories; the HTTP surface is unchanged, the response bytes are unchanged, the schema is unchanged. The parity tests are the safety net — they prove the new code returns the same data the old code does.

No feature flag is necessary. The drain-rows fix that just shipped was deployed the same way (commit → semantic-release → ECR → SSH deploy) and that's the path this work follows.

## Open Questions

1. **Where does the counting-driver shim live?** Options: (a) a `internal/testutil` package shared across domain test suites, (b) inline in each `*_test.go` that needs it, (c) a vendored upstream library like `github.com/DATA-DOG/go-sqlmock` (but that's for mock SQL, not counting). Tentative lean: (a) — counting connection-level statements is a generic enough utility that a shared `testutil/sqlcount` helper pays off the first time a second domain needs it. The shim is ~30 lines; nothing exotic.
2. **Should the statement-count assertion live in a benchmark or a regular test?** Benchmarks don't run on CI by default in the current Makefile / workflow; a regular `Test` does. Tentative lean: write it as a regular test that asserts `count == 3`, plus a sibling benchmark for the wall-time number. CI catches regressions, dev workflow gets the perf signal.
3. **Do we want to backport the same pattern to `attachPersonalRecordEvents`?** That helper is already bulk-fetching via a single `IN`-clause query, so it isn't N+1. It is, however, one more place a future contributor could accidentally introduce the pattern. Tentative lean: leave it alone; the parity tests + concurrency soak above are the regression guard, and we don't need a code change in code that's already correct.
