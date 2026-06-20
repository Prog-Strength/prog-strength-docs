# Implementation Plan — Workout Heart-Rate Enrichment via Garmin TCX

**SOW**: [`sows/workout-tcx-enrichment.md`](../sows/workout-tcx-enrichment.md)
**Repos**: `prog-strength-api`, `prog-strength-web`, `prog-strength-docs`
**Branch**: `feat/workout-tcx-enrichment`

This plan derives directly from the SOW (which is itself prescriptive). It
records the concrete file-level changes, the precedent each mirrors, and the
test matrix. Execution is subagent-driven with a spec review and a
code-quality review per repo before the local CI gate.

## Context discovered

- Migration head is `032`; next free number is **`033`**.
- golangci-lint is pinned at **v2.12.2** (`.github/workflows/ci.yml`); the
  gate is `golangci-lint run`, `go vet ./...`, `go mod tidy` drift check,
  `go test ./...`.
- `activities` current schema (post-`015`/`016`): `activity_type` CHECK is
  `IN ('running','walking','cycling','other')`; three indexes live on it
  (`idx_activities_dedup`, `idx_activities_user_start`,
  `idx_activities_user_type_start`); `activity_trackpoints` FK →
  `activities(id) ON DELETE CASCADE`; `activity_best_efforts` FK likewise.
- The run flow: `IngestTCX` (parse → validate → `normalizeActivityType` →
  `summarize` → `repo.Create`) in `internal/activity/`. `repo.Create` is
  type-agnostic (inserts activity + trackpoints + best-efforts, archives to
  S3, dedups via `idx_activities_dedup`). The run summarizer/validator are
  distance-centric; the run validator rejects any TCX with no non-zero
  distance.
- `GET /activities` feed is `repo.List` (cursor) + `repo.ListInRange`
  (range, also used by the dashboard). `running-metrics` and best-efforts use
  separate `activity_type = 'running'` filtered queries.
- Web: all API calls in `lib/api.ts`; running upload is `importRunningTcx`
  + page-private `UploadTCXModal` (running) + `HeartRateChart` (running,
  HR-vs-distance). Shared `StatTile`, `ToolbarButton` components. Tests are
  vitest + @testing-library/react.

## API changes (`prog-strength-api`)

### 1. Migration `033_workout_tcx_association.sql`
Mirror the `015` rebuild precedent exactly.
- `PRAGMA defer_foreign_keys = 1;`
- Create `activities` rebuild with widened CHECK
  `IN ('running','walking','cycling','other','strength_training')`,
  `INSERT INTO ... SELECT *` copy, drop old, rename, recreate all three
  indexes. `activity_trackpoints`/`activity_best_efforts` untouched (FK
  survives via deferred check at COMMIT).
- `ALTER TABLE workouts ADD COLUMN activity_id TEXT;`
- `CREATE UNIQUE INDEX idx_workouts_activity ON workouts(activity_id)
  WHERE activity_id IS NOT NULL AND deleted_at IS NULL;`
- Migration tests: empty DB applies; populated DB keeps activities +
  trackpoints + dedup index; `idx_workouts_activity` rejects a 2nd live
  workout on the same activity.

### 2. `strength_training` activity type
- Add `ActivityStrengthTraining ActivityType = "strength_training"` and to
  `Valid()`. **Not** added to `normalizeActivityType` (strength rows are
  only ever created explicitly by the workout endpoints).

### 3. Strength summarizer + validator (`internal/activity/`)
- `tcx_summarizer_strength.go`: `summarizeStrength(p *parsedTCX) Activity`
  — distance-free. `start_time` from first trackpoint (fallback first lap),
  `duration_seconds` = last − first tp, avg/max HR (nil if none), calories
  = sum lap `<Calories>` (nil if absent), ~300-pt downsample (reuse stride
  logic) keeping `elapsed_seconds` + `heart_rate_bpm` only; distance 0,
  paces/elevation nil, best_efforts empty; `ActivityType =
  strength_training`.
- Strength validation: reject `tcx_no_effort_data` when **no HR samples and
  no calories**. Malformed → `tcx_parse_failed` (via parseTCX).
- `IngestStrengthTCX(ctx, repo, userID, r)`: parse → strength-validate →
  `summarizeStrength` → set type/source (`manual_tcx`) → `repo.Create`;
  dedup → `ErrDuplicate` (+ existing lookup).
- New slug `tcx_no_effort_data`.
- Tests: `tcx_summarizer_strength_test.go` with fixtures
  `strength_session.tcx`, `strength_no_hr.tcx`, `no_effort_data.tcx`,
  `malformed.tcx`, + a run TCX through the strength path.

### 4. Workout model + repository
- `Workout.ActivityID *string` (`json:"activity_id"`).
- `Validate()`: remove `ErrExercisesRequired` (keep `UserID`/`PerformedAt`
  required, per-exercise validation unchanged).
- Thread `activity_id` through insert/update/get/list SQL + scans.
- Repo test: zero-exercise workout passes `Validate()`.

### 5. Endpoints under `/workouts`
Extend `workout.Handler` with an `activity.Repository` dependency (new
`NewHandler` arg; update callers + tests).
- `POST /workouts/imports` → 201, empty workout from TCX, `performed_at` =
  TCX start, `ended_at` = start+duration, `activity_id` set, enrichment
  embedded. Errors: `tcx_parse_failed`, `tcx_no_effort_data`,
  `duplicate_activity` (409, `existing:{kind,id}`), `file_too_large` (413),
  `unsupported_media_type` (415), `storage_failed` (500).
- `POST /workouts/{id}/tcx` → 200 attach; 404 missing/unowned; 409
  `workout_tcx_exists` if already attached; does not touch times.
- `DELETE /workouts/{id}/tcx` → 204 detach (clear `activity_id`, soft-delete
  activity; idempotent).
- DTO enrichment: detail includes summary + `trackpoints`; list includes
  summary without `trackpoints`. `enrichment` null when `activity_id` null.
- None of these call the activity `PlanMatcher`. Create-from-TCX fires the
  workout matcher once like any new workout (attach/detach do not).
- `duplicate_activity` `kind`: `workout` if some live workout references the
  existing activity, else `run`.
- Structured logs: `user_id`, `workout_id`, `source_activity_id`,
  `outcome ∈ {imported,attached,duplicate,invalid}`.
- Handler tests per SOW § Tests.

### 6. Activities-feed exclusion
- Add `AND activity_type != 'strength_training'` to `List` and
  `ListInRange`. Test: feed excludes strength; running-metrics/best-efforts
  unaffected.

## Web changes (`prog-strength-web`)

- `lib/api.ts`: `createWorkoutFromTCX`, `attachWorkoutTCX`,
  `detachWorkoutTCX`; extend `Workout` with `activity_id: string | null` and
  optional `enrichment` (`avg_heart_rate_bpm`, `max_heart_rate_bpm`,
  `total_calories`, `duration_seconds`, `start_time`, `trackpoints?`). New
  `WorkoutTcxExistsError`; reuse `DuplicateRunError`-style for
  `duplicate_activity`.
- `WorkoutTCXUploadModal` (sibling of `UploadTCXModal`) — idle → uploading →
  success/error; `duplicate_activity` → "already in your log" + link;
  `workout_tcx_exists` → "already has a file attached — detach it first".
- Activities → Workouts toolbar: "Log from TCX" `ToolbarButton` beside
  "Start live workout"; on success navigate to `/workouts/{id}`.
- Workout detail: "Attach TCX" / "Detach TCX" header action (+ confirm
  modal); "♥ from Garmin" subtitle indicator; `HeartRateEffortCard`
  (4 `StatTile`s + `WorkoutHeartRateChart`, HR vs **elapsed time**) inserted
  between numbers row and exercise list, rendered only when enrichment
  present; empty-exercise state verified.
- Conform to design system tokens; mirror the running `HeartRateChart`
  styling for the chart (x-axis = elapsed time).
- Unit tests: `WorkoutHeartRateChart` renders on elapsed-time axis;
  workout detail renders card only when enrichment present + empty-exercise
  state.

## Docs (`prog-strength-docs`)
- Flip `sows/workout-tcx-enrichment.md` to `status: shipped`, body Status
  `Shipped`, Last updated `2026-06-19`. Open the docs PR with the required
  template.

## Rollout / merge order
1. `prog-strength-api` (migration + endpoints) — additive, deploy first.
2. `prog-strength-web` — depends on the API contract.
3. Hand-test the smoke checklist in the deployed env.
