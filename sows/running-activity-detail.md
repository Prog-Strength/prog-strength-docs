---
type: sow
status: ready_for_implementation
repos:
  - prog-strength-web
  - prog-strength-docs
---

# Running Activity Detail — Splits-Ledger-Spine

**Status**: Ready for implementation · **Last updated**: 2026-06-18

> Frontend SOW. It implements a chosen DX variant and therefore inherits that DX's
> `scope` — here **`in-system`**. The visual foundation (design-system **v0.4**,
> oura-calm) is already decided, so this SOW **conforms** to it; it does **not**
> re-tone the system or touch shared tokens. One repo does the work
> (`prog-strength-web`); `prog-strength-docs` only flips the DX to `selected` and
> marks this SOW shipped.

## Introduction

`/running/[id]` (`app/(app)/running/[id]/page.tsx`) is where a runner lands after
tapping a single run — the page that should answer *"how did that run actually
go?"* It loads one imported TCX activity (`getRunningSession`, which returns the
per-trackpoint series), best-effort-surfaces the planned workout it fulfilled
(`getPlannedWorkoutBySession`), and carries inline-rename and Delete actions. The
**functionality is solid and not in question**: the activity loads, the eight
summary tiles compute, the trackpoints plot, the plan-linkage resolves, and
rename / unlink / delete work.

What's unsatisfying — and what the Design Exploration `dx/running-activity-detail.md`
(PR Prog-Strength/prog-strength-web#81) set out to fix — is that the page renders
**every run the same way: an undifferentiated metrics grid** (eight equal stat
tiles, two lookalike line charts, an empty elevation box) even when the run is
plainly *not* generic. For the representative run — a completed `W7 D2 — Interval
Run` — three things get lost: the **session's structure** (the reps live in the
pace trace but draw as one flat line), the **planned-vs-actual story** (reduced to
a one-line ✓ banner), and the **data is dominated by a device dropout** near mile 2
(a `30:04 /mi` spike that owns the pace axis and squashes the real signal).

The DX explored that surface across **five `in-system` variants** — all sharing
design-system v0.4 (soft near-black field, periwinkle accent as app-chrome only,
the run discipline hue green-teal `--discipline-run-*`, Manrope), diverging only on
composition and which element is the hero. The owner selected **`splits-ledger-spine`**
(reference: **Runalyze / Garmin Connect**): the run as **a log of segments**, with
a **per-distance splits table as the backbone**. The page reads top-to-bottom as a
ledger — each split a row (distance · time · pace · avg HR · elevation Δ), the
fastest/slowest marked — with a **segmented toggle** that flips the spine between
**per-mile splits** and the **detected work/recovery intervals**. Summary tiles
fold into a compact gridded header band; the charts demote to a thin supporting
**pace strip** beneath the table. Type is uniform, fine, tabular — hierarchy from
**alignment, not size**; color is **color-as-state only** (green-teal on the pace
column, desaturated success/danger on standout rows and on a rep's hit/miss).

This SOW reimplements that variant **production-quality** against the real route
and the real data. Because `scope: in-system`, **no palette, accent, type, or
design-system change** — it conforms to v0.4. `prog-strength-api` is **not**
touched: every input already exists on the detail GET.

## Proposed Solution

Rebuild the **body** of `app/(app)/running/[id]/page.tsx` from the
eight-tiles-plus-three-charts grid into the ledger composition, keeping the
page's chrome and orchestration (auth/load, `← Runs` back-link, inline-editable
name, Delete modal, plan banner / Unlink, loading / error / not-found states,
`useDistanceUnit`). The mockup on the DX branch
(`app/design-explore/running-activity-detail/_components/SplitsLedgerSpine.tsx` +
`_fixtures.ts`) is the **visual spec, not code to promote**.

The real engineering — and the part the mockup *doesn't* solve, because it
hard-codes `SPLITS`, `SEGMENTS`, and `TRACE` — is a **pure derivation layer** that
turns the live `trackpoints` into the three things the ledger renders:

1. **Per-distance splits** — bucket trackpoints into per-mile (or per-km, by the
   user's unit) splits: distance · elapsed time · avg pace · avg HR · elevation Δ,
   with the fastest / slowest split marked. Honest about the trailing **partial
   split** and about missing columns (HR/elevation null → a dash, never a fake
   zero).
2. **Winsorized pace** — a clamp threshold above which a sample is treated as a
   device dropout (the mockup's `PACE_CLAMP_MAX`), so the mile-2 glitch is excluded
   from both the plotted strip and the derived split/segment stats, and the line is
   **broken at the gap** rather than spiking. This is the single most important fix
   on the page.
3. **Detected work/recovery intervals** — segment the (cleaned) pace trace into
   warm-up / work / recovery / cool-down bouts so the interval session's structure
   reads. This is **reconstructed from the trace itself** — a planned run carries
   **no machine-readable rep list** (only `run_type` + free-text `run_details`; see
   the data note in the DX) — and is surfaced **only when the structure is real**
   (an `intervals`-typed run with a detectable alternating pattern), degrading to
   miles-only otherwise.

The page wires these derivations (computed in a `useMemo` over the real
`RunningSession.trackpoints` and `useDistanceUnit`) into the ledger: a compact
header band, the Splits section with the **miles | intervals** segmented toggle,
and the demoted pace strip. The plan linkage (`run_details` prescription) rides
along as **context text** against which the intervals are read — not parsed into
guaranteed-correct targets.

## Goals and Non-Goals

### Goals

- **Add a pure, tested derivation module** (e.g. `lib/running-splits.ts`),
  unit-aware via the existing `mi`/`km` conversion, that from
  `RunningSession.trackpoints` produces:
  - **`splits`** — per-mile (or per-km) buckets: `{ label, distanceMi/Km,
    durationSec, avgPace, avgHr | null, elevDelta | null, fastest?, slowest? }`,
    including the trailing partial split, with fastest/slowest flagged among
    full splits.
  - **winsorized pace** — a clamp/threshold that drops dropout samples (pace above
    the threshold, or gapped `null`) from plotted + derived stats, and yields the
    sparkline path **broken at gaps** (no straight-line bridge through a dropout
    that distorts the shape; the strip's "dropout bridged"/"faster ↑" affordance
    communicates the handling).
  - **detected intervals** — a pace-based segmenter returning
    `warmup | work | recovery | cooldown` segments with per-segment
    distance/time/pace/HR, exposed **only** when `run_type === "intervals"` *and*
    the detector finds a plausible alternating pattern; otherwise the function
    signals "no reliable structure" and the surface falls back to miles-only.
- **Rebuild the `/running/[id]` body into the splits-ledger composition**, over the
  **existing** data layer, conforming to design-system v0.4:
  - **Header band** — fold the eight summary tiles into a **compact gridded band**
    (the variant's 4/8-col divided strip): Distance · Time · Avg pace · Best · Avg
    HR · Max HR · Calories · Elev (`—` when null). The run name stays the
    inline-editable heading; the plan linkage renders as the green-teal **✓ pill**
    (`✓ {plan.name}`) with Unlink preserved.
  - **Splits spine** — the hairline-bordered table as the backbone: uniform fine
    **tabular figures**, hierarchy from alignment; `Split · Dist · Time · Pace ·
    Avg HR · Elev Δ` rows with hairline rules; **green-teal** (`--discipline-run-fg`)
    on the pace column; **success/danger** tags on the fastest/slowest rows;
    elevation Δ a dash when no barometric data.
  - **Segmented toggle** — design-system pill control flipping **miles ↔
    intervals**; the Intervals view shows the detected work/recovery rows
    (`Segment · Dist · Time · Pace · HR · vs target`), work bouts dotted in the run
    hue, recoveries calm-neutral. The toggle is **present only when intervals are
    available**; otherwise the spine is miles-only with no dead tab.
  - **Demoted pace strip** — a single delicate winsorized pace sparkline beneath
    the table, **faster ↑** (a rising line never reads as slowing), with the dropout
    handled. (Whether a synced HR strip rides alongside, and the fate of the
    elevation panel, are Open Questions.)
- **Surface the prescription as context** — when the run is linked, display the
  `run_details` text (e.g. the `8 × 400m @ ~6:30/mi` line) so the detected
  intervals are read against the plan; the **`vs target` delta** is shown only when
  a target pace can be derived reliably (Open Questions).
- **Preserve every behavior and chrome** — auth redirect on no-token / 401; the
  `← Runs` back-link to `/activities?view=running`; **inline rename** (optimistic
  + rollback + toast); the **Delete** confirm modal and post-delete redirect; the
  **plan banner / Unlink** flow (`getPlannedWorkoutBySession` best-effort, `null`
  → no plan affordance); `notFound` / error / loading states; and **`useDistanceUnit`**
  driving `/mi` vs `/km`, pace, and the splits bucket size.
- **Handle every hard state the DX enumerated**: the **dropout** (tamed, never owns
  the axis); **no elevation** (Δ dashes / panel degrades, no flat-line chart);
  **unlinked run** (no ✓ pill, no prescription, Intervals tab absent — still a
  complete activity readout); **non-interval run** (`easy`/`threshold` → miles-only,
  no manufactured reps); **sparse vs dense trace** (few vs hundreds of trackpoints,
  table + strip survive both); **missing HR** (dash column, strip degrades).
- **Tests** — the derivation module is the priority: per-mile bucketing incl.
  partial split; winsorization excluding the dropout from stats and breaking the
  line; fastest/slowest marking; interval detection finding the pattern on an
  interval fixture *and* declining on an easy/sparse one; unit switching (mi↔km).
  Plus a component test that the ledger renders splits, the toggle appears only when
  intervals exist, and rename/delete/unlink still fire. **CI green**
  (lint/format/typecheck/test/build).

### Non-Goals

- **Any design-system or shared-token change.** `scope: in-system` against v0.4 —
  this SOW **conforms**; it does not re-tone tokens, change the accent/type, or edit
  `design-system.md`'s decided sections. (Contrast `sows/activities-page-redesign.md`,
  which was a greenfield pick adopted app-wide.)
- **Any backend / API / SDK change.** `prog-strength-api` is **not** in `repos:`.
  All inputs already exist on the detail GET (`getRunningSession` trackpoints +
  `getPlannedWorkoutBySession`). No new endpoints, no schema change.
- **Adding machine-readable rep structure to the plan model.** A planned run has
  only `run_type` + free-text `run_details` by design — intervals are reconstructed
  from the **trace**, not by inventing plan fields. Out of scope to change that.
- **Promoting the DX mockup code.** `SplitsLedgerSpine.tsx` / `_fixtures.ts` are
  the visual spec, not code to copy; the `design-explore/running-activity-detail`
  route stays gated (`designExploreEnabled` / `NEXT_PUBLIC_ENABLE_DESIGN_EXPLORE`),
  untouched, and never ships. The other four variants are not built; the DX PR is
  **closed, never merged**.
- **Redesigning other running surfaces** — the runs list, `/activities?view=running`,
  and `progress/running/[distanceKey]` are out of scope; this is the **detail page**
  only.
- **Changing the run data model, the rename/delete/unlink semantics, or unit
  logic** — presentation rebuild + derivation over existing orchestration.

## Implementation Details

### Derivation layer (`prog-strength-web/lib`)

A new pure module (no React, no fetch) — the production replacement for the
mockup's hard-coded `SPLITS` / `SEGMENTS` / `TRACE`. Inputs: `trackpoints`
(`RunningTrackpoint[]` — `distance_meters`, `elapsed_seconds`, `pace_sec_per_km |
null`, `heart_rate_bpm | null`, `elevation_meters | null`), the linked plan's
`run_type`, and the active unit. Responsibilities:

- **Splits** — walk trackpoints accumulating into per-unit-distance buckets
  (boundary by `distance_meters`), computing each split's elapsed time, avg pace
  (from the bucket, **excluding winsorized samples**), avg HR (skip nulls; null if
  none), and elevation Δ (null when no barometric data); emit a final partial
  split; flag fastest/slowest among **full** splits.
- **Winsorization** — a clamp threshold (port the mockup's `PACE_CLAMP_MAX`
  intent: pace slower than ~11:00 /mi-equivalent, or `null`, is a gap/dropout);
  excluded samples are dropped from stats and **break** the sparkline path.
- **Interval detection** — classify the cleaned pace series into warm-up / work /
  recovery / cool-down by relative pace (e.g. work = sustained faster-than-session
  bouts alternating with slower recoveries), returning segments only when the
  pattern is plausible; expose a boolean the surface uses to decide whether to
  offer the Intervals tab. Conservative by design — a false "no intervals" (drop to
  miles-only) is far better than fabricated reps.
- Convert sec/km ↔ the active unit at the boundary; keep all stored values metric.

All of the above is deterministic and table-driven by fixtures derived from the
DX's representative run (interval session, dropout, no elevation) plus an easy-run
and a sparse-trace fixture for the degrade paths.

### Surface rebuild (`prog-strength-web/app/(app)/running/[id]/page.tsx`)

Keep the component's top half (state, effects, handlers, `notFound`/error/loading,
`EditableName`, `DeleteConfirmModal`, `CompletesPlanBanner`/Unlink). Replace the
render body (the `StatTile` grid + `HeartRateChart`/`PaceChart`/`ElevationChart`
block) with: the **compact header band**, the **Splits section** (label + segmented
toggle + the miles/intervals table), and the **pace strip**. Compute the
derivations in a `useMemo` over `session.trackpoints` + `unit` + `completesPlan?.run_type`.
Reuse design-system primitives and tokens (`--surface`, `--border`,
`--radius-card`, `--discipline-run-*`, `--success`/`--danger`, tabular-nums); build
the table/strip as small local components mirroring the mockup's structure but wired
to derived data. Charts are **demoted** to the single pace strip (see Open
Questions for the HR strip / elevation panel decision).

### Tests (`prog-strength-web`)

- **Derivation** — the priority suite: bucketing + partial split; winsorization
  excludes the dropout and breaks the line; fastest/slowest; interval detection
  positive (interval fixture) and negative (easy + sparse fixtures); unit switch.
- **Component** — renders splits from a known fixture; the **intervals toggle
  appears only when intervals are detected**; rename / delete / unlink still fire
  their mutations and optimistic updates (reuse existing API/toast mocks).

## Rollout

1. **`prog-strength-docs`** — flip `dx/running-activity-detail.md` to
   `status: selected` (winning idiom `splits-ledger-spine`); mark this SOW
   `shipped` on merge.
2. **`prog-strength-web`** — the derivation module + the `/running/[id]` rebuild
   + tests, in one PR. Vercel preview to verify on the representative run.

### Verification after rollout

- `/running/[id]` for the **interval run** reads as a ledger: a compact header
  band, a per-mile splits spine with fastest/slowest marked and green-teal pace,
  and a **miles | intervals** toggle whose Intervals view shows the detected
  work/recovery bouts with the `run_details` prescription as context.
- The **mile-2 dropout** no longer owns the axis — it's excluded from the splits
  stats and the pace strip is winsorized with the line broken at the gap; **faster
  ↑** holds so a rising line never reads as improvement.
- **Degrade paths**: an **unlinked / easy** run shows miles-only with no Intervals
  tab, no ✓ pill, no manufactured reps, and still feels complete; **no-elevation**
  Δ dashes cleanly; a **sparse trace** and **missing HR** render without holes.
- **Behavior intact**: `← Runs`, inline **rename** (optimistic + rollback),
  **Delete** (confirm + redirect), plan **Unlink**, and the **mi/km** display unit
  all work against the real API.
- `design-system.md` is **unchanged**; the `design-explore/running-activity-detail`
  route stays gated / 404 in production; **no DX mockup code shipped**; the DX PR is
  closed (never merged).

## Open Questions

1. **`vs target` source.** The Intervals table wants a target pace, but a planned
   run only has free-text `run_details` (`"… 8 × 400m @ 5K effort (~6:30/mi) …"`) —
   the mockup hard-codes `targetWorkPace: 390`. **Lean:** do **not** depend on
   parsing free text for correctness — display `run_details` verbatim as context,
   and show the signed `vs target` delta **only** when a tolerant parser confidently
   extracts a pace (e.g. a `~m:ss/mi` token); otherwise omit the column. A wrong
   "you missed target" is worse than no verdict. Confirm appetite for the small
   tolerant parser vs. context-text-only at review.
2. **Interval-detection confidence on real, noisy traces.** Pace-based segmentation
   on sparse or jittery TCX data can mis-segment. **Lean:** gate the Intervals tab
   on `run_type === "intervals"` **and** a plausibility check (enough alternating
   bouts), and fail **closed** to miles-only when unsure — never show fabricated
   reps. Tune thresholds against the representative fixture; revisit if real runs
   under/over-detect.
3. **Fate of the existing HR / elevation charts.** The variant demotes charts to a
   single pace strip. **Lean:** ship the winsorized **pace** strip as the supporting
   figure and **drop the elevation panel** when elevation is null (the
   representative case); decide whether a **synced HR** strip rides alongside (it
   adds the cardiac-drift story the ledger otherwise loses) or is cut for the calm
   density the idiom wants. Confirm on preview.
4. **Splits unit & partial final split.** Splits bucket by the user's unit (per-mile
   vs per-km). **Lean:** bucket per active unit, render the trailing partial split
   with its true distance label (e.g. `0.2 mi`) and exclude partials from
   fastest/slowest. Confirm the labeling reads cleanly for km users.
