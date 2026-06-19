---
type: sow
status: draft
repos:
  - prog-strength-web
  - prog-strength-docs
---

# Bodyweight Readings Table — Timeline-Rail

**Status**: Draft · **Last updated**: 2026-06-18

## Introduction

The `/bodyweight` page shipped its redesign from the
[`dx/bodyweight-page`](../dx/bodyweight-page.md) exploration
([`sows/bodyweight-page-redesign.md`](./bodyweight-page-redesign.md), the
`trend-band-analyst` direction). That work fixed the **chart** — the smoothed
trend now beats the daily noise — but treated the **log region** as one element
among many, so it shipped essentially as-is: a `DATE / TIME / WEIGHT` grid with
pencil/trash glyphs. It is the one part of an otherwise purpose-built
trend-watching page that still reads like a database export.

[`dx/bodyweight-readings-table`](../dx/bodyweight-readings-table.md) (explored in
[prog-strength-web#80](https://github.com/Prog-Strength/prog-strength-web/pull/80))
narrowed the lens to just the log region and produced five variants. The
**`timeline-rail`** direction was selected. This SOW builds it for real,
production-quality, conforming to the design system.

The chart answers "which way am I trending"; the log should answer "what
happened, day by day." Today's grid answers neither — it lists rows and encodes
none of the three things that make a weight log worth keeping: **the delta since
last time, that a morning and an evening reading belong to the same day, and
whether the day moved up or down.** `timeline-rail` reframes the log as
**chronology**: a vertical rail where each calendar day is a node, a day's
readings cluster as beads on that node, and day-over-day change reads as movement
along the rail.

## Proposed Solution

Replace the log region of `/bodyweight` — the desktop `<table>`, the mobile card
list, and their shared pagination — with a single new client component,
`components/bodyweight/bodyweight-readings-timeline.tsx`, rendering the
`timeline-rail` idiom at both breakpoints via container queries.

- A **vertical rail** runs down the left. Each **calendar day** (local-date
  bucket over `measured_at`) is a **node** on the rail, newest first.
- A day's readings render as **beads** clustered at that node: one bead for a
  single-reading day; two (or more) beads with the **intra-day spread** bracketed
  between them for a multi-reading day. The day's **average** is the node's
  headline figure.
- The **day-over-day delta** (this day's average vs the previous day's) reads as
  a short **connector** between adjacent nodes — **accent when the move is down**
  (toward the cut goal), **muted slate when up**. The rail itself is the
  structural color.
- Edit/delete remain reachable per reading (desktop icon buttons revealed on the
  bead/row; mobile taps open the existing
  `components/bodyweight/bodyweight-action-sheet.tsx`).
- The **log toolbar** above the region (a `Log` create action on the left; the
  goal affordance `◎ Goal: 178 lb` / `Set goal weight` on the right) is preserved.

The component is **presentational over already-fetched data** — the page keeps
owning fetch, create/edit/delete mutation, and goal state; the timeline receives
`entries` + `goal` as props and emits edit/delete intents back up. This isolates
the new view, shrinks the 66 KB `page.tsx`, and keeps the data flow unchanged.

No chart, stat-tile, or API changes. Everything the view adds (day grouping,
averages, deltas, spreads) is derived client-side from the flat readings, exactly
as the chart's smoothing layer already does.

## Goals and Non-Goals

### Goals

- Replace the log region's table + mobile cards with the `timeline-rail`
  component, at **both** breakpoints (desktop frame and ≤360 px phone).
- Make **multiple readings on one day** read as *one day with N readings* (beads
  + intra-day spread), never as orphan adjacent rows. This is the surface's whole
  reason to exist.
- Encode **day-over-day direction** via the accent/muted connector split, so up
  vs down reads instantly without color guesswork.
- Group strictly by **local calendar date** over `measured_at`, so the late-PM /
  early-AM boundary (e.g. `Jun 15 11:24 PM` vs `Jun 12 12:44 AM`) is
  unambiguous — readings ~80 min apart across midnight land on different nodes.
- Preserve edit, delete, create (`Log`), goal display/`Set goal weight`, units
  (`lb`/`kg`), and most-recent-first ordering.
- Handle every required visual state (see Implementation Details → States).
- Conform to **design-system.md v0.4** tokens; introduce no new palette, accent,
  or type.

### Non-Goals

- **No chart or stat-tile changes.** The trend band, headline stats, and range
  controls shipped by `bodyweight-page-redesign` are untouched.
- **No API or data-model changes.** `GET /bodyweight` and `GET /me/bodyweight-goal`
  are unchanged; all derived values are client-side.
- **No new design tokens.** This is `scope: in-system`; it does not re-decide
  palette/accent/type.
- **No goal-editing rework.** The goal affordance behaves exactly as today; only
  its placement in the new toolbar is in scope.
- **The DX exploration code is not merged or salvaged.** PR #80 is a disposable
  throwaway; this is a clean reimplementation. Do not import from
  `app/design-explore/`.

## Implementation Details

### Design-system conformance (read first)

Conform to [`design-system.md`](../design-system.md) **v0.4** and reference its
tokens — never hard-code hex that duplicates a token.

- **Accent**: the single desaturated **periwinkle** (`--accent` `#9aa6d6`, with
  `--accent-soft` / `--accent-line`). This **replaced** the old violet.
- **Typography**: **Manrope** throughout; **no display face** (the Oswald accent
  was dropped). Numerals carry `-0.03em` tracking. `Geist_Mono` only for
  incidental mono figures if a bead needs it.
- **Surface**: soft near-black ramp; **hairline borders** as default; **14 px**
  panel radius, full-pill on chips.

Note: the DX ticket prose and PR #80 description still say "violet / Nunito /
Oswald" in places — that wording predates the v0.4 re-tone
([`sows/activities-page-redesign.md`](./activities-page-redesign.md)). The
**living `design-system.md` is the source of truth**; the exploration variants
already render in the periwinkle/Manrope tokens.

### Surface being replaced

In `app/(app)/bodyweight/page.tsx`: the desktop `<table>` (Date · Time · Weight ·
actions), the mobile card list (`sm:hidden`, taps open `BodyweightActionSheet`),
and the shared client-side pagination (`PAGE_SIZE = 20`, the `page` state, and
the `pageEntries` slice). Locate these by those anchors rather than line numbers
(the file is large and shifts). The toolbar (`Log` action + goal affordance)
sits directly above and is preserved.

### Data shape & derivation (client-side)

From `lib/api.ts`:

```ts
type BodyweightEntry = {
  id: string;
  weight: number;
  unit: "lb" | "kg";
  measured_at: string; // RFC3339 — carries BOTH calendar day AND time-of-day
  created_at: string;
};
type BodyweightGoal = {
  weight: number;
  unit: "lb" | "kg";
  created_at: string | null; // null ⇒ never set ⇒ "Set goal weight"
  updated_at: string | null;
};
```

Add a pure helper (e.g. `lib/bodyweight-grouping.ts`, unit-tested in isolation)
that turns `BodyweightEntry[]` into ordered **day-groups**, each:

- `dateKey` — local calendar date (the node identity; grouping key).
- `readings` — that day's entries sorted within the day, each retaining `id`,
  `weight`, `measured_at` (for the bead time label), `unit`.
- `average` — mean weight for the day (the node headline).
- `spread` — `max − min` weight when `readings.length > 1`, else `null`.
- `deltaVsPrevDay` — `average − previousDay.average` (chronologically previous
  day-group), or `null` for the first/oldest group; its **sign** drives the
  connector color (down = accent, up = muted).

`measured_at` is the only source of both the day bucket and the time-of-day
label; bucket by **local** date, consistent with the rest of the app's
timezone-aware date windowing.

### Rail composition

- **Rail & nodes**: a hairline vertical rail; each day-group is a node. The rail
  is the structural color; medium type, generous vertical rhythm, no oversized
  hero numeral.
- **Beads**: single-reading day → one bead at the node with the value + time.
  Multi-reading day → a bead per reading with their times, and the **intra-day
  spread** bracketed between them, so the cluster reads as one day.
- **Connector / delta**: between a node and the next, a short connector segment
  carrying the day-over-day delta; **`--accent` for a down move, `--muted` slate
  for an up move**. The accent earns its place on *meaning* (direction, active
  node), never as generic row chrome. Do not introduce a second hue; goal context,
  if surfaced per-node, uses a restrained slate marker.
- **Both breakpoints**: desktop and ≤360 px phone via container queries. Define
  the bead/time treatment for both — the phone frame must not truncate the spread
  or the delta.

### Edit / delete / create

- **Desktop**: per-reading edit + delete affordances revealed on hover/focus of
  the bead or its row, wired to the page's existing `editingEntry` /
  `deletingEntry` handlers.
- **Mobile**: tapping a bead/day opens the existing `BodyweightActionSheet`
  (edit / delete) — reuse it; do not fork.
- **Create**: the toolbar `Log` action keeps opening the create-reading modal
  unchanged.

### Pagination — whole-day packing

The current slice (`entriesInRange.slice(start, start + 20)`) cuts on raw
readings and **would split a day's beads across two pages**, which in the rail
idiom reads as two different days — unacceptable. Paginate by **day-group with
whole-day packing**: fill a page up to ~20 readings but **never split a
day-group across a page boundary** (a day that would straddle the cap moves
wholly to the next page). Page controls stay below the region. Most-recent day at
the top of page 1.

### States to handle (render/verify all)

- **Multi-per-day** — morning + late-night reading 1–3 lb apart read as one day,
  spread visible.
- **Single-reading day** — one bead at the node; not broken or over-decorated.
- **Up vs down** — consecutive day-averages move both ways; connector color makes
  the sign instant.
- **Day boundary across a late reading** — `11:24 PM` and `12:44 AM` land on
  different nodes (local-date grouping).
- **No goal set** (`created_at: null`) — toolbar shows `Set goal weight`; any
  per-node goal context degrades to nothing, never a broken stat.
- **Pagination boundary** — page 1 of N at the 20-reading cap with whole-day
  packing; no split day.
- **Empty / first reading** — zero readings → "Tap Log to add your first
  reading"; one-reading case has no delta/trend yet but still feels like a place
  to start.
- **Unit** — `lb` or `kg` from the entry; no assumption of a 3-digit lb number or
  fixed decimal width.

## Testing

- **Unit (Vitest)** for the grouping helper: local-date bucketing incl. the
  cross-midnight boundary; average; spread; `deltaVsPrevDay` sign for up/down/
  first-day; single- vs multi-reading days; mixed `lb`/`kg`; empty input.
- **Whole-day packing**: a day-group straddling the 20-reading cap moves wholly
  to the next page; page counts and ordering correct.
- **Component render** of each required state against static fixtures (mirror the
  DX ticket's representative fixture — a multi-per-day month above a 178 lb goal),
  at both breakpoints.
- **Regression**: edit, delete, create, and goal display still function via the
  preserved handlers and `BodyweightActionSheet`.
- `lint`, `typecheck`, and the project build pass.

## Rollout

Single PR to `prog-strength-web` (component + grouping helper + `page.tsx`
swap-in + tests), plus this SOW's status flip in `prog-strength-docs`. No
flags, migrations, or API coordination — purely a client-side view swap on an
existing page. Verify against a real account with multi-per-day history before
merge. Standard rebase-only merge.

## Open Questions

1. **Status**: leave `draft` for review, or flip to `ready_for_implementation`
   so the autonomous developer can dispatch it?
2. **Delta basis**: day-over-day uses **day-average vs previous day-average**
   (specified above). Confirm that over "latest reading vs previous reading,"
   which would make the connector jumpier on multi-reading days.
3. **Per-node goal context**: surface "X to go" / a goal marker on each node, or
   keep goal context entirely in the toolbar (simplest, fewest accents)? Default:
   toolbar only.
