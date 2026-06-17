---
type: dx
status: draft
surface: settings
idioms:
  - linear-minimal
  - grouped-savebar
  - two-pane-nav
  - athletic-identity
  - conversational-inline
references:
  - Linear
  - Vercel
  - Stripe
  - Whoop
  - Notion
scope: in-system
variant_count: 5
repos:
  - prog-strength-web
  - prog-strength-docs
---

# DX: User Settings

**Status**: Draft · **Last updated**: 2026-06-16

> A DX (Design Exploration) is the platform's **divergent** work type. Unlike a SOW it does not converge on one correct implementation — it produces N differentiated visual variants of a single frontend surface, side by side on one comparison route, awaiting a human pick. It **never merges** and ships no production code; the chosen direction feeds a downstream SOW that builds it for real.

## Context

`/settings` is where a Prog Strength user manages who they are to the product — the name their coach calls them, their public `@handle` and bio, their height, their avatar, their unit preferences, their Google Calendar connection — plus a read-only look at their daily AI allowance. It's a surface every beta user touches early (to set a username and units) and returns to rarely.

Today it works but reads as an unfinished form, and the reason is an **inconsistent, repetitive interaction model**. The page is a vertical stack of identical bordered cards, one per field. The three text fields and the height field each carry **their own "Save" button** — four near-identical violet buttons marching down the page — while the segmented toggles right beside them (distance, weight, calendar detail) **save instantly with no button at all**. So two settings that sit inches apart commit completely differently, and the eye is drawn to a column of redundant Save buttons rather than to the user's own information. Nothing is grouped or weighted; the avatar, the username with its live availability check, and the usage bar all sit in the same flat slate box at the same visual altitude.

This DX explores what this surface should look and feel like as a **considered settings page**: one coherent save model instead of per-field buttons next to silent toggles, a clear visual hierarchy for the user's identity, and a layout that still reads cleanly as more settings land. Because the *how do edits commit* question is the heart of what's wrong here, **the save model is itself a design axis** in this exploration — the variants don't just relayout the same buttons, they each propose a different, internally-consistent answer to "how does a change get saved," so the pick is as much about which interaction feels right as which composition looks best. It is a visual/interaction exploration only — the fields, their validation rules, and the underlying `PATCH /me` / avatar / calendar endpoints are untouched.

## The surface

The authenticated page at `app/(app)/settings/page.tsx`, rendered **inside the app shell** (sidebar + header chrome stay; the variant owns the scrolling content column). Today it's a single `max-w-2xl` column of four titled sections. Every field reads/writes through the shared `useProfile()` context (`GET`/`PATCH /me`) so an edit propagates to the sidebar account row and the chat payload instantly; the distance unit is the one exception, held in a localStorage-backed `DistanceUnitContext`. Restyle and re-compose freely; keep the data, the validation, and the commit semantics intact.

The settings, grouped as they are today (a variant may regroup or relabel, but every one of these must be present and editable):

**Profile**

- **Display name** — required, non-empty, ≤ 60 chars. "The name your coach calls you by." Empty is a validation error.
- **Username** (`@handle`) — 3–30 chars, must start with a lowercase letter, then lowercase letters / digits / underscores (`/^[a-z][a-z0-9_]{2,29}$/`). Nullable (a user may not have set one). **Live availability probe**: once a valid, changed candidate settles (400 ms debounce) the UI shows *checking / available / taken*; the server is authoritative and a save can still 400 (reserved word) or 409 (taken). Copy warns the handle is the public profile URL and the old one stops working with no redirect. This is the one field with asynchronous inline state — a variant must keep that legible.
- **Bio** — free text, ≤ 160 **runes** (code points, so emoji count as one), with a live `{count}/160` counter; empty clears it. "A short blurb shown on your public profile."
- **Height** — a number shown in the user's familiar unit: **inches** when their distance preference is miles, **centimeters** when kilometers. Stored canonically as `height_cm`; convert at the edge (1 in = 2.54 cm). Nullable — empty clears it and the helper reads "No height set."
- **Avatar** — a circular preview (the resolved image, or **initials** derived from the display name when none), a Change/Upload action (a hidden file input; PNG/JPG/WebP ≤ 2 MB, guarded client-side, authoritative server-side), and a Remove action shown only when an avatar is set.

**Usage**

- **Daily AI allowance** — **read-only.** A progress bar of `percentUsed` with a "Resets in Xh Ym" countdown (re-rendered on a 60 s tick). Threshold colors: accent/violet 0–79 %, **amber ≥ 80 %**, **red at 100 %**; at 100 % the subtext flips to "You've used your daily AI allowance. New allowance available in …". It's a status readout, not a control — variants must not make it look editable.

**Units**

- **Distance** — segmented Miles / Kilometers. Drives the Running views *and* which unit Height is entered in. Backed by `DistanceUnitContext` (localStorage), saves instantly.
- **Weight** — segmented Pounds / Kilograms. Bodyweight and workout volume. On the profile, optimistic save.

**Google Calendar**

- **Calendar sync** — a connection row with three states: *checking* (on mount), *connected* (shows a Disconnect button that revokes server-side), and *absent/revoked* (shows a Connect button that navigates the browser to the API's OAuth connect endpoint and back to `/settings`).
- **Default event detail** — segmented Time block / Full agenda. What a synced event shows for a planned workout. Optimistic save.

Realistic fixture for the mockups (mirrors `ResolvedProfile` + the usage and calendar snapshots — static is fine and preferred; wire nothing live):

```ts
const profile = {
  display_name: "Jimmy",
  username: "jwall317",          // @handle; may be null
  bio: "Bigggggg hybrid athlete", // ≤ 160 runes
  height_cm: 175.3,               // shown as 69 in (miles user)
  avatar_url: "/fixtures/avatar-bear.png", // or null → initials "J"
  weight_unit: "lb",              // "lb" | "kg"
  distance_unit: "mi",            // "mi" | "km"  → height in inches
  calendar_default_detail: "time_block", // "time_block" | "full_agenda"
};
const usage = { percentUsed: 43, resetsInLabel: "41m", capped: false }; // amber ≥80, red at 100
const calendar = { status: "connected" }; // "connected" | "absent" | "checking"
```

**States every variant must look intentional in:** profile **loading** (controls disabled, no layout jump); a field **dirty → saving → saved**, expressed in that variant's own save model; username **checking / available / taken** and a **validation error** (empty name, bad handle, taken handle); bio **at the 160 cap**; height **unset** ("No height set"); avatar **empty** (initials) vs. set (Change + Remove); calendar **connected vs. absent**; usage **near and at the cap** (amber/red, capped sentence); and a **narrow / mobile** width where the chosen layout must reflow, not break.

## Idioms

Five genuinely distinct directions. All are **in-system** per [`../design-system.md`](../design-system.md): the dark slate ramp, the violet accent (`#8b7cf6`), Nunito as the base family (its scoped condensed athletic display available where noted), rounded soft form, the macro/activity tints reserved for their data. They diverge on **type scale**, **color logic**, **spacing rhythm**, **composition**, **and — uniquely for this surface — save model**: each idiom proposes one internally-consistent answer to how an edit commits, replacing today's per-field-button-beside-silent-toggle split. The fixed points are the field set, the validation/availability semantics, and that the usage bar stays an unmistakably read-only status element.

- **linear-minimal** — Linear's settings calm. **Drop the cards entirely**: the whole page becomes one quiet form, settings as `label`-left / `control`-right rows separated by **hairlines, not boxes**, under small-caps section headers. **Save model: auto-save on blur / debounce**, with a tiny "Saved ✓" that fades in beside the row and clears — no buttons anywhere (the toggles already work this way, so the whole page finally matches them). Type: small, restrained, hierarchy by **weight and spacing**, not size. Color: near-monochrome slate; violet only on focus rings and active toggles. Spacing: tight, even row rhythm, generous left-to-right gutter. The disciplined baseline — the page stops looking like a stack and starts looking like a single instrument.

- **grouped-savebar** — Vercel / GitHub account settings. **Keep the sectioned cards** (they group well) but **kill every per-field button**: edits accumulate in local state, and a single **sticky "Save changes" bar** slides up from the bottom the moment anything is dirty, with a **Discard** affordance and a count ("2 unsaved changes"). Validation and the username probe gate the bar, not individual rows. Type: mid form-label scale, comfortable. Color: neutral cards; **violet concentrated on the one save bar** as the page's single call to action. Spacing: roomy card grouping with clear section breaks. The "explicit commit, done once" answer — one button for the whole page instead of one per field.

- **two-pane-nav** — Stripe / Notion settings shell. A **left sub-nav rail** (Profile · Usage · Units · Calendar) drives a **right detail pane** that shows one group at a time, so the page scans in a glance and **scales as settings grow** instead of becoming a longer scroll. **Save model: per-control instant auto-save** with a small inline confirmation, matching the toggles. Type: mid scale, strong rail section labels. Color: the **active rail item is an accent-soft pill with an accent-line border** — the exact app-nav language, so settings feels like a room inside the app. Spacing: structured two-pane with a generous right pane; collapses to a **top tab strip over a single pane** on narrow widths. The most forward-looking for a settings area that will accrete options.

- **athletic-identity** — Whoop's stat-forward identity, in our voice. **Profile is reframed as a hero identity card** that reads like the top of the user's own public profile: a **large avatar**, the display name in the **scoped condensed athletic display face**, the `@handle` beneath, the bio, and **height as a stat chip** — and the **daily allowance rendered as a bold stat ring / dial** rather than a thin bar. **Save model: per-section "Edit" mode** — a section is read-only display until you hit Edit, which flips its fields editable with **Save / Cancel** for the whole group (so identity isn't a perpetual row of inputs). Type: condensed display for the identity and the big allowance numeral, Nunito for everything else. Color: **violet used confidently** as a primary, the allowance ring carrying the amber/red threshold. Spacing: punchy, card-forward, deliberate section gaps. The most brand-forward — "this is *my* athlete profile," not a form.

- **conversational-inline** — the product's chat-led character applied to settings. The page reads as a short set of **plain-language statements with the editable value inline and emphasized**: "Your coach calls you **Jimmy**." · "You're **@jwall317**." · "You measure distance in **miles** and weight in **pounds**." · "You're **175 cm** tall." **Save model: click a bold value to edit it in place**, it commits on confirm (and the username's available/taken state resolves right under the word) — the least form-like commit of the five. Type: comfortable reading scale, sentence rhythm, not a label/field grid. Color: **violet marks every editable value**, so what's changeable is obvious without a single visible input at rest. Spacing: roomy, line-by-line, the calmest page. Distinctive and on-brand for a product you *talk* to — the risk it has to solve is making avatar and the read-only allowance sit naturally among prose, not feel bolted on.

## References

- **Linear** — its **boxless, hairline-divided settings form** and auto-save calm: settings as quiet rows under small section headers, one restrained accent. Drives `linear-minimal`.
- **Vercel** — its **dirty-state sticky save bar** ("You have unsaved changes" → Save / Discard) over grouped account cards, with the accent reserved for that bar. Drives `grouped-savebar` (GitHub's settings cards reinforce the grouping).
- **Stripe** — its **left-rail + detail-pane settings shell** that scales to many groups and scans at a glance. Drives `two-pane-nav` (Notion's settings modal reinforces the rail-driven pane).
- **Whoop** — its **stat-forward identity and bold dial/ring numerals on charcoal**, a single confident accent. Drives `athletic-identity`'s hero card and allowance ring.
- **Notion** — its **inline, click-the-value-to-edit** property editing where nothing looks like a form until you engage it. Drives `conversational-inline` (ChatGPT's plain-language register reinforces the voice).

## Selection criteria

A note-to-self for the pick, not a rubric the worker optimizes against. When I compare these I'm weighing:

- **Does it actually kill the repetition?** The whole reason for this DX is the column of per-field Save buttons sitting next to silent toggles. Whichever I pick, that inconsistency has to be gone and the page has to commit edits *one* coherent way.
- **Does the save model feel trustworthy?** Auto-save is clean but must never lose an edit silently or fire mid-validation; a save bar must never let me navigate away thinking I saved when I didn't. The model I choose has to make "is this saved?" unambiguous.
- **Does the username probe still read clearly?** It's the one field with live async state (checking / available / taken / reserved / 409). A layout that mangles that feedback — or an inline-edit model that hides it — loses something real.
- **Does the usage bar stay unmistakably read-only?** It must not look like a control I can drag or set, in any layout — including `athletic-identity`'s ring.
- **Does it stay native to the app?** I land here from the dark slate/violet shell; settings shouldn't feel like a different product. In-system is the point.
- **Does it scale as settings grow?** Calendar and units already arrived after profile; more will come. The layout I pick should still read well at twice the field count, not just today's.
- **Is it intentional, not a stock settings template?** Plenty of these read as a generic form. I want it to feel like *ours* — with the athletic or conversational register available if it earns its place.

---

> **Lifecycle.** `status:` is editorial — the owner is the dispatch gate. It moves `draft` → `exploring` (worker running) → `awaiting_selection` (draft PR open, owner deciding) → `selected` / `abandoned`.
>
> **Handoff.** This DX ends at a *chosen variant*, not merged code. I open the draft `[DX — DO NOT MERGE]` PR, compare the variants on the preview deploy, pick one, tick its box, set `status: selected` (noting the winning idiom), and **close the PR — never merge it.** Then I open a SOW: *"implement the settings page per the `<chosen-idiom>` variant from `dx/settings`, production-quality, conforming to the design system."*
