# Running Activity Detail — Splits-Ledger-Spine Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Rebuild the `/running/[id]` detail body from an undifferentiated metrics grid into the `splits-ledger-spine` composition — a per-distance splits table backbone with a miles↔intervals toggle and a demoted winsorized pace strip — backed by a new pure derivation layer that turns live trackpoints into splits, a cleaned pace trace, and detected intervals.

**Architecture:** A new pure, React-free, unit-aware module `lib/running-splits.ts` derives everything visual from `RunningSession.trackpoints` (+ the linked plan's `run_type` + the active distance unit). The page keeps all its existing orchestration (auth/load, rename, delete, plan banner/unlink, states) and swaps only the render body, computing the derivation in a single `useMemo` and feeding three new prop-driven presentational components. No API, schema, or design-system change — conforms to design-system v0.4 tokens.

**Tech Stack:** Next.js 16 App Router, React 19, TypeScript, Tailwind v4 (CSS-variable tokens), Vitest + Testing Library. Charts are hand-rolled SVG (no recharts on this page after the rebuild).

---

## Background & Constraints (read before any task)

- **Repo:** `prog-strength-web`. Branch `feat/running-activity-detail` off `main`.
- **Conventional commits required** (commitlint + Husky `commit-msg`). Use `feat(running): …` / `test(running): …` style. Do NOT use `--no-verify` or `HUSKY=0`.
- **Pre-commit hook** runs lint-staged (ESLint `--fix` + Prettier) then `tsc --noEmit`. Let it run.
- **CI gate** (must be green locally before push): `npm run lint`, `npm run format:check`, `npm run typecheck`, `npm run test`, `npm run build`.
- **All distances/elevations are stored in meters; paces in seconds-per-kilometer.** Conversion to the user's unit happens at the presentation boundary. Keep stored/derived metric values metric internally; convert pace to per-active-unit only where the table/strip need it.
- **Design-system v0.4 tokens only** — never hard-code hex that duplicates a token. Relevant tokens:
  - Surfaces: `--surface` `#15171b`, `--surface-2` `#191c21`, `--border`, `--radius-card` (14px).
  - Text: `--foreground`, `--muted`, `--faint`.
  - Run discipline hue: `--discipline-run-bg`, `--discipline-run-fg`, `--discipline-run-dot` (green-teal — used for the pace column, the ✓ pill, the work dot; NEVER the accent).
  - State: `--success` `#86b39f`, `--danger` `#c79292` (fastest/slowest tags, vs-target).
  - Accent (`--accent` / `--accent-fg`) is app chrome — used only for the active segment of the segmented toggle, per the form-controls spec.
  - Tabular figures: Tailwind `tabular-nums` + `tracking-[-0.03em]` on numeric cells.
- **The DX mockup is the visual spec, not code to copy.** Reference (read once, do not import): branch `origin/dx/running-activity-detail`, files `app/design-explore/running-activity-detail/_components/SplitsLedgerSpine.tsx` and `app/design-explore/running-activity-detail/_fixtures.ts`. The mockup hard-codes `SPLITS`/`SEGMENTS`/`TRACE`; this plan replaces them with real derivations.

### Key existing types (in `lib/api.ts`, do not redefine)

```ts
export type RunType = "easy" | "threshold" | "intervals";

export type RunningTrackpoint = {
  sequence: number;
  elapsed_seconds: number;
  distance_meters: number;
  heart_rate_bpm: number | null;
  pace_sec_per_km: number | null;
  elevation_meters: number | null;
};

export type RunningSession = {
  /* …summary fields… */ trackpoints?: RunningTrackpoint[];
};

export type PlannedWorkout = {
  /* … */ run_type: RunType | null; run_details: string | null; /* … */
};
```

### Conversion constants (already used across the app)

```ts
const METERS_PER_MILE = 1609.344;
const METERS_PER_KM = 1000;
// secPerMile = secPerKm * (METERS_PER_MILE / METERS_PER_KM)
const KM_PER_MILE = METERS_PER_MILE / METERS_PER_KM; // 1.609344
```

---

## File Structure

- **Create:** `lib/running-splits.ts` — pure derivation module (splits, winsorized pace strip, interval detection, target-pace parsing). One responsibility: turn trackpoints → ledger data.
- **Create:** `lib/running-splits.test.ts` — the priority test suite.
- **Create:** `app/(app)/running/[id]/_components/RunHeaderBand.tsx` — the compact gridded summary band + ✓ pill/Unlink + prescription context text.
- **Create:** `app/(app)/running/[id]/_components/SplitsSpine.tsx` — the "Splits" label + miles↔intervals segmented toggle + the two tables.
- **Create:** `app/(app)/running/[id]/_components/PaceStrip.tsx` — the demoted winsorized pace sparkline.
- **Modify:** `app/(app)/running/[id]/page.tsx` — keep chrome/orchestration; replace the StatTile grid + recharts block with the band + spine + strip wired to a `useMemo` derivation. Add an inline `handleUnlink`.
- **Create:** `app/(app)/running/[id]/page.test.tsx` — component test (renders splits, toggle gating, rename/delete/unlink still fire).

The three recharts components (`running/_components/{HeartRateChart,PaceChart,ElevationChart}.tsx`) are only imported by this page today; after the rebuild they become unused here. **Leave the files in place** (do not delete — out of scope), just stop importing them.

---

## Task 1: Derivation module `lib/running-splits.ts` (+ tests) — the priority

**Files:**
- Create: `lib/running-splits.ts`
- Test: `lib/running-splits.test.ts`

This is the engineering core. It is pure (no React, no fetch), deterministic, and unit-aware. TDD: write the test for each behavior, watch it fail, implement, watch it pass.

### Public interface (define exactly these names)

```ts
import type { RunningTrackpoint, RunType } from "./api";

export type DistanceUnit = "mi" | "km";

/** Pace slower than this (sec/km) is treated as a device dropout and winsorized
 *  out of stats + plotted as a gap. 410 sec/km ≈ 11:00 /mi (the mockup's
 *  PACE_CLAMP_MAX of 660 sec/mi, expressed in the stored metric unit). */
export const PACE_DROPOUT_SEC_PER_KM = 410;

export type Split = {
  /** 0-based bucket index along the run. */
  index: number;
  /** True for the trailing bucket that is shorter than one full unit. */
  partial: boolean;
  /** Real bucket distance in meters (full ≈ one unit; partial < one unit). */
  distanceMeters: number;
  /** Real elapsed time across the bucket, seconds. */
  durationSec: number;
  /** Average pace in sec per ACTIVE UNIT, computed from clean samples only;
   *  null when the bucket has no clean distance. */
  avgPaceSecPerUnit: number | null;
  /** Average HR over samples with HR; null when none. */
  avgHr: number | null;
  /** Elevation change over the bucket in meters; null when no barometric data. */
  elevDeltaMeters: number | null;
  /** Marked among FULL splits only (≥2 full splits required to mark). */
  fastest: boolean;
  slowest: boolean;
};

export type PaceStripPoint = {
  /** Cumulative distance in the active unit. */
  distanceUnit: number;
  /** Winsorized pace in sec per active unit; null = dropout/gap → break the line. */
  paceSecPerUnit: number | null;
};

export type SegmentKind = "warmup" | "work" | "recovery" | "cooldown";

export type IntervalSegment = {
  kind: SegmentKind;
  /** 1-based index among work bouts (recoveries share the preceding rep's number). */
  rep: number | null;
  label: string; // "Warm-up" | "Rep 1" | "Recovery 1" | "Cool-down"
  distanceMeters: number;
  durationSec: number;
  avgPaceSecPerUnit: number | null;
  avgHr: number | null;
};

export type RunningDerivation = {
  splits: Split[];
  paceStrip: PaceStripPoint[];
  /** Present (non-null) only when intervals were confidently detected. */
  intervals: IntervalSegment[] | null;
  /** True if any sample was winsorized (powers the strip's "dropout bridged" note). */
  hasDropout: boolean;
  /** True if any trackpoint carried elevation data. */
  hasElevation: boolean;
  /** True if any trackpoint carried HR data. */
  hasHr: boolean;
};

/** Top-level entry point the page calls in a useMemo. */
export function deriveRunningActivity(
  trackpoints: RunningTrackpoint[],
  runType: RunType | null,
  unit: DistanceUnit,
): RunningDerivation;

/** Tolerant parser: extract a target work pace from free-text run_details.
 *  Returns sec per ACTIVE UNIT, or null when nothing is confidently found. */
export function parseTargetPace(
  runDetails: string | null,
  unit: DistanceUnit,
): number | null;
```

### Algorithm details (implement precisely; tests below pin the behavior)

**Helpers**
- `const bucketMeters = unit === "mi" ? METERS_PER_MILE : METERS_PER_KM;`
- `secPerMeterToUnit(secPerMeter)` → `secPerMeter * bucketMeters`.
- A trackpoint is **clean** iff `pace_sec_per_km != null && pace_sec_per_km > 0 && pace_sec_per_km <= PACE_DROPOUT_SEC_PER_KM`.
- Iterate **consecutive pairs** `(a, b)`; a "segment" spans `a→b` with `dDist = b.distance_meters - a.distance_meters` (skip if `<= 0`) and `dTime = b.elapsed_seconds - a.elapsed_seconds` (clamp negative to 0). The segment is **clean** iff `b` is clean.

**Splits**
- Bucket a segment by `Math.floor(a.distance_meters / bucketMeters)`.
- Per bucket accumulate: `distanceMeters += dDist`, `durationSec += dTime` (always — honest totals); and for clean segments only: `cleanDist += dDist`, `cleanTime += dTime`.
- `avgPaceSecPerUnit = cleanDist > 0 ? secPerMeterToUnit(cleanTime / cleanDist) : null`.
- `avgHr`: mean of `b.heart_rate_bpm` over segments where it is non-null; null if none.
- `elevDeltaMeters`: `(last non-null elevation in bucket) - (first non-null elevation in bucket)`; null if the bucket has no elevation samples.
- `partial`: a bucket is partial iff its `distanceMeters < bucketMeters * 0.95` AND it is the last bucket.
- `fastest`/`slowest`: among **full** splits with a non-null `avgPaceSecPerUnit`, mark the min-pace one `fastest` and the max-pace one `slowest`. Only mark when there are **≥2** such full splits (otherwise leave all false).

**Pace strip**
- One `PaceStripPoint` per trackpoint (in order): `distanceUnit = distance_meters / bucketMeters`; `paceSecPerUnit = clean ? secPerKm→perUnit(pace_sec_per_km) : null` (null breaks the line at the dropout). `secPerKm→perUnit`: `unit === "mi" ? secPerKm * KM_PER_MILE : secPerKm`.

**Interval detection** (`runType === "intervals"` only; otherwise `intervals = null`)
- Build clean segments (as above) with `{ dDist, dTime, pace = secPerMeterToUnit(dTime/dDist), hr: b.heart_rate_bpm }`.
- If fewer than **8** clean segments → return `null` (too sparse to trust).
- `avgPace = (Σ dTime over clean) / (Σ dDist over clean)` → per unit. For each segment `rel = pace / avgPace`. Class: `fast` if `rel <= 0.90`, `slow` if `rel >= 1.05`, else `neutral`. Forward-fill `neutral` from the previous class (default `slow` at the start).
- Coalesce consecutive equal classes into bouts (sum dist/time, hr-mean over hr-present). Merge any bout with `distanceMeters < 60` into the previous bout (relabel to previous class) and re-coalesce — denoises jitter.
- `workBouts` = `fast` bouts. **Plausibility:** require `workBouts.length >= 3` AND every adjacent pair of work bouts is separated by at least one non-work bout (alternation). If not plausible → return `null`.
- Assign kinds: all leading `slow` bouts before the first work bout collapse to a single **warmup**; all trailing `slow` after the last work bout collapse to a single **cooldown**; `slow` bouts between works are **recovery**; `fast` bouts are **work**. Number work bouts 1..N (`rep`); a recovery takes the rep number of the work bout that precedes it.
- Labels: `"Warm-up"`, `"Rep {n}"`, `"Recovery {n}"`, `"Cool-down"`. Each segment carries `distanceMeters`, `durationSec`, `avgPaceSecPerUnit` (per unit), `avgHr` (null if none).
- **Conservative by design:** when unsure, return `null` (surface falls back to miles-only) — never fabricate reps.

**`parseTargetPace`**
- Search `runDetails` (case-insensitive) for a pace token of the form `m:ss` immediately associated with a `/mi`, `/km`, `per mile`, or `per km` unit, optionally preceded by `~` or wrapped in parens — e.g. matches `~6:30/mi`, `(6:30 /mi)`, `8:00 per km`. Regex sketch: `/(\d{1,2}):([0-5]\d)\s*\/?\s*(mi|km)\b/i` (also accept `per\s+(mile|km|kilometer)`).
- Convert the matched seconds to the active unit: if token unit is `mi` → secPerMi; convert to active unit (`mi`: as-is; `km`: `secPerMi / KM_PER_MILE`). If token unit is `km` → secPerKm; convert (`km`: as-is; `mi`: `secPerKm * KM_PER_MILE`). Round to integer seconds.
- If no confidently-unit-qualified token is found, return `null` (do NOT grab a bare number — `400m` must not parse as a pace).

### Test plan (`lib/running-splits.test.ts`)

Build small deterministic fixtures **inside the test file** (arrays of `RunningTrackpoint`). Provide a helper to synthesize a trackpoint stream from segment specs so tests stay readable. Cover:

- [ ] **Step 1: Per-mile bucketing incl. partial split.** A ~2.2 mi steady run (constant pace, dense samples) in `unit: "mi"` yields 3 splits: two full (`partial: false`, `distanceMeters ≈ 1609`) and one trailing `partial: true` with `distanceMeters` ≈ 0.2 mi. Assert split count, `partial` flags, and that full-split `distanceMeters` ≈ `METERS_PER_MILE` (within 1 m).
- [ ] **Step 2: Winsorization excludes the dropout from stats.** Insert one trackpoint with `pace_sec_per_km = 1804` (≈30:04/mi) mid-run. Assert that split's `avgPaceSecPerUnit` is close to the surrounding clean pace (NOT inflated by the glitch), and that `deriveRunningActivity(...).hasDropout === true`.
- [ ] **Step 3: Winsorization breaks the strip line.** Assert the `paceStrip` point at the dropout has `paceSecPerUnit === null`, and its neighbors are non-null (gap, not a bridge).
- [ ] **Step 4: Fastest / slowest marking.** A run with three full miles at distinctly different paces marks exactly one `fastest` (lowest pace) and one `slowest` (highest pace), and the partial split is never marked. A run with only one full split marks neither.
- [ ] **Step 5: Missing columns honest.** A fixture with all `heart_rate_bpm: null` → every split `avgHr === null` and `hasHr === false`. A fixture with all `elevation_meters: null` → every split `elevDeltaMeters === null` and `hasElevation === false`.
- [ ] **Step 6: Interval detection positive.** Synthesize an `intervals` fixture: ~1 mi warm-up (slow), then 6×(400m fast @ ~6:30/mi + 200m slow recovery), then a cool-down. With `runType: "intervals"`, assert `intervals !== null`, the work-bout count is ≥ 3 (ideally 6), kinds include `warmup` first and `cooldown` last, and `rep` numbers increment on work bouts.
- [ ] **Step 7: Interval detection negative — easy run.** The same steady fixture from Step 1 with `runType: "easy"` → `intervals === null`. Also pass the steady fixture with `runType: "intervals"` (no alternating fast bouts) → still `intervals === null` (fails closed).
- [ ] **Step 8: Interval detection negative — sparse trace.** A trace with < 8 clean segments and `runType: "intervals"` → `intervals === null`.
- [ ] **Step 9: Unit switching (mi ↔ km).** The same trackpoints derived with `unit: "km"` produce more, ~1000 m buckets; with `unit: "mi"`, ~1609 m buckets. Assert the bucket count differs accordingly and a known split's `avgPaceSecPerUnit` matches the expected sec/km vs sec/mi (ratio ≈ `KM_PER_MILE`).
- [ ] **Step 10: `parseTargetPace`.** `"… 8 × 400m @ 5K effort (~6:30/mi) …"` with `unit: "mi"` → 390 (±1). Same string with `unit: "km"` → `Math.round(390 / KM_PER_MILE)` (≈242). A string with no unit-qualified pace (e.g. `"8 × 400m hard"`) → `null`. A `/km` token converts correctly to `mi`.
- [ ] **Step 11:** Run `npx vitest run lib/running-splits.test.ts` → all pass. Commit: `feat(running): add splits/intervals derivation module`.

> Note: tolerances — pace assertions use `toBeCloseTo`/`expect(...).toBeLessThan` ranges, not exact equality, because synthesized elapsed-time integers introduce rounding. Make fixtures dense enough (e.g. samples every ~20–30 m) that bucket boundaries land cleanly.

---

## Task 2: Presentational components (`_components/`)

**Files:**
- Create: `app/(app)/running/[id]/_components/RunHeaderBand.tsx`
- Create: `app/(app)/running/[id]/_components/SplitsSpine.tsx`
- Create: `app/(app)/running/[id]/_components/PaceStrip.tsx`

All three are **pure, prop-driven** (no fetch, no context reads except where noted) so they can be built before wiring. Mirror the mockup's structure (read `SplitsLedgerSpine.tsx` once for the visual) but type the props to the derivation output. Match neighboring Tailwind/token usage.

### `RunHeaderBand.tsx`

Props:
```ts
{
  cells: { label: string; value: string }[]; // 8 pre-formatted cells from the page
  planName?: string | null;       // when linked → green-teal ✓ pill
  prescription?: string | null;    // run_details text → context line
  onUnlink?: () => void;           // present when linked
  unlinking?: boolean;
}
```
- Renders (only when `planName`): a row with the **green-teal ✓ pill** — `✓ {planName}` styled with `background: var(--discipline-run-bg); color: var(--discipline-run-fg)`, rounded-full, `text-[10px] font-semibold uppercase tracking-wider` — and, beside it, an **Unlink** text button (`text-xs text-[var(--muted)] hover:text-[var(--danger)]`, disabled while `unlinking`, label `Unlinking…`/`Unlink`).
- Renders (only when `prescription`): a muted context line `text-xs text-[var(--muted)]` showing the prescription text.
- Renders the **8-cell band**: `grid grid-cols-4 sm:grid-cols-8 divide-x divide-[var(--border)] overflow-hidden rounded-[var(--radius-card)] border border-[var(--border)]`; each cell `bg-[var(--surface)] px-3 py-2` with value (`text-sm font-semibold tabular-nums tracking-[-0.03em]`) over label (`text-[9px] uppercase tracking-wider text-[var(--faint)]`).

### `SplitsSpine.tsx`

Props:
```ts
{
  splits: Split[];
  intervals: IntervalSegment[] | null;
  unitLabel: "mi" | "km";
  formatDistance: (meters: number) => string; // from the page's context
  hasTargetColumn: boolean;          // whether to render a meaningful vs-target column
  targetPaceSecPerUnit: number | null;
}
```
- Local `Mode = "miles" | "intervals"`, `useState<Mode>("miles")`.
- Header row: `Splits` label (`text-[10px] font-semibold uppercase tracking-wider text-[var(--faint)]`) + the **segmented toggle** — rendered **only when `intervals` is non-null**. Toggle is the design-system pill: `rounded-full border border-[var(--border)] bg-[var(--surface)] p-0.5`; active segment `bg-[var(--accent)] text-[var(--accent-fg)]`, inactive `text-[var(--muted)] hover:text-[var(--foreground)]`. When `intervals` is null, force `miles` and render no toggle.
- **Miles table** (`Split · Dist · Time · Pace · Avg HR · Elev Δ`): hairline-bordered `rounded-[var(--radius-card)]`, `tabular-nums`, right-aligned numerics, left-aligned label. Pace cell colored `var(--discipline-run-fg)`. Fastest/slowest rows get a `success`/`danger` tag pill beside the pace (color-mix tint like the mockup's `Tag`). Use:
  - Split label: full → `Mi {index+1}` (or `Km {index+1}`); partial → `{formatDistance(distanceMeters)} {unitLabel}`.
  - Dist: `formatDistance(s.distanceMeters)`.
  - Time: `formatDuration(s.durationSec)` (from `lib/format`).
  - Pace: `fmtPaceUnit(s.avgPaceSecPerUnit)` → local `m:ss` formatter, `—` when null.
  - Avg HR: `s.avgHr ?? "—"`.
  - Elev Δ: `s.elevDeltaMeters == null ? "—"` else signed meters e.g. `+12 m` / `−4 m`.
- **Intervals table** (`Segment · Dist · Time · Pace · HR · vs target`), rendered when mode is `intervals`: rows from `intervals` filtered to `work`/`recovery` (warm-up/cool-down may be shown muted or omitted — show all four, with warmup/cooldown muted, to keep the session legible; work rows get a `--discipline-run-dot` dot + `--discipline-run-fg` pace, recovery/others calm-neutral `--muted`). `vs target`: when `hasTargetColumn && targetPaceSecPerUnit != null` and the row is `work`, show signed delta `avgPaceSecPerUnit - targetPaceSecPerUnit` via a `fmtPaceDelta` (`+m:ss` slower danger / `−m:ss` faster success); otherwise `—`.
- Provide local `fmtPaceUnit` and `fmtPaceDelta` helpers (mirror the mockup's `fmtPace`/`fmtPaceDelta`).

### `PaceStrip.tsx`

Props:
```ts
{ points: PaceStripPoint[]; hasDropout: boolean; }
```
- Hand-rolled SVG sparkline, `faster ↑` (invert: lowest pace → top). Build the path **breaking at null** points (start a new `M` after a gap, like `connectNulls={false}`). Stroke `var(--discipline-run-dot)`, `vectorEffect="non-scaling-stroke"`. Card: `rounded-[var(--radius-card)] border border-[var(--border)] bg-[var(--surface)] px-4 py-3`. Caption row: left `Pace trace · faster ↑` (`text-[10px] uppercase tracking-wider text-[var(--faint)]`), right `dropout bridged` shown **only when `hasDropout`**.
- Compute y-scale from non-null paces (min→top). If `< 2` non-null points, render a muted `No pace data` placeholder instead of an empty SVG.

- [ ] **Step 1:** Implement the three components.
- [ ] **Step 2:** `npm run typecheck && npm run lint` clean.
- [ ] **Step 3:** Commit: `feat(running): add ledger header band, splits spine, pace strip components`.

> Component-level rendering is verified through the page test in Task 4 (these are leaf components; a dedicated test per leaf is optional but a small `SplitsSpine` test for the toggle-gating is welcome).

---

## Task 3: Rewire `page.tsx`

**Files:**
- Modify: `app/(app)/running/[id]/page.tsx`

Keep everything above the render body unchanged: imports for auth/api/context, all state, `handleAuthError`, the load `useEffect`, `handleRename`, `handleDelete`, `notFound`/`error`/`loading` returns, the `<header>` chrome (back-link, `EditableName`, date, Delete button), `EditableName`, `PencilIcon`, `DeleteConfirmModal`, `CenteredMessage`.

Changes:
- [ ] **Step 1:** Remove the old chart imports (`HeartRateChart`/`PaceChart`/`ElevationChart`) and the `StatTile` import. Add:
  ```ts
  import { unlinkPlannedWorkout } from "@/lib/api"; // add to existing api import
  import { deriveRunningActivity, parseTargetPace } from "@/lib/running-splits";
  import { RunHeaderBand } from "./_components/RunHeaderBand";
  import { SplitsSpine } from "./_components/SplitsSpine";
  import { PaceStrip } from "./_components/PaceStrip";
  ```
  Keep `CompletesPlanBanner`? No — the ✓ pill + Unlink now live in `RunHeaderBand`. Remove the `CompletesPlanBanner` import and usage. (Do not modify the shared `CompletesPlanBanner` component file — other pages use it.)
- [ ] **Step 2:** Replace the `{ hrData, paceData, elevData } = useMemo(...)` block with:
  ```ts
  const derivation = useMemo(
    () => deriveRunningActivity(session?.trackpoints ?? [], completesPlan?.run_type ?? null, unit),
    [session, completesPlan, unit],
  );
  const targetPace = useMemo(
    () => parseTargetPace(completesPlan?.run_details ?? null, unit),
    [completesPlan, unit],
  );
  ```
- [ ] **Step 3:** Add an inline `handleUnlink` mirroring `CompletesPlanBanner`'s logic (uses `getToken`, `unlinkPlannedWorkout`, `handleAuthError`, `toast`, and a `unlinking` state). On success `setCompletesPlan(null)`; on failure toast the error.
  ```ts
  const [unlinking, setUnlinking] = useState(false);
  async function handleUnlink() {
    if (!completesPlan) return;
    const token = getToken();
    if (!token) { router.replace("/login"); return; }
    setUnlinking(true);
    try {
      await unlinkPlannedWorkout(token, completesPlan.id);
      setCompletesPlan(null);
    } catch (err) {
      if (handleAuthError(err)) return;
      toast.error(err instanceof Error ? err.message : "Unlink failed");
    } finally {
      setUnlinking(false);
    }
  }
  ```
- [ ] **Step 4:** Replace the body `<div className="mx-auto … max-w-4xl …">` contents (the `CompletesPlanBanner` + StatTile grid + charts) with the new composition. Build the 8 header cells from the **session** summary fields (not the derivation):
  ```ts
  const cells = [
    { label: "Distance", value: `${formatDistance(session.distance_meters)} ${unitLabel}` },
    { label: "Time", value: formatDuration(session.duration_seconds) },
    { label: "Avg pace", value: `${formatPace(session.avg_pace_sec_per_km)} /${unitLabel}` },
    { label: "Best", value: session.best_pace_sec_per_km != null ? `${formatPace(session.best_pace_sec_per_km)} /${unitLabel}` : "—" },
    { label: "Avg HR", value: session.avg_heart_rate_bpm != null ? `${session.avg_heart_rate_bpm}` : "—" },
    { label: "Max HR", value: session.max_heart_rate_bpm != null ? `${session.max_heart_rate_bpm}` : "—" },
    { label: "Calories", value: session.total_calories != null ? String(session.total_calories) : "—" },
    { label: "Elev", value: session.elevation_gain_meters != null ? `${session.elevation_gain_meters.toFixed(0)} m` : "—" },
  ];
  ```
  Then render (use a narrower `max-w-3xl` to match the ledger idiom):
  ```tsx
  <RunHeaderBand
    cells={cells}
    planName={completesPlan?.name ?? (completesPlan ? "Planned run" : null)}
    prescription={completesPlan?.run_details ?? null}
    onUnlink={completesPlan ? handleUnlink : undefined}
    unlinking={unlinking}
  />
  <SplitsSpine
    splits={derivation.splits}
    intervals={derivation.intervals}
    unitLabel={unitLabel}
    formatDistance={formatDistance}
    hasTargetColumn={targetPace != null}
    targetPaceSecPerUnit={targetPace}
  />
  <PaceStrip points={derivation.paceStrip} hasDropout={derivation.hasDropout} />
  ```
- [ ] **Step 5:** Update the component's top JSDoc to describe the ledger composition.
- [ ] **Step 6:** `npm run typecheck && npm run lint && npm run format:check && npm run build` clean. Commit: `feat(running): rebuild run detail body as splits ledger`.

---

## Task 4: Component test `page.test.tsx`

**Files:**
- Create: `app/(app)/running/[id]/page.test.tsx`

Mirror the mocking pattern in `app/(app)/planned-workouts/[id]/page.test.tsx` and the existing running mocks. Mock `next/navigation`, `@/lib/auth`, and `@/lib/api` (spread the real module, override `getRunningSession`, `getPlannedWorkoutBySession`, `renameRunningSession`, `deleteRunningSession`, `unlinkPlannedWorkout`). Wrap render in `DistanceUnitProvider` (or mock `useDistanceUnit`) and the `ToastProvider` if the page reads it — match how other running tests handle these contexts (check `app/(app)/running/_components` tests or the nutrition/personal-records page tests for the toast/provider wiring).

- [ ] **Step 1: Renders splits from a known fixture.** A `RunningSession` with a synthesized interval trackpoint stream linked to an `intervals` `PlannedWorkout`. Assert the splits table renders (e.g. a `Mi 1` row appears) and the header band shows the distance.
- [ ] **Step 2: Toggle appears only when intervals detected.** With the interval fixture → the `intervals` toggle button is present. With an `easy` run fixture (or a steady trace) → no `intervals` toggle (miles-only).
- [ ] **Step 3: Rename still fires.** Click the name, type, blur/Enter → `renameRunningSession` called with the new name (optimistic update visible).
- [ ] **Step 4: Delete still fires.** Click Delete → confirm modal → confirm → `deleteRunningSession` called and `router.push("/activities?view=running")`.
- [ ] **Step 5: Unlink still fires.** With a linked plan, click Unlink → `unlinkPlannedWorkout` called and the ✓ pill disappears.
- [ ] **Step 6:** `npm run test` → all green. Commit: `test(running): cover ledger render, toggle gating, rename/delete/unlink`.

---

## Task 5: Full local gate + final review

- [ ] **Step 1:** Run the full CI gate locally from repo root:
  ```bash
  npm run lint && npm run format:check && npm run typecheck && npm run test && npm run build
  ```
  All must pass. Fix any failure in the code (never silence a rule or skip a test).
- [ ] **Step 2:** Self-check against the SOW's Verification list: interval run reads as a ledger; mile-2 dropout no longer owns the axis (excluded from stats + gapped strip); degrade paths (unlinked/easy → miles-only no toggle/pill; no-elevation → Δ dashes; sparse/missing-HR render cleanly); behavior intact (back-link, rename, delete, unlink, mi/km); design-system unchanged; no DX mockup code shipped.
- [ ] **Step 3:** Final code review of the whole branch diff.

---

## Self-Review (author checklist — done at plan-writing time)

1. **Spec coverage:** Derivation module (splits + partial, winsorization, intervals, units, target parse) → Task 1. Header band, splits spine, segmented toggle, pace strip → Tasks 2–3. Prescription context + vs-target → Tasks 1 (`parseTargetPace`) + 2–3. Preserve chrome/behavior → Task 3 (kept intact) + Task 4 (tested). Hard states → Task 1 tests (dropout, no-HR, no-elev, sparse, non-interval) + Task 2/3 degrade rendering. Tests + CI → Tasks 1, 4, 5. Conforms to v0.4, no API change → constraints honored throughout.
2. **Placeholder scan:** none — all interfaces, constants, and test assertions are concrete.
3. **Type consistency:** `Split`, `IntervalSegment`, `PaceStripPoint`, `RunningDerivation`, `deriveRunningActivity`, `parseTargetPace`, `PACE_DROPOUT_SEC_PER_KM` used identically across tasks. Pace fields are always `…SecPerUnit`; distances `…Meters` except `PaceStripPoint.distanceUnit`.
