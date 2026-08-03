---
status: shipped
repos:
  - prog-strength-web
  - prog-strength-docs
---

# Multi-Photo Upload

**Status**: Shipped · **Last updated**: 2026-08-03

## Introduction

Attaching photos to an activity is one at a time. Open the picker, choose a file, wait, open the picker again. A run with five photos worth keeping is five round trips through the same dialog, and the friction is enough that the photos stay in the camera roll — which is the exact failure the [Activity Photos](activity-photos.md) SOW existed to prevent.

The fix is small, and it is small *because* of work already done. [Photo Upload Off the Request Path](photo-upload-direct-to-s3.md) turned each upload into an independent three-phase flow — reserve, direct PUT to S3, commit — with a background worker that already drains a queue rather than handling one item per tick. Uploading five photos is running that flow five times. Nothing about it needed to be designed for batches; it already is one.

Under the synchronous endpoint this SOW retired, the same feature would have meant five sequential requests each capped at ten seconds, each able to die without a status code.

After this ships: on any activity detail page, the user selects several photos at once and they attach.

## Proposed Solution

**Client-only.** `PhotoStrip` gains multi-select and a sequential upload loop. No endpoint, schema, or worker change — N photos is N independent three-phase flows.

Sequential rather than parallel, for three reasons that happen to agree:

- Parallel uploads of several multi-megabyte files contend for the same residential upstream, so the batch finishes no sooner and every progress bar crawls.
- Only one upload in flight means `reservePhoto`'s `SELECT COALESCE(MAX(position), -1) + 1` cannot interleave with itself, so positions stay dense and distinct.
- A "photo 3 of 8, 62%" display is literally true, rather than a summary invented over concurrent transfers.

## Goals and Non-Goals

### Goals

- **Select and attach several photos in one interaction**, on every activity detail route (`PhotoStrip` is shared, so this lands on workouts, running and hiking together).
- **Partial success is preserved.** Photos that upload are attached; failures are reported and do not roll anything back.
- **Over-selection is refused before anything uploads**, naming how many slots remain.
- **Honest progress**: which photo of how many, and that photo's transfer percentage.

### Non-Goals

- **Uploads surviving navigation.** Leaving the page abandons whatever has not finished. Keeping a batch alive would mean lifting upload state out of the component into app-level context plus a way to report completion from a page the user has left — machinery far larger than the feature.
- **Retrying failed files in place.** Failures are reported; the user re-picks. Holding failed `File` objects and rendering a retry surface is a second UI for a case that should be rare.
- **All-or-nothing batches.** Rolling back committed photos over one bad file throws away good work, and the rollback can itself fail.
- **Per-photo captions during upload.** Captions are already applied after the fact through `PATCH /activities/{id}/photos/{photo_id}`; batch does not change that.
- **Parallel uploads.** See above — slower in practice and it reintroduces the position race.
- **Server-side cap enforcement.** Deliberately deferred; see [Known Gap](#known-gap).
- **Mobile.** `prog-strength-mobile` has no activity-photo upload at all.

## Implementation Details

### Selection and validation

The file input gains `multiple`. `handleFiles` already receives a `FileList` and discards everything past `[0]`; that becomes a loop.

Validation runs **before any upload starts**, in this order:

1. **Cap.** If `photos.length + files.length > MAX_PER_ACTIVITY`, reject the entire selection with a message naming the remaining slots ("You can add 3 more photos — you selected 6"). Nothing uploads. Refusing up front beats filling slots in selection order, which would silently pick winners and leave the UI owing an answer to "which three did it take?".
2. **Per-file type and size.** A file failing either is recorded as a failure and skipped — it does **not** abort the batch. One HEIC in a selection of eight must not cost the other seven.

### Upload loop

A `for` loop awaiting `uploadActivityPhotoDirect` per file, each iteration wrapped in its own `try`/`catch` so one failure cannot abort the rest. State per iteration is `{ index, total, fraction }`, with `fraction` fed by the existing per-file `onProgress`.

`onPhotosChanged` is called after each successful commit rather than once at the end, so photos appear in the strip as they land instead of all at once. The `processing` placeholder and the poll-until-none-processing effect are untouched — both already handle several in-flight photos, having been built for the single-photo case without that being a constraint.

### Progress

Replaces the current `uploadProgress: number | null` with a batch object. The button label reads `Uploading 3 of 8 — 62%`.

### Failure reporting

Failures accumulate as `{ name, reason }` and produce one grouped toast: *"3 photos failed — 1 unsupported format, 2 network errors."*

Two classes are distinguished because the user's next action differs:

- **Rejected** — wrong type, too large. Re-pick a different file.
- **Failed** — network or server. Retrying the same file is reasonable.

### Testing

Component-level against `PhotoStrip` with the API client mocked, matching the existing tests:

- A batch of three uploads all three, in selection order.
- A mid-batch failure leaves earlier successes attached **and still attempts the later files** — the assertion that a failure does not abort the loop.
- An over-cap selection uploads nothing and reports the remaining slots.
- Progress reports the correct `index`/`total` as the batch advances.
- A file rejected on type is skipped without aborting the batch.

## Known Gap

The per-activity cap and position assignment remain **client-enforced**. `reservePhoto` reads `CountLive` and then inserts non-atomically, so concurrent reserves can pass the check before any row lands, and `MAX(position)+1` can return the same value twice.

Sequential uploading means this client stops triggering either race — the exposure is strictly smaller after this ships than before it. But a second client, a direct API call, or two browser tabs could still exceed the cap or duplicate a position. Positions are deliberately non-unique (reorder rewrites the whole set, so intermediate duplicates must be legal) and `id` breaks ties, so a duplicate is cosmetic rather than corrupting.

The fix is a transactional `reservePhoto` — count and insert in one transaction — at roughly twenty lines plus a concurrency test. It is deferred rather than forgotten. Triggers to do it:

- `prog-strength-mobile` gains photo upload, giving a second concurrent client, or
- the platform stops being single-user.

Recording this rather than fixing it now is a choice, not an oversight.

## Open Questions

- **Should the batch be cancellable mid-flight?** `putPhotoToStorage` already accepts an `AbortSignal` and nothing passes one. Wiring a Cancel button to abort the in-flight PUT and stop the loop is small, but it raises a question this SOW otherwise avoids: a cancelled batch leaves some photos attached, which is the partial-success model again in a place the user did not choose it. Recommend leaving it out of v1 and seeing whether anyone reaches for it.
- **Is there a selection size worth refusing outright?** The cap bounds it at ten today, so the question is moot until `max_per_activity` rises. Worth remembering that the client holds every selected `File` in memory for the duration of the batch.
