---
status: draft
depends_on:
  - sows/agent-vector-memory.md
  - sows/centralized-api-config.md
repos:
  - prog-strength-api
  - prog-strength-docs
---

# Multi-Source Agent Memory

**Status**: Draft · **Last updated**: 2026-06-21

## Introduction

The shipped [Agent Vector Memory](agent-vector-memory.md) feature gives the **Prog Strength** coach semantic memory over the user — but only from **one source of signal: chat conversations**. A background job distills durable observations from idle chat sessions, embeds them, and retrieves the relevant ones at request time. That works, but it leaves a large body of equally durable, equally unstructured signal on the floor: **the notes the user writes in their workout logs.** Users routinely record context there that never becomes a structured row — "left shoulder cranky again, dropped to 3×5," "travelling this week, hotel gym only," "knee felt great after the new warm-up," "cutting hard before the wedding." Today that text is written once and only ever read back inside the workout-history UI; the coach never sees it.

This work does two things. First, it **generalizes the distillation pipeline** from a chat-specific job into a source-agnostic one that can ingest unstructured signal from *any* registered source through one well-defined interface. Second, it **adds workout-log notes as the second source** on top of that generalized seam, including a backfill of the existing note history so the feature is useful on day one rather than only for notes written after it ships.

The generalization is the point, not an accident of adding one source. The user intends to add further sources over time (the design names none specifically, but the seam is built so that each future source is a small adapter, not a pipeline rewrite). The retrieval and storage halves of the existing feature are already source-agnostic and are **left almost entirely untouched** — a vector is a vector regardless of where its text came from. The change is concentrated in the **write path**: what gets distilled, from where, and how each origin is tracked and traced.

This stays inside the boundaries the original feature set. It is still **not** a second structured-recall system — workout *sets, weights, and dates* remain the domain of the existing MCP tools over `app.db`; only the free-text **notes** attached to workouts are treated as unstructured signal. It remains **boring**: no new datastore, no new container, no new service. New memories live in the same `app.db` tables via `sqlite-vec`, the Go API remains the sole reader and writer of vectors, and everything is covered by the existing Litestream→S3 replication.

## Proposed Solution

The existing distillation job in `prog-strength-api`'s `vectormemory` domain is hardwired to chat through a `SessionSource` interface (`IdleUndistilled` / `Conversation` / `MarkDistilled`) whose every method is shaped around a chat session. This work replaces that with a generic **`MemorySource`** interface and a small **registry** of sources that the job ranges over. Chat becomes the first implementer of that interface — its current logic moves behind it unchanged — and a new **workout-note source** becomes the second.

**The source seam.** A `MemorySource` knows how to do three things: yield units of content that have *settled* (gone quiet past that source's own window) and have not yet been distilled; yield *all* historical undistilled units (for the one-time backfill); and *mark* a unit done so it isn't re-examined. A unit carries the text to distill, the owning `user_id`, a per-source framing hint for the distiller, and typed provenance identifying which origin record it came from. The distillation job becomes a loop over registered sources — for each, pull settled units, distill each via one Haiku call, dedup + embed + store — with no knowledge of what a "chat session" or a "workout" is.

**The workout-note source.** Its unit of distillation is **one workout**. The Prog Strength schema stores user-written notes in two places — `workouts.notes` (one per workout) and `workout_exercises.notes` (per exercise within a workout) — and the source composes both into a single lightly-structured text blob per workout, handed to one Haiku call. This gives one Haiku call per *noted* workout (workouts with no notes are never distilled), clean provenance back to a single workout row, and a distiller that sees a note in the context of the exercise it was about. A workout is eligible once it has settled — `workouts.updated_at` older than a configurable window — so a log that's still being edited isn't distilled mid-write.

**Typed provenance.** The existing `agent_memories` table ties each memory to its origin with a hard, chat-specific foreign key (`source_session_id`). This work keeps **typed foreign keys per source** rather than a polymorphic reference, preserving DB-level referential integrity and per-source `ON DELETE CASCADE`: a `source_type` discriminator column says which origin a memory came from, and the matching nullable FK column (`source_session_id` for chat, `source_workout_id` for workout notes) points at the exact record, enforced by a `CHECK` constraint. A memory still dies with its source — when a workout is deleted, its distilled memories cascade away, exactly as a chat session's do today.

**Backfill.** Unlike chat (which launched against a near-empty history), workout notes already exist in volume in `app.db`. The original one-time backfill command is extended to be **source-aware**: it iterates the same source registry and, per source, distills *all* historical undistilled units using the cheaper async Anthropic Message Batches API and OpenAI Batch embeddings API. One backfill run seeds both chat history and the entire existing workout-note corpus, idempotently — re-running skips anything already marked distilled.

**What does not change.** The read path (`POST /internal/memory/retrieve`), the embedding model and `vec_agent_memories` table, the per-user threshold-gated KNN search, the agent's parallel best-effort retrieval call, the admin inspection/probe tooling, and the `[vectormemory]` config block are all source-agnostic and carry over unchanged. Cross-source dedup comes for free: because the write-time dedup probe is a per-user nearest-neighbour search, a workout-note observation that merely restates something already learned from chat is skipped automatically. **The `prog-strength-agent` repo is not touched** — retrieval already returns memories without regard to their origin.

## Goals and Non-Goals

### Goals

- Generalize the distillation job's `SessionSource` interface into a source-agnostic **`MemorySource`** interface plus a registry the job ranges over, with chat reworked as the first implementer (behaviour identical to today).
- Extend the `MemorySource` interface with an `AllUndistilled` method used only by the backfill, so every source — present and future — gets backfill support by implementing the interface.
- Add a **`WorkoutNoteSource`** that treats one workout as one distillation unit, composing `workouts.notes` and the workout's `workout_exercises.notes` into a single blob, eligible once `workouts.updated_at` has settled past a configurable window, scoped per user and excluding soft-deleted workouts.
- Migration `035_multi_source_memory.sql`: rebuild `agent_memories` to add a `source_type` discriminator, make `source_session_id` nullable, add `source_workout_id` (nullable FK → `workouts(id) ON DELETE CASCADE`), and add a `CHECK` constraint that the FK matching `source_type` is the populated one; backfill existing rows to `source_type='chat_session'`. Add `memory_distilled_at` to `workouts`.
- Per-source settle windows in the `[vectormemory]` config block: keep `session_idle_minutes` for chat, add `workout_settle_minutes` (starting point ~10) for workout notes.
- A per-source distiller framing hint so a terse workout note is distilled with notes-appropriate guidance while sharing the single `Distiller` (Haiku) client.
- Extend the one-time backfill command to iterate the source registry and seed both chat history and the full existing workout-note corpus via the batch APIs, idempotently.
- The existing admin inspection dump (`GET /admin/memories`) surfaces `source_type` so distillation quality can be eyeballed per source.
- Preserve every invariant of the shipped feature: single API writer of `app.db`, MCP uninvolved, per-user vector scoping, model-version tagging on every vector, best-effort agent retrieval, kill-switch via config.

### Non-Goals

- **Touching the read path or the agent.** Retrieval, the `vec_agent_memories` table, the embedding model, and the agent's retrieval integration are unchanged. Memories from a new source flow into the existing search with no read-side work. `prog-strength-agent` is out of scope.
- **A polymorphic / source-table abstraction for provenance.** Provenance stays as typed per-source FK columns. Each new source adds one nullable FK column and one migration — an accepted, deliberate cost in exchange for DB-level referential integrity.
- **Structured workout-data recall.** Sets, reps, weights, dates, bodyweight, goals remain served by the existing MCP tools. Only free-text *notes* are ingested as unstructured signal.
- **Re-distilling edited notes.** A workout marked `memory_distilled_at` is not re-distilled if its note is later edited — matching chat, where a re-opened conversation isn't re-distilled. Flagged in Open Questions.
- **Real-time / per-write distillation.** Workout notes are distilled by the same batched settle-then-sweep job as chat, not synchronously on save.
- **Per-set or per-field note ingestion beyond `workout_exercises.notes`.** The two existing note columns are the scope; no new note surfaces are introduced.
- **User-visible memory management or source attribution in the UI.** Memories remain an internal context-enrichment mechanism, inspected by the operator only (unchanged from the original feature).

## Implementation Details

The sections below follow the write-path lifecycle: source interface → data model → workout-note source → distillation job → backfill → config → tests → rollout. All changes are in `prog-strength-api` unless noted.

### The `MemorySource` Interface

Today, in `internal/vectormemory/job.go`:

```go
type SessionSource interface {
    IdleUndistilled(ctx context.Context, cutoff time.Time, limit int) ([]IdleSession, error)
    Conversation(ctx context.Context, sessionID string) ([]ConversationMessage, error)
    MarkDistilled(ctx context.Context, sessionID string, at time.Time) error
}
```

This is replaced by a source-agnostic interface. A source yields self-contained units (text already assembled, so the job never knows how a source builds its content) and records progress:

```go
// MemorySource is one origin of unstructured signal to distill into memories.
// One implementation per source type; all are held in a registry the job ranges over.
type MemorySource interface {
    // SourceType is the stable discriminator stored on every memory this source
    // produces, e.g. "chat_session" or "workout_note".
    SourceType() string

    // PendingUnits returns units that have settled (gone idle past this source's
    // own window) and are not yet distilled, up to limit.
    PendingUnits(ctx context.Context, now time.Time, limit int) ([]DistillUnit, error)

    // AllUndistilled returns every not-yet-distilled unit, ignoring the settle
    // window. Used only by the one-time backfill; cursor-paginated.
    AllUndistilled(ctx context.Context, cursor string, limit int) ([]DistillUnit, string, error)

    // MarkDistilled records that a unit was processed (even with zero observations).
    MarkDistilled(ctx context.Context, unitID string, at time.Time) error
}

// DistillUnit is one self-contained unit of content ready to distil.
type DistillUnit struct {
    UnitID     string     // source-local id (chat session id, workout id)
    UserID     string
    Content    string     // the assembled text handed to the distiller
    PromptHint string     // source-specific framing appended to the distiller prompt
    Source     Provenance // which typed FK column(s) the resulting memory fills
}

// Provenance names the origin so the repository writes the right typed FK + discriminator.
type Provenance struct {
    SourceType string  // "chat_session" | "workout_note"
    SessionID  *string // set iff SourceType == "chat_session"
    MessageID  *int64  // best-effort, chat only
    WorkoutID  *string // set iff SourceType == "workout_note"
}
```

The chat source keeps its existing SQL (`IdleUndistilled` / `SessionMessages` / `MarkDistilled` in `internal/chat/sqlite_repository.go`) but is wrapped to satisfy `MemorySource`: `PendingUnits` runs the existing idle query and assembles the conversation into `DistillUnit.Content`; `Provenance` carries `SourceType:"chat_session"` and the session/message ids; the chat `PromptHint` is empty (the current behaviour). The adapter lives where the current one does, `internal/server/vectormemory.go`.

A `registry := []MemorySource{chatSource, workoutNoteSource}` is built in `server.New` and passed to the job.

### Data Model

One migration, `internal/db/migrations/035_multi_source_memory.sql` (rebase onto the latest ledger entry before implementing; `034_agent_memory.sql` is current on `main`).

SQLite cannot relax a column's `NOT NULL` or add a table-level `CHECK` in place, so `agent_memories` is rebuilt-and-copied (trivial at current volume). The current table:

```sql
CREATE TABLE agent_memories (
    id                INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id           TEXT NOT NULL,
    distilled_text    TEXT NOT NULL,
    source_session_id TEXT NOT NULL REFERENCES chat_sessions(id) ON DELETE CASCADE,
    source_message_id INTEGER REFERENCES chat_messages(id) ON DELETE SET NULL,
    embedding_model   TEXT NOT NULL,
    embedding_dim     INTEGER NOT NULL,
    superseded_at     DATETIME,
    created_at        DATETIME NOT NULL
);
```

becomes:

```sql
CREATE TABLE agent_memories_new (
    id                INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id           TEXT NOT NULL,
    distilled_text    TEXT NOT NULL,
    source_type       TEXT NOT NULL,                          -- 'chat_session' | 'workout_note'
    source_session_id TEXT REFERENCES chat_sessions(id) ON DELETE CASCADE,   -- now nullable
    source_message_id INTEGER REFERENCES chat_messages(id) ON DELETE SET NULL,
    source_workout_id TEXT REFERENCES workouts(id) ON DELETE CASCADE,        -- new
    embedding_model   TEXT NOT NULL,
    embedding_dim     INTEGER NOT NULL,
    superseded_at     DATETIME,
    created_at        DATETIME NOT NULL,
    CHECK (
        (source_type = 'chat_session' AND source_session_id IS NOT NULL AND source_workout_id IS NULL) OR
        (source_type = 'workout_note' AND source_workout_id IS NOT NULL AND source_session_id IS NULL)
    )
);
-- copy existing rows, all of which are chat-sourced:
INSERT INTO agent_memories_new (id, user_id, distilled_text, source_type,
    source_session_id, source_message_id, source_workout_id,
    embedding_model, embedding_dim, superseded_at, created_at)
SELECT id, user_id, distilled_text, 'chat_session',
    source_session_id, source_message_id, NULL,
    embedding_model, embedding_dim, superseded_at, created_at
FROM agent_memories;
DROP TABLE agent_memories;
ALTER TABLE agent_memories_new RENAME TO agent_memories;
-- recreate indexes from 034 (user_id; source_session_id) and add one for source_workout_id.
```

The `id` values are preserved by the copy, so the `memory_id` join to the untouched `vec_agent_memories` virtual table stays valid — no vector rows are rewritten. The migration also adds the workout distillation marker:

```sql
ALTER TABLE workouts ADD COLUMN memory_distilled_at DATETIME;  -- NULL = not yet distilled
```

### Go changes in `internal/vectormemory`

- **`models.go`** — `Memory` and `NewMemory` gain `SourceType string` and `SourceWorkoutID *string`; `SourceSessionID` becomes `*string` (nullable). 
- **`sqlite_repository.go`** — the `Insert` statement writes `source_type`, `source_session_id`, `source_message_id`, `source_workout_id` (the FK not matching the discriminator passed as `NULL`), driven by the `Provenance` on the unit. The dedup probe, KNN search, and supersede writes are unchanged (source-agnostic).
- **`service.go`** — `DistillSession` generalizes to `DistillUnit`: it takes a `DistillUnit`, calls the `Distiller` with `unit.Content` and `unit.PromptHint`, and writes each observation with `unit.Source` provenance. Retrieval is untouched.
- **`job.go`** — `runDistill` ranges over `[]MemorySource`; per tick, per source, it calls `PendingUnits(now, batchSize)`, distills each unit, and `MarkDistilled` on success (mark-on-success-only semantics preserved). The 5-minute tick and batch size carry over; the idle cutoff is computed per source from its configured window.

### The Workout-Note Source

A new adapter (in `internal/server/vectormemory.go` alongside the chat adapter, or a sibling file) over the workout repository.

- **`PendingUnits`** selects eligible workouts:

  ```sql
  SELECT w.id, w.user_id
  FROM workouts w
  WHERE w.deleted_at IS NULL
    AND w.memory_distilled_at IS NULL
    AND w.updated_at < ?                       -- now - workout_settle_minutes
    AND (
      (w.notes IS NOT NULL AND TRIM(w.notes) <> '')
      OR EXISTS (
        SELECT 1 FROM workout_exercises we
        WHERE we.workout_id = w.id
          AND we.notes IS NOT NULL AND TRIM(we.notes) <> ''
      )
    )
  ORDER BY w.updated_at ASC
  LIMIT ?;
  ```

  For each returned workout it assembles `DistillUnit.Content` from the workout note and its exercises' notes — lightly structured, e.g. a leading `Workout notes:` line followed by `<exercise name>: <note>` lines for each noted exercise (exercise names joined from `workout_exercises`→`exercises`). `PromptHint` frames these as terse training-log notes. `Provenance{SourceType:"workout_note", WorkoutID:&id}`.

- **`AllUndistilled`** is the same query without the `updated_at` settle clause, cursor-paginated on `(updated_at, id)` for the backfill.
- **`MarkDistilled`** → `UPDATE workouts SET memory_distilled_at = ? WHERE id = ?`.

The distiller prompt continues to extract only *durable, stable* observations and to return an empty array when a note holds nothing worth remembering (a bare "good session" yields nothing); the workout `PromptHint` tunes that judgement for terse notes. Prompt wording is iterated against real notes via the existing admin inspection dump.

### Backfill

The existing one-time `cmd/memory-backfill` command is made source-aware: it ranges over the same registry and, per source, drains `AllUndistilled` in pages, distilling via the Anthropic Message Batches API and embedding via the OpenAI Batch embeddings API (half-price async, fine for a one-shot), inserting text + vector rows with the same dedup check as the live path, and calling `MarkDistilled` per unit. Idempotent and resumable: a re-run skips anything already stamped (`memory_distilled_at`/distilled session). One invocation seeds chat history and the full existing workout-note corpus.

### Configuration

The `[vectormemory]` block in the centralized `config.toml` gains one knob; everything else is reused:

| Key | Default (starting point) | Meaning |
| --- | --- | --- |
| `workout_settle_minutes` | `10` | A workout with no edit for this long is eligible for distillation. Separate from `session_idle_minutes` (chat) because a log settles faster than a conversation. |

No new secrets — the same `OPENAI_API_KEY` and `ANTHROPIC_API_KEY` cover the new source. The `enabled` kill-switch continues to gate the whole feature, including the new source.

### Tests

| Repo | Coverage |
| --- | --- |
| `prog-strength-api` | **Migration:** `035` rebuilds `agent_memories` preserving `id` values and existing rows (all stamped `source_type='chat_session'`); the `CHECK` constraint rejects a row whose FK doesn't match its `source_type`; `workouts.memory_distilled_at` exists. **Workout source:** `PendingUnits` selects only settled, undistilled, non-deleted workouts that have a workout note *or* an exercise note; ignores note-less workouts; composes content from both note columns; `MarkDistilled` sets the column; `AllUndistilled` ignores the settle window and paginates. **Job:** ranges over multiple sources in one tick; a unit yielding zero observations is still marked; an error on one unit/source logs and continues without blocking others. **Repository:** `Insert` writes the correct typed FK + `source_type` per provenance; cross-source dedup probe skips a workout-note observation near-identical to an existing chat memory. **Backfill:** drains both sources; idempotent re-run is a no-op. **Retrieval unchanged:** existing retrieval/threshold tests still pass with mixed-source rows present. Providers tested against fakes; no live API calls. |
| `prog-strength-docs` | This SOW; status transitions. |

Follow existing conventions: stdlib `testing`, `httptest`, per-test temp `app.db` via `dbtest`, fakes for external providers.

### Rollout

1. **Migration + generalized seam**, behind the existing `enabled` flag. Chat distillation behaviour is byte-for-byte unchanged; ship and confirm no regression in the existing feature.
2. **Workout-note source live** for new/edited workouts via the settle-then-sweep job. Watch the admin inspection dump (`source_type='workout_note'`) and iterate the distiller `PromptHint` against real notes before relying on the output.
3. **Backfill** the existing workout-note corpus via the batch APIs once the distiller prompt looks right on live notes. Idempotent, so it can be re-run after a prompt tweak (against not-yet-distilled workouts).
4. Retrieval needs no rollout step — it already returns the new memories the moment they exist, gated by the same threshold tuned for the shipped feature.

## Open Questions

1. **Re-distilling edited notes.** A workout is distilled once and not revisited if its note is later edited (matches chat). If users meaningfully revise notes after the settle window, consider resetting `memory_distilled_at` on note edit — at the cost of possible duplicate/superseding churn. Defer until real usage shows it matters.
2. **Workout-note content assembly.** The exact structure of the composed blob (how exercise notes are labelled, whether to include the workout name/date for context) should be iterated against real notes via the inspection dump. The starting format is a first draft.
3. **`workout_settle_minutes` default.** 10 minutes is a starting point; validate against how users actually edit logs (a workout logged set-by-set during a session may keep bumping `updated_at`).
4. **Distiller framing per source.** Whether one shared distiller prompt with a per-source `PromptHint` is sufficient, or whether workout notes warrant a materially different system prompt. Start with the hint; split only if quality demands it.
5. **Settle signal for workouts.** `updated_at` is assumed to bump when a workout or its exercises change. Confirm during implementation that exercise-note edits propagate to `workouts.updated_at`; if not, the settle query must also consider child-row timestamps.
