---
status: ready_for_implementation
depends_on:
  - sows/custom-meals.md
repos:
  - prog-strength-web
  - prog-strength-agent
  - prog-strength-api
  - prog-strength-docs
---

# Photo Meal Logging

**Status**: Ready for Implementation · **Last updated**: 2026-06-05

## Introduction

A user who just had dinner at Chipotle takes a photo of the receipt — chicken bowl, brown rice, black beans, salsa, chips, side of guac — and wants to log it. The custom-meals SOW (`sows/custom-meals.md`) gives them a path: open the chat, type "I had a Chipotle chicken bowl with chips and guac," and the agent estimates macros and logs a custom meal after confirmation. That works, but the user typed for 30 seconds when they could have snapped a picture and said "log this." For someone tracking macros every meal — the user this app is built for — that 30 seconds compounds across three meals a day for weeks.

Claude already handles images natively. The agent's `ChatMessage.content` is already typed as `str | list[dict]`, which is exactly the multimodal content-block shape the Anthropic SDK expects, so the agent service has zero type widening to do. The whole feature comes down to plumbing: a paperclip in the web composer, a multimodal payload in the user turn, a routing override on the agent so vision turns hit a vision-capable model instead of Haiku, and one paragraph of prompt explaining what to do with the image. The downstream tool the agent calls is `log_custom_meal` from the prior SOW — this feature is a new on-ramp to the same tool, not a new entity.

For v0 we deliberately do not store image bytes anywhere. Receipts and menus contain PII (names, addresses, payment method tails, location) and the privacy story is much cleaner if the bytes pass through the agent into Claude and immediately get discarded. On session reopen, the user's image turn renders as `[image attached] <whatever text they typed>` — a plain-text placeholder in the existing `chat_messages.content` column. The image is genuinely gone after the request completes; we can revisit thumbnails in a follow-up if users ask for them.

The same natural-language correction loop the custom-meals SOW already locks in carries over: the agent proposes macros, the user can adjust ("bump protein to 55, it was a bigger bowl"), and the agent re-proposes or logs based on the user's wording. The model only fires `log_custom_meal` after explicit user confirmation. Multi-item receipts produce multiple `log_custom_meal` calls — one per line item — either batched after a single confirmation or split across turns; the model judges based on the user's flow.

## Proposed Solution

The web chat composer (`app/(app)/chat/page.tsx`) gains an attach affordance: a paperclip icon button slotted between the existing mic and the textarea. Clicking it opens a file picker scoped to `image/jpeg`, `image/png`, and `image/webp`. The textarea area additionally accepts drag-and-drop and paste-from-clipboard, both routing through the same `setImage` callback. The selected image renders as a small chip above the textarea — a 48×48 thumbnail (drawn from a blob URL), the filename, and a `×` to dismiss. Size cap is 5 MB and format cap is the three above; both validated client-side with a toast on rejection. The send button enables on either text or image, where today it requires text.

On send, the user message's `content` field switches from a plain string to a typed content-block list: `[{type: "image", source: {type: "base64", media_type, data}}, {type: "text", text: <user text or " ">}]`. (Claude requires a text block alongside the image; an empty user message becomes a single space rather than no text block, so the model always has something to read.) The page's existing `send()` function plumbs this through to the agent's `POST /chat` exactly the way it ships a plain string today — the type widening is a one-liner on the TypeScript side.

For persistence, the web does *not* ship the multimodal content array to the API. It ships the placeholder string `[image attached] <text>` via the existing `appendChatTurn` call. The API treats this as the user's textual content and stores it as today; on reopen, the user message renders that string verbatim in the scrollback. The image bytes only ever exist in the browser's memory and in the body of the agent request — never on disk.

The agent (`prog-strength-agent`) needs three changes. First, `_route_and_stream` checks for any user message whose `content` is a list containing an `image`-typed block and short-circuits the intent classifier when present. Vision turns go directly to the vision-capable model the project standardizes on (currently Sonnet 4.6); they never get routed to Haiku, which would silently degrade. Second, the prompt grows a paragraph teaching the model what to do with the image: identify the line items if it's a receipt, identify the visible portion if it's a plate of food, ask for clarification if it's a menu or ambiguous, propose macros, ask confirmation, and only call `log_custom_meal` after an explicit user yes. Third, telemetry records a `had_image` boolean per turn so usage is measurable.

The API (`prog-strength-api`) has no schema change. `chat_messages.content` stays a `TEXT` column; the web ships the placeholder string already formed.

No MCP changes. No mobile in this SOW.

## Goals and Non-Goals

### Goals

- Add an attach affordance to the web chat composer in `app/(app)/chat/page.tsx`: a paperclip icon button between the mic and the textarea. Clicking opens a file picker (`<input type="file" accept="image/jpeg,image/png,image/webp">`).
- Add drag-and-drop support on the textarea area (entire chat footer is a drop target) routing through the same selection callback.
- Add paste-from-clipboard support on the textarea (`onPaste` handler reading `image/*` MIME items).
- Render the selected image as a small chip above the textarea: a 48×48 thumbnail (blob URL), the filename, and a `×` button. Selecting a new image replaces the existing chip; the previous blob URL is revoked.
- Validate client-side: size ≤ 5 MB, MIME type in `{image/jpeg, image/png, image/webp}`. Toast on rejection with the specific reason ("Image must be under 5 MB" / "Use JPG, PNG, or WebP").
- Enable the Send button when there is text OR an image attached. Disable when both are empty.
- On send: build the user message's `content` as `[{type: "image", source: {type: "base64", media_type, data}}, {type: "text", text: <user text or " ">}]` and POST to `${agentUrl}/chat` exactly as today, preserving the streaming response path.
- The user message renders in the chat scrollback with the image visible (rendered from the in-memory blob URL). The text part renders as today.
- When persisting the user turn via `appendChatTurn`, the web ships the placeholder string `[image attached] <text>` (single space text becomes `[image attached]` with no trailing space). Image bytes do not reach the API.
- On session resume (`getChatSession`), past user turns that originally had an image come back with the `[image attached]` placeholder already in their `content`. Render the string verbatim — no special handling for the marker; the renderer treats it as ordinary text.
- Widen the TypeScript `ChatMessage.content` type on the web client to `string | Array<ContentBlock>` so the in-flight (current-session) user turns can carry the multimodal payload without `any`-casting.
- In the agent's `_route_and_stream`, detect any user message whose `content` is a list containing a block where `type == "image"`. When present, skip intent classification entirely and force routing to the vision-capable model (Sonnet 4.6 today, sourced from the same constant the existing routing uses). Add a unit test covering the detection.
- In the agent's system prompt (`prompt.py`), add a paragraph adjacent to the existing nutrition / chat guidance covering image handling:
  - Receipt: identify line items, estimate macros for each, propose a list inline in the reply, ask the user to confirm.
  - Plate of food: identify the visible portion, estimate macros for what's shown, propose, ask the user to confirm. If a side dish is partially visible, note the uncertainty in the proposal.
  - Menu or ambiguous photo: ask the user what they had if their text didn't say.
  - Always confirm before calling `log_custom_meal`. Multi-item receipts may produce multiple `log_custom_meal` calls in a single assistant turn after one user confirmation, or one per turn — the model judges based on user flow.
  - On user correction in the next turn ("bump protein to 55"), adjust the proposal and re-ask rather than logging eagerly.
- Telemetry: add a `had_image` boolean to the turn instrumentation. Persisted alongside the existing telemetry fields.
- New agent tests (`test_model_harness.py`):
  - A chat with a user message whose content includes an image block routes to Sonnet (no intent classifier call) and forwards the multimodal content to the model unchanged.
  - With a mocked image-of-receipt + "log this" flow, the harness sees no immediate `log_custom_meal` (the model proposes first); after a confirmation turn, the harness sees `log_custom_meal` with the proposed macros.
- The custom-meals SOW must merge before this one; this SOW does not re-spec `log_custom_meal`.

### Non-Goals

- **Mobile.** No camera capture, no photo-library picker, no mobile-specific composer affordances. Catches up in a follow-up SOW.
- **Persisting image bytes.** Not in the agent process, not on the API, not in S3 thumbnails. The placeholder string in `chat_messages.content` is the only record after the request completes. Files a follow-up if users complain that history feels lossy.
- **Multiple images per turn.** v0 is one image per turn. Multi-image (two-page receipt, front + back of label) is a follow-up.
- **A separate "Log from photo" page or modal outside chat.** Chat is the surface. Adding a parallel non-chat flow doubles the prompt-and-confirmation infrastructure for the same end-state.
- **OCR-only fallback.** When Claude vision can't make out the image, the model tells the user to send a clearer photo. No second-pass OCR library.
- **Cost / rate limiting.** Pre-launch volume makes per-user image quotas not worth the engineering yet. Files a follow-up if usage trajectory says otherwise.
- **Confidence scores or "I'm 60% sure this is X".** The proposal is a flat number. The natural-language correction loop is the calibration mechanism.
- **Image editing in the composer.** No crop, no rotate, no brightness. Pick the photo, send.
- **Camera capture directly via the file input's `capture="environment"` attribute.** Only meaningful on mobile, which is deferred.
- **API or DB schema changes.** `chat_messages.content` stays `TEXT`; nothing else is touched.
- **MCP changes.** `log_custom_meal` is the downstream and it's already specced in the custom-meals SOW.
- **Recipe / pantry inference from photos.** The model identifies what was eaten and logs it as a custom meal. Promoting a photo-logged custom meal to the pantry happens through the same "Want me to save…" ask the custom-meals SOW already specs.

## Implementation Details

### Web composer (`app/(app)/chat/page.tsx`)

State additions:

```ts
const [pendingImage, setPendingImage] = useState<{
  blob: Blob;
  previewUrl: string;        // URL.createObjectURL(blob), revoked on replace/dismiss
  filename: string;
  mediaType: "image/jpeg" | "image/png" | "image/webp";
} | null>(null);
const fileInputRef = useRef<HTMLInputElement | null>(null);
```

Helpers (file-local):

```ts
const MAX_IMAGE_BYTES = 5 * 1024 * 1024;
const ALLOWED_TYPES = new Set(["image/jpeg", "image/png", "image/webp"] as const);

function acceptImageFile(file: File, toast: ReturnType<typeof useToast>): {
  blob: Blob; mediaType: "image/jpeg" | "image/png" | "image/webp";
} | null {
  if (!ALLOWED_TYPES.has(file.type as never)) {
    toast.error("Use JPG, PNG, or WebP.");
    return null;
  }
  if (file.size > MAX_IMAGE_BYTES) {
    toast.error("Image must be under 5 MB.");
    return null;
  }
  return { blob: file, mediaType: file.type as never };
}
```

Composer JSX in the footer (placed after the mic button, before the textarea):

```tsx
<button
  type="button"
  onClick={() => fileInputRef.current?.click()}
  disabled={streaming || loading || !sessionId}
  aria-label="Attach image"
  title="Attach image"
  className="flex h-[44px] w-[44px] shrink-0 items-center justify-center rounded-lg border border-[var(--border)] bg-[var(--surface)] text-[var(--muted)] transition hover:text-[var(--foreground)] disabled:opacity-40"
>
  <PaperclipIcon />
</button>
<input
  ref={fileInputRef}
  type="file"
  accept="image/jpeg,image/png,image/webp"
  hidden
  onChange={(e) => {
    const file = e.target.files?.[0];
    if (file) onSelectImage(file);
    e.currentTarget.value = "";
  }}
/>
```

`onSelectImage(file)` validates via `acceptImageFile`, revokes the previous `previewUrl` if any, and sets `pendingImage`. `onDismissImage()` revokes the URL and clears state.

Drag-and-drop: wrap the composer footer in a `onDragOver` / `onDrop` handler that intercepts `dataTransfer.files[0]` and routes it through `onSelectImage`. Suppress default drop behavior on the document so the browser doesn't navigate to the dropped image.

Paste-from-clipboard: textarea `onPaste` handler iterates `e.clipboardData.items`, picks the first `image/*` item, and routes through `onSelectImage`.

Preview chip above the textarea (rendered when `pendingImage != null`):

```tsx
<div className="mx-auto flex max-w-2xl items-center gap-2 px-1 pb-1">
  <div className="inline-flex items-center gap-2 rounded-md border border-[var(--border)] bg-[var(--surface)] px-2 py-1.5">
    <img src={pendingImage.previewUrl} alt="" className="h-10 w-10 rounded object-cover" />
    <span className="text-xs text-[var(--muted)] max-w-[180px] truncate">{pendingImage.filename}</span>
    <button onClick={onDismissImage} aria-label="Remove image" className="text-[var(--muted)] hover:text-[var(--foreground)]">✕</button>
  </div>
</div>
```

Send button gating:

```ts
const sendDisabled = streaming || loading || !sessionId ||
  (input.trim().length === 0 && pendingImage === null);
```

### Web → agent payload

Existing `send()` builds the `messages` array. The current user turn's `content` becomes:

```ts
const userContent = pendingImage
  ? [
      {
        type: "image",
        source: {
          type: "base64",
          media_type: pendingImage.mediaType,
          data: await blobToBase64(pendingImage.blob),
        },
      },
      { type: "text", text: input.trim() || " " },
    ]
  : input.trim();
```

`blobToBase64` is a tiny `FileReader.readAsDataURL` wrapper that strips the `data:…;base64,` prefix and returns the raw base64 payload. The rest of the `send()` pipeline ships `userContent` unchanged — the agent `POST /chat` body already accepts `string | list[dict]` for `content`.

After a successful send (the SSE stream starts), `pendingImage` clears and its blob URL is revoked. The user message rendered in the scrollback is the in-memory `Message` shape with content as the multimodal array — the renderer paints the image from the blob URL and the text inline.

### Web persistence (`appendChatTurn` call)

`appendChatTurn` in `lib/api.ts` currently accepts plain string content. No widening — the web converts before persisting:

```ts
const persistedUserContent = pendingImage
  ? (input.trim() ? `[image attached] ${input.trim()}` : "[image attached]")
  : input.trim();

await appendChatTurn(token, sessionId, {
  user: { content: persistedUserContent },
  assistant: { /* same as today */ },
});
```

The API stores the placeholder string. On `getChatSession`, the user message comes back with that same string in `content`. The renderer treats it as ordinary text — no special-casing.

### Web TypeScript types

`lib/agent.ts` and `lib/api.ts`: the in-flight `ChatMessage` type widens:

```ts
export type ContentBlock =
  | { type: "text"; text: string }
  | {
      type: "image";
      source: { type: "base64"; media_type: "image/jpeg" | "image/png" | "image/webp"; data: string };
    };

export type ChatMessage = {
  role: "user" | "assistant";
  content: string | ContentBlock[];
};
```

`appendChatTurn` keeps its plain-string `content` shape — that's the persisted form.

### Web composer: `PaperclipIcon`

Local component, 16×16 viewBox, matching the existing icon vocabulary (1.75 stroke, rounded join). Standard paperclip path — a copy of any of the small SVGs already in the sidebar / log view is fine.

### Agent: routing change (`_route_and_stream`)

In `prog-strength-agent/src/prog_strength_agent/server.py` (or wherever `_route_and_stream` lives), add a small helper:

```python
def _has_image(messages: list[dict[str, Any]]) -> bool:
    """True if any user message carries an image content block."""
    for m in messages:
        if m.get("role") != "user":
            continue
        content = m.get("content")
        if isinstance(content, list):
            for block in content:
                if isinstance(block, dict) and block.get("type") == "image":
                    return True
    return False
```

Inside `_route_and_stream`, before the intent classifier call:

```python
if _has_image(messages):
    # Vision turns force the vision-capable model and skip the
    # classifier. Skipping saves one API call and guarantees Haiku
    # never gets picked for a turn that needs to see.
    chosen_model = VISION_MODEL  # Sonnet 4.6 today.
    # ... emit model_chosen SSE event with chosen_model.
    # ... continue with the existing stream pipeline.
else:
    # Existing intent-classified routing.
    ...
```

`VISION_MODEL` is a new module-level constant defaulting to the same Sonnet model the existing routing reaches for. If `routing.py` already exports the Sonnet model id, import that.

### Agent: prompt update (`prompt.py`)

Add the following paragraph adjacent to the existing nutrition / logging guidance:

```text
Logging meals from a photo. When the user attaches an image:

- Receipt: list out the items you can read, estimate macros per item,
  propose them as a list in your reply, and ask the user to confirm.
  Multi-item receipts may produce multiple log_custom_meal calls in
  a single reply after one user "yes" — only call the tool after the
  user confirms.

- Plate of food: identify what's visible, estimate macros for the
  portion shown (not the menu portion — what's actually on the
  plate). If a side dish is partially obscured, say so in your
  proposal.

- Menu or other ambiguous photo: ask the user what they actually
  had. Don't guess at meal choice from a menu alone.

If the user corrects your proposal ("bump protein to 55, it was a
bigger bowl"), revise the numbers and re-ask, don't fire log_custom_meal
on the corrected values until the user confirms.

Never call log_custom_meal eagerly on the first turn that carries an
image. Always propose first, then log on the user's "yes."
```

### Agent: telemetry

`TurnInstrumentation` (or the equivalent telemetry struct) gains a `had_image: bool` field set when `_has_image(messages)` is true. Flows through to the existing telemetry POST.

### Agent: tests (`test_model_harness.py`)

- `test_vision_turn_skips_intent_classifier_and_routes_to_sonnet`: build a `messages` list with one user message whose content is `[{"type": "image", "source": {...}}, {"type": "text", "text": "log this"}]`, run `_route_and_stream` (or the relevant routing function), assert no intent-classifier call and that `chosen_model == VISION_MODEL`.
- `test_vision_turn_proposes_before_logging`: with a mocked Anthropic response that returns text only (no tool call) on the first turn, assert that no MCP tool is invoked. Then with a follow-up user turn "yes" and a mocked response that returns a `log_custom_meal` tool_use, assert the tool fires with the proposed macros.
- `test_text_only_turn_unchanged`: a text-only `messages` list still routes through the existing intent classifier path. Regression guard.

### API

No changes. `chat_messages.content` stays `TEXT`. The API tests don't need updates — the placeholder string is just text.

### Telemetry consumer (Go API)

`internal/telemetry/` already accepts a JSON body with optional fields. The new `had_image` boolean is additive; if older clients omit it, the field defaults to `false` server-side. The handler does not need a schema change unless the telemetry table has a strict column list — verify the migration before implementation.

### Cost monitoring (no enforcement)

The telemetry boolean enables future analysis ("what % of turns carry an image?"). No quotas, no rate limits. Revisit when an actionable signal exists.

### Tests

**Web**: no test runner. Verification:

- `npx tsc --noEmit` — clean
- `npx eslint app/\(app\)/chat/page.tsx lib/agent.ts lib/api.ts` — clean
- `npx next build` — all routes generate
- Hand-test checklist:
  - Paperclip → file picker → pick a JPG → preview chip appears, send button enables (with empty text).
  - Paperclip → pick a 6 MB image → error toast, no chip.
  - Paperclip → pick a `.svg` (or any non-image) → error toast.
  - Drag a JPG onto the composer → preview chip appears.
  - Paste a screenshot from clipboard → preview chip appears.
  - With chip present, click `×` → chip removed.
  - With chip present + text "log this", click Send → user message in scrollback shows the image inline + the text below.
  - Reload the page or navigate away + back to the same session via History → that turn renders with `[image attached] log this` as the user message (no image visible).
  - Agent reply proposes macros and asks confirmation; reply with "yes" → log shows up on `/nutrition`.

**Agent**: see § Agent: tests above.

**API**: no behavior change to test.

### Rollout

1. Merge `sows/custom-meals.md` first (this SOW's hard dependency).
2. Merge `prog-strength-agent` first (the routing + prompt change is backwards-compatible — text-only turns route exactly as today, so the agent can ship before the web client).
3. Merge `prog-strength-web` next.
4. Deploy both.
5. Hand-test in the deployed environment with a real account: snap a receipt photo, send through chat, confirm the proposal, verify the custom meal lands on `/nutrition`.

Each repo's PR is independently revertable. The API has no changes to roll back.
