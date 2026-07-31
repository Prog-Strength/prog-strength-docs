---
status: draft
repos:
  - prog-strength-api
  - prog-strength-infra
  - prog-strength-web
  - prog-strength-docs
---

# Activity Videos

**Status**: Draft · **Last updated**: 2026-07-31

## Introduction

[Activity Photos](activity-photos.md) closed half the gap: a session's detail page can now show what it *looked like*, not just what it measured. But a photo can't hold the part of a session that was moving — the last set of a PR attempt, the switchback that took forty minutes, the view swinging across a ridgeline. Those go to the camera roll and never meet the activity again.

Video is the other half of the same keepsake story, and it is worth building for exactly the reason photos were: the session record is the only place these belong. A video in the camera roll is a video you'll never find again. A video on the hike's detail page is attached to the date, the route, the elevation profile, and the heart rate — it's the thing that makes a training log worth keeping for years.

This SOW is scoped by one product fact: **Prog Strength is a single-user platform for the foreseeable future, and the owner's stated priority is preserving the original quality of what they capture.** That is not the usual video-product constraint. It removes the pressure to transcode, to ladder bitrates, or to optimize delivery, and it makes fidelity the thing to protect rather than the thing to trade away. The design below leans on that hard, and Open Questions flags exactly where the design would have to change if the platform ever stopped being single-user.

After this ships: on any activity detail page, the user taps **Add video**, picks a clip, and it uploads with a progress bar and plays back inline on that page at the resolution it was recorded.

## Proposed Solution

A new `activity_video` table hangs many ordered, optionally-captioned videos off any activity, mirroring `activity_photo`. As with photos, videos attach to the **base** `activities` row, so every registered activity type — present and future — supports them with no descriptor changes.

Three decisions distinguish this from photos, and each follows from a constraint the photo pipeline doesn't face.

**Bytes never touch the API host.** The photo path reads the whole upload into memory (`io.ReadAll` → `PhotoStore.Put(ctx, key, contentType, body []byte)`) and the store interface itself takes a byte slice. That is fine for a 6 MB JPEG and impossible for video: one minute of iPhone 4K is roughly 400 MB, and the host is a `t4g.small` with 2 GB of RAM shared with Litestream, Caddy, Grafana, and the agent. Rather than refactor the upload and store layers to stream, the client uploads **directly to S3 through a presigned URL** the API mints. The API handles two small JSON requests and never sees a video byte. This is strictly less work than a streaming refactor *and* strictly safer for the host.

**Nothing is transcoded — the stored object is the uploaded file, byte for byte.** This is the fidelity requirement taken literally, and it is also the only option available: extracting even a single frame from a video requires ffmpeg, which means a binary in the image or cgo, and the photo pipeline is deliberately pure Go to stay out of the Alpine/musl fight `sqlite-vec` caused. Transcoding on two burstable ARM vCPUs is a non-starter regardless. Storing the original is the highest-fidelity option *and* the cheapest to build — the rare case where the ambitious answer and the simple one agree.

**The poster frame is generated client-side.** The photo SOW insists the server is authoritative on output, and that was right for photos. For video, honoring it would buy a whole ffmpeg dependency to produce one JPEG. Instead the browser seeks a `<video>` element to an early frame, draws it to a canvas, and uploads that JPEG *through the existing photo pipeline*, which strips its EXIF and bounds its dimensions server-side. The server stays authoritative over the one artifact it can actually process.

On the read surface, a **shared `VideoStrip`** renders on **every** activity detail route — `/workouts/[id]`, `/running/[id]`, `/hiking/[id]`, and any route a future type adds. This is called out as a first-class requirement rather than an implementation detail because the photo rollout got it wrong: `PhotoStrip` shipped wired into `/workouts/[id]` alone, leaving runs and hikes with no way to attach a photo at all, and needed [`prog-strength-web#135`](https://github.com/Prog-Strength/prog-strength-web/pull/135) to correct. See [Route coverage](#route-coverage-the-lesson-from-photos).

Videos are **excluded from the timeline feed in v1**. This is a deliberate scope cut with a privacy dividend, explained under Non-Goals.

## Goals and Non-Goals

### Goals

- **Many ordered videos per activity**, optionally captioned, attached to the `activities` base table so every registered type — present and future — supports them with no descriptor changes.
- **Original-fidelity storage.** The stored object is byte-identical to the file the user selected. No transcode, no re-encode, no downscale, no bitrate ceiling.
- **Direct-to-S3 upload via presigned URL**, so a multi-hundred-megabyte file never transits or buffers on the API host.
- **A poster frame per video**, generated client-side and stored through the existing server-authoritative image pipeline.
- **Inline playback** on the activity detail page, with the poster shown before play.
- **`VideoStrip` on every activity detail route**, verified by a test per route — not one route with the others "to follow."
- **Upload progress**, because a 400 MB upload without a progress bar reads as a hang.
- **A dedicated private S3 bucket** with orphan reaping and bucket-scoped IAM, provisioned in `prog-strength-infra`, mirroring `activity_photo_storage`.
- **Graceful degradation**: with no bucket configured the video endpoints return 503 and the rest of the API is unaffected, so `api` and `infra` can ship in either order.

### Non-Goals

- **Transcoding, thumbnails-from-video, bitrate ladders, or HLS.** Requires ffmpeg; contradicts the pure-Go pipeline and the fidelity goal. See Open Questions for the HEVC playback consequence, which is the real cost of this choice.
- **Videos in the timeline feed.** Two reasons, and the second is the load-bearing one. (a) Feed bandwidth: a video cover in a scrolling feed is a different cost profile than a 200 KB thumbnail. (b) **Privacy**: storing originals means the QuickTime container keeps its GPS metadata, which the photo re-encode strips. Keeping videos off the social surface means those coordinates are only ever reachable by the owner, which is what makes "store the original" acceptable without ffmpeg-based metadata stripping. If videos ever enter the feed, this decision must be revisited *first*.
- **Mobile.** `prog-strength-mobile` is deliberately absent from `repos:`. Web only for v1.
- **Editing** — trimming, rotation, filters, captions burned in.
- **Videos on non-activity entities** — meals, bodyweight readings, steps.
- **Moderation or NSFW detection.** Single-user, pre-launch.
- **CloudFront or any CDN.** Documented as the scale-out path, deliberately not built.
- **Agent/MCP access.** The agent can neither see nor attach videos; `prog-strength-agent` and `prog-strength-mcp` need no changes, which is why they're absent from `repos:`.
- **Per-user storage quotas.** See Open Questions.

## Implementation Details

### Data Model

Migration `048_activity_videos.sql` — purely additive, one `CREATE TABLE` plus indexes, no table rebuild. Mirrors `047_activity_photos.sql` closely enough that it should be read side by side with it.

| Column | Type | Description |
| --- | --- | --- |
| `id` | TEXT | Primary key. |
| `activity_id` | TEXT NOT NULL | FK → `activities(id)` `ON DELETE CASCADE`. |
| `user_id` | TEXT NOT NULL | Denormalized owner: ownership checks avoid a join, and it's the first partition of the S3 key. |
| `s3_key` | TEXT NOT NULL | Object key of the stored original. |
| `poster_s3_key` | TEXT | Object key of the poster JPEG. Nullable — a video whose poster generation failed is still a valid keepsake and must not be lost. |
| `content_type` | TEXT NOT NULL | The uploaded container's type (`video/mp4`, `video/quicktime`). Not normalized, because nothing re-encodes. |
| `byte_size` | INTEGER NOT NULL | Size of the stored object, confirmed by a `HEAD` at commit rather than trusted from the client. |
| `duration_seconds` | REAL | Client-reported from the `<video>` element. Advisory — used for display, never for policy. |
| `width` / `height` | INTEGER | Client-reported natural dimensions. Advisory, same reasoning. |
| `status` | TEXT NOT NULL | `pending` or `ready`. See Write Path. |
| `caption` | TEXT | Nullable, bounded by `videos.caption_max_chars`. |
| `position` | INTEGER NOT NULL | 0-based sort order within the activity. |
| `created_at` / `updated_at` | DATETIME NOT NULL | |
| `deleted_at` | DATETIME | Soft delete, mirroring `activities` and `activity_photo`. |

Indexes mirror the photo table: `(activity_id, position, id)` for the strip read, `(user_id, created_at DESC)` for per-user aggregates. Add `(status, created_at)` to make the pending-row reaper cheap.

S3 layout reuses the photo key builder's Hive partitioning verbatim, with a `variant=video` / `variant=poster` level:

```
user_id=…/activity_type=…/year=…/month=…/day=…/activity_id=…/variant=video/{video_id}.{ext}
```

> **Carry the photo trap forward.** `buildPhotoKey` rejects any type `ActivityType.Valid()` doesn't list, and `Valid()` is a hand-maintained switch — a newly registered activity type gets a **500** on upload until it's added there. Whatever key builder this SOW uses must share that code path, and the recipe entry in [`adding-an-activity-type.md`](../adding-an-activity-type.md) must be extended to name videos too.

### Write Path

A three-step handshake. The API mints credentials and records state; the bytes go around it.

1. **`POST /activities/{id}/videos`** — the client sends filename, content type, declared byte size, duration, and dimensions. The API validates the content type against an allowlist, checks the per-activity count cap, mints a `video_id`, writes a row with `status='pending'`, and returns a **presigned upload URL**.

2. **The client uploads directly to S3.** Progress comes from the browser's upload progress events.

   > **Use a presigned POST policy, not a presigned PUT.** A presigned PUT cannot constrain the object's size — the client could upload a file of any size to a URL minted for a small one. A presigned POST carries a `content-length-range` condition that S3 itself enforces. If a PUT is used for implementation reasons, the commit step below **must** `HEAD` the object and delete-and-reject anything over the cap; the cap cannot be enforced client-side.

3. **`POST /activities/{id}/videos/{video_id}/complete`** — the client posts the poster JPEG (multipart, small, through the existing image pipeline for EXIF stripping and bounding). The API `HEAD`s the uploaded object to confirm it exists and record its true `byte_size`, stores the poster, and flips `status` to `ready`. Only `ready` rows are returned by reads.

**Orphans.** A `pending` row whose client vanished mid-upload leaves a row and possibly an object. Reap both on the same principle the photo pipeline already uses: tag the object `photo-status=orphaned` (rename the tag key to something media-neutral) so the bucket's existing lifecycle rule collects it, and soft-delete `pending` rows older than a configurable age. This is a startup sweep or a periodic job, not a request-path concern.

### API Surface

| Method | Path | Purpose |
| --- | --- | --- |
| `POST` | `/activities/{id}/videos` | Reserve a video, return a presigned upload target. |
| `POST` | `/activities/{id}/videos/{video_id}/complete` | Attach the poster, confirm the object, flip to `ready`. |
| `PATCH` | `/activities/{id}/videos/{video_id}` | Set/clear caption. |
| `PUT` | `/activities/{id}/videos/order` | Reorder; body carries the complete id list. |
| `DELETE` | `/activities/{id}/videos/{video_id}` | Soft-delete the row, tag the objects for reaping. |

Reads: `GET /activities/{id}` gains a `videos` array beside `photos`, populated by an unconditional `attachVideos` alongside `attachPhotos` — **not** gated on `activity_type`. Each entry carries presigned `url` and `poster_url`, minted with the same windowed presigner photos use so repeat views are browser-cache hits.

Config lives in a `[videos]` block in `config.toml` next to `[photos]` — public literals, not env vars, per the repo's config-vs-secrets convention: `max_per_activity`, `max_upload_bytes`, `allowed_content_types`, `presign_window_hours`, `upload_url_ttl_minutes`, `poster_max_edge_px`, `poster_jpeg_quality`, `caption_max_chars`, `pending_reap_after_hours`.

### Frontend

Conforms to [`design-system.md`](../design-system.md) and introduces no new tokens: the 14px panel radius on the video container and poster, `--accent` (periwinkle `#9aa6d6`) for interactive affordances and focus rings only, Manrope throughout.

- **`components/activity-detail/VideoStrip.tsx`** — a horizontal strip of poster thumbnails in aspect-ratio boxes (reserved from stored dimensions, so no reflow), each with a play affordance and a duration badge. For the owner: an **Add video** button, and an edit mode with per-video delete, caption editing, and move-left / move-right. Sits beside `PhotoStrip` in the same shared folder, for the same reason.
- **Playback** opens the `full` object in a `<video controls>` — in the existing `PhotoLightbox` shell if it generalizes cleanly, otherwise a sibling. Poster shown until play; nothing preloads beyond metadata.
- **Upload UX**: a progress bar driven by real upload progress. `fetch` does not expose upload progress, so this needs `XMLHttpRequest` or a streamed request — call it out so it isn't discovered late.

### Route coverage: the lesson from photos

**Every activity detail route renders `VideoStrip`.** Today that is exactly three:

- `app/(app)/workouts/[id]/page.tsx`
- `app/(app)/running/[id]/page.tsx`
- `app/(app)/hiking/[id]/page.tsx`

This is a **completion criterion, not a nice-to-have**. The photo rollout wired `PhotoStrip` into `/workouts/[id]` only; the SOW said "the activity detail page" in the singular and never enumerated routes, so a one-route implementation satisfied a literal reading while leaving the summit-selfie-on-a-hike story — the SOW's own motivating example — impossible to perform. It took a follow-up PR to fix, and the same shape had already happened once with the heart-rate-zones widget ([`prog-strength-web#133`](https://github.com/Prog-Strength/prog-strength-web/pull/133)).

Therefore:

- Each of the three routes gets a test asserting the strip renders. A shared component with one call site is not done.
- The API's `videos` array is populated for **any** activity type, gated on data and never on `activity_type`.
- [`adding-an-activity-type.md`](../adding-an-activity-type.md) gains videos alongside photos in its "does NOT come free" client opt-in entry, so the next activity type inherits the requirement rather than rediscovering the bug.

### Infrastructure

A new `activity_video_storage` module in `prog-strength-infra`, copied from `activity_photo_storage`:

- Private, un-versioned bucket; orphan-tag lifecycle reaping.
- Bucket-scoped IAM on the instance role, adding `s3:PutObject` presign capability.
- **A CORS configuration** — new relative to photos, and required: the browser PUT/POSTs cross-origin directly to S3. Allowed origins must cover the production web origin and Vercel preview deployments.
- `VIDEO_BUCKET_NAME` added to `compose/api/config.env`, `deploy/api.sh`'s required keys, **and `compose/api/docker-compose.yml`'s api service environment.** All three. The photo rollout did the first two and not the third, so uploads returned 503 on a fully green deploy — see [`prog-strength-infra#58`](https://github.com/Prog-Strength/prog-strength-infra/pull/58), which also added a CI check that now catches exactly this omission.

### Cost

Single-user, so the absolute numbers are small, but they're worth stating since this is the first feature where storage isn't rounding error. S3 Standard is ~$0.023/GB-month; egress is ~$0.09/GB. A hundred one-minute 1080p clips is roughly 6 GB — about **$0.14/month** to store. Playback egress is bounded by how often the owner rewatches. Both are acceptable; neither would be at multi-user scale without a CDN and a transcode ladder.

## Open Questions

1. **HEVC playback — the real cost of not transcoding.** iPhones record HEVC/H.265 by default. That plays in Safari and largely does not in Chrome or Firefox, so a video stored untouched may be unplayable in the owner's own browser. Three options, and this needs a decision before implementation:
   - **Accept and flag** *(recommended)* — store anything in the allowlist, detect HEVC, and show "this format may not play in this browser; it's stored at full quality." Preserves the keepsake, which is the point; never silently rejects a moment the user can't re-record.
   - **Reject HEVC at upload** — guarantees playback, but hands the user a failure at the moment they're trying to save something, and the fix (iOS → Camera → Formats → *Most Compatible*) only helps future recordings.
   - **Transcode a playback copy** — solves it properly and pulls in everything this SOW is built to avoid.
2. **Does the poster survive a failed generation?** The schema allows a null `poster_s3_key`. Confirm the strip renders a sensible placeholder rather than a broken box.
3. **Upload size cap.** What's the real ceiling — 500 MB, 1 GB, 2 GB? Drives the presigned POST condition and the browser's practical timeout behavior.
4. **Pending-row reaping cadence.** Startup sweep, periodic goroutine, or a manual admin endpoint? The volume is tiny; the simplest correct option probably wins.
5. **Does `PhotoLightbox` generalize to video, or does it fork?** Worth a look before assuming either.
6. **Multi-user re-evaluation trigger.** Storing originals with GPS metadata intact is sound *because* videos stay off the social surface and there's one user. If either changes, metadata stripping becomes mandatory and ffmpeg comes back on the table. Worth recording as the explicit trigger rather than rediscovering it.
