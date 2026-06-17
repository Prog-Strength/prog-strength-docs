---
type: sow
status: ready_for_implementation
repos:
  - prog-strength-web
  - prog-strength-docs
---

# Landing Page — Feature-Showcase-Stack

**Status**: Ready for implementation · **Last updated**: 2026-06-16

## Introduction

`/login` is the unauthenticated front door — the first thing every beta invitee sees and the only Prog Strength surface a visitor reaches before signing in. Today it's a single centered card (the "P" `BrandMark`, the wordmark, one tagline, a **Continue with Google** button, and the "No password to manage." reassurance) on bare slate. It works, but it sells nothing: a visitor learns the product *name* and that it uses Google sign-in, and nothing about **what the product is** — that on the other side of that button is an AI coach you talk to in plain English that logs your training, tracks macros and lifts and PRs, and is actually good.

The Design Exploration `dx/landing-page.md` (PR Prog-Strength/prog-strength-web#63) explored that surface as a real landing page across five in-system variants, and the owner selected **`feature-showcase-stack`**: a compact hero, then a vertical, editorial **stack of alternating feature rows** — each pairing a one-line benefit with a small mock/screenshot (chat logging, macro rings, PR tracking, calendar streak), image-left/image-right — each row carrying its own relevant macro/activity tint, unified by violet CTAs top and bottom. The Notion feature-row register: the most "scrolling landing page" of the five.

This is a **web-only redesign** (plus docs bookkeeping). It rebuilds `app/login/page.tsx` into that landing page. **There is no backend change**: the auth mechanism, the routes, and the product's information architecture are untouched, and every visual is built from **static fixtures that look real** — nothing is wired live. The SOW inherits the DX's `scope: in-system`: it conforms to the design system (slate ramp, the single violet accent `#8b7cf6`, Nunito, rounded soft form, the macro tints) per [`../design-system.md`](../design-system.md), so the front door looks like the premium dark app you land in the instant you sign in.

## Proposed Solution

One shippable change: rebuild the unauthenticated `/login` page, rendered **outside the app shell** (no sidebar, no authed chrome), into the feature-showcase-stack landing page. The variant on the DX branch (`app/design-explore/landing-page/_variants/FeatureShowcaseStack.tsx` + its `_shared` fixtures/mocks) is the **visual spec**, not code to promote — it's reimplemented production-quality against the real login route.

The composition, top to bottom:

- **Compact hero** — the brand lockup, a headline + one or two lines of subcopy leading with the AI-logging hook (*"Tell it what you trained. It does the rest."* / "Log meals and workouts in plain English. Track macros, lifts, and PRs. Talk to a coach that actually knows your training." — copy may be reworded), and the **Continue with Google** CTA. Deliberately compact: it sets up the scroll rather than dominating it.
- **Alternating feature-row stack** — 3–4 rows, image-left/image-right alternating, each pairing a one-line benefit with a small mock/screenshot, each row picking up **its own relevant macro/activity tint** as a subtle section accent:
  1. **AI coach chat** — natural-language logging and training Q&A. Its mock is the **proof element**: the real turkey-burger → macro-table exchange rendered as a static chat snippet (user line → assistant reply via Sonnet 4.6 with a `✓ Log Consumption` tool chip + the macro table). This row carries the macro-tint palette (protein green / fat amber / carb blue).
  2. **Nutrition & macros** — daily macro tracking with the protein/fat/carb rings; nutrition-green accent.
  3. **Workouts & PRs** — log lifts, track exercises, watch estimated 1RMs climb; workouts-amber accent.
  4. **Calendar & progress** — a training calendar with streaks and progress charts; calendar-blue accent.
- **Closing CTA** — a second **Continue with Google** at the bottom of the scroll, with the same reassurance line. (The variant repeats the CTA top + bottom — it never invents a second auth path.)

The fixed points from the DX carry through unchanged: the OAuth CTA wiring, the inline multicolor Google "G" glyph, the "We use Google sign-in to identify you. No password to manage." reassurance, the "P" + wordmark lockup, and the already-signed-in `replace('/chat')` redirect.

## Goals and Non-Goals

### Goals

- **Rebuild `app/login/page.tsx`** into the feature-showcase-stack landing page: compact hero → alternating feature-row stack → closing CTA, rendered outside the app shell across the full viewport.
- **Conform to the design system** ([`../design-system.md`](../design-system.md)): the slate ramp, violet accent (`#8b7cf6`), Nunito base family, rounded soft form, and macro tints — drawn from the existing CSS-variable tokens the login page already uses (`var(--surface)`, `var(--border)`, `var(--muted)`, …), not new hex. Token coherence with the redesigned app shell (`sows/chat-and-app-shell-redesign.md`).
- **Preserve the OAuth CTA exactly**: links to `${config.apiUrl}/auth/google/login?return_to=${encodeURIComponent(returnTo)}`, computed client-side; the button is **disabled until `return_to` resolves** and styled as not-yet-ready (never broken) in that beat. Keep the inline multicolor Google "G" glyph and the "No password to manage." reassurance as trust signals. CTA repeated top + bottom; no second auth path.
- **The proof element renders cleanly** — the turkey-burger → macro-table chat exchange as a static fixture (tool chip + "via Sonnet 4.6" + the macro table with protein/fat/carb tints and the Total row), legible at all widths. Showing the exchange beats describing the AI in prose.
- **Each feature row carries its own macro/activity tint** (nutrition green, workouts amber, calendar blue) as a subtle section accent, unified by violet CTAs.
- **Intentional at every state**: the default full landing view, a **narrow/mobile** width where the hero, chat snippet, and feature rows reflow (alternating rows stack to a single column, not break), and the **CTA-disabled** pre-`return_to` beat.
- Preserve the already-signed-in redirect (`isAuthenticated()` → `replace('/chat')`) — behavioral, invisible to the layout.
- Keep web CI green (lint/format/typecheck/test/build), updating/adding tests for the new login UI.

### Non-Goals

- **Any backend change.** Auth, routes, and IA are untouched; `prog-strength-api` is not in `repos:`. Everything the chosen design needs is static and buildable in web today.
- **Wiring anything live.** The chat exchange, macro table, rings, PR, and calendar visuals are **static fixtures that look real** — no API calls, no live data on an unauthenticated page.
- **A second auth path.** No email/password (none exists). The single action is Continue with Google; it may appear top and bottom but is one path.
- **A full marketing site / launch campaign.** This is one intentional landing page for a pre-launch beta — confident without over-promising features that aren't there yet. No pricing, testimonials, blog, footer sitemap, or multi-page marketing IA.
- **Changing the brand lockup, the Google glyph, or the reassurance copy** beyond restyling their presentation — they're fixed trust/identity signals.
- **Promoting the DX mockup code.** The variant (`app/design-explore/landing-page/_variants/FeatureShowcaseStack.tsx`, `_shared/{fixtures,mocks,cta}`) is the visual spec, not code to copy. The `design-explore` route stays feature-gated behind `NEXT_PUBLIC_ENABLE_DESIGN_EXPLORE`, is untouched, and never ships to production.
- **The other four variants.** centered-hero-spotlight, split-screen-product, athletic-display-bold, and conversation-led are not built; the DX PR is closed (never merged).

## Implementation Details

### Web: the landing page (`prog-strength-web`)

Rebuild `app/login/page.tsx` (and any new `_components` it needs) into the feature-showcase-stack landing, rendered outside the app shell. Token conformance first: reuse the existing CSS-variable tokens the page already references (`var(--surface)`, `var(--surface-2)`, `var(--border)`, `var(--muted)`, `var(--foreground)`) and the violet accent / macro tints from `design-system.md` — reference tokens, not new hex.

- **Hero** (`_components`) — the `BrandMark` + "Prog Strength" lockup, a headline + 1–2 subcopy lines leading with the AI-logging hook, and the primary `GoogleCtaButton`. Compact vertical rhythm so it reads as the opener of a scroll, not a full-screen splash.
- **GoogleCtaButton** (extract from the current inline button) — preserves the exact OAuth href (`${config.apiUrl}/auth/google/login?return_to=${encodeURIComponent(returnTo)}`), the `disabled`-until-`returnTo`-resolves behavior with a not-yet-ready style, the inline multicolor Google "G", and the reassurance line. Reused for the top and bottom CTAs.
- **FeatureRow** (`_components`) — an alternating image-left/image-right row taking a one-line benefit, a small mock/screenshot, and a tint token. Alternation by index; collapses to a single stacked column at narrow widths. Tints: chat row → no single tint (carries the full macro palette inside its mock), nutrition → green, workouts → amber, calendar → blue.
- **ChatProofMock** — the static turkey-burger → macro-table exchange (user bubble; assistant card with the `✓ Log Consumption` tool chip, "via Sonnet 4.6", and the macro table: Columbus Turkey Burger `240 / 30 / 13 / 2`, Bibigo Sticky White Rice `310 / 6 / 1 / 71`, **Total `550 / 36 / 14 / 73`**; columns Item · Cal · P · F · C; a coach-tone closing line). Macro tints in the table. This is the one mock that must read cleanly at every width — it's the proof.
- **Supporting mocks** — small, on-brand static renders for the macro rings, a PR/1RM stat, and a calendar streak. Static SVG / styled markup, no new deps, no live data.
- **Closing CTA** — a second `GoogleCtaButton` + reassurance at the foot of the stack.
- Preserve the already-signed-in `useEffect` redirect (`isAuthenticated()` → `router.replace('/chat')`) exactly.

The DX `_shared` fixtures (the `Message`/`ToolCall` shapes, the macro numbers) are the source of the copy and data; reimplement them as local fixtures for the login route rather than importing from the `design-explore` tree (which stays gated and out of the production bundle).

### Tests

- **Web**: `app/login` + `_components` tests — the hero renders the lockup, headline, and primary CTA; the feature-row stack renders all rows with alternating sides and the correct per-row tint; the **ChatProofMock** renders the macro table with the Total row from a known fixture; the CTA href is composed correctly once `returnTo` resolves and the button is **disabled** before it; both top and bottom CTAs preserve the Google glyph and reassurance; the already-signed-in path triggers `replace('/chat')`; and the layout reflows to a single column at a narrow width. lint/format/typecheck/test/build green.

## Rollout

1. **`prog-strength-web`** — the login-page rebuild. Vercel preview to verify the hero, the alternating feature rows with their tints, the chat proof mock, both CTAs (including the disabled pre-`return_to` beat), the narrow/mobile reflow, and that signing in still works end-to-end.
2. **`prog-strength-docs`** — flip `dx/landing-page.md` to `status: selected` (idiom `feature-showcase-stack`); mark this SOW shipped on merge.

### Verification after rollout

- `/login` (signed out): compact hero leading with the AI-logging hook + Continue with Google; an alternating feature-row scroll (chat logging, macro rings, PRs, calendar) each with its own tint; a closing CTA — all on the app's slate/violet/Nunito theme.
- The chat proof mock renders the turkey-burger → macro table with the Total row and tints, legible; it's *shown*, not described in prose.
- The Google CTA is the obvious primary action, disabled until `return_to` resolves, with the "G" glyph and "No password to manage." intact; clicking it completes Google sign-in and lands in `/chat`.
- An already-signed-in visitor is redirected to `/chat` and never sees the landing content.
- At a narrow/mobile width the hero, chat snippet, and feature rows reflow to a single column — nothing breaks.
- The `design-explore/landing-page` route remains 404 in production (gate off); no DX mockup code shipped.

## Open Questions

- **Number of feature rows** — 3 vs 4. The chat-proof row is non-negotiable (it's the proof); nutrition, workouts, and calendar are the candidates. Lean: all four, since the alternating rhythm wants an even-ish stack and each maps to a real surface. Drop calendar first if the scroll feels long.
- **Supporting mock fidelity** — how polished the rings / PR / calendar mocks need to be versus the hero chat mock. Lean: the chat exchange is the hero mock and gets the most fidelity; the others are smaller, suggestive static renders. Visual call at implementation.
- **Hero screenshot** — the DX allows an optional framed app screenshot to ground "this is a polished app." Lean: skip a literal screenshot in favor of the feature-row mocks carrying that weight (the stack *is* the product tour); revisit if the hero feels thin.
