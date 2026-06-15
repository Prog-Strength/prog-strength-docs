---
status: shipped
repos:
  - prog-strength-api
  - prog-strength-web
  - prog-strength-mobile
  - prog-strength-docs
---

# User Timeline (Activity Feed)

**Status**: Shipped · **Last updated**: 2026-06-15

## Introduction

**Prog Strength** records a lot of what a user does — every strength workout, every run imported from a watch, every personal record and running best effort — but there is no single place where a user sees the shape of their own training over time. The data is siloed by surface: workouts live on the Activities tab, runs live one view over, PRs live on Personal Records, best efforts live inside individual runs. A lifter who opens the app on Sunday and wants to answer "what did this week actually look like?" has to visit three or four pages and assemble the story in their head. There is no narrative, no chronology, nothing that says "here is everything you did, newest first."

This SOW introduces the **Timeline** — a reverse-chronological activity feed of the signed-in user's own training events. The first cut is deliberately single-user: it shows *your* workouts, *your* runs, *your* PRs and best efforts, most recent first, as a scrollable feed of cards. But the Timeline is not really a solo feature. It is the seed of a **social timeline**. The next planned step for Prog Strength is friends/followers support, at which point this same surface becomes the place where you see your friends' training alongside your own, congratulate them on a PR, and trade comments. Building the friends graph is its own SOW; this one carves out the signed-in user's timeline and — critically — ships the *interaction primitives* the social feature will depend on, so they are designed, built, and battle-tested before any second user exists.

That is why this v1, despite being single-user, supports **comments** and **multiple reaction types** from day one. A user can react to and comment on their own posts. On its face that is a slightly odd thing to do alone, but it is the entire point: when friends arrive, comments and reactions are already a stable, tested part of the data model and the UI. The hard part of a social feed is rarely the social graph — it is the interaction surface and the post identity those interactions hang off. This SOW gets that right first, behind a single-user feed, so the friends SOW is mostly graph plumbing and a one-line change to "who can see this post."

After this work ships, every user has a **Timeline** entry in the web sidebar and a Timeline screen in the mobile app. Opening it shows their training history as a feed; each post can be reacted to (👍 Like, 💪 Strong, 🔥 Fire, 🎉 Celebrate) and commented on. Nothing about the existing workout / activity / PR data changes — the Timeline is an additive, read-mostly surface layered on top of records the app already keeps, plus three small new tables for the feed index and its interactions.

## Proposed Solution

The Timeline is built on a **hybrid feed-index** architecture. Rather than copying workout/run/PR content into a feed table (which drifts when the source is edited) or deriving the feed at read time from a `UNION` over heterogeneous tables (which makes post identity and pagination painful and scales badly to a social feed), we introduce a thin **feed index**: a `timeline_post` row per training event that stores only *which* event it points at and *when* it happened — not the content. Comments and reactions foreign-key cleanly to `timeline_post.id`. Post content is rendered at read time by **hydrating** from the existing source tables, so a post always reflects the current state of its underlying workout or run. Ordering is a single indexed keyset scan on `(user_id, occurred_at)`, which is exactly the shape a friends feed needs later (`user_id ∈ followee-set`).

A new `internal/timeline` domain in the Go API owns three tables — `timeline_post`, `timeline_comment`, `timeline_reaction` — plus a migration (`019_timeline.sql`). Posts are created through an **idempotent `EnsurePost`** call wired into the existing write paths (workout completion, run ingest, PR recorded, best effort recorded) and via a one-time **backfill** that seeds the index from existing history — mirroring the pattern already established by `internal/activity/backfill.go`. Cross-domain content is fetched through a small `SourceHydrator` interface so the timeline domain never imports workout/activity internals directly, keeping the boundary clean.

The API exposes a keyset-paginated `GET /timeline` plus comment and reaction endpoints. The web app gains a `/timeline` page and sidebar entry; the mobile app gains a Timeline screen. Both render the same conceptual feed of source-typed post cards with a reaction bar and a flat comment thread, reusing each platform's shared API client.

The single piece of deliberate forward-scaffolding is a `visibility` column on `timeline_post` (defaulting to `private`) and an authorization split between **visibility** ("can this user see this post?") and **ownership** ("can this user modify this comment/reaction?"). In v1, visibility is always self-only. When the friends SOW lands, it changes only the `canView` rule and adds a feed query over the followee set — no schema migration of existing posts, no rework of comments or reactions.

## Goals and Non-Goals

### Goals

- A reverse-chronological **feed of the signed-in user's own** training events: completed **workouts**, **runs**, **PRs**, and **running best efforts**.
- **Flat comments** on any post the user can see, with delete of one's own comment.
- **Four reaction types** — `like`, `strong`, `fire`, `celebrate` — that are *stackable per user* (a user may apply 💪 and 🔥 to the same post) but not duplicable (one of each type per user per post), and individually toggleable.
- A **stable post identity** (`timeline_post.id`) that comments and reactions foreign-key to, so the social SOW inherits working interaction primitives.
- **Keyset (cursor) pagination** on the feed, newest first.
- A **backfill** that seeds the feed index from all existing workouts, runs, PRs, and best efforts.
- A web **Timeline page** + sidebar entry and a mobile **Timeline screen**.
- An architecture in which adding the **friends/followers feed** is a small, well-isolated follow-up (graph tables + a `canView` change + a feed query over the followee set).

### Non-Goals

- The **friends/followers social graph** and the multi-author feed itself — that is the next SOW. This SOW only scaffolds for it.
- **MCP / agent** access to the timeline. The agent does not read or post to the feed in v1.
- **Bodyweight and nutrition** as post sources. v1 sources are training/achievement events only.
- **Threaded / nested comments.** Comments are flat.
- **Editing comments.** Edit = delete and re-add in v1.
- **Notifications** (in-app or push) for comments/reactions.
- **Real-time updates** (websockets/SSE). The feed is fetched and refreshed on navigation/pull-to-refresh.

## Implementation Details

### Data Model

New migration `internal/db/migrations/019_timeline.sql` (next in sequence after `018_nutrition_lookup_cache.sql`). Three tables. IDs use the existing `internal/id` generator; timestamps follow the conventions of the surrounding migrations.

**`timeline_post`** — the feed index. One row per training event that appears in a timeline.

- `id TEXT PRIMARY KEY`
- `user_id TEXT NOT NULL` — the **author** of the post (whose training this is)
- `source_type TEXT NOT NULL` — `CHECK (source_type IN ('workout','run','pr','best_effort'))`
- `source_id TEXT NOT NULL` — the id of the underlying record in its source domain
- `occurred_at` `TIMESTAMP NOT NULL` — the event's natural time (workout date, run start, PR achievement date); **drives feed ordering**
- `visibility TEXT NOT NULL DEFAULT 'private'` — `CHECK (visibility IN ('private','friends','public'))` (forward-scaffolding; always `private` in v1)
- `created_at`, `updated_at`
- `UNIQUE (user_id, source_type, source_id)` — makes `EnsurePost` idempotent and prevents duplicate posts for the same event
- Index `(user_id, occurred_at DESC, id)` — serves the keyset feed query

**`timeline_comment`** — flat comments.

- `id TEXT PRIMARY KEY`
- `post_id TEXT NOT NULL REFERENCES timeline_post(id) ON DELETE CASCADE`
- `user_id TEXT NOT NULL` — the commenter
- `body TEXT NOT NULL` — validated, max 2000 chars, non-empty after trim
- `created_at`, `updated_at`, `deleted_at TIMESTAMP NULL` — soft delete (preserves thread positions / future moderation)
- Index `(post_id, created_at)` — comments listed oldest-first under a post

**`timeline_reaction`** — typed reactions.

- `id TEXT PRIMARY KEY`
- `post_id TEXT NOT NULL REFERENCES timeline_post(id) ON DELETE CASCADE`
- `user_id TEXT NOT NULL`
- `type TEXT NOT NULL` — `CHECK (type IN ('like','strong','fire','celebrate'))`
- `created_at`
- `UNIQUE (post_id, user_id, type)` — a user may stack distinct types but not duplicate one; makes add idempotent
- Index `(post_id)` — reaction summaries aggregated per post

### Domain Layout

The `internal/timeline` package follows the established domain anatomy seen in `internal/activity` and `internal/workout`: `model.go`, `errors.go`, `metrics.go`, `repository.go` (the interface), `sqlite_repository.go`, `memory_repository.go`, `handler.go` (with `Mount(r chi.Router)`), `backfill.go`, and matching `*_test.go` files. It is registered in `internal/server/server.go` alongside the others:

```go
timeline.NewHandler(timelineRepo, hydrator).Mount(r)
```

The repository interface covers: `EnsurePost`, `ListFeed` (keyset), `GetPost`, `AddComment`, `DeleteComment`, `ListComments`, `AddReaction`, `RemoveReaction`, and the batch aggregate reads (`ReactionSummaries`, `CommentCounts`) used to decorate a feed page without N+1 queries. Both `sqlite_repository.go` and `memory_repository.go` implement it, matching the dual-implementation pattern the other domains use for fast unit tests.

### Write Path — `EnsurePost` and the `SourceHydrator`

Posts are created idempotently. `EnsurePost(ctx, userID, sourceType, sourceID, occurredAt)` inserts a `timeline_post` row and is a no-op on the `UNIQUE (user_id, source_type, source_id)` conflict. Because it is idempotent, the **same code path serves both the live write hooks and the backfill**, and a re-run can never duplicate posts.

To avoid coupling the source domains to the timeline (and to avoid import cycles), the timeline package defines a small publisher interface that the source services depend on:

```go
// Provided by the timeline domain, injected into workout/activity/PR services.
type Publisher interface {
    EnsurePost(ctx context.Context, p PostRef) error
}
```

The relevant write paths call it after the source record is committed:

- **Workout** — on workout completion → `EnsurePost(workout owner, "workout", workoutID, workoutDate)`
- **Run** — on activity ingest where kind is running → `EnsurePost(owner, "run", activityID, startedAt)`
- **PR** — when a personal record is recorded → `EnsurePost(owner, "pr", prID, achievedAt)`
- **Best effort** — when a running best effort is recorded → `EnsurePost(owner, "best_effort", bestEffortID, achievedAt)`

Publishing is **best-effort and non-blocking to the source write**: a failure to publish is logged and metered, never surfaced to the user, because the backfill/reconcile path can always repair a gap (the index is reconstructable from source). This keeps a feed-index hiccup from ever failing a workout save.

Content is **never stored** on the post. At read time the handler hydrates a page of posts through a `SourceHydrator`:

```go
type SourceHydrator interface {
    // Batch-fetch rendered content for a page of posts, grouped by source_type.
    Hydrate(ctx context.Context, refs []PostRef) (map[PostRef]PostContent, error)
}
```

`PostContent` is the rendered, platform-agnostic shape each card needs: a `title` (e.g. "Push day"), a `subtitle`/summary, a small set of `metrics` chips (e.g. `12 sets · 8,400 lb`, `5.0 mi · 41:12`), and a `href` deep-link to the existing detail page for that source. The hydrator implementation lives in the wiring layer (server package) as an adapter over the `workout`, `activity`, and PR repositories, batching one query per source type per page. Because content is rendered from source, edits to a workout/run reflect in the feed with no post-update machinery.

### Social Scaffolding (Forward-Looking)

Three design choices exist solely to make the friends/followers SOW small and low-risk. They are called out here so the next SOW knows exactly what it inherits:

1. **Author-keyed posts.** `timeline_post.user_id` is the author. The v1 feed query is `WHERE user_id = :viewer`. The social feed is the same query with `WHERE user_id IN (:followees ∪ :viewer)` — no schema change.
2. **`visibility` column.** Present and enforced from day one (always `private` in v1). The social SOW flips defaults and adds `friends`/`public` semantics without backfilling existing rows.
3. **Authorization split.** Two distinct checks, both isolated in one place: `canView(post, viewer)` (v1: `post.user_id == viewer`) and `canModerate(comment, viewer)` (`comment.user_id == viewer`). The friends SOW changes **only** `canView`. Comments and reactions — already `user_id`-scoped — work for a second user with no change.

What the next SOW adds (explicitly **not** in this one): follower/following tables, follow request/accept flow, the multi-author feed query and any fan-out, per-post visibility controls in the UI, and notifications.

### API Surface

All routes mount under `/timeline`, behind the existing auth middleware; the caller's id comes from `authctx.UserIDFrom(ctx)`. Responses use the `httpresp` helpers (`OK`, `Created`, `Error`, `ErrorWithCode`, `ServerError`) exactly as `internal/activity/handler.go` does.

- **`GET /timeline?limit=&before=`** — keyset page of the viewer's posts, newest first. `before` is the opaque cursor (encodes `occurred_at` + `id`). Each post in the response carries: `id`, `source_type`, `source_id`, `occurred_at`, `visibility`, hydrated `content` (`title`, `subtitle`, `metrics[]`, `href`), `reactions` (`{ summary: { type: count }, mine: [type] }`), and `comment_count`. Response includes `next_before` (null when exhausted). Reaction summaries and comment counts are batch-loaded for the page, not per-post.
- **`GET /timeline/posts/{id}`** — a single post with the same shape plus the full flat `comments[]` (oldest-first, excluding soft-deleted). 404 if not viewable.
- **`POST /timeline/posts/{id}/comments`** — body `{ "body": string }`. Validates non-empty/≤2000 chars and `canView`. Returns `201` with the comment DTO.
- **`DELETE /timeline/posts/{id}/comments/{commentId}`** — soft-deletes; requires `canModerate` (commenter owns it). `204`.
- **`PUT /timeline/posts/{id}/reactions/{type}`** — idempotent add of `type ∈ {like,strong,fire,celebrate}`; requires `canView`. `200`/`201`.
- **`DELETE /timeline/posts/{id}/reactions/{type}`** — removes the viewer's reaction of that type. `204`.

DTOs and JSON tags follow the conventions in the existing handlers. The authorization helpers (`canView`, `canModerate`) are the single isolated point the social SOW will revisit.

### Web — Timeline Page

> **Critical implementer note:** the web repo's `AGENTS.md` / `CLAUDE.md` are explicit — **"This is NOT the Next.js you know."** Before writing code in `prog-strength-web`, consult the relevant guide in `node_modules/next/dist/docs/` and the validated tab/page patterns already in `docs/superpowers/specs/`. The Activities consolidation (`app/(app)/activities/`) is the structural reference for a new `(app)` page.

- **Route:** `app/(app)/timeline/page.tsx` — the feed shell. Owns the cursor/load-more state and renders the list.
- **Sidebar:** add `{ href: "/timeline", label: "Timeline", icon: <TimelineIcon /> }` to the nav array in `components/sidebar.tsx`, slotted high (directly under Chat). A new "feed/activity-pulse" line glyph anchors it.
- **API client:** add timeline functions to the web client layer in `lib/`, following the existing client pattern (the same one the Activities/Nutrition pages use).
- **Components:**
  - `TimelineFeed` — renders the page, handles "load more" via `next_before`, empty state, and optimistic refresh.
  - `TimelinePostCard` — switches presentation on `source_type`, reusing existing workout/run/PR visual vocabulary (stat tiles, metric chips) and linking `content.href` to the existing detail pages.
  - `ReactionBar` — the four reaction buttons with counts and active state; **optimistic** toggle against `PUT`/`DELETE`.
  - `CommentThread` — flat list + composer; delete affordance only on the viewer's own comments.
  - Reuse existing toast and modal patterns.

### Mobile — Timeline Screen

- **Navigation:** a new Timeline route under the Expo Router `app/` tree, added to the app's primary navigation; nativewind/Tailwind styling consistent with the repo.
- **Components:** mirror the web set (`TimelineFeed`, `TimelinePostCard`, `ReactionBar`, `CommentThread`) under `components/`, reusing the shared mobile API client in `lib/`.
- **Process:** mobile follows the established per-phase parity workflow (research → plan → subagent) noted in the project's mobile parity tracking. This SOW provides the contract; if the dispatcher prefers, the mobile build may be split into its own follow-up implementation phase after API + web land (see Rollout).

### Backfill

A one-time backfill seeds `timeline_post` from existing history for all users, mirroring `internal/activity/backfill.go`: iterate workouts, runs, PRs, and best efforts, calling the idempotent `EnsurePost` with each record's natural `occurred_at`. Because `EnsurePost` is conflict-safe, the backfill is re-runnable and converges. It runs as a one-off on deploy (command/admin trigger consistent with how prior backfills in this repo are invoked). Comments and reactions start empty.

### Tests

- **Repository** (`sqlite` + `memory`): `EnsurePost` idempotency, feed keyset ordering/pagination, `UNIQUE` enforcement on reactions, `ON DELETE CASCADE` of comments/reactions when a post is removed, soft-delete exclusion of comments, and batch `ReactionSummaries`/`CommentCounts`.
- **Handler:** feed pagination shape and cursor; reaction add/remove and stacking distinct types; comment create/delete; and **authorization** — seed a *second* user and assert they cannot view, comment on, or react to the first user's post (locks in the `canView`/`canModerate` split the social SOW depends on).
- **Hydrator:** with fakes for the workout/activity/PR repos, asserting batched fetch and correct `PostContent` per `source_type`.
- **Web:** vitest coverage for `ReactionBar` optimistic toggle, feed load-more, and the comment composer.
- **Mobile:** component/render tests per repo conventions.

### Rollout

Entirely additive — no breaking changes to existing endpoints, data, or surfaces. Recommended order:

1. **API** — migration `019`, `internal/timeline` domain, write hooks, hydrator, endpoints, and backfill. Ship and run the backfill.
2. **Web** — the `/timeline` page and sidebar entry against the live API.
3. **Mobile** — the Timeline screen, either in this dispatch or as a follow-up parity phase.

No feature flag is required; the feed is empty-safe (renders an empty state pre-backfill) and the endpoints are new.

## Open Questions

- **Reaction semantics** — this SOW assumes *stackable distinct* reactions (Slack-style: a user may have 💪 and 🔥 on one post), enforced by `UNIQUE (post_id, user_id, type)`. If single-choice (LinkedIn-style: one reaction per user per post) is preferred, change the constraint to `UNIQUE (post_id, user_id)` and the `PUT` to replace.
- **PR / best-effort `occurred_at`** — use the achievement date (when the PR was set), not the row's `created_at`, so older PRs surfaced by the backfill land in the right chronological place.
- **Mobile in this dispatch vs. a follow-up** — API + web are the critical path; mobile can ride along or be split per the parity workflow. Dispatcher's call.
- **Comment editing** — intentionally out of scope (delete + re-add). Revisit only if it becomes a real friction point after friends ship.
