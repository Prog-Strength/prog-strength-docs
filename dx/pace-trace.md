---
type: dx
status: awaiting_selection
surface: pace-trace
idioms:
  - editorial-area-recap
  - terminal-dense-analytical
  - split-aligned-bars
  - warm-organic-rolling-band
  - linear-minimal-gradient-strip
references:
  - Strava
  - Runalyze
  - Whoop
  - Apple Fitness
  - Garmin Connect
scope: in-system
variant_count: 5
repos:
  - prog-strength-web
  - prog-strength-docs
---

# DX: Pace Trace

**Status**: Awaiting selection · **Last updated**: 2026-06-29

> A DX (Design Exploration) is the platform's **divergent** work type. Unlike a
> SOW it does not converge on one correct implementation — it produces N
> differentiated visual variants of a single frontend surface, side by side on
> one comparison route, awaiting a human pick. It **never merges** and ships no
> production code; the chosen direction feeds a downstream SOW that builds it for
> real.

## Context

The **pace trace** lives on the run-detail page (`/running/[id]`), today a
demoted supporting strip tucked beneath the splits ledger (`PaceStrip.tsx`). Its
job is to answer a simple question a runner asks of every run — *"where did I
speed up and where did I fade?"* — by drawing pace across the whole distance,
not just the per-split averages the ledger already gives.

The data underneath it is good and the maths are honest: pace is winsorized so a
GPS dropout can't spike the line, the scale is inverted so faster reads as
*higher* (a rising line never means slowing down), and the path breaks cleanly
across a missing sample rather than lying with a straight bridge. None of that is
in question here.

What's unsatisfying is the **presentation**. The trace is a 64px-tall hairline
sparkline with no axis, no labels, no gridlines, and no anchor of any kind — a
jagged line floating in a card over a `PACE TRACE · FASTER ↑` caption. Three
things go wrong because of it:

- **It has no scale, so it carries no quantity.** You can see the line wobble,
  but you can't read what pace any point *is* — was that surge a 4:30/km or a
  6:00/km? The shape is there; the numbers it implies are not.
- **It has no anchors, so it floats.** With no distance axis and no start/finish
  marks, the runner can't say *where* on the run a given dip happened. "Faster
  somewhere in the middle" is the most it communicates.
- **It's too small to trust.** At 64px the winsorized noise still reads as a
  dense scribble, so the eye can't separate a real pace surge from sample jitter.
  The honest dropout handling is invisible at this size.

And there's a prior question worth putting on the table: a continuous line may
not even be the right *form* for run pace. The splits ledger already speaks in
per-split averages; a line speaks in instantaneous samples; bars, bands, and
rolling curves each tell a different version of the same run. This DX explores
the **form** as much as the styling — the variants are allowed to abandon the
line entirely if a better shape earns it.

## The surface

The component renders one run's pace series against distance. It is pure and
prop-driven — it receives a derived series and a dropout flag and draws; all the
winsorizing and bucketing happens upstream in `lib/running-splits.ts`.

**Props / data shape** (`PaceStripPoint[]` plus `hasDropout: boolean`):

```ts
type PaceStripPoint = {
  distanceUnit: number;          // cumulative distance in the user's active unit (km or mi)
  paceSecPerUnit: number | null; // winsorized pace, sec per unit; null = dropout/gap → break the line
};
```

The page also has the splits ledger sitting directly above this widget, so a
variant may align to it (shared x-domain, column-aligned bars) or stand fully
independent.

**Realistic fixture** — a ~5 km run in kilometres, a negative split with one mid-run
GPS dropout (the two `null`s) and a closing surge:

```ts
const points: PaceStripPoint[] = [
  { distanceUnit: 0.0, paceSecPerUnit: 348 }, // 5:48/km — easing in
  { distanceUnit: 0.5, paceSecPerUnit: 335 },
  { distanceUnit: 1.0, paceSecPerUnit: 330 },
  { distanceUnit: 1.5, paceSecPerUnit: 322 },
  { distanceUnit: 2.0, paceSecPerUnit: 318 },
  { distanceUnit: 2.5, paceSecPerUnit: null }, // dropout — line breaks here
  { distanceUnit: 3.0, paceSecPerUnit: null },
  { distanceUnit: 3.5, paceSecPerUnit: 312 },
  { distanceUnit: 4.0, paceSecPerUnit: 305 },
  { distanceUnit: 4.5, paceSecPerUnit: 298 }, // 4:58/km — closing surge
  { distanceUnit: 5.0, paceSecPerUnit: 291 },
];
const hasDropout = true;
```

**States every variant must handle:**

- **Dense** — a long run (15–25 km) with hundreds of underlying samples; the form
  must stay legible, not collapse into a scribble.
- **Sparse** — a short run (≤ 2 km) with only a handful of points; the form must
  not look broken or empty when there's barely a curve to draw.
- **Dropout** — one or more `null` runs in the middle; the gap must read as
  *"data missing here,"* never as a flat fast/slow stretch. When any sample was
  winsorized, the `dropout bridged` affordance still needs a home.
- **Empty** — fewer than two plottable points → the current muted *"No pace data"*
  placeholder, restyled to the variant.

**Fixed conventions** that hold across every variant (they're correct and
load-bearing): faster pace must never read as *lower* than slower pace, and a
dropout must never be drawn as a continuous line across the gap.

## Idioms

Five directions. Each pairs a **chart form** with its own **type scale**, **color
logic**, and **spacing rhythm** — the spread is meant to test whether the line is
salvageable, replaceable, or wrong. Each variant also decides the widget's
**prominence** on the page (hero chart vs. refined strip); that's part of what's
being explored.

- **editorial-area-recap** — Promote the trace to a **hero area chart**: pace as a
  filled region under the curve, a real y-axis labelled in mm:ss per unit, a
  distance x-axis, and the run's story annotated inline ("fastest km," the surge,
  the dropout). Large editorial type scale, generous margins, a single accent
  fill at low opacity over charcoal. The line stops floating because the axes and
  annotations anchor it. Leans on **Strava**'s run-recap area chart.

- **terminal-dense-analytical** — Keep the line but make it a **serious analytical
  instrument**: full background gridlines, horizontal pace-zone bands, distance
  ticks along x, monospace tabular numerals on both axes, tight spacing rhythm,
  minimal chrome. Small type, high data-ink density, nothing decorative. This is
  the "training tool" reading — it answers *what pace, exactly, where* without a
  single annotation. Leans on **Runalyze**'s gridded pace/power traces.

- **split-aligned-bars** — Abandon the continuous line entirely. Render pace as
  **per-split bars** — one bar per split, column-aligned to the splits ledger
  directly above, fastest and slowest marked, bar length encoding pace (inverted:
  longer/taller = faster). The dropout becomes a single muted/hatched bar rather
  than a gap in a line. Calm, even spacing rhythm that rhymes with the ledger's
  rows; medium type. This is the direct counter-proposal to the line, and the one
  that most reuses what the page already knows. Leans on **Apple Fitness** and
  **Garmin Connect** split breakdowns.

- **warm-organic-rolling-band** — Smooth the jaggedness instead of fighting it:
  draw a **rolling-average pace curve** as the hero line, with the raw winsorized
  samples behind it as a faint shaded **envelope/band** showing variability. Soft,
  rounded curve; warmer, lower-contrast palette; relaxed spacing; a few gentle
  labels at start / mid / finish rather than a full axis. The dropout reads as a
  break in *both* the band and the curve. This is the most direct answer to "the
  noise makes it unreadable" — it separates signal from jitter visually. A blend
  direction, grounded in the calm end of **Strava** and **Apple Fitness**.

- **linear-minimal-gradient-strip** — Stay a **compact strip**, but make it
  readable through **color, not gridlines**: a single horizontal pace band where
  hue/intensity along its length encodes relative effort (a one-accent gradient on
  charcoal), with just two or three anchor labels (start pace, fastest, finish
  pace) and a slim distance baseline. Minimal type, maximal restraint, the
  tightest vertical footprint of the five — the bet that the demoted strip is
  actually the right call and only needs *legibility*, not promotion. Leans on
  **Whoop**'s single-accent-on-charcoal effort bands.

## References

- **Strava** — the run-recap **area chart**: filled pace region against distance,
  split markers, tap-to-scrub, grade shading. The genre default for run pace; take
  its area-fill-as-hero and inline split annotation, not its busier overlays.
- **Runalyze** — analytical **gridded traces**: real gridlines, pace-zone y-bands,
  monospace axis numerals, high data-ink density. Take its rigor — the sense that
  every point has a readable value.
- **Whoop** — **single-accent-on-charcoal** effort/strain bands with almost no
  axis chrome, reading through color intensity rather than gridlines. Take its
  palette discipline and its compact, label-light band; it's the closest fit to
  the v0.4 dark surface.
- **Apple Fitness** — clean **split-bar pace breakdowns** for general athletes:
  even rows, fastest/slowest emphasis, calm spacing. Take its bar-per-split
  legibility and restraint.
- **Garmin Connect** — split tables and **rolling-pace curves** with honest axes.
  Take its split-aligned bar treatment and its willingness to show a smoothed pace
  curve over raw samples.

## Selection criteria

A note-to-self for the pick, not a rubric for the worker to optimize against:

- **Does it read at a glance, without a legend?** The whole complaint is that
  today's strip means nothing on first look. The winner should answer "where did
  I speed up / fade" in under a second.
- **Can I read a quantity off it?** At least the fast/slow extremes should carry
  an actual mm:ss value — the line currently carries none.
- **Does the dropout stay honest?** The gap must read as missing data, never as a
  flat stretch, and the `dropout bridged` signal must survive in whatever form the
  variant takes.
- **Does the "faster = up / longer / brighter" convention hold — or get replaced
  by something clearer?** Inverted-pace confuses people; a variant that makes
  direction obvious without a caption is a plus.
- **Does it earn its vertical space?** This is the real fork: a hero chart has to
  justify pushing the rest of the page down, while a refined strip has to prove it
  can be legible while staying small. Decide which bet the page actually wants —
  and whether the splits ledger above makes the continuous line redundant or
  complementary.

---

> **Lifecycle.** `status:` is editorial — the owner is the dispatch gate. It moves
> `draft` → `exploring` (worker running) → `awaiting_selection` (draft PR open,
> owner deciding) → `selected` / `abandoned`. The worker sets `awaiting_selection`
> on the `dx/pace-trace` branch as it opens the PR; the owner sets the terminal
> value when they close it.
>
> **Handoff.** A DX ends at a *chosen variant*, not merged code. Open the draft
> `[DX — DO NOT MERGE]` PR, compare the variants on the preview deploy, pick one,
> tick its box, set `status: selected` (noting the winning idiom), and **close the
> PR — never merge it.** Then open a normal SOW: *"implement `pace-trace` per the
> `<chosen-idiom>` variant from `dx/pace-trace`, production-quality, conforming to
> the design system."* The mockup code is never promoted as-is.
