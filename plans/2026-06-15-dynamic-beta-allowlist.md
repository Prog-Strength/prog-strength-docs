# Implementation Plan: Dynamic Beta Allowlist

**SOW**: `sows/dynamic-beta-allowlist.md`
**Date**: 2026-06-15
**Repos touched**: `prog-strength-api`, `prog-strength-infra`, `prog-strength-docs`

## Summary

Move the beta-tester allowlist out of the boot-time `BETA_ALLOWED_EMAILS`
env var and into the SQLite database, and add an admin-only HTTP surface
(`/admin/beta-emails`) for managing it. The OAuth gate stops consulting a
frozen in-memory map and instead does an indexed lookup per login. A
one-time, idempotent boot seed carries the live env-var list into the
table so the cutover is zero-touch. No change to the gate's externally
observable behavior.

## Important deviation from the SOW text

The SOW (written 2026-06-14) says migrations are "currently through `018`"
and names the new migration `019_beta_allowed_emails.sql`. Since the SOW
was written, `019_steps.sql` and `020_timeline.sql` have landed on `main`.
The next free migration number is **`021`**. This plan uses
**`internal/db/migrations/021_beta_allowed_emails.sql`**. The SOW's intent
("the next migration") is preserved; only the number differs. Same for the
backfill gating note ("after migration 019 ships" → after 021).

## Architecture notes (verified against the codebase)

- Repository pattern: `internal/user/` has `repository.go` (interface),
  `user.go` (model), `sqlite_repository.go`, `memory_repository.go`,
  `errors.go`, with compile-time `var _ Repository = (*X)(nil)` asserts and
  an injectable `now func() time.Time` for clock control in tests. Mirror
  this exactly for `internal/beta`.
- Migrations are embedded (`//go:embed migrations/*.sql`), forward-only, no
  down files, applied in numeric order by `internal/db/migrate.go`; tracked
  in `schema_migrations`. Header convention: `-- migrations/NNN_*.sql` then
  a prose comment block. Use `CREATE TABLE IF NOT EXISTS`.
- Router is `go-chi/chi/v5`. JWT-gated routes are mounted inside
  `r.Group(func(r chi.Router){ r.Use(auth.RequireUser(jwtSecret)); ... })`
  in `internal/server/server.go`. Each domain has `NewHandler(...).Mount(r)`.
- Responses use the `internal/httpresp` helpers (`OK`, `Created`, `Error`,
  `ServerError`). Path params via `chi.URLParam(r, "...")`. User id via
  `authctx.UserIDFrom(ctx)`.
- Config: `internal/config/config.go` `Load()` reads env vars; `splitCSV`
  trims/drops empties and returns `nil` when empty. `BetaAllowedEmails` is
  already `splitCSV(os.Getenv("BETA_ALLOWED_EMAILS"))`.
- `auth.NewHandler(auth.Config{...}, userRepo)` is constructed in
  `server.go`; `auth.Config` currently carries `BetaAllowedEmails`, which
  `NewHandler` freezes into `betaAllowedEmails map[string]struct{}`, read by
  `isBetaAllowed` in the `googleCallback` gate.
- Backfill precedent: idempotent methods gated on an empty-table check
  (`BackfillOneRepMaxHistory`, `BackfillActivityBestEfforts`,
  `backfillTimeline`) invoked in `server.New` after `db.Migrate`, using
  `context.Background()` and logging a one-line summary.
- `bodyweight.NewHandler(bodyweightRepo, userRepo)` is precedent for a
  domain handler taking the user repo — the beta handler will do the same
  to resolve the calling admin's email for `added_by`.

## Dependency direction (no import cycles)

- `auth` imports `beta` only for a tiny `beta.Checker` interface
  (`IsAllowed(ctx, email) (bool, error)`) that `beta.Repository` satisfies.
- `beta` does **not** import `auth`. The beta HTTP handler reads the caller
  id via `authctx.UserIDFrom` and resolves the email via `user.Repository`.
- `auth.RequireAdmin` lives in `internal/auth` (it already imports `user`).

---

## Tasks

### Task 1 — Migration `021_beta_allowed_emails.sql`
Create `internal/db/migrations/021_beta_allowed_emails.sql`:

```sql
-- migrations/021_beta_allowed_emails.sql
-- Dynamic beta allowlist: move the closed-beta email gate out of the
-- BETA_ALLOWED_EMAILS env var (frozen at boot) and into the database so a
-- tester can be authorized with one admin API call — no secret edit, no
-- redeploy. See prog-strength-docs/sows/dynamic-beta-allowlist.md.
--
-- email is normalized (lower-cased, trimmed) before storage; the PRIMARY
-- KEY gives O(log n) IsAllowed lookups and enforces dedup. added_by is the
-- admin email that added the row, or the sentinel
-- "seed:BETA_ALLOWED_EMAILS" for rows carried over by the one-time boot
-- seed (nullable to accommodate it). note is an optional free-text label.

CREATE TABLE IF NOT EXISTS beta_allowed_emails (
    email     TEXT PRIMARY KEY,
    added_at  DATETIME NOT NULL,
    added_by  TEXT,
    note      TEXT
);
```

**Verify**: file present; `go build ./...` still compiles (migration is
embedded, applied at boot).

### Task 2 — `internal/beta` package: model, interface, repos
Create:
- `internal/beta/beta.go` — `AllowedEmail` model:
  ```go
  type AllowedEmail struct {
      Email   string    `json:"email"`
      AddedAt time.Time `json:"added_at"`
      AddedBy *string   `json:"added_by"`
      Note    *string   `json:"note"`
  }
  ```
  and a `Checker` interface with the single method
  `IsAllowed(ctx context.Context, email string) (bool, error)`.
  Add a `normalizeEmail(email string) string` =
  `strings.ToLower(strings.TrimSpace(email))` (package-private helper).
  Define the sentinel `const SeedAddedBy = "seed:BETA_ALLOWED_EMAILS"`.
- `internal/beta/repository.go` — `Repository` interface:
  ```go
  type Repository interface {
      IsAllowed(ctx context.Context, email string) (bool, error)
      Add(ctx context.Context, email, addedBy, note string) error
      Remove(ctx context.Context, email string) (bool, error)
      List(ctx context.Context) ([]AllowedEmail, error)
  }
  ```
  (`Repository` embeds/contains `Checker`'s method.) Add
  `var _ Repository = (*SQLiteRepository)(nil)` etc.
- `internal/beta/errors.go` if any sentinels are needed (mirror user).
- `internal/beta/sqlite_repository.go` — `SQLiteRepository{db, now}`,
  `NewSQLiteRepository(db)`:
  - `IsAllowed`: if table empty → `(true, nil)`. Emptiness check must be
    cheap: `SELECT EXISTS(SELECT 1 FROM beta_allowed_emails)`. If non-empty,
    `SELECT EXISTS(SELECT 1 FROM beta_allowed_emails WHERE email = ?)` with
    the normalized email. (A single query is acceptable too, but the
    "empty ⇒ allow-all" branch must short-circuit; keep it cheap, no full
    scan.)
  - `Add`: normalize; `INSERT OR IGNORE INTO beta_allowed_emails(email,
    added_at, added_by, note) VALUES(?, ?, ?, ?)` with `added_at = now()
    UTC`. Empty `addedBy`/`note` → store SQL NULL (use `sql.NullString` or
    `nullable(s)` helper). Idempotent: re-adding existing email is not an
    error and does not duplicate (OR IGNORE keeps the original row).
  - `Remove`: normalize; `DELETE ...`; return `(RowsAffected>0, nil)`.
  - `List`: `SELECT ... ORDER BY added_at ASC`; scan into `[]AllowedEmail`.
- `internal/beta/memory_repository.go` — `MemoryRepository` with
  `sync.RWMutex`, `map[string]AllowedEmail` keyed by normalized email,
  injectable `now`. Same semantics, including empty-map ⇒ `IsAllowed` true,
  idempotent `Add`, `Remove` returns presence, `List` sorted by `AddedAt`
  asc (stable). Defensive copies on the way out.

**Verify**: `go build ./...`, `go vet ./internal/beta/...`.

### Task 3 — `internal/beta` repository tests
`internal/beta/sqlite_repository_test.go` + `memory_repository_test.go`
(or one table-driven file exercising both via the interface, mirroring how
`internal/user` tests are structured — use a `newSQLiteBetaRepo(t)` helper
with `db.Open(t.TempDir()/...)` + `db.Migrate`). Cases per SOW § Tests:
- add then `IsAllowed` true; `IsAllowed` false for absent email;
- case/whitespace insensitivity (`"Foo@Bar.com "` matches `foo@bar.com`);
- `Add` idempotency (second add no error, no dup — assert `List` len);
- `Remove` returns `true` then `false`;
- `List` ordering by `added_at` asc (use injected `now` to stagger);
- **empty-table ⇒ `IsAllowed` returns `true` for any email** (both backends).

### Task 4 — Seed-on-boot backfill
Add `SeedFromEnv(ctx context.Context, emails []string) (int, error)` to
`internal/beta/sqlite_repository.go` (method on `*SQLiteRepository`):
- If `emails` is empty → return `(0, nil)` (no-op).
- Cheap empty-table check (`SELECT EXISTS...`); if **non-empty**, return
  `(0, nil)` — never overwrite admin edits or re-seed.
- Else `INSERT OR IGNORE` each normalized email with
  `added_by = SeedAddedBy`, `added_at = now()`. Return count attempted/
  inserted for the caller to log.

Tests in `internal/beta/...`:
- empty table + non-empty env → rows seeded with sentinel `added_by`;
- non-empty table → left untouched (no overwrite, returns 0);
- empty env → no-op;
- second boot does not re-seed/duplicate (call twice, assert stable).

### Task 5 — Config: `ADMIN_EMAILS`
In `internal/config/config.go`: add `AdminEmails []string` to `Config` and
`AdminEmails: splitCSV(os.Getenv("ADMIN_EMAILS"))` in `Load()`. Mirror the
existing `BetaAllowedEmails` line exactly. Add a config test if the package
has one for env parsing (only if a parallel test exists; otherwise skip).

### Task 6 — Auth: swap gate to `beta.Checker` + `RequireAdmin`
In `internal/auth`:
- `handler.go`: remove the `betaAllowedEmails map[string]struct{}` field and
  the boot-time map build in `NewHandler`; remove `isBetaAllowed`'s map
  lookup. Add a `betaChecker beta.Checker` field. Change `NewHandler`'s
  signature to accept the checker — preferred:
  `NewHandler(cfg Config, users user.Repository, betaChecker beta.Checker)`.
  Remove `BetaAllowedEmails` from `auth.Config`.
  Replace `isBetaAllowed(email)` with a call to
  `h.betaChecker.IsAllowed(r.Context(), u.Email)` in `googleCallback`.
  On checker error → `httpresp.ServerError(...)` (do **not** mint a token on
  error — fail closed for the gate). On `false` → existing
  `redirectBetaRequired` / `403` path, unchanged. On `true` → existing token
  path, unchanged.
- `middleware.go`: add
  `func RequireAdmin(users user.Repository, adminEmails []string) func(http.Handler) http.Handler`.
  Behavior: read `authctx.UserIDFrom(ctx)`; if absent → `403` (fail-closed
  on misconfiguration — there is no user even though it sits behind
  `RequireUser`). If `adminEmails` is empty → `403` for everyone
  (fail-closed). Resolve user via `users.GetByID`; if not found → `403`.
  Compare normalized user email against the normalized `adminEmails` set;
  not a member → `403 forbidden`. On pass → `next.ServeHTTP`. Build the
  normalized admin set once per middleware construction (closure), not per
  request.

**Verify**: `go build ./...`. Note this changes `auth.NewHandler`'s
signature — Task 8 (server wiring) updates the call site; until then the
build of `internal/server` will fail, which is expected mid-stream. Run
`go build ./internal/auth/... ./internal/beta/...` to verify these packages.

### Task 7 — Auth tests: `RequireAdmin` + gate-via-checker
`internal/auth/middleware_test.go` (extend): admin email in `ADMIN_EMAILS`
passes; non-admin → `403`; empty `ADMIN_EMAILS` → deny everyone; no user in
context → deny. Use an in-memory `user.MemoryRepository` seeded with users.
If existing handler tests construct `NewHandler`, update them to pass a
`beta.MemoryRepository` (or a tiny stub checker). Add/adjust a gate test:
empty beta repo (allow-all) mints token; non-empty repo without the email
redirects `#error=beta_required`.

### Task 8 — Server wiring + seed invocation + mount admin routes
In `internal/server/server.go`:
- Instantiate `betaRepo` in both SQLite and in-memory branches
  (`beta.NewSQLiteRepository(database)` / `beta.NewMemoryRepository()`),
  alongside the other repos.
- In the SQLite branch, after `db.Migrate` and alongside the other
  backfills, call
  `n, err := betaRepo.(*beta.SQLiteRepository).SeedFromEnv(context.Background(), cfg.BetaAllowedEmails)`;
  on err return it; `log.Printf("seeded %d beta allowed email(s) from BETA_ALLOWED_EMAILS", n)`
  (only log when n>0, matching the other backfills' quiet no-op style — a
  single summary line is fine).
- Update the `auth.NewHandler(auth.Config{...}, userRepo)` call: drop
  `BetaAllowedEmails` from the `auth.Config` literal and pass `betaRepo` as
  the new third arg.
- Inside the existing JWT-gated `r.Group`, mount the admin routes:
  `beta.NewHandler(betaRepo, userRepo).Mount(r)` (the handler internally
  wraps its routes with `auth.RequireAdmin(userRepo, cfg.AdminEmails)` — see
  Task 9). Keep `auth.RequireUser` provided by the enclosing group.

**Verify**: `go build ./...` now succeeds again.

### Task 9 — `internal/beta` HTTP handler + tests
`internal/beta/handler.go`:
- `Handler{repo Repository, users user.Repository, adminEmails []string}`;
  `NewHandler(repo Repository, users user.Repository, adminEmails []string)`.
  Wait — to avoid the handler re-importing the admin gate logic, mount as:
  ```go
  func (h *Handler) Mount(r chi.Router) {
      r.Group(func(r chi.Router) {
          r.Use(auth.RequireAdmin(h.users, h.adminEmails))
          r.Get("/admin/beta-emails", h.list)
          r.Post("/admin/beta-emails", h.add)
          r.Delete("/admin/beta-emails/{email}", h.remove)
      })
  }
  ```
  This means `beta` imports `auth` for `RequireAdmin`, while `auth` imports
  `beta` for `Checker` → **import cycle**. Resolve by mounting the admin
  middleware in `server.go` instead: keep `beta.Handler` free of `auth`,
  and in `server.go` wrap the beta routes:
  ```go
  r.Group(func(r chi.Router) {
      r.Use(auth.RequireAdmin(userRepo, cfg.AdminEmails))
      beta.NewHandler(betaRepo, userRepo).Mount(r)
  })
  ```
  So `beta.Handler.Mount` registers the three routes **without** the admin
  middleware (the enclosing group supplies it), and `beta.NewHandler(repo,
  users)` takes just repo + user repo (for resolving the caller's email for
  `added_by`). This is the chosen design — no cycle.
- Routes & semantics (§ API Surface):
  - `GET /admin/beta-emails` → `200` `{ "emails": [ {email, added_at,
    added_by, note} ] }` sorted by `added_at` asc (repo already sorts).
  - `POST /admin/beta-emails` body `{ "email": "...", "note": "optional" }`.
    Validate non-empty email (after normalize) → else `400`. Resolve caller
    email via `users.GetByID(authctx.UserIDFrom(ctx))` for `added_by`.
    Determine pre-existence (e.g. `IsAllowed`/`List`) to choose status:
    `201` if newly added, `200` if already present (idempotent). Call
    `repo.Add`. Return the created/existing row.
  - `DELETE /admin/beta-emails/{email}` → URL-decode + normalize the path
    param; `repo.Remove`; `204` if removed, `404` if not present.
  - Non-admin `403` is enforced by the enclosing `RequireAdmin` group, so
    handler tests for the `403` path exercise `RequireAdmin` (Task 7) or are
    written as integration tests through a mounted router.
- `internal/beta/handler_test.go`: add→list round-trip; idempotent re-add
  (second POST → `200`, list len unchanged); delete then delete-again
  (`204` then `404`); malformed/empty email → `400`. Drive handlers with
  `httptest` + `authctx.WithUserID`, seeding a user in a
  `user.MemoryRepository` so `added_by` resolves. For the non-admin `403`
  matrix, mount a chi router with `RequireAdmin` and assert each verb 403s
  for a non-admin user.

**Verify**: `go build ./...`, `go test ./internal/beta/... ./internal/auth/...`.

### Task 10 — OAuth gate integration test
In `internal/auth` (or `internal/server` if an integration harness exists),
add a test proving the no-restart path end to end against the in-memory beta
repo: allowed email mints a token; disallowed redirected with
`#error=beta_required`; after `betaRepo.Add(...)` the same email's callback
succeeds. If a full OAuth callback harness is impractical to stand up,
assert via the `isBetaAllowed`-equivalent path (the handler calling
`betaChecker.IsAllowed`) that the decision flips after an `Add` — the SOW's
intent is to prove the dynamic (no-restart) behavior.

### Task 11 — Docs: README.md + AGENTS.md
- `prog-strength-api/README.md`: in the env-var table, change the
  `BETA_ALLOWED_EMAILS` row description to note it is now the **seed-only**
  source for the DB-backed allowlist (slated for removal), and add an
  `ADMIN_EMAILS` row ("Comma-separated operator emails; gates
  `/admin/beta-emails`. Empty = admin surface disabled (fail-closed).").
  Add a short subsection documenting the three `/admin/beta-emails`
  endpoints.
- `prog-strength-api/AGENTS.md`: update the "Beta gate" note(s) to describe
  the table-backed allowlist (`beta_allowed_emails`, migration 021), the
  `/admin/beta-emails` admin endpoints behind `ADMIN_EMAILS`, and that
  `BETA_ALLOWED_EMAILS` is now seed-only.

### Task 12 — Deploy wiring (api workflows)
In `prog-strength-api/.github/workflows/release.yml` and
`manual-deploy.yml`, add `ADMIN_EMAILS` in three places each, mirroring
`BETA_ALLOWED_EMAILS` exactly:
1. `env:` block: `ADMIN_EMAILS: ${{ secrets.ADMIN_EMAILS }}` (right after
   the `BETA_ALLOWED_EMAILS` line).
2. `with: envs:` comma-list: insert `ADMIN_EMAILS` right after
   `BETA_ALLOWED_EMAILS`.
3. `.env` heredoc: `ADMIN_EMAILS=${ADMIN_EMAILS}` right after the
   `BETA_ALLOWED_EMAILS=${BETA_ALLOWED_EMAILS}` line.

### Task 13 — Infra: docker-compose
In `prog-strength-infra/compose/api/docker-compose.yml`, add
`- ADMIN_EMAILS=${ADMIN_EMAILS}` to the api service `environment:` block,
directly after the existing `- BETA_ALLOWED_EMAILS=${BETA_ALLOWED_EMAILS}`
line.

---

## Final verification

- `cd /workspace/prog-strength-api && gofmt -l . && go build ./... && go vet ./... && go test ./...`
- Confirm migration applies cleanly (covered by repo tests that run
  `db.Migrate`).
- `prog-strength-infra`: YAML is structurally valid (compose env line only).

## Test plan (per SOW § Tests — all covered above)

- `internal/beta` repo (sqlite + memory): membership, case/whitespace
  insensitivity, idempotent Add, Remove presence toggle, List ordering,
  empty-table allow-all. (Tasks 3)
- Seed-on-boot: seed, no-overwrite, empty-env no-op, no re-seed. (Task 4)
- Admin middleware: admin passes, non-admin 403, empty list denies, no-user
  denies. (Task 7)
- Admin handler: add→list, idempotent re-add, delete then 404, malformed
  400, non-admin 403 every verb. (Task 9)
- OAuth gate: allowed mints, disallowed redirects, post-Add success without
  restart. (Task 10)

## Rollout (for the docs PR / operator)

Backward-compatible and self-seeding. Deploy order: **api first** (migration
creates the table; first boot seeds it from the still-present
`BETA_ALLOWED_EMAILS` secret — live list carries over, no window where the
gate is wrong), then **infra** compose change is only needed so the
container receives `ADMIN_EMAILS`. Set the `ADMIN_EMAILS` GitHub Actions
secret on `prog-strength-api` before/with the deploy; until set, admin
routes are inert (fail-closed) and the gate runs off the seeded table.
`BETA_ALLOWED_EMAILS` becomes vestigial after first boot; a follow-up
removes it.
