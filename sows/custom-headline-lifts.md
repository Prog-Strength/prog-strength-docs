# Custom Headline Lifts

**Status**: Shipped · **Last updated**: 2026-05-30

## Introduction

Headline lifts are hardcoded in the Go API today: three exercises (bench press, back squat, deadlift) are surfaced on the Personal Records page for every user. That list was a deliberate v1 simplification — defining the headline set in one place kept the eventual mobile client in sync with the web client without either repo shipping a config update — and the original Personal Records SOW explicitly filed user customization as a follow-up.

That follow-up is this work. Lifters care about different lifts: a powerlifter's three barbell lifts versus a hybrid athlete's overhead press and weighted pull-ups versus a bodybuilder's incline dumbbell bench and Romanian deadlift. Forcing every user through the same three slugs makes the trophy-case framing of the page weaker the further the user gets from a strict powerlifting template. After this feature ships, every lifter sees the lifts *they* care about on Personal Records, while new users still land on the same opinionated default that ships today.

## Proposed Solution

A new `user_headline_exercises` table stores each user's selection of exercise slugs along with display ordering. The existing curated global list — `workout.HeadlineExercises` in the Go API — stays put and serves two roles: the default for new users (anyone with zero rows in the new table falls back to it on read) and the "reset to defaults" payload the frontend can ship back when a user wants to start over.

The read path on `GET /personal-records` is unchanged in shape: same response, same DTOs, same per-lift estimated 1RM math. The handler just consults the new table first and falls back to `HeadlineExercises` when the user has no rows yet.

The write path is one new endpoint: `PUT /me/headline-exercises` accepts the user's full ordered selection as a JSON array of exercise slugs and replaces their row set in a single transaction. The modal on the Personal Records page submits this payload when the user hits Save. There are no per-row create / update / delete endpoints — the set-replacement model removes a class of partial-save drift bugs and matches the way the UI thinks about the data (the modal is a single editor, not a list of edit affordances). A third endpoint, `GET /headline-exercises/defaults`, exposes the global default list so the modal can label which exercises are part of the curated set and offer a "Reset to defaults" action without baking the slugs into the frontend.

A "Customize" button on the Personal Records page opens a modal listing the full exercise catalog (already a backend endpoint), grouped by muscle group, with checkboxes pre-populated from the user's current selection. The user toggles exercises, optionally reorders them, hits Save, the modal posts the new list, and the Personal Records page refetches.

## Goals and Non-Goals

### Goals

- Persist one row per `(user, exercise)` in a new `user_headline_exercises` table, including a `position` column so the user can control display order.
- `GET /personal-records` reads from the new table when the user has rows; falls back to `workout.HeadlineExercises` when they don't. Response shape and DTOs are unchanged.
- Add `PUT /me/headline-exercises` that replaces the user's full headline-lift selection in one transaction. Body is an ordered array of exercise slugs.
- Add `GET /me/headline-exercises` so the modal can fetch the user's current selection without first navigating to Personal Records.
- Add `GET /headline-exercises/defaults` so the modal can show "default" annotations and implement "Reset to defaults" without hardcoding slugs in the frontend.
- Validate on write that every slug exists in the exercise catalog. Reject with `400 Bad Request` on unknown slugs.
- Cap the selection at a sensible maximum (proposed: 12) so the Personal Records page stays scannable and a misbehaving client can't blow up the row set.
- Add a "Customize" affordance on the Personal Records page that opens the modal and persists changes via the new endpoint.

### Non-Goals

- **Per-rep-range customization** (e.g. a separate 1RM, 3RM, 5RM headline per exercise). The existing personal-records SOW filed this as out of scope and that stays — this work is purely about *which exercises* the user tracks, not how many records per exercise.
- **Sharing or cloning headline-lift sets between users.** "Copy a friend's setup" is interesting but out of scope; for a single-user beta there is no friend to copy from yet.
- **An admin UI for editing the global default.** `workout.HeadlineExercises` stays in Go code. Editing it requires a code change and a deploy, same as today.
- **Per-program presets** (e.g. "Apply a 5/3/1 template"). Adjacent feature, separate UI, separate decision about whether **Prog Strength** wants opinionated training templates at all.
- **Bulk import from another tracker.** Out of scope. If a user wants to seed a long list quickly, they pick from the catalog modal the same way they would for one entry.
- **Live multi-device sync.** The web client refetches when the Personal Records page mounts or the modal closes; the mobile client (when it lands) does the same. No push or subscription model.

## Implementation Details

### Data Model

A new migration `005_user_headline_exercises.sql` creates one table. The table is intentionally scoped to the existing `exercises` catalog; if cardio (running, cycling, etc.) lands later, it gets its own catalog and a sibling `user_headline_<discipline>` table — the framing word "headline" is the shared concept, the storage stays per-discipline.

| Column | Type | Description |
| --- | --- | --- |
| `user_id` | text | The owning user. Part of the composite primary key. |
| `exercise_id` | text | Exercise slug. References `exercises(id)`. Part of the composite primary key. |
| `position` | integer | 0-indexed display order. Unique per `user_id` via a separate index so the UI gets stable ordering. |
| `created_at` | datetime | Row creation timestamp. |

Composite primary key: `(user_id, exercise_id)` — a user cannot select the same exercise twice as a headline lift.

Indexes:

- One unique composite index on `(user_id, position)`. Serves two roles in one index: SQLite reuses it for the ordered read ("give me this user's selection in display order") *and* enforces slot uniqueness within a user. A second plain index would just duplicate storage. The write path always rewrites positions densely from 0 inside the same transaction that deletes the prior rows, so the uniqueness invariant is trivial to maintain.

Foreign keys:

- `exercise_id REFERENCES exercises(id)` — slug must exist in the curated catalog. No `ON DELETE` clause is specified because the catalog is admin-curated and effectively append-only; a future cleanup tool that removes a catalog row would need to NULL or delete dependent rows explicitly.

The `users` table is unchanged. No "has_customized" flag is added — the presence of any row in `user_headline_exercises` for a given `user_id` is the answer.

### Write Path

One transactional operation, triggered by `PUT /me/headline-exercises`:

- **Validate input** — non-empty list, length ≤ 12, every slug present in the exercise catalog, no duplicate slugs. Reject with `400 Bad Request` and a specific error message on any failure.
- **Begin transaction.**
- **Delete all existing rows** for `user_id`.
- **Insert the new rows** with `position` set from the request array's index. Same `created_at` for the whole set, taken once at the start of the transaction.
- **Commit.**

No partial-update semantics, no PATCH endpoint. The set-replacement pattern means clients that lose a race (two browser tabs editing simultaneously) get last-writer-wins on a coherent set rather than a half-merged hybrid — easier to reason about for a UI that already treats the modal as a single editor.

### Read Path

`GET /personal-records` (unchanged path, unchanged response shape):

1. Look up the user's headline lift selection from `user_headline_exercises` in `position` order.
2. If the result is empty, substitute `workout.HeadlineExercises` as the slug list.
3. Proceed with the existing handler logic — resolve names from the catalog, look up each PR, compute the current estimated 1RM, emit the DTO list.

`GET /me/headline-exercises` returns just the user's selection — the same slug list the modal needs to pre-populate, plus the resolved exercise display names for rendering. Falls back to `workout.HeadlineExercises` when the user has no rows, so the modal opens with the defaults pre-selected for first-time customizers.

`GET /headline-exercises/defaults` returns the contents of `workout.HeadlineExercises` plus resolved display names. Public-ish — no per-user data, but still gated by auth to match the rest of the surface.

### API Surface

Three new endpoints; one existing endpoint's behavior changes (read path only — response shape is unchanged).

- **`GET /me/headline-exercises`** (auth required) — returns the user's current selection in `position` order, falling back to defaults when the user has no rows.
  - Response: `{ "data": [{"exercise_id": "barbell-bench-press", "exercise_name": "Barbell Bench Press", "position": 0, "is_default": true}, ...] }`
  - `is_default` is a per-row flag indicating whether this slug is also in the curated default list — gives the modal a cheap way to annotate "(default)" badges without a second request.
- **`PUT /me/headline-exercises`** (auth required) — replaces the user's selection.
  - Request: `{ "exercise_ids": ["barbell-bench-press", "barbell-overhead-press", "neutral-grip-pull-up"] }`
  - Response on success: `200 OK` with the freshly-saved list in the same shape as `GET /me/headline-exercises`.
  - Validation errors return `400` with `{"error": "..."}` — specific messages for empty list, oversized list, unknown slugs, and duplicates.
- **`GET /headline-exercises/defaults`** (auth required) — returns `workout.HeadlineExercises` resolved against the catalog.
  - Response: `{ "data": [{"exercise_id": "barbell-bench-press", "exercise_name": "Barbell Bench Press"}, ...] }`
  - Auth-gated only to match the rest of the surface; nothing here is per-user.

`GET /personal-records` keeps its existing path, auth requirements, and response shape. The only change is that the slug list it iterates over is sourced per-user instead of from the global constant.

### Backfill or Migration

**Mechanism**: none. No backfill is needed. The new table starts empty, and the read path's fallback to `workout.HeadlineExercises` covers every existing user transparently. Existing users see no behavior change until they explicitly customize.

**Recoverability**: if `user_headline_exercises` is somehow corrupted or wiped, every user reverts to the global default and customization is lost. That's recoverable in the sense that the user can re-customize from the modal; the only durable damage is the inconvenience. No derived-table rebuild is possible (the user's intent is not reconstructable from any other source), so the schema does not pretend otherwise.

**Scale boundary**: with a cap of 12 rows per user and one row inserted/deleted per customize event, this table stays trivially small for a long time. The dominant read query — "rows for user X ordered by position" — hits the primary key index. No scale boundary in sight for the foreseeable future.

## Resolutions

1. **Drag-and-drop reordering vs. checkbox-only in the modal.** Shipped the lean: checkboxes only. The modal records check order as the persisted ordering (the user's act of checking a box appends to the end of the selection), and there's no drag-and-drop affordance. Adding reordering later is a frontend-only change since `position` already supports it; the API has been reorder-ready since day one.

2. **Maximum selection cap.** Shipped at 12, surfaced as the `MaxHeadlineExercises` Go constant and mirrored as `MAX_SELECTION` in the modal. Server-side validation rejects over-cap PUTs with a 400; the modal disables further checks once the cap is hit so users hit the wall in the UI rather than at submit time.

3. **`is_default` flag in the read response.** Shipped on. `GET /me/headline-exercises` returns `is_default: true` on rows whose slug is also in `workout.HeadlineExercises`, and the modal uses it to render a small "default" badge in the picker without a second round trip.

4. **Behavior when a user PUTs the default list verbatim.** Shipped option (a): the write path stores whatever the user submits — no special-case detection of "this matches defaults, fall back instead." Predictable for the user (saved lifts stay saved) and matches the way set-replacement endpoints conventionally work elsewhere in the API. A future "follow defaults" toggle could revisit this without a schema change.

5. **Catalog removal of a slug a user has selected.** Shipped the conservative default: the FK on `exercise_id REFERENCES exercises(id)` has no `ON DELETE` clause, so a catalog removal would fail until dependent rows were explicitly cleaned up. Theoretical today — the catalog is still admin-curated and effectively append-only — and we'll add an explicit cleanup path the day that changes.
