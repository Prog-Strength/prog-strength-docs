---
status: shipped
repos:
  - prog-strength-api
  - prog-strength-web
  - prog-strength-mcp
  - prog-strength-infra
  - prog-strength-docs
---

# Whoop Integration (Recovery v1)

**Status**: Shipped · **Last updated**: 2026-07-22

## Introduction

**Prog Strength** knows what a lifter does — workouts, runs, steps, bodyweight — but nothing about how recovered they are while doing it. Wearables have that signal, and Whoop exposes it through the most application-friendly API of the major devices: a standard OAuth 2.0 authorization-code flow, a documented v2 REST API, and push webhooks when new data is scored. This SoW builds the first device integration on that foundation: a user connects their Whoop account from Settings, and their daily recovery metrics — resting heart rate, HRV, and recovery score — flow into Prog Strength automatically.

Two things make this more than a single-feature build. First, it establishes the *pattern* for every future device integration (Apple Health, Garmin, Strava): an encrypted third-party token connection, a provider card on a new Settings → Integrations tab, and a per-user-per-day metric table. Second, it forces one small piece of shared infrastructure into existence — extracting the AES-256-GCM token crypto out of `calendarsync` into a reusable package — so the third and fourth providers are additive, not copy-paste.

The Google Calendar sync feature is the direct blueprint: `internal/calendarsync` + `internal/calendarconn` on the API, `GoogleCalendarConnectionRow` on the web. Whoop mirrors its shape end to end, with two deliberate deviations covered below: Whoop rotates refresh tokens (single-use), and ingestion is inbound (Whoop calls us) rather than outbound-on-user-action.

## Proposed Solution

A user clicks **Connect** on a new Whoop card under Settings → Integrations and completes Whoop's OAuth consent (`read:recovery read:profile offline`). The API stores the token pair encrypted at rest in a new `user_whoop_connection` row, captures the user's Whoop user ID (required to route webhooks), and immediately backfills the last 30 days of recoveries into a new `user_whoop_recovery` table — one row per user per calendar date, shaped like `user_steps`.

From then on, ingestion is push-driven. Whoop POSTs to a public `POST /webhooks/whoop` endpoint whenever a recovery is scored. The handler validates the HMAC signature and treats the event as a **poke, not a payload**: rather than dereferencing the record ID in the event (which in v2 is a sleep UUID, not a recovery ID), it re-syncs a small recent window of recoveries for that user via `GET /v2/recovery`. Idempotent upserts make Whoop's five-retry delivery, duplicate events, and out-of-order arrival all trivially safe. No scheduler, no queue, no polling — the first background-ish ingestion in the stack costs zero new infrastructure.

On the read side: the dashboard gains a recovery mini-card (today's resting HR with a 7-day spark), a date-windowed `GET /whoop/recovery` endpoint follows the house timezone+local-date convention, and a new MCP `whoop` module lets the agent read recovery data in coaching conversations.

## Goals and Non-Goals

### Goals

- A user can connect their Whoop account from Settings → Integrations, see connection status, and disconnect; disconnect revokes the token at Whoop and marks the connection revoked.
- Daily recovery score, resting heart rate, and HRV persist per user per local calendar date; new scores arrive within minutes of Whoop scoring them, with no user action and no polling.
- Connecting backfills the trailing ~30 days so the dashboard is non-empty on day one.
- The Settings page restructures into **Profile** and **Integrations** tabs; the Google Calendar card moves to Integrations without behavior change; the tab is URL-addressable so OAuth redirects land back on it.
- The dashboard shows a recovery card (today's resting HR + 7-day spark) for connected users, conforming to the design system; users without a connection see no card.
- The agent can read recovery data through a new MCP tool using the standard bearer-forwarding pattern.
- Token crypto is extracted to a shared package; `calendarsync` and Whoop both consume it with no behavior change to calendar sync.
- A token refresh failure (revoked at Whoop's side) degrades gracefully: connection status becomes `error`, the Settings card offers **Reconnect**, and no data is silently dropped without a visible state.

### Non-Goals

- **Sleep, strain, cycle, and workout ingestion.** Whoop exposes them and the connection plumbing built here supports them, but each is its own product surface. Recovery only in v1.
- **Other providers.** Apple Health, Garmin, and Strava each get their own SoW; this SoW's job is to leave them a groove to slot into (Integrations tab, `tokencrypt`, the per-day metric table shape).
- **A dedicated recovery page.** History charts beyond the dashboard spark are a follow-up once there is data worth charting.
- **A background scheduler.** Webhooks make one unnecessary for Whoop. If a future provider (Garmin) forces polling, the scheduler is designed then.
- **Mobile.** The mobile client gets the Integrations surface and recovery tile in a later parity phase.
- **Deleting recovery history on disconnect.** Ingested rows are the user's data and survive disconnect; only the connection and tokens are destroyed. (Whoop-initiated `recovery.deleted` events *do* delete the affected row — that is a data correction, not an account action.)
- **Agent proactivity.** The agent gets a read tool; prompting it to volunteer recovery-aware coaching is future tuning, not scope here.

## Implementation Details

### Data Model (`prog-strength-api`)

Two new tables in `038_user_whoop_connection.sql` and `039_user_whoop_recovery.sql`.

`user_whoop_connection` — one row per user, mirroring `user_calendar_connection` (026) with Whoop-specific additions:

| Column | Type | Description |
| --- | --- | --- |
| `user_id` | `TEXT PK` | FK to `users`. |
| `whoop_user_id` | `INTEGER UNIQUE NOT NULL` | Whoop's numeric user ID, captured at connect from `GET /v2/user/profile/basic`. This is the join key for inbound webhooks, which identify users only by Whoop ID. |
| `access_token_enc` / `access_token_nonce` | `BLOB` | AES-256-GCM. Whoop access tokens live ~1 hour, so unlike the calendar connection the access token is persisted, not held only in memory. |
| `refresh_token_enc` / `refresh_token_nonce` | `BLOB` | AES-256-GCM. **Single-use**: every refresh returns a new pair and invalidates the old (see Token Refresh). |
| `token_expires_at` | `TIMESTAMP` | Access-token expiry from `expires_in`; refreshed proactively when within a small skew window. |
| `scopes` | `TEXT` | Granted scopes as returned. |
| `status` | `TEXT CHECK IN ('connected','revoked','error')` | `error` = refresh failed with `invalid_grant` (user revoked at Whoop, or a lost rotation); surfaces as **Reconnect** in Settings. |
| `connected_at`, `updated_at` | `TIMESTAMP` | Standard. |

`user_whoop_recovery` — steps-shaped daily metric table (cf. `019_steps.sql`), hard-deleted, upsert on conflict:

| Column | Type | Description |
| --- | --- | --- |
| `user_id` | `TEXT` | FK to `users`. |
| `date` | `TEXT` | Local calendar date `YYYY-MM-DD` (derivation below). `UNIQUE(user_id, date)`. |
| `recovery_score` | `REAL` nullable | Whoop 0–100. |
| `resting_heart_rate` | `REAL` nullable | bpm — the v1 display metric. |
| `hrv_rmssd_milli` | `REAL` nullable | ms. |
| `cycle_id` | `INTEGER` | The Whoop cycle the recovery belongs to; join key for date derivation. |
| `sleep_id` | `TEXT` | UUID of the associated sleep. Stored because v2 webhook events identify recoveries by sleep UUID — this is the delete key. |
| `created_at`, `updated_at` | `TIMESTAMP` | Standard. |

Only records with `score_state = SCORED` are upserted; `PENDING` and `UNSCORABLE` are skipped (a later re-score arrives as another webhook and upserts normally). Whoop can update an already-scored day; the upsert overwrites — latest wins.

**Date derivation.** A recovery record carries no timezone; its cycle does. The sync fetches `GET /v2/recovery` and `GET /v2/cycle` over the same window and joins on `cycle_id`; `date` is the recovery's `created_at` (the instant Whoop scored it — wake time plus phone sync) adjusted by the cycle's `timezone_offset`. This pins the recovery to the day the user woke up into, in the timezone they were actually in. (Corrected 2026-07-23, api PR #76: the original spec derived from the cycle's `start`, but Whoop cycles run sleep-onset → sleep-onset, so cycle start is the previous evening's bedtime and every recovery landed one day early — "today" never had data.)

### Token Crypto Extraction (`prog-strength-api`)

`internal/calendarsync/crypto.go` (`NewCipher`, `Encrypt`, `Decrypt`, `KeyFromEnv`) moves to a new shared `internal/tokencrypt` package. `calendarsync` switches to importing it — mechanical, no behavior change, covered by the existing crypto tests moving along with it. Whoop uses the same package with its **own key** (`WHOOP_TOKEN_ENC_KEY`), so a leaked key compromises one provider, not all.

### OAuth & Connection Lifecycle (`prog-strength-api`)

New packages `internal/whoopconn` (model + repository, cf. `calendarconn`) and `internal/whoopsync` (OAuth, Whoop client, sync service, webhook handler, cf. `calendarsync`). Routes mirror the calendar mount split:

| Route | Mount | Behavior |
| --- | --- | --- |
| `GET /auth/whoop/connect` | authed | Builds the authorize URL (`https://api.prod.whoop.com/oauth/oauth2/auth`) with scopes `read:recovery read:profile offline` and the HMAC-bound CSRF state pattern from `calendarsync/oauth.go` (Whoop requires state ≥ 8 chars — the existing scheme already exceeds this). Accepts `return_to` for the post-connect redirect, validated same-origin as calendar does. |
| `GET /auth/whoop/callback` | public | Validates state, exchanges the code at `https://api.prod.whoop.com/oauth/oauth2/token`, fetches `GET /v2/user/profile/basic` for `whoop_user_id`, upserts the connection as `connected`, kicks off the 30-day backfill (best-effort, inline — failure logs and leaves the connection usable; the next webhook or reconnect fills the gap), redirects to `return_to`. |
| `GET /me/whoop/connection` | authed | Status, `connected_at` — shape of `GET /me/calendar/connection`. |
| `DELETE /me/whoop/connection` | authed | Calls Whoop `DELETE /v2/user/access` (best-effort), marks the row `revoked`, wipes token columns. Recovery rows untouched. |

Reconnecting from `revoked`/`error` is just the connect flow again — the callback upsert overwrites the row.

### Token Refresh (`prog-strength-api`)

The Whoop client wraps every outbound call with get-or-refresh: if `token_expires_at` is within skew, POST the refresh grant, then **persist the new access+refresh pair before using it** — with single-use rotation, a crash between rotation and persistence orphans the connection, so the write comes first. SQLite's single-writer semantics plus a per-user serialization in the service (webhook handling and backfill for one user never refresh concurrently) keep the rotation race away. A refresh rejected with `invalid_grant` marks the connection `error` and stops outbound calls until reconnect.

### Webhook Ingestion (`prog-strength-api`)

`POST /webhooks/whoop`, mounted public. Per request:

1. **Verify signature**: `X-WHOOP-Signature` must equal `base64(HMAC-SHA256(X-WHOOP-Signature-Timestamp + rawBody, client_secret))`, constant-time compare, and the millisecond timestamp must be within ±5 minutes. Failures return 401 and log.
2. **Route**: look up the connection by payload `user_id` (Whoop's ID) with status `connected`; unknown or non-connected users are acked 204 and dropped (Whoop retries failures 5× over an hour — never fail an event we will never want).
3. **Handle**: `recovery.updated` → run the recent-window sync for that user (fetch the last ~10 recoveries + matching cycles, upsert each). `recovery.deleted` → delete the row whose `sleep_id` matches the event's `id` (the v2 payload identifies recoveries by sleep UUID — the reason `sleep_id` is a stored column). All other event types → 204, ignored.
4. **Respond fast**: the window sync is a handful of Whoop calls; run it inline and return 2xx. If Whoop latency ever makes this a timeout risk, the escape hatch is ack-then-goroutine — noted, not built.

The poke-not-payload design means the webhook body's `id` field (a sleep UUID in v2, with no direct recovery-fetch endpoint) never needs dereferencing, and every delivery is a full idempotent repair of the recent window.

### Backfill & Read API (`prog-strength-api`)

The connect-time backfill pages `GET /v2/recovery` (`limit=25`, `nextToken`) back ~30 days, joins cycles, upserts — two pages for a typical user, well under any rate limit.

Reads:

- `GET /whoop/recovery?timezone=&since=&until=` — local-date window per the house convention, converted via `internal/daterange`; returns the day rows ordered by date. 404-style empty distinction is not needed — no connection simply yields an empty list, and the connection endpoint answers "is it set up".
- The dashboard summary endpoint grows a `recovery` block: today's row (if any) plus the trailing 7 days of resting HR for the spark. Present only when a `connected` connection exists — the web uses its absence to hide the card.

### Config & Secrets (`prog-strength-api`, `prog-strength-infra`)

Three new `${VAR}` entries in `config.toml` under a `[whoop]` section, wired through `internal/config` exactly like the calendar/Google pair: `client_id = "${WHOOP_CLIENT_ID}"` (deployment-bound, not a hard secret, but env-supplied like `GOOGLE_CLIENT_ID`), `client_secret = "${WHOOP_CLIENT_SECRET}"`, `token_enc_key = "${WHOOP_TOKEN_ENC_KEY}"` (32 bytes, generated once). Redirect URL derives from the configured API base URL as calendar does. Values are added to the GitHub secrets backing the `api` service container and seeded through the existing `seed-secrets.yml` workflow — no new Terraform resources, no new infra module.

### Frontend (`prog-strength-web`)

**Settings tabs.** `app/(app)/settings/page.tsx` gains a two-tab header — **Profile** and **Integrations** — driven by a `?tab=` search param (default `profile`), so `return_to=/settings?tab=integrations` round-trips through OAuth. The existing draft/`SaveBar` mechanics stay scoped to Profile (integration actions were already immediate and out-of-band). The Google Calendar card moves under Integrations unchanged. Tab affordance conforms to the design system (`prog-strength-docs/design-system.md`) — this is the first tab control in Settings, so it should look like the segmented controls already in the system rather than inventing a new pattern.

**Whoop card.** `WhoopConnectionRow` mirrors `GoogleCalendarConnectionRow`: disconnected → **Connect Whoop** navigates to `${config.apiUrl}/auth/whoop/connect?return_to=…`; connected → status line with connected-since and a **Disconnect** action (immediate `DELETE`, confirm affordance consistent with calendar's); `error` status → explanatory line and **Reconnect**. New `lib/api.ts` functions: `getWhoopConnection`, `disconnectWhoop`, plus the `DashboardSummary` type extension.

**Dashboard card.** A recovery `MiniCard` following the existing dashboard grammar: `Kpi` for today's resting HR (with "no data yet today" state), a 7-day `Spark` of resting HR, and the recovery score as a secondary line. Rendered only when the summary contains the `recovery` block. Deep-link target is Settings → Integrations until a recovery page exists.

### MCP (`prog-strength-mcp`)

New `src/prog_strength_mcp/whoop.py` following the `steps.py` template: a `get_whoop_recovery` tool taking `timezone` + `since`/`until` local dates, forwarding the bearer token, calling the new API read endpoint through `APIClient`. The tool description tells the agent what the metrics mean (recovery score 0–100, resting HR bpm, HRV ms) and that an empty result for a user who mentions their Whoop likely means the integration isn't connected — point them to Settings → Integrations. Registered in `server.py`.

### Tests

- `tokencrypt`: existing crypto tests move with the package; add a two-key isolation test.
- `whoopsync`: signature validation (valid, tampered body, stale timestamp, constant-time path), state round-trip, token-exchange and refresh-rotation persistence ordering (new pair stored before old pair discarded; `invalid_grant` → `error` status), date derivation from cycle `timezone_offset` including negative offsets and DST-adjacent dates, window-sync idempotency (same webhook delivered 5× → one row), `recovery.deleted` handling, skip of `PENDING`/`UNSCORABLE`.
- Handler tests mirroring `calendarsync`'s for the four connection routes and the webhook route (unknown Whoop user → 204).
- Web: connection-row state rendering (disconnected/connected/error) and tab param round-trip.

### Rollout & Ops

One-time manual setup (owner: Jimmy), documented here because it cannot be automated:

1. Create the app in the Whoop Developer Dashboard; register redirect URLs for prod (`https://<api-host>/auth/whoop/callback`) and local dev (`http://localhost:8080/auth/whoop/callback`).
2. Register the webhook URL (`https://<api-host>/webhooks/whoop`) in the dashboard, v2 model.
3. Generate `WHOOP_TOKEN_ENC_KEY` (32 random bytes, base64) and add it plus the client ID/secret to the `api` GitHub secrets; run the seed-secrets workflow.

Ship order within the SoW: API (migrations, crypto extraction, connection + webhook + reads) → infra secret seed → web → MCP. First real-world validation is Jimmy connecting his own account and watching the morning webhook land.

## Open Questions

- **Whoop rate limits** are documented only as "429 possible" with no published numbers. The v1 call pattern (webhook-triggered window syncs plus a 2-page backfill per connect) is far below any plausible limit; if 429s appear, honor `Retry-After` on the window sync and rely on Whoop's webhook retries — no proactive throttling built until observed.
- **Whoop developer app approval** — Whoop dashboards have historically gated apps by review tier for >10 users. Fine for beta (single-digit users); worth checking the current policy before public launch.
