---
type: sow
status: ready_for_implementation
repos:
  - prog-strength-web
  - prog-strength-docs
---

# Settings Page Redesign — Grouped Save-Bar

**Status**: Ready for implementation · **Last updated**: 2026-06-17

## Introduction

`/settings` is where a user manages who they are in the product — their display name, public `@handle`, bio, height, unit preferences, avatar, Google Calendar connection, and a read-only readout of their daily AI allowance. It works, but it has no single save model. Today (`app/(app)/settings/page.tsx`, one ~900-line client component) the profile text fields — name, username, bio, height — each carry **their own per-field "Save" button**, while unit preferences and the calendar default-detail toggle persist **silently and optimistically** the instant you flip them, and the avatar uploads the moment you pick a file. So the page asks you to think about saving in three incompatible ways at once: a column of Save buttons sitting next to toggles that never had one. There's no way to see "what have I changed?", no way to discard, and the visual hierarchy is a flat run of rows rather than grouped, scannable sections.

The Design Exploration `dx/settings.md` (PR Prog-Strength/prog-strength-web#66) explored that surface across five in-system variants, and the owner selected **`grouped-savebar`**: Vercel / GitHub-style **sectioned cards** that group fields logically, with **no per-field buttons** anywhere, and a single **sticky "Save changes" bar** that slides up only when something is dirty — showing a dirty count ("2 unsaved changes"), a **Discard**, and one **Save** that's gated by validation. Violet is concentrated on that one bar (and the active toggle pills), so the page finally has a single, obvious point of action and one coherent save story.

This is a **web-only redesign** (plus docs bookkeeping). It rebuilds `app/(app)/settings/page.tsx` against the **existing** data layer — `useProfile()` / `updateMe`, `uploadAvatar` / `deleteAvatar`, `checkUsernameAvailable`, `getCalendarConnection` / `disconnectCalendar`, `useUsage()`. **There is no backend change**: every endpoint the chosen design needs already ships. The SOW inherits the DX's `scope: in-system` — it conforms to the design system (slate ramp, the single violet accent `#8b7cf6`, Nunito, rounded soft cards) per [`../design-system.md`](../design-system.md).

## Proposed Solution

One shippable change: rebuild the authed `/settings` page (rendered inside the app shell) into the grouped-savebar layout. The variant on the DX branch (`app/(app)/design-explore/settings/_components/variant-grouped-savebar.tsx` + `fixtures.ts`) is the **visual spec**, not code to promote — it's reimplemented production-quality against the real route, wired to the real profile context instead of fixtures.

The core change is the **save model**. A single page-level `draft` state holds every editable field, diffed against an immutable `initial` baseline captured from `useProfile()`:

- Editing any field updates `draft` only — **no field saves on its own**. (This replaces the four per-field Save buttons *and* the silent optimistic toggles with one model.)
- A **sticky save bar** at the foot of the page appears only when `dirtyCount > 0`, showing `"{n} unsaved change(s)"`, a **Discard** (resets `draft` to `initial`), and a **Save changes** button.
- **Save is gated**: disabled while the display name is empty, or while the username is mid-probe / taken / reserved / charset-invalid. When blocked, the bar still shows but the button is disabled with an inline reason ("checking username…" / "fix username first").
- **Save** issues one `updateMe(patch)` with just the dirty keys, then flashes "All changes saved ✓" (~1.8s) and the bar retracts. On error the bar stays and surfaces the failure.

The composition, top to bottom — grouped cards (`rounded-2xl`, hairline border, soft shadow, small-caps-ish bold section title over a divider):

- **Profile card** — avatar (image, or initials via `initialsOf`) with an out-of-band Upload/Change/Remove; display name (required, ≤60); username (`@`-prefixed, live availability); bio (textarea, 160-**rune** counter).
- **Daily AI allowance card** — read-only usage readout: progress bar + percentage + "Resets in …", with the **accent → amber (≥80%) → red (100%)** threshold. Unmistakably not editable; never contributes to dirty state.
- **Units card** — distance (mi/km) and weight (lb/kg) as segmented violet pills. Distance also drives the height unit (in/cm).
- **Google Calendar card** — connection status (connected / absent / checking) with Connect (OAuth redirect) or Disconnect, plus the default-event-detail segmented toggle.

Two fields stay deliberately **out of the draft/save-bar model**, because they're inherently immediate actions, not form edits: **avatar** upload/remove (a file operation — uploads on pick, as today) and **calendar connect/disconnect** (an OAuth round-trip / server revoke). Everything else flows through the draft and the one Save bar.

## Goals and Non-Goals

### Goals

- **Rebuild `app/(app)/settings/page.tsx`** into the grouped-savebar layout: grouped cards (Profile · Daily AI allowance · Units · Google Calendar) with **no per-field Save buttons and no silent optimistic toggles** — a single page-level `draft` and **one sticky save bar**.
- **One coherent save model**: edits accumulate in `draft`; the sticky bar appears only when dirty, shows the dirty count + **Discard** + **Save changes**; Save issues **one `updateMe(patch)`** with only the changed keys, then a "saved ✓" flash; the bar retracts. Discard resets to the baseline.
- **Gate Save on validation**: disabled while display name is empty or the username is checking / taken / reserved / charset-invalid, with an inline reason in the bar.
- **Reuse the existing data layer unchanged** — `useProfile()` + `update`/`uploadAvatar`/`removeAvatar`, `checkUsernameAvailable`, `getCalendarConnection`/`disconnectCalendar`, `useUsage()`/`UsageBar`, `useToast()`, `useDistanceUnit()`. No new endpoints, no new SDK methods.
- **Preserve every field state and semantic** the current page has: username live-availability (checking / available / taken / charset-invalid, debounced probe via `checkUsernameAvailable`); bio **rune** counter (code points, not UTF-16) hard-capped at 160; height as canonical cm with in/cm display per distance unit, including the **unset** state; avatar image-vs-initials (`initialsOf`) with the 2 MB / png·jpeg·webp client guard; calendar connected / absent / checking; the usage readout's **accent → amber (≥80%) → red (100%)** thresholds and "resets in" countdown; an unmistakably **read-only** allowance.
- **Keep avatar and calendar as immediate out-of-band actions** (not part of the draft/save-bar), matching today's behavior.
- **Conform to the design system** ([`../design-system.md`](../design-system.md)): slate ramp, violet accent (`#8b7cf6`), Nunito, `rounded-2xl` cards, full-pill toggles — via the existing CSS-variable tokens (`var(--surface)`, `var(--border)`, `var(--muted)`, `var(--accent)`, `var(--accent-fg)`, `var(--warning)`, `var(--danger)`, …), not new hex. Violet concentrated on the save bar + active toggle pills.
- **Intentional at every state**: the clean (non-dirty) page with no bar; the dirty state with the count; the save-blocked state (empty name / bad username) with the disabled button + reason; the post-save flash; and a **narrow/mobile** width where cards and the save bar reflow (single column; the bar must not obscure content — keep the bottom padding reservation).
- Preserve the context's 401 handling (`clearToken()` → redirect to `/login`) and per-field error surfacing (inline + toast).
- Keep web CI green (lint/format/typecheck/test/build), rewriting `app/(app)/settings/page.test.tsx` for the new save model.

### Non-Goals

- **Any backend change.** Every endpoint already exists; `prog-strength-api` is not in `repos:`. No schema, route, or SDK changes.
- **A new "reserved username" server check.** The DX mock distinguishes a `reserved` state from `taken`, but production `checkUsernameAvailable` returns a boolean (free / taken) and the server is authoritative on reserved words. Keep the real two-way result — render server rejection as taken/error — rather than inventing a reserved endpoint. (See Open Questions.)
- **Folding avatar or calendar into the save bar.** They remain immediate actions; only the draftable profile/unit fields gate the bar.
- **Changing the field set, the username/bio/height semantics, the OAuth flow, or unit conversions.** This is a layout + save-model redesign, not a behavior change to the underlying data.
- **A two-pane / left-nav settings shell** or any of the other four variants (linear-minimal, two-pane-nav, athletic-identity, conversational-inline). The DX PR is closed (never merged).
- **Promoting the DX mockup code.** `variant-grouped-savebar.tsx` + `fixtures.ts` are the visual spec, not code to copy; the `design-explore/settings` route stays feature-gated (`NEXT_PUBLIC_DESIGN_EXPLORE`), untouched, and never ships to production.

## Implementation Details

### Web: the settings page (`prog-strength-web`)

Rebuild `app/(app)/settings/page.tsx` (and the `_components` it needs) into the grouped-savebar layout, rendered inside the existing app shell. Token conformance first: reuse the CSS-variable tokens the page already references — reference tokens, not new hex.

- **Page-level draft state** — capture an immutable `initial` from `useProfile()` (display name, username, bio, height-as-display-string, distance unit, weight unit, calendar default detail) and a `draft` of the same shape. `set(key, value)` updates `draft`. `dirtyKeys = keys where draft[k] !== initial[k]`; `dirtyCount = dirtyKeys.length`. When the profile context updates after a successful save, re-baseline `initial` to the returned profile so the bar retracts and a fresh diff starts.
- **SaveBar** (`_components`) — sticky at the foot (`sticky bottom-…`, soft shadow, `border-[var(--accent-line)]`); rendered only when `dirtyCount > 0 || savedFlash`. Left: `"{dirtyCount} unsaved change(s)"`, or the inline block reason when `canSave` is false, or "All changes saved ✓" during the flash. Right: **Discard** (text button → reset `draft = initial`) and **Save changes** (violet, `disabled={!canSave}`). Reserve bottom padding on the page container so the bar never covers the last card.
- **Save handler** — build the patch from `dirtyKeys` only, mapping to `updateMe`'s shape (e.g. `height` display-string → canonical `height_cm` at the boundary, `""` → `null`; username normalized lowercase; distance/weight/detail passed through). Call `useProfile().update(patch)`. On success: re-baseline, `setSavedFlash(true)` then clear after ~1.8s. On error: keep the bar, toast/inline the message. `canSave = dirtyCount > 0 && !nameEmpty && !handleBlocked`.
- **Card / Field / Toggle primitives** (`_components`) — `Card({title, children})` (`rounded-2xl` border, divider under a bold title, padded body); `Field({label, hint, children})` (label-over-control, muted hint); `SegmentedToggle` (full-pill, active = `bg-[var(--accent)] text-[var(--accent-fg)]`). Inputs use the shared focus-ring-violet input style.
- **Profile card** — avatar (resolved image, else `initialsOf(displayName)` on `var(--surface-2)`) with Upload/Change/Remove wired to `uploadAvatar` / `removeAvatar` **immediately** (out-of-band, keep the 2 MB / png·jpeg·webp client guard + toast on reject); display name (required, `maxLength=60`, inline "required" when empty); username (`@`-prefixed, `maxLength=30`, lowercased, `spellCheck=false`); bio (`textarea`, `clampRunes(…, 160)` on change, `runeLength` counter in `tabular-nums`).
- **UsernameField** — preserve the debounced (~400ms) probe via `checkUsernameAvailable(token, normalized)`, fired only when the handle is dirty and charset-valid (`/^[a-z][a-z0-9_]{2,29}$/`). States: idle / checking / available / taken / error → inline status line. The probe result feeds `handleBlocked` (checking, taken, invalid → block Save). The handle never had a per-field button; now it gates the one bar instead.
- **Daily AI allowance card** — reuse the existing `UsageBar` / `useUsage()` readout (bar + percentage + "resets in", accent→amber→red thresholds, capped message). Restyle into the card; it is read-only and excluded from `draft`/dirty.
- **Units card** — distance (mi/km) and weight (lb/kg) `SegmentedToggle`s bound to `draft`. Distance drives the height unit label (in/cm) and the height display conversion. (Distance unit also persists to `useDistanceUnit()` / localStorage on save, matching current behavior — confirm both the context and `updateMe` stay in sync.)
- **Google Calendar card** — `getCalendarConnection()` on mount (connected / absent / checking); Connect (redirect to `${config.apiUrl}/auth/google/calendar/connect?return_to=…/settings`) or Disconnect (`disconnectCalendar` + reload), both **immediate**, out-of-band. Default-event-detail `SegmentedToggle` bound to `draft`.

The DX `fixtures.ts` (the `Draft` shape, the field set, the helper functions `initialsOf` / `runeLength` / `clampRunes` / `usageColor` / `heightToDisplay`) is the source of the shape and the copy; reimplement the helpers as local utilities for the settings route (or reuse the production equivalents already in `page.tsx`) rather than importing from the gated `design-explore` tree.

### Tests

- **Web**: rewrite `app/(app)/settings/page.test.tsx` for the new model — editing a field shows the save bar with the correct dirty **count**; Discard reverts every field and hides the bar; **Save issues one `updateMe` with only the changed keys** and flashes "saved ✓", then re-baselines (bar gone); Save is **disabled** when the display name is empty and when the username probe returns taken / is checking / is charset-invalid, with the inline reason; the username debounced probe transitions checking → available/taken; the bio rune counter caps at 160 (incl. an emoji case); height converts in↔cm and the unset state round-trips; avatar upload/remove fire **immediately** and stay out of the dirty count; calendar connect/disconnect stay out of the dirty count; the usage readout colors at 50 / 80 / 100%. Keep the existing API/context/toast mocks. lint/format/typecheck/test/build green.

## Rollout

1. **`prog-strength-web`** — the settings-page rebuild. Vercel preview to verify the grouped cards, the single sticky save bar (appear-on-dirty, dirty count, Discard, gated Save, saved-flash), that avatar + calendar remain immediate, the read-only usage readout, and the narrow/mobile reflow — then confirm a real save persists end-to-end against the API.
2. **`prog-strength-docs`** — flip `dx/settings.md` to `status: selected` (idiom `grouped-savebar`); mark this SOW shipped on merge.

### Verification after rollout

- `/settings` (signed in): four grouped cards — Profile, Daily AI allowance, Units, Google Calendar — on the app's slate/violet/Nunito theme, with **no per-field Save buttons** and **no silent toggles** for draftable fields.
- Editing any field (name, username, bio, height, units, calendar detail) slides up **one** sticky save bar showing the dirty count; Discard reverts everything; Save persists in a single request and shows "saved ✓", then the bar retracts.
- Save is disabled — with a visible reason — when the display name is empty or the username is checking / taken / charset-invalid; the username availability line resolves live as you type.
- The bio counter caps at 160 runes; height shows in the unit chosen under Units and handles "no height set"; the avatar shows the image or initials and uploads/removes immediately; Google Calendar connects/disconnects immediately — none of these four immediate actions touch the save bar.
- The Daily AI allowance reads as plainly read-only, with the bar coloring accent → amber (≥80%) → red (100%) and a "resets in" countdown.
- At a narrow/mobile width the cards stack to a single column and the save bar reflows without obscuring the last card.
- The `design-explore/settings` route remains 404 in production (gate off); no DX mockup code shipped.

## Open Questions

- **`reserved` username state** — the DX mock shows a distinct "reserved" message; production `checkUsernameAvailable` only returns free/taken and the server rejects reserved handles on save. Lean: keep the real two-way probe and let a reserved handle surface as taken/error (inline) — don't add a reserved endpoint just for the label. Revisit only if a dedicated reserved message is judged worth a small API addition (a separate SOW).
- **Distance-unit dual write** — distance unit lives both in `useDistanceUnit()` (localStorage) and in the profile (`distance_unit`). Lean: on Save, write through both (as today) so the cross-device profile value and the local context stay in sync; confirm there's no flash from a stale localStorage value after save.
- **Avatar/calendar vs. an unsaved draft** — since those are immediate but other fields may be dirty, an upload mid-edit won't be lost (it's out-of-band) but could feel inconsistent next to a pending bar. Lean: accept it — they're genuinely different action types — but consider a subtle "saved" toast on avatar/calendar so the user sees they persisted independently of the bar.
</content>
</invoke>
