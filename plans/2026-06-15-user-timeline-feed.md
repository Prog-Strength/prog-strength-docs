# Implementation Plan: User Timeline (Activity Feed)

**SOW**: `sows/user-timeline-feed.md`
**Date**: 2026-06-15
**Repos**: prog-strength-api, prog-strength-web, prog-strength-docs (mobile deferred — see below)

This plan turns the SOW into discrete, subagent-sized tasks. Each task is
implemented by a subagent, then spec-reviewed and code-quality-reviewed
before moving on. Build/test gates are noted per task.

## Dispatcher decisions

1. **Mobile is deferred to a follow-up parity phase.** The SOW explicitly
   makes this the dispatcher's call ("Mobile in this dispatch vs. a
   follow-up — API + web are the critical path; mobile can ride along or be
   split per the parity workflow. Dispatcher's call.") and the Rollout lists
   mobile as step 3, "either in this dispatch or as a follow-up parity
   phase." Two concrete reasons to split it here:
   - The mobile app's primary navigation is a **5-tab bottom bar** (Chat ·
     Activities · Calendar · Nutrition · Progress), already at the iOS
     practical max. Adding a Timeline surface requires a navigation design
     decision (nest under an existing tab vs. replace one vs. a hub entry)
     that the SOW does not specify and that the per-phase mobile parity
     workflow (research → plan → subagent) is built to make.
   - API + web are the critical path and the highest-value, lowest-risk cut.
   This dispatch therefore ships **API + web**, and the docs PR records
   mobile as the explicit next parity phase. This is a sanctioned scope
   call, not a cut corner.

## Reconciliations (where the SOW assumed state the repos don't have)

The SOW is well aligned with the codebase. A few specifics resolved against
the actual source:

1. **Write-path hook points are the handlers, not deep in the repos.** The
   cleanest, least-invasive seam for best-effort publishing is at the end of
   the existing handlers, after the source write has succeeded:
   - **Workout** — `internal/workout/handler.go` `create` (and `update`).
     After the workout is persisted and its PR events attached
     (`attachPersonalRecordEvents` already runs here), publish one `workout`
     post plus one `pr` post per `PersonalRecordEvent` set by that workout.
   - **Run + best efforts** — `internal/activity/handler.go` `uploadTCX`
     success branch (new activity only, not the `ErrDuplicate` path). When
     `activity_type == running`, publish one `run` post, plus one
     `best_effort` post per `ActivityBestEffort` on the activity.
   The `timeline.Publisher` interface is injected into the `workout` and
   `activity` handlers as an **optional (nil-safe) field**: existing handler
   constructions in tests pass `nil` and skip publishing, so the blast radius
   on existing tests is zero. Publishing is best-effort: each `EnsurePost`
   error is logged + metered and never affects the HTTP response.

2. **`pr` posts are per `PersonalRecordEvent`.** `personal_record_events`
   rows have stable `ID` + `AchievedAt`. `source_id = event.ID`,
   `occurred_at = event.AchievedAt`. One post per PR break, which is what the
   feed wants chronologically (SOW Open Question #4 — use achievement date,
   not `created_at`).

3. **`best_effort` posts have a composite source id.** `activity_best_efforts`
   is keyed by `(activity_id, distance_key)` with no standalone id. Use
   `source_id = "<activity_id>:<distance_key>"` and
   `occurred_at = activity.StartTime`. The hydrator splits on `:` to render.

4. **`occurred_at` per source**: workout → `PerformedAt`; run →
   `StartTime`; pr → `event.AchievedAt`; best_effort → run `StartTime`.

5. **No `/timeline` group exists yet.** New `internal/timeline` domain,
   registered in `internal/server/server.go` inside the JWT-gated group
   alongside the others, with the SQLite/memory repo selection mirroring the
   existing domains. The `SourceHydrator` adapter lives in the server wiring
   layer (an adapter over the `workout` and `activity` repositories), keeping
   the timeline package free of cross-domain imports.

6. **Backfill is invoked on startup**, gated on `timeline_post` being empty,
   exactly like `BackfillActivityBestEfforts` / `BackfillPersonalRecords` in
   `server.go` (SQLite mode only). It iterates workouts, runs, PR events, and
   best efforts and calls the idempotent `EnsurePost`.

## API tasks (prog-strength-api)

### T1 — migration 019 + timeline domain core types
Pure types and schema, no implementations.
- `internal/db/migrations/019_timeline.sql`: `timeline_post`,
  `timeline_comment`, `timeline_reaction` exactly as the SOW Data Model
  section specifies (TEXT ids, DATETIME timestamps matching surrounding
  migrations, `CHECK` on `source_type`/`visibility`/`type`,
  `UNIQUE(user_id, source_type, source_id)`,
  `UNIQUE(post_id, user_id, type)`, FKs `ON DELETE CASCADE`, the
  `(user_id, occurred_at DESC, id)`, `(post_id, created_at)`, and
  `(post_id)` indexes). Match the SQL style of `018_*.sql` / `016_*.sql`.
- `internal/timeline/model.go`: `Post`, `Comment`, `Reaction` domain
  structs; `PostRef{UserID, SourceType, SourceID, OccurredAt}`;
  `PostContent{Title, Subtitle, Metrics []string, Href string}`;
  `ReactionSummary` (type→count + viewer's own); `SourceType`,
  `ReactionType`, `Visibility` enums each with a `Valid()` method and the
  closed value sets from the SOW.
- `internal/timeline/errors.go`: `ErrNotFound`, `ErrForbidden` (or reuse
  not-found for non-viewable per the SOW's "404 if not viewable"),
  `ErrValidation`, `ErrStorage` — sentinel pattern with `errors.Is`.
- `internal/timeline/metrics.go`: Prometheus counters mirroring the other
  domains' `metrics.go` (publish failures, etc.).
- `internal/timeline/repository.go`: the `Repository` interface —
  `EnsurePost`, `ListFeed` (keyset), `GetPost`, `AddComment`,
  `DeleteComment`, `ListComments`, `AddReaction`, `RemoveReaction`,
  `ReactionSummaries` (batch), `CommentCounts` (batch). Also declare the
  `Publisher` and `SourceHydrator` interfaces here (or in model.go).
- **Gate**: `go build ./...`.

### T2 — memory + sqlite repositories + tests
- `internal/timeline/memory_repository.go` and `sqlite_repository.go`,
  both with `var _ Repository = (*…)(nil)`. SQLite uses `database/sql`,
  conflict-safe `EnsurePost` (`INSERT … ON CONFLICT DO NOTHING` on the
  unique key), keyset `ListFeed` (`WHERE user_id=? AND (occurred_at,id) <
  (?,?) ORDER BY occurred_at DESC, id DESC LIMIT ?`), soft-delete-aware
  `ListComments`, idempotent `AddReaction`, batch `ReactionSummaries` /
  `CommentCounts`.
- `*_repository_test.go` for both: `EnsurePost` idempotency, feed keyset
  ordering/pagination, reaction `UNIQUE` stacking (one of each type per
  user, distinct types stack), `ON DELETE CASCADE` of comments/reactions,
  soft-delete exclusion of comments, batch aggregates with no N+1.
- **Gate**: `go test ./internal/timeline/...`.

### T3 — handler + DTOs + auth split + tests
- `internal/timeline/handler.go`: `Mount(r chi.Router)` under `/timeline`
  with the six endpoints from the SOW API Surface section; DTOs + JSON
  tags following `internal/activity/handler.go`; opaque cursor
  encode/decode for `before`/`next_before` (base64 of `occurred_at`+`id`);
  `httpresp` helpers; `auth.UserIDFrom(ctx)`; the isolated `canView(post,
  viewer)` (v1: `post.UserID == viewer`) and `canModerate(comment, viewer)`
  helpers. Feed page hydrates via the injected `SourceHydrator` and
  batch-loads reaction summaries + comment counts.
- `handler_test.go` against the memory repo + a fake hydrator: feed
  pagination shape and cursor; reaction add/remove + stacking distinct
  types; comment create/delete; and the **second-user authorization** test
  (seed user B, assert they cannot view / comment on / react to user A's
  post — 404), locking in the `canView`/`canModerate` split.
- **Gate**: `go test ./internal/timeline/...`.

### T4 — backfill + server wiring + write-path hooks
- `internal/timeline/backfill.go`: idempotent seed gated on empty
  `timeline_post`, iterating workouts, running activities, PR events, and
  best efforts, calling `EnsurePost`. Logs processed/skipped counts.
- `internal/server/server.go`: construct the timeline repo (sqlite/memory),
  build the `SourceHydrator` adapter over the workout + activity repos,
  build the `Publisher`, inject the publisher into the workout + activity
  handlers, mount `timeline.NewHandler(repo, hydrator).Mount(r)`, and call
  the backfill on startup (SQLite mode, after migrations).
- Write-path hooks per Reconciliation #1, all best-effort/non-blocking.
- **Gate**: `go build ./...`, `go test ./...`, `golangci-lint run`.

## Web tasks (prog-strength-web)

### W1 — API client + sidebar + page shell
- `lib/api.ts`: `listTimeline`, `getTimelinePost`, `addComment`,
  `deleteComment`, `addReaction`, `removeReaction` + the `TimelinePost`,
  `TimelineComment`, `ReactionSummary`, `TimelinePage` types, following the
  existing `unwrap`/Bearer pattern.
- `components/sidebar.tsx`: a `TimelineIcon` line glyph + `{ href:
  "/timeline", label: "Timeline", icon: <TimelineIcon /> }` slotted
  directly under Chat.
- `app/(app)/timeline/page.tsx`: client page shell (Suspense + inner),
  mirroring `app/(app)/activities/page.tsx`, rendering `<TimelineFeed/>`.
- **Gate**: `npm run typecheck && npm run lint`.

### W2 — feed components + tests
- `components/timeline/`: `TimelineFeed` (cursor/load-more via
  `next_before`, empty + error states), `TimelinePostCard` (switch on
  `source_type`, reuse stat-tile/metric-chip vocabulary, link
  `content.href`), `ReactionBar` (four buttons, counts, active state,
  **optimistic** PUT/DELETE), `CommentThread` (flat list + composer, delete
  only on own comments). Reuse the existing toast pattern.
- Vitest: `ReactionBar` optimistic toggle, feed load-more, comment composer.
- **Gate**: `npm run typecheck && npm run lint && npm run test && npm run build`.

## Docs + PRs

Flip `sows/user-timeline-feed.md` to `status: shipped` /
`**Status**: Shipped` / `**Last updated**: 2026-06-15` on
`feat/user-timeline-feed`; open PRs in api, web, and docs. The docs PR body
follows the operator template and records mobile as the next parity phase.

## Rollout (for the docs PR)

1. **prog-strength-api** — migration 019, `internal/timeline` domain, write
   hooks, hydrator, endpoints, backfill. Merges + deploys first; until the
   API is live the web page's `GET /timeline` 404s.
2. **prog-strength-web** — `/timeline` page + sidebar, against the live API.
3. **mobile** — deferred follow-up parity phase.

Entirely additive; empty-safe pre-backfill; no feature flag.
