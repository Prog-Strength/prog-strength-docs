# Prog Strength Design System

**Status**: v0.1 · **Last updated**: 2026-06-16

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

Dark, app-wide. Surfaces are a slate ramp (below), never pure black or a light
background.

### Color

- **Neutrals — slate ramp**: base `#15171c`, panel/rail `#1e2128`, raised
  `#272b33`, raised-2 `#2e333c`; text `#eef0f4`, muted `#9aa1ad`, faint
  `#6b7280`; hairline borders `rgba(255,255,255,0.07)`–`0.10`.
- **Accent — violet**: `#8b7cf6` (dark `#7765ec`, soft `rgba(139,124,246,0.14)`,
  line `rgba(139,124,246,0.30)`). **Replaces the legacy blue.** One accent only;
  reserved for primary actions, active/selected states, and the user's own
  emphasis (e.g. their chat bubbles).
- **Macro tints**: protein `#34d399` (green), fat `#fbbf24` (amber), carb
  `#60a5fa` (blue), each on a ~13%-alpha background. For nutrition macro chips.
- **Activity tonal hues**: per-discipline desaturated hues — **run** and **lift**
  today, the system **extensible** to mobility/core later — re-toned to sit on
  the dark slate surface. Never saturated category-loud colors.

### Typography

- **Primary**: **Nunito** (rounded humanist sans), weights 600 / 700 / 800,
  a comfortable mid-contrast scale.
- **Scoped display accent**: an athletic condensed display face (Oswald-style)
  is permitted on specific *athletic* surfaces (e.g. the timeline feed's titles
  and big stat numerals) as a **display accent only**, not the base family.

### Form & depth

- **Radii**: `rounded-2xl`/`rounded-3xl` on cards and panels; full-pill radius
  on chips and buttons.
- **Depth**: soft shadows for gentle elevation; hairline borders, not hard lines.
- **Nav**: rounded pills; active = accent-soft fill + accent-line border +
  accent-colored glyph.

### Component patterns (descriptive — grows as surfaces land)

- **Cards / panels**: rounded, slate panel surface, hairline border.
- **Chips / pills**: rounded-full — stat chips, tool-call chips, macro chips.
- **Message bubbles**: user = violet accent; assistant = raised slate.
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

- Whether the athletic condensed display generalizes beyond the timeline.
- A formal spacing scale, elevation levels, and form-control (input/select/toggle)
  specs — to be codified as more surfaces land.
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

## Changelog

- **v0.1** (2026-06-16) — seeded from the first three explorations: dark theme,
  the slate ramp, the violet accent (replacing blue), Nunito + a scoped athletic
  display, rounded soft form, the macro tints and activity tonal hues, the
  coaching voice, and the four reactions.
