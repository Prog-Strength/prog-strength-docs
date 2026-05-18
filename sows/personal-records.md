# Personal Records

**Status**: Draft · **Last updated**: 2026-05-18

## Introduction

**Prog Strength** estimates each lifter's one rep max from their working sets — useful for tracking direction, but it is still an estimate. Lifters also care about their *actual* personal record on each headline lift: the heaviest weight they have ever moved on that exercise, regardless of rep count. That number is what they reference when planning a max attempt, when bragging, and when sanity-checking whether their estimated-1RM trajectory has caught up with reality.

The app has no place to record or surface those personal records today. A lifter who hits a heavy set has nothing to show for it inside **Prog Strength** beyond a workout buried in their log. This feature gives that lift a home: a Personal Records view that surfaces each headline lift's PR (with the rep count it was set at), the date and workout where it happened, and the lifter's *current* recency-weighted estimated one rep max next to it so the gap between estimate and lived reality is visible at a glance. A separate event log of PR breaks lets the workout views badge sessions that produced a record, and gives the chat agent enough signal to congratulate the lifter when one is broken.

## Proposed Solution

**Prog Strength** maintains a personal records table keyed by `(user, exercise)`. Every time a workout is logged, its sets are scanned against the current personal record for each exercise in that workout; any set whose weight beats the current record updates it, with the rep count recorded alongside. Each record links back to the workout that produced it so the lifter can navigate straight to the session.

Alongside the PR table, an append-only `personal_record_events` table records every PR break that has happened across the user's history. The workout list and detail endpoints join against this table so a workout that produced a record can be badged inline with a 🏆 icon, and the chat agent can read the same table to congratulate the lifter on recent breaks.

The user's current recency-weighted estimated one rep max for each exercise is computed at request time from the existing `exercise_one_rep_max_history` table — no new denormalized storage is introduced. A large gap between the estimate and the PR weight is a clear cue that the lifter has gotten stronger since their last record-setting session and may want to test their actual capacity again. The complement also works: if the user hits a heavy multi-rep PR (e.g. 305 × 3), their estimated 1RM rises with it via Epley, so the comparison stays honest without forcing the lifter to attempt a true single just to update the record.

The set of "headline" lifts surfaced on the Personal Records page (bench press, back squat, deadlift to start) lives in backend Go code rather than in any frontend. That way the eventual mobile client and the existing web client see the same headline set without either repo having to be kept in sync.

## Goals and Non-Goals

### Goals

- Persist one row per `(user, exercise)` in a new `personal_records` table tracking the heaviest weight the user has lifted on that exercise, with the rep count of the record-setting set.
- Persist one row per PR break in a new `personal_record_events` table — an append-only log keyed by workout, used to badge PR-breaking sessions in the workout views and to drive the agent's congratulations.
- Maintain both tables in lockstep with `workouts` via the workout repository's Create/Update/Delete paths so neither drifts.
- Define the headline lift set in backend Go code so the web client (and the future mobile client) share a single source of truth.
- Surface a Personal Records sidebar entry on the frontend that displays each headline lift with PR weight, rep count, achievement date, and the current recency-weighted estimated 1RM for comparison. Headline lifts the user has never trained appear as empty placeholder cards.
- Add a new single-workout detail route reachable from the Personal Records page so the lifter can jump straight to the session that produced a record. The route reuses the existing readonly workout view component.
- Backfill both tables on the first startup after the migration ships by scanning each user's existing workouts.

### Non-Goals

- **Per-rep-range PRs (separate 1RM, 3RM, 5RM, 10RM rows per exercise).** The MVP records a single PR per exercise — the heaviest weight at any rep count — and carries that rep count alongside. Tracking distinct PRs across rep ranges is a separate UI and ranking model and is out of scope here.
- **User-curated headline lift sets.** The backend defines the headline set; users cannot add or remove lifts. Personal customization is a follow-up.
- **Manual PR entry or correction.** PRs are derived from logged workouts. A lifter who wants to record a PR from before they started using **Prog Strength** must backfill it by logging the workout itself.
- **A denormalized cache for the current estimated one rep max.** The recency-weighted baseline is computed on demand from `exercise_one_rep_max_history` — sub-millisecond for the data scales we care about. Adding a separate table or column to cache it would buy negligible read performance at the cost of another write path to keep consistent. If profiling later shows it is actually hot, we can denormalize with data. (This matches the original 1RM history SOW's explicit non-goal of persisting the baseline.)
- **Cross-exercise lift grouping** (e.g. a single "Squat PR" spanning both high-bar and low-bar back squat). Those are mechanically different lifts; conflating them would obscure honest data. The PR table is keyed per exercise slug; the headline lift list picks one exercise per category for the headline view.
- **Lifetime current-PR history.** Only the *current* record per `(user, exercise)` is persisted in `personal_records`. The event log captures the moment a record was broken; once superseded, the prior record is no longer indexed for lookup. Original workouts remain in the log, so the historical lift itself is preserved through `workouts`.
- **A dedicated PR activity feed endpoint for the agent.** The agent reads recent PR breaks via the existing workout list endpoint (each workout carries its associated event rows) and the personal records endpoint (each row carries `achieved_at`). A dedicated `/personal-records/events` feed can be added later if the agent's needs outgrow these.

## Implementation Details

### Data Model

#### `personal_records`

One row per `(user, exercise)` where the user has at least one logged set on that exercise.

| Column | Type | Description |
| --- | --- | --- |
| `id` | text | Primary key. |
| `user_id` | text | Owning user. |
| `exercise_id` | text | Catalog slug. References `exercises(id)`. |
| `workout_id` | text | Foreign key to `workouts`. The session that currently holds the record. |
| `weight` | real | The load lifted on the record-setting set. The single load-bearing field for "is this a PR?" comparisons. |
| `reps` | integer | Reps performed on the record-setting set (1, 3, 5, etc.). Carried for display and Epley sanity checks; not used for ranking. |
| `unit` | text (`lb` / `kg`) | Unit of the record-setting set. |
| `achieved_at` | text (RFC3339) | Mirrors the parent workout's `performed_at`. |
| `created_at` | text (RFC3339) | Row creation timestamp. |
| `updated_at` | text (RFC3339) | Row last-modified timestamp; touched when a new record overwrites the old. |

**Uniqueness**: `UNIQUE(user_id, exercise_id)` — at most one current PR per user per exercise. Doubles as the primary read-side index.

**Secondary index**: `(user_id, achieved_at DESC)` for the Personal Records page query *"this user's PRs, most recently set first."*

**Foreign keys**: `workout_id → workouts(id)` with `ON DELETE CASCADE`. Defensive only — workouts are soft-deleted in practice; the repository's Delete path explicitly recomputes affected PRs.

#### `personal_record_events`

One row per PR break. Append-only at the API surface — entries are derived from `workouts` and are rebuilt on workout edits rather than mutated in place.

| Column | Type | Description |
| --- | --- | --- |
| `id` | text | Primary key. |
| `user_id` | text | Owning user. |
| `exercise_id` | text | Catalog slug. References `exercises(id)`. |
| `workout_id` | text | Foreign key to `workouts`. The session that produced this PR break. |
| `weight` | real | New PR weight. |
| `reps` | integer | Reps at the new PR weight. |
| `unit` | text (`lb` / `kg`) | Unit of the new PR. |
| `previous_weight` | real \| null | Previous PR weight, or `null` if this was the user's first logged set on this exercise. |
| `previous_reps` | integer \| null | Reps at the previous PR weight. |
| `previous_unit` | text \| null | Unit of the previous PR weight. |
| `achieved_at` | text (RFC3339) | Mirrors the parent workout's `performed_at`. |
| `created_at` | text (RFC3339) | Row creation timestamp. |

**Indexes**:

- `(workout_id)` — the workout list endpoint's "did any of these workouts break a PR?" join.
- `(user_id, achieved_at DESC)` — recent-break queries (agent congratulations, future activity feeds).

**Foreign keys**: `workout_id → workouts(id)` with `ON DELETE CASCADE`. The repository's Delete path also rebuilds events from the remaining workouts (see Write Path) so the cascade is again defensive.

#### Headline lift configuration

A package-level constant slice in the backend identifies which exercises are surfaced on the Personal Records page:

```go
// internal/personalrecord/headline.go (or similar)
var HeadlineLifts = []string{
    "barbell-bench-press",
    "barbell-high-bar-back-squat",
    "barbell-deadlift",
}
```

Slugs must exist in the exercise catalog; a startup or test-time check enforces this so a typo doesn't ship silently. The catalog already contains the first two; `barbell-deadlift` is added to the catalog as part of this work if not already present.

### Write Path

Both new tables are fully derivable from `workouts`. The workout repository maintains them inside the same transactions that write workouts:

- **Workout created** — for each exercise in the new workout, find the workout's heaviest set on that exercise. If its weight beats the current PR for `(user_id, exercise_id)`, upsert the PR row and insert a corresponding `personal_record_events` row capturing the new and previous values.
- **Workout updated** — full-replacement semantics mean any set could have been added, removed, or had its weight or reps changed. The repository performs a per-user re-derivation for every exercise that the update could have touched (the exercises in the new workout plus any exercise whose current PR currently references this `workout_id`). The PR row is recomputed and every `personal_record_events` row owned by that `(user_id, exercise_id)` whose chronological position is on or after the edited workout is rebuilt from a chronological scan.
- **Workout deleted** — for every PR row whose `workout_id` matches the deleted workout, recompute from the user's remaining non-deleted workouts. The events table is rebuilt for each affected `(user_id, exercise_id)` from the same chronological scan. If no qualifying set remains for an exercise, both its PR row and its events rows are deleted.

The "recompute" operation underlying all three paths is: for a given `(user_id, exercise_id)`, walk the user's non-deleted workouts in chronological order, track the running max weight, and emit (a) the final running max as the PR row and (b) one event row each time the running max increased. Same function services the live write path and the backfill.

**Tie semantics on incoming sets**: a set whose weight exactly matches the current PR does *not* overwrite the record and does *not* emit an event. The existing record stays, preserving the original date's claim to the achievement.

### Algorithms

**PR detection rule.** A set qualifies as a candidate for the PR if and only if:

```
set.weight > current_pr.weight   (or no current PR exists for this exercise)
```

Notably, the rule has *no* `reps == 1` constraint. A lifter who hits 305 × 3 when their current PR is 300 × 1 sets a new PR; their estimated one rep max via Epley rises in step, so the disparity between PR weight and estimated 1RM stays meaningful without forcing the lifter into a true 1RM attempt just to update the record.

**Unit reconciliation.** Most users log in a single unit, but mixed-unit users exist. When comparing a candidate set against the current PR in a different unit, convert before comparing:

```
1 kg = 2.20462 lb
```

The stored row keeps the record-setting set's original unit; the conversion is only used at compare time. This deliberately diverges from the 1RM history table's most-common-wins approach: a single record cannot be split across two units, so one has to win the comparison.

**Backfill recompute.** Identical to the runtime recompute, applied per `(user_id, exercise_id)` for every user/exercise pair that has ever had a logged set. The same function services the live write path so backfilled rows match what `Create` would have produced.

### API Surface

Three new or modified endpoints, all behind the existing user-auth middleware:

- **`GET /workouts/{id}` (new)** — returns a single workout owned by the authed user. The repository already has `GetByID`; this exposes it over HTTP. Ownership check uses the same compare-and-return-404 pattern as `PUT` / `DELETE` so workout IDs cannot be enumerated. The response includes a `personal_records_set` field — an array of any `personal_record_events` rows whose `workout_id` matches.

- **`GET /workouts` (modified)** — existing list endpoint, now bulk-joins against `personal_record_events` and includes the same `personal_records_set` array per workout. The join is a single `WHERE workout_id IN (...)` query against the events table so the list endpoint stays a single round trip.

- **`GET /personal-records` (new)** — returns one row per *headline* lift, sorted by `achieved_at DESC` (placeholders sort last). Headline lifts the user has never trained still appear, with PR fields set to `null`. Each entry's response shape:

  ```json
  {
    "exercise_id": "barbell-bench-press",
    "exercise_name": "Barbell Bench Press",
    "workout_id": "...",
    "weight": 305.0,
    "reps": 3,
    "unit": "lb",
    "achieved_at": "2026-03-15T18:32:00Z",
    "current_estimated_1rm": 332.4,
    "estimated_1rm_unit": "lb"
  }
  ```

  `current_estimated_1rm` is computed per request via `RecencyWeightedBaseline` against `exercise_one_rep_max_history`. Returns `null` when the user has no entries in the baseline window for that exercise.

The agent reads PR context through the same two routes: `GET /personal-records` for current state and `GET /workouts` for recent breaks (via the embedded `personal_records_set`).

### Backfill or Migration

- **Mechanism**: in-process backfill after the SQL migration creates both empty tables, gated by `existing > 0` row-count checks on each. Same pattern as `BackfillOneRepMaxHistory`. The backfill calls the same recompute function the live write path uses, so backfilled rows match what `Create` would have produced.
- **Recoverability**: truncate `personal_records` and `personal_record_events` and restart the API. The next boot finds them empty and rebuilds from `workouts`.
- **Scale boundary**: at current beta volumes (low thousands of workouts) the backfill completes well under a minute inline with the migration. If the table grows past a few hundred thousand workouts the backfill should split into a separate CLI command; until then, inline-with-migration is the simpler story.

## Open Questions

1. **PR ties within a single workout.** If a workout has two sets at the same weight on the same exercise, they tie. Both reference the same workout so the record is the same either way. Documenting that we ignore this rather than picking which set "won."

2. **Same-workout PR break for an exercise that was already a PR earlier in the workout.** Within a single workout, if a lifter ramps up — e.g. 295 × 1, 300 × 1, 305 × 1 — should each rep emit its own event row, or should we collapse them into a single event for the workout's heaviest set? **Lean: collapse to one event per `(workout, exercise)` so the workout gets a single 🏆 rather than three.** Each event's `previous_weight` references the PR weight *before* the workout, not the intra-workout intermediate.

3. **Soft-delete reversal.** Workouts are soft-deleted today; un-deletion is not supported. The recompute on workout delete drops events rather than tombstoning them, so re-creating a deleted workout would re-emit events as if it were new — fine. **Lean: do nothing now; revisit if un-delete is ever scoped.**
