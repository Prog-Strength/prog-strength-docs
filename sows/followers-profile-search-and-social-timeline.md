---
status: proposed
repos:
  - prog-strength-api
  - prog-strength-web
  - prog-strength-mobile
  - prog-strength-docs
---

# Followers, Profile Search & Social Timeline

**Status**: Proposed · **Last updated**: 2026-06-14

## Introduction

**Prog Strength** just grew a private **Timeline** — a reverse-chronological feed of the signed-in user's own workouts, runs, PRs, and running best efforts, with comments and stackable reactions already built and tested against a single author. That SOW (`user-timeline-feed.md`) was deliberately the seed of a social feature: it shipped the interaction primitives a multi-author feed depends on, and it called out exactly what the next step would add — "follower/following tables, follow request/accept flow, the multi-author feed query, per-post visibility, and notifications." This SOW is that next step.

After this work ships, a lifter can **find another user** by display name or username, **request to follow** them, and — once that request is accepted — see that person's training surfaced on their own Timeline alongside their own. Following is **one-way and approval-gated**: A requesting to follow B, and B accepting, lets A see B's activities; it says nothing about whether B follows A. Reciprocity is a separate request in the other direction. Users get a **public profile** keyed by a new, user-settable **username**, **followers/following lists**, and a **requests inbox** where incoming follow requests wait to be accepted or rejected.

This is, by design, mostly graph plumbing. The Timeline already author-keys every post (`timeline_post.user_id`), already carries a `visibility` column, and already isolates the single authorization decision (`canView`) the social feed needs to change. The hard, easy-to-get-wrong parts — post identity, comments, reactions, pagination — are done. What remains is the follow graph, a username, discovery, and widening one feed query from "my posts" to "my posts ∪ the posts of people I've been accepted to follow."

## Proposed Solution

Four additive pieces, all behind the **single-writer invariant** (every mutation, search, and feed read goes through the Go API; the MCP server is not involved in v1):

1. **Username** — a new first-class, user-settable, unique field on the existing `users` table, decoupled from OAuth exactly as `display_name` already is. It is the stable, shareable handle a profile is keyed and searched by. This extends the shipped **User Profile & Preferences** work rather than introducing a new identity system.

2. **Follow graph + state machine** — a single `follows` table with a `status` enum (`pending` → `accepted`) and a small set of guarded transitions (request / accept / reject / cancel / unfollow / remove-follower). The relationship is directional; reciprocity is two independent rows.

3. **Discovery surfaces** — profile-by-username, profile **search** by display name or username, and **followers/following lists**, all served from the Go API and rendered on web and mobile.

4. **Timeline fan-out** — **read-time**: the existing feed query widens from `WHERE user_id = :viewer` to `WHERE user_id IN (:accepted-followees ∪ :viewer)`, and the existing `canView` helper gains one clause — an accepted follower may see a followee's non-private posts. No new feed table, no materialization, no fan-out-on-write. For a small SQLite-backed personal app this is the correct and far simpler model; the keyset index the Timeline already built (`(user_id, occurred_at DESC, id)`) serves the multi-author query directly.

A user's **profile is publicly discoverable** — anyone can find it and see avatar, display name, @username, and follower/following counts — but their **activities are gated**: the Timeline of another user is visible only to an accepted follower. **Followers/following lists are public.** A **requests inbox** (the set of `pending` rows addressed to the viewer) gives the target an in-app place to see and act on incoming requests without building a notification system. **Blocking, notifications, and follow-request rate-limiting are deferred**, with the schema deliberately leaving room for each.

Web gets the full surface; mobile follows with the same surface as a parity phase. The agent does not read or write the social graph in v1.

## Goals and Non-Goals

### Goals

- A **username** on every user: unique, case-insensitive, user-settable from Settings, decoupled from OAuth, validated against a charset and a reserved-name denylist.
- A **directional, approval-gated follow graph** with a complete, guarded state machine: **request**, **accept**, **reject**, **cancel** (requester withdraws a pending request), **unfollow** (follower ends an accepted relationship), **remove-follower** (followee ejects an accepted follower).
- A **requests inbox**: the target sees incoming `pending` requests and can accept or reject them.
- A **public profile** keyed by username — avatar, display name, @username, follower/following counts visible to anyone; **activities gated** behind an accepted follow.
- **Profile search** by display name or username, ranked and paginated.
- **Public followers/following lists** per user, paginated.
- **Read-time timeline fan-out**: an accepted follower sees the followee's **entire** shareable history (newest-first, hydrated from source), interleaved with their own, via a one-clause change to `canView` and a widened feed query.
- **Per-post privacy** retained through the existing `visibility` column: posts are shareable-to-followers by default, with `private` as an owner-only escape hatch covering any source type.
- The same surface on **web and mobile**, reusing each platform's shared API client.

### Non-Goals

- **Blocking.** v1 ships remove-follower (eject an accepted follower) but no block relationship. Schema leaves room; blocking is its own follow-up.
- **Notifications** (push or in-app). Incoming requests and acceptances are surfaced by *querying* state (the requests inbox), not by a notification system. Deferred; the `pending`/`accepted` states make the schema notification-ready.
- **Follow-request rate-limiting.** v1 enforces only a soft cap on outstanding pending requests (abuse backstop); real rate-limiting is deferred.
- **MCP / agent access** to the social graph. The agent does not answer "who follows me?" or post on a user's behalf in v1, consistent with the Timeline SOW.
- **Auto-accept / public-account mode.** Every follow requires explicit acceptance in v1; there is no "anyone can follow me without approval" toggle.
- **Mutual-follow ("friends") semantics, DMs, mentions, sharing externally.** Out of scope.
- **FTS5 / fuzzy search.** v1 uses SQLite `LIKE` prefix/substring matching; full-text or typo-tolerant search is a later concern if discovery volume justifies it.
- **New activity-generation.** The Timeline remains the activity source; this SOW only gates who sees those existing activities.

## Implementation Details

### Data Model (`prog-strength-api`)

**Username column** on the existing `users` table (new migration, next in sequence after the Timeline's `019_timeline.sql`):

| Column | Type | Description |
| --- | --- | --- |
| `username` | `TEXT` | Canonical, lowercased handle. Nullable until first set (existing users have none until they pick one). `UNIQUE`, case-insensitive (store lowercase; reject on case-insensitive collision). |

Constraints, enforced at the API write edge (not only the DB):

- **Charset & shape:** `^[a-z][a-z0-9_]{2,29}$` — 3–30 chars, must start with a letter, lowercase letters / digits / underscore only. Input is lowercased before validation and storage, so `@JimLifts` and `@jimlifts` are the same handle and collide.
- **Reserved-name denylist:** route-colliding and impersonation-risk names (`me`, `api`, `admin`, `settings`, `search`, `timeline`, `profile`, `users`, `support`, `prog`, `progstrength`, …) are rejected. The list lives in code, alongside the validator.
- **Rename:** allowed from Settings. A rename changes the profile URL with **no redirect from the old handle** in v1 (documented trade-off; old links 404). The freed handle becomes immediately available to others — acceptable at this scale; revisit if squatting/impersonation becomes real.

**`follows`** — the follow graph. New migration (a single `0NN_follows.sql`, next after the username migration):

- `id TEXT PRIMARY KEY` — uses the existing `internal/id` generator.
- `follower_id TEXT NOT NULL` — the user who wants to subscribe (the requester).
- `followee_id TEXT NOT NULL` — the user being followed (the approver / activity author).
- `status TEXT NOT NULL` — `CHECK (status IN ('pending','accepted'))`.
- `created_at TIMESTAMP NOT NULL` — when the request was made.
- `accepted_at TIMESTAMP NULL` — stamped on transition to `accepted`; null while pending.
- `CHECK (follower_id <> followee_id)` — no self-follow.
- `UNIQUE (follower_id, followee_id)` — at most one relationship per ordered pair; makes request idempotent-ish (a duplicate request is a conflict, not a second row).
- Index `(followee_id, status)` — serves "my followers" and the requests inbox (`status='pending'`).
- Index `(follower_id, status)` — serves "who I follow" and the accepted-followee set for the feed query.

**Terminal actions delete the row** rather than recording terminal statuses. `reject`, `cancel`, `unfollow`, and `remove-follower` all remove the `follows` row, returning the pair to "no relationship" and allowing a fresh request later. This keeps the enum to two live states and the queries trivial; it deliberately forgoes a rejection-history audit trail (not needed in v1, and re-request friction is better solved by the pending cap than by sticky `rejected` rows). The status enum is left open for a future `blocked` value without migration of existing rows.

Single-writer invariant holds: all of this is mutated only through the Go API.

### Follow State Machine (`prog-strength-api`)

A new `internal/follow` domain, following the established domain anatomy (`model.go`, `errors.go`, `metrics.go`, `repository.go`, `sqlite_repository.go`, `memory_repository.go`, `handler.go` with `Mount(r chi.Router)`, and matching `*_test.go`), registered in `internal/server/server.go` alongside the others. The repository interface and the transitions it guards:

| Action | Actor | Precondition | Effect |
| --- | --- | --- | --- |
| **Request** | follower | no existing row for `(follower, followee)`; followee exists; not self | insert `status='pending'` |
| **Accept** | followee | a `pending` row addressed to the actor | `pending → accepted`, stamp `accepted_at` |
| **Reject** | followee | a `pending` row addressed to the actor | delete row |
| **Cancel** | follower | own `pending` row | delete row |
| **Unfollow** | follower | own `accepted` row | delete row |
| **Remove-follower** | followee | an `accepted` row where actor is followee | delete row |

Every transition validates the actor's relationship to the row (`authctx.UserIDFrom(ctx)` must be the correct side) and returns a typed error (`ErrNotFound`, `ErrInvalidState`, `ErrSelfFollow`, `ErrAlreadyExists`, `ErrPendingCapExceeded`) mapped to HTTP status by the handler. Re-requesting after a reject is allowed (the row was deleted), bounded only by the pending cap below.

**Pending-request cap (abuse backstop).** A request is rejected with `ErrPendingCapExceeded` when the requester already has more than a configured number of outstanding `pending` rows (e.g. 200). This is a coarse backstop, not rate-limiting; it caps unbounded request-spam without per-time-window accounting. The number is a constant in the domain, not a per-user setting.

Both `sqlite_repository.go` and `memory_repository.go` implement the interface, matching the dual-implementation pattern the other domains use for fast unit tests.

### API Surface (`prog-strength-api`)

All routes behind the existing auth middleware; the actor is `authctx.UserIDFrom(ctx)`. Responses use the `httpresp` helpers exactly as the existing handlers do.

**Follow graph**

- **`POST /follows`** — body `{ "followee": "<username|user_id>" }`. Creates a `pending` request. `201`. Errors: self-follow (`400`), unknown followee (`404`), existing relationship (`409`), pending cap (`429`).
- **`POST /follows/{username}/accept`** — followee accepts a pending request from `{username}`. `200`.
- **`POST /follows/{username}/reject`** — followee rejects a pending request. `204`.
- **`DELETE /follows/{username}`** — context-sensitive teardown by the actor: **cancel** if the actor is the follower on a pending row, **unfollow** if accepted. `204`.
- **`DELETE /followers/{username}`** — **remove-follower**: the actor (followee) ejects an accepted follower `{username}`. `204`.
- **`GET /follows/requests`** — the **requests inbox**: `pending` rows addressed to the viewer (incoming), paginated, each carrying the requester's public profile summary. (A `?direction=outgoing` variant lists the viewer's own pending requests so the UI can show "Requested" state.)

**Lists & relationship state**

- **`GET /users/{username}/followers`** — keyset-paginated list of accepted followers (public). Each entry is a public profile summary plus the viewer's own relationship-to-that-user state (so follow buttons render correctly).
- **`GET /users/{username}/following`** — keyset-paginated list of accepted followees (public).
- The viewer's relationship to any listed/searched user is expressed as a small `relationship` field — one of `none | requested | pending_incoming | following | self` — computed per row so every surface can render the right action affordance without extra round-trips.

**Profile & search**

- **`GET /users/{username}`** — the public profile: `username`, `display_name`, `avatar_url` (presigned, reusing the Profile SOW's serving path), `follower_count`, `following_count`, and the viewer's `relationship`. Always viewable (profiles are public); it does **not** include activities.
- **`GET /users/search?q=&limit=&cursor=`** — profile search (below).

**Timeline** — the existing `GET /timeline` and post/comment/reaction routes are unchanged in shape; only the feed query and `canView` change (below). A new optional `GET /timeline?user=<username>` scopes the feed to a single author's posts for rendering a profile's activity tab, returning the gated empty/locked state when the viewer is not an accepted follower.

### Profile Search (`prog-strength-api`)

SQLite `LIKE`-based matching over the `users` table — **no FTS5** at this scale. The query `q` is lowercased and matched against `username` (prefix) and `display_name` (case-insensitive substring). Results are ranked:

1. **Exact** username match (`username = q`)
2. **Prefix** username match (`username LIKE q || '%'`)
3. **Substring** display-name match (`lower(display_name) LIKE '%' || q || '%'`)

ties broken by follower_count then username, paginated by an opaque cursor (rank bucket + tiebreak key). A small index supports the username prefix scan; display-name substring is a scan bounded by the result `limit` (acceptable at this scale — noted for revisit if the user table grows). Search returns public profile summaries with the viewer's `relationship` field; it does **not** filter by any block relationship (none exists in v1) and surfaces the searcher's own record too (harmless; `relationship: self`).

### Timeline Fan-out (`prog-strength-api`)

This is the one-clause change the Timeline SOW pre-scaffolded. Two touch-points in `internal/timeline`, both already isolated:

1. **Feed query.** `ListFeed` widens from `WHERE user_id = :viewer` to `WHERE user_id IN (:viewer ∪ :accepted-followees) AND <visibility allows>`. The accepted-followee set is the `follower_id = :viewer AND status = 'accepted'` projection of `follows`, fetched once per feed page (small; a follow list, not a fan-out). The existing keyset index `(user_id, occurred_at DESC, id)` serves the `IN` query directly. The cross-domain dependency is injected as a small interface (`AcceptedFollowees(ctx, viewerID) ([]string, error)`) provided by `internal/follow`, so the timeline package never imports the follow internals — matching the `SourceHydrator`/`Publisher` boundary pattern already in place.

2. **`canView`.** Gains one clause. v1 Timeline: `post.user_id == viewer`. Now:

   ```
   canView(post, viewer) =
       post.user_id == viewer                                  // own post
    || (post.visibility != 'private'                           // shareable, AND
        && acceptedFollow(follower=viewer, followee=post.user_id))   // viewer follows author
   ```

   `canModerate(comment, viewer)` is unchanged (`comment.user_id == viewer`). A non-follower, or a follower viewing a `private` post, gets the existing `404`/locked behavior.

**Per-post visibility & shareable history.** New posts now default to `visibility='friends'` (shareable to accepted followers) rather than `'private'`; existing rows seeded by the Timeline backfill are migrated from `'private'` to `'friends'` in the same migration so a user's accepted followers see their **entire** history (the chosen backfill / whole-history semantics), not only posts from the acceptance point forward. `'private'` remains available as an owner-only escape hatch on any post or source type — this is the lever for "make this one activity private," and covers the "are any activity types private by default?" question: none are, but any can be made so. (A per-source-type default-visibility preference is explicitly out of scope; all four source types — workout, run, PR, best effort — are shareable by default.)

Because fan-out is read-time and content is hydrated from source, there is **no feed materialization, no fan-out worker, and no backfill of feed rows** for the social feature. Accepting a follow is a single row update; the follower's next feed load simply includes a wider `user_id` set.

### Web (`prog-strength-web`)

> **Critical implementer note:** the web repo's `AGENTS.md` / `CLAUDE.md` are explicit — **"This is NOT the Next.js you know."** Consult the relevant guide in `node_modules/next/dist/docs/` and the validated page patterns in `docs/superpowers/specs/` before writing code. The Timeline page (`app/(app)/timeline/`) and the Settings page from the Profile SOW are the structural references.

- **Username in Settings** — add a username field to the existing Settings page (from the Profile SOW): availability-checked input (debounced `GET /users/{username}` / a lightweight availability probe), charset/reserved-name validation surfaced inline, with the rename trade-off (old URL stops working) noted in helper text.
- **Profile page** — `app/(app)/u/[username]/page.tsx`: avatar, display name, @username, follower/following counts (each linking to the list), a context-sensitive primary action (`Follow` / `Requested` / `Following` / `Edit profile` when self), and an **activities tab** that renders the gated Timeline (`GET /timeline?user=…`) or a locked/empty state for non-followers.
- **Search** — a profile search entry (search field in the sidebar/top area) backed by `GET /users/search`, rendering ranked result rows with inline follow buttons.
- **Requests inbox** — a surface (e.g. a `/timeline` header affordance or a dedicated route) listing incoming pending requests with Accept / Reject, plus an outgoing "Requested" indicator wherever a profile/search row shows a pending state.
- **Followers / following lists** — paginated list views reachable from the profile counts, each row carrying its own relationship action.
- **API client** — add follow/profile/search functions to the web client layer in `lib/`, following the existing client pattern (the same one Timeline/Activities/Nutrition use). Reuse existing toast/modal primitives; follow/accept actions are **optimistic** against their endpoints.

### Mobile (`prog-strength-mobile`)

Mobile ships the same surface, following the established per-phase parity workflow (research → plan → subagent) noted in the project's mobile parity tracking. Screens mirror the web set under the Expo Router `app/` tree with nativewind/Tailwind styling consistent with the repo:

- A **profile screen** (own + others) with avatar/name/@username, counts, the context-sensitive follow action, and the gated activities tab.
- **Search**, **requests inbox**, and **followers/following list** screens.
- A **username** field added to the mobile profile/settings surface.
- Components reuse the shared mobile API client in `lib/`; relationship state and optimistic follow actions mirror web.

The SOW provides the API contract; the dispatcher may split mobile into its own implementation phase after API + web land (see Rollout).

### Tests

- **`internal/follow` repository** (`sqlite` + `memory`): every transition and its guards — request/accept/reject/cancel/unfollow/remove; `UNIQUE` and self-follow rejection; re-request after delete; pending-cap enforcement; the accepted-followee and follower/following projections; requests-inbox query.
- **`internal/follow` handler:** each endpoint's status codes and authorization — e.g. only the followee can accept; only the follower can cancel; remove-follower requires an accepted row addressed to the actor; a third party cannot mutate someone else's relationship.
- **Username validation:** charset, length, leading-char, case-folding collisions, reserved-name rejection, rename freeing the old handle.
- **Search:** ranking order (exact > prefix > substring), pagination cursor stability, case-insensitivity.
- **Timeline fan-out (in `internal/timeline`):** with a fake `AcceptedFollowees` provider — an accepted follower sees the followee's `friends` posts and **not** their `private` posts; a `pending` (not accepted) follower sees nothing; unfollow/remove immediately revokes visibility; own-post visibility unchanged; the multi-author keyset page orders correctly across authors. This locks in the `canView` change against regressions.
- **Web:** vitest coverage for the follow-button optimistic state machine (none→requested→following and teardown), search results rendering, and the requests-inbox accept/reject flow.
- **Mobile:** component/render tests per repo conventions.

### Rollout

Entirely additive — no breaking changes to existing endpoints, data, or surfaces. Recommended order matches the brief's riskiest/most-valuable-first build order:

1. **API — username.** Migration adding `username`, validator + reserved list, the Settings write path, and availability lookup. Prerequisite for everything else.
2. **API — follow graph.** `follows` migration, `internal/follow` domain and state machine, endpoints. Validate with manual API calls.
3. **API — lists + search.** Followers/following endpoints and profile search.
4. **API — timeline fan-out.** Widen `ListFeed`, the `canView` clause, the visibility default + backfill-row migration to `friends`, and the `?user=` scoped feed. Ship behind the existing Timeline surface.
5. **Web** — username in Settings, profile page, search, requests inbox, lists, follow actions, against the live API.
6. **Mobile** — the same surface as a parity phase (this dispatch or a follow-up).

No feature flag is required: the follow endpoints are new, the username column is nullable, and the feed is empty-safe (a user with no accepted follows sees exactly their own Timeline, as today).

## Open Questions

- **Old-handle redirects on rename.** v1 does **not** redirect an old username to the new one (old profile links 404, freed handle is immediately reusable). If impersonation-via-recycled-handle or broken shared links become real, add a `username_history` table mapping retired handles → user_id with a grace window. Deferred as premature at single-digit-user scale.
- **Blocking.** Out of scope for v1; the `status` enum and delete-on-terminal model leave room for a `blocked` state (or a separate `blocks` table) without migrating existing rows. The first concrete need (harassment, an unwanted accepted follower who keeps re-requesting after removal) should trigger its own small SOW.
- **Notifications.** Deferred. The requests inbox surfaces incoming requests by query; acceptance is observed by the requester on their next load. The `pending`/`accepted` states are notification-ready when an in-app or push notification SOW lands.
- **Search at scale.** `LIKE`-based substring matching on `display_name` is a bounded scan, fine for now. Revisit FTS5 (and typo tolerance / ranking by mutual-follow signals) only if the user table or search volume grows enough to matter.
- **Default post visibility migration.** This SOW flips the Timeline's default from `private` to `friends` and migrates backfilled rows accordingly, so accepted followers see full history. If any users have meanwhile marked posts private (they cannot yet — there's no UI in the Timeline SOW), that intent must be preserved; since no private-marking UI exists prior to this work, the blanket migration is safe.
- **Per-source-type privacy defaults.** All four source types are shareable by default with `private` as a per-post escape hatch. A "make all my runs private by default" preference is explicitly deferred; it would be a per-user, per-source-type default applied at `EnsurePost` time.
- **MCP/agent involvement.** Out of scope in v1 (the agent neither reads the graph nor posts). If "who follows me?" / "did anyone accept my request?" becomes a desired agent capability, it is a thin read-only proxy through these same API endpoints — no schema change.
