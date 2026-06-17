---
type: dx
status: selected
selected_idiom: true-month-grid
surface: calendar-view-refresh
idioms:
  - true-month-grid
  - week-band-rows
  - agenda-hybrid
  - panel-driven
references:
  - Google Calendar
  - Fantastical
  - Notion Calendar
  - Linear
  - Whoop
scope: in-system
variant_count: 4
repos:
  - prog-strength-web
  - prog-strength-docs
---

# DX: Calendar View (refresh)

**Status**: Selected (`true-month-grid`) · **Last updated**: 2026-06-16

> A second pass at the monthly calendar. The first exploration
> ([`dx/calendar-view.md`](calendar-view.md)) was `greenfield` and picked
> `warm-organic-coaching` — a warm amber/clay palette — **before** the app-wide
> design system existed. The design system has since settled on a violet accent
> on a slate ramp (and explicitly records that "the calendar's original clay
> accent yields to violet"), but the calendar surface was never fully re-toned,
> so it still reads in orange and now looks foreign next to the rest of the app.
> On top of the color drift, the surface has accumulated three concrete problems
> worth fixing together. This DX re-explores the calendar **`in-system`** — every
> variant uses the settled tokens and diverges only on *composition* — so the
> winner can be implemented as a calendar that finally belongs to the app.

## Context

The monthly training calendar (`/calendar`) is one of the most-visited surfaces
in Prog Strength — it's where a user sees their training month at a glance: what
they lifted, what they ran, what's planned, and how each week is shaping up. The
**data and functionality are solid**: the grid renders logged lifts, logged
runs, planned sessions, and the merged "planned-then-completed" case correctly;
weekly rollups compute; the day digest and the view/edit modals all work.

What's wrong is the **look**, and it's wrong in four specific ways now that the
rest of the app has moved on:

1. **It's still orange.** Activity pills, the today marker, the per-week progress
   strip, and the "Plan a workout" button all render in amber/clay — the legacy
   `warm-organic-coaching` accent. The design system replaced this with violet
   app-wide. The calendar is the conspicuous holdout, and the warm chips clash
   with the slate/violet shell around them.
2. **It shows weeks that aren't in the month.** Below the current month's weeks,
   the grid keeps rendering whole extra week-band rows of the *next* month
   (greyed "Rest week — recovery counts too" bands for early July), so the page
   scrolls far past where the month ends. The month should end where the month
   ends.
3. **The month navigation looks bad.** Month switching is a header utility bar
   with a title and two ghost chevrons floating next to the primary action. It
   reads as browser chrome, not as part of a training tool, and it's the first
   thing in the visual hierarchy.
4. **The day view and its modals are weak.** The selected-day digest is a stack
   of dashed-outline rows that looks unfinished. Clicking an event opens a
   centered **view modal** (workout title + agenda) and an **edit form modal**
   ("Edit planned workout") whose inputs, selects, muscle-group chips (saturated
   red/blue/yellow), and superset grouping predate every design-system decision
   and look the roughest of anything in the app.

A better calendar re-tones cleanly into the app, stops at the month boundary,
navigates in a way that feels native, and gives the day detail and its
edit-flow a treatment a user would be glad to open every morning. This DX finds
the *composition* that does that — across the grid, the navigation, the day
detail, and both modals as **one cohesive surface** — before a SOW builds it.
No production calendar code changes here.

## The surface

This DX covers the **whole calendar experience as one surface**: the month grid,
its navigation, the selected-day detail, the event **view** modal, and the event
**edit** form. A variant must give a single coherent answer to all of them — they
are explored and (later) implemented together.

### Anatomy on screen today

- **Header / chrome** — month + year title ("June 2026"), a coaching subtitle
  ("Nice work, Jimmy Wallace — you've trained 9 days this month."), a `Today`
  button, a primary `Plan a workout` action, and prev/next month chevrons.
- **Month summary stat row** — six hero stats for the visible month: **Lift
  Time** (`7h 20m`), **Run Time** (`1h 55m`), **Run Distance** (`11.7 mi`),
  **Avg Pace** (`9:49 /mi`), **Longest Run** (`4.2 mi`), **Activities** (`9`).
- **Month grid** — a Mon→Sun grid of week-band rows. Each day cell shows the day
  number and zero or more **event pills**. Each week row carries a **per-week
  coaching strip** ("● ● ● ○ ○ ● ●  You trained 5 of 7 days" + that week's lift
  time / run distance / steps).
- **Day digest** — below the grid, the selected day expands to a dated list
  ("Tuesday, June 16, 2026 · 2 activities"), each row showing session name, a
  `PLANNED` tag where applicable, scheduled time, and a one-line summary.
- **Event view modal** — clicking an event opens a centered modal: title
  ("W5 D1 - Chest & Back"), a type tag, a `PLANNED` status, a "Synced to Google
  Calendar" note, a WHEN block, and an **AGENDA** — a numbered exercise list with
  per-exercise **muscle-group chips** (Chest / Back / Shoulders) and set/rep
  lines. Footer: `Close` / `Start workout`.
- **Event edit form** — the pencil opens "Edit planned workout": an Activity
  toggle (Lift / Run), Name, Date / Start / Duration, a derived "Ends at", Notes,
  and an **Agenda** list of exercise rows with per-set detail and a **superset**
  grouping treatment. Footer: `Cancel` / `Save changes`.

### Data shapes

Events are a discriminated union (`components/calendar/types.ts`):

```ts
type CalendarEvent =
  | { kind: "workout";  startMs; workout }          // a logged lift
  | { kind: "run";      startMs; run }              // a logged run
  | { kind: "planned";  startMs; planned }          // forward-looking scheduled intent
  | { kind: "completed-planned"; startMs; planned; logged }  // planned + the session that fulfilled it, merged into one "done" entry
```

Weekly rollups (`components/calendar/weekly-overview.tsx`):

```ts
type WeeklyStat = { weekStart: Date; activities: number; liftMinutes: number; runMeters: number; steps: number };
```

A planned workout's agenda (what the view/edit modals render) is an ordered list
of exercises, each with a name, optional muscle-group tags, one or more set
specs (`reps × sets`), and an optional **superset** grouping that binds two or
more exercises together.

### States the variants must handle

These are where lazy designs fall apart, so render them all:

- **Tri-state legibility, re-toned.** Completed-lift vs completed-run vs planned
  vs completed-planned must be instantly distinguishable **without orange**. Use
  the design system's **activity tonal hues** (run / lift, desaturated for the
  dark slate) for type, and **violet** for selection/today and primary action —
  not a warm palette. Encoding should not be color-only (glyphs, fills, outlines,
  position all help), so the tri-state survives for a color-blind eye too.
- **Month-bounded grid (the fix).** Render **only the current month's weeks**.
  No trailing week-band rows for the next month; no "Rest week — recovery counts
  too" bands past month end. Leading/trailing days of adjacent months appear
  **only** as greyed cells inside the first and last week rows, never as whole
  extra rows.
- **Navigation, re-thought.** The header-bar-with-chevrons is explicitly
  rejected. Each variant proposes its own month navigation that reads as part of
  the tool, not browser chrome. The `Today` jump and the `Plan a workout`
  primary action must remain reachable.
- **Day detail, finished.** The selected-day view must look intentional and
  polished — not a stack of dashed outlines. It is the connective tissue between
  the grid and the modals.
- **Event view modal, re-toned.** Same information, design-system surfaces:
  muscle-group chips drop the saturated red/blue/yellow for restrained tints
  consistent with the macro-tint / tonal-hue logic; the modal is a slate panel
  with hairline borders and violet primary action.
- **Edit form, finished.** The form is the roughest surface in the app today.
  The design system lists **form-control specs (input / select / toggle) as
  "not yet decided"** — so this DX is the surface that *proposes* them. Variants
  should style Name/Date/Start/Duration/Notes inputs, the Lift/Run toggle, the
  select menus, and the **superset grouping** as a coherent set; the winning
  treatment becomes a candidate to codify back into `design-system.md`. (See
  Selection criteria.)
- **Dense day** — a single day can stack 3–4 sessions (Jun 16 carries four
  planned). The treatment must not truncate into uselessness.
- **Empty days / sparse weeks** — most days are empty; some weeks have 0
  activities. Empty should feel intentional, not broken.
- **Current day** highlighted; **recovery weeks** (sessions named "Recovery
  Week …") read as part of the intensity story, not as errors.

### Representative fixture

Reuse the month from the first exploration so variants render realistically — a
month with a completed first half, a planned second half, a dense day, a
recovery week, and empty stretches:

- **Week of Jun 1** — 5 activities: completed lifts Mon/Wed/Sat/Sun, completed
  easy run Tue. Summary: 5 sessions · lift 4h 48m · run 3.9 mi.
- **Week of Jun 8** — 3 activities ("Recovery Week"): Wed lift, Sat lift, Sun
  run. Summary: 3 sessions · lift 2h 32m · run 3.6 mi · 11,000 steps.
- **Week of Jun 15** — today is Mon Jun 15 (completed easy run ✓). **Jun 16 is
  dense**: four planned sessions stacked (W7 D2 Interval Run, W5 D1 Chest & Back,
  plus two more). Thu/Sat/Sun carry planned sessions. Summary: 1 activity so far
  · run 4.2 mi · 10,000 steps.
- **Weeks of Jun 22 and Jun 29** — empty (0 activities). The last week row holds
  Jun 29–30 plus greyed Jul 1–5 cells — and **stops there** (no July rows).

For the modals, use **W5 D1 - Chest & Back** as the worked example: a planned
Lift, Tue Jun 16 · 5:00 PM – 6:30 PM, "Synced to Google Calendar", six exercises
(Barbell Bench Press · Barbell Bent Over Row · Incline Dumbbell Bench Press ·
Dumbbell Tripod Row · Machine Fly · Pull Up), with Machine Fly + Pull Up bound as
a **superset**, each carrying muscle-group tags (Chest / Back / Shoulders) and
set/rep specs. (Per the dumbbell-weight convention, any weights shown are
per-dumbbell — but this fixture is rep/set-based, so weight rarely appears.)

These are mockups: **static fixtures that look real are preferred** — do not wire
the variants to live data services.

## Idioms

Four genuinely distinct **compositions** of the same surface. Because
`scope: in-system`, they share the palette (slate ramp + violet accent), type
(Nunito), and form language (rounded-2xl/3xl cards, full-pill chips) — they
**diverge on layout, structure, density, navigation pattern, and where the day
detail and edit flow live**, not on color or type family. "Color logic" below
means *how much accent each direction spends and where* — not a different
palette.

- **true-month-grid** — A disciplined, bordered Mon→Sun month grid in the
  vocabulary of **Google Calendar / Fantastical**, re-toned to slate. Exactly the
  current month's week rows (the bug fix is structural here, not patched);
  leading/trailing days are greyed cells inside the boundary rows. Compact,
  uniform cells; the per-week rollup collapses into a **slim 7th "week" column**
  on the right rather than a full-width strip. **Type scale:** small and even,
  hierarchy from weight and the day-number treatment. **Color logic:** frugal —
  near-monochrome slate, tonal-hue left-bars/dots to mark run vs lift, violet
  spent **only** on today and the selected cell. **Spacing rhythm:** tight,
  gridded, calendar-native. **Navigation:** a compact inline month stepper
  (month label + chevrons rendered as a single quiet pill control, or a
  click-to-open month menu), demoted out of the hero position. **Day detail:**
  selecting a day expands an inline panel beneath the grid. **Modals:** centered
  slate panels, re-toned. This is the "make it a real calendar, done right"
  direction.

- **week-band-rows** — Evolves the **current** horizontal week-band structure
  instead of replacing it, but executes it with discipline and the coaching
  warmth the design system endorses (Whoop's tone, re-toned to slate/violet).
  Each week is a generous row of seven day cards plus a **first-class per-week
  coaching strip** — the "● ● ● ○ ○ ● ●  You trained 5 of 7 days" dots and that
  week's totals become a real summary element, not an afterthought rail. Only the
  current month's weeks render. **Type scale:** larger and more comfortable than
  true-grid; the day cards breathe. **Color logic:** violet carries the
  streak/progress dots and the selected week's ring; activity hues stay
  desaturated. **Spacing rhythm:** airy, vertical, one week per band with real
  separation. **Navigation:** the month moves via large prev/next affordances
  attached to the band stack (or a left/right swipe metaphor), with the title as
  a section header rather than a utility bar. **Day detail:** the selected day's
  digest sits directly under its band. **Modals:** roomy, coaching-toned. This is
  "what exists now, finally polished."

- **agenda-hybrid** — Reframes the surface around the **list**, in the vocabulary
  of **Notion Calendar / Fantastical's sidebar**. A small, glanceable **month
  mini-grid** (date-picker scale — a month fits in a compact block) sits beside a
  prominent **agenda column** that is the main content: the selected day's (or
  week's) sessions as full, readable rows with title, time, type hue, status, and
  a one-line summary. This directly upgrades the weak day view by making it the
  hero. **Type scale:** dramatic — tiny grid numerals against large agenda rows.
  **Color logic:** the mini-grid is near-monochrome with violet on today/selected
  and tonal dots for activity; the agenda rows carry the tonal-hue accents.
  **Spacing rhythm:** two-column, the grid quiet and the agenda spacious.
  **Navigation:** clicking the mini-grid *is* navigation; the visible month
  changes by paging the small grid, no header bar. **Day detail:** there is no
  separate digest — the agenda is the day detail. **Modals:** because detail
  already lives in the column, the view modal can be lighter; edit still opens a
  focused form. This is "the calendar as an agenda."

- **panel-driven** — The month grid stays primary (true-grid-like discipline,
  month-bounded), but **everything that is a modal today becomes a docked
  right-hand side panel** — in the vocabulary of **Linear's issue panel**.
  Selecting a day slides in a panel with that day's sessions; selecting an event
  shows its detail in the same panel; the **edit form opens in the panel too**,
  not as a centered overlay. The grid never gets covered by an opaque modal, so
  the month stays in view while you read or edit a session — continuous and
  app-like. **Type scale:** even grid, structured panel with clear section
  headers (WHEN / AGENDA / form fields). **Color logic:** violet anchors the
  panel header and primary action; the grid is restrained. **Spacing rhythm:**
  grid + a persistent right column; the panel has its own internal rhythm.
  **Navigation:** a slim segmented/stepper control above the grid. **Day detail
  & modals:** unified into the docked panel — this is the direction that most
  directly answers "the modals look bad" by removing the overlay entirely and
  giving the edit form a calm, full-height home with proper form controls. This
  is "no modals — a workspace."

## References

- **Google Calendar** (month view) — take its **strict month-bounded grid** and
  the convention that adjacent-month days are quiet greyed cells *inside* the
  boundary rows, never extra rows. Grounds the structural fix in
  `true-month-grid` (and the grid discipline in `panel-driven`).
- **Fantastical** — take its **calm month grid paired with a readable day/agenda
  reveal** and its quiet inline month stepper (no chrome-y header bar). Drives the
  navigation rethink in `true-month-grid` and the agenda framing in
  `agenda-hybrid`.
- **Notion Calendar** — take its **mini-grid + dominant agenda column** split and
  its restrained, near-monochrome event rows with a single accent. Drives
  `agenda-hybrid`.
- **Linear** (issue side-panel + forms) — take its **docked right panel** that
  replaces modal overlays, its crisp section headers, and especially its
  **restrained, well-spaced form controls** (inputs, selects, toggles). Drives
  `panel-driven` and sets the bar for the edit-form treatment across all four.
- **Whoop** — take its **coaching tone and confident per-period summary** (the
  "you trained N of 7" framing as a first-class element), re-toned to slate/violet.
  Informs `week-band-rows`.

## Selection criteria

A note-to-self for the pick, not a rubric for the worker. When I compare these
I'm trying to decide:

- **Does it finally look like the rest of the app?** The whole reason for this
  pass is that the calendar is the orange holdout. The winner has to read as
  slate/violet Prog Strength at a glance, with zero warm-palette residue.
- **Is the tri-state still obvious without orange?** Completed-lift vs
  completed-run vs planned vs completed-planned is the heart of the surface — the
  re-toned encoding has to stay instantly legible (and ideally not color-only).
- **Does the month stop where the month stops?** No phantom next-month rows. The
  grid should feel bounded and intentional, and the dense day (4 stacked) and the
  empty weeks should both survive the treatment.
- **Does navigation feel native, not chrome?** I want to switch months without
  the surface feeling like an admin tool. Whichever nav pattern disappears into
  the tool best wins points.
- **Is the day detail something I'm glad to open?** The digest and the view modal
  are where I actually read a session — they should feel finished, not like a
  placeholder list of dashed rows.
- **Does the edit form set a form-control direction I'd want app-wide?** The
  design system hasn't decided input/select/toggle conventions yet. This is the
  surface that proposes them — so I'm also judging the winning form treatment as
  a candidate to **codify back into `design-system.md`**: do its inputs, selects,
  the Lift/Run toggle, and the superset grouping feel like the form language I'd
  want every form in the app to adopt?

---

> **Lifecycle.** `status:` is editorial — the owner is the dispatch gate. It moves
> `draft` → `exploring` (worker running) → `awaiting_selection` (draft PR open,
> owner deciding) → `selected` / `abandoned`. The worker sets `awaiting_selection`
> on the `dx/calendar-view-refresh` branch as it opens the PR; the owner sets the
> terminal value when they close it.
>
> **Handoff.** A DX ends at a *chosen variant*, not merged code. I open the draft
> `[DX — DO NOT MERGE]` PR, compare the variants on the preview deploy, pick one,
> tick its box, set `status: selected` (noting the winning idiom), and **close the
> PR — never merge it.** Then I open a SOW: *"implement the calendar view per the
> `<chosen-idiom>` variant from `dx/calendar-view-refresh`, production-quality,
> conforming to the design system — including the month-bounded grid, the new
> navigation, the day detail, and both the view and edit surfaces."* If the
> winning edit-form treatment is worth standardizing, that SOW (or a follow-up)
> also codifies the form-control specs into `design-system.md`.
