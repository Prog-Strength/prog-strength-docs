---
type: dx
status: draft
surface: calendar-day-detail
idioms:
  - inbox-triage
  - timeline-spine
  - itinerary-cards
  - stat-led-dossier
  - split-ledger
references:
  - Superhuman
  - Linear
  - Fantastical
  - Things
  - Whoop
scope: in-system
variant_count: 5
repos:
  - prog-strength-web
  - prog-strength-docs
---

# DX: Calendar Day Detail (master–detail day view)

**Status**: Draft · **Last updated**: 2026-06-17

## Context

The `/calendar` page shipped its grid in the `true-month-grid` direction
(`dx/calendar-view-refresh.md`, selected). That exploration deliberately scoped
itself to the month grid and navigation, and called out the one surface it left
unfinished: **the selected-day detail panel** — "a stack of dashed-outline rows
that looks unfinished." This DX is that follow-up.

The panel (the `DayDigest` component) is what a user actually reads every
morning: tap a day, see what you did and what's planned. Today it renders the
day's events as full-bleed banner rows stacked in a single column. On a phone
that's fine. On a 1400px+ desktop it falls apart — a row for "W7 D2 – Interval
Run" is maybe 10% content and 90% dead space to its right, so the most-used
read-out on the page looks the least finished. There is also no in-place way to
*read* an event: a planned event opens a centered modal, a logged session
navigates away to its detail page, so inspecting the day is a series of context
hops off the calendar.

A better day detail uses the width it's given, lets you read an event without
leaving the calendar, and reads as a deliberate, polished surface that's
cohesive with the month grid sitting above it. This DX explores how that
two-pane day view should *look and compose* — not whether to build it.

## The surface

The selected-day detail panel beneath the month grid (`components/calendar/
day-digest.tsx`), plus the per-event row treatments
(`planned-banner.tsx`, `run-banner.tsx`, `workout-banner.tsx`,
`completed-planned-banner.tsx`).

### The fixed structure all variants share

The *structure* is decided; this DX explores its visual composition, not its
skeleton. Every variant must realize the same master–detail shape:

- **Desktop (`lg` and up, ≥1024px): two panes.** A left **event list** (the
  day's activities) and a right **detail pane** that shows the selected event in
  place. Selecting a row on the left fills the pane on the right; the pane is
  never empty when the day has events (first event auto-selected; a grid-pill
  click seeds the selection).
- **Below `lg`: one pane.** The right detail pane is **hidden**; only the list
  renders. Tapping a row uses the existing surfaces — **planned → the
  planned-workout modal**, **logged workout/run → its `/workouts/{id}` or
  `/running/{id}` detail page**, **completed-planned → the logged session**. No
  desktop two-pane on mobile, no new mobile detail surface.
- **The activity-color rail is preserved.** A row's left rail is colored by
  **activity type** — run = `--discipline-run-dot` (green), lift =
  `--discipline-lift-dot` (blue) — and the detail pane echoes that same hue for
  the selected event. This is the cohesion contract with the month-grid pills
  above; it does **not** vary by idiom. (Making *planned* rails activity-colored
  rather than status-violet is governed by
  `sows/calendar-event-detail-and-activity-colors.md`; variants assume that
  end-state — planned events read in their activity hue, status is conveyed by
  shape + badge, not color.)
- **No backend.** Everything renders from the `CalendarEvent` already in hand.

Because palette, accent, and type are fixed (`scope: in-system`), the variants
diverge on **composition, structure, and density** — how the list and the detail
pane are laid out, how time and agenda are expressed, how dense or generous the
rhythm is — not on color or font.

### Data shapes

Events are a discriminated union (`components/calendar/types.ts`):

```ts
type CalendarEvent =
  | { kind: "workout";  startMs; workout }                    // a logged lift
  | { kind: "run";      startMs; run }                        // a logged run
  | { kind: "planned";  startMs; planned }                    // forward-looking intent
  | { kind: "completed-planned"; startMs; planned; logged }   // planned + the session that fulfilled it
```

The fields that matter for composition:

- **Planned** carries `name`, `activity_kind` (`lift` | `run`), `scheduled_start`
  **and `scheduled_end`** (RFC3339 — a real time block), `status` (`planned` |
  `completed` | `skipped`), `run_type`, `run_details`, an ordered `exercises`
  agenda (each with target sets), and `google_sync_status` (drives the sync
  glyph / Resync affordance).
- **Logged** workouts/runs carry a start time and a real **`duration_seconds` /
  `elapsed_seconds`**, plus the session's metrics (sets, distance, pace, HR).
- A planned agenda is an ordered exercise list, each with a name, optional
  muscle-group tags, set/rep specs, and optional **superset** grouping.

Both planned and logged events therefore have a real **time window and duration**
— enough to map onto a time axis (the `timeline-spine` idiom depends on this).

### States the variants must handle

These are where lazy day-detail designs fall apart — render them all:

- **Single event** — the common case. The detail pane and a one-row list must
  both read as intentional, not sparse. (This is the case that looks worst
  today.)
- **Dense day** — 3–4 stacked events (Jun 16 carries four planned). The list
  stays scannable; the pane still shows exactly one selected event.
- **Activity & lifecycle legibility** — completed-lift vs completed-run vs
  planned vs completed-planned vs **skipped**, all distinguishable at a glance
  using the activity tonal hue for *type* and shape/badge for *status* (not
  color-only). Skipped is the rarest and must still read clearly.
- **Selection state** — which list row is active must be unmistakable, and the
  pane must track it. On a grid-pill click the matching event is pre-selected.
- **Empty / rest day** — no events and no steps: a single deliberate rest-day
  state spanning the panel (no empty two-pane), with the "Plan a workout →"
  affordance. Steps-only days show the steps banner but no event list hops.
- **Mobile collapse** — at narrow widths the pane is gone and rows are
  tap-to-open targets into the existing modal/detail surfaces. The list must
  stand on its own without the pane.

### Representative fixture

Use **Thursday, Jun 18** as the worked single-vs-multi example (the surface in
the screenshot that prompted this): two planned events —
**W7 D2 – Interval Run** (planned run, 12:00 PM–1:00 PM, run_type `intervals`)
and **W5 D2 – Legs** (planned lift, 6:00 PM–7:00 PM, "No agenda"), both
`PLANNED`, the run with a green rail and the lift with a blue rail.

For the dense and lifecycle states reuse the first exploration's month so this
stays consistent with the shipped grid:

- A **dense day** (Jun 16) with four stacked planned sessions.
- A **completed-planned** worked example — **W5 D1 – Chest & Back** (planned
  Lift, 5:00 PM–6:30 PM, "Synced to Google Calendar", six exercises incl. a
  Machine Fly + Pull Up superset, muscle-group tags) merged with the logged lift
  that fulfilled it — so the detail pane has rich agenda + actuals to compose.
- A **skipped** planned session and a **rest day** with nothing logged.

These are mockups: **static fixtures that look real are preferred** — do not wire
variants to live data services. (Per the dumbbell-weight convention, any weight
shown is per-dumbbell.)

## Idioms

Five genuinely distinct **compositions** of the same master–detail day view.
Each keeps the fixed structure, palette, accent, and type above, and diverges on
how the list and detail pane are arranged, how time/agenda are expressed, and the
spacing rhythm.

- **inbox-triage** — leans on **Superhuman**. The left list is a dense,
  uniform-height stack of scannable triage rows (rail · title · time · terse
  meta), tuned for fast top-to-bottom scanning; the right pane is a focused
  "opened item" reading view — title block, a thin meta line, the agenda body,
  and a **pinned action row** at the bottom (Edit / Start / Skip · Resync). Tight
  type scale, hairline row separators, generous pane. The feel is "triage the
  day, then read the one you opened."

- **timeline-spine** — leans on **Fantastical** / Google Calendar's day view.
  The left "list" is reorganized as a vertical **time spine**: hour ticks with
  each event drawn as a block positioned and sized by its `scheduled_start`/
  `scheduled_end` (or logged `duration_seconds`). The hour window **clamps to the
  event range ± padding** so a 2-event day doesn't render a dead 24-hour ribbon.
  The right pane details the selected block. Type scale is quiet and
  numeric-forward (time labels lead); spacing rhythm is driven by the time axis,
  not by cards. This is the only idiom where vertical position encodes *when*.

- **itinerary-cards** — leans on **Things** (Cultured Code). The left column is a
  set of calm, soft-rounded **itinerary cards**, one per event, with generous
  internal padding and larger tap targets; the right pane is an expanded card
  with **sectioned disclosure** (agenda, sets, sync, actions) revealed as tidy
  blocks. Larger type scale, roomier spacing rhythm, the most editorial and
  "glad to open every morning" of the five. Empty/rest day gets a deliberately
  satisfying treatment here.

- **stat-led-dossier** — leans on **Whoop**. The detail pane **leads with the
  event's numbers** as a compact metric dossier — for a logged session its
  actuals (duration, distance/pace or volume, HR); for a planned one its target
  block (time window, planned exercise/run-type count) — then the agenda beneath.
  The left list is a slim, metric-tinted **index** (rail · title · one key
  number). Data-forward, single-accent-on-slate, tightest information density of
  the five; the activity hue does double duty as the metric accent (still
  in-system).

- **split-ledger** — leans on **Linear** (minimal). The most restrained,
  chrome-light composition: the left list is a quiet **ledger** of rows separated
  only by hairlines — no card backgrounds — and the right pane is a wide reading
  column where hierarchy comes almost entirely from **typography and whitespace**
  rather than borders or fills. Structure-through-type, the largest negative
  space, the calmest surface. The test of whether the day detail can feel
  finished with the least visual weight.

## References

- **Superhuman** — its split-inbox model: a dense, uniform triage list on the
  left and a single focused reading pane on the right with a pinned action bar.
  Take the *list-density + focused-pane + bottom action row* pattern, not its
  light theme. (Drives `inbox-triage`.)
- **Linear** — hairline-separated rows, restrained density, hierarchy from type
  weight and spacing rather than chrome, and a clean list↔detail split. Take the
  *minimal, type-led, borderless composition*. (Drives `split-ledger`; also the
  general restraint bar.)
- **Fantastical** / Google Calendar — the day-view **time spine**: hour ticks
  with events as proportionally-sized blocks placed by start/end. Take the
  *time-as-vertical-position* mapping and light-touch time labels — but clamp the
  visible hour window so sparse days don't sprawl. (Drives `timeline-spine`.)
- **Things** (Cultured Code) — calm itinerary cards, generous padding, soft
  rounded surfaces, larger type, and genuinely pleasant empty states. Take the
  *editorial card rhythm and roomy spacing*. (Drives `itinerary-cards`.)
- **Whoop** — its metric-dossier density and single-accent-on-charcoal stat
  blocks. Take the *stat-led detail header and metric-tinted index rows*, mapped
  onto the slate palette and activity hues. (Drives `stat-led-dossier`.)

## Selection criteria

A note-to-self for the gate, not a rubric for the worker:

- **Is this the surface I'm glad to open every morning?** The day detail is the
  most-read part of the calendar. Which composition makes "what did I do / what's
  next" feel intentional rather than dutiful?
- **Does it earn the width?** The whole reason for two panes is that a full-bleed
  row looks unfinished on desktop. Which one actually uses the space rather than
  centering a narrow column in a wide void?
- **Does `timeline-spine` survive the sparse day?** A literal time axis is the
  most distinctive but the most at-risk on a 1-event day. If its clamped window
  still looks deliberate on Jun 18, it's a real contender; if it trades
  horizontal dead space for vertical, it isn't.
- **Does the detail pane actually reduce friction?** It exists to let me read an
  event without a modal/nav hop. Which pane composition makes that worth it for
  both a planned agenda and a logged session's actuals?
- **Does mobile still hold up?** At narrow widths the pane is gone and rows are
  tap-to-open. The list half of the winning idiom has to stand on its own.
- **Cohesion with the shipped grid.** It sits directly under the `true-month-grid`
  calendar. The winner should feel like the same product, not a second design
  language — same slate surfaces, violet selection, activity hues.

---

> **Lifecycle.** `status:` is editorial — the owner is the dispatch gate. It moves
> `draft` → `exploring` (worker running) → `awaiting_selection` (draft PR open,
> owner deciding) → `selected` / `abandoned`. The worker sets `awaiting_selection`
> on the `dx/calendar-day-detail` branch as it opens the PR; the owner sets the
> terminal value when they close it.
>
> **Handoff.** A DX ends at a *chosen variant*, not merged code. Open the draft
> `[DX — DO NOT MERGE]` PR, compare the variants on the preview deploy, pick one,
> tick its box, set `status: selected` (noting the winning idiom), and **close the
> PR — never merge it.** Then open a normal SOW: *"implement the calendar day
> detail per the `<chosen-idiom>` variant from `dx/calendar-day-detail`,
> production-quality, conforming to the design system."* The mockup code is never
> promoted as-is.
