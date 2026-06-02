# Intent-Driven Context Enrichment Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Expand the agent's router from `tier`-only to `{tier, intent}`, prefetch intent-specific MCP data, inject it into the system prompt, persist intent on `chat_sessions` for multi-turn continuity, and surface intent distribution on the agent Grafana dashboard.

**Architecture:** Single Haiku call returns structured `{tier, intent}`. New `IntentRegistry` module per intent declares prefetch tool calls + rules string + data formatter. Harness runs prefetch in parallel after MCP session init, composes system prompt as `base + rules + data`, then runs the existing tool-use loop. Intent persists on `chat_sessions.last_intent` via a new internal API endpoint, providing a hint to the router on subsequent turns.

**Tech Stack:** Python 3.12 (FastAPI + Anthropic SDK + MCP SDK + httpx + Prometheus client) for the agent, Go 1.22+ (chi + database/sql + mattn/go-sqlite3) for the API, Grafana 11.x JSON model for the dashboard.

**Repo paths used in this plan:**
- API: `/Users/jimmywallace/Desktop/prog-strength/repos/prog-strength-api`
- Agent: `/Users/jimmywallace/Desktop/prog-strength/repos/prog-strength-agent`
- Infra: `/Users/jimmywallace/Desktop/prog-strength/repos/prog-strength-infra`

**Source spec:** `prog-strength-docs/sows/intent-driven-context-enrichment.md`

---

## Phase 1 — API: schema + types + repository

### Task 1: chat_sessions migration

**Files:**
- Create: `prog-strength-api/internal/db/migrations/010_chat_sessions_last_intent.sql`

- [ ] **Step 1: Write migration**

```sql
-- migrations/010_chat_sessions_last_intent.sql
-- Adds the "last intent classified for this conversation" pair of
-- columns used by the intent-driven-context-enrichment SOW. The
-- agent reads these as a hint to its router and writes the new
-- value back via the existing telemetry POST.
--
-- Both columns nullable. NULL means "no prior intent known," which
-- the router falls back to gracefully by classifying on conversation
-- context alone.

ALTER TABLE chat_sessions ADD COLUMN last_intent TEXT;
ALTER TABLE chat_sessions ADD COLUMN last_intent_at DATETIME;
```

- [ ] **Step 2: Verify migration applies cleanly**

Run from `prog-strength-api/`:
```bash
go test ./internal/db/...
```
Expected: PASS (the migration runner test picks up new files automatically).

- [ ] **Step 3: Commit**

```bash
git add internal/db/migrations/010_chat_sessions_last_intent.sql
git commit -m "feat(db): add last_intent + last_intent_at to chat_sessions"
```

---

### Task 2: agent_turns telemetry migration

**Files:**
- Create: `prog-strength-api/internal/db/telemetry_migrations/002_agent_turns_intent.sql`

- [ ] **Step 1: Write migration**

```sql
-- telemetry_migrations/002_agent_turns_intent.sql
-- Adds the per-turn intent classification (and prefetch perf flags)
-- to agent_turns. Mirrors the SOW's TurnInstrumentation additions on
-- the Python side; the telemetry handler now accepts these on the
-- POST /internal/telemetry/turns body.
--
-- intent is the closed-enum string from the router
-- (log_nutrition | log_workout | log_bodyweight | analyze_progress |
-- general). Empty string means the router didn't run or completely
-- failed — kept distinct from "general" which is a deliberate
-- classifier output.

ALTER TABLE agent_turns ADD COLUMN intent TEXT NOT NULL DEFAULT '';
ALTER TABLE agent_turns ADD COLUMN intent_prefetch_duration_ms INTEGER NOT NULL DEFAULT 0;
ALTER TABLE agent_turns ADD COLUMN intent_prefetch_failed INTEGER NOT NULL DEFAULT 0;
```

- [ ] **Step 2: Verify migration applies**

```bash
go test ./internal/telemetry/...
```
Expected: PASS (telemetry sqlite_repository tests open a fresh DB and run all telemetry_migrations).

- [ ] **Step 3: Commit**

```bash
git add internal/db/telemetry_migrations/002_agent_turns_intent.sql
git commit -m "feat(telemetry): add intent + prefetch perf cols to agent_turns"
```

---

### Task 3: AgentTurn struct + telemetry repo write

**Files:**
- Modify: `prog-strength-api/internal/telemetry/telemetry.go`
- Modify: `prog-strength-api/internal/telemetry/sqlite_repository.go`
- Modify: `prog-strength-api/internal/telemetry/sqlite_repository_test.go`

- [ ] **Step 1: Write failing test (assert intent round-trips through InsertTurn)**

In `internal/telemetry/sqlite_repository_test.go`, add a new test alongside the existing turn round-trip test (find the existing `TestInsertTurn` or equivalent and copy its setup):

```go
func TestInsertTurn_PersistsIntentFields(t *testing.T) {
	repo, cleanup := newTestTelemetryRepo(t) // existing helper
	defer cleanup()

	want := AgentTurn{
		ID:                       "turn-int-1",
		UserID:                   "u-1",
		SessionID:                "s-1",
		Model:                    "claude-haiku-4-5-20251001",
		RoutedTier:               "simple",
		RouterModel:              "claude-haiku-4-5-20251001",
		Intent:                   "log_nutrition",
		IntentPrefetchDurationMs: 87,
		IntentPrefetchFailed:     false,
		CompletionReason:         "end_turn",
		StartedAt:                time.Now().UTC(),
		EndedAt:                  time.Now().UTC(),
		CreatedAt:                time.Now().UTC(),
	}
	if err := repo.InsertTurn(context.Background(), want); err != nil {
		t.Fatalf("insert: %v", err)
	}

	var (
		gotIntent           string
		gotPrefetchDuration int
		gotPrefetchFailed   int
	)
	err := repo.db.QueryRow(
		`SELECT intent, intent_prefetch_duration_ms, intent_prefetch_failed
		   FROM agent_turns WHERE id = ?`, want.ID,
	).Scan(&gotIntent, &gotPrefetchDuration, &gotPrefetchFailed)
	if err != nil {
		t.Fatalf("readback: %v", err)
	}
	if gotIntent != "log_nutrition" || gotPrefetchDuration != 87 || gotPrefetchFailed != 0 {
		t.Fatalf("got intent=%q prefetch=%d failed=%d", gotIntent, gotPrefetchDuration, gotPrefetchFailed)
	}
}
```

- [ ] **Step 2: Run test to verify it fails**

```bash
go test ./internal/telemetry/... -run TestInsertTurn_PersistsIntentFields -v
```
Expected: FAIL — `AgentTurn` has no `Intent` field, compile error.

- [ ] **Step 3: Add fields to AgentTurn struct**

In `internal/telemetry/telemetry.go`, after the `Error *string` line in `AgentTurn`, insert:

```go
	// Intent classification produced by the agent's router. One of
	// log_nutrition | log_workout | log_bodyweight | analyze_progress |
	// general. Empty string means the router didn't run.
	Intent string
	// Wall-clock time spent running the intent's prefetch tool calls
	// (asyncio.gather across multiple MCP tools). Zero when no
	// prefetch ran (intent == general or router failed).
	IntentPrefetchDurationMs int
	// True when any prefetch tool call raised. The harness still
	// composes the prompt and runs the turn — degraded enrichment is
	// silent — but this flag lets the dashboard surface the rate.
	IntentPrefetchFailed bool
```

- [ ] **Step 4: Extend the INSERT statement in sqlite_repository.go**

In `internal/telemetry/sqlite_repository.go`, find the existing `InsertTurn` function and update both the INSERT SQL and the `Exec` args to include the three new columns. The column list and value list must match position-for-position:

```go
const insertAgentTurn = `
INSERT INTO agent_turns (
    id, user_id, session_id, model, routed_tier, router_model,
    router_latency_ms, input_tokens, output_tokens,
    cache_creation_tokens, cache_read_tokens, total_latency_ms,
    time_to_first_token_ms, completion_reason, error,
    started_at, ended_at, created_at,
    intent, intent_prefetch_duration_ms, intent_prefetch_failed
) VALUES (
    ?, ?, ?, ?, ?, ?,
    ?, ?, ?,
    ?, ?, ?,
    ?, ?, ?,
    ?, ?, ?,
    ?, ?, ?
)`
```

…and pass the three new values at the end of the `Exec` call (`t.Intent, t.IntentPrefetchDurationMs, intToInt(t.IntentPrefetchFailed)` — wrap the bool as `0`/`1` so SQLite stores an INTEGER consistent with the migration's column type). Define a tiny local helper:

```go
func boolToInt(b bool) int {
    if b {
        return 1
    }
    return 0
}
```

- [ ] **Step 5: Run test to verify it passes**

```bash
go test ./internal/telemetry/... -run TestInsertTurn_PersistsIntentFields -v
```
Expected: PASS.

- [ ] **Step 6: Run full telemetry suite to confirm no regressions**

```bash
go test ./internal/telemetry/...
```
Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add internal/telemetry/telemetry.go internal/telemetry/sqlite_repository.go internal/telemetry/sqlite_repository_test.go
git commit -m "feat(telemetry): persist intent classification on agent_turns"
```

---

### Task 4: telemetry HTTP handler accepts intent

**Files:**
- Modify: `prog-strength-api/internal/telemetry/handler.go`
- Modify: `prog-strength-api/internal/telemetry/handler_test.go` (or wherever the handler test lives — check for `_test.go` next to handler.go)

- [ ] **Step 1: Write failing handler test**

```go
func TestTurnHandler_PersistsIntent(t *testing.T) {
	h, repo := newTestTelemetryHandler(t) // existing helper, returns handler + memory repo or test sqlite repo

	body := strings.NewReader(`{
		"id": "turn-1",
		"user_id": "u-1",
		"session_id": "s-1",
		"model": "claude-haiku-4-5-20251001",
		"routed_tier": "simple",
		"router_model": "claude-haiku-4-5-20251001",
		"router_latency_ms": 400,
		"completion_reason": "end_turn",
		"started_at": "2026-06-02T00:00:00Z",
		"ended_at":   "2026-06-02T00:00:01Z",
		"intent": "log_nutrition",
		"intent_prefetch_duration_ms": 87,
		"intent_prefetch_failed": false
	}`)
	req := httptest.NewRequest("POST", "/internal/telemetry/turns", body)
	w := httptest.NewRecorder()

	h.turn(w, req)

	if w.Code != http.StatusCreated {
		t.Fatalf("status: got %d want 201, body=%s", w.Code, w.Body.String())
	}
	got, ok := repo.lastInsertedTurn() // existing helper
	if !ok || got.Intent != "log_nutrition" || got.IntentPrefetchDurationMs != 87 || got.IntentPrefetchFailed {
		t.Fatalf("turn not persisted with intent fields: %+v", got)
	}
}
```

If there's no existing handler test file, create `internal/telemetry/handler_test.go` and put the needed `newTestTelemetryHandler` setup at the top (it should construct an in-memory or sqlite-backed repo and a `Handler{repo: repo, now: func() time.Time { ... }}`).

- [ ] **Step 2: Run test to verify it fails**

```bash
go test ./internal/telemetry/... -run TestTurnHandler_PersistsIntent -v
```
Expected: FAIL — `turnRequest` has no `Intent` field, so the JSON decodes but the persisted `AgentTurn` has zero values.

- [ ] **Step 3: Extend turnRequest and the AgentTurn construction**

In `internal/telemetry/handler.go`, add to `turnRequest`:

```go
	Intent                   string `json:"intent"`
	IntentPrefetchDurationMs int    `json:"intent_prefetch_duration_ms"`
	IntentPrefetchFailed     bool   `json:"intent_prefetch_failed"`
```

…and in the `turn` handler where `AgentTurn{}` is constructed, copy the new fields through:

```go
		Intent:                   req.Intent,
		IntentPrefetchDurationMs: req.IntentPrefetchDurationMs,
		IntentPrefetchFailed:     req.IntentPrefetchFailed,
```

- [ ] **Step 4: Run test to verify it passes**

```bash
go test ./internal/telemetry/... -run TestTurnHandler_PersistsIntent -v
```
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add internal/telemetry/handler.go internal/telemetry/handler_test.go
git commit -m "feat(telemetry): accept intent fields on POST /internal/telemetry/turns"
```

---

### Task 5: chat.Session + repository — intent getter/setter

**Files:**
- Modify: `prog-strength-api/internal/chat/session.go`
- Modify: `prog-strength-api/internal/chat/repository.go`
- Modify: `prog-strength-api/internal/chat/memory_repository.go`
- Modify: `prog-strength-api/internal/chat/sqlite_repository.go`
- Modify: `prog-strength-api/internal/chat/memory_repository_test.go`
- Modify: `prog-strength-api/internal/chat/sqlite_repository_test.go`

- [ ] **Step 1: Write failing tests for both repos**

Append to `memory_repository_test.go`:

```go
func TestMemoryRepository_SessionIntentRoundTrip(t *testing.T) {
	repo := NewMemoryRepository()
	ctx := context.Background()
	s := &Session{ID: "11111111-2222-4333-8444-555555555555", UserID: "u-1"}
	if err := repo.CreateSession(ctx, s); err != nil {
		t.Fatalf("create: %v", err)
	}

	gotIntent, gotAt, err := repo.GetSessionIntent(ctx, s.ID)
	if err != nil {
		t.Fatalf("get empty: %v", err)
	}
	if gotIntent != nil || gotAt != nil {
		t.Fatalf("expected nil intent on fresh session, got %v / %v", gotIntent, gotAt)
	}

	when := time.Now().UTC().Truncate(time.Second)
	if err := repo.SetSessionIntent(ctx, s.ID, "log_nutrition", when); err != nil {
		t.Fatalf("set: %v", err)
	}

	gotIntent, gotAt, err = repo.GetSessionIntent(ctx, s.ID)
	if err != nil {
		t.Fatalf("get after set: %v", err)
	}
	if gotIntent == nil || *gotIntent != "log_nutrition" || gotAt == nil || !gotAt.Equal(when) {
		t.Fatalf("intent round-trip mismatch: %v / %v", gotIntent, gotAt)
	}
}

func TestMemoryRepository_GetSessionIntent_UnknownSession(t *testing.T) {
	repo := NewMemoryRepository()
	_, _, err := repo.GetSessionIntent(context.Background(), "11111111-2222-4333-8444-555555555555")
	if !errors.Is(err, ErrNotFound) {
		t.Fatalf("want ErrNotFound, got %v", err)
	}
}
```

…and mirror both into `sqlite_repository_test.go` using whatever helper that file already uses to spin up an SQLite-backed repo (look for an existing `newTestSQLiteRepo` or similar).

- [ ] **Step 2: Run tests to verify they fail**

```bash
go test ./internal/chat/... -run SessionIntent -v
```
Expected: FAIL — `GetSessionIntent`/`SetSessionIntent` don't exist.

- [ ] **Step 3: Extend the Session struct**

In `internal/chat/session.go`, add to the `Session` struct (below `DeletedAt`):

```go
	// LastIntent is the most recent non-general intent classification
	// emitted by the agent's router for this conversation. Nil for
	// sessions with no classified intent yet (fresh sessions, or only
	// general turns so far). Read by the agent before classifying the
	// next turn; written by the telemetry POST when the agent finishes
	// a non-general turn. JSON-hidden; surfaced via the dedicated
	// /internal/chat-sessions/{id}/intent endpoint instead of the
	// regular session payload.
	LastIntent   *string    `json:"-"`
	LastIntentAt *time.Time `json:"-"`
```

- [ ] **Step 4: Extend the Repository interface**

In `internal/chat/repository.go`, add to the `Repository` interface:

```go
	// GetSessionIntent returns the session's last non-general intent
	// classification (and the timestamp of that classification). Both
	// pointers are nil when the session has no prior intent. Returns
	// ErrNotFound when the session doesn't exist or is soft-deleted.
	// NOTE: no user-id scoping — this endpoint sits behind the docker
	// network boundary (callers are the agent's internal API client),
	// not behind user auth.
	GetSessionIntent(ctx context.Context, sessionID string) (*string, *time.Time, error)

	// SetSessionIntent persists a new intent classification on the
	// session. Idempotent: calling with the same (intent, at) is a no-op
	// in observable terms. The caller is expected to only invoke this
	// for non-general intents — the API does not filter.
	SetSessionIntent(ctx context.Context, sessionID, intent string, at time.Time) error
```

- [ ] **Step 5: Implement on the memory repository**

In `internal/chat/memory_repository.go`, add the two methods. Follow the existing locking pattern (whatever the file uses — likely `r.mu.Lock()` / `r.mu.RLock()`):

```go
func (r *MemoryRepository) GetSessionIntent(ctx context.Context, sessionID string) (*string, *time.Time, error) {
	r.mu.RLock()
	defer r.mu.RUnlock()
	s, ok := r.sessions[sessionID]
	if !ok || s.DeletedAt != nil {
		return nil, nil, ErrNotFound
	}
	// Defensive copies — callers must not hold pointers to internal state.
	var intent *string
	var at *time.Time
	if s.LastIntent != nil {
		v := *s.LastIntent
		intent = &v
	}
	if s.LastIntentAt != nil {
		v := *s.LastIntentAt
		at = &v
	}
	return intent, at, nil
}

func (r *MemoryRepository) SetSessionIntent(ctx context.Context, sessionID, intent string, at time.Time) error {
	r.mu.Lock()
	defer r.mu.Unlock()
	s, ok := r.sessions[sessionID]
	if !ok || s.DeletedAt != nil {
		return ErrNotFound
	}
	s.LastIntent = &intent
	atCopy := at
	s.LastIntentAt = &atCopy
	r.sessions[sessionID] = s
	return nil
}
```

- [ ] **Step 6: Implement on the SQLite repository**

In `internal/chat/sqlite_repository.go`, add:

```go
func (r *SQLiteRepository) GetSessionIntent(ctx context.Context, sessionID string) (*string, *time.Time, error) {
	const q = `
SELECT last_intent, last_intent_at
  FROM chat_sessions
 WHERE id = ? AND deleted_at IS NULL`
	var intent sql.NullString
	var at sql.NullTime
	err := r.db.QueryRowContext(ctx, q, sessionID).Scan(&intent, &at)
	if errors.Is(err, sql.ErrNoRows) {
		return nil, nil, ErrNotFound
	}
	if err != nil {
		return nil, nil, err
	}
	var outIntent *string
	var outAt *time.Time
	if intent.Valid {
		v := intent.String
		outIntent = &v
	}
	if at.Valid {
		v := at.Time
		outAt = &v
	}
	return outIntent, outAt, nil
}

func (r *SQLiteRepository) SetSessionIntent(ctx context.Context, sessionID, intent string, at time.Time) error {
	const q = `
UPDATE chat_sessions
   SET last_intent = ?, last_intent_at = ?, updated_at = ?
 WHERE id = ? AND deleted_at IS NULL`
	res, err := r.db.ExecContext(ctx, q, intent, at, time.Now().UTC(), sessionID)
	if err != nil {
		return err
	}
	rows, err := res.RowsAffected()
	if err != nil {
		return err
	}
	if rows == 0 {
		return ErrNotFound
	}
	return nil
}
```

- [ ] **Step 7: Compile-time assertions stay green**

The package likely has `var _ Repository = (*MemoryRepository)(nil)` and the SQLite equivalent. Run:

```bash
go build ./internal/chat/...
```
Expected: no compile errors.

- [ ] **Step 8: Run tests to verify they pass**

```bash
go test ./internal/chat/... -run SessionIntent -v
go test ./internal/chat/...
```
Expected: PASS on both.

- [ ] **Step 9: Commit**

```bash
git add internal/chat/session.go internal/chat/repository.go internal/chat/memory_repository.go internal/chat/sqlite_repository.go internal/chat/memory_repository_test.go internal/chat/sqlite_repository_test.go
git commit -m "feat(chat): add Get/SetSessionIntent repository methods"
```

---

### Task 6: chat MountInternal — GET /internal/chat-sessions/{id}/intent

**Files:**
- Modify: `prog-strength-api/internal/chat/handler.go`
- Create: `prog-strength-api/internal/chat/handler_internal_test.go`

- [ ] **Step 1: Write failing handler test**

Create `internal/chat/handler_internal_test.go`:

```go
package chat

import (
	"context"
	"encoding/json"
	"net/http"
	"net/http/httptest"
	"strings"
	"testing"
	"time"

	"github.com/go-chi/chi/v5"
)

func TestGetSessionIntent_Found(t *testing.T) {
	repo := NewMemoryRepository()
	ctx := context.Background()
	id := "11111111-2222-4333-8444-555555555555"
	if err := repo.CreateSession(ctx, &Session{ID: id, UserID: "u-1"}); err != nil {
		t.Fatalf("create: %v", err)
	}
	when := time.Now().UTC().Truncate(time.Second)
	if err := repo.SetSessionIntent(ctx, id, "log_nutrition", when); err != nil {
		t.Fatalf("set: %v", err)
	}

	r := chi.NewRouter()
	NewHandler(repo).MountInternal(r)

	w := httptest.NewRecorder()
	req := httptest.NewRequest("GET", "/internal/chat-sessions/"+id+"/intent", nil)
	r.ServeHTTP(w, req)

	if w.Code != http.StatusOK {
		t.Fatalf("status: got %d, body=%s", w.Code, w.Body.String())
	}

	var resp struct {
		Data struct {
			Intent   *string    `json:"intent"`
			IntentAt *time.Time `json:"intent_at"`
		} `json:"data"`
	}
	if err := json.NewDecoder(w.Body).Decode(&resp); err != nil {
		t.Fatalf("decode: %v", err)
	}
	if resp.Data.Intent == nil || *resp.Data.Intent != "log_nutrition" {
		t.Fatalf("intent: got %v", resp.Data.Intent)
	}
	if resp.Data.IntentAt == nil || !resp.Data.IntentAt.Equal(when) {
		t.Fatalf("intent_at: got %v", resp.Data.IntentAt)
	}
}

func TestGetSessionIntent_NoIntentYet(t *testing.T) {
	repo := NewMemoryRepository()
	id := "11111111-2222-4333-8444-555555555555"
	_ = repo.CreateSession(context.Background(), &Session{ID: id, UserID: "u-1"})

	r := chi.NewRouter()
	NewHandler(repo).MountInternal(r)

	w := httptest.NewRecorder()
	req := httptest.NewRequest("GET", "/internal/chat-sessions/"+id+"/intent", nil)
	r.ServeHTTP(w, req)

	if w.Code != http.StatusOK {
		t.Fatalf("status: got %d, body=%s", w.Code, w.Body.String())
	}
	if !strings.Contains(w.Body.String(), `"intent":null`) {
		t.Fatalf("expected null intent in body, got %s", w.Body.String())
	}
}

func TestGetSessionIntent_UnknownSession_Returns200WithNulls(t *testing.T) {
	// SOW: return the same shape on 404 to keep the agent client trivial.
	repo := NewMemoryRepository()
	r := chi.NewRouter()
	NewHandler(repo).MountInternal(r)

	w := httptest.NewRecorder()
	req := httptest.NewRequest("GET", "/internal/chat-sessions/11111111-2222-4333-8444-555555555555/intent", nil)
	r.ServeHTTP(w, req)

	if w.Code != http.StatusOK {
		t.Fatalf("status: got %d", w.Code)
	}
	if !strings.Contains(w.Body.String(), `"intent":null`) {
		t.Fatalf("expected null intent in body, got %s", w.Body.String())
	}
}
```

- [ ] **Step 2: Run test to verify it fails**

```bash
go test ./internal/chat/... -run SessionIntent -v
```
Expected: FAIL — `MountInternal` doesn't exist.

- [ ] **Step 3: Add MountInternal to the handler**

In `internal/chat/handler.go`, after the existing `Mount` method, add:

```go
// MountInternal registers chat routes that sit behind the docker
// network boundary (and Caddy's refusal to proxy /internal/*) rather
// than the user-JWT auth middleware. The only consumer is the agent
// service reading the session's prior intent on the way into /chat.
func (h *Handler) MountInternal(r chi.Router) {
	r.Route("/internal/chat-sessions", func(r chi.Router) {
		r.Get("/{id}/intent", h.getInternalIntent)
	})
}

type internalIntentResponse struct {
	Intent   *string    `json:"intent"`
	IntentAt *time.Time `json:"intent_at"`
}

func (h *Handler) getInternalIntent(w http.ResponseWriter, r *http.Request) {
	id := chi.URLParam(r, "id")
	intent, at, err := h.repo.GetSessionIntent(r.Context(), id)
	if err != nil && !errors.Is(err, ErrNotFound) {
		httpresp.ServerError(w, r.Context(), "get chat session intent", err)
		return
	}
	// SOW: 404 returns the same shape as "session has no intent yet"
	// so the agent's client doesn't have to distinguish.
	httpresp.OK(w, "got chat session intent", internalIntentResponse{
		Intent:   intent,
		IntentAt: at,
	})
}
```

Add `"time"` to the imports if not already present.

- [ ] **Step 4: Run tests to verify they pass**

```bash
go test ./internal/chat/... -run SessionIntent -v
go test ./internal/chat/...
```
Expected: PASS on both.

- [ ] **Step 5: Commit**

```bash
git add internal/chat/handler.go internal/chat/handler_internal_test.go
git commit -m "feat(chat): add internal GET /chat-sessions/{id}/intent"
```

---

### Task 7: Wire telemetry handler to write last_intent on chat_sessions

**Files:**
- Modify: `prog-strength-api/internal/telemetry/handler.go`
- Modify: `prog-strength-api/internal/telemetry/handler_test.go`
- Modify: `prog-strength-api/internal/server/server.go`

- [ ] **Step 1: Write failing test (telemetry POST with non-general intent writes through to chat repo)**

In the telemetry handler test, add:

```go
type fakeIntentSink struct {
	calls []intentSinkCall
}

type intentSinkCall struct {
	sessionID string
	intent    string
	at        time.Time
}

func (s *fakeIntentSink) SetSessionIntent(ctx context.Context, sessionID, intent string, at time.Time) error {
	s.calls = append(s.calls, intentSinkCall{sessionID, intent, at})
	return nil
}

func TestTurnHandler_WritesLastIntentForNonGeneral(t *testing.T) {
	repo := newTestTelemetryRepo(t) // existing
	sink := &fakeIntentSink{}
	h := NewHandlerWithIntentSink(repo, sink) // see Step 3

	body := strings.NewReader(`{
		"id": "turn-1",
		"user_id": "u-1",
		"session_id": "11111111-2222-4333-8444-555555555555",
		"model": "claude-haiku-4-5-20251001",
		"routed_tier": "simple",
		"router_model": "claude-haiku-4-5-20251001",
		"router_latency_ms": 400,
		"completion_reason": "end_turn",
		"started_at": "2026-06-02T00:00:00Z",
		"ended_at":   "2026-06-02T00:00:01Z",
		"intent": "log_nutrition"
	}`)
	req := httptest.NewRequest("POST", "/internal/telemetry/turns", body)
	w := httptest.NewRecorder()
	h.turn(w, req)
	if w.Code != http.StatusCreated {
		t.Fatalf("status: got %d, body=%s", w.Code, w.Body.String())
	}
	if len(sink.calls) != 1 || sink.calls[0].intent != "log_nutrition" {
		t.Fatalf("sink: %+v", sink.calls)
	}
}

func TestTurnHandler_SkipsLastIntentWriteForGeneral(t *testing.T) {
	repo := newTestTelemetryRepo(t)
	sink := &fakeIntentSink{}
	h := NewHandlerWithIntentSink(repo, sink)

	body := strings.NewReader(`{
		"id": "turn-2",
		"user_id": "u-1",
		"session_id": "11111111-2222-4333-8444-555555555555",
		"model": "claude-haiku-4-5-20251001",
		"routed_tier": "simple",
		"router_model": "claude-haiku-4-5-20251001",
		"router_latency_ms": 400,
		"completion_reason": "end_turn",
		"started_at": "2026-06-02T00:00:00Z",
		"ended_at":   "2026-06-02T00:00:01Z",
		"intent": "general"
	}`)
	req := httptest.NewRequest("POST", "/internal/telemetry/turns", body)
	w := httptest.NewRecorder()
	h.turn(w, req)
	if w.Code != http.StatusCreated {
		t.Fatalf("status: got %d", w.Code)
	}
	if len(sink.calls) != 0 {
		t.Fatalf("sink should be empty, got %+v", sink.calls)
	}
}
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
go test ./internal/telemetry/... -run LastIntent -v
```
Expected: FAIL — `NewHandlerWithIntentSink` doesn't exist.

- [ ] **Step 3: Extend Handler with optional intent sink**

In `internal/telemetry/handler.go`, define the interface (small enough to live in this file rather than its own):

```go
// IntentSink is the subset of chat.Repository the telemetry handler
// uses to write through the most recent non-general intent to the
// session row. Optional — Handler.intentSink may be nil, in which
// case the write-through is skipped (useful for tests that don't
// care about chat state).
type IntentSink interface {
	SetSessionIntent(ctx context.Context, sessionID, intent string, at time.Time) error
}
```

Change `Handler` to include the field and add a new constructor:

```go
type Handler struct {
	repo       Repository
	intentSink IntentSink // may be nil
	now        func() time.Time
}

func NewHandler(repo Repository) *Handler {
	return &Handler{repo: repo, now: time.Now}
}

func NewHandlerWithIntentSink(repo Repository, sink IntentSink) *Handler {
	return &Handler{repo: repo, intentSink: sink, now: time.Now}
}
```

At the end of the `turn` handler's success path (right before `httpresp.Created(...)`), insert:

```go
	if h.intentSink != nil && t.Intent != "" && t.Intent != "general" {
		// Fire-and-forget by design — a chat-sessions write failure
		// must not turn a successful telemetry insert into a 500.
		// Caller has already returned to the agent by the time this
		// runs; we just log and move on.
		if err := h.intentSink.SetSessionIntent(r.Context(), t.SessionID, t.Intent, t.EndedAt); err != nil {
			// chat.ErrNotFound when the session isn't persisted (the
			// frontend currently mints session IDs without always
			// calling POST /chat-sessions first; treat as expected).
			// All other errors are worth a log line.
			log.Printf("telemetry: write last_intent failed: session=%s err=%v", t.SessionID, err)
		}
	}
```

Add `"log"` to imports if not already present.

- [ ] **Step 4: Wire the sink in server.go**

In `internal/server/server.go`, find where `telemetry.NewHandler(...)` is called. Replace with `telemetry.NewHandlerWithIntentSink(telemetryRepo, chatRepo)` (the exact local variable names depend on the existing wiring — chase the existing telemetry handler instantiation and inject the same `chat.Repository` that the chat handler already uses).

- [ ] **Step 5: Mount the chat internal routes alongside the existing chat mount**

Still in `server.go`, find the existing `chatHandler.Mount(r)` call and add a sibling call:

```go
	chatHandler.MountInternal(r) // /internal/chat-sessions/{id}/intent
```

The internal route ends up under the same router tree as `/internal/telemetry`, which Caddy already refuses to proxy externally — no separate auth wiring needed.

- [ ] **Step 6: Run all tests**

```bash
go test ./...
```
Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add internal/telemetry/handler.go internal/telemetry/handler_test.go internal/server/server.go
git commit -m "feat(telemetry): write through last_intent to chat_sessions for non-general turns"
```

---

## Phase 2 — Agent: prompts + intents

### Task 8: Add single-serving rule to base SYSTEM_PROMPT

**Files:**
- Modify: `prog-strength-agent/src/prog_strength_agent/prompt.py`
- Modify: `prog-strength-agent/tests/test_prompt.py`

- [ ] **Step 1: Write failing test**

In `tests/test_prompt.py`, add:

```python
def test_system_prompt_includes_single_serving_default():
    """Defense in depth: the assume-one-serving convention lands in
    the base prompt so the model has it even on a 'general' intent
    classification."""
    from prog_strength_agent.prompt import SYSTEM_PROMPT
    assert "Default to one serving" in SYSTEM_PROMPT
    assert "quantity=1" in SYSTEM_PROMPT
```

- [ ] **Step 2: Run test to verify it fails**

```bash
uv run pytest tests/test_prompt.py::test_system_prompt_includes_single_serving_default -v
```
Expected: FAIL — string not present yet.

- [ ] **Step 3: Add the convention to SYSTEM_PROMPT**

In `src/prog_strength_agent/prompt.py`, inside `SYSTEM_PROMPT` under the "Conventions you must follow" section (right after the "Performed_at defaults to now if omitted." paragraph), insert a new convention paragraph:

```
**Default to one serving for nutrition logs.** When the user logs a \
food without specifying servings ("log a protein shake," "had eggs \
for breakfast"), assume `quantity=1`. Only ask for servings if the \
user's wording is genuinely ambiguous. The serving-size unit on the \
pantry item itself tells you what "one serving" means.

```

- [ ] **Step 4: Run test to verify it passes**

```bash
uv run pytest tests/test_prompt.py -v
```
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/prog_strength_agent/prompt.py tests/test_prompt.py
git commit -m "feat(prompt): default to one serving on nutrition logs"
```

---

### Task 9: Add _compose_system_prompt helper

**Files:**
- Modify: `prog-strength-agent/src/prog_strength_agent/prompt.py`
- Modify: `prog-strength-agent/tests/test_prompt.py`

- [ ] **Step 1: Write failing tests**

```python
def test_compose_system_prompt_includes_all_sections_when_provided():
    from prog_strength_agent.prompt import compose_system_prompt
    out = compose_system_prompt(base="BASE", rules="RULES", data="DATA")
    assert "BASE" in out
    assert "RULES" in out
    assert "DATA" in out
    # Ordering matters: base first, then rules, then data.
    assert out.index("BASE") < out.index("RULES") < out.index("DATA")


def test_compose_system_prompt_omits_empty_sections():
    from prog_strength_agent.prompt import compose_system_prompt
    assert compose_system_prompt(base="BASE", rules="", data="") == "BASE"
    assert "RULES" not in compose_system_prompt(base="BASE", rules="", data="DATA")
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
uv run pytest tests/test_prompt.py::test_compose_system_prompt_includes_all_sections_when_provided -v
```
Expected: FAIL — function does not exist.

- [ ] **Step 3: Implement compose_system_prompt**

In `src/prog_strength_agent/prompt.py`, after `build_chat_system_prompt`, add:

```python
def compose_system_prompt(*, base: str, rules: str = "", data: str = "") -> str:
    """Concatenate the base system prompt with optional intent-specific
    rules and data blocks. Empty sections are skipped entirely (not
    rendered as blank separators) so a `general` intent or a failed
    prefetch produces a prompt visually identical to today's.
    """
    parts = [base]
    if rules:
        parts.append(rules)
    if data:
        parts.append(data)
    return "\n\n".join(parts)
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
uv run pytest tests/test_prompt.py -v
```
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/prog_strength_agent/prompt.py tests/test_prompt.py
git commit -m "feat(prompt): add compose_system_prompt for intent-driven sections"
```

---

### Task 10: IntentRegistry skeleton + general intent

**Files:**
- Create: `prog-strength-agent/src/prog_strength_agent/intents.py`
- Create: `prog-strength-agent/tests/test_intents.py`

- [ ] **Step 1: Write failing tests**

Create `tests/test_intents.py`:

```python
"""Unit tests for the IntentRegistry — declarative module of per-intent
prefetch + rules + data formatting."""

from __future__ import annotations

import pytest

from prog_strength_agent.intents import IntentRegistry


@pytest.mark.asyncio
async def test_run_general_returns_empty_blocks():
    rules, data = await IntentRegistry.run("general", session=None)
    assert rules == ""
    assert data == ""


@pytest.mark.asyncio
async def test_run_unknown_intent_returns_empty_blocks():
    rules, data = await IntentRegistry.run("definitely_not_an_intent", session=None)
    assert rules == ""
    assert data == ""


def test_known_intents_enum():
    assert set(IntentRegistry.known()) == {
        "log_nutrition",
        "log_workout",
        "log_bodyweight",
        "analyze_progress",
        "general",
    }
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
uv run pytest tests/test_intents.py -v
```
Expected: FAIL — module does not exist.

- [ ] **Step 3: Implement the skeleton**

Create `src/prog_strength_agent/intents.py`:

```python
"""Intent registry for the Prog Strength agent.

Each known intent declares a `prefetch` (async; runs MCP tool calls in
parallel against an already-open session), a `rules` string (system
prompt addendum the model sees verbatim), and a `format` function
(turns the prefetched dict into a data block for the prompt).

See prog-strength-docs/sows/intent-driven-context-enrichment.md.
"""

from __future__ import annotations

import asyncio
import logging
from dataclasses import dataclass
from typing import Any, Awaitable, Callable, Iterable

log = logging.getLogger(__name__)

PrefetchFn = Callable[[Any], Awaitable[dict[str, Any]]]
FormatFn = Callable[[dict[str, Any]], str]

KNOWN_INTENTS: tuple[str, ...] = (
    "log_nutrition",
    "log_workout",
    "log_bodyweight",
    "analyze_progress",
    "general",
)


@dataclass(frozen=True)
class IntentSpec:
    intent: str
    rules: str
    prefetch: PrefetchFn
    format: FormatFn


_SPECS: dict[str, IntentSpec] = {}


def _register(spec: IntentSpec) -> None:
    _SPECS[spec.intent] = spec


async def _noop_prefetch(_session: Any) -> dict[str, Any]:
    return {}


def _noop_format(_data: dict[str, Any]) -> str:
    return ""


# general: no enrichment. Registered explicitly so look-ups succeed for
# the routine path; rules/prefetch/format are all no-ops.
_register(IntentSpec(
    intent="general",
    rules="",
    prefetch=_noop_prefetch,
    format=_noop_format,
))


class IntentRegistry:
    """Static facade — no instances. Each public method is a classmethod
    so the harness can call it as `IntentRegistry.run(...)`.
    """

    @classmethod
    def known(cls) -> Iterable[str]:
        return KNOWN_INTENTS

    @classmethod
    async def run(cls, intent: str, session: Any) -> tuple[str, str]:
        """Run the intent's prefetch and return (rules_block, data_block).

        Failure semantics: any exception inside prefetch is caught, the
        intent's rules are returned unchanged, and the data block is
        empty. The harness records `intent_prefetch_failed=True` based
        on the `failed` return tuple, but this method itself never
        raises.
        """
        spec = _SPECS.get(intent)
        if spec is None:
            return "", ""
        try:
            data = await spec.prefetch(session)
        except Exception:  # noqa: BLE001 — broad by design
            log.exception("intent prefetch failed: intent=%s", intent)
            return spec.rules, ""
        try:
            return spec.rules, spec.format(data)
        except Exception:
            log.exception("intent format failed: intent=%s", intent)
            return spec.rules, ""
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
uv run pytest tests/test_intents.py -v
```
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/prog_strength_agent/intents.py tests/test_intents.py
git commit -m "feat(intents): add IntentRegistry skeleton with general no-op intent"
```

---

### Task 11: log_nutrition intent spec

**Files:**
- Modify: `prog-strength-agent/src/prog_strength_agent/intents.py`
- Modify: `prog-strength-agent/tests/test_intents.py`

- [ ] **Step 1: Write failing test (with a fake MCP session)**

Append to `tests/test_intents.py`:

```python
class _FakeMCPResult:
    def __init__(self, text: str):
        # Mirrors mcp.types.TextContent's duck-typed shape.
        self.content = [type("Block", (), {"text": text})()]
        self.isError = False


class _FakeSession:
    def __init__(self, responses: dict[str, str]):
        self._responses = responses

    async def call_tool(self, name: str, args: dict):
        return _FakeMCPResult(self._responses.get(name, "[]"))


@pytest.mark.asyncio
async def test_log_nutrition_prefetch_fans_out_pantry_and_recipes():
    session = _FakeSession({
        "list_pantry_items": '[{"id":"p-1","name":"Whey","calories":120,"protein_g":24,"fat_g":1,"carbs_g":3,"serving_size":1,"serving_unit":"scoop"}]',
        "list_recipes":      '[{"id":"r-1","name":"Standard Breakfast","components":[{"pantry_item_id":"p-1","quantity":1}]}]',
    })
    rules, data = await IntentRegistry.run("log_nutrition", session)

    assert "Assume one serving" in rules
    assert "Whey" in data
    assert "Standard Breakfast" in data
    assert "PANTRY" in data
    assert "RECIPES" in data


@pytest.mark.asyncio
async def test_log_nutrition_prefetch_failure_returns_rules_without_data():
    class _FailingSession:
        async def call_tool(self, name: str, args: dict):
            raise RuntimeError("MCP exploded")
    rules, data = await IntentRegistry.run("log_nutrition", _FailingSession())
    assert "Assume one serving" in rules
    assert data == ""
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
uv run pytest tests/test_intents.py -k log_nutrition -v
```
Expected: FAIL — `log_nutrition` not registered, returns empty rules.

- [ ] **Step 3: Add the log_nutrition spec**

In `src/prog_strength_agent/intents.py`, add helpers and the spec near the bottom (above the `IntentRegistry` class definition):

```python
import json


def _decode_tool_result(result: Any) -> Any:
    """MCP tool results come back as a `content` list of text blocks
    in JSON-stringified form. Stitch them together and json.loads the
    result. Returns [] on empty/bad payloads so callers don't have to
    branch.
    """
    parts = []
    for block in getattr(result, "content", []) or []:
        text = getattr(block, "text", None)
        if isinstance(text, str):
            parts.append(text)
    raw = "".join(parts).strip()
    if not raw:
        return []
    try:
        return json.loads(raw)
    except json.JSONDecodeError:
        return []


async def _log_nutrition_prefetch(session: Any) -> dict[str, Any]:
    pantry_task = session.call_tool("list_pantry_items", {})
    recipes_task = session.call_tool("list_recipes", {})
    pantry_res, recipes_res = await asyncio.gather(pantry_task, recipes_task)
    return {
        "pantry": _decode_tool_result(pantry_res),
        "recipes": _decode_tool_result(recipes_res),
    }


def _log_nutrition_format(data: dict[str, Any]) -> str:
    lines = []
    lines.append("USER'S CURRENT PANTRY (id · name · per-serving macros):")
    for item in data.get("pantry", []):
        macros = (
            f"{item.get('calories', 0)} kcal · "
            f"{item.get('protein_g', 0)}P / "
            f"{item.get('fat_g', 0)}F / "
            f"{item.get('carbs_g', 0)}C"
        )
        serving = f"{item.get('serving_size', 1)} {item.get('serving_unit', 'serving')}"
        lines.append(
            f"- {item.get('id', '?')} · {item.get('name', '?')} · "
            f"{macros} per {serving}"
        )
    if not data.get("pantry"):
        lines.append("- (empty — the user has not saved any pantry items yet)")
    lines.append("")
    lines.append("USER'S CURRENT RECIPES (id · name · component count):")
    for r in data.get("recipes", []):
        components = r.get("components") or []
        lines.append(
            f"- {r.get('id', '?')} · {r.get('name', '?')} · "
            f"{len(components)} component(s)"
        )
    if not data.get("recipes"):
        lines.append("- (empty — the user has not saved any recipes yet)")
    return "\n".join(lines)


_LOG_NUTRITION_RULES = """\
The user is logging a meal or snack. Assume one serving unless the \
user specifies otherwise. The user's saved pantry items and recipes \
are listed below — search them by name first before asking the user \
for macros or creating a new pantry item. Only ask follow-up \
questions about details you genuinely cannot infer.\
"""


_register(IntentSpec(
    intent="log_nutrition",
    rules=_LOG_NUTRITION_RULES,
    prefetch=_log_nutrition_prefetch,
    format=_log_nutrition_format,
))
```

Add `import json` at the top of the file if not already present.

- [ ] **Step 4: Run tests to verify they pass**

```bash
uv run pytest tests/test_intents.py -v
```
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/prog_strength_agent/intents.py tests/test_intents.py
git commit -m "feat(intents): add log_nutrition spec (pantry + recipes prefetch)"
```

---

### Task 12: log_workout intent spec

**Files:**
- Modify: `prog-strength-agent/src/prog_strength_agent/intents.py`
- Modify: `prog-strength-agent/tests/test_intents.py`

- [ ] **Step 1: Write failing test**

```python
@pytest.mark.asyncio
async def test_log_workout_prefetch_includes_catalog_and_recent_5():
    session = _FakeSession({
        "list_exercises": '[{"id":"barbell-bench-press","name":"Barbell Bench Press","muscle_groups":["chest"]},{"id":"back-squat","name":"Back Squat","muscle_groups":["quads"]}]',
        # 10 recent workouts; the formatter should slice to 5.
        "list_workouts": '[' + ','.join(
            f'{{"id":"w-{i}","performed_at":"2026-05-{i:02d}T18:00:00Z","exercises":[]}}'
            for i in range(1, 11)
        ) + ']',
    })
    rules, data = await IntentRegistry.run("log_workout", session)
    assert "exercise catalog" in rules.lower() or "look up exercise slugs" in rules.lower()
    assert "barbell-bench-press" in data
    assert "back-squat" in data
    assert "EXERCISE CATALOG" in data
    assert "RECENT WORKOUTS" in data
    assert "w-10" in data
    assert "w-6" in data    # 10..6 = last 5
    assert "w-5" not in data
```

- [ ] **Step 2: Run test to verify it fails**

```bash
uv run pytest tests/test_intents.py -k log_workout -v
```
Expected: FAIL — `log_workout` not registered.

- [ ] **Step 3: Add the spec**

In `intents.py`, add (below `_log_nutrition_format`):

```python
async def _log_workout_prefetch(session: Any) -> dict[str, Any]:
    catalog_task = session.call_tool("list_exercises", {})
    workouts_task = session.call_tool("list_workouts", {})
    catalog_res, workouts_res = await asyncio.gather(catalog_task, workouts_task)
    workouts = _decode_tool_result(workouts_res)
    # API returns ~50 most-recent; slice to 5 for prompt size.
    return {
        "catalog": _decode_tool_result(catalog_res),
        "recent_workouts": workouts[:5],
    }


def _log_workout_format(data: dict[str, Any]) -> str:
    lines = []
    lines.append("EXERCISE CATALOG (slug · name · primary muscle groups):")
    for e in data.get("catalog", []):
        muscles = ", ".join(e.get("muscle_groups", []) or [])
        lines.append(f"- {e.get('id', '?')} · {e.get('name', '?')} · {muscles}")
    if not data.get("catalog"):
        lines.append("- (catalog unavailable)")
    lines.append("")
    lines.append("RECENT WORKOUTS (id · performed_at · exercise count):")
    for w in data.get("recent_workouts", []):
        lines.append(
            f"- {w.get('id', '?')} · "
            f"{w.get('performed_at', '?')} · "
            f"{len(w.get('exercises') or [])} exercise(s)"
        )
    if not data.get("recent_workouts"):
        lines.append("- (no recent workouts logged)")
    return "\n".join(lines)


_LOG_WORKOUT_RULES = """\
The user is logging a completed workout. The exercise catalog is \
below — match the user's wording to an exercise slug without asking \
unless genuinely ambiguous. The user's last few workouts are also \
below; if they say "same as last time" or use a shorthand, infer \
from those before asking. Look up exercise slugs from the catalog \
before calling create_workout.\
"""


_register(IntentSpec(
    intent="log_workout",
    rules=_LOG_WORKOUT_RULES,
    prefetch=_log_workout_prefetch,
    format=_log_workout_format,
))
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
uv run pytest tests/test_intents.py -v
```
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/prog_strength_agent/intents.py tests/test_intents.py
git commit -m "feat(intents): add log_workout spec (catalog + recent 5 workouts)"
```

---

### Task 13: log_bodyweight intent spec

**Files:**
- Modify: `prog-strength-agent/src/prog_strength_agent/intents.py`
- Modify: `prog-strength-agent/tests/test_intents.py`

- [ ] **Step 1: Write failing test**

```python
@pytest.mark.asyncio
async def test_log_bodyweight_prefetch_calls_list_with_14_day_window(monkeypatch):
    captured_args: dict[str, Any] = {}

    class _CaptureSession:
        async def call_tool(self, name: str, args: dict):
            captured_args[name] = args
            return _FakeMCPResult(
                '[{"id":"b-1","weight":205.4,"unit":"lb","measured_at":"2026-05-30T07:00:00Z"}]'
            )

    rules, data = await IntentRegistry.run("log_bodyweight", _CaptureSession())
    assert "bodyweight" in rules.lower()
    assert "205.4" in data
    assert captured_args.get("list_bodyweight", {}).get("since") is not None
```

- [ ] **Step 2: Run test to verify it fails**

```bash
uv run pytest tests/test_intents.py -k log_bodyweight -v
```
Expected: FAIL.

- [ ] **Step 3: Add the spec**

```python
from datetime import datetime, timedelta, timezone


async def _log_bodyweight_prefetch(session: Any) -> dict[str, Any]:
    since = (datetime.now(timezone.utc) - timedelta(days=14)).isoformat().replace("+00:00", "Z")
    res = await session.call_tool("list_bodyweight", {"since": since})
    return {"entries": _decode_tool_result(res)}


def _log_bodyweight_format(data: dict[str, Any]) -> str:
    entries = data.get("entries") or []
    if not entries:
        return "RECENT BODYWEIGHT (last 14 days): (no entries yet)"
    lines = ["RECENT BODYWEIGHT (last 14 days, most recent first):"]
    for e in entries:
        lines.append(
            f"- {e.get('measured_at', '?')} · "
            f"{e.get('weight', '?')} {e.get('unit', '?')}"
        )
    return "\n".join(lines)


_LOG_BODYWEIGHT_RULES = """\
The user is logging a bodyweight reading. Default the unit to whatever \
they used most recently (visible in the entries below). If the new \
reading is meaningfully different from the recent trend, acknowledge \
it briefly; otherwise just confirm the log.\
"""


_register(IntentSpec(
    intent="log_bodyweight",
    rules=_LOG_BODYWEIGHT_RULES,
    prefetch=_log_bodyweight_prefetch,
    format=_log_bodyweight_format,
))
```

Add the `datetime/timedelta/timezone` import block at the top of the file if not already present.

- [ ] **Step 4: Run tests to verify they pass**

```bash
uv run pytest tests/test_intents.py -v
```
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/prog_strength_agent/intents.py tests/test_intents.py
git commit -m "feat(intents): add log_bodyweight spec (14-day window)"
```

---

### Task 14: analyze_progress intent spec

**Files:**
- Modify: `prog-strength-agent/src/prog_strength_agent/intents.py`
- Modify: `prog-strength-agent/tests/test_intents.py`

- [ ] **Step 1: Write failing test**

```python
@pytest.mark.asyncio
async def test_analyze_progress_prefetch_includes_workouts_and_macros():
    captured: dict[str, Any] = {}

    class _CaptureSession:
        async def call_tool(self, name: str, args: dict):
            captured[name] = args
            return _FakeMCPResult({
                "list_workouts": '[' + ','.join(
                    f'{{"id":"w-{i}","performed_at":"2026-05-{(i%28)+1:02d}T18:00:00Z","exercises":[]}}'
                    for i in range(1, 25)
                ) + ']',
                "get_daily_macros": '[{"date":"2026-05-30","calories":2200,"protein_g":180,"fat_g":70,"carbs_g":230}]',
            }[name])

    rules, data = await IntentRegistry.run("analyze_progress", _CaptureSession())
    assert "favor citing" in rules.lower() or "already have a recent window" in rules.lower()
    # Workouts capped to 20
    assert data.count("w-") <= 20
    assert "2200" in data
    assert captured.get("get_daily_macros", {}).get("since") is not None
    assert captured.get("get_daily_macros", {}).get("until") is not None
```

- [ ] **Step 2: Run test to verify it fails**

```bash
uv run pytest tests/test_intents.py -k analyze_progress -v
```
Expected: FAIL.

- [ ] **Step 3: Add the spec**

```python
async def _analyze_progress_prefetch(session: Any) -> dict[str, Any]:
    now = datetime.now(timezone.utc)
    since = (now - timedelta(days=30)).isoformat().replace("+00:00", "Z")
    until = now.isoformat().replace("+00:00", "Z")
    workouts_task = session.call_tool("list_workouts", {})
    macros_task = session.call_tool("get_daily_macros", {"since": since, "until": until})
    workouts_res, macros_res = await asyncio.gather(workouts_task, macros_task)
    workouts = _decode_tool_result(workouts_res)
    return {
        "workouts": workouts[:20],
        "daily_macros": _decode_tool_result(macros_res),
    }


def _analyze_progress_format(data: dict[str, Any]) -> str:
    lines = ["RECENT WORKOUTS (last 20, most recent first):"]
    for w in data.get("workouts", []):
        lines.append(
            f"- {w.get('id', '?')} · "
            f"{w.get('performed_at', '?')} · "
            f"{len(w.get('exercises') or [])} exercise(s)"
        )
    lines.append("")
    lines.append("DAILY MACROS (last 30 days; date · kcal · P/F/C):")
    for d in data.get("daily_macros", []):
        lines.append(
            f"- {d.get('date', '?')} · "
            f"{d.get('calories', 0)} kcal · "
            f"{d.get('protein_g', 0)}/{d.get('fat_g', 0)}/{d.get('carbs_g', 0)}"
        )
    return "\n".join(lines)


_ANALYZE_PROGRESS_RULES = """\
The user wants analysis or planning advice. You already have a recent \
window of workouts and macros below. Favor citing specifics from this \
data over calling more tools; only fetch more if the user asks about \
a time range outside the included window.\
"""


_register(IntentSpec(
    intent="analyze_progress",
    rules=_ANALYZE_PROGRESS_RULES,
    prefetch=_analyze_progress_prefetch,
    format=_analyze_progress_format,
))
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
uv run pytest tests/test_intents.py -v
```
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/prog_strength_agent/intents.py tests/test_intents.py
git commit -m "feat(intents): add analyze_progress spec (last 20 workouts + 30d macros)"
```

---

## Phase 3 — Agent: router + telemetry + api client

### Task 15: RouterDecision + structured-output classification

**Files:**
- Modify: `prog-strength-agent/src/prog_strength_agent/model_router.py`
- Create: `prog-strength-agent/tests/test_model_router.py`

- [ ] **Step 1: Write failing test**

Create `tests/test_model_router.py`:

```python
"""Unit tests for ModelRouter — structured-output (tier, intent)
classification plus the prior_intent hint plumbing."""

from __future__ import annotations

from types import SimpleNamespace
from unittest.mock import AsyncMock

import pytest

from prog_strength_agent.model_router import (
    ModelRouter,
    RouterDecision,
)


def _tool_use_block(tier: str, intent: str):
    return SimpleNamespace(
        type="tool_use",
        name="classify_request",
        input={"tier": tier, "intent": intent},
    )


def _resp(blocks):
    return SimpleNamespace(content=blocks, usage=None, stop_reason="tool_use")


@pytest.mark.asyncio
async def test_route_parses_tool_use_into_router_decision():
    client = SimpleNamespace(
        messages=SimpleNamespace(
            create=AsyncMock(return_value=_resp([_tool_use_block("simple", "log_nutrition")]))
        )
    )
    router = ModelRouter(client=client, router_model="claude-haiku-4-5-20251001")

    decision = await router.route(
        messages=[{"role": "user", "content": "log a protein shake for dinner"}],
        telemetry=None,
        prior_intent=None,
    )
    assert isinstance(decision, RouterDecision)
    assert decision.tier == "simple"
    assert decision.intent == "log_nutrition"


@pytest.mark.asyncio
async def test_route_falls_back_to_simple_general_on_classifier_error():
    client = SimpleNamespace(
        messages=SimpleNamespace(create=AsyncMock(side_effect=RuntimeError("boom")))
    )
    router = ModelRouter(client=client, router_model="claude-haiku-4-5-20251001")

    decision = await router.route(messages=[{"role": "user", "content": "hi"}], telemetry=None)
    assert decision == RouterDecision(tier="simple", intent="general")


@pytest.mark.asyncio
async def test_route_falls_back_when_response_has_no_tool_use():
    client = SimpleNamespace(
        messages=SimpleNamespace(
            create=AsyncMock(return_value=_resp([SimpleNamespace(type="text", text="oops")]))
        )
    )
    router = ModelRouter(client=client, router_model="claude-haiku-4-5-20251001")
    decision = await router.route(messages=[{"role": "user", "content": "hi"}], telemetry=None)
    assert decision == RouterDecision(tier="simple", intent="general")


@pytest.mark.asyncio
async def test_route_includes_prior_intent_hint_in_user_message_when_provided():
    create_mock = AsyncMock(return_value=_resp([_tool_use_block("simple", "log_nutrition")]))
    client = SimpleNamespace(messages=SimpleNamespace(create=create_mock))
    router = ModelRouter(client=client, router_model="claude-haiku-4-5-20251001")

    await router.route(
        messages=[{"role": "user", "content": "a protein shake"}],
        telemetry=None,
        prior_intent="log_nutrition",
    )

    sent = create_mock.await_args.kwargs
    # Hint is delivered as user-message context; verify the substring lands somewhere.
    payload = str(sent["messages"])
    assert "log_nutrition" in payload  # hint present
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
uv run pytest tests/test_model_router.py -v
```
Expected: FAIL — `RouterDecision` doesn't exist, `route` returns a string.

- [ ] **Step 3: Replace model_router.py with the structured-output version**

Open `src/prog_strength_agent/model_router.py` and replace its contents:

```python
"""Structured-output classifier for incoming chat requests.

Calls Haiku with a forced tool_use that returns
{tier: simple|complex, intent: <known intent>}. Same cost as the
previous one-word-output router (~$0.0001/call, ~500ms p50).

Failure mode: any exception or malformed output falls back to
RouterDecision(tier="simple", intent="general") — the cheapest+safest
default. The user gets a possibly-degraded response rather than a 500.

See prog-strength-docs/sows/intent-driven-context-enrichment.md.
"""

from __future__ import annotations

import logging
from dataclasses import dataclass
from typing import Any

from anthropic import AsyncAnthropic

from prog_strength_agent.intents import KNOWN_INTENTS
from prog_strength_agent.telemetry import TurnInstrumentation, now_ms

log = logging.getLogger(__name__)


@dataclass(frozen=True)
class RouterDecision:
    tier: str
    intent: str


_ROUTER_TOOL = {
    "name": "classify_request",
    "description": "Classify the user's request by intent and required model tier.",
    "input_schema": {
        "type": "object",
        "properties": {
            "tier": {"type": "string", "enum": ["simple", "complex"]},
            "intent": {"type": "string", "enum": list(KNOWN_INTENTS)},
        },
        "required": ["tier", "intent"],
    },
}


ROUTER_SYSTEM_PROMPT = """\
You are a routing classifier for the Prog Strength training assistant.
Output BOTH a tier and an intent for the user's latest request via
the classify_request tool.

tier:
- simple — CRUD or lookup (logging a workout, listing exercises).
- complex — multi-step analysis, planning, trend reasoning.

intent:
- log_nutrition — the user wants to record food/drink they consumed.
- log_workout — the user is reporting a completed workout.
- log_bodyweight — the user is logging a bodyweight reading.
- analyze_progress — the user wants insight, trends, or planning advice.
- general — anything else (greeting, lookup, off-topic).

When uncertain on intent, pick "general". When uncertain on tier,
pick "simple". Both err on the side of cheap and safe.
"""


_FALLBACK = RouterDecision(tier="simple", intent="general")


class ModelRouter:
    def __init__(self, client: AsyncAnthropic, router_model: str):
        self.client = client
        self.router_model = router_model

    async def route(
        self,
        messages: list[dict[str, Any]],
        telemetry: TurnInstrumentation | None = None,
        prior_intent: str | None = None,
    ) -> RouterDecision:
        started = now_ms()
        user_text = _last_user_text(messages)
        if not user_text:
            _record(telemetry, self.router_model, now_ms() - started, _FALLBACK)
            return _FALLBACK

        hint = ""
        if prior_intent:
            hint = (
                f"\n\nPrevious classified intent for this conversation: "
                f"{prior_intent}. Use it as a hint when the latest message "
                f"is a short follow-up; switch when the user clearly pivots."
            )

        try:
            resp = await self.client.messages.create(
                model=self.router_model,
                max_tokens=200,
                system=ROUTER_SYSTEM_PROMPT,
                tools=[_ROUTER_TOOL],
                tool_choice={"type": "tool", "name": "classify_request"},
                messages=[
                    {"role": "user", "content": user_text + hint},
                ],
            )
        except Exception:
            log.exception("router classification call failed")
            _record(telemetry, self.router_model, now_ms() - started, _FALLBACK)
            return _FALLBACK

        decision = _parse_decision(resp)
        log.info(
            "router: tier=%s intent=%s (chars=%d, hint=%s)",
            decision.tier, decision.intent, len(user_text),
            "y" if prior_intent else "n",
        )
        _record(telemetry, self.router_model, now_ms() - started, decision)
        return decision


def _parse_decision(resp: Any) -> RouterDecision:
    for block in getattr(resp, "content", []) or []:
        if getattr(block, "type", None) != "tool_use":
            continue
        if getattr(block, "name", None) != "classify_request":
            continue
        payload = getattr(block, "input", None) or {}
        tier = payload.get("tier") if isinstance(payload, dict) else None
        intent = payload.get("intent") if isinstance(payload, dict) else None
        if tier not in ("simple", "complex"):
            return _FALLBACK
        if intent not in KNOWN_INTENTS:
            return _FALLBACK
        return RouterDecision(tier=tier, intent=intent)
    return _FALLBACK


def _record(
    telemetry: TurnInstrumentation | None,
    router_model: str,
    latency_ms: int,
    decision: RouterDecision,
) -> None:
    if telemetry is None:
        return
    telemetry.router_model = router_model
    telemetry.router_latency_ms = latency_ms
    telemetry.routed_tier = decision.tier
    telemetry.intent = decision.intent


def _last_user_text(messages: list[dict[str, Any]]) -> str:
    for msg in reversed(messages):
        if msg.get("role") != "user":
            continue
        content = msg.get("content")
        if isinstance(content, str):
            return content
        if isinstance(content, list):
            parts: list[str] = []
            for block in content:
                if not isinstance(block, dict):
                    continue
                if block.get("type") == "text" and isinstance(block.get("text"), str):
                    parts.append(block["text"])
            return " ".join(parts).strip()
        return ""
    return ""
```

NOTE: this references `telemetry.intent` — the field is added in Task 16. The import is fine at module import time but the assignment in `_record` will only have effect once Task 16 lands. The tests in this task pass `telemetry=None` so this is a no-op for the test path; full integration coverage lands after Task 16.

- [ ] **Step 4: Run tests to verify they pass**

```bash
uv run pytest tests/test_model_router.py -v
```
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/prog_strength_agent/model_router.py tests/test_model_router.py
git commit -m "feat(router): structured-output {tier, intent} classification"
```

---

### Task 16: TurnInstrumentation + Prometheus + telemetry payload

**Files:**
- Modify: `prog-strength-agent/src/prog_strength_agent/telemetry.py`

- [ ] **Step 1: Write failing test (add to test_model_router for full instrumentation coverage)**

Append to `tests/test_model_router.py`:

```python
@pytest.mark.asyncio
async def test_route_populates_intent_on_telemetry():
    from prog_strength_agent.telemetry import TurnInstrumentation

    client = SimpleNamespace(
        messages=SimpleNamespace(
            create=AsyncMock(return_value=_resp([_tool_use_block("complex", "analyze_progress")]))
        )
    )
    router = ModelRouter(client=client, router_model="claude-haiku-4-5-20251001")
    t = TurnInstrumentation.new(user_id="u-1", session_id=None)

    await router.route(messages=[{"role": "user", "content": "how is my bench progressing?"}], telemetry=t)
    assert t.routed_tier == "complex"
    assert t.intent == "analyze_progress"
```

Also add to a new top-level `tests/test_telemetry.py` (create if it doesn't exist):

```python
"""Tests for TurnInstrumentation field additions + telemetry payload shape."""

from __future__ import annotations

from prog_strength_agent.telemetry import TurnInstrumentation, _build_turn_payload


def test_payload_includes_intent_fields():
    t = TurnInstrumentation.new(user_id="u-1", session_id="s-1")
    t.routed_tier = "simple"
    t.router_model = "claude-haiku-4-5-20251001"
    t.model = "claude-haiku-4-5-20251001"
    t.completion_reason = "end_turn"
    t.intent = "log_nutrition"
    t.intent_prefetch_duration_ms = 87
    t.intent_prefetch_failed = False

    body = _build_turn_payload(t)
    assert body["intent"] == "log_nutrition"
    assert body["intent_prefetch_duration_ms"] == 87
    assert body["intent_prefetch_failed"] is False
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
uv run pytest tests/test_telemetry.py tests/test_model_router.py -v
```
Expected: FAIL — `intent` field and `_build_turn_payload` helper do not exist.

- [ ] **Step 3: Extend TurnInstrumentation**

In `src/prog_strength_agent/telemetry.py`, inside the `TurnInstrumentation` dataclass, add right after `routed_tier`:

```python
    # Intent classification — populated by ModelRouter.route().
    # Empty string means the router never produced a value (cold
    # exception in the classifier call); "general" is a deliberate
    # output meaning "no specific intent recognized."
    intent: str = ""

    # Intent prefetch instrumentation — populated by ModelHarness.
    intent_prefetch_duration_ms: int = 0
    intent_prefetch_failed: bool = False
```

- [ ] **Step 4: Refactor the turn payload to a named helper**

Find `TelemetryClient._send_turn` in `telemetry.py`. Replace the inline `body = { ... }` block with a call to a new module-level helper `_build_turn_payload(t)`:

```python
async def _send_turn(self, t: TurnInstrumentation) -> None:
    body = _build_turn_payload(t)
    await self._post("/internal/telemetry/turns", body)
```

…and add (above the `TelemetryClient` class):

```python
def _build_turn_payload(t: TurnInstrumentation) -> dict:
    """Marshal a TurnInstrumentation into the JSON shape the API's
    POST /internal/telemetry/turns endpoint expects. Pulled out of
    the client so tests can assert on the wire format without
    poking through httpx mocks.
    """
    return {
        "id": t.turn_id,
        "user_id": t.user_id,
        "session_id": t.session_id,
        "model": t.model,
        "routed_tier": t.routed_tier,
        "router_model": t.router_model,
        "router_latency_ms": t.router_latency_ms,
        "input_tokens": t.input_tokens,
        "output_tokens": t.output_tokens,
        "cache_creation_tokens": t.cache_creation_tokens,
        "cache_read_tokens": t.cache_read_tokens,
        "total_latency_ms": t.total_latency_ms,
        "time_to_first_token_ms": t.time_to_first_token_ms,
        "completion_reason": t.completion_reason,
        "error": t.error,
        "started_at": _iso(t.started_at),
        "ended_at": _iso(t.ended_at),
        "intent": t.intent,
        "intent_prefetch_duration_ms": t.intent_prefetch_duration_ms,
        "intent_prefetch_failed": t.intent_prefetch_failed,
    }
```

- [ ] **Step 5: Add new Prometheus series**

In the same file, alongside the existing `AGENT_TOKENS_TOTAL` etc., add:

```python
AGENT_INTENT_CLASSIFICATIONS_TOTAL = Counter(
    "agent_intent_classifications_total",
    "Intent classifications produced by the model router.",
    ["intent"],
)
AGENT_INTENT_PREFETCH_DURATION_SECONDS = Histogram(
    "agent_intent_prefetch_duration_seconds",
    "Time spent running an intent's prefetch tool calls (parallel asyncio.gather).",
    ["intent"],
    buckets=(0.01, 0.05, 0.1, 0.25, 0.5, 1.0, 2.0, 5.0),
)
```

…and extend `record_prometheus_metrics(t)` at the bottom of its body:

```python
    if t.intent:
        AGENT_INTENT_CLASSIFICATIONS_TOTAL.labels(intent=t.intent).inc()
        if t.intent_prefetch_duration_ms > 0:
            AGENT_INTENT_PREFETCH_DURATION_SECONDS.labels(intent=t.intent).observe(
                t.intent_prefetch_duration_ms / 1000.0,
            )
```

- [ ] **Step 6: Run tests to verify they pass**

```bash
uv run pytest tests/test_telemetry.py tests/test_model_router.py -v
```
Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add src/prog_strength_agent/telemetry.py tests/test_telemetry.py tests/test_model_router.py
git commit -m "feat(telemetry): add intent fields, metrics, and payload helper"
```

---

### Task 17: API client — get_session_intent helper

**Files:**
- Create: `prog-strength-agent/src/prog_strength_agent/api_client.py`
- Create: `prog-strength-agent/tests/test_api_client.py`

- [ ] **Step 1: Write failing test (using respx for HTTP mocking)**

Create `tests/test_api_client.py`:

```python
"""Tests for the agent's tiny API client (today: just get_session_intent)."""

from __future__ import annotations

import httpx
import pytest
import respx

from prog_strength_agent.api_client import APIClient


@pytest.mark.asyncio
@respx.mock
async def test_get_session_intent_returns_value_on_200():
    respx.get("http://api:8080/internal/chat-sessions/abc/intent").mock(
        return_value=httpx.Response(
            200,
            json={"data": {"intent": "log_nutrition", "intent_at": "2026-06-02T00:00:00Z"}},
        )
    )
    client = APIClient(base_url="http://api:8080")
    intent = await client.get_session_intent("abc")
    assert intent == "log_nutrition"
    await client.aclose()


@pytest.mark.asyncio
@respx.mock
async def test_get_session_intent_returns_none_when_null():
    respx.get("http://api:8080/internal/chat-sessions/abc/intent").mock(
        return_value=httpx.Response(200, json={"data": {"intent": None, "intent_at": None}}),
    )
    client = APIClient(base_url="http://api:8080")
    assert await client.get_session_intent("abc") is None
    await client.aclose()


@pytest.mark.asyncio
@respx.mock
async def test_get_session_intent_returns_none_on_5xx():
    respx.get("http://api:8080/internal/chat-sessions/abc/intent").mock(
        return_value=httpx.Response(500, text="boom"),
    )
    client = APIClient(base_url="http://api:8080")
    assert await client.get_session_intent("abc") is None
    await client.aclose()


@pytest.mark.asyncio
@respx.mock
async def test_get_session_intent_returns_none_on_network_error():
    respx.get("http://api:8080/internal/chat-sessions/abc/intent").mock(
        side_effect=httpx.ConnectError("nope")
    )
    client = APIClient(base_url="http://api:8080")
    assert await client.get_session_intent("abc") is None
    await client.aclose()
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
uv run pytest tests/test_api_client.py -v
```
Expected: FAIL — module does not exist.

- [ ] **Step 3: Implement the client**

Create `src/prog_strength_agent/api_client.py`:

```python
"""Thin HTTP client for the Prog Strength API's internal endpoints.

Today's only consumer is the agent's pre-router step that reads the
session's last classified intent as a hint. Any failure (timeout,
5xx, network error) is swallowed and surfaced as `None` so a sluggish
API never adds user-visible latency on top of the classifier.

Network boundary: lives entirely under /internal/* on the API, which
Caddy refuses to proxy. No auth header to set.
"""

from __future__ import annotations

import logging

import httpx

log = logging.getLogger(__name__)


class APIClient:
    def __init__(
        self,
        base_url: str,
        *,
        timeout_seconds: float = 0.2,
    ):
        # 200ms timeout — fast enough that a single sluggish call
        # doesn't add user-visible latency on top of the ~500ms
        # classifier. Empty base_url disables the client; useful for
        # local dev when the API container isn't running.
        self._client = httpx.AsyncClient(base_url=base_url, timeout=timeout_seconds)

    async def aclose(self) -> None:
        await self._client.aclose()

    async def get_session_intent(self, session_id: str) -> str | None:
        """Return the session's last classified intent, or None on
        missing-session / any failure. Never raises.
        """
        if not session_id:
            return None
        try:
            resp = await self._client.get(
                f"/internal/chat-sessions/{session_id}/intent",
            )
            if resp.status_code >= 400:
                return None
            payload = resp.json()
        except Exception:
            log.exception("api_client: get_session_intent failed")
            return None
        data = payload.get("data") if isinstance(payload, dict) else None
        if not isinstance(data, dict):
            return None
        intent = data.get("intent")
        if isinstance(intent, str) and intent:
            return intent
        return None
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
uv run pytest tests/test_api_client.py -v
```
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/prog_strength_agent/api_client.py tests/test_api_client.py
git commit -m "feat(agent): add APIClient.get_session_intent helper"
```

---

### Task 18: ModelHarness — intent param, prefetch wiring, composed prompt

**Files:**
- Modify: `prog-strength-agent/src/prog_strength_agent/model_harness.py`
- Modify: `prog-strength-agent/tests/` (add new file: `test_model_harness.py`)

- [ ] **Step 1: Write failing test**

Create `tests/test_model_harness.py`:

```python
"""Tests for the intent-aware additions to ModelHarness.

The full SSE tool-use loop is covered by integration tests in CI; this
file pins the small surface that intent wiring touches: that
stream_chat composes the system prompt from base + intent rules + data
before invoking the model.
"""

from __future__ import annotations

from unittest.mock import patch

import pytest

from prog_strength_agent.prompt import compose_system_prompt


def test_compose_system_prompt_orders_sections():
    out = compose_system_prompt(base="BASE", rules="RULES", data="DATA")
    assert out.index("BASE") < out.index("RULES") < out.index("DATA")


@pytest.mark.asyncio
async def test_harness_uses_intent_registry_to_compose_prompt(monkeypatch):
    """Stub IntentRegistry.run to return known blocks and assert the
    final system prompt the harness would have sent.
    """
    from prog_strength_agent.intents import IntentRegistry
    from prog_strength_agent import model_harness as mh

    async def fake_run(intent, session):
        return "RULES_BLOCK", "DATA_BLOCK"

    monkeypatch.setattr(IntentRegistry, "run", classmethod(lambda cls, intent, session: fake_run(intent, session)))

    composed = await mh.build_intent_aware_prompt(
        base="BASE",
        intent="log_nutrition",
        session=None,
    )
    assert "BASE" in composed
    assert "RULES_BLOCK" in composed
    assert "DATA_BLOCK" in composed
```

- [ ] **Step 2: Run test to verify it fails**

```bash
uv run pytest tests/test_model_harness.py -v
```
Expected: FAIL — `build_intent_aware_prompt` doesn't exist.

- [ ] **Step 3: Extract the prompt-composition helper from the harness**

In `src/prog_strength_agent/model_harness.py`, at the top of the file (after imports), add:

```python
from prog_strength_agent.intents import IntentRegistry
from prog_strength_agent.prompt import compose_system_prompt


async def build_intent_aware_prompt(
    *,
    base: str,
    intent: str,
    session: Any,
) -> str:
    """Compose the per-turn system prompt: base + intent rules + intent
    data. Pulled out of stream_chat so tests can pin the composition
    without mocking the entire Anthropic SDK + MCP session.
    """
    rules, data = await IntentRegistry.run(intent, session)
    return compose_system_prompt(base=base, rules=rules, data=data)
```

- [ ] **Step 4: Add the intent parameter to stream_chat and wire prefetch in**

In the same file, change the `stream_chat` signature:

```python
async def stream_chat(
    self,
    messages: list[dict[str, Any]],
    user_token: str,
    telemetry: TurnInstrumentation | None = None,
    system_prompt: str | None = None,
    intent: str = "general",
) -> AsyncGenerator[bytes, None]:
```

Inside, locate the line `await session.initialize()` and the subsequent `mcp_tools = (await session.list_tools()).tools`. Right after these, insert:

```python
            prefetch_started = now_ms()
            try:
                composed_system_prompt = await build_intent_aware_prompt(
                    base=system_prompt or SYSTEM_PROMPT,
                    intent=intent,
                    session=session,
                )
                prefetch_failed = False
            except Exception:
                log.exception("intent prefetch composition failed")
                composed_system_prompt = system_prompt or SYSTEM_PROMPT
                prefetch_failed = True
            if telemetry is not None:
                telemetry.intent_prefetch_duration_ms = now_ms() - prefetch_started
                telemetry.intent_prefetch_failed = prefetch_failed
```

Then change the `system=system_prompt or SYSTEM_PROMPT,` line inside `self.client.messages.stream(...)` to:

```python
                    system=composed_system_prompt,
```

(IntentRegistry.run itself never raises — its own try/except returns empty blocks on failure — so the outer try/except here is belt-and-suspenders against a future regression that wraps something else.)

- [ ] **Step 5: Run tests to verify they pass**

```bash
uv run pytest tests/test_model_harness.py tests/test_prompt.py tests/test_intents.py -v
```
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add src/prog_strength_agent/model_harness.py tests/test_model_harness.py
git commit -m "feat(harness): inject intent rules + data into per-turn system prompt"
```

---

### Task 19: server.py — wire RouterDecision + intent end-to-end

**Files:**
- Modify: `prog-strength-agent/src/prog_strength_agent/server.py`

- [ ] **Step 1: Instantiate APIClient at module scope**

In `server.py`, after the existing `telemetry_client = ...` line, add:

```python
from prog_strength_agent.api_client import APIClient

api_client = APIClient(base_url=config.api_url) if config.api_url else None
```

- [ ] **Step 2: Rewrite `_route_and_stream` to read prior intent + pass intent through**

Replace `_route_and_stream` with:

```python
async def _route_and_stream(
    messages: list[dict[str, Any]],
    user_token: str,
    telemetry: TurnInstrumentation,
    system_prompt: str,
) -> AsyncGenerator[bytes, None]:
    """Classify the request's tier+intent, dispatch to the matching
    harness with the intent in hand, then fire telemetry on the way
    out.

    Both router and harness fail soft: a broken classifier collapses
    to (simple, general); a broken prefetch leaves the agent with its
    pre-SOW behavior. Either way /chat keeps streaming bytes.
    """
    try:
        prior_intent: str | None = None
        if api_client is not None and telemetry.session_id:
            prior_intent = await api_client.get_session_intent(telemetry.session_id)

        decision = await router_obj.route(
            messages, telemetry=telemetry, prior_intent=prior_intent,
        )
        harness = HARNESSES.get(decision.tier, HARNESSES["simple"])
        async for chunk in harness.stream_chat(
            messages,
            user_token,
            telemetry,
            system_prompt=system_prompt,
            intent=decision.intent,
        ):
            yield chunk
    finally:
        record_prometheus_metrics(telemetry)
        if telemetry_client is not None:
            telemetry_client.record_turn(telemetry)
```

- [ ] **Step 3: Verify no other call sites break**

```bash
uv run pytest -v
```
Expected: PASS — existing tests use `router_obj.route(messages, telemetry=...)` patterns that the new signature accepts (`prior_intent` defaults to None).

- [ ] **Step 4: Commit**

```bash
git add src/prog_strength_agent/server.py
git commit -m "feat(server): wire prior_intent fetch + intent dispatch into /chat"
```

---

## Phase 4 — Dashboard

### Task 20: Add intent panels to Grafana agent dashboard

**Files:**
- Modify: `prog-strength-infra/monitoring/grafana/dashboards/agent.json`

- [ ] **Step 1: Locate the tier panels to mirror**

Open `monitoring/grafana/dashboards/agent.json`. The two tier panels live around lines 214 ("Routing decisions over time (by tier)") and 255 ("Routing share (24h)"). Note their `id`, `gridPos`, and `targets[].expr` shapes — you'll copy the structure for the intent counterparts.

- [ ] **Step 2: Insert the time-series panel for intent rate**

After the closing `}` of the tier rate panel, insert a new panel object with:
- `title`: `"Intent classifications over time"`
- `description`: `"Rate of intent classifications coming out of the model router. Stacks each intent so you can read both the absolute rate and the split."`
- `id`: a new integer not in use elsewhere (scan existing `id` values; pick the next available)
- `gridPos`: shift the existing panels below it down by the new panel's height, OR place adjacent to the tier rate panel (e.g. immediately to the right with `x` adjusted)
- `targets`: one entry with `expr: "sum by (intent) (rate(agent_intent_classifications_total[5m]))"` and `legendFormat: "{{intent}}"`
- All other fields (datasource, fieldConfig, options): copy verbatim from the tier rate panel

- [ ] **Step 3: Insert the donut for 24h intent share**

After the closing `}` of the tier share donut panel, insert a similarly-shaped piechart panel with:
- `title`: `"Intent share (24h)"`
- `description`: `"Share of turns by intent over the last 24 hours. A skew toward 'general' means the classifier isn't recognizing specific intents — worth a look at the router prompt."`
- `id`: another new integer
- `gridPos`: place adjacent to the tier share donut
- `targets`: `expr: "sum by (intent) (increase(agent_intent_classifications_total[24h]))"`, `legendFormat: "{{intent}}"`
- Otherwise copy the tier share donut verbatim

- [ ] **Step 4: Validate JSON parses**

```bash
python3 -c "import json; json.load(open('monitoring/grafana/dashboards/agent.json'))"
```
Expected: no output (parses cleanly).

- [ ] **Step 5: Commit**

```bash
git add monitoring/grafana/dashboards/agent.json
git commit -m "feat(grafana): add intent rate + share panels to agent dashboard"
```

---

## End-to-end smoke

After all tasks land in their respective repos, do one local end-to-end check before opening PRs:

1. Start the API + agent + MCP stack (`docker compose up` from `prog-strength-infra/`).
2. Open the web app and start a fresh chat: "log a protein shake for dinner."
3. Verify in the agent logs: `router: tier=simple intent=log_nutrition (chars=…, hint=n)`.
4. Verify the assistant response uses the pantry directly (e.g. matches a saved "Whey Isolate" item) and does NOT ask for protein/calorie macros.
5. Send a follow-up: "and a banana." Verify the agent logs show `hint=y` (the prior intent rode through).
6. Hit `http://localhost:3000/d/...` (Grafana) and confirm `agent_intent_classifications_total` is populated under the new panels.

If any of these fail, debug in the corresponding phase before opening PRs.

---

## Spec coverage check

| SOW requirement | Plan task |
|---|---|
| Router emits `{tier, intent}` in single Haiku call | Task 15 |
| Static `IntentRegistry` with prefetch/rules/format | Task 10–14 |
| Harness runs prefetch + composes prompt | Task 18 |
| `chat_sessions.last_intent` + `last_intent_at` columns | Task 1, 5 |
| Internal `GET /chat-sessions/{id}/intent` | Task 6 |
| `agent_turns.intent*` columns | Task 2, 3 |
| Telemetry payload carries intent fields | Task 4, 16 |
| Telemetry write-through to `chat_sessions.last_intent` for non-general | Task 7 |
| `agent_intent_classifications_total{intent}` counter | Task 16 |
| `agent_intent_prefetch_duration_seconds{intent}` histogram | Task 16 |
| Grafana intent rate panel | Task 20 |
| Grafana 24h intent share donut | Task 20 |
| Base prompt "assume one serving" convention | Task 8 |
| Multi-turn intent persistence via prior_intent hint | Task 15, 17, 19 |
| Failure modes: router fallback, prefetch graceful skip, intent endpoint best-effort | Task 10, 15, 17 |
| Tests covering each unit | Task 3, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18 |

Every SOW requirement is covered.
