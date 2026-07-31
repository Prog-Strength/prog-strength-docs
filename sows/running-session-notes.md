---
status: in_review
depends_on:
  - sows/multi-source-agent-memory.md
  - sows/unified-activity-model.md
  - sows/run-detail-session-recap-parity.md
repos:
  - prog-strength-api
  - prog-strength-docs
---

# Activity Session Notes

**Status**: In Review · **Last updated**: 2026-07-30

> Filed as "Running Session Notes" on 2026-06-21 and kept at that path so the
> DX and plans that link here don't break. The scope widened when the unified
> activity model landed: notes are a base-row column now, so this SOW covers
> every endurance session — runs, hikes, walks, rides — not runs alone.

## Introduction

A **Prog Strength** runner who finishes a run and opens its detail page can read
back every number the watch recorded — pace, splits, heart rate, calories — but
there is **nowhere to say how it actually felt.** Lifts have notes; endurance
sessions have none. The one thing a coach most wants after a session —
*"legs were dead the first two miles then it clicked," "humid, backed off on
purpose," "knee twinge at mile 4"* — has no home.

That gap is doubly costly because the coach now has a way to *use* that text.
[Multi-Source Agent Memory](multi-source-agent-memory.md) generalized the vector-
memory distillation pipeline from chat-only into a **source-agnostic registry**,
and added workout-log notes as the second source — so a typed-out observation in a
lift log becomes durable semantic memory the coach retrieves later. Endurance
sessions are the obvious third source of exactly that kind of signal.

**What changed since this SOW was drafted.** Two things, and they pull in
opposite directions:

1. [Unified Activity Model](unified-activity-model.md) collapsed
   `running_sessions` and `workouts` into one `activities` base table — which
   already carries `notes`, `updated_at`, and `memory_distilled_at` on every
   row, for every session type. **Most of this SOW's original data model
   shipped as a side effect of that migration.** The endpoints it described
   (`PATCH /running/{id}`) no longer exist.
2. [Run Detail Session Recap Parity](run-detail-session-recap-parity.md)
   shipped the notes **UI** in `prog-strength-web` — a click-to-edit
   `NotesEditor` on the run and hike detail pages — against the write endpoint
   this SOW was supposed to deliver. That endpoint was never built, so **the
   editor is live in production and every save fails** with
   `400 name or environment is required`. This is no longer additive work; it
   is a user-visible break.

## Proposed Solution

**The write path.** `PATCH /activities/{id}` — which today accepts `name` and
`environment` — also accepts an optional `notes` field in the same
partial-update body. That is the exact request the shipped web client already
sends, so no client change is needed anywhere. Notes are a base-row column, so
one handler serves every activity type: the hike page's editor starts working
at the same moment the run page's does.

`PUT /activities/{id}` is deliberately **not** the answer even though it already
accepts `notes`: it is a full replacement that routes through the type's
`DetailStore`, so a note-only PUT would delete the session's detail row —
distance, elevation, calibration, and the GPS route with it.

**The activity-note memory source.** A new `activityNoteSource` implements the
`MemorySource` interface and is registered third:
`[chatSource, workoutNoteSource, activityNoteSource]`. Its unit of distillation
is **one session**: a non-strength activity with a non-empty note is eligible
once it has **settled** (no edit for a configurable window), is handed as one
`DistillUnit` to the single shared Haiku distiller with an endurance-specific
`PromptHint`, and is stamped done so it is never re-examined. Note-less sessions
are never distilled, so the pipeline spends nothing on the common case. The job,
the distiller, the dedup probe, the embedding step, and retrieval are all reused
unchanged — this is a new adapter, not new pipeline.

The two note sources **partition** the `activities` table by type: strength
belongs to `workoutNoteSource` (which fuses in per-exercise notes), everything
else to `activityNoteSource`. No session is ever eligible for both.

**Typed provenance.** `agent_memories.source_workout_id` was re-pointed at
`activities(id)` by the unification migration, and since unification that table
holds every session type — so an endurance note's provenance FK is the same
column a lift note's is. The only schema change needed is widening the
discriminator `CHECK` to admit a third `source_type`, `'activity_note'`.

**One source type for every endurance sport**, not `run_note` / `hike_note` /
`ride_note`: the sport is recoverable by joining `activities.activity_type`, so
denormalizing it into the discriminator would only buy a table rebuild per
future activity type — precisely the churn the Go type registry exists to
avoid. "Which hike notes produced memories?" is a join, not a schema decision.

**What does not change.** The read path (`POST /internal/memory/retrieve`), the
embedding model, the `vec_agent_memories` table, the per-user threshold-gated KNN
search, the distiller, and the `[vectormemory]` config block are all source-
agnostic and carry over untouched. **`prog-strength-agent` and `prog-strength-mcp`
are not touched** — the agent retrieves memories without regard to origin, and
structured run data (distance, pace, splits) was never the unstructured-signal
path. A session note flows to the coach the moment it is distilled, with zero
read-side work.

## Goals and Non-Goals

### Goals

- Extend `PATCH /activities/{id}` to accept an optional `notes` field
  (trimmed, ≤ 2000 chars, empty string clears to NULL) alongside the existing
  `name` and `environment` edits, as one partial-update handler. Every
  successful write bumps `updated_at`.
- Add `Repository.UpdateNotes`, mirroring `Rename`: ownership-scoped, live-rows-
  only, re-reads so the response reflects exactly what persisted.
- Unblock the already-shipped web editor on **both** the run and hike detail
  pages with no client change.
- Add an **`activityNoteSource`** implementing `MemorySource`: one non-strength
  activity = one distillation unit, eligible once its `notes` is non-empty and
  `updated_at` has settled past a configurable window, scoped per user and
  excluding soft-deleted sessions.
- Migration `045_activity_note_memory.sql`: rebuild `agent_memories` to widen
  the `CHECK` to `source_type IN ('workout_note', 'activity_note')` on the
  shared FK, preserving `id` values so the `vec_agent_memories` join survives.
- An `activity_settle_minutes` knob in the `[vectormemory]` config block
  (starting point 10), separate from chat's `session_idle_minutes` and
  strength's `workout_settle_minutes`.
- An endurance-specific distiller `PromptHint` framing the content as a terse
  post-session training note, sharing the single Haiku `Distiller`.
- A context header per unit — sport, the author's **local** session date, name,
  distance, duration — so "felt flat the whole way" is distilled knowing what it
  was attached to.
- Extend `cmd/memory-backfill` to drain the new source (free given the registry
  already ranges sources), idempotently.

### Non-Goals

- **Any web or mobile UI.** Web already shipped its editor
  ([run-detail-session-recap-parity](run-detail-session-recap-parity.md)); this
  SOW makes it work. Mobile has no notes UI on endurance sessions and does not
  gain one here.
- **Surfacing provenance at retrieval time.** `Match` returns text, distance,
  and created_at — the coach recalls a fact without knowing it came from a run
  note versus a chat. Deliberate, inherited from Multi-Source Agent Memory, and
  independently changeable later.
- **Renaming `source_workout_id`.** The column name predates unification and now
  holds any activity id. Renaming it would touch the admin dump's JSON contract
  and ~10 files for no functional gain; the field is documented instead.
- **Generalizing the memory pipeline.** That work is done — this SOW only adds
  an implementer.
- **Re-distilling edited notes.** A session stamped `memory_distilled_at` is not
  re-distilled if its note is later edited — matching chat and workout notes.
  Flagged in Open Questions.
- **Per-trackpoint or structured annotations.** The single free-text `notes`
  field is the scope; no per-split or per-segment note surfaces.
- **Strength notes.** Already handled by `workoutNoteSource`, and deliberately
  left there so per-exercise notes keep being fused into one lift unit.

## Implementation Details

All changes are in `prog-strength-api`.

### Data Model

**No new columns.** `activities` already carries `notes`, `updated_at`, and
`memory_distilled_at` for every type (migration 042). One migration,
`internal/db/migrations/045_activity_note_memory.sql`, widens the
`agent_memories` discriminator:

```sql
CHECK (
    (source_type = 'chat_session'  AND source_session_id IS NOT NULL AND source_workout_id IS NULL) OR
    (source_type IN ('workout_note', 'activity_note') AND source_workout_id IS NOT NULL AND source_session_id IS NULL)
)
```

SQLite cannot widen a table-level `CHECK` in place, so the table is
rebuilt-and-copied. Rows copy verbatim (their `source_type` and FKs are already
correct), `id` values are preserved so the `memory_id` join to the untouched
`vec_agent_memories` virtual table stays valid, and the three indexes are
recreated — the same shape as the rebuilds in 035 and 042.

### Write Path

- **PATCH `/activities/{id}`** — the handler accepts any combination of `name`,
  `notes`, and `environment`; an all-absent body is still a 400 (with the error
  text widened to name the third field). `name` keeps its rule (non-blank,
  ≤ 200 chars). `notes` is trimmed, capped at **2000 chars**, and an empty
  string clears it — a blank note is legitimate input, unlike a blank name.
  A single `UPDATE`, `NULLIF`-ing the empty string to NULL; no S3, no
  trackpoint touch. Absent keys leave their columns alone (PATCH, not PUT).
- **Response shape** — the base DTO, without the detail-only derived blocks
  (splits, strip summary, zones, route). The web client merges it over its
  loaded session, so the omitted keys are preserved client-side rather than
  clobbered.
- **Settle invariant** — because the source keys off `updated_at`, a note still
  being edited keeps bumping `updated_at` and stays out of the eligibility
  window until the athlete stops typing for `activity_settle_minutes`.

### API Surface

| Endpoint | Change |
| --- | --- |
| `GET /activities/{id}` | **Unchanged** — the detail DTO already carries `notes`. |
| `GET /activities` (list) | **Unchanged** — list items already carry `notes`. |
| `PATCH /activities/{id}` | Body accepts optional `notes` (≤ 2000 chars, `""` clears) in addition to `name` / `environment`. |

Auth is the existing per-user ownership check; no new endpoints, no new auth
surface, no MCP or agent change.

### The Activity-Note Source

A new adapter in `internal/server/activity_note_source.go`, alongside the chat
and workout adapters, implementing `MemorySource`.

- **`SourceType()`** → `"activity_note"`.
- **`PendingUnits`** selects eligible sessions from base columns alone:

  ```sql
  SELECT a.id, a.user_id, COALESCE(u.timezone, 'UTC')
  FROM activities a
  JOIN users u ON u.id = a.user_id
  WHERE a.activity_type <> 'strength_training'
    AND a.deleted_at IS NULL
    AND a.memory_distilled_at IS NULL
    AND a.notes IS NOT NULL AND TRIM(a.notes) <> ''
    AND a.updated_at < ?                      -- now - activity_settle_minutes
  ORDER BY a.updated_at ASC
  LIMIT ?;
  ```

  Each unit's `Content` is a one-line context header plus the note body:

  ```
  running · 2026-07-21 · Morning Run · 5.0 mi · 41:12
  Notes: legs were dead the first two miles then it clicked
  ```

  The date is the author's **local** calendar date — a 9pm run stored in UTC
  would otherwise be attributed to the following day, which matters when a
  memory says "always struggles on Mondays." Distance and duration are read back
  through the activity repository's canonical projection rather than a
  hand-rolled join, so a sixth detail table needs no change here. `PromptHint`
  frames the note as terse post-session shorthand and explicitly tells the
  distiller **not** to restate distance/pace/duration — those are already
  structured data the coach reads directly.
- **`CountPending`** is the same predicate uncapped, feeding the existing idle
  backlog gauge.
- **`AllUndistilled`** drops the settle clause and keyset-paginates on
  `(updated_at, id)`, sharing the workout source's opaque base64 cursor codec.
- **`MarkDistilled`** stamps `activities.memory_distilled_at`.

`Provenance{SourceType: "activity_note", WorkoutID: &id}` — the same typed FK
the workout source writes, on a row whose `activity_type` names the sport.

### Configuration

The `[vectormemory]` block gains one knob; everything else is reused:

| Key | Default | Meaning |
| --- | --- | --- |
| `activity_settle_minutes` | `10` | An endurance session whose `notes` hasn't changed for this long is eligible for distillation. Its own knob because a session note is usually one short write typed straight afterwards, so it settles faster than a chat or a lift log. |

No new secrets, and the existing `enabled` kill-switch continues to gate the
whole feature including this source.

### Backfill

`cmd/memory-backfill` already ranges the source registry, so the new source is
picked up by registration alone. In practice its corpus is empty at first ship
(the editor has never successfully written a note), so the backfill matters for
notes written between shipping and running it. Idempotent and resumable.

### Tests

| Repo | Coverage |
| --- | --- |
| `prog-strength-api` | **PATCH:** a notes-only body is a complete patch; the note round-trips; `""` clears to SQL NULL; the 2000-char cap is enforced at the boundary (2000 ok, 2001 rejected); a note edit leaves `name` alone and vice versa; a non-owner gets 404. **Repository:** `UpdateNotes` round-trips, stores NULL for empty, bumps `updated_at`, and is ownership-scoped. **Migration:** existing chat + workout rows survive with their ids (vec join intact); the widened CHECK accepts `activity_note` on the shared FK, rejects it with a NULL FK, rejects it with a chat FK also set, and still rejects an unknown source type; an activity-note memory cascades with its activity; `foreign_key_check` clean; the migrated DDL is byte-identical to a fresh database's. **Source:** `PendingUnits` selects only settled, undistilled, live, noted, non-strength sessions — excluding strength (owned by the workout source), note-less, still-settling, soft-deleted, and already-distilled rows; the content header carries sport, local date, name, distance, duration; provenance is `activity_note` + the activity id with the chat FK nil; `CountPending` is uncapped where `PendingUnits` is capped; `MarkDistilled` stamps the column; `AllUndistilled` ignores the settle window and paginates in `updated_at` order. Providers faked; no live API calls. |
| `prog-strength-docs` | This SOW; status transitions. |

Verified additionally by a live smoke against a real server on a fresh
migrated database: migration 045 applies, a note-only PATCH returns 200 with
the note, the detail GET reads it back, `""` clears it, an over-cap note is
400, and the PATCH response carries the base fields (distance, duration) the
web client merges over its loaded session.

### Rollout

1. **Migration + PATCH `notes`.** This alone repairs the production break — the
   shipped web editor starts saving on both the run and hike pages the moment
   the API deploys. No client release required.
2. **Activity-note source live** under the existing `vectormemory.enabled`
   flag. Watch the admin inspection dump (`source_type='activity_note'`) and
   iterate the `PromptHint` against real notes before relying on the output.
3. **Backfill** any notes written between step 1 and step 2 via the batch APIs
   once the distiller prompt looks right. Idempotent, so re-runnable after a
   prompt tweak.
4. Retrieval needs no rollout step — it returns the new memories the moment they
   exist, gated by the same threshold tuned for the shipped feature.

## Open Questions

1. **Re-distilling edited notes.** A session is distilled once and not revisited
   if its note is later edited (matches chat and workouts). If athletes
   meaningfully revise notes after the settle window, consider resetting
   `memory_distilled_at` on note edit — at the cost of possible
   duplicate/superseding churn. Defer until real usage shows it matters.
2. **Contextual header content.** How much context to prepend (name? date?
   distance? the linked plan's `run_type`?) should be iterated against real notes
   via the inspection dump. The starting format deliberately avoids a join to
   planned workouts.
3. **`activity_settle_minutes` default.** 10 minutes assumes a note is a single
   short write. Someone who jots a note, navigates away, and returns to add more
   keeps bumping `updated_at` — harmless, but worth watching.
4. **A cap on note length vs. distillation cost.** 2000 chars is one cheap Haiku
   unit today. If notes get long enough to matter, the cap is the lever.
5. **Origin-blind retrieval.** The coach can't currently say "you mentioned after
   a run last month…" because `Match` carries no `source_type`. Adding it is a
   contained change (retrieval query + the agent-facing payload) if the recall
   experience wants the attribution.
