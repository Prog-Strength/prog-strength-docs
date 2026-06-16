---
status: shipped
repos:
  - prog-strength-api
  - prog-strength-web
  - prog-strength-docs
---

# Social Profiles: Discovery, Bio, Graphs & Author Avatars

**Status**: Shipped · **Last updated**: 2026-06-16

## Introduction

Prog Strength already ships a social layer: user search (`GET /users/search`), a full follow-request state machine (`/follows` with pending → accepted, accept/reject/cancel), public profile pages at `/u/[username]` showing avatar, display name, @handle, follower/following counts, and a privacy-gated activity feed. All of this merged in web PR #45 and is live.

But the social layer has a reachability hole that makes it *feel* broken. `username` is **optional** — it is only ever set through Settings (`PATCH /me`), never at signup — and user search returns handle-less users by display-name match. The web `ProfileRow` addresses the profile route and the follow button **by handle**: a user with `username = null` renders in search results but is *un-clickable and un-followable* (`ProfileRow.tsx:28,51`). So any user who never opened Settings to claim a handle shows up in search as a dead end. This is the bug behind "I search for a user, they come up, but I can't follow them or open their profile."

This SOW closes that hole and then makes profiles richer and the timeline more social:

1. **Reachability** — guarantee every user has a handle (backfill + auto-assign at signup), so the existing follow/profile UI just works.
2. **Bio** — a new user field, editable in Settings and shown on the profile.
3. **Profile graphs** — a new *public* per-user stats endpoint feeding two weekly graphs (lift-session minutes, running distance), gated by the same privacy rule as the activity feed.
4. **Author avatars** — embed the author's name + avatar in timeline post and comment payloads so the feed reads as a social feed, not anonymous cards.

### Goals

- Every user is reachable by handle; no handle-less dead ends in search, followers/following lists, or the requests inbox.
- A user can write a short bio in Settings and it appears on their public profile.
- A profile shows two weekly graphs (lift-session minutes and running distance) over the last 12 weeks, locked for non-followers of private accounts.
- Timeline cards and comments show the author's avatar and display name, linking to their profile.

### Non-goals (deferred)

- **Mobile parity** — this is web + API only. Note for future planning: the mobile app currently has *none* of the social layer (no search, follow, public profiles, timeline feed, or comments — only own-profile/Settings with avatar upload). A mobile social effort therefore builds the whole stack from scratch and belongs in its own phased SOW, not here.
- **Notifications** for new followers/comments.
- **Handle-change UX** beyond the existing Settings editor.
- **New graph types** beyond the two named (no PR/volume/HR graphs on profiles yet).
- **Bio rich text / links / mentions** — plain text only.

## Repos & boundaries

- **prog-strength-api** (Go) — owns the data model, migrations, and all endpoints. Single writer.
- **prog-strength-web** (Next.js) — consumes the API via the `lib/api.ts` fetch wrappers; no direct DB access.
- **prog-strength-docs** — this SOW.

The MCP/agent layer is untouched; none of these surfaces are agent tools.

---

## Part 1 — Handle reachability (api)

**Problem:** `username` is nullable and unset for any user who never visited Settings. Handle-less users are discoverable but unreachable.

**Decision:** guarantee a handle for every user. Keep the column nullable in the schema (no constraint surgery), but guarantee population on both existing and new rows. The web `ProfileRow` already degrades gracefully for null handles, so once nulls are gone the existing UI is fully functional with **no frontend change** for reachability.

### Handle generator

A shared helper that produces a valid, unique handle:

- Slugify `display_name`: lowercase, ASCII-fold, replace runs of non-`[a-z0-9_]` with nothing (or `_`), trim to the username max length, and ensure it satisfies the existing `ValidateUsername` charset/shape/reserved-name rules.
- If the slug is empty or fully reserved/invalid, fall back to `user_<short-id>` derived from the user ID.
- Resolve collisions by appending an incrementing numeric suffix (`name`, `name2`, `name3`, …) until `Update`/insert succeeds against the unique index. Cap retries; on exhaustion fall back to the id-based form, which is unique by construction.

The generator lives in `internal/user` next to `ValidateUsername` and is covered by unit tests (slug happy path, empty/garbage display name → id fallback, collision suffixing, reserved-name avoidance).

### Backfill migration

A **Go-migration** (per the repo's migration-runner Go-migration capability, as used by planned-workout reconciliation) that selects all `users` where `username IS NULL AND deleted_at IS NULL` and assigns each a generated handle, committing per-user so a mid-run collision doesn't roll back the whole batch. Idempotent: re-running finds no null handles and is a no-op.

### Signup path

At user creation, if no username is provided, assign a generated handle so `username` is never null going forward. Existing Settings flow still lets the user change it later (subject to the existing uniqueness/validation rules).

### Tests

- Generator unit tests (above).
- Migration test: seed handle-less + handled users, run, assert all non-deleted users have valid unique handles and pre-existing handles are untouched.
- Creation test: a user created without a username comes back with a valid handle.

---

## Part 2 — Bio (api + web)

### API

- **Migration:** add nullable `bio TEXT` to `users`.
- **Model/validation:** add `Bio *string` to the `User` struct. Validate at the write edge: max **160** characters (count by runes), plain text. Empty string clears it (stored as `NULL` or empty per existing nullable-field convention).
- **`updateMe`:** accept `bio` as a partial-update field (pointer-presence; `nil` = absent, `""` = clear).
- **DTOs:** add `bio` to `meResponse` and to `publicProfileDTO`.

### Web

- **Settings** (`/settings`): add a bio `<textarea>` with a live character counter (160 max, disable/trim on overflow), next to the existing display-name/avatar editors. Persists via the existing `updateMe` wrapper.
- **Profile** (`/u/[username]`): render bio under the name/@handle in the header. An empty bio renders nothing (no placeholder on others' profiles). Add `bio` to the `PublicProfile` and `ResolvedProfile` types in `lib/api.ts`.

### Tests

- Go: bio length validation (160 boundary), clear-via-empty-string, presence in both DTOs.
- Web: Settings editor enforces the counter/cap; profile header renders bio when present and nothing when absent.

---

## Part 3 — Profile graphs + public stats endpoint (api + web)

### API — new endpoint

`GET /users/{username}/stats` returns two weekly time series over the **last 12 weeks** (server computes week buckets in the *target user's* timezone, consistent with existing analytics):

- **`lift_session_minutes`** — per week, the summed session duration `EndedAt − PerformedAt` over workouts that *have* an `EndedAt`. **Caveat:** workouts without an end time contribute nothing (their duration is unknown); this is expected and documented, not a bug.
- **`running_distance_meters`** — per week, summed `DistanceMeters` of running activities.

Each series is a dense array of 12 `{ week_start, value }` points (zero-filled weeks included) so the client can render a continuous chart without gap-filling.

**Privacy gate — identical to the activity feed:**

- **Self** and **accepted followers** → full series.
- **Public account** → full series for anyone.
- **Private account, viewer not an accepted follower** → a `locked` response (mirror the timeline's locked convention — same status/shape the web already handles for `listUserTimeline`).

Follower/following counts in the profile header are unchanged and stay visible regardless (they come from `publicProfileDTO`, not this endpoint).

The stats query reuses existing workout/activity repositories; add a focused repository method per series rather than overloading the own-user metrics handlers.

### Web — profile page

- New `lib/api.ts` wrapper `getProfileStats(token, username)` returning the two series plus a `locked` flag, following the `listUserTimeline` pattern.
- Render two charts below the profile header, **reusing the existing recharts components** `workout-duration-chart.tsx` (lift-session minutes) and `running-mileage-chart.tsx` (running distance), adapting their props to the stats series shape. Extract a shared presentational chart if the existing components are too coupled to the analytics pages — otherwise reuse directly.
- **Locked** → show the same locked treatment the activity feed already uses (no chart, gated message).
- **Empty series** (all zeros / no activity) → a small "No activity yet" empty state instead of a flat-line chart.

### Tests

- Go: bucketing correctness (12 dense weeks, timezone, end-less workouts excluded from lift minutes), and the full privacy matrix (self / public / accepted-follower / private-non-follower → locked).
- Web: profile renders both charts from a stats fixture, the locked state, and the empty state.

---

## Part 4 — Author avatars on timeline cards + comments (api + web)

### API — DTO enrichment

- **Post DTO:** add an embedded `author { user_id, username, display_name, avatar_url }` to each timeline post. The feed handler **batch-resolves** user summaries for the page in one lookup over the distinct author IDs (with avatar presigning where it already happens), avoiding N+1.
- **Comment DTO:** add the same `author` object to each comment returned by `getPost`. Batch-resolve over the thread's distinct commenter IDs.
- Reuse the existing profile-summary resolution path (the one search/followers lists use) so avatar presigning and OAuth fallback behave identically.

### Web

- Add `author: ProfileSummary` to the `TimelinePost` and `TimelineComment` types in `lib/api.ts`.
- **`TimelinePostCard`:** render the author's `Avatar` + display name in the card header, linking to `/u/[handle]`. Keep the existing source-type emoji/label as a secondary line.
- **`CommentThread`:** render each commenter's `Avatar` + name beside the comment body, replacing the bare-text rows. The existing own-comment delete affordance is preserved (still keyed on `user_id`).

### Tests

- Go: feed and single-post responses include a populated `author` for every post/comment; batch resolution issues no per-row query (assert query count or use the batched repository method).
- Web: post card renders author identity and links to the profile; comment rows render commenter identity; delete still shows only on own comments.

---

## Rollout & sequencing

1. **Part 1 (reachability)** ships first and independently — it's the live bug. After the backfill migration runs, search/follow/profile work end-to-end.
2. **Parts 2–4** are independent of each other and can land in any order. Each is a vertical slice (migration/DTO → endpoint → web surface) with its own tests.

No feature flags required; each part is additive and backward-compatible (new nullable column, new endpoint, additive DTO fields).

## Risks & edge cases

- **Handle collisions at scale** — handled by suffix retry + id-based fallback; the unique index is the source of truth.
- **Reserved-name slugs** — generator must run candidates through `ValidateUsername`; covered by tests.
- **End-less workouts** — silently excluded from lift-minutes; documented, surfaced as "No activity yet" when a user has only end-less sessions.
- **Private-account leakage** — graphs are gated identically to the feed; the privacy matrix is explicitly tested.
- **Avatar presigning cost** — batch-resolve per page, not per row; reuse the existing summary path that already presigns.
