# Prog Strength Design System

**Status**: v0.3 · **Last updated**: 2026-06-18

> Seeded from the first design explorations (calendar, chat & app shell, timeline).
> This is the **canonical record of decided visual conventions** — start small,
> grow as DXs land. Its job is to stop already-decided choices from being
> re-litigated on every new exploration.

## How this is used

- **DX `scope: in-system`** → every variant **uses the tokens below**. Variants
  diverge only on *layout, structure, density, and composition* — **not** on
  palette, accent, or type family. (That's how a DX stays a useful spread
  without re-deciding the foundation.)
- **DX `scope: greenfield`** → may explore beyond the system, but the **Fixed
  Points** below still hold.
- **SOWs** → conform. Reference these tokens; never hard-code raw hex that
  duplicates a token.
- **Implementation**: the tokens are implemented in `prog-strength-web` (CSS
  variables / Tailwind theme), introduced by
  [`sows/chat-and-app-shell-redesign.md`](sows/chat-and-app-shell-redesign.md).
  **This doc is the source of truth; the web tokens implement it.**

## Fixed Points (inviolable — hold even for `greenfield`)

- The **Prog Strength brand**: the "P" mark and the product name.
- **Dark theme** for all app surfaces. (A light theme was explicitly rejected
  for in-app surfaces during the calendar and chat explorations.)

## Decided

### Theme

Dark, app-wide. Surfaces are a soft near-black ramp (below), never pure black or
a light background.

### Color

- **Neutrals — soft near-black ramp**: base `#0e0f12`, surface `#15171b`, raised
  `#191c21`, raised-2 `#1e2127`; text `#c8cad0`, muted `#7d818c`, faint
  `#565a63`; hairline borders `rgba(255,255,255,0.06)`–`0.10`.
- **Accent — desaturated periwinkle**: `#9aa6d6` (dark `#8490c4`, soft
  `rgba(154,166,214,0.14)`, line `rgba(154,166,214,0.30)`). **Replaces the
  violet.** It is the primary action / active-state color (active/selected
  states, the user's own emphasis, e.g. their chat bubbles) **and** the lifting
  domain hue.
- **Dual domain accents** — the canonical activity hues: **periwinkle `#9aa6d6`
  = lift**, **sage `#7fae9e` = run** (sage soft `rgba(127,174,158,0.14)`, line
  `rgba(127,174,158,0.30)`). These reconcile with the `--discipline-*` tokens:
  lift → periwinkle, run → sage; **mobility/core stay reserved** for later. Never
  saturated category-loud colors.
- **Status — desaturated**: success `#86b39f`, danger `#c79292`, warning
  `#d6b87f` (amber re-toned to sit calmly on the new field).
- **Macro tints**: protein `#34d399` (green), fat `#fbbf24` (amber), carb
  `#60a5fa` (blue), each on a ~13%-alpha background. For nutrition macro chips.
  **Unchanged** — they await nutrition's own migration SOW.

### Typography

- **Primary**: **Manrope** (fine, even geometric sans), a comfortable, exact
  scale — nothing shouts.
- **Tight numeric tracking**: numerals and big stat numbers carry `-0.03em`
  tracking, so the figures sit precise and settled.
- **No display accent**: the Oswald display accent is **dropped** — the calm
  idiom is deliberately uniform-Manrope with no display face. `Geist_Mono` is
  kept for incidental mono.

### Form & depth

- **Radii**: a calmer **14px panel radius** is the default on cards and panels;
  full-pill radius on chips and buttons.
- **Depth**: **hairline borders as the default**, not hard lines; soft shadows
  for gentle elevation only where needed.
- **Nav**: rounded pills; active = accent-soft fill + accent-line border +
  accent-colored glyph.

### Form controls

The calendar plan edit form is the first real, interactive form-control set in
the app; its treatment is the language every future form should adopt. Tokens
first, descriptive (a reusable component primitive is a later DS work item — see
Open). Provenance: [`sows/calendar-view-refresh.md`](sows/calendar-view-refresh.md).
The spec below still holds under v0.3 — it is **re-toned** (the soft near-black
field, periwinkle accent, and the 14px panel default), not rewritten.

- **Field surface** — text/number inputs, textareas, and selects share one
  rounded slate surface: `--surface-2` fill with a **hairline `--border`**
  (a real border, not transparent), comfortable padding, `--foreground` text and
  `--muted` placeholder. Radius `rounded-lg` for full-width fields, `rounded-md`
  for dense grid cells.
- **Focus** — an accent ring, never the browser default: `outline-none` +
  border to `--accent` + a 1px `--accent-line` ring. Consistent across inputs,
  selects, popover triggers, and the toggles.
- **Labels** — quiet field labels above the control: uppercase, ~`text-[11px]`,
  `font-semibold`, `tracking-wide`, in `--faint`. They read as metadata, not as
  headings.
- **Segmented toggle** — a binary/enum choice (e.g. Lift/Run, run type) is a
  **full-pill segmented control**: a `--surface` track with a hairline border;
  the active segment is an `--accent` fill with `--accent-fg` text; inactive
  segments are `--muted`, brightening to `--foreground` on hover.
- **Grouping / superset** — related rows bound into a group are marked with an
  **`--accent-line` left-bar** (`border-l-2`) and a small uppercase `--accent`
  group label ("Superset"). The same accent-line idiom marks any
  "these belong together" grouping in a form.
- **Primary vs secondary actions** — the form's primary action (Save / Plan /
  Start) is a solid `--accent` pill with `--accent-fg`; secondary (Cancel /
  Close) is a `--surface-2` pill with a hairline border. Disabled drops opacity.

### Component patterns (descriptive — grows as surfaces land)

- **Cards / panels**: 14px-radius panel surface, hairline border.
- **Chips / pills**: rounded-full — stat chips, tool-call chips, macro chips.
- **Message bubbles**: user = periwinkle accent; assistant = raised surface.
- **List / rail items**: rounded rows; active = accent-soft.

### Voice

Coaching tone. Consistency and streak framing are a first-class product element
(a greeting, "you trained N of M days", encouraging — never scolding —
microcopy), not decoration.

### Reactions

The four reaction types are canonical: 👍 like · 💪 strong · 🔥 fire ·
🎉 celebrate. Don't reduce them to a single "kudos" or rename them; restyling
their presentation is fine.

## Open / not yet decided

State things honestly so a `greenfield` DX knows where it has room.

- A formal spacing scale and elevation levels — to be codified as more surfaces
  land. (Form-control specs are now decided — see **Decided → Form controls**;
  a reusable component primitive for them is still open, recommended once a
  second form surface lands.)
- Empty / loading / error patterns as reusable primitives.
- Data-viz conventions (radar, route maps, sparklines) beyond their first use.
- **Mobile** (`prog-strength-mobile`) — this doc is web-first today; mobile
  adoption is a later decision.

## Provenance

- **Calendar** — warm-organic *structure and coaching feel*, re-toned dark and
  conformed to these tokens (its original clay accent yields to violet):
  [`sows/calendar-view-redesign.md`](sows/calendar-view-redesign.md) ·
  [`dx/calendar-view.md`](dx/calendar-view.md).
- **App-wide foundation tokens** (slate / violet / Nunito / rounded soft form):
  [`sows/chat-and-app-shell-redesign.md`](sows/chat-and-app-shell-redesign.md) ·
  [`dx/chat-and-app-shell.md`](dx/chat-and-app-shell.md).
- **Timeline** — athletic display character, conformed to these tokens:
  [`sows/timeline-redesign.md`](sows/timeline-redesign.md) ·
  [`dx/timeline.md`](dx/timeline.md).
- **Form controls** — the input / select / segmented-toggle / superset-grouping
  specs, established by the calendar plan edit form and codified here:
  [`sows/calendar-view-refresh.md`](sows/calendar-view-refresh.md) ·
  [`dx/calendar-view-refresh.md`](dx/calendar-view-refresh.md) (`true-month-grid`).
- **Activities** — the oura-calm-minimal re-tone (soft near-black ramp,
  periwinkle + sage dual accents, Manrope, 14px hairline panels):
  [`sows/activities-page-redesign.md`](sows/activities-page-redesign.md) ·
  [`dx/activities-page.md`](dx/activities-page.md) (`oura-calm-minimal`).

## Changelog

- **v0.3** (2026-06-18) — re-toned to oura-calm-minimal (provenance: [`sows/activities-page-redesign.md`](sows/activities-page-redesign.md)): soft near-black neutral ramp replaces slate; desaturated periwinkle accent replaces violet; documented the dual domain accents (periwinkle = lift, sage = run); desaturated status colors; Manrope replaces Nunito as the primary family and the Oswald display accent is dropped; 14px hairline panels as the default form. Fixed Points (the "P" mark, the dark theme) hold; macro tints unchanged pending nutrition's migration.
- **v0.2** (2026-06-17) — added **Form controls** under Decided (rounded slate
  field surface + hairline, uppercase faint labels, accent focus ring, the pill
  segmented toggle, and the accent-line grouping/superset treatment), from the
  calendar plan edit form ([`sows/calendar-view-refresh.md`](sows/calendar-view-refresh.md));
  removed the corresponding "not yet decided" item.
- **v0.1** (2026-06-16) — seeded from the first three explorations: dark theme,
  the slate ramp, the violet accent (replacing blue), Nunito + a scoped athletic
  display, rounded soft form, the macro tints and activity tonal hues, the
  coaching voice, and the four reactions.
