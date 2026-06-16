---
type: dx
status: selected
selected_idiom: soft-modern-messenger
surface: chat-and-app-shell
idioms:
  - minimal-canvas
  - warm-editorial-thread
  - command-workbench
  - coach-branded
  - soft-modern-messenger
references:
  - ChatGPT
  - Claude
  - Linear
  - Raycast
scope: greenfield
variant_count: 5
repos:
  - prog-strength-web
  - prog-strength-docs
---

# DX: Chat & App Shell

**Status**: Selected · **Last updated**: 2026-06-16

> **Selected: `soft-modern-messenger`.** Picked at the selection gate from
> [prog-strength-web#58](https://github.com/Prog-Strength/prog-strength-web/pull/58)
> (the `[DX — DO NOT MERGE]` draft PR). At the gate two scope calls widened the
> pick: the redesign covers the **whole app shell** (the shared sidebar, not
> just the chat page), and it **adopts the variant's violet accent**
> (`#8b7cf6`) as the app accent, replacing the old blue. Implementation is
> carried by the downstream SOW
> [`sows/chat-and-app-shell-redesign.md`](../sows/chat-and-app-shell-redesign.md),
> which establishes the soft-modern-messenger system as app-wide design tokens,
> restyles the shared shell, and fully realizes the chat surface. The DX branch
> and PR #58 are disposable and can be closed/deleted now that the decision
> lives in the SOW.

## Context

The AI chat (`/chat`) is the front door of Prog Strength — it's the first nav item and the surface where a user actually *talks* to the agent: logs meals in natural language, asks about their training, resumes past conversations. The agent itself is good. The interface around it is not. Next to modern AI chat apps it looks dated and a little hollow: an empty near-black canvas with one boxed prompt card, plain pill message bubbles, a utilitarian composer, and the whole thing framed by a wide 12-item sidebar that competes with the conversation for attention. ChatGPT and Claude — the apps this user benchmarks against — feel calm, modern, and content-first; Prog Strength's chat feels like a generic admin panel that happens to have a text box.

This DX explores how the **chat experience and the app shell that frames it** should look, together — because you can't modernize the conversation without also addressing the rail, the empty state, the session history, and how the whole thing breathes. It's a visual/layout exploration only: the existing nav destinations (Timeline, Calendar, Nutrition, etc.) and the product's information architecture stay as-is; what's on the table is how the shell *looks and lays out*, not which sections exist. The goal is to find a direction that feels like a modern, premium AI product — and unmistakably like Prog Strength — before committing engineering effort to building it.

## The surface

The chat route (`app/(app)/chat/page.tsx`) inside the app shell (`app/(app)/layout.tsx` + `components/sidebar.tsx`). A variant must render the whole thing — rail, conversation, composer, history — because the point is how they cohere. The parts:

- **App shell / left rail** (`components/sidebar.tsx`) — the Prog Strength "P" wordmark + a collapse chevron at top; a 12-item nav (Chat, Timeline, Search, Requests, Activities, Exercises, Calendar, Nutrition, Bodyweight, Progress, Personal Records, Settings); the signed-in user (avatar + "Jimmy Wallace") pinned at the bottom. Today it's a wide, always-expanded, flat dark column. Variants may rethink its treatment (icon rail vs labeled, collapsed by default, how the user/footer reads) — but **keep all 12 destinations and the brand identity**; this is visual, not an IA cull.
- **Chat header** — "Chat" title; a **Voice on/off** toggle (TTS playback of assistant turns); **+ New chat**; **History**.
- **Empty state** — a centered card: the "P" mark, "Ask about your training.", and an example prompt ("Try *what chest exercises are in the catalog?* or paste a workout log…"). ChatGPT's "Where should we begin?" is the calibre to beat.
- **Conversation** — alternating turns:
  - **User turns**: right-aligned blue bubbles; may carry an **attached image** (photo-meal logging).
  - **Assistant turns**: left-aligned, and *richer than plain text* — this is the part generic chat UIs get wrong. An assistant turn carries:
    - a **model-attribution badge** — "via Sonnet 4.6" / "via Haiku 4.5" (the agent routes across Claude models per turn);
    - **tool-call chips** with state — e.g. `✓ Log Consumption` (states: running / ok / error), often several per turn;
    - **rich markdown** — notably **macro tables** (columns: Item · Cal · P · F · C, with a bold **Total** row), bold inline stats ("**640 cal / 38g P / 14g F / 95g C**"), and a coach-tone closing line with the occasional emoji.
  - **Streaming / loading**: the assistant turn builds incrementally; there's a "thinking" state before text arrives.
- **Composer** — a **mic** (voice input), an **attach** (image, 5 MB cap), a text field ("Message Prog Strength…"), and **Send**. Disabled while a turn is in flight; shows a daily-AI-allowance state when the user hits their cap.
- **Session history** (`components/chat/chat-history-drawer.tsx`) — currently a **right-side drawer** listing past sessions (title, message count, relative time, delete), e.g. "Lunch Macros Logged · 4 messages · 22 hours ago". ChatGPT/Claude promote this to a **persistent left pane**; whether history stays a drawer or becomes a pane is fair game for a variant to explore.

**The data shape** (for realistic fixtures — the message model in `page.tsx`):

```ts
type ToolCall = { name: string; state: "running" | "ok" | "error" };
type Message = {
  role: "user" | "assistant";
  content: string;            // markdown for assistant turns (tables, bold, emoji)
  tools?: ToolCall[];         // assistant only — e.g. [{name:"Log Consumption", state:"ok"}]
  model?: string;             // assistant only — "claude-sonnet-4-6" → "via Sonnet 4.6"
  image?: string;             // user only — an attached meal photo
};
```

**Representative fixture** (use the nutrition-logging conversation from the screenshots — it exercises every rich element):

1. User: "For lunch, I just had a turkey burger and bibigo white rice"
2. Assistant (*via Sonnet 4.6*, tools: `✓ Log Consumption ✓ Log Consumption`): "Logged! Here's your lunch:" + a macro **table** (Columbus Turkey Burger 240/30/13/2; Bibigo Sticky White Rice 310/6/1/71; **Total 550/36/14/73**) + "Solid macro split — good protein hit with the carbs to fuel the rest of your day. 💪"
3. User: "I also had some cherries with lunch too"
4. Assistant (*via Haiku 4.5*, tools: `✓ Log Consumption`): "Got it — added the cherries (20 cherries). Your lunch total is now **640 cal / 38g P / 14g F / 95g C**. Nice carb load for the day, bro."

Also render the **empty state** and the **history list** (use the real-looking titles from the screenshots: "Weight Check And Progress Tracking", "Quick Snack Calorie Logging", "Recovery Week Shoulders And Arms"…). These are mockups — **static fixtures that look real are preferred**; do not wire to the live agent.

**States the variants must handle** (where lazy chat redesigns fall apart): the empty state; a populated multi-turn conversation; an assistant turn with **multiple tool-call chips + a table** (the distinctive Prog Strength element — a beautiful chat that mangles the macro table or drops the model badge fails); a **streaming/thinking** state; an **error** turn; a **user image attachment**; **voice-on**; **history open**; and the **collapsed rail**.

## Idioms

Five genuinely distinct directions. All stay **dark** (every reference and the current app are dark; `scope: greenfield` here means rethinking the shell's structure and chrome, not the base theme). Each must diverge along **type scale**, **color logic**, and **spacing rhythm** — not just accent color. The fixed points are the Prog Strength identity (the "P" mark, the brand) and that the rich assistant elements above stay legible.

- **minimal-canvas** — ChatGPT's content-first calm. Vast negative space; the empty state is a centered hero ("Ask about your training.") above a **single prominent rounded input pill** with inline action chips; the left rail collapses to **icons only**. Assistant turns are **full-width text blocks, not bubbles**; near-monochrome with one restrained accent. Type: clean system sans, modest scale, hierarchy by weight. Spacing: sparse and airy — the conversation is the only thing on screen.

- **warm-editorial-thread** — Claude's warmth and readability, on dark. A warm-undertoned charcoal (not pure black); a **comfortable centered reading column**; a humanist or lightly-serifed display face for headings with **generous leading**; assistant turns read as flowing prose, tool results as soft paper-like cards. Type: editorial, mid-to-large, calm. Color: warm neutrals + a single muted accent. Spacing: roomy, reading-first — the opposite of dense.

- **command-workbench** — Linear / Raycast / IDE. **Dense and multi-pane**: a persistent labeled nav *and* an always-visible session-history pane (history is promoted out of the drawer) flanking the thread; **monospace metadata** (model badges and tool calls as tight structured cards with status glyphs `▶ running / ✓ ok / ✗ error`); keyboard-first affordances. Type: small utilitarian sans + mono for metadata. Color: a single sharp accent on neutral graphite. Spacing: tight, gridded, maximal info density — the deliberate opposite of minimal-canvas.

- **coach-branded** — Prog Strength's own identity, turned up. The brand accent as a **primary, confident color**; the agent framed as a **coach with personality** — bolder display type, energetic message styling, strong user/assistant distinction, motivational microcopy treated as a first-class element (not an afterthought line). Type: bold, athletic, confident scale. Color: brand-forward — the accent carries the experience, not a timid blue bubble. Spacing: punchy and deliberate. This is the one that looks like *no other* AI app.

- **soft-modern-messenger** — best-in-class polished modern chat. Refined **rounded bubbles** with soft depth/shadow and clear speaker distinction; approachable mid-contrast neutrals + one friendly accent; comfortable rounded spacing; tool-call cards and tables styled as tidy rounded panels. Type: rounded, friendly, comfortable. The "modern chat done genuinely well" baseline — familiar, warm, and clean without being any single product's clone.

## References

- **ChatGPT** — take its **content-first negative space**, the centered empty-state hero, the single prominent input pill with inline action chips, and the icon-minimal rail. Drives `minimal-canvas`.
- **Claude** — take its **calm warmth and reading-column readability**, the comfortable line length and leading, and the unobtrusive chrome. Drives `warm-editorial-thread`.
- **Linear** — take its **crisp dense typography, multi-pane structure, and single-accent restraint**. Drives `command-workbench`.
- **Raycast** — take its **keyboard-first, command-surface density** and tight metadata styling. Reinforces `command-workbench`.
- (`coach-branded` draws on **Prog Strength's own** athletic identity rather than an external product — that's the point of it.)

## Selection criteria

A note-to-self for the pick, not a rubric the worker optimizes against. When I compare these I'm weighing:

- **Does it feel like a modern, premium AI product** — in the same league as ChatGPT/Claude — rather than an admin panel with a text box? That's the whole reason for this exploration.
- **Do the Prog Strength-specific rich elements still shine** — the model-attribution badge, the tool-call chips with state, and especially the **macro table**? A gorgeous generic chat that flattens these into plain text is a downgrade, not an upgrade.
- **Does it still read as Prog Strength**, not a reskin of someone else's app? The branded direction tests how far toward "ours" I want to go vs. how far toward familiar-and-safe.
- **Does the shell get out of the way?** The current 12-item rail dominates; I want the conversation to be the center of gravity while keeping every destination reachable.
- **Does session history feel right** — is the drawer fine, or does a persistent pane (ChatGPT/Claude-style) make resuming conversations feel more natural?
- **Does it hold up populated and empty** — a long tool-heavy conversation *and* the first-run blank slate both need to look intentional.

---

> **Lifecycle.** `status:` is editorial — the owner is the dispatch gate. It moves `draft` → `exploring` (worker running) → `awaiting_selection` (draft PR open, owner deciding) → `selected` / `abandoned`.
>
> **Handoff.** This DX ends at a *chosen variant*, not merged code. I open the draft `[DX — DO NOT MERGE]` PR, compare the variants on the preview deploy, pick one, tick its box, set `status: selected` (noting the winning idiom), and **close the PR — never merge it.** Then I open a SOW: *"implement chat-and-app-shell per the `<chosen-idiom>` variant from `dx/chat-and-app-shell`, production-quality, conforming to the design system."*
