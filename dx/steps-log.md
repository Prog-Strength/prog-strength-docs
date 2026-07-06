---
type: dx
status: draft
surface: steps-log
idioms:
  - week-accordion
  - month-chapter
  - timeline-rail
  - summary-band-rows
  - week-card-pages
references:
  - Apple Fitness
  - Gentler Streak
  - Whoop
  - Linear
  - GitHub
  - Copilot Money
scope: in-system
variant_count: 5
repos:
  - prog-strength-web
  - prog-strength-docs
---

# DX: Steps Log

**Status**: Draft · **Last updated**: 2026-07-05

> A DX (Design Exploration) is the platform's **divergent** work type. Unlike a
> SOW it does not converge on one correct implementation — it produces N
> differentiated visual variants of a single frontend surface, side by side on
> one comparison route, awaiting a human pick. It **never merges** and ships no
> production code; the chosen direction feeds a downstream SOW that builds it for
> real.

## Context

The Steps tab of `/activities` (`?view=steps`) shipped its hero treatment from
[`dx/steps-view`](./steps-view.md) — the `goal-ring-hero` direction was selected
and built by
[`sows/steps-view-goal-ring.md`](../sows/steps-view-goal-ring.md). That work
fixed the **headline** — the progress ring, goal-relative bars, and supporting
Days-hit / Best / Total strip now answer "am I hitting my goal?" at a glance. But
it treated the **log region** as one element among many, and the direction that
won wasn't the one (`sparkline-ledger`) that re-thought the log. So the log
shipped essentially as a flat list: every logged day is an identical ring-row,
stacked newest-first, with a **Load more** button that appends indefinitely.

That reads as a **wall of text** once a user has more than a few weeks of
history — especially on `Last 30 days` or `All`, where the screenshot shows
dozens of visually identical rows with no hierarchy. The ring-row treatment
helped each day, but it didn't give the list any **structure**: no week
boundaries, no sense of "how was this week vs last week," no monthly rollup when
the timeframe spans June and July, and no real pagination — just an uncapped
scroll that grows forever.

This DX narrows the lens to **just the log region** and explores it properly.
It matters now because the log is the lower half of an otherwise purpose-built
steps view. The ring and chart answer trend and attainment; the log should
answer **"what happened, day by day — and how did each week and month add up?"**
Today it just lists days and leaves the user to do the mental aggregation.

`scope: in-system`: the foundation is decided (design-system v0.4, oura-calm) —
soft near-black ramp, single periwinkle accent, Manrope, goal-meaning via accent
and success-green. Variants do **not** re-litigate palette, accent, or type. They
diverge on **layout, structure, density, and composition** — how weeks and months
are grouped, what the aggregate summaries look like, and how pagination caps the
list. No hero-ring, bar-chart, or page-chrome changes here; this is the log
region only.

## The surface

The log region of the Steps tab, today rendered in
`prog-strength-web/components/activities/steps-view.tsx` (the `<section>` below
the Daily steps chart, roughly lines 232–255: the Log-steps toolbar, the
`StepsTable` ring-rows, and the keyset **Load more** control). A variant owns
this whole block and may reorganize any of it, but must account for all of it:

- **Log toolbar** — a `✎ Log steps` action (pencil affordance, opens the
  create/edit modal) on the left, and a goal affordance on the right
  (`◎ Goal: 10,000`, or `Set steps goal` when no goal is set). Sits directly
  above the list.
- **The list** — one entry per logged calendar day, sorted **most-recent-first**.
  Each row today carries a **per-day mini-ring**, the date (`Sun, Jul 5`), the
  step count (`17,000`), an optional **goal-%** (`170%`), and **edit + delete**
  icon buttons. Rows are card-like (`border`, `rounded`, `bg-[var(--surface)]`),
  stacked with `gap-2`.
- **Pagination** — keyset via `listSteps(token, { limit, before })`, `PAGE_SIZE =
  25`, **Load more** appends the next page. There is no page cap, no week/month
  boundary awareness, and no "page N of M" — the list grows until the user stops
  clicking.

The data shape (from `prog-strength-web/lib/api.ts`):

```ts
type StepsEntry = {
  id: string;
  date: string;  // YYYY-MM-DD — a calendar day, no time-of-day
  steps: number;
  created_at: string;
  updated_at: string;
};

type StepsGoal = {
  goal: number;  // the user's daily step target, e.g. 10,000
};

type StepsPage = {
  steps: StepsEntry[];
  next_before: string | null;  // keyset cursor for the next page
};
```

Endpoints: `GET /activities/steps` with optional `since` / `until` (range mode,
used by the ring/chart above this region) or `limit` / `before` (keyset mode,
used by the log). `GET /me/steps-goal` → `StepsGoal`. Everything a variant
wants to *add* — **week grouping**, **month grouping**, **week avg / total /
days-hit**, **month avg / total / days-hit**, **week-over-week delta**, a
**month-over-month delta** — is derived **client-side** from the flat
`{ date, steps }` list plus the single `goal`; the API returns only daily totals.
One entry per calendar day (no intra-day complication).

**Week and month bucketing.** Use **Monday-start calendar weeks** (consistent with
the dashboard activity streak's Mon→Sun `week[]` and the profile-stats
`week_start` convention elsewhere in the product). A week bucket is the Mon–Sun
span containing the entry's local calendar day. A month bucket is the calendar
month (`YYYY-MM`). Partial weeks at the start or end of a selected period still
get a week header — label them honestly (`Jun 30 – Jul 6`, not "Week 27"). When
the active period filter (`Last 7 / 30 / 90 days · All`, passed as `days` from
the parent Activities page) spans **two or more calendar months**, variants must
show **month-level aggregate headers** in addition to week groups. When the period
fits in one month (`Last 7 days` in mid-July), month headers may be omitted —
week grouping alone is enough.

**Aggregate fields** (derived per week and per month, goal-relative fields null
when no goal):

| Field | Week | Month |
| ----- | ---- | ----- |
| `days_logged` | count of entries in bucket | count of entries in bucket |
| `days_hit` | entries with `steps ≥ goal` | entries with `steps ≥ goal` |
| `total_steps` | sum of `steps` | sum of `steps` |
| `avg_steps` | `total / days_logged` (not calendar days) | `total / days_logged` |
| `attainment_pct` | `round(avg / goal × 100)` | `round(avg / goal × 100)` |
| `best_day` | `{ date, steps }` max in bucket | `{ date, steps }` max in bucket |

**Color logic (in-system).** The list is neutral by default; the **periwinkle
accent and success-green earn their place on goal-meaning** — a week/month
attainment at or over 100%, a day hit, the active expanded group — never as a
generic row tint. Under-goal aggregates stay muted slate. Steps has no
`--discipline-*` hue; reuse the ring-row goal logic from the shipped tab.

**Visual states the variants must handle** (render them all in the mockup):

- **Multi-week, multi-month history.** The fixture spans late June through early
  July so week headers and a **July** month divider both appear under `Last 30
  days`. The layout must not feel like 30 identical rows.
- **A strong week vs a weak week.** One week averages ~12k (mostly hits); the
  next averages ~4k (mostly misses). The contrast must read from the week
  summary without expanding every day.
- **Partial weeks.** The oldest week in a `Last 7 days` window may have only 3
  logged days; the week header still shows `3/3 days hit` (logged denominator),
  not `3/7`.
- **Sparse logging.** A week with 2 entries among 7 calendar days — aggregates
  use the **logged-days denominator** (matching the shipped `daysHit/daysLogged`
  strip), not calendar days in the period.
- **No goal set.** Goal affordance shows `Set steps goal`; week/month aggregates
  omit attainment % and days-hit; day rows lose the goal-% column gracefully.
- **Pagination boundary.** A variant must **cap visible content** — no infinite
  Load-more wall. Define whether pagination is by **week groups**, **month
  chapters**, or a **fixed row count**, and ensure a week group is **never split
  across two pages** (carry the whole week to the next page, or paginate at week
  boundaries only).
- **Empty / first entry.** Zero entries → the existing empty state ("No steps
  logged yet" + Log CTA). One entry → one day visible inside its week group; the
  week summary still renders without looking broken.
- **Both breakpoints.** Desktop and mobile — week/month headers, summaries, and
  day rows must stay scannable on a narrow phone.

**Representative fixture** (mirror the screenshot — goal 10,000, `Last 30
days` active, multi-week history crossing June→July). Recent entries, newest
first; extend to ~35 logged days so pagination and multi-month grouping both
show. Only logged days appear (gaps are not rows).

| date (local) | steps  | notes |
| ------------ | ------ | ----- |
| Sun, Jul 5   | 17,000 | current week, big beat |
| Sat, Jul 4   |  2,000 | current week, miss |
| Fri, Jul 3   |  8,500 | current week, near-miss |
| Thu, Jul 2   | 11,200 | current week, hit |
| Wed, Jul 1   | 10,400 | current week, hit — **July** month starts |
| Tue, Jun 30  |  6,100 | prior week, miss |
| Mon, Jun 29  |  9,800 | prior week, near-miss |
| Sun, Jun 28  | 12,500 | prior week, hit |
| Sat, Jun 27  | 13,000 | prior week, hit |
| Fri, Jun 26  |  4,200 | prior week, miss |
| Thu, Jun 25  |  3,800 | prior week, miss |
| Wed, Jun 24  | 14,000 | prior week, best day in fixture |
| Tue, Jun 23  | 10,100 | prior week, hit |
| Mon, Jun 22  | 11,000 | prior week, hit |
| …            | …      | continue ~20 more days into mid-June for pagination |

Derived examples a variant can show:

- **Week of Jun 30 – Jul 6** (partial current week): avg ≈ **9,867** from 6
  logged days, **4/6 days hit**, total 59,200, best Jul 5 17,000.
- **Week of Jun 23 – Jun 29**: avg ≈ **9,486**, **5/7 days hit**, a strong week.
- **Week of Jun 16 – Jun 22**: avg ≈ **5,200**, **1/6 days hit**, a weak week —
  the contrast the grouping should surface.
- **July 2026** (month rollup): avg, total, days-hit across all July entries in
  the window.
- **June 2026** (month rollup): same for June entries.

These are mockups: **static fixtures that look real are preferred** — do not
wire the variants to live steps services. Mock pagination with a `hasMore` flag
and 2–3 pages of fixture data.

## Idioms

Five genuinely distinct compositions of the *same* near-black / periwinkle /
Manrope surface. None re-decides palette, accent, or type — they diverge along
**type scale** (how hard week/month summaries lean on big tabular figures vs
small ring-rows), **color logic** (how accent and success-green mark
goal-attainment at week/month/day levels), and **spacing rhythm** (airy
accordion ↔ dense interstitial bands ↔ card-per-week). Each makes a **different
element the hero** of the log history.

- **week-accordion** — Heroes **the collapsible week**. Each Monday–Sunday span
  is an accordion section: a **week header row** carries the date range, **week
  avg** (large tabular), **days hit** (`4/6`), and a small **week total**; a
  chevron expands to reveal the **day ring-rows** inside. **Month dividers**
  (e.g. `July 2026 — avg 10,200 · 8/12 days hit`) appear as sticky-ish section
  breaks when the period spans months. **Pagination = N weeks per page** (e.g. 4
  weeks), never splitting a week. Default: **current week expanded**, older weeks
  collapsed so the list opens with one week of days, not thirty rows. Type scale:
  week header is the anchor; day rows stay small. Color logic: success-green on
  the week header when `attainment_pct ≥ 100%`. Spacing rhythm: generous padding
  inside expanded weeks, tight collapsed headers. Borrowing **Apple Fitness**
  activity history sections and **Gentler Streak**'s week recap. → the scannable
  default: summaries first, detail on demand.

- **month-chapter** — Heroes **the month rollup**. When the period spans multiple
  months, each month is a **chapter**: a prominent **month banner** (avg, total,
  days-hit, best day) opens the chapter, then **week sub-headers** (lighter
  weight, one line each) nest inside, then day rows at the leaf. Single-month
  periods (`Last 7 days`) collapse to week sub-headers only — no empty month
  chrome. **Pagination = one month chapter per page** on `All` / long ranges; on
  `30d` a single page may hold two short chapters. Type scale: month banner uses
  the largest figures in the log region; week lines are caption-sized. Color
  logic: accent on the month avg when it's the period's headline number. Spacing
  rhythm: airy between chapters, tighter within. Borrowing **Copilot Money**
  account statements and **Whoop** monthly recap emails. → answers "how was
  July vs June?" before "what happened Tuesday?"

- **timeline-rail** — Heroes **chronology along a spine**. A vertical rail runs
  down the left; each **week is a node** on it, labeled with the week avg and
  days-hit. **Day ring-rows cluster as beads** at the active week's node (one
  week visible expanded at a time, or all weeks shown as nodes with days tucked
  in). **Month transitions** are milestone labels on the rail (`July`). The
  inter-week **connector** can encode week-over-week avg delta (accent when up,
  muted when down). **Pagination = N week-nodes per page.** Type scale: the
  node's week avg is the size anchor. Color logic: the rail + connectors carry
  structural color; success on nodes whose week cleared goal. Spacing rhythm:
  generous vertical rhythm between week nodes. Borrowing the shipped bodyweight
  [`timeline-rail`](../sows/bodyweight-readings-table-timeline-rail.md) idiom and
  **GitHub**'s activity timeline. → reframes the log as *time moving*, not a flat
  stack.

- **summary-band-rows** — Heroes **interstitial summary bands** inside a
  continuous list. The day ring-rows stay flat (closest to today's shipped
  list), but **non-collapsible band rows** are inserted at week and month
  boundaries — a full-width **week summary strip** (`Jun 23 – Jun 29 · avg 9,486
  · 5/7 hit · 66.4k total`) and a heavier **month divider strip** when the
  month changes. Bands are not expandable; every day row remains visible on the
  current page. **Pagination = fixed day count per page (e.g. 20)** with the rule
  that a page break **never falls inside a week** — if the 20th row mid-week,
  the page includes the rest of that week. Type scale: bands use small-caps
  labels + medium tabular figures; day rows unchanged. Color logic: accent/success
  only on the band's attainment figure. Spacing rhythm: dense — bands are
  hairline-separated, minimal vertical gap. Borrowing **Linear**'s grouped issue
  lists and **Robinhood**'s interstitial earnings rows. → the smallest diff from
  today's wall: structure without hiding days.

- **week-card-pages** — Heroes **the week card as the page unit**. Each week
  renders as a **self-contained card**: the card header is the week summary
  (avg in a mini-ring, days-hit, total); the card body lists that week's day
  rows. Cards stack newest-first; **only 3–4 week cards** appear per page, with
  **Prev / Next** (or page dots) below — never a Load-more that appends without
  bound. Month labels sit as a slim **eyebrow above the first week card** of
  each month. Tapping a card header collapses/expands the day list (optional —
  default expanded for the current week, collapsed for older). Type scale: card
  header ring is the visual anchor; month eyebrow is whisper-small. Color logic:
  card border tint shifts success when the week cleared goal. Spacing rhythm:
  card gaps (`gap-4`), breathable. Borrowing **Whoop** week-in-review cards and
  **Apple Fitness** Awards tiles. → pagination by week cards makes the cap
  feel natural: one screen holds a handful of weeks, not thirty days.

## References

In-system, so "what to take" from each is **structural** — composition, density,
and data legibility, not their palettes or type:

- **Apple Fitness** (activity history) — take its **period sections** that
  summarize a week before you drill into days, and the **ring-at-a-glance**
  summary size. Drives `week-accordion` and `week-card-pages`.
- **Gentler Streak** (weekly recap) — take its **gentle week-level framing**
  (avg, days on target) without scolding on a miss week. Reinforces
  `week-accordion`.
- **Whoop** (monthly / weekly performance emails) — take the **month-as-chapter**
  hierarchy where the long-range story comes first. Drives `month-chapter` and
  `week-card-pages`.
- **Copilot Money** (statement sections) — take the **section header band** that
  orients you inside a long scroll before the line items. Drives `month-chapter`.
- **Linear** (grouped lists) — take **hairline interstitial group headers** that
  break a dense list without collapsing detail. Drives `summary-band-rows`.
- **GitHub** (activity timeline) — take the **vertical spine with milestone
  labels** where time reads as movement. Drives `timeline-rail`.

## Selection criteria

A note-to-self for the pick, not a rubric the worker optimizes against. When I
compare these I'm trying to decide:

- **Does the log finally stop feeling like a wall of text** while keeping every
  day **editable** (edit/delete) and **fast to scan**? The `Log steps` action
  can't get buried behind collapsed chrome.
- **Do week summaries earn their place** — can I tell a strong week from a weak
  one without expanding every day — and do **month rollups** appear when June
  and July are both in view?
- **Is pagination bounded** — no infinite Load-more growth — and does a page
  break **never split a week** confusingly?
- **Does partial-week / sparse-logging math read honestly** (`4/6 days hit`, not
  `4/7`) matching the shipped ring strip's logged-days denominator?
- **Does the no-goal case degrade gracefully** — no `NaN%` on week/month bands,
  day rows still usable?
- **Does it hold up at both breakpoints** and across the period filter — a sparse
  `Last 7 days` (one week group) and a long `All` (many month chapters / pages)
  both looking intentional?
- **Does it sit right under the shipped goal-ring hero and chart** — the log is
  the lower third of an already-redesigned tab, so the winner must feel like it
  belongs beneath that ring, not like a different app's table.

---

> **Lifecycle.** `status:` is editorial — the owner is the dispatch gate. It moves
> `draft` → `exploring` (worker running) → `awaiting_selection` (draft PR open,
> owner deciding) → `selected` / `abandoned`. The worker sets `awaiting_selection`
> on the `dx/steps-log` branch as it opens the PR; the owner sets the terminal
> value when they close it.
>
> **Handoff.** This DX ends at a *chosen variant*, not merged code. I open the
> draft `[DX — DO NOT MERGE]` PR, compare the variants on the preview deploy,
> pick one, tick its box, set `status: selected` (noting the winning idiom), and
> **close the PR — never merge it.** Then I open a SOW: *"implement the steps
> log per the `<chosen-idiom>` variant from `dx/steps-log`, production-quality,
> conforming to the design system."*
