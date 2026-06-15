---
status: proposed
repos:
  - prog-strength-api
  - prog-strength-mcp
  - prog-strength-agent
  - prog-strength-web
  - prog-strength-infra
  - prog-strength-docs
---

# Planned Workouts & Google Calendar Sync

**Status**: Proposed · **Last updated**: 2026-06-15

## Introduction

**Prog Strength** today is entirely retroactive. Every surface — the calendar, the activities tab, the progress charts — answers "what did I do?" Nothing answers "what am I *going* to do?" A lifter who wants to plan a training week has to keep it in their head, a notes app, or a spreadsheet, and there's no bridge between that plan and the session they eventually log. The calendar renders completed workouts and runs by date; there is no forward-looking entity on it at all.

This SOW introduces **planned workouts**: a new first-class entity, distinct from a logged session, that lets a user lay out training ahead of time — a bare time block, or a fully-specified agenda of exercises, sets, reps, and target weight/RPE — and then watch each plan move through its lifecycle from *planned* to *completed* (linked to the actual session that fulfilled it) or *skipped*. Planned workouts render on the Prog Strength calendar as forward-looking events, and an opted-in user can **push them to Google Calendar**, where the same lifecycle plays out: an empty reserved slot becomes a planned agenda becomes a completed session annotated with what was actually done. The result is a training audit trail visible in both places.

The agent is a first-class consumer. "Plan my week" must work end-to-end through natural language — the agent composes several planned workouts and, optionally, schedules them to Google Calendar — *before* the calendar-sync layer is built, because the planned-workout schema is the shared output target for both the agent and the calendar push. One structure, multiple consumers.

A note on the OAuth architecture, because it differs from the original brief's assumption: **Prog Strength does not use Auth.js / NextAuth.** The Go API owns the Google OAuth flow directly — the web app redirects to `GET /auth/google/login` on the API, which runs the consent flow and returns a short-lived JWT (in the callback URL hash, stored in `localStorage`). Today it requests only `userinfo.email` + `userinfo.profile`, with no offline access and no refresh token. This is good news for calendar sync: enabling it is a **separate, incremental server-side OAuth flow** the API initiates on demand, rather than surgery on a session library. The rest of this document is written to that reality.

This is a large feature; it is delivered in four phases (below), riskiest-and-most-valuable first, so the planned-workout core and the agent path prove out before the operational surface of persisted Google credentials is introduced.

## Proposed Solution

A new `internal/planned_workout` domain in the Go API owns the planned-workout entity and all of its lifecycle transitions. It follows the established domain anatomy (`model`, `handler` with `Mount(r)`, `repository` + sqlite/memory implementations, `errors`, `metrics`, tests) and the **single-writer invariant**: all planned-workout CRUD and *all* Google Calendar writes go through this domain. The MCP server and the agent never touch the database or Google directly — they call the Go API, exactly as the existing `workouts`/`bodyweight` MCP tools do.

A planned workout carries a scheduled UTC time window plus the IANA timezone it was created in, an `activity_kind` discriminator (`lift` in v1), an optional agenda (planned exercises → planned sets with target reps/weight/RPE), a lifecycle `status` (`planned` → `completed` / `skipped`), an optional polymorphic link to the logged session that completed it, and the Google-sync bookkeeping (`google_event_id`, `google_sync_status`, `last_sync_error`). The agenda is optional: a planned workout may be a bare time block or a fully-specified plan.

Google Calendar sync is **opt-in and one-way (Prog Strength → Google)**. A user who never connects their calendar never triggers refresh-token storage or the extra consent scope. Opting in runs a second, incremental OAuth authorization (calendar scope + offline access) whose returned refresh token is encrypted and stored server-side in a dedicated `user_calendar_connection` table — required because completion updates may happen decoupled from any live session (e.g. the agent logs a session days later). When a planned workout is scheduled, created-with-sync, edited, completed, or deleted, the API mints a Google access token from the stored refresh token and writes the corresponding event. Prog Strength remains the source of truth; Google writes are best-effort and their status is tracked on the planned workout, so a failed write degrades to a resyncable state rather than a lost plan.

The web app renders planned workouts on the existing calendar as visually-distinct forward-looking events, provides a create/edit form and a "Connect Google Calendar" affordance, and exposes the time-block-vs-full-agenda detail choice as both a user default and a per-workout override.

## Goals and Non-Goals

### Goals

- A **planned workout** entity: scheduled time window + optional agenda (exercises/sets with target reps/weight/RPE) + lifecycle status, created via web, agent, or API.
- **"Plan my week" via the agent** — natural-language creation (and optional scheduling) of multiple planned workouts, working end-to-end before the calendar layer exists.
- **Opt-in, one-way Google Calendar sync** with a persisted, encrypted, server-side refresh token for opted-in users.
- **Lifecycle propagation**: a logged session linked to a planned workout flips its status to `completed` and updates both the PS calendar event and the Google event with actual details.
- **Two calendar detail levels** — time-block-only and full-agenda — selectable as a user default and overridable per workout.
- The **single-writer invariant** preserved: MCP/agent never touch the DB or Google; all writes go through the Go API.
- Prog Strength is the **source of truth**; Google write failures are tracked and resyncable.

### Non-Goals

- **Two-way sync.** Reading changes back from Google Calendar into Prog Strength is out of scope for v1.
- **Planned runs.** The schema carries an `activity_kind` discriminator and a polymorphic completion link so planned runs (and other activity types) extend later without migration churn, but v1 plans only **lifting** workouts.
- **Mobile UI.** v1 ships the web + agent surfaces; the mobile planned-workout UI is a later parity phase. (The API/MCP it would consume are built here.)
- **Inferred plan↔session matching.** Linking a logged session to a plan is an explicit action, not heuristically guessed by date/lift.
- **Async retry queue / KMS envelope encryption.** v1 uses synchronous best-effort Google writes with a resyncable status, and app-level AES-256-GCM for the token. Both are called out as future hardening.
- **Calendar providers other than Google.**

## Implementation Details

### Phase 1 — `planned_workout` domain + CRUD (prog-strength-api)

New migration `internal/db/migrations/021_planned_workouts.sql` (next after `020_timeline.sql`; renumber if another lands first). Tables:

**`planned_workouts`**

- `id TEXT PRIMARY KEY`, `user_id TEXT NOT NULL`
- `name TEXT`
- `activity_kind TEXT NOT NULL DEFAULT 'lift'` — `CHECK (activity_kind IN ('lift'))` in v1; the column exists now so adding `'run'` later is a CHECK change, not a schema migration
- `scheduled_start_utc DATETIME NOT NULL`, `scheduled_end_utc DATETIME NOT NULL`
- `timezone TEXT NOT NULL` — IANA name captured at creation; the canonical tz for rendering the Google event and for server-side writes that have no request timezone
- `status TEXT NOT NULL DEFAULT 'planned'` — `CHECK (status IN ('planned','completed','skipped'))`
- `notes TEXT`
- `completed_session_id TEXT`, `completed_session_kind TEXT` — nullable polymorphic link (`CHECK (completed_session_kind IN ('workout','activity'))`); set on completion
- `calendar_detail TEXT` — nullable per-workout override (`CHECK (calendar_detail IN ('time_block','full_agenda'))`); null falls back to the user default
- `google_event_id TEXT`, `google_sync_status TEXT`, `last_sync_error TEXT` — Google bookkeeping (see Phase 3); `google_sync_status IN ('pending','synced','failed')` or null when never scheduled
- `created_at`, `updated_at`, `deleted_at`
- Index `(user_id, scheduled_start_utc)` — the week-range query

**`planned_workout_exercises`** and **`planned_sets`** mirror the logged `workout_exercises`/`sets` shape but are **target-oriented**: a planned set holds `target_reps`, `target_weight`, `unit`, and a nullable `target_rpe` (the logged `Set` has no RPE; the planned agenda introduces it). Both cascade-delete with the planned workout. The agenda is optional — a planned workout with zero exercises is a valid bare time block.

The domain registers in `internal/server/server.go` (`plannedworkout.NewHandler(repo, ...).Mount(r)`) and uses `authctx.UserIDFrom` + `httpresp` like every other handler. Endpoints:

- `POST /planned-workouts` — create (bare block or with agenda).
- `GET /planned-workouts?since=&until=` — list a date range (the "week" query); also a `GET /planned-workouts/{id}`.
- `PUT /planned-workouts/{id}` — edit (reschedule, change agenda, change `calendar_detail`).
- `DELETE /planned-workouts/{id}` — soft delete.
- `POST /planned-workouts/{id}/skip` — status → `skipped`.

A new **`timezone`** column is added to `users` (same migration or a small sibling) and surfaced on `GET /me` / `PUT /me`, defaulting to `UTC` until set; the web/agent populate it. This is the canonical timezone for server-side Google writes.

**Phase 1 exit:** planned workouts can be created/listed/edited/deleted/skipped via the API and shown with a minimal web list, validated by manual entry. No agent, no Google.

### Phase 2 — MCP tools + agent (prog-strength-mcp, prog-strength-agent)

A new `src/prog_strength_mcp/planned_workouts.py` registered in `server.py` (`planned_workouts.register(mcp, api)`), thin over the Phase 1 endpoints, forwarding the inbound `Authorization` header via the existing `_auth_header_or_raise()` pattern. Tools:

- `create_planned_workout(...)` — scheduled window + optional agenda (bare block or full plan).
- `list_planned_workouts(since, until)` — the week view.
- `update_planned_workout(...)` / `skip_planned_workout(id)`.
- `complete_planned_workout(planned_workout_id, session_id, session_kind)` — added in Phase 4; stubbed here.
- `schedule_workout_to_calendar(planned_workout_id, detail_level)` — added in Phase 3; stubbed here.

The agent prompt (`prog-strength-agent`) is extended so the model knows planned workouts exist and how to compose them: **"plan my week"** = call `create_planned_workout` N times across the week, then optionally schedule. The corresponding `api_client.py` methods are added in the MCP package.

**Phase 2 exit:** a user can say "plan an upper/lower split for next week" and the agent creates the planned workouts end-to-end, visible on the web calendar. Still no Google.

### Phase 3 — Opt-in Google Calendar push + refresh-token storage (prog-strength-api, prog-strength-web, prog-strength-infra)

**Incremental OAuth.** The Go API gains a second OAuth flow alongside login, requesting the calendar scope with offline access:

- `GET /auth/google/calendar/connect` — redirects to Google with `scope=https://www.googleapis.com/auth/calendar.events`, `access_type=offline`, `include_granted_scopes=true`, `prompt=consent` (the last forces a refresh token even on re-consent). This is independent of the login JWT and does not disturb the existing session.
- `GET /auth/google/calendar/callback` — exchanges the code, stores the encrypted refresh token, and records the user's primary calendar id.
- `GET /me/calendar/connection` — connection status (connected / revoked / absent) for the UI.
- `DELETE /me/calendar/connection` — revokes the token at Google and deletes the row (disconnect).

**Storage.** New migration `022_user_calendar_connection.sql`:

```
user_calendar_connection(
  user_id            TEXT PRIMARY KEY,
  refresh_token_enc  BLOB NOT NULL,   -- AES-256-GCM ciphertext
  refresh_token_nonce BLOB NOT NULL,  -- per-row GCM nonce
  google_calendar_id TEXT NOT NULL,   -- usually 'primary'
  scopes             TEXT NOT NULL,
  status             TEXT NOT NULL,   -- 'connected' | 'revoked'
  connected_at, updated_at
)
```

The refresh token is encrypted with **app-level AES-256-GCM**, keyed by a 32-byte secret from env `CALENDAR_TOKEN_ENC_KEY`, with a fresh nonce per row. The API has no KMS/Secrets-Manager wiring today and per-call KMS would add cost and latency to every calendar write; KMS envelope encryption is the documented future-hardening path. **prog-strength-infra** provisions `CALENDAR_TOKEN_ENC_KEY` through the same secret-delivery mechanism the API already uses for `JWT_SIGNING_KEY` / `GOOGLE_CLIENT_SECRET`, and the Google Cloud OAuth consent screen must have the `calendar.events` scope enabled (a one-time console step, noted in Rollout).

**Writing events.** A `calendarsync` package mints a Google access token from the stored refresh token on demand (short in-memory cache acceptable) and writes events:

- Scheduling a planned workout (`POST /planned-workouts/{id}/schedule` with a `detail_level`, or `calendar_sync: true` on create/update) inserts a Google event and stores its `google_event_id`.
- Subsequent edits **rewrite the whole event** (`events.patch` keyed by `google_event_id`) — summary + description + start/end. No diffing; volume is a handful of writes per user per week and `patch` is idempotent.
- Delete removes the Google event.
- The event description honors the **detail level**: `time_block` → a reserved slot linking back to Prog Strength; `full_agenda` → exercises/sets/targets in the description. The effective level is `planned_workouts.calendar_detail ?? users.calendar_default_detail` (a new `users` preference, defaulting to `time_block`).

**Failure handling.** Every write is best-effort. On success: `google_sync_status='synced'`. On failure: `'failed'` + `last_sync_error`, and the plan persists in PS regardless. A `404` from Google (event deleted out-of-band) drops `google_event_id` and marks it unsynced. A refresh that fails because the token was revoked sets `user_calendar_connection.status='revoked'`, flips affected plans to a resyncable state, and the UI prompts re-consent. A `POST /planned-workouts/{id}/resync` (and a UI action) re-attempts. No background queue in v1.

The Phase 2 MCP `schedule_workout_to_calendar` tool is wired to the schedule endpoint here.

**Web.** A "Connect Google Calendar" control (reads `GET /me/calendar/connection`), the per-workout and default detail-level toggles, and per-plan sync status/resync affordance.

**Phase 3 exit:** an opted-in user (or the agent) schedules a planned workout and it appears on their Google Calendar; disconnecting revokes cleanly.

### Phase 4 — Completion propagation (prog-strength-api, prog-strength-mcp, prog-strength-web)

When a session is logged against a planned workout — via live logging, the agent, or manual entry — the plan is completed and both events reflect what was actually done:

- `POST /planned-workouts/{id}/complete` with `{ session_id, session_kind }` sets `status='completed'`, stores the polymorphic link, and (if the plan is Google-synced) rewrites the Google event to show the **actual** logged details and a "completed" marker. The PS calendar then shows the plan as fulfilled, linked to the real session.
- Linking is **explicit**: the live-workout flow may carry an optional `planned_workout_id` so finishing it auto-calls complete; the agent and manual paths pass the ids. No date/lift inference.
- The MCP `complete_planned_workout` tool (stubbed in Phase 2) is wired here.

**Phase 4 exit:** the full lifecycle — empty block → planned agenda → completed session with actuals — is visible in both Prog Strength and Google Calendar.

### Tests

- **API:** repository (sqlite + memory) for planned-workout CRUD, agenda cascade, status transitions, week-range queries, and the polymorphic completion link; handler tests for each endpoint incl. authz (a user can't touch another's plans). `calendarsync` with a faked Google client: token-refresh, event create/patch/delete, the detail-level description rendering, and each failure path (`failed` status, 404 unsync, revoked re-consent). Encryption round-trip (encrypt→store→decrypt) and a wrong-key failure.
- **MCP:** tool forwarding + arg shaping for each tool, against a faked API client (mirrors existing MCP tests).
- **Web:** the planned-workout create/edit form, calendar rendering of planned vs logged events, the connect/detail/resync affordances.

### Rollout / Ops prerequisites

Phases ship in order; each is independently valuable (1–2 deliver planning + the agent with zero Google surface). Before Phase 3 can deploy:

1. **Google Cloud console:** enable the Calendar API and add `calendar.events` to the OAuth consent screen for the existing client. One-time, manual.
2. **Secret:** provision `CALENDAR_TOKEN_ENC_KEY` (32 random bytes) via the API's existing secret delivery, managed in `prog-strength-infra`. Losing/rotating this key invalidates stored refresh tokens (users re-consent) — document the rotation procedure.

All migrations are additive; no existing data changes. The feature is dormant for any user who never connects a calendar.

## Open Questions (resolved here; listed for traceability)

The brief's open questions are resolved in the body above. Summary of the decisions and the few that remain genuinely deferrable:

- **Token encryption** → app-level AES-256-GCM, key in env; KMS deferred.
- **OAuth upgrade** → second incremental server-side flow (the API owns OAuth; there is no Auth.js). 
- **Plan↔session link** → nullable polymorphic FK, explicit linkage.
- **Detail default** → both (user default + per-workout override).
- **Google update granularity** → full rewrite via idempotent `events.patch`.
- **Runs vs lifts** → lift-only v1, schema-extensible to runs.
- **Timezone** → persisted on the user + on each plan; canonical for server-side writes.
- **Failure modes** → PS source of truth, best-effort writes, resyncable status, no queue.
- **Remaining/deferrable:** the precise `users.calendar_default_detail` UX placement (settings vs first-connect prompt), and whether `resync` should eventually become an automatic background sweep — both safe to settle during Phase 3 implementation.
