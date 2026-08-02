---
status: draft
repos:
  - prog-strength-api
  - prog-strength-web
  - prog-strength-infra
  - prog-strength-docs
---

# Photo Upload Off the Request Path

**Status**: Draft · **Last updated**: 2026-08-02

## Introduction

A 6 MB JPEG cannot be attached to a running activity. The browser reports a transport-level failure — `(failed) net::ERR_SSL_BAD_RECORD_MAC_ALERT`, no HTTP status — and the photo is silently lost. Small photos still work, which is what made this look like a network flake rather than a product defect.

It is not a flake. `POST /activities/{id}/photos` performs the entire upload, decode, re-encode, and two S3 writes inside **one HTTP request**, and that request is bounded by the API's global `ReadTimeout: 10s` / `WriteTimeout: 10s` (`internal/server/server.go:838-841`). There is no per-route override. Every photo upload has ten seconds — measured from the moment the request headers land — to do all of it.

Measured against that budget, with production `config.toml` values on a 12 MP source (what a 6 MB JPEG is):

```
source JPEG = 7.80 MB
processPhoto took 999ms -> full 10.02 MB (4032x3024), thumb 70 KB
```

That is one second on an M-series laptop. Production is a burstable two-vCPU Graviton (`variables.tf:126` defaults to `t4g.small`) shared with `mcp`, `agent`, Caddy, Prometheus, and Grafana — several times slower, and worse once CPU credits deplete. Add the 6 MB transfer over a residential upstream (~5s at 10 Mbps) and the S3 PUT of a **10 MB** full variant. The re-encode at `full_jpeg_quality = 95` and native resolution produces an object *larger than the file the user picked*, so the host uploads more bytes than it received.

Both timeout paths were reproduced against a server configured with these exact settings:

```
[ReadTimeout]  client err = write tcp ...: use of closed network connection
               server-side err = read tcp ...: i/o timeout
[WriteTimeout] client err = EOF
```

In both cases the connection is torn down with **no HTTP response at all**. That is why the browser surfaces a transport error instead of a clean `413` or `500` — there is nothing to surface. The failure is invisible to any error handling the client has, and invisible to any status-code-based alerting the API has.

The [Activity Videos](activity-videos.md) SOW moved video bytes off this host and said of the path it was leaving behind: *"That is fine for a 6 MB JPEG and impossible for video."* The first half of that sentence was optimistic. This SOW retires it.

After this ships: the user picks a photo of any size the product allows, watches a real progress bar, and the photo appears — with no ten-second cliff, and no request that can die without saying why.

## Proposed Solution

Photos adopt the two-phase presigned upload that videos already use — `reserve` → direct browser PUT to S3 → `commit` — **and** move the image pipeline off the request path into an asynchronous worker.

The second half is the load-bearing part, and it is what makes this more than a copy of the video design.

### Why direct-to-S3 alone does not fix this

Direct-to-S3 removes one of the two problems. It does not remove the other, and shipping only half would make things worse.

Today the bytes make **two sequential network trips**, and the second cannot begin until the first completes:

```
browser --6 MB--> Caddy --6 MB--> API   (whole file buffered in RAM)
                                   |
                                   +-- decode + re-encode x2   (~1s local; several on t4g)
                                   |
                                   +-- PUT 10 MB --> S3
                                   |
                                   +-- SQLite insert + 2 presigns
                                  <-- 201
        \_________________________ all inside one 10s deadline _________________________/
```

Two-phase upload collapses the first trip. The 6 MB goes browser→S3 once, as a plain PUT that is **not an API request at all** — no Go handler, no `WriteTimeout`, no 2 GB host buffering the file. The API handles two small JSON round-trips instead. That is a genuine and large win: it is the difference between the slowest part of the operation being subject to a ten-second ceiling and it not being subject to one.

But **photos are not videos**, and the difference is the thing that breaks a naive port. Video is stored byte-for-byte — the video SOW's central decision, and the reason nothing has to happen server-side after the object lands. A photo *must* be re-encoded: dropping the source's EXIF/GPS is the privacy mechanism, it is documented as unconditional (`photo_pipeline.go`), and photos — unlike videos — are published to the social timeline, so that guarantee is load-bearing rather than defensive.

So if `commit` runs `processPhoto` synchronously, the ten-second budget is not removed. It is *reloaded, with a download added to it*:

```
commit --> GET 6 MB from S3 --> decode + re-encode --> PUT 10 MB to S3 --> 200
           \___________________ still inside the same 10s deadline ___________________/
```

That is strictly worse than today for the processing half, because the host now has to fetch the bytes back that it previously received for free. Direct-to-S3 with a synchronous commit trades a client-upload timeout for a server-processing timeout and calls it progress.

The fix has to be both: **the bytes bypass the host, and the work leaves the request.** `commit` records that the object landed and returns immediately; a background worker does the pipeline and flips the row to ready.

### What this buys

- **No ten-second ceiling anywhere on the path.** The upload is S3's problem; the processing is a worker's problem. Neither is a bounded HTTP handler.
- **Perceived latency drops to the upload itself.** Today the user waits for transfer *plus* processing *plus* re-upload before the UI moves. After this, the UI advances as soon as the PUT completes; the render fills in behind it.
- **The host stops buffering whole images in RAM.** `ParseMultipartForm(32 MiB)` plus `io.ReadAll` plus a full-resolution RGBA working buffer, on a 2 GB box shared with five other services, goes away from the request path entirely.
- **Failures become visible.** A processing failure is a row in a known state with a logged error, not a severed TCP connection that no status-code metric will ever count.

### What it costs

- A `pending` → `ready` lifecycle on `activity_photo`, and read surfaces that tolerate a photo that exists but is not yet renderable.
- A worker, plus a reaper for reservations whose upload never arrived — both of which `activity_video` already needed and can be generalized from.
- A round-trip that was one request becomes three.

## Goals and Non-Goals

### Goals

- **Photo bytes never transit the API host.** The client PUTs directly to the activity-photo bucket through a presigned URL.
- **The image pipeline runs off the request path**, so no photo operation is bounded by the API's global 10s timeouts.
- **The server stays authoritative on EXIF/GPS stripping and output dimensions.** Every stored variant is produced by `processPhoto`. No client-supplied bytes are ever served as a photo.
- **A `pending` state that the UI renders honestly** — the photo appears in the strip immediately with a processing affordance, rather than vanishing until the worker finishes.
- **Abandoned reservations are reaped**, so a cancelled or failed upload leaves neither a permanent pending row nor an orphan object.
- **The synchronous endpoint keeps working through the transition**, so `api` and `web` can ship in either order.
- **CORS on the activity-photo bucket**, which does not exist today and is required the moment a browser PUTs to it.
- **An immediate mitigation ships first** — a per-request deadline extension on the existing endpoint — so the bug is not open for the length of this project. See [Sequencing](#sequencing).

### Non-Goals

- **Changing the fidelity policy.** `full_max_edge_px = 20000` / `full_jpeg_quality = 95` were chosen deliberately in [`prog-strength-api#94`](https://github.com/Prog-Strength/prog-strength-api/pull/94) and are not revisited here. This SOW makes that policy affordable rather than arguing with it.
- **Client-side re-encoding as the privacy mechanism.** A browser canvas re-encode does strip EXIF, but a presigned PUT accepts whatever the client sends — so the guarantee would rest on client cooperation. Videos accepted that and paid for it by staying out of the timeline. Photos are *in* the timeline; the server must remain authoritative.
- **Raising the global `ReadTimeout`/`WriteTimeout`.** Weakening every route's slow-loris protection to accommodate one upload path is the wrong trade.
- **Mobile.** `prog-strength-mobile` uses only the avatar endpoint (`lib/api.ts:1543`); it has no activity-photo upload, which is why it is absent from `repos:`.
- **Avatar and TCX uploads.** Both sit under the same 10s ceiling and both should eventually follow this pattern. Out of scope here; tracked in Open Questions.
- **A general-purpose job queue.** One worker goroutine with a claim query is enough at single-user scale. See Open Questions.
- **CloudFront or any CDN.**
- **Agent/MCP access to photos.** Unchanged; neither repo needs edits.

## Implementation Details

### Data Model

Migration `051_activity_photo_pending.sql` — additive-only, no rebuild-and-copy.

The columns 047 declared `NOT NULL` (`s3_key`, `thumb_s3_key`, `byte_size`, `width`, `height`) stay `NOT NULL`. SQLite cannot drop a `NOT NULL` in place, and the rebuild-and-copy lesson from 042/045 is not worth re-learning for this. Instead, `reserve` writes the same placeholders the video path already established — `s3_key = ""`, then `SetS3Key` once the id is known (`video_handler.go`) — and the dimension columns take `0` until the worker fills them.

```sql
-- status is the three-phase upload's state:
--   'pending'    reserved; the client is uploading the original to S3
--   'processing' commit confirmed the object; the worker holds it
--   'ready'      variants written; the row is renderable
--   'failed'     the worker gave up; original retained for diagnosis
-- Reads return 'ready' and 'processing'; only 'ready' resolves to URLs.
-- No CHECK constraint, for the same reason content_type has none.
ALTER TABLE activity_photo ADD COLUMN status TEXT NOT NULL DEFAULT 'ready';

-- The uploaded source object, distinct from the derived full/thumb variants.
-- Retained until the worker succeeds, then tagged for lifecycle reaping — it is
-- the only thing that makes a failed render retryable.
ALTER TABLE activity_photo ADD COLUMN original_s3_key TEXT;

-- Attempt accounting, so a poison image cannot be retried forever.
ALTER TABLE activity_photo ADD COLUMN attempts INTEGER NOT NULL DEFAULT 0;
ALTER TABLE activity_photo ADD COLUMN last_error TEXT;

-- Backs both the worker's claim query and the pending-row reaper.
CREATE INDEX idx_activity_photo_status ON activity_photo(status, created_at);
```

`DEFAULT 'ready'` is what makes this additive: every existing row was written by the synchronous path and is, by definition, already ready. No backfill.

### Write Path

**Phase 1 — `POST /activities/{id}/photos/reserve`.** Mirrors `reserveVideo` closely enough to read side by side. Resolves the owned live parent through the existing `resolveVideoParent` equivalent (worth extracting to a shared `resolveActivityParent` — both surfaces now need it), validates the declared content type against the image allowlist, checks the declared size against `max_upload_bytes` as a courtesy, enforces `max_per_activity` against the live count, inserts a `pending` row, builds the original's key, and returns a presigned PUT plus its expiry.

The declared size is not the enforcement point. A client can lie; `commit` HEADs the object and re-checks against the real size, exactly as `completeVideo` does.

**Phase 2 — browser PUTs the original directly to S3.** No API involvement. This is the step that used to be bounded by `WriteTimeout` and now is not.

**Phase 3 — `POST /activities/{id}/photos/{photo_id}/commit`.** HEADs the object to confirm it landed and to learn its true size. Oversize → tag the object orphaned, retire the reservation, `413`. Otherwise it accepts an optional `caption`, flips the row `pending` → `processing`, and returns `202` with the row's current DTO. **It does not touch the image.** This handler is a couple of small queries and one HEAD; it fits inside the global 10s budget with three orders of magnitude to spare, so no deadline exception is needed.

### Processing Worker

A single goroutine started alongside the existing background workers in `internal/server`, on a short tick:

1. Claim one `processing` row whose `attempts` is under the cap, oldest first, via a conditional `UPDATE ... RETURNING` so a future second worker cannot double-claim.
2. `GET` the original from S3 into memory. This is the one place a whole image is still buffered — but it is on a worker with no request deadline and a controllable concurrency of one, not on an inbound request path with five other services contending.
3. Run `processPhoto` with the unchanged production opts.
4. `PUT` the full and thumb variants under the existing variant keys (`buildPhotoKey`, unchanged).
5. Update the row: real `s3_key`, `thumb_s3_key`, `byte_size`, `width`, `height`, `status = 'ready'`.
6. Tag the original orphaned so the bucket's lifecycle rule reaps it.

A decode failure is terminal — the bytes are not a usable image — so it goes straight to `failed` without retrying. Transient failures (S3, disk) increment `attempts` and retry with backoff, up to the cap, then `failed` with `last_error` recorded.

A **reaper** on a slower tick retires rows left `pending` past the presign TTL (the upload never happened) and tags any object under their reserved key. `activity_video` needs exactly this and should share the implementation rather than grow a second copy.

### Read Path

`toPhotoDTO` gains a status branch. `ready` behaves exactly as today. `processing` returns the DTO with `url`/`thumb_url` null and `status` populated, so the client can render a placeholder at the right position. `pending` and `failed` are excluded from reads entirely — a reservation that never uploaded and a render that gave up are both non-photos as far as the UI is concerned.

`photoDTO` gains `status`. This is an additive field on an existing response shape; the timeline cover query must filter to `status = 'ready'` so a processing photo never becomes a feed post's cover.

### API Surface

| Method | Path | Change |
|---|---|---|
| `POST` | `/activities/{id}/photos/reserve` | **new** — mint presigned PUT, insert `pending` row |
| `POST` | `/activities/{id}/photos/{photo_id}/commit` | **new** — HEAD, flip to `processing`, `202` |
| `POST` | `/activities/{id}/photos` | **deprecated**, retained through the transition |
| `PATCH` | `/activities/{id}/photos/{photo_id}` | unchanged |
| `PUT` | `/activities/{id}/photos/order` | unchanged |
| `DELETE` | `/activities/{id}/photos/{photo_id}` | unchanged; also tags `original_s3_key` when present |

The synchronous endpoint stays mounted so `api` can deploy before `web`. It is removed in a follow-up once the client has moved — and its removal is what finally retires the 10s exposure, so it should not be left indefinitely.

### Frontend

`PhotoStrip`'s upload handler moves from one `uploadActivityPhoto` call to `reserve` → `PUT` → `commit`. The PUT uses `XMLHttpRequest` rather than `fetch` for upload progress events — the same choice the video uploader made, and the reason a 6 MB upload stops reading as a hang.

A `processing` photo renders in place at its position with a subtle shimmer rather than an image. The strip polls the activity detail endpoint on a short interval while any photo is `processing`, and stops when none are. Polling is the right call here over anything push-based: it is a handful of requests over a few seconds, on a page the user is already looking at.

Because `PhotoStrip` is shared, this lands on `/workouts/[id]`, `/running/[id]`, and `/hiking/[id]` together — the route-coverage lesson [`prog-strength-web#135`](https://github.com/Prog-Strength/prog-strength-web/pull/135) taught, and a per-route test asserting the upload affordance exists.

### Infrastructure

**The activity-photo bucket has no CORS configuration.** `modules/activity_photo_storage` never needed one, because no browser has ever talked to it directly — and `modules/activity_video_storage/variables.tf:17` says so explicitly: *"Required — unlike photos, video bytes go browser→S3, so S3 itself answers the CORS preflight."* That stops being true here. Without it the browser's preflight fails and every upload dies before a byte moves.

`modules/activity_photo_storage` gains an `aws_s3_bucket_cors_configuration` and a `cors_allowed_origins` variable, mirroring the video module: `PUT` and `HEAD`, origins set to the production web origin plus the Vercel preview wildcard, matching the API's own `cors.allowed_origins`.

The instance role already holds `PutObject`/`GetObject`/`HeadObject` on this bucket for the synchronous path; the worker's `GET` needs no new grant. The existing orphan lifecycle rule reaps originals once tagged, so no new rule is needed — the tag key is already what the rule matches.

### Configuration

New knobs in `config.toml` under `[photos]`, all non-secret and version-controlled:

```toml
upload_url_ttl_minutes = 15   # presigned PUT lifetime; mirrors [videos]
process_max_attempts   = 3    # transient-failure retry cap
process_tick_seconds   = 2    # worker poll interval
reap_after_minutes     = 30   # retire pending rows older than this
```

`max_upload_bytes = 33554432` is unchanged, but its enforcement moves: the presign carries a `content-length-range`, and `commit` re-checks the HEAD size. The comment justifying 32 MiB currently reasons about `ParseMultipartForm` buffering on a 4 GB host — that rationale no longer applies and the comment must be rewritten rather than left to mislead.

### Testing

- Unit: `reserve` rejects bad content types, oversize declarations, and a full activity; `commit` rejects a missing object (`409 upload_incomplete`), an oversize real object (`413`, with orphan tagging and reservation retirement), and a non-pending row (`409`).
- Worker: success writes both variants and flips to `ready`; a decode failure goes terminal without retry; a transient S3 failure increments `attempts` and retries; the cap sends it to `failed` with `last_error` set.
- Concurrency: two workers cannot claim the same row.
- Reaper: a `pending` row past TTL is retired and its key tagged.
- Read: a `processing` photo serialises with null URLs and is excluded from timeline covers; a `ready` photo is byte-identical in shape to today's DTO plus `status`.
- Regression: the deprecated synchronous endpoint still works end to end while mounted.
- Web: per-route upload affordance; progress events fire; the strip stops polling once nothing is `processing`.

### Sequencing

This SOW is the durable fix, and it is several days of work across three repos. The bug is live now.

1. **Immediate, separate from this SOW — done.** `internal/uploadwindow` extends the deadline for the current request with `http.ResponseController` `SetReadDeadline`/`SetWriteDeadline`, raising the ceiling to two minutes for the three upload routes (`uploadPhoto`, `uploadAvatar`, `readTCXUpload`) without touching the global server timeouts every other route depends on. Small, targeted, immediately unblocks 6 MB photos.
2. `infra` — CORS on the photo bucket. Independent, deployable alone, inert until used.
3. `api` — migration, `reserve`/`commit`, worker, reaper, read-path status. Synchronous endpoint stays mounted.
4. `web` — three-phase uploader, progress, processing state, polling.
5. `api` — remove the deprecated endpoint, and `internal/uploadwindow` with it once no route still moves file bytes through this process.

Step 1 is the reason this SOW does not need to be rushed. Step 5 is the reason it must not be abandoned halfway.

## Open Questions

- **Should `avatar` and `tcx` follow?** Both sit under the same 10s ceiling. Avatar is capped at 5 MiB and re-encodes on the request path — the same defect with a smaller blast radius. TCX archives the original to S3 and parses synchronously; a large multi-hour ride file is a plausible trigger. Both want this pattern. Neither is in scope here, and doing all three at once triples the review surface. Recommend: fix all three under step 1, migrate photos here, revisit the others once this pattern has run in production.
- **One worker or a claim-based pool?** Single-user, and the pipeline is CPU-bound on two burstable vCPUs — a pool would contend with the API rather than add throughput. The claim query is written to be safe for more than one worker so this can change without a migration, but v1 runs exactly one.
- **Is `processing` worth showing at all?** The alternative is optimistic rendering: display the client's local `ObjectURL` in the strip until the server's version is ready. Better-feeling, and meaningfully more state to get wrong — the local blob has to survive a navigation, and it lies about what is actually stored. Recommend the honest placeholder for v1.
- **Should the original be retained rather than reaped?** `full` is currently the archival copy, and the SOW for photos says so. Keeping the true original would make the fidelity policy revisable after the fact — re-render from source if the pipeline ever changes — at roughly double the storage. The S3 footprint dashboard exists to answer whether that is affordable; this is a product call, not a technical one.
- **What surfaces a `failed` photo to the user?** Nothing does, in this design — it is excluded from reads and visible only in logs and the row. At single-user scale that is arguably fine and arguably a silent data-loss path, which is the exact class of problem that produced this SOW. An alert on `failed` count is cheap and worth considering before this ships.
