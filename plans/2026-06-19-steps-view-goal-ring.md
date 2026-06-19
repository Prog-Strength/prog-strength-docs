# Implementation Plan: Steps View — Goal-Ring-Hero

**SOW:** `sows/steps-view-goal-ring.md` · **Created:** 2026-06-19 · **Branch:** `feat/steps-view-goal-ring`

A **frontend-only**, single-repo (`prog-strength-web`) presentation rebuild of
the Activities → Steps tab into the **`goal-ring-hero`** idiom. No backend, no
API, no design-system change. `scope: in-system` against design-system **v0.4**
(oura-calm-minimal): conform to the existing tokens, never introduce new hues.

The work is two pieces:

1. A pure, tested derivation module (`lib/steps-stats.ts`) ported from the DX
   mockup's `_data.ts` (`deriveStats` + `buildAxis`), trimmed to what goal-ring
   needs, wired to a real `new Date()` and production's `days: number | null`.
2. A rebuild of `StepsView`'s **render body** (hero ring + supporting strip +
   goal-relative bars + ring-row log) over the **unchanged** data layer.

The DX mockup (`app/design-explore/steps-view/_variants/goal-ring-hero.tsx`,
`_data.ts`) is the **visual spec, not code to promote** — it never ships.

---

## Task 1 — Derivation module `lib/steps-stats.ts` (+ tests)

A pure, React-free module. Exports:

- `type Day = { date: string; dow: number; label: string; weekday: string;
  steps: number | null; logged: boolean; hit: boolean; pct: number | null;
  delta: number | null }`
- `type Stats = { avg: number | null; total: number; best: { date; steps } | null;
  daysLogged: number; daysHit: number; daysInPeriod: number;
  attainmentPct: number | null; hitRatePct: number | null }`
  (the mockup's `currentStreak` field is **dropped** — the SOW says the streak
  field is not needed for this idiom.)
- `deriveStats(entries, days, goal)` → `Stats`
- `buildAxis(entries, days, goal)` → `Day[]` (oldest → newest)
- `entriesInPeriod(entries, days)` → `StepsEntry[]` newest-first (helper used by
  both the recent-rows derivation and `deriveStats`)
- formatting helpers as needed (`fmt`, `fmtK`) OR reuse the existing
  `formatSteps`; keep the module pure.

**Key differences from the mockup `_data.ts`:**

- The period is production's `days: number | null` (`null` = All), **not** a
  `Period` enum. Signatures take `days: number | null` directly.
- "Today" is a **real `new Date()`** (mockup pins `TODAY`). To keep tests
  deterministic, the date math must read `new Date()` once per call; tests use
  `vi.useFakeTimers()` / `vi.setSystemTime()` to pin it.
- Local-date parsing consistent with the existing `parseLocalDate`/`isoDate`
  helpers in `steps-view.tsx` (re-implement equivalently in the module; the
  inline ones in the component can then import from here or stay — keep one
  source of truth where reasonable, but do not change the component's behavior).
- `deriveStats` uses the existing `StepsEntry` type from `@/lib/api` (which has
  `created_at`/`updated_at`); the mockup's local `StepsEntry` is narrower.
- All goal-relative fields **null when `goal` is falsy/0** (`attainmentPct`,
  `hitRatePct`, per-day `pct`/`delta`, `hit = false`).
- `daysHit` denominator = **logged days** (`daysHit/daysLogged`), per Open
  Question #4 lean.
- `buildAxis` for `All` (`days === null`) runs earliest-entry → today; for a
  bounded window runs (today − (days−1)) → today. Unlogged days are
  `steps: null` gaps, **never 0**. Empty entries → `[]`.

**Tests** (`lib/steps-stats.test.ts`, table-driven, `vi.setSystemTime`):

- `deriveStats`: avg/total/best over the DX fixture; `daysHit`/`attainmentPct`
  with a goal; the `94%` near-miss avg rounds to `94`, not away; **no-goal
  nulls** (`attainmentPct`/`hitRatePct` null, `daysHit` 0); empty → all-null
  shape.
- `buildAxis`: gap-fills a bounded window (`7`/`30`) **and** `All`; unlogged days
  are `null` not `0` (assert the Jun 9 / May 30 gaps); the 14,000 big day present
  with its real value; `hit`/`pct`/`delta` correct over/under goal; no-goal →
  `pct`/`delta` null, `hit` false; empty entries → `[]`.

Use the DX fixture (the 14,000 big day, Jun 16/17 misses, Jun 9 + May 30 gaps,
avg ≈ 94% of 10,000) as the representative window; pin "today" to 2026-06-18 in
those tests so the fixture's relative window is deterministic.

---

## Task 2 — Rebuild `StepsView` body (`components/activities/steps-view.tsx`)

**Keep verbatim** (the orchestration / data layer): the `days` prop, `refetch`
dual fetch (range + first keyset page), `loadMore`, `handleSave`/`handleDelete`/
`handleSaveGoal`, the three modals (`StepsLogModal`, `StepsGoalModal`,
`ModalShell`/`ModalFooter`), auth/401 redirect, error/loading guards,
`hasGoal`/`goalValue`.

**Replace the render body** (today's four `StatTile`s + `ChartCard` +
`StepsTable` styling) with:

- **Hero ring** (`Ring`) — average centered, fill = `attainmentPct` toward goal
  in `--accent`, **clearing to `--success` at ≥100%**; `{pct}% of {goal} goal`
  caption. No goal → neutral `--surface-3` track, average centered, "no goal
  set" caption, **no `NaN%`**. Drive from `deriveStats(rangeEntries, days,
  goalValue ?? 0)`.
- **EmptyRing** — when `stats.avg === null` (no entries in range): the "your
  ring fills as you log" start state with a Log-steps button wired to
  `setShowLog(true)`. No broken chart frame.
- **Supporting stat strip** — compact `Days hit` (`{daysHit}/{daysLogged}`,
  success-toned; `—` no goal) · `Best` · `Total`. Absorbs today's four tiles so
  **Avg is unambiguously the hero**.
- **Goal-relative bars** — **Open Question #1 lean: keep recharts.** Implement
  goal-relative as a **stacked base + crest**: `base = min(steps, goal)`,
  `crest = max(0, steps − goal)`, per-`Cell` tones (`CHART_STEPS_UNDER` base,
  `CHART_STEPS_MET` crest; no-goal → single `CHART_LIFT_LINE` bar). **Unlogged
  days render as gaps** (axis includes `null` days from `buildAxis`; recharts
  skips `null` values → a gap, not a 0 bar). Keep the dashed goal `ReferenceLine`
  and the goal-into-domain `YAxis` fix. The 14,000 big day must not flatten the
  rest (domain handles it). Drive from `buildAxis(rangeEntries, days,
  goalValue ?? 0)`.
  - Fall back to a hand-rolled div bar set **only** if the recharts crest/gap
    reads materially worse; note the decision at review.
- **Ring-row log** — **keep the full editable, keyset-paginated table** (Open
  Question #2 lean: full table, not a 6-row preview). Restyle each row to carry a
  **per-day mini-ring + goal-%** (success-toned on a hit) alongside the
  date/steps and the edit/delete buttons; keep **Load more**, the `Log steps`
  toolbar button, and the `Goal` affordance + their modals. The Log/Goal
  affordances must not get buried.

Build `Ring`, `MiniRing`, and the bars as **small local components** mirroring
the mockup but wired to derived data + the shared `@/lib/chart-colors` tokens
and CSS vars (`--accent`, `--success`, `--surface-3`, `--border-strong`). Never
raw hex or a new steps hue.

The per-row `pct` should come from `buildAxis`/a small derivation (or compute
`Math.round(steps/goal*100)`), consistent with `deriveStats`.

Replace the inline `computeStats` with `deriveStats` from the module.

**Component tests** (extend `steps-view.test.tsx`):

- Ring renders the average; the `% of goal` caption is present; the ring clears
  to success at ≥100% (assert the success stroke / caption tone).
- No-goal ring is neutral (no `NaN%`, "no goal set" caption).
- Bars render base/crest/gap correctly (assert `Cell` fills incl. the success
  crest and that an unlogged day is a gap, not a 0 bar).
- Log rows show the mini-ring + `%`.
- `edit` / `delete` / `Log steps` / `Goal` still fire and refetch (port/keep the
  existing mutation tests — the existing tests reference `Avg daily steps` /
  `Total steps` tile text and the `Goal NN,NNN` ReferenceLine label, which the
  rebuild changes; **update those assertions** to the new surface rather than
  deleting the coverage).

The recharts test mock in `steps-view.test.tsx` already stubs `Bar`/`Cell`/
`ReferenceLine`; keep it, adjusting expectations for the stacked base+crest
(there will now be up to two `Bar`s / more `Cell`s).

---

## States to cover (from the SOW)

over vs under goal · the big day (no flatten) · the near-miss avg (`94%` legible)
· unlogged gaps (never 0) · mixed density `7/30/90/All` · empty/first-entry
(EmptyRing) · no goal (neutral ring, accent bars, no %, no goal line, no `NaN%`)
· both breakpoints (ring centered, bars responsive, rows readable on mobile).

## Non-Goals

No backend/API/SDK change · no design-system/token change · no other Activities
tab or page-chrome change · no data-model/period/keyset/modal-behavior change ·
no promoting the DX mockup (the `design-explore/steps-view` route stays gated;
the DX PR is closed, never merged).

## Verification / gate

`npm run lint && npm run format:check && npm run typecheck && npm run test &&
npm run build` all green locally before pushing. Manually reason through the
`7/30/90/All`, goal vs no-goal, empty, and both-breakpoint states against the
component.

## Rollout

1. `prog-strength-docs` — flip `dx/steps-view.md` to `status: selected`
   (winning idiom `goal-ring-hero`); mark this SOW `shipped` on merge.
2. `prog-strength-web` — derivation module + Steps-tab rebuild + tests, one PR.
