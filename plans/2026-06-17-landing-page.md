# Landing Page — Feature-Showcase-Stack Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Rebuild the unauthenticated `/login` page in `prog-strength-web` into the selected `feature-showcase-stack` landing page — a compact hero, a vertical stack of alternating image-left/image-right feature rows (chat proof, nutrition macros, workouts/PRs, calendar streak), and a closing CTA — rendered outside the app shell, built entirely from static fixtures, with **zero backend change**. Conforms to the design system (slate ramp, single violet accent `#8b7cf6`, Nunito, rounded soft form, macro tints) using existing CSS-variable tokens.

**Architecture:** Presentation-layer only, scoped to the `app/login` route. Extract the inline OAuth button into a reusable `GoogleCtaButton` that preserves the exact href, the disabled-until-`returnTo` behavior, the multicolor Google "G", and the reassurance line. Add page-private `_components/` (Hero, FeatureRow, ChatProofMock, supporting mocks) and a local `_fixtures.ts` (chat exchange + macro numbers, reimplemented from the DX `_shared` spec rather than imported from the gated `design-explore` tree). `page.tsx` keeps the existing `useEffect` already-signed-in redirect and the client-side `returnTo` computation, and composes the sections. No new dependencies, no API calls, no live data.

**Tech Stack:** Next.js 16 App Router, React 19, TypeScript, Tailwind v4 (utility classes + CSS-variable theme tokens), Vitest + Testing Library (jsdom), co-located `*.test.tsx`.

---

## File Structure

**Created:**
- `prog-strength-web/app/login/_fixtures.ts` — local static fixtures: the `Message`/`ToolCall`-shaped chat exchange (turkey-burger → macro table), the macro table rows + total, and the supporting mock data (macro rings totals, a PR/1RM stat, a calendar streak). Pure data, no React.
- `prog-strength-web/app/login/_components/google-cta-button.tsx` — the extracted OAuth CTA + reassurance line; reused top and bottom.
- `prog-strength-web/app/login/_components/hero.tsx` — brand lockup + headline + subcopy + primary `GoogleCtaButton`.
- `prog-strength-web/app/login/_components/feature-row.tsx` — alternating image-left/image-right row (benefit text + mock + tint); collapses to one column at narrow widths.
- `prog-strength-web/app/login/_components/chat-proof-mock.tsx` — the static turkey-burger → macro-table chat exchange (tool chip, "via Sonnet 4.6", macro table with tints + Total row).
- `prog-strength-web/app/login/_components/supporting-mocks.tsx` — `MacroRingsMock`, `PrStatMock`, `CalendarStreakMock` (static SVG / styled markup).
- Co-located tests: `app/login/page.test.tsx`, `_components/google-cta-button.test.tsx`, `_components/feature-row.test.tsx`, `_components/chat-proof-mock.test.tsx`.

**Modified:**
- `prog-strength-web/app/login/page.tsx` — recomposed from a single centered card into hero → feature-row stack → closing CTA. Preserves the `isAuthenticated()` → `router.replace('/chat')` redirect and the `returnTo` computation; the inline button markup moves into `GoogleCtaButton`.

**Untouched (verify, don't change):** `lib/config.ts`, `lib/auth.ts`, `components/brand-mark.tsx`, `app/layout.tsx`, `app/globals.css` (all needed tokens already exist), the `app/(app)` shell, and the `app/design-explore` tree (not in this checkout; stays gated, never shipped).

---

## Tokens (reference, do not introduce new hex)

All exist in `app/globals.css`:
- Surfaces: `var(--background)`, `var(--surface)`, `var(--surface-2)`, `var(--surface-3)`; text `var(--foreground)`, `var(--muted)`, `var(--faint)`; borders `var(--border)`, `var(--border-strong)`.
- Accent (CTAs, top + bottom): `var(--accent)` `#8b7cf6`, `var(--accent-dark)`, `var(--accent-soft)`, `var(--accent-line)`, `var(--accent-fg)`.
- Macro/section tints: protein green `var(--macro-protein)` `#34d399`, fat amber `var(--macro-fat)` `#fbbf24`, carb blue `var(--macro-carb)` `#60a5fa` (each with a `*-bg` ~13%-alpha companion). Feature-row tints: nutrition→protein-green, workouts→fat-amber, calendar→carb-blue; the chat row carries the full macro palette inside its mock (no single row tint).
- Radii/depth: `var(--radius-card)`, `var(--radius-card-lg)`, `var(--radius-pill)`, `var(--shadow-soft)`.

---

## Fixed points carried through unchanged (from the SOW)

- The OAuth href: `${config.apiUrl}/auth/google/login?return_to=${encodeURIComponent(returnTo)}`, computed client-side; the CTA is **disabled until `returnTo` resolves** and styled not-yet-ready (never broken).
- The inline multicolor Google "G" glyph (the existing 4-path SVG).
- The reassurance line: "We use Google sign-in to identify you. No password to manage."
- The "P" `BrandMark` + "Prog Strength" wordmark lockup.
- The already-signed-in `useEffect` → `router.replace('/chat')` redirect.
- Single auth path: the CTA may appear top + bottom, but it is one action; no email/password.

## Chat proof fixture (exact, from the SOW)

User line: a natural-language meal log ("had a turkey burger and rice", reworded acceptably). Assistant reply via **Sonnet 4.6** with a `✓ Log Consumption` tool chip and a macro table:

| Item | Cal | P | F | C |
| --- | --- | --- | --- | --- |
| Columbus Turkey Burger | 240 | 30 | 13 | 2 |
| Bibigo Sticky White Rice | 310 | 6 | 1 | 71 |
| **Total** | **550** | **36** | **14** | **73** |

Columns labeled Item · Cal · P · F · C; macro tints on P/F/C; a coach-tone closing line. This mock must read cleanly at every width — it is the proof element.

---

## Tasks

### Task 1 — Local fixtures + GoogleCtaButton
- [ ] Create `app/login/_fixtures.ts` with the chat exchange (`Message`/`ToolCall` shapes), the macro table rows + total, and supporting-mock data. Numbers exactly as above.
- [ ] Create `_components/google-cta-button.tsx` extracting the current button: takes `loginHref: string | null`; renders the multicolor "G", "Continue with Google", disabled/not-ready style when `loginHref` is falsy, and the reassurance line beneath. No behavior change vs. today's inline button.
- [ ] Tests: `google-cta-button.test.tsx` — composes the href when ready; `aria-disabled` + non-interactive when not ready; renders glyph + reassurance.
- **Spec-review + code-quality-review before Task 2.**

### Task 2 — ChatProofMock + supporting mocks
- [ ] `_components/chat-proof-mock.tsx` — user bubble (violet) + assistant card (raised slate) with the tool chip, "via Sonnet 4.6", the macro table (tinted P/F/C, Total row), and a coach-tone closing line. Legible at all widths.
- [ ] `_components/supporting-mocks.tsx` — `MacroRingsMock` (hand-rolled SVG donuts, protein/carb/fat tints), `PrStatMock` (a 1RM stat climbing), `CalendarStreakMock` (a week/streak grid). Static, no deps, no live data.
- [ ] Tests: `chat-proof-mock.test.tsx` — renders the macro table including the Total row `550 / 36 / 14 / 73` from the fixture.
- **Spec-review + code-quality-review before Task 3.**

### Task 3 — Hero, FeatureRow, page composition
- [ ] `_components/hero.tsx` — `BrandMark` + "Prog Strength" lockup, headline leading with the AI-logging hook + 1–2 subcopy lines, primary `GoogleCtaButton`. Compact vertical rhythm.
- [ ] `_components/feature-row.tsx` — props: benefit copy, mock node, tint token, and side (alternation by index). Image-left/image-right at wide widths; single stacked column at narrow widths.
- [ ] Rebuild `app/login/page.tsx`: keep the redirect + `returnTo` logic; compose hero → 4 feature rows (chat, nutrition, workouts, calendar) → closing CTA, full viewport, outside the app shell.
- [ ] Tests: `feature-row.test.tsx` (alternating side + tint); `page.test.tsx` (lockup/headline/primary CTA; all rows + tints; CTA disabled pre-`returnTo` then href composed; both CTAs keep glyph + reassurance; already-signed-in triggers `replace('/chat')`; single-column reflow at narrow width).
- **Spec-review + code-quality-review.**

### Task 4 — Gate green + ship
- [ ] `npm run lint && npm run format:check && npm run typecheck && npm run test && npm run build` all green.
- [ ] Branch `feat/landing-page`, conventional commit, push, open PR.

---

## Verification (per SOW Rollout)

- `/login` signed out: compact hero with the AI-logging hook + Continue with Google; alternating feature-row scroll (chat, macro rings, PRs, calendar) each with its tint; closing CTA — all slate/violet/Nunito.
- Chat proof mock renders the turkey-burger → macro table with the Total row and tints, legible — shown, not described.
- Google CTA is the obvious primary action, disabled until `return_to` resolves, "G" glyph + "No password to manage." intact.
- Already-signed-in visitor → `/chat`, never sees the landing.
- Narrow/mobile width: hero, chat snippet, feature rows reflow to one column — nothing breaks.
- `design-explore/landing-page` remains 404 in production; no DX mockup code shipped.
</content>
</invoke>
