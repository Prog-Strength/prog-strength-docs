---
type: dx
status: draft
surface: bodyweight-readings-table
idioms:
  - ledger-dense
  - day-card-feed
  - timeline-rail
  - sparkline-rows
  - stat-ledger
references:
  - Linear
  - Things
  - Cardiogram
  - Robinhood
  - TrendWeight
  - Copilot Money
scope: in-system
variant_count: 5
repos:
  - prog-strength-web
  - prog-strength-docs
---

# DX: Bodyweight Readings Table

**Status**: Draft · **Last updated**: 2026-06-18

> A DX (Design Exploration) is the platform's **divergent** work type. Unlike a
> SOW it does not converge on one correct implementation — it produces N
> differentiated visual variants of a single frontend surface, side by side on
> one comparison route, awaiting a human pick. It **never merges** and ships no
> production code; the chosen direction feeds a downstream SOW that builds it for
> real.

## Context

The bodyweight page (`/bodyweight`) shipped its redesign from the
[`dx/bodyweight-page`](./bodyweight-page.md) exploration — the `trend-band-analyst`
direction was selected and built by
[`sows/bodyweight-page-redesign.md`](../sows/bodyweight-page-redesign.md). That
exploration fixed the **chart**: the smoothed trend now beats the daily noise,
which was the page's headline problem. But it treated the **log table** as one
element among many, and the direction that won wasn't the one (`weigh-in-journal`)
that re-thought the log. So the table shipped essentially as-is: a
`DATE / TIME / WEIGHT` grid with pencil/trash glyphs — the exact "reads
spreadsheet-y" complaint the page DX named but didn't resolve.

This DX narrows the lens to **just the log region** and explores it properly. It
matters now because the table is the one part of an otherwise purpose-built
trend-watching page that still reads like a database export. A weigh-in history
should express the things that make a weight log worth keeping — **the delta
since last time, that a morning and an evening reading belong to the *same day*,
whether today was up or down** — none of which the current grid encodes. The
chart answers "which way am I trending"; the log should answer "what happened,
day by day," and right now it just lists rows.

`scope: in-system`: the foundation is decided and was reaffirmed by the page
redesign — dark slate ramp, the single **violet** accent, Nunito, and the
Oswald-style condensed display face for big numerals. Variants do **not**
re-litigate palette, accent, or type. They diverge on **layout, structure,
density, and composition** — what the row (or card, or node) is, how
multiple-readings-per-day is made legible, and what the history's hero is. No
chart and no stat-tile code changes here; this is the log region only.

## The surface

The log region of `/bodyweight`, today rendered inline in
`prog-strength-web/app/(app)/bodyweight/page.tsx` (the `<table>` around line
934, the mobile card list around line 974, pagination at `PAGE_SIZE = 20`). A
variant owns this whole block and may reorganize any of it, but must account for
all of it:

- **Log toolbar** — a `Log` action (pencil affordance, opens the create-reading
  modal) on the left, and a goal affordance on the right (`◎ Goal: 178 lb`, or
  `Set goal weight` when no goal is set). Sits directly above the list.
- **The list** — one entry per measurement, sorted **most-recent-first**.
  - **Desktop** (`sm:` and up): a real `<table>` — **Date · Time · Weight ·
    [actions]** — with right-aligned weight, and edit + delete icon buttons per
    row revealed in the actions column.
  - **Mobile** (below `sm:`): the table is hidden; each entry is a tappable card
    that opens `components/bodyweight/bodyweight-action-sheet.tsx` (edit /
    delete). A variant must define the row treatment for **both** breakpoints.
- **Multiple readings on one day** appear today as **adjacent independent rows**
  with no visual link — e.g. `Wed, Jun 17 · 10:48 PM · 187.4 lb` immediately
  above `Wed, Jun 17 · 9:47 AM · 185 lb`. Making this pairing intentional is the
  central job of this DX.
- **Pagination** — client-side, **20 rows per page**, with page controls below
  the list. (The screenshot shows the un-paginated scroll length; the real
  surface caps at 20.)

The data shape (from `prog-strength-web/lib/api.ts`):

```ts
type BodyweightEntry = {
  id: string;
  weight: number;
  unit: "lb" | "kg";
  measured_at: string; // RFC3339 — carries BOTH the calendar day AND time-of-day
  created_at: string;
};

type BodyweightGoal = {
  weight: number;
  unit: "lb" | "kg";
  created_at: string | null; // null ⇒ goal never set ⇒ "Set goal weight"
  updated_at: string | null;
};
```

Endpoints: `GET /bodyweight` → `BodyweightEntry[]`, `GET /me/bodyweight-goal` →
`BodyweightGoal`. Everything a variant wants to *add* — the **day grouping**, the
**daily average**, the **delta vs the prior reading or prior day**, an
**intra-day range**, a **logged-day streak**, a **micro-sparkline** — is derived
**client-side** from `measured_at` + `weight`; the API returns only flat
readings. `measured_at` is the only source of both the day bucket and the
time-of-day label, so day-grouping is a local-calendar-day group over it.

**Color logic (in-system).** The list is neutral by default; the **violet accent
earns its place on meaning, not chrome** — the direction of a delta, the active
day-group, the summary band — never as a generic row tint. A *down* delta
(cutting, toward goal in the fixture) reads as the accent; an *up* delta stays
muted slate. Do not introduce a second hue for the table; goal context, if a
variant surfaces it per-row, uses a restrained slate marker, matching how the
shipped chart reconciled the legacy palette to the accent.

**Visual states the variants must handle** (render them all in the mockup — this
is where lazy table designs fall apart):

- **Multi-per-day.** A morning and a late-night reading on the same date, 1–3 lb
  apart. The pairing must read as *one day with two readings*, with the intra-day
  spread visible — not two orphan rows. This is the surface's whole reason to
  exist.
- **Single-reading day.** Most days have one reading; the day-group treatment
  must not look broken or over-decorated when there's nothing to pair.
- **Up vs down.** Consecutive readings (and consecutive day-averages) move both
  directions; the delta's sign and direction must read instantly via the
  accent/muted split, not color guesswork.
- **Day boundary across a late reading.** `11:24 PM` and `12:44 AM` belong to
  *different* calendar days despite being ~80 minutes apart — grouping is by
  local date, and the treatment must get the boundary visibly right.
- **No goal set.** `created_at: null` — the toolbar shows `Set goal weight`, and
  any per-row goal context degrades to nothing, never a broken stat.
- **Pagination boundary.** Page 1 of N with the 20-row cap; a day-group that
  would straddle a page break must not split a day across two pages in a way that
  reads as two different days (define the behavior).
- **Empty / first reading.** Zero readings ("Tap Log to add your first
  reading"), and the one-reading case where there's no delta or trend yet — the
  region should still feel like a place to start.
- **Unit.** `lb` or `kg`, taken from the entry; the row can't assume a 3-digit lb
  number or a fixed decimal width.

**Representative fixture** (mirror the screenshot — a multi-per-day month, above
goal, trending roughly flat-to-up, goal set at 178 lb, unit lb). Recent entries,
newest first:

| measured_at (local)      | weight  |
| ------------------------ | ------- |
| Wed, Jun 17 · 10:48 PM   | 187.4   |
| Wed, Jun 17 · 9:47 AM    | 185.0   |
| Tue, Jun 16 · 9:17 AM    | 185.4   |
| Mon, Jun 15 · 11:24 PM   | 187.8   |
| Mon, Jun 15 · 4:00 PM    | 186.4   |
| Sun, Jun 14 · 6:20 PM    | 181.8   |
| Sun, Jun 14 · 7:00 AM    | 185.2   |
| Fri, Jun 12 · 5:32 PM    | 183.8   |
| Fri, Jun 12 · 12:44 AM   | 188.2   |
| Wed, Jun 10 · 9:11 AM    | 180.2   |
| Sun, Jun 7 · 10:14 AM    | 179.6   |
| Sat, Jun 6 · 8:02 AM     | 180.8   |
| Sat, Jun 6 · 12:03 AM    | 182.4   |

Derived examples a variant can show: **Jun 17** day-avg ≈ **186.2** from two
readings (185.0 / 187.4, intra-day spread **2.4 lb**), **+0.8** vs Jun 16's
185.4; **Jun 15** two readings (186.4 / 187.8, spread 1.4); **Jun 14** spans
3.4 lb across the day. Several single-reading days (Jun 16, 10, 7). These are
mockups: **static fixtures that look real are preferred** — do not wire the
variants to live bodyweight services.

## Idioms

Five genuinely distinct compositions of the *same* dark-slate / violet / Nunito
surface. None re-decides palette, accent, or type — they diverge along **type
scale** (how hard they lean on the condensed display face and tabular figures vs
Nunito weight for hierarchy), **color logic** (how the single violet accent
separates meaning from neutral chrome), and **spacing rhythm** (tight blotter ↔
airy feed). Each makes a **different element the hero** of the history and leans
on a different reference.

- **ledger-dense** — The **power-user table, earned**. Keeps a true grid but
  makes it precise: tighter rows, weights in **tabular figures right-aligned and
  decimal-aligned** so the column scans like a ledger, and a dedicated **delta
  column** (`▼ 0.6`, accent on a down move, muted on an up). Same-day readings
  are **bracketed into a day-group** with a hairline and a single day-average in
  the gutter, so two readings read as one day without leaving the table form.
  Borrowing **Linear** and a trading blotter: tiny functional type, structural
  color (accent only on the delta sign and the active row), tight vertical
  rhythm. → the densest answer to *spreadsheet-y*: still a table, but one a
  power user would actually want.

- **day-card-feed** — The **journal, fully realized**. The grid dissolves into
  **one soft card per calendar day**, newest first. The card headline is the
  **day's average** in the condensed display face; morning/evening readings
  become **paired sub-pills inside the card** with their times and the
  **intra-day range**; a **day-over-day delta chip** (down = accent, up = muted)
  sits in the corner. Single-reading days collapse to a slim card with no pair.
  Borrowing **Things** and **Cardiogram**: large leading, generous spacing,
  accent only on the delta chip. → the airy answer; the only idiom that makes
  multi-per-day *the structure*, not an annotation.

- **timeline-rail** — The **history as a time spine**. A vertical rail runs down
  the left; each **date is a node** on it, and a day's readings cluster at that
  node (two readings = two beads on one node, with the spread between them). The
  delta reads as **movement along the rail** — the next node sits with a short
  accent/muted connector showing the day-over-day change. Borrowing **GitHub's
  activity timeline** and **Oura**: medium type, the **rail itself is the
  structural color**, generous vertical rhythm. → reframes the log as *chronology*
  rather than rows, and makes the day boundary (the late-PM / early-AM case)
  visually unambiguous.

- **sparkline-rows** — The **history that doubles as a chart**. Each **day-group
  row** carries a tiny inline **sparkline** beside the number — either that day's
  readings or a trailing 7-day micro-trend — so scanning the list *is* glancing
  at the trend, at row resolution. The number stays in the condensed display
  face; the spark is drawn in the accent. Borrowing **Robinhood** and
  **Whoop**'s list rows: mid-dense, restrained, the accent spent entirely on the
  inline trend. → bridges table and chart; the answer for someone who wants the
  list to *show* motion, not just state.

- **stat-ledger** — The **summary-led list**. A compact **summary band** sits
  atop the region — this-week average, the delta, a **logged-day streak**, maybe
  "X lb to go" pulled from the goal — and the list **below** is stripped to the
  essentials (day, value, a quiet delta). Strong **type-scale contrast**: big
  band numerals in the display face, small calm rows under them. Borrowing
  **TrendWeight**'s table-plus-summary and **Copilot Money**'s account headers:
  the accent lives **on the band**, the rows stay neutral. → answers
  *spreadsheet-y* by giving the log a **headline** instead of a header row, so the
  region opens with meaning before it lists data.

## References

In-system, so "what to take" from each is **structural** — composition, density,
and data legibility, not their palettes or type:

- **Linear** (dense issue lists) — take its **earned table density**: tabular
  alignment, hairline grouping, single-accent restraint, keyboard-scannable rows.
  Drives `ledger-dense`.
- **Things** (to-do day sections) — take its **soft per-day card grouping** and
  generous, calm leading. Drives `day-card-feed`.
- **Cardiogram** (heart-rate day entries) — take its **grouping of multiple
  same-day readings under one day** with a per-day summary. Reinforces
  `day-card-feed`.
- **GitHub activity timeline / Oura** — take the **vertical time spine** with
  events as nodes and change read as movement along it. Drives `timeline-rail`.
- **Robinhood / Whoop list rows** — take the **inline per-row sparkline** that
  turns a list into a glanceable trend. Drives `sparkline-rows`.
- **TrendWeight** (weight table) and **Copilot Money** (account header band) —
  take the **summary band atop a stripped list**, where the headline stats carry
  the accent and the rows stay quiet. Drives `stat-ledger`.

## Selection criteria

A note-to-self for the pick, not a rubric the worker optimizes against. When I
compare these I'm trying to decide:

- **Does it finally stop feeling like a spreadsheet** while staying *fast to
  scan* and *quick to add to*? The toolbar's `Log` action can't get buried in
  service of a prettier list.
- **Is multiple-readings-per-day now intentional** — a morning/evening pair under
  one day with a visible intra-day spread — instead of two duplicate-looking
  rows? This is the core job; the variant that nails it is doing the work.
- **Does day-over-day direction read instantly** — up vs down, via the
  accent/muted split — without me decoding a number?
- **Does the late-night / early-morning day boundary** (the 11:24 PM vs 12:44 AM
  case) land on the *right calendar day* and look deliberate?
- **Does it hold up at both breakpoints** — the desktop list and the mobile
  card/action-sheet — and survive the **20-row pagination boundary** without
  splitting a day across pages confusingly?
- **Does it still read as Prog Strength** — native to the dark slate / violet /
  Nunito system, accent spent on meaning — rather than a re-skin or a second
  palette?
- **Does it sit right *under the shipped `trend-band-analyst` chart*** — the log
  is the lower half of an already-redesigned page, so the winner has to feel like
  it belongs beneath that chart, not like a different app's table.

---

> **Lifecycle.** `status:` is editorial — the owner is the dispatch gate. It
> moves `draft` → `exploring` (worker running) → `awaiting_selection` (draft PR
> open, owner deciding) → `selected` / `abandoned`. The worker sets
> `awaiting_selection` on the `dx/bodyweight-readings-table` branch as it opens
> the PR; the owner sets the terminal value when they close it.
>
> **Handoff.** This DX ends at a *chosen variant*, not merged code. I open the
> draft `[DX — DO NOT MERGE]` PR, compare the variants on the preview deploy,
> pick one, tick its box, set `status: selected` (noting the winning idiom), and
> **close the PR — never merge it.** Then I open a SOW: *"implement the bodyweight
> readings table per the `<chosen-idiom>` variant from
> `dx/bodyweight-readings-table`, production-quality, conforming to the design
> system."*
