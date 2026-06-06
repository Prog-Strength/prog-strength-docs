---
status: draft
repos:
  - prog-strength-infra
  - prog-strength-api
  - prog-strength-web
  - prog-strength-docs
---

# Running Tracking via Garmin TCX Import

**Status**: Draft · **Last updated**: 2026-06-06

## Introduction

Prog Strength tracks strength workouts and nutrition well, but the cardio side of training is invisible to it. A user finishes a 6-mile morning run, sees their average pace and heart-rate trace on their Garmin watch, exports the activity as a TCX file from Garmin Connect — and then has nowhere in Prog Strength to put it. The 500 calories burned on that run never enter the daily energy balance the nutrition page reasons about; the pacing and HR data that tell the user how their aerobic base is progressing live in a separate ecosystem the rest of Prog Strength can't read. The compound effect is that a user who lifts and runs has to context-switch between two apps for one training week, and the macro / energy story Prog Strength tells stays half-true.

The Garmin TCX file is the natural seam to close this. Anyone with a Garmin watch already exports activities in TCX through Garmin Connect's "Export to TCX" button — the file is XML, contains a `<Sport>` tag, lap summaries with calories, and a dense trackpoint stream with time / distance / heart rate / altitude per second. The data needed for "show me a run" is already in the file the user can download in three clicks. Prog Strength gains a Running sidebar tab where a user uploads a TCX, sees the run rendered with the metrics they care about — average and max HR, average and best pace, distance, duration, calories, elevation gain — plus charts of heart rate and pace progression across the run. The list view across runs gives them a "how's my training going" feel in one glance.

The work is scoped tight on purpose. v0 is the existing-Garmin-user happy path — one TCX, one run, displayed well. No multi-sport, no mobile upload, no agent integration, no personal-records analytics, no week-over-week trend charts. Each of those is a clear follow-up SOW once the data model has settled and we see how users actually use the running tab.

## Proposed Solution

The TCX file is parsed once at upload time and the data it contains is split across two stores. The summary fields the run-list and stat tiles need — distance, duration, paces, HR stats, calories, elevation gain, start time, Garmin activity ID — are persisted to a new `running_sessions` table in SQLite. A downsampled trackpoint series (~300 points, regardless of original cadence) is persisted to a companion `running_trackpoints` table; the charts on the detail page read it directly. The original TCX file lands in a new private S3 bucket (`prog-strength-tcx-uploads`) keyed by `runs/<user_id>/<session_id>.tcx` as the source of truth — if we later want richer data we can re-parse from S3, no second upload from the user. Storing summaries alongside the file means the list view loads with a single fast query and a marathon's 15,000 trackpoints don't enter any read path that isn't asking for them.

The Go API gains a `internal/running/` domain package mirroring `internal/nutrition/`: model, repository (interface + memory + sqlite), handler, plus a small TCX parser/validator/summarizer that uses stdlib `encoding/xml` against Go structs that match the relevant subset of the TCX schema. Five endpoints under `/running` cover upload, list, detail, edit name, and delete; a sixth (`GET /running/metrics`) aggregates this-week / this-month / recent-avg-pace / all-time for the landing-page stats banner. Deduplication is enforced by a `UNIQUE(user_id, garmin_activity_id)` constraint at the DB layer, so a user who re-exports and re-uploads the same run gets a 409 with the existing session's ID — never a duplicate row.

The web client adds a Running entry to the sidebar nav and a `/running` route with two pages. The landing page shows four stat tiles on top (this week's distance + WoW delta, this month's distance + run count, recent avg pace, all-time totals) and a cursor-paginated list of runs below; an "Upload TCX" button in the header opens a modal that POSTs the file as `multipart/form-data`. The detail page shows a 4×2 grid of stat tiles (Distance, Avg Pace, Avg HR, Calories on top; Duration, Best Pace, Max HR, Elev Gain on bottom), then Heart Rate and Pace charts rendered side-by-side, then Elevation full-width below. All three charts share an x-axis of distance (mi or km per the user's preference) with synchronized hover tooltips via Recharts' `syncId`. The run name is inline-editable, and a Delete button in the header soft-deletes the run and navigates back to the list.

Display units are not hard-coded. The user's existing weight kg/lbs preference establishes the pattern; this SOW extends it with a parallel `preferred_distance_unit` (`miles` | `kilometers`) read from the existing user-preferences endpoint. A `DistanceUnitContext` on the web exposes `formatDistance(meters)` and `formatPace(secPerKm)` formatters so no component does unit math on its own. Internally everything stays metric (m, s, m/s) and the conversion happens at render time.

The new S3 bucket is provisioned via a new `modules/tcx_storage/` Terraform module that mirrors `modules/backup/` — private, versioned, SSE-S3 encrypted, with public-access blocked and a lifecycle rule for noncurrent versions. The bucket name is exposed as a root output and wired into the API container's `.env` via the existing compose pattern. The backend EC2 instance's IAM role gains a new policy granting `s3:PutObject / GetObject / DeleteObject / ListBucket` scoped to that bucket only — added via a `modules/tcx_storage/iam.tf` policy attached to `instance_role_name`, matching how the backup module attaches its Litestream policy.

No MCP, agent, or mobile changes in this SOW. The API contract is stable from day one so those clients pick it up in their own dispatches.

## Goals and Non-Goals

### Goals

- Provision a new `prog-strength-tcx-uploads` S3 bucket via a new `modules/tcx_storage/` Terraform module (mirroring `modules/backup/`): private, versioned, SSE-S3 encrypted, all public access blocked, lifecycle rule expiring noncurrent versions after 30 days. Expose `tcx_bucket_name` as a root-level output.
- Attach an IAM policy to the backend EC2 instance's role granting `s3:PutObject`, `s3:GetObject`, `s3:DeleteObject`, and `s3:ListBucket` scoped to that bucket only. Follow the `modules/backup/` pattern — the policy is owned by the `tcx_storage` module and attached to `instance_role_name`.
- Add a new migration `internal/db/migrations/01N_running.sql` (next free number at implementation time) creating `running_sessions` and `running_trackpoints` per the Data Model section. Includes `UNIQUE(user_id, garmin_activity_id)`, soft-delete column on sessions, `FOREIGN KEY ... ON DELETE CASCADE` on trackpoints, and an index on `(user_id, start_time DESC) WHERE deleted_at IS NULL`.
- Create a new `internal/running/` domain package containing `model.go`, `tcx_parser.go`, `tcx_validator.go`, `tcx_summarizer.go`, `s3_archiver.go`, `repository.go` (interface), `memory_repository.go`, `sqlite_repository.go`, and `handler.go`. Match the file naming and ownership patterns of `internal/nutrition/`.
- Mount under `/running` (auth via existing `auth.RequireUser` middleware):
  - `POST /running/sessions/imports` — multipart upload with field `file`. Parse → validate Sport=Running → check dedup → upload to S3 → write session + trackpoints in one DB transaction. Returns 201 + the new session, 409 with `{existing_session_id}` on dedup, 400 with a slug for each validation class, 413 on file > 10 MB, 500 on storage failure (DB rolled back, S3 object best-effort deleted).
  - `GET /running/sessions?limit=50&before=<ISO>` — cursor-paginated list newest first; sessions only, no trackpoints; returns `{sessions, next_before}`.
  - `GET /running/sessions/{id}` — returns the full session including its downsampled `trackpoints` array.
  - `PATCH /running/sessions/{id}` — body `{name: string}`. Updates only the name; returns the updated session. Name is non-blank, ≤ 200 chars.
  - `DELETE /running/sessions/{id}` — soft-delete (sets `deleted_at`). Trackpoints and S3 object are retained. Returns 204.
  - `GET /running/metrics` — aggregates for the stats banner: `current_week`, `current_month`, `recent_avg_pace_sec_per_km` (last 30 days), `all_time`. Each includes distance and run count where applicable.
- TCX parser supports the relevant subset of the schema: Activity (Sport, Id), Lap (StartTime, TotalTimeSeconds, DistanceMeters, Calories), Trackpoint (Time, DistanceMeters, HeartRateBpm.Value, AltitudeMeters). Other elements are ignored by `encoding/xml`. Reject with `tcx_parse_failed` if the file is malformed, `tcx_not_running` if Sport ≠ "Running", `tcx_empty` if no trackpoint has a non-zero distance.
- Compute summary fields from trackpoints (not from Lap roll-ups, which Garmin sometimes rounds): `distance_meters`, `duration_seconds`, `avg_pace_sec_per_km`, `avg_heart_rate_bpm`, `max_heart_rate_bpm`, `elevation_gain_meters` (sum of positive elevation deltas only). `total_calories` is the sum of `<Calories>` across laps — Garmin's number, trusted as-is.
- Compute `best_pace_sec_per_km` as the fastest 1-km rolling window across the trackpoints, not the fastest instantaneous trackpoint pace. See § Algorithms.
- Downsample to ~300 trackpoints per session: `stride = max(1, len(raw) / 300)`. Keep every `stride`-th point plus always the first and last. For each kept point, compute instantaneous pace from the previous kept point's elapsed time and distance.
- Add `preferred_distance_unit` (values: `miles`, `kilometers`) to the existing user-preferences model, endpoint, and DTO. Follow exactly the pattern used by the existing weight `preferred_weight_unit` preference.
- Web: add `/running` route under `app/(app)/running/`:
  - `page.tsx` — landing per Layout B (stats banner + list). Header: "Running" title + "Upload TCX" button. Stats banner: 4 `StatTile`s fed by `GET /running/metrics`. List: cursor-paginated rows linking to `/running/{id}` with a "Load more" button. Empty-state card explains how to export a TCX from Garmin Connect.
  - `[id]/page.tsx` — detail per Layout B (4×2 tiles + side-by-side HR/Pace + full-width Elevation). Header: breadcrumb, inline-editable run name (calls `PATCH /running/sessions/{id}`), subtitle with date/time/duration, Delete button (confirms via a small modal — soft-deletes and navigates back).
  - Components under `_components/`: `StatTile`, `RunListRow`, `UploadTCXModal`, `HeartRateChart`, `PaceChart`, `ElevationChart`, `DistanceUnitContext` (exposes `formatDistance(meters)` and `formatPace(secPerKm)`).
- Sidebar nav: add `{ label: "Running", href: "/running", icon: <RunIcon /> }` to the `NAV` array in `components/sidebar.tsx`. Place between Nutrition and Progress.
- Recharts time-series for the three charts. Use `syncId` to synchronize hover tooltips across them. X-axis is distance (mi or km per the user preference) for all three; Y-axis is bpm / pace / elevation respectively.
- Upload modal supports a file input (`accept=".tcx,application/xml"`) and drag-drop. States: idle → uploading → success (modal closes, list refetches, toast fires) / error (inline message). The 409 duplicate response renders "This run is already in your log" with a "View run" link to `existing_session_id`.
- Settings page extends to include a distance-unit segmented control (Miles / Kilometers), styled the same way the weight-unit control is today.
- API responses use the shape `{error: <message>, code: <slug>}`. The handler emits the slugs listed in the API Surface section.
- Observability: the import handler logs structured fields `user_id`, `garmin_activity_id`, `outcome ∈ {imported, duplicate, invalid, storage_failed}`. Per-route Prometheus metrics (request count + latency) come for free from existing middleware.
- New tests (per repo): see § Tests.

### Non-Goals

- **Multi-sport support.** Cycling and swimming TCXs are explicitly rejected in v0 with a clear error. The data model is running-shaped (pace, splits, elevation) — making it sport-generic now would force premature abstraction. A follow-up SOW can introduce a sport dimension and split-out per-sport schema where they diverge.
- **Mobile.** The mobile app's running surface (read-only view, native TCX picker via expo-file-system) is a follow-up dispatch once the web is in real use.
- **MCP and agent integration.** The agent can't introspect runs and can't help the user import. Future SOW: `list_running_sessions`, `get_running_session`, `get_running_metrics` tools + a small prompt rule for running questions.
- **Personal records and analytics.** No fastest-5K, fastest-mile, longest-run, pace-zones, HR-zones, lactate-threshold estimates, training-load, or recovery suggestions in v0. The data is there to compute them — the UI surface and decision-making are deferred.
- **Trend charts across runs.** No "pace over the last 12 weeks" or "weekly distance" chart in v0. The stats banner gives a coarse trend signal; richer trend surfaces are a follow-up.
- **Lap or split breakdowns.** Garmin's TCX laps are persisted only as part of the source-of-truth TCX in S3, not in SQLite. A future "Splits" panel on the detail page can re-parse from S3 without a schema change.
- **GPS / map view.** Trackpoint position (lat/lon) is intentionally not mapped into the Go structs in v0. The TCX in S3 retains it; a future "Route" tab can re-parse without a migration.
- **Elapsed-time x-axis toggle on the detail charts.** Charts use distance only. Adding a Distance / Elapsed-time toggle is a small follow-up; running by distance is the dominant runner mental model so v0 just commits to it.
- **Garmin Connect API direct integration.** Users export TCX manually for v0. OAuth + the Garmin Wellness/Activity API is large enough to be its own SOW.
- **Other TCX sources.** Strava and Apple Health also export TCX, and most should work, but v0 only commits to Garmin's flavor; tickets for other sources are out of scope unless the validator already accepts them by accident.
- **Editable fields other than name.** Stats are derived from the file; editing them by hand would un-couple the row from the TCX. Re-upload (delete + new TCX) is the path if a stat is wrong.
- **Batch upload.** v0 accepts one file per request. Backfilling N runs means N uploads. Multi-file is a clean follow-up if users ask.
- **Calorie computation.** The TCX `<Calories>` field is used as-is. No HR + duration + weight-based recomputation; no Keytel formula. Garmin's number is calibrated to the user's profile and Prog Strength doesn't yet store the demographics needed to do better.
- **Mobile- or watch-direct sync.** Out of scope.
- **Running goal tracking** (e.g., weekly mileage targets, race countdowns). Follow-up.

## Implementation Details

### Data Model

One new migration: `internal/db/migrations/01N_running.sql` (use the next free migration number at implementation time; current head is 012).

**Table `running_sessions`:**

| Column | Type | Description |
| --- | --- | --- |
| `id` | TEXT | Primary key. UUID generated server-side. |
| `user_id` | TEXT | Owner. Not a FK only because the existing pattern in nutrition/workouts doesn't FK to users either. |
| `garmin_activity_id` | TEXT | From `<Activity Id="...">` — Garmin's per-activity ISO timestamp. Dedup key. |
| `start_time` | DATETIME | UTC. From the first lap's `StartTime` attr. |
| `name` | TEXT | Nullable. Initial value from TCX `<Notes>` if present, else null. User-editable via PATCH. |
| `distance_meters` | REAL | Total distance, computed from trackpoint cumulative distance. |
| `duration_seconds` | INTEGER | Total elapsed time, computed from first/last trackpoint timestamps. |
| `avg_pace_sec_per_km` | REAL | `duration_seconds / (distance_meters / 1000)`. |
| `best_pace_sec_per_km` | REAL | Fastest 1-km rolling window. See § Algorithms. |
| `avg_heart_rate_bpm` | INTEGER | Mean of trackpoint HR values, rounded. Nullable if no HR data. |
| `max_heart_rate_bpm` | INTEGER | Max trackpoint HR. Nullable if no HR data. |
| `total_calories` | INTEGER | Sum of `<Calories>` across laps. Nullable if absent. |
| `elevation_gain_meters` | REAL | Sum of positive elevation deltas across trackpoints. Nullable if no elevation data. |
| `tcx_s3_key` | TEXT | `runs/<user_id>/<session_id>.tcx`. |
| `created_at` | DATETIME | Server-side `time.Now().UTC()` at insert. |
| `deleted_at` | DATETIME | Soft-delete timestamp. Nullable. |

Constraints and indexes:

- `UNIQUE(user_id, garmin_activity_id)` — enforces dedup; concurrent uploads of the same TCX result in one row + one 409.
- `CREATE INDEX idx_running_sessions_user_start ON running_sessions(user_id, start_time DESC) WHERE deleted_at IS NULL;` — partial index optimizing the list query.

**Table `running_trackpoints`:**

| Column | Type | Description |
| --- | --- | --- |
| `session_id` | TEXT | FK to `running_sessions(id)` with `ON DELETE CASCADE`. |
| `sequence` | INTEGER | Order within the session (0..N). Composite PK with `session_id`. |
| `elapsed_seconds` | INTEGER | Offset from session start. |
| `distance_meters` | REAL | Cumulative distance at this point. |
| `heart_rate_bpm` | INTEGER | Nullable. |
| `pace_sec_per_km` | REAL | Instantaneous pace at this point. Computed from delta vs the previous kept point. Nullable. |
| `elevation_meters` | REAL | Nullable. |

`PRIMARY KEY (session_id, sequence)`. The composite key fits the access pattern — we always read all trackpoints for a single session in order, never query trackpoints standalone.

### Write Path

The upload endpoint is the only write that touches both the DB and S3. Order matters because we want no orphans on either side:

- **Parse + validate in memory.** If the file is malformed, missing trackpoints, or not a Running activity, return 4xx — nothing persisted.
- **Compute summary + downsampled trackpoints in memory.** Pure functions, no I/O.
- **Open a SQLite transaction.** Insert the `running_sessions` row, then the trackpoints. **Do not commit yet.** The `UNIQUE` constraint may fire here on dedup — translate to 409 with `existing_session_id` (re-query by `user_id + garmin_activity_id`).
- **Upload the TCX to S3** at `runs/<user_id>/<session_id>.tcx`. If the put fails, roll back the transaction and return 500 with `code: storage_failed`. The S3 client is constructed once at API startup using the backend EC2 instance's role credentials (same pattern as Litestream).
- **Commit the DB transaction.** On commit failure (rare — SQLite WAL mode usually commits cleanly), best-effort `DeleteObject` the S3 file. Log the orphan if delete also fails — we can sweep later.
- **Return 201** with the new session DTO.

PATCH name is straightforward: validate non-blank + ≤ 200 chars, single `UPDATE` on `running_sessions`. DELETE is a single `UPDATE running_sessions SET deleted_at = ? WHERE id = ? AND user_id = ?`. Neither touches S3.

### Algorithms

**Best pace via 1-km rolling window.** A naïve "minimum pace_sec_per_km across all trackpoints" picks up GPS jitter — Garmin's distance jumps by half a meter between consecutive 1-second samples are normal; dividing that delta by 1 second produces noise that beats any real best-pace number. Instead:

```
walk trackpoints with two pointers (left, right):
  expand right until cumulative_distance[right] - cumulative_distance[left] >= 1000 meters
  if window <= 1km:
    window_pace = (elapsed[right] - elapsed[left]) / 1.0  // seconds per km
    best = min(best, window_pace)
  advance left
```

The window is anchored on distance, not on count of trackpoints — a fast runner will see a 1-km window in fewer trackpoints than a walker, and the algorithm doesn't care. If the session is shorter than 1 km total, `best_pace_sec_per_km` is null.

**Downsampling.** The chart axis is rendered in pixels and Recharts can't usefully draw more points than there are horizontal pixels at the typical detail-page width. Target ~300 points:

```
stride = max(1, len(raw_trackpoints) / 300)
kept = [raw_trackpoints[i] for i in range(0, len(raw), stride)]
if kept[-1] is not raw_trackpoints[-1]:
    kept.append(raw_trackpoints[-1])
```

For each kept point, recompute `pace_sec_per_km` between consecutive kept points so the pace shown in the chart reflects the kept-point cadence, not the raw one (raw-point pace would oscillate with the GPS noise floor and look terrible smoothed only by Recharts).

**Elevation gain.** Sum of positive deltas between consecutive trackpoints. Negative deltas are ignored — descents don't subtract from "gain." If no trackpoint has an altitude value, store null rather than 0; null distinguishes "no data" from "flat course."

### API Surface

All endpoints require JWT auth via existing middleware. Error responses use `{ "error": "<message>", "code": "<slug>" }`.

`POST /running/sessions/imports`
- `Content-Type: multipart/form-data` with field `file`.
- 201 + `RunningSessionDTO` on success.
- 400 with code `tcx_not_running` | `tcx_parse_failed` | `tcx_empty` on validation failure.
- 409 with code `duplicate_run` and body `{ error, code, existing_session_id }` on dedup.
- 413 with code `file_too_large` on file > 10 MB.
- 415 with code `unsupported_media_type` on non-multipart or missing `file` field.
- 500 with code `storage_failed` on S3 / DB failure.

`GET /running/sessions?limit=50&before=<ISO>`
- 200 + `{ sessions: RunningSessionDTO[], next_before: string | null }`. Newest first. `limit` defaults to 50, max 100. `sessions` excludes the `trackpoints` array — the list view doesn't need it.

`GET /running/sessions/{id}`
- 200 + `RunningSessionDTO` including the `trackpoints` array. 404 if the session is missing or belongs to another user.

`PATCH /running/sessions/{id}`
- Body: `{ "name": string }`.
- 200 + the updated `RunningSessionDTO`. 400 if name is blank or > 200 chars. 404 as above.

`DELETE /running/sessions/{id}`
- 204 on success. 404 as above. Soft-delete; trackpoints and TCX file retained.

`GET /running/metrics`
- 200 + the aggregates payload:
  ```json
  {
    "current_week": { "distance_meters": 29773.4, "run_count": 3, "delta_pct_vs_prior_week": 12 },
    "current_month": { "distance_meters": 116356.8, "run_count": 8 },
    "recent_avg_pace_sec_per_km": 311.6,
    "all_time": { "distance_meters": 663087.3, "run_count": 47 }
  }
  ```
- "Week" is the user's local-time ISO week boundary. The user's timezone comes from the existing `client_timezone` plumbing the nutrition aggregator uses.

`RunningSessionDTO` shape (camelCase keys not shown; the existing codebase uses snake_case):

```json
{
  "id": "uuid",
  "garmin_activity_id": "2026-06-05T13:23:45.000Z",
  "name": "Morning Run",
  "start_time": "2026-06-05T13:23:45Z",
  "distance_meters": 9990.4,
  "duration_seconds": 3074,
  "avg_pace_sec_per_km": 307.7,
  "best_pace_sec_per_km": 261.0,
  "avg_heart_rate_bpm": 148,
  "max_heart_rate_bpm": 172,
  "total_calories": 512,
  "elevation_gain_meters": 39.1,
  "created_at": "2026-06-05T13:35:11Z",
  "trackpoints": [
    { "sequence": 0, "elapsed_seconds": 0, "distance_meters": 0, "heart_rate_bpm": 102, "pace_sec_per_km": null, "elevation_meters": 1654.2 },
    ...
  ]
}
```

`trackpoints` is present on the single-session detail response and absent from list-response items.

### Tests

**Go API:**

- `internal/running/tcx_parser_test.go`, `tcx_validator_test.go`, `tcx_summarizer_test.go`: table-driven against `internal/running/testdata/`:
  - `typical_5k.tcx` — summary math (distance ± 10 m, duration ± 1 s, avg HR exact, calories exact, elevation gain ± 1 m).
  - `intervals.tcx` — verifies the 1-km rolling `best_pace` algorithm picks the fastest km, not the noisiest instantaneous trackpoint; verifies downsampling preserves first/last and peak-HR shape.
  - `marathon.tcx` — ~15 K trackpoints — stride math sanity-checked, ~300 kept points.
  - `biking.tcx` — validator rejects with `tcx_not_running`.
  - `empty.tcx`, `malformed.tcx` — parser/validator reject without panic.
  - Round-trip unit sanity: metric stored → `formatDistance` / `formatPace` produce correct miles + min/mi strings.
- `internal/running/sqlite_repository_test.go`: temp SQLite per test, exercise insert + list newest-first + get-by-id + soft-delete + ownership isolation (user A cannot read user B's runs) + unique-constraint behavior on `(user_id, garmin_activity_id)`.
- `internal/running/handler_test.go` with `httptest` + in-memory repo + a fake S3 client (interface, not the AWS SDK): 201 import path, 409 dedup with `existing_session_id` in body, 400 for each `code` slug, 413 oversized, 500 + DB rollback when fake S3 returns an error, list pagination via `before=`, PATCH name happy + validation, soft-delete then 404 on follow-up GET. Auth middleware bypassed via test context, same trick `nutrition/handler_test.go` uses.
- Migration test: existing `db.Migrate()` harness applies cleanly on an empty DB.

**Web:**

- Component unit tests for `DistanceUnitContext`'s `formatDistance(meters)` and `formatPace(secPerKm)` under both `miles` and `kilometers` preferences. Pure functions; covering them prevents whole-class unit bugs.
- No E2E framework introduced. PR description includes a smoke checklist:
  - Upload a known-good TCX via the modal → row appears in the list → click → detail renders all 8 tiles + 3 charts.
  - Upload the same TCX again → modal shows the duplicate message + View Run link.
  - Upload a biking TCX → inline error in the modal.
  - Edit run name inline → page updates without full refresh.
  - Delete a run → confirms, soft-deletes, navigates back to list, row gone.
  - Toggle distance unit in settings → list + detail re-render with the new unit.
  - Empty state with zero runs renders the "how to export from Garmin Connect" card.

**Infra:**

- `terraform validate` runs in CI for the new `modules/tcx_storage/`.
- Expected `terraform plan` against prod: only `+` lines — `aws_s3_bucket.tcx_uploads`, `aws_s3_bucket_versioning`, `aws_s3_bucket_server_side_encryption_configuration`, `aws_s3_bucket_lifecycle_configuration`, `aws_s3_bucket_public_access_block`, `aws_iam_policy.tcx_uploads_rw`, `aws_iam_role_policy_attachment` — no destructive changes. The PR description lists this as the expected plan.

### Rollout

Coordinated cross-repo dispatch. Merge order:

1. `prog-strength-infra` — new `modules/tcx_storage/` + IAM attachment + root output + tfvars wiring. Apply via the standard PR review flow. Bucket exists before the API tries to write to it.
2. `prog-strength-api` — migration + `internal/running/` package + new endpoints + `preferred_distance_unit` preference field. The API picks up the new bucket from its `.env` (`TCX_BUCKET_NAME`). Deploys via the existing release workflow.
3. `prog-strength-web` — sidebar nav entry, `/running` route, components, settings extension. Deploys via the existing release workflow.
4. Hand-test in the deployed environment: real account, real TCX, walk the smoke checklist from § Tests.

Each PR is independently revertable. The API change is additive — no existing read or write path changes. If the web PR ships before the user has runs, the landing page renders the empty-state card and zero-value metrics tile.

## Open Questions

1. **Default `preferred_distance_unit` for existing users.** Options: (a) backfill all existing users to `miles` (US default; matches the assumption behind the rest of the product), (b) leave the column nullable and render miles by default on the client when null, (c) prompt the user to choose on first visit to the Running tab. Tentative lean: (a) — backfill in the migration. Mirrors how the existing weight-unit preference handles unset values, and avoids both the "null branching" code path and the "first-visit modal" UX.
2. **X-axis units toggle on the detail charts.** Distance vs elapsed time. Some runners read paces best plotted against time (especially for intervals where distance compresses on rest). Options: (a) ship distance only in v0; (b) add a small Distance / Time segmented control above the chart row. Tentative lean: (a). Distance is the dominant default; the toggle is a clean follow-up.
3. **What happens to the TCX in S3 when a session is soft-deleted, then hard-purged.** Soft-delete keeps the object. We currently have no hard-purge path. Options: (a) accept the orphan growth (S3 lifecycle versioning expiration helps); (b) add a periodic GC that deletes S3 objects for sessions where `deleted_at < now - 90 days`; (c) hard-delete the object at soft-delete time and accept that undelete becomes impossible. Tentative lean: (a) — the storage is cheap relative to the operational complexity of (b), and (c) closes a recovery door that costs us nothing to leave open.
4. **TCX files larger than 10 MB.** The cap is generous for typical Garmin output but a multi-day FKT or an ultra with high-res sensor data could exceed it. Options: (a) keep the cap, return 413 with a friendly message; (b) raise to 25 MB; (c) chunked / streamed parse. Tentative lean: (a) until we see a real 413. The handler logs the rejection with byte count so we'll know when to revisit.
