---
type: dx
status: draft
surface: personal-records-lifts
idioms:
  - readiness-gauge
  - leaderboard-rows
  - editorial-cards
  - compact-dashboard
  - momentum-timeline
references:
  - Whoop
  - Oura
  - Linear
  - Strava
  - Apple Fitness
  - Gentler Streak
scope: in-system
variant_count: 5
repos:
  - prog-strength-web
  - prog-strength-docs
---

# DX: Personal Records — Lifts

**Status**: Draft · **Last updated**: 2026-06-18

> A DX (Design Exploration) is the platform's **divergent** work type. Unlike a
> SOW it does not converge on one correct implementation — it produces N
> differentiated visual variants of a single frontend surface, side by side on
> one comparison route, awaiting a human pick. It **never merges** and ships no
> production code; the chosen direction feeds a downstream SOW that builds it for
> real.

## Context

The Personal Records page (`/personal-records`) is the app's **trophy case** —
the place a lifter goes to see their bests. Its Lifts view shipped as a uniform
grid of identical cards: per lift, the heaviest tested set (weight × reps + date)
on top, the current recency-weighted **estimated 1RM** below it with a `+X vs PR`
delta, an amber **"Time for a max?"** badge, and a `View workout →` link with an
expand-to-progression-chart chevron. It works, but it doesn't feel like a trophy
case, and four things specifically undercut it:

- **The real signal is buried.** The whole point of pairing the tested PR with
  the current estimated 1RM is the **gap between them** — when your estimate has
  outrun your last tested max, it's time to attempt a new one. Today that gap
  reads as two disconnected numbers plus a redundant amber badge; the card never
  *expresses* "your estimate has pulled ahead." Worse, in the real fixture the
  badge fires on **7 of 8 lifts** — a binary that's true almost everywhere is
  noise, not a cue. The gap has a **magnitude** and the page should rank by it,
  not slap the same sticker on every card.
- **It's flat — no hierarchy.** Every lift is an identical box of equal visual
  weight; bench, squat, and deadlift all look the same. There's no sense of
  which PR is most impressive, most stale, or most "due," and no ordering — just
  a grid in whatever order the backend returns.
- **Boxy label chrome.** Each card repeats the same uppercase furniture
  (`CURRENT ESTIMATED 1RM`, `SET ON …`, `View workout →`) so the page reads like
  a data dump rather than a celebration of bests. The signal-to-chrome ratio is
  low.
- **The empty state is dead weight.** A headline lift the user has never PR'd
  (Deadlift in the fixture) renders a full-height card showing only `—` and
  `NO RECORD YET`, sitting in the grid with the same visual weight as a real PR.

This DX explores the **entire Lifts view of the page** afresh — not just the
card. It matters now because the trophy case is a page a user *wants* to visit,
and right now it under-rewards the visit: the numbers that should make a lifter
feel their progress are there, but the composition flattens them.

`scope: in-system`: the foundation is decided (see
[`../design-system.md`](../design-system.md) v0.4) and is **not** re-litigated
here — soft near-black ramp, the single **periwinkle** accent (`#9aa6d6`),
**Manrope** with tight numeric tracking and **no display face**, 14px panel
radius, hairline borders. Variants diverge on **layout, structure, density, and
composition** — what a "PR" looks like, how the page ranks/groups them, how the
PR-vs-estimate gap is made legible, and what the page's hero is.

## The surface

The Lifts view of `/personal-records`, today composed of
`prog-strength-web/app/(app)/personal-records/page.tsx` (page shell) plus its
`_components/`: `LiftsView.tsx` (the grid + loading/error/empty messages),
`LiftPRCard.tsx` (one card), `ProgressionChart.tsx` (the expandable chart),
`ExpandChevron.tsx`, and `format.ts`. A variant owns this whole view and may
reorganize any of it, but must account for all of it:

- **Page header** — title `Personal Records`, the segmented **Lifts / Running**
  `ViewSwitcher` beside it, a **`Customize`** button on the right (opens
  `components/headline-exercises-modal.tsx`, the headline-lift selection modal;
  Lifts-only — hidden on Running), and an intro sentence. The switcher must keep
  working (Running is a real sibling view), but a variant may **recompose the
  page chrome** — the header, the switcher's placement, the intro, and the
  Customize affordance are all in scope. The **Running view's internals are
  not** — only the page-level shell it shares with Lifts.
- **The grid** — one card per **backend-curated headline lift**, in API order.
  Today `grid-cols-1 sm:grid-cols-2 lg:grid-cols-3`, capped at `max-w-4xl`. The
  **count is user-controlled** via Customize, so a variant must look right at
  ~3 lifts and at ~12, not just the 8 in the fixture.
- **The card**, per lift:
  - Exercise name (**truncates** — "Barbell High Bar Back S…", "Incline Dumbbell
    Bench…").
  - Heaviest tested set: **weight × reps** + `Set on <date>`.
  - **Current estimated 1RM** + a `+X vs PR` delta (shown only when `|gap| ≥ 1`;
    positive = success-toned, else muted).
  - A **"Time for a max?"** badge when the gap is a **≥ 5% of PR** (the
    `readyForAttempt` rule).
  - `View workout →` (links to `/workouts/<workout_id>`) and an **expand
    chevron** revealing the inline `ProgressionChart` for that exercise.
  - **Empty / never-PR'd** lift (`weight === null`): `—`, `NO RECORD YET`, no
    estimate, no badge, no chevron, no link.

The data shape (`PersonalRecord` in `prog-strength-web/lib/api.ts`), one row per
headline lift from `GET /personal-records`:

```ts
type PersonalRecord = {
  exercise_id: string;
  exercise_name: string;
  workout_id: string | null;        // null ⇒ never PR'd
  weight: number | null;            // heaviest-set weight; per-DUMBBELL for DB lifts
  reps: number | null;              // reps at that weight
  unit: "lb" | "kg" | null;
  achieved_at: string | null;       // RFC3339; the "Set on" date
  current_estimated_1rm: number | null;   // recency-weighted estimate
  estimated_1rm_unit: "lb" | "kg" | null;
};
```

The **readiness** quantity every variant should express is derived client-side:
`gap = current_estimated_1rm − weight`, `gapPct = gap / weight × 100`, and the
shipped binary `ready = gapPct ≥ 5`. Variants are encouraged to treat `gapPct`
as a **continuous magnitude** (rank/scale by it) rather than only as the 5%
threshold — that's the core fix for the "badge fires on everything" problem.

**Dumbbell convention.** For dumbbell lifts, `weight` is **per dumbbell**, not
the pair total — "Dumbbell Bench Press 105 × 8" means 105 lb *in each hand*. The
shipped card shows a bare `105 lb` with no indication of this; a variant may
choose to make per-dumbbell explicit, but must not silently imply a combined
load.

**Color logic (in-system).** The view is neutral by default. Three meanings, three
disciplined uses of the palette — never as generic card tint:
- **periwinkle accent** (`#9aa6d6`) — app chrome / active state only (the active
  switcher segment, the user's emphasis). It is **not** an activity hue.
- **lift discipline tone** (steel-blue `--discipline-lift-*`: `#1a2a3c` / `#8cbce8`
  / `#5598d8`) — available as the page's *discipline* tone since this view is
  entirely lifts, but it must **never read as selection/"active"** (that's the
  accent's job).
- **the readiness signal** — `warning` (`#d6b87f`) is the natural home for "time
  for a max"; the positive `+X vs PR` delta uses `success` (`#86b39f`). A variant
  that ranks by `gapPct` should let the signal **scale** (more gap → more
  emphasis), not toggle.

**Visual states the variants must handle** (render them all in the mockup):

- **Ready, by degree.** Most lifts have a positive gap; a small gap (e.g. +20.2
  on a 155 PR ≈ 13%) and a large one (+121.7 on a 365 set of 10 ≈ 33%) must read
  as *different intensities*, not the same badge. The variant that ranks them is
  doing the core work.
- **At/near max.** A lift whose estimate sits within a rep or two of the tested
  PR (small/zero gap) — the "you've recently tested, nothing to chase" state. The
  shipped card hides the delta under `|gap| < 1`; the variant must make
  "freshly tested" read as *fine*, not broken.
- **Never PR'd / empty.** `weight === null` (Deadlift) — no PR, no estimate. Must
  not occupy the same weight as a real record; give it a deliberate home (a quiet
  group, a prompt to test it), never a dead full-height card.
- **High-rep vs low-rep PR.** `× 2` (a true near-max) vs `× 10` (a rep-out whose
  Epley-style estimate runs hot) — the rep count is context the card should keep
  legible, since it conditions how much to trust the estimate.
- **Long exercise names.** "Barbell High Bar Back Squat", "Incline Dumbbell Bench
  Press" — truncation or wrapping must be designed, not accidental.
- **Variable lift count.** 3 lifts and 12 lifts must both compose well (user
  controls the set via Customize).
- **Loading / error.** Today plain muted sentences ("Loading personal records…",
  the error string). A variant should define a treatment that fits its layout
  (e.g. skeleton rows/cards).
- **Unit.** `lb` or `kg` from the row; don't assume a 3-digit lb figure or fixed
  decimal width.

**Representative fixture** (mirror the screenshot — 8 backend-curated headline
lifts, one never tested, unit lb):

| Exercise                    | PR set       | Set on        | Est. 1RM | Gap (≈ %)      |
| --------------------------- | ------------ | ------------- | -------- | -------------- |
| Barbell Bench Press         | 305 lb × 2   | May 11, 2026  | 326.9    | +21.9 (≈ 7%)   |
| Barbell High Bar Back Squat | 315 lb × 4   | May 4, 2026   | 363.3    | +48.3 (≈ 15%)  |
| Barbell Bent Over Row       | 205 lb × 3   | May 11, 2026  | 233.7    | +28.7 (≈ 14%)  |
| Dumbbell Bench Press        | 105 lb × 8 † | Apr 28, 2026  | 136.8    | +31.8 (≈ 30%)  |
| Incline Dumbbell Bench Press| 95 lb × 8 †  | Jun 16, 2026  | 116.0    | +21.0 (≈ 22%)  |
| Barbell Deadlift            | — (no record)| —             | —        | — (empty)      |
| Leg Press                   | 365 lb × 10  | Apr 22, 2026  | 486.7    | +121.7 (≈ 33%) |
| Barbell Military Press      | 155 lb × 5   | Jun 13, 2026  | 175.2    | +20.2 (≈ 13%)  |

† per-dumbbell weight. These are mockups: **static fixtures that look real are
preferred** — do not wire variants to live PR services. Note the spread of
gap-% (7%–33%) and the two most-recent PRs (Jun 16, Jun 13) vs the stale ones
(Apr 22, Apr 28) — the fixture is built so ranking-by-readiness and
ordering-by-recency both produce a visibly different order than the current
flat grid.

## Idioms

Five genuinely distinct compositions of the *same* near-black / periwinkle /
Manrope surface. None re-decides palette, accent, or type — they diverge along
**type scale** (how hard they lean on big tabular figures vs Manrope weight for
hierarchy), **color logic** (how the accent / discipline tone / readiness signal
separate meaning from neutral chrome), and **spacing rhythm** (tight blotter ↔
airy editorial). Each makes a **different element the hero** and leans on a
different reference.

- **readiness-gauge** — The **gap, made the hero**. Each lift carries a
  horizontal **track**: the tested PR is the filled bar, the current estimate a
  marker *ahead* of it, and the **lit segment between them is the readiness**.
  "Time for a max" stops being a badge and becomes the gap visibly opening up —
  a bigger `gapPct` lights more of the track. Medium type, tabular figures but
  not huge; the **gauge** carries the hierarchy, the warning tone lives only in
  the lit gap. Borrowing **Whoop / Oura** readiness bars: a single quantified
  bar per row, restrained color spent entirely on the meaningful segment. → the
  most direct answer to *buried signal* — you *see* which lifts have pulled ahead.

- **leaderboard-rows** — The trophy case as a **ranked list**, not a grid. One
  dense row per lift, **ordered by readiness** (`gapPct` descending), with strong
  horizontal hierarchy: rank/name · big tabular **PR figure** · estimate · a
  **degree-of-readiness** chip that scales with the gap. Never-PR'd lifts sink
  into a quiet **"Not yet tested"** group at the bottom. Tight rows, structural
  color (the chip and the top row earn the accent; everything else neutral),
  tabular alignment. Borrowing **Linear**'s earned row density and **Strava**'s
  segment leaderboards. → kills *no hierarchy* and *boxy chrome* in one move, and
  finally gives the empty state a deliberate home.

- **editorial-cards** — The **card form, elevated**. Keeps cards but kills the
  repeated label furniture: each card opens with one confident **headline** (the
  lift, large), and the PR + estimate read as a single stat *line* — "305 × 2,
  est. 327, +22" — rather than two boxed, labeled stats. Big type-scale contrast,
  generous leading, captions demoted to faint, periwinkle/warning spent only on
  the readiness cue. Borrowing **Apple Fitness** summary cards and editorial
  sport layouts: airy, magazine rhythm, one hero per card. → the answer to *boxy
  label chrome* that stays celebratory — it still feels like a trophy.

- **compact-dashboard** — The **whole case at a glance**. Trades celebration for
  overview: every lift visible at once in a tight tile grid, each tile a
  mini-stat — PR figure, a small est-1RM delta, and a **readiness dot or trend
  spark**. Small functional type, status-encoded color (warning dot scales with
  readiness, muted when fresh), very tight vertical rhythm, high information
  density. Borrowing **Whoop**'s overview grid and a trading dashboard. → the
  power-user "show me everything" answer; the deliberate density opposite of
  editorial-cards, and the one that scales best to 12 lifts.

- **momentum-timeline** — PRs reframed around **recency and momentum**. Orders
  lifts by **what's most "due"** (stale PR + estimate pulled ahead = top), shows
  **time-since-PR** prominently, and promotes the per-card **progression spark**
  to a glanceable inline trend so you read trajectory without expanding. "Time
  for a max" becomes *structural* — it's about staleness, not a binary badge.
  Medium feed type, color driven by due-ness (warm = stale-and-ready, calm =
  freshly tested), medium feed rhythm. Borrowing **Gentler Streak**'s "due"
  framing and **Strava**'s trend surfaces. → makes the signal about *when*, and
  surfaces the chart the current page hides behind a chevron.

## References

In-system, so "what to take" from each is **structural** — composition,
density, and data legibility, not their palettes or type:

- **Whoop** — take its **single quantified bar per metric** (recovery/strain) and
  its **dense overview grid**. Drives `readiness-gauge` and `compact-dashboard`.
- **Oura** — take its **readiness-as-a-bar** legibility, one meaningful number
  with a restrained accent on the meaning. Reinforces `readiness-gauge`.
- **Linear** — take its **earned row density**: tabular alignment, hairline
  grouping, single-accent restraint, keyboard-scannable rows. Drives
  `leaderboard-rows`.
- **Strava** — take **segment leaderboards** (ranked bests) and its **trend
  surfaces**. Reinforces `leaderboard-rows` and `momentum-timeline`.
- **Apple Fitness** — take its **summary cards**: one hero stat per card, large
  type, generous spacing, labels demoted. Drives `editorial-cards`.
- **Gentler Streak** — take its **"due / recovery" framing**, ordering by what
  needs attention rather than by name. Drives `momentum-timeline`.

## Selection criteria

A note-to-self for the pick, not a rubric the worker optimizes against. When I
compare these I'm trying to decide:

- **Does the PR-vs-estimate gap finally read as one thing** — "your estimate has
  pulled ahead, by *this much*" — instead of two disconnected numbers and a
  redundant badge?
- **Is readiness differentiated by degree**, so the page tells me *which* lifts
  are most due instead of stamping "Time for a max?" on nearly all of them?
- **Does the page have hierarchy** — can I tell at a glance which PR is the
  standout / the most stale / the most due — rather than a flat equal-weight grid?
- **Did the label chrome get out of the way** so the numbers carry the page and it
  feels like a *trophy case*, not a data dump?
- **Does the never-PR'd lift have a deliberate home** (a prompt, a quiet group) —
  not a dead full-height card?
- **Does it hold up across the real range** — 3 lifts and 12, long truncating
  names, a `× 2` near-max next to a `× 10` rep-out, lb and kg, loading/error?
- **Does it still read as Prog Strength** — native to the near-black / periwinkle
  / Manrope system, accent spent on meaning, the lift discipline tone never
  masquerading as selection — rather than a re-skin?
- **Does the page chrome still work** — the Lifts/Running switcher and Customize
  affordance present and obvious — even after a variant recomposes the header?

---

> **Lifecycle.** `status:` is editorial — the owner is the dispatch gate. It
> moves `draft` → `exploring` (worker running) → `awaiting_selection` (draft PR
> open, owner deciding) → `selected` / `abandoned`. The worker sets
> `awaiting_selection` on the `dx/personal-records-lifts` branch as it opens the
> PR; the owner sets the terminal value when they close it.
>
> **Handoff.** This DX ends at a *chosen variant*, not merged code. I open the
> draft `[DX — DO NOT MERGE]` PR, compare the variants on the preview deploy,
> pick one, tick its box, set `status: selected` (noting the winning idiom), and
> **close the PR — never merge it.** Then I open a SOW: *"implement the Personal
> Records Lifts view per the `<chosen-idiom>` variant from
> `dx/personal-records-lifts`, production-quality, conforming to the design
> system."*
