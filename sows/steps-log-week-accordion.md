---
status: draft
repos:
  - prog-strength-web
  - prog-strength-docs
---

# Steps Log — Week-Accordion

**Status**: Draft · **Last updated**: 2026-07-06

> Frontend SOW. It implements the chosen variant from
> [`dx/steps-log`](../dx/steps-log.md) and therefore inherits that DX's
> **`scope: in-system`**. The visual foundation (design-system **v0.4**,
> oura-calm) is already decided, so this SOW **conforms** to it; it does **not**
> re-tone the system or touch shared tokens. The work is in `prog-strength-web`;
> `prog-strength-docs` flips the DX to `selected` and marks this SOW shipped on
> merge.

## Introduction

The Steps tab of `/activities` (`?view=steps`) shipped its hero treatment from
[`dx/steps-view`](../dx/steps-view.md) ([`sows/steps-view-goal-ring.md`](./steps-view-goal-ring.md),
the `goal-ring-hero` direction). That work fixed the **headline** — the progress
ring, goal-relative bars, and Days-hit / Best / Total strip answer "am I hitting
my goal?" at a glance. But it treated the **log region** as one element among
many, so the log shipped as a flat, unbounded newest-first list of identical
ring-rows with a **Load more** button that appends forever — a wall of text with
no week boundaries, no strong-week-vs-weak-week signal, and no monthly rollup when
the history crosses months.

[`dx/steps-log`](../dx/steps-log.md) (explored in
[prog-strength-web#111](https://github.com/Prog-Strength/prog-strength-web/pull/111))
narrowed the lens to just the log region and produced five variants. The owner
selected **`week-accordion`**. This SOW builds it for real, production-quality,
conforming to the design system.

The ring and chart answer trend and attainment; the log should answer **"what
happened day by day — and how did each week add up?"** `week-accordion` reframes
the log as **summaries first, detail on demand**: each Monday–Sunday span is a
collapsible section whose header carries the week avg, days-hit, and total; the
current week opens expanded by default, older weeks stay collapsed; month
dividers appear when the loaded history spans two calendar months; pagination is
**four weeks per page**, never splitting a week.

## Proposed Solution

Replace the log region's flat `StepsTable` + unbounded **Load more** with a new
client component (e.g. `components/activities/steps-log-accordion.tsx`) rendering
the `week-accordion` idiom, wired to the existing Steps tab orchestration.

- Each **Monday-start week** (Mon–Sun, matching the dashboard streak convention)
  is an **accordion section**: a clickable header row with chevron, date range
  (`Jun 29 – Jul 5`), **week avg** (large tabular, success-green when
  `attainment_pct ≥ 100%`), **days hit** (`4/6`, logged-days denominator),
  **week total**, and a chevron. Expanding reveals that week's **day ring-rows**
  (mini-ring, date, steps, goal-%, edit/delete) — the same row atoms the tab
  ships today.
- **Month dividers** (`July 2026 — avg 10,200 · 8/12 hit`) insert when the
  visible page's weeks cross a calendar-month boundary and the loaded history
  spans **two or more months**. Single-month views omit month chrome.
- **Default expansion**: only the **newest (current) week** is open on first
  render; older weeks on the page are collapsed so the region opens with one
  week of day rows, not thirty.
- **Pagination**: **4 weeks per page**, whole-week packing only — a page never
  splits a week. **← Newer / Older →** controls with `Weeks · page N of M`
  replace Load more. Moving to an older page that needs weeks not yet fetched
  triggers the existing keyset `listSteps` cursor until those weeks are available
  or the history is exhausted.
- The **log toolbar** above the region (`Log steps` + goal affordance) and the
  Log / Edit / Goal **modals** stay owned by `StepsView`; the accordion receives
  entries + goal as props and emits edit/delete intents back up.

Add a pure grouping helper (e.g. `lib/steps-grouping.ts`, unit-tested in
isolation) porting the mockup's `aggregate`, `bucketByWeek`, and
`spansMultipleMonths` from `app/design-explore/steps-log/_data.ts`. Reuse
`parseLocalDate` / `isoDate` / `dayPct` from `lib/steps-stats.ts` where they
already exist — do not fork date parsing.

No hero-ring, bar-chart, or API changes. Everything the accordion adds is derived
client-side from the flat `{ date, steps }` list plus the goal, exactly as the
ring layer already does.

## Goals and Non-Goals

### Goals

- **Add a pure, tested steps-grouping module** (e.g. `lib/steps-grouping.ts`):
  - **`aggregate(entries, goal)`** → `{ daysLogged, daysHit, total, avg,
    attainmentPct, best }` — goal-relative fields **null when no goal**;
    `daysHit` counts logged days with `steps ≥ goal`; `avg = round(total /
    daysLogged)`.
  - **`bucketByWeek(entries, goal)`** → `WeekBucket[]` newest-first, each with
    `key` (Monday ISO), `label` (`Jun 29 – Jul 5`), `entries` (newest-first),
    and `agg`.
  - **`spansMultipleMonths(entries)`** → boolean for month-divider visibility.
  - Monday-start weeks via `weekStart(date)`; local-date parsing consistent
    with `steps-stats`.
- **Replace `StepsTable` + Load more with `week-accordion`**, conforming to v0.4:
  - **Week header** — chevron, range label, caption line (`59,200 total · 4/6
    days hit`), **26px tabular week avg** with `avg / day` caption; success-green
    avg when week attainment ≥ 100%.
  - **Month divider** — `surface-2` band with month name + rollup (`avg · X/Y
    hit`) when `spansMultipleMonths` and the month changes between adjacent weeks
    on the page.
  - **Day rows inside an expanded week** — preserve shipped mini-ring, date,
    steps, goal-%, edit/delete; same card styling as today (`background` surface
    inside the accordion panel).
  - **Accordion default** — newest week expanded, others collapsed; toggling
    persists for the session (local `useState`, no URL sync required).
  - **Week pagination** — `WEEKS_PER_PAGE = 4`; Prev/Next pager hidden when
    `pageCount ≤ 1`; labels `← Newer` / `Older →` and `Weeks · page N of M`.
- **Preserve the tab's orchestration outside the log list**:
  - Dual fetch split unchanged: **range fetch** (`since`/`until` from `days`)
    powers ring/chart/stats; **keyset log fetch** (`limit` / `before`) powers
    the accordion's entry pool.
  - `upsertStepsForDate` / `deleteStepsForDate` / `putStepsGoal` with
    **refetch after mutation**; auth/401 redirect; Log/Edit/Goal modals
    unchanged in behavior.
  - Log toolbar (`Log steps` + goal affordance) stays directly above the
    accordion — not buried inside collapsed chrome.
- **Handle every state the DX enumerated**: strong vs weak week readable from
  collapsed headers; partial/sparse weeks (`4/6 hit`, not `4/7`); multi-month
  dividers; **no goal** → no attainment % / days-hit on headers or day rows, no
  `NaN%`; empty → existing `EmptyState`; single entry → one week, expanded;
  both breakpoints (accordion headers and day rows scannable on mobile).
- **Tests** — grouping module fixtures mirroring the DX ticket (strong week Jun
  22–28, weak week Jun 15–21, partial current week, no-goal nulls); component
  tests for week header avg + success clear, month divider insertion, default
  expansion, pager visibility, edit/delete still wired; **CI green**
  (lint/format/typecheck/test/build).

### Non-Goals

- **Any backend / API / SDK change.** Everything is derived client-side from
  existing `listSteps` (range + keyset) and `getStepsGoal`. `prog-strength-api`
  is **not** in `repos:`.
- **Any design-system or shared-token change.** `scope: in-system` against v0.4
  — conform only; no token/accent/type edit, no `design-system.md` change.
- **Hero ring, Daily steps chart, or Activities page chrome** — period filter and
  tab bar are untouched.
- **Scoping the log to the `days` period filter.** Today the log keyset fetch is
  **all history** while the ring/chart respect `days`; this SOW preserves that
  split. Aligning log visibility to the period filter is a follow-up.
- **Promoting the DX mockup code.** `week-accordion.tsx` / `_data.ts` / `_ui.tsx`
  on the `dx/steps-log` branch are the visual spec, not code to copy; the
  `design-explore/steps-log` route stays gated and never ships. PR #111 is
  **closed, never merged**.

## Implementation Details

### Design-system conformance (read first)

Conform to [`design-system.md`](../design-system.md) **v0.4** — reference tokens,
never duplicate hex.

- **Accent / success**: periwinkle `--accent` for in-progress goal fill;
  `--success` when day/week attainment clears goal. Under-goal aggregates stay
  foreground/muted — not danger-red on a normal low-step week.
- **Typography**: Manrope; week avg uses tight tabular tracking (`-0.03em`);
  month divider uses small-caps (`11px`, `tracking-[0.16em]`).
- **Surface**: accordion shell `surface` + hairline border; expanded day rows on
  `background`; month divider on `surface-2`.

### Surface being replaced

In `components/activities/steps-view.tsx`: the log `<section>`'s `StepsTable`,
its `loadMore` / `loadingMore` / `nextBefore` wiring, and the **Load more**
button. The toolbar row (`Log steps` + `GoalAffordance`) and modals stay in
`StepsView`; only the list body swaps.

### Grouping module (`lib/steps-grouping.ts`)

Port the mockup's derivation from `app/design-explore/steps-log/_data.ts`:

```ts
type Aggregate = {
  daysLogged: number;
  daysHit: number | null;
  total: number;
  avg: number;
  attainmentPct: number | null;
  best: { date: string; steps: number };
};

type WeekBucket = {
  key: string;       // Monday YYYY-MM-DD
  start: Date;
  end: Date;
  label: string;     // "Jun 29 – Jul 5"
  entries: StepsEntry[]; // newest-first
  agg: Aggregate;
};
```

Import `parseLocalDate`, `isoDate`, and `dayPct` from `steps-stats.ts` rather
than re-declaring them. Table-driven Vitest over the DX fixture anchors: Jun
22–28 strong week (`5/7 hit`, avg ≈ 9800), Jun 15–21 weak week (`1/6 hit`, avg ≈
5750), partial current week, `spansMultipleMonths` true for June+July fixture,
no-goal nulls throughout.

### Accordion composition

Mirror `app/design-explore/steps-log/_variants/week-accordion.tsx`:

- Outer shell: `divide-y divide-[var(--border)]` inside a rounded `surface`
  card.
- **`WeekSection` header** — full-width `<button aria-expanded>`; chevron rotates
  when open; right-aligned **26px avg** with `avg / day` caption below.
- **Expanded panel** — `ul` of day ring-rows with `gap-2`, `px-4 pb-4 pt-1`.
- **`MonthDivider`** — rendered before the first week on a page whose entries
  fall in a new calendar month (compare month of adjacent weeks on the current
  page); rollup computed from **all loaded entries** in that month, not only the
  weeks visible on this page.
- Wire **real** edit/delete handlers (mockup's `DayActions` is inert); preserve
  `onEdit(entry)` / `onDelete(date)` from today's `StepsTable`.

### Pagination and keyset fetch

Replace unbounded Load more with **week-index pagination**:

1. **`StepsView` keeps accumulating** keyset pages in `logEntries` (as today).
2. **`bucketByWeek(logEntries, goal)`** produces the full week list newest-first.
3. **UI page** = slice `[page × 4 … page × 4 + 4)` of week buckets — never split
   a week.
4. **Older navigation**: if the requested page needs week buckets whose Monday is
   **before** the oldest entry currently loaded **and** `nextBefore !== null`,
   fetch the next keyset page first (show loading on the pager), append, re-bucket,
   then render. Repeat until the page can render or history is exhausted.
5. **Reset page to 0** on refetch after mutation or on first mount.

Remove the Load-more button entirely. Hide the pager when `pageCount ≤ 1`.

### States to handle (render/verify all)

- **Strong vs weak week** — collapsed headers alone distinguish Jun 22–28 from
  Jun 15–21 (avg + days-hit visible without expanding).
- **Partial / sparse week** — `4/6 days hit` on a week with six logged days, not
  `4/7`.
- **Multi-month** — July divider appears between June and July weeks when both
  months are present in loaded history; omitted when history is single-month.
- **Default expansion** — only newest week open; toggling an older week does not
  collapse the current week unless the user closes it.
- **No goal** — headers show avg + total only; day rows omit goal-%; no `NaN%`.
- **Empty** — zero `logEntries` → existing `EmptyState` (accordion not rendered).
- **Single entry** — one week section, expanded, one day row.
- **Mobile** — week header avg and days-hit remain readable; day rows don't clip
  edit/delete.

## Testing

- **`lib/steps-grouping.ts` (Vitest)** — `aggregate`, `bucketByWeek`,
  `spansMultipleMonths`, Monday-start boundary (Sun Jun 29 belongs to week
  starting Jun 23); no-goal nulls; empty input → empty buckets.
- **Component (`steps-log-accordion.test.tsx` or extend `steps-view.test.tsx`)**
  — week header renders avg and success-green at ≥100% attainment; month divider
  when fixture spans months; newest week expanded by default; pager hidden for ≤4
  weeks; edit/delete callbacks fire from day rows.
- **Integration in `StepsView`** — refetch after save/delete still re-buckets;
  navigating to an older week page triggers keyset fetch when needed.
- `lint`, `typecheck`, and the project build pass.

## Rollout

1. **`prog-strength-docs`** — flip [`dx/steps-log.md`](../dx/steps-log.md) to
   `status: selected` (winning idiom `week-accordion`); mark this SOW `shipped`
   on merge. Close [prog-strength-web#111](https://github.com/Prog-Strength/prog-strength-web/pull/111)
   without merging.
2. **`prog-strength-web`** — grouping module + accordion component +
   `StepsView` swap-in + tests, in one PR. Vercel preview: verify across a
   multi-week account, goal vs no-goal, empty/first-entry, month divider on
   June→July history, week pager on long history, and mobile.

### Verification after rollout

- `/activities` → Steps log reads **summaries first**: collapsed week headers
  show avg + days-hit; expanding reveals ring-rows; the current week opens
  expanded by default — not a wall of thirty identical rows.
- **Month dividers** appear when history crosses months; **strong vs weak weeks**
  are distinguishable from headers alone.
- **Pagination is bounded** — four weeks per page, no Load more; older pages
  fetch via keyset when needed; no week is split across pages.
- Edit, delete, Log steps, and Goal still work with refetch; **no goal** and
  **empty** degrade cleanly.
- Sits beneath the shipped goal-ring hero and chart as the same v0.4 surface;
  `design-explore/steps-log` stays gated; DX PR #111 closed (never merged).

## Open Questions

1. **Grouping module location: `steps-grouping.ts` vs extending
   `steps-stats.ts`.** The ring/chart derivations live in `steps-stats.ts`; week
   bucketing is log-specific. **Lean:** a separate `lib/steps-grouping.ts` that
   imports shared date helpers from `steps-stats.ts` — keeps each module one job.
2. **Month rollup scope on the divider.** The mockup recomputes the month aggregate
   from the full static fixture. In production, **all loaded keyset entries** in
   that month are the practical source — the rollup grows as the user pages older.
   **Lean:** accept that; a month divider on page 1 reflects loaded July entries
   only, not entries not yet fetched. Confirm at review.
3. **Keyset page size vs week pagination.** Today `PAGE_SIZE = 25` flat rows. With
   week packing, 25 rows may be 3–4 weeks. **Lean:** keep `PAGE_SIZE = 25` for
   API calls; UI paginates weeks separately at 4 per page. Increase `PAGE_SIZE`
   only if older-page navigation feels fetch-heavy in preview.
4. **Accordion expansion persistence across refetch.** **Lean:** reset open state
   to "newest week only" on refetch after mutation so a saved edit doesn't leave
   stale expansion state; optional enhancement to preserve toggles is out of scope.
