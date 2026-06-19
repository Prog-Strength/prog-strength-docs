---
type: sow
status: ready_for_implementation
repos:
  - prog-strength-web
  - prog-strength-docs
---

# Steps View — Goal-Ring-Hero

**Status**: Ready for implementation · **Last updated**: 2026-06-18

> Frontend SOW. It implements a chosen DX variant and therefore inherits that DX's
> `scope` — here **`in-system`**. The visual foundation (design-system **v0.4**,
> oura-calm) is already decided, so this SOW **conforms** to it; it does **not**
> re-tone the system or touch shared tokens. The work is in `prog-strength-web`;
> `prog-strength-docs` only flips the DX to `selected` and marks this SOW shipped.

## Introduction

The **Steps tab** of `/activities` (`components/activities/steps-view.tsx`,
rendered under `?view=steps`) shipped from the activities-page redesign
(`sows/activities-page-redesign.md`, which created design-system v0.4). That
redesign set the whole page's tone across four tabs at once and did not give Steps
the attention its own content deserves: it shipped with the page's generic
furniture — **four identical stat tiles** (`Avg daily steps` no louder than `Total
steps`), a **plain recharts bar chart** with a dashed goal line, and a
**`Date · Steps` log table** that reads like a spreadsheet export.

The Design Exploration `dx/steps-view.md` (PR Prog-Strength/prog-strength-web#87)
narrowed the lens to just this tab. **Average daily steps is one of the headline
health metrics** a user checks — often the single number that says whether their
non-training activity is trending right — yet it's one of four look-alike tiles,
the chart says little about *goal attainment*, and the log is a grid of rows. The
DX explored the tab across **five `in-system` variants** (all sharing v0.4: soft
near-black ramp, single periwinkle accent, desaturated success/danger, Manrope, 14px
hairline panels), diverging only on composition. The owner selected
**`goal-ring-hero`** (reference: **Apple Fitness** rings + **Gentler Streak**):
**goal attainment is the hero**. A **progress ring** leads the region — the
attainment percent filling toward the goal, the **average in its center**, the fill
clearing from periwinkle accent to **success-green** once at/over goal. The chart
bars are drawn **goal-relative** (a muted base climbing to the goal line, a success
crest topping over-goal days, unlogged days as hairline gaps — not zeros), and each
**log row carries a tiny per-day ring + goal-%**. It's the most visceral "did I hit
it" answer, and it sits beside the already-shipped Workouts and Running tabs.

This SOW reimplements that variant **production-quality** against the real tab and
data. Because `scope: in-system`, **no palette, accent, type, or design-system
change** — it conforms to v0.4. There is **no backend change**: everything the
variant shows is derived **client-side** from the existing
`listSteps` / `getStepsGoal` endpoints.

## Proposed Solution

Rebuild the body of `StepsView` into the goal-ring composition, keeping the tab's
orchestration intact: the `days` period prop (from the parent Activities page's
`7 / 30 / 90 / All` filter), the **dual fetch** (a range fetch windowing the
ring/bars/stats, and the **keyset-paginated** log fetch), the goal/log **modals**,
and the **refetch-after-mutation** behavior. The mockup on the DX branch
(`app/design-explore/steps-view/_variants/goal-ring-hero.tsx` + `_data.ts`) is the
**visual spec, not code to promote**.

Two pieces of real work the current tab doesn't carry:

1. **A pure, tested derivation layer.** The mockup's `deriveStats` and `buildAxis`
   are the production replacement for today's small `computeStats`. `deriveStats`
   adds **`daysHit` / `daysLogged` / `attainmentPct`** (today only avg/total/best
   exist); `buildAxis` builds a **continuous day axis** across the period where
   **unlogged days are gaps (`steps: null`), never zeros** — the single most
   important fix for an honest chart. Ported with a real `new Date()` for "today"
   (the mockup pins `TODAY`) and reading production's `days: number | null` period
   (the mockup's `Period` enum maps to it).

2. **The goal-ring composition over the real, editable surface.** The hero ring,
   the supporting Days-hit/Best/Total strip, the goal-relative bars, and the
   per-row mini-rings — wired to the derivations and the live data, while
   **preserving the full editable, paginated log** (the mockup shows a 6-row
   preview; production keeps the real table with edit/delete + Load more, restyled
   with the ring-row treatment — the Log/Goal affordances must not get buried).

## Goals and Non-Goals

### Goals

- **Add a pure, tested steps-derivation module** (e.g. `lib/steps-stats.ts`),
  porting the mockup's `deriveStats` + `buildAxis`:
  - **`deriveStats(entries, days, goal)`** → `{ avg, total, best, daysLogged,
    daysHit, attainmentPct, hitRatePct }` (the streak field is not needed for this
    idiom). `daysHit = entries in period with steps ≥ goal`; `attainmentPct =
    round(avg / goal × 100)`; all goal-relative fields **null when no goal**.
  - **`buildAxis(entries, days, goal)`** → a `Day[]` oldest→newest spanning the
    window (or earliest-entry→today for `All`), each `{ date, steps: number | null,
    logged, hit, pct, delta }` — **unlogged days are `null` gaps, not zeros**.
  - Uses a real `new Date()` and local-date parsing matching the existing
    `parseLocalDate`/`isoDate` helpers; replaces the inline `computeStats`.
- **Rebuild `StepsView`'s body into goal-ring-hero**, conforming to v0.4:
  - **Hero ring** — the **average** centered, the ring fill = `attainmentPct`
    toward the goal in the **periwinkle accent**, **clearing to success-green at
    ≥100%**; a `{pct}% of {goal} goal` caption. **No goal** → a calm neutral track
    with the average plainly centered and a "no goal set" caption (no `NaN%`).
  - **Supporting stat strip** — a compact `Days hit` (`{daysHit}/{daysLogged}`,
    success-toned; `—` with no goal) · `Best` · `Total`, absorbing today's four
    tiles into the ring + this strip so **Avg is unambiguously the hero**.
  - **Goal-relative bars** — bars read against the **goal line as the dominant
    reference**: a muted base climbs to the line, a **success crest** tops days that
    clear it; **unlogged days render as a hairline gap tick, not a zero bar**; **no
    goal** → plain accent bars. Must survive the **one big day** (14,000 ≈ 2.5×) not
    flattening the rest, and both sparse (`7d`) and dense (`90d`/`All`) windows.
  - **Log rows** — **keep the full editable, keyset-paginated table** (edit +
    delete per row, **Load more**, the `Log steps` + `Goal` affordances and their
    modals), **restyled** so each row carries the **per-day mini-ring + goal-%**
    (success-toned on a hit). The editing/pagination is core functionality and must
    not be reduced to a static preview (Open Question #2).
- **Use the in-system color tokens** — the shared `@/lib/chart-colors`
  (`CHART_STEPS_MET` = success, `CHART_STEPS_UNDER` = muted, `CHART_LIFT_LINE` =
  accent) and the CSS vars (`--accent`, `--success`, `--surface-3`), **never raw
  hex or a new steps hue** (steps has no `--discipline-*` tone — the accent +
  status colors carry goal-meaning).
- **Preserve the tab's orchestration** — the `days` prop and period filter; the
  **range fetch** (ring/bars/stats) + **keyset log fetch** (`PAGE_SIZE`,
  `next_before`, Load more) split; `upsertStepsForDate` / `deleteStepsForDate` /
  `putStepsGoal` with **refetch after mutation**; auth/401 redirect; the Log/Edit
  and Goal **modals** unchanged in behavior.
- **Handle every state the DX enumerated**: **over vs under goal** (win vs muted);
  **the big day** (axis doesn't flatten the rest); **the near-miss average** (`94%`
  sitting just under goal reads as "almost," not rounded away); **unlogged days**
  (gaps, never 0-step bars/rows); **mixed density** across `7/30/90/All`;
  **empty / first-entry** (the `EmptyRing` "your ring fills as you log" state, no
  broken chart frame); **no goal set** (ring neutral, bars accent, rows no %, goal
  line absent — all graceful, no `NaN%`); **both breakpoints** (ring centered, bars
  responsive, log rows readable on mobile).
- **Tests** — the derivation module is the priority: `deriveStats` (avg/total/best,
  `daysHit`/`attainmentPct`, and the **no-goal nulls**); `buildAxis` (gap-filling
  for a bounded window **and** `All`, unlogged days as `null` not `0`, the
  big-day/near-miss fixture). Plus component tests: the ring renders the average and
  clears to success at ≥100%, the no-goal ring is neutral, bars render
  base/crest/gap correctly, log rows show the mini-ring + %, and **edit / delete /
  Log steps / Goal still fire** with refetch. **CI green** (lint/format/typecheck/
  test/build).

### Non-Goals

- **Any backend / API / SDK change.** Everything is derived client-side from the
  existing `listSteps` (range + keyset) and `getStepsGoal`; `putStepsGoal` /
  `upsertStepsForDate` / `deleteStepsForDate` are unchanged. `prog-strength-api` is
  **not** in `repos:`.
- **Any design-system or shared-token change.** `scope: in-system` against v0.4 —
  conform only; no token/accent/type edit, no `design-system.md` change.
- **Redesigning the other Activities tabs** (Overview / Workouts / Running) or the
  **page chrome** — the period filter and the tab bar are owned by the parent
  Activities page; this SOW restyles only the **Steps** tab body and must keep
  reading as a sibling of the shipped tabs.
- **Changing the steps data model, the period semantics, the keyset pagination, or
  the modals' behavior** — presentation rebuild over the existing orchestration.
- **Promoting the DX mockup code.** `goal-ring-hero.tsx` / `_data.ts` are the visual
  spec, not code to copy; the `design-explore/steps-view` route stays gated
  (`designExploreEnabled` / `NEXT_PUBLIC_ENABLE_DESIGN_EXPLORE`) and never ships.
  The other four variants are not built; the DX PR is **closed, never merged**.

## Implementation Details

### Derivation module (`prog-strength-web/lib`)

A pure, React-free module (e.g. `steps-stats.ts`) porting the mockup's `_data.ts`
`deriveStats` + `buildAxis` (and the `Day` / `Stats` types), trimmed to what
goal-ring needs. Replaces `computeStats` in `steps-view.tsx`. Real `new Date()`;
local-date parsing consistent with the existing `parseLocalDate`/`isoDate`. The
period maps from production's `days: number | null` (`null` = All → axis runs
earliest-entry→today). Table-driven tests over the DX's representative fixture (the
14,000 big day, the `94%` near-miss avg, the Jun 16/17 misses, unlogged gaps) plus a
no-goal and an All-period fixture.

### Steps tab rebuild (`components/activities/steps-view.tsx`)

Keep the component's data layer verbatim (the `refetch` dual fetch, keyset
`loadMore`, `handleSave`/`handleDelete`/`handleSaveGoal`, the modals, error/loading
guards). Replace the **render body** — the four `StatTile`s, the `ChartCard`, and
the `StepsTable` styling — with: the **hero ring** (port the mockup's `Ring`), the
**Days-hit/Best/Total strip**, the **goal-relative bars** (over `buildAxis`), and
the **ring-row log**. Drive the ring/bars/stats from `deriveStats(rangeEntries,
days, goalValue)` + `buildAxis(rangeEntries, days, goalValue)`; keep the log table
fed by the keyset `logEntries`. Build the ring/mini-ring/bars as small local
components mirroring the mockup but wired to derived data and the shared
chart-color tokens.

### Tests (`prog-strength-web`)

- **`steps-stats.ts`** — `deriveStats` + `buildAxis` per the fixtures above,
  including no-goal nulls and gap-not-zero.
- **Component** (extend `steps-view.test.tsx`) — ring average + success-clear at
  ≥100%, no-goal neutral ring, base/crest/gap bars, ring-row %, and that
  edit/delete/Log/Goal still fire and refetch.

## Rollout

1. **`prog-strength-docs`** — flip `dx/steps-view.md` to `status: selected`
   (winning idiom `goal-ring-hero`); mark this SOW `shipped` on merge.
2. **`prog-strength-web`** — the derivation module + Steps-tab rebuild + tests, in
   one PR. Vercel preview to verify across `7 / 30 / 90 / All`, goal vs no-goal,
   empty/first-entry, and both breakpoints.

### Verification after rollout

- `/activities` → Steps reads with **goal attainment as the hero**: a ring with the
  **average centered**, filling in the accent and **clearing to success at ≥100%**,
  the `% of goal` caption making the `94%` near-miss legible — Avg is unambiguously
  the headline, not one of four equal tiles.
- The **goal-relative bars** show muted base + success crest against the goal line,
  the **14,000 day doesn't flatten** the rest, and **unlogged days are gaps**, not
  0-step bars.
- The **log** still edits, deletes, logs, sets the goal, and **Load more** paginates
  — now with a per-row **mini-ring + goal-%**; the Log/Goal affordances are obvious.
- **Degrades** cleanly: **no goal** → neutral ring + accent bars + no `NaN%`;
  **empty/first-entry** → the EmptyRing start state; sparse `7d` and long `All` both
  look intentional; mobile and desktop both hold.
- It **sits beside Workouts and Running** as the same v0.4 surface; the
  `design-explore/steps-view` route stays gated / 404 in production; **no DX mockup
  code shipped**; the DX PR is closed (never merged); `design-system.md` unchanged.

## Open Questions

1. **Goal-relative bars: recharts vs hand-rolled.** The mockup hand-rolls the bars
   as divs (faithful base/crest split + hairline gap ticks); today's chart uses
   recharts (tooltips, responsive axis, consistent with the Workouts/Running trend
   charts). **Lean:** keep **recharts** — implement goal-relative as a **stacked
   base + crest** (`min(steps, goal)` + `max(0, steps − goal)`) with per-`Cell`
   tones and `null` days rendering as gaps, retaining the tooltip and the
   goal-into-domain fix already in `ChartCard`. Fall back to hand-rolled only if the
   crest/gap reads materially worse in recharts. Confirm at review.
2. **Log region: full table vs preview.** The mockup shows 6 recent ring-rows; the
   real tab has an editable, keyset-paginated table. **Lean:** **keep the full
   editable/paginated table**, applying the ring-row treatment to each row (mini-ring
   + goal-%) and preserving edit/delete + Load more — the DX explicitly warns the
   `Log steps` action can't get buried. Alternative: a short ring-row preview + a
   "view all" expand. Confirm full-table.
3. **`All`-period axis cost.** `buildAxis` gap-fills every calendar day from the
   earliest entry to today; for a multi-year `All` history that's a very wide axis.
   **Lean:** gap-fill directly for `7/30/90` and accept it for `All` at current data
   sizes; if `All` gets heavy, **bucket weekly** (or cap) for the chart only — the
   ring/stats are unaffected. Flag for a perf check on the longest realistic range.
4. **`daysHit` denominator.** `daysHit/daysLogged` counts **logged** days, so an
   unlogged day neither helps nor hurts. **Lean:** keep the **logged-days**
   denominator (you can't "hit" a day you didn't log) rather than calendar days in
   the period; revisit if "3/30" framing (calendar) reads more honestly than "3/5"
   (logged). Confirm.
