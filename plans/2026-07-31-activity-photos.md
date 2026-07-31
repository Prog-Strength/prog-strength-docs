# Activity Photos Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let a user attach many ordered, optionally-captioned photos to any activity; store server-authoritative re-encoded JPEG variants in a private S3 bucket; render a photo strip + lightbox on the web activity detail page, a cover thumbnail with `+N` badge on the timeline card, and capture/display on mobile.

**Architecture:** A new `activity_photo` table hangs photos off the `activities` base table (one implementation for every type). Bytes flow through the API: decode → apply EXIF orientation → resize (CatmullRom) → re-encode two JPEG variants (`full`, `thumb`), which strips all EXIF/GPS. Objects land Hive-partitioned in a dedicated bucket. Retrieval is windowed presigned GETs (byte-identical URL within a 6h window → browser-cache hits). Graceful 503 when no bucket configured. Pure Go, no cgo.

**Tech Stack:** Go 1.25 (chi, aws-sdk-go-v2, `golang.org/x/image/{draw,webp}`, stdlib `image/jpeg`+`image/png`), SQLite; Terraform (AWS provider ~>6); Next.js 16 / React 19 / Vitest / Tailwind; Expo 55 / React Native / NativeWind.

**Repos & feature branch:** `feat/activity-photos` in each of: `prog-strength-api`, `prog-strength-infra`, `prog-strength-web`, `prog-strength-mobile`, `prog-strength-docs`.

## Local gate (run before every commit/push in the API repo)

```
export CGO_CFLAGS="-I/home/developer/sqlite-include"
export PATH="$(go env GOPATH)/bin:$PATH"
gofmt -l .            # must print nothing
go build ./...
go vet ./...
golangci-lint run --timeout=5m
go mod tidy && git diff --exit-code go.mod go.sum
go test ./...
```

`sqlite3.h` lives at `/home/developer/sqlite-include` (needed by the `sqlite-vec` cgo binding). golangci-lint is pinned **v2.12.2**.

---

## Reference patterns (read these; do not re-derive)

- **S3 key builder & validation:** `internal/activity/tcx_key.go` — reuse `idPartPattern`, `ErrInvalidKeyPart`, `ErrInvalidActivityType`. Layout uses `activityDate.UTC()`.
- **Object store seam:** `internal/user/avatar_store.go` — `AvatarStore` interface (`Put`, `PresignGet`, `TagOrphaned`), `S3AvatarStore`, orphan tag constants. Mirror exactly for photos.
- **Write handler:** `internal/user/handler.go` `uploadAvatar` (lines 241–331) — `MaxBytesReader`, `ParseMultipartForm`, `MaxBytesError` → 413 `file_too_large`, `FormFile`, `http.DetectContentType` → 415 `unsupported_media_type`, `httpresp.ErrorWithCode`, best-effort `TagOrphaned`.
- **Unified activity handler & routing:** `internal/activity/handler.go` (`Mount`, `Handler` struct, setters, `activityDTO`, `toActivityDTO`), `internal/activity/unified_handler.go`.
- **Repository:** `internal/activity/repository.go` (`Repository` interface, `Get(ctx, userID, id)`), `internal/activity/sqlite_repository.go`, test helper `newMigratedDB(t)` in `sqlite_repository_test.go`.
- **Config:** `internal/config/config.go` — `Storage` section + flat `Config` fields + `Load` mapping; `config.toml` `[storage]` + `[hr_zones]` as the section-with-literals precedent.
- **Timeline hydration:** `internal/timeline/handler.go` (`contentDTO`, `postDTO`, `toContentDTO`), `internal/timeline/model.go` (`PostContent`, `PostRef`), `internal/server/timeline_hydrator.go` (`hydrateSessions`).
- **Server wiring:** `internal/server/server.go` — avatar store construction (147–154), activity handler construction + setters (526–570), timeline hydrator (555), user handler mount (603).
- **Migrations:** `internal/db/migrations/NNN_*.sql`, additive style of `038_activity_route.sql`; embedded via `//go:embed migrations/*.sql`.
- **ID:** `internal/id/` → `id.New()` (16-byte hex).
- **Infra:** `modules/avatar_storage/{main,variables,outputs,versions}.tf`; root `main.tf` module block, `outputs.tf`, `variables.tf` object var, `environments/prod.tfvars`, `compose/api/config.env`, `deploy/api.sh` `REQUIRED_ENV_KEYS`.
- **Web:** `lib/api.ts` (`uploadAvatar` FormData precedent), `app/(app)/workouts/[id]/page.tsx`, `app/(app)/timeline/_components/TimelinePostCard.tsx`, `lib/profile-context.tsx` (`useProfile`), `components/workout-details-edit-modal.tsx` (modal w/ Escape + scroll-lock), `app/globals.css` tokens (`--accent`, `--radius-card` 14px, `--accent-line`, `--accent-soft`).
- **Mobile:** `lib/api.ts` (`uploadAvatar` RN FormData `{uri,name,type}` precedent, `PickedImage`), `app/(tabs)/activities/{workout,run}/[id].tsx`, `app/settings.tsx` (ImagePicker usage), `app.json` (expo-image-picker plugin permissions), `tailwind.config.js`.

Config constants (from SOW): `max_per_activity=10`, `max_upload_bytes=12582912`, `full_max_edge_px=2048`, `full_jpeg_quality=82`, `thumb_max_edge_px=480`, `thumb_jpeg_quality=78`, `presign_window_hours=6`, `caption_max_chars=200`. Orphan tag: `photo-status=orphaned`. Bucket env var: `PHOTO_BUCKET_NAME`.

---

# PART A — prog-strength-api

Work from `/workspace/prog-strength-api` on branch `feat/activity-photos`. TDD throughout. Commit after each task with a conventional-commit message (`feat:`/`test:`, lowercase subject).

## Task A1: Migration 046 — activity_photo table

**Files:** Create `internal/db/migrations/046_activity_photos.sql`

- [ ] Write the migration. Purely additive: one `CREATE TABLE activity_photo` + two indexes. No `CHECK` on `content_type` (house style, per migration 042). Columns exactly per the SOW Data Model table:
  `id TEXT PRIMARY KEY`, `activity_id TEXT NOT NULL REFERENCES activities(id) ON DELETE CASCADE`, `user_id TEXT NOT NULL`, `s3_key TEXT NOT NULL`, `thumb_s3_key TEXT NOT NULL`, `content_type TEXT NOT NULL`, `byte_size INTEGER NOT NULL`, `width INTEGER NOT NULL`, `height INTEGER NOT NULL`, `caption TEXT`, `position INTEGER NOT NULL`, `created_at DATETIME NOT NULL`, `updated_at DATETIME NOT NULL`, `deleted_at DATETIME`.
  Indexes: `idx_activity_photo_activity(activity_id, position, id)`, `idx_activity_photo_user(user_id, created_at DESC)`. `position` is NOT unique (documented rationale). Header comment in the style of 045/038.
- [ ] Verify it applies: `newMigratedDB(t)` in any repo test already runs all migrations; run `go test ./internal/db/... ./internal/activity/...` — expected PASS (migration applies clean).
- [ ] Commit: `feat: add migration 046 activity_photo table`

## Task A2: photo_key.go — Hive-partitioned S3 key builder

**Files:** Create `internal/activity/photo_key.go`, `internal/activity/photo_key_test.go`

- [ ] **TDD:** Write `photo_key_test.go` first, table-driven in the shape of `tcx_key_test.go`. Cases: rejects `/`, `=`, `.`, whitespace, empty in `userID`/`activityID`/`photoID` (→ `ErrInvalidKeyPart`); rejects unregistered activity type (→ `ErrInvalidActivityType`); asserts exact layout for both `full` and `thumb`; asserts a `start_time` in a non-UTC zone (e.g. `-07:00`) partitions on its **UTC** date. Also reject an invalid `variant`.
- [ ] Run: `go test ./internal/activity/ -run PhotoKey` — expect FAIL (undefined).
- [ ] Implement `buildPhotoKey(userID string, activityType ActivityType, activityStart time.Time, activityID, photoID string, variant photoVariant) (string, error)` reusing `idPartPattern`, `ErrInvalidKeyPart`, `ErrInvalidActivityType`. Define `photoVariant` type with `photoVariantFull="full"`, `photoVariantThumb="thumb"`, and validate `photoID` with `idPartPattern`. Layout:
  `user_id={u}/activity_type={t}/year={yyyy}/month={mm}/day={dd}/activity_id={a}/variant={v}/{photoID}.jpg` using `activityStart.UTC()`. Always `.jpg`.
- [ ] Run tests — expect PASS.
- [ ] Commit: `feat: add activity photo s3 key builder`

## Task A3: Image pipeline — decode, orient, resize, re-encode two variants

**Files:** Create `internal/activity/photo_pipeline.go`, `internal/activity/photo_pipeline_test.go`, fixtures under `internal/activity/testdata/photos/`. Add deps `golang.org/x/image/draw`, `golang.org/x/image/webp` (run `go get`, then `go mod tidy`).

- [ ] **TDD:** Write golden tests. Generate fixtures programmatically in the test (a small gradient image encoded as JPEG/PNG; for WebP decode, ship one tiny committed `.webp` fixture — WebP is decode-only via x/image). Assert, for each of the 8 EXIF `Orientation` values applied to a JPEG fixture: output magic bytes are JPEG (`\xFF\xD8\xFF`) on both variants; output long edge ≤ configured max and aspect ratio preserved; orientation applied (a portrait-flagged landscape-pixel image comes out with the expected width<height, using a distinctly-colored corner marker to verify the transform); an under-size image is NOT upscaled; **no EXIF/GPS survives** in either variant (re-parse output with an EXIF reader or assert the APP1/EXIF marker is absent). Cover JPEG, PNG, WebP source decode.
- [ ] Run — expect FAIL.
- [ ] Implement a `processPhoto(src []byte, opts photoPipelineOpts) (full, thumb processedImage, err error)` where `processedImage` carries `bytes []byte`, `width`, `height`. Steps: sniff/decode via `image/jpeg`, `image/png`, `golang.org/x/image/webp`; read EXIF `Orientation` (parse the APP1 EXIF segment for JPEG — a small local reader for the Orientation tag is fine; PNG/WebP have none) and apply the correct rotate/flip for all 8 values **before** resize; resize with `golang.org/x/image/draw` `draw.CatmullRom` preserving aspect, clamping long edge, never upscaling; encode both variants with `image/jpeg` at the configured quality. `opts` carries `fullMaxEdge, fullQuality, thumbMaxEdge, thumbQuality int`. The re-encode is the only EXIF path (no fast-path skip) — assert this in tests.
- [ ] Run — expect PASS. Run `gofmt`, `go vet`, `golangci-lint run`.
- [ ] Commit: `feat: add server-authoritative photo image pipeline`

## Task A4: PhotoStore seam + windowed presigner

**Files:** Create `internal/activity/photo_store.go`, `internal/activity/photo_store_test.go`

- [ ] **TDD:** Write `photo_store_test.go` for the windowed presigner logic that needs no network: two presigns one second apart inside the same window are byte-identical; two straddling a window boundary differ; `X-Amz-Expires` equals `2 * window`. Test `windowedPresigner` in isolation (construct with a static `aws.Credentials` and region, call its presign method twice with controlled `now`). Also add a `FakePhotoStore` (in-memory: records `Put`s keyed by key, returns a deterministic URL from `PresignGet`, records `TagOrphaned`) — used by handler tests in A6/A7.
- [ ] Run — expect FAIL.
- [ ] Implement:
  - `PhotoStore` interface: `Put(ctx, key, contentType string, body []byte, cacheControl string) error`, `PresignGet(ctx, key string) (string, error)`, `TagOrphaned(ctx, key string) error`. (Include `cacheControl` so the SOW's `Cache-Control: private, max-age=31536000, immutable` is written on Put; or set it as a constant inside Put — pick one and be consistent.)
  - `windowedPresigner`: built on `github.com/aws/aws-sdk-go-v2/aws/signer/v4` `Signer.PresignHTTP`, given `now`, `window time.Duration`, bucket, region, credentials. `signingTime = now.Truncate(window)`, `X-Amz-Expires = 2*window`. Provide a `now func() time.Time` seam (default `time.Now`) for tests.
  - `S3PhotoStore` implementing `PhotoStore` with `Put`/`TagOrphaned` via `s3.Client` (mirror `S3AvatarStore`) and `PresignGet` via the `windowedPresigner`. Orphan tag constants `photoOrphanTagKey="photo-status"`, `photoOrphanTagValue="orphaned"` with the mandatory "MUST match the lifecycle rule tag filter" comment.
  - `NewS3PhotoStore(ctx, bucket, region string, window time.Duration) (*S3PhotoStore, error)`.
- [ ] Run — expect PASS. Gate.
- [ ] Commit: `feat: add photo store seam with windowed presigner`

## Task A5: Photo repository (CRUD + batched cover query)

**Files:** Create `internal/activity/photo_repository.go`, `internal/activity/photo_repository_test.go`. Extend `internal/activity/sqlite_repository.go`.

- [ ] **TDD:** `photo_repository_test.go` using `newMigratedDB(t)`. Insert an activity first (use the existing repo/test helpers to create an activity, or insert a minimal `activities` row). Test: `InsertPhoto` sets `position = COALESCE(MAX(position),-1)+1` scoped to activity; `ListByActivity` returns live photos ordered `(position, id)`, excluding soft-deleted and excluding photos whose **parent activity** is soft-deleted; `CountLive` per activity; `SoftDeletePhoto`; `UpdateCaption`; `ReorderPhotos` rewrites positions in one tx; `CoverPhotosByActivityIDs(ids)` returns one cover (rn=1 via `ROW_NUMBER() OVER (PARTITION BY activity_id ORDER BY position, id)`) + count per activity in a **single** query; lifecycle: soft-deleting the parent activity hides photos, restoring (clear `deleted_at`) brings them back, hard delete cascades.
- [ ] Run — expect FAIL.
- [ ] Implement a `PhotoRepository` interface + methods on `*SQLiteRepository` (or a dedicated struct sharing the `*sql.DB`). Model type `ActivityPhoto` (domain struct with all columns). The cover query joins `activities` and filters `activities.deleted_at IS NULL AND activity_photo.deleted_at IS NULL`. Returns `map[activityID]struct{Cover ActivityPhoto; Count int}`.
- [ ] Run — expect PASS. Gate.
- [ ] Commit: `feat: add activity photo repository`

## Task A6: Config — [photos] section + photo bucket wiring

**Files:** `internal/config/config.go`, `config.toml`, `internal/config/config_test.go`

- [ ] **TDD:** Extend `config_test.go` to assert the parsed defaults for the new `[photos]` fields and `PhotoBucketName` from `[storage]`.
- [ ] Add to `[storage]` in `config.toml`: `photo_bucket_name = "${PHOTO_BUCKET_NAME}"`. Add `[photos]` section with the 8 literals from the SOW. Mirror in `fileConfig.Storage` (`PhotoBucketName string \`toml:"photo_bucket_name"\``) and a new `fileConfig.Photos` struct; add `PhotoBucketName string` + a `Photos PhotosConfig` struct to `Config`; map in `Load`. Add `PhotosConfig` with `MaxPerActivity, MaxUploadBytes int64, FullMaxEdgePx, FullJPEGQuality, ThumbMaxEdgePx, ThumbJPEGQuality int, PresignWindowHours int, CaptionMaxChars int`. Add defaults in `applyDefaults` if a value is zero (so an operator override of one knob doesn't zero the rest). Photo bucket, like avatar, requires `AWSRegion` when set — extend the existing region validation.
- [ ] Run `go test ./internal/config/...` — PASS. Gate.
- [ ] Commit: `feat: add photos config section and photo bucket name`

## Task A7: Photo HTTP handler — write path

**Files:** Create `internal/activity/photo_handler.go`, `internal/activity/photo_handler_test.go`. Wire routes in `handler.go` `Mount`.

- [ ] **TDD:** `photo_handler_test.go` against `FakePhotoStore` + `newMigratedDB`, hermetic. Cases: happy-path `POST /activities/{id}/photos` (multipart field `photo`) → 201 with hydrated DTO (both URLs presigned); oversize → 413 `file_too_large`; disallowed sniffed type → 415 `unsupported_media_type`; over-limit (`max_per_activity`) → 409 `photo_limit_reached`; another user's activity → 404 (no existence leak); second-put failure → no row written AND first object tagged orphaned (make `FakePhotoStore.Put` fail on the 2nd call); `PATCH` caption over-length → 400; `PUT .../order` rejects subset/extra/duplicate → 400, valid reorder idempotent + positions match indices; `DELETE` soft-deletes + tags both objects; storage unconfigured (nil store) → 503 `photo_storage_unavailable`.
- [ ] Run — expect FAIL.
- [ ] Implement handlers on the activity `Handler` (add `photoStore PhotoStore`, `photoRepo PhotoRepository`, `photoCfg PhotosConfig`, and `now` is already present; add `SetPhotoStore(store, repo, cfg)` setter, nil-safe). Endpoints per SOW Write Path:
  - `POST /activities/{id}/photos`: nil store → 503; resolve activity for user (`repo.Get`, miss → 404); `MaxBytesReader`+`ParseMultipartForm(maxUploadBytes)`; `FormFile("photo")`; `io.ReadAll` (413 on `MaxBytesError`); `http.DetectContentType` allowlist `{image/jpeg,image/png,image/webp}` → else 415; `CountLive` ≥ max → 409; run pipeline; build both keys from `(user_id, activity_type, activity.StartTime, activity_id, photo_id, variant)`; `Put` full then thumb — if thumb put fails, `TagOrphaned` the full key (best-effort) and return 500 without inserting; insert row (`position` via COALESCE); 201 with hydrated photo DTO.
  - `PATCH /activities/{id}/photos/{photo_id}`: caption-only, validate `<= caption_max_chars` → 400; ownership 404.
  - `PUT /activities/{id}/photos/order`: body `{photo_ids:[...]}`; reject unless the set exactly equals the activity's live photo ids (no subset/extra/dupe) → 400; rewrite positions in one tx.
  - `DELETE /activities/{id}/photos/{photo_id}`: soft-delete, best-effort tag both objects orphaned (log failures).
  - Register routes inside `Mount`'s `/activities` subrouter: `r.Post("/{id}/photos", ...)`, `r.Patch("/{id}/photos/{photo_id}", ...)`, `r.Put("/{id}/photos/order", ...)`, `r.Delete("/{id}/photos/{photo_id}", ...)`.
  - Photo DTO shape: `{id, url, thumb_url, width, height, caption, position}`.
- [ ] Run — expect PASS. Gate.
- [ ] Commit: `feat: add activity photo write endpoints`

## Task A8: Read path — detail photos + activity-delete orphan tagging

**Files:** `internal/activity/unified_handler.go` (attach `photos` to detail DTO), `internal/activity/handler.go` (`activityDTO` gains `Photos []photoDTO`), delete path in `handler.go` tags photo objects before cascade.

- [ ] **TDD:** Add tests: `GET /activities/{id}` includes `photos` ordered `(position,id)` filtered `deleted_at IS NULL`; empty array (not null) when none; hard-deleting an activity tags its photo objects orphaned before the FK cascade.
- [ ] Run — FAIL.
- [ ] Implement: `activityDTO.Photos []photoDTO \`json:"photos"\`` (always present, `[]` when empty — but omit entirely when photo storage unconfigured per SOW "reads simply omit photos"; simplest: populate from repo when `photoRepo != nil`, else leave nil→omit with `omitempty`). Hydrate in `buildDetailDTO`/`get`. In the activity hard-delete handler, before deleting, list the activity's photo keys and best-effort `TagOrphaned` both variants for each (log failures), then proceed with existing delete.
- [ ] Run — PASS. Gate.
- [ ] Commit: `feat: include photos on activity detail read and tag on delete`

## Task A9: Timeline card hydration — cover photo + count (batched)

**Files:** `internal/timeline/model.go` (`PostContent` gains `Photo *PostPhoto`, `PhotoCount int`), `internal/timeline/handler.go` (`contentDTO` gains `photo`/`photo_count`, `toContentDTO` maps them), `internal/server/timeline_hydrator.go` (`hydrateSessions` batch-loads covers), plus a way to pass the photo repo into the hydrator.

- [ ] **TDD:** In `internal/server/timeline_hydrator_test.go`, assert an N-post page of session posts issues **exactly one** photo cover query (inject a counting `PhotoRepository`), and that the cover thumb_url + count land on the right refs. In `internal/timeline/handler_test.go`, assert `contentDTO` serializes `photo` (nullable object `{thumb_url,width,height}`) and `photo_count`.
- [ ] Run — FAIL.
- [ ] Implement: `PostContent` gains `Photo *PostPhoto` (`ThumbURL string; Width, Height int`) and `PhotoCount int`. `contentDTO` gains `Photo *photoCoverDTO \`json:"photo"\`` and `PhotoCount int \`json:"photo_count"\``. `toContentDTO` maps them. `timelineHydrator` gains a `photoRepo PhotoRepository` field + `newTimelineHydrator` param; in `hydrateSessions`, after collecting session activity ids, call `CoverPhotosByActivityIDs(ids)` once and presign the cover thumb (needs the photo store — pass it too, or have the repo return keys and presign here). Set `Photo`/`PhotoCount` on each session ref's `PostContent`. Mobile/other source types unchanged.
- [ ] Run — PASS. Gate.
- [ ] Commit: `feat: hydrate timeline cover photo and count`

## Task A10: Server wiring + graceful degradation

**Files:** `internal/server/server.go`

- [ ] Construct the photo store when `cfg.PhotoBucketName != ""` (mirror avatar block 147–154), passing region + `time.Duration(cfg.Photos.PresignWindowHours)*time.Hour`. Build the photo repository from the same `database`. Call `activityHandler.SetPhotoStore(photoStore, photoRepo, cfg.Photos)` (nil store when unconfigured → endpoints 503, reads omit). Pass `photoRepo` (+ store for presign) into `newTimelineHydrator`. When unconfigured, the hydrator must skip presigning gracefully (cover omitted).
- [ ] Verify: `go build ./...`, `go test ./internal/server/... ./internal/activity/... ./internal/timeline/...` — PASS.
- [ ] Run the **full gate** (`go test ./...`, lint, vet, mod tidy). Fix any drift.
- [ ] Commit: `feat: wire activity photo store and repository`

---

# PART B — prog-strength-infra

Work from `/workspace/prog-strength-infra` on branch `feat/activity-photos`. Gate: `terraform fmt -recursive`, `terraform init -backend=false`, `terraform validate`, `tflint --init && tflint --recursive`.

## Task B1: activity_photo_storage module + root wiring

**Files:** Create `modules/activity_photo_storage/{main,variables,outputs,versions}.tf`. Edit root `main.tf`, `outputs.tf`, `variables.tf`, `environments/prod.tfvars`, `compose/api/config.env`, `deploy/api.sh`.

- [ ] Create the module by copying `modules/avatar_storage` structure. Bucket `Purpose = "activity-photos"`; SSE-AES256; full public-access block; **no versioning**; lifecycle rule `expire-orphaned-photos` filtering tag `photo-status=orphaned` expiring after `var.orphan_expiration_days`; IAM policy (`s3:ListBucket` on bucket; `s3:GetObject/PutObject/PutObjectTagging/DeleteObject` on objects) named `${var.name_prefix}-activity-photos`, attached to `var.instance_role_name`. Carry the "tag MUST match the API's TagOrphaned call" warning comment. Variables: `name_prefix`, `instance_role_name`, `bucket_name`, `orphan_expiration_days` (default 7). Output `bucket_name`. `versions.tf` matching avatar.
- [ ] Root `variables.tf`: add `activity_photo_storage` object var (`bucket_name`, `orphan_expiration_days`) with defaults (`prog-strength-activity-photos`, 7). Root `main.tf`: add `module "activity_photo_storage"` block mirroring `avatar_storage`. Root `outputs.tf`: add `photo_bucket_name = module.activity_photo_storage.bucket_name` (comment: set as `PHOTO_BUCKET_NAME`). `environments/prod.tfvars`: add the `activity_photo_storage` block. `compose/api/config.env`: add `PHOTO_BUCKET_NAME=prog-strength-activity-photos`. `deploy/api.sh`: add `PHOTO_BUCKET_NAME` to `REQUIRED_ENV_KEYS`.
- [ ] Gate: fmt, validate, tflint all clean.
- [ ] Commit: `feat(activity-photos): add activity photo storage bucket, IAM, and env wiring`

---

# PART C — prog-strength-web

Work from `/workspace/prog-strength-web` on branch `feat/activity-photos`. Gate: `npm run lint`, `npm run format:check`, `npm run typecheck`, `npm run test`, `npm run build`. Conform to `design-system.md`: `--accent` (#9aa6d6) only for interactive affordances/focus rings, 14px panel radius on image containers + lightbox shell, Manrope, accent focus ring. No new tokens.

## Task C1: API client — photo types + endpoint functions

**Files:** `lib/api.ts` (+ colocated test if the file has one)

- [ ] Add `ActivityPhoto` type (`{id, url, thumb_url, width, height, caption, position}`) and a timeline cover type (`{thumb_url, width, height}`). Extend the activity/workout detail type with `photos?: ActivityPhoto[]` and the `TimelinePost.content` (or `TimelinePost`) with `photo?: {thumb_url,width,height} | null` and `photo_count?: number` — match the API DTO exactly (photo/photo_count live on the card content per A9; verify the actual JSON shape the timeline handler emits).
- [ ] Add functions mirroring `uploadAvatar`: `uploadActivityPhoto(token, activityId, file, caption?)` (multipart field `photo`), `updatePhotoCaption(token, activityId, photoId, caption)`, `reorderPhotos(token, activityId, photoIds)`, `deleteActivityPhoto(token, activityId, photoId)`. Use the existing `unwrap`/error conventions.
- [ ] Gate (typecheck/lint at minimum). Commit: `feat(activities): add activity photo api client`

## Task C2: PhotoStrip + Lightbox on activity detail

**Files:** Create `components/activity-detail/PhotoStrip.tsx`, `components/activity-detail/PhotoLightbox.tsx` (+ tests). Mount in `app/(app)/workouts/[id]/page.tsx` beneath the session header.

- [ ] PhotoStrip: horizontal strip of 14px-radius thumbnails, each in an aspect-ratio box from `width/height` (reserve space to prevent reflow). Owner-only (`useProfile` id === activity `user_id`): **Add photo** affordance (file input, client-side type/size pre-check as UX only) and an edit mode with per-photo delete, caption edit, move-left/move-right (reorder issues one `reorderPhotos` with the full list). Lightbox: modal opening the `full` variant with prev/next, keyboard arrows + Escape, caption beneath, focus trapped (reuse the modal precedent's Escape + scroll-lock pattern). Alt text = caption, falling back to a generated description from the activity.
- [ ] Tests (Vitest + RTL): renders thumbnails with aspect boxes; owner sees Add/edit affordances, non-owner does not; lightbox opens/navigates/closes on keyboard; reorder calls the client once with the full id list.
- [ ] Gate. Commit: `feat(activities): add photo strip and lightbox to detail page`

## Task C3: Timeline cover thumbnail + `+N` badge

**Files:** `app/(app)/timeline/_components/TimelinePostCard.tsx` (+ test)

- [ ] When `content.photo` present, render a single cover thumbnail in an aspect-ratio box (no reflow); show a `+N` badge (`photo_count - 1`) when `photo_count > 1`, styled with `--accent-soft`/`--accent-line`/`--accent`. Tapping the card still routes to the detail page (no carousel). Test: cover renders, badge appears only when count>1 and shows the right N, absent when no photo.
- [ ] Gate. Commit: `feat(timeline): add cover photo and +N badge to cards`

---

# PART D — prog-strength-mobile

Work from `/workspace/prog-strength-mobile` on branch `feat/activity-photos`. Gate: `npm run typecheck`, `npm run lint`, `npm run format:check`, `npx expo-doctor`. Detail-page display + capture/upload only; **no timeline**.

## Task D1: deps, app.json permissions, api client

**Files:** `package.json` (add `expo-image-manipulator` via `npx expo install`), `app.json`, `lib/api.ts`

- [ ] `npx expo install expo-image-manipulator` (SDK-55 compatible). Update the `expo-image-picker` plugin `photosPermission`/`cameraPermission` strings in `app.json` to include workout/activity photos (keep existing meal/profile mentions). If native change requires it per repo convention (AGENTS.md), bump `runtimeVersion`/fingerprint.
- [ ] Add to `lib/api.ts` (RN FormData `{uri,name,type}` precedent): `ActivityPhoto` type, `uploadActivityPhoto(token, activityId, image: PickedImage, caption?)`, `deleteActivityPhoto`, `updatePhotoCaption`, `reorderPhotos`. Add `photos?: ActivityPhoto[]` to the workout/activity detail types (keep in sync with web).
- [ ] Gate (typecheck/lint/expo-doctor). Commit: `feat(activities): add photo deps, permissions, and api client`

## Task D2: Photo strip + add action + full-screen viewer

**Files:** Create `components/activities/photo-strip.tsx`, `components/activities/photo-add-button.tsx`, `components/activities/photo-viewer-modal.tsx`. Mount in `app/(tabs)/activities/workout/[id].tsx` and `run/[id].tsx`.

- [ ] Add-photo action: camera or library via `expo-image-picker`; downscale with `expo-image-manipulator` before upload (bandwidth only — server is authoritative). Photo strip: horizontally-scrolling thumbnails (aspect boxes), tap opens a full-screen `Modal` viewer. Match web information design; dark theme tokens; ≥44pt touch targets; loading/empty/error states. Owner is the logged-in user (detail screen only loads own activities).
- [ ] Gate. Commit: `feat(activities): add photo capture, strip, and viewer on mobile`

---

# PART E — prog-strength-docs (status flip)

Work from `/workspace/prog-strength-docs` on branch `feat/activity-photos`.

## Task E1: Mark SOW shipped

- [ ] Edit `sows/activity-photos.md`: frontmatter `status: shipped`; body `**Status**: Shipped`; `**Last updated**: 2026-07-31`.
- [ ] Commit: `docs: mark activity-photos as shipped`

---

## Self-Review / spec coverage checklist

- Many ordered captioned photos on base table (A1, A5) · Hive S3 layout w/ variant partition on UTC start (A2) · server-authoritative compression both variants (A3) · EXIF+GPS stripped, orientation applied, 8 values (A3) · stable windowed presign URLs (A4) · batched feed hydration, one query/page (A9) · strip+lightbox web (C2) · cover+badge web (C3) · capture/upload+display mobile (D2) · dedicated bucket+orphan lifecycle+IAM (B1) · graceful 503 / either-order deploy (A6/A7/A10/B1) · config knobs committed (A6) · all test items in SOW Testing section mapped to A2/A3/A4/A5/A7/A8/A9.
