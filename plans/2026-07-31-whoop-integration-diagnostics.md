# WHOOP Integration Diagnostics, Resync & Liveness Alerting — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make a silent WHOOP ingestion failure observable and recoverable — admin endpoints over connection state + an idempotent resync (`prog-strength-api`), a `pst whoop doctor`/`resync` chain diagnostic (`prog-strength-tooling`), and two provisioned alert rules plus a dashboard connection-health panel (`prog-strength-infra`).

**Architecture:** Instrument and alert on the *absence of success*, not only the presence of failure. The API grows a new `internal/whoopadmin` package (three admin-gated read/write routes) mounted inside the existing `RequireAdmin` group, plus a router-level `api_webhook_misroute_total` counter (chi `NotFound` handler) and an `api_whoop_connections{status}` gauge exporter. The CLI joins those admin endpoints with CloudWatch log evidence to identify which link in the chain broke. Infra adds `rules-whoop.yml` (Rule A dead-ingestion backstop, Rule B misroute) and a connection-health dashboard panel.

**Tech Stack:** Go 1.25 (chi, database/sql + go-sqlite3, prometheus/client_golang, stdlib tests, golangci-lint v2.12.2), Python 3.12 (Typer, httpx, boto3, rich, pytest + respx, black 100col, ruff), Grafana provisioning YAML + dashboard JSON.

**Cross-cutting conventions (every task):**
- Read the sibling code named in each task before writing. Match its patterns exactly (naming, error wrapping, response envelope, test shape). Do not introduce new dependencies.
- TDD: write the failing test first, watch it fail, implement, watch it pass, commit.
- Conventional-commit messages, lowercase subject. Co-author footer on every commit:
  `Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>`
- **prog-strength-api gate (run before declaring a task done):**
  `GOTOOLCHAIN=auto` is set — the repo's go.mod pins go 1.25.12 and the toolchain auto-downloads. `CGO_ENABLED=1` (go-sqlite3 + sqlite-vec are cgo).
  ```
  cd /workspace/prog-strength-api
  go build ./...
  go vet ./...
  go run github.com/golangci/golangci-lint/v2/cmd/golangci-lint@v2.12.2 run   # (golangci-lint v2.12.2 is on PATH; either works)
  go mod tidy && git diff --exit-code go.mod go.sum
  go test ./...
  ```
  gosec is active: parameterized SQL only, no silencing. Never add `//nolint`, never `--no-verify`.
- **prog-strength-tooling gate:** `export PATH="$HOME/.local/bin:$PATH"` then
  ```
  cd /workspace/prog-strength-tooling
  uv run black --check .
  uv run ruff check .
  uv run pytest -q
  ```
- **prog-strength-infra gate:** no terraform files are touched, so terraform/tflint gates do not apply. Validate the YAML and JSON parse:
  ```
  python3 -c "import yaml; yaml.safe_load(open('monitoring/grafana/provisioning/alerting/rules-whoop.yml'))"
  python3 -c "import json; json.load(open('monitoring/grafana/dashboards/whoop.json'))"
  ```

---

## Reconciliations with the SOW (read before starting — these override the SOW's literal wording where the repo differs)

1. **No in-memory repositories exist.** The SOW's Data Model says "Both interfaces have in-memory and SQLite implementations." They do not — `whoopconn` and `whooprecovery` each ship a single `SQLiteRepository` (per AGENTS.md: "single SQLite implementation per domain"). The interface doc-comments mention in-memory aspirationally; ignore that. Add the new methods to the SQLite implementations and exercise them through the existing test suites (`whoopconn/repository_shared_test.go`'s `runRepositoryContract`, `whooprecovery/repository_test.go`), which run against an ephemeral SQLite DB via `internal/db/dbtest`. **Do not create in-memory implementations.**

2. **`SyncSince` must surface upsert/skip counts.** The SOW types it `SyncSince(...) error` yet the `POST /admin/whoop/resync` response body returns `{upserted, skipped_unscored, skipped_no_cycle, skipped_bad_date}`. The observable endpoint contract wins: `syncWindow` already computes those four counts locally (`service.go:141,189-192`) but discards them. Task A3 introduces a `SyncResult` value, changes the unexported `syncWindow` to return `(SyncResult, error)`, and has `SyncSince` return `(SyncResult, error)`. The existing `SyncWindow`/`Backfill` wrappers keep their `error`-only signatures (they discard the result), so no existing caller changes.

3. **`502 WHOOP API failure` needs a classifiable error.** To map upstream WHOOP fetch failures to `502`, Task A3 adds an exported sentinel `ErrUpstream` and wraps the `Recoveries`/`Cycles` fetch errors with it (`%w` chain, backward-compatible). The handler maps `errors.Is(err, whoopsync.ErrUpstream)` → `502`.

4. **Dashboard thresholds are already partly present.** `whoop.json` panel 2 ("Successful syncs (24h)") already has red-below-1; panel 3 ("Recovery rows upserted (24h)") currently has *yellow*-below-1. Task I2 changes panel 3's below-1 step to **red** (per the SOW's "Add red-below-1 thresholds to … both") and adds the missing connection-health panel. Panel 2 is left as-is (already correct).

5. **Gauge exporter has no clock dependency.** The SOW mentions "a fake clock" for the exporter test; the exporter only reads `whoopconn.List` and sets a gauge per status — no time logic — so its unit test drives a `refresh(ctx)` method directly with a fake repository. The 5-minute cadence is a `time.Ticker` in `Run(ctx)`, stopped on context cancellation.

---

## File Structure

### prog-strength-api
- Modify: `internal/whooprecovery/repository.go` — add `Latest`, `CountForUser` to the interface.
- Modify: `internal/whooprecovery/sqlite_repository.go` — implement both.
- Modify: `internal/whooprecovery/repository_test.go` — cases for both (incl. `Latest` no-rows → `(nil, nil)`).
- Modify: `internal/whoopconn/repository.go` — add `List` to the interface.
- Modify: `internal/whoopconn/sqlite_repository.go` — implement `List`.
- Modify: `internal/whoopconn/repository_shared_test.go` — a `List` case in `runRepositoryContract`.
- Modify: `internal/whoopsync/service.go` — `SyncResult`, `syncWindow` returns `(SyncResult, error)`, `ErrUpstream`, `SyncSince`.
- Modify: `internal/whoopsync/service_test.go` (or add) — `SyncSince` label + counts + upstream-error tests.
- Create: `internal/whoopadmin/handler.go` — three admin routes + narrow consumer interfaces + DTOs.
- Create: `internal/whoopadmin/handler_test.go`.
- Create: `internal/whoopadmin/connections_gauge.go` — `api_whoop_connections` gauge + exporter.
- Create: `internal/whoopadmin/connections_gauge_test.go`.
- Modify: `internal/server/metrics.go` — `api_webhook_misroute_total` counter + `providerFragments` + `webhookMisrouteNotFound` handler.
- Create: `internal/server/metrics_misroute_test.go` — counter behaviour.
- Modify: `internal/server/server.go` — register `r.NotFound`, construct + mount `whoopadmin.Handler`, start the gauge exporter, cancel it on shutdown.

### prog-strength-tooling
- Modify: `src/prog_strength_tooling/cloudwatch.py` — rename `_build_client`→`build_client`, `_describe_failure`→`describe_failure` (module-level, shared).
- Modify: `tests/test_cloudwatch.py` — update references.
- Create: `src/prog_strength_tooling/whooplogs.py` — CloudWatch delivery scan + sync scan.
- Create: `tests/test_whooplogs.py`.
- Modify: `src/prog_strength_tooling/client.py` — `WhoopAdminClient` (connections list/get, resync).
- Modify: `src/prog_strength_tooling/models.py` — WHOOP admin DTOs.
- Create: `tests/test_whoop_client.py`.
- Create: `src/prog_strength_tooling/whoop.py` — diagnosis engine (`Diagnosis`, 7 checks, degraded path).
- Create: `tests/test_whoop_diagnosis.py`.
- Modify: `src/prog_strength_tooling/render.py` — `render_diagnosis`, `render_resync`.
- Create: `src/prog_strength_tooling/commands/whoop.py` — Typer sub-app (`doctor`, `resync`), exit codes.
- Modify: `src/prog_strength_tooling/cli.py` — mount `whoop.app`.
- Create: `tests/test_whoop_cli.py`.

### prog-strength-infra
- Create: `monitoring/grafana/provisioning/alerting/rules-whoop.yml` — Rule A + Rule B.
- Modify: `monitoring/grafana/dashboards/whoop.json` — panel 3 threshold red; add connection-health panel.

### prog-strength-docs
- Modify: `sows/whoop-integration-diagnostics.md` — status flip (handled by the controller in the workflow's step 4, not a subagent task).

---

## Task A1 (API): `whooprecovery.Latest` + `CountForUser`

**Files:**
- Modify: `internal/whooprecovery/repository.go`
- Modify: `internal/whooprecovery/sqlite_repository.go`
- Test: `internal/whooprecovery/repository_test.go`

Read `internal/whooprecovery/{repository.go,recovery.go,sqlite_repository.go,repository_test.go}` first. The recovery table columns are: `id, user_id, date, recovery_score, resting_heart_rate, hrv_rmssd_milli, cycle_id, sleep_id, created_at, updated_at`; unique `(user_id, date)`; `date` is a `YYYY-MM-DD` string. `Entry` fields are `ID, UserID, Date, RecoveryScore *float64, RestingHeartRate *float64, HRVRmssdMilli *float64, CycleID int64, SleepID string, CreatedAt, UpdatedAt`. The existing `ListRange` shows the exact scan/row-mapping pattern to copy (nullable float handling, row → `Entry`).

- [ ] **Step 1: Add to the `Repository` interface** (`repository.go`), after `ListRange`:

```go
	// Latest returns the newest row for the user by date (date DESC LIMIT 1),
	// or (nil, nil) when the user has no rows — absence is an expected state
	// for a freshly connected account, not an error.
	Latest(ctx context.Context, userID string) (*Entry, error)

	// CountForUser returns the number of recovery rows stored for the user.
	CountForUser(ctx context.Context, userID string) (int, error)
```

- [ ] **Step 2: Write failing tests** in `repository_test.go`. Mirror the existing `TestListRange_WindowOrderingAndIsolation` setup (`repo := NewSQLiteRepository(dbtest.New(t))`, upsert a few rows across dates and users). Assert:
  - `Latest` returns the row with the newest `date` for the user (not another user's), with its fields populated.
  - `Latest` returns `(nil, nil)` for a user with no rows.
  - `CountForUser` returns the exact count for the user and `0` for an unknown user, and is user-scoped (does not count another user's rows).

- [ ] **Step 3: Run tests, verify they fail** — `go test ./internal/whooprecovery/ -run 'Latest|CountForUser' -v` → FAIL (methods undefined).

- [ ] **Step 4: Implement in `sqlite_repository.go`.** Copy `ListRange`'s scan pattern for `Latest`:

```go
func (r *SQLiteRepository) Latest(ctx context.Context, userID string) (*Entry, error) {
	const q = `SELECT id, user_id, date, recovery_score, resting_heart_rate, hrv_rmssd_milli, cycle_id, sleep_id, created_at, updated_at
FROM user_whoop_recovery WHERE user_id = ? ORDER BY date DESC LIMIT 1`
	row := r.db.QueryRowContext(ctx, q, userID)
	// scan into an Entry using the same nullable-float handling ListRange uses;
	// on sql.ErrNoRows return (nil, nil).
	// ... (mirror ListRange's per-row mapping) ...
}

func (r *SQLiteRepository) CountForUser(ctx context.Context, userID string) (int, error) {
	const q = `SELECT COUNT(*) FROM user_whoop_recovery WHERE user_id = ?`
	var n int
	if err := r.db.QueryRowContext(ctx, q, userID).Scan(&n); err != nil {
		return 0, fmt.Errorf("whooprecovery: count for user: %w", err)
	}
	return n, nil
}
```
Use `errors.Is(err, sql.ErrNoRows)` in `Latest` to return `(nil, nil)`. Match the exact column-scan order and nullable handling from `ListRange` in the same file — do not invent a new mapping.

- [ ] **Step 5: Run tests, verify they pass** — `go test ./internal/whooprecovery/ -v`.

- [ ] **Step 6: Full API gate + commit.** Run the API gate. Commit:
```
feat(whooprecovery): add Latest and CountForUser reads
```

---

## Task A2 (API): `whoopconn.List`

**Files:**
- Modify: `internal/whoopconn/repository.go`
- Modify: `internal/whoopconn/sqlite_repository.go`
- Test: `internal/whoopconn/repository_shared_test.go`

Read `internal/whoopconn/{repository.go,connection.go,sqlite_repository.go,repository_shared_test.go}` first. The connections table columns: `user_id, whoop_user_id, access_token_enc, access_token_nonce, refresh_token_enc, refresh_token_nonce, token_expires_at, scopes, status, connected_at, updated_at`. `Connection` (metadata, no token material) is `{UserID, WhoopUserID int64, Scopes, Status, TokenExpiresAt, ConnectedAt, UpdatedAt}`. `Get` in the same file shows the exact metadata SELECT + scan (copy its column list and time handling — do NOT select token columns).

- [ ] **Step 1: Add to the `Repository` interface** (`repository.go`), after `Get`:
```go
	// List returns all connections (any status), ordered by updated_at DESC.
	// Metadata only — never token material, consistent with Get.
	List(ctx context.Context) ([]Connection, error)
```

- [ ] **Step 2: Write a failing case** inside `runRepositoryContract` in `repository_shared_test.go` (a new `t.Run("List", ...)` block). Upsert two connections for different users with different `updated_at` (pass distinct `now` values to `Upsert`); flip one to a non-connected status via `SetStatus`. Assert `List` returns both, ordered by `updated_at DESC`, carries the right metadata, and (paranoia) that token material is not exposed (there is nowhere to leak it — `Connection` has no token fields — but assert the statuses/order). Also assert an empty DB returns `len == 0` with no error.

- [ ] **Step 3: Run, verify fail** — `go test ./internal/whoopconn/ -run Contract -v` → FAIL (List undefined).

- [ ] **Step 4: Implement `List`** in `sqlite_repository.go`, copying `Get`'s metadata column list and scan:
```go
func (r *SQLiteRepository) List(ctx context.Context) ([]Connection, error) {
	const q = `SELECT user_id, whoop_user_id, scopes, status, token_expires_at, connected_at, updated_at
FROM user_whoop_connection ORDER BY updated_at DESC`
	rows, err := r.db.QueryContext(ctx, q)
	if err != nil {
		return nil, fmt.Errorf("whoopconn: list: %w", err)
	}
	defer rows.Close()
	var out []Connection
	for rows.Next() {
		// scan one Connection using the same column mapping/time parsing as Get
	}
	if err := rows.Err(); err != nil {
		return nil, fmt.Errorf("whoopconn: list rows: %w", err)
	}
	return out, nil
}
```
Match `Get`'s exact time-column handling (the same file parses `token_expires_at`/`connected_at`/`updated_at`). Ensure `rows.Close()` and `rows.Err()` are handled (gosec/lint).

- [ ] **Step 5: Run, verify pass** — `go test ./internal/whoopconn/ -v`.

- [ ] **Step 6: API gate + commit.**
```
feat(whoopconn): add List for the admin surface and gauge exporter
```

---

## Task A3 (API): `whoopsync.SyncSince` + surfaced counts + upstream sentinel

**Files:**
- Modify: `internal/whoopsync/service.go`
- Test: `internal/whoopsync/service_test.go` (add cases; read it first to reuse its fakes)

Read `internal/whoopsync/service.go` (esp. `syncWindow` lines 113-211, `SyncWindow`/`Backfill` 98-111, `ErrReconnectNeeded` 17-21) and `internal/whoopsync/service_test.go` to reuse the existing fake `whoopAPI`/`tokenRefresher`/repos and fake-clock (`now func() time.Time`) test harness.

- [ ] **Step 1: Add the result type + upstream sentinel** near the top of `service.go`:
```go
// ErrUpstream wraps a failure talking to the WHOOP API (fetch of recoveries or
// cycles). Callers that surface HTTP status can map it to 502 rather than 500.
var ErrUpstream = errors.New("whoopsync: whoop api error")

// SyncResult reports what a sync did with the recoveries it fetched. The four
// counts sum to len(recoveries fetched). Returned by SyncSince so an operator
// resync can report the outcome.
type SyncResult struct {
	Upserted         int
	SkippedUnscored  int
	SkippedNoCycle   int
	SkippedBadDate   int
}
```

- [ ] **Step 2: Write failing tests** in `service_test.go`, using the existing fakes:
  - `TestSyncSince_LabelsAdminResync`: a fake API returning e.g. 2 scored recoveries with matching cycles; call `SyncSince(ctx, userID, 30*24*time.Hour, 25)`. Assert the returned `SyncResult.Upserted == 2`. Assert the metric increment used `kind="admin_resync"` and NOT `kind="window"`. Read the metric via `github.com/prometheus/client_golang/prometheus/testutil` — `testutil.ToFloat64(prometheus... )` on the `syncs_total`/`api_whoop_syncs_total` vec with the specific labels. (Look at how `metrics.go` exposes `syncsTotal`; use `syncsTotal.WithLabelValues("admin_resync","ok")` with `testutil.ToFloat64`. If the counter is package-private, the test is in-package `whoopsync` so it can read it directly.)
  - `TestSyncSince_UpstreamErrorIsClassified`: fake API's `Recoveries` returns an error; assert `errors.Is(err, ErrUpstream)`.
  - Confirm `SyncSince` returns `ErrReconnectNeeded` (already produced by `validToken`) when the connection is not connected — reuse the existing pattern that tests `SyncWindow`'s reconnect path if present.

- [ ] **Step 3: Run, verify fail.**

- [ ] **Step 4: Implement.**
  - Change the unexported signature to `func (s *Service) syncWindow(ctx context.Context, kind, userID string, start, end time.Time, limit int) (SyncResult, error)`.
  - Every early `return err` inside `syncWindow` becomes `return SyncResult{}, err`.
  - Wrap the two fetch failures with the sentinel:
    ```go
    recoveries, err := s.api.Recoveries(ctx, accessToken, start, end, limit)
    if err != nil {
        return SyncResult{}, fmt.Errorf("%w: fetch recoveries: %w", ErrUpstream, err)
    }
    cycles, err := s.api.Cycles(ctx, accessToken, start, end, limit)
    if err != nil {
        return SyncResult{}, fmt.Errorf("%w: fetch cycles: %w", ErrUpstream, err)
    }
    ```
  - At the end, build and return the result:
    ```go
    result = "ok"
    return SyncResult{Upserted: upserted, SkippedUnscored: skippedUnscored, SkippedNoCycle: skippedNoCycle, SkippedBadDate: skippedBadDate}, nil
    ```
  - Update `SyncWindow` and `Backfill` to discard the result:
    ```go
    func (s *Service) SyncWindow(ctx context.Context, userID string, limit int) error {
        now := s.now()
        _, err := s.syncWindow(ctx, "window", userID, now.Add(-recentWindow), now, limit)
        return err
    }
    func (s *Service) Backfill(ctx context.Context, userID string) error {
        now := s.now()
        _, err := s.syncWindow(ctx, "backfill", userID, now.Add(-backfillWindow), now, backfillLimit)
        return err
    }
    ```
  - Add `SyncSince`:
    ```go
    // SyncSince runs an operator-triggered resync over [now-window, now]. It is a
    // thin wrapper over syncWindow with kind="admin_resync" — a deliberate label
    // so an operator investigating an outage does not increment the window-sync
    // liveness counter the dead-ingestion alert watches.
    func (s *Service) SyncSince(ctx context.Context, userID string, window time.Duration, limit int) (SyncResult, error) {
        now := s.now()
        return s.syncWindow(ctx, "admin_resync", userID, now.Add(-window), now, limit)
    }
    ```

- [ ] **Step 5: Run, verify pass.** Also run the whole package: `go test ./internal/whoopsync/`.

- [ ] **Step 6: API gate + commit.** The webhook handler and server wiring still compile (SyncWindow/Backfill signatures unchanged). Commit:
```
feat(whoopsync): add SyncSince for admin resync with admin_resync metric label
```

---

## Task A4 (API): `internal/whoopadmin` package — admin routes

**Files:**
- Create: `internal/whoopadmin/handler.go`
- Test: `internal/whoopadmin/handler_test.go`

Read `internal/beta/handler.go` (the "admin gate applied by enclosing group, no auth import" pattern + `httpresp` usage), `internal/httpresp/` (`OK`, `Error`, `ServerError`), and the DTOs in the SOW's API Surface section. This package must NOT import `internal/auth`.

- [ ] **Step 1: Define the package, narrow consumer interfaces, DTOs, and constructor** in `handler.go`:
```go
// Package whoopadmin exposes an admin-gated view over WHOOP connection state
// and an operator-triggered resync. It is kept out of whoopsync (whose handler
// already spans OAuth, connection CRUD, and recovery reads) because it has a
// different auth model: like beta, the admin gate is applied by the enclosing
// RequireAdmin router group in server.go, so this package does not import auth.
package whoopadmin

// connReader is the whoopconn read surface this handler needs.
type connReader interface {
	List(ctx context.Context) ([]whoopconn.Connection, error)
	Get(ctx context.Context, userID string) (*whoopconn.Connection, error)
}

// recoveryReader is the whooprecovery read surface this handler needs.
type recoveryReader interface {
	Latest(ctx context.Context, userID string) (*whooprecovery.Entry, error)
	CountForUser(ctx context.Context, userID string) (int, error)
}

// resyncer is the whoopsync write surface this handler needs.
type resyncer interface {
	SyncSince(ctx context.Context, userID string, window time.Duration, limit int) (whoopsync.SyncResult, error)
}

type Handler struct {
	conns connReader
	rec   recoveryReader
	svc   resyncer
	now   func() time.Time
}

func NewHandler(conns connReader, rec recoveryReader, svc resyncer, now func() time.Time) *Handler {
	if now == nil {
		now = time.Now
	}
	return &Handler{conns: conns, rec: rec, svc: svc, now: now}
}

// Mount registers the three admin routes WITHOUT an admin gate — the caller
// wraps them in an auth.RequireAdmin group (see server.go), matching beta.
func (h *Handler) Mount(r chi.Router) {
	r.Get("/admin/whoop/connections", h.listConnections)
	r.Get("/admin/whoop/connections/{userID}", h.getConnection)
	r.Post("/admin/whoop/resync", h.resync)
}
```
DTO (JSON tags exactly as the SOW's example):
```go
type connectionView struct {
	UserID             string  `json:"user_id"`
	WhoopUserID        int64   `json:"whoop_user_id"`
	Status             string  `json:"status"`
	Scopes             string  `json:"scopes"`
	TokenExpiresAt     string  `json:"token_expires_at"`
	TokenExpired       bool    `json:"token_expired"`
	ConnectedAt        string  `json:"connected_at"`
	UpdatedAt          string  `json:"updated_at"`
	LatestRecoveryDate *string `json:"latest_recovery_date"` // null when no rows
	RecoveryRowCount   int     `json:"recovery_row_count"`
}
```
Times render as RFC3339 (`t.UTC().Format(time.RFC3339)`); `latest_recovery_date` is the `Entry.Date` string (already `YYYY-MM-DD`) or `nil`.

- [ ] **Step 2: Write failing handler tests** in `handler_test.go` using in-package fakes (implement only the three narrow interfaces — trivial structs with func fields or fixed returns). Cover:
  - `listConnections` happy path: two connections, one with recovery rows (Latest + count) and one without (`latest_recovery_date` null, count 0); assert JSON shape + `token_expired` computed against an injected `now`.
  - `getConnection` happy path (one user) and `404` when `conns.Get` returns `whoopconn.ErrNotFound`.
  - `resync` happy path returns the mapped counts; `window_days` defaults to 30 when omitted and is clamped to `[1,90]` (test 0→1 and 1000→90 by asserting the `window` passed to a spy `resyncer`); `404` when the connection is absent; `409` when status != connected (body names the status); `409` when `svc.SyncSince` returns `whoopsync.ErrReconnectNeeded`; `502` when it returns a `whoopsync.ErrUpstream`-wrapped error; `500` otherwise; `400` when `user_id` is empty.
  - Drive handlers with `httptest` + chi router mounting `h.Mount`. Assert status codes and decoded envelopes (`data` object). Use `httptest.NewRequest`/`ResponseRecorder`.

- [ ] **Step 3: Run, verify fail.**

- [ ] **Step 4: Implement the handlers.** Sketch:
  - `buildView(ctx, conn)`: compute `token_expired := h.now().After(conn.TokenExpiresAt)`; `latest, _ := h.rec.Latest(ctx, conn.UserID)` → date pointer or nil; `count, _ := h.rec.CountForUser(ctx, conn.UserID)`. (N+1 across List is acceptable and documented — one row per opted-in user; revisit >10k per SOW.) Propagate real errors via `httpresp.ServerError`.
  - `listConnections`: `conns.List` → map each to `connectionView` → `httpresp.OK(w, "listed whoop connections", struct{ Connections []connectionView }{...})` with json tag `connections`.
  - `getConnection`: `chi.URLParam(r,"userID")`; `conns.Get`; `errors.Is(err, whoopconn.ErrNotFound)` → `httpresp.Error(w, 404, "whoop connection not found")`; else build view → `httpresp.OK(w, "whoop connection", view)`.
  - `resync`: decode `{user_id string, window_days int}`; `400` if `user_id==""`; default `window_days` 30 if `<=0` unset (treat missing as 30 — decode into `*int` or default after decode), clamp to `[1,90]`; look up `conns.Get(userID)` → `404` on ErrNotFound; if `conn.Status != whoopconn.StatusConnected` → `httpresp.Error(w, 409, fmt.Sprintf("whoop connection status is %q; reconnect required", conn.Status))`; else `res, err := h.svc.SyncSince(ctx, userID, time.Duration(windowDays)*24*time.Hour, resyncPageLimit)`; on error map `ErrReconnectNeeded`→409, `ErrUpstream`→502, else `httpresp.ServerError`; on success `httpresp.OK(w, "resynced whoop recovery", resyncOutcome{Upserted: res.Upserted, ...})` with json tags `upserted, skipped_unscored, skipped_no_cycle, skipped_bad_date`.
  - Add `const resyncPageLimit = 25 // WHOOP v2 list page size; mirrors backfill`.
  - Envelope wrappers:
    ```go
    type listResponse struct{ Connections []connectionView `json:"connections"` }
    type resyncOutcome struct {
        Upserted        int `json:"upserted"`
        SkippedUnscored int `json:"skipped_unscored"`
        SkippedNoCycle  int `json:"skipped_no_cycle"`
        SkippedBadDate  int `json:"skipped_bad_date"`
    }
    ```

- [ ] **Step 5: Run, verify pass** — `go test ./internal/whoopadmin/ -v`.

- [ ] **Step 6: API gate + commit.**
```
feat(whoopadmin): add admin WHOOP connection reads and resync endpoint
```

---

## Task A5 (API): `api_whoop_connections` gauge + exporter

**Files:**
- Create: `internal/whoopadmin/connections_gauge.go`
- Test: `internal/whoopadmin/connections_gauge_test.go`

Read `internal/whoopsync/metrics.go` for the `prometheus.NewGaugeVec` + `prometheus.MustRegister(...)` in `init()` convention and `api_whoop_*` naming. The gauge is `api_whoop_connections{status}` with the closed status set `connected`, `revoked`, `error`.

- [ ] **Step 1: Write failing tests** in `connections_gauge_test.go`:
  - A fake `connLister` returning e.g. 2 `connected`, 1 `error`, 0 `revoked`.
  - Construct the exporter, call `exp.refresh(ctx)` once, assert `testutil.ToFloat64(connectionsGauge.WithLabelValues("connected")) == 2`, `"error" == 1`, `"revoked" == 0` (all three statuses always set, so a status dropping to zero is observable).
  - A second refresh after the fake returns fewer connections drops the gauge accordingly (proves each refresh re-sets all three, not just increments).

- [ ] **Step 2: Run, verify fail.**

- [ ] **Step 3: Implement `connections_gauge.go`:**
```go
package whoopadmin

// api_whoop_connections gauges connection health by status, refreshed every
// refreshInterval. It gates the dead-ingestion alert (an empty DB must not
// page) and gives the ps-whoop dashboard a connection-health panel.
var connectionsGauge = prometheus.NewGaugeVec(
	prometheus.GaugeOpts{
		Name: "api_whoop_connections",
		Help: "Current count of WHOOP connections by status.",
	},
	[]string{"status"},
)

func init() { prometheus.MustRegister(connectionsGauge) }

const refreshInterval = 5 * time.Minute

// allStatuses is the closed set always published, so a status going to zero is
// visible rather than a stale non-zero sample lingering.
var allStatuses = []whoopconn.Status{whoopconn.StatusConnected, whoopconn.StatusRevoked, whoopconn.StatusError}

type connLister interface {
	List(ctx context.Context) ([]whoopconn.Connection, error)
}

type ConnectionsExporter struct {
	conns connLister
}

func NewConnectionsExporter(conns connLister) *ConnectionsExporter {
	return &ConnectionsExporter{conns: conns}
}

// Run refreshes immediately, then every refreshInterval, until ctx is cancelled.
func (e *ConnectionsExporter) Run(ctx context.Context) {
	if err := e.refresh(ctx); err != nil {
		slog.WarnContext(ctx, "whoopadmin: connection gauge refresh failed", "error", err)
	}
	t := time.NewTicker(refreshInterval)
	defer t.Stop()
	for {
		select {
		case <-ctx.Done():
			return
		case <-t.C:
			if err := e.refresh(ctx); err != nil {
				slog.WarnContext(ctx, "whoopadmin: connection gauge refresh failed", "error", err)
			}
		}
	}
}

func (e *ConnectionsExporter) refresh(ctx context.Context) error {
	conns, err := e.conns.List(ctx)
	if err != nil {
		return err
	}
	counts := map[whoopconn.Status]int{}
	for _, c := range conns {
		counts[c.Status]++
	}
	for _, s := range allStatuses {
		connectionsGauge.WithLabelValues(string(s)).Set(float64(counts[s]))
	}
	return nil
}
```

- [ ] **Step 4: Run, verify pass.**

- [ ] **Step 5: API gate + commit.**
```
feat(whoopadmin): add api_whoop_connections gauge exporter
```

---

## Task A6 (API): `api_webhook_misroute_total` counter + chi NotFound handler

**Files:**
- Modify: `internal/server/metrics.go`
- Test: `internal/server/metrics_misroute_test.go`

Read `internal/server/metrics.go` (existing `prometheus.NewCounterVec` + `init()` registration, `<unmatched>` handling). The counter must be silent except for real provider misdeliveries.

- [ ] **Step 1: Write failing tests** in `metrics_misroute_test.go`:
  - Call `webhookMisrouteNotFound` (the handler func) with `httptest.NewRequest("POST", "/webhooks/whoop,", nil)` and a recorder; assert `testutil.ToFloat64(webhookMisrouteTotal.WithLabelValues("whoop"))` increased by 1 and the response is `404`.
  - Call it with `/.env` and with `/webhooks/incoming/stripe.json`; assert the `whoop` counter did NOT increase (capture before/after) and both still return `404`.
  - (Case-insensitivity) `/Webhooks/WHOOP` increments `whoop`.

- [ ] **Step 2: Run, verify fail.**

- [ ] **Step 3: Implement in `metrics.go`:**
```go
// api_webhook_misroute_total counts 404s whose path matches a known webhook
// provider fragment — a delivery to a path we do not serve. Deliberately narrow
// (a tiny hard-coded registry) so it is silent except when a real provider is
// misdelivering; a general /webhooks/ 404 counter would readmit scanner noise.
var webhookMisrouteTotal = prometheus.NewCounterVec(
	prometheus.CounterOpts{
		Name: "api_webhook_misroute_total",
		Help: "Total 404 requests whose path matches a known webhook provider fragment, labeled by provider.",
	},
	[]string{"provider"},
)

// providerFragments maps a lowercase path fragment to the provider it
// identifies. Kept tiny and hard-coded on purpose (see the counter's help).
var providerFragments = map[string]string{"whoop": "whoop"}

// webhookMisrouteNotFound is the chi NotFound handler: it increments the
// misroute counter when the 404 path matches a known provider fragment, then
// serves the standard 404 so behaviour is otherwise unchanged.
func webhookMisrouteNotFound(w http.ResponseWriter, r *http.Request) {
	path := strings.ToLower(r.URL.Path)
	for frag, provider := range providerFragments {
		if strings.Contains(path, frag) {
			webhookMisrouteTotal.WithLabelValues(provider).Inc()
			break
		}
	}
	http.NotFound(w, r)
}
```
Add `webhookMisrouteTotal` to the existing `prometheus.MustRegister(...)` call in `init()`. Add `"strings"` to imports.

- [ ] **Step 4: Run, verify pass** — `go test ./internal/server/ -run Misroute -v`.

- [ ] **Step 5: API gate + commit.**
```
feat(server): count misrouted webhook deliveries via a NotFound handler
```

---

## Task A7 (API): server wiring — mount admin surface, start exporter, register NotFound

**Files:**
- Modify: `internal/server/server.go`

Read `server.go` around: router creation + middleware boundary (60-130), the whoop enable block (464-499), the `RequireAdmin` group (740-750), `Server` struct (~48), `New`'s return (~782), and `Run` (794-813). The whoop repos `whoopConnRepo`/`whoopRecoveryRepo` are constructed at 271-272; the `whoopSvc` is currently local to the enable block (486).

- [ ] **Step 1: Register the NotFound handler.** After the "--- All r.Use() calls must be above this line ---" boundary (near `r.Handle("/metrics", ...)`), add:
```go
	// Misrouted-webhook observability: a provider posting to a path we don't
	// serve (e.g. a trailing comma) 404s at the router, upstream of every
	// api_whoop_* counter. This NotFound handler makes that 404 observable
	// (see api_webhook_misroute_total) while still serving the standard 404.
	r.NotFound(webhookMisrouteNotFound)
```

- [ ] **Step 2: Hoist the pieces the admin surface + exporter need.** Add a lifecycle context for background jobs near the top of `New` (before the whoop block):
```go
	// Cancelable context for server-owned background goroutines (e.g. the WHOOP
	// connection-gauge exporter). Cancelled by Run on shutdown.
	bgCtx, bgCancel := context.WithCancel(context.Background())
```
Declare `var whoopAdminHandler *whoopadmin.Handler` in the outer scope alongside `var whoopHandler *whoopsync.Handler` (line 473). Inside the enable block (after `whoopSvc` is built at line 486), add:
```go
			whoopAdminHandler = whoopadmin.NewHandler(whoopConnRepo, whoopRecoveryRepo, whoopSvc, nil)
			// Publish the connection-health gauge every 5 minutes; gates the
			// dead-ingestion alert and drives the dashboard connection panel.
			go whoopadmin.NewConnectionsExporter(whoopConnRepo).Run(bgCtx)
```
(`whoopConnRepo` satisfies both `whoopadmin.connReader` and `connLister`; `whoopRecoveryRepo` satisfies `recoveryReader`; `whoopSvc` satisfies `resyncer`.)

- [ ] **Step 3: Mount inside the `RequireAdmin` group** (740-750), after the `vmHandler` mount, guarded like `vmHandler`:
```go
			// Admin WHOOP surface — connection reads + operator resync. Present
			// only when the WHOOP integration is enabled (same guard as the
			// authed half); the admin gate is this enclosing group.
			if whoopAdminHandler != nil {
				whoopAdminHandler.Mount(r)
			}
```

- [ ] **Step 4: Cancel background jobs on shutdown.** Add a field to `Server`:
```go
type Server struct {
	httpServer *http.Server
	bgCancel   context.CancelFunc
}
```
Set it in the returned struct at the end of `New`: `return &Server{httpServer: ..., bgCancel: bgCancel}, nil`. In `Run`, in the `<-ctx.Done()` branch (before/after `Shutdown`), call it:
```go
	case <-ctx.Done():
		log.Println("shutdown signal received")
		if s.bgCancel != nil {
			s.bgCancel()
		}
		shutdownCtx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
		defer cancel()
		return s.httpServer.Shutdown(shutdownCtx)
```
Add the `whoopadmin` import. Ensure `bgCancel` is not reported as unused when whoop is disabled — it is always wired into the returned `Server`, so it is used; the exporter goroutine simply never starts. (If the linter flags an unused `bgCtx` when whoop is disabled: it is captured by the returned struct via `bgCancel`, and `bgCtx` is referenced in the enable block; a disabled build still compiles because `bgCtx` is only *used* in the block — if `unused` fires, keep `bgCtx` used by passing it even in the disabled path is unnecessary. `context.WithCancel` returns both; `bgCancel` is always used, `bgCtx` is used only inside the enable block. Go does not flag a variable used in *some* branch. This compiles.)

- [ ] **Step 5: Verify wiring.** There is no server-level integration test harness for the full router in this repo, so verify by the gate: `go build ./...`, `go vet ./...`, lint, `go test ./...` (the whoopadmin + server package tests cover behaviour). Manually re-read the diff to confirm the guard mirrors `vmHandler`.

- [ ] **Step 6: API gate + commit.**
```
feat(server): wire the admin WHOOP surface and connection-gauge exporter
```

---

## Task T1 (tooling): share `build_client` / `describe_failure` in `cloudwatch.py`

**Files:**
- Modify: `src/prog_strength_tooling/cloudwatch.py`
- Test: `tests/test_cloudwatch.py`

Read `cloudwatch.py` fully and `tests/test_cloudwatch.py`. The two helpers `_build_client(cfg)` and `_describe_failure(exc, group, cfg)` are currently module-private and used by `_search_group`/`search`. `whooplogs.py` (Task T3) will reuse them, so they become public (un-underscored).

- [ ] **Step 1: Update the test** `tests/test_cloudwatch.py`: change every `monkeypatch.setattr(cloudwatch, "_build_client", ...)` to `"build_client"` and any direct references to `_describe_failure` to `describe_failure`. Run to confirm it now fails against the not-yet-renamed source.

- [ ] **Step 2: Rename in `cloudwatch.py`.** `def _build_client` → `def build_client`; `def _describe_failure` → `def describe_failure`; update the internal call sites (`_search_group`, `search`, anywhere else in the file). Keep behaviour identical.

- [ ] **Step 3: Run** `uv run pytest tests/test_cloudwatch.py -q` → PASS.

- [ ] **Step 4: tooling gate + commit.**
```
refactor(cloudwatch): make build_client and describe_failure shareable
```

---

## Task T2 (tooling): WHOOP admin API client

**Files:**
- Modify: `src/prog_strength_tooling/client.py`
- Modify: `src/prog_strength_tooling/models.py`
- Test: `tests/test_whoop_client.py`

Read `client.py` (`MemoryClient`, `ClientError`, `APIError`, `_unwrap`), `models.py` (Pydantic DTO style), and `tests/test_client.py` (respx pattern). Mirror `MemoryClient` exactly.

- [ ] **Step 1: Add Pydantic models** in `models.py` for the admin payloads (fields per the SOW's JSON):
```python
class WhoopConnection(BaseModel):
    user_id: str
    whoop_user_id: int
    status: str
    scopes: str
    token_expires_at: str
    token_expired: bool
    connected_at: str
    updated_at: str
    latest_recovery_date: str | None = None
    recovery_row_count: int

class WhoopResyncOutcome(BaseModel):
    upserted: int
    skipped_unscored: int
    skipped_no_cycle: int
    skipped_bad_date: int
```
Match the existing models' base-class / config conventions in the file.

- [ ] **Step 2: Write failing tests** in `tests/test_whoop_client.py` (respx), mirroring `test_client.py`:
  - `list_connections()` → `GET {base}/admin/whoop/connections`, unwraps `data.connections` into `list[WhoopConnection]`, sends `Authorization: Bearer`.
  - `get_connection(user_id)` → `GET .../admin/whoop/connections/{user_id}`, unwraps `data` → `WhoopConnection`; a `404` raises `APIError` with status 404.
  - `resync(user_id, window_days)` → `POST .../admin/whoop/resync` with JSON body `{"user_id":..., "window_days":...}`, unwraps `data` → `WhoopResyncOutcome`.
  - Constructing the client with no token raises `ClientError` (mirror `MemoryClient.__init__`).

- [ ] **Step 3: Run, verify fail.**

- [ ] **Step 4: Implement `WhoopAdminClient`** in `client.py`, structurally identical to `MemoryClient` (same `__init__` token guard, `httpx.Client` with base_url + bearer header + timeout, `__enter__/__exit__/close`, `_unwrap` — reuse the existing module-level `_unwrap` if it is a staticmethod, otherwise copy the pattern). Methods return the Pydantic models above.

- [ ] **Step 5: Run, verify pass.**

- [ ] **Step 6: tooling gate + commit.**
```
feat(whoop): add WhoopAdminClient for the admin connection + resync endpoints
```

---

## Task T3 (tooling): `whooplogs.py` — CloudWatch delivery + sync evidence

**Files:**
- Create: `src/prog_strength_tooling/whooplogs.py`
- Test: `tests/test_whooplogs.py`

Read `cloudwatch.py` again (now with public `build_client`/`describe_failure`, `LogsConfig`, `FilterLogEvents` paginator usage, `logparse.py`) and `tests/test_cloudwatch.py` (fake paginator + `Stubber`). Reuse `build_client`/`describe_failure` — do not duplicate credential/region error handling.

`whooplogs.py` provides two scans over the API log group (`config.LOG_GROUP_PREFIX` + api), each using `FilterLogEvents` with a quoted-substring filter and client-side aggregation:

- **Delivery scan** — every `POST` request line whose URI contains `whoop`, grouped by `(uri, status)` with a count. The API logs are JSON `slog` lines (see `logparse.py`); extract method, uri/path, and status. A misdelivery shows up as e.g. `(/webhooks/whoop,, 404) → 97`.
- **Sync scan** — `whoopsync: sync complete` log lines carrying `kind`, `upserted`, `skipped_*`; count how many have `kind=window` and sum `upserted`. (These are the lines emitted by `service.go`'s `slog.InfoContext(... "whoopsync: sync complete" ...)`.)

- [ ] **Step 1: Write failing tests** in `tests/test_whooplogs.py` using the fake-paginator/`Stubber` approach from `test_cloudwatch.py`:
  - A fixture of API log events replaying the outage: 97 events for `POST /webhooks/whoop,` with status `404`, zero `whoopsync: sync complete kind=window` lines → delivery scan returns a single `(uri="/webhooks/whoop,", status=404, count=97)` group; sync scan returns `window_sync_count=0, upserted_total=0`.
  - A healthy fixture: several `POST /webhooks/whoop` `204` deliveries + several `kind=window` sync-complete lines with `upserted>0` → delivery scan groups them under `(/webhooks/whoop, 204)`; sync scan returns positive counts.

- [ ] **Step 2: Run, verify fail.**

- [ ] **Step 3: Implement `whooplogs.py`.** Define small result dataclasses (e.g. `DeliveryGroup(uri, status, count)`, `DeliveryScan(groups)`, `SyncScan(window_sync_count, upserted_total)`), and two functions `scan_deliveries(cfg: LogsConfig, window) -> DeliveryScan` and `scan_syncs(cfg: LogsConfig, window) -> SyncScan`. Both: `client = build_client(cfg)`; paginate `client.get_paginator("filter_log_events").paginate(logGroupName=<api group>, filterPattern='"whoop"', startTime=..., endTime=...)`; parse each event `message` (JSON via `logparse` where possible; fall back to substring checks). Wrap botocore failures with `describe_failure(exc, group, cfg)` raising `cloudwatch.CloudWatchError`. Keep the filter pattern a literal `"whoop"` substring match (matches JSON and text lines identically, per `cloudwatch.py`'s documented rationale).

- [ ] **Step 4: Run, verify pass.**

- [ ] **Step 5: tooling gate + commit.**
```
feat(whoop): add whooplogs CloudWatch delivery and sync scans
```

---

## Task T4 (tooling): `whoop.py` — diagnosis engine

**Files:**
- Create: `src/prog_strength_tooling/whoop.py`
- Test: `tests/test_whoop_diagnosis.py`

Read `whooplogs.py` (Task T3) and `client.py`'s `WhoopAdminClient` (Task T2). This module runs the seven checks from the SOW and returns a `Diagnosis`. It must degrade gracefully when the admin token is absent (checks 6 & 7 become "skipped", with a reason) — a missing token must not block the five log-derived checks.

The seven checks (see SOW "The checks" table):

| # | Check | Source | Fires when |
|---|-------|--------|-----------|
| 1 | Deliveries arriving | logs | No POST whose URI contains `whoop` in window |
| 2 | Deliveries reaching the handler | logs | Any delivery on a path other than `/webhooks/whoop`, or any non-2xx |
| 3 | Signatures accepted | logs | `401` responses present |
| 4 | Deliveries producing syncs | logs | deliveries > 0 but no `kind=window` sync-complete lines |
| 5 | Syncs landing rows | logs | `upserted == 0` while any `skipped_*` > 0 (derivable from sync-complete lines) |
| 6 | Connection health | admin API | status ≠ connected, or token expired |
| 7 | Data freshness | admin API | newest recovery date older than 48h |

- [ ] **Step 1: Write failing tests** in `tests/test_whoop_diagnosis.py`. The engine takes injected evidence (delivery scan, sync scan) and an optional admin client / connection data, so tests pass fakes directly (no network):
  - **Regression fixture (the outage):** delivery scan = one group `(/webhooks/whoop,, 404, 97)`, sync scan = `(0, 0)`, no admin data → `Diagnosis` has a finding for check 2 whose evidence names the offending path `/webhooks/whoop,` and whose fix references the WHOOP Developer Dashboard webhook URL; overall status is unhealthy (findings present).
  - **Healthy fixture:** delivery scan = `(/webhooks/whoop, 204, N)`, sync scan = `(window_sync_count>0, upserted_total>0)`, admin connection = connected, token not expired, latest recovery within 48h → no findings.
  - **Degraded (no token):** admin data absent → checks 6 & 7 reported as `skipped` with a reason; the five log checks still run.
  - Check-3 fires when a delivery group has status 401; check-1 fires when there are zero whoop POSTs.

- [ ] **Step 2: Run, verify fail.**

- [ ] **Step 3: Implement `whoop.py`.** Define:
```python
@dataclass
class Finding:
    check: str          # short id/name, e.g. "delivery-path"
    symptom: str
    evidence: str
    fix: str

@dataclass
class CheckResult:
    name: str
    ok: bool
    skipped: bool = False
    reason: str | None = None
    finding: Finding | None = None

@dataclass
class Diagnosis:
    checks: list[CheckResult]
    @property
    def findings(self) -> list[Finding]: ...
    @property
    def healthy(self) -> bool: ...      # no findings and no errors
```
A `diagnose(deliveries: DeliveryScan, syncs: SyncScan, connection: WhoopConnection | None, token_present: bool, now: datetime) -> Diagnosis` pure function assembles the seven `CheckResult`s from the injected evidence. Checks 6/7 → `skipped` when `not token_present or connection is None`. Freshness (check 7): parse `latest_recovery_date` (`YYYY-MM-DD`) and compare to `now - 48h`. The finding text for check 2 must include the concrete offending path and the corrective action + expected URL `https://api.progstrength.fitness/webhooks/whoop` (see the SOW's example block). Keep the orchestration that actually calls `whooplogs`/`WhoopAdminClient` in the command layer (Task T5) so `diagnose` stays pure and unit-testable.

- [ ] **Step 4: Run, verify pass.**

- [ ] **Step 5: tooling gate + commit.**
```
feat(whoop): add the doctor diagnosis engine with seven chain checks
```

---

## Task T5 (tooling): `commands/whoop.py` + render + cli wiring

**Files:**
- Create: `src/prog_strength_tooling/commands/whoop.py`
- Modify: `src/prog_strength_tooling/render.py`
- Modify: `src/prog_strength_tooling/cli.py`
- Test: `tests/test_whoop_cli.py`

Read `commands/logs.py` (Typer sub-app, `--since`/`--json`, exit codes `0`/`1`/`2`, error handling), `commands/memory.py` (`resolve_admin`, `MissingTokenError`, `_fail`), `render.py` (tables, panels, `_print_json`, `err_console`), `cli.py` (`app.add_typer(...)`), and `config.py` (`resolve`, `resolve_admin`, `resolve_logs`, `CLOUDWATCH_ENVIRONMENTS`).

CLI surface:
```
pst whoop doctor [--user ID] [--since 7d] [--json] [--env ...] [--profile ...] [--region ...] [--token ...] [--api ...]
pst whoop resync --user ID [--days 30] [--env ...] [--token ...] [--api ...]
```

- [ ] **Step 1: Write failing CLI tests** in `tests/test_whoop_cli.py` using `typer.testing.CliRunner` + respx (admin calls) + monkeypatching `whooplogs.scan_deliveries`/`scan_syncs` (or `cloudwatch.build_client`) to return fixture scans:
  - Regression: log scan fixtures replay the outage; `pst whoop doctor` (no token, so degraded, or with token) prints a check-2 finding naming `/webhooks/whoop,` and exits `1`.
  - Healthy: fixtures healthy + admin connection healthy → exit `0`, no findings.
  - `--json` emits the serialised `Diagnosis`.
  - Config/AWS error path exits `2` (e.g. `CloudWatchError` raised).
  - Token-absent `doctor` still runs and reports checks 6/7 as skipped (exit `1` only if a log check found something, else `0`).
  - `resync --user X --days 200` clamps to 90 in the request body (respx assertion), prints the outcome; `resync` maps client `APIError(404)` / `409` to a clear message and non-zero exit.

- [ ] **Step 2: Run, verify fail.**

- [ ] **Step 3: Implement.**
  - `render.py`: add `render_diagnosis(diagnosis, *, as_json)` — table/panel of each check (✓/✗/skipped, symptom, evidence, fix), findings highlighted; and `render_resync(outcome, *, as_json)`. Follow the existing rich table + `_print_json` patterns; findings to `console`, errors to `err_console`.
  - `commands/whoop.py`: a `app = typer.Typer(...)` with `doctor` and `resync` commands.
    - `doctor`: resolve logs config via `config.resolve_logs(...)`; run `whooplogs.scan_deliveries`/`scan_syncs`. For the two admin checks: try `config.resolve_admin(...)` + `WhoopAdminClient` — if `MissingTokenError`, set `token_present=False` and skip (do not fail). Build the `Diagnosis` via `whoop.diagnose(...)`, render, and set exit code: `0` healthy, `1` findings present, `2` on `ConfigError`/`CloudWatchError`/`APIError` (catch and print to `err_console`, `raise typer.Exit(2)`). Mirror `logs.py`'s exit-code constants and try/except shape.
    - `resync`: `config.resolve_admin(...)` (raises `MissingTokenError` → the missing-token panel + exit); `WhoopAdminClient.resync(user, days)`; render outcome; map `APIError` to a message + exit `1`.
  - `cli.py`: `from .commands import ... whoop` and `app.add_typer(whoop.app, name="whoop")`.

- [ ] **Step 4: Run, verify pass** — `uv run pytest tests/test_whoop_cli.py -q`.

- [ ] **Step 5: tooling gate + commit.**
```
feat(whoop): add pst whoop doctor and resync commands
```

---

## Task I1 (infra): `rules-whoop.yml` — dead-ingestion + misroute alerts

**Files:**
- Create: `monitoring/grafana/provisioning/alerting/rules-whoop.yml`

Read `monitoring/grafana/provisioning/alerting/rules-vector-memory.yml` (structure) and `README.md` (conventions: `noDataState: OK` for error counters; `0.5` midpoint thresholds; no `$` in YAML). Model the file on `rules-vector-memory.yml`. Group `name: whoop`, `folder: Prog Strength Alerts`, `interval: 1m`. Both rules annotate `__dashboardUid__: ps-whoop`.

Implement exactly (this is the SOW's specified PromQL and alert config — do not "improve" it; the operator validates in Grafana):

- [ ] **Step 1: Write the file.**
  - **Rule A — dead ingestion (cause-agnostic backstop).** `uid: whoop-dead-ingestion`, `title: WHOOP ingestion dead — critical`, severity `critical`. Data step A (`datasourceUid: prometheus`, `relativeTimeRange.from: 129600` (36h), `instant: true`):
    ```
    (
      sum(increase(api_whoop_syncs_total{kind="window",result="ok"}[36h])) or vector(0)
    )
    and on()
    (sum(api_whoop_connections{status="connected"}) > 0)
    ```
    Threshold step C (`__expr__`, `type: threshold`, `expression: A`): evaluator `type: lt`, `params: [0.5]`. `for: 0s`. `noDataState: Alerting` (deliberate departure from the file-level convention — this is a liveness monitor, not an error counter; document it in a comment). `execErrState: Alerting`. Summary annotation explaining zero window syncs in 36h with ≥1 connected account.
  - **Rule B — misrouted deliveries (fast, specific).** `uid: whoop-webhook-misroute`, `title: WHOOP webhook misrouted — warning`, severity `warning`. Data step A (`from: 3600` (1h), `instant: true`):
    ```
    sum by (provider) (increase(api_webhook_misroute_total[1h]))
    ```
    Threshold step C: evaluator `type: gt`, `params: [0.5]`. `for: 0s`. `noDataState: OK` (standard for a counter that should not normally exist). `execErrState: Alerting`. Summary names the provider (`{{ .Labels.provider }}`) and points at the WHOOP Developer Dashboard URL.
  - Lead the file with a top comment (like vector-memory's) explaining: `or vector(0)` materialises a zero so a never-existed `{kind="window"}` series still fires; `and on()` gates on ≥1 connected account; `noDataState: Alerting` on Rule A is the deliberate exception; Rule B is the fast/specific complement.
  - No `$` anywhere; use `.Labels.provider` in annotations, never `$labels`.

- [ ] **Step 2: Validate YAML** — `python3 -c "import yaml,sys; d=yaml.safe_load(open('monitoring/grafana/provisioning/alerting/rules-whoop.yml')); assert d['groups'][0]['name']=='whoop'; print('ok', [r['uid'] for r in d['groups'][0]['rules']])"`.

- [ ] **Step 3: Commit.**
```
feat(alerting): add WHOOP dead-ingestion and webhook-misroute rules
```

---

## Task I2 (infra): `whoop.json` dashboard — threshold + connection-health panel

**Files:**
- Modify: `monitoring/grafana/dashboards/whoop.json`

Read the actual `monitoring/grafana/dashboards/whoop.json` first and reconcile with the SOW (see Reconciliation #4). Current state: panel 2 "Successful syncs (24h)" already has red-below-1; panel 3 "Recovery rows upserted (24h)" has *yellow*-below-1; panel 11 "Recovery rows by disposition" shows the `sum by (...)` timeseries pattern to mirror for a status breakdown.

- [ ] **Step 1: Change panel 3's below-1 threshold step colour** from `yellow` to `red` (the `{"color": "...", "value": null}` step), per the SOW's "Add red-below-1 thresholds to … both". Leave panel 2 as-is (already red-below-1).

- [ ] **Step 2: Add a connection-health panel** driven by the new gauge `api_whoop_connections{status}`. Add it under an appropriate row (e.g. after the syncs row). Use a stat or timeseries panel with target `expr: "api_whoop_connections"` (instant stat with `legendFormat: "{{status}}"`, or `sum by (status) (api_whoop_connections)` for a timeseries), a fresh unique `id` (higher than any existing panel id), a `gridPos` that doesn't overlap existing panels, `datasource {type: prometheus, uid: prometheus}`, `unit: short`, `decimals: 0`. Give it a `title` like "Connections by status" and a `description` noting it exposes connections flipping to `error`/`revoked` — the health the dashboard previously couldn't show. Match the JSON shape of an existing stat/timeseries panel in the file exactly (copy one and adapt).

- [ ] **Step 3: Validate JSON** — `python3 -c "import json; d=json.load(open('monitoring/grafana/dashboards/whoop.json')); ids=[p['id'] for p in d['panels']]; assert len(ids)==len(set(ids)), 'dup panel ids'; print('ok', d['uid'], 'panels:', len(ids))"`. Confirm panel ids are unique and the file still parses.

- [ ] **Step 4: Commit.**
```
feat(dashboard): render zero syncs red and add WHOOP connection-health panel
```

---

## Self-Review (controller, before dispatching)

- **Spec coverage:** Goals 1-7 map to: G1 (connection state) → A1/A2/A4; G2 (idempotent resync) → A3/A4; G3 (`doctor`) → T3/T4/T5; G4 (`resync`) → T2/T5; G5 (misroute alert) → A6/I1; G6 (dead-ingestion alert incl. never-existed series) → A5/A3/I1; G7 (zero visually distinct) → I2. Non-goals (scheduled reconciliation, WHOOP dashboard automation, general misroute framework, user-facing surfacing, arch change) are all respected.
- **Type consistency:** `SyncResult`/`SyncSince`/`ErrUpstream` (A3) are consumed by `whoopadmin.resyncer` (A4); `whoopconn.List` (A2) satisfies both `whoopadmin.connReader` and `connLister` (A4/A5); `whooprecovery.Latest`/`CountForUser` (A1) satisfy `recoveryReader` (A4); `webhookMisrouteNotFound`/`webhookMisrouteTotal` (A6) are wired in A7; `cloudwatch.build_client`/`describe_failure` (T1) consumed by T3; `WhoopAdminClient` + models (T2) consumed by T4/T5; `Diagnosis` (T4) rendered by T5.
- **Ordering:** A1→A2→A3→A4→A5→A6→A7 (repos+service before handler before wiring). T1→T2→T3→T4→T5. I1, I2 independent.
