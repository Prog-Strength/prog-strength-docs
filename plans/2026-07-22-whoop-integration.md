# Whoop Integration (Recovery v1) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let a user connect their Whoop account from Settings → Integrations so their daily recovery metrics (resting HR, HRV, recovery score) flow into Prog Strength automatically via OAuth + push webhooks, mirroring the Google Calendar sync feature end-to-end.

**Architecture:** Mirror the Google Calendar blueprint (`internal/calendarconn` + `internal/calendarsync` on the API, `GoogleCalendarConnectionRow` on the web). Extract AES-256-GCM token crypto from `calendarsync` into a shared `internal/tokencrypt` package. Persist an encrypted connection (`user_whoop_connection`) and a steps-shaped daily metric table (`user_whoop_recovery`). Ingestion is push-driven: a public HMAC-verified `POST /webhooks/whoop` treats each event as a poke and re-syncs a recent window of recoveries via the Whoop v2 REST API. Reads follow the house timezone+local-date convention; the dashboard gains a recovery mini-card; a new MCP `whoop` module lets the agent read recovery data.

**Tech Stack:** Go 1.25 (chi, database/sqlite, testify-free stdlib tests, golangci-lint v2.12.2), Next.js 16 / TypeScript / Vitest, Python 3.12 / FastMCP / pytest, Terraform + GitHub Actions.

**Cross-cutting conventions (every task):**
- Follow the blueprint package's existing patterns exactly. Read the sibling calendar/steps code before writing.
- TDD: write the failing test first, watch it fail, implement, watch it pass, commit.
- Conventional-commit messages, lowercase subject: `feat(whoop): …`. Co-author footer `Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>`.
- Go: `go build ./...`, `go vet ./...`, `golangci-lint run` (v2.12.2), `go mod tidy` (no drift), `go test ./...` must all pass. gosec is active — use `crypto/subtle`/`hmac.Equal` for constant-time compares, parameterized SQL only.
- **Migration numbering deviation:** the SOW names the tables `038_`/`039_`, but `038_activity_route.sql` already exists on main. Use **`039_user_whoop_connection.sql`** and **`040_user_whoop_recovery.sql`**.

---

## File Structure

### prog-strength-api
- Create: `internal/tokencrypt/tokencrypt.go`, `internal/tokencrypt/tokencrypt_test.go` — shared AES-256-GCM cipher (moved from calendarsync/crypto.go).
- Modify: `internal/calendarsync/crypto.go` (delete, re-export shim not needed — switch imports), `internal/calendarsync/*.go` — import `tokencrypt` instead of local crypto.
- Create: `internal/db/migrations/039_user_whoop_connection.sql`, `internal/db/migrations/040_user_whoop_recovery.sql`.
- Create: `internal/whoopconn/{connection.go,repository.go,sqlite_repository.go,errors.go,repository_shared_test.go,sqlite_repository_test.go}` — connection model + repo (mirror `calendarconn`).
- Create: `internal/whooprecovery/{recovery.go,repository.go,sqlite_repository.go,errors.go,repository_test.go}` — daily recovery rows repo (mirror `steps`).
- Create: `internal/whoopsync/{oauth.go,client.go,service.go,handler.go,webhook.go,doc.go}` + tests — OAuth, Whoop v2 client, sync service, connection HTTP handlers, webhook handler.
- Modify: `internal/config/config.go`, `config.toml` — `[whoop]` section.
- Modify: `internal/server/server.go` — DI wiring + route mounts.
- Modify: `internal/dashboard/{dto.go,handler.go,whoop.go(new)}` — recovery summary block.

### prog-strength-web
- Modify: `lib/api.ts` — `WhoopConnection` type, `getWhoopConnection`, `disconnectWhoop`, `DashboardSummary` recovery extension.
- Modify: `app/(app)/settings/page.tsx` + `_components/` — Profile/Integrations tabs via `?tab=`, move Google card to Integrations, add `WhoopConnectionRow`.
- Create: `app/(app)/dashboard/_components/whoop-card.tsx` — recovery MiniCard; wire into dashboard page.
- Tests co-located `*.test.tsx`/`*.test.ts`.

### prog-strength-mcp
- Create: `src/prog_strength_mcp/whoop.py` — `get_whoop_recovery` tool (mirror `steps.py`).
- Modify: `src/prog_strength_mcp/api_client.py` — `get_whoop_recovery` method.
- Modify: `src/prog_strength_mcp/server.py` — register.
- Create: `tests/test_whoop_tools.py`.

### prog-strength-infra
- Modify: `.github/workflows/seed-secrets.yml`, `deploy/api.sh`, `compose/api/docker-compose.yml`, `compose/api/config.env` — three `WHOOP_*` secrets.

### prog-strength-docs
- Modify: `sows/whoop-integration.md` — status flip (done by orchestrator, not a subagent task).

---

## Task 1 (API): Extract token crypto to `internal/tokencrypt`

**Files:**
- Create: `internal/tokencrypt/tokencrypt.go` (move `NewCipher`, `Encrypt`, `Decrypt`, `KeyFromEnv` from `internal/calendarsync/crypto.go`)
- Create: `internal/tokencrypt/tokencrypt_test.go` (move `internal/calendarsync/crypto_test.go`)
- Modify: `internal/calendarsync/crypto.go` — delete; update all `calendarsync` references to use `tokencrypt.Cipher` etc.
- Modify: `internal/server/server.go` — `calendarsync.KeyFromEnv`/`NewCipher` call sites → `tokencrypt.*`

**Spec:** Mechanical, behavior-preserving move. The new package exposes the same `Cipher` type and `NewCipher([]byte)`, `(*Cipher).Encrypt`, `(*Cipher).Decrypt`, and a `KeyFromEnv(raw string) ([]byte, error)`. Change error-string prefixes from `calendarsync:` to `tokencrypt:`. `KeyFromEnv` must be **provider-agnostic** — its error text must not name a specific env var (the old text said `CALENDAR_TOKEN_ENC_KEY`; make it generic like `token encryption key`). `calendarsync` keeps its own call sites working by importing `tokencrypt`. `server.go` builds the calendar cipher via `tokencrypt`.

- [ ] **Step 1:** Move `crypto.go` → `internal/tokencrypt/tokencrypt.go`, `package tokencrypt`, rename error prefixes to `tokencrypt:`, make `KeyFromEnv` error text env-var-agnostic.
- [ ] **Step 2:** Move `crypto_test.go` → `internal/tokencrypt/tokencrypt_test.go`, `package tokencrypt`. Add a **two-key isolation test**: data encrypted with cipher A fails to decrypt with cipher B (built from a different key) — `Decrypt` returns an error.
- [ ] **Step 3:** `go test ./internal/tokencrypt/...` — expect PASS.
- [ ] **Step 4:** In `calendarsync`, delete `crypto.go`/`crypto_test.go`, replace `*Cipher`/`NewCipher`/`KeyFromEnv` references with `tokencrypt.*`. Update `server.go` call sites (`calendarsync.KeyFromEnv` → `tokencrypt.KeyFromEnv`, `calendarsync.NewCipher` → `tokencrypt.NewCipher`; `Handler`/`Service`/`TokenSource` fields typed `*Cipher` become `*tokencrypt.Cipher`).
- [ ] **Step 5:** `go build ./... && go vet ./... && go test ./...` — all PASS. `golangci-lint run` clean. `go mod tidy` no drift.
- [ ] **Step 6:** Commit `refactor(tokencrypt): extract AES-256-GCM token crypto to shared package`.

---

## Task 2 (API): Migrations for connection + recovery tables

**Files:**
- Create: `internal/db/migrations/039_user_whoop_connection.sql`
- Create: `internal/db/migrations/040_user_whoop_recovery.sql`

**Spec:** Mirror `026_user_calendar_connection.sql` and `019_steps.sql`. Migrations are embedded via `go:embed migrations/*.sql` and applied in numeric order; a test that boots a migrated DB will exercise them.

`039_user_whoop_connection.sql`:
```sql
-- user_whoop_connection: one row per user holding the encrypted Whoop OAuth
-- token pair + connection metadata. Mirrors user_calendar_connection (026)
-- with Whoop-specific additions: whoop_user_id (the inbound-webhook join key),
-- a persisted access token (Whoop access tokens live ~1h), and single-use
-- refresh rotation. Token blobs are AES-256-GCM; the key lives outside the DB
-- (WHOOP_TOKEN_ENC_KEY).
CREATE TABLE IF NOT EXISTS user_whoop_connection (
    user_id             TEXT PRIMARY KEY,
    whoop_user_id       INTEGER NOT NULL UNIQUE,
    access_token_enc    BLOB NOT NULL,
    access_token_nonce  BLOB NOT NULL,
    refresh_token_enc   BLOB NOT NULL,
    refresh_token_nonce BLOB NOT NULL,
    token_expires_at    DATETIME NOT NULL,
    scopes              TEXT NOT NULL,
    status              TEXT NOT NULL CHECK (status IN ('connected','revoked','error')),
    connected_at        DATETIME NOT NULL,
    updated_at          DATETIME NOT NULL
);
```

`040_user_whoop_recovery.sql`:
```sql
-- user_whoop_recovery: steps-shaped daily recovery metric table (cf. 019_steps),
-- one row per (user, local calendar date). Hard-deleted, upsert on conflict —
-- latest score wins. sleep_id is stored because v2 webhook delete events
-- identify a recovery by its associated sleep UUID.
CREATE TABLE IF NOT EXISTS user_whoop_recovery (
    id                 TEXT PRIMARY KEY,
    user_id            TEXT NOT NULL,
    date               TEXT NOT NULL,
    recovery_score     REAL,
    resting_heart_rate REAL,
    hrv_rmssd_milli    REAL,
    cycle_id           INTEGER NOT NULL,
    sleep_id           TEXT NOT NULL,
    created_at         DATETIME NOT NULL,
    updated_at         DATETIME NOT NULL,
    UNIQUE (user_id, date)
);
CREATE INDEX IF NOT EXISTS idx_user_whoop_recovery_user_date ON user_whoop_recovery(user_id, date DESC);
CREATE INDEX IF NOT EXISTS idx_user_whoop_recovery_sleep ON user_whoop_recovery(user_id, sleep_id);
```

- [ ] **Step 1:** Create both files with the SQL above.
- [ ] **Step 2:** Add/extend a migration test (mirror how existing migration tests boot a DB) asserting a freshly-migrated DB has both tables (query `sqlite_master`). If the repo has no per-migration test, verify via an existing "all migrations apply" test.
- [ ] **Step 3:** `go test ./internal/db/...` — PASS.
- [ ] **Step 4:** Commit `feat(whoop): add user_whoop_connection and user_whoop_recovery migrations`.

---

## Task 3 (API): `internal/whoopconn` connection repository

**Files:**
- Create: `internal/whoopconn/connection.go`, `repository.go`, `sqlite_repository.go`, `errors.go`, `repository_shared_test.go`, `sqlite_repository_test.go`

**Spec:** Mirror `internal/calendarconn` exactly, adapted to the connection columns. The model carries no token material for reads; tokens are fetched explicitly.

```go
type Status string
const (
    StatusConnected Status = "connected"
    StatusRevoked   Status = "revoked"
    StatusError     Status = "error"
)

// Connection is the metadata view (no token material).
type Connection struct {
    UserID         string
    WhoopUserID    int64
    Scopes         string
    Status         Status
    TokenExpiresAt time.Time
    ConnectedAt    time.Time
    UpdatedAt      time.Time
}

// Tokens is the decrypted-at-caller token bundle stored/read together.
type TokenBundle struct {
    AccessTokenEnc    []byte
    AccessTokenNonce  []byte
    RefreshTokenEnc   []byte
    RefreshTokenNonce []byte
    ExpiresAt         time.Time
}

type Repository interface {
    // Upsert inserts or replaces the connection, status=connected, connected_at
    // set on first insert (preserved on update), updated_at bumped. Reconnect
    // from revoked/error flips status back to connected.
    Upsert(ctx context.Context, userID string, whoopUserID int64, tokens TokenBundle, scopes string, now time.Time) error
    Get(ctx context.Context, userID string) (*Connection, error)
    GetByWhoopUserID(ctx context.Context, whoopUserID int64) (*Connection, error) // webhook routing
    GetTokens(ctx context.Context, userID string) (*TokenBundle, error)
    // UpdateTokens persists a rotated access+refresh pair + expiry (refresh flow).
    UpdateTokens(ctx context.Context, userID string, tokens TokenBundle, now time.Time) error
    SetStatus(ctx context.Context, userID string, status Status, now time.Time) error
    // Revoke marks status=revoked and wipes token columns (disconnect).
    Revoke(ctx context.Context, userID string, now time.Time) error
    Exists(ctx context.Context, userID string) (bool, error)
}
```
`ErrNotFound = errors.New("whoopconn: not found")`. Upsert uses `ON CONFLICT(user_id) DO UPDATE` for connection + token columns (whoop_user_id updated too — a reconnect could theoretically re-fetch it). `Revoke` sets `status='revoked'` and writes empty BLOBs / zero expiry to the token columns; token columns are `NOT NULL` so use `X''` (empty blob) — acceptable because reads of a revoked row never decrypt. `GetByWhoopUserID` returns `ErrNotFound` when absent.

- [ ] **Step 1:** Write `repository_shared_test.go` contract exercising Upsert→Get, GetByWhoopUserID, GetTokens round-trip, UpdateTokens, SetStatus, Revoke (tokens wiped, status revoked), reconnect flips status, Exists, ErrNotFound paths.
- [ ] **Step 2:** Implement `connection.go` (types), `errors.go`, `repository.go` (interface), `sqlite_repository.go`.
- [ ] **Step 3:** `sqlite_repository_test.go` runs the contract against a migrated DB (mirror calendarconn's `newMigratedDB`/`dbtest`).
- [ ] **Step 4:** `go test ./internal/whoopconn/...` PASS; lint clean.
- [ ] **Step 5:** Commit `feat(whoop): add whoopconn connection repository`.

---

## Task 4 (API): `internal/whooprecovery` daily recovery repository

**Files:**
- Create: `internal/whooprecovery/recovery.go`, `repository.go`, `sqlite_repository.go`, `errors.go`, `repository_test.go`

**Spec:** Mirror `internal/steps` (date-keyed daily upsert). 

```go
type Entry struct {
    ID               string
    UserID           string
    Date             string  // YYYY-MM-DD local
    RecoveryScore    *float64
    RestingHeartRate *float64
    HRVRmssdMilli    *float64
    CycleID          int64
    SleepID          string
    CreatedAt        time.Time
    UpdatedAt        time.Time
}

type Repository interface {
    // Upsert replaces the row for (user, date) — latest wins. ID generated on
    // insert, preserved on conflict; created_at preserved, updated_at bumped.
    Upsert(ctx context.Context, e Entry, now time.Time) error
    // ListRange returns rows with date in [since, until] (inclusive, either may
    // be ""), newest-first.
    ListRange(ctx context.Context, userID, since, until string) ([]Entry, error)
    // DeleteBySleepID removes the row whose sleep_id matches (webhook delete).
    // No error when absent.
    DeleteBySleepID(ctx context.Context, userID, sleepID string) error
}
```
Nullable metrics stored as SQL NULL when the pointer is nil. ID generation must match the repo's existing id scheme (check how `steps` generates ids — likely a shared `id`/`ulid` helper; reuse it). Upsert: `INSERT … ON CONFLICT(user_id,date) DO UPDATE SET recovery_score=excluded…, resting_heart_rate=excluded…, hrv_rmssd_milli=excluded…, cycle_id=excluded…, sleep_id=excluded…, updated_at=excluded.updated_at` (preserve id + created_at).

- [ ] **Step 1:** Write `repository_test.go`: Upsert insert then re-upsert same date overwrites (one row, latest values, created_at preserved), ListRange window filtering + ordering, nullable metrics round-trip (nil ↔ NULL), DeleteBySleepID removes matching row and is a no-op when absent.
- [ ] **Step 2:** Implement the package.
- [ ] **Step 3:** `go test ./internal/whooprecovery/...` PASS; lint clean.
- [ ] **Step 4:** Commit `feat(whoop): add whooprecovery daily metric repository`.

---

## Task 5 (API): Whoop v2 client + OAuth helpers (`internal/whoopsync`)

**Files:**
- Create: `internal/whoopsync/oauth.go` (state encode/decode — reuse the calendar HMAC-bound CSRF scheme verbatim; authorize-URL builder; token exchange + refresh; revoke), `client.go` (Whoop v2 REST: profile, recovery+cycle paging), `doc.go` (package doc)
- Create: `internal/whoopsync/oauth_test.go`, `client_test.go`

**Spec:** 
OAuth (`oauth.go`) — copy calendar's `encodeState`/`decodeState`/`stateSignature` (HMAC-SHA256 over `random:userID`, constant-time verify). Constants:
- Authorize URL `https://api.prod.whoop.com/oauth/oauth2/auth`; token URL `https://api.prod.whoop.com/oauth/oauth2/token`; scopes `read:recovery read:profile offline`.
- Use `golang.org/x/oauth2` with an explicit `Endpoint{AuthURL, TokenURL}` (both overridable in tests via a struct field so `oauth_test.go` can point at an `httptest.Server`). `AuthCodeURL(state)` — Whoop returns a refresh token when `offline` scope is present (no `prompt=consent` needed, but harmless to omit).
- `exchangeCode(ctx, code) (*Tokens, error)` returns access, refresh, expiry, scopes. Refresh token MUST be non-empty (`offline` granted) else `ErrNoRefreshToken`.
- `refreshTokens(ctx, refreshToken) (*Tokens, error)` POSTs `grant_type=refresh_token` (Whoop rotates: response carries a NEW refresh token — return it). A `401`/`invalid_grant` maps to a sentinel `ErrInvalidGrant`.
- `revoke(ctx, accessToken) error` best-effort `DELETE https://api.prod.whoop.com/v2/user/access` with bearer.

Client (`client.go`) — injected `*http.Client` (caller owns timeout). Methods with bearer:
- `Profile(ctx, accessToken) (WhoopProfile{UserID int64, …}, error)` → `GET /v2/user/profile/basic`.
- `Recoveries(ctx, accessToken, start, end time.Time, limit int) ([]RecoveryRecord, error)` → paged `GET /v2/recovery` (`limit`, `nextToken`, `start`,`end` ISO). Follow `next_token` until exhausted or a page cap.
- `Cycles(ctx, accessToken, start, end time.Time, limit int) ([]CycleRecord, error)` → paged `GET /v2/cycle`.
- Wire shapes (v2): recovery record `{cycle_id int64, sleep_id string, score_state string, score:{recovery_score float, resting_heart_rate float, hrv_rmssd_milli float}}`; cycle record `{id int64, start string(RFC3339), timezone_offset string("-08:00")}`. Use `map`/structs with `json` tags; tolerate missing `score` (PENDING/UNSCORABLE). Classify non-2xx: `429`→`ErrRateLimited` (carry `Retry-After`), `401`→`ErrTokenRejected`, else generic error with status.

Types shared across client/service go in `client.go` (or a small `types.go`). Keep files focused.

- [ ] **Step 1 (oauth_test.go):** state round-trip (encode→decode returns same random+userID), tamper/truncate/bad-sig rejected; token exchange against httptest returns tokens incl. rotated refresh; refresh returns the NEW refresh token; `invalid_grant` → `ErrInvalidGrant`; empty refresh on exchange → `ErrNoRefreshToken`.
- [ ] **Step 2 (client_test.go):** Profile decodes user id; Recoveries follows `next_token` across two pages and stops; Cycles paging; 429→`ErrRateLimited`; bearer header present; malformed/absent `score` tolerated.
- [ ] **Step 3:** Implement `oauth.go`, `client.go`, `doc.go`.
- [ ] **Step 4:** `go test ./internal/whoopsync/...` PASS (client+oauth); lint clean (gosec: constant-time compare in decodeState).
- [ ] **Step 5:** Commit `feat(whoop): add Whoop v2 oauth + REST client`.

---

## Task 6 (API): Sync service — window sync, date derivation, refresh rotation (`internal/whoopsync/service.go`)

**Files:**
- Create: `internal/whoopsync/service.go`, `internal/whoopsync/service_test.go`

**Spec:** The `Service` orchestrates: load connection → get-or-refresh access token (persist rotated pair BEFORE use) → fetch recoveries + cycles over a window → join on `cycle_id` → derive local `date` → upsert SCORED rows.

```go
type Service struct {
    conns   whoopconn.Repository
    rec     whooprecovery.Repository
    cipher  *tokencrypt.Cipher
    client  whoopClient   // interface over client.go methods (for test fakes)
    oauth   tokenRefresher // interface over oauth refresh/exchange (for fakes)
    now     func() time.Time
    mu      keyedMutex     // per-user serialization (webhook vs backfill)
}
```
- `validToken(ctx, userID) (accessToken string, err error)`: Get tokens; if `token_expires_at` within skew (e.g. 2 min), call `oauth.refreshTokens`, **`conns.UpdateTokens` with the new encrypted pair before returning/using it**; on `ErrInvalidGrant` → `conns.SetStatus(error)` and return `ErrReconnectNeeded`. Serialize per-user via `mu` so concurrent webhook+backfill never double-refresh.
- `SyncWindow(ctx, userID string, limit int) error`: recoveries+cycles over `[now-window, now]`; build `map[cycleID]cycle`; for each recovery with `score_state=="SCORED"`, derive date and upsert; skip PENDING/UNSCORABLE.
- `Backfill(ctx, userID string) error`: same but 30-day window, higher page limit.
- **Date derivation** (pure, exported-for-test helper `deriveDate(cycleStart string, tzOffset string) (string, error)`): parse cycle `start` (RFC3339 UTC), apply `timezone_offset` (`"-08:00"`) to get local wall time, format `YYYY-MM-DD`. Correct across negative offsets and DST-adjacent dates (the offset is authoritative — no zone lookup needed).
- Idempotency: re-running `SyncWindow` with the same data yields the same single row per date (guaranteed by the repo's `UNIQUE(user_id,date)` upsert).

- [ ] **Step 1 (service_test.go, using fakes for client/oauth/repos):**
  - `deriveDate`: `("2026-01-15T12:00:00Z","-08:00")→"2026-01-15"`; a UTC morning that rolls back a day under a negative offset; a positive offset (`"+11:00"`) rolling forward; a DST-adjacent date.
  - SyncWindow upserts only SCORED records; PENDING/UNSCORABLE skipped.
  - Idempotency: same window synced 5× → one row per date.
  - Refresh ordering: when expiry is stale, `UpdateTokens` is called with the rotated pair **before** any client call uses the new token (assert call order via fake); `invalid_grant` → connection set to `error`, returns `ErrReconnectNeeded`.
- [ ] **Step 2:** Implement `service.go` (+ `keyedMutex` helper if not already present in the repo — check for an existing per-key mutex util first).
- [ ] **Step 3:** `go test ./internal/whoopsync/...` PASS; lint clean.
- [ ] **Step 4:** Commit `feat(whoop): add recovery window-sync service with token rotation`.

---

## Task 7 (API): Connection HTTP handlers + backfill kickoff (`internal/whoopsync/handler.go`)

**Files:**
- Create: `internal/whoopsync/handler.go`, `internal/whoopsync/handler_test.go`

**Spec:** Mirror `calendarsync/handler.go` split mount.

| Route | Mount | Behavior |
| --- | --- | --- |
| `GET /auth/whoop/connect` | authed | Build authorize URL with HMAC-bound state carrying userID; accept `?return_to`, same-origin-validated via `originmatch.AllowReturnTo`; set CSRF cookie (random half) like calendar; 302 to Whoop. |
| `GET /auth/whoop/callback` | public | Validate state (cookie random + signed userID); exchange code; fetch profile for `whoop_user_id`; encrypt both tokens; `conns.Upsert(connected)`; kick off `Backfill` inline best-effort (failure logs, connection stays usable); 302 to `return_to`. |
| `GET /me/whoop/connection` | authed | JSON `{status, connected_at}` (+ `whoop_user_id`? keep to status+connected_at like calendar). `absent` when no row. |
| `DELETE /me/whoop/connection` | authed | Best-effort `oauth.revoke`; `conns.Revoke` (status revoked, tokens wiped). Recovery rows untouched. 204. |

`Handler` struct fields: `oauthConfig`/endpoints, `conns whoopconn.Repository`, `cipher *tokencrypt.Cipher`, `client whoopClient`, `oauth` helpers, `svc *Service` (for Backfill), `httpClient`, `returnToAllowedOrigins []string`, `stateHMACKey []byte`. `MountPublic(r)` → callback; `MountAuthed(r)` → connect + connection GET/DELETE.

- [ ] **Step 1 (handler_test.go, chi router + injected-user middleware like calendar):**
  - connect: 302, `Location` host is `api.prod.whoop.com`, state present, CSRF cookie set; disallowed `return_to` → 400.
  - callback: valid state+code stores an encrypted `connected` connection with the profile's whoop_user_id, 302 to return_to; invalid/tampered state → error, no row; backfill failure still yields a usable connection (302).
  - GET connection: returns status/connected_at; `absent` when none.
  - DELETE: calls revoke, marks revoked, wipes tokens, 204.
- [ ] **Step 2:** Implement `handler.go`.
- [ ] **Step 3:** `go test ./internal/whoopsync/...` PASS; lint clean.
- [ ] **Step 4:** Commit `feat(whoop): add connection oauth + status/disconnect handlers`.

---

## Task 8 (API): Webhook handler (`internal/whoopsync/webhook.go`)

**Files:**
- Create: `internal/whoopsync/webhook.go`, `internal/whoopsync/webhook_test.go`

**Spec:** `POST /webhooks/whoop`, mounted **public**.
1. Read raw body (bounded via `http.MaxBytesReader`). Verify: `X-WHOOP-Signature == base64(HMAC-SHA256(X-WHOOP-Signature-Timestamp + rawBody, client_secret))`, constant-time (`hmac.Equal`); timestamp (ms) within ±5 min. Fail → 401 + log.
2. Parse `{user_id int64, type string, id string(sleep uuid), …}`. Look up connection by `user_id` with status `connected`; unknown/non-connected → 204 (drop; never fail an event we don't want).
3. `type=="recovery.updated"` → `svc.SyncWindow(userID, ~10)`. `type=="recovery.deleted"` → `rec.DeleteBySleepID(userID, event.id)`. Any other type → 204.
4. Run inline, return 2xx.

`Handler` gets `webhookSecret []byte` (the Whoop client secret) + `conns` + `rec` + `svc` + `now`. Mount on the public router.

- [ ] **Step 1 (webhook_test.go):**
  - valid signature + `recovery.updated` for a connected user → triggers SyncWindow (fake svc records call), 2xx.
  - tampered body / bad signature / stale timestamp → 401, no sync.
  - constant-time path exercised (valid vs invalid sig both compared).
  - unknown whoop user → 204, no sync.
  - `recovery.deleted` → DeleteBySleepID with the event `id`.
  - idempotency: same delivery 5× → SyncWindow idempotent (assert repo yields one row via an integration-style fake, or assert SyncWindow called 5× is safe).
- [ ] **Step 2:** Implement `webhook.go`.
- [ ] **Step 3:** `go test ./internal/whoopsync/...` PASS; lint clean (gosec G120 — bound body).
- [ ] **Step 4:** Commit `feat(whoop): add HMAC-verified recovery webhook handler`.

---

## Task 9 (API): Read endpoint + dashboard recovery block

**Files:**
- Create: `internal/whoopsync/read_handler.go` (or extend handler.go) + test — `GET /whoop/recovery`
- Modify: `internal/dashboard/dto.go`, `internal/dashboard/handler.go`; Create `internal/dashboard/whoop.go` + test

**Spec:**
- `GET /whoop/recovery?timezone=&since=&until=` (authed): use `daterange`/house convention to resolve the local-date window, call `rec.ListRange`, return `{data: [dayRows ordered by date]}`. No connection → empty list.
- Dashboard summary grows a `recovery` block, present only when a `connected` whoop connection exists: `{today: {date, resting_heart_rate, recovery_score, hrv_rmssd_milli}|null, resting_hr_spark: [last 7 days resting HR]}`. Absence hides the web card. Wire a `whoopConnRepo`/`whoopRecoveryRepo` into the dashboard `Handler` (add constructor params) and a pure `buildWhoop(...)` builder mirroring `buildSteps`. Reads degrade to nil on error (don't 500).

- [ ] **Step 1:** Tests: read endpoint returns ordered window rows + empty list when none; `buildWhoop` returns nil when no connection, populated block with today + 7-day spark when connected.
- [ ] **Step 2:** Implement read handler + dashboard block + DTO.
- [ ] **Step 3:** `go test ./...` PASS; lint clean.
- [ ] **Step 4:** Commit `feat(whoop): add recovery read endpoint and dashboard block`.

---

## Task 10 (API): Config `[whoop]` + server wiring

**Files:**
- Modify: `config.toml`, `internal/config/config.go`, `internal/server/server.go`

**Spec:**
- `config.toml` — new top-level section:
```toml
[whoop]
# OAuth client id: deployment-bound identifier (not a hard secret) → env.
client_id = "${WHOOP_CLIENT_ID}"
# OAuth client secret (also the webhook HMAC key) → secret → env.
client_secret = "${WHOOP_CLIENT_SECRET}"
# Public OAuth callback URL — NOT a secret. Prod literal; local dev via env.
redirect_url = "https://api.progstrength.fitness/auth/whoop/callback"
# base64 32-byte AES-256-GCM key encrypting stored Whoop tokens. Secret → env.
# Empty disables the Whoop integration.
token_enc_key = "${WHOOP_TOKEN_ENC_KEY}"
```
- `config.go` — add `WhoopClientID/WhoopClientSecret/WhoopRedirectURL/WhoopTokenEncKey` to `Config`, a `Whoop` sub-struct to `fileConfig` (`toml:"whoop"`), `interp()` the four, map into `Config`, and add env overrides mirroring the calendar ones (e.g. `WHOOP_REDIRECT_URL`).
- `server.go` — mirror the calendar DI block: enable Whoop only when `WhoopClientID && WhoopClientSecret && WhoopTokenEncKey && WhoopRedirectURL` are set; build `tokencrypt` cipher from `WhoopTokenEncKey`, whoopconn/whooprecovery repos, client (`&http.Client{Timeout: 8s}`), oauth config, service, handler, webhook handler; `MountPublic` (callback + `/webhooks/whoop`) outside auth, `MountAuthed` (connect + `/me/whoop/connection` + `/whoop/recovery`) inside the JWT group. Pass the whoop repos into the dashboard handler constructor. Log `whoop: enabled/disabled`.

- [ ] **Step 1:** Add config + a config test (env interpolation resolves `${WHOOP_*}`; missing disables).
- [ ] **Step 2:** Wire `server.go`. Add a server smoke test if the repo has one for calendar (route mounted when configured).
- [ ] **Step 3:** `go build ./... && go vet ./... && go test ./... && golangci-lint run` all green; `go mod tidy` no drift.
- [ ] **Step 4:** Commit `feat(whoop): wire config and server routes`.

---

## Task 11 (Web): API client + types

**Files:** Modify `lib/api.ts`

**Spec:** Add (mirror `getCalendarConnection`/`disconnectCalendar`):
```ts
export type WhoopConnection = {
  status: "connected" | "revoked" | "error" | "absent";
  connected_at?: string;
};
export async function getWhoopConnection(token: string): Promise<WhoopConnection> // GET /me/whoop/connection, empty → {status:"absent"}
export async function disconnectWhoop(token: string): Promise<void>                // DELETE /me/whoop/connection
```
Extend `DashboardSummary` with `recovery: DashboardRecovery | null` and a `DashboardRecovery` type `{ today: { resting_heart_rate: number|null; recovery_score: number|null; hrv_rmssd_milli: number|null } | null; resting_hr_spark: number[] }`. Match the API JSON shape from Task 9 exactly.

- [ ] Steps: implement, `npm run typecheck && npm run lint && npm run test`, commit `feat(whoop): add web api client for whoop connection + recovery`.

---

## Task 12 (Web): Settings Profile/Integrations tabs + Whoop card

**Files:** Modify `app/(app)/settings/page.tsx`, add `app/(app)/settings/_components/whoop-connection-row.tsx` (+ test), possibly a small `tabs` piece in `_components`.

**Spec:**
- Add a two-tab header **Profile** / **Integrations** driven by `?tab=` (default `profile`), URL-addressable so `return_to=/settings?tab=integrations` round-trips. Use `useSearchParams`/`router.replace` (Next 16 App Router). Tab affordance conforms to the design system segmented-control idiom (accent-soft fill + accent-line border on active — reuse the existing pill/segmented pattern; **do not invent new tokens**).
- Profile tab: existing profile card + SaveBar (draft mechanics stay scoped to Profile).
- Integrations tab: the existing Google Calendar card (moved unchanged) + a new Whoop card.
- `WhoopConnectionRow` mirrors `GoogleCalendarConnectionRow`: disconnected → **Connect Whoop** navigates to `${config.apiUrl}/auth/whoop/connect?return_to=${origin}/settings?tab=integrations`; connected → status line with connected-since + **Disconnect** (immediate DELETE, confirm affordance consistent with calendar); `error` status → explanatory line + **Reconnect** (same connect nav). Use design-system tokens only.

- [ ] **Step 1 (whoop-connection-row.test.tsx):** renders disconnected (Connect), connected (status + Disconnect), error (Reconnect) states; disconnect calls API then reloads.
- [ ] **Step 2:** Add a settings test for the tab param round-trip (default profile; `?tab=integrations` shows Integrations).
- [ ] **Step 3:** Implement. `npm run typecheck && lint && test` green.
- [ ] **Step 4:** Commit `feat(whoop): settings integrations tab + whoop connection card`.

---

## Task 13 (Web): Dashboard recovery mini-card

**Files:** Create `app/(app)/dashboard/_components/whoop-card.tsx` (+ test); wire into `app/(app)/dashboard/page.tsx`.

**Spec:** A recovery `MiniCard` following dashboard grammar: `Kpi`/BigNum for today's resting HR (with a "no data yet today" empty state), a 7-day `Spark` of resting HR, recovery score as a secondary line. Rendered **only** when the summary contains the `recovery` block (connected users). Deep-link target is `/settings?tab=integrations`. Design-system tokens only; resting-HR is a metric, not an activity — do not use `--accent` as a data series (use `--muted`/foreground or a neutral spark stroke).

- [ ] **Step 1 (whoop-card.test.tsx):** renders today's resting HR + spark when present; "no data yet today" when today null but block present; card omitted when block absent.
- [ ] **Step 2:** Implement + wire conditional render in dashboard page.
- [ ] **Step 3:** `npm run typecheck && lint && test && build` green.
- [ ] **Step 4:** Commit `feat(whoop): dashboard recovery mini-card`.

---

## Task 14 (MCP): `whoop` module + client method

**Files:** Create `src/prog_strength_mcp/whoop.py`, modify `src/prog_strength_mcp/api_client.py`, `src/prog_strength_mcp/server.py`; create `tests/test_whoop_tools.py`.

**Spec:** Mirror `steps.py`. One tool `get_whoop_recovery(timezone, since=None, until=None)` forwarding the bearer via `_auth_header_or_raise()`, calling `api.get_whoop_recovery(auth, timezone=…, since=…, until=…)` → `GET /whoop/recovery`. Tool description explains metrics (recovery score 0–100, resting HR bpm, HRV ms) and that an empty result for a user who mentions Whoop likely means the integration isn't connected → point to Settings → Integrations. `api_client.py`: `get_whoop_recovery` builds params `{timezone, since?, until?}`, passes `headers={"Authorization": auth}`, `_raise_for_status`, returns `data` dict/list. Register in `server.py` (`import whoop`; `whoop.register(mcp, api)`).

- [ ] **Step 1 (tests, respx + monkeypatch pattern from test_steps_tools):** client forwards auth + params and returns data; missing auth → RuntimeError before HTTP; APIError → RuntimeError with status.
- [ ] **Step 2:** Implement. `ruff check . && ruff format --check . && pytest` green.
- [ ] **Step 3:** Commit `feat(whoop): add get_whoop_recovery MCP tool`.

---

## Task 15 (Infra): Whoop secrets

**Files:** Modify `.github/workflows/seed-secrets.yml`, `deploy/api.sh`, `compose/api/docker-compose.yml`, `compose/api/config.env`.

**Spec:** Add `WHOOP_CLIENT_SECRET` and `WHOOP_TOKEN_ENC_KEY` as secrets everywhere the calendar secrets (`GOOGLE_CLIENT_SECRET`, `CALENDAR_TOKEN_ENC_KEY`) appear; add `WHOOP_CLIENT_ID` as **non-secret config** where `GOOGLE_CLIENT_ID` appears (`config.env` + docker-compose + `deploy/api.sh` required list).
- `seed-secrets.yml` (api job): add `WHOOP_CLIENT_SECRET` and `WHOOP_TOKEN_ENC_KEY` to the `env:` block, the `jq --arg` list, and the JSON payload. Treat them like the other **optional** provider secrets (the integration self-disables when absent) — i.e. seeded via `with_entries(select(.value!=""))` but **not** added to the `required_secrets` gate, so a not-yet-configured Whoop app doesn't block a deploy. (Rationale mirrors the SOW: empty `token_enc_key`/client disables Whoop; the rest of the API is unaffected.)
- `compose/api/docker-compose.yml`: add `- WHOOP_CLIENT_ID=${WHOOP_CLIENT_ID:-}`, `- WHOOP_CLIENT_SECRET=${WHOOP_CLIENT_SECRET:-}`, `- WHOOP_TOKEN_ENC_KEY=${WHOOP_TOKEN_ENC_KEY:-}` near the calendar block, with a short comment.
- `compose/api/config.env`: add `WHOOP_CLIENT_ID=` (leave value blank — owner fills at rollout; matches the "deployment-bound, env-supplied" note; do NOT invent a client id).
- `deploy/api.sh`: do **not** add Whoop keys to `REQUIRED_ENV_KEYS` (optional until the owner sets up the Whoop app), matching the seed workflow's non-gated treatment.

- [ ] **Step 1:** Make the four edits.
- [ ] **Step 2:** `pre-commit run --all-files` (fmt/shellcheck/yaml) if available; else `terraform fmt -check` is N/A (no .tf changes). Validate yaml parses.
- [ ] **Step 3:** Commit `feat(secrets): add Whoop integration secrets to api service`.

---

## Task 16 (Docs): SOW status flip

Handled by the orchestrator (not a subagent): set `status: shipped` in frontmatter, `**Status**: Shipped`, `**Last updated**: 2026-07-22`, commit `docs: mark whoop-integration as shipped` on `feat/whoop-integration`, and open the templated PR.

---

## Self-Review

- **Spec coverage:** connection lifecycle (T3,T7), recovery persistence (T4), webhook push ingestion (T8), backfill (T6/T7), settings tabs + card (T12), dashboard card (T13), MCP tool (T14), tokencrypt extraction with own key (T1,T10), graceful refresh-failure → `error` status + Reconnect (T6,T7,T12), config/secrets (T10,T15), tests (each task). Non-goals (sleep/strain, other providers, recovery page, scheduler, mobile, disconnect-preserves-data) are respected — no tasks build them; `Revoke` leaves recovery rows untouched (T3/T7).
- **Type consistency:** `whoopconn.Status{Connected,Revoked,Error}`, `TokenBundle`, `whooprecovery.Entry` (pointer metrics), `Service.SyncWindow/Backfill/deriveDate`, web `WhoopConnection.status ∈ {connected,revoked,error,absent}`, `DashboardRecovery.resting_hr_spark` used consistently across API DTO (T9) and web type (T11) and card (T13).
- **Migration numbers:** 039/040 (038 taken) — flagged in the API PR.
- **Placeholders:** none — each task names exact files, signatures, routes, and test cases.
