---
type: sow
status: draft
repos:
  - prog-strength-web
  - prog-strength-docs
---

# Pace Trace — Editorial Area Recap

**Status**: Draft · **Last updated**: 2026-06-29

> Frontend SOW. It implements a chosen DX variant and therefore inherits that DX's
> `scope` — here **`in-system`**. The visual foundation (design-system **v0.4**) is
> already decided, so this SOW **conforms** to it; it does **not** re-tone the
> system or touch shared tokens. One repo does the work (`prog-strength-web`);
> `prog-strength-docs` only flips the DX to `selected` and marks this SOW shipped.

## Introduction

On the run-detail page (`app/(app)/running/[id]/page.tsx`), the **pace trace** is
where a runner should be able to see *"where did I speed up and where did I
fade?"* across the whole run — the thing the per-split ledger above it can't show,
because the ledger only speaks in split averages.

Today it can't answer that question. The live `PaceStrip.tsx` is a 64px-tall
hairline sparkline with no axis, no labels, no gridlines, and no anchor of any
kind — a jagged line floating in a card over a `PACE TRACE · FASTER ↑` caption.
The maths underneath are honest (pace is winsorized so a GPS dropout can't spike
the line, the scale is inverted so faster reads *higher*, and the path breaks
cleanly across a dropout instead of bridging it), but the presentation throws all
of that away: the strip carries **no readable quantity** (you can't tell whether a
dip was a 4:30/km or a 6:00/km), **no anchors** (no distance axis, so "faster
somewhere in the middle" is the most it says), and it's **too small to trust** (at
64px the winsorized jitter still reads as a scribble).

The Design Exploration `dx/pace-trace.md` (PR Prog-Strength/prog-strength-web#104)
explored that surface across **five `in-system` variants** — all sharing
design-system v0.4 (soft near-black field, single periwinkle accent
`--accent` `#9aa6d6`, Manrope), diverging only on form, density, and prominence,
and all keeping the two load-bearing conventions: faster never reads as lower, and
a dropout never bridges as a continuous line. The owner selected
**`editorial-area-recap`** (reference: **Strava**'s run-recap area chart): the
trace is **promoted to a hero area chart** — pace as a filled region under the
curve, a real **y-axis labelled in m:ss per unit**, a **distance x-axis**, and the
run's story annotated inline (the fastest sample, the dropout). The axes and
annotations are precisely what stop the line from floating; the low-opacity accent
fill gives it the visual weight the 64px strip never had.

Once this ships, the pace trace stops being a decorative scribble and becomes a
readable chart: a runner can point to a moment in the run and read an actual pace
off the axis, see exactly where the negative split kicked in, and see honestly
where the GPS dropped — all without a legend.

## Proposed Solution

Replace the live supporting strip with a production **area-chart** component that
implements the `editorial-area-recap` variant against the **real route and the
real data**. The mockup on the DX branch
(`app/design-explore/pace-trace/_components/VariantEditorialAreaRecap.tsx` +
`fixtures.ts`) is the **visual spec, not code to promote** — it draws against
throwaway fixtures, hard-codes its own pace formatter, carries a gallery-only
`compact` mode, and has no accessibility or real-data-density handling.

Concretely:

- **Rename and rebuild** `app/(app)/running/[id]/_components/PaceStrip.tsx` →
  `PaceRecap.tsx`. The name "strip" no longer fits a hero chart. It stays pure and
  prop-driven over the existing derivation output; it gains one prop (`unit`) for
  its axis labels.
- **Wire it to the real derivation.** It already receives
  `derivation.paceStrip` (`PaceStripPoint[]`) and `derivation.hasDropout`. No new
  data is derived and **`prog-strength-api` is not touched** — every input already
  exists on the detail GET and in `lib/running-splits.ts`.
- **Add one tested pure helper** for the m:ss axis labels, because the existing
  unit-context formatters are the *wrong* tool here (see Algorithms — they convert
  sec/km → unit, but `paceStrip` pace is already per-unit, so they would
  double-convert).
- **Promote it on the page** to a full-height chart in its existing slot beneath
  the splits ledger (see Open Questions for the placement call).

The hero composition — axis frame, gradient area, inline annotations — is the
mockup's job and is settled. The real engineering, the part the mockup doesn't
solve, is **real-data density** (hundreds-of-buckets long runs, multiple dropout
windows, overcrowded x-ticks), the **pace-formatting hazard**, **accessibility**,
and **responsive behaviour** on a narrow viewport.

## Goals and Non-Goals

### Goals

- Replace `PaceStrip` with a hero area chart: filled pace area, a labelled m:ss
  y-axis, a distance x-axis with the unit suffix, faithful to the
  `editorial-area-recap` mockup.
- Preserve both load-bearing conventions: **faster pace sits higher** (inverted y),
  and the line/area **break at every dropout** — never bridged.
- Read correctly off real data: pace axis labels show the **per-unit** pace (no
  double conversion), in km or mi per the user's `useDistanceUnit` preference.
- Handle every required state against real data: **dense** (long run, many
  buckets), **sparse** (≤ a handful of points), **dropout** (one or more `null`
  windows), and **empty** (< 2 plottable points → restyled "No pace data").
- Keep the "dropout bridged" affordance: when `hasDropout`, surface it in the
  subtitle and mark the dropout window on the chart.
- Be accessible: the SVG carries an `aria-label`/`<title>` summarising the run's
  pace (fastest / slowest / dropout), so it isn't an unlabelled graphic.
- Stay `in-system`: only existing v0.4 tokens (`--accent`, `--accent-soft`,
  `--accent-line`, `--surface`, `--surface-3`, `--border`, `--foreground`,
  `--muted`, `--faint`, `--radius-card`, `--radius-pill`). No new tokens, no
  re-tone.

### Non-Goals

- **No API or derivation change.** `prog-strength-api` and the `buildPaceStrip`
  maths in `lib/running-splits.ts` are untouched; this is presentation only.
- **No new chart form.** The four rejected idioms (split bars, gridded analytical,
  rolling band, gradient strip) are not built. This SOW implements the one
  selected variant.
- **No mobile.** `prog-strength-mobile` is out of scope; this is the web
  run-detail page only.
- **No change to the page's other elements** — the splits ledger
  (`SplitsSpine`), header band, heart-rate zones, and page orchestration
  (auth/load, back-link, rename, delete, plan banner) are left as-is apart from the
  one mount line.
- **No promotion of the DX scaffolding.** The throwaway
  `app/design-explore/pace-trace/` tree is not merged; the DX PR is closed, never
  merged.
- **No interactivity.** Tap-to-scrub, crosshair, and tooltips are not in this SOW
  (the variant is a static recap); they can be a follow-up.

## Implementation Details

### Component

`app/(app)/running/[id]/_components/PaceRecap.tsx` (renamed from `PaceStrip.tsx`),
pure and prop-driven:

```ts
function PaceRecap({
  points,      // derivation.paceStrip — PaceStripPoint[]
  hasDropout,  // derivation.hasDropout — boolean
  unit,        // "km" | "mi" from useDistanceUnit (for axis labels)
}): JSX.Element
```

`PaceStripPoint` is unchanged: `{ distanceUnit: number; paceSecPerUnit: number | null }`,
where `paceSecPerUnit` is winsorized pace in **seconds per active unit**, and
`null` marks a dropout/gap that must break the line.

Geometry follows the mockup: a single responsive `<svg viewBox="0 0 W H">` with
`className="w-full"`, an x-domain from min→max `distanceUnit`, and an **inverted**
y-domain (fastest pace at top) padded ~18% so the curve never kisses the frame.
The path is built by walking `points`, emitting a fresh `M` after every `null` so
each clean run is its own line+area segment (`area` closes down to the baseline
and back); this is what keeps the dropout from bridging.

Drop the mockup's `compact` prop — production always renders the hero form. The
empty branch (`plottable.length < 2`) renders the restyled "No pace data" card.

### Algorithms

**Pace formatting — do not double-convert.** This is the one real correctness trap.
The unit context exposes `formatPace(secPerKm)` / `formatPaceValue(secPerKm, unit)`,
which **convert** a sec/km value into the active unit and format it. But
`paceStrip` pace is **already** in seconds per active unit (the derivation took
`unit` upstream). Passing it through those formatters would apply the km↔mi factor
a second time — in miles, a real `8:03/mi` (483 sec/unit) gets re-scaled to a
nonsense `~12:57`. (In km the factor is 1, so the bug hides in km and only shows in
miles — exactly the kind of thing that slips past a km-only test.) The axis labels
and the fastest/slowest subtitle therefore need a **bare seconds → "m:ss"**
formatter that does **no** unit conversion:

```ts
// seconds (already per active unit) → "m:ss"; em-dash for non-finite.
export function formatPaceClock(sec: number): string {
  if (!Number.isFinite(sec)) return "—";
  const total = Math.round(sec);
  return `${Math.floor(total / 60)}:${String(total % 60).padStart(2, "0")}`;
}
```

This mirrors the mockup's `fmtPace`. Add it as an exported, unit-tested helper
(co-located in `PaceRecap.tsx` or a small `lib/pace-format.ts`) — not as another
method on the unit context, so it can't be confused with the converting formatters.

**Dense-data handling.** Real long runs produce many buckets and possibly more than
one dropout window. Two guards beyond the mockup:

1. **x-ticks must not overcrowd.** The mockup emits one tick per integer distance
   (fine at 5 km, 18 ticks at 18 km, worse for ultras). Cap the axis to ~6–8 ticks
   by choosing a "nice" integer step (1, 2, 5, 10 …) from the distance span, so
   labels stay readable and never collide.
2. **Annotations stay singular.** Pick exactly one "fastest" sample (global min
   pace) to annotate, and label the dropout once. With multiple dropout windows,
   render the muted band behind **each** gap but only one "dropout" caption near
   the first — many captions would clutter the hero.

### Page integration

In `app/(app)/running/[id]/page.tsx`, update the single mount line to the renamed
component and pass the active unit:

```tsx
// before
<PaceStrip points={derivation.paceStrip} hasDropout={derivation.hasDropout} />
// after
<PaceRecap points={derivation.paceStrip} hasDropout={derivation.hasDropout} unit={unit} />
```

`unit` is already in scope from `useDistanceUnit()`. Everything else on the page —
the header band, `SplitsSpine`, `HeartRateZones`, and all orchestration — is
unchanged. See Open Questions for whether the chart's slot moves.

### Accessibility & responsive

- The `<svg role="img">` gains an `aria-label` (or child `<title>`) summarising the
  chart: e.g. *"Pace recap: fastest 4:51, slowest 5:48 per km; one GPS dropout."*
  Decorative gridlines/ticks stay unlabelled.
- The chart is width-responsive via `viewBox` + `w-full`. Verify the m:ss y-axis
  labels and x-ticks remain legible at a phone-width column; if 12px axis text
  crowds, step the x-ticks down (per the density guard) rather than shrinking type
  below the design system's floor.

### Tests

Extend `app/(app)/running/[id]/page.test.tsx` and/or add a focused
`PaceRecap.test.tsx`:

- **Dropout breaks the path** — a fixture with a `null` window yields **two**
  `<path>` line segments (assert the path count / multiple `M` commands), never a
  single bridged path.
- **Inversion** — the fastest sample renders at a **smaller** y than the slowest
  (faster is higher).
- **Axis labels** — y-axis ticks format as m:ss via `formatPaceClock` and are
  **not** doubly converted (a 348 sec/unit point labels as `5:48`, not `~3:36`).
- **Empty** — < 2 plottable points renders the "No pace data" card, not an SVG.
- **Subtitle** — `hasDropout` toggles the "dropout bridged" copy.
- **Helper unit test** — `formatPaceClock` for typical, sub-minute, and non-finite
  inputs.

## Open Questions

1. **Where does the hero chart sit on the page?** The variant *promotes* the trace,
   but the splits ledger (`SplitsSpine`, itself a previously-selected DX hero) is
   the page's structural backbone. Options: (a) keep the chart in its current slot
   **beneath** the ledger, just full-height; (b) move it **above** the ledger as
   the page's lead visual. **Tentative lean: (a)** — the ledger stays the backbone
   and the pace recap is the hero *visual* beneath it, which keeps one structural
   spine and avoids two competing heroes. Cheap to flip if it reads better on top.
2. **Keep the "Faster is higher" pill?** The mockup shows a pill badge stating the
   inverted convention. It's helpful but it's chrome. **Tentative lean: keep it** —
   inverted-pace axes confuse people and one small pill is cheaper than a
   mislabelled-axis support question. Drop it only if it crowds the header on
   narrow widths.
