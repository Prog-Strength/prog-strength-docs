# Photo Meal Logging — Implementation Plan

> **For agentic workers:** Implement task-by-task. Each task is implemented by a
> subagent, then spec-reviewed and code-quality-reviewed before moving on.
> Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let a user attach one image (receipt or plate of food) to a chat turn
and log the meal from it. The web composer gains a paperclip / drag-drop / paste
affordance; the user turn ships a multimodal `content` block to the agent; the
agent forces a vision-capable model (Sonnet 4.6) for image turns, skips the
intent classifier, and gets a prompt paragraph teaching it to propose macros and
log via the existing `log_custom_meal` tool after confirmation. No image bytes
are persisted anywhere — the placeholder string `[image attached] <text>` is the
only record after the request completes.

**SOW:** `sows/photo-meal-logging.md`

**Hard dependency (already shipped):** `sows/custom-meals.md` —
`log_custom_meal` is the downstream tool, already present in `main` of every
repo. This SOW does not re-spec it.

**This is a new on-ramp to an existing tool, not a new entity.** No MCP changes.
No DB schema change to the application database. The only persistence-layer
change is an additive telemetry column (`had_image`) so usage is measurable.

## Repos, toolchains, merge order

- `prog-strength-api` — Go 1.25.3, module
  `github.com/jwallace145/progressive-overload-fitness-tracker`. Telemetry only.
- `prog-strength-agent` — Python ≥3.12, `uv`. Routing + prompt + telemetry +
  tests.
- `prog-strength-web` — Next.js 16 (read `node_modules/next/dist/docs/` before
  writing Next code — this is NOT the Next.js in your training data), no test
  runner; verify with `tsc`/`eslint`.
- `prog-strength-docs` — this plan + the SOW status flip.

**Merge order (per SOW rollout):** agent first (backwards-compatible: text-only
turns route exactly as today), then web. The API telemetry change is independent
and additive. Each PR is independently revertable.

---

## Repo: prog-strength-api

### Task API1 — Telemetry `had_image` column

The agent will POST a `had_image` boolean on `/internal/telemetry/turns`. The
handler decodes with `json.NewDecoder(...).Decode` (no `DisallowUnknownFields`),
so an unknown field would be silently dropped — but to actually persist and
query the signal we add the column, mirroring the precedent set by
`telemetry_migrations/002_agent_turns_intent.sql`.

Files:
- `internal/db/telemetry_migrations/003_agent_turns_had_image.sql` (new)
- `internal/telemetry/telemetry.go` (`AgentTurn` struct)
- `internal/telemetry/handler.go` (`turnRequest` + mapping in `turn`)
- `internal/telemetry/sqlite_repository.go` (`InsertTurn` column list + value)
- existing telemetry tests as needed (`handler_test.go`,
  `sqlite_repository_test.go`)

- [ ] Create `003_agent_turns_had_image.sql` with a header comment in the style
  of 002:
  ```sql
  ALTER TABLE agent_turns ADD COLUMN had_image INTEGER NOT NULL DEFAULT 0;
  ```
  `had_image` is a boolean stored as 0/1; older clients that omit it default to
  `false` (0) via the column default AND the Go zero value, so existing turns
  and pre-feature agents keep writing without a 400.
- [ ] Add `HadImage bool` to the `AgentTurn` struct (place it adjacent to the
  intent fields with a short comment: true when the user's turn carried an image
  content block).
- [ ] Add `HadImage bool \`json:"had_image"\`` to `turnRequest` and set
  `HadImage: req.HadImage` in the `AgentTurn` literal in `turn()`.
- [ ] Extend `InsertTurn`'s `INSERT INTO agent_turns (...)` column list, the
  `VALUES (?, …)` placeholder count, and the argument slice with `t.HadImage`.
  SQLite stores Go `bool` as 0/1 via the driver — match how
  `IntentPrefetchFailed` is already bound.
- [ ] Update telemetry tests that assert the round-tripped turn so they cover
  `had_image` (both true and the default-false path). Confirm the migration test
  (if one enumerates expected columns) includes the new column.

**Verify:** `go build ./...`, `go test ./internal/telemetry/...`, and the
migration runner test all pass.

---

## Repo: prog-strength-agent

Merge this repo first. All changes are backwards-compatible — a text-only turn
takes exactly today's path.

### Task AG1 — `_has_image` detection + vision routing + `had_image` telemetry

Files: `src/prog_strength_agent/server.py`,
`src/prog_strength_agent/telemetry.py`.

- [ ] In `telemetry.py`: add `had_image: bool = False` to `TurnInstrumentation`
  (adjacent to the intent fields) and add `"had_image": t.had_image` to
  `_build_turn_payload`.
- [ ] In `server.py`: add a module-level `VISION_MODEL = config.complex_model`
  with a comment (vision turns force the capable tier — Sonnet 4.6 today —
  sourced from the same config slot the `complex` harness already pins; this
  guarantees a vision turn never lands on Haiku). The existing
  `HARNESSES["complex"]` is already constructed with `config.complex_model`, so
  reuse it as the vision harness rather than constructing a second one.
- [ ] Add a pure helper `_has_image(messages: list[dict[str, Any]]) -> bool`
  exactly as the SOW specifies: true iff any `role == "user"` message has a
  list `content` containing a block with `type == "image"`. Keep it pure and
  side-effect-free so it is unit-testable (the same pattern as
  `_maybe_inject_timezone`).
- [ ] In `_route_and_stream`, branch before the classifier:
  - When `_has_image(messages)`: set `telemetry.had_image = True`, do **not**
    call `api_client.get_session_intent` or `router_obj.route`, select
    `harness = HARNESSES["complex"]`, and set `telemetry.routed_tier =
    "complex"` and `telemetry.intent = "general"` so Prometheus
    (`record_prometheus_metrics`) still records the turn (it skips turns with an
    empty `routed_tier`). The harness emits `model_chosen` with its own
    `self.model` (== `VISION_MODEL`) as the first SSE event, unchanged.
  - Else: today's path exactly (prior_intent lookup → `router_obj.route` →
    `HARNESSES.get(decision.tier, …)`), with `telemetry.had_image` left `False`.
  - Pass `intent` to `harness.stream_chat` accordingly (`"general"` on the
    vision branch).
  - Keep the `finally:` telemetry block unchanged.

**Why "general" intent on the vision branch:** the intent registry's prefetch
for `general` is a no-op, so the system prompt is the base prompt (plus the new
image paragraph from AG2) with no spurious prefetch — correct for a vision turn.

**Verify:** `uv run pytest -q` green (AG3 adds the new coverage).

### Task AG2 — Prompt image-handling paragraph

File: `src/prog_strength_agent/prompt.py`.

- [ ] Add the image-handling guidance to `SYSTEM_PROMPT`, placed immediately
  after the existing "Logging a meal the user describes in chat." paragraph
  (the nutrition/logging guidance cluster), before `## Tone`. Use the SOW's
  wording (receipt → list items + per-item macros + confirm; plate → visible
  portion, note partially-obscured sides; menu/ambiguous → ask what they had;
  corrections → revise + re-ask, don't log eagerly; never call `log_custom_meal`
  on the first image turn — always propose first, log on the user's "yes"). Keep
  the existing backslash-continuation line style so the rendered prompt has no
  stray hard newlines mid-sentence.
- [ ] Do not alter `build_chat_system_prompt`'s date prefix or
  `compose_system_prompt`.

**Verify:** `uv run pytest -q` (existing `test_prompt.py` still green).

### Task AG3 — Vision routing tests

File: `tests/test_model_harness.py` (add to it; do not disturb existing tests).

- [ ] `test_has_image_detects_image_block` / `…_false_for_text_only` /
  `…_ignores_assistant_image`: unit-test the pure `_has_image` helper against
  string content, a `[image, text]` user block, a text-only list, and an
  assistant-role message carrying an image (must be False — only user turns
  count).
- [ ] `test_vision_turn_skips_intent_classifier_and_routes_to_sonnet`: drive
  `_route_and_stream` with a one-message `[image, text:"log this"]` conversation.
  Monkeypatch `server.router_obj.route` to record-and-fail if called, and
  monkeypatch `server.HARNESSES["complex"]` with a fake harness whose
  `stream_chat` is an async generator yielding a `model_chosen` event carrying
  its `self.model`. Assert: `router_obj.route` was **not** called; the fake
  complex harness was the one invoked; the streamed `model_chosen.model ==
  VISION_MODEL`; and `telemetry.had_image is True`. Use a fake
  `TurnInstrumentation` and a dummy token. (Set `server.api_client = None` for
  the test so the prior-intent lookup is skipped regardless.)
- [ ] `test_text_only_turn_unchanged`: a text-only `messages` list still calls
  `router_obj.route` (monkeypatched to return a known `RouterDecision`) and
  selects the matching harness; assert `telemetry.had_image is False`. Regression
  guard.
- [ ] Proposes-before-logging behavioral coverage: the SOW's
  `test_vision_turn_proposes_before_logging` asserts the model proposes on the
  image turn and only fires `log_custom_meal` after a "yes". Driving the full
  Anthropic streaming context manager + MCP session is heavier than the existing
  test infra supports (see the comment block already in this file). Implement it
  at the seam that is actually testable: assert that the routing/telemetry layer
  forwards the multimodal user content to the harness **unchanged** (no
  rewriting of the image block) and that nothing in the routing path invokes a
  tool. If a faithful end-to-end mock proves tractable within the harness's
  existing helpers, prefer it; otherwise document the seam in a comment exactly
  as the timezone helper block does, so the coverage rationale is explicit. Do
  not leave a skipped/empty test.

**Verify:** `uv run pytest -q` all green.

---

## Repo: prog-strength-web

Merge after the agent. Read `node_modules/next/dist/docs/` before writing code.
The whole feature is concentrated in `app/(app)/chat/page.tsx` plus small type
additions in `lib/`.

### Task WEB1 — Types + base64 helper

Files: `lib/agent.ts`, `lib/api.ts`.

- [ ] Add an exported `ContentBlock` discriminated union (text block + image
  block with `source: {type: "base64"; media_type: "image/jpeg"|"image/png"|
  "image/webp"; data: string}`) in `lib/agent.ts`. Export a `ChatRequestMessage`
  (or widen the existing in-flight message type) whose `content` is
  `string | ContentBlock[]`. This is the **in-flight** (current-session) shape
  the page posts to the agent — keep it distinct from `lib/api.ts`'s persisted
  `ChatMessage` (whose `content` stays `string`; that is the stored form). Do
  **not** widen `appendChatTurn`'s payload type or the persisted `ChatMessage`.
- [ ] Add a small `blobToBase64(blob: Blob): Promise<string>` helper (a
  `FileReader.readAsDataURL` wrapper that strips the `data:…;base64,` prefix and
  returns the raw base64). Co-locate it where the page can import it (e.g.
  `lib/agent.ts` or a small `lib/image.ts`); file-local in the page is also
  acceptable if cleaner — choose one and keep it single-sourced.
- [ ] No `any` casts. `tsc --noEmit` and `eslint` on the touched files clean.

### Task WEB2 — Composer affordance + send + persistence + scrollback render

File: `app/(app)/chat/page.tsx` (and the shared toast via
`@/components/toast`'s `useToast`).

State + helpers (per SOW §Implementation Details):
- [ ] Add `pendingImage` state (`{blob, previewUrl, filename, mediaType} | null`)
  and a `fileInputRef`. Add file-local `MAX_IMAGE_BYTES = 5*1024*1024` and
  `ALLOWED_TYPES = new Set(["image/jpeg","image/png","image/webp"])`.
- [ ] `acceptImageFile(file, toast)`: reject wrong MIME (`toast.error("Use JPG,
  PNG, or WebP.")`) and oversize (`toast.error("Image must be under 5 MB.")`),
  else return `{blob, mediaType}`. The page reads `useToast()` from
  `@/components/toast` (returns `{success,error,info,…}`); the layout already
  wraps the app in `ToastProvider`.
- [ ] `onSelectImage(file)`: validate, revoke any previous `previewUrl`, set
  `pendingImage` with a fresh `URL.createObjectURL(blob)`. `onDismissImage()`:
  revoke + clear.

Composer JSX (footer, `mx-auto … max-w-2xl` row):
- [ ] Add a `PaperclipIcon` local component (16×16 viewBox, 1.75 stroke, rounded
  join — match `MicIcon`'s vocabulary).
- [ ] Insert the paperclip `<button>` after the mic button, before the textarea,
  with the hidden `<input type="file" accept="image/jpeg,image/png,image/webp">`
  wired to `onSelectImage` and resetting `e.currentTarget.value` after. Disable
  on `streaming || loading || !sessionId`.
- [ ] Drag-and-drop: wrap the footer in `onDragOver`/`onDrop` handlers routing
  `dataTransfer.files[0]` through `onSelectImage`; suppress the default
  document-level drop so the browser doesn't navigate to the file.
- [ ] Paste: textarea `onPaste` handler picks the first `image/*` item from
  `clipboardData.items` and routes through `onSelectImage`.
- [ ] Preview chip above the textarea (rendered when `pendingImage != null`):
  48×48 thumbnail from `previewUrl`, truncated filename, and a `×` /
  `onDismissImage` button — markup per the SOW.
- [ ] Send gating: enable on `input.trim().length > 0 || pendingImage !== null`
  (still gated by `streaming || loading || !sessionId`). Update both the
  computed disabled expression and the button.

Send + persistence + render (per SOW §Web → agent payload / persistence):
- [ ] In `send()`: drop the leading `if (!trimmed …) return` so an image-only
  turn (empty text) can send; gate instead on
  `(!trimmed && !pendingImage) || streaming || loading || !sessionId`.
- [ ] Build `userContent`: when `pendingImage`, an array
  `[{type:"image", source:{type:"base64", media_type, data: await
  blobToBase64(blob)}}, {type:"text", text: trimmed || " "}]`; else `trimmed`.
  Post it as the current user turn's `content` in the `messages` array to
  `${agentUrl}/chat`, preserving the streaming path and all existing fields
  (`session_id`, `client_timezone`, `voice_mode`).
- [ ] The optimistic in-scrollback user `Message` must render the image inline.
  Widen the page's local `Message.content` to `string | ContentBlock[]` (import
  the type from WEB1) and render the user image in `MessageBubble`. **Render the
  image from the content block's base64 as a `data:` URL** (`data:${media_type};
  base64,${data}`) rather than the composer's blob URL — this avoids any blob-URL
  lifecycle coupling with the chip (which is revoked on send) and cannot leak.
  Text part renders as today. Assistant rendering is unchanged (always string).
- [ ] Persistence: ship the placeholder string via `appendChatTurn` —
  `pendingImage ? (trimmed ? \`[image attached] ${trimmed}\` : "[image
  attached]") : trimmed`. Image bytes never reach the API. The assistant half of
  the turn is unchanged.
- [ ] After the stream starts successfully, clear `pendingImage` and revoke its
  chip `previewUrl`. On send failure, keep behavior consistent with today's
  rollback.
- [ ] Title generation: keep passing the **string** form (placeholder or text)
  to `titleAndPatch` — `/title` and the persisted message are string-typed.
- [ ] On session resume, persisted user turns come back with `[image attached]
  …` already in `content` (a plain string) and render verbatim — no marker
  special-casing. `persistedToUI` already maps `content` straight through;
  confirm the widened `Message` type still accepts the string and the bubble
  renders it as text.

**Verify (no web test runner):**
- `npx tsc --noEmit` clean.
- `npx eslint "app/(app)/chat/page.tsx" lib/agent.ts lib/api.ts` clean.
- `npx next build` generates all routes.
- Reason through the SOW's hand-test checklist (paperclip→pick JPG→chip+send
  enables; oversize→toast; non-image→toast; drag; paste; dismiss; send shows
  image inline; reload shows `[image attached] …`).

---

## Repo: prog-strength-docs

### Task DOCS1 — Commit plan + mark SOW shipped

- [ ] This plan file is committed on `feat/photo-meal-logging`.
- [ ] Edit `sows/photo-meal-logging.md`: frontmatter `status: shipped`; body
  header `**Status**: Shipped`; body header `**Last updated**: 2026-06-06`.
- [ ] Commit: `docs: mark photo-meal-logging as shipped`.
- [ ] The docs PR body notes that merging it marks the SOW shipped — the owner's
  one-action completion signal.

---

## Out of scope (from the SOW — do not implement)

Mobile / camera capture; persisting image bytes (agent, API, or S3); multiple
images per turn; a separate "Log from photo" page; OCR fallback; cost / rate
limiting; confidence scores; in-composer image editing; any MCP change; any
application-DB schema change (`chat_messages.content` stays `TEXT`).
