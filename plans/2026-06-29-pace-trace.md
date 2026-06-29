# Implementation Plan — Pace Trace (Editorial Area Recap)

**SOW**: [`sows/pace-trace.md`](../sows/pace-trace.md) · **Created**: 2026-06-29

## Summary

Replace the 64px supporting `PaceStrip` sparkline on the run-detail page with a
production hero **area chart** (`PaceRecap`) that implements the
`editorial-area-recap` DX variant against the real derivation data. Presentation
only — no API or derivation change. One repo: `prog-strength-web`.

This plan conforms to design-system **v0.4** (soft near-black field, single
periwinkle `--accent`, Manrope). No new tokens.

## Scope / files

- **New** `lib/pace-format.ts` — exported pure `formatPaceClock(sec)` helper +
  `lib/pace-format.test.ts`.
- **Rename + rebuild** `app/(app)/running/[id]/_components/PaceStrip.tsx` →
  `PaceRecap.tsx` (hero area chart, `unit` prop added).
- **New** `app/(app)/running/[id]/_components/PaceRecap.test.tsx`.
- **Edit** `app/(app)/running/[id]/page.tsx` — swap the single mount line
  `<PaceStrip …>` → `<PaceRecap … unit={unit} />` and update the import + the
  page's doc comment wording ("pace strip" → "pace recap").

Untouched: `prog-strength-api`, `lib/running-splits.ts` (derivation maths),
`SplitsSpine`, `RunHeaderBand`, `HeartRateZones`, page orchestration, the
`app/design-explore/pace-trace/` DX scaffolding (never merged).

## Open-question resolutions (per SOW tentative leans)

1. **Placement**: keep the chart in its **current slot beneath** the splits
   ledger (option a). One structural spine; the recap is the hero visual below.
2. **"Faster is higher" pill**: **keep it** in the header, conforming to the
   mockup. Hide/wrap gracefully on narrow widths rather than crowding.

## Task 1 — `formatPaceClock` helper (+ test)

`lib/pace-format.ts`:

```ts
/** seconds (already per active unit) → "m:ss"; em-dash for non-finite. */
export function formatPaceClock(sec: number): string {
  if (!Number.isFinite(sec)) return "—";
  const total = Math.round(sec);
  return `${Math.floor(total / 60)}:${String(total % 60).padStart(2, "0")}`;
}
```

Crucially this does **no unit conversion** — `paceStrip` pace is already
per-active-unit, so the context's `formatPace`/`formatPaceValue` (which apply
the km↔mi factor) would double-convert. Keep this helper separate from the
unit context so it can't be confused with the converting formatters.

Test `lib/pace-format.test.ts`: typical (`348 → "5:48"`), sub-minute
(`42 → "0:42"`), rounding (`299.6 → "5:00"`), and non-finite (`NaN`,
`Infinity` → `"—"`).

## Task 2 — `PaceRecap` component (+ test)

Rebuild as a pure, prop-driven hero area chart. Signature:

```ts
function PaceRecap({
  points,      // derivation.paceStrip — PaceStripPoint[]
  hasDropout,  // derivation.hasDropout — boolean
  unit,        // "km" | "mi" from useDistanceUnit (axis labels)
}): JSX.Element
```

Geometry (follows mockup `VariantEditorialAreaRecap`, **hero form only — drop
the `compact` prop**):

- Single responsive `<svg viewBox="0 0 W H" className="w-full" role="img">`.
- x-domain min→max `distanceUnit`; **inverted** y-domain (fastest at top),
  padded ~18% so the curve never kisses the frame.
- Path walks `points`, emitting a fresh `M` after every `null` so each clean
  run is its own line + area segment (area closes to baseline). **Never bridge a
  dropout.**
- Empty branch: `plottable.length < 2` → restyled "No pace data" card (no SVG).

Productionizing beyond the mockup:

- **Pace labels via `formatPaceClock`** (Task 1) — no double-convert. y-axis
  ticks and the fastest/slowest subtitle use it.
- **x-tick density guard**: cap to ~6–8 ticks via a "nice" integer step
  (1, 2, 5, 10, 20, 50…) chosen from the distance span, so an 18 km / ultra run
  doesn't render colliding per-integer ticks. Append the unit suffix to the last
  tick.
- **Annotations stay singular**: annotate exactly one global-min "fastest"
  sample; render the muted dropout band behind **each** `null` gap but only one
  "dropout" caption (near the first gap).
- **Accessibility**: `role="img"` + `aria-label` (or child `<title>`)
  summarising the run, e.g. *"Pace recap: fastest 4:51, slowest 5:48 per km;
  one GPS dropout."* Decorative gridlines/ticks stay unlabelled
  (`aria-hidden` where appropriate).
- **Header**: "Pace recap" title, subtitle `Fastest {lo} · slowest {hi} /{unit}`
  with the dropout-bridged clause appended when `hasDropout`; keep the
  "Faster is higher" pill.
- **Tokens only** (v0.4): `--accent`, `--accent-soft`, `--accent-line`,
  `--surface`, `--surface-3`, `--border`, `--foreground`, `--muted`, `--faint`,
  `--radius-card`, `--radius-pill`. Gradient area = low-opacity `--accent`.

Test `PaceRecap.test.tsx`:

- **Dropout breaks the path** — a `null`-window fixture yields **two** `<path>`
  line segments (multiple `M`s), never one bridged path.
- **Inversion** — the fastest sample renders at a **smaller** y than the slowest.
- **Axis labels** — y ticks format m:ss via `formatPaceClock`, **not** doubly
  converted (a `348` sec/unit point labels `5:48`, not `~3:36`/`~12:57`).
- **Empty** — `< 2` plottable points renders "No pace data", not an SVG.
- **Subtitle** — `hasDropout` toggles the dropout-bridged copy.
- **Accessibility** — the SVG carries an `aria-label`/`<title>` summary.

## Task 3 — Page integration

`app/(app)/running/[id]/page.tsx`:

```tsx
// import
import { PaceRecap } from "./_components/PaceRecap";
// mount (current slot, beneath SplitsSpine)
<PaceRecap points={derivation.paceStrip} hasDropout={derivation.hasDropout} unit={unit} />
```

`unit` is already in scope from `useDistanceUnit()`. Update the page doc-comment
wording. Nothing else changes. Extend `page.test.tsx` only if needed (the
focused `PaceRecap.test.tsx` carries the chart assertions).

## Verification gate (before push)

`npm run lint && npm run format:check && npm run typecheck && npm run test && npm run build`
— all green. Husky `pre-commit` (lint-staged + tsc) and `commit-msg`
(commitlint) hooks run; never bypass. Conventional Commit messages.

## Out of scope (non-goals)

API/derivation change; mobile; other page elements; the four rejected idioms;
DX scaffolding promotion; interactivity (scrub/crosshair/tooltips).
