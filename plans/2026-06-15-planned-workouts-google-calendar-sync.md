# Planned Workouts & Google Calendar Sync — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Introduce a first-class `planned_workout` entity (scheduled window + optional target agenda + lifecycle status), make it creatable via web/agent/API, add opt-in one-way Google Calendar sync with an encrypted server-side refresh token, and propagate completion from logged sessions.

**Architecture:** A new `internal/planned_workout` Go domain owns the entity and *all* Google Calendar writes (single-writer invariant). MCP/agent never touch the DB or Google — they call the API. Google sync is opt-in: a second incremental server-side OAuth flow stores an AES-256-GCM-encrypted refresh token; calendar writes are synchronous best-effort with a resyncable status. Prog Strength is the source of truth.

**Tech Stack:** Go 1.25 (chi v5, database/sql + SQLite, `golang.org/x/oauth2`), Python 3.12 (FastMCP, httpx; Anthropic agent), Next.js 16 / React 19 / TypeScript (web), Terraform + docker-compose (infra).

**Delivery is phased (riskiest-and-most-valuable first):**
- Phase 1 — `planned_workout` domain + CRUD (prog-strength-api). Tasks 1–3.
- Phase 2 — MCP tools + agent "plan my week" (prog-strength-mcp, prog-strength-agent). Tasks 4–5.
- Phase 3 — Opt-in Google push + refresh-token storage (prog-strength-api, prog-strength-web, prog-strength-infra). Tasks 6–10.
- Phase 4 — Completion propagation (prog-strength-api, prog-strength-mcp, prog-strength-web). Tasks 11–13.

**Conventions every task must follow (verified against the codebase):**
- Go domain anatomy mirrors `internal/bodyweight`: `<domain>.go` (model + `Validate()`), `errors.go` (sentinel errors via `errors.New`), `repository.go` (interface with doc comments, ownership enforced at storage layer — cross-user IDs return `ErrNotFound`), `memory_repository.go` + `sqlite_repository.go` (both with `var _ Repository = (*T)(nil)` compile check, `now func() time.Time` field defaulting to `time.Now`), `handler.go` (`NewHandler`, `Mount(r chi.Router)`, DTOs, `auth.UserIDFrom(r.Context())` for user id, `httpresp.OK/Created/Error/ServerError`), and `*_test.go` alongside.
- IDs: `id.New()` (16-byte hex). Timestamps: always `.UTC()`, `created_at`/`updated_at`/`deleted_at`, soft-delete via `deleted_at IS NULL` filter.
- Migrations: `internal/db/migrations/NNN_name.sql`, forward-only, embedded via `//go:embed`, applied in numeric order. **Next number is 021** (latest is `020_timeline.sql`); renumber if another lands first.
- Range queries use RFC3339 `since`(inclusive)/`until`(exclusive) parsed with the `parseSinceUntil` pattern.
- HTTP envelope: success `{service,version,message,data}`, error `{service,version,error,code}`. Web client unwraps `body.data`.
- Tests: `go test ./...` (API), `uv run pytest` (mcp/agent), `npm run test` + `npm run typecheck` + `npm run lint` (web), `terraform fmt`/`validate` (infra).
- Commits: Conventional Commits (`feat(scope): ...`), imperative, frequent. End commit messages with the `Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>` trailer.

---

## Phase 1 — `planned_workout` domain + CRUD (prog-strength-api)

### Task 1: Migration 021 + user timezone & calendar_default_detail columns

**Files:**
- Create: `internal/db/migrations/021_planned_workouts.sql`
- Modify: `internal/user/user.go` (add fields + validation), `internal/user/sqlite_repository.go` (INSERT/SELECT/UPDATE column lists + scans), `internal/user/memory_repository.go` (carry fields through Create/Update), `internal/user/handler.go` (`meResponse` + `updateMeRequest` + apply block + `resolveMe`)
- Test: `internal/user/sqlite_repository_test.go`, `internal/user/handler_test.go` (extend existing)

**Migration `021_planned_workouts.sql`** — create three tables plus two `users` columns. Follow the SOW schema exactly:

```sql
-- planned_workouts: forward-looking training entity with lifecycle + Google bookkeeping.
CREATE TABLE IF NOT EXISTS planned_workouts (
    id                     TEXT PRIMARY KEY,
    user_id                TEXT NOT NULL,
    name                   TEXT,
    activity_kind          TEXT NOT NULL DEFAULT 'lift' CHECK (activity_kind IN ('lift')),
    scheduled_start_utc    DATETIME NOT NULL,
    scheduled_end_utc      DATETIME NOT NULL,
    timezone               TEXT NOT NULL,
    status                 TEXT NOT NULL DEFAULT 'planned' CHECK (status IN ('planned','completed','skipped')),
    notes                  TEXT,
    completed_session_id   TEXT,
    completed_session_kind TEXT CHECK (completed_session_kind IN ('workout','activity')),
    calendar_detail        TEXT CHECK (calendar_detail IN ('time_block','full_agenda')),
    google_event_id        TEXT,
    google_sync_status     TEXT CHECK (google_sync_status IN ('pending','synced','failed')),
    last_sync_error        TEXT,
    created_at             DATETIME NOT NULL,
    updated_at             DATETIME NOT NULL,
    deleted_at             DATETIME
);
CREATE INDEX IF NOT EXISTS idx_planned_workouts_user_start
    ON planned_workouts (user_id, scheduled_start_utc);

-- planned_workout_exercises: ordered target exercises (optional agenda).
CREATE TABLE IF NOT EXISTS planned_workout_exercises (
    id                 TEXT PRIMARY KEY,
    planned_workout_id TEXT NOT NULL REFERENCES planned_workouts(id) ON DELETE CASCADE,
    exercise_id        TEXT NOT NULL,
    order_index        INTEGER NOT NULL,
    notes              TEXT
);
CREATE INDEX IF NOT EXISTS idx_planned_workout_exercises_pw
    ON planned_workout_exercises (planned_workout_id, order_index);

-- planned_sets: target-oriented sets (introduces target_rpe, absent from logged sets).
CREATE TABLE IF NOT EXISTS planned_sets (
    id                          TEXT PRIMARY KEY,
    planned_workout_exercise_id TEXT NOT NULL REFERENCES planned_workout_exercises(id) ON DELETE CASCADE,
    order_index                 INTEGER NOT NULL,
    target_reps                 INTEGER,
    target_weight               REAL,
    unit                        TEXT,
    target_rpe                  REAL
);
CREATE INDEX IF NOT EXISTS idx_planned_sets_pwe
    ON planned_sets (planned_workout_exercise_id, order_index);

-- users: canonical IANA timezone (server-side Google writes) + calendar detail default.
ALTER TABLE users ADD COLUMN timezone TEXT NOT NULL DEFAULT 'UTC';
ALTER TABLE users ADD COLUMN calendar_default_detail TEXT NOT NULL DEFAULT 'time_block';
```

> Note: SQLite enforces `ON DELETE CASCADE` only when `PRAGMA foreign_keys=ON`. Check `internal/db/open.go` / `db.Open`; if foreign keys are not enabled connection-wide, the repository must delete child rows explicitly inside the same transaction (the planned_workout sqlite repo in Task 2 does this regardless, so cascade is belt-and-suspenders). Confirm during implementation.

**User model/handler changes:**
- `user.User`: add `Timezone string \`json:"timezone"\`` and `CalendarDefaultDetail string \`json:"calendar_default_detail"\``.
- `User.Validate()`: timezone must be non-empty and parse via `time.LoadLocation` (reject invalid IANA names with a new `ErrInvalidTimezone`); `calendar_default_detail` must be `time_block` or `full_agenda` (new `ErrInvalidCalendarDetail`). Default values come from the migration so existing rows are valid.
- sqlite repo `Create`/`GetByID`/`GetByEmail`/`Update`: add `timezone`, `calendar_default_detail` to the column lists and scan targets (mirror how `height_cm` threads through).
- memory repo: carry the two new fields through Create/Update (copy semantics).
- `meResponse`: add `Timezone string \`json:"timezone"\`` + `CalendarDefaultDetail string \`json:"calendar_default_detail"\``; populate in `resolveMe`.
- `updateMeRequest`: add `Timezone *string` + `CalendarDefaultDetail *string`; apply only when non-nil, before `u.Validate()`.

**Steps (TDD):**
- [ ] Write failing user repo test: create a user, default `timezone="UTC"` and `calendar_default_detail="time_block"`; update both to `America/New_York` / `full_agenda` and read back.
- [ ] Write failing handler test: `PATCH /me {"timezone":"America/New_York","calendar_default_detail":"full_agenda"}` returns 200 with the new values in the resolved profile; invalid timezone → 400; invalid detail → 400.
- [ ] Run tests, verify they fail.
- [ ] Add the migration file.
- [ ] Implement model/validation/repo/handler changes.
- [ ] Run `go test ./internal/user/... ./internal/db/...`, verify pass; `go build ./...`.
- [ ] Commit: `feat(planned-workouts): add 021 migration + user timezone & calendar detail default`.

### Task 2: `planned_workout` domain — model, errors, repository (memory + sqlite)

**Files:**
- Create: `internal/planned_workout/planned_workout.go` (model + `Validate()`), `internal/planned_workout/errors.go`, `internal/planned_workout/repository.go`, `internal/planned_workout/memory_repository.go`, `internal/planned_workout/sqlite_repository.go`
- Test: `internal/planned_workout/planned_workout_test.go`, `internal/planned_workout/memory_repository_test.go`, `internal/planned_workout/sqlite_repository_test.go`

> Package name `plannedworkout` (Go package, no underscore), directory `internal/planned_workout` to match the SOW's `plannedworkout.NewHandler` reference. Confirm the existing repo convention; `internal/bodyweight` uses a single-word package so a one-word `plannedworkout` package in a `planned_workout` dir is consistent with Go norms.

**Model (`planned_workout.go`):**

```go
type Status string
const (
    StatusPlanned   Status = "planned"
    StatusCompleted Status = "completed"
    StatusSkipped   Status = "skipped"
)

type ActivityKind string
const ActivityKindLift ActivityKind = "lift"

type SessionKind string
const (
    SessionKindWorkout  SessionKind = "workout"
    SessionKindActivity SessionKind = "activity"
)

type CalendarDetail string
const (
    DetailTimeBlock  CalendarDetail = "time_block"
    DetailFullAgenda CalendarDetail = "full_agenda"
)

type GoogleSyncStatus string
const (
    SyncPending GoogleSyncStatus = "pending"
    SyncSynced  GoogleSyncStatus = "synced"
    SyncFailed  GoogleSyncStatus = "failed"
)

// PlannedSet is a target-oriented set. All targets are optional (a bare
// time block has no sets; a partially-specified plan may omit weight/rpe).
type PlannedSet struct {
    ID           string
    OrderIndex   int
    TargetReps   *int
    TargetWeight *float64
    Unit         *string
    TargetRPE    *float64
}

type PlannedExercise struct {
    ID         string
    ExerciseID string   // catalog slug; not FK-validated here (mirrors workout)
    OrderIndex int
    Notes      *string
    Sets       []PlannedSet
}

type PlannedWorkout struct {
    ID                   string
    UserID               string
    Name                 *string
    ActivityKind         ActivityKind
    ScheduledStartUTC    time.Time
    ScheduledEndUTC      time.Time
    Timezone             string
    Status               Status
    Notes                *string
    CompletedSessionID   *string
    CompletedSessionKind *SessionKind
    CalendarDetail       *CalendarDetail   // nil = fall back to user default
    GoogleEventID        *string
    GoogleSyncStatus     *GoogleSyncStatus
    LastSyncError        *string
    Exercises            []PlannedExercise
    CreatedAt            time.Time
    UpdatedAt            time.Time
    DeletedAt            *time.Time
}
```

`Validate()` (handler-side clean-400 guard, mirrors `bodyweight.Entry.Validate`):
- `UserID` non-empty.
- `ActivityKind` must equal `lift` (`ErrInvalidActivityKind`).
- `ScheduledStartUTC` and `ScheduledEndUTC` non-zero; end strictly after start (`ErrInvalidWindow`).
- `Timezone` non-empty and `time.LoadLocation`-valid (`ErrInvalidTimezone`).
- `Status` in the enum (`ErrInvalidStatus`).
- If `CalendarDetail != nil`, must be a valid enum (`ErrInvalidCalendarDetail`).
- If `CompletedSessionKind != nil`, must be valid; if either of CompletedSessionID/Kind is set, both must be set (`ErrInvalidCompletionLink`).
- Each set's `Unit`, when non-nil, must be `lb`/`kg`; `TargetRPE`, when non-nil, in `[1,10]` (`ErrInvalidRPE`); `TargetReps`/`TargetWeight`, when non-nil, > 0.

**errors.go:** sentinel `errors.New` values: `ErrNotFound`, `ErrInvalidActivityKind`, `ErrInvalidWindow`, `ErrInvalidTimezone`, `ErrInvalidStatus`, `ErrInvalidCalendarDetail`, `ErrInvalidCompletionLink`, `ErrInvalidRPE`, `ErrInvalidSet`, plus an `isValidationError(err) bool` helper.

**repository.go interface** (ownership enforced at storage layer; cross-user → `ErrNotFound`):

```go
type Repository interface {
    // Create persists the planned workout and its full agenda (exercises +
    // sets) atomically. Implementation sets all IDs and timestamps; status
    // defaults to planned when empty. Validate runs first.
    Create(ctx context.Context, pw *PlannedWorkout) error
    // Get returns a planned workout with its agenda, scoped to user_id.
    Get(ctx context.Context, userID, id string) (*PlannedWorkout, error)
    // List returns the user's non-deleted plans whose scheduled_start_utc is
    // in [since,until) (nil = open bound), ordered scheduled_start_utc ASC,
    // each with its agenda hydrated.
    List(ctx context.Context, userID string, since, until *time.Time) ([]PlannedWorkout, error)
    // Update overwrites mutable fields and REPLACES the agenda (delete child
    // rows + reinsert) atomically. Preserves id/user_id/created_at and the
    // Google bookkeeping fields unless explicitly set on pw. Validate runs first.
    Update(ctx context.Context, pw *PlannedWorkout) error
    // Delete soft-deletes (sets deleted_at). Cross-user/missing → ErrNotFound.
    Delete(ctx context.Context, userID, id string) error
    // SetStatus transitions status (e.g. skip). Cross-user/missing → ErrNotFound.
    SetStatus(ctx context.Context, userID, id string, status Status) error
    // SetCompletion sets status=completed + the polymorphic session link.
    SetCompletion(ctx context.Context, userID, id, sessionID string, kind SessionKind) error
    // SetGoogleSync persists Google bookkeeping (event id, status, last error)
    // after a calendar write attempt. nil eventID clears it (404 path).
    SetGoogleSync(ctx context.Context, userID, id string, eventID *string, status GoogleSyncStatus, lastErr *string) error
}
```

> `SetCompletion`, `SetGoogleSync` are used in Phases 3–4 but defined now so the interface is stable. Memory + sqlite implement all methods now; Phase 3/4 wire the handlers.

**sqlite_repository.go:** Use a transaction for Create/Update (insert parent + children). Hydrate agenda in Get/List with secondary queries (or a join + in-Go grouping). On Update, `DELETE FROM planned_workout_exercises WHERE planned_workout_id=?` (cascade clears sets) then reinsert. Filter every read with `deleted_at IS NULL` and `user_id=?`. Scan nullable columns with `sql.NullString`/`sql.NullFloat64`/`sql.NullInt64`/`sql.NullTime` → pointer fields.

**memory_repository.go:** Mutex-guarded maps; deep-copy on store and return so callers can't mutate internal state (mirror `bodyweight` memory repo). Implement the same atomic agenda replacement semantics.

**Steps (TDD):** Write repository tests FIRST covering, for BOTH memory and sqlite (use a shared table-driven helper or a `newRepo` factory per impl):
- Create a bare time block (no agenda) → Get returns it, status `planned`, empty `Exercises`.
- Create with a 2-exercise / 3-set agenda → Get returns the agenda in order with all targets (incl `target_rpe`, nullable weight).
- List by week range returns only plans in `[since,until)`, ordered by start ASC; excludes other users' plans and soft-deleted plans.
- Update reschedules + replaces the agenda (old exercises/sets gone, new ones present); preserves `created_at`.
- Delete soft-deletes (Get → `ErrNotFound`, not in List).
- SetStatus → skipped; cross-user → `ErrNotFound`.
- SetCompletion sets status+link; SetGoogleSync round-trips event id/status/error and nil-clears.
- Cross-user Get/Update/Delete → `ErrNotFound`.
- [ ] Run tests (fail), implement, run `go test ./internal/planned_workout/...` (pass), `go build ./...`.
- [ ] Commit: `feat(planned-workouts): add domain model, errors, and repositories`.

### Task 3: `planned_workout` HTTP handler + server wiring

**Files:**
- Create: `internal/planned_workout/handler.go`, `internal/planned_workout/handler_test.go`
- Modify: `internal/server/server.go` (declare `plannedWorkoutRepo`, construct sqlite/memory, mount in the `auth.RequireUser` group)
- Test: handler_test.go

**Handler** (`NewHandler(repo Repository, userRepo user.Repository)` — userRepo for default timezone fallback when a request omits it). `Mount`:
```go
r.Route("/planned-workouts", func(r chi.Router) {
    r.Post("/", h.create)
    r.Get("/", h.list)
    r.Get("/{id}", h.get)
    r.Put("/{id}", h.update)
    r.Delete("/{id}", h.delete)
    r.Post("/{id}/skip", h.skip)
    // /schedule, /resync, /complete added in Phases 3–4.
})
```

**DTOs** (request + response). Request JSON uses snake_case fields: `name`, `scheduled_start` (RFC3339), `scheduled_end` (RFC3339), `timezone` (IANA; when omitted, default to the user's `timezone` from userRepo), `notes`, `calendar_detail` (`time_block`/`full_agenda`/omitted), and `exercises: [{exercise_id, notes?, sets: [{target_reps?, target_weight?, unit?, target_rpe?}]}]`. `order_index` is derived from array position (mirrors workout). Response DTO serializes the full planned workout incl agenda, `status`, `google_sync_status`, `google_event_id`, `last_sync_error`, timestamps.

Handlers:
- `create`: decode, fill timezone default, build `PlannedWorkout` (status `planned`), `repo.Create`, 201 `Created`. Validation errors → 400, else `ServerError`.
- `list`: `parseSinceUntil` (reuse the bodyweight pattern; copy the helper into this package or extract — keep it local to avoid cross-domain coupling), `repo.List`, 200.
- `get`: `repo.Get`, `ErrNotFound` → 404.
- `update`: decode, load existing (`repo.Get`, 404 if missing), overlay provided fields + agenda, `repo.Update`, 200.
- `delete`: `repo.Delete`, 404/200.
- `skip`: `repo.SetStatus(...StatusSkipped)`, 404/200.

**server.go wiring:** add `var plannedWorkoutRepo plannedworkout.Repository`; in sqlite branch `plannedWorkoutRepo = plannedworkout.NewSQLiteRepository(database)`, in memory branch `plannedworkout.NewMemoryRepository()`; inside the `r.Group(auth.RequireUser)` block: `plannedworkout.NewHandler(plannedWorkoutRepo, userRepo).Mount(r)`.

**Steps (TDD):** handler_test.go using an `httptest` server with the memory repo + a context-injected user id (mirror `bodyweight/handler_test.go` setup):
- POST bare block → 201, GET returns it.
- POST with agenda → 201, agenda persisted and returned in order.
- GET `?since=&until=` returns week range.
- PUT reschedules + replaces agenda → 200.
- DELETE → 200, subsequent GET → 404.
- POST `/{id}/skip` → status skipped.
- Authz: user B cannot GET/PUT/DELETE/skip user A's plan (→ 404).
- Bad timezone / bad calendar_detail / end<=start → 400.
- [ ] Run (fail), implement, run `go test ./...`, `go build ./...` (pass).
- [ ] Commit: `feat(planned-workouts): add HTTP handler and server wiring`.

**Phase 1 exit:** planned workouts can be created/listed/edited/deleted/skipped via the API. No agent, no Google.

---

## Phase 2 — MCP tools + agent (prog-strength-mcp, prog-strength-agent)

### Task 4: MCP `planned_workouts.py` tools + api_client methods

**Files:**
- Create: `src/prog_strength_mcp/planned_workouts.py`
- Modify: `src/prog_strength_mcp/server.py` (import + `planned_workouts.register(mcp, api)` in alpha order after `pantry`), `src/prog_strength_mcp/api_client.py` (add methods)
- Test: `tests/test_planned_workouts_tools.py`

Mirror `workouts.py`: `_auth_header_or_raise()` forwards the inbound `Authorization` header via `get_http_headers`; tools are `@mcp.tool` closures over `api`; `APIError` → `RuntimeError(f"API error ({e.status_code}): {e.message}")`. Pydantic input models for the agenda (`PlannedSetInput{target_reps?, target_weight?, unit?, target_rpe?}`, `PlannedExerciseInput{exercise_id, notes?, sets}`).

Tools:
- `create_planned_workout(scheduled_start, scheduled_end, timezone, name=None, notes=None, calendar_detail=None, exercises=None)` → POST `/planned-workouts`.
- `list_planned_workouts(since, until)` → GET `/planned-workouts?since=&until=`.
- `update_planned_workout(planned_workout_id, ...)` → PUT `/planned-workouts/{id}`.
- `skip_planned_workout(planned_workout_id)` → POST `/planned-workouts/{id}/skip`.
- `complete_planned_workout(planned_workout_id, session_id, session_kind)` — **Phase 4 wires the endpoint; here it is fully implemented as a forward to POST `/planned-workouts/{id}/complete`** (the API endpoint arrives in Phase 4; the tool can ship now and will 404 until then — document that in the docstring). To honor the SOW's "stubbed here," implement it to call the endpoint but note in the docstring it requires the Phase 4 API.
- `schedule_workout_to_calendar(planned_workout_id, detail_level=None)` — same approach: forwards to POST `/planned-workouts/{id}/schedule`; Phase 3 lands the endpoint.

> Decision: implement these two as real forwards (not raise-NotImplemented stubs) so Phase 3/4 need no MCP change beyond tests. Their docstrings note the API dependency. This is within the SOW's intent ("added in Phase N; stubbed here") — the tool surface is present from Phase 2.

api_client.py methods (mirror `create_workout`/`list_workouts`): one method per endpoint, `headers={"Authorization": auth_header}`, unwrap `resp.json().get("data")`, `_raise_for_status`. Add: `create_planned_workout`, `list_planned_workouts`, `update_planned_workout`, `skip_planned_workout`, `complete_planned_workout`, `schedule_workout_to_calendar`.

**Steps (TDD):** `tests/test_planned_workouts_tools.py` using `respx` for api_client request/auth assertions and the `_ExplodingAPI` + monkeypatched `get_http_headers` pattern for the tool auth guard (mirror `tests/test_steps_tools.py`):
- api_client `create_planned_workout` POSTs the right body (incl agenda) and forwards auth → returns `data`.
- `list_planned_workouts` sets `since`/`until` query params.
- `schedule_workout_to_calendar` / `complete_planned_workout` hit the right path with the right body.
- Tool raises `RuntimeError` when the Authorization header is absent.
- Tool maps `APIError` → `RuntimeError`.
- [ ] Run (fail), implement, `uv run pytest` (pass), `uv run ruff check`.
- [ ] Commit: `feat(planned-workouts): add MCP tools and api_client methods`.

### Task 5: Agent prompt + `plan_workout` intent + router classification

**Files:**
- Modify: `src/prog_strength_agent/prompt.py` (add a "Planned workouts" capability + conventions section to `SYSTEM_PROMPT`), `src/prog_strength_agent/intents.py` (add `plan_workout` intent: rules + prefetch + format + register + `KNOWN_INTENTS`), `src/prog_strength_agent/model_router.py` (add `plan_workout` to `ROUTER_SYSTEM_PROMPT`, tier=complex)
- Test: `tests/test_prompt.py`, `tests/test_intents.py`, `tests/test_model_router.py` (extend)

`SYSTEM_PROMPT` additions (match the existing tool-list + bold-convention style):
- Under "What you can do", add the planned-workout tools (`create_planned_workout`, `list_planned_workouts`, `update_planned_workout`, `skip_planned_workout`, `schedule_workout_to_calendar`).
- A conventions block: **"Planning ahead vs. logging."** A *planned workout* is forward-looking (a future training session); `create_workout` logs a *completed* one. **"Plan my week"** = call `create_planned_workout` once per training day across the requested week, each with a `scheduled_start`/`scheduled_end` window in the user's timezone and an optional target agenda (exercises/sets with target reps/weight/RPE — look up slugs from the catalog first, same as logging). Compose a sensible split (e.g. upper/lower) and space rest days. Only call `schedule_workout_to_calendar` when the user explicitly asks to put plans on their Google Calendar. **"Targets use target_reps/target_weight/target_rpe"**, distinct from logged sets.

`intents.py` — register `plan_workout`:
- `KNOWN_INTENTS` += `"plan_workout"`.
- `_plan_workout_prefetch`: gather `list_exercises` (catalog) + `list_workouts` (recent, slice 5) like `log_workout`.
- `_plan_workout_format`: render catalog slugs + recent workouts for reference.
- `_PLAN_WORKOUT_RULES`: "The user wants to plan future training. Use create_planned_workout once per training day; build the schedule in their timezone; look up exercise slugs from the catalog below; only schedule to Google Calendar if they explicitly ask."

`model_router.py` — add to the intent list in `ROUTER_SYSTEM_PROMPT`: `plan_workout — the user wants to schedule/plan future workouts ("plan my week", "set up next week's training").` Tier guidance: planning is `complex`.

**Steps (TDD):**
- [ ] `test_intents.py`: `known()` set now includes `plan_workout`; prefetch returns catalog + recent_workouts; format renders slugs.
- [ ] `test_prompt.py`: `SYSTEM_PROMPT` still embedded; the planned-workout convention text present.
- [ ] `test_model_router.py`: router prompt mentions `plan_workout` (and existing classification tests still pass).
- [ ] Run (fail), implement, `uv run pytest` (pass), `uv run ruff check`.
- [ ] Commit: `feat(planned-workouts): teach the agent to plan a week of workouts`.

**Phase 2 exit:** "plan an upper/lower split for next week" creates planned workouts end-to-end, visible on the web calendar (once Task 10 web rendering lands; the API + agent path itself works now). Still no Google.

---

## Phase 3 — Opt-in Google Calendar push + refresh-token storage

### Task 6: Token encryption + `user_calendar_connection` storage + config

**Files (prog-strength-api):**
- Create: `internal/db/migrations/022_user_calendar_connection.sql`, `internal/calendarsync/crypto.go` (+ `crypto_test.go`), `internal/calendarconn/` domain (`connection.go`, `errors.go`, `repository.go`, `memory_repository.go`, `sqlite_repository.go`, tests)
- Modify: `internal/config/config.go` (add `CalendarTokenEncKey string` from `CALENDAR_TOKEN_ENC_KEY`)

**Migration `022_user_calendar_connection.sql`:**
```sql
CREATE TABLE IF NOT EXISTS user_calendar_connection (
    user_id             TEXT PRIMARY KEY,
    refresh_token_enc   BLOB NOT NULL,
    refresh_token_nonce BLOB NOT NULL,
    google_calendar_id  TEXT NOT NULL,
    scopes              TEXT NOT NULL,
    status              TEXT NOT NULL CHECK (status IN ('connected','revoked')),
    connected_at        DATETIME NOT NULL,
    updated_at          DATETIME NOT NULL
);
```

**`internal/calendarsync/crypto.go`:** AES-256-GCM. `NewCipher(key []byte) (*Cipher, error)` requires `len(key)==32` (else error). `Encrypt(plaintext []byte) (ciphertext, nonce []byte, err error)` (fresh 12-byte random nonce per call). `Decrypt(ciphertext, nonce []byte) ([]byte, error)`. Key is read from `CALENDAR_TOKEN_ENC_KEY` (32 raw bytes; if provided as base64/hex, document the expected encoding — use base64 std encoding decoded to 32 bytes; validate length at startup and surface a clear error). Tests: round-trip encrypt→decrypt; wrong-key decrypt fails; tampered ciphertext fails; nonce differs across calls.

**`internal/calendarconn` domain:** model `Connection{UserID, GoogleCalendarID, Scopes string, Status (connected/revoked), ConnectedAt, UpdatedAt}` plus, internal to the repo, the encrypted token blob + nonce. Repository:
```go
type Repository interface {
    Upsert(ctx context.Context, userID string, refreshTokenEnc, nonce []byte, calendarID, scopes string, now time.Time) error
    Get(ctx context.Context, userID string) (*Connection, error) // metadata only (no token)
    GetRefreshToken(ctx context.Context, userID string) (enc, nonce []byte, err error) // for minting access tokens
    SetStatus(ctx context.Context, userID string, status string, now time.Time) error
    Delete(ctx context.Context, userID string) error
    Exists(ctx context.Context, userID string) (bool, error)
}
```
`ErrNotFound` when absent. memory + sqlite impls + tests.

**config.go:** add `CalendarTokenEncKey: os.Getenv("CALENDAR_TOKEN_ENC_KEY")` field with a doc comment (additive, optional — empty disables calendar sync, mirroring `AvatarBucketName`). Do NOT make it required (feature is dormant without it).

**Steps (TDD):** crypto tests + calendarconn repo tests first (round-trip, wrong-key, upsert/get/getrefresh/setstatus/delete/exists, cross-user isolation), then migration + impls + config.
- [ ] Run (fail), implement, `go test ./...`, `go build ./...`.
- [ ] Commit: `feat(calendar-sync): add token encryption, calendar-connection storage, and config`.

### Task 7: Incremental calendar OAuth flow + connection endpoints

**Files (prog-strength-api):**
- Create: `internal/calendarsync/oauth.go` (calendar OAuth config + connect/callback handlers), `internal/calendarsync/handler.go` (the `/me/calendar/connection` + `/auth/google/calendar/*` routes), tests
- Modify: `internal/server/server.go` (construct calendarsync handler with config + calendarconn repo + cipher; mount the public `/auth/google/calendar/*` routes near auth, and the authed `/me/calendar/connection` routes in the RequireUser group)

OAuth config: a second `oauth2.Config` reusing `GOOGLE_CLIENT_ID/SECRET` but `Scopes: ["https://www.googleapis.com/auth/calendar.events"]`, `RedirectURL = GOOGLE_REDIRECT_URL`'s sibling calendar callback (add a `GOOGLE_CALENDAR_REDIRECT_URL` config var, or derive). Endpoints:
- `GET /auth/google/calendar/connect` (authed — needs the logged-in user; read JWT). Redirect to Google with `access_type=offline`, `include_granted_scopes=true`, `prompt=consent`, a CSRF `state` cookie, and a `return_to` cookie (reuse the existing whitelist machinery from `auth`). Because this requires the user id, gate it behind `auth.RequireUser` (token can come via cookie or `Authorization`; the web app will navigate the browser, which carries the auth cookie).
- `GET /auth/google/calendar/callback` (public): validate state, exchange the code, extract the **refresh token**, encrypt it, fetch the user's primary calendar id (`GET https://www.googleapis.com/calendar/v3/calendars/primary` or default to `"primary"`), `calendarconn.Upsert(status=connected)`, then redirect to `return_to` (or JSON). If Google returns no refresh token (user previously consented without `prompt=consent`), surface a clear error.
- `GET /me/calendar/connection` (authed): returns `{status: "connected"|"revoked"|"absent", google_calendar_id?, scopes?, connected_at?}`.
- `DELETE /me/calendar/connection` (authed): best-effort revoke at Google (`POST https://oauth2.googleapis.com/revoke?token=<refresh>`), then `calendarconn.Delete`.

**Steps (TDD):** handler tests with a faked Google token exchange + userinfo/calendar client (inject an `http.Client` or an interface so tests don't hit the network). Test: connect redirect carries the right scope/params + sets state; callback stores an encrypted token and marks connected; `GET connection` reflects status; DELETE revokes + removes. Use the memory calendarconn repo + a test cipher.
- [ ] Run (fail), implement, `go test ./...`.
- [ ] Commit: `feat(calendar-sync): add incremental Google OAuth flow and connection endpoints`.

### Task 8: `calendarsync` event writer + schedule/resync endpoints

**Files (prog-strength-api):**
- Create: `internal/calendarsync/client.go` (Google Calendar event create/patch/delete + access-token minting from the stored refresh token, with a short in-memory token cache), `internal/calendarsync/render.go` (detail-level event summary/description rendering), `internal/calendarsync/service.go` (orchestration: load connection, mint token, write event, persist `google_sync_status`/`google_event_id`/`last_sync_error` on the planned workout), tests
- Modify: `internal/planned_workout/handler.go` (`POST /planned-workouts/{id}/schedule`, `POST /planned-workouts/{id}/resync`; honor `calendar_sync: true` on create/update), `internal/server/server.go` (inject the calendarsync service into the planned_workout handler — `SetCalendarSync(svc)` setter, mirroring `workoutHandler.SetPublisher`)

Event writer:
- Mint access token from refresh token via `oauth2.Config.TokenSource` using a token with only the refresh token set; cache the access token in memory keyed by user until near expiry.
- `events.insert` on schedule (store `google_event_id`, status `synced`).
- `events.patch` on edit/complete (full rewrite of summary/description/start/end; idempotent).
- `events.delete` on delete.
- Start/end use the plan's `scheduled_start_utc`/`scheduled_end_utc` with the plan's `timezone`.

Detail rendering (`render.go`): effective level = `pw.CalendarDetail ?? user.CalendarDefaultDetail`.
- `time_block`: summary `name` (or "Planned workout"); description = a reserved-slot note + a link back to Prog Strength.
- `full_agenda`: description lists exercises → sets with targets (reps × weight @RPE).

Failure handling (best-effort, Prog Strength stays source of truth):
- success → `SetGoogleSync(eventID, synced, nil)`.
- failure → `SetGoogleSync(prevEventID, failed, errMsg)`, plan persists.
- Google `404` on patch/delete → `SetGoogleSync(nil, ... )` (drop event id, mark unsynced/resyncable).
- Refresh failure due to revoked token → `calendarconn.SetStatus(revoked)`, mark affected plan resyncable; the UI prompts re-consent.

Endpoints: `POST /{id}/schedule` body `{detail_level?}` → set `google_sync_status=pending`, attempt write; `POST /{id}/resync` re-attempts. `calendar_sync: true` on create/update triggers a schedule write after the DB write. All writes are synchronous best-effort; a Google error never fails the API write (return 200/201 with the plan's sync status reflected).

> Wire the MCP `schedule_workout_to_calendar` tool (already forwarding to `/schedule` from Task 4) — no MCP change needed; just confirm the endpoint contract matches.

**Steps (TDD):** `calendarsync` tests with a faked Google client interface: token-mint, event insert/patch/delete, both detail renderings (assert description content), and each failure path (failed status persisted, 404 drops event id, revoked flips connection status). planned_workout handler tests for schedule/resync happy + failure paths using a fake service.
- [ ] Run (fail), implement, `go test ./...`, `go build ./...`.
- [ ] Commit: `feat(calendar-sync): write Google events with detail levels and resyncable failure handling`.

### Task 9: infra — provision `CALENDAR_TOKEN_ENC_KEY`

**Files (prog-strength-infra):**
- Modify: `compose/api/docker-compose.yml` (add `- CALENDAR_TOKEN_ENC_KEY=${CALENDAR_TOKEN_ENC_KEY:-}` and, if a calendar redirect var is introduced, `- GOOGLE_CALENDAR_REDIRECT_URL=${GOOGLE_CALENDAR_REDIRECT_URL:-}` to the `api` service `environment:` block, next to `GOOGLE_CLIENT_SECRET`)

Use the `${VAR:-}` default form (the value is delivered via the host `.env` written by the API repo's deploy workflow — not committed here). Add a short note in the repo's docs/AGENTS if a secrets list exists, documenting the new var and that losing/rotating it invalidates stored refresh tokens (users re-consent).

**Steps:**
- [ ] Edit the compose file; `terraform fmt -recursive` + `terraform validate` (the compose file is not Terraform, but run repo lint/`docker compose config` if available to sanity-check YAML).
- [ ] Commit: `feat(api): wire CALENDAR_TOKEN_ENC_KEY into the API environment`.

### Task 10: web — planned workouts on the calendar, create/edit form, Connect Google Calendar, detail toggles, sync/resync

**Files (prog-strength-web):**
- Modify: `lib/api.ts` (types + client fns: `PlannedWorkout`, `listPlannedWorkouts`, `createPlannedWorkout`, `updatePlannedWorkout`, `deletePlannedWorkout`, `skipPlannedWorkout`, `schedulePlannedWorkout`, `resyncPlannedWorkout`, `getCalendarConnection`, `disconnectCalendar`; extend `meResponse`/profile type with `timezone` + `calendar_default_detail`), `app/(app)/calendar/page.tsx` (fetch + render planned workouts as forward-looking events), `components/calendar/types.ts` (add a `planned` event kind), `components/calendar/day-cell.tsx` + `day-digest.tsx` (render planned pills distinctly), `app/(app)/settings/page.tsx` (Connect Google Calendar section + `calendar_default_detail` control)
- Create: `components/planned-workout-modal.tsx` (create/edit form, mirrors `workout-modal.tsx`), `app/auth/calendar-callback/page.tsx` (optional — if the calendar OAuth uses a web return_to), `components/calendar/connect-google-calendar.tsx`
- Test: `app/(app)/calendar/page.test.tsx` (extend), `components/planned-workout-modal.test.tsx`, settings test

Scope:
- Calendar: add a fourth parallel fetch `listPlannedWorkouts(token, {since, until})`; bucket planned workouts by local date; render as visually-distinct forward-looking pills (e.g. dashed/outlined accent) with status (planned/completed/skipped) and per-plan sync status + a resync affordance when `google_sync_status==='failed'`.
- Create/edit modal: time window (datetime-local start/end), name, notes, optional agenda (exercises from the catalog + target sets reps/weight/rpe), `calendar_detail` override (default/time_block/full_agenda), and a "sync to Google Calendar" checkbox when connected.
- Settings: a "Google Calendar" section reading `getCalendarConnection` — shows Connect (links to `${apiUrl}/auth/google/calendar/connect?return_to=...`) or Connected + Disconnect; plus a `calendar_default_detail` segmented control (`Time block` / `Full agenda`) persisted via `PATCH /me`.

**Steps (TDD where practical — Vitest + Testing Library, mock `lib/api`):**
- [ ] Tests: calendar renders a planned pill on its date distinctly from a logged workout; modal submits the right create payload (incl agenda); settings shows Connect when absent and Disconnect when connected; detail control PATCHes `/me`.
- [ ] Run (fail), implement, `npm run test`, `npm run typecheck`, `npm run lint` (pass).
- [ ] Commit: `feat(calendar): render planned workouts, add create/edit form and Google Calendar connect`.

**Phase 3 exit:** an opted-in user (or the agent) schedules a planned workout and it appears on Google Calendar; disconnecting revokes cleanly.

---

## Phase 4 — Completion propagation (prog-strength-api, prog-strength-mcp, prog-strength-web)

### Task 11: API — `POST /planned-workouts/{id}/complete` + Google rewrite

**Files (prog-strength-api):**
- Modify: `internal/planned_workout/handler.go` (`complete` handler on `POST /{id}/complete` with `{session_id, session_kind}`), `handler_test.go`. Optionally accept an optional `planned_workout_id` on the live-workout finish path (only if low-risk; otherwise the explicit complete endpoint suffices for v1).

`complete`: load plan (404 if missing/cross-user), validate `session_kind` in `workout`/`activity`, `repo.SetCompletion(...)` (status→completed + link), then if the plan is Google-synced (`google_event_id != nil`), call the calendarsync service to **rewrite** the event with actual logged details + a "completed" marker (best-effort; persist sync status). Return 200 with the updated plan.

**Steps (TDD):** test complete sets status+link; cross-user → 404; invalid kind → 400; when synced, the calendarsync service's patch is invoked (fake service asserts the call); when not synced, no Google call.
- [ ] Run (fail), implement, `go test ./...`.
- [ ] Commit: `feat(planned-workouts): complete a plan and rewrite its Google event with actuals`.

### Task 12: MCP — confirm `complete_planned_workout` against the live endpoint

**Files (prog-strength-mcp):**
- Modify: `tests/test_planned_workouts_tools.py` (add/confirm a test that `complete_planned_workout` forwards `{session_id, session_kind}` to `POST /planned-workouts/{id}/complete`). The tool itself already exists from Task 4; this task verifies the contract and adjusts the body shape if needed.

**Steps:**
- [ ] Add the respx test; run `uv run pytest`.
- [ ] Commit (only if changes): `test(planned-workouts): verify complete tool forwards session link`.

### Task 13: web — link a logged session to a plan + show fulfilled state

**Files (prog-strength-web):**
- Modify: `lib/api.ts` (`completePlannedWorkout(token, id, {session_id, session_kind})`), the calendar/digest to show a completed plan linked to its session, and (if scoped in) a "mark as completed / link session" affordance.
- Test: extend the calendar test for the completed rendering.

**Steps:**
- [ ] Tests for completed-state rendering + the complete call; implement; `npm run test`/`typecheck`/`lint`.
- [ ] Commit: `feat(calendar): show planned workouts completing into logged sessions`.

**Phase 4 exit:** the full lifecycle — empty block → planned agenda → completed session with actuals — is visible in both Prog Strength and Google Calendar.

---

## Rollout / Ops prerequisites (from the SOW)

Before Phase 3 deploys:
1. Google Cloud console: enable the Calendar API and add `calendar.events` to the OAuth consent screen for the existing client. One-time, manual.
2. Provision `CALENDAR_TOKEN_ENC_KEY` (32 random bytes, base64-encoded) via the API's existing secret delivery, managed in `prog-strength-infra`. Losing/rotating it invalidates stored refresh tokens (users re-consent) — document the rotation procedure.

All migrations are additive; no existing data changes. The feature is dormant for any user who never connects a calendar.

## Self-Review notes

- Spec coverage: Phase 1 (Tasks 1–3) covers the entity + CRUD + week query + skip + the `users.timezone`/`calendar_default_detail` columns. Phase 2 (4–5) covers MCP tools + agent "plan my week". Phase 3 (6–10) covers encryption, storage, OAuth, event writing with both detail levels, failure handling, infra secret, and the web surfaces. Phase 4 (11–13) covers completion propagation across API/MCP/web. Non-goals (two-way sync, planned runs, mobile, inferred matching, async queue/KMS, non-Google providers) are excluded.
- The single-writer invariant holds: only `internal/planned_workout` + `internal/calendarsync` write the DB and Google; MCP/agent/web only call the API.
- Type consistency: `CalendarDetail`, `Status`, `SessionKind`, `GoogleSyncStatus` enums are defined once in Task 2 and reused in Tasks 7/11. The MCP/web field names (`scheduled_start`, `scheduled_end`, `calendar_detail`, `detail_level`, `session_kind`) match the API DTOs.
</content>
</invoke>
