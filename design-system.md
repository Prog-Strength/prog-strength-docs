# Prog Strength Design System

**Status**: v0.4.2 · **Last updated**: 2026-06-29

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
  violet.** One accent only — the primary action / active-state color
  (active/selected states, the user's own emphasis, e.g. their chat bubbles). It
  is **app chrome, not an activity hue**; activity discipline colors stay distinct
  (see Activity tonal hues below).
- **Status — desaturated**: success `#86b39f`, danger `#c79292`, warning
  `#d6b87f` (amber re-toned to sit calmly on the new field).
- **Macro tints**: protein `#34d399` (green), fat `#fbbf24` (amber), carb
  `#60a5fa` (blue), each on a ~13%-alpha background. For nutrition macro chips.
  **Unchanged** — they await nutrition's own migration SOW.
- **Activity tonal hues**: per-discipline desaturated hues — **run** and **lift**
  today, the system **extensible** to mobility/core later — toned to sit on the
  dark near-black surface and **kept distinct from the periwinkle accent** (an
  activity must never read as selection / "today"). Each discipline has a
  three-token set — `bg` (tint fill), `fg` (text), `dot` (accent: left bar / pill
  border / focus ring):

  | Discipline | `--discipline-*-bg` | `--discipline-*-fg` | `--discipline-*-dot` |
  | --- | --- | --- | --- |
  | **run** (green-teal) | `#16302a` | `#82d3b8` | `#46b893` |
  | **lift** (steel-blue) | `#1a2a3c` | `#8cbce8` | `#5598d8` |
  | mobility *(reserved)* | `#1f3330` | `#8fd6c4` | `#4fbfa3` |
  | core *(reserved)* | `#2c2440` | `#c2adf0` | `#9a7fe0` |

  **Activity type owns activity color** — a session reads in its discipline hue
  on every surface (month-grid pill, timeline rail), and lifecycle *status*
  (planned / completed / skipped) is carried by **shape + badge**, never by the
  color slot (no green-for-completed, no periwinkle-for-planned — the accent
  stays app-chrome/selection only). The mapping lives behind a single resolver
  (`lib/activity-colors.ts` → `activityColors(type)`) so a future
  user-customizable palette is a one-place change.

- **Lift intensity ramp**: a four-stop ramp of the **lift** steel-blue hue —
  the **canonical encoding of graded intensity**, for any graphic that shades a
  region by how hard it was worked. It interpolates from dim (a tier just above
  the unworked near-black silhouette) to bright, where the top tier **is**
  `--discipline-lift-fg`, so "harder-worked = more saturated steel-blue":

  | Token | Value | Tier |
  | --- | --- | --- |
  | `--discipline-lift-1` | `#39405a` | 1 — lightly worked |
  | `--discipline-lift-2` | `#5a6493` | 2 |
  | `--discipline-lift-3` | `#7d88c2` | 3 |
  | `--discipline-lift-4` | `#aab4dd` | 4 — hardest worked (= `--discipline-lift-fg`) |

  First use: the **workout detail muscle body-map** (front/back silhouette,
  worked regions shaded by set volume) — provenance
  [`sows/workout-detail-refinements.md`](sows/workout-detail-refinements.md).
  Reusable by any future per-intensity graphic (a heatmap, a load-distribution
  bar). The ramp is **additive to the existing `--discipline-lift-*` triplet**
  (`bg` / `fg` / `dot`), not a replacement; it lives alongside it in
  `app/globals.css` (with `--color-discipline-lift-1..4` Tailwind aliases).

- **HR-zone scale**: the five `--zone-1..5` tokens are a **separable calm
  five-tone scale** — `--zone-1` `#6b7280` (slate), `--zone-2` `#6f93c2`
  (dusty blue), `--zone-3` `#6fb39a` (muted teal-green), `--zone-4` `#d0a86e`
  (muted amber), `--zone-5` `#cc8077` (dusty coral). It is **cool→warm**
  (low HR intensity reads cool, max reads warm — the Garmin mental model
  without the saturated stoplight) with each hue held at the **status-tone
  saturation register**, so zones stay legible against each other and native to
  oura-calm. The fills encode *intensity, not value judgment* (coral = hottest
  zone, not "bad") and are reused for the **saturation register only** — they
  do not alias `--success`/`--danger`/`--warning`, and **no zone is the
  accent** (zones never read as selection). Consumed only by the running-detail
  HR-zones widget. This **supersedes the periwinkle single-hue ramp** (whose
  steps were near-indistinguishable and whose Z5 collided with the accent) —
  provenance
  [`sows/running-heart-rate-zones-ranked-bars.md`](sows/running-heart-rate-zones-ranked-bars.md).

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
  periwinkle accent, Manrope, 14px hairline panels):
  [`sows/activities-page-redesign.md`](sows/activities-page-redesign.md) ·
  [`dx/activities-page.md`](dx/activities-page.md) (`oura-calm-minimal`).
- **Activity palette** — the enumerated `--discipline-*` token sets (run / lift
  live, mobility / core reserved) and the "activity type owns the color, status
  is shape + badge" rule, centralized behind `activityColors()`:
  [`sows/calendar-event-detail-and-activity-colors.md`](sows/calendar-event-detail-and-activity-colors.md).
- **HR-zone scale** — the separable cool→warm `--zone-1..5` five-tone scale
  (status-tone saturation, distinct from the accent), superseding the
  periwinkle ramp, alongside the running-detail HR-zones widget's rebuild into
  ranked per-zone bars:
  [`sows/running-heart-rate-zones-ranked-bars.md`](sows/running-heart-rate-zones-ranked-bars.md).

## Changelog

- **v0.4.2** (2026-06-29) — retuned the HR-zone scale (`--zone-1..5`) from the
  near-identical periwinkle single-hue ramp to a **separable cool→warm
  five-tone scale** (slate → dusty blue → muted teal-green → muted amber →
  dusty coral) at the status-tone saturation register, distinct from the accent
  (the old Z5 == `--accent` collision is retired). Calm and dark-native — not a
  green→red heat map — so zones stay legible at small fill widths. Consumed only
  by the running-detail HR-zones widget, now rendered as ranked per-zone bars
  ([`sows/running-heart-rate-zones-ranked-bars.md`](sows/running-heart-rate-zones-ranked-bars.md)).
  No re-tone of the v0.4 foundation.
- **v0.4.1** (2026-06-19) — added the four-stop **lift intensity ramp**
  (`--discipline-lift-1..4`, dim → bright steel-blue, tier 4 = `--discipline-lift-fg`)
  as the canonical encoding of graded intensity, additive to the existing
  `--discipline-lift-*` triplet. First use: the workout-detail muscle body-map
  ([`sows/workout-detail-refinements.md`](sows/workout-detail-refinements.md)).
  No re-tone of the v0.4 foundation.
- **v0.4** (2026-06-18) — re-toned to oura-calm-minimal (provenance: [`sows/activities-page-redesign.md`](sows/activities-page-redesign.md)): soft near-black neutral ramp replaces slate; desaturated periwinkle accent replaces violet; desaturated status colors; Manrope replaces Nunito as the primary family and the Oswald display accent is dropped; 14px hairline panels as the default form. The `--discipline-*` activity hues (run green-teal / lift steel-blue) are **kept distinct from the accent** per v0.3's "activity ≠ selection" rule — they are *not* re-toned to the accent hue. Fixed Points (the "P" mark, the dark theme) hold; macro tints unchanged pending nutrition's migration.
- **v0.3** (2026-06-18) — enumerated the `--discipline-*` activity palette
  (run / lift token sets with values; mobility / core reserved) and recorded the
  rule that **activity type owns the activity color** on every surface while
  lifecycle status is conveyed by shape + badge (retiring green-for-completed
  and violet-for-planned), centralized behind `activityColors()`
  ([`sows/calendar-event-detail-and-activity-colors.md`](sows/calendar-event-detail-and-activity-colors.md)).
- **v0.2** (2026-06-17) — added **Form controls** under Decided (rounded slate
  field surface + hairline, uppercase faint labels, accent focus ring, the pill
  segmented toggle, and the accent-line grouping/superset treatment), from the
  calendar plan edit form ([`sows/calendar-view-refresh.md`](sows/calendar-view-refresh.md));
  removed the corresponding "not yet decided" item.
- **v0.1** (2026-06-16) — seeded from the first three explorations: dark theme,
  the slate ramp, the violet accent (replacing blue), Nunito + a scoped athletic
  display, rounded soft form, the macro tints and activity tonal hues, the
  coaching voice, and the four reactions.
