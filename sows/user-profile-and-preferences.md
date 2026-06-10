---
status: ready_for_implementation
repos:
  - prog-strength-api
  - prog-strength-agent
  - prog-strength-web
  - prog-strength-infra
  - prog-strength-docs
---

# User Profile & Preferences

**Status**: Ready for Implementation · **Last updated**: 2026-06-09

## Introduction

A **Prog Strength** user's identity is currently a pass-through from their OAuth provider. `display_name` exists on the `users` record but is populated once at signup from the OAuth name claim and never edited again — many providers ship a full legal name there, which is what the agent ends up calling the user even when they'd rather be called something shorter. There is no static body metric on the user record either; the agent has bodyweight time-series data to lean on but no way to know the lifter is 6'4" without asking inside the conversation. And the web sidebar has no identity affordance at all — it's pure navigation with a bare "Sign out" pinned at the bottom, leaving the avatar nowhere to live.

This SoW makes the profile user-editable and gives identity a home in the UI. After it ships, a lifter can pick the name the coach calls them by, record their height once, and replace the OAuth avatar with their own picture. The sidebar grows an account row at the bottom built around that avatar, with Sign out folded into a menu that opens from it.

## Proposed Solution

Add two nullable columns to the existing `users` table — `height_cm` (canonical unit, converted at display edges) and `avatar_key` (S3 object key) — and make the existing `display_name` editable through a new write endpoint. Profile reads go through a single resolved `GET /me` that applies fallback logic server-side: `display_name` falls back to the OAuth name claim when null-effective, and `avatar_key` falls back to the OAuth avatar URL when unset. The avatar URL the client receives is always a freshly minted, time-limited presigned S3 GET.

Avatar storage lives in a new `prog-strength-avatars` S3 bucket keyed as `user_id=<id>/<uuid>.<ext>` — partitioned per user, no date partitioning (there is exactly one current avatar per user; this isn't a time-series). Each upload writes a *new* object with a UUID filename and updates `avatar_key` on the user record; old objects are reaped asynchronously by an S3 lifecycle rule rather than synchronously on upload. This gives a trivially correct "latest wins" without cache-busting headaches on presigned URLs and keeps the upload path fast.

The agent picks up `display_name` and `height_cm` through the same prompt-prefix bootstrap that already injects `client_timezone` — `build_chat_system_prompt` grows two more parameters and prepends a "you are talking to <name>, who is <height>" line to the system prompt. The web client gains a Settings page with the three editable fields, and the sidebar's bottom-left becomes an account row built around the avatar; collapsing the sidebar collapses it to the avatar icon alone.

## Goals and Non-Goals

### Goals

- A user can edit `display_name` from the Settings page; the value persists, the agent calls the user by it, and it appears in the sidebar account row.
- A user can record their height; it persists as `height_cm`, displays in their familiar unit (cm or in), and the agent can reference it in conversation.
- A user can upload an avatar; it displays in the sidebar account anchor; removing it reverts to the OAuth avatar fallback.
- The sidebar bottom-left renders an account row with avatar + display name; Sign out is reachable from a menu opened by clicking it. The row collapses to an icon when the sidebar collapses.
- All profile reads/writes flow through the Go API. Avatar objects live under the partitioned S3 key schema; URLs served to clients are presigned and time-limited.
- Invalid input (out-of-range height, oversized or wrong-content-type image) is rejected with clear errors.

### Non-Goals

- **BMI as a surfaced or computed metric.** Height-derived body composition is a weak signal for a strength-training audience (heavy muscle reads as "overweight"); if any height-derived metric is exposed later, it gets its own SoW with agent-side caveats.
- **Image resizing or transformation pipelines.** Cap upload size and content type at the API; revisit only if bandwidth becomes a real cost.
- **Editing OAuth-managed fields** (email, provider identity). Email change requires re-verification and is its own SoW.
- **Unit-preference UI.** `weight_unit` and `distance_unit` already exist on the user record and are settable at signup. Building a Settings UI to flip them is a likely follow-up but out of scope here.
- **Mobile.** The mobile client gets the same profile surface in a separate SoW; this work is web-only.

## Implementation Details

### Data Model (`prog-strength-api`)

Two new nullable columns on the existing `users` table — no new tables, no migration of existing rows:

| Column | Type | Description |
| --- | --- | --- |
| `height_cm` | `REAL` | Static body metric. Canonical unit is cm; clients convert at the display edge. Nullable — height is optional, never inferred. |
| `avatar_key` | `TEXT` | S3 object key under the `prog-strength-avatars` bucket. Null means "use OAuth avatar fallback." |

`display_name` already exists on `users` and is required (see `internal/user/user.go`). The change here is making it editable post-signup rather than fixed at OAuth onboarding. The `Validate()` rule that rejects empty `display_name` stays — the *resolved* fallback to the OAuth name happens at the read edge in `GET /me`, not by writing the OAuth name into the column.

Single-writer invariant holds: all reads/writes go through the Go API. The MCP server, agent, web, and mobile clients hit HTTP endpoints; none touch SQLite directly.

### API Surface (`prog-strength-api`)

- **`GET /me`** — returns the resolved profile. Server applies fallback logic: `display_name` is the stored value (always non-empty per the existing validator); `avatar_url` is either a freshly generated presigned S3 GET URL (when `avatar_key` is set) or the OAuth-provider avatar URL (when null). Response also includes `height_cm` as-is.
- **`PATCH /me`** — partial update of `display_name` and/or `height_cm`. Empty `display_name` is rejected (mirrors `Validate()`). `height_cm` validated to a sane range (e.g. 50–250 cm); anything outside is a 400 with a clear error.
- **`POST /me/avatar`** — multipart upload. Enforces a size cap (e.g. 2 MB) and a content-type allowlist (`image/png`, `image/jpeg`, `image/webp`). On success: writes a new S3 object with a fresh UUID filename, updates `avatar_key` on the user record, returns the resolved profile.
- **`DELETE /me/avatar`** — clears `avatar_key`. Does not delete the S3 object inline; the lifecycle rule handles cleanup. Returns the resolved profile (which now carries the OAuth avatar URL).

`PATCH /me` deliberately does not touch `avatar_key` — avatar mutation goes through the upload/delete endpoints, which carry their own validation and S3 side effects.

### Write Path

- **Edit name or height** — `PATCH /me` writes the row, transaction-bounded; agent picks up the new value on the next `/chat` request (the client passes the resolved profile in each request, same as `client_timezone` today).
- **Upload avatar** — `POST /me/avatar` validates → uploads to S3 with a fresh UUID key → updates `avatar_key` on the user row. The previous `avatar_key` is overwritten in the DB; the old S3 object remains until the lifecycle rule reaps it.
- **Remove avatar** — `DELETE /me/avatar` nulls `avatar_key`. The S3 object is not deleted inline (see Open Questions for the rationale).

### Avatar Storage & Partitioning (`prog-strength-infra`)

New Terraform module adding the `prog-strength-avatars` S3 bucket. Key schema:

```
s3://prog-strength-avatars/user_id=<id>/<uuid>.<ext>
```

Mirrors the established TCX-import partitioning convention but **without date partitioning** — avatars are not time-series and there is exactly one current avatar per user, so `user_id=<id>/` is the right partition granularity. Each upload writes a fresh UUID-named object:

- Avoids cache-busting problems with presigned URLs (a changed avatar is a new key, so the old presigned URL just expires naturally).
- Gives a trivially correct "latest wins" by updating `avatar_key` on the user record.

Cleanup of orphaned objects (previous avatars after an upload or delete) runs via an **S3 lifecycle rule** rather than synchronous delete on the upload path. The rule's expiry window is a separate decision (see Open Questions).

### Avatar Serving

Presigned S3 GET URLs, generated by the Go API on `GET /me` with a short expiry. The web client fetches a fresh resolved profile per session load, so URL staleness is bounded by session lifetime. The presigned-URL expiry should be longer than the typical session reload cadence but not so long that the URL becomes a long-lived shareable link.

### Agent Integration (`prog-strength-agent`)

Extend `build_chat_system_prompt` in `prog-strength-agent/src/prog_strength_agent/prompt.py` to accept `display_name: str` and `height_cm: float | None`. The function already prepends a `client_timezone` line; it grows a sibling line of the form:

```
You are talking to <display_name>. They are <height_cm> cm tall.
```

(Height line omitted when `None`.) The `/chat` request payload from the web client carries these alongside `client_timezone`, sourced from the resolved profile the client already holds for the sidebar. No new MCP tool is introduced — height and name are context, not data to query.

**Guidance for the agent**: height is an input for conversational context, not a body-composition signal. The system prompt should not volunteer height-derived inferences (e.g. "you should weigh X for your frame"). If the user explicitly asks, the agent can answer using the value but should not editorialize on BMI or similar.

### Frontend (`prog-strength-web`)

- **Settings page** — three editable fields:
  - Display name (text input, required, non-empty).
  - Height (numeric input in the user's familiar unit per `distance_unit` — `cm` or `in` — stored as `height_cm`). Conversion at input and display only.
  - Avatar (image picker with preview + a "Remove" button when an avatar is set; preview shows the OAuth avatar when none is set).
- **Sidebar account anchor** — bottom-left of the sidebar, an account row composed of the resolved avatar + display name. Click opens a small menu that contains "Sign out" (consolidating today's standalone Sign out) and any other future account actions. When the sidebar collapses, the row collapses to just the avatar icon; the menu still opens from it.
- **Settings nav item** — remains the entry point for editing the profile; the account anchor is for identity display and session actions only.
- Visual style matches the existing dark theme. The Settings page reuses existing form primitives.

### Backfill or Migration

- **`height_cm` and `avatar_key`** are nullable; no backfill. Existing users see "no height set" on the Settings page and the OAuth avatar in the sidebar until they edit.
- **`display_name`** is unchanged — it's already populated for every existing user from OAuth at signup, so the new "edit name" UI just exposes the existing column.
- The `prog-strength-avatars` bucket starts empty. No object backfill.

## Open Questions

1. **Old avatar object cleanup.** Async S3 lifecycle rule (current plan, fast upload path, modest storage cost while objects age out) vs. synchronous delete-on-upload (no lifecycle policy needed, slightly slower upload due to the second S3 call). Tentative lean: **lifecycle rule**, with the expiry window itself a sub-decision (probably 7 days — long enough to recover from a botched upload, short enough not to accumulate). Flip to synchronous if the lifecycle policy adds operational complexity disproportionate to the storage cost.
2. **Presigned URL expiry window.** Long enough for an opened tab to keep rendering the avatar across navigation (and survive a brief network blip) but short enough that a leaked URL isn't a long-lived hotlink. Options: 15 min, 1 h, 24 h. Tentative lean: **1 hour** — typical session lifetimes are well under that, the frontend re-fetches the resolved profile on cold load anyway, and 24 h starts feeling shareable.
3. **Display-name length cap.** No cap today on the existing column. The sidebar row needs *some* upper bound or it visually breaks. Options: 60 chars (matches typical OAuth name length), 32 chars (tighter, forces user to pick a short handle). Tentative lean: **60**, enforced server-side at `PATCH /me`, with client-side soft truncation on the sidebar row.
