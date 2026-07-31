---
status: shipped
repos:
  - prog-strength-api
  - prog-strength-infra
  - prog-strength-web
  - prog-strength-mobile
  - prog-strength-docs
---

# Activity Photos

**Status**: Shipped · **Last updated**: 2026-07-31

## Introduction

Prog Strength records what a training session *measured* — distance, pace, heart rate, elevation, sets and reps — and nothing about what it *looked like*. A user finishes a hike at a ridgeline, takes a selfie at the summit, and has nowhere to put it. The photo goes to their camera roll, the hike goes to Prog Strength, and the two never meet again. Six months later the activity detail page is a set of charts with no reminder of where they actually were.

That gap matters more now than it did a release ago. The timeline defaults to `friends` visibility, follows and profile discovery have shipped, and the feed is the app's social surface — but every card in it is text and numbers. A feed of pace splits is a training log; a feed of pace splits *with the photo from the top of the climb* is something people actually open. Photos are the cheapest available upgrade to the social surface because the social plumbing already exists: `timeline_post` points at a source record and hydrates card content at read time, so a photo attached to an activity shows up in the feed without a new post type, a new privacy model, or a backfill.

The unified activity model makes this a single feature rather than five. Every training session — a lift, a run, a hike, a kickboxing class, and every type not yet registered — is one row in the `activities` base table. A photo attaches to `activity_id`, so this ships once for the base and every present and future type inherits it with no descriptor changes.

After this ships: a user finishes a hike on their phone, taps **Add photo**, takes a selfie or picks from their library, and the image appears on the hike's detail page and on the hike's timeline card for the people who follow them.

## Proposed Solution

A new `activity_photo` table hangs many ordered, optionally-captioned photos off any activity. Because photos attach to the **base** table rather than to a type descriptor, this is one implementation that covers every activity type — the code lands in `internal/activity` next to the unified handler, not in a per-type module.

Bytes flow through the API, not around it. The client downscales before upload as a bandwidth courtesy, but **the server is the authority on output**: it decodes the upload, applies the EXIF orientation, resizes to a bounded long edge, and re-encodes two JPEG variants — a `full` for the detail page and a `thumb` for the feed. Re-encoding is what makes the privacy story work, since it drops every EXIF tag including the GPS coordinates a phone camera embeds in a selfie. The whole pipeline is pure Go with no cgo, which keeps the Alpine/musl container build out of the trouble `sqlite-vec` caused.

Objects land in a dedicated private S3 bucket under a Hive-partitioned key that mirrors the TCX archiver's layout — `user_id=…/activity_type=…/year=…/month=…/day=…/activity_id=…/variant=…/{photo_id}.jpg` — partitioned on the activity's UTC start time. The layout is Athena-readable as written, co-locates every object for one activity under a single prefix, and makes `variant` addressable by lifecycle rules and IAM independently of the full-size image.

Retrieval is presigned GETs, as with avatars, with one change that is the whole point of the "low latency at volume" requirement: the signer is called with an **explicit rounded signing time**, so every request inside a six-hour window mints a byte-identical URL. A stable URL is a usable HTTP cache key, so scrolling the timeline twice does not re-download every thumbnail. Presigning remains a local HMAC — hydrating a feed page costs microseconds and zero S3 round trips.

On the read surfaces, the activity detail page gains a photo strip with a lightbox, and the timeline card gains a single cover thumbnail with a `+N` badge that taps through to the detail page's full gallery. Mobile gets capture-and-upload plus detail-page display; it has no timeline tab yet, so it gets no feed changes.

## Goals and Non-Goals

### Goals

- **Many ordered photos per activity**, with optional captions, attached to the `activities` base table so every registered type — present and future — supports them with no descriptor changes.
- **A Hive-partitioned S3 layout** consistent with `buildTCXKey`, partitioned on the activity's UTC start time, with `variant` as a real partition level.
- **Server-authoritative compression**: bounded dimensions and quality for both variants regardless of what the client uploads.
- **EXIF stripped on write**, including GPS, with orientation applied before re-encode so photos are never rendered sideways.
- **Stable presigned URLs** within a configurable window, so repeat views are browser-cache hits rather than S3 fetches.
- **Batched feed hydration**: one photo query per timeline page, never one per card.
- **Photo strip and lightbox** on the web activity detail page; **cover thumbnail with `+N` badge** on the web timeline card.
- **Capture and upload on mobile** (camera or library) with detail-page display.
- **A dedicated private bucket** with orphan-tag lifecycle reaping and an IAM policy scoped to it, provisioned in `prog-strength-infra`.
- **Graceful degradation**: with no bucket configured the photo endpoints return 503 and the rest of the API is unaffected, so `api` and `infra` can ship in either order.

### Non-Goals

- **Moderation, reporting, or NSFW detection.** Pre-launch, single-user; a follow-up when the user base warrants it.
- **Video.** Images only.
- **Photos in chat, MCP, or agent tool surfaces.** The agent can neither see nor attach activity photos. `prog-strength-agent` and `prog-strength-mcp` need no changes, which is why they are absent from `repos:`.
- **Server-side HEIC decode.** No pure-Go decoder exists and cgo is off the table; clients transcode to JPEG (`expo-image-picker` already does by default).
- **CloudFront or any CDN.** Documented below as the scale-out path, deliberately not built.
- **Drag-and-drop reordering on web.** Move-left / move-right buttons in v1.
- **An inline carousel in the timeline feed.** One cover image per card; the gallery lives on the detail page.
- **Photos on non-activity entities** — meals, bodyweight readings, steps.
- **A mobile timeline.** Mobile has no timeline tab; adding one is out of scope.
- **Per-user storage quotas.** See Open Questions.

## Implementation Details

### Data Model

Migration `046_activity_photos.sql` is purely additive — one `CREATE TABLE` and two indexes, no table rebuild.

| Column | Type | Description |
| --- | --- | --- |
| `id` | TEXT | Primary key. |
| `activity_id` | TEXT NOT NULL | FK → `activities(id)` `ON DELETE CASCADE`. |
| `user_id` | TEXT NOT NULL | Denormalized owner. Ownership checks and per-user aggregates avoid a join, and it is the first partition of the S3 key. |
| `s3_key` | TEXT NOT NULL | Object key of the `full` variant. |
| `thumb_s3_key` | TEXT NOT NULL | Object key of the `thumb` variant. |
| `content_type` | TEXT NOT NULL | Always `image/jpeg` after processing. Stored anyway so a future output format is a data change rather than a schema change. |
| `byte_size` | INTEGER NOT NULL | Size of the stored `full` object, post-compression. |
| `width` | INTEGER NOT NULL | Stored `full` width in pixels. |
| `height` | INTEGER NOT NULL | Stored `full` height in pixels. |
| `caption` | TEXT | Nullable; validated `<= photos.caption_max_chars` in the domain. |
| `position` | INTEGER NOT NULL | 0-based sort order within the activity. |
| `created_at` | DATETIME NOT NULL | |
| `updated_at` | DATETIME NOT NULL | |
| `deleted_at` | DATETIME | Soft delete, mirroring `activities`. |

Indexes:

- `idx_activity_photo_activity(activity_id, position, id)` — serves the per-activity ordered read and the cover-photo window function.
- `idx_activity_photo_user(user_id, created_at DESC)` — serves per-user aggregates (audit, a future quota).

`width`/`height` are not decoration: the client uses them to reserve an aspect-ratio box before the image loads, which is what keeps the timeline from shifting layout mid-scroll.

Three deliberate schema choices:

- **`position` is not `UNIQUE`.** A `UNIQUE(activity_id, position)` constraint forces a reorder to dodge transient collisions (renumber to negatives, then back) for no real benefit. Ordering ties break on `id`, and the reorder endpoint rewrites the whole set in one transaction anyway.
- **No `CHECK` on `content_type`.** This follows migration 042's rationale exactly: SQLite cannot widen a `CHECK` without rebuilding the table, and Go already owns the allowlist. The `CHECK`-free convention is now the house style for enum-ish columns.
- **Soft delete on both sides.** Activities are soft-deleted, so the FK cascade only fires on a hard delete. Soft-deleting an activity leaves its photo rows untouched — they become unreachable because the read path filters on the parent — and restoring the activity brings the photos back for free. This is why photo reads must filter on the parent's `deleted_at`, not only their own.

Photos deliberately live on the **base** table rather than in a type detail table or a `Descriptor`. A photo is not type-specific divergence; it is a universal property of "a training session happened." Putting it in the base means `internal/activity/photo_*.go` sits beside the unified handler and every type registered in the registry — including types that do not exist yet — gets photos without touching its descriptor. This is the same reasoning that put `notes` and `total_calories` in the base row.

### S3 Schema

Objects land in a dedicated bucket under:

```
user_id={user_id}/activity_type={type}/year={yyyy}/month={mm}/day={dd}/activity_id={activity_id}/variant={full|thumb}/{photo_id}.jpg
```

A new `internal/activity/photo_key.go` builds it, and because it sits in the same package as `tcx_key.go` it **reuses `idPartPattern`, `ErrInvalidKeyPart`, and `ErrInvalidActivityType` directly** rather than re-deriving the validation. That matters: those rejections (`/`, `=`, `.`, whitespace) are what keep a user id from forging a fake partition level or a traversal-shaped key, and two independently-maintained copies of that rule is exactly how one of them drifts.

The design decisions embedded in that layout:

- **The date partition is the activity's `start_time` converted to UTC**, not the upload time and not the user's local date. This is the same choice `buildTCXKey` documents and for the same reasons — S3 keys are global, the user's timezone preference can change, and a display-zone change should never reshuffle the bucket. It also means a photo uploaded three days after a hike files under the hike's date, so every object belonging to one activity shares one prefix.
- **`activity_id=` is a partition level, not a bare path segment.** Keeping every segment in `key=value` form means the whole path is Athena- and Glue-readable with no custom SerDe, and it makes "delete everything for this activity" a single prefix operation.
- **`variant=` is a partition level rather than a filename suffix** (`{photo_id}_thumb.jpg`). As a partition it can be targeted independently by lifecycle rules, IAM statements, and S3 Inventory — e.g. a future rule transitioning only `variant=full` to a colder class, without touching the thumbnails the feed depends on.
- **The extension is always `.jpg`.** The server re-encodes both variants, so the key builder is total: no content-type→extension map to keep in sync with the store, which is a small ongoing maintenance cost the avatar path pays (`contentTypeToExt`) and this one does not.

The bucket is separate from the avatars bucket rather than a shared prefix within it. The two have different object-size profiles, different lifecycle rules, and different growth curves, and a separate bucket keeps the IAM blast radius of a photo-handling bug away from profile images.

### Image Pipeline

Everything below is pure Go. No cgo — that is a hard constraint, not a preference, given the `u_int8_t` build failure `sqlite-vec` produced on Alpine/musl.

1. **Decode.** `image/jpeg` and `image/png` from the standard library; `golang.org/x/image/webp` for WebP (decode-only, which is all that is needed since output is always JPEG).
2. **Apply EXIF orientation.** Read the `Orientation` tag and apply the corresponding rotation/flip *before* resizing. This is the single most likely user-visible bug in the feature: iPhones store portrait photos as landscape pixels plus a rotation flag, so a pipeline that ignores the tag renders every phone selfie sideways. All eight orientation values are handled.
3. **Resize.** `golang.org/x/image/draw` with the `CatmullRom` kernel, preserving aspect ratio, clamping the long edge. Images already under the limit are not upscaled.
4. **Encode.** Both variants as JPEG via `image/jpeg`.

Two variants are produced:

| Variant | Long edge | Quality | Consumer |
| --- | --- | --- | --- |
| `full` | `photos.full_max_edge_px` (2048) | `photos.full_jpeg_quality` (82) | Detail-page strip and lightbox. |
| `thumb` | `photos.thumb_max_edge_px` (480) | `photos.thumb_jpeg_quality` (78) | Timeline cover image. |

The re-encode is also the privacy mechanism. Decoding to a pixel buffer and re-encoding drops **all** EXIF metadata, including the GPS coordinates a phone embeds by default. Without this, a selfie taken at the end of a neighborhood run would publish the user's home location into a `friends`-visibility feed. Stripping is a side effect of the pipeline rather than a separate step that can be forgotten, which is the right place for it — but it is asserted explicitly in tests so a future "fast path that skips re-encode when the image is already small" cannot silently reintroduce the leak.

Client-side downscaling (browser canvas on web, `expo-image-manipulator` on mobile) is a **bandwidth optimization only**. The server clamps regardless, so a client that skips it, gets it wrong, or is bypassed entirely produces the same stored result.

### Write Path

All routes sit under the existing user-JWT auth and mount beside the unified activity handler.

- **`POST /activities/{id}/photos`** — `multipart/form-data`, field `photo`, optional `caption`.
  1. Resolve the activity for the authenticated user. A miss returns **404**, including when the activity exists but belongs to someone else — the endpoint must not confirm the existence of another user's activity.
  2. `http.MaxBytesReader` bounds the body at `photos.max_upload_bytes`; overflow returns **413** `file_too_large`.
  3. `http.DetectContentType` sniffs the bytes — the client's declared type is not trusted — and a type outside `{image/jpeg, image/png, image/webp}` returns **415** `unsupported_media_type`.
  4. Count live photos on the activity; at `photos.max_per_activity` return **409** `photo_limit_reached`.
  5. Run the image pipeline, producing both variants.
  6. Build both S3 keys from `(user_id, activity_type, activity.start_time, activity_id, photo_id, variant)`.
  7. `Put` both objects. **If the second put fails, the first is best-effort tagged orphaned and no row is written** — a failed upload leaves no dangling database reference, and the lifecycle rule reaps the stranded object.
  8. Insert the row with `position = COALESCE(MAX(position), -1) + 1` scoped to the activity.
  9. **201** with the hydrated photo DTO, both URLs presigned.
- **`PATCH /activities/{id}/photos/{photo_id}`** — caption only. Body `{"caption": string|null}`. Over-length returns **400**.
- **`PUT /activities/{id}/photos/order`** — body `{"photo_ids": [...]}`, the **complete** ordered list. The request is rejected with **400** unless the submitted set is exactly the activity's live photo ids — no subsets, no extras, no duplicates. Positions are rewritten to the array index in one transaction. Full-list semantics make the operation idempotent and eliminate partial-reorder states; a "move this one to index N" API would need conflict handling that buys nothing here.
- **`DELETE /activities/{id}/photos/{photo_id}`** — sets `deleted_at` and best-effort tags **both** objects `photo-status=orphaned`. Tagging failures are logged, not surfaced; the row is already gone from the user's view.
- **Activity hard delete** — the FK cascade removes the rows. The activity delete path tags the photo objects orphaned before the cascade, so the objects are reaped rather than stranded.
- **Storage unconfigured** — when no photo bucket is set, the write endpoints return **503** `photo_storage_unavailable` and reads simply omit photos. This mirrors the avatar handler and is what lets `prog-strength-api` and `prog-strength-infra` ship in either order.

The `PhotoStore` seam mirrors `AvatarStore` exactly — `Put`, `PresignGet`, `TagOrphaned` — with an `S3PhotoStore` in production and a `FakePhotoStore` keeping handler tests hermetic.

The orphan tag key/value (`photo-status` / `orphaned`) **must stay identical to the lifecycle rule's tag filter** in the Terraform module. Both sides carry the same warning comment the avatar path carries, because the failure mode is silent: a mismatch means orphaned objects accumulate forever, and a lifecycle rule written without the tag filter would expire live photos.

### Read Path

`GET /activities/{id}` grows a `photos` array — `{id, url, thumb_url, width, height, caption, position}` — ordered by `(position, id)` and filtered to `deleted_at IS NULL`.

The timeline card DTO grows two fields: `photo` (`{thumb_url, width, height}` or `null`) and `photo_count`. Hydration adds **one batched query per feed page** over the page's activity ids, using a window function for the cover photo alongside a count:

```
ROW_NUMBER() OVER (PARTITION BY activity_id ORDER BY position, id) AS rn
```

taking `rn = 1` per activity. This follows the batched-hydration pattern already established for workout lists; a per-card query would be an N+1 on the app's most-scrolled surface and is asserted against in tests.

#### Presign windowing

The latency requirement is not about how fast S3 serves a byte — it is about not asking S3 at all on the second view. Presigning with the standard presign client bakes the current timestamp into the signature, so every request produces a *different* URL for the same object, and a different URL is a different HTTP cache key. Scrolling the feed twice therefore re-downloads every thumbnail.

The fix is a `windowedPresigner` built on `v4.Signer.PresignHTTP`, which accepts an explicit `signingTime`:

```
window       = photos.presign_window_hours
signingTime  = now.Truncate(window)
X-Amz-Expires = 2 * window
```

Every request inside a window signs at the same instant and therefore produces a **byte-identical URL**, which the browser cache, and any intermediary, treat as one resource. The `2 * window` expiry guarantees a URL minted in the last second of a window remains valid for a full window afterward, so a client never holds a URL that expires immediately. Objects are written with `Cache-Control: private, max-age=31536000, immutable`, which is safe because keys are photo-id-addressed and never overwritten.

The security posture is unchanged in kind from avatars — a leaked URL is usable until it expires — and slightly longer in duration (up to 12 hours versus the avatar path's 1 hour). That is an accepted trade for cacheability on a surface that renders dozens of images per scroll; the objects are not secrets, the bucket remains fully private, and nothing about a photo URL grants access to any other object.

Presigning is local HMAC computation with no network call, so hydrating a 20-card feed page costs microseconds and zero S3 requests regardless of windowing.

### API Surface

| Method | Path | Purpose |
| --- | --- | --- |
| `POST` | `/activities/{id}/photos` | Upload a photo (multipart). → 201 |
| `PATCH` | `/activities/{id}/photos/{photo_id}` | Edit caption. |
| `PUT` | `/activities/{id}/photos/order` | Rewrite the full photo order. |
| `DELETE` | `/activities/{id}/photos/{photo_id}` | Soft-delete a photo. |
| `GET` | `/activities/{id}` | *Extended* — now includes `photos`. |
| `GET` | `/timeline` | *Extended* — cards now include `photo` and `photo_count`. |

All require a user JWT. Error codes: `file_too_large` (413), `unsupported_media_type` (415), `photo_limit_reached` (409), `photo_storage_unavailable` (503).

### Configuration

Per the project's config convention, only the bucket name is env-sourced — Terraform owns it — and every tuning knob is a committed value in `config.toml`.

`[storage]` gains one line alongside `avatar_bucket_name` and `tcx_bucket_name`:

```toml
photo_bucket_name = "${PHOTO_BUCKET_NAME}"
```

And a new section:

```toml
[photos]
max_per_activity     = 10
max_upload_bytes     = 12582912   # 12 MiB
full_max_edge_px     = 2048
full_jpeg_quality    = 82
thumb_max_edge_px    = 480
thumb_jpeg_quality   = 78
presign_window_hours = 6
caption_max_chars    = 200
```

None of these are secrets, so none of them become environment variables or GitHub secrets. Tuning compression quality or the photo cap is a reviewable commit, not an untracked console change.

### Infrastructure

A new `modules/activity_photo_storage` module, structured on `modules/avatar_storage`:

- Private bucket with SSE-AES256 and a full public-access block. Photos are never served publicly; clients receive presigned GETs.
- **No versioning** — each upload writes a fresh photo-id-keyed object and is never overwritten, so "latest wins" needs no help.
- Lifecycle rule expiring **only** objects tagged `photo-status=orphaned`, after `orphan_expiration_days`. Live photos carry no tag and are therefore never matched. A naive age-based expiration is explicitly wrong here: it would delete photos whose rows still reference them.
- IAM policy scoped to this bucket — `s3:ListBucket` on the bucket, and `s3:GetObject` / `s3:PutObject` / `s3:PutObjectTagging` / `s3:DeleteObject` on its objects — attached to the EC2 instance role. Credentials come from the instance profile; no access keys exist anywhere.
- Output `bucket_name`, surfaced on the API host as `PHOTO_BUCKET_NAME`.

### Frontend

Both surfaces conform to [`design-system.md`](../design-system.md) and introduce no new tokens: the 14px panel radius on image containers and the lightbox shell, `--accent` (periwinkle `#9aa6d6`) only for interactive affordances and focus rings, Manrope throughout, and the existing accent ring for focus rather than the browser default.

**Web** (`prog-strength-web`):

- `components/activity-detail/PhotoStrip.tsx` — a horizontal strip of 14px-radius thumbnails beneath the session header, each in an aspect-ratio box derived from `width`/`height`. For the owner, an **Add photo** affordance and an edit mode exposing per-photo delete, caption editing, and move-left / move-right buttons. Reordering issues one `PUT …/order` with the full list.
- A minimal lightbox: tap opens the `full` variant in a modal with previous/next navigation, keyboard arrows and Escape, the caption beneath, and focus trapped in the dialog.
- Timeline card gains a single cover thumbnail with a `+N` badge when `photo_count > 1`, inside an aspect-ratio box so the feed does not reflow as images load. Tapping the card goes to the activity detail page and its full strip — the feed itself gets no carousel.

**Mobile** (`prog-strength-mobile`):

- An **Add photo** action on the activity detail screens offering camera or library via `expo-image-picker` (already a dependency), with `expo-image-manipulator` added for pre-upload downscaling.
- A horizontally-scrolling photo strip with a full-screen viewer, matching the web's information design.
- Camera and photo-library usage descriptions in `app.json` for the iOS permission prompts.
- **No timeline changes** — mobile has no timeline tab.

### Testing

- `photo_key_test.go`, table-driven in the shape of `tcx_key_test.go`: rejects `/`, `=`, `.`, whitespace, and empty id parts; rejects an unregistered activity type; asserts the exact partition layout; and asserts that a `start_time` in a non-UTC zone partitions on its **UTC** date, not its local one.
- Image-pipeline golden tests over JPEG, PNG, and WebP fixtures across **all eight** EXIF orientation values: assert output dimensions and orientation, JPEG magic bytes on both variants, that an under-size image is not upscaled, and that **no EXIF or GPS data survives** in either output.
- Handler tests against `FakePhotoStore` — hermetic, no AWS: happy-path upload, oversize → 413, disallowed type → 415, over-limit → 409, another user's activity → 404, second-put failure → no row written and first object tagged.
- Presign-window test: two presigns one second apart inside a window are byte-identical; two straddling a boundary differ; the expiry is twice the window.
- Reorder tests: full-list validation rejects subsets, extras, and duplicates; a valid reorder is idempotent; positions match array indices afterward.
- Lifecycle tests: soft-deleting an activity leaves photo rows intact and hidden; restoring the activity restores them; a hard delete cascades.
- A timeline test asserting an N-post page issues exactly **one** photo query.

### Backfill or Migration

**No backfill.** `activity_photo` is a new table with no existing data to populate, and migration 046 is additive — a `CREATE TABLE` plus two indexes, with no rebuild of `activities` or any child.

**Recoverability.** The migration is a single additive transaction; a failure leaves the schema untouched. There is no derived data to truncate and no id remapping to unwind.

**Ordering across repos.** The Terraform bucket must exist and `PHOTO_BUCKET_NAME` must be set on the API host before uploads work, but because an unset bucket degrades to 503 on the photo endpoints and to omitted photos on reads, `prog-strength-api` and `prog-strength-infra` can merge and deploy in either order without an outage.

**Scale boundary.** The design holds until either photo volume makes per-request presigning-plus-S3-origin latency the bottleneck — at which point the read path moves behind CloudFront with Origin Access Control and signed URLs, replacing the presigner without touching the schema or the key layout — or the single EC2 host's CPU becomes a constraint during image processing, at which point the pipeline moves to a presigned direct-to-S3 upload with an S3-triggered Lambda generating thumbnails. Both are strictly additive changes to this design; neither requires re-keying stored objects.

## Open Questions

1. **Storage class.** STANDARD for everything, versus a lifecycle transition of `variant=full` to STANDARD-IA or Intelligent-Tiering after ~90 days. *Lean: STANDARD only.* Intelligent-Tiering charges per-object monitoring, which is poor value at thumbnail sizes, and the `variant=` partition means a transition rule can be added later without touching stored keys.
2. **Per-user storage quota.** No cap, versus a total-bytes ceiling per user. *Lean: no cap pre-launch.* `max_per_activity` already bounds the pathological case, and `idx_activity_photo_user` makes a quota query cheap whenever it is wanted.
3. **PNG passthrough.** Always re-encode to JPEG, versus preserving PNG when the source is PNG (screenshots of pace charts compress badly and look worse as JPEG). *Lean: always JPEG.* A total key builder and a single output format are worth more than screenshot fidelity in a photo feature; if screenshots become a real use case it is a per-photo `content_type` change, which the column already anticipates.
4. **Alt text.** Reuse `caption` as the `alt` attribute, versus a separate `alt_text` column. *Lean: reuse `caption`, falling back to a generated description from the activity ("Photo from a 5 mi run on July 12") when the caption is empty.* A second free-text field that users will not fill in is worse for accessibility than a sensible generated default.
