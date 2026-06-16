---
type: sow
status: draft
repos:
  - prog-strength-web
  - prog-strength-docs
---

# Chat & App Shell Redesign — Soft-Modern-Messenger, App-Wide

**Status**: Draft · **Last updated**: 2026-06-16

## Introduction

The AI chat (`/chat`) is the front door of Prog Strength, and the app shell — the left sidebar and frame — wraps every page. Both read dated next to modern AI products: a flat dark sidebar of muted rows, plain message pills, a utilitarian composer, and session history hidden behind a right-side drawer. The Design Exploration `dx/chat-and-app-shell.md` (PR Prog-Strength/prog-strength-web#58) put five directions side by side, and the owner selected **`soft-modern-messenger`**: a polished, approachable slate-dark messenger with refined rounded bubbles, soft depth, Nunito rounded type, and a single friendly violet accent. At the selection gate two scope calls were made: the redesign covers the **whole app shell** (the shared sidebar, not just the chat page), and it **adopts the variant's violet accent** (`#8b7cf6`) as the app's accent, replacing today's blue.

Those two calls make this more than a one-surface redesign. Restyling the shared shell and changing the app accent means this SOW **establishes the app's foundational visual language** — the slate neutral ramp, the violet accent, Nunito typography, the rounded-card radii and soft-shadow depth — that every page will inherit. In other words, this is the first real increment of the design system the platform always intended to grow (the DS work type from `sows/design-exploration-work-type.md`), arriving out of necessity rather than as a separate pass. This SOW introduces those decisions as **named, reusable design tokens**, not hard-coded values, so the rest of the app — and the other in-flight redesigns — can conform to one source of truth.

Scope is deliberately bounded so this stays shippable: it delivers the **foundation tokens**, the **restyled shared shell**, and a **fully realized chat surface**. Every other page *inherits* the new shell and tokens (and keeps working), but its page-specific content is not redesigned here — that's each page's own SOW. The chat surface itself gets the full treatment: a three-column messenger layout (nav rail · a persistent conversation-list pane that retires the History drawer · the chat thread), rounded user/assistant bubbles, the model-attribution pills, the tool-call chips, the macro card, the meal-photo attachment, and the empty / typing / error states — all preserving the existing chat behavior (streaming, voice, image attach, model routing, daily-cap, telemetry).

## Proposed Solution

A "work type" decision at the gate turned this into three layers, built bottom-up:

1. **Foundation tokens (app-wide).** Introduce the soft-modern-messenger system as named tokens in `prog-strength-web`'s styling layer: a slate neutral ramp (`#15171c` base, `#1e2128` panels/rails, `#272b33` raised, hairline white-alpha borders), the **violet accent** (`#8b7cf6` / `#7765ec`, plus soft/line tints) replacing the current blue, **Nunito** as the type family with the variant's weight/size steps, and rounded radii + soft shadows. These become the single source of truth; components reference tokens, never raw hex.

2. **Shared shell (app-wide).** Restyle the global sidebar (`components/sidebar.tsx`) and the `(app)` layout to the variant's rail: the brand mark in a rounded violet-tinted badge with a "Prog Strength / AI Coach" lockup, the 12 nav destinations as rounded pills (active = violet-soft fill + violet glyph), and the user card pinned at the bottom. All 12 destinations and the IA stay exactly as they are — this is visual only. Because the shell is shared, every page picks this up immediately.

3. **Chat surface (the focus).** Rebuild the chat page's presentation as a three-column messenger — the shared rail, a **persistent conversation-list pane** ("Chats": searchable session list, new-chat affordance, active-session highlight) that **replaces the right-side History drawer**, and the thread. The thread renders rounded **user bubbles** (violet) and **assistant bubbles** (raised slate) carrying the existing rich content: **model pills** ("via Sonnet 4.6"), **tool-call chips** with state, the **macro card** (the nutrition table re-rendered as a styled card with P/F/C tinted chips and a total), **meal-photo** attachments, and **typing** and **error** states, plus a polished **empty state** and **composer** (mic / attach / input / send). All existing chat logic is preserved; only presentation changes.

The accent flip (blue → violet) happens at the token layer, so it propagates everywhere the old blue token was used — including other pages' incidental accents — in one place.

## Goals and Non-Goals

### Goals

- **App-wide foundation tokens** for the soft-modern-messenger system: slate neutral ramp, **violet accent** (replacing blue), Nunito type scale, rounded radii, soft shadows — named and reusable, referenced everywhere instead of hard-coded.
- **Restyle the shared shell** (`components/sidebar.tsx` + `(app)` layout) to the variant's rail: brand lockup, rounded nav pills with violet-active state, user card. All 12 destinations and IA unchanged.
- **Rebuild the chat surface** as the three-column messenger: nav rail · persistent conversation-list pane · thread.
- **Promote session history** from the right-side `ChatHistoryDrawer` to the **persistent conversation-list pane** (searchable, new-chat, active highlight), preserving resume/delete behavior.
- **Preserve and restyle the rich chat content**: rounded user/assistant bubbles, **model-attribution pills**, **tool-call chips with state**, the **macro card** with P/F/C tinted chips + total, **meal-photo** attachments, and **typing** + **error** + **empty** states.
- **Preserve all chat behavior**: streaming, voice (mic input + Voice on/off TTS), image attach (5 MB), per-turn model routing, daily-usage cap, telemetry — unchanged logic.
- **Tokenize the accent flip** so blue → violet happens once at the token layer and propagates app-wide.
- Update the calendar-view redesign SOW to conform to these tokens (see Interactions), and the `prog-strength-docs` record.
- Keep CI's gate green (`lint`, `format`, `typecheck`, `test`, `build`), updating chat/sidebar tests to the new structure.

### Non-Goals

- **Redesigning other pages' content.** Calendar, Timeline, Nutrition, Progress, etc. *inherit* the new shell + tokens and keep working, but their page bodies are not redesigned here — each is its own SOW. Some pages will look transitional until then; acceptable.
- **A separate, exhaustive design-system document.** This SOW lays the *token foundation* the DS will formalize; authoring a full `design-system.md` + the DS work-type machinery remains a follow-up (it now has real tokens to codify).
- **Information-architecture changes.** No change to which nav destinations exist, their grouping, or routing. Visual only.
- **Any change to chat behavior, the agent, tools, model routing, or data.** Presentation only — a behavior change is a regression, not a goal.
- **Promoting the DX mockup code.** The variant (`app/design-explore/chat-and-app-shell/_variants/soft-modern-messenger.tsx`, on simplified static fixtures) is the **visual spec, not code to copy** — reimplement against the real chat page, `Message`/`ToolCall` model, and `components/sidebar.tsx`. The `design-explore` route is untouched and never ships.
- **A light theme.** The system is slate-dark, as chosen.

## Implementation Details

All paths in `prog-strength-web`. Lifecycle: tokens → shell → chat surface → tests.

### 1. Foundation tokens

Add the soft-modern-messenger system to the app's token layer (CSS variables / Tailwind theme — wherever `--muted` and the current accent live today). Define and name:

- **Neutrals (slate ramp)** — base `#15171c`, panel/rail `#1e2128`, raised `#272b33`, raised-2 `#2e333c`; text `#eef0f4`, muted `#9aa1ad`, faint `#6b7280`; hairline borders `rgba(255,255,255,0.07)` / `0.10`.
- **Accent (violet, replaces blue)** — `#8b7cf6`, dark `#7765ec`, soft `rgba(139,124,246,0.14)`, line `rgba(139,124,246,0.30)`. **Repoint the existing blue accent token to violet** so every current consumer (user bubbles, Send, active nav, links, focus rings) flips in one change.
- **Macro tints** — protein green `#34d399`, fat amber `#fbbf24`, carb blue `#60a5fa` (each with a ~13%-alpha bg) for the macro card chips.
- **Type** — Nunito as the family (via `next/font`), with the variant's size steps (≈11/13/14/15/18/22) and weights (600/700/800).
- **Radii + shadow** — rounded-2xl/3xl card radii, full-pill radius, and the soft depth shadows the variant uses.

Components must reference these tokens, not raw hex — this is what lets other pages and the calendar SOW conform.

### 2. Shared shell (`components/sidebar.tsx` + `app/(app)/layout.tsx`)

Restyle the global sidebar to the variant's rail: a rounded violet-tinted brand badge with the "Prog Strength / AI Coach" lockup; the **same 12 nav destinations** as rounded pills (active = violet-soft fill + violet-line border + violet glyph; idle = muted with hover lift); the user card pinned at the bottom (initials avatar, name, plan/status line). Keep the collapse affordance and every existing route/destination. The variant's "AI Coach" subtitle and "Free plan" line should map to **real values** (e.g. the user's actual plan/usage state) or be dropped if no real source exists — do not ship invented status. Because this component is shared, the restyle lands on every page at once.

### 3. Chat surface (`app/(app)/chat/page.tsx` + `components/chat/*`)

Rebuild the chat presentation as the three-column messenger, preserving the page's existing logic (send, streaming, voice, image attach, model routing, daily-cap, telemetry — all unchanged):

- **Conversation-list pane** — a new persistent column ("Chats"): a search pill, a new-chat button, and the session list (each: brand-mark avatar, title, relative time, message count, active-session violet highlight). This **replaces `ChatHistoryDrawer`** as the home for session history; preserve resume-on-click and delete. (The drawer component is retired or repurposed into the pane.)
- **Header** — assistant avatar with a presence dot, "Chat", and the model/status line; keep Voice on/off and New chat.
- **Thread** — rounded **user bubbles** (violet) and **assistant bubbles** (raised slate). Assistant bubbles carry: a **model pill** (`modelLabel`: "via Sonnet 4.6"), **tool-call chips** with status glyph (running/ok/error), the **macro card** (the markdown nutrition table → a styled card: per-item rows, a **Total** row, P/F/C tinted chips), **bold inline stats**, and coach-tone text. User bubbles may carry a **meal-photo** attachment. Include the **typing** ("thinking") and **error** turn states.
- **Empty state** — the polished first-run state (brand mark, "Ask about your training.", example prompts) in the new language.
- **Composer** — rounded input with mic (voice), attach (image), and a violet Send; keep the disabled-in-flight and daily-cap states.

Build the presentational pieces as focused components (bubbles, tool chip, model pill, macro card, meal photo, conversation-list, composer) so the 1600-line page's logic stays intact while its render tree is replaced.

### Interactions with other in-flight work

- **Calendar-view redesign (`sows/calendar-view-redesign.md`, draft).** That SOW predates this accent decision and specifies a warm **clay** accent + warm tonal hues. Now that **violet is the app accent and slate the base**, the calendar SOW should **conform to these tokens** — keep its *structure and coaching feel* (rounded per-week panels, streak strip) but draw its accent and surfaces from the shared tokens (violet on slate), and re-tone its activity hues to sit on the slate ramp. This SOW should land (or at least merge its token layer) first; update the calendar SOW's palette section to reference the tokens rather than introduce clay. Called out so the two don't ship conflicting accents.
- **Timeline DX (`dx/timeline.md`).** Whatever wins there should likewise conform to these tokens at implementation.

### Tests

The shell and chat DOM change substantially, so tests are updated, not just passed through:

- `components/sidebar` tests — assert the restyled rail renders all 12 destinations, the active/idle states, and the user card; keep navigation behavior.
- `app/(app)/chat/page.test.tsx` and `components/chat/*` tests — assert the conversation-list pane renders sessions (and resume/delete still work), the bubbles/model-pill/tool-chip/macro-card render for a known fixture, and **all existing behavior tests hold** (send, streaming, voice toggle, image attach, cap). Retire/replace `chat-history-drawer` tests for the pane.
- Keep CI green locally before pushing: `npm run lint`, `format:check`, `typecheck`, `test`, `build`.

## Rollout

Web-only, no API or migration, no coordination window — but a broad visual blast radius (the shell + tokens touch every page).

1. **`prog-strength-web`** — foundation tokens (incl. the blue→violet repoint and Nunito), the shared shell restyle, the chat-surface rebuild, and test updates. One PR. The Vercel preview is the place to sanity-check **every** page still renders coherently under the new shell/tokens, not just chat — spot-check Calendar, Timeline, Nutrition, Progress for anything that hard-coded the old blue or assumed the old sidebar.
2. **`prog-strength-docs`** — flip `dx/chat-and-app-shell.md` to `status: selected` (idiom `soft-modern-messenger`), update `sows/calendar-view-redesign.md`'s palette section to reference the new tokens, and mark this SOW shipped on merge.

### Verification after rollout

- `/chat`: three-column messenger — restyled rail, the persistent Chats pane (search, new chat, active highlight, resume + delete), rounded violet user bubbles and slate assistant bubbles with model pills, tool-call chips, the macro card (P/F/C chips + total), meal-photo attachment, and the typing/error/empty states. All chat behavior intact: send, streaming, voice on/off, image attach, model routing, daily cap.
- The shared sidebar shows the new rail on **every** page, all 12 destinations reachable, violet active state.
- Spot-check other pages: they inherit the shell + violet accent and still function; nothing relied on the old blue token or the drawer.

## Open Questions

- **"AI Coach" / "Free plan" labels** — the variant invents these. Map to real values (plan/usage state) or drop; don't ship invented status. Settle in implementation.
- **Conversation-list pane vs. responsive collapse** — a persistent third column is great on desktop; on narrow viewports it likely collapses back to a drawer/sheet. Confirm the responsive behavior (the variant is desktop-width); the pane shouldn't crowd the thread on smaller screens.
- **Accent repoint blast radius** — repointing the blue token to violet flips every current consumer. Audit for any page that hard-codes blue outside the token (those won't flip and will look stale); fold fixes in or note them.
- **Design-system formalization** — this SOW *creates* the tokens; codifying them into a `design-system.md` + the DS work-type machinery (so future DX/SOWs reference one document) is the natural next step now that real tokens exist. Out of scope here, but this is the moment the DS becomes worth standing up.
