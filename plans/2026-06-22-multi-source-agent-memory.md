# Multi-Source Agent Memory Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Generalize `prog-strength-api`'s chat-only memory distillation pipeline into a source-agnostic `MemorySource` seam, and add workout-log notes as a second source (live + backfilled).

**Architecture:** The write path of the shipped Agent Vector Memory feature is reworked behind a generic `MemorySource` interface that yields self-contained `DistillUnit`s. Chat becomes the first implementer (behaviour byte-for-byte identical); a new workout-note source becomes the second. Provenance is tracked with typed per-source FK columns (`source_session_id` for chat, `source_workout_id` for workout notes) plus a `source_type` discriminator and a `CHECK` constraint. The read path, embedding model, vector table, retrieval, and the agent are untouched.

**Tech Stack:** Go 1.25 (stdlib `testing`, `httptest`), chi, `mattn/go-sqlite3` + `asg017/sqlite-vec` (cgo), SQLite (`app.db`), Anthropic Messages + Batches API, OpenAI Embeddings + Batch API, Prometheus.

---

## Build environment note (read first)

The repo builds with cgo against `sqlite-vec`, which `#include`s `sqlite3.h`. On this box the header is at `/home/developer/sqlite-headers/include` and `export CGO_CFLAGS="-I/home/developer/sqlite-headers/include"` is already in `~/.bashrc`. If a build fails with `fatal error: sqlite3.h: No such file or directory`, run `export CGO_CFLAGS="-I/home/developer/sqlite-headers/include"` first. All `go build` / `go test` / `golangci-lint` commands below assume this is set.

## CI gate (every task must keep this green)

Run before considering any task done, from `/workspace/prog-strength-api`:

```
export CGO_CFLAGS="-I/home/developer/sqlite-headers/include"
gofmt -l .                 # must print nothing
go vet ./...
go run github.com/golangci/golangci-lint/v2/cmd/golangci-lint@v2.12.2 run
go test ./...
go mod tidy                # must produce no go.mod/go.sum diff
```

No `--no-verify`, no `//nolint`, no `gosec` suppressions, no skipped tests. If a check fails, fix the code.

## File structure

| File | Responsibility | Task |
| --- | --- | --- |
| `internal/db/migrations/035_multi_source_memory.sql` | Rebuild `agent_memories` (discriminator + typed FKs + CHECK), add `workouts.memory_distilled_at` | 1 |
| `internal/db/migration_035_multi_source_memory_test.go` | Migration assertions | 1 |
| `internal/vectormemory/models.go` | `Memory`/`NewMemory` gain `SourceType`, `SourceWorkoutID`; `SourceSessionID` nullable | 2 |
| `internal/vectormemory/sqlite_repository.go` | `Insert`/`Dump` write/read the typed provenance columns | 2 |
| `internal/vectormemory/handler.go` | Admin dump surfaces `source_type` (+ `source_workout_id`) | 2 |
| `internal/vectormemory/job.go` | `MemorySource` interface + `DistillUnit` + `Provenance`; `runDistill` ranges over a registry | 3 |
| `internal/vectormemory/service.go` | `DistillSession` → `DistillUnit`; exported `RenderConversation` | 3 |
| `internal/vectormemory/provider.go` + `anthropic.go` + `batch.go` | `Distiller.Distill` / `DistillBatch` take a per-source `promptHint` | 3 |
| `internal/server/vectormemory.go` | Chat adapter → `MemorySource`; `BuildMemorySources` registry builder | 3, 4 |
| `internal/server/server.go` | Build the registry, pass to `StartDistillation` | 3, 4 |
| `internal/server/workout_note_source.go` | `WorkoutNoteSource` adapter | 4 |
| `internal/config/config.go` + `config.toml` | `WorkoutSettleMinutes` knob | 4 |
| `cmd/memory-backfill/main.go` | Source-aware backfill over the registry | 5 |

---

## Task 1: Migration 035 — multi-source provenance schema

**Files:**
- Create: `internal/db/migrations/035_multi_source_memory.sql`
- Test: `internal/db/migration_035_multi_source_memory_test.go`

**Context:** `agent_memories` (created in `034_agent_memory.sql`) ties every memory to a chat session via a `NOT NULL` FK `source_session_id`. SQLite cannot relax `NOT NULL` or add a table-level `CHECK` in place, so the table is rebuilt-and-copied (trivial at current volume). The `id` values MUST be preserved because `vec_agent_memories.memory_id` (the untouched vec0 virtual table) joins back to them — no vector rows are rewritten. Migrations are discovered by `//go:embed migrations/*.sql` and applied in one transaction per file (`internal/db/migrate.go`); a new numbered `.sql` file is auto-registered. `034` is current on `main` — confirm with `ls internal/db/migrations/ | tail -3` and renumber to the next integer if `main` has advanced. The test helpers `newEmptyDB(t)` and `applyMigrationsThrough(t, conn, 0, nil)` (apply all) live in `internal/db/*_test.go`; `columnExists(t, db, table, column)` is in `migration_034_agent_memory_test.go`.

- [ ] **Step 1: Write the failing migration test**

Create `internal/db/migration_035_multi_source_memory_test.go`:

```go
package db

import (
	"strings"
	"testing"
)

// TestMigrate035_MultiSourceMemory verifies the agent_memories rebuild
// preserves existing chat rows (stamped source_type='chat_session') and id
// values, that the CHECK constraint enforces FK/discriminator agreement, that
// the new source_workout_id cascade works, and that workouts gained
// memory_distilled_at.
func TestMigrate035_MultiSourceMemory(t *testing.T) {
	t.Parallel()
	conn := newEmptyDB(t)

	// Pause right before 035 to seed a chat-sourced memory under the OLD schema.
	applyMigrationsThrough(t, conn, 35, func(t *testing.T, db *sql.DB) {
		const now = "2026-06-20T12:00:00Z"
		if _, err := db.Exec(`INSERT INTO chat_sessions (id, user_id, title, created_at, updated_at, last_message_at) VALUES ('sess-1','u1','',?,?,?)`, now, now, now); err != nil {
			t.Fatalf("seed chat_sessions: %v", err)
		}
		if _, err := db.Exec(`INSERT INTO agent_memories (user_id, distilled_text, source_session_id, embedding_model, embedding_dim, created_at) VALUES ('u1','squats monday','sess-1','text-embedding-3-small',1536,?)`, now); err != nil {
			t.Fatalf("seed agent_memories: %v", err)
		}
		// Seed the matching vec row so we can prove the join key survives.
		vec := "[" + strings.Repeat("0,", 1535) + "1]"
		var id int64
		if err := db.QueryRow(`SELECT id FROM agent_memories WHERE source_session_id='sess-1'`).Scan(&id); err != nil {
			t.Fatalf("read seeded id: %v", err)
		}
		if _, err := db.Exec(`INSERT INTO vec_agent_memories (memory_id, user_id, embedding) VALUES (?, 'u1', ?)`, id, vec); err != nil {
			t.Fatalf("seed vec row: %v", err)
		}
	})

	// Existing row carried over, stamped chat_session, id preserved, vec join intact.
	var srcType string
	var id int64
	if err := conn.QueryRow(`SELECT id, source_type FROM agent_memories WHERE source_session_id='sess-1'`).Scan(&id, &srcType); err != nil {
		t.Fatalf("read migrated row: %v", err)
	}
	if srcType != "chat_session" {
		t.Fatalf("existing rows must be stamped chat_session, got %q", srcType)
	}
	var vecMemID int64
	if err := conn.QueryRow(`SELECT memory_id FROM vec_agent_memories WHERE memory_id=?`, id).Scan(&vecMemID); err != nil {
		t.Fatalf("vec join key not preserved for id %d: %v", id, err)
	}

	// workouts gained the distillation marker.
	if !columnExists(t, conn, "workouts", "memory_distilled_at") {
		t.Fatal("workouts should have memory_distilled_at after 035")
	}

	// CHECK rejects a chat_session row missing its session FK.
	if _, err := conn.Exec(`INSERT INTO agent_memories (user_id, distilled_text, source_type, embedding_model, embedding_dim, created_at) VALUES ('u1','x','chat_session','m',1,'2026-06-20T12:00:00Z')`); err == nil {
		t.Fatal("CHECK should reject chat_session with NULL source_session_id")
	}
	// CHECK rejects a workout_note row whose session FK is also set.
	if _, err := conn.Exec(`INSERT INTO agent_memories (user_id, distilled_text, source_type, source_session_id, source_workout_id, embedding_model, embedding_dim, created_at) VALUES ('u1','x','workout_note','sess-1','w1','m',1,'2026-06-20T12:00:00Z')`); err == nil {
		t.Fatal("CHECK should reject workout_note with a non-NULL source_session_id")
	}

	// A workout-note memory cascades when its workout is deleted.
	const now = "2026-06-20T12:00:00Z"
	if _, err := conn.Exec(`INSERT INTO workouts (id, user_id, name, performed_at, notes, created_at, updated_at) VALUES ('w1','u1','leg day','2026-06-20T10:00:00Z','felt strong',?,?)`, now, now); err != nil {
		t.Fatalf("seed workout: %v", err)
	}
	if _, err := conn.Exec(`INSERT INTO agent_memories (user_id, distilled_text, source_type, source_workout_id, embedding_model, embedding_dim, created_at) VALUES ('u1','strong on leg day','workout_note','w1','text-embedding-3-small',1536,?)`, now); err != nil {
		t.Fatalf("insert workout-note memory: %v", err)
	}
	if _, err := conn.Exec(`DELETE FROM workouts WHERE id='w1'`); err != nil {
		t.Fatalf("delete workout: %v", err)
	}
	var remaining int
	if err := conn.QueryRow(`SELECT count(*) FROM agent_memories WHERE source_workout_id='w1'`).Scan(&remaining); err != nil {
		t.Fatalf("count after cascade: %v", err)
	}
	if remaining != 0 {
		t.Fatalf("workout-note memory should cascade-delete with its workout, %d remaining", remaining)
	}

	// The new index exists.
	var idx string
	if err := conn.QueryRow(`SELECT name FROM sqlite_master WHERE type='index' AND name='idx_agent_memories_source_workout'`).Scan(&idx); err != nil {
		t.Fatalf("idx_agent_memories_source_workout should exist: %v", err)
	}
}
```

Add the `"database/sql"` import (the closure param uses `*sql.DB`). Mirror the exact arg order `applyMigrationsThrough` expects (see its signature in `migration_033_workout_tcx_test.go`: `(t, db, pauseAt int, before func(t, db))` — `pauseAt` is the version to pause *before*; here pass `35`). If `applyMigrationsThrough`'s `before` runs *before* applying version `pauseAt`, the seed lands under the pre-035 schema as intended. Verify the helper's semantics by reading it first; adapt the seed-vs-assert split to match.

- [ ] **Step 2: Run the test — expect FAIL** (`035` file does not exist yet; the `before` closure may even error on the old schema if `pauseAt` semantics differ — read the helper and fix the test harness usage until the failure is specifically "035 behaviour missing", not a harness misuse).

Run: `go test ./internal/db/ -run TestMigrate035 -v`

- [ ] **Step 3: Write the migration**

Create `internal/db/migrations/035_multi_source_memory.sql`:

```sql
-- migrations/035_multi_source_memory.sql
-- Multi-Source Agent Memory: generalize agent_memories provenance from a
-- chat-only FK to a source_type discriminator + typed per-source FK columns.
-- SQLite cannot relax NOT NULL or add a table CHECK in place, so the table is
-- rebuilt-and-copied; id values are preserved so the vec_agent_memories join
-- (memory_id) stays valid and no vectors are rewritten. See
-- prog-strength-docs/sows/multi-source-agent-memory.md.

CREATE TABLE agent_memories_new (
    id                INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id           TEXT NOT NULL,
    distilled_text    TEXT NOT NULL,
    source_type       TEXT NOT NULL,                                          -- 'chat_session' | 'workout_note'
    source_session_id TEXT REFERENCES chat_sessions(id) ON DELETE CASCADE,    -- nullable now
    source_message_id INTEGER REFERENCES chat_messages(id) ON DELETE SET NULL,
    source_workout_id TEXT REFERENCES workouts(id) ON DELETE CASCADE,         -- new
    embedding_model   TEXT NOT NULL,
    embedding_dim     INTEGER NOT NULL,
    superseded_at     DATETIME,
    created_at        DATETIME NOT NULL,
    CHECK (
        (source_type = 'chat_session' AND source_session_id IS NOT NULL AND source_workout_id IS NULL) OR
        (source_type = 'workout_note' AND source_workout_id IS NOT NULL AND source_session_id IS NULL)
    )
);

-- Existing rows are all chat-sourced.
INSERT INTO agent_memories_new (id, user_id, distilled_text, source_type,
    source_session_id, source_message_id, source_workout_id,
    embedding_model, embedding_dim, superseded_at, created_at)
SELECT id, user_id, distilled_text, 'chat_session',
    source_session_id, source_message_id, NULL,
    embedding_model, embedding_dim, superseded_at, created_at
FROM agent_memories;

DROP TABLE agent_memories;
ALTER TABLE agent_memories_new RENAME TO agent_memories;

-- Recreate the 034 indexes and add one for the new workout FK (cascade lookups).
CREATE INDEX idx_agent_memories_user ON agent_memories(user_id);
CREATE INDEX idx_agent_memories_source_session ON agent_memories(source_session_id);
CREATE INDEX idx_agent_memories_source_workout ON agent_memories(source_workout_id);

-- Workout distillation marker. NULL = not yet distilled.
ALTER TABLE workouts ADD COLUMN memory_distilled_at DATETIME;
```

Note on FK enforcement: the rebuild runs inside the migration transaction with `foreign_keys` already ON. No other table has an FK pointing *at* `agent_memories`, so the drop/rename is safe; the copy satisfies the new table's own FKs (existing `source_session_id` values are valid, `source_workout_id` is NULL). Do **not** add `PRAGMA foreign_keys` statements (a no-op mid-transaction, and the rebuild doesn't need them).

- [ ] **Step 4: Run the test — expect PASS**

Run: `go test ./internal/db/ -run TestMigrate035 -v`
Expected: PASS. Then run the whole package: `go test ./internal/db/` (the existing `034` test must still pass).

- [ ] **Step 5: Run the gate, then commit**

```
gofmt -l . && go vet ./internal/db/ && go test ./internal/db/
git add internal/db/migrations/035_multi_source_memory.sql internal/db/migration_035_multi_source_memory_test.go
git commit -m "feat(vectormemory): migration 035 multi-source provenance schema"
```

---

## Task 2: Data model + repository + admin dump (chat behaviour identical)

**Files:**
- Modify: `internal/vectormemory/models.go`
- Modify: `internal/vectormemory/sqlite_repository.go`
- Modify: `internal/vectormemory/handler.go`
- Modify: `internal/vectormemory/sqlite_repository_test.go`, `internal/vectormemory/handler_test.go`, `internal/vectormemory/service_test.go` (whatever the field changes break)

**Context:** Migration 035 (Task 1) added `source_type`, `source_workout_id`, made `source_session_id` nullable, and added the CHECK. Now teach the Go layer to write and read those columns — but ONLY for chat (`source_type='chat_session'`), so behaviour is identical to today. The `MemorySource`/`Provenance` seam comes in Task 3; this task keeps `DistillSession(ctx, userID, sessionID, messages)`'s signature unchanged and just has it stamp chat provenance internally. `NewMemory` is the insert input (`service.go` and `cmd/memory-backfill/main.go` both build it); `Memory` is the read/dump row. `SourceSessionID` is currently a non-nullable `string` on both — it becomes `*string`. Every CHECK-satisfying chat insert sets `source_type='chat_session'`, a non-nil `source_session_id`, and a nil `source_workout_id`.

- [ ] **Step 1: Update the failing tests first (TDD)**

In `internal/vectormemory/sqlite_repository_test.go`, update existing insert/dump assertions to the new shape and add a test that `Insert` writes `source_type` and that a workout-note provenance writes `source_workout_id` with NULL `source_session_id`. Example additions (adapt to the existing test scaffolding/helpers in that file):

```go
func TestInsert_WritesChatProvenance(t *testing.T) {
	// ...open migrated db, seed chat_sessions 'sess-1' for user 'u1' (FK)...
	repo := NewSQLiteRepository(db)
	sess := "sess-1"
	id, err := repo.Insert(ctx, NewMemory{
		UserID: "u1", DistilledText: "lifts heavy", SourceType: "chat_session",
		SourceSessionID: &sess, EmbeddingModel: "text-embedding-3-small",
		EmbeddingDim: 1536, Embedding: unitVec(), CreatedAt: time.Now().UTC(),
	})
	if err != nil { t.Fatalf("insert: %v", err) }
	var st string
	var sid sql.NullString
	var wid sql.NullString
	if err := db.QueryRow(`SELECT source_type, source_session_id, source_workout_id FROM agent_memories WHERE id=?`, id).Scan(&st, &sid, &wid); err != nil {
		t.Fatal(err)
	}
	if st != "chat_session" || !sid.Valid || sid.String != "sess-1" || wid.Valid {
		t.Fatalf("bad provenance: type=%q sid=%v wid=%v", st, sid, wid)
	}
}

func TestInsert_WritesWorkoutProvenance(t *testing.T) {
	// ...seed workouts 'w1' for user 'u1' (FK)...
	repo := NewSQLiteRepository(db)
	wid := "w1"
	id, err := repo.Insert(ctx, NewMemory{
		UserID: "u1", DistilledText: "left shoulder cranky", SourceType: "workout_note",
		SourceWorkoutID: &wid, EmbeddingModel: "text-embedding-3-small",
		EmbeddingDim: 1536, Embedding: unitVec(), CreatedAt: time.Now().UTC(),
	})
	if err != nil { t.Fatalf("insert: %v", err) }
	var st string
	var sid, gotWid sql.NullString
	_ = db.QueryRow(`SELECT source_type, source_session_id, source_workout_id FROM agent_memories WHERE id=?`, id).Scan(&st, &sid, &gotWid)
	if st != "workout_note" || sid.Valid || !gotWid.Valid || gotWid.String != "w1" {
		t.Fatalf("bad workout provenance: type=%q sid=%v wid=%v", st, sid, gotWid)
	}
}
```

`unitVec()` — a 1536-float slice (e.g. all zero with a trailing 1). Reuse whatever the existing tests already use to build embeddings; do not introduce a redundant helper if one exists.

In `internal/vectormemory/handler_test.go`, update the dump test to assert the row JSON now carries `source_type` (and `source_workout_id` when set).

- [ ] **Step 2: Run tests — expect FAIL** (compile errors: `NewMemory` has no `SourceType`/`SourceWorkoutID`; `SourceSessionID` type mismatch).

Run: `go test ./internal/vectormemory/ 2>&1 | head -30`

- [ ] **Step 3: Update `models.go`**

```go
// Memory is one distilled, durable observation about a user, plus its
// provenance and the embedding-model metadata that guards against
// comparing vectors from different models.
type Memory struct {
	ID              int64
	UserID          string
	DistilledText   string
	SourceType      string  // "chat_session" | "workout_note"
	SourceSessionID *string // set iff SourceType == "chat_session"
	SourceMessageID *int64  // best-effort, chat only
	SourceWorkoutID *string // set iff SourceType == "workout_note"
	EmbeddingModel  string
	EmbeddingDim    int
	SupersededAt    *time.Time
	CreatedAt       time.Time
}

// NewMemory is the insert input: the text row fields plus the vector. The
// repo writes the text row and the vector row in one transaction. SourceType
// selects which typed FK is populated (the other is written NULL).
type NewMemory struct {
	UserID          string
	DistilledText   string
	SourceType      string
	SourceSessionID *string
	SourceMessageID *int64
	SourceWorkoutID *string
	EmbeddingModel  string
	EmbeddingDim    int
	Embedding       []float32
	CreatedAt       time.Time
}
```

Also add `SourceType` and `SourceWorkoutID *string` to the `Match` struct only if a reader needs them — retrieval is untouched, so leave `Match` as-is unless a test requires it (YAGNI).

- [ ] **Step 4: Update `sqlite_repository.go`**

`Insert` writes all four provenance columns; the FK not matching `source_type` goes in as NULL via `nullableString`:

```go
res, err := tx.ExecContext(ctx, `
	INSERT INTO agent_memories (
		user_id, distilled_text, source_type, source_session_id,
		source_message_id, source_workout_id, embedding_model, embedding_dim, created_at
	) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)
`,
	m.UserID,
	m.DistilledText,
	m.SourceType,
	nullableString(m.SourceSessionID),
	nullableInt64(m.SourceMessageID),
	nullableString(m.SourceWorkoutID),
	m.EmbeddingModel,
	m.EmbeddingDim,
	m.CreatedAt.UTC(),
)
```

Add the helper next to `nullableInt64`:

```go
// nullableString maps a *string to sql.NullString so a nil typed-FK column is
// written as SQL NULL rather than the empty string (which would violate the
// agent_memories CHECK / FK).
func nullableString(p *string) sql.NullString {
	if p == nil {
		return sql.NullString{}
	}
	return sql.NullString{String: *p, Valid: true}
}
```

`Dump` selects and scans the new columns:

```go
rows, err := r.db.QueryContext(ctx, `
	SELECT id, user_id, distilled_text, source_type, source_session_id,
	       source_message_id, source_workout_id, embedding_model, embedding_dim,
	       superseded_at, created_at
	FROM agent_memories
	WHERE (? = '' OR user_id = ?)
	ORDER BY created_at DESC, id DESC
	LIMIT ? OFFSET ?
`, userID, userID, limit, offset)
```

```go
var (
	m            Memory
	sourceSess   sql.NullString
	sourceMsgID  sql.NullInt64
	sourceWkt    sql.NullString
	supersededAt sql.NullTime
)
if err := rows.Scan(
	&m.ID, &m.UserID, &m.DistilledText, &m.SourceType,
	&sourceSess, &sourceMsgID, &sourceWkt,
	&m.EmbeddingModel, &m.EmbeddingDim, &supersededAt, &m.CreatedAt,
); err != nil {
	return nil, fmt.Errorf("vectormemory: scan memory: %w", err)
}
if sourceSess.Valid {
	m.SourceSessionID = &sourceSess.String
}
if sourceMsgID.Valid {
	m.SourceMessageID = &sourceMsgID.Int64
}
if sourceWkt.Valid {
	m.SourceWorkoutID = &sourceWkt.String
}
if supersededAt.Valid {
	m.SupersededAt = &supersededAt.Time
}
```

The `Search`/`NearestDistance` queries select `am.source_session_id` into a `Match`; since `Match.SourceSessionID` stays a plain `string`, scan it through a `sql.NullString` and assign `.String` (a workout-note memory has a NULL session id → empty string, which is correct and unused by the agent). Update the `Search` scan accordingly so a NULL doesn't error.

- [ ] **Step 5: Update `service.go`'s chat insert to stamp provenance**

In `DistillSession`, change the `repo.Insert` call so it sets the discriminator and the typed FK (signature of `DistillSession` is unchanged):

```go
sessionID := sessionID // already a param
if _, err := s.repo.Insert(ctx, NewMemory{
	UserID:          userID,
	DistilledText:   obs,
	SourceType:      "chat_session",
	SourceSessionID: &sessionID,
	EmbeddingModel:  s.cfg.EmbedModel,
	EmbeddingDim:    s.cfg.EmbedDim,
	Embedding:       vec,
	CreatedAt:       s.now().UTC(),
}); err != nil {
```

(Use a local copy of the loop-stable `sessionID` so `&sessionID` is safe.)

- [ ] **Step 6: Update `cmd/memory-backfill/main.go`'s insert** the same way (set `SourceType: "chat_session"`, `SourceSessionID: &sess.id`). Use a local copy of `sess.id` for the pointer. This keeps the backfill compiling; Task 5 generalizes it fully.

- [ ] **Step 7: Update `handler.go` dump DTO**

```go
type memoryDTO struct {
	DistilledText   string     `json:"distilled_text"`
	UserID          string     `json:"user_id"`
	SourceType      string     `json:"source_type"`
	SourceSessionID *string    `json:"source_session_id,omitempty"`
	SourceWorkoutID *string    `json:"source_workout_id,omitempty"`
	EmbeddingModel  string     `json:"embedding_model"`
	EmbeddingDim    int        `json:"embedding_dim"`
	CreatedAt       time.Time  `json:"created_at"`
	SupersededAt    *time.Time `json:"superseded_at,omitempty"`
}
```

and map `SourceType`, `SourceSessionID`, `SourceWorkoutID` from the `Memory`.

- [ ] **Step 8: Run tests — expect PASS**

Run: `go test ./internal/vectormemory/`
Expected: PASS (chat round-trips unchanged; new provenance asserted).

- [ ] **Step 9: Gate + commit**

```
gofmt -l . && go vet ./... && go test ./...
git add -A
git commit -m "feat(vectormemory): typed per-source provenance on agent_memories"
```

---

## Task 3: Generalize the chat-only seam into a `MemorySource` registry

**Files:**
- Modify: `internal/vectormemory/job.go` (the interface + DistillUnit/Provenance, runDistill ranges over a registry)
- Modify: `internal/vectormemory/service.go` (`DistillSession` → `DistillUnit`; export `RenderConversation`)
- Modify: `internal/vectormemory/provider.go`, `internal/vectormemory/anthropic.go`, `internal/vectormemory/batch.go` (`promptHint`)
- Modify: `internal/server/vectormemory.go` (chat adapter → `MemorySource`; `BuildMemorySources`)
- Modify: `internal/server/server.go` (build registry, pass to `StartDistillation`)
- Modify: tests in `internal/vectormemory/` (`job_test.go`, `service_test.go`, `anthropic_test.go`, `batch_test.go`) and `cmd/memory-backfill/main.go` (DistillBatch signature)

**Context:** This is the core generalization. Today `job.go` defines `SessionSource` (chat-shaped) and `service.DistillSession(ctx, userID, sessionID, messages)` renders the conversation and distills it. After this task the job ranges over `[]MemorySource`; each source yields self-contained `DistillUnit`s (text already assembled, provenance attached); the service distills a `DistillUnit` without knowing what a "session" or "workout" is. The chat source keeps its exact SQL (`internal/chat/sqlite_repository.go` `IdleUndistilled`/`SessionMessages`/`MarkDistilled`) but is wrapped to satisfy `MemorySource` in `internal/server/vectormemory.go`, assembling the transcript into `DistillUnit.Content` via the (now exported) `vectormemory.RenderConversation`. The distiller gains a per-source `promptHint` appended to its system prompt — empty for chat (behaviour identical), notes-specific for workouts (Task 4). The Prometheus metric *names* must not change (they back the `ps-vector-memory` Grafana dashboard); the internal counters now count "units" generically.

**Design decisions (do not re-derive):**
- `MemorySource` has FIVE methods: the four in the SOW (`SourceType`, `PendingUnits`, `AllUndistilled`, `MarkDistilled`) plus `CountPending(ctx, now) (int, error)`. `CountPending` preserves the existing `api_vectormemory_idle_sessions` backlog gauge (PR #59), which `PendingUnits` cannot feed because it is capped at `limit`. The gauge is set to the **sum** of `CountPending` across sources, keeping the metric name/shape (and the dashboard) intact. This 5th method is a deliberate, minimal extension of the SOW's 4-method sketch to honour the SOW Goal "preserve every invariant … metrics carry over"; call it out in the PR body.
- Each source owns its settle window: `PendingUnits(ctx, now, limit)` and `CountPending(ctx, now)` compute their own cutoff from a window the adapter is constructed with. The job no longer computes a cutoff.
- `service.DistillUnit(ctx, unit DistillUnit) (int, error)` replaces `DistillSession`. (A method named `DistillUnit` on `*Service` does not collide with the package-level type `DistillUnit` — method names are scoped to the receiver.)

- [ ] **Step 1: Define the interface + types in `job.go` (write the job test first)**

In `internal/vectormemory/job_test.go`, add a fake `MemorySource` and a test that one tick ranges over two sources, distills each `PendingUnits` result, marks each on success, and that an error on one unit/source is logged and does not block the others. Model it on the existing job test's fakes. Key assertions:
- two sources → both `PendingUnits` called once per tick;
- a unit whose distiller returns zero observations is still `MarkDistilled`;
- a source whose `PendingUnits` errors does not prevent the other source's units from being marked.

Then replace `SessionSource` with:

```go
// MemorySource is one origin of unstructured signal to distill into memories.
// One implementation per source type; all are held in a registry the job
// ranges over. Implementations live consumer-side (package server) so
// vectormemory never imports chat or workout.
type MemorySource interface {
	// SourceType is the stable discriminator stored on every memory this
	// source produces, e.g. "chat_session" or "workout_note".
	SourceType() string

	// PendingUnits returns units that have settled (gone idle past this
	// source's own window, relative to now) and are not yet distilled, up to
	// limit, oldest-settled first.
	PendingUnits(ctx context.Context, now time.Time, limit int) ([]DistillUnit, error)

	// CountPending returns the full, un-capped settled-and-undistilled backlog
	// for the same window PendingUnits selects against — feeds the idle
	// backlog gauge, which the capped PendingUnits cannot.
	CountPending(ctx context.Context, now time.Time) (int, error)

	// AllUndistilled returns a page of not-yet-distilled units, ignoring the
	// settle window. Used only by the one-time backfill; cursor-paginated,
	// returning the next cursor ("" when exhausted).
	AllUndistilled(ctx context.Context, cursor string, limit int) ([]DistillUnit, string, error)

	// MarkDistilled records that a unit was processed (even with zero
	// observations) so it isn't re-examined.
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

// Provenance names the origin so the repository writes the right typed FK +
// discriminator.
type Provenance struct {
	SourceType string  // "chat_session" | "workout_note"
	SessionID  *string // set iff SourceType == "chat_session"
	MessageID  *int64  // best-effort, chat only
	WorkoutID  *string // set iff SourceType == "workout_note"
}
```

Delete `IdleSession` and `ConversationMessage`'s job-local usages only if nothing else needs them — `ConversationMessage` stays in `service.go` (the chat adapter still uses it to assemble Content). Keep `ConversationMessage`.

- [ ] **Step 2: Rewrite `runDistill`/`distillOnce` to range over the registry**

`StartDistillation(ctx, sources []MemorySource)`. `distillOnce` loops sources; per source: `CountPending` (sum into the `idleSessions` gauge after the loop), `PendingUnits(now, distillBatchSize)`, then for each unit `s.DistillUnit(ctx, unit)` and `src.MarkDistilled(ctx, unit.UnitID, now)` on success. Preserve the mark-on-success-only policy, the per-unit error isolation (`sawStageError`/`continue`), and every metric. A `PendingUnits`/`CountPending` error on one source increments the right stage error and is logged, then the loop continues to the next source. Keep `now := s.now()` once per tick. Example skeleton:

```go
func (s *Service) distillOnce(ctx context.Context, sources []MemorySource) error {
	now := s.now()
	start := now
	defer func() {
		end := s.now()
		sweepDuration.Observe(end.Sub(start).Seconds())
		lastSweepTimestamp.Set(float64(end.Unix()))
	}()

	backlog := 0
	selected, distilled := 0, 0
	sawStageError := false
	for _, src := range sources {
		if n, err := src.CountPending(ctx, now); err != nil {
			s.log.WarnContext(ctx, "vectormemory distillation: count pending failed",
				slog.String("source_type", src.SourceType()), slog.Any("error", err))
		} else {
			backlog += n
		}

		units, err := src.PendingUnits(ctx, now, distillBatchSize)
		if err != nil {
			stageErrorsTotal.WithLabelValues("select").Inc()
			sawStageError = true
			s.log.ErrorContext(ctx, "vectormemory distillation: select pending units failed",
				slog.String("source_type", src.SourceType()), slog.Any("error", err))
			continue
		}
		selected += len(units)
		sessionsSelectedTotal.Add(float64(len(units)))

		for _, unit := range units {
			if _, err := s.DistillUnit(ctx, unit); err != nil {
				sawStageError = true
				s.log.WarnContext(ctx, "vectormemory distillation: distill unit failed, leaving unmarked for retry",
					slog.String("source_type", src.SourceType()),
					slog.String("unit_id", unit.UnitID),
					slog.String("user_id", unit.UserID),
					slog.Any("error", err))
				continue
			}
			if err := src.MarkDistilled(ctx, unit.UnitID, now); err != nil {
				stageErrorsTotal.WithLabelValues("mark").Inc()
				sawStageError = true
				s.log.WarnContext(ctx, "vectormemory distillation: mark distilled failed",
					slog.String("source_type", src.SourceType()),
					slog.String("unit_id", unit.UnitID),
					slog.Any("error", err))
				continue
			}
			sessionsDistilledTotal.Inc()
			distilled++
		}
	}
	idleSessions.Set(float64(backlog))

	lastSuccessTimestamp.Set(float64(s.now().Unix()))
	result := "success"
	if sawStageError {
		result = "partial"
	}
	sweepsTotal.WithLabelValues(result).Inc()
	s.log.InfoContext(ctx, "vectormemory distillation: sweep complete",
		slog.Int("selected", selected), slog.Int("distilled", distilled), slog.String("result", result))
	return nil
}
```

Note: the previous code returned the select error to mark a "batch select failed → error" sweep. With multiple sources, one source's select failure should not abort the others, so it is treated as a stage error (`partial`) and the sweep still completes. Keep the initial-sweep-on-start behaviour in `runDistill` (call `distillOnce` once before the ticker). Adjust the `sweepsTotal{result="error"}` usage: it's now only reachable if you choose to keep a batch-level abort; since we don't, document that `error` is reserved/unused or drop it from the result set — prefer keeping the label value defined for dashboard stability and simply not emitting it.

- [ ] **Step 3: `service.go` — `DistillUnit` + exported `RenderConversation`**

Rename `DistillSession` to `DistillUnit(ctx, unit DistillUnit) (int, error)`. It distills `unit.Content` with `unit.PromptHint`, embeds, dedups, and inserts each observation with provenance derived from `unit.Source`:

```go
func (s *Service) DistillUnit(ctx context.Context, unit DistillUnit) (int, error) {
	distillStart := s.now()
	observations, distillUsage, err := s.distiller.Distill(ctx, unit.Content, unit.PromptHint)
	distillDuration.Observe(s.now().Sub(distillStart).Seconds())
	if err != nil {
		stageErrorsTotal.WithLabelValues("distill").Inc()
		return 0, fmt.Errorf("vectormemory: distill unit: %w", err)
	}
	// ...token metrics, observations==0 early return, embed (unchanged)...

	for i, obs := range observations {
		vec := vecs[i]
		// ...dedup probe unchanged...
		if _, err := s.repo.Insert(ctx, NewMemory{
			UserID:          unit.UserID,
			DistilledText:   obs,
			SourceType:      unit.Source.SourceType,
			SourceSessionID: unit.Source.SessionID,
			SourceMessageID: unit.Source.MessageID,
			SourceWorkoutID: unit.Source.WorkoutID,
			EmbeddingModel:  s.cfg.EmbedModel,
			EmbeddingDim:    s.cfg.EmbedDim,
			Embedding:       vec,
			CreatedAt:       s.now().UTC(),
		}); err != nil {
			// ...insert-failure policy unchanged...
		}
	}
	// ...
}
```

Replace the private `renderConversation` with an exported `RenderConversation(messages []ConversationMessage) string` (same body) so the chat adapter and backfill can share it. Update internal callers. Keep all logging keys; swap `session_id` for `unit_id`/`source_type` where appropriate.

- [ ] **Step 4: `Distiller` gains `promptHint`**

`provider.go`:

```go
type Distiller interface {
	Distill(ctx context.Context, content, promptHint string) ([]string, DistillUsage, error)
	Configured() bool
}
```

`anthropic.go`: `distillRequestBody(model, conversation, promptHint string)` — append the hint to the system prompt when non-empty:

```go
func distillRequestBody(model, conversation, promptHint string) map[string]any {
	system := distillSystemPrompt
	if strings.TrimSpace(promptHint) != "" {
		system = distillSystemPrompt + "\n\n" + promptHint
	}
	return map[string]any{
		"model":      model,
		"max_tokens": distillMaxTokens,
		"system":     system,
		// ...unchanged...
	}
}
```

Update `AnthropicDistiller.Distill` to take and pass `promptHint`. `batch.go`: `DistillBatch(ctx, conversations []string, promptHint string)` — pass `promptHint` into each `distillRequestBody(d.model, conv, promptHint)` call. (All conversations in one batch call share a source, so a single hint per call is correct.)

- [ ] **Step 5: Rewrite the chat adapter + add the registry builder in `internal/server/vectormemory.go`**

```go
// chatMemorySource adapts the chat SQLite repository to vectormemory's
// MemorySource. It lives here (not in vectormemory) so vectormemory never
// imports chat. idleWindow is the chat settle window (cfg.SessionIdleMinutes).
type chatMemorySource struct {
	chat       *chat.SQLiteRepository
	idleWindow time.Duration
}

var _ vectormemory.MemorySource = (*chatMemorySource)(nil)

func (s *chatMemorySource) SourceType() string { return "chat_session" }

func (s *chatMemorySource) PendingUnits(ctx context.Context, now time.Time, limit int) ([]vectormemory.DistillUnit, error) {
	cutoff := now.Add(-s.idleWindow)
	rows, err := s.chat.IdleUndistilled(ctx, cutoff, limit)
	if err != nil {
		return nil, err
	}
	units := make([]vectormemory.DistillUnit, 0, len(rows))
	for _, r := range rows {
		msgs, err := s.chat.SessionMessages(ctx, r.ID)
		if err != nil {
			return nil, err
		}
		conv := make([]vectormemory.ConversationMessage, len(msgs))
		for i, m := range msgs {
			conv[i] = vectormemory.ConversationMessage{Role: string(m.Role), Content: m.Content}
		}
		sid := r.ID
		units = append(units, vectormemory.DistillUnit{
			UnitID:     r.ID,
			UserID:     r.UserID,
			Content:    vectormemory.RenderConversation(conv),
			PromptHint: "", // chat behaviour is unchanged
			Source:     vectormemory.Provenance{SourceType: "chat_session", SessionID: &sid},
		})
	}
	return units, nil
}

func (s *chatMemorySource) CountPending(ctx context.Context, now time.Time) (int, error) {
	return s.chat.CountIdleUndistilled(ctx, now.Add(-s.idleWindow))
}

func (s *chatMemorySource) MarkDistilled(ctx context.Context, unitID string, at time.Time) error {
	return s.chat.MarkDistilled(ctx, unitID, at)
}
```

`AllUndistilled` for chat — page over `chat_sessions WHERE memory_distilled_at IS NULL AND deleted_at IS NULL` ordered by `(last_message_at, id)`, decoding/encoding the cursor (e.g. base64 of the last `last_message_at|id`). The chat repo has no such method; add a thin `AllUndistilledSessions(ctx, cursor, limit)` to `internal/chat/sqlite_repository.go` (mirrors `IdleUndistilled` without the cutoff, with keyset pagination), returning `[]IdleSession` plus the next cursor — OR keep the cursor logic in the adapter with a parametrised query through a new repo method. Keep the SQL in the chat repo (that's where chat SQL lives) and the assembly in the adapter. Implement a minimal keyset cursor; an empty cursor starts at the beginning, `""` returned means exhausted.

Add the registry builder used by `server.New` and the backfill:

```go
// BuildMemorySources constructs the distillation source registry. Order is the
// iteration order of the job and backfill. Chat first; workout-note added in a
// later change. Lives here so the adapters (which import chat/workout) stay out
// of the vectormemory package.
func BuildMemorySources(chatRepo *chat.SQLiteRepository, cfg config.VectorMemoryConfig) []vectormemory.MemorySource {
	return []vectormemory.MemorySource{
		&chatMemorySource{chat: chatRepo, idleWindow: time.Duration(cfg.SessionIdleMinutes) * time.Minute},
	}
}
```

(Workout source is appended in Task 4; its constructor will need the `*sql.DB`, so `BuildMemorySources` grows a `db *sql.DB` parameter then.)

- [ ] **Step 6: `server.go` wiring**

Replace `vmService.StartDistillation(context.Background(), vmSessionSource{chat: chatSQLiteRepo})` with:

```go
vmSources := BuildMemorySources(chatSQLiteRepo, cfg.VectorMemory)
vmService.StartDistillation(context.Background(), vmSources)
```

- [ ] **Step 7: Fix `cmd/memory-backfill/main.go` compile break** from the `DistillBatch` signature change — pass `""` as the chat `promptHint` for now (Task 5 generalizes the backfill). Keep it green.

- [ ] **Step 8: Update tests** — `job_test.go` (registry ranging, new fake), `service_test.go` (`DistillUnit` + provenance, `RenderConversation`), `anthropic_test.go`/`batch_test.go` (assert the hint is appended to the system prompt when non-empty and absent when empty). Verify behaviour-identical-for-chat: an empty `promptHint` produces exactly today's request body.

- [ ] **Step 9: Run the gate — expect PASS**

```
export CGO_CFLAGS="-I/home/developer/sqlite-headers/include"
gofmt -l . && go vet ./... && go test ./... && go run github.com/golangci/golangci-lint/v2/cmd/golangci-lint@v2.12.2 run
```

- [ ] **Step 10: Commit**

```
git add -A
git commit -m "feat(vectormemory): generalize distillation into a MemorySource registry"
```

---

## Task 4: Workout-note source + config knob

**Files:**
- Create: `internal/server/workout_note_source.go`
- Modify: `internal/server/vectormemory.go` (`BuildMemorySources` grows the `*sql.DB` param + appends the workout source)
- Modify: `internal/server/server.go` (pass `database` into `BuildMemorySources`)
- Modify: `internal/config/config.go` (`WorkoutSettleMinutes` field + decode), `config.toml` (`workout_settle_minutes = 10`)
- Modify: `internal/config/config_test.go`
- Create: `internal/server/workout_note_source_test.go`

**Context:** The workout-note source's unit of distillation is **one workout**. Notes live in `workouts.notes` (one per workout) and `workout_exercises.notes` (per exercise); the source composes both into one lightly-structured blob per workout. A workout is eligible once it has settled (`workouts.updated_at` older than `now - workout_settle_minutes`), is not soft-deleted (`deleted_at IS NULL`), is not yet distilled (`memory_distilled_at IS NULL`), and has at least one non-empty note (workout-level OR any exercise-level). Workouts with no notes are never distilled. **Settle signal (SOW Open Q5, confirmed):** `internal/workout/sqlite_repository.go` `Update` rewrites all `workout_exercises` and bumps `workouts.updated_at` in one transaction, so an exercise-note edit DOES bump `workouts.updated_at` — the settle query needs only `workouts.updated_at`, no child-timestamp check. The adapter queries the DB directly (`*sql.DB`) since it spans `workouts`, `workout_exercises`, and `exercises`; this is consistent with the chat adapter wrapping the chat repo. Exercise display names come from `exercises.name` joined via `workout_exercises.exercise_id`.

**Content assembly format (first draft; SOW Open Q2 says iterate against real notes via the admin dump):**
- If `workouts.notes` is non-empty: a leading line `Workout notes: <notes>`.
- For each `workout_exercises` row with a non-empty note, in `exercise_order`: a line `<exercise name>: <note>` (fall back to the `exercise_id` slug if the name is somehow absent).
- Join lines with `\n`. Trim. (Workout name/date deliberately omitted from the first draft — easy to add later.)

**PromptHint (workout):**
```
The following are terse free-text notes a user wrote in their training log for a single workout. They are abbreviated and shorthand (e.g. "L shoulder cranky, dropped to 3x5", "hotel gym only this week"). Extract only durable, stable facts worth remembering across future sessions — recurring injuries or limitations, equipment/travel constraints, and lasting preferences. Ignore one-off in-session minutiae and anything that merely restates the logged sets/weights.
```

- [ ] **Step 1: Add the config knob (test first)**

`internal/config/config_test.go`: extend the `VectorMemoryConfig` fixture with `WorkoutSettleMinutes: 10` and assert `Load` decodes `workout_settle_minutes`. `internal/config/config.go`: add `WorkoutSettleMinutes int` to `VectorMemoryConfig`, `WorkoutSettleMinutes int \`toml:"workout_settle_minutes"\`` to the `fileConfig.VectorMemory` struct, and map it in `Load`. `config.toml`: under `[vectormemory]`, after `session_idle_minutes = 30`, add:

```toml
# A workout with no edit for this long is eligible for note distillation.
# Separate from session_idle_minutes because a log settles faster than a chat.
workout_settle_minutes = 10
```

- [ ] **Step 2: Write the workout-source test (TDD)**

`internal/server/workout_note_source_test.go` — open a fully-migrated temp `app.db`, seed exercises + workouts + workout_exercises, and assert:
- `PendingUnits` returns only workouts that are settled (`updated_at` < cutoff), undistilled, non-deleted, and have a workout note OR an exercise note; note-less workouts are excluded; soft-deleted excluded; too-recently-updated excluded.
- `DistillUnit.Content` contains the workout note line and a `<exercise name>: <note>` line for each noted exercise, and the `Provenance` is `{SourceType:"workout_note", WorkoutID:&id}` with a non-empty `PromptHint`.
- `MarkDistilled` sets `memory_distilled_at`.
- `CountPending` counts the full eligible backlog ignoring the limit.
- `AllUndistilled` ignores the settle window (returns a too-recently-updated noted workout) and paginates via the cursor (page size smaller than the seed count returns the rest on the next call, then `""`).

Seed helper sketch:

```go
func seedWorkout(t *testing.T, db *sql.DB, id, userID, note, updatedAt string) { /* INSERT workouts */ }
func seedExerciseNote(t *testing.T, db *sql.DB, workoutID, exerciseID, name, note string, order int) { /* INSERT exercises + workout_exercises */ }
```

- [ ] **Step 3: Implement `internal/server/workout_note_source.go`**

```go
package server

import (
	"context"
	"database/sql"
	"encoding/base64"
	"fmt"
	"strings"
	"time"

	"github.com/jwallace145/progressive-overload-fitness-tracker/internal/vectormemory"
)

const workoutNotePromptHint = `The following are terse free-text notes ...` // full text above

// workoutNoteSource distils one workout's free-text notes (workouts.notes plus
// its workout_exercises.notes) into durable memories. It reads app.db directly
// because a unit spans workouts, workout_exercises, and exercises.
type workoutNoteSource struct {
	db           *sql.DB
	settleWindow time.Duration
}

var _ vectormemory.MemorySource = (*workoutNoteSource)(nil)

func (s *workoutNoteSource) SourceType() string { return "workout_note" }

func (s *workoutNoteSource) PendingUnits(ctx context.Context, now time.Time, limit int) ([]vectormemory.DistillUnit, error) {
	cutoff := now.Add(-s.settleWindow).UTC()
	rows, err := s.db.QueryContext(ctx, `
		SELECT w.id, w.user_id
		FROM workouts w
		WHERE w.deleted_at IS NULL
		  AND w.memory_distilled_at IS NULL
		  AND w.updated_at < ?
		  AND (
		    (w.notes IS NOT NULL AND TRIM(w.notes) <> '')
		    OR EXISTS (
		      SELECT 1 FROM workout_exercises we
		      WHERE we.workout_id = w.id
		        AND we.notes IS NOT NULL AND TRIM(we.notes) <> ''
		    )
		  )
		ORDER BY w.updated_at ASC
		LIMIT ?
	`, cutoff, limit)
	if err != nil {
		return nil, err
	}
	return s.assembleUnits(ctx, rows)
}

// CountPending mirrors PendingUnits' WHERE without the LIMIT.
func (s *workoutNoteSource) CountPending(ctx context.Context, now time.Time) (int, error) {
	cutoff := now.Add(-s.settleWindow).UTC()
	var n int
	err := s.db.QueryRowContext(ctx, `
		SELECT COUNT(*) FROM workouts w
		WHERE w.deleted_at IS NULL AND w.memory_distilled_at IS NULL AND w.updated_at < ?
		  AND ((w.notes IS NOT NULL AND TRIM(w.notes) <> '')
		       OR EXISTS (SELECT 1 FROM workout_exercises we WHERE we.workout_id = w.id AND we.notes IS NOT NULL AND TRIM(we.notes) <> ''))
	`, cutoff).Scan(&n)
	return n, err
}

// AllUndistilled is PendingUnits without the settle clause, keyset-paginated on
// (updated_at, id) via an opaque cursor, for the one-time backfill.
func (s *workoutNoteSource) AllUndistilled(ctx context.Context, cursor string, limit int) ([]vectormemory.DistillUnit, string, error) {
	after, err := decodeWorkoutCursor(cursor)
	if err != nil {
		return nil, "", err
	}
	rows, err := s.db.QueryContext(ctx, `
		SELECT w.id, w.user_id, w.updated_at
		FROM workouts w
		WHERE w.deleted_at IS NULL
		  AND w.memory_distilled_at IS NULL
		  AND (w.updated_at, w.id) > (?, ?)
		  AND ((w.notes IS NOT NULL AND TRIM(w.notes) <> '')
		       OR EXISTS (SELECT 1 FROM workout_exercises we WHERE we.workout_id = w.id AND we.notes IS NOT NULL AND TRIM(we.notes) <> ''))
		ORDER BY w.updated_at ASC, w.id ASC
		LIMIT ?
	`, after.updatedAt, after.id, limit)
	// ...scan ids + last (updated_at,id) for next cursor; assemble; return "" when < limit rows...
}

func (s *workoutNoteSource) MarkDistilled(ctx context.Context, unitID string, at time.Time) error {
	_, err := s.db.ExecContext(ctx, `UPDATE workouts SET memory_distilled_at = ? WHERE id = ?`, at.UTC(), unitID)
	return err
}
```

`assembleUnits` reads workout ids/users, then for each workout loads `workouts.notes` and the ordered `(exercises.name, workout_exercises.notes)` rows (one query per workout is fine at this volume; or a single join ordered by `w.id, we.exercise_order`), composes `Content` per the format above, and builds the `DistillUnit` with `PromptHint: workoutNotePromptHint` and `Provenance{SourceType:"workout_note", WorkoutID:&id}`. Compose with a small `buildWorkoutContent(workoutNote string, exNotes []struct{Name, Note string}) string` helper so it's unit-testable. The cursor helpers (`decodeWorkoutCursor`/`encodeWorkoutCursor`) base64 a `"<updated_at>|<id>"` string; an empty cursor decodes to a zero `updatedAt`/`id` that sorts before everything. SQLite compares `updated_at` as text — store/compare the same RFC3339-ish format the rest of the repo uses (read how `workouts.updated_at` is written in `internal/workout/sqlite_repository.go` and match it; tuple comparison `(a,b) > (?,?)` works in modern SQLite — verify, else expand to `updated_at > ? OR (updated_at = ? AND id > ?)`).

- [ ] **Step 4: Extend `BuildMemorySources`**

```go
func BuildMemorySources(db *sql.DB, chatRepo *chat.SQLiteRepository, cfg config.VectorMemoryConfig) []vectormemory.MemorySource {
	return []vectormemory.MemorySource{
		&chatMemorySource{chat: chatRepo, idleWindow: time.Duration(cfg.SessionIdleMinutes) * time.Minute},
		&workoutNoteSource{db: db, settleWindow: time.Duration(cfg.WorkoutSettleMinutes) * time.Minute},
	}
}
```

Update the `server.New` call site to `BuildMemorySources(database, chatSQLiteRepo, cfg.VectorMemory)`.

- [ ] **Step 5: Run tests — expect PASS**, then the full gate.

```
go test ./internal/config/ ./internal/server/ ./internal/vectormemory/
gofmt -l . && go vet ./... && go test ./... && go run github.com/golangci/golangci-lint/v2/cmd/golangci-lint@v2.12.2 run
```

- [ ] **Step 6: Commit**

```
git add -A
git commit -m "feat(vectormemory): add workout-note distillation source"
```

---

## Task 5: Source-aware backfill

**Files:**
- Modify: `cmd/memory-backfill/main.go`
- Modify: `cmd/memory-backfill/*_test.go` (whichever tests cover `backfill`)

**Context:** The one-time `memory-backfill` command currently enumerates undistilled chat sessions directly, distills them in one Anthropic batch, embeds in one OpenAI batch, dedups + inserts, and marks. Generalize it to range over the SAME registry the live job uses (`server.BuildMemorySources`) and, per source, drain `AllUndistilled` in pages — distilling each page via the batch APIs with that source's `PromptHint`, embedding, dedup+inserting with each unit's `Provenance`, and `MarkDistilled` per unit. One run seeds chat history AND the full existing workout-note corpus, idempotently (a re-run skips already-stamped units). The command may import `internal/server` for `BuildMemorySources` (no import cycle: server does not import cmd). `DistillBatch` now takes a `promptHint`; call it once per page with the source's hint. Keep `--dry-run` (spends on the batch APIs to produce counts but writes/marks nothing). Keep the cost-visibility log lines.

**Design:** Replace `loadUndistilledSessions`/`session`/`chatStore` plumbing with a per-source page loop:

```go
for _, src := range sources {
	cursor := ""
	for {
		units, next, err := src.AllUndistilled(ctx, cursor, backfillPageSize)
		if err != nil { return err }
		if len(units) == 0 { break }
		// render already done: unit.Content is the assembled text
		contents := make([]string, len(units))
		for i, u := range units { contents[i] = u.Content }
		obsPerUnit, err := d.distiller.DistillBatch(ctx, contents, units[0].PromptHint) // hint is per-source
		// ...flatten, embed batch, dedup+insert with unit.Source provenance, MarkDistilled per unit (unless dry-run)...
		if next == "" { break }
		cursor = next
	}
}
```

Notes:
- `units[0].PromptHint` is safe because every unit from one source shares the hint; guard the empty-page case (already `break`ed above).
- The insert uses `vectormemory.NewMemory{ SourceType: u.Source.SourceType, SourceSessionID: u.Source.SessionID, SourceWorkoutID: u.Source.WorkoutID, ... }`.
- `MarkDistilled(ctx, u.UnitID, now)` per processed unit (skip in dry-run), matching the live mark-on-success-only-but-mark-zero-observation policy.
- `backfillPageSize` — a new const (e.g. 200). Document why paging (the workout corpus may be large; one giant batch is fine for Anthropic but keyset paging keeps memory bounded and makes the run resumable).
- Build the registry in `run(...)` via `server.BuildMemorySources(database, chatRepo, cfg.VectorMemory)` and pass it into the testable `backfill`.

- [ ] **Step 1: Update/write the backfill test** to drive two fake `MemorySource`s through `AllUndistilled` pagination and assert: both sources drained; per-source `PromptHint` forwarded to `DistillBatch`; provenance-correct inserts; idempotent re-run (a source whose `AllUndistilled` returns nothing is a no-op); dry-run inserts/marks nothing. The existing batch fakes (`batchDistiller`/`batchEmbedder`) can stay; the `chatStore` seam is replaced by the `MemorySource` registry. Keep tests provider-fake; no live API calls.

- [ ] **Step 2: Run — expect FAIL**, then implement Step from the design above.

- [ ] **Step 3: Implement, run — expect PASS.**

- [ ] **Step 4: Full gate, then commit**

```
export CGO_CFLAGS="-I/home/developer/sqlite-headers/include"
gofmt -l . && go vet ./... && go test ./... && go run github.com/golangci/golangci-lint/v2/cmd/golangci-lint@v2.12.2 run && go mod tidy && git diff --exit-code go.mod go.sum
git add -A
git commit -m "feat(vectormemory): source-aware memory backfill"
```

---

## Final review (after all tasks)

- Dispatch a final code reviewer over the whole branch diff (`git diff main...HEAD`).
- Re-run the full gate from a clean shell.
- Confirm the SOW's Goals are each satisfied (interface generalized, `AllUndistilled` backfill seam, `WorkoutNoteSource`, migration 035, `workout_settle_minutes`, per-source `PromptHint`, source-aware backfill, admin dump `source_type`, every shipped invariant preserved).
- Confirm the Non-Goals are respected (read path / agent / `vec_agent_memories` / embedding model untouched; no polymorphic provenance; only free-text notes ingested; no re-distill-on-edit; no per-write distillation; no UI).

## Self-review notes (author)

- **Spec coverage:** Goals map to Tasks 3 (interface + `AllUndistilled`), 4 (`WorkoutNoteSource` + config + per-source hint), 1 (migration 035), 2 (typed provenance + admin dump `source_type`), 5 (source-aware backfill). Invariants (single writer, MCP uninvolved, per-user scoping, model-version tagging, best-effort retrieval, kill-switch) are preserved by leaving the read path and `Insert`'s vector handling untouched.
- **Deliberate SOW deviation:** `MemorySource` has a 5th method `CountPending` to preserve the existing idle-backlog Prometheus gauge (PR #59) that the capped `PendingUnits` cannot feed. Flagged in the Task 3 PR body.
- **Type consistency:** `DistillUnit`/`Provenance` field names are used identically in `job.go`, `service.go`, the chat adapter, the workout source, and the backfill. `Distiller.Distill(ctx, content, promptHint)` and `DistillBatch(ctx, conversations, promptHint)` signatures are consistent across `provider.go`, `anthropic.go`, `batch.go`, `service.go`, and `cmd/memory-backfill`.
