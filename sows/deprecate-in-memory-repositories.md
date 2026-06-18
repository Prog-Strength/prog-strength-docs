---
type: sow
status: draft
repos:
  - prog-strength-api
  - prog-strength-docs
---

# Deprecate the In-Memory Repository Layer (SQLite-Only)

**Status**: Draft · **Last updated**: 2026-06-17

## Introduction

`prog-strength-api` carries **two** implementations of every repository. Each
domain package under `internal/` ships a `repository.go` interface, a real
`sqlite_repository.go`, and a hand-written `memory_repository.go` — 14 of them, on
the order of **3,400 lines** of parallel persistence logic (`activity`, `beta`,
`bodyweight`, `calendarconn`, `chat`, `exercise`, `follow`, `nutrition`,
`nutritionlookup`, `planned_workout`, `steps`, `timeline`, `user`, `workout`).

Production **already runs SQLite** (`mattn/go-sqlite3`; `db.Open(path)` +
`db.Migrate`). The in-memory implementations exist only for two jobs: the
`DATABASE_URL`-unset branch in `server.go` (local dev without a database) and the
**46 test files** that construct `NewMemoryRepository()` directly. Both jobs can be
served by an **ephemeral SQLite** database — a throwaway temp-file DB with the real
migrations applied — which the codebase can already produce with `db.Open` +
`db.Migrate`. The second implementation has outlived its purpose.

Worse, it's a correctness hazard, not just dead weight. Tests that pass against the
in-memory repos do **not** prove the SQLite path: the two implementations can drift
on row ordering, NULL handling, foreign-key/cascade behavior, uniqueness
constraints, and transaction semantics. A green suite today can hide a bug that
only the production (SQLite) repository would surface. Maintaining the pair also
doubles the work on every schema or query change.

This SOW **removes the in-memory repository layer entirely** and consolidates on
SQLite everywhere. Tests run against an ephemeral SQLite database via a shared
helper, so they exercise the real production code path; `DATABASE_URL` becomes
**required**. This is a backend-only cleanup (plus docs) — **no schema change, no
API behavior change, no repository-interface change**. It is a large but mechanical
PR: one new helper, ~46 test-file call-site swaps, 14 file deletions, one
`server.go` simplification.

## Proposed Solution

Make SQLite the single repository implementation, and make ephemeral SQLite the way
tests and local dev get a database.

1. **A shared ephemeral-SQLite test helper.** Add `dbtest.New(t) *sql.DB` (a small
   new package, e.g. `internal/db/dbtest`) that opens a **temp-file** SQLite
   database under `t.TempDir()`, runs the embedded migrations via `db.Migrate`, and
   registers `t.Cleanup` to close it. A temp file — not `:memory:` — because the
   app uses `database/sql` connection pooling, and a `:memory:` DSN gives each
   pooled connection its *own* empty database unless you opt into `cache=shared`
   plus a single connection; a temp file matches production semantics exactly with
   no shared-cache footgun, and `t.TempDir()` cleans itself up.
2. **Migrate every test to the SQLite repo.** Replace each `NewMemoryRepository()`
   call site with the package's `NewSQLiteRepository(dbtest.New(t))` (and the
   matching constructor args — e.g. `exercise.Catalog`, the activity archiver).
   Table-driven `repository_test.go`s that asserted against the memory repo now
   assert against SQLite — the intended coverage gain.
3. **Delete the in-memory layer.** Remove all 14 `internal/*/memory_repository.go`
   and any test that existed *only* to validate the in-memory implementation.
4. **Require `DATABASE_URL`.** Delete the in-memory `else` branch in `server.go`;
   when `DATABASE_URL` is unset the server fails fast with a clear, actionable
   message instead of silently starting an ephemeral in-memory backend.
5. **Update docs** so local dev sets `DATABASE_URL`.

The repository **interfaces are untouched**, so handlers, services, and the server
wiring's SQLite branch don't change. This is a deletion-and-rewire, not a redesign.

## Goals and Non-Goals

### Goals

- **Add `internal/db/dbtest` (or equivalent) with `New(t) *sql.DB`** — opens a
  temp-file SQLite DB under `t.TempDir()`, applies the embedded migrations with
  `db.Migrate`, sets the same pragmas as production (`db.Open` already applies
  `_foreign_keys=on`), and `t.Cleanup`s the handle. One helper, reused everywhere.
- **Migrate all ~46 test files** off `NewMemoryRepository()` onto
  `NewSQLiteRepository(dbtest.New(t), …)` across every affected package
  (`planned_workout`, `workout`, `user`, `nutritionlookup`, `nutrition`,
  `bodyweight`, `steps`, `auth`, `activity`, `timeline`, `follow`, `chat`,
  `calendarsync`, `beta`, `calendarconn`). Behavior of each test is preserved; only
  the backing repository changes.
- **Delete all 14 `internal/*/memory_repository.go`** and any
  `memory_repository_test.go` (or equivalent) that tested the in-memory impl itself
  — that coverage is now redundant with the SQLite path.
- **Make `DATABASE_URL` required**: remove the in-memory branch in
  `internal/server/server.go`; when `cfg.DatabaseURL == ""` the server exits with a
  clear fatal error ("DATABASE_URL is required; set it to a SQLite file path, e.g.
  `DATABASE_URL=./dev.db`"). The existing SQLite path (open → migrate → wire the
  SQLite repos) is unchanged.
- **Point any API-booting harness/script at an ephemeral SQLite DB** — audit for
  callers that started the API relying on no-`DATABASE_URL` in-memory mode
  (notably the **macro eval harness**, which runs the real Go API) and give them a
  temp/ephemeral `DATABASE_URL` so they keep working. (Cross-repo follow-up if a
  caller lives outside `prog-strength-api` — see Open Questions.)
- **Keep CI green** — `go vet`, `go test ./...`, and the existing lint/build all
  pass with the SQLite-backed tests; no skipped tests, no reduced coverage.
- **No production behavior change** — same endpoints, same responses, same SQLite
  schema and migrations. This is purely an internal consolidation.

### Non-Goals

- **The in-memory TCX archiver (`activity.NewMemoryArchiver`).** It's object
  storage (the no-S3 dev/test fallback), not a repository, and is selected
  independently of `DATABASE_URL`. It stays exactly as-is.
- **Telemetry DB wiring.** Telemetry is already SQLite-only (its own
  `TELEMETRY_DATABASE_URL` / `db.MigrateTelemetry`, defaulting alongside the app DB)
  and has no in-memory repository in the 14. Untouched.
- **Any schema, migration, or query change.** The SQLite repositories are used
  as-is; this SOW does not alter SQL, add indexes, or change behavior.
- **Repository *interface* changes.** Method signatures in each `repository.go`
  stay identical, so handlers/services need no edits.
- **Switching the database engine** (e.g. to Postgres) or changing the SQLite
  driver. Out of scope.
- **Reworking how the running server is deployed.** Production already sets
  `DATABASE_URL`; only the *unset* (local-dev) behavior changes.

## Implementation Details

Order follows the lifecycle: build the test substrate, migrate consumers onto it,
then delete the old layer and tighten the server.

### Test helper (`internal/db/dbtest`)

A new, dependency-light package so every domain test can import it without import
cycles (it depends only on `internal/db`, not on any domain package):

```go
// New opens a throwaway SQLite database for a test: a temp-file DB with the
// embedded migrations applied. Closed automatically via t.Cleanup.
func New(t *testing.T) *sql.DB {
    t.Helper()
    path := filepath.Join(t.TempDir(), "app.db")
    database, err := db.Open(path)        // applies _foreign_keys=on, WAL
    if err != nil { t.Fatalf("dbtest open: %v", err) }
    if err := db.Migrate(database); err != nil { t.Fatalf("dbtest migrate: %v", err) }
    t.Cleanup(func() { _ = database.Close() })
    return database
}
```

If a package needs the same DB shared across sub-tests, it calls `New` once in the
parent and threads the `*sql.DB` through — the helper stays trivial.

### Test migration (per affected package)

Each call site swaps the constructor and nothing else:

```go
// before
h := NewHandler(NewMemoryRepository())
// after
h := NewHandler(NewSQLiteRepository(dbtest.New(t)))
```

- Constructors that take extra args keep them: `exercise.NewSQLiteRepository(db)`
  seeds from `exercise.Catalog` per its existing signature;
  `activity.NewSQLiteRepository(db, archiver)` keeps using the **memory archiver**
  (unchanged) in tests.
- Table-driven repository tests that loop assertions over a freshly-built repo build
  a fresh `dbtest.New(t)` per case (or per sub-test) so cases stay isolated, exactly
  as a fresh `NewMemoryRepository()` did.
- Tests whose *sole purpose* was to validate in-memory behavior are deleted, not
  ported. Tests validating real behavior (CRUD, ordering, constraints) are kept and
  now run against SQLite.
- Cross-package consumers in tests (`auth`, `calendarsync`, `timeline`) that built
  another package's memory repo as a dependency swap to that package's SQLite
  constructor over a shared `dbtest.New(t)`.

### Delete the in-memory layer

Remove all 14 `internal/*/memory_repository.go` and their impl-specific tests. After
the test migration, nothing references `NewMemoryRepository` — verify with a
repo-wide grep (`grep -rn "NewMemoryRepository" internal` returns nothing) as the
deletion gate.

### Server wiring (`internal/server/server.go`)

The DB setup currently branches:

```go
if cfg.DatabaseURL != "" {
    // open, migrate, wire SQLite repos
} else {
    // wire in-memory repos   ← delete this
}
```

Remove the `else`. When `cfg.DatabaseURL == ""`, log a fatal, actionable error and
exit (`log.Fatal`) rather than starting. Keep the SQLite branch's open/migrate/wire
exactly as-is. The telemetry block (its own `TelemetryDatabaseURL` handling) is left
untouched.

### Docs

Update the local-run instructions in `README.md`, `CONTRIBUTING.md`, and
`DEPLOYMENT.md` to set `DATABASE_URL` (e.g. `DATABASE_URL=./dev.db go run ./cmd/...`),
and remove any text that describes the no-database in-memory mode.

### Tests

- New: `internal/db/dbtest` has a small self-test (opens, migrates, a trivial
  round-trip) so the helper itself is covered.
- Changed: ~46 files now build SQLite-backed repos; the suite is the regression net.
- Gate: `go test ./...` green, `grep -rn "NewMemoryRepository\|memory_repository" internal`
  returns nothing, and `go build ./...` succeeds.

## Rollout

1. **`prog-strength-api`** — one PR: add `dbtest`, migrate the ~46 tests, delete the
   14 in-memory repos, require `DATABASE_URL`, update docs. CI must be green
   (`go test ./...`, vet, build) and the no-reference grep must come back empty.
2. **Eval harness / dev scripts** — if the macro eval harness (or any script) booted
   the API without `DATABASE_URL`, update it to pass an ephemeral SQLite path in the
   same PR (if in-repo) or a fast follow (if cross-repo — see Open Questions).
3. **`prog-strength-docs`** — mark this SOW shipped on merge.

### Verification after rollout

- `go test ./...` passes with every repository test running against SQLite; no test
  references `NewMemoryRepository`; no `memory_repository.go` remains.
- Starting the server **without** `DATABASE_URL` exits immediately with the clear
  "DATABASE_URL is required" message (not a silent in-memory start).
- Starting the server **with** `DATABASE_URL=./dev.db` opens, migrates, and serves
  exactly as before; a manual smoke (create a user, log a workout, read it back)
  persists to the file.
- Production deploy (which already sets `DATABASE_URL`) is unaffected — same schema,
  endpoints, and responses.
- The in-memory TCX archiver still serves the no-S3 dev path (unchanged).

## Open Questions

1. **Where the eval harness lives.** The macro eval harness "runs the real Go API";
   if it boots the API from **outside** `prog-strength-api` (e.g. in `prog-strength-mcp`
   or a separate `evals/` tree) relying on in-memory mode, requiring `DATABASE_URL`
   breaks it. **Lean:** if it's in-repo, fix it in this PR; if it's cross-repo, add
   that repo to a fast-follow SOW rather than widening this one's `repos:`. Confirm
   the harness's location and DB assumption during implementation.
2. **`:memory:` vs temp file for the helper.** Temp file is the safe default
   (matches pooling/production semantics). **Lean:** temp file; revisit `:memory:`
   + `cache=shared` only if test-suite wall-clock becomes a problem (unlikely — the
   migrations are small and `t.TempDir()` is on a fast tmpfs on CI).
3. **Shared helper location/name.** `internal/db/dbtest` keeps it next to `db` with
   no import cycle. **Lean:** that path; if a domain test needs fixtures beyond a
   bare migrated DB, those stay in the domain's own test files, not in `dbtest`
   (it stays a thin opener).
