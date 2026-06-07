# Implementation Plan: Running Tracking via Garmin TCX Import

**SOW**: `sows/running-tracking-via-tcx-import.md`
**Date**: 2026-06-06
**Repos**: prog-strength-infra, prog-strength-api, prog-strength-web, prog-strength-docs

This plan turns the SOW into discrete, subagent-sized tasks. Each task is
implemented by a subagent, then spec-reviewed and code-quality-reviewed
before moving on. Build/test gates are noted per task.

## Reconciled decisions (where the SOW assumed state the repos don't have)

The SOW was written slightly ahead of the codebase. Three reconciliations,
all additive and called out in the PRs:

1. **Error `code` field.** The current error envelope is
   `{service, version, error}` (`internal/httpresp/response.go`) with no
   `code`. Add `httpresp.ErrorWithCode(w, status, msg, code)` writing
   `{service, version, error, code}` (code omitempty). The 409 dedup
   response additionally carries `existing_session_id`, written via a
   small inline struct in the running handler.

2. **Preference write path.** `weight_unit` is stored on `users`, set at
   OAuth signup, and exposed **read-only** via `GET /me`; there is no
   settings page and no preference-write endpoint. To let the web persist
   the distance-unit toggle, add an additive `PATCH /me` that updates
   `display_name`, `weight_unit`, and `distance_unit` (all optional) via
   the existing `user.Repository.Update`. `distance_unit` is added to
   `users` exactly like `weight_unit` (enum, model field, GET /me).

3. **Default distance unit / settings UI.** Per Open Question #1 lean (a),
   the migration backfills existing users to `mi` and the column defaults
   to `mi`. The web gets a new `/settings` route (none exists today) with a
   distance-unit segmented control (and a weight-unit control alongside, to
   establish the "styled like the weight-unit control" pattern the SOW
   references), reachable from a new sidebar "Settings" entry.

Internally everything stays metric; conversion is render-time only.

## Tasks

### T1 — infra: `modules/tcx_storage/` (prog-strength-infra)
Mirror `modules/backup/` exactly.
- `modules/tcx_storage/{versions,variables,main,outputs}.tf`: private S3
  bucket `prog-strength-tcx-uploads`, versioning enabled, SSE-S3 (AES256),
  public-access-block all true, lifecycle expiring noncurrent versions
  after 30 days. IAM policy `${name_prefix}-tcx` granting
  `s3:PutObject/GetObject/DeleteObject` on `arn/*` and `s3:ListBucket` on
  the bucket arn, attached to `var.instance_role_name`.
- Root wiring: `variable "tcx_storage"` in `variables.tf`, `module
  "tcx_storage"` in `main.tf` (passing `name_prefix`, `bucket_name`,
  `instance_role_name = module.compute.instance_role_name`), root output
  `tcx_bucket_name`, and `tcx_storage = { bucket_name = "prog-strength-tcx-uploads" }`
  in `environments/prod.tfvars`.
- Gate: terraform not installed locally → mirror byte-for-byte on the
  backup module; rely on CI `fmt -check`/`validate`. PR lists the expected
  additive-only plan.

### T2 — api: migration (prog-strength-api)
- `internal/db/migrations/013_running.sql`: `running_sessions` +
  `running_trackpoints` per Data Model, `UNIQUE(user_id, garmin_activity_id)`,
  partial index `idx_running_sessions_user_start`, trackpoints composite PK
  + `FOREIGN KEY(session_id) ... ON DELETE CASCADE`. Also
  `ALTER TABLE users ADD COLUMN distance_unit TEXT NOT NULL DEFAULT 'mi'
  CHECK(distance_unit IN ('mi','km'))` (backfills existing rows to `mi`).
- `internal/db/migrate_test.go`: add a test that 013 applies cleanly and
  the new tables/constraints behave (unique fires, cascade deletes).
- Gate: `go test ./internal/db/...`.

### T3 — api: distance_unit preference + PATCH /me (prog-strength-api)
- `internal/user/distance_unit.go`: `DistanceUnit` enum (`mi`,`km`) +
  `Valid()`, mirroring `weight_unit.go`.
- `internal/user/user.go`: add `DistanceUnit` field (json `distance_unit`)
  + validate.
- `internal/user/sqlite_repository.go`: include `distance_unit` in INSERT,
  SELECTs, and UPDATE.
- `internal/user/handler.go`: add `PATCH /me` updating optional
  `display_name`/`weight_unit`/`distance_unit`; returns updated user.
- `internal/user/handler_test.go` + memory repo: cover GET includes
  distance_unit, PATCH happy + validation.
- Gate: `go test ./internal/user/...`.

### T4 — api: running domain pure core (prog-strength-api)
- `internal/running/model.go`: `Session`, `Trackpoint`, TCX parse structs.
- `tcx_parser.go` (encoding/xml subset), `tcx_validator.go`
  (`tcx_parse_failed`/`tcx_not_running`/`tcx_empty`), `tcx_summarizer.go`
  (distance/duration/avg+max HR/elevation gain/calories/avg pace, best-pace
  1km rolling window, ~300pt downsample with first/last + per-kept-point
  pace).
- `internal/running/testdata/`: typical_5k, intervals, marathon (~15k),
  biking, empty, malformed `.tcx` (generated fixtures).
- `tcx_parser_test.go`, `tcx_validator_test.go`, `tcx_summarizer_test.go`.
- Gate: `go test ./internal/running/...`.

### T5 — api: running repository + s3 archiver (prog-strength-api)
- `repository.go` (interface), `memory_repository.go`, `sqlite_repository.go`
  (insert session+trackpoints in one tx, list newest-first cursor,
  get-by-id with trackpoints, soft-delete, patch name, metrics aggregates,
  unique-violation → ErrDuplicate with existing id), `errors.go`,
  `s3_archiver.go` (Archiver interface + AWS SDK impl + a fake for tests).
- `sqlite_repository_test.go`: insert/list/get/soft-delete/ownership/unique.
- Gate: `go test ./internal/running/...`. Confirm AWS SDK dependency
  available (mirror however Litestream/SDK is vendored; if no SDK present,
  the real archiver uses `github.com/aws/aws-sdk-go-v2` added to go.mod).

### T6 — api: running handler + wiring (prog-strength-api)
- `handler.go`: 6 endpoints under `/running` per API Surface, DTOs
  (snake_case), structured import logging, `httpresp.ErrorWithCode`, 409
  body with `existing_session_id`, 413 at 10 MB, 415 non-multipart.
- `internal/httpresp/response.go`: add `ErrorWithCode`.
- `internal/server/server.go`: construct running repo + archiver from
  `TCX_BUCKET_NAME` env (graceful if empty → noop/local archiver in dev),
  mount `running.NewHandler(...)` in the JWT group.
- `handler_test.go`: 201/409/400-per-slug/413/list pagination/PATCH/DELETE
  then 404, fake S3 failure → 500 + rollback.
- Gate: `go build ./... && go test ./...`.

### T7 — web: api client + DistanceUnitContext + test setup (prog-strength-web)
- `lib/api.ts`: `RunningSession`/`Trackpoint`/metrics types; `listRunning`,
  `getRunning`, `importRunningTcx` (multipart, surfaces 409
  existing_session_id), `renameRunning`, `deleteRunning`, `getRunningMetrics`;
  extend `User` with `distance_unit`; `updateMe` (PATCH).
- `lib/distance-unit-context.tsx`: provider seeded from `getMe`, exposes
  `unit`, `setUnit` (optimistic + PATCH /me), `formatDistance(meters)`,
  `formatPace(secPerKm)`. Wrap app in `app/layout.tsx`.
- Vitest: add `vitest` + `@testing-library/react` devDeps, `vitest.config.ts`,
  `test` script; `lib/distance-unit-context.test.tsx` covering formatters
  under both units.
- Gate: `npm run typecheck` + `npm test`.

### T8 — web: sidebar, running pages, settings (prog-strength-web)
- `components/sidebar.tsx`: add `Running` (RunIcon) between Nutrition and
  Progress, and a `Settings` entry.
- `app/(app)/running/page.tsx` (Layout B: 4 StatTiles from metrics +
  cursor list + empty state + Upload button) and `[id]/page.tsx` (4×2 tiles,
  HR/Pace side-by-side, full-width Elevation, inline-edit name, delete modal).
- `app/(app)/running/_components/`: `StatTile`, `RunListRow`,
  `UploadTCXModal` (file input + drag-drop, idle/uploading/success/error,
  409 → "already in your log" + View run), `HeartRateChart`, `PaceChart`,
  `ElevationChart` (Recharts, shared `syncId`, distance x-axis per unit).
- `app/(app)/settings/page.tsx`: distance-unit + weight-unit segmented
  controls wired to context/PATCH /me.
- Gate: `npm run typecheck` + `npm run lint` + `npm run build`.

### T9 — docs: status flip (prog-strength-docs)
- Frontmatter `status: shipped`; body `**Status**: Shipped`,
  `**Last updated**: 2026-06-06`. Commit on this branch.

## Rollout / merge order
infra → api → web (per SOW Rollout). Docs PR is the ship signal.
