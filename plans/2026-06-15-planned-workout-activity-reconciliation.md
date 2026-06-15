# Planned Workout ↔ Activity Reconciliation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** When a run/lift is logged, automatically complete-and-link the matching planned session; give a one-tap unlink escape hatch, revert links on delete, surface the reverse relationship in the web UI, add Go-migration support to the DB runner, and backfill existing history.

**Architecture:** A new `planned_workout.Service` owns the single completion code path (`LinkCompletion`) and the matcher (`OnSessionLogged`/`OnSessionDeleted`). The `activity` and `workout` packages each declare a consumer-owned `PlanMatcher` port (same seam style as `timeline.Publisher`) fired best-effort at ingest and delete; server wiring adapts the port to the shared service. The DB runner is extended so Go migrations share the SQL migrations' version ledger and per-migration transaction; the first Go migration backfills links with a frozen copy of the matcher rule. The web app gains an unlink API call, a reverse-lookup endpoint, and "✓ Completes your planned …" + Unlink affordances on the run/workout detail pages and the planned banner.

**Tech Stack:** Go (chi, database/sql, go-sqlite3), Next.js (App Router, TypeScript), Vitest.

**Repos:** `prog-strength-api`, `prog-strength-web`, `prog-strength-docs`. Work on branch `feat/planned-workout-activity-reconciliation` in each (already created).

---

## Key facts about the existing code (read before starting)

- **`planned_workout` has no service layer today** — `internal/planned_workout/handler.go` calls `repo` + `calendar` directly. The `complete` handler (handler.go ~550-624) does: validate body → `repo.Get` (404 guard) → `repo.SetCompletion` → best-effort `calendar.RewriteCompleted` (only when `plan.GoogleEventID != nil && *plan.GoogleEventID != ""` and `h.calendar != nil`) → re-`Get` → return `toDTO`.
- **`CalendarScheduler`** interface (handler.go ~45-54): `Schedule`, `Resync`, `Delete`, `RewriteCompleted`. `Resync(ctx, userID, planID)` re-renders the event for the plan **in its current status** using the non-completed renderer — so after clearing completion (status→planned) it renders the normal planned event. This is the inverse of `RewriteCompleted`.
- **Repository** interface (`internal/planned_workout/repository.go`): has `Get`, `List(ctx, userID, since, until *time.Time)` (returns non-deleted plans whose `ScheduledStartUTC ∈ [since, until)`, **all statuses**, ascending), `SetCompletion`, `SetStatus`, etc. Two impls: `SQLiteRepository` (raw SQL, `internal/planned_workout/sqlite_repository.go`) and `MemoryRepository` (`memory_repository.go`). Shared contract tests in `repository_shared_test.go`.
- **`SetCompletion` SQL** (sqlite_repository.go ~223): `UPDATE planned_workouts SET status=?, completed_session_id=?, completed_session_kind=?, updated_at=? WHERE id=? AND user_id=? AND deleted_at IS NULL`, wrapped in `affectedOrNotFound(res, err)`.
- **PlannedWorkout** fields used here: `ID, UserID, Name *string, ActivityKind` (`ActivityKindLift="lift"`/`ActivityKindRun="run"`), `ScheduledStartUTC time.Time`, `Timezone string` (IANA), `Status` (`StatusPlanned/StatusCompleted/StatusSkipped`), `CompletedSessionID *string`, `CompletedSessionKind *SessionKind` (`SessionKindWorkout="workout"`/`SessionKindActivity="activity"`), `GoogleEventID *string`.
- **Activity** (`internal/activity/model.go`): `ID, UserID string`, `ActivityType` (`ActivityRunning="running"`), `StartTime time.Time` (UTC). Created in `IngestTCX` (`internal/activity/ingest.go` ~34-70) via `repo.Create`; the **handler** `uploadTCX` (`internal/activity/handler.go` ~212-293) calls `IngestTCX` then best-effort `h.publish(...)` for running activities. Delete: handler `delete` (~452-472) → `repo.SoftDelete(userID, id)`.
- **Workout** (`internal/workout/workout.go`): `ID, UserID string`, `PerformedAt time.Time` (UTC). Created in handler `create` (`internal/workout/handler.go` ~421-490) via `h.repo.Create`, then best-effort `h.publishWorkoutPosts(...)`. Delete: handler `delete` (~600-638) → `h.repo.Delete(workoutID)`.
- **Port pattern to copy** — `timeline.Publisher`: interface declared in `internal/timeline/repository.go` (~86-94), implemented by `internal/server/timeline_publisher.go`, injected via `handler.SetPublisher(p)`, called through a nil-safe `publish` helper (`if h.publisher == nil { return }`). Activity/workout never import `timeline`.
- **Server wiring** (`internal/server/server.go`): `plannedWorkoutRepo` built ~140/185; `plannedWorkoutHandler := plannedworkout.NewHandler(plannedWorkoutRepo, userRepo)` + `SetCalendarSync` ~423-427; `calendarScheduler *calendarsync.Service` built ~330-373 (nil when calendar unconfigured); `activityHandler := activity.NewHandler(activityRepo)` + `SetPublisher` ~438-440; `workoutHandler := workout.NewHandler(workoutRepo, exerciseRepo)` + `SetPublisher` ~399-401.
- **DB runner** (`internal/db/migrate.go`): `//go:embed migrations/*.sql`; `Migrate(db *sql.DB)` ensures `schema_migrations(version INTEGER PRIMARY KEY, applied_at …)`, parses `NNN_` versions, sorts, applies pending each in a tx via `applyMigration`, records version. Highest existing SQL migration: **027** (`027_planned_workout_run_kind.sql`). Next free version: **028**. Tests in `internal/db/migrate_test.go` use `newMigratedDB(t)` (fresh `sqlite3` file in `t.TempDir()` with `_foreign_keys=on&_journal_mode=WAL`).
- **Web** (`prog-strength-web`): all API calls are plain `fetch` in `lib/api.ts` (envelope `{data}` unwrapped by `unwrap<T>`); existing `completePlannedWorkout`/`skipPlannedWorkout`/`resyncPlannedWorkout` (~2550-2607). Types `PlannedWorkout` (~2415), `RunningSession` (~1600), `Workout` (~58). Run detail `app/(app)/running/[id]/page.tsx`; workout detail `app/(app)/workouts/[id]/page.tsx`; `components/calendar/planned-banner.tsx` already renders the "View logged run/workout →" link (~145-163). Tests are Vitest; mock `@/lib/api` and `@/lib/auth`, stub `fetch` with `{ ok, status, json: async () => ({ data }) }`.

---

## Phase 1 — API engine (live auto-link)

### Task 1: Repository methods — `ClearCompletion` + `GetByCompletedSession`

**Files:**
- Modify: `internal/planned_workout/repository.go` (interface)
- Modify: `internal/planned_workout/sqlite_repository.go`
- Modify: `internal/planned_workout/memory_repository.go`
- Test: `internal/planned_workout/repository_shared_test.go`

- [ ] **Step 1: Add both methods to the `Repository` interface** in `repository.go`, after `SetCompletion`:

```go
	// ClearCompletion reverts a completed plan to "planned" and clears its
	// completion link. Inverse of SetCompletion. Returns ErrNotFound when the
	// plan is missing, soft-deleted, or cross-user.
	ClearCompletion(ctx context.Context, userID, id string) error

	// GetByCompletedSession returns the user's non-deleted plan whose completion
	// link points at (sessionID, kind), or ErrNotFound when none does.
	GetByCompletedSession(ctx context.Context, userID, sessionID string, kind SessionKind) (*PlannedWorkout, error)
```

- [ ] **Step 2: Write failing shared contract tests** in `repository_shared_test.go`. Find the existing table of shared subtests (a function run against both impls; mirror its style and the existing `SetCompletion` subtest). Add:

```go
	t.Run("ClearCompletion reverts status and clears link", func(t *testing.T) {
		repo := newRepo(t)
		pw := seedPlanned(t, repo) // helper used by existing tests; a planned lift plan
		if err := repo.SetCompletion(ctx, pw.UserID, pw.ID, "sess-1", SessionKindWorkout); err != nil {
			t.Fatalf("SetCompletion: %v", err)
		}
		if err := repo.ClearCompletion(ctx, pw.UserID, pw.ID); err != nil {
			t.Fatalf("ClearCompletion: %v", err)
		}
		got, err := repo.Get(ctx, pw.UserID, pw.ID)
		if err != nil {
			t.Fatalf("Get: %v", err)
		}
		if got.Status != StatusPlanned {
			t.Errorf("status = %q, want planned", got.Status)
		}
		if got.CompletedSessionID != nil || got.CompletedSessionKind != nil {
			t.Errorf("link not cleared: id=%v kind=%v", got.CompletedSessionID, got.CompletedSessionKind)
		}
	})

	t.Run("ClearCompletion cross-user is ErrNotFound", func(t *testing.T) {
		repo := newRepo(t)
		pw := seedPlanned(t, repo)
		if err := repo.ClearCompletion(ctx, "other-user", pw.ID); !errors.Is(err, ErrNotFound) {
			t.Errorf("err = %v, want ErrNotFound", err)
		}
	})

	t.Run("GetByCompletedSession finds the linking plan", func(t *testing.T) {
		repo := newRepo(t)
		pw := seedPlanned(t, repo)
		if err := repo.SetCompletion(ctx, pw.UserID, pw.ID, "act-9", SessionKindActivity); err != nil {
			t.Fatalf("SetCompletion: %v", err)
		}
		got, err := repo.GetByCompletedSession(ctx, pw.UserID, "act-9", SessionKindActivity)
		if err != nil {
			t.Fatalf("GetByCompletedSession: %v", err)
		}
		if got.ID != pw.ID {
			t.Errorf("got plan %s, want %s", got.ID, pw.ID)
		}
	})

	t.Run("GetByCompletedSession no match is ErrNotFound", func(t *testing.T) {
		repo := newRepo(t)
		if _, err := repo.GetByCompletedSession(ctx, "u1", "nope", SessionKindActivity); !errors.Is(err, ErrNotFound) {
			t.Errorf("err = %v, want ErrNotFound", err)
		}
	})
```

Adapt `newRepo`/`seedPlanned`/`ctx` names to whatever the existing shared test harness uses. If there is no `seedPlanned` helper, create a planned plan inline using the same construction the existing `SetCompletion` subtest uses.

- [ ] **Step 3: Run tests, verify they fail** (compile error / not implemented):

Run: `cd /workspace/prog-strength-api && go test ./internal/planned_workout/ -run 'ClearCompletion|GetByCompletedSession' -v`
Expected: FAIL (methods undefined).

- [ ] **Step 4: Implement in `sqlite_repository.go`.** Mirror the `SetCompletion` style (`r.now().UTC()`, `affectedOrNotFound`). For `GetByCompletedSession`, reuse the package's existing row-scan/select helper used by `Get` (find how `Get` scans a `*PlannedWorkout`, including agenda hydration, and reuse it):

```go
func (r *SQLiteRepository) ClearCompletion(ctx context.Context, userID, planID string) error {
	now := r.now().UTC()
	res, err := r.db.ExecContext(ctx, `
		UPDATE planned_workouts SET
			status = ?, completed_session_id = NULL, completed_session_kind = NULL, updated_at = ?
		WHERE id = ? AND user_id = ? AND deleted_at IS NULL
	`, string(StatusPlanned), now, planID, userID)
	return affectedOrNotFound(res, err)
}

func (r *SQLiteRepository) GetByCompletedSession(ctx context.Context, userID, sessionID string, kind SessionKind) (*PlannedWorkout, error) {
	var id string
	err := r.db.QueryRowContext(ctx, `
		SELECT id FROM planned_workouts
		WHERE user_id = ? AND completed_session_id = ? AND completed_session_kind = ? AND deleted_at IS NULL
		LIMIT 1
	`, userID, sessionID, string(kind)).Scan(&id)
	if errors.Is(err, sql.ErrNoRows) {
		return nil, ErrNotFound
	}
	if err != nil {
		return nil, err
	}
	return r.Get(ctx, userID, id) // reuse full hydration (agenda, etc.)
}
```

If `errors`/`database/sql` are not already imported in the file, add them.

- [ ] **Step 5: Implement in `memory_repository.go`.** Mirror the existing `SetCompletion` (mutex, `ErrNotFound` on missing/cross-user/soft-deleted). Return a deep copy consistent with how other memory getters return (`Get`):

```go
func (r *MemoryRepository) ClearCompletion(ctx context.Context, userID, planID string) error {
	r.mu.Lock()
	defer r.mu.Unlock()
	existing, ok := r.plans[planID]
	if !ok || existing.UserID != userID || existing.DeletedAt != nil {
		return ErrNotFound
	}
	existing.Status = StatusPlanned
	existing.CompletedSessionID = nil
	existing.CompletedSessionKind = nil
	existing.UpdatedAt = r.nowFunc().UTC()
	return nil
}

func (r *MemoryRepository) GetByCompletedSession(ctx context.Context, userID, sessionID string, kind SessionKind) (*PlannedWorkout, error) {
	r.mu.RLock()
	var foundID string
	for id, p := range r.plans {
		if p.UserID == userID && p.DeletedAt == nil &&
			p.CompletedSessionID != nil && *p.CompletedSessionID == sessionID &&
			p.CompletedSessionKind != nil && *p.CompletedSessionKind == kind {
			foundID = id
			break
		}
	}
	r.mu.RUnlock()
	if foundID == "" {
		return nil, ErrNotFound
	}
	return r.Get(ctx, userID, foundID)
}
```

Match the memory repo's actual mutex type (`sync.Mutex` vs `sync.RWMutex`) and field/clock names (`r.nowFunc`/`r.now`) — adapt to what's there. If it uses `sync.Mutex`, use `r.mu.Lock()/Unlock()` in the getter too.

- [ ] **Step 6: Run tests, verify pass:**

Run: `cd /workspace/prog-strength-api && go test ./internal/planned_workout/ -v` then `go build ./...`
Expected: PASS, build clean.

- [ ] **Step 7: Commit**

```bash
cd /workspace/prog-strength-api
git add internal/planned_workout
git commit -m "feat(planned-workouts): add ClearCompletion and GetByCompletedSession repo methods"
```

---

### Task 2: `planned_workout.Service` — `LinkCompletion` + `Unlink`; refactor `complete`; add `unlink` & `by-session` endpoints

**Files:**
- Create: `internal/planned_workout/service.go`
- Modify: `internal/planned_workout/handler.go`
- Test: `internal/planned_workout/service_test.go`
- Test: `internal/planned_workout/handler_complete_test.go` (keep green; extend for unlink)

**Context:** This extracts the shared completion path so the HTTP handler and the auto-matcher (Task 3) are identical downstream, and adds the unlink + reverse-lookup HTTP surface. The handler keeps `repo`/`userRepo`/`calendar` fields but routes completion/unlink through the service. The service holds `repo` and a nil-safe `calendar CalendarScheduler`.

- [ ] **Step 1: Create `service.go`:**

```go
package planned_workout

import (
	"context"
	"log"
)

// Service owns the single completion code path shared by the HTTP complete
// handler and the auto-matcher, plus the unlink inverse. Google Calendar
// rewrites are best-effort: a Google failure is logged, never returned.
type Service struct {
	repo     Repository
	calendar CalendarScheduler // may be nil when calendar sync is unconfigured
}

func NewService(repo Repository) *Service {
	return &Service{repo: repo}
}

// SetCalendar wires the (optional) calendar scheduler after construction,
// mirroring Handler.SetCalendarSync.
func (s *Service) SetCalendar(c CalendarScheduler) { s.calendar = c }

// LinkCompletion marks plan planID completed and links it to (sessionID, kind),
// then best-effort rewrites the Google event to its completed form. Returns the
// updated plan. ErrNotFound when the plan is missing/cross-user; validation
// errors are surfaced for the caller to translate to 400.
func (s *Service) LinkCompletion(ctx context.Context, userID, planID, sessionID string, kind SessionKind) (*PlannedWorkout, error) {
	plan, err := s.repo.Get(ctx, userID, planID)
	if err != nil {
		return nil, err
	}
	if err := s.repo.SetCompletion(ctx, userID, planID, sessionID, kind); err != nil {
		return nil, err
	}
	if plan.GoogleEventID != nil && *plan.GoogleEventID != "" && s.calendar != nil {
		actualText := "Completed — logged " + string(kind) + " session " + sessionID
		if err := s.calendar.RewriteCompleted(ctx, userID, planID, actualText); err != nil {
			log.Printf("planned-workout link completion: rewrite google event (plan %s): %v", planID, err)
		}
	}
	return s.repo.Get(ctx, userID, planID)
}

// Unlink reverts a completed plan to planned, clears the link, and best-effort
// re-renders the Google event back to its non-completed form (via Resync, which
// renders the plan in its current — now planned — status).
func (s *Service) Unlink(ctx context.Context, userID, planID string) (*PlannedWorkout, error) {
	plan, err := s.repo.Get(ctx, userID, planID)
	if err != nil {
		return nil, err
	}
	if err := s.repo.ClearCompletion(ctx, userID, planID); err != nil {
		return nil, err
	}
	if plan.GoogleEventID != nil && *plan.GoogleEventID != "" && s.calendar != nil {
		if err := s.calendar.Resync(ctx, userID, planID); err != nil {
			log.Printf("planned-workout unlink: re-render google event (plan %s): %v", planID, err)
		}
	}
	return s.repo.Get(ctx, userID, planID)
}
```

- [ ] **Step 2: Write failing `service_test.go`** using `MemoryRepository` and a fake `CalendarScheduler` recording calls. Model the fake on whatever fake the existing `handler_complete_test.go` uses (reuse it if exported in-package):

```go
package planned_workout

import (
	"context"
	"testing"
)

type fakeCalendar struct {
	rewriteCompletedCalls int
	resyncCalls           int
}

func (f *fakeCalendar) Schedule(ctx context.Context, userID, planID, detail string) error { return nil }
func (f *fakeCalendar) Resync(ctx context.Context, userID, planID string) error           { f.resyncCalls++; return nil }
func (f *fakeCalendar) Delete(ctx context.Context, userID, planID string) error           { return nil }
func (f *fakeCalendar) RewriteCompleted(ctx context.Context, userID, planID, actualText string) error {
	f.rewriteCompletedCalls++
	return nil
}

func TestService_LinkCompletion_setsStatusAndRewrites(t *testing.T) {
	ctx := context.Background()
	repo := NewMemoryRepository()
	pw := mustSeedSyncedPlan(t, repo) // planned plan WITH a non-empty GoogleEventID
	cal := &fakeCalendar{}
	svc := NewService(repo)
	svc.SetCalendar(cal)

	got, err := svc.LinkCompletion(ctx, pw.UserID, pw.ID, "act-1", SessionKindActivity)
	if err != nil {
		t.Fatalf("LinkCompletion: %v", err)
	}
	if got.Status != StatusCompleted || got.CompletedSessionID == nil || *got.CompletedSessionID != "act-1" {
		t.Fatalf("plan not linked: %+v", got)
	}
	if cal.rewriteCompletedCalls != 1 {
		t.Errorf("rewriteCompleted calls = %d, want 1", cal.rewriteCompletedCalls)
	}
}

func TestService_Unlink_revertsAndResyncs(t *testing.T) {
	ctx := context.Background()
	repo := NewMemoryRepository()
	pw := mustSeedSyncedPlan(t, repo)
	cal := &fakeCalendar{}
	svc := NewService(repo)
	svc.SetCalendar(cal)
	if _, err := svc.LinkCompletion(ctx, pw.UserID, pw.ID, "act-1", SessionKindActivity); err != nil {
		t.Fatalf("link: %v", err)
	}
	got, err := svc.Unlink(ctx, pw.UserID, pw.ID)
	if err != nil {
		t.Fatalf("Unlink: %v", err)
	}
	if got.Status != StatusPlanned || got.CompletedSessionID != nil {
		t.Fatalf("plan not reverted: %+v", got)
	}
	if cal.resyncCalls != 1 {
		t.Errorf("resync calls = %d, want 1", cal.resyncCalls)
	}
}
```

Write `mustSeedSyncedPlan` as a small helper that constructs and `repo.Create`s a planned plan with `GoogleEventID` set to a non-empty pointer (use the same plan construction the existing package tests use; set `GoogleEventID` to a pointer to e.g. "evt-1"). If the existing tests already export such a helper, reuse it.

- [ ] **Step 3: Run, verify fail.** Run: `cd /workspace/prog-strength-api && go test ./internal/planned_workout/ -run 'TestService_' -v` → FAIL (undefined).

- [ ] **Step 4: Wire the service into the Handler.** In `handler.go`:
  - Add a field `svc *Service` to the `Handler` struct.
  - In `NewHandler`, set `h.svc = NewService(repo)` (so the handler always has a usable service even without calendar).
  - In `SetCalendarSync(s CalendarScheduler)`, additionally call `h.svc.SetCalendar(s)` (keep the existing `h.calendar = s`).
  - Add an accessor `func (h *Handler) Service() *Service { return h.svc }` (server wiring shares this instance with the matcher).

- [ ] **Step 5: Refactor the `complete` handler body** to call the service. Replace the `repo.Get` guard + `repo.SetCompletion` + inline Google rewrite + re-`Get` block with:

```go
	updated, err := h.svc.LinkCompletion(r.Context(), userID, id, req.SessionID, kind)
	if err != nil {
		if errors.Is(err, ErrNotFound) {
			httpresp.Error(w, http.StatusNotFound, "planned workout not found")
			return
		}
		if isValidationError(err) {
			httpresp.Error(w, http.StatusBadRequest, err.Error())
			return
		}
		httpresp.ServerError(w, r.Context(), "complete planned workout", err)
		return
	}
	httpresp.OK(w, "planned workout completed", toDTO(updated))
```

Keep the body decode + `session_id`/`session_kind` required + kind-enum validation exactly as-is above this block.

- [ ] **Step 6: Add the `unlink` handler and route.** In `Mount`, add inside the `/planned-workouts` route group: `r.Post("/{id}/unlink", h.unlink)`. Add the handler (mirror `skip`'s auth/id guards):

```go
func (h *Handler) unlink(w http.ResponseWriter, r *http.Request) {
	userID, ok := auth.UserIDFrom(r.Context())
	if !ok {
		httpresp.ServerError(w, r.Context(), "missing user in context", errors.New("auth middleware not applied"))
		return
	}
	id := chi.URLParam(r, "id")
	if id == "" {
		httpresp.Error(w, http.StatusBadRequest, "planned workout id is required")
		return
	}
	updated, err := h.svc.Unlink(r.Context(), userID, id)
	if err != nil {
		if errors.Is(err, ErrNotFound) {
			httpresp.Error(w, http.StatusNotFound, "planned workout not found")
			return
		}
		httpresp.ServerError(w, r.Context(), "unlink planned workout", err)
		return
	}
	httpresp.OK(w, "planned workout unlinked", toDTO(updated))
}
```

- [ ] **Step 7: Add the reverse-lookup `by-session` handler and route.** In `Mount`, add (place it BEFORE `r.Get("/{id}", h.get)` so the literal path is matched first — chi matches static before param, but list it before to be safe): `r.Get("/by-session", h.bySession)`. Handler:

```go
func (h *Handler) bySession(w http.ResponseWriter, r *http.Request) {
	userID, ok := auth.UserIDFrom(r.Context())
	if !ok {
		httpresp.ServerError(w, r.Context(), "missing user in context", errors.New("auth middleware not applied"))
		return
	}
	sessionID := r.URL.Query().Get("session_id")
	kindStr := r.URL.Query().Get("session_kind")
	if sessionID == "" || kindStr == "" {
		httpresp.Error(w, http.StatusBadRequest, "session_id and session_kind are required")
		return
	}
	kind := SessionKind(kindStr)
	if kind != SessionKindWorkout && kind != SessionKindActivity {
		httpresp.Error(w, http.StatusBadRequest, "session_kind must be 'workout' or 'activity'")
		return
	}
	plan, err := h.repo.GetByCompletedSession(r.Context(), userID, sessionID, kind)
	if err != nil {
		if errors.Is(err, ErrNotFound) {
			httpresp.Error(w, http.StatusNotFound, "no planned workout completed by this session")
			return
		}
		httpresp.ServerError(w, r.Context(), "lookup planned workout by session", err)
		return
	}
	httpresp.OK(w, "planned workout found", toDTO(plan))
}
```

- [ ] **Step 8: Add handler-level tests for unlink** in `handler_complete_test.go` (or a new `handler_unlink_test.go`), mirroring the existing complete handler test setup (router build, auth context injection, fake calendar): POST `/planned-workouts/{id}/unlink` on a completed plan returns 200 with `status:"planned"` and cleared link; unknown id returns 404. Add one `bySession` test: after completing a plan, `GET /planned-workouts/by-session?session_id=…&session_kind=…` returns the plan; missing params → 400; no match → 404.

- [ ] **Step 9: Run tests + build:**

Run: `cd /workspace/prog-strength-api && go test ./internal/planned_workout/ -v && go build ./...`
Expected: PASS, clean.

- [ ] **Step 10: Commit**

```bash
cd /workspace/prog-strength-api
git add internal/planned_workout
git commit -m "feat(planned-workouts): add Service (LinkCompletion/Unlink), unlink + by-session endpoints"
```

---

### Task 3: Matcher rule + `OnSessionLogged` / `OnSessionDeleted` on the service

**Files:**
- Create: `internal/planned_workout/matcher.go`
- Test: `internal/planned_workout/matcher_test.go`
- Modify: `internal/planned_workout/service.go` (add the two methods)

**Context:** The matcher selects which planned plan a freshly logged session completes (time + kind only) and links it via `LinkCompletion`. The selection rule is a pure function so it is unit-testable and so the backfill (Task 6) can carry a frozen copy. Best-effort: errors are logged, never returned.

- [ ] **Step 1: Write `matcher.go` with the pure selection rule:**

```go
package planned_workout

import "time"

// kindToActivityKind maps a logged session's kind to the plan ActivityKind it
// can complete: an activity (always a running activity at the hook site)
// completes a "run" plan; a workout completes a "lift" plan.
func kindToActivityKind(kind SessionKind) ActivityKind {
	if kind == SessionKindActivity {
		return ActivityKindRun
	}
	return ActivityKindLift
}

// sameLocalDay reports whether the plan's scheduled start and the session start
// fall on the same calendar day rendered in the plan's own IANA timezone. A plan
// with an unloadable timezone never matches.
func sameLocalDay(plan PlannedWorkout, sessionStartUTC time.Time) bool {
	loc, err := time.LoadLocation(plan.Timezone)
	if err != nil {
		return false
	}
	ps := plan.ScheduledStartUTC.In(loc)
	ss := sessionStartUTC.In(loc)
	py, pm, pd := ps.Date()
	sy, sm, sd := ss.Date()
	return py == sy && pm == sm && pd == sd
}

// selectPlan returns the candidate plan a session of the given kind/start
// completes, or nil. Candidates are planned-status plans of the matching kind on
// the same local day; the winner is the one whose ScheduledStartUTC is closest
// to the session start, breaking ties by earliest ScheduledStartUTC then by ID
// (fully deterministic; exact ties are effectively impossible in practice).
func selectPlan(plans []PlannedWorkout, sessionStartUTC time.Time, kind SessionKind) *PlannedWorkout {
	wantKind := kindToActivityKind(kind)
	var best *PlannedWorkout
	var bestDelta time.Duration
	for i := range plans {
		p := plans[i]
		if p.Status != StatusPlanned || p.ActivityKind != wantKind || p.DeletedAt != nil {
			continue
		}
		if !sameLocalDay(p, sessionStartUTC) {
			continue
		}
		delta := p.ScheduledStartUTC.Sub(sessionStartUTC)
		if delta < 0 {
			delta = -delta
		}
		if best == nil || delta < bestDelta ||
			(delta == bestDelta && (p.ScheduledStartUTC.Before(best.ScheduledStartUTC) ||
				(p.ScheduledStartUTC.Equal(best.ScheduledStartUTC) && p.ID < best.ID))) {
			b := p
			best = &b
			bestDelta = delta
		}
	}
	return best
}
```

- [ ] **Step 2: Write failing `matcher_test.go`** — table-driven, covering every Testing-section case (same-day match, off-schedule same-day match, two-a-day nearest-start, wrong-kind no-match, already-completed guard, timezone-boundary bucketing, no-candidate no-op). Build plans with a small helper. Example shape:

```go
package planned_workout

import (
	"testing"
	"time"
)

func tz(t *testing.T, name string) string { t.Helper(); if _, err := time.LoadLocation(name); err != nil { t.Fatalf("tz %s: %v", name, err) }; return name }

func plan(id string, kind ActivityKind, status Status, startUTC time.Time, timezone string) PlannedWorkout {
	return PlannedWorkout{ID: id, UserID: "u1", ActivityKind: kind, Status: status, ScheduledStartUTC: startUTC, ScheduledEndUTC: startUTC.Add(time.Hour), Timezone: timezone}
}

func TestSelectPlan(t *testing.T) {
	utc := func(y int, mo time.Month, d, h, mi int) time.Time { return time.Date(y, mo, d, h, mi, 0, 0, time.UTC) }
	ny := tz(t, "America/New_York")

	tests := []struct {
		name      string
		plans     []PlannedWorkout
		start     time.Time
		kind      SessionKind
		wantID    string // "" => no match
	}{
		{
			name:   "same-day run match",
			plans:  []PlannedWorkout{plan("p1", ActivityKindRun, StatusPlanned, utc(2026, 6, 15, 17, 30), ny)},
			start:  utc(2026, 6, 15, 18, 0),
			kind:   SessionKindActivity,
			wantID: "p1",
		},
		{
			name:   "off-schedule but same local day still matches",
			plans:  []PlannedWorkout{plan("p1", ActivityKindRun, StatusPlanned, utc(2026, 6, 15, 13, 0), ny)},
			start:  utc(2026, 6, 15, 23, 0), // 7pm NY, same NY day
			kind:   SessionKindActivity,
			wantID: "p1",
		},
		{
			name: "two-a-day nearest scheduled start wins",
			plans: []PlannedWorkout{
				plan("early", ActivityKindRun, StatusPlanned, utc(2026, 6, 15, 11, 0), ny),
				plan("late", ActivityKindRun, StatusPlanned, utc(2026, 6, 15, 22, 0), ny),
			},
			start:  utc(2026, 6, 15, 21, 30),
			kind:   SessionKindActivity,
			wantID: "late",
		},
		{
			name:   "wrong kind: lift plan, run session",
			plans:  []PlannedWorkout{plan("p1", ActivityKindLift, StatusPlanned, utc(2026, 6, 15, 17, 0), ny)},
			start:  utc(2026, 6, 15, 18, 0),
			kind:   SessionKindActivity,
			wantID: "",
		},
		{
			name:   "already completed plan is not a candidate",
			plans:  []PlannedWorkout{plan("p1", ActivityKindRun, StatusCompleted, utc(2026, 6, 15, 17, 0), ny)},
			start:  utc(2026, 6, 15, 18, 0),
			kind:   SessionKindActivity,
			wantID: "",
		},
		{
			name:   "timezone boundary: UTC same day but different NY day -> no match",
			plans:  []PlannedWorkout{plan("p1", ActivityKindRun, StatusPlanned, utc(2026, 6, 15, 2, 0), ny)}, // 10pm Jun 14 NY
			start:  utc(2026, 6, 15, 20, 0),                                                                  // 4pm Jun 15 NY
			kind:   SessionKindActivity,
			wantID: "",
		},
		{
			name:   "no candidates",
			plans:  nil,
			start:  utc(2026, 6, 15, 18, 0),
			kind:   SessionKindActivity,
			wantID: "",
		},
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			got := selectPlan(tt.plans, tt.start, tt.kind)
			if tt.wantID == "" {
				if got != nil {
					t.Fatalf("got %s, want no match", got.ID)
				}
				return
			}
			if got == nil || got.ID != tt.wantID {
				t.Fatalf("got %v, want %s", got, tt.wantID)
			}
		})
	}
}
```

- [ ] **Step 3: Run, verify pass:** `cd /workspace/prog-strength-api && go test ./internal/planned_workout/ -run TestSelectPlan -v` → PASS.

- [ ] **Step 4: Add the two best-effort service methods** to `service.go`:

```go
// OnSessionLogged best-effort links a freshly logged session to the planned
// workout it completes, if any. A no-candidate result is a clean no-op; all
// failures are logged, never returned (matching must not fail ingest).
func (s *Service) OnSessionLogged(ctx context.Context, userID, sessionID string, kind SessionKind, sessionStartUTC time.Time) {
	// Query a generous UTC window around the session so day-bucketing in any
	// timezone is covered; selectPlan does the precise local-day filtering.
	since := sessionStartUTC.Add(-36 * time.Hour)
	until := sessionStartUTC.Add(36 * time.Hour)
	plans, err := s.repo.List(ctx, userID, &since, &until)
	if err != nil {
		log.Printf("plan matcher: list candidates (user %s): %v", userID, err)
		return
	}
	match := selectPlan(plans, sessionStartUTC, kind)
	if match == nil {
		return
	}
	if _, err := s.LinkCompletion(ctx, userID, match.ID, sessionID, kind); err != nil {
		log.Printf("plan matcher: link completion (plan %s, session %s): %v", match.ID, sessionID, err)
	}
}

// OnSessionDeleted reverts the plan (if any) whose completion link points at the
// deleted session back to planned, clearing the link and re-rendering the Google
// event. Best-effort; a no-link session is a clean no-op.
func (s *Service) OnSessionDeleted(ctx context.Context, userID, sessionID string, kind SessionKind) {
	plan, err := s.repo.GetByCompletedSession(ctx, userID, sessionID, kind)
	if errors.Is(err, ErrNotFound) {
		return
	}
	if err != nil {
		log.Printf("plan matcher: reverse lookup (session %s): %v", sessionID, err)
		return
	}
	if _, err := s.Unlink(ctx, userID, plan.ID); err != nil {
		log.Printf("plan matcher: revert on delete (plan %s): %v", plan.ID, err)
	}
}
```

Add `"errors"` and `"time"` to `service.go` imports.

- [ ] **Step 5: Write service matcher integration tests** in `service_test.go`: seed a planned run plan + call `OnSessionLogged(ctx, user, "act-1", SessionKindActivity, sameDayStart)` → plan becomes completed and linked to "act-1"; then `OnSessionDeleted(ctx, user, "act-1", SessionKindActivity)` → plan back to planned, link cleared. Add a no-candidate `OnSessionLogged` test asserting the plan stays planned. Use the `MemoryRepository` and `fakeCalendar`.

- [ ] **Step 6: Run tests + build:** `cd /workspace/prog-strength-api && go test ./internal/planned_workout/ -v && go build ./...` → PASS, clean.

- [ ] **Step 7: Commit**

```bash
cd /workspace/prog-strength-api
git add internal/planned_workout
git commit -m "feat(planned-workouts): add time+kind matcher and OnSessionLogged/OnSessionDeleted"
```

---

### Task 4: `PlanMatcher` ports + ingest/delete hooks in `activity` and `workout`

**Files:**
- Create: `internal/activity/matcher.go`
- Create: `internal/workout/matcher.go`
- Modify: `internal/activity/handler.go`
- Modify: `internal/workout/handler.go`
- Test: `internal/activity/handler_test.go` (or matcher-focused test file)
- Test: `internal/workout/handler_test.go`

**Context:** Mirror the `timeline.Publisher` seam exactly: each package declares the port it owns, the handler holds a nil-safe field set via a setter, and fires it best-effort. Activity fires `OnSessionLogged` only for running activities (v1 strictly matches running); both fire `OnSessionDeleted` on delete. The implementation is provided by server wiring (Task 5).

- [ ] **Step 1: Create `internal/activity/matcher.go`:**

```go
package activity

import (
	"context"
	"time"
)

// SessionRef identifies a freshly logged (or deleted) running activity for the
// plan matcher. StartUTC is the activity start time in UTC, used to bucket the
// session into a local calendar day.
type SessionRef struct {
	SessionID string
	StartUTC  time.Time
}

// PlanMatcher is the seam the activity package depends on to best-effort link a
// logged running activity to the planned workout it completes (and to revert
// that link when the activity is deleted). It is declared here and implemented
// in the server wiring layer so this package never imports planned_workout.
// All methods are best-effort: failures must not affect ingest or delete.
type PlanMatcher interface {
	OnSessionLogged(ctx context.Context, userID string, ref SessionRef)
	OnSessionDeleted(ctx context.Context, userID, sessionID string)
}
```

- [ ] **Step 2: Wire the port into the activity handler.** In `handler.go`: add field `planMatcher PlanMatcher`; add `func (h *Handler) SetPlanMatcher(m PlanMatcher) { h.planMatcher = m }`; add nil-safe helpers:

```go
func (h *Handler) matchSession(ctx context.Context, userID string, ref SessionRef) {
	if h.planMatcher == nil {
		return
	}
	h.planMatcher.OnSessionLogged(ctx, userID, ref)
}

func (h *Handler) unmatchSession(ctx context.Context, userID, sessionID string) {
	if h.planMatcher == nil {
		return
	}
	h.planMatcher.OnSessionDeleted(ctx, userID, sessionID)
}
```

In `uploadTCX`, in the existing `if a.ActivityType == ActivityRunning { … }` block (where timeline posts are published), also fire the matcher:

```go
		h.matchSession(r.Context(), a.UserID, SessionRef{SessionID: a.ID, StartUTC: a.StartTime})
```

In the `delete` handler, after `repo.SoftDelete(...)` succeeds and before writing the 204, add:

```go
	h.unmatchSession(r.Context(), userID, activityID)
```

(The reverse-lookup no-ops for never-linked activities, so deleting a walk is harmless — no need to load the activity type.)

- [ ] **Step 3: Create `internal/workout/matcher.go`** — same shape, kind is implicitly "lift":

```go
package workout

import (
	"context"
	"time"
)

// SessionRef identifies a freshly logged (or deleted) lifting workout for the
// plan matcher. StartUTC is the workout's PerformedAt in UTC.
type SessionRef struct {
	SessionID string
	StartUTC  time.Time
}

// PlanMatcher is the seam the workout package depends on to best-effort link a
// logged lift to the planned workout it completes (and revert on delete).
// Declared here, implemented in server wiring so this package never imports
// planned_workout. Best-effort: failures must not affect create or delete.
type PlanMatcher interface {
	OnSessionLogged(ctx context.Context, userID string, ref SessionRef)
	OnSessionDeleted(ctx context.Context, userID, sessionID string)
}
```

- [ ] **Step 4: Wire the port into the workout handler.** In `handler.go`: add `planMatcher PlanMatcher` field, `SetPlanMatcher`, and the same nil-safe `matchSession`/`unmatchSession` helpers (adapt to workout's `SessionRef`). In `create`, after `h.repo.Create(...)` succeeds (alongside `h.publishWorkoutPosts`):

```go
	h.matchSession(r.Context(), workout.UserID, SessionRef{SessionID: workout.ID, StartUTC: workout.PerformedAt})
```

In `delete`, after `h.repo.Delete(workoutID)` succeeds, add (the delete handler already loaded the workout for its ownership check — reuse that `userID`):

```go
	h.unmatchSession(r.Context(), userID, workoutID)
```

If the workout `delete` handler does not already have `userID` in scope, use the user id it resolved for the ownership check (find it; it does an ownership lookup).

- [ ] **Step 5: Tests** — add a small fake `PlanMatcher` in each package's test file recording `OnSessionLogged`/`OnSessionDeleted` calls. Assert:
  - activity: a TCX upload of a running activity calls `OnSessionLogged` once with the new activity's id; a non-running upload does **not**; deleting an activity calls `OnSessionDeleted` with that id. (Reuse the existing handler-test harness that already exercises `uploadTCX`/`delete`; inject the fake via `SetPlanMatcher`.)
  - workout: creating a workout calls `OnSessionLogged` with the new id and `PerformedAt`; deleting calls `OnSessionDeleted`.

  Keep these best-effort — also add one test where the fake returns/panics? No: the port methods return nothing. Instead assert ingest still succeeds when the matcher is nil (don't set it) — the existing tests already cover the nil path.

- [ ] **Step 6: Run + build:** `cd /workspace/prog-strength-api && go test ./internal/activity/ ./internal/workout/ -v && go build ./...` → PASS, clean.

- [ ] **Step 7: Commit**

```bash
cd /workspace/prog-strength-api
git add internal/activity internal/workout
git commit -m "feat(activity,workout): add PlanMatcher port + best-effort ingest/delete hooks"
```

---

### Task 5: Server wiring — adapters connect the ports to the shared service

**Files:**
- Create: `internal/server/plan_matcher.go`
- Modify: `internal/server/server.go`
- Test: build + existing server tests

**Context:** Construct one `planned_workout.Service` (shared with the HTTP handler via its `Service()` accessor so the calendar wiring is identical) and adapt it to `activity.PlanMatcher` and `workout.PlanMatcher`.

- [ ] **Step 1: Create `internal/server/plan_matcher.go`:**

```go
package server

import (
	"context"

	"github.com/jwallace145/progressive-overload-fitness-tracker/internal/activity"
	plannedworkout "github.com/jwallace145/progressive-overload-fitness-tracker/internal/planned_workout"
	"github.com/jwallace145/progressive-overload-fitness-tracker/internal/workout"
)

// activityPlanMatcher adapts the shared planned-workout service to the
// activity.PlanMatcher port (a logged activity is always a running activity at
// the hook site, so it completes a "run" plan).
type activityPlanMatcher struct{ svc *plannedworkout.Service }

var _ activity.PlanMatcher = (*activityPlanMatcher)(nil)

func (m *activityPlanMatcher) OnSessionLogged(ctx context.Context, userID string, ref activity.SessionRef) {
	m.svc.OnSessionLogged(ctx, userID, ref.SessionID, plannedworkout.SessionKindActivity, ref.StartUTC)
}

func (m *activityPlanMatcher) OnSessionDeleted(ctx context.Context, userID, sessionID string) {
	m.svc.OnSessionDeleted(ctx, userID, sessionID, plannedworkout.SessionKindActivity)
}

// workoutPlanMatcher adapts the shared service to the workout.PlanMatcher port
// (a logged workout completes a "lift" plan).
type workoutPlanMatcher struct{ svc *plannedworkout.Service }

var _ workout.PlanMatcher = (*workoutPlanMatcher)(nil)

func (m *workoutPlanMatcher) OnSessionLogged(ctx context.Context, userID string, ref workout.SessionRef) {
	m.svc.OnSessionLogged(ctx, userID, ref.SessionID, plannedworkout.SessionKindWorkout, ref.StartUTC)
}

func (m *workoutPlanMatcher) OnSessionDeleted(ctx context.Context, userID, sessionID string) {
	m.svc.OnSessionDeleted(ctx, userID, sessionID, plannedworkout.SessionKindWorkout)
}
```

Confirm the module path matches the repo's `go.mod` (`github.com/jwallace145/progressive-overload-fitness-tracker`).

- [ ] **Step 2: Wire in `server.go`.** After `plannedWorkoutHandler` is constructed and `SetCalendarSync` is (conditionally) called, get the shared service and inject the adapters into the activity/workout handlers:

```go
	planService := plannedWorkoutHandler.Service()
	activityHandler.SetPlanMatcher(&activityPlanMatcher{svc: planService})
	workoutHandler.SetPlanMatcher(&workoutPlanMatcher{svc: planService})
```

Place these lines after BOTH the `activityHandler`/`workoutHandler` are constructed AND `plannedWorkoutHandler.SetCalendarSync(...)` has run, so the service already has its calendar (when configured). The activity/workout handlers are constructed later than the planned-workout handler in the current file — put the three lines after all three handlers exist (e.g., just before/after `activityHandler.Mount(r)`), and ensure `plannedWorkoutHandler.SetCalendarSync` ran earlier. If ordering makes that awkward, move the `SetPlanMatcher` calls to the end of the handler-construction section.

- [ ] **Step 3: Build + vet + test:**

Run: `cd /workspace/prog-strength-api && go build ./... && go vet ./... && go test ./internal/server/ ./internal/planned_workout/ ./internal/activity/ ./internal/workout/`
Expected: clean, PASS.

- [ ] **Step 4: Commit**

```bash
cd /workspace/prog-strength-api
git add internal/server
git commit -m "feat(server): wire PlanMatcher adapters to the shared planned-workout service"
```

---

## Phase 3 — Go-migration runner + backfill

### Task 6: Go-migration support in the DB runner

**Files:**
- Modify: `internal/db/migrate.go`
- Create: `internal/db/go_migrations.go` (registry)
- Test: `internal/db/migrate_test.go`

**Context:** Extend the runner so Go migrations share the SQL ledger and per-migration transaction. Each version is *either* a SQL file or a Go migration. Existing SQL migrations are unaffected. Keep a registry the real backfill registers into, plus a test seam so tests can inject a synthetic Go migration without touching the real registry.

- [ ] **Step 1: Refactor `migrate.go`.** Generalize the internal `migration` type and split discovery from application so a Go-migration list can be merged in:

```go
// goMigration is a registered Go migration. It shares the SQL migrations'
// version ledger and per-migration transaction. A Go migration operates on raw
// SQL via its *sql.Tx and MUST be self-contained, frozen, and DB-only (no
// service calls, no network I/O) so a rebuilt DB always reconciles identically.
type goMigration struct {
	Version int
	Name    string
	Run     func(ctx context.Context, tx *sql.Tx) error
}

// migration is one applyable unit: exactly one of Filename or Run is set.
type migration struct {
	Version  int
	Filename string                                      // SQL migration (empty for Go)
	Name     string                                      // Go migration name (empty for SQL)
	Run      func(ctx context.Context, tx *sql.Tx) error // Go migration body (nil for SQL)
}
```

Rewrite `Migrate` to delegate to a helper that takes the registered Go migrations, so production uses the real registry and tests can pass their own:

```go
func Migrate(db *sql.DB) error {
	return migrateWith(db, registeredGoMigrations())
}

func migrateWith(db *sql.DB, goMigs []goMigration) error {
	ctx := context.Background()
	if err := ensureMigrationsTable(ctx, db); err != nil {
		return fmt.Errorf("ensure migrations table: %w", err)
	}
	migrations, err := collectMigrations(goMigs)
	if err != nil {
		return err
	}
	for _, m := range migrations {
		applied, err := isApplied(ctx, db, m.Version)
		if err != nil {
			return fmt.Errorf("check if migration %d applied: %w", m.Version, err)
		}
		if applied {
			continue
		}
		log.Printf("applying migration %d: %s", m.Version, m.label())
		if err := applyMigration(ctx, db, m); err != nil {
			return fmt.Errorf("apply migration %d: %w", m.Version, err)
		}
	}
	log.Println("migrations complete")
	return nil
}

func (m migration) label() string {
	if m.Filename != "" {
		return m.Filename
	}
	return "go:" + m.Name
}
```

`collectMigrations` reads the embedded SQL files (existing logic), appends the Go migrations, rejects duplicate versions across the two sources, and sorts by version:

```go
func collectMigrations(goMigs []goMigration) ([]migration, error) {
	entries, err := migrationsFS.ReadDir("migrations")
	if err != nil {
		return nil, fmt.Errorf("read migrations dir: %w", err)
	}
	seen := map[int]string{}
	var migrations []migration
	for _, entry := range entries {
		if entry.IsDir() || !strings.HasSuffix(entry.Name(), ".sql") {
			continue
		}
		version, err := parseVersion(entry.Name())
		if err != nil {
			log.Printf("skip malformed migration %s: %v", entry.Name(), err)
			continue
		}
		if prev, dup := seen[version]; dup {
			return nil, fmt.Errorf("duplicate migration version %d (%s and %s)", version, prev, entry.Name())
		}
		seen[version] = entry.Name()
		migrations = append(migrations, migration{Version: version, Filename: entry.Name()})
	}
	for _, gm := range goMigs {
		if prev, dup := seen[gm.Version]; dup {
			return nil, fmt.Errorf("duplicate migration version %d (%s and go:%s)", gm.Version, prev, gm.Name)
		}
		seen[gm.Version] = "go:" + gm.Name
		migrations = append(migrations, migration{Version: gm.Version, Name: gm.Name, Run: gm.Run})
	}
	sort.Slice(migrations, func(i, j int) bool { return migrations[i].Version < migrations[j].Version })
	return migrations, nil
}
```

Update `applyMigration` to branch on SQL vs Go (both record the version in the same tx):

```go
func applyMigration(ctx context.Context, db *sql.DB, m migration) error {
	tx, err := db.BeginTx(ctx, nil)
	if err != nil {
		return fmt.Errorf("begin transaction: %w", err)
	}
	defer func() { _ = tx.Rollback() }()

	if m.Run != nil {
		if err := m.Run(ctx, tx); err != nil {
			return fmt.Errorf("exec go migration: %w", err)
		}
	} else {
		content, err := migrationsFS.ReadFile(filepath.Join("migrations", m.Filename))
		if err != nil {
			return fmt.Errorf("read migration file: %w", err)
		}
		if _, err := tx.ExecContext(ctx, string(content)); err != nil {
			return fmt.Errorf("exec migration: %w", err)
		}
	}
	if _, err := tx.ExecContext(ctx, "INSERT INTO schema_migrations (version) VALUES (?)", m.Version); err != nil {
		return fmt.Errorf("record migration: %w", err)
	}
	return tx.Commit()
}
```

Remove the now-unused old `migration`-building loop from `Migrate` (it moved into `collectMigrations`). Keep `ensureMigrationsTable`, `isApplied`, `parseVersion` unchanged.

- [ ] **Step 2: Create `internal/db/go_migrations.go`** with the registry (empty for now; Task 7 adds 028):

```go
package db

// registeredGoMigrations returns all Go migrations compiled into the binary, in
// no particular order (the runner sorts by version and shares the SQL ledger).
// Each future data backfill appends its migration here so it versions and ships
// in git alongside the schema change that necessitated it.
func registeredGoMigrations() []goMigration {
	return []goMigration{
		// 028 appended in the backfill task.
	}
}
```

- [ ] **Step 3: Write failing tests** in `migrate_test.go`. Use `migrateWith` with a synthetic Go migration so the real registry stays untouched:

```go
func TestGoMigration_AppliesOnceAndRecordsVersion(t *testing.T) {
	path := filepath.Join(t.TempDir(), "test.db")
	conn, err := sql.Open("sqlite3", path+"?_foreign_keys=on&_journal_mode=WAL")
	if err != nil {
		t.Fatalf("open: %v", err)
	}
	t.Cleanup(func() { _ = conn.Close() })

	var runs int
	gm := goMigration{
		Version: 9001,
		Name:    "test_marker",
		Run: func(ctx context.Context, tx *sql.Tx) error {
			runs++
			_, err := tx.ExecContext(ctx, `CREATE TABLE go_migration_marker (n INTEGER)`)
			return err
		},
	}
	if err := migrateWith(conn, []goMigration{gm}); err != nil {
		t.Fatalf("first migrate: %v", err)
	}
	if runs != 1 {
		t.Fatalf("runs after first = %d, want 1", runs)
	}
	var version int
	if err := conn.QueryRow(`SELECT version FROM schema_migrations WHERE version = 9001`).Scan(&version); err != nil {
		t.Fatalf("version not recorded: %v", err)
	}
	// Re-run: skipped.
	if err := migrateWith(conn, []goMigration{gm}); err != nil {
		t.Fatalf("second migrate: %v", err)
	}
	if runs != 1 {
		t.Fatalf("runs after second = %d, want 1 (should be skipped)", runs)
	}
}

func TestGoMigration_RunsInVersionOrderWithSQL(t *testing.T) {
	path := filepath.Join(t.TempDir(), "test.db")
	conn, err := sql.Open("sqlite3", path+"?_foreign_keys=on&_journal_mode=WAL")
	if err != nil {
		t.Fatalf("open: %v", err)
	}
	t.Cleanup(func() { _ = conn.Close() })

	// A Go migration at a version BELOW the real SQL migrations would run before
	// the tables it expects exist; instead assert ordering via a recorded list.
	var order []int
	mk := func(v int) goMigration {
		return goMigration{Version: v, Name: "ord", Run: func(ctx context.Context, tx *sql.Tx) error { order = append(order, v); return nil }}
	}
	if err := migrateWith(conn, []goMigration{mk(9003), mk(9002)}); err != nil {
		t.Fatalf("migrate: %v", err)
	}
	if len(order) != 2 || order[0] != 9002 || order[1] != 9003 {
		t.Fatalf("order = %v, want [9002 9003]", order)
	}
}

func TestCollectMigrations_RejectsDuplicateVersion(t *testing.T) {
	// 027 exists as a SQL migration; a Go migration claiming 27 must error.
	if _, err := collectMigrations([]goMigration{{Version: 27, Name: "dup", Run: func(ctx context.Context, tx *sql.Tx) error { return nil }}}); err == nil {
		t.Fatal("expected duplicate-version error, got nil")
	}
}
```

Add any missing imports (`context`, `database/sql`, `path/filepath`).

- [ ] **Step 4: Run tests + full package + build:**

Run: `cd /workspace/prog-strength-api && go test ./internal/db/ -v && go build ./...`
Expected: PASS (including all pre-existing migration tests), clean.

- [ ] **Step 5: Commit**

```bash
cd /workspace/prog-strength-api
git add internal/db/migrate.go internal/db/go_migrations.go internal/db/migrate_test.go
git commit -m "feat(db): support Go migrations sharing the SQL version ledger"
```

---

### Task 7: Backfill migration 028 — reconcile existing history

**Files:**
- Create: `internal/db/migration_028_backfill_planned_workout_links.go`
- Modify: `internal/db/go_migrations.go` (register 028)
- Test: `internal/db/migration_028_backfill_test.go`

**Context:** A frozen, DB-only Go migration that replays the matcher over already-logged sessions oldest-first. It carries its **own copy** of the same-day + kind + nearest-start rule (no service calls) so a rebuilt DB always reconciles identically. Idempotent (already-linked sessions skipped; only `planned` plans are candidates); a no-op on a fresh DB.

- [ ] **Step 1: Create `migration_028_backfill_planned_workout_links.go`.** Self-contained: raw SQL over the `*sql.Tx`, frozen copy of the rule.

```go
package db

import (
	"context"
	"database/sql"
	"sort"
	"time"
)

// migration028 backfills planned_workout completion links by replaying a frozen
// copy of the live matcher (same-day + kind + nearest scheduled start, timezone
// bucketed via time.LoadLocation) over already-logged running activities and
// workouts, oldest-first. DB-only and frozen: it never calls service code, so a
// rebuilt DB always reconciles identically regardless of future matcher changes.
func migration028() goMigration {
	return goMigration{
		Version: 28,
		Name:    "backfill_planned_workout_links",
		Run:     backfillPlannedWorkoutLinks,
	}
}

// bfSession is one already-logged session considered for backfill.
type bfSession struct {
	id        string
	userID    string
	startUTC  time.Time
	kind      string // "activity" (running) or "workout" (lift)
	createdAt time.Time
}

// bfPlan is a frozen snapshot of a candidate planned plan.
type bfPlan struct {
	id           string
	activityKind string // "run" / "lift"
	startUTC     time.Time
	timezone     string
}

func backfillPlannedWorkoutLinks(ctx context.Context, tx *sql.Tx) error {
	sessions, err := bfLoadSessions(ctx, tx)
	if err != nil {
		return err
	}
	// Oldest-first reproduces what would have happened had the feature been live.
	sort.SliceStable(sessions, func(i, j int) bool { return sessions[i].createdAt.Before(sessions[j].createdAt) })

	for _, s := range sessions {
		wantKind := "lift"
		if s.kind == "activity" {
			wantKind = "run"
		}
		plan, err := bfSelectPlan(ctx, tx, s, wantKind)
		if err != nil {
			return err
		}
		if plan == nil {
			continue
		}
		if _, err := tx.ExecContext(ctx, `
			UPDATE planned_workouts
			SET status = 'completed', completed_session_id = ?, completed_session_kind = ?, updated_at = ?
			WHERE id = ? AND status = 'planned' AND deleted_at IS NULL
		`, s.id, s.kind, s.startUTC.UTC(), plan.id); err != nil {
			return err
		}
	}
	return nil
}

// bfLoadSessions selects running activities and lifting workouts that are not
// already some plan's completed session.
func bfLoadSessions(ctx context.Context, tx *sql.Tx) ([]bfSession, error) {
	var out []bfSession

	actRows, err := tx.QueryContext(ctx, `
		SELECT a.id, a.user_id, a.start_time, a.created_at
		FROM activities a
		WHERE a.activity_type = 'running' AND a.deleted_at IS NULL
		  AND NOT EXISTS (
		    SELECT 1 FROM planned_workouts p
		    WHERE p.completed_session_kind = 'activity' AND p.completed_session_id = a.id
		  )
	`)
	if err != nil {
		return nil, err
	}
	for actRows.Next() {
		var s bfSession
		s.kind = "activity"
		if err := actRows.Scan(&s.id, &s.userID, &s.startUTC, &s.createdAt); err != nil {
			actRows.Close()
			return nil, err
		}
		out = append(out, s)
	}
	if err := actRows.Err(); err != nil {
		actRows.Close()
		return nil, err
	}
	actRows.Close()

	woRows, err := tx.QueryContext(ctx, `
		SELECT w.id, w.user_id, w.performed_at, w.created_at
		FROM workouts w
		WHERE w.deleted_at IS NULL
		  AND NOT EXISTS (
		    SELECT 1 FROM planned_workouts p
		    WHERE p.completed_session_kind = 'workout' AND p.completed_session_id = w.id
		  )
	`)
	if err != nil {
		return nil, err
	}
	for woRows.Next() {
		var s bfSession
		s.kind = "workout"
		if err := woRows.Scan(&s.id, &s.userID, &s.startUTC, &s.createdAt); err != nil {
			woRows.Close()
			return nil, err
		}
		out = append(out, s)
	}
	if err := woRows.Err(); err != nil {
		woRows.Close()
		return nil, err
	}
	woRows.Close()

	return out, nil
}

// bfSelectPlan is the FROZEN copy of selectPlan: planned-status plans of the
// matching kind, same local calendar day in the plan's own timezone, nearest
// scheduled start (ties: earliest start, then smallest id).
func bfSelectPlan(ctx context.Context, tx *sql.Tx, s bfSession, wantKind string) (*bfPlan, error) {
	rows, err := tx.QueryContext(ctx, `
		SELECT id, activity_kind, scheduled_start_utc, timezone
		FROM planned_workouts
		WHERE user_id = ? AND status = 'planned' AND activity_kind = ? AND deleted_at IS NULL
	`, s.userID, wantKind)
	if err != nil {
		return nil, err
	}
	defer rows.Close()

	var best *bfPlan
	var bestDelta time.Duration
	for rows.Next() {
		var p bfPlan
		if err := rows.Scan(&p.id, &p.activityKind, &p.startUTC, &p.timezone); err != nil {
			return nil, err
		}
		if !bfSameLocalDay(p, s.startUTC) {
			continue
		}
		delta := p.startUTC.Sub(s.startUTC)
		if delta < 0 {
			delta = -delta
		}
		if best == nil || delta < bestDelta ||
			(delta == bestDelta && (p.startUTC.Before(best.startUTC) ||
				(p.startUTC.Equal(best.startUTC) && p.id < best.id))) {
			b := p
			best = &b
			bestDelta = delta
		}
	}
	if err := rows.Err(); err != nil {
		return nil, err
	}
	return best, nil
}

func bfSameLocalDay(p bfPlan, sessionStartUTC time.Time) bool {
	loc, err := time.LoadLocation(p.timezone)
	if err != nil {
		return false
	}
	ps := p.startUTC.In(loc)
	ss := sessionStartUTC.In(loc)
	py, pm, pd := ps.Date()
	sy, sm, sd := ss.Date()
	return py == sy && pm == sm && pd == sd
}
```

Note on time scanning: go-sqlite3 returns `DATETIME` columns as `time.Time` when scanned into `time.Time` (the existing repos rely on this). If a scan fails because a column comes back as string, fall back to scanning into a string and `time.Parse(time.RFC3339, …)` — but first try `time.Time` directly to match the existing repository code.

- [ ] **Step 2: Register 028** in `go_migrations.go`:

```go
func registeredGoMigrations() []goMigration {
	return []goMigration{
		migration028(),
	}
}
```

- [ ] **Step 3: Write failing test** `migration_028_backfill_test.go`. Build a DB at schema 027 by running the full `Migrate` (which now includes 028 — so instead test the backfill function directly against a `Migrate`d DB, seeding data, then invoking the registered migration in a tx). Cleanest approach: run all migrations on a fresh DB (fresh-DB no-op), then seed plans + sessions and call `backfillPlannedWorkoutLinks` inside a manual tx, asserting links and idempotency.

```go
func TestBackfill028_LinksAndIsIdempotent(t *testing.T) {
	conn := newMigratedDB(t) // full Migrate incl. 028 (no-op on empty DB)

	// Seed a user, a planned run plan today, and a running activity the same local day.
	seedUser(t, conn, "u1")
	// planned run plan at 2026-06-15 17:30 UTC, tz America/New_York
	mustExec(t, conn, `INSERT INTO planned_workouts
		(id, user_id, activity_kind, scheduled_start_utc, scheduled_end_utc, timezone, status, created_at, updated_at)
		VALUES ('plan1','u1','run','2026-06-15T17:30:00Z','2026-06-15T18:30:00Z','America/New_York','planned','2026-06-01T00:00:00Z','2026-06-01T00:00:00Z')`)
	// running activity at 2026-06-15 18:00 UTC
	mustExec(t, conn, `INSERT INTO activities
		(id, user_id, activity_type, ingest_source, source_activity_id, start_time, distance_meters, duration_seconds, tcx_s3_key, created_at)
		VALUES ('act1','u1','running','manual_tcx','src1','2026-06-15T18:00:00Z',5000,1500,'k1','2026-06-15T18:05:00Z')`)

	runBackfill := func() {
		tx, err := conn.Begin()
		if err != nil { t.Fatalf("begin: %v", err) }
		if err := backfillPlannedWorkoutLinks(context.Background(), tx); err != nil { t.Fatalf("backfill: %v", err) }
		if err := tx.Commit(); err != nil { t.Fatalf("commit: %v", err) }
	}
	runBackfill()

	var status, sid, skind string
	if err := conn.QueryRow(`SELECT status, completed_session_id, completed_session_kind FROM planned_workouts WHERE id='plan1'`).Scan(&status, &sid, &skind); err != nil {
		t.Fatalf("read plan: %v", err)
	}
	if status != "completed" || sid != "act1" || skind != "activity" {
		t.Fatalf("plan not linked: status=%s sid=%s skind=%s", status, sid, skind)
	}

	// Idempotent: re-running links nothing new and leaves the plan completed.
	runBackfill()
	var count int
	conn.QueryRow(`SELECT COUNT(*) FROM planned_workouts WHERE id='plan1' AND status='completed' AND completed_session_id='act1'`).Scan(&count)
	if count != 1 {
		t.Fatalf("idempotency broken: count=%d", count)
	}
}

func TestBackfill028_FreshDBNoOp(t *testing.T) {
	conn := newMigratedDB(t)
	tx, _ := conn.Begin()
	if err := backfillPlannedWorkoutLinks(context.Background(), tx); err != nil {
		t.Fatalf("backfill on empty DB: %v", err)
	}
	_ = tx.Commit()
}

func TestBackfill028_WrongKindAndTwoADay(t *testing.T) {
	conn := newMigratedDB(t)
	seedUser(t, conn, "u1")
	// two planned runs same NY day; nearest scheduled start should win.
	mustExec(t, conn, `INSERT INTO planned_workouts (id,user_id,activity_kind,scheduled_start_utc,scheduled_end_utc,timezone,status,created_at,updated_at) VALUES
		('early','u1','run','2026-06-15T15:00:00Z','2026-06-15T16:00:00Z','America/New_York','planned','2026-06-01T00:00:00Z','2026-06-01T00:00:00Z'),
		('late','u1','run','2026-06-15T22:00:00Z','2026-06-15T23:00:00Z','America/New_York','planned','2026-06-01T00:00:00Z','2026-06-01T00:00:00Z')`)
	mustExec(t, conn, `INSERT INTO activities (id,user_id,activity_type,ingest_source,source_activity_id,start_time,distance_meters,duration_seconds,tcx_s3_key,created_at) VALUES
		('act1','u1','running','manual_tcx','src1','2026-06-15T21:30:00Z',5000,1500,'k1','2026-06-15T21:35:00Z')`)
	tx, _ := conn.Begin()
	if err := backfillPlannedWorkoutLinks(context.Background(), tx); err != nil { t.Fatalf("backfill: %v", err) }
	tx.Commit()
	var lateStatus, earlyStatus string
	conn.QueryRow(`SELECT status FROM planned_workouts WHERE id='late'`).Scan(&lateStatus)
	conn.QueryRow(`SELECT status FROM planned_workouts WHERE id='early'`).Scan(&earlyStatus)
	if lateStatus != "completed" || earlyStatus != "planned" {
		t.Fatalf("nearest-start wrong: late=%s early=%s", lateStatus, earlyStatus)
	}
}
```

Add `mustExec` and reuse `seedUser`/`newMigratedDB` from the existing test file (check `seedUser`'s exact column set; adapt the INSERT to the real `users` schema). If `seedUser` already inserts a fixed id, use that id consistently.

- [ ] **Step 4: Run + build:**

Run: `cd /workspace/prog-strength-api && go test ./internal/db/ -v && go build ./...`
Expected: PASS, clean.

- [ ] **Step 5: Commit**

```bash
cd /workspace/prog-strength-api
git add internal/db/migration_028_backfill_planned_workout_links.go internal/db/go_migrations.go internal/db/migration_028_backfill_test.go
git commit -m "feat(db): backfill planned-workout links (migration 028)"
```

---

## Phase 4 — Web

### Task 8: API client functions — `unlinkPlannedWorkout` + `getPlannedWorkoutBySession`

**Files:**
- Modify: `lib/api.ts`
- Test: `lib/api.test.ts`

**Context:** Mirror the existing `resyncPlannedWorkout`/`completePlannedWorkout` patterns (plain `fetch`, bearer header, `unwrap`). The by-session lookup returns `null` on 404 (not an error) because "no plan completed by this session" is the common case.

- [ ] **Step 1: Write failing tests** in `lib/api.test.ts` (reuse the file's `mockFetchOk`/`BASE`/`TOKEN` helpers; add a `mockFetch404` if not present):

```ts
it("unlinkPlannedWorkout POSTs to /unlink and returns the updated plan", async () => {
  const plan = { id: "p1", status: "planned", completed_session_id: null } as unknown;
  const fetchMock = mockFetchOk(plan);
  const result = await unlinkPlannedWorkout(TOKEN, "p1");
  expect(result).toMatchObject({ id: "p1", status: "planned" });
  expect(fetchMock).toHaveBeenCalledWith(`${BASE}/planned-workouts/p1/unlink`, {
    method: "POST",
    headers: { "Content-Type": "application/json", Authorization: `Bearer ${TOKEN}` },
  });
});

it("getPlannedWorkoutBySession returns the plan when found", async () => {
  const plan = { id: "p1", status: "completed", completed_session_id: "act1" } as unknown;
  const fetchMock = mockFetchOk(plan);
  const result = await getPlannedWorkoutBySession(TOKEN, "act1", "activity");
  expect(result).toMatchObject({ id: "p1" });
  expect(fetchMock).toHaveBeenCalledWith(
    `${BASE}/planned-workouts/by-session?session_id=act1&session_kind=activity`,
    { headers: { Authorization: `Bearer ${TOKEN}` } },
  );
});

it("getPlannedWorkoutBySession returns null on 404", async () => {
  vi.stubGlobal("fetch", vi.fn().mockResolvedValue({ ok: false, status: 404, json: async () => ({ error: "no planned workout completed by this session" }) }));
  const result = await getPlannedWorkoutBySession(TOKEN, "act1", "activity");
  expect(result).toBeNull();
});
```

- [ ] **Step 2: Run, verify fail:** `cd /workspace/prog-strength-web && npx vitest run lib/api.test.ts` → FAIL (undefined functions).

- [ ] **Step 3: Implement** in `lib/api.ts`, next to `resyncPlannedWorkout`:

```ts
export async function unlinkPlannedWorkout(token: string, id: string): Promise<PlannedWorkout> {
  const resp = await fetch(`${config.apiUrl}/planned-workouts/${encodeURIComponent(id)}/unlink`, {
    method: "POST",
    headers: { "Content-Type": "application/json", Authorization: `Bearer ${token}` },
  });
  const updated = await unwrap<PlannedWorkout | null>(resp, null);
  if (!updated) throw new Error("API did not return the unlinked planned workout");
  return updated;
}

// getPlannedWorkoutBySession returns the planned workout a logged session
// completes, or null when none does (a 404 from the API is the common case).
export async function getPlannedWorkoutBySession(
  token: string,
  sessionId: string,
  sessionKind: CompletedSessionKind,
): Promise<PlannedWorkout | null> {
  const params = new URLSearchParams({ session_id: sessionId, session_kind: sessionKind });
  const resp = await fetch(`${config.apiUrl}/planned-workouts/by-session?${params.toString()}`, {
    headers: { Authorization: `Bearer ${token}` },
  });
  if (resp.status === 404) return null;
  const plan = await unwrap<PlannedWorkout | null>(resp, null);
  return plan;
}
```

Verify the actual query-string format the test asserts matches `URLSearchParams` output (`session_id=act1&session_kind=activity`). If `unwrap` throws on non-OK before you can check 404, special-case `resp.status === 404` first (as above).

- [ ] **Step 4: Run tests:** `cd /workspace/prog-strength-web && npx vitest run lib/api.test.ts` → PASS.

- [ ] **Step 5: Commit**

```bash
cd /workspace/prog-strength-web
git add lib/api.ts lib/api.test.ts
git commit -m "feat(api): add unlinkPlannedWorkout and getPlannedWorkoutBySession"
```

---

### Task 9: Reverse "Completes your planned …" + Unlink on run/workout detail pages

**Files:**
- Create: `components/completes-plan-banner.tsx`
- Modify: `app/(app)/running/[id]/page.tsx`
- Modify: `app/(app)/workouts/[id]/page.tsx`
- Test: `components/completes-plan-banner.test.tsx`

**Context:** On a run/workout detail page, look up the plan this session completes; if found, render "✓ Completes your planned {name}" with an Unlink button that calls `unlinkPlannedWorkout` and dismisses the banner. The banner is the natural place to undo a bad match.

- [ ] **Step 1: Create the shared `components/completes-plan-banner.tsx`:**

```tsx
"use client";

import { useState } from "react";
import type { PlannedWorkout } from "@/lib/api";
import { unlinkPlannedWorkout } from "@/lib/api";
import { getToken } from "@/lib/auth";

export function CompletesPlanBanner({
  plan,
  onUnlinked,
}: {
  plan: PlannedWorkout;
  onUnlinked?: () => void;
}) {
  const [unlinking, setUnlinking] = useState(false);
  const [dismissed, setDismissed] = useState(false);
  const [error, setError] = useState<string | null>(null);

  if (dismissed) return null;

  const label = plan.name?.trim()
    ? `your planned ${plan.name.trim()}`
    : plan.activity_kind === "run"
      ? "your planned run"
      : "your planned workout";

  const handleUnlink = async () => {
    const token = getToken();
    if (!token) return;
    setUnlinking(true);
    setError(null);
    try {
      await unlinkPlannedWorkout(token, plan.id);
      setDismissed(true);
      onUnlinked?.();
    } catch (e) {
      setError(e instanceof Error ? e.message : "Unlink failed");
      setUnlinking(false);
    }
  };

  return (
    <div className="flex items-center justify-between gap-3 rounded-lg border border-[var(--border)] bg-[var(--surface)] px-3 py-2 text-sm">
      <span className="text-[var(--foreground)]">
        ✓ Completes {label}
      </span>
      <div className="flex items-center gap-2">
        {error && <span className="text-xs text-[var(--danger)]">{error}</span>}
        <button
          type="button"
          onClick={handleUnlink}
          disabled={unlinking}
          className="text-xs font-medium text-[var(--muted)] transition hover:text-[var(--danger)] disabled:opacity-50"
        >
          {unlinking ? "Unlinking…" : "Unlink"}
        </button>
      </div>
    </div>
  );
}
```

Match the project's CSS variable names / utility conventions by glancing at `planned-banner.tsx` (e.g. `var(--accent)`, `var(--danger)`, `var(--muted)`); adjust class names to the codebase's actual tokens.

- [ ] **Step 2: Write failing component test** `components/completes-plan-banner.test.tsx`:

```tsx
/// <reference types="vitest/globals" />
import { render, screen, fireEvent, waitFor } from "@testing-library/react";
import { CompletesPlanBanner } from "./completes-plan-banner";

const unlinkMock = vi.hoisted(() => vi.fn());
vi.mock("@/lib/auth", () => ({ getToken: () => "test-token" }));
vi.mock("@/lib/api", async (orig) => ({
  ...(await orig<typeof import("@/lib/api")>()),
  unlinkPlannedWorkout: unlinkMock,
}));

const PLAN = { id: "p1", name: "Easy Run", activity_kind: "run", status: "completed" } as never;

describe("CompletesPlanBanner", () => {
  it("renders the plan name and unlinks on click", async () => {
    unlinkMock.mockResolvedValue({ id: "p1", status: "planned" });
    const onUnlinked = vi.fn();
    render(<CompletesPlanBanner plan={PLAN} onUnlinked={onUnlinked} />);
    expect(screen.getByText(/Completes your planned Easy Run/)).toBeInTheDocument();
    fireEvent.click(screen.getByRole("button", { name: "Unlink" }));
    await waitFor(() => expect(unlinkMock).toHaveBeenCalledWith("test-token", "p1"));
    await waitFor(() => expect(onUnlinked).toHaveBeenCalled());
    expect(screen.queryByText(/Completes your planned Easy Run/)).not.toBeInTheDocument();
  });
});
```

- [ ] **Step 3: Run, verify fail:** `cd /workspace/prog-strength-web && npx vitest run components/completes-plan-banner.test.tsx` → FAIL.

- [ ] **Step 4: Implement, verify pass:** Create the component (Step 1), run the test → PASS.

- [ ] **Step 5: Wire into the run detail page** `app/(app)/running/[id]/page.tsx`. Add state `const [completesPlan, setCompletesPlan] = useState<PlannedWorkout | null>(null);`. In the effect that loads the run (after the session is fetched), also call `getPlannedWorkoutBySession(token, id, "activity").then(setCompletesPlan).catch(() => {})` (best-effort; ignore errors). Render `{completesPlan && <CompletesPlanBanner plan={completesPlan} onUnlinked={() => setCompletesPlan(null)} />}` near the top of the body (under the header). Import the component, `getPlannedWorkoutBySession`, and the `PlannedWorkout` type.

- [ ] **Step 6: Wire into the workout detail page** `app/(app)/workouts/[id]/page.tsx` identically, with kind `"workout"`: `getPlannedWorkoutBySession(token, id, "workout")`.

- [ ] **Step 7: Run web tests + typecheck + lint:**

Run: `cd /workspace/prog-strength-web && npx vitest run && npx tsc --noEmit && npm run lint`
Expected: PASS / no type errors / lint clean (fix any issues).

- [ ] **Step 8: Commit**

```bash
cd /workspace/prog-strength-web
git add components/completes-plan-banner.tsx components/completes-plan-banner.test.tsx "app/(app)/running/[id]/page.tsx" "app/(app)/workouts/[id]/page.tsx"
git commit -m "feat(web): show 'Completes your planned …' + Unlink on run/workout detail"
```

---

### Task 10: Unlink affordance on the planned banner / planned-workout detail

**Files:**
- Modify: `components/calendar/planned-banner.tsx`
- Modify: `app/(app)/planned-workouts/[id]/page.tsx`
- Test: `components/calendar/planned-banner.test.tsx` (create if absent)

**Context:** The planned-workout detail page and the calendar `PlannedBanner` already render a completed plan with a "View logged run/workout →" link. Add an **Unlink** affordance next to it. Only the detail page (which owns the plan state) wires the actual unlink; the calendar passes no `onUnlink` and the affordance simply doesn't render there.

- [ ] **Step 1: Extend `PlannedBanner`** to accept an optional `onUnlink?: () => void` prop. In the completed-link section (the existing `{completed && planned.completed_session_id && ( … )}` block ~145-163), after the "View logged … →" button, render:

```tsx
        {onUnlink && (
          <button
            type="button"
            onClick={onUnlink}
            className="text-xs font-medium text-[var(--muted)] transition hover:text-[var(--danger)]"
          >
            Unlink
          </button>
        )}
```

Add `onUnlink` to the component's props type/destructure.

- [ ] **Step 2: Wire the detail page** `app/(app)/planned-workouts/[id]/page.tsx`. Add a handler mirroring the existing `handleResync`:

```tsx
  const handleUnlink = async () => {
    if (!plan) return;
    const token = getToken();
    if (!token) {
      router.replace("/login");
      return;
    }
    try {
      setPlan(await unlinkPlannedWorkout(token, plan.id));
    } catch (e) {
      setError(e instanceof Error ? e.message : "Unlink failed");
    }
  };
```

Import `unlinkPlannedWorkout`. Pass `onUnlink={handleUnlink}` to the `<PlannedBanner … />` render.

- [ ] **Step 3: Test.** Add/extend `components/calendar/planned-banner.test.tsx`: render a completed plan WITH `onUnlink` → an "Unlink" button shows and clicking calls the handler; render WITHOUT `onUnlink` → no "Unlink" button. (Mock nothing beyond what's needed; `onUnlink` is a plain `vi.fn()`.)

```tsx
/// <reference types="vitest/globals" />
import { render, screen, fireEvent } from "@testing-library/react";
import { PlannedBanner } from "./planned-banner";

const COMPLETED = {
  id: "p1", name: "Easy Run", activity_kind: "run", status: "completed",
  completed_session_id: "act1", completed_session_kind: "activity",
  scheduled_start: "2026-06-15T17:30:00Z", scheduled_end: "2026-06-15T18:30:00Z",
  timezone: "America/New_York", exercises: [],
} as never;

describe("PlannedBanner unlink", () => {
  it("shows Unlink when onUnlink provided and fires it", () => {
    const onUnlink = vi.fn();
    render(<PlannedBanner planned={COMPLETED} exerciseMap={new Map()} defaultOpen onUnlink={onUnlink} />);
    fireEvent.click(screen.getByRole("button", { name: "Unlink" }));
    expect(onUnlink).toHaveBeenCalled();
  });
  it("hides Unlink when onUnlink absent", () => {
    render(<PlannedBanner planned={COMPLETED} exerciseMap={new Map()} defaultOpen />);
    expect(screen.queryByRole("button", { name: "Unlink" })).not.toBeInTheDocument();
  });
});
```

Adapt the `COMPLETED` fixture to the real `PlannedWorkout` shape/required props the component reads (check `planned-banner.tsx`).

- [ ] **Step 4: Run web tests + typecheck + lint:**

Run: `cd /workspace/prog-strength-web && npx vitest run && npx tsc --noEmit && npm run lint`
Expected: PASS / clean.

- [ ] **Step 5: Commit**

```bash
cd /workspace/prog-strength-web
git add "components/calendar/planned-banner.tsx" "components/calendar/planned-banner.test.tsx" "app/(app)/planned-workouts/[id]/page.tsx"
git commit -m "feat(web): add Unlink affordance to the planned banner"
```

---

## Final verification (before opening PRs)

- [ ] **API:** `cd /workspace/prog-strength-api && go build ./... && go vet ./... && go test ./...` — all green.
- [ ] **Web:** `cd /workspace/prog-strength-web && npx vitest run && npx tsc --noEmit && npm run lint && npm run build` — all green.

## Spec coverage self-check

- Auto-link on log → Tasks 3,4,5. Robust matching (generic names, off-schedule) → Task 3 (time+kind only). Unlink → Tasks 2 (API), 9/10 (web). Revert on delete → Tasks 3,4. Reverse display → Tasks 2 (by-session), 9. Backfill in git → Tasks 6,7. Go-migration runner → Task 6. Consumer-defined port (no import cycle) → Task 4. One link path (LinkCompletion shared) → Task 2. Frozen backfill rule → Task 7. Edge cases (two-a-day, already-completed, wrong-kind, tz drift) → Tasks 3,7 tests.
