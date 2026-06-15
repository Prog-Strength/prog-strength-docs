# Implementation Plan: Followers, Profile Search & Social Timeline

**SOW**: `sows/followers-profile-search-and-social-timeline.md`
**Created**: 2026-06-15
**Repos**: prog-strength-api, prog-strength-web, prog-strength-mobile, prog-strength-docs

This plan turns the SOW into discrete, subagent-sized tasks. Each task is
implemented by a subagent, then spec-reviewed and code-quality-reviewed
before the next begins. Build order follows the SOW's Rollout: API
username → follow graph → lists+search → timeline fan-out → web → mobile.

The whole feature is **additive** and rides the single-writer invariant
(every mutation/read goes through the Go API; MCP/agent untouched in v1).

---

## Repo facts (verified against the cloned repos)

- Latest API migration is `020_timeline.sql` (the SOW says "after 019" but
  019 is steps; timeline landed as 020). **Next numbers: 021, 022, 023.**
- Domain anatomy (per `internal/timeline`, `internal/user`): `errors.go`,
  `metrics.go` (only where a silent failure needs metering), `model.go`,
  `repository.go` (context-first interface), `sqlite_repository.go`,
  `memory_repository.go` (`sync.RWMutex`, defensive copies),
  `handler.go` with `Mount(r chi.Router)`, and `*_test.go`.
- Auth: `authctx.UserIDFrom(ctx)` inside handlers; routes mounted under the
  `r.Group` that applies `auth.RequireUser`.
- Responses: `httpresp.OK/Created/Error/ErrorWithCode/ServerError`.
- IDs: `id.New()`.
- Cross-domain seams are interfaces **defined in the consuming domain** and
  implemented as adapters in `internal/server/*.go` (see
  `timeline_publisher.go`, `timeline_hydrator.go`). This is exactly how
  `AcceptedFollowees` gets injected into timeline.
- SQLite test DB: `db.Migrate(conn)` over a temp file (`newMigratedDB(t)`
  pattern). Handler tests build requests with `authctx.WithUserID` + chi
  route context.
- Web: Next.js 16 App Router, all API calls via `lib/api.ts`, pages are
  `fetch + useState`, page-private `_components/`. Settings at
  `app/(app)/settings`, Timeline at `app/(app)/timeline`. **"This is NOT
  the Next.js you know"** — consult `node_modules/next/dist/docs/` and
  `docs/superpowers/specs/` patterns.
- Mobile: Expo Router under `app/`, nativewind, shared client in `lib/api.ts`.

---

## Task 1 — API: Username

**Migration `021_user_username.sql`**
```sql
ALTER TABLE users ADD COLUMN username TEXT;
CREATE UNIQUE INDEX idx_users_username ON users(username);
```
(`username` is stored lowercased; uniqueness is therefore case-insensitive
by construction. NULL until set; SQLite allows multiple NULLs in a UNIQUE
index, so unset users don't collide.)

**Validator + reserved list** — new `internal/user` code (e.g.
`username.go`), exported `ValidateUsername(string) (canonical string, error)`:
- lowercase input first, then match `^[a-z][a-z0-9_]{2,29}$` (3–30 chars).
- reject reserved names (denylist constant): `me, api, admin, settings,
  search, timeline, profile, profiles, users, user, support, prog,
  progstrength, prog-strength, follows, followers, following, login,
  logout, auth, oauth, health, metrics` (+ the SOW's examples). Keep the
  list beside the validator.
- typed errors: `ErrUsernameInvalid`, `ErrUsernameReserved`,
  `ErrUsernameTaken` (collision surfaces from the unique index).

**Write path** — extend `PATCH /me` (`updateMeRequest`) with
`username *string`. On set: validate → canonicalize → persist; map
collision (sqlite unique constraint) → 409, invalid/reserved → 400.
Rename is allowed (no redirect; old handle frees immediately — documented).
Add `username` to the `User` model + scan columns + `meResponse`.

**Availability** — `GET /users/{username}` already returns 404 when no such
profile (Task 3 builds the full profile); for Task 1 the Settings
availability probe can rely on that. If Task 3 isn't merged yet, the web
debounce still works against the username write returning 409. (Keep the
availability behavior consistent: a free, valid handle → not-found/available.)

**Tests**: charset/length/leading-char, case-folding collision
(`JimLifts` == `jimlifts`), reserved rejection, rename frees old handle,
unique-collision → taken. Both repo backends.

---

## Task 2 — API: Follow graph + state machine

**Migration `022_follows.sql`**
```sql
CREATE TABLE follows (
    id          TEXT PRIMARY KEY,
    follower_id TEXT NOT NULL,
    followee_id TEXT NOT NULL,
    status      TEXT NOT NULL,
    created_at  DATETIME NOT NULL,
    accepted_at DATETIME,
    CHECK (status IN ('pending','accepted')),
    CHECK (follower_id <> followee_id),
    UNIQUE (follower_id, followee_id)
);
CREATE INDEX idx_follows_followee_status ON follows(followee_id, status);
CREATE INDEX idx_follows_follower_status ON follows(follower_id, status);
```

**`internal/follow` domain** — full anatomy. Repository interface:
- `Request(ctx, followerID, followeeID) (Follow, error)` — guard: no row
  exists, not self, followee exists (existence checked in handler/repo via
  user repo or a passed-in resolver), pending-cap. Insert `pending`.
- `Accept(ctx, followeeID, followerID) error` — pending row addressed to
  actor → accepted + stamp `accepted_at`.
- `Reject(ctx, followeeID, followerID) error` — delete pending row.
- `Cancel(ctx, followerID, followeeID) error` — delete own pending row.
- `Unfollow(ctx, followerID, followeeID) error` — delete own accepted row.
- `RemoveFollower(ctx, followeeID, followerID) error` — delete accepted row
  where actor is followee.
- `Get(ctx, followerID, followeeID) (Follow, error)` — for relationship.
- `CountPending(ctx, followerID) (int, error)` — cap enforcement.
- `AcceptedFollowees(ctx, viewerID) ([]string, error)` — feed projection
  (this is the method the timeline interface will call).
- `ListFollowers(ctx, followeeID, accepted, limit, cursor)` /
  `ListFollowing(...)` — keyset paginated (Task 3 consumes; can live here).
- `ListRequests(ctx, viewerID, direction, limit, cursor)` — inbox.
- `CountFollowers/CountFollowing(ctx, userID) (int, error)` — profile counts.
- `Relationship(ctx, viewerID, otherID) (Relationship, error)` — returns
  `none | requested | pending_incoming | following | self`.

Typed errors: `ErrNotFound, ErrInvalidState, ErrSelfFollow,
ErrAlreadyExists, ErrPendingCapExceeded`. Pending cap = const (200).

`metrics.go` optional. Both sqlite + memory backends.

**Handler / routes** (under auth group). Bodies accept `username|user_id`:
- `POST /follows` `{ "followee": "<username|id>" }` → 201; 400 self,
  404 unknown, 409 existing, 429 cap.
- `POST /follows/{username}/accept` → 200
- `POST /follows/{username}/reject` → 204
- `DELETE /follows/{username}` → 204 (cancel if pending-as-follower,
  unfollow if accepted)
- `DELETE /followers/{username}` → 204 (remove-follower)
- `GET /follows/requests?direction=incoming|outgoing&limit=&cursor=` → 200,
  each row carries requester/target public profile summary + relationship.

The handler needs to resolve `{username}` → user_id and check followee
existence; inject a small user-lookup interface (`UserResolver`) so follow
doesn't import user internals (or accept the user repo via an interface).

**Wire into `internal/server/server.go`**: construct follow repo (sqlite/
memory), mount handler in the auth group.

**Tests**: every transition + guard, UNIQUE/self-follow rejection,
re-request after delete, pending-cap, accepted-followee + follower/following
projections, requests-inbox query; handler authorization (only followee
accepts, only follower cancels, third party can't mutate). Both backends.

---

## Task 3 — API: Profile, lists, search

Depends on Tasks 1 & 2. Mostly read endpoints; reuse follow repo +
user repo. Avatar URL reuses the Profile SOW presign path (`avatar_store`).

- `GET /users/{username}` → public profile: `username, display_name,
  avatar_url (presigned), follower_count, following_count, relationship`.
  Always viewable; **no** activities. 404 if unknown username.
- `GET /users/{username}/followers?limit=&cursor=` → keyset list of accepted
  followers; each entry = public profile summary + viewer's relationship to
  that user.
- `GET /users/{username}/following?limit=&cursor=` → same for followees.
- `GET /users/search?q=&limit=&cursor=` → ranked profile search:
  1. exact username (`username = q`)
  2. prefix username (`username LIKE q || '%'`)
  3. substring display_name (`lower(display_name) LIKE '%'||q||'%'`)
  ties → follower_count desc then username; opaque cursor = (rank bucket +
  tiebreak key). Returns public summaries + relationship; includes the
  searcher (`relationship: self`); no block filtering (none exists).

A shared `profileSummary` DTO + a relationship-computation helper is reused
across follow lists, search, and profile. Decide ownership: simplest is a
small profile/discovery handler in `internal/user` that depends on the
follow repo via an interface, OR extend the follow handler. **Pick the
boundary that avoids import cycles** — `user` may depend on a follow
read-interface injected from server wiring (mirrors the AcceptedFollowees
pattern). Document the choice in the PR.

**Tests**: ranking order, cursor stability across pages, case-insensitivity,
relationship correctness per row, profile counts, unknown username → 404.

---

## Task 4 — API: Timeline fan-out

Depends on Task 2 (AcceptedFollowees).

**Migration `023_timeline_default_visibility.sql`**
```sql
-- Flip default for new posts and widen already-backfilled rows so accepted
-- followers see the author's entire history. Safe: no private-marking UI
-- existed before this SOW, so every 'private' row is a backfilled default.
UPDATE timeline_post SET visibility = 'friends' WHERE visibility = 'private';
```
Also change the column default to `'friends'` for new inserts. SQLite can't
`ALTER COLUMN` a default cheaply; the cleaner path is to set the default at
the **write edge** (`EnsurePost` / the post-creation code now writes
`'friends'`) and keep the migration as the data backfill. Verify where the
visibility literal is set on insert in `timeline/sqlite_repository.go` +
`memory_repository.go` and the publisher; change `private` → `friends`
there. (If a table default truly must change, do it via table-rebuild;
prefer the write-edge approach + data migration.)

**AcceptedFollowees interface** — define in `internal/timeline`:
```go
type AcceptedFollowees interface {
    AcceptedFollowees(ctx context.Context, viewerID string) ([]string, error)
}
```
Adapter in `internal/server/` over the follow repo; inject into
`timeline.NewHandler` and/or the repo's `ListFeed`.

**ListFeed widen** — `WHERE user_id IN (:viewer ∪ accepted-followees)`.
Fetch the followee set once per page (handler calls `AcceptedFollowees`,
passes the id set into a new repo signature, e.g.
`ListFeed(ctx, userIDs []string, limit, before)`), keeping the existing
keyset index `(user_id, occurred_at DESC, id DESC)`. Build the `IN (?,?..)`
placeholder list dynamically (mirror the existing dynamic-clause style).

**canView** — one added clause:
```
canView(post, viewer) =
    post.user_id == viewer
 || (post.visibility != 'private' && acceptedFollow(viewer, post.user_id))
```
The handler resolves `acceptedFollow` from the followee set it already
fetched (or a point lookup). `canModerate` unchanged. Non-follower or
follower-on-private → existing 404/locked.

**`GET /timeline?user=<username>`** — scoped single-author feed for the
profile activity tab. Resolve username → id; if viewer is the author or an
accepted follower, return that author's non-private posts (newest-first,
hydrated); otherwise the gated locked/empty state.

**Tests** (in `internal/timeline`, fake `AcceptedFollowees`): accepted
follower sees `friends` not `private`; pending (non-accepted) sees nothing;
unfollow/remove revokes immediately; own-post visibility unchanged;
multi-author keyset ordering correct across authors; `?user=` gating.

---

## Task 5 — Web: API client + username in Settings

- `lib/api.ts`: add typed functions following the existing `unwrap`/Bearer
  pattern — `getProfile(username)`, `searchProfiles(q, cursor)`,
  `listFollowers/listFollowing(username, cursor)`, `requestFollow(followee)`,
  `acceptFollow/rejectFollow/cancelFollow/unfollow(username)`,
  `removeFollower(username)`, `listFollowRequests(direction, cursor)`,
  `listUserTimeline(username, before)`. Add the TS types
  (`PublicProfile`, `ProfileSummary`, `Relationship`, `FollowRequest`, etc.).
- Settings page (`app/(app)/settings`): username field with **debounced
  availability check** (probe `getProfile` → 404 = available, 200 = taken;
  or a dedicated availability call), inline charset/reserved validation
  mirroring the server validator, helper text noting the rename trade-off
  (old URL stops working). Optimistic-safe save via `updateMe`.
- vitest: username validation/availability UI states.

---

## Task 6 — Web: Profile page, search, requests inbox, lists

- `app/(app)/u/[username]/page.tsx`: avatar, display name, @username,
  follower/following counts (link to lists), context-sensitive primary
  action (`Follow`/`Requested`/`Following`/`Edit profile` when self), and an
  **activities tab** rendering `listUserTimeline` or a locked/empty state
  for non-followers. Reuse Timeline post-card components where possible.
- Search: a search entry (sidebar/top area) → `searchProfiles`, ranked
  rows with inline follow buttons.
- Requests inbox: surface (timeline header affordance or dedicated route)
  listing incoming pending requests with Accept/Reject; outgoing "Requested"
  indicator wherever a profile/search row is pending.
- Followers/following list views reachable from profile counts; each row
  carries its own relationship action.
- Follow/accept actions are **optimistic** with rollback on error
  (mirror the Timeline ReactionBar optimistic pattern). Reuse toast/modal.
- vitest: follow-button state machine (none→requested→following + teardown),
  search results rendering, inbox accept/reject.

---

## Task 7 — Mobile: parity surface

Same surface as web, Expo Router under `app/`, nativewind styling, shared
`lib/api.ts` client. Screens: profile (own + others) with counts +
context-sensitive action + gated activities tab; search; requests inbox;
followers/following lists; username field in settings. Optimistic actions
mirror web. Component/render tests per repo conventions. Follow the existing
mobile parity workflow (research the matching web surface → mirror it).

---

## Task 8 — Docs + PRs

- In `prog-strength-docs`, on `feat/followers-profile-search-and-social-timeline`:
  flip SOW frontmatter `status: shipped`, body `**Status**: Shipped`,
  `**Last updated**: 2026-06-15`. Commit
  `docs: mark followers-profile-search-and-social-timeline as shipped`.
- Push every feature branch; `gh pr create` per modified repo.
- Docs PR body uses the required template (summary, implementation PR
  bullets in repos-frontmatter order, deployment order with reasons,
  verification steps). Capture each implementation PR URL.

**Deployment order**: api (migrations + endpoints) → web → mobile. Within
api the four tasks ship as one PR/branch (sequential migrations 021–023).
Web 4xx until api deploys; mobile likewise.
