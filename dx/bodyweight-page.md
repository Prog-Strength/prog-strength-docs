---
type: dx
status: awaiting_selection
surface: bodyweight-page
idioms:
  - trend-band-analyst
  - single-metric-hero
  - weigh-in-journal
  - calm-canvas
  - split-control-room
references:
  - TrendWeight
  - Libra
  - Whoop
  - Oura
  - Apple Health
  - Linear
scope: in-system
variant_count: 5
repos:
  - prog-strength-web
  - prog-strength-docs
---

# DX: Bodyweight Page

**Status**: Awaiting selection · **Last updated**: 2026-06-18

> A DX (Design Exploration) is the platform's **divergent** work type. Unlike a
> SOW it does not converge on one correct implementation — it produces N
> differentiated visual variants of a single frontend surface, side by side on
> one comparison route, awaiting a human pick. It **never merges** and ships no
> production code; the chosen direction feeds a downstream SOW that builds it for
> real.

## Context

The bodyweight page (`/bodyweight`) is where a Prog Strength user watches the
one number that tells them whether a cut or a bulk is actually working. They
step off the scale, log a reading (often more than once a day — a morning and an
evening weigh-in), and want a single honest answer: *which way am I trending,
and through how much daily noise?* The **functionality is solid and not in
question here** — multi-per-day logging works, the chart already plots
individual readings as dots and a daily-average trend line through them, the
30/60/90/All range tabs filter, the goal line renders when set, and the log
table edits and deletes. What's unsatisfying is the **look and the
composition**.

Three things drag it down today:

- **The chart doesn't make the trend win over the noise.** This is a weight
  graph — the entire job is for the *trend* to read clearly through the
  day-to-day scatter. Right now the raw-reading dots (loud amber) compete with
  the daily-average line for attention, so the surface reads as "a cloud of
  points with a line in it" rather than "here's your trajectory, here's the
  spread around it." Apps built for this (TrendWeight, Libra) solve it by making
  the smoothed trend the unmistakable subject and pushing the raw readings to a
  faint background.
- **The log reads spreadsheet-y.** `DATE / TIME / WEIGHT` columns of small text
  with pencil/trash glyphs — a database table, not a record of weigh-ins you
  keep. It doesn't express the things that actually matter in a weight log: the
  delta since last time, that two readings belong to the *same day*, whether
  today was up or down. Same complaint the nutrition log earned before its
  redesign.
- **The composition is a generic analytics stack.** Chart, then a row of four
  flat stat tiles (Average / Goal / Min / Max), then a table — a layout that
  could belong to any metric. Nothing is the hero, so the page never delivers
  the half-second "here's where I'm at and where I'm headed" hit that a
  single-metric product like Whoop or Oura lands immediately.

This DX exists to find the *direction* for a bodyweight page that feels
purpose-built for trend-watching — before committing engineering effort to
building one. `scope: in-system`: the foundation is decided (dark slate ramp,
the single **violet** accent, Nunito, and the Oswald-style condensed display
face the system reserves for big stat numerals), so the variants do **not**
re-litigate palette, accent, or type. They diverge on **layout, structure,
density, and composition** — what the hero is, how the trend is separated from
the noise, and how the weigh-in history is treated. No production bodyweight
code changes here.

## The surface

The `/bodyweight` view (`prog-strength-web/app/(app)/bodyweight/page.tsx`).
Anatomy of what's on screen today (a variant should account for all of it,
though it may reorganize, demote, or fold chrome together in service of its
idiom):

- **Range tabs** — `30 days · 60 days · 90 days · All`, filtering from *now*
  backward; the active tab is accent-filled. Default 30.
- **The chart** — a Recharts `ComposedChart` combining: a **daily-average trend
  line** (one point per calendar day, plotted at noon), a **scatter of raw
  readings** (every individual weigh-in as a dot), and a **dashed goal line**
  when a goal is set. Y-axis is forced to include the goal so it's always in
  frame.
- **Four stat tiles** — **Average** (`183.4 lb`, with the delta over the range,
  e.g. `-0.2 lb (-0.1%)`), **Goal** (`178 lb`, with `5.4 lb to go`), **Min**
  (`179.6 lb`, dated), **Max** (`188.2 lb`, dated). Each tile carries a thin
  colored top strip.
- **Log toolbar** — a `Log` action (opens the create-reading modal) and a goal
  affordance (`Goal: 178 lb`, or `Set goal weight`).
- **The log table** — one row per measurement, **Date / Time / Weight**, sorted
  most-recent-first, with edit + delete affordances (a tap-to-open action sheet
  on mobile). Paginated at 20 rows. Multiple readings on one day appear as
  adjacent rows.

The data shapes (from `prog-strength-web/lib/api.ts`):

```ts
type BodyweightEntry = {
  id: string;
  weight: number;
  unit: "lb" | "kg";
  measured_at: string; // RFC3339 — carries both the day AND the time-of-day
  created_at: string;
};

type BodyweightGoal = {
  weight: number;
  unit: "lb" | "kg";
  created_at: string | null; // null ⇒ goal never set ⇒ no goal line, "Set goal weight"
  updated_at: string | null;
};
```

Endpoints: `GET /bodyweight` → `BodyweightEntry[]`, `GET /me/bodyweight-goal` →
`BodyweightGoal`. **The daily-average series is computed client-side**, not by
the API: readings are grouped by local calendar day, arithmetic-meaned, and the
mean is plotted at noon of that day. Min/Max/Average and the range delta are
likewise derived on the client from the filtered readings. The trend line is a
*plain daily mean*, not a smoothed/weighted average — see the idioms, where a
proper smoothing curve (TrendWeight-style EWMA) is one of the things a variant
may introduce visually.

**Palette note (a discrepancy to reconcile, like the nutrition macro-tints
were).** The chart colors are hardcoded and predate the design system: the trend
line is `#3b82f6` — which is exactly **the legacy blue the system says the
violet accent replaces** — raw dots are amber `#fcd34d`, the goal line green
`#10b981`. Because `scope: in-system`, variants should **re-anchor on the system**:
the **violet accent** is the natural owner of the trend (the subject of the
page), with raw readings demoted to a muted slate and the goal given a
restrained, legible marker. Variants do not invent a loud new multi-color chart
palette — one accent, slate neutrals, and at most one quiet secondary for the
goal.

**Known data bug (out of scope here, fix separately).** The screenshot that
prompted this DX shows the chart tooltip rendering `Daily avg : 1780150948286
lb` — a raw noon **timestamp** leaking into the daily-average value where a
weight belongs (`1780150948286` ms ≈ a 2026 date, i.e. the point's `t`, not its
`avg`). The current `main` formatting/data code appears to plot the correct
field, so this is a number/data-binding bug to **verify against `main` and fix
out-of-band** — it is not part of this visual exploration, but the variants'
tooltips must obviously show a rounded weight (`182.4 lb`), never a timestamp.

**Visual states the variants must handle** (where lazy designs fall apart, so
render them all in the mockup):

- **Trend up vs. down.** The range delta can be negative (cutting, the happy
  case) or positive (gaining). The direction of travel — and whether that's
  *toward or away from* the goal — should read instantly.
- **Heavy daily spread.** A morning reading and a late-night reading can differ
  by 2–3 lb on the same day (the screenshot shows exactly this). The trend must
  stay legible *through* that spread — this is the whole point of the surface.
- **No goal set.** `created_at: null` means no goal line and no "X to go" — the
  hero must degrade gracefully to an inviting "set a goal" affordance, not a
  blank or a broken stat.
- **Above vs. below goal.** Readings can sit above the goal (the screenshot:
  ~185 current vs 178 goal) or below it; the encoding shouldn't assume one side.
- **Sparse vs. dense range.** `All` with months of daily data is dense; a new
  user on `30 days` may have five points with gaps. The trend line connects
  day-averages with visible gaps on sparse data and must not look broken.
- **Empty / first-reading.** Zero readings ("Tap Log to add your first
  reading"), and the one-reading case where there's no trend yet — the page
  should still feel like a place to start.
- **Unit.** Weight is `lb` or `kg` (taken from the most recent entry); the
  treatment can't assume a 3-digit lb number.

**Representative fixture** (mirror the screenshot so variants render realistically
— a multi-per-day month, above goal, trending roughly flat-to-up, goal set):

- Range: **30 days**, ending **Wed, Jun 17 2026**. Unit **lb**. Goal **178**.
- Derived stats: **Average 183.4** (`-0.2 lb / -0.1%` over the range),
  **Min 179.6** (Jun 7), **Max 188.2** (Jun 12), **5.4 lb to go**.
- A spread of daily readings, several days with **two readings** (morning +
  evening) 1–3 lb apart, e.g. **Sat May 30**: readings 182.0 and 182.4 → daily
  avg ~182.2. Recent rows: Wed Jun 17 — 185.0 (9:47 AM); Tue Jun 16 — 185.4
  (9:17 AM); Mon Jun 15 — 187.8 (11:24 PM) *and* an earlier same-day reading.
- The trend line wanders ~180 → 188 across the month while the goal line sits
  flat at 178 below most of it.

These are mockups: **static fixtures that look real are preferred** — do not
wire the variants to live bodyweight services.

## Idioms

Five genuinely distinct compositions of the *same* dark-slate / violet / Nunito
surface. Because `scope: in-system`, none re-decides palette, accent, or type —
they diverge along **type scale** (how hard they lean on the Oswald-style
condensed display face and size vs Nunito weight for hierarchy), **color logic**
(how the single violet accent and a muted slate are deployed to separate trend
from noise), and **spacing rhythm** (airy hero ↔ dense control room). Each makes
a *different element the hero* and leans on a different reference. The Fixed
Points hold: the "P" identity and the dark theme stay.

- **trend-band-analyst** — The **trend wins, decisively**. Borrowing
  **TrendWeight / Libra**, the smoothed trend line (a visible EWMA-style curve,
  not the raw daily mean) becomes the unmistakable subject: drawn in the violet
  accent, thick and confident, wrapped in a faint **±spread band**, while the
  raw readings drop to small, low-contrast slate dots in the background. A small
  **rate-of-change** readout ("−0.4 lb/wk") annotates the curve. The chart is
  enlarged to own the page; stats compress to a thin caption strip beneath. Mid-
  dense, analytical type scale; color logic is *one accent on neutral* — violet
  trend, everything else muted. → directly answers *the trend doesn't beat the
  noise*; the cleanest separation of signal from spread.

- **single-metric-hero** — Borrowing **Whoop / Oura**, the page opens with a
  **big confident hero**: the current weight (or current trend value) in the
  **Oswald-style condensed display face at a huge size**, a direction arrow and
  the range delta beside it, and "X lb to go" as a quiet sub-line — the
  half-second "where am I at" hit. A compact trend **sparkline** sits under the
  hero; the full chart and the log live calmly below. Dramatic type-scale
  contrast (hero numeral huge, everything else small); color logic restrained —
  the accent marks direction/goal-progress, the chart stays minimal. → answers
  *generic analytics stack* by giving the surface an obvious, emotional hero.

- **weigh-in-journal** — The **log is the star**. Weigh-in history becomes a
  dated **journal feed** instead of a four-column grid: each day is a soft
  entry showing the day's value, a **delta chip vs the prior day** (down = accent,
  up = muted), and — when there are two readings — the **morning/evening pair
  grouped under that day** rather than as orphan rows, with a tiny intra-day
  range. The chart shrinks to a slim strip across the top. Comfortable,
  editorial Nunito leading with more air between entries; color logic uses the
  accent only on the delta chips. → answers *spreadsheet-y log* and is the only
  idiom that makes multi-per-day legible **in the history**, not just the chart.

- **calm-canvas** — Borrowing **Apple Health + Linear**, the whole surface is
  **one large, quiet chart**. The four stat tiles dissolve into **on-plot
  annotations** — min, max, average, and the goal are labeled directly on the
  curve where they occur, not in separate boxes — so the graph carries its own
  legend and the page is mostly breathing room. The log folds into a secondary
  drawer/panel reached from the canvas. Generous spacing, minimal chrome; color
  logic is maximally restrained — violet trend, faint slate readings, a single
  hairline goal marker. → answers *generic composition* from the opposite end of
  the hero: subtraction, not a big numeral.

- **split-control-room** — A denser **two-pane** layout (a **Linear**-style
  working surface): the **chart + trend stats occupy one column** and the **full
  weigh-in ledger stays permanently visible** in the other, so trajectory and
  history sit side by side instead of stacked behind a scroll. Multi-per-day is
  handled with grouped sub-rows in the ledger; hovering a row highlights its
  point on the chart. Crisp, functional type; gridded, efficient, keyboard-
  friendly density — the power-user end of the spread. Color logic is structural:
  the accent marks the active/selected reading and the trend, neutrals do the
  rest. → answers *composition* by re-thinking the information architecture, and
  keeps the log first-class instead of below the fold.

## References

In-system, so "what to take" from each is **structural** — composition, density,
and data legibility, not their palettes or type:

- **TrendWeight** (web weight tracker) — take its core move: the **smoothed
  trend line as the subject**, raw scale readings demoted to faint background
  points, and a stated **rate of change**. Drives `trend-band-analyst`.
- **Libra** (Android weight trend) — take its **noise-band around the trend**
  and its insistence that the daily scatter is context, not the headline.
  Reinforces `trend-band-analyst`.
- **Whoop** (today screen) — take the **big, confident single hero metric** and
  the first-half-second "here's your state" read. Drives `single-metric-hero`.
- **Oura** (trends) — take its **direction-of-travel framing** (arrows, "you're
  trending down") and calm restraint. Reinforces `single-metric-hero`.
- **Apple Health** (a single metric's detail view) — take its **large, quiet
  chart with on-plot annotations** and minimal surrounding chrome. Drives
  `calm-canvas`.
- **Linear** (two-pane / dense lists) — take its **two-pane composability,
  keyboard-first density, and single-accent restraint**. Drives
  `split-control-room` and informs `calm-canvas`.

## Selection criteria

A note-to-self for the pick, not a rubric the worker optimizes against. When I
compare these I'm trying to decide:

- **Does the trend beat the noise?** This is a weight graph — at a glance I
  should see which way I'm going, with the daily morning/evening spread reading
  as *context around* the trajectory, not as competing clutter. The variant that
  nails this is doing the core job.
- **Does the log finally stop feeling like a spreadsheet** while staying fast to
  scan and quick to add to — and does it make **multiple readings on one day**
  feel intentional (a grouped pair with a delta), not like duplicate rows?
- **Is there a clear hero**, so the page delivers a "here's where I'm at and
  where I'm headed" hit instead of reading as a generic chart-plus-tiles
  analytics page?
- **Does the above/below-goal and trending-up-vs-down state read instantly**,
  including the no-goal-set degrade?
- **Does it still read as Prog Strength** — native to the dark slate / violet /
  Nunito system, with the legacy-blue chart finally reconciled to the accent —
  rather than a re-skin?
- The heavy same-day spread (May 30's two readings) and a sparse early-user
  range should both survive the treatment without looking broken.

---

> **Lifecycle.** `status:` is editorial — the owner is the dispatch gate. It
> moves `draft` → `exploring` (worker running) → `awaiting_selection` (draft PR
> open, owner deciding) → `selected` / `abandoned`. The worker sets
> `awaiting_selection` on the `dx/bodyweight-page` branch as it opens the PR; the
> owner sets the terminal value when they close it.
>
> **Handoff.** This DX ends at a *chosen variant*, not merged code. I open the
> draft `[DX — DO NOT MERGE]` PR, compare the variants on the preview deploy,
> pick one, tick its box, set `status: selected` (noting the winning idiom), and
> **close the PR — never merge it.** Then I open a SOW: *"implement bodyweight-page
> per the `<chosen-idiom>` variant from `dx/bodyweight-page`, production-quality,
> conforming to the design system."*
