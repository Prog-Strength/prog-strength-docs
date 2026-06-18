# Activities Page Redesign & System Re-tone (Oura-Calm-Minimal) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Re-tone the Prog Strength web token layer to the oura-calm-minimal visual language app-wide (values-only, names stable) and rebuild `/activities` (all four tabs) as the first fully-realized surface in that language — calm hairline panels, Manrope numerals, and delicate single-stroke line charts — over the existing data layer with no backend change.

**Architecture:** Two repos. In `prog-strength-docs`, promote the language into `design-system.md` (v0.3) and flip the DX to `selected`. In `prog-strength-web`, the change is driven through the existing semantic CSS tokens in `app/globals.css` (re-point values, keep names) plus a Manrope swap in `app/layout.tsx`; then restyle the activities composition (page shell, stat tiles, charts, lists) to consume those tokens. Charts hardcode hex (recharts SVG attributes can't resolve `var()`), so chart colors are re-pointed to oura hex mirroring the tokens.

**Tech Stack:** Next.js 16 App Router, React 19, TypeScript, Tailwind v4 (`@theme inline`), recharts 3, Vitest + Testing Library (jsdom).

---

## Context every implementer needs

- **Token names never change.** Only the *values* in `app/globals.css` `:root` move to oura. Consumers reference `var(--accent)`, `var(--surface)`, etc., so the app inherits the re-tone for free. The only structural add is a new `--accent-2` (sage, running) family + its `@theme inline` mapping lines.
- **The oura palette (from the SOW, authoritative):**
  - Neutrals: `--background #0e0f12`, `--surface #15171b`, `--surface-2 #191c21`, `--surface-3 #1e2127` (a hair above surface-2), text `--foreground #c8cad0`, `--muted #7d818c`, `--faint #565a63`; hairlines `--border rgba(255,255,255,0.06)`, `--border-strong rgba(255,255,255,0.10)`.
  - Accent (periwinkle, lift + primary): `--accent #9aa6d6`, `--accent-dark #8490c4`, `--accent-fg #0e0f12` (dark text on the light periwinkle so pill labels stay legible), `--accent-soft rgba(154,166,214,0.14)`, `--accent-line rgba(154,166,214,0.30)`.
  - Secondary accent (sage, run): `--accent-2 #7fae9e`, `--accent-2-soft rgba(127,174,158,0.14)`, `--accent-2-line rgba(127,174,158,0.30)`.
  - Status: `--success #86b39f`, `--danger #c79292`, `--warning #d6b87f` (amber re-toned to sit calmly).
  - Discipline hues fold into the accents: `--discipline-lift-*` → periwinkle, `--discipline-run-*` → sage. Keep mobility/core reserved (re-tone lightly so they still read on the darker field).
  - Macro tints (`--macro-*`) are **out of scope** — leave them exactly as-is.
  - Radii: `--radius-card: 0.875rem` (14px).
- **Type:** Manrope is the primary family (replaces Nunito). Drop the Oswald display accent (Open Question 2 lean). Keep `Geist_Mono`. Numerals use tight tracking (`-0.03em`) — the calm tiles and big numbers set `tracking-[-0.03em]`.
- **`/workouts` and `/running` routes only `redirect()` into `/activities` tabs** — so the analytics + trend charts (`workouts-analytics`, `running-analytics`, `workout-duration-chart`, `workout-volume-chart`, `running-time-chart`, `running-mileage-chart`) are activities-only. Re-toning them is in scope and does not blast other surfaces.
- **Charts in jsdom:** recharts renders nothing under a zero-width `ResponsiveContainer`. Tests that need to assert on chart series mock `recharts` to passthrough/stub divs (see `components/activities/steps-view.test.tsx` for the canonical mock). Prefer testing pure aggregation functions + component behavior over SVG geometry.
- **Behavior is preserved everywhere.** This is a presentation rebuild over existing data orchestration. Do not change fetch logic, grouping, unit conversion, routing, modals, optimistic updates, or toasts.
- **Verify gate (run before claiming done):** `npm run lint` (0 errors; do not add new warnings), `npm run format:check`, `npm run typecheck`, `npm run test`, and `npm run build`. Conventional Commits enforced by Husky; never `--no-verify`.

## File Structure

`prog-strength-web`:
- `app/globals.css` — token values + `@theme inline` (+`--accent-2` family, font vars).
- `app/layout.tsx` — Manrope/Geist_Mono fonts, drop Oswald.
- `components/stat-tile.tsx` — calm reusable tile.
- `app/(app)/activities/page.tsx` — period-filter pills + tab bar quiet treatment, 14px hairline panels.
- `components/view-button.tsx` — quiet active/inactive treatment (used by analytics switchers).
- `components/activities/activities-combined-chart.tsx` — dual single-stroke line chart (+ test).
- `components/workout-duration-chart.tsx`, `components/workout-volume-chart.tsx`, `components/running-time-chart.tsx`, `components/running-mileage-chart.tsx` — Area→single-stroke Line, re-toned.
- `components/activities/steps-view.tsx` — bar chart re-tone (over/under goal), table/badges (+ test).
- `components/activities/workouts-view.tsx` — session-card PR trophy badge re-tone.
- Tests co-located `*.test.tsx`.

`prog-strength-docs`:
- `design-system.md` — v0.3 re-tone.
- `dx/activities-page.md` — status → selected.
- `sows/activities-page-redesign.md` — status → shipped (done in the final ship step, not a task here).

---

### Task 1: Token re-tone (`app/globals.css`)

**Files:**
- Modify: `app/globals.css` (`:root` block lines ~6-64, `@theme inline` block lines ~66-105, `html, body` line ~107-112)

- [ ] **Step 1: Re-point the `:root` token values** to the oura palette listed in "Context" — neutrals, accent (periwinkle), status, discipline hues. Add new `--accent-2`, `--accent-2-soft`, `--accent-2-line` (sage). Set `--radius-card: 0.875rem`. Leave `--macro-*` untouched. Keep every existing token **name**. Update the leading comment to reference the oura-calm-minimal re-tone and `sows/activities-page-redesign.md`.

Exact accent block to land:
```css
  /* Accent — desaturated periwinkle (oura-calm-minimal; replaces violet).
     Primary actions + active states + the lifting domain hue. Solid hex so
     `/NN` opacity modifiers on existing consumers keep working. */
  --accent: #9aa6d6;
  --accent-dark: #8490c4;
  --accent-fg: #0e0f12;
  --accent-soft: rgba(154, 166, 214, 0.14);
  --accent-line: rgba(154, 166, 214, 0.3);

  /* Secondary domain accent — sage (running). Peer to the periwinkle
     lifting hue, not a louder "primary". */
  --accent-2: #7fae9e;
  --accent-2-soft: rgba(127, 174, 158, 0.14);
  --accent-2-line: rgba(127, 174, 158, 0.3);
```

Discipline hues (fold lift→periwinkle, run→sage; keep mobility/core reserved, re-toned for the darker field):
```css
  --discipline-run-bg: #16241f;
  --discipline-run-fg: #9cc7b8;
  --discipline-run-dot: #7fae9e;
  --discipline-lift-bg: #1b1f2e;
  --discipline-lift-fg: #aab4dd;
  --discipline-lift-dot: #9aa6d6;
  --discipline-mobility-bg: #182622;
  --discipline-mobility-fg: #8fc4b6;
  --discipline-mobility-dot: #6fae9b;
  --discipline-core-bg: #211d2e;
  --discipline-core-fg: #b4a9d6;
  --discipline-core-dot: #8f82c0;
```

- [ ] **Step 2: Add `@theme inline` mappings** for the three new `--accent-2*` tokens (mirroring the `--accent*` lines) so Tailwind utilities like `text-accent-2`/`bg-accent-2` exist:
```css
  --color-accent-2: var(--accent-2);
  --color-accent-2-soft: var(--accent-2-soft);
  --color-accent-2-line: var(--accent-2-line);
```
Leave the rest of `@theme inline` structurally unchanged (font vars handled in Task 2).

- [ ] **Step 3: Verify** `npm run build` succeeds and `grep -n "var(--accent-2" app/globals.css` shows the mapping. There is no unit test for raw CSS; the build is the gate. Confirm no token *name* was removed (diff review).

- [ ] **Step 4: Commit** — `style(tokens): re-tone web token layer to oura-calm-minimal`

---

### Task 2: Manrope font swap (`app/layout.tsx` + `app/globals.css`)

**Files:**
- Modify: `app/layout.tsx` (imports lines 1-22, `className` line 39)
- Modify: `app/globals.css` (`@theme inline` font vars lines ~102-104, `html, body` font-family line ~111)

- [ ] **Step 1: Swap fonts in `app/layout.tsx`.** Replace the `Nunito`/`Oswald` imports with `Manrope`; keep `Geist_Mono`. Manrope supports a weight range, so:
```tsx
import { Manrope, Geist_Mono } from "next/font/google";

const manrope = Manrope({
  variable: "--font-manrope",
  subsets: ["latin"],
});

const geistMono = Geist_Mono({
  variable: "--font-geist-mono",
  subsets: ["latin"],
});
```
Update the `<html className=...>` to `` `${manrope.variable} ${geistMono.variable} h-full antialiased dark` `` (drop the oswald variable).

- [ ] **Step 2: Re-point font vars in `app/globals.css`.** In `@theme inline`, set `--font-sans: var(--font-manrope);` and **remove** the `--font-display` line (Oswald is dropped). Keep `--font-mono: var(--font-geist-mono);`. In `html, body`, change the font-family to `var(--font-manrope), ui-sans-serif, system-ui, sans-serif;`.

- [ ] **Step 3: Confirm no dangling Oswald/Nunito references** — `grep -rn "font-oswald\|font-nunito\|Oswald\|Nunito\|font-display" app components lib` returns nothing (or only this plan). If any component used `font-display`, replace it with the default sans (Manrope) — there should be none after the timeline check; verify.

- [ ] **Step 4: Verify** `npm run build` and `npm run typecheck` pass.

- [ ] **Step 5: Commit** — `style(type): swap base family to Manrope, drop Oswald display accent`

---

### Task 3: `design-system.md` v0.3 + DX flip (`prog-strength-docs`)

**Files:**
- Modify: `design-system.md` (Status line, Color/Typography/Form-&-depth sections, Provenance, Changelog)
- Modify: `dx/activities-page.md` (frontmatter `status:` + body `**Status**:` line)

- [ ] **Step 1: Bump the header** — `**Status**: v0.3 · **Last updated**: 2026-06-18`.

- [ ] **Step 2: Rewrite the Color section** to the oura ramp: soft near-black neutrals (`#0e0f12` base → `#15171b`/`#191c21`/`#1e2127` surfaces; text `#c8cad0`/`#7d818c`/`#565a63`; hairlines `rgba(255,255,255,0.06)`–`0.10`); periwinkle `#9aa6d6` as the primary accent (replacing violet); document the **dual domain accents** (periwinkle = lift, sage `#7fae9e` = run) as the canonical activity hues; re-toned status (`--success #86b39f`, `--danger #c79292`, `--warning #d6b87f`). Note the macro tints are unchanged pending nutrition's migration.

- [ ] **Step 3: Rewrite the Typography section** — Manrope is the primary family (replacing Nunito); state the tight numeric tracking convention (`-0.03em` on numerals); record that the Oswald display accent is **dropped** (the calm idiom is uniform-Manrope), `Geist_Mono` kept for incidental mono. Remove the "athletic condensed display generalizes" Open item now that it's resolved.

- [ ] **Step 4: Rewrite Form & depth** — calmer 14px panel radius + hairline borders as the default; keep the pill chips/buttons and the form-control language, re-toned. (Form controls section's structure holds — just note the re-tone.)

- [ ] **Step 5: Provenance + Changelog.** Add a provenance bullet pointing at `sows/activities-page-redesign.md` and `dx/activities-page.md` (`oura-calm-minimal`). Add a `**v0.3** (2026-06-18)` changelog entry summarizing the oura re-tone (soft near-black ramp, periwinkle+sage dual accents, Manrope, dropped Oswald, 14px hairline panels). Keep the "Fixed Points hold" framing intact (P mark + dark theme unchanged).

- [ ] **Step 6: Flip the DX** — in `dx/activities-page.md`, set frontmatter `status: selected` and the body `**Status**:` line to `Selected` with the chosen idiom `oura-calm-minimal` noted (match the existing body phrasing).

- [ ] **Step 7: Commit** — `docs(design-system): re-tone to oura-calm-minimal (v0.3) and select activities DX`

---

### Task 4: Calm StatTile + quiet activities chrome

**Files:**
- Modify: `components/stat-tile.tsx`
- Modify: `app/(app)/activities/page.tsx` (period-filter pills ~109-122, tab-bar panel container ~133-176)
- Modify: `components/view-button.tsx`
- Test: `components/stat-tile.test.tsx` (create)

- [ ] **Step 1: Calm StatTile.** Keep the `{value, label, sub, tone}` API. Hairline panel: `rounded-[var(--radius-card)] border border-[var(--border)] bg-[var(--surface)] px-4 py-3`. Value: `text-2xl font-semibold tabular-nums tracking-[-0.03em]`. Label: `text-[11px] font-semibold uppercase tracking-wider text-[var(--faint)]`. Replace the hardcoded positive `text-[#86efac]` with `text-[var(--success)]`; negative stays `text-[var(--danger)]`; neutral `text-[var(--foreground)]`. `sub` stays `text-xs tabular-nums text-[var(--muted)]`.

- [ ] **Step 2: Test StatTile** — render with `tone="positive"`/`"negative"`/`"neutral"` and assert the value text shows and (for positive) carries the success color class; assert label renders uppercased text. Keep it a light render test.

- [ ] **Step 3: Quiet period-filter pills** in `page.tsx`: active = `bg-[var(--accent)] text-[var(--accent-fg)]`; inactive = `border border-[var(--border)] bg-[var(--surface)] text-[var(--faint)] hover:text-[var(--foreground)]`. Use `rounded-full`. (Faint inactive, soft periwinkle active — the calm treatment.)

- [ ] **Step 4: Tab-bar / panel containers** in `page.tsx`: wrap content panels with the 14px hairline treatment where cards exist; the toolbar row keeps the `ToolbarButton` active underline but switch the active underline color to `border-[var(--accent)]` (soft periwinkle active) and inactive labels to `text-[var(--faint)]` brightening to `--foreground`. Header `<h1>` stays.

- [ ] **Step 5: ViewButton quiet treatment** — active = `bg-[var(--accent-soft)] text-[var(--accent)] border border-[var(--accent-line)]`; inactive = `border border-[var(--border)] bg-[var(--surface)] text-[var(--faint)] hover:text-[var(--foreground)]`; `rounded-md`. (Calmer than the solid-accent fill.)

- [ ] **Step 6: Verify** `npm run typecheck && npm run lint && npm run test` pass; the existing overview/steps tests still pass (StatTile API unchanged).

- [ ] **Step 7: Commit** — `feat(activities): calm stat tiles and quiet period/tab chrome`

---

### Task 5: Overview dual single-stroke line chart

**Files:**
- Modify: `components/activities/activities-combined-chart.tsx`
- Test: `components/activities/activities-combined-chart.test.tsx` (create)

- [ ] **Step 1: Convert the chart from a stacked `AreaChart` to a `LineChart` with two 1px `Line`s.** Keep `buildCombined`, the props, the loading/empty states, and the truncated note exactly. Replace `Area`/`AreaChart`/`defs` gradients with:
  - `LineChart` from recharts.
  - `<Line dataKey="workout_minutes" name="Lifting" stroke="#9aa6d6" strokeWidth={1} dot={false} type="monotone" isAnimationActive={false} />` (periwinkle, mirrors `--accent`/`--discipline-lift`).
  - `<Line dataKey="running_minutes" name="Running" stroke="#7fae9e" strokeWidth={1} dot={false} type="monotone" isAnimationActive={false} />` (sage, mirrors `--accent-2`/`--discipline-run`).
  - Remove `stackId` (lines overlay, not stack — both must stay legible where they cross and where one is zero).
  - Hairline grid: `<CartesianGrid stroke="rgba(255,255,255,0.06)" strokeDasharray="3 3" vertical={false} />`.
  - Axes: `stroke="#565a63"`, `tick={{ fill: "#565a63", fontSize: 11 }}` (mirrors `--faint`).
  - Tooltip restyle: `contentStyle={{ backgroundColor: "#15171b", border: "1px solid rgba(255,255,255,0.10)", borderRadius: "0.875rem", padding: "6px 10px", fontSize: "12px" }}`, `cursor={{ stroke: "#565a63", strokeWidth: 1 }}`. Keep the `labelFormatter`/`formatter` so rows read "Lifting 1h 30m".
  - Keep `<Legend .../>` with the same alignment.
  Add a short comment that the hex mirrors the design-system tokens (recharts SVG attrs can't resolve `var()`).

- [ ] **Step 2: Test — dual series + rest-week zero.** Mock `recharts` (passthrough/stub like steps-view.test.tsx) so `Line` becomes a stub exposing its `name`/`dataKey` via a `data-*` attribute. Render `ActivitiesCombinedChart` with a fixture where one week has both lifting and running minutes and another (rest) week has both at zero. Assert both a Lifting and a Running line render. Since the data build is the load-bearing logic, also assert the chart isn't in the empty state when any series is non-zero, and IS empty when all weeks are zero across both series.

Stub example:
```tsx
vi.mock("recharts", () => {
  const pass = (n: string) => ({ children }: { children?: React.ReactNode }) =>
    <div data-recharts={n}>{children}</div>;
  return {
    ResponsiveContainer: ({ children }: { children: React.ReactNode }) => <div>{children}</div>,
    LineChart: pass("LineChart"),
    Line: ({ name, dataKey }: { name?: string; dataKey?: string }) =>
      <div data-recharts="Line" data-name={name} data-key={dataKey} />,
    CartesianGrid: () => <div data-recharts="CartesianGrid" />,
    XAxis: () => <div />, YAxis: () => <div />,
    Tooltip: () => <div />, Legend: () => <div />,
  };
});
```
Assert e.g. `screen.getByText`/`container.querySelector('[data-name="Lifting"]')` and `[data-name="Running"]` both present for the non-empty fixture.

- [ ] **Step 3: Verify** `npm run typecheck && npm run lint && npm run test && npm run build`.

- [ ] **Step 4: Commit** — `feat(activities): dual single-stroke overview line chart`

---

### Task 6: Workouts/Running trend charts → single-stroke lines

**Files:**
- Modify: `components/workout-duration-chart.tsx`
- Modify: `components/workout-volume-chart.tsx`
- Modify: `components/running-time-chart.tsx`
- Modify: `components/running-mileage-chart.tsx`

- [ ] **Step 1: For each of the four charts, convert `Area`→`Line` (1px), drop the gradient `defs`/fill, and re-tone.** Keep all aggregation, props, loading/empty/truncated logic identical. Per chart:
  - Lifting charts (`workout-duration-chart`, `workout-volume-chart`): `stroke="#9aa6d6"` (periwinkle), `strokeWidth={1}`, `dot={false}` (or a tiny `r:2` dot in the same hue), `activeDot={{ r: 3 }}`, `type="monotone"`, `isAnimationActive={false}`.
  - Running charts (`running-time-chart`, `running-mileage-chart`): `stroke="#7fae9e"` (sage), same line treatment.
  - Replace the shared chrome hex in all four: grid `stroke="rgba(255,255,255,0.06)"`; axis `stroke`/`tick.fill` `#565a63`; tooltip `contentStyle` bg `#15171b`, border `1px solid rgba(255,255,255,0.10)`, `borderRadius "0.875rem"`; cursor `stroke "#565a63"`.
  - Use `LineChart` instead of `AreaChart`; keep `XAxis`/`YAxis`/`Tooltip` props and formatters.
  - Add the same one-line "hex mirrors tokens" comment.

- [ ] **Step 2: Verify** `npm run typecheck && npm run lint && npm run test && npm run build`. (These charts have no dedicated unit tests; the build + typecheck are the gate, and the analytics wrappers render them.)

- [ ] **Step 3: Commit** — `feat(activities): re-render workout & running trend charts as single-stroke lines`

---

### Task 7: Steps bar chart re-tone (over/under goal) + badge re-tone

**Files:**
- Modify: `components/activities/steps-view.tsx` (`BAR_COLOR`, `ChartCard`, `TargetIcon`)
- Modify: `components/activities/steps-view.test.tsx` (extend)

- [ ] **Step 1: Over/under-goal bars.** In `ChartCard`, color each bar by whether it meets the goal: when a goal is set, bars `>= goal` use `--success` (`#86b39f`), bars `< goal` use the faint neutral (`#565a63` / a muted periwinkle `#9aa6d6` at low emphasis — pick the muted periwinkle `#9aa6d6` for met-context but the SOW wants over vs under to "read differently", so use success `#86b39f` for met and a desaturated `#5b6168` for under). Implement via recharts `<Cell>` per data point (map `data` to `<Bar><Cell fill=.../></Bar>`), or a `fill` function. When no goal is set, all bars use the calm periwinkle `#9aa6d6`. Keep `radius={[2,2,0,0]}`, `isAnimationActive={false}`.
  - Goal `ReferenceLine`: keep dashed, re-tone `stroke="#86b39f"` (success) and label `fill="#86b39f"`, `strokeDasharray="6 4"`.
  - Re-tone grid/axis/tooltip hex to the oura values (grid `rgba(255,255,255,0.06)`, axis `#565a63`, tooltip bg `#15171b` + `rgba(255,255,255,0.10)` border + `0.875rem` radius).
  - Replace `const BAR_COLOR = "#3b82f6"` with the new logic/constants.

- [ ] **Step 2: Re-tone `TargetIcon`** (`text-emerald-500` → `text-[var(--success)]`) and the steps panels to the 14px hairline radius where they use `rounded-lg` for the chart card and table container (`rounded-[var(--radius-card)]`).

- [ ] **Step 3: Test — over vs under bars.** Extend `steps-view.test.tsx`. Using the existing recharts mock, add `Cell` to the mock as a stub that exposes its `fill` via `data-fill`. Provide a fixture with one day above goal and one below (goal 10000; days 12000 and 8000). Assert two `Cell`s render with **distinct** `data-fill` values (one success, one under-tone). Keep the existing goal-reference-line and tiles tests passing.

- [ ] **Step 4: Verify** `npm run typecheck && npm run lint && npm run test && npm run build`.

- [ ] **Step 5: Commit** — `feat(activities): re-tone steps bar chart with over/under-goal bars`

---

### Task 8: PR trophy badge re-tone + Running delta test

**Files:**
- Modify: `components/activities/workouts-view.tsx` (`TrophyIcon`)
- Test: `components/activities/running-view.test.tsx` (create)
- Test: `components/activities/workouts-view.test.tsx` (create)

- [ ] **Step 1: Re-tone the PR trophy badge.** In `workouts-view.tsx` `TrophyIcon`, replace `bg-amber-500/15 text-amber-200` with the re-toned warning token: `bg-[var(--warning)]/15 text-[var(--warning)]` (calm amber on the new field). Keep the count + title behavior.

- [ ] **Step 2: Test the Running delta (up/down/hidden).** Create `running-view.test.tsx` mocking `next/navigation`, `@/lib/auth`, `@/lib/api` (`getRunningMetrics`, `listRunningSessions`), `@/lib/distance-unit-context` (return `formatDistance`/`formatPace`/`unitLabel`), `@/components/toast`, and the heavy children (`RunningAnalytics`, `RunHistoryList`, `UploadTCXModal`) as stub divs. Three cases via `getRunningMetrics` `current_week.delta_pct_vs_prior_week`:
  - positive (e.g. `12`) → renders "+12% vs last week" and the value tile carries the positive (`--success`) tone class.
  - negative (e.g. `-8`) → renders "-8% vs last week" with the negative (`--danger`) tone class.
  - null → no "vs last week" sub text rendered.
  Assert on the rendered sub-text and tone (the tone maps to StatTile's color class; assert the sub text presence/absence and the +/- sign).

- [ ] **Step 3: Test workout card PR badge + long-note truncation.** Create `workouts-view.test.tsx` mocking the API (`listWorkouts`, `listExercises`, `deleteWorkout`) + auth + router + the `WorkoutModal`/`WorkoutDetailsBody`/`WorkoutsAnalytics` children. Render a workout with `personal_records_set` non-empty → assert the trophy badge title text ("set a new personal record") appears. Render a workout with a very long `notes` string → assert the note element has the `truncate` class. (Mirror the existing overview/steps test mock style.)

- [ ] **Step 4: Verify** `npm run typecheck && npm run lint && npm run test && npm run build`.

- [ ] **Step 5: Commit** — `feat(activities): re-tone PR badge and cover running delta + workout badges with tests`

---

## Self-Review notes

- **Spec coverage:** Token re-tone (T1), Manrope/drop-Oswald (T2), design-system v0.3 + DX flip (T3), calm tiles + quiet chrome (T4), dual single-stroke overview chart (T5), trend charts as lines (T6), steps over/under bars + dashed goal line (T7), PR badge + delta/badge tests (T8). Behavior preserved throughout (no data-layer edits). Macro tints, mobility/core reservation, fixed points (P mark + dark theme), and the gated `design-explore` route are all untouched per Non-Goals.
- **Hard states:** rest-week zero (T5 test), dual series legible/crossing/zero (T5), over vs under goal (T7 test), delta positive/negative/null (T8 test), PR badge + long-note truncation (T8 test), sparse vs dense windows (existing `buildWeeklyBuckets` logic, unchanged).
- **Token names stable:** only values change; the single structural add is `--accent-2*` + its mappings.
- **Chart color note:** hex is re-pointed (not `var()`) because recharts writes SVG presentation attributes; the values mirror the tokens and are commented as such.
