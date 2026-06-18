---
type: sow
status: shipped
repos:
  - prog-strength-web
  - prog-strength-docs
---

# Activities Page Redesign & System Re-tone — Oura-Calm-Minimal

**Status**: Shipped · **Last updated**: 2026-06-18

> Frontend SOW. It implements a chosen DX variant and therefore inherits that DX's
> `scope` — here **`greenfield`** — and, because the owner chose to **adopt the
> variant's visual language app-wide**, it also re-tones the design system itself.

## Introduction

`/activities` is the analytics home of Prog Strength — the surface a lifter opens
to answer "how's my training going?" across **lifting**, **running**, and
**steps**. It works today (`app/(app)/activities/page.tsx`: a period filter, four
`?view=` tabs, and the `ActivitiesOverviewView` / `WorkoutsView` / `RunningView` /
`StepsView` components, each with its stat row, trend chart, and history list).
But it reads like a generic dark-mode admin dashboard: rows of identical glassy
stat cards, two area charts that look the same whether they plot hours or miles,
and three tabs that feel like three mini-apps sharing a tab bar. It has no point
of view — exactly the wrong thing for the page meant to make a lifter *feel* their
progress.

The Design Exploration `dx/activities-page.md` (PR Prog-Strength/prog-strength-web#74)
explored that surface across five **greenfield** variants — free to leave the
app's slate/violet/Nunito system, with only the "P" mark and the dark theme
fixed — each rendering all four tabs over a representative "Last 30 days" fixture
(a rest-week trough, 🏆 PRs, an over- and an under-goal step day). The owner
selected **`oura-calm-minimal`**: training as a *calm instrument* — a desaturated,
low-contrast world drawing on **Oura / Linear**, with fine **Manrope** type, a
soft near-black field, hairline panels, and delicate single-stroke (1px) line
charts instead of competing filled areas.

Because this was a greenfield pick **adopted app-wide** (the owner's call), this
SOW does two things:

1. **Re-tones the design system** to the oura-calm-minimal language — updating
   [`../design-system.md`](../design-system.md) and the web **token layer**
   (`app/globals.css` + fonts) so the whole app inherits the new palette, accent,
   and type through the existing semantic tokens.
2. **Rebuilds `/activities`** production-quality as the **first fully-realized
   surface** in the new language — all four tabs, over the **existing** data layer,
   with **no backend change**.

Other surfaces (nutrition, calendar, chat, …) inherit the new tokens automatically
and stay functional, but their bespoke compositions are **not** redesigned here —
each migrates in its own follow-up SOW. `prog-strength-api` is not touched.

## Proposed Solution

Two coordinated changes, one shippable PR per repo.

**Part 1 — System re-tone (the token layer is the lever).** The app already
consumes *semantic* tokens (`var(--accent)`, `var(--surface)`, `var(--foreground)`,
the font variables) defined once in `app/globals.css` and exposed to Tailwind via
`@theme inline`. So adopting oura app-wide is a **values-only** change: re-point
the existing token *values* to oura's palette and swap the base font to Manrope,
**keeping every token name stable** so all current consumers inherit the new look
with no per-file edits. `design-system.md` is updated in lockstep (it's the source
of truth the dispatch worker and future SOWs conform to). The variant's two domain
accents — periwinkle for lifting, sage for running — fold into the existing
per-discipline tonal-hue tokens, and desaturated positive/negative greens/reds
replace the loud status colors. This is a re-tone, not a re-architecture: names,
structure, and the dark theme hold.

**Part 2 — Activities rebuild.** The variant on the DX branch
(`app/design-explore/activities-page/_variants/oura.tsx` + `_fixtures.ts`) is the
**visual spec**, not code to promote — it's reimplemented production-quality
against the real route, wired to the real data instead of fixtures. Each of the
four views is restyled into the calm-instrument language: hairline panels (14px
radius, soft near-black), 11px uppercase letter-spaced section labels in the faint
tone, big quiet numerals with tight numeric tracking, and **delicate single-stroke
line charts** (the signature move — the Overview chart becomes two 1px lines,
periwinkle lifting + sage running, legible where they cross, not two fighting area
fills). All behavior is preserved: the period filter, `?view=` tabs, per-week and
per-month grouping, the per-tab actions (Start live workout, Upload TCX, Log
steps), and the step-log edit/delete.

## Goals and Non-Goals

### Goals

- **Re-tone the web token layer** (`app/globals.css` `:root` + `@theme inline`) to
  oura-calm-minimal, **changing token values, not names**, so the app inherits the
  language through existing consumers:
  - Neutrals → soft near-black field: `--background #0e0f12`, `--surface #15171b`,
    `--surface-2 #191c21`, `--surface-3` a hair above; text `--foreground #c8cad0`,
    `--muted #7d818c`, `--faint #565a63`; hairlines `--border rgba(255,255,255,0.06)`,
    `--border-strong rgba(255,255,255,0.10)`.
  - Accent → **desaturated periwinkle** `--accent #9aa6d6` (replaces violet
    `#8b7cf6`), with re-derived `--accent-dark` / `--accent-soft` / `--accent-line`;
    add a **secondary domain accent** for running, **sage** `#7fae9e`.
  - Status → desaturated: `--success #86b39f`, `--danger #c79292`; keep an amber
    `--warning` (re-toned to sit calmly on the new field).
  - Re-tone the per-discipline tonal hues to match: `--discipline-lift-*` →
    periwinkle, `--discipline-run-*` → sage (keep mobility/core reserved).
- **Swap the base type family to Manrope** via `next/font` in `app/layout.tsx`
  (replacing Nunito as the primary), with tight numeric tracking (~`-0.03em`) on
  numerals; decide the Oswald display-accent's fate (see Open Questions).
- **Promote the language into `design-system.md`** — rewrite the Color, Typography,
  and Form-&-depth sections to the oura tokens, bump the version (v0.2 → v0.3), and
  record that the activities redesign is the provenance, the way prior DS sections
  cite their originating SOW.
- **Rebuild `/activities` (all four tabs) in the new language**, over the existing
  data layer, as the reference surface:
  - **Overview** — two quiet stat tiles rows (`Total time · Sessions · Workouts ·
    Runs`; `Total volume · Total mileage · PRs · Avg session`) and the **dual
    single-stroke line chart** (Lifting periwinkle + Running sage, weekly hours,
    hover tooltip), plus the steps summary.
  - **Workouts** — `Start live workout` action; summary panel (`Total time ·
    Sessions · PRs` + metric toggle + trend line); per-week groups of session
    cards (name, 🏆 PR badge, `time · N exercises · volume`, truncated note,
    expand, edit/delete).
  - **Running** — `Upload TCX` action; KPI row (`This week` + `+/-% vs last week`
    delta, `This month · N runs`, `Avg pace (30D)`, `All time · N runs`); summary
    panel (distance/time toggle + trend line); per-month→per-week run cards
    (`Distance · Time · Pace · Avg HR`).
  - **Steps** — KPI row (`Avg · Total · Best day · Goal %`); **daily bar chart with
    a dashed goal line** (over/under-goal bars read differently); editable log
    table (`Date · Steps`, edit/delete).
- **Reuse the existing data layer unchanged** — `listWorkouts`, `listExercises`,
  `listRunningSessions`, `getRunningMetrics`, `listSteps`, `getStepsGoal`; the
  existing `?view=` routing, the `30d` default timeframe, `getMe`-driven display
  unit (lb/mi vs kg/km via `useDistanceUnit`), and the SI→imperial conversions in
  the views. **No new endpoints, no SDK changes, no API repo.**
- **Preserve every behavior and fixed point**: the period filter (`7 / 30 / 90 /
  All`) recomputing all tabs; per-week (`This week · Jun 15 – 21`) and per-month
  (`June 2026 · …`) grouping; the three per-tab actions and their modals (live
  workout entry, the TCX upload modal, step logging); workout expand and step-log
  edit/delete with their existing optimistic updates and toasts; timezone-aware
  date windows.
- **Handle every hard state** the DX called out: a **rest week** where both
  Overview series and the week groups go to zero (reads as intentional rest, not
  broken); **two overlaid series staying legible** where they cross and where one
  is zero; steps **over vs under goal** against the dashed line; a delta that is
  **positive, negative, or null** (hide when no prior week); **PR badges and long
  names/notes** that truncate; **sparse (`7 days`) vs dense (`All`)** without the
  layout breaking; and the **tab handoff** reading as one surface.
- **Keep the rest of the app functional under the new tokens** — a visual sanity
  pass over the highest-traffic surfaces (chat/app-shell, nutrition, calendar) to
  confirm they inherit the re-tone without broken contrast or unreadable text;
  fix only token-level regressions, not compositions.
- Keep web CI green (lint/format/typecheck/test/build), adding tests for the new
  activities views and the dual-series/over-goal logic.

### Non-Goals

- **Any backend change.** Every activities endpoint already exists;
  `prog-strength-api` is not in `repos:`. No schema, route, or SDK changes.
- **Redesigning other surfaces' compositions.** Nutrition, calendar, chat,
  bodyweight, progress, settings, the landing page, etc. **inherit the new tokens**
  but their bespoke layouts are migrated in **separate follow-up SOWs** — this SOW
  redesigns the *activities* composition only and re-tones the *shared tokens*.
- **A token *rename* or token-system re-architecture.** Values change, names don't;
  `@theme inline` mapping structure is untouched.
- **Re-toning the nutrition macro tints** (`--macro-protein/-fat/-carb`). They're
  domain-specific to nutrition chips and out of scope here; nutrition's migration
  SOW revisits them.
- **Changing the activities data model, endpoints, grouping, or unit logic** —
  presentation rebuild only over the existing orchestration.
- **Promoting the DX mockup code.** `oura.tsx` + `_fixtures.ts` are the visual spec,
  not code to copy; the `design-explore/activities-page` route stays gated
  (`designExploreEnabled` / `NEXT_PUBLIC_DESIGN_EXPLORE`), untouched, and never
  ships to production.
- **The other four variants** (editorial, whoop, brutalist, strava) — not built;
  the DX PR is closed, never merged.
- **A light theme or any change to the Fixed Points** (the "P" mark, the dark
  theme) — both hold.

## Implementation Details

Order follows the lifecycle: re-tone the shared foundation, then rebuild the
surface on top of it.

### Design system (`prog-strength-docs/design-system.md`)

Source of truth, updated first so the web tokens implement a written decision.
Bump **v0.2 → v0.3**. Rewrite:

- **Color** — replace the slate ramp with the soft near-black ramp; replace the
  violet accent with desaturated periwinkle `#9aa6d6` (primary) and document the
  **dual domain accents** (periwinkle = lift, sage `#7fae9e` = run) as the
  canonical activity hues; re-tone status colors. Note the macro tints are
  unchanged pending nutrition's migration.
- **Typography** — Manrope is the primary family (replacing Nunito); state the
  tight numeric tracking convention; resolve the Oswald display-accent (drop, or
  keep as a scoped accent — Open Questions).
- **Form & depth** — calmer 14px panel radius and hairline borders as the default;
  keep the pill chips/buttons and the form-control language, re-toned.
- Add a one-line provenance pointing at `sows/activities-page-redesign.md`, and
  keep the "Fixed Points hold" framing intact.

### Web token layer (`prog-strength-web`)

- **`app/globals.css`** — update the `:root` token **values** per the Goals
  (neutrals, accent + new `--accent-2`/domain hues, status, hairlines, 14px
  `--radius-card`). Keep all token **names** and the `@theme inline` mapping;
  add a mapping line for the new `--accent-2`/running token. Because consumers
  reference these vars, the app-wide re-tone lands here.
- **`app/layout.tsx`** — load **Manrope** via `next/font/google` as the base
  family (replace the `nunito` import/variable wiring); keep `Geist_Mono`; apply
  the Oswald display-accent decision (Open Questions). Ensure the body family
  resolves to Manrope.

### Activities surface (`prog-strength-web`)

Rebuild the presentation of `app/(app)/activities/page.tsx` and the four view
components into the calm-instrument language, **keeping each view's data
orchestration** (the `timeframe` state and `30d` default, `?view=` routing,
`getMe`/`useDistanceUnit` display unit, the `listWorkouts`/`listRunningSessions`/
`getRunningMetrics`/`listSteps`/`getStepsGoal` calls and their `useMemo`
aggregation/grouping). This is a presentation rebuild over existing logic.

- **Shared chrome** — restyle the **period filter** pills and the **tab bar**
  (`ToolbarButton`s) to the quiet treatment: faint inactive, a soft periwinkle
  active state; hairline-panel containers (14px radius, `--surface`, 1px
  `--border`).
- **Stat tiles** — a reusable calm tile: 11px uppercase letter-spaced label in
  `--faint`, big numeral in `--foreground` with `-0.03em` tracking; no glassy
  card, just a hairline panel. Used by every tab's KPI/stat rows.
- **Charts** (`components/activities/activities-combined-chart.tsx` and the per-tab
  trend charts) — re-render as **delicate single-stroke lines**, not filled areas.
  Overview's is the **dual series**: periwinkle (`--discipline-lift`/`--accent`)
  lifting + sage (`--discipline-run`/`--accent-2`) running, 1px strokes, a faint
  hairline grid, and the existing hover tooltip restyled. Keep both series legible
  where they cross and where a series sits at zero (the rest-week case). The
  Workouts/Running trend charts and the Steps **bar** chart (with the dashed
  `--success` goal line, over/under bars distinguished) adopt the same restraint.
- **Lists** — `WorkoutsView` session cards and `RunningView` run cards become calm
  hairline rows grouped by week/month: name, 🏆 PR badge (re-toned), the meta line
  (`time · N exercises · volume` / `Distance · Time · Pace · Avg HR`), truncated
  note, expand chevron, and **edit/delete preserved**. `StepsView`'s log table
  keeps its `Date · Steps` rows and edit/delete.
- **Actions** — `Start live workout`, `Upload TCX` (the existing
  `uploadModalOpen` modal), and `Log steps` keep their flows; restyle only.

The DX `_fixtures.ts` (the representative-period numbers and the session/run/step
shapes) is the source of the visual spec and the test fixtures; reimplement against
the real `Workout` / `RunningSession` / `RunningMetrics` / `StepsEntry` / `StepsGoal`
types from `lib/api.ts`, not by importing from the gated `design-explore` tree.

### Tests

- **Activities views** — the dual-series Overview chart renders **both** Lifting
  and Running series and a tooltip from a known weekly fixture, including the
  **rest-week zero** case; stat tiles render the right numerals per timeframe; the
  Running delta renders **up (green), down (red), and hidden (null)**; the Steps
  bars render **over vs under** the goal line distinctly; workout cards render the
  🏆 badge and truncate long notes; **step-log edit/delete still fire** their
  mutations and optimistically update. Reuse the existing API/toast mocks.
- **Token sanity** — a light check that the re-toned tokens resolve (no dangling
  `var(--…)`), plus a manual preview pass over chat/nutrition/calendar for contrast
  regressions. lint/format/typecheck/test/build green.

## Rollout

1. **`prog-strength-docs`** — `design-system.md` v0.3 re-tone; flip
   `dx/activities-page.md` to `status: selected` (idiom `oura-calm-minimal`); mark
   this SOW shipped on merge.
2. **`prog-strength-web`** — the token re-tone + Manrope swap + activities rebuild,
   in one PR. Vercel preview to verify the activities surface *and* that the rest
   of the app inherits the new language without broken contrast.

### Verification after rollout

- `/activities` (signed in): a calm, low-contrast instrument — quiet stat tiles,
  delicate single-stroke charts, hairline panels, Manrope numerals — across all
  four tabs, reading as **one** surface rather than three dashboards.
- The **Overview dual chart** shows both Lifting (periwinkle) and Running (sage)
  as 1px lines, legible where they cross and where the rest-week sends them to
  zero; the tooltip works.
- **Steps** over-goal vs under-goal bars read differently against the dashed goal
  line; the **Running delta** shows correctly for an up week, a down week, and a
  no-prior-week (hidden).
- Period filter, `?view=` tabs, per-week/month grouping, **Start live workout**,
  **Upload TCX**, **Log steps**, workout expand, and step-log **edit/delete** all
  still work and persist against the real API; the display unit (lb/mi vs kg/km)
  is respected.
- The rest of the app (chat, nutrition, calendar) has shifted to the new
  palette/type via the shared tokens and remains **readable and functional** —
  no broken contrast — even though those compositions aren't redesigned yet.
- `design-system.md` reads v0.3 with the oura tokens as the decided language; the
  `design-explore/activities-page` route remains gated/404 in production; no DX
  mockup code shipped.

## Open Questions

1. **Low-contrast `--foreground` app-wide.** Oura's body text is `#c8cad0` —
   noticeably lower contrast than today's `#eef0f4`. On the calm activities surface
   it's the point, but app-wide it touches dense text (chat, nutrition log) and
   accessibility. **Lean:** adopt `#c8cad0` as the new `--foreground` for the calm
   language, but verify WCAG AA on small text during the preview pass and nudge it
   brighter only if a real surface fails. Confirm on preview.
2. **Oswald display accent.** The system currently permits an Oswald condensed
   display face on athletic surfaces; oura-calm-minimal is deliberately
   uniform-Manrope with no display face. **Lean:** drop the Oswald display accent
   from the system (it fights the calm instrument idiom); keep `Geist_Mono` for
   incidental mono. Flag so a reviewer can veto before the font wiring changes.
3. **Single vs dual primary accent.** Oura uses periwinkle (lift) and sage (run) as
   peer domain hues, with no single loud "primary action" color. **Lean:** make
   periwinkle the global `--accent` (primary actions/active states) and sage the
   running/secondary domain hue; reconcile both with the existing `--discipline-*`
   tokens rather than adding a parallel system.
4. **Blast radius of the app-wide re-tone in one PR.** Re-toning shared tokens
   shifts every surface at once. **Lean:** ship the token re-tone + activities
   rebuild together (the tokens are the whole point of "app-wide") and rely on the
   preview sanity pass + the per-surface follow-up SOWs; if a surface regresses
   badly, scope a quick token-only hotfix rather than holding this SOW. Confirm the
   appetite for the simultaneous shift at review.
