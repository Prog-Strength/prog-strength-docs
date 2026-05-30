# Persistent Chat Sessions

**Status**: Draft · **Last updated**: 2026-05-30

## Introduction

Chat is the way a **Prog Strength** user logs workouts and asks the coach about their training, but today every conversation is throwaway. Close the tab or background the app and the history is gone — even refreshing the web page wipes the screen. The agent itself is stateless on purpose (each `/chat` call ships the full message array), and neither the API nor the agent persists what was said. That's fine for "ask one question" but not for a product where lifters routinely come back to a thread to clarify yesterday's log or extend the same line of questioning across sessions.

Users want chat history. Specifically:

- After sending a few messages, the conversation should still be there tomorrow.
- Past conversations should be browsable from a list — "what did I ask about back squats two weeks ago?" should be answerable.
- Tapping a past conversation should reopen it and let the user keep going from where they left off.
- Old conversations shouldn't grow unbounded — there needs to be a cap so storage doesn't become a per-user liability.

After this work ships, every chat the user sends gets saved as part of a named session, the chat surface gains a history list on both web and mobile, and the oldest sessions evict automatically once a per-user cap is hit. The agent itself stays stateless; persistence lives in the API alongside the rest of the user data.

## Proposed Solution

A new `chat` domain in `prog-strength-api` owns two SQLite tables — `chat_sessions` (one row per conversation) and `chat_messages` (one row per turn-side). The API exposes a small CRUD surface for sessions plus an append endpoint for turns. The agent stays unchanged: it remains stateless, and the client continues to send `messages: [...]` to `/chat` exactly as today.

The client owns the lifecycle. When the user opens chat fresh, the client calls `POST /chat-sessions` to mint a new session id, then streams the first turn against the agent as usual. After the stream completes, the client posts the turn (the user message + the assistant reply + any tool metadata) to the API. The session's title is set server-side from the first user message — first ~60 characters, trimmed. Subsequent turns append in the same shape.

Browsing history is a `GET /chat-sessions` paginated list ordered by `last_message_at DESC`. Opening a session is `GET /chat-sessions/{id}` which returns the session plus its full message array; the client renders that, then continues exactly as it would for a fresh session — the user types, the client streams against the agent (sending the rehydrated message history), and the new turn appends back to the same session.

Pruning is **count-based + eager**: a per-user cap of 50 sessions. When the client creates session #51, the API deletes the oldest session (by `last_message_at`) inside the same transaction, cascading to its messages. There's no grace period and no undo — evicted sessions are gone. The cap is generous enough that day-to-day use never trips it; it only kicks in for users who let conversations pile up over months.

Both the web client and the mobile client gain a "Chat history" surface — a side panel/drawer on web, a screen pushed onto the chat tab's stack on mobile — plus a "New chat" affordance for explicitly starting a fresh session.

## Goals and Non-Goals

### Goals

- Persist every chat turn the user completes (user message + assistant reply + any tool metadata) against a `chat_sessions` row.
- Auto-generate a session title from the first user message (first ~60 chars). No manual rename in v1.
- Surface a history list on both web (sidebar / drawer) and mobile (history screen reachable from the chat tab). Each row shows the title and the relative time of the last message.
- Tapping a history entry restores the conversation: the client renders the full message log and the next user turn continues against the same session id.
- Allow the user to delete a session explicitly. Deletion cascades to its messages.
- Cap each user at 50 sessions. Hitting the cap evicts the least-recently-touched session (and its messages) in the same transaction that creates the new one.
- Agent stays stateless: no changes to `/chat`'s contract, no agent → API write path, no new persistence inside the agent.
- API surface stays small and shaped like the rest of the domain repos: a single `chat.Handler` mounted on the chi router with `Mount(chi.Router)`, `chat.Repository` interface with in-memory + SQLite implementations, response envelope via `httpresp`.

### Non-Goals

- **Search across sessions.** The history list is chronological and scannable; full-text search of past conversations is a follow-up.
- **Tagging / categorization / folders.** Users can rely on the auto-generated title alone.
- **Title editing.** The auto-generated title is what's shown. Manual rename is a tiny endpoint but adds UI and edge cases (validation, length cap, etc.) — file as a follow-up.
- **Multi-device live sync of an in-progress stream.** If the user starts a turn on mobile and the assistant is mid-stream, opening the same session on the web doesn't show that turn streaming live. The turn appears once it finishes and the client appends it.
- **Stream-resume after a client crash.** If the user closes the tab while the assistant is still streaming, the turn is lost. The persisted history only contains completed turns. Recovering partial streams would require server-side buffering and a resume protocol — much bigger scope.
- **Per-message edit / delete.** Sessions are append-only from the user's perspective. The whole session can be deleted; individual turns can't be cherry-picked.
- **Sharing or exporting sessions.** No public links, no `.txt` / `.md` export.
- **Cross-user references.** Sessions are strictly per `user_id`; never visible to anyone else.
- **Token-level cost tracking.** The persisted assistant message stores the model name (e.g. `claude-sonnet-4-6`) but not input/output token counts. Cost analysis is a separate telemetry concern.

## Implementation Details

### Data Model

Two new tables in `internal/db/migrations/008_chat_sessions.sql`.

`chat_sessions`:

| Column | Type | Description |
| --- | --- | --- |
| `id` | text | UUIDv4 generated client-side and passed to the API on create. |
| `user_id` | text | Owning user. No FK constraint (matches the existing `workouts` table convention). |
| `title` | text | Auto-generated from the first user message (first ~60 chars, trimmed). Empty string until the first message lands. |
| `created_at` | datetime | First write. |
| `updated_at` | datetime | Bumped on any session-level change (title set, delete). |
| `last_message_at` | datetime | Bumped to the most recent message's `created_at`. Drives the history list sort + the eviction order. |
| `deleted_at` | datetime | Soft-delete sentinel. Read paths filter `deleted_at IS NULL`. |

Indexes:

- One unique index on `(user_id, last_message_at DESC) WHERE deleted_at IS NULL` — serves the history list query and the "find the oldest session for this user" query the eviction path needs.

`chat_messages`:

| Column | Type | Description |
| --- | --- | --- |
| `id` | integer | Autoincrement primary key. |
| `session_id` | text | `REFERENCES chat_sessions(id) ON DELETE CASCADE`. Eviction of a session takes its messages with it. |
| `position` | integer | 0-indexed order within the session. Unique per `session_id` via a composite index. The position is set by the API at append time as `(current_max + 1)`; clients don't supply it. |
| `role` | text | `CHECK (role IN ('user', 'assistant'))`. The Anthropic API's role enum. |
| `content` | text | The visible message text. |
| `model` | text | Nullable. Set to the Claude model name on assistant rows (`claude-sonnet-4-6`, etc.) so the history view can surface "via Sonnet" badges the way live chat does today. Null for user rows. |
| `tools_json` | text | Nullable. JSON blob of tool calls that fired during this assistant turn (name + state). The chat UI re-hydrates these to show the same "agent called X" hint chips it shows live. Null for user rows. |
| `created_at` | datetime | Set server-side at append. |

Indexes:

- Composite index on `(session_id, position)` — primary read pattern is "give me every message in this session in order".

Foreign keys:

- `chat_messages.session_id REFERENCES chat_sessions(id) ON DELETE CASCADE`. The eviction path relies on CASCADE so the API can delete one session row and trust SQLite to clean up messages in the same transaction.

No FK from `chat_sessions.user_id` to `users.id` — matches the existing `workouts` table convention (users come from OAuth and aren't necessarily in the DB before the first write).

### Write Path

Three endpoints touch the write side. All require auth.

**`POST /chat-sessions`**

Creates an empty session for the authed user. Request body: `{ "id": "<uuidv4>" }` — the client mints the id so optimistic UI can render the new session immediately while the request flies.

Inside one transaction:

- **Eviction check.** Count the user's `deleted_at IS NULL` sessions. If the count is `>=` the cap (default 50), find the oldest by `last_message_at` and `DELETE` it. CASCADE removes its messages.
- **Insert** the new row with `title = ''`, timestamps set to `now()`, `last_message_at = now()`.

Returns the created session.

**`POST /chat-sessions/{id}/messages`**

Appends one turn (the user message + the assistant reply). Request body shape:

```json
{
  "user": { "content": "How was last week's volume?" },
  "assistant": {
    "content": "Looks like 4 sessions, 12k total reps...",
    "model": "claude-sonnet-4-6",
    "tools": [{ "name": "list_workouts", "state": "ok" }]
  }
}
```

Inside one transaction:

- **Authorize.** Reject 404 if the session doesn't exist, was soft-deleted, or belongs to another user.
- **Set the title if empty.** On the first append (current `title == ''`), set `title = firstSixtyChars(user.content)`.
- **Append two rows** to `chat_messages`. The user row first, then the assistant row, with `position = current_max + 1` and `current_max + 2` respectively.
- **Bump session timestamps.** `updated_at = now()`, `last_message_at = now()`.

Returns the updated session + the two new message rows.

The two-rows-per-turn shape (vs one row with both sides) matches the Anthropic role enum and lets future features (e.g. mid-turn edit, retry an assistant reply) land without a schema rewrite.

**`DELETE /chat-sessions/{id}`**

Soft-deletes the session (sets `deleted_at = now()`). Messages aren't touched on soft delete — they hang off the row and become invisible. The eviction path uses a hard `DELETE` (CASCADE-aware) rather than soft delete so storage is actually reclaimed; user-initiated delete uses soft delete so a future "trash" / restore feature is cheap to layer on top. The two paths are documented in the handler.

### Read Path

Two endpoints serve the history surface. Both require auth.

**`GET /chat-sessions`**

Lists the authed user's non-deleted sessions, most-recently-active first. Query params:

- `limit` (default 50, max 50 — matches the eviction cap so there's never a need to paginate beyond it)
- `offset` (default 0; included for completeness even though normal use never trips it)

Response: array of `{ id, title, last_message_at, message_count }`. The `message_count` is denormalized at the SQL level via a `LEFT JOIN ... COUNT` so the UI can show "12 messages" badges without a follow-up query. (This is a one-statement query thanks to the join, not an N+1 fan-out — same discipline as the workout-list batched-hydration SOW.)

**`GET /chat-sessions/{id}`**

Returns the full session + every message in `position ASC` order. Two batched queries (the session row + a single `SELECT * FROM chat_messages WHERE session_id = ? ORDER BY position`) — bounded by the cap on messages per session (no explicit cap today, but practically a session rarely exceeds 50 turns).

If the session is soft-deleted or owned by another user, return 404.

### API Surface

| Path | Method | Auth | Purpose |
| --- | --- | --- | --- |
| `/chat-sessions` | GET | required | List the authed user's sessions, most-recently-active first. |
| `/chat-sessions` | POST | required | Create a new (empty) session. Body: `{ id }`. Evicts the oldest session when at cap. |
| `/chat-sessions/{id}` | GET | required | Return the session + all its messages. 404 on miss / soft-delete / other user. |
| `/chat-sessions/{id}` | DELETE | required | Soft-delete the session. |
| `/chat-sessions/{id}/messages` | POST | required | Append one turn (user + assistant rows in a single transaction). Sets the session title on the first call. |

No new endpoints on the agent. The agent's `/chat` continues to accept a `messages: [...]` array and stream a reply.

### Algorithms

Title generation is intentionally dumb:

```
title = firstUserMessage.trimSpace().truncate(60).trimSpace()
```

Edge cases handled:

- Empty user message after trimming → fall back to `"New chat"`.
- Truncation lands mid-word → accept it; no fancy word-boundary handling for v1.

An LLM-generated title (e.g. "ask Claude to summarize the conversation in four words") is more reader-friendly but adds an extra round-trip and per-session cost. Not worth it for v1 when the user's first message is usually self-explanatory.

### Backfill or Migration

No backfill — chat is unpersisted today, so the migration only creates the empty tables. Existing users start with zero sessions; their next chat creates the first one.

If/when we ever want to retroactively snapshot live conversations into history we'd add a one-time write-on-stream path, but that's strictly forward-looking.

### Web client changes (`prog-strength-web`)

- `app/(app)/chat/page.tsx` — the existing chat page. Two changes:
  - On mount, generate a UUIDv4 client-side and `POST /chat-sessions` to mint a session (or load an existing one if the URL is `/chat?session=<id>`). Render the existing UI either against the new empty session or against the rehydrated message history.
  - After each completed stream, `POST /chat-sessions/{id}/messages` with the user + assistant pair. Failures surface inline; the message stays visible regardless so the user isn't punished for a transient write.
- A new history panel — likely a drawer / overlay opened from a sidebar "Chat history" entry (or a "Sessions" button in the chat page header, exact placement is a UX call) that hits `GET /chat-sessions` and renders the list. Tapping a row → `/chat?session=<id>`. A small ✕ on each row deletes (with a confirm).
- Sidebar entry for "Chat history" alongside the existing "Chat" entry.

### Mobile client changes (`prog-strength-mobile`)

- `app/(tabs)/chat.tsx` — same shape as web:
  - Generate UUID + create session on mount (or load by id when arriving via the history screen).
  - After each completed stream, POST the turn.
- A new `app/(tabs)/chat/history.tsx` route pushed onto the chat tab's stack. Header button on the main chat screen ("History" / clock icon) → pushes the history screen. Each row pushes back onto chat with the selected session.
- "New chat" affordance — header button or fab. Same UUID + create flow as initial mount.

### Pruning

Eager only. The cap (50) is enforced inside the `POST /chat-sessions` transaction:

```
SELECT COUNT(*) FROM chat_sessions WHERE user_id = ? AND deleted_at IS NULL
-- if count >= cap:
DELETE FROM chat_sessions
  WHERE id = (
    SELECT id FROM chat_sessions
    WHERE user_id = ? AND deleted_at IS NULL
    ORDER BY last_message_at ASC
    LIMIT 1
  )
INSERT INTO chat_sessions (...)
```

A background sweep (delete sessions older than N days regardless of cap) was considered and rejected — eager keeps the data lifecycle in the request path, which is easier to reason about and trivially correct: storage can never grow past `cap × users`. If a follow-up adds a per-user time-based retention preference, that's the moment to add a background job.

Frontend pruning is purely cosmetic — the API caps at 50 so the UI never needs to render more than that. The history list's `limit=50` query naturally returns everything the user has.

## Open Questions

1. **Should `chat_messages.tools_json` be a real `tool_calls` table instead of a JSON blob?** A normalized table makes "show me every workout the agent logged for me last month" trivially queryable. The blob is simpler and matches how the chat UI consumes tools (opaque metadata for badge rendering). Tentative lean: blob for v1, lift to a table only if/when an analytical query actually needs it. The JSON we store stays the same shape either way, so a future migration would be a one-time table-fill rather than a contract break.
2. **Where does the history list live on web — a drawer, a dropdown, or its own route?** A drawer keeps it overlaid on the chat surface (less navigation friction). A dropdown is more compact but limits the row count visible at once. A dedicated `/chat/history` route makes the URL meaningful (shareable by the user to themselves across devices). Tentative lean: dedicated `/chat/history` route with a "Recent" dropdown on the main chat page as a quick-jump shortcut — both ergonomics, neither expensive.
3. **What happens when the user picks a history entry mid-stream of a current conversation?** Option A: confirm dialog ("Switching will abandon the in-progress reply. Continue?"). Option B: silently abandon — the stream's already going to be lost on navigation anyway. Tentative lean: silent abandon; the agent reply isn't persisted until the stream completes, and the user clicking a history row is unambiguous intent.
4. **Should the eviction cap be configurable per user?** A "Pro" tier could plausibly raise it. v1 hardcodes 50 as a constant in the chat package; the cap moves to a per-user column or a config table if/when there's a billing tier that depends on it. Tentative lean: hardcoded for v1, no per-user override yet.
5. **Title regeneration on edit?** If the user's first message was "hi" and the assistant asks "what would you like to know?" and then the real question comes in turn 2, the title is "hi" forever. We could re-derive the title from the first non-trivial message, or expose a manual rename. Tentative lean: leave the dumb title alone for v1, surface manual rename as a small follow-up.
