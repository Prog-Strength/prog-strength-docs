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

And it appears at better quality than it does today. Investigating the timeout meant measuring the pipeline stage by stage, which turned up something the timeout had been hiding: the most expensive stage is also the one that damages the photo. That finding reshaped the design, and the case for it is in [The re-encode buys nothing](#the-re-encode-buys-nothing-and-costs-fidelity).

## Proposed Solution

Three changes, and the third was not obvious until the pipeline was actually measured:

1. Photos adopt the two-phase presigned upload videos already use — `reserve` → direct browser PUT to S3 → `commit`.
2. The remaining server-side work moves off the request path into an asynchronous worker.
3. **The full-resolution re-encode is deleted.** The stored photo becomes the user's original bytes with their metadata rewritten in place — losslessly — rather than a re-compressed copy.

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

But **photos are not videos**, and the difference is the thing that breaks a naive port. Video is stored byte-for-byte — the video SOW's central decision, and the reason nothing has to happen server-side after the object lands. A photo cannot be: the object served from the bucket must have its GPS removed first, that guarantee is documented as unconditional (`photo_pipeline.go`), and photos — unlike videos — are published to the social timeline, so it is load-bearing rather than defensive. A presigned GET serves S3 directly with no server in the path, so there is no serve-time hook. Whatever sits under the serving key must *already* be clean.

So something must still run server-side after the upload lands, and if `commit` runs it synchronously the ten-second budget is not removed. It is *reloaded, with a download added to it*:

```
commit --> GET 6 MB from S3 --> rewrite --> PUT to S3 --> 200
           \_________ still inside the same 10s deadline _________/
```

Direct-to-S3 with a synchronous commit trades a client-upload timeout for a server-processing timeout and calls it progress.

The fix has to be both: **the bytes bypass the host, and the work leaves the request.** `commit` records that the object landed and returns immediately; a background worker does the rest and flips the row to ready.

### The re-encode buys nothing, and costs fidelity

The above says work must happen after upload. It does not say that work is `processPhoto` as it stands. Measuring each stage on a 12 MP source with production config — the shape of a 6 MB phone photo — makes the case for cutting most of it:

```
source JPEG            = 7.80 MB
DecodeConfig (bounds)  = 4µs        4032x3024
jpeg.Decode            = 250ms
applyOrientation(6)    = 80ms
full variant  q95      = 304ms      10.02 MB   (128% of source)
thumb 800/q85          = 205ms      70 KB
```

**The full variant costs 304 ms of CPU and comes out 28% larger than the file it was made from — and it is a worse image.** Re-encoding an already-lossy JPEG is generation loss; it cannot recover detail, only discard more. `config.toml` currently reasons about this variant as though it were the opposite:

> the `full` variant is the ARCHIVAL copy. No original bytes are retained anywhere... whatever `full` stores is the best copy that will ever exist for that photo.

The intent behind [`prog-strength-api#94`](https://github.com/Prog-Strength/prog-strength-api/pull/94) was sound — the old 2048 px clamp really was destroying phone photos, and lifting it was right. But raising the ceiling to native resolution at q95 does not preserve the original; it produces a slightly-degraded, larger copy while throwing the original away. The only way to store "the best copy that will ever exist" is to store the actual bytes.

There is a second, quieter loss. Go's `image/jpeg` encoder emits only `DQT`, `SOF0`, `DHT`, `SOS` — no `APP2` segment, verified by walking the markers of its output. **Every ICC colour profile is discarded on re-encode.** A modern phone shoots Display P3; the stored copy carries no profile, so every viewer renders it as sRGB and the colours shift. The pipeline built to protect fidelity is quietly degrading it in two independent ways.

### Strip the metadata losslessly instead

None of the privacy requirement needs a re-encode. A JPEG is a sequence of marker segments, and EXIF lives in `APP1`. The file can be rewritten keeping only what should survive and dropping the rest, **without touching the entropy-coded scan data at all** — pure Go, no cgo, which matters given the `sqlite-vec`/musl history. PNG (`eXIf`) and WebP (`EXIF`) carry the same metadata in their own chunk structures and take the same treatment.

The rule is a **whitelist, not a blocklist**, because location does not only live in EXIF GPS tags — XMP (`APP1`, different namespace) and IPTC (`APP13`) can both carry coordinates, and MakerNotes carry camera serial numbers. Rebuild with only:

- **`APP2` / ICC profile** — kept, which *fixes* the colour bug above rather than preserving it.
- **EXIF `Orientation`** — kept, so browsers rotate correctly. Keeping it also deletes the 80 ms `applyOrientation` pass and its full-resolution RGBA allocation, since nothing needs baking into pixels any more.

Everything else goes: GPS, timestamps, `Make`/`Model`, MakerNote, XMP, IPTC, comments.

What remains server-side is a decode and a thumbnail — the decode is unavoidable, since Go's stdlib does not expose JPEG's DCT-domain scaled decoding:

| | today | proposed |
|---|---|---|
| CPU per photo | ~840 ms | ~460 ms |
| stored per photo | 10.09 MB | 6.07 MB |
| full-size fidelity | generation loss, no ICC | the original bytes, ICC intact |

Cheaper, smaller, and higher fidelity at once — the rare case where the ambitious answer and the simple one agree, which is the same note the video SOW struck for a different reason.

### What this buys

- **No ten-second ceiling anywhere on the path.** The upload is S3's problem; the processing is a worker's problem. Neither is a bounded HTTP handler.
- **Perceived latency drops to the upload itself.** Today the user waits for transfer *plus* processing *plus* re-upload before the UI moves. After this, the UI advances as soon as the PUT completes; the render fills in behind it.
- **The host stops buffering whole images in RAM.** `ParseMultipartForm(32 MiB)` plus `io.ReadAll` plus a full-resolution RGBA working buffer, on a 2 GB box shared with five other services, goes away from the request path entirely.
- **Failures become visible.** A processing failure is a row in a known state with a logged error, not a severed TCP connection that no status-code metric will ever count.
- **Photos get better, not just faster.** The stored full-size image stops being a degraded re-compression and becomes the original bytes, ICC profile intact — roughly half the CPU and ~40% less storage as a side effect.

### What it costs

- A `pending` → `ready` lifecycle on `activity_photo`, and read surfaces that tolerate a photo that exists but is not yet renderable.
- A worker, plus a reaper for reservations whose upload never arrived — both of which `activity_video` already needed and can be generalized from.
- A round-trip that was one request becomes three.
- **A hand-rolled metadata rewriter across three container formats**, replacing a stdlib re-encode that, whatever else is wrong with it, cannot produce an unreadable file. This is the real cost of the design and it is why the strip carries the heaviest test burden in [Testing](#testing) and an entry in [Open Questions](#open-questions).
- **A staging object that briefly holds unstripped GPS.** It is never presigned, deleted on success, and lifecycle-expired as a backstop — but it exists, and the design has to keep saying so.

## Goals and Non-Goals

### Goals

- **Photo bytes never transit the API host.** The client PUTs directly to the activity-photo bucket through a presigned URL.
- **The remaining image work runs off the request path**, so no photo operation is bounded by the API's global 10s timeouts.
- **The server stays authoritative on metadata stripping.** No object is reachable through a presigned GET until the server has rewritten its metadata. The guarantee never rests on client cooperation.
- **The stored full-size photo is the user's original bytes**, metadata-rewritten but never re-compressed — strictly higher fidelity than today, and ~40% less storage.
- **ICC colour profiles survive**, fixing a silent colour shift the current re-encode introduces on every wide-gamut photo.
- **A `pending` state that the UI renders honestly** — the photo appears in the strip immediately with a processing affordance, rather than vanishing until the worker finishes.
- **Abandoned reservations are reaped**, so a cancelled or failed upload leaves neither a permanent pending row nor an orphan object.
- **The synchronous endpoint keeps working through the transition**, so `api` and `web` can ship in either order.
- **CORS on the activity-photo bucket**, which does not exist today and is required the moment a browser PUTs to it.
- **An immediate mitigation ships first** — a per-request deadline extension on the existing endpoint — so the bug is not open for the length of this project. See [Sequencing](#sequencing).

### Non-Goals

- **Re-encoding the full-size photo at all.** This SOW deletes that step rather than making it affordable. `full_max_edge_px` and `full_jpeg_quality` stop applying to the stored photo; see [Configuration](#configuration) for what becomes of them. This *serves* the goal of [`prog-strength-api#94`](https://github.com/Prog-Strength/prog-strength-api/pull/94) — the highest-fidelity copy the user will ever have — more completely than #94 itself could.
- **Client-side metadata stripping as the privacy mechanism.** A browser canvas re-encode does drop EXIF, but a presigned PUT accepts whatever the client sends — so the guarantee would rest on client cooperation. Videos accepted that and paid for it by staying out of the timeline. Photos are *in* the timeline; the server must remain authoritative.
- **Lossless rotation of pixels.** Keeping the `Orientation` tag is simpler, is what browsers already honour, and avoids the MCU-boundary edge cases a `jpegtran`-style transform has on dimensions that are not multiples of the block size.
- **Re-processing existing photos.** Everything already stored was written by the old pipeline and stays as it is; its originals are gone and cannot be recovered. See [Backfill](#backfill).
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
--   'ready'      stripped photo + thumb written; the row is renderable
--   'failed'     the worker gave up; original retained for diagnosis
-- Reads return 'ready' and 'processing'; only 'ready' resolves to URLs.
-- No CHECK constraint, for the same reason content_type has none.
ALTER TABLE activity_photo ADD COLUMN status TEXT NOT NULL DEFAULT 'ready';

-- The STAGING object the client PUT to, under a separate `uploads/` prefix.
-- It still carries the source's GPS, so it is never presigned for GET and is
-- deleted the moment the worker has written the stripped copy. Distinct from
-- s3_key (the serving object) precisely so the two can never be confused.
ALTER TABLE activity_photo ADD COLUMN upload_s3_key TEXT;

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
2. `GET` the staged object from S3 into memory. This is the one place a whole image is still buffered — but on a worker with no request deadline and a concurrency of one, not on an inbound request path contending with five other services.
3. **Validate.** The presigned PUT accepts whatever the client sent, so everything `uploadPhoto` does today still has to happen here: sniff with `http.DetectContentType` against the allowlist, and run `boundDimensions` off `DecodeConfig` (4 µs) before any pixel is decoded. This is the decompression-bomb guard and it does not move.
4. **Strip.** Rewrite the container keeping only `APP2`/ICC and EXIF `Orientation`; drop GPS, XMP, IPTC, MakerNote, timestamps, comments. Entropy-coded scan data is copied through untouched.
5. **Thumbnail.** Decode once, CatmullRom to `thumb_max_edge_px`, encode at `thumb_jpeg_quality`. Unchanged from today, and now the only re-encode in the system.
6. `PUT` the stripped photo under the serving key and the thumb under the thumb key (`buildPhotoKey`, unchanged).
7. Update the row: `s3_key`, `thumb_s3_key`, `byte_size`, `width`, `height`, `status = 'ready'`.
8. **`DELETE` the staged object** — not a lifecycle tag. It is the only copy carrying GPS and it should stop existing as soon as it is redundant, not in three days when the rule next runs.

A validation or strip failure is terminal — the bytes are not a usable image of an allowed type — so it goes straight to `failed` without retrying. Transient failures (S3, disk) increment `attempts` and retry with backoff, up to the cap, then `failed` with `last_error` recorded.

The strip must be **verified, not assumed**: re-read the rewritten bytes and confirm they still decode, and that no disallowed segment survives, before the `PUT` in step 6. A metadata rewrite that silently corrupts the scan data would destroy the only copy of the photo, and unlike a re-encode there is no second chance.

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
| `DELETE` | `/activities/{id}/photos/{photo_id}` | unchanged; also deletes the staged object when one is still present |

The synchronous endpoint stays mounted so `api` can deploy before `web`. It is removed in a follow-up once the client has moved — and its removal is what finally retires the 10s exposure, so it should not be left indefinitely.

### Frontend

`PhotoStrip`'s upload handler moves from one `uploadActivityPhoto` call to `reserve` → `PUT` → `commit`. The PUT uses `XMLHttpRequest` rather than `fetch` for upload progress events — the same choice the video uploader made, and the reason a 6 MB upload stops reading as a hang.

A `processing` photo renders in place at its position with a subtle shimmer rather than an image. The strip polls the activity detail endpoint on a short interval while any photo is `processing`, and stops when none are. Polling is the right call here over anything push-based: it is a handful of requests over a few seconds, on a page the user is already looking at.

Because `PhotoStrip` is shared, this lands on `/workouts/[id]`, `/running/[id]`, and `/hiking/[id]` together — the route-coverage lesson [`prog-strength-web#135`](https://github.com/Prog-Strength/prog-strength-web/pull/135) taught, and a per-route test asserting the upload affordance exists.

### Infrastructure

**The activity-photo bucket has no CORS configuration.** `modules/activity_photo_storage` never needed one, because no browser has ever talked to it directly — and `modules/activity_video_storage/variables.tf:17` says so explicitly: *"Required — unlike photos, video bytes go browser→S3, so S3 itself answers the CORS preflight."* That stops being true here. Without it the browser's preflight fails and every upload dies before a byte moves.

`modules/activity_photo_storage` gains an `aws_s3_bucket_cors_configuration` and a `cors_allowed_origins` variable, mirroring the video module: `PUT` and `HEAD`, origins set to the production web origin plus the Vercel preview wildcard, matching the API's own `cors.allowed_origins`.

The instance role already holds `PutObject`/`GetObject`/`HeadObject` for the synchronous path; the worker's `GET` needs no new grant, but the staged-object cleanup needs **`DeleteObject`**, which it does not currently have.

A short lifecycle rule on the `uploads/` prefix — expire after one day — is the backstop for staged objects the worker never got to. It is a backstop, not the mechanism: the worker deletes on success, and this catches only the rows that died between upload and processing. Given the prefix holds the one GPS-bearing copy of each photo, this should be the most aggressive rule in the bucket, not the same three-day orphan window everything else gets.

### Configuration

New knobs in `config.toml` under `[photos]`, all non-secret and version-controlled:

```toml
upload_url_ttl_minutes = 15   # presigned PUT lifetime; mirrors [videos]
process_max_attempts   = 3    # transient-failure retry cap
process_tick_seconds   = 2    # worker poll interval
reap_after_minutes     = 30   # retire pending rows older than this
```

**`full_max_edge_px` and `full_jpeg_quality` are deleted**, along with the block comment justifying them. Nothing re-encodes the full-size photo any more, so a knob controlling how it is re-encoded is worse than useless — it would read as live policy. `thumb_max_edge_px` and `thumb_jpeg_quality` stay and are now the only image-quality knobs in the system.

`max_upload_bytes = 33554432` is unchanged in value but changes in both meaning and enforcement. Enforcement moves to the presign's `content-length-range` plus `commit`'s HEAD re-check. Its *rationale* changes more: the current comment reasons about `ParseMultipartForm` buffering on a shared 4 GB host, which stops being true, and it also claims the ceiling "covers any phone/camera JPEG or **HEIC**" — which is wrong today for an unrelated reason (Go's sniffer has no HEIC branch, so HEIC uploads 415 before size is ever consulted). Both halves of that comment must be rewritten rather than left to mislead; the HEIC gap itself is tracked separately.

### Testing

The metadata rewrite is the part that carries real risk — it is the only step that can destroy the sole copy of a photo, and the only step whose failure is a privacy incident rather than a broken image. It gets the most coverage:

- **Strip, privacy:** a fixture JPEG carrying GPS, XMP, IPTC, MakerNote and a comment yields output where **none** of those segments survive. Asserted by walking markers, not by trusting the writer. Table-driven over JPEG, PNG (`eXIf`) and WebP (`EXIF`).
- **Strip, preservation:** `APP2`/ICC survives byte-identically; EXIF `Orientation` survives with its value intact; **the entropy-coded scan data is byte-identical to the source**. That last assertion is what makes "lossless" a fact rather than a claim.
- **Strip, round-trip:** the rewritten bytes still decode, and to the same dimensions.
- **Strip, adversarial:** truncated files, a JPEG with no `APP1` at all, a file with two `APP1` segments, EXIF with a bogus segment length, and a file whose declared length would run past `EOF` — none panic; all either produce valid output or fail terminally.
- **Colour regression:** a source with a Display P3 profile keeps it. This is the bug the current pipeline has; a test that fails against `processPhoto` and passes against the new path is the proof it is fixed.
- Unit: `reserve` rejects bad content types, oversize declarations, and a full activity; `commit` rejects a missing object (`409 upload_incomplete`), an oversize real object (`413`, with cleanup and reservation retirement), and a non-pending row (`409`).
- Worker: success writes both objects, flips to `ready`, and **deletes the staged object**; a validation failure goes terminal without retry; a transient S3 failure increments `attempts` and retries; the cap sends it to `failed` with `last_error` set.
- Worker, privacy: after a successful run the staged key is gone — asserted directly, since this is the whole reason the staging prefix exists.
- Concurrency: two workers cannot claim the same row.
- Reaper: a `pending` row past TTL is retired and its staged object deleted.
- Read: a `processing` photo serialises with null URLs and is excluded from timeline covers; a `ready` photo is byte-identical in shape to today's DTO plus `status`.
- Regression: the deprecated synchronous endpoint still works end to end while mounted.
- Web: per-route upload affordance; progress events fire; the strip stops polling once nothing is `processing`.

### Backfill

There is none, and there cannot be. Every photo already stored was re-encoded by the current pipeline and its original bytes were never retained — `config.toml` says so plainly ("No original bytes are retained anywhere"). Those photos keep whatever fidelity they have and keep no ICC profile; nothing can restore what was discarded at upload time.

This is worth stating rather than leaving implicit, because it puts a price on delay: every photo uploaded between now and step 3 is one more that permanently carries the generation loss. It is a mild argument for sequencing the `api` work sooner rather than letting it sit behind other roadmap items.

`DEFAULT 'ready'` on the new `status` column is what keeps the migration additive — existing rows are, by definition, already done.

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
- **~~Should the original be retained rather than reaped?~~** *Resolved by the design above.* The question assumed the stored `full` was a derivative and the original was a second, optional copy costing double the storage. Once the re-encode is deleted, the stored photo **is** the original, and the only extra copy is the GPS-bearing staged upload — which should be deleted as fast as possible rather than retained. The trade-off dissolved: higher fidelity and ~40% *less* storage, not more.
- **Is a lossless strip the right pure-Go bet?** It replaces a well-understood stdlib re-encode with a hand-rolled container rewrite across three formats, and the failure mode is worse — a re-encode that goes wrong produces a bad image, a rewrite that goes wrong produces an unreadable one, and there is no second copy. The mitigations are the verify-before-`PUT` step and the adversarial tests above. The alternative that avoids all of it is keeping the re-encode purely as insurance, at 304 ms, 28% size inflation, generation loss, and the ICC bug. Recommend the strip, but this is the one decision here worth a second opinion before implementation starts.
- **What surfaces a `failed` photo to the user?** Nothing does, in this design — it is excluded from reads and visible only in logs and the row. At single-user scale that is arguably fine and arguably a silent data-loss path, which is the exact class of problem that produced this SOW. An alert on `failed` count is cheap and worth considering before this ships.
