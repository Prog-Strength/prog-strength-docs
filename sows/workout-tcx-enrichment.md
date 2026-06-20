---
status: shipped
repos:
  - prog-strength-api
  - prog-strength-web
  - prog-strength-docs
---

# Workout Heart-Rate Enrichment via Garmin TCX

**Status**: Shipped · **Last updated**: 2026-06-19

## Introduction

Prog Strength logs a strength workout as exercises and sets — what you lifted, how heavy, for how many reps. That's the right primary record, and it stays the primary record: a user who logs a bench session with five exercises and nothing else has logged a complete, correct workout. But the watch on their wrist saw more. A Garmin "Strength Training" activity captures a per-second heart-rate trace, the calories the session burned, and the precise start and duration — data that today has nowhere to go in Prog Strength. The user exports the activity as a TCX from Garmin Connect and there's no place to attach it to the bench session it describes.

This SOW closes that gap. It lets a user **optionally** enrich a strength workout with the heart-rate and effort data from a Garmin TCX — either by attaching a file to a workout they already logged, or by starting a new workout *from* a TCX and filling in the exercises afterward. The enrichment is always optional and always additive: no existing logging flow changes, and a workout without a TCX is exactly as valid as it is today.

This is deliberately the *inverse* of the existing run-import flow. For a run, the TCX **is** the workout — pace, distance, and heart rate are the whole point, and the upload is the primary entrypoint (see [running-tracking-via-tcx-import.md](running-tracking-via-tcx-import.md)). For a strength workout the exercises are the point and the TCX is a supporting layer of detail. The two flows also differ in the data they carry: a strength TCX has **no distance** — you're not moving across the ground — so the run importer's distance/pace/elevation machinery doesn't apply, and the run importer would in fact reject a strength file outright (its validator fails any TCX with no non-zero distance). This work reuses the parts that generalize (XML parsing, S3 archival, trackpoint downsampling, the `activities` table as the home for any recorded session) and adds a distance-free summarization path for the parts that don't.

## Proposed Solution

A strength TCX lands in the existing `activities` domain, not in a parallel store. The codebase already generalized its old running-only `Session` type into a sport-agnostic `Activity` (see migration `015_activities_generalize.sql`): a single `activities` table with a normalized `activity_type`, nullable pace/HR/calorie fields, a companion `activity_trackpoints` table, S3 archival of the source TCX, and an `(user_id, ingest_source, source_activity_id)` dedup index. Heart-rate, calories, downsampled trackpoints, and source-file archival are therefore already solved; this SOW extends that domain with one new activity type and one distance-free summarization path, rather than building a second enrichment store inside the workout domain.

The link between the two domains is a single nullable column: `workouts.activity_id`. When a user attaches a TCX to a strength workout, the file is parsed and summarized into a new `activities` row with `activity_type = 'strength_training'`, and the workout's `activity_id` is pointed at it. The workout keeps owning its exercises and sets; the activity owns the heart-rate trace and calories. This is the pragmatic first step toward a future where `activities` is the single top-level record for *every* kind of session — that fuller unification (folding the workout's exercise/set detail into the activity itself) is explicitly deferred to its own follow-up SOW. By putting the TCX data in `activities` now, that later migration is an extension rather than a teardown.

Because a strength activity is genuinely a member of the `activities` table, the read paths that should not see it are made type-aware. The two running-specific aggregations — `GET /activities/running-metrics` and the running best-efforts queries — already filter to `activity_type = 'running'`, so they ignore strength rows for free. The standalone activities **feed** (`GET /activities` list and the day-bucketed overview) is updated to exclude `strength_training`, because a strength session's canonical surface is its workout entry, not a second row in the activities list. And the planned-workout reconciliation hook is not fired for these activities — the workout they belong to already completes a `workout`-kind plan, so firing the `activity`-kind matcher would double-count. These exclusions are the type-driven seam that future activity types plug into.

Two new entrypoints appear on the web, both reusing existing patterns. In the Activities → Workouts view, a **"Log from TCX"** toolbar action sits beside "Start live workout"; it opens an upload modal, mints an empty workout with the TCX attached, and lands the user on that workout's detail page to add exercises. On the workout detail page, an **"Attach TCX"** header action (which flips to **"Detach TCX"** once a file is attached) covers the retroactive case. When a workout has an attached TCX, the detail page renders one new self-contained **"Heart rate & effort"** card — four stat tiles (Avg HR, Max HR, Calories, Active time) and a heart-rate-over-time chart on an elapsed-time x-axis — inserted between the existing numbers row and the exercise list. The rest of the carefully-built "session-recap" layout is untouched.

To support starting a workout from a TCX before any exercises exist, the workout model's validation is relaxed to allow **zero exercises**. An empty workout is simply one the user hasn't filled in yet; the detail page already renders an empty exercise state with an "+ Add exercise" affordance. No draft/complete state machine is introduced.

No new infrastructure: the strength TCX is archived to the same S3 bucket the run importer already uses, via the same `activity.Archiver`. No MCP, agent, or mobile changes. Calories are displayed on the workout but are **not** wired into any energy-balance computation in this SOW — see Non-Goals.

## Goals and Non-Goals

### Goals

- Add a new migration `internal/db/migrations/0NN_workout_tcx_association.sql` (next free number at implementation time; current head is `032`) that:
  - Extends the `activities.activity_type` CHECK constraint to include `'strength_training'`. SQLite cannot alter a CHECK in place, so this is a table rebuild following the precedent already set by `015_activities_generalize.sql` (PRAGMA `foreign_keys=off` within the migration transaction, create the new table with the widened CHECK, `INSERT INTO ... SELECT` copy, drop old, rename, recreate all three indexes that live on `activities` — `idx_activities_dedup`, `idx_activities_user_start`, and `idx_activities_user_type_start`). The `activity_trackpoints` FK to `activities(id)` must survive the rebuild intact.
  - Adds a nullable `activity_id TEXT` column to `workouts` (a soft reference to `activities(id)`; no hard FK, matching the repo's existing cross-domain convention where `workouts.user_id` is not FK'd either).
  - Adds `CREATE UNIQUE INDEX idx_workouts_activity ON workouts(activity_id) WHERE activity_id IS NOT NULL AND deleted_at IS NULL;` — enforces at most one live workout per activity (and thus the "one TCX per workout" rule from the other direction).
- Add `ActivityStrengthTraining ActivityType = "strength_training"` to `internal/activity/activity_type.go` and its `Valid()` switch. It is **not** added to any sport-normalization path — strength activities are only ever created by the workout-TCX endpoints, which set the type explicitly.
- Add a distance-free summarization path in `internal/activity/` (e.g. `tcx_summarizer_strength.go`) that reuses the existing `tcx_parser.go` and computes, from a parsed TCX: `start_time`, `duration_seconds` (last trackpoint timestamp − first), `avg_heart_rate_bpm`, `max_heart_rate_bpm` (nullable when no HR samples), `total_calories` (sum of lap `<Calories>`, nullable when absent), and a ~300-point downsampled trackpoint series carrying `elapsed_seconds` + `heart_rate_bpm` only. `distance_meters` is `0`, and `avg/best_pace`, `elevation_gain`, and `best_efforts` are left nil/empty. Downsampling reuses the existing ~300-point stride logic.
- Relax `workout.Workout.Validate()` to **remove `ErrExercisesRequired`** — a workout may persist with zero exercises. `UserID` and `PerformedAt` remain required; per-exercise validation is unchanged for any exercises that *are* present.
- Add `ActivityID *string` to the `workout.Workout` model and thread it through the workout repository (insert, update, get, list) and the SQLite repository's column mapping.
- Mount three new endpoints under `/workouts` (auth via the existing `auth.RequireUser` middleware):
  - `POST /workouts/imports` — multipart upload (field `file`). Parse + strength-validate → create the `strength_training` activity (archive TCX to S3, dedup via the existing unique index) → create an **empty** workout with `performed_at` = TCX start and `ended_at` = TCX start + duration, `activity_id` set → `201` + the new workout DTO (with embedded enrichment). This is the "Log from TCX" path.
  - `POST /workouts/{id}/tcx` — multipart upload (field `file`). The retroactive attach. `409 workout_tcx_exists` if the workout already has an `activity_id`. Otherwise parse + strength-validate → create the activity → set `activity_id` → `200` + the updated workout DTO. Does **not** modify the workout's `performed_at`/`ended_at`.
  - `DELETE /workouts/{id}/tcx` — detach. Clears `activity_id` and soft-deletes the linked activity (sets `deleted_at`; trackpoints and the S3 object are retained, consistent with run soft-delete). `204`. "Replace" is detach-then-attach; no dedicated replace endpoint.
- Embed the linked activity's enrichment in the workout DTOs: the single-workout `GET /workouts/{id}` includes the HR/calorie summary **and** the downsampled HR `trackpoints`; the list `GET /workouts` includes a lightweight summary (a `has_tcx` flag plus avg HR and calories) but **never** the trackpoints array. The handler loads the linked activity for the workout(s) being returned.
- Creating or attaching a strength activity must **not** invoke the activity `PlanMatcher` (`activityPlanMatcher.OnSessionLogged`). These endpoints write the activity directly via the repository; the workout's own `workoutPlanMatcher` remains the single reconciliation signal. Detach must not fire `OnSessionDeleted` for the activity matcher.
- Make the standalone activities feed type-aware: exclude `activity_type = 'strength_training'` from the `GET /activities` list query and from the day-bucketed activities overview aggregation. Confirm via test that `running-metrics` and best-efforts (already `running`-filtered) are unaffected.
- Web — `lib/api.ts`: add `createWorkoutFromTCX(token, file)`, `attachWorkoutTCX(token, workoutId, file)`, `detachWorkoutTCX(token, workoutId)`; extend the `Workout` type with `activity_id: string | null` and an optional `enrichment` object (`avg_heart_rate_bpm`, `max_heart_rate_bpm`, `total_calories`, `duration_seconds`, `start_time`, and `trackpoints?` on the detail load).
- Web — Activities → Workouts view (`app/(app)/activities/page.tsx`): add a **"Log from TCX"** `ToolbarButton` in the workouts-view left action group, beside "Start live workout". It opens the upload modal; on success it navigates to `/workouts/{id}` for the newly created empty workout.
- Web — workout detail (`app/(app)/workouts/[id]/page.tsx`):
  - Header action: **"Attach TCX"** when `activity_id` is null, **"Detach TCX"** (with a small confirm modal) when set. The subtitle gains a quiet "♥ from Garmin" indicator when a TCX is attached.
  - New `HeartRateEffortCard` component: a bordered card titled "Heart rate & effort" with four `StatTile`s (Avg HR, Max HR, Calories, Active time) and a `WorkoutHeartRateChart` below. Rendered only when enrichment is present, inserted between the numbers row and the muscle summary / exercise list. The existing recap layout is otherwise unchanged.
  - `WorkoutHeartRateChart`: a Recharts line chart of HR vs **elapsed time** (x-axis is time, not distance — there is no distance). Reuses the project's existing chart styling.
  - The empty-exercise state (zero exercises + "+ Add exercise") must render correctly for a workout created from a TCX before any exercise is added — verify the existing detail page handles an empty `exercises` array.
- Web — upload modal: generalize the existing running `UploadTCXModal` into a shared modal (or a sibling `WorkoutTCXUploadModal`) whose success/error handling is parameterized. States: idle → uploading → success / error. The `409 duplicate_activity` response renders "This file is already in your log" with a link to where the activity already lives (a run or another workout); `409 workout_tcx_exists` renders "This workout already has a file attached — detach it first."
- API error responses use the existing `{ "error": <message>, "code": <slug> }` shape. New slugs: `tcx_parse_failed`, `tcx_no_effort_data`, `duplicate_activity` (409, body includes `{ existing: { kind: "run" | "workout", id } }`), `workout_tcx_exists` (409), `file_too_large` (413), `unsupported_media_type` (415).
- Observability: the attach / import handlers log structured fields `user_id`, `workout_id` (where applicable), `source_activity_id`, and `outcome ∈ {imported, attached, duplicate, invalid}`. Per-route Prometheus metrics come for free from existing middleware.
- Tests per repo — see § Tests.

### Non-Goals

- **Energy-balance integration.** Calories are *displayed* on the workout but are not summed into any net-calorie / energy-balance number — because no such number exists anywhere in the product today (verified: the nutrition and dashboard surfaces are intake-only; runs do not feed an energy balance either, despite the motivational framing in the run-import SOW). Introducing net-calorie energy balance is a sizable nutrition-domain feature that should ingest *all* activity calories (runs, walks, and strength) and surface on the nutrition page and dashboard. It is deferred to its own SOW.
- **Full activities/workout unification (Option C).** This SOW links a workout to an activity; it does not fold the workout's exercise/set data into the `activities` table or make a strength session a literal subtype of `Activity`. That larger refactor is a deliberate follow-up; the `activity_id` link is the bridge toward it.
- **Mobile.** No changes to the mobile app. The attach/import endpoints are stable from day one for a future mobile dispatch.
- **MCP and agent integration.** The agent cannot attach a TCX or read a workout's heart-rate data. A future SOW can add the tools and a prompt rule.
- **Deriving activity type from the TCX `<Sport>` tag.** A TCX attached to a strength workout is always typed `strength_training` regardless of its sport tag. If a user attaches a *running* TCX to a strength workout, it is still summarized for HR/calories only; pace/distance/elevation/best-efforts are ignored and it does not appear in the running surfaces. (If the user wanted a run, the run importer is the correct entrypoint.)
- **Overwriting manually-entered workout times.** Attaching a TCX to an existing workout never changes its `performed_at`/`ended_at`. Only the create-from-TCX path prefills those (because there is nothing else to go on), and they remain user-editable.
- **Editing the enrichment by hand.** HR/calorie values are derived from the file. Correcting them means detach + re-attach. No manual HR/calorie fields.
- **HR zones, training load, recovery, or strength-specific HR analytics.** The data is present; the analysis is deferred.
- **Best-efforts / pace / elevation for strength activities.** Not computed or stored — they are meaningless for a stationary session.
- **Batch attach.** One file per request. Backfilling N workouts means N attaches.
- **A draft/complete workout state.** Zero-exercise workouts are allowed outright; no status column or state machine is introduced.

## Implementation Details

### Data Model

One new migration, `internal/db/migrations/0NN_workout_tcx_association.sql` (next free number at implementation; head is `032`).

**1. Widen the `activities.activity_type` CHECK** to `IN ('running', 'walking', 'cycling', 'other', 'strength_training')`. Follow the `015_activities_generalize.sql` rebuild precedent exactly:

- `PRAGMA foreign_keys = OFF;` (the migration runs inside a transaction).
- `CREATE TABLE activities_new (...)` identical to the current `activities` schema but with the widened CHECK.
- `INSERT INTO activities_new SELECT * FROM activities;`
- `DROP TABLE activities;` then `ALTER TABLE activities_new RENAME TO activities;`
- Recreate all three indexes that lived on `activities`: `idx_activities_dedup` (the `(user_id, ingest_source, source_activity_id) WHERE deleted_at IS NULL` unique index), `idx_activities_user_start`, and `idx_activities_user_type_start`.
- `activity_trackpoints` is untouched, but its FK targets `activities(id)`; verify with a populated-DB migration test that trackpoints still resolve after the rename.

**2. Link column on `workouts`:**

| Column | Type | Description |
| --- | --- | --- |
| `activity_id` | TEXT | Nullable. Soft reference to `activities(id)`. Null = no TCX attached. |

- `CREATE UNIQUE INDEX idx_workouts_activity ON workouts(activity_id) WHERE activity_id IS NOT NULL AND deleted_at IS NULL;` — at most one live workout per activity.

No other schema changes. The strength activity reuses the existing `activities` and `activity_trackpoints` columns; running-only fields (`distance_meters`, paces, elevation) are simply left at zero/null for strength rows, exactly as the model's nullable-pointer contract already intends.

### Write Paths

**Create-from-TCX (`POST /workouts/imports`):**

1. Parse + strength-validate the multipart file in memory. Reject `tcx_parse_failed` (malformed) or `tcx_no_effort_data` (no HR samples *and* no calories — nothing worth enriching with) or `413 file_too_large` (> 10 MB, matching the run cap).
2. Summarize into an `Activity` (`activity_type = strength_training`, `ingest_source = manual_tcx`). Within one DB transaction: insert the activity row, insert its trackpoints. The existing dedup unique index may fire here → translate to `409 duplicate_activity` (re-query to populate `existing`).
3. Archive the raw TCX to S3 via the existing `activity.Archiver`. On failure: roll back, best-effort delete the S3 object, `500 storage_failed`.
4. Insert the empty workout (`activity_id` set, `performed_at`/`ended_at` from the TCX). Commit.
5. Return `201` + the workout DTO with embedded enrichment.

**Attach (`POST /workouts/{id}/tcx`):**

1. Load the workout (404 if missing / not owned). If `activity_id` is already set → `409 workout_tcx_exists`.
2. Steps 1–3 of the create path (parse, summarize, persist activity, archive).
3. `UPDATE workouts SET activity_id = ?, updated_at = ? WHERE id = ? AND user_id = ?`. `performed_at`/`ended_at` are **not** touched.
4. Return `200` + the updated workout DTO.

**Detach (`DELETE /workouts/{id}/tcx`):**

1. Load the workout (404 as above). If `activity_id` is null → `204` (idempotent no-op).
2. In one transaction: `UPDATE workouts SET activity_id = NULL ...` and soft-delete the activity (`UPDATE activities SET deleted_at = ? WHERE id = ?`). Trackpoints and the S3 object are retained.
3. `204`.

None of these three paths call the activity `PlanMatcher`. The create/attach paths *may* surface the workout to `workoutPlanMatcher` through the existing workout write path's hook (attach does not create a workout, so it does not re-fire; create-from-TCX fires the workout matcher once, like any new workout).

### Summarization

The strength summarizer is the run summarizer with the distance-dependent computations removed:

- `start_time` = first trackpoint's timestamp (fallback: first lap `StartTime`).
- `duration_seconds` = last trackpoint timestamp − first.
- `avg_heart_rate_bpm` = mean of present HR samples, rounded; nil if none.
- `max_heart_rate_bpm` = max HR sample; nil if none.
- `total_calories` = sum of lap `<Calories>`; nil if absent.
- Trackpoints: downsample to ~300 via the existing stride logic, keeping `elapsed_seconds` and `heart_rate_bpm`; `distance_meters` = 0, `pace`/`elevation` = nil.
- `distance_meters` = 0; `avg_pace`, `best_pace`, `elevation_gain_meters` = nil; `best_efforts` = empty.

"Active time" shown on the card is `duration_seconds`. A strength TCX's elapsed span includes inter-set rest, which is the correct notion of session duration here.

### API Surface

All endpoints require JWT auth. Error bodies use `{ "error": <message>, "code": <slug> }`.

`POST /workouts/imports`
- `multipart/form-data`, field `file`.
- `201` + `WorkoutDTO` (empty `exercises`, enrichment embedded).
- `400` `tcx_parse_failed` | `tcx_no_effort_data`; `409` `duplicate_activity` (`{ error, code, existing: { kind, id } }`); `413` `file_too_large`; `415` `unsupported_media_type`; `500` `storage_failed`.

`POST /workouts/{id}/tcx`
- `multipart/form-data`, field `file`.
- `200` + updated `WorkoutDTO`. `404` if workout missing/not owned. `409` `workout_tcx_exists` if already attached. Same `400`/`409 duplicate_activity`/`413`/`415`/`500` as above.

`DELETE /workouts/{id}/tcx`
- `204`. `404` if workout missing/not owned. Idempotent when nothing is attached.

`WorkoutDTO` enrichment shape (added fields; existing workout fields unchanged):

```json
{
  "id": "uuid",
  "activity_id": "uuid-or-null",
  "performed_at": "2026-06-19T13:12:00Z",
  "ended_at": "2026-06-19T14:04:00Z",
  "exercises": [],
  "enrichment": {
    "source_activity_id": "2026-06-19T13:12:00.000Z",
    "start_time": "2026-06-19T13:12:00Z",
    "duration_seconds": 3120,
    "avg_heart_rate_bpm": 148,
    "max_heart_rate_bpm": 172,
    "total_calories": 512,
    "trackpoints": [
      { "sequence": 0, "elapsed_seconds": 0, "heart_rate_bpm": 102 },
      { "sequence": 1, "elapsed_seconds": 11, "heart_rate_bpm": 110 }
    ]
  }
}
```

`enrichment` is `null` when `activity_id` is null. `enrichment.trackpoints` is present only on the single-workout detail response; list responses include `enrichment` with HR/calorie summary but omit `trackpoints`.

### Tests

**Go API:**

- `internal/activity/tcx_summarizer_strength_test.go`: table-driven against new `internal/activity/testdata/` fixtures:
  - `strength_session.tcx` — Sport `Other`, dense HR, lap calories, **no distance** — avg/max HR exact, calories exact, duration ± 1 s, distance/pace/elevation nil, ~300 downsampled points.
  - `strength_no_hr.tcx` — calories but no HR samples — avg/max HR nil, still summarizes (calories present).
  - `no_effort_data.tcx` — no HR and no calories — rejected `tcx_no_effort_data`.
  - `malformed.tcx` — rejected `tcx_parse_failed` without panic.
  - A run TCX summarized through the strength path yields HR/calories and ignores distance (documents the "attach a run by mistake" behavior).
- `internal/workout/` validation test: a workout with zero exercises now passes `Validate()`; `UserID`/`PerformedAt` still required.
- `internal/workout/handler_test.go` (httptest + in-memory repos + fake `activity.Archiver`):
  - `POST /workouts/imports` → 201, empty exercises, `performed_at`/`ended_at` from TCX, enrichment populated.
  - `POST /workouts/{id}/tcx` → 200, `activity_id` set, workout times **unchanged**; second attach → `409 workout_tcx_exists`; duplicate source id → `409 duplicate_activity` with `existing`.
  - `DELETE /workouts/{id}/tcx` → 204, `activity_id` cleared, linked activity soft-deleted; idempotent second call → 204.
  - Detail response includes `enrichment.trackpoints`; list response includes `enrichment` without `trackpoints`.
  - Validation slugs for `tcx_parse_failed`, `tcx_no_effort_data`, `file_too_large`.
- `internal/activity/` (or `internal/server/`) test asserting the `GET /activities` list and overview exclude `strength_training`, and `running-metrics` / best-efforts are unchanged.
- Migration tests: `db.Migrate()` applies cleanly on an **empty** DB and on a **populated** DB (existing activities + trackpoints survive the CHECK rebuild and the dedup index is intact afterward); `idx_workouts_activity` rejects a second live workout pointing at the same activity.

**Web:**

- Unit test for `WorkoutHeartRateChart` rendering on an elapsed-time axis from a trackpoint array.
- Unit test that the workout detail renders the `HeartRateEffortCard` only when `enrichment` is present, and the empty-exercise state when `exercises` is empty.
- No E2E framework introduced. PR smoke checklist:
  - "Log from TCX" in the Workouts toolbar → upload a strength TCX → lands on a new empty workout detail with the Heart rate & effort card and an empty exercise list → add an exercise → it persists.
  - On an existing workout with no TCX, "Attach TCX" → card appears, header flips to "Detach TCX", "♥ from Garmin" shows in the subtitle. Workout date/time unchanged.
  - Attach a second file to the same workout → "already has a file attached" message.
  - Attach a file already in the log (a run, or another workout) → duplicate message with a link to where it lives.
  - Detach → confirm → card disappears, header flips back to "Attach TCX".
  - A logged strength session does **not** appear as its own row in the Activities → Overview / Running lists.
  - A workout with no TCX is unchanged end to end.

### Rollout

Coordinated two-repo dispatch (no infra; the S3 bucket and `activity.Archiver` already exist):

1. `prog-strength-api` — migration (CHECK rebuild + `workouts.activity_id`), `strength_training` type, strength summarizer, the three `/workouts` endpoints, DTO enrichment, the activities-feed exclusion, and the relaxed workout validation. Additive; deploys via the existing release workflow.
2. `prog-strength-web` — api-client functions, the "Log from TCX" toolbar action, the workout-detail attach/detach + Heart rate & effort card, and the shared upload modal. Deploys via the existing release workflow.
3. Hand-test in the deployed environment with a real account and a real Garmin strength-training TCX; walk the smoke checklist.

Each PR is independently revertable. The API change touches no existing read or write path destructively — `workouts.activity_id` is nullable and defaults to null, the workout-validation relaxation only *accepts* more, and the activities-feed exclusion is a `WHERE` narrowing.

## Open Questions

1. **Validity floor for a strength TCX.** The tentative rule rejects only files with *neither* HR samples *nor* calories (`tcx_no_effort_data`). Alternative: require HR specifically (calories alone is thin enrichment for a card built around a heart-rate chart). Tentative lean: accept HR-or-calories and let the card degrade gracefully (hide the chart when there are no HR samples), since some users run wrist-HR off but still get a calorie estimate. Revisit if calorie-only cards look empty.
2. **"Log from TCX" empty-workout persistence.** The empty workout is persisted immediately on import (it appears in the workouts list with zero exercises until the user fills it in). Alternative: hold it client-side until the first exercise is added. Tentative lean: persist immediately — zero-exercise workouts are explicitly allowed, and persisting avoids losing the attached activity if the user navigates away. A stale empty workout is a minor cost; a future sweep could prune zero-exercise, zero-enrichment workouts if they accumulate.
3. **Naming of the new sport label in any shared UI.** Internally `strength_training`; the workout surfaces never show it (the workout *is* the surface). No user-facing "Strength" activity badge is introduced, since these rows are excluded from the activities feed. Flag if a future surface needs a label.
4. **Shared vs. duplicated upload modal.** The plan generalizes the running `UploadTCXModal`. If parameterizing it proves invasive to the running flow, fall back to a sibling `WorkoutTCXUploadModal` that shares only the file-input primitive. Decide at implementation time based on how cleanly the success/error/duplicate handling factors out.
