---
status: draft
depends_on:
  - sows/multi-source-agent-memory.md
  - sows/running-tracking-via-tcx-import.md
repos:
  - prog-strength-api
  - prog-strength-docs
---

# Running Session Notes

**Status**: Draft · **Last updated**: 2026-06-21

## Introduction

A **Prog Strength** runner who finishes a run and opens its detail page can read
back every number the watch recorded — pace, splits, heart rate, calories — but
there is **nowhere to say how it actually felt.** Lifts have notes (`workouts.notes`
and `workout_exercises.notes`); runs have none. The `running_sessions` table
carries an editable `name` and nothing else free-text, so the one thing a coach
most wants after a session — *"legs were dead the first two miles then it
clicked," "humid, backed off on purpose," "knee twinge at mile 4"* — has no home.
The product encourages runners to reflect on their sessions, but the running
surface gives them no place to do it.

That gap is doubly costly because the coach now has a way to *use* that text.
[Multi-Source Agent Memory](multi-source-agent-memory.md) generalized the vector-
memory distillation pipeline from chat-only into a **source-agnostic registry**,
and added workout-log notes as the second source — so a typed-out observation in a
lift log becomes durable semantic memory the coach retrieves later. Runs are the
obvious third source of exactly that kind of signal, and today every word of it is
missing because the field to hold it doesn't exist.

This work does two small, well-precedented things. First, it **gives a running
session a free-text `notes` field**, written through the same edit endpoint that
already renames a run. Second, it **registers runs as the third memory source** on
the existing registry, so a run note is distilled into the coach's memory exactly
the way a workout note is. Both halves live entirely in `prog-strength-api`; the
agent and MCP are untouched (retrieval is already origin-blind), and no web UI
ships here — the presentation of notes on the running detail page is a parked
design exploration, decided separately.

## Proposed Solution

**The notes field.** `running_sessions` gains a nullable `notes` column. The
existing `PATCH /running/{id}` endpoint — which today validates and updates `name`
— is extended to accept an optional `notes` field in the same partial-update body,
so renaming a run and annotating it share one handler and one round-trip. The
detail GET returns `notes`; the list endpoint does not (mirroring how trackpoints
are detail-only, keeping the list query lean).

**The run-note memory source.** A new `RunNoteSource` implements the
`MemorySource` interface introduced by Multi-Source Agent Memory and is registered
third: `[chatSource, workoutNoteSource, runNoteSource]`. Its unit of distillation
is **one running session**: a session with a non-empty note is eligible once it has
**settled** (no edit for a configurable window), is handed as one `DistillUnit` to
the single shared Haiku distiller with a run-specific `PromptHint`, and is stamped
done so it is never re-examined. Note-less runs are never distilled. The job, the
distiller, the dedup probe, the embedding step, and retrieval are all reused
unchanged — this is a new adapter, not new pipeline.

**Settle signal.** The settle window needs a timestamp that bumps when the *note*
is written, and `running_sessions` has only `created_at` (import time, before any
note exists). The migration therefore adds `updated_at`, bumped on every PATCH, and
the source settles on `updated_at < now - run_settle_minutes`.

**Typed provenance.** Following the typed-FK choice already set by Multi-Source
Agent Memory (per-source nullable FK columns under a `source_type` discriminator
with a `CHECK` constraint, not a polymorphic reference), `agent_memories` gains a
third case: `source_type = 'run_note'` with a `source_running_session_id` FK to
`running_sessions(id) ON DELETE CASCADE`. A run's distilled memories die with the
run, exactly as chat and workout memories die with theirs.

**Backfill.** The source-aware `cmd/memory-backfill` command already ranges the
registry; the run-note source drops in and its historical notes are seeded via the
same batch APIs, idempotently. In practice the run-note backfill is near-empty on
first ship (the field is new, so the only notes that exist are any written between
this SOW landing and the backfill running) — but implementing `AllUndistilled` is
free given the interface, and keeps every source uniformly backfillable.

**What does not change.** The read path (`POST /internal/memory/retrieve`), the
embedding model, the `vec_agent_memories` table, the per-user threshold-gated KNN
search, the distiller, and the `[vectormemory]` config block are all source-
agnostic and carry over untouched. **`prog-strength-agent` and `prog-strength-mcp`
are not touched** — the agent retrieves memories without regard to origin, and
structured run data (distance, pace, splits) was never the unstructured-signal
path. A run note flows to the coach the moment it is distilled, with zero read-side
work.

## Goals and Non-Goals

### Goals

- Add a nullable `notes TEXT` column to `running_sessions`, returned by the detail
  GET and omitted from the list response.
- Extend `PATCH /running/{id}` to accept an optional `notes` field (length-capped),
  alongside the existing `name` edit, as one partial-update handler.
- Add an `updated_at DATETIME` column to `running_sessions`, bumped on every PATCH,
  to give the memory source a settle signal (the run's `created_at` is import time,
  not note-write time).
- Add a **`RunNoteSource`** implementing `MemorySource`: one running session = one
  distillation unit, eligible once its `notes` is non-empty and `updated_at` has
  settled past a configurable window, scoped per user and excluding soft-deleted
  runs.
- Migration `03N_run_note_memory.sql`: add `notes`, `updated_at`, and
  `memory_distilled_at` to `running_sessions`; rebuild `agent_memories` to add the
  third provenance case (`source_type = 'run_note'` + `source_running_session_id`
  FK → `running_sessions(id) ON DELETE CASCADE`), widening the `CHECK` constraint.
- A `run_settle_minutes` knob in the `[vectormemory]` config block (starting point
  ~10), separate from chat's `session_idle_minutes` and workouts'
  `workout_settle_minutes`.
- A run-specific distiller `PromptHint` framing the content as a terse post-run
  training note, sharing the single Haiku `Distiller`.
- Extend `cmd/memory-backfill` to drain the run-note source (free given the
  registry already ranges sources), idempotently.
- The admin inspection dump surfaces `source_type = 'run_note'` so distillation
  quality can be eyeballed per source.

### Non-Goals

- **Any web UI.** No notes editor, no presentation of notes on the running detail
  page. How notes (and the parked HR-zones and elevation widgets) render is a
  separate design exploration; this SOW ships only the field, the API, and the
  ingestion. `prog-strength-web` is out of scope and not in `repos`.
- **The SDK.** `prog-strength-sdk` is not updated here; the web client gains its
  `notes` accessor in the downstream presentation SOW that consumes the parked DX.
- **Touching the agent or MCP.** Retrieval is origin-blind and unchanged;
  structured run-data recall is not in question. `prog-strength-agent` and
  `prog-strength-mcp` are untouched.
- **Generalizing the memory pipeline.** That work is done — Multi-Source Agent
  Memory built the `MemorySource` interface, registry, typed-provenance pattern,
  and source-aware backfill. This SOW only adds an implementer. It **depends on**
  that SOW being in place (or landing first).
- **Re-distilling edited notes.** A run stamped `memory_distilled_at` is not re-
  distilled if its note is later edited — matching chat and workout notes. Flagged
  in Open Questions.
- **Per-trackpoint or structured run annotations.** The single free-text `notes`
  field is the scope; no per-split or per-segment note surfaces are introduced.
- **Notes on walking/cycling/other activities.** The column lives on
  `running_sessions`, which today holds running activities only; other disciplines
  are a later concern if and when they get their own session tables.

## Implementation Details

All changes are in `prog-strength-api`. The sections follow the lifecycle:
data → write → source → backfill → config → tests → rollout.

### Data Model

One migration, `internal/db/migrations/03N_run_note_memory.sql` (use the next free
number at implementation time — this lands **after** Multi-Source Agent Memory's
`035_multi_source_memory.sql`, so `036+`; rebase onto the latest ledger entry
before implementing).

**`running_sessions` gains three columns:**

| Column | Type | Description |
| --- | --- | --- |
| `notes` | TEXT | Nullable. User-authored free-text reflection on the run. Edited via PATCH. |
| `updated_at` | DATETIME | Bumped on every PATCH (name or notes). Backfilled to `created_at` for existing rows. Drives the memory source's settle window. |
| `memory_distilled_at` | DATETIME | NULL = not yet distilled. Set by the run-note source's `MarkDistilled`. |

```sql
ALTER TABLE running_sessions ADD COLUMN notes TEXT;
ALTER TABLE running_sessions ADD COLUMN updated_at DATETIME;
ALTER TABLE running_sessions ADD COLUMN memory_distilled_at DATETIME;  -- NULL = not yet distilled
UPDATE running_sessions SET updated_at = created_at WHERE updated_at IS NULL;
```

**`agent_memories` gains the third provenance case.** Multi-Source Agent Memory's
`035` rebuilt this table with a `source_type` discriminator and a `CHECK` covering
`chat_session` and `workout_note`. SQLite cannot widen a table-level `CHECK` in
place, so the table is rebuilt-and-copied again (trivial at current volume) to add
the run case. After `035`, the table is:

```sql
-- post-035 shape (chat_session | workout_note)
CREATE TABLE agent_memories (
    id                INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id           TEXT NOT NULL,
    distilled_text    TEXT NOT NULL,
    source_type       TEXT NOT NULL,
    source_session_id TEXT REFERENCES chat_sessions(id) ON DELETE CASCADE,
    source_message_id INTEGER REFERENCES chat_messages(id) ON DELETE SET NULL,
    source_workout_id TEXT REFERENCES workouts(id) ON DELETE CASCADE,
    embedding_model   TEXT NOT NULL,
    embedding_dim     INTEGER NOT NULL,
    superseded_at     DATETIME,
    created_at        DATETIME NOT NULL,
    CHECK ( ... chat_session / workout_note cases ... )
);
```

becomes:

```sql
CREATE TABLE agent_memories_new (
    id                       INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id                  TEXT NOT NULL,
    distilled_text           TEXT NOT NULL,
    source_type              TEXT NOT NULL,           -- 'chat_session' | 'workout_note' | 'run_note'
    source_session_id        TEXT REFERENCES chat_sessions(id) ON DELETE CASCADE,
    source_message_id        INTEGER REFERENCES chat_messages(id) ON DELETE SET NULL,
    source_workout_id        TEXT REFERENCES workouts(id) ON DELETE CASCADE,
    source_running_session_id TEXT REFERENCES running_sessions(id) ON DELETE CASCADE,  -- new
    embedding_model          TEXT NOT NULL,
    embedding_dim            INTEGER NOT NULL,
    superseded_at            DATETIME,
    created_at               DATETIME NOT NULL,
    CHECK (
        (source_type = 'chat_session' AND source_session_id IS NOT NULL AND source_workout_id IS NULL AND source_running_session_id IS NULL) OR
        (source_type = 'workout_note' AND source_workout_id IS NOT NULL AND source_session_id IS NULL AND source_running_session_id IS NULL) OR
        (source_type = 'run_note'     AND source_running_session_id IS NOT NULL AND source_session_id IS NULL AND source_workout_id IS NULL)
    )
);
-- copy existing rows verbatim (their source_type and FKs are already correct from 035):
INSERT INTO agent_memories_new (id, user_id, distilled_text, source_type,
    source_session_id, source_message_id, source_workout_id, source_running_session_id,
    embedding_model, embedding_dim, superseded_at, created_at)
SELECT id, user_id, distilled_text, source_type,
    source_session_id, source_message_id, source_workout_id, NULL,
    embedding_model, embedding_dim, superseded_at, created_at
FROM agent_memories;
DROP TABLE agent_memories;
ALTER TABLE agent_memories_new RENAME TO agent_memories;
-- recreate the indexes from 035 (user_id; source_session_id; source_workout_id)
-- and add one for source_running_session_id.
```

The `id` values are preserved by the copy, so the `memory_id` join to the untouched
`vec_agent_memories` virtual table stays valid — no vector rows are rewritten,
exactly as in `035`.

### Write Path

- **PATCH `/running/{id}`** — the existing handler edits `name`; it is extended to a
  partial update accepting either or both of `name` and `notes`. `name` keeps its
  current rule (non-blank, ≤ 200 chars). `notes` is optional, trimmed, capped at
  **2000 chars**, and may be set to empty/null to clear it. Every successful PATCH
  also sets `updated_at = now`. A single `UPDATE` on `running_sessions`; no S3, no
  trackpoint touch. Unchanged-field semantics: an absent key leaves that column
  alone (PATCH, not PUT).
- **DELETE** — unchanged (`UPDATE ... SET deleted_at`). A soft-deleted run is
  excluded from the source's eligibility query, and the `ON DELETE CASCADE` on
  `source_running_session_id` means a hard delete (should one ever occur) cascades
  its memories away.
- **Settle invariant** — because the source keys off `updated_at`, a note that is
  still being edited keeps bumping `updated_at` and stays out of the eligibility
  window until the runner stops typing for `run_settle_minutes`.

### API Surface

| Endpoint | Change |
| --- | --- |
| `GET /running/{id}` | Detail DTO gains `notes` (string or null). |
| `GET /running` (list) | **Unchanged** — `notes` is detail-only, like trackpoints. |
| `PATCH /running/{id}` | Request body accepts optional `notes` (≤ 2000 chars, nullable) in addition to `name`. Response is the updated detail DTO including `notes`. |

Auth is the existing per-user ownership check on the running domain; no new
endpoints, no new auth surface.

### The Run-Note Source

A new adapter over the running repository, alongside the chat and workout adapters
in `internal/server/vectormemory.go` (or a sibling file), implementing
`MemorySource`.

- **`SourceType()`** → `"run_note"`.

- **`PendingUnits`** selects eligible runs:

  ```sql
  SELECT id, user_id, name, notes, start_time, distance_meters
  FROM running_sessions
  WHERE deleted_at IS NULL
    AND memory_distilled_at IS NULL
    AND notes IS NOT NULL AND TRIM(notes) <> ''
    AND updated_at < ?                       -- now - run_settle_minutes
  ORDER BY updated_at ASC
  LIMIT ?;
  ```

  For each row it assembles `DistillUnit.Content` as the note framed by light
  context — a leading line identifying the run (its `name`, date, and distance) so
  an observation like *"felt flat the whole way"* is distilled knowing it was a
  5-mile run on a given day — followed by the note body. `PromptHint` frames it as a
  terse post-run training note (extract durable, stable observations about the
  runner — recurring niggles, conditions that recur, how training is trending — not
  one-off race-day minutiae). `Provenance{SourceType:"run_note", RunningSessionID:&id}`.

- **`AllUndistilled`** is the same query without the `updated_at` settle clause,
  cursor-paginated on `(updated_at, id)`, for the backfill.

- **`MarkDistilled`** → `UPDATE running_sessions SET memory_distilled_at = ? WHERE id = ?`.

The registry built in `server.New` becomes
`[]MemorySource{chatSource, workoutNoteSource, runNoteSource}`. The
`Provenance` struct from Multi-Source Agent Memory gains a `RunningSessionID *string`
field (set iff `SourceType == "run_note"`), and the repository `Insert` writes
`source_running_session_id` when that provenance is present (the other typed FKs
`NULL`). No change to the dedup probe, KNN search, supersede logic, or the job loop
itself — it already ranges whatever registry it is handed.

### Configuration

The `[vectormemory]` block gains one knob; everything else is reused:

| Key | Default (starting point) | Meaning |
| --- | --- | --- |
| `run_settle_minutes` | `10` | A run whose `notes` has not changed for this long is eligible for distillation. Separate from `session_idle_minutes` (chat) and `workout_settle_minutes` because a run note is typically a single short write that settles quickly. |

No new secrets — the existing `OPENAI_API_KEY` / `ANTHROPIC_API_KEY` cover the new
source, and the `enabled` kill-switch continues to gate the whole feature including
this source.

### Backfill

`cmd/memory-backfill` already ranges the source registry, so the run-note source is
picked up with no command change beyond registering it. It drains the source's
`AllUndistilled` in pages, distilling via the Anthropic Message Batches API and
embedding via the OpenAI Batch embeddings API, inserting with the same dedup check
as the live path and stamping `memory_distilled_at` per run. Idempotent and
resumable: a re-run skips anything already stamped.

- **Mechanism** — the existing source-aware one-shot CLI; no new command.
- **Recoverability** — re-runnable; only un-stamped runs are processed, so a partial
  failure is recovered by re-invoking.
- **Scale boundary** — the field is new, so the historical run-note corpus is
  effectively empty at first ship; the batch path comfortably handles the corpus at
  any realistic volume it later reaches, identical to the workout-note backfill.

### Tests

| Repo | Coverage |
| --- | --- |
| `prog-strength-api` | **Migration:** adds `notes` / `updated_at` / `memory_distilled_at` to `running_sessions` and backfills `updated_at = created_at`; rebuilds `agent_memories` preserving `id` values and existing rows; the widened `CHECK` accepts a `run_note` row with `source_running_session_id` set and rejects one whose FK doesn't match its `source_type`. **PATCH:** sets `notes` (and bumps `updated_at`); enforces the 2000-char cap; clears `notes` on empty; leaves `name` untouched when only `notes` is sent (and vice versa); a non-owner is rejected. **Detail/list:** detail GET returns `notes`; list omits it. **Run source:** `PendingUnits` selects only settled, undistilled, non-deleted runs with a non-empty note; ignores note-less and still-settling runs; composes content with the contextual header; `MarkDistilled` sets the column; `AllUndistilled` ignores the settle window and paginates. **Repository:** `Insert` writes `source_running_session_id` + `source_type='run_note'` per provenance; cross-source dedup skips a run-note observation near-identical to an existing chat/workout memory. **Job/backfill:** the job ranges three sources in one tick and a zero-observation run is still marked; backfill drains the run source and an idempotent re-run is a no-op. **Cascade:** deleting a `running_sessions` row cascades its `agent_memories`. **Retrieval unchanged:** existing retrieval/threshold tests pass with run-sourced rows present. Providers faked; no live API calls. |
| `prog-strength-docs` | This SOW; status transitions. |

Follow existing conventions: stdlib `testing`, `httptest`, per-test temp `app.db`
via `dbtest`, fakes for external providers.

### Rollout

1. **Migration + notes field + PATCH**, behind no new flag (the field and endpoint
   are inert until something writes a note). Confirm the running domain's existing
   tests and the name-edit path are unregressed.
2. **Run-note source live** under the existing `vectormemory.enabled` flag. Watch
   the admin inspection dump (`source_type='run_note'`) and iterate the run
   `PromptHint` against real notes before relying on the output — the field being
   new means the first real notes appear only after step 1 ships.
3. **Backfill** any run notes written between ship and now via the batch APIs once
   the distiller prompt looks right. Idempotent, so re-runnable after a prompt tweak.
4. Retrieval needs no rollout step — it returns the new memories the moment they
   exist, gated by the same threshold tuned for the shipped feature.

> The downstream **running-detail presentation SOW** (consuming the parked DX for
> notes, HR-zones, and elevation) is what gives runners a UI to write notes through.
> Until it ships, notes are writable only via the raw PATCH endpoint — acceptable,
> since this SOW's job is the data + ingestion seam, not the surface.

## Open Questions

1. **Re-distilling edited notes.** A run is distilled once and not revisited if its
   note is later edited (matches chat and workouts). If runners meaningfully revise
   run notes after the settle window, consider resetting `memory_distilled_at` on
   note edit — at the cost of possible duplicate/superseding churn. Defer until
   real usage shows it matters.
2. **Contextual header content.** How much run context to prepend to the note
   (name? date? distance? linked plan's `run_type`?) should be iterated against real
   notes via the inspection dump. The starting format — name + date + distance —
   deliberately avoids a join to planned workouts; `run_type` is added only if
   distillation quality clearly wants it.
3. **`run_settle_minutes` default.** 10 minutes assumes a run note is a single short
   write. Validate against how runners actually annotate (someone who jots a note,
   navigates away, and returns to add more would keep bumping `updated_at`).
4. **Shared `updated_at` semantics.** `updated_at` bumps on any PATCH, including a
   pure rename with no note. That merely delays an existing note's distillation
   slightly (harmless), but confirm during implementation that the name-edit path
   and any future writer set `updated_at` consistently so the settle signal stays
   honest.
5. **Notes for non-running disciplines.** The field is on `running_sessions` only.
   If walking/cycling sessions later get their own tables, each would need its own
   notes column and source adapter — uniform with this pattern, but out of scope
   now.
