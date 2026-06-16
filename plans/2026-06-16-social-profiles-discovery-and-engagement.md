# Social Profiles: Discovery, Bio, Graphs & Author Avatars — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Close the handle-reachability hole, add user bios, add a privacy-gated public per-user weekly stats endpoint with profile graphs, and embed author identity in timeline posts and comments — across `prog-strength-api` (Go) and `prog-strength-web` (Next.js).

**Architecture:** API is the single writer (data model, migrations, endpoints); web consumes via `lib/api.ts` wrappers. Each of the four SOW parts is an additive, backward-compatible vertical slice (new nullable column / new endpoint / additive DTO fields). No feature flags. Part 1 (reachability) is independent and ships first; Parts 2–4 are mutually independent.

**Tech Stack:** Go 1.25 (chi router, SQLite, aws-sdk-go-v2, embedded SQL + registered Go migrations), Next.js 16 App Router + React 19 + TypeScript + Tailwind v4 + recharts + Vitest.

---

## Conventions every task must follow

**Go repo (`/workspace/prog-strength-api`):**
- TDD: write the failing test first, then the minimal implementation.
- Match existing style: short comments explain *why* not *what*; one type/concept per file in domain packages; no new third-party deps.
- Lint/format gate (run before any commit is considered done):
  - `gofmt -l .` reports nothing.
  - `go run github.com/golangci/golangci-lint/v2/cmd/golangci-lint@v2.12.2 run` (CI-pinned; `golangci-lint v2.12.2` is already on PATH and may be used directly).
  - `go vet ./...`
  - `go mod tidy` produces no diff in `go.mod`/`go.sum`.
  - `go test ./...` (CI runs `-race -cover`).
- Conventional-commit subjects, lowercase (e.g. `feat(user): generate handles for handle-less users`).

**Web repo (`/workspace/prog-strength-web`):**
- TDD where there's logic: Vitest + Testing Library, co-located `*.test.ts(x)`.
- All API calls go through `lib/api.ts` typed wrappers — never ad-hoc `fetch` in a component.
- Tailwind utility classes + theme CSS variables (`var(--muted)`, `var(--border)`, `var(--surface)`, `var(--danger)`). Match neighboring surfaces.
- Gate before commit: `npm run typecheck && npm run lint && npm run test`; `npm run build` if routes/imports/config changed. Do not introduce new lint warnings.
- Conventional-commit subjects, lowercase.

---

## File structure (what each task touches)

**API — new files:**
- `internal/user/handle_generator.go` — `GenerateHandle` + slug helper.
- `internal/user/handle_generator_test.go`
- `internal/db/migration_029_backfill_usernames.go` — Go migration (registered as version 29).
- `internal/db/migration_029_backfill_usernames_test.go`
- `internal/db/migrations/030_user_bio.sql` — add `bio TEXT`.
- `internal/user/profile_stats_handler.go` — `GET /users/{username}/stats` handler + DTO + 12-week bucketing.
- `internal/user/profile_stats_handler_test.go`
- `internal/workout/stats.go` (or method on existing repo file) — `ListCompletedSessionsSince`.
- `internal/activity/stats.go` (or method on existing repo file) — `ListRunningSamplesSince`.

**API — modified files:**
- `internal/auth/handler.go` — assign generated handle at user creation.
- `internal/db/go_migrations.go` — register `migration029()`.
- `internal/user/user.go` — add `Bio *string`.
- `internal/user/username.go` or `user.go` — bio validation (160-rune cap).
- `internal/user/handler.go` — `updateMeRequest.Bio`, apply in `updateMe`, `meResponse.Bio`.
- `internal/user/discovery_handler.go` — `publicProfileDTO.Bio`; mount `/users/{username}/stats`; inject stats providers.
- `internal/workout/repository.go`, `internal/activity/repository.go` — interface additions.
- `internal/timeline/handler.go` + `internal/timeline/model.go` (or new `internal/timeline/author.go`) — `author` on `postDTO`/`commentDTO`, batch resolution, `ProfileResolver` interface.
- `internal/server/server.go` — wire stats providers into `DiscoveryHandler` and the `ProfileResolver` into the timeline handler.

**Web — modified files:**
- `lib/api.ts` — `bio` on `ResolvedProfile`/`PublicProfile`; `getProfileStats` + types; `author` on `TimelinePost`/`TimelineComment`.
- `app/(app)/settings/page.tsx` — `<BioRow>`.
- `app/(app)/u/[username]/page.tsx` — bio in header + stats charts section.
- `app/(app)/u/[username]/_components/` — new `ProfileStats.tsx` (+ test).
- `components/profile-stats-chart.tsx` (or adapt existing) — presentational weekly chart.
- `app/(app)/timeline/_components/TimelinePostCard.tsx` — author header.
- `app/(app)/timeline/_components/CommentThread.tsx` — author rows.

---

## TASK 1 — Handle generator (api, Part 1)

**Files:**
- Create: `internal/user/handle_generator.go`
- Create: `internal/user/handle_generator_test.go`

**Context:** `username` is a nullable public handle, stored lowercased, validated by `ValidateUsername(raw) (canonical string, err)` in `internal/user/username.go`. Rules: must match `^[a-z][a-z0-9_]{2,29}$` (len 3–30), not in the reserved set; constants `UsernameMinLen=3`, `UsernameMaxLen=30`; errors `ErrUsernameInvalid`, `ErrUsernameReserved`. `User.ID` is a string (UUID-ish). We need a deterministic helper that turns a display name + user ID into a *valid* candidate handle, plus a collision-resolving driver.

**Design — two functions, no I/O in the pure layer:**

```go
// slugifyHandle converts a display name into a ValidateUsername-passing
// candidate, or "" if nothing usable remains. Lowercases, ASCII-folds
// (drop non-ASCII rather than transliterate — keeps the helper dependency-free),
// replaces every run of disallowed chars with a single underscore, trims
// leading/trailing underscores and leading digits (handle must start [a-z]),
// then truncates to UsernameMaxLen. Returns "" when the result is empty,
// too short, or reserved.
func slugifyHandle(displayName string) string

// fallbackHandle derives a guaranteed-valid unique-by-construction handle
// from the user id: "user_" + first 8 lowercased [a-z0-9] chars of the id
// (stripping dashes). Always passes ValidateUsername.
func fallbackHandle(userID string) string

// GenerateHandle returns a valid, unique handle for (displayName, userID).
// exists(candidate) must report whether a handle is already taken (queries
// the unique index, excluding soft-deleted rows). Strategy:
//   1. base := slugifyHandle(displayName); if "" -> base := fallbackHandle(userID)
//   2. try base, then base+"2", base+"3", ... up to maxHandleSuffix attempts,
//      returning the first that ValidateUsername accepts AND !exists().
//      (Suffixing must not push past UsernameMaxLen: trim base so base+suffix fits.)
//   3. on exhaustion, return fallbackHandle(userID) (unique by construction).
// Returns a canonical (already-validated) handle.
func GenerateHandle(displayName, userID string, exists func(string) (bool, error)) (string, error)
```

- [ ] **Step 1: Write failing tests** in `handle_generator_test.go`:
  - `slugifyHandle("Sam Lifter")` → `"sam_lifter"`; passes `ValidateUsername`.
  - `slugifyHandle("  Ünîçødé 你好 ")` → ASCII-folded/dropped result that passes `ValidateUsername` (e.g. `"nd"` is too short → expect `""`; pick an input that exercises folding, e.g. `slugifyHandle("Café Crawl")` → `"caf_crawl"` passes). Assert via `ValidateUsername` round-trip rather than hardcoding fragile output.
  - `slugifyHandle("")` → `""`; `slugifyHandle("!!!")` → `""`; `slugifyHandle("42")` → `""` (digits-only / too short).
  - `slugifyHandle("admin")` → `""` (reserved, since reserved candidates are unusable).
  - `fallbackHandle("3f2a9c10-dead-beef")` → starts with `"user_"`, passes `ValidateUsername`.
  - `GenerateHandle` happy path: `exists` always false → returns `slugifyHandle(displayName)`.
  - `GenerateHandle` collision suffixing: `exists` returns true for `"sam"` and `"sam2"`, false for `"sam3"` → returns `"sam3"`.
  - `GenerateHandle` empty/garbage display name → returns a `fallbackHandle`-shaped value (`user_…`).
  - `GenerateHandle` reserved slug → does not return the reserved word; returns fallback or suffix.
  - `GenerateHandle` exhaustion (`exists` always true) → returns `fallbackHandle(userID)`.
- [ ] **Step 2: Run** `go test ./internal/user/ -run Handle -v` → FAIL (undefined).
- [ ] **Step 3: Implement** the three functions in `handle_generator.go`. ASCII-fold by dropping bytes `>= 0x80`; map any char not in `[a-z0-9]` to a separator; collapse separators to single `_`. Pick `maxHandleSuffix = 50`.
- [ ] **Step 4: Run** the tests → PASS. Run `gofmt -l internal/user` (empty), `golangci-lint run ./internal/user/...`, `go vet ./internal/user/...`.
- [ ] **Step 5: Commit** `feat(user): add unique handle generator for handle-less users`.

---

## TASK 2 — Auto-assign handle at signup + backfill migration (api, Part 1)

**Files:**
- Modify: `internal/auth/handler.go` (`findOrCreateUser`, ~lines 264–296)
- Create: `internal/db/migration_029_backfill_usernames.go`
- Create: `internal/db/migration_029_backfill_usernames_test.go`
- Modify: `internal/db/go_migrations.go` (register `migration029()`)

**Context:** New users are created in `findOrCreateUser` with `Username == nil`. The repo `Create` inserts username as-is; the unique index `idx_users_username` is partial (`WHERE deleted_at IS NULL`) and SQLite allows multiple NULLs. Go migrations are registered in `internal/db/go_migrations.go` (`registeredGoMigrations()` currently returns `[]goMigration{migration028()}`); the runner wraps each migration's `Run(ctx, tx)` in a **single transaction** and records it in `schema_migrations` (see `migration_028_backfill_planned_workout_links.go` for the precedent shape). Versions 1–27 are SQL, 28 is Go; **next version is 29**.

> **Note on "commit per-user":** the SOW asks for per-user commits so a collision doesn't roll back the batch. The migration runner is single-transaction (matching migration 028), so we instead resolve collisions *within* the transaction via the generator's suffix-retry — a collision never aborts the run, and re-running is a no-op (idempotent) because no `username IS NULL` rows remain. Document this in the migration's doc comment.

**Signup change:** After a new `user.User` is built in `findOrCreateUser` (before `h.users.Create`), the username is still nil. Because `GenerateHandle` needs the user ID (assigned during `Create`) and an `exists` check, assign the handle **after** a successful `Create`, then `Update`:

```go
if err := h.users.Create(ctx, newUser); err != nil {
    return nil, err
}
if newUser.Username == nil {
    handle, gErr := user.GenerateHandle(newUser.DisplayName, newUser.ID, func(c string) (bool, error) {
        _, e := h.users.GetByUsername(ctx, c)
        if errors.Is(e, user.ErrNotFound) {
            return false, nil
        }
        return e == nil, e // taken if found; propagate real errors
    })
    if gErr != nil {
        return nil, gErr
    }
    newUser.Username = &handle
    if err := h.users.Update(ctx, newUser); err != nil {
        return nil, err
    }
}
```
(Confirm `h.users` exposes `GetByUsername` and `ErrNotFound`; the discovery handler uses both, and `auth.Handler` holds the same `user` repository — if the auth handler's repo interface lacks `GetByUsername`, widen that interface minimally.)

**Migration 029 — `backfillUsernames(ctx, tx)`:**
- Select `id, display_name FROM users WHERE username IS NULL AND deleted_at IS NULL`.
- For each row, generate a handle with an `exists` closure that checks the **same transaction**: `SELECT 1 FROM users WHERE username = ? AND deleted_at IS NULL` (so suffixes account for handles assigned earlier in this same run). Reuse `user.GenerateHandle`. Import is `internal/user` — confirm no import cycle (`internal/db` does not currently import `internal/user`; if a cycle arises, move the pure generator usage behind a tiny function-typed param or duplicate `slugify` — prefer importing `user`; verify with `go build`).
- `UPDATE users SET username = ?, updated_at = ? WHERE id = ?` within the tx.
- Doc comment: idempotent, frozen logic, in-tx suffix retry rationale.

- [ ] **Step 1: Write failing migration test** `migration_029_backfill_usernames_test.go`: build an in-memory/temp SQLite DB, run migrations up to 028, insert: a user with `username IS NULL`, a second null-username user whose display name slugifies to the **same** base (assert they get distinct handles via suffixing), a user with an existing handle `"keep_me"` (assert untouched), and a soft-deleted null-username user (assert it is **not** assigned). Run `backfillUsernames`, then assert: every non-deleted user has a non-null handle that passes `user.ValidateUsername`, handles are unique, `"keep_me"` is unchanged, the deleted user is still null. Re-run the migration → no error, no changes (idempotent).
- [ ] **Step 2: Write failing signup test** (in `internal/auth` or wherever user-creation is tested): a user created via the create path with no username comes back with a non-nil handle that passes `ValidateUsername`. (If auth handler tests are heavy to set up, add the assertion to an existing `findOrCreateUser` test.)
- [ ] **Step 3: Run** both → FAIL.
- [ ] **Step 4: Implement** the migration, register `migration029()` in `go_migrations.go`, and the signup change.
- [ ] **Step 5: Run** `go test ./internal/db/... ./internal/auth/... ./internal/user/...` → PASS. Lint/vet/format clean.
- [ ] **Step 6: Commit** `feat(user): auto-assign handles at signup and backfill existing users`.

---

## TASK 3 — Bio field end-to-end on the API (api, Part 2)

**Files:**
- Create: `internal/db/migrations/030_user_bio.sql`
- Modify: `internal/user/user.go` (`User.Bio *string`, validation)
- Modify: `internal/user/sqlite_repository.go` (Create/Update/GetByID/GetByUsername/GetByEmail SELECT+INSERT+UPDATE column list — add `bio`)
- Modify: `internal/user/handler.go` (`updateMeRequest.Bio`, apply, `meResponse.Bio`, resolve)
- Modify: `internal/user/discovery_handler.go` (`publicProfileDTO.Bio` + populate in `getProfile`)
- Tests: `internal/user/*_test.go`

**Context:** `User` struct is in `user.go`; `meResponse` (handler.go:54-65) and `updateMeRequest` (110-118) use pointer-presence partial updates; `publicProfileDTO` (discovery_handler.go:90-98) is returned by `getProfile`. The SQLite repo lists every column explicitly in INSERT/UPDATE/SELECT — `bio` must be added to **all** of them consistently. Bio rules: nullable `TEXT`, max **160 runes**, plain text; empty string clears it (store as NULL per the existing `HeightCm`-style nullable convention).

**Migration 030 (`030_user_bio.sql`):**
```sql
-- Short, plain-text user bio shown on the public profile. Nullable; validated
-- (<=160 runes) at the API write edge. Empty input clears it to NULL.
ALTER TABLE users ADD COLUMN bio TEXT;
```

**Validation** (add to `User.Validate()` or a dedicated `ValidateBio`): `if u.Bio != nil && utf8.RuneCountInString(*u.Bio) > 160 { return ErrBioTooLong }`. Add `ErrBioTooLong` (client error mapped to 400, matching how username validation errors map).

**updateMe semantics:** `Bio *string` on the request. `nil` = absent (don't touch). Non-nil: trim? The web trims; the API treats `""` as clear → set `u.Bio = nil`. So:
```go
if req.Bio != nil {
    if *req.Bio == "" {
        u.Bio = nil
    } else {
        u.Bio = req.Bio
    }
}
```
Validate after applying. Add `Bio *string `json:"bio"`` to `meResponse` (populated in `resolveMe`) and to `publicProfileDTO` (populated in `getProfile`).

- [ ] **Step 1: Write failing Go tests:**
  - Validation: 160 runes OK; 161 runes → `ErrBioTooLong`; a 160-rune multibyte string (e.g. 160 `é`) is OK (counts runes, not bytes).
  - `updateMe`: setting `bio` persists and appears in `meResponse`; sending `""` clears it; omitting `bio` leaves it unchanged.
  - `getProfile`: `publicProfileDTO` includes `bio` (present when set, null when unset).
- [ ] **Step 2: Run** → FAIL.
- [ ] **Step 3: Implement** migration, struct field, validation, repo column wiring (all SELECT/INSERT/UPDATE statements), handler request/response/apply, DTO field. Verify repo round-trips bio (a `GetByID` after `Update` returns it).
- [ ] **Step 4: Run** `go test ./internal/user/...` → PASS. Lint/vet/format/tidy clean.
- [ ] **Step 5: Commit** `feat(user): add editable bio to profile and updateMe`.

---

## TASK 4 — Public profile stats endpoint + privacy gate (api, Part 3)

**Files:**
- Create: `internal/user/profile_stats_handler.go`
- Create: `internal/user/profile_stats_handler_test.go`
- Modify: `internal/workout/repository.go` + `internal/workout/sqlite_repository.go` — add `ListCompletedSessionsSince`.
- Modify: `internal/activity/repository.go` + `internal/activity/sqlite_repository.go` — add `ListRunningSamplesSince`.
- Modify: `internal/user/discovery_handler.go` — mount route + new handler deps.
- Modify: `internal/server/server.go` — wire workout/activity providers.

**Context:** `GET /users/{username}/stats` returns two dense 12-week weekly series, bucketed in the **target user's** timezone (IANA string on `User.Timezone`, default UTC). Existing analytics bucket in Go with local week bounds (see `internal/activity/metrics.go`: `localWeekBounds(now, loc)` = `[Monday 00:00 local, next Monday)`; aggregating in Go because SQLite `date()` is DST-wrong). Reuse that *pattern* (Monday-based local weeks). Privacy gate is **identical to the scoped activity feed** (`timeline.listFeed` with `?user=`): self or accepted-follower → full; otherwise → `locked`. There is **no account-level public/private flag** in the data model — privacy is per-post; so the gate reduces to self-or-accepted-follower, and any other viewer gets `locked`. Document this explicitly.

**Repository methods (raw projections; bucketing happens in the handler):**
```go
// workout.Repository
// ListCompletedSessionsSince returns (PerformedAt, EndedAt) for the user's
// non-deleted workouts with a non-null EndedAt occurring at/after `since`.
// Workouts without an EndedAt are excluded (their duration is unknown).
ListCompletedSessionsSince(ctx context.Context, userID string, since time.Time) ([]SessionDuration, error)
// where type SessionDuration struct{ PerformedAt, EndedAt time.Time }

// activity.Repository
// ListRunningSamplesSince returns (StartTime, DistanceMeters) for the user's
// non-deleted running activities starting at/after `since`.
ListRunningSamplesSince(ctx context.Context, userID string, since time.Time) ([]RunSample, error)
// where type RunSample struct{ StartTime time.Time; DistanceMeters float64 }
```
SQL mirrors the existing `RunningMetrics` query shape (filter `deleted_at IS NULL`, `activity_type = ActivityRunning` for runs; `EndedAt IS NOT NULL` for workouts). `since` is the start of the earliest of the 12 week buckets (computed in the handler, in UTC instant terms).

**Handler — `profileStatsHandler` (a small struct, or methods added to `DiscoveryHandler`):** Prefer adding to `DiscoveryHandler` so route grouping stays with the other `/users/{username}/*` routes. Inject two narrow provider interfaces (defined in `internal/user`) so the user package doesn't import workout/activity concretely:
```go
type LiftSessionSource interface {
    ListCompletedSessionsSince(ctx context.Context, userID string, since time.Time) ([]LiftSession, error)
}
type RunningSampleSource interface {
    ListRunningSamplesSince(ctx context.Context, userID string, since time.Time) ([]RunningSample, error)
}
// LiftSession{ PerformedAt, EndedAt time.Time }; RunningSample{ StartTime time.Time; DistanceMeters float64 }
```
Server wires thin adapters that call the real repos and map to these local types (avoids an import cycle and keeps the user package self-contained).

**12-week bucketing (pure, testable helper in the handler file):**
```go
// weekStarts returns 12 Monday-00:00 local boundaries, oldest first, ending
// with the Monday of the week containing `now`.
func weekStarts(now time.Time, loc *time.Location) []time.Time

// bucketByWeek assigns each sample's timestamp to its local week and sums
// `value`, returning a dense []point aligned to starts (zero-filled).
```
Response DTO:
```go
type statsPoint struct {
    WeekStart time.Time `json:"week_start"`
    Value     float64   `json:"value"`
}
type profileStatsDTO struct {
    LiftSessionMinutes    []statsPoint `json:"lift_session_minutes"`
    RunningDistanceMeters []statsPoint `json:"running_distance_meters"`
    Locked                bool         `json:"locked,omitempty"`
}
```
Lift minutes per session = `EndedAt.Sub(PerformedAt).Minutes()` (guard against negatives → treat <0 as 0). Bucket by `PerformedAt` in `loc`. Running bucket by `StartTime`. Each series is exactly 12 points, zero-filled.

**Handler flow (`GET /users/{username}/stats`):**
1. Resolve viewer (auth) and target via `GetByUsername` (404 if not found, like `getProfile`).
2. Privacy: `if target.ID != viewer { rel := h.follows.Relationship(ctx, viewer, target.ID); if rel != follow.RelationshipFollowing { return locked } }`. Locked response: `httpresp.OK(w, "profile stats", profileStatsDTO{LiftSessionMinutes: nil, RunningDistanceMeters: nil, Locked: true})` — but match the timeline's shape: the web reads `locked`. For locked, return empty/omitted series + `locked:true`. (Keep series as `[]statsPoint{}` non-nil so JSON is `[]` not `null`, consistent with the feed.)
3. Load `loc` from `target.Timezone` (`time.LoadLocation`; fall back to UTC on error).
4. `now := time.Now().UTC()`; `starts := weekStarts(now, loc)`; `since := starts[0]`.
5. Pull both sources since `since`, bucket, build dense series, return `{lift_session_minutes, running_distance_meters}`.

**Mount** in `DiscoveryHandler.Mount`: `r.Get("/users/{username}/stats", h.getStats)` (after the existing `/users/{username}` routes).

- [ ] **Step 1: Write failing tests** `profile_stats_handler_test.go` using fake/in-memory sources + a fake follow-relationship:
  - Bucketing: 12 dense points; a session spanning a known week lands in the right bucket; an **end-less** workout (no `EndedAt`) is excluded from lift minutes (fed by `ListCompletedSessionsSince` already excluding it — assert the source filter, and assert the series is all-zero when only end-less sessions exist); timezone correctness (a Sunday-23:00-local run lands in the local week, not the UTC week — pick a tz like `America/New_York` and a boundary instant).
  - Privacy matrix: self → full series; accepted-follower (`RelationshipFollowing`) → full; non-follower (`RelationshipNone`) → `locked:true` with empty series; (there is no public-account branch — assert non-follower is always locked).
  - Unknown username → 404.
- [ ] **Step 2: Write failing repo tests** for `ListCompletedSessionsSince` (excludes end-less + soft-deleted + before `since`) and `ListRunningSamplesSince` (only running, excludes soft-deleted + before `since`).
- [ ] **Step 3: Run** → FAIL.
- [ ] **Step 4: Implement** repo methods, interfaces + adapters, bucketing helpers, handler, route, server wiring.
- [ ] **Step 5: Run** `go test ./internal/user/... ./internal/workout/... ./internal/activity/...` → PASS. Lint/vet/format/tidy clean. `go build ./...`.
- [ ] **Step 6: Commit** `feat(user): add privacy-gated weekly profile stats endpoint`.

---

## TASK 5 — Author identity on timeline posts + comments (api, Part 4)

**Files:**
- Modify: `internal/timeline/handler.go` (`postDTO`/`commentDTO` gain `Author`; `assemblePosts` + `getPost` batch-resolve; new `ProfileResolver` dep)
- Create (or add to handler.go): `internal/timeline/author.go` — `ProfileResolver` interface + `authorDTO` + mapping.
- Modify: `internal/timeline/<constructor>` — add `profiles ProfileResolver` field/param to `NewHandler`.
- Modify: `internal/server/server.go` — pass an adapter over the existing profile-summary provider.
- Tests: `internal/timeline/*_test.go`

**Context:** `assemblePosts` (handler.go) already batch-loads hydration, reaction summaries, and comment counts over a page's post IDs — add a batch author resolve over the page's **distinct `post.UserID`**. `getPost` lists comments via `h.repo.ListComments` then maps with `toCommentDTO` — add a batch author resolve over the thread's **distinct commenter IDs**. The existing summary path is `server.followProfileProvider.ProfileSummaries(ctx, userIDs) (map[string]follow.ProfileSummary, error)` which already presigns avatars + OAuth-fallback. Define a timeline-local interface so the package stays decoupled; server provides an adapter.

```go
// internal/timeline/author.go
type Author struct {
    UserID      string
    Username    *string
    DisplayName string
    AvatarURL   *string
}
type ProfileResolver interface {
    // Authors batch-resolves user summaries; missing IDs are simply absent.
    Authors(ctx context.Context, userIDs []string) (map[string]Author, error)
}
type authorDTO struct {
    UserID      string  `json:"user_id"`
    Username    *string `json:"username"`
    DisplayName string  `json:"display_name"`
    AvatarURL   *string `json:"avatar_url"`
}
```
Add `Author authorDTO `json:"author"`` to `postDTO` and `commentDTO`. In `assemblePosts`: gather distinct `post.UserID`, call `h.profiles.Authors`, attach. A post whose author is unresolved should still render (defensive: zero-value author with the known `UserID`) — but in practice the author always exists; log if missing, mirroring the hydration-missing handling. In `getPost`: gather distinct comment `UserID`s, resolve, attach to each `commentDTO`.

**Server adapter:** wrap `followProfileProvider` (which returns `follow.ProfileSummary{UserID,DisplayName,Username,AvatarURL}`) into `timeline.ProfileResolver` mapping to `timeline.Author`. Pass into `timeline.NewHandler`.

- [ ] **Step 1: Write failing tests** with a fake `ProfileResolver` that records the IDs it was asked for:
  - Feed: every `postDTO` has a populated `author` (user_id/display_name/avatar_url) for its post; the resolver is called **once** with the page's distinct author IDs (assert no per-post call — i.e. number of resolver invocations == 1 and it received deduped IDs). 
  - Single post: `getPost` response includes `author` on the post and on **every** comment; commenters resolved in one batch.
  - Delete affordance unaffected (comment still carries `user_id`).
- [ ] **Step 2: Run** → FAIL.
- [ ] **Step 3: Implement** interface, DTO fields, batch resolves, constructor wiring, server adapter.
- [ ] **Step 4: Run** `go test ./internal/timeline/... ./internal/server/...` → PASS. Lint/vet/format/tidy clean. `go build ./...`.
- [ ] **Step 5: Commit** `feat(timeline): embed author identity in posts and comments`.

---

## TASK 6 — Bio on the web (web, Part 2)

**Files:**
- Modify: `lib/api.ts` (`bio` on `ResolvedProfile` + `PublicProfile`; `updateMe` patch already passes arbitrary fields — confirm `bio` is accepted in its patch type)
- Modify: `app/(app)/settings/page.tsx` (new `<BioRow>` following `<DisplayNameRow>`/`<HeightRow>` pattern)
- Modify: `app/(app)/u/[username]/page.tsx` (render bio under name/@handle)
- Tests: `app/(app)/settings/*.test.tsx`, `app/(app)/u/[username]/*.test.tsx`

**Context:** `ResolvedProfile` (api.ts:301-318) is GET/PATCH `/me`; `PublicProfile = ProfileSummary & {follower_count,following_count}` (api.ts:2714-2717). `updateMe(token, patch)` PATCHes `/me`. Settings rows are self-contained components calling profile-context `update({ field })`; `<HeightRow>` is the closest pattern (input + Save, disabled when unchanged/saving). Profile header is in `u/[username]/page.tsx` lines ~130-182. `Avatar` component takes `{url,name,size}`.

**Changes:**
- `ResolvedProfile`: add `bio: string | null`. `PublicProfile`/`ProfileSummary`: add `bio: string | null` to `PublicProfile` (per SOW "add `bio` to `PublicProfile` and `ResolvedProfile`"). Ensure `updateMe`'s patch type permits `bio?: string`.
- `<BioRow>`: a `<textarea>` (3–4 rows) with a live character counter `{count}/160`, hard-capped at 160 (trim/prevent input past 160 runes — use `[...str].length` for rune count, not `.length`). Save button disabled when unchanged or saving; on save call `update({ bio: trimmed })` (send `""` to clear). Match the card chrome class used by other rows (`flex flex-col gap-3 rounded-lg border border-[var(--border)] bg-[var(--surface)] px-4 py-4`).
- Profile header: render bio as a `<p>` under the @handle, only when `profile.bio` is truthy (no placeholder for empty/others).

- [ ] **Step 1: Write failing tests:**
  - `BioRow`: typing past 160 chars is prevented/trimmed and the counter shows `160/160`; Save is disabled when value equals the persisted bio; clicking Save calls `updateMe` with `{bio: ...}`.
  - Profile header: renders the bio text when `profile.bio` is set; renders nothing (no bio element) when `bio` is null.
- [ ] **Step 2: Run** `npm run test` (the new specs) → FAIL.
- [ ] **Step 3: Implement** the type additions, `<BioRow>`, header rendering.
- [ ] **Step 4: Run** `npm run typecheck && npm run lint && npm run test` → green; `npm run build` (routes/components touched).
- [ ] **Step 5: Commit** `feat(settings): add editable bio and show it on the profile`.

---

## TASK 7 — Profile stats charts on the web (web, Part 3)

**Files:**
- Modify: `lib/api.ts` (`getProfileStats` + `ProfileStats` type, mirroring `listUserTimeline`'s locked handling)
- Create: `app/(app)/u/[username]/_components/ProfileStats.tsx` (+ test)
- Create or adapt: a presentational weekly area chart reused by both series (extract a shared component if `workout-duration-chart.tsx`/`running-mileage-chart.tsx` are too coupled — they are: their `summarize()` is bound to `Workout[]`/`RunningSession[]`, so **extract a small shared presentational `WeeklySeriesChart`** taking `{ points: {week_start, value}[]; unit/label; valueFormatter }` rather than forcing the analytics components).
- Modify: `app/(app)/u/[username]/page.tsx` (render `<ProfileStats username={handle} />` below the header)

**Context:** New endpoint returns `{ lift_session_minutes: {week_start,value}[], running_distance_meters: {week_start,value}[], locked?: boolean }`. `listUserTimeline` (api.ts:2946-2962) is the locked-handling precedent: returns the page plus `locked?: boolean`, never throws on locked. `ProfileActivityFeed` shows the locked treatment ("visible to accepted followers") and the empty treatment ("No activity yet"). Charts are recharts; distance unit comes from `useDistanceUnit()`; formatting helpers in `lib/format.ts`/`lib/chart-format.ts`.

**`getProfileStats(token, username)`:**
```ts
type StatsPoint = { week_start: string; value: number };
type ProfileStats = {
  lift_session_minutes: StatsPoint[];
  running_distance_meters: StatsPoint[];
  locked?: boolean;
};
// GET /users/{username}/stats — same Bearer pattern; return parsed body as-is.
```

**`<ProfileStats>`:** fetch on mount (token from auth; 401 → clear + redirect like other pages). States:
- `locked` → same gated message component/treatment as `ProfileActivityFeed`'s locked state ("…visible to accepted followers").
- both series all-zero (no activity) → "No activity yet" empty state (no flat-line chart).
- otherwise → two `WeeklySeriesChart`s: lift-session minutes (format minutes) and running distance (convert meters → user distance unit via existing helpers).

**`WeeklySeriesChart`** (presentational): props `{ points: StatsPoint[]; label: string; emptyHint?: string; valueFormatter: (v:number)=>string; unitLabel?: string }`. Recharts `AreaChart` over `points` keyed `week_start`→x, `value`→y. Keep styling consistent with existing charts (same axis/tooltip treatment).

- [ ] **Step 1: Write failing tests** `ProfileStats.test.tsx` with a mocked `getProfileStats`:
  - Renders both charts from a stats fixture (assert two chart regions / titles present).
  - `locked: true` → renders the gated message, no charts.
  - all-zero series → renders "No activity yet", no charts.
- [ ] **Step 2: Run** → FAIL.
- [ ] **Step 3: Implement** wrapper + types, `WeeklySeriesChart`, `ProfileStats`, page wiring.
- [ ] **Step 4: Run** `npm run typecheck && npm run lint && npm run test && npm run build` → green.
- [ ] **Step 5: Commit** `feat(profile): add weekly lift and running graphs to the profile`.

---

## TASK 8 — Author avatars on timeline cards + comments (web, Part 4)

**Files:**
- Modify: `lib/api.ts` (`author: ProfileSummary` on `TimelinePost` + `TimelineComment`)
- Modify: `app/(app)/timeline/_components/TimelinePostCard.tsx` (author header)
- Modify: `app/(app)/timeline/_components/CommentThread.tsx` (author rows)
- Tests: co-located `*.test.tsx`

**Context:** API now embeds `author {user_id, username, display_name, avatar_url}` on each post and comment (Task 5). `ProfileSummary` (api.ts:2702-2708) already has those fields (minus `relationship`, which the author object omits — define `author` as `{user_id; username: string|null; display_name; avatar_url: string|null}` rather than full `ProfileSummary`, since the API author DTO has no relationship). `TimelinePostCard` header (lines 36-42) shows source emoji/label + time; `CommentThread` rows (lines 119-146) are bare text + delete (keyed `c.user_id === currentUserId`). `Avatar` takes `{url,name,size}`. Link target `/u/{username}` only when `username` is non-null (it now always is post-backfill, but guard anyway).

**Changes:**
- Types: add `author: { user_id: string; username: string | null; display_name: string; avatar_url: string | null }` to `TimelinePost` and `TimelineComment`.
- `TimelinePostCard`: render `<Avatar url={post.author.avatar_url} name={post.author.display_name} size={32}/>` + display name (linking to `/u/{post.author.username}` when present) as the **primary** header line; keep the existing source emoji/label + time as a **secondary** line.
- `CommentThread`: each row renders the commenter's `<Avatar size={24}>` + display name beside the body; preserve the own-comment delete affordance (still `c.user_id === currentUserId`).

- [ ] **Step 1: Write failing tests:**
  - Post card renders the author's display name and links to `/u/{handle}`; secondary line still shows the source label.
  - Comment rows render each commenter's identity (name/avatar); delete button shows only on own comments (`user_id === currentUserId`), still works.
- [ ] **Step 2: Run** → FAIL.
- [ ] **Step 3: Implement** type + component changes. Update any existing post/comment test fixtures to include the new `author` field so the suite stays green.
- [ ] **Step 4: Run** `npm run typecheck && npm run lint && npm run test && npm run build` → green.
- [ ] **Step 5: Commit** `feat(timeline): show author avatars on cards and comments`.

---

## Self-review (spec coverage)

- Part 1 reachability → Tasks 1 (generator), 2 (signup + backfill migration). ✓
- Part 2 bio → Task 3 (api), Task 6 (web). ✓
- Part 3 graphs/stats → Task 4 (api endpoint + privacy + bucketing), Task 7 (web charts + locked + empty). ✓
- Part 4 author avatars → Task 5 (api DTO enrichment + batch), Task 8 (web post card + comments). ✓
- Tests called out per SOW section are encoded in each task's Step 1. ✓
- No account-level privacy flag exists; the stats gate mirrors the scoped feed (self/accepted-follower → full, else locked) — documented in Task 4. ✓
- Rollout: Part 1 independent and first; Parts 2–4 independent — reflected in commit/PR ordering and the docs PR deployment section.

## Sequencing for execution

Implement in task order 1→8 (API parts 1–4 then web parts 2–4). API and web are separate repos/branches (`feat/social-profiles-discovery-and-engagement` in each). Web tasks depend on the API DTO shapes being defined (they're additive and the web can be built/tested against fixtures regardless). Run each repo's full local gate before pushing.
