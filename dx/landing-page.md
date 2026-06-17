---
type: dx
status: awaiting_selection
surface: landing-page
idioms:
  - centered-hero-spotlight
  - split-screen-product
  - athletic-display-bold
  - feature-showcase-stack
  - conversation-led
references:
  - Linear
  - Vercel
  - Whoop
  - Notion
  - ChatGPT
scope: in-system
variant_count: 5
repos:
  - prog-strength-web
  - prog-strength-docs
---

# DX: Landing Page

**Status**: Awaiting selection · **Last updated**: 2026-06-17

> **Worker note (2026-06-17)**: All 5 variants built on branch `dx/landing-page`
> in `prog-strength-web`, behind the `NEXT_PUBLIC_ENABLE_DESIGN_EXPLORE` flag at
> `/design-explore/landing-page`. Draft PR open as the selection artifact — pick
> an idiom, tick its box, set `status: selected`, and **close the PR (never
> merge)**, then open the implementation SOW.

> A DX (Design Exploration) is the platform's **divergent** work type. Unlike a SOW it does not converge on one correct implementation — it produces N differentiated visual variants of a single frontend surface, side by side on one comparison route, awaiting a human pick. It **never merges** and ships no production code; the chosen direction feeds a downstream SOW that builds it for real.

## Context

`/login` is the unauthenticated front door — the first thing every beta invitee sees and the only Prog Strength surface a visitor reaches before signing in. Today it's a single centered card: the "P" mark in a soft tile, the wordmark, one tagline ("Sign in to track and chat about your training."), a **Continue with Google** button, and a "No password to manage." reassurance line, all on bare slate. It works, but it sells nothing — a visitor learns the product *name* and that it uses Google sign-in, and nothing about **what the product does**. There's no hint that the thing on the other side of that button is an AI coach you talk to in plain English, that it tracks workouts and macros and PRs, or that it's any good.

This DX explores a real **landing page** for that surface: a hero value prop, the one element that makes Prog Strength distinctive (you *talk* to a coach that logs your training from a sentence), and a couple of honest feature highlights — with **Continue with Google** still the obvious primary action. It is **not** a full marketing site and shouldn't over-promise for a pre-launch beta; the bar is "a real, intentional landing page that looks like the premium dark app you land in the moment you sign in," not a launch campaign. It is a visual/layout exploration only — the auth mechanism, the routes, and the product's information architecture are untouched.

## The surface

The unauthenticated page at `app/login/page.tsx`, rendered **outside the app shell** (no sidebar, no authed chrome). A variant owns the whole viewport from the brand lockup down. The hard constraints — keep these working, restyle freely:

- **The OAuth CTA.** The single action is "Continue with Google", which links to `${config.apiUrl}/auth/google/login?return_to=${origin}/auth/callback` (computed client-side; the button is disabled until `return_to` is known). Keep the **inline multicolor Google "G" glyph** and the **"We use Google sign-in to identify you. No password to manage."** reassurance — they're trust signals, not decoration. There is exactly one CTA; variants may repeat it (top + bottom) but must not invent a second auth path (no email/password — none exists).
- **The brand lockup.** The "P" `BrandMark` + "Prog Strength" wordmark. A Fixed Point; restyle its presentation, don't replace it.
- **Already-signed-in redirect.** If `isAuthenticated()`, the page replaces to `/chat` — purely behavioral, invisible to the mockup; variants don't render anything for it.

Around that, variants **add** landing content built from the **real product** (static fixtures that look real are preferred — do not wire anything live):

- **Hero** — a headline + one or two lines of subcopy stating the value prop. Lead with the AI-logging hook. Suggested copy (variants may reword): *"Tell it what you trained. It does the rest."* / *"Log meals and workouts in plain English. Track macros, lifts, and PRs. Talk to a coach that actually knows your training."*
- **The proof element — the chat exchange.** The distinctive thing. Render the real nutrition-logging fixture as a static chat snippet:
  1. **User:** "For lunch, I just had a turkey burger and bibigo white rice"
  2. **Assistant** (*via Sonnet 4.6*, tool chip `✓ Log Consumption`): "Logged! Here's your lunch:" + a macro **table** — Columbus Turkey Burger `240 / 30 / 13 / 2`, Bibigo Sticky White Rice `310 / 6 / 1 / 71`, **Total `550 / 36 / 14 / 73`** (columns: Item · Cal · P · F · C) + a coach-tone line ("Solid macro split. 💪").
  This carries the macro-tint colors (protein green, fat amber, carb blue) and shows the product's actual character. A landing page that describes the AI in prose instead of *showing* this exchange is the weaker outcome.
- **Feature highlights** — 3–4, each one line + an icon or a small mock, drawn from real surfaces:
  - **AI coach chat** — natural-language logging and training Q&A.
  - **Nutrition & macros** — daily macro tracking with the protein/fat/carb rings.
  - **Workouts & PRs** — log lifts, track exercises, watch estimated 1RMs climb.
  - **Calendar & progress** — a training calendar with streaks, bodyweight and progress charts.
- **Optional app screenshot/mock** — a framed glimpse of a real surface (the calendar, the macro rings, or the timeline) to ground "this is a polished app."

Data shape for the chat fixture (mirrors the live message model, for realistic mockups only):

```ts
type ToolCall = { name: string; state: "running" | "ok" | "error" };
type Message = {
  role: "user" | "assistant";
  content: string;            // markdown for assistant turns (tables, bold, emoji)
  tools?: ToolCall[];         // assistant only — e.g. [{ name: "Log Consumption", state: "ok" }]
  model?: string;             // assistant only — "claude-sonnet-4-6" → "via Sonnet 4.6"
};
```

**States to handle.** This surface is simpler than an authed one, but a variant still must look intentional at: the **default** full landing view; a **narrow/mobile** width (the hero, chat snippet, and feature list must reflow, not break); and the **CTA-disabled** beat before `return_to` resolves (the button styled as not-yet-ready, never as broken).

## Idioms

Five genuinely distinct directions. All are **in-system**: dark slate ramp, the violet accent (`#8b7cf6`), Nunito as the base family, rounded soft form, the macro tints for nutrition elements — per [`../design-system.md`](../design-system.md). They diverge on **type scale**, **color logic**, **spacing rhythm**, and **composition**, **not** on palette/accent/type family. The fixed points are the Prog Strength identity (the "P" mark, the wordmark) and that the macro table and Google CTA stay legible and trustworthy.

- **centered-hero-spotlight** — Linear's marketing calm, on our dark. A single centered column: the lockup, a **large display headline**, a quiet subline, the Google CTA, and **one framed app screenshot** floating below with a soft violet glow behind the brand mark. Type: big headline, modest body, hierarchy by scale. Color: near-monochrome slate with violet reserved for the CTA and the glow; macro tints appear only inside the screenshot. Spacing: airy, generous vertical rhythm, lots of negative space. The restrained, premium baseline.

- **split-screen-product** — Vercel/Stripe asymmetry. Two columns: **left** is the value copy + Google CTA + reassurance, left-aligned and dense; **right** is a **full-bleed live-looking product panel** (the chat exchange with the macro table, raised on the slate ramp). Type: mid-scale, left-aligned, tight headline. Color: violet anchors the left CTA; the right panel carries the macro-tint palette. Spacing: compact and structured on the left, edge-to-edge on the right — the conversation/product does the talking. Collapses to stacked on narrow widths.

- **athletic-display-bold** — Prog Strength's athletic identity, turned up. Leans the design system's **scoped condensed athletic display face** (Oswald-style) for a bold, near-uppercase headline ("TRAIN. LOG. PROGRESS." energy) and big stat numerals; the violet accent used **confidently** as a primary color, not a timid button tint. Type: large condensed display + Nunito body. Color: brand-forward — violet carries the page. Spacing: punchy headline block, then deliberate section gaps. Whoop/Nike-Training motivational register; the variant that looks like *no other* SaaS landing.

- **feature-showcase-stack** — Notion's feature-row scroll. Below a compact hero, a **vertical stack of alternating feature rows** — each pairs a one-line benefit with a small mock/screenshot (chat logging, macro rings, PR tracking, calendar streak), image-left/image-right alternating. Type: section headers + readable body, mid scale. Color: **each row picks up its own relevant macro/activity tint** as a subtle section accent (nutrition green, etc.), unified by violet CTAs top and bottom. Spacing: roomy, clearly sectioned editorial rhythm — the most "scrolling landing page" of the five.

- **conversation-led** — the hero **is** the chat. Front and center, the stylized turkey-burger → macro-table exchange *is* the headline; a short framing line and the Google CTA sit beneath it, so the product demonstrates its killer feature literally instead of describing it. Type: chat-comfortable, mid scale, friendly. Color: violet user bubble, raised-slate assistant card, macro tints in the table — our soft-modern-messenger language applied to a marketing hero. Spacing: comfortable rounded, the exchange given room to breathe. ChatGPT's "talk to it" hero, in Prog Strength's voice.

## References

- **Linear** — its **centered dark product hero**: a big headline over a single framed product shot, generous negative space, one restrained accent. Drives `centered-hero-spotlight`.
- **Vercel** — its **split hero with a live product panel** and crisp left-aligned copy. Drives `split-screen-product` (Stripe's product-forward right panel reinforces it).
- **Whoop** — its **athletic single-accent-on-charcoal palette and bold stat typography**. Drives `athletic-display-bold`.
- **Notion** — its **alternating feature-row scroll**, each section with its own light accent and a paired visual. Drives `feature-showcase-stack`.
- **ChatGPT** — its **conversation-as-the-hero** framing: the chat itself is the pitch. Drives `conversation-led`.

## Selection criteria

A note-to-self for the pick, not a rubric the worker optimizes against. When I compare these I'm weighing:

- **Does it stay unmistakably native to the dark app?** The user lands in the dark slate/violet shell the instant they sign in — a landing page that feels like a different product is jarring. In-system is the whole point here.
- **Does the AI-logging hook land in the first screen?** The distinctive thing is that you *talk* to it and it logs your training. If a visitor scrolls away not knowing that, the page failed regardless of how pretty it is.
- **Does the macro table / chat fixture still read cleanly?** It's the proof element; a variant that mangles the table or flattens the exchange into prose loses the one thing worth showing.
- **Does it look intentional, not a stock SaaS template?** Dark-hero-with-a-gradient is a dime a dozen; I want it to read as *ours*, with the athletic register available if I want to lean into it.
- **Is the Google CTA still the obvious primary action**, and do the trust signals (the "G" glyph, "No password to manage.") survive the restyle?
- **Does it hold up honest for a pre-launch beta** — confident without over-promising features that aren't there yet?

---

> **Lifecycle.** `status:` is editorial — the owner is the dispatch gate. It moves `draft` → `exploring` (worker running) → `awaiting_selection` (draft PR open, owner deciding) → `selected` / `abandoned`.
>
> **Handoff.** This DX ends at a *chosen variant*, not merged code. I open the draft `[DX — DO NOT MERGE]` PR, compare the variants on the preview deploy, pick one, tick its box, set `status: selected` (noting the winning idiom), and **close the PR — never merge it.** Then I open a SOW: *"implement the landing page per the `<chosen-idiom>` variant from `dx/landing-page`, production-quality, conforming to the design system."*
