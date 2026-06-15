---
status: ready_for_implementation
repos:
  - prog-strength-api
  - prog-strength-infra
  - prog-strength-docs
---

# Dynamic Beta Allowlist

**Status**: Ready for implementation · **Last updated**: 2026-06-14

## Introduction

Access to Prog Strength during the closed beta is gated by an email allowlist. Today that allowlist is the `BETA_ALLOWED_EMAILS` environment variable: a comma-separated string read once at process start in `internal/config/config.go`, parsed into a `[]string`, and frozen into an in-memory `map[string]struct{}` when `auth.NewHandler` is constructed. The gate runs at the Google OAuth callback (`internal/auth/handler.go`): an allowed email receives a JWT; anyone else completes OAuth (a user row is created for sign-up visibility) but is redirected back to the frontend with `#error=beta_required` and never gets a token, so they can't reach the agent and burn Anthropic credits. An empty allowlist disables the gate entirely.

The problem is operational, not functional. Because the value lives in a GitHub Actions secret and is read only at boot, **adding a single beta tester requires editing a write-only secret and redeploying the API container** — a full restart of production to flip one boolean. The secret being write-only also means the current list can't be read back, so every edit risks silently dropping existing testers. For a closed beta that will grow to at most a few hundred users, this is far too much ceremony per change.

This SOW moves the allowlist out of process configuration and into the database, and adds an admin-only HTTP surface for managing it. After this lands, authorizing a new tester is one authenticated API call that takes effect on the next login — no secret edit, no deploy, no restart, and no risk of clobbering the existing list. The scope is deliberately tight: a SQLite table, a small repository, three admin endpoints, and a one-time seed from the existing env var. No web UI, no MCP/agent tooling, and no change to the gate's externally observable behavior.

## Proposed Solution

The allowlist becomes a row set in the existing SQLite database rather than a frozen-at-boot map. This is the natural store: the API already runs on SQLite with numbered migrations (currently through `018`) and Litestream replication to S3 from a single EC2 instance, and it already exposes a clean repository pattern (interface + `sqlite_*` + `memory_*` implementations, e.g. `internal/user/`). A few-hundred-row allowlist is trivially within SQLite's envelope, and reusing the existing datastore means no new AWS dependency, IAM policy, SDK, or Terraform module. DynamoDB was considered and rejected for exactly this reason — a second datastore earns its keep only under horizontal scale or a serverless runtime, neither of which applies to a single-instance API.

A new migration `019_beta_allowed_emails.sql` creates a `beta_allowed_emails` table keyed by normalized (lower-cased, trimmed) email. A new `internal/beta` package provides a `Repository` interface — `IsAllowed`, `Add`, `Remove`, `List` — with SQLite and in-memory implementations mirroring the conventions in `internal/user`. The OAuth gate stops consulting the boot-time map and instead calls `beta.Repository.IsAllowed(ctx, email)`. Because the OAuth callback fires only at login (an infrequent, human-paced event), a single indexed lookup per login is free; there is no in-memory cache to keep coherent and therefore no invalidation problem. The "empty allowlist disables the gate" semantics are preserved at the repository level: `IsAllowed` returns `true` for everyone when the table is empty, so local/dev and pre-beta environments behave exactly as they do today.

Continuity with the current production allowlist is handled by a one-time seed. On boot, if the `beta_allowed_emails` table is empty and `BETA_ALLOWED_EMAILS` is non-empty, the API inserts the env-var emails into the table (idempotent `INSERT OR IGNORE`, guarded by the table-empty check so reruns and post-seed edits are never overwritten). This mirrors the in-process backfill pattern used elsewhere in the repo and means the cutover preserves the live list with zero manual steps. After the first boot on the new code, `BETA_ALLOWED_EMAILS` is vestigial — it remains wired for one release as the seed source and a documented rollback lever, then a follow-up removes it.

Management is exposed as three endpoints under a new `/admin/beta-emails` route group: `GET` to list, `POST` to add, and `DELETE` to remove. The group sits behind the existing `auth.RequireUser` middleware and an additional admin gate. Admin identity reuses the same pattern as the beta list itself: a new `ADMIN_EMAILS` env var (comma-separated, parsed by the existing `splitCSV`) names the operators. The admin middleware resolves the authenticated user ID (which `RequireUser` already injects into the request context) to the user's email via the user repository and checks membership; a non-admin gets `403`. This keeps the bootstrap trivial — one env var, set once, naming you as the sole admin — and introduces no new auth token to leak in a curl command, since admin calls are authenticated with the operator's ordinary user JWT obtained by logging in normally.

One property is explicitly preserved end to end: the gate exists to stop non-allowed users from obtaining a token and spending Anthropic credits. Adding an email grants access on that user's next login; removing an email prevents future logins from minting a token. Removal does not revoke a JWT already issued — the tokens are stateless and the gate runs only at login — so a removed user retains access until their current token expires. That matches today's behavior exactly (the env-var gate has the same property) and the short token TTL bounds the window; true session revocation is out of scope and noted below.

## Goals and Non-Goals

### Goals

- Add migration `internal/db/migrations/019_beta_allowed_emails.sql` creating `beta_allowed_emails(email TEXT PRIMARY KEY, added_at DATETIME NOT NULL, added_by TEXT, note TEXT)`. `email` stores the normalized (lower-cased, trimmed) address; `added_by` records the admin email that added the row (nullable to accommodate the env seed, which sets it to a sentinel like `"seed:BETA_ALLOWED_EMAILS"`); `note` is an optional free-text label.
- Add a new `internal/beta` package with a `Repository` interface — `IsAllowed(ctx, email) (bool, error)`, `Add(ctx, email, addedBy, note) error`, `Remove(ctx, email) (bool, error)`, `List(ctx) ([]AllowedEmail, error)` — plus `sqlite_repository.go` and `memory_repository.go` implementations following the `internal/user` conventions. All reads and writes normalize email with `strings.ToLower(strings.TrimSpace(email))` so lookups are case- and whitespace-insensitive, matching the current gate.
- `IsAllowed` returns `(true, nil)` when the table is empty, preserving the "empty allowlist disables the gate" semantics that local dev and pre-beta rely on. The emptiness check must be cheap (e.g. `SELECT EXISTS(...)` / `LIMIT 1`), not a full scan.
- Change the OAuth gate to consult the repository instead of the boot-time map. Replace the `betaAllowedEmails map[string]struct{}` field and `isBetaAllowed` map lookup in `internal/auth/handler.go` with a call to an injected `beta.Checker` interface (the `IsAllowed` method only, to keep the auth package's dependency surface minimal). Wire the concrete repository in `internal/server/server.go` where `auth.NewHandler` is constructed.
- One-time seed on boot: if `beta_allowed_emails` is empty and `BETA_ALLOWED_EMAILS` is non-empty, insert each env email via `INSERT OR IGNORE` with `added_by = "seed:BETA_ALLOWED_EMAILS"`. Guarded by the table-empty check so it runs at most once and never overwrites admin edits. Log a one-line summary of how many rows were seeded.
- Add `ADMIN_EMAILS` to the config: a new `AdminEmails []string` field in `internal/config/config.go`, parsed via the existing `splitCSV(os.Getenv("ADMIN_EMAILS"))`.
- Add an admin-gate middleware in `internal/auth` (e.g. `RequireAdmin`) that reads the user ID from context (`auth.UserIDFrom`), resolves it to an email via the user repository, and returns `403 forbidden` unless that email is in `AdminEmails`. An empty `ADMIN_EMAILS` denies all admin routes (fail-closed) — the admin surface is inert until an operator is configured.
- Add an `internal/beta` HTTP handler mounting three routes under `/admin/beta-emails`, behind `auth.RequireUser` + the admin gate (see § API Surface): `GET` (list), `POST` (add), `DELETE /{email}` (remove). Add is idempotent (re-adding an existing email is a `200`/`204`, not an error); remove returns `404` when the email isn't present.
- Wire `ADMIN_EMAILS` through the deployment path: add it to the `envs:` list and env block of `prog-strength-api/.github/workflows/release.yml` and `.../manual-deploy.yml` (sourced from a new `secrets.ADMIN_EMAILS`), and add `- ADMIN_EMAILS=${ADMIN_EMAILS}` to the API service in `prog-strength-infra/compose/api/docker-compose.yml`, alongside the existing `BETA_ALLOWED_EMAILS` line.
- Update `prog-strength-api/README.md` (the env-var table) and `prog-strength-api/AGENTS.md` (the "Beta gate" note) to describe the table-backed allowlist, the `/admin/beta-emails` endpoints, and `ADMIN_EMAILS`. Note that `BETA_ALLOWED_EMAILS` is now seed-only and slated for removal.
- Tests per § Tests.

### Non-Goals

- **No web or mobile admin UI.** v1 manages the allowlist via the HTTP endpoints (curl or a small script). A web admin surface is a clean follow-up once the endpoints exist.
- **No MCP / agent tooling.** The agent does not get a tool to add or remove beta users; granting access is an operator action, not a chat action. (A future SOW could add a guarded admin tool if desired.)
- **No session/JWT revocation.** Removing an email blocks future logins but does not invalidate an already-issued token; the existing token TTL bounds the window. This is unchanged from today's behavior.
- **No role system.** `ADMIN_EMAILS` is a flat operator list, not a general RBAC model. A persisted roles table is out of scope.
- **No self-service waitlist or invite flow.** The `beta-locked` page's "email us to request access" copy is unchanged; this SOW only changes how an operator fulfills such a request.
- **No DynamoDB or other new datastore**, and no new infrastructure beyond the two env-var wirings above.

## API Surface

All three routes are mounted behind `auth.RequireUser(jwtSecret)` and the new admin gate. Email path parameters are URL-encoded and normalized server-side.

- `GET /admin/beta-emails` → `200` with `{ "emails": [ { "email": "...", "added_at": "RFC3339", "added_by": "...", "note": "..." } ] }`, sorted by `added_at` ascending. `403` if the caller is not an admin.
- `POST /admin/beta-emails` with body `{ "email": "...", "note": "optional" }` → `201` (or `200` if already present; idempotent) with the created/existing row. `400` on a malformed/empty email. `403` for non-admins. `added_by` is set to the calling admin's email.
- `DELETE /admin/beta-emails/{email}` → `204` on removal, `404` if the email was not on the list, `403` for non-admins.

The OAuth callback's externally observable contract is unchanged: allowed emails get a token; disallowed emails are redirected with `#error=beta_required`; an empty allowlist lets everyone through.

## Data Model

```sql
-- migrations/019_beta_allowed_emails.sql
CREATE TABLE IF NOT EXISTS beta_allowed_emails (
    email     TEXT PRIMARY KEY,        -- normalized: lower-cased, trimmed
    added_at  DATETIME NOT NULL,
    added_by  TEXT,                     -- admin email, or "seed:BETA_ALLOWED_EMAILS"
    note      TEXT
);
```

The `PRIMARY KEY` on the normalized email gives O(log n) lookup for `IsAllowed` and enforces dedup; no secondary index is needed at this scale.

## Tests

- **`internal/beta` repository (sqlite + memory, table-driven against the interface):** add then `IsAllowed` true; `IsAllowed` false for an absent email; case- and whitespace-insensitivity (`Foo@Bar.com ` matches `foo@bar.com`); `Add` idempotency (second add does not error and does not duplicate); `Remove` returns `true` then `false`; `List` ordering by `added_at`; and the critical empty-table case where `IsAllowed` returns `true` for any email.
- **Seed-on-boot:** empty table + non-empty `BETA_ALLOWED_EMAILS` seeds the rows with the sentinel `added_by`; non-empty table is left untouched (no overwrite); empty env is a no-op. A second boot does not re-seed or duplicate.
- **Admin middleware:** a user whose email is in `ADMIN_EMAILS` passes; a non-admin gets `403`; an empty `ADMIN_EMAILS` denies everyone (fail-closed); a request with no user in context (misconfiguration) is denied.
- **Admin handler:** add → list round-trip; idempotent re-add; delete then delete-again (`204` then `404`); malformed email `400`; non-admin `403` on every verb.
- **OAuth gate integration:** an allowed email mints a token; a disallowed email is redirected with `#error=beta_required`; after an admin `POST` adds an email, a subsequent OAuth callback for that email succeeds — proving the no-restart path end to end against the in-memory repository.

## Rollout

The change is backward-compatible and self-seeding. On deploy, the migration creates the table and the first boot seeds it from the still-present `BETA_ALLOWED_EMAILS` secret, so the live allowlist carries over with no manual action and no window where the gate is wrong. `ADMIN_EMAILS` must be set (as a GitHub Actions secret on `prog-strength-api`) before the admin endpoints are usable; until then the admin routes are inert (fail-closed) and the gate continues to work off the seeded table. Once a boot has seeded the table, `BETA_ALLOWED_EMAILS` no longer affects the gate and can be removed in a follow-up change that also strips its config field and deploy wiring.

## Open Questions

1. **Admin model longevity.** `ADMIN_EMAILS` is the minimal viable operator gate. If the beta ever needs more than one or two operators, or operators who shouldn't share a flat list, a persisted roles table replaces it — but that's premature today.
2. **Audit depth.** `added_by` + `added_at` give a basic trail. If we later want removal history or who-removed-whom, an append-only `beta_allowlist_events` log is the natural extension; v1 keeps only current state.
