# Recovery Page & Sidebar Tab Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Give Whoop recovery data a first-class `/recovery` page (score ring + trend charts + day log) reachable from a new sidebar tab, fed entirely by the existing `GET /whoop/recovery` read endpoint.

**Architecture:** A new client page `app/(app)/recovery/page.tsx` in the authed shell fetches the Whoop connection + a range of recovery day rows in one `Promise.all`, then renders a render-gate (empty/error states → hero → trends → log). All three page sections derive from the same fetched rows client-side (the steps-view "hero → chart → log" grammar). A new `lib/api.ts` wrapper (`listWhoopRecovery`) follows the house timezone+local-date query convention; zero API-side changes. Band thresholds live in a pure `lib/recovery.ts` helper so they are unit-testable.

**Tech Stack:** Next.js 16 App Router, React 19, TypeScript, Tailwind v4 (CSS-variable tokens), recharts, Vitest + Testing Library.

---

## File Structure

**Create:**
- `app/(app)/recovery/page.tsx` — the client page: fetch, render-gate, range-toggle state, section composition.
- `app/(app)/recovery/page.test.tsx` — render-gate + wiring tests.
- `app/(app)/recovery/_components/recovery-hero.tsx` — score ring + resting-HR / HRV MiniStats.
- `app/(app)/recovery/_components/recovery-hero.test.tsx`
- `app/(app)/recovery/_components/recovery-trends.tsx` — the three recharts trend panels.
- `app/(app)/recovery/_components/recovery-trends.test.tsx`
- `app/(app)/recovery/_components/recovery-log.tsx` — reverse-chronological day log.
- `app/(app)/recovery/_components/recovery-log.test.tsx`
- `lib/recovery.ts` — pure band-mapping + chart-data helpers.
- `lib/recovery.test.ts`
- `components/segmented-toggle.tsx` — SegmentedToggle promoted out of settings `_components` (second consumer = this page; matches the "promote on second use" convention).
- `components/segmented-toggle.test.tsx`

**Modify:**
- `lib/api.ts` — add `WhoopRecoveryDay` type + `listWhoopRecovery` function.
- `lib/api.test.ts` — add `listWhoopRecovery` tests.
- `lib/chart-colors.ts` — add `CHART_RECOVERY_*` line + band-fill constants.
- `components/sidebar.tsx` — add the `Recovery` NAV entry + `RecoveryIcon`.
- `components/sidebar.test.tsx` — assert the Recovery item renders + goes active on `/recovery`.
- `app/(app)/dashboard/page.tsx` — flip `DEEP_LINKS.recovery` to `/recovery`.
- `app/(app)/dashboard/page.test.tsx` — assert recovery card deep-links to `/recovery`.
- `app/(app)/settings/_components/primitives.tsx` — remove `SegmentedToggle` (moved to `components/`).
- `app/(app)/settings/_components/primitives.test.tsx` — remove the moved `SegmentedToggle` describe block.
- `app/(app)/settings/page.tsx` — import `SegmentedToggle` from `@/components/segmented-toggle`.

**Conventions all tasks follow (read before starting any task):**
- All API calls go through `lib/api.ts` wrappers; never ad-hoc `fetch` in a component.
- Theme colors are CSS variables (`var(--success)`, `var(--muted)`, …). recharts SVG attrs cannot resolve `var()`, so charts use the hex constants from `lib/chart-colors.ts`.
- Tests co-locate as `*.test.ts(x)`. Component tests mock `@/lib/api`, `@/lib/auth`, and `next/navigation`; recharts is mocked to plain divs (see the steps-view and dashboard test idioms).
- Conventional Commits are enforced by a Husky `commit-msg` hook; commit each task with a `feat(recovery): …` / `refactor: …` subject. **Never** use `--no-verify`.
- Band thresholds (the one piece of real logic): **success ≥ 67, warning 34–66, danger ≤ 33.**

---

### Task 1: API client — `WhoopRecoveryDay` + `listWhoopRecovery`

**Files:**
- Modify: `lib/api.ts` (add after `disconnectWhoop`, near line 3140)
- Test: `lib/api.test.ts`

The API contract (from `sows/whoop-integration.md` line 128): `GET /whoop/recovery?timezone=&since=&until=` — a local-date window per the house convention (`internal/daterange`), returning day rows ordered by date. No connection simply yields an empty list. Day-row fields mirror the dashboard's `today` snapshot: `date`, `recovery_score`, `resting_heart_rate`, `hrv_rmssd_milli` (all three metrics nullable). The envelope is `{ data: [...] }`, unwrapped by the existing `unwrap()` helper with `[]` as the empty default.

- [ ] **Step 1: Write the failing tests**

Add to `lib/api.test.ts`. Ensure `listWhoopRecovery` is added to the existing `import { … } from "./api"` block at the top of the file (find the existing import and add the name; do not add a second import statement).

```typescript
describe("listWhoopRecovery", () => {
  const rows = [
    { date: "2026-07-01", recovery_score: 72, resting_heart_rate: 54, hrv_rmssd_milli: 88.4 },
    { date: "2026-07-02", recovery_score: null, resting_heart_rate: null, hrv_rmssd_milli: null },
  ];

  it("sends timezone + local since/until dates and returns the rows", async () => {
    const fetchMock = mockFetchOk(rows);

    const result = await listWhoopRecovery(TOKEN, {
      timezone: "America/Denver",
      since: "2026-06-03",
      until: "2026-07-02",
    });

    expect(result).toEqual(rows);
    expect(fetchMock).toHaveBeenCalledWith(
      `${BASE}/whoop/recovery?timezone=America%2FDenver&since=2026-06-03&until=2026-07-02`,
      { headers: { Authorization: `Bearer ${TOKEN}` } },
    );
  });

  it("omits since/until when not provided", async () => {
    const fetchMock = mockFetchOk(rows);

    await listWhoopRecovery(TOKEN, { timezone: "America/Denver" });

    expect(fetchMock).toHaveBeenCalledWith(`${BASE}/whoop/recovery?timezone=America%2FDenver`, {
      headers: { Authorization: `Bearer ${TOKEN}` },
    });
  });

  it("returns an empty list when the envelope data is missing", async () => {
    mockFetchOk(undefined);
    expect(await listWhoopRecovery(TOKEN, { timezone: "America/Denver" })).toEqual([]);
  });

  it("rejects with the API error text on a non-ok response", async () => {
    mockFetchError("boom");
    await expect(listWhoopRecovery(TOKEN, { timezone: "America/Denver" })).rejects.toThrow("boom");
  });
});
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `npx vitest run lib/api.test.ts -t listWhoopRecovery`
Expected: FAIL — `listWhoopRecovery is not a function` / not exported.

- [ ] **Step 3: Implement the type + function**

Add to `lib/api.ts` immediately after `disconnectWhoop` (around line 3140), keeping it in the Whoop section:

```typescript
/**
 * A single day's Whoop recovery row. `date` is the user-local calendar day
 * (YYYY-MM-DD) the reading belongs to; all three metrics are nullable because
 * Whoop may have no scored recovery for a day (PENDING/UNSCORABLE, or a night
 * with no sleep). `hrv_rmssd_milli` is HRV in milliseconds.
 */
export type WhoopRecoveryDay = {
  date: string; // YYYY-MM-DD
  recovery_score: number | null;
  resting_heart_rate: number | null;
  hrv_rmssd_milli: number | null;
};

/**
 * GET /whoop/recovery. Returns the user's daily recovery rows ordered by date
 * for the local-date window implied by `since`/`until` (inclusive, YYYY-MM-DD)
 * in `timezone` (IANA name, e.g. "America/Denver"), per the house
 * timezone+local-date convention. Call sites pass local dates + the browser
 * timezone; the client never constructs UTC instants. A user with no Whoop
 * connection simply yields an empty list.
 */
export async function listWhoopRecovery(
  token: string,
  opts: { timezone: string; since?: string; until?: string },
): Promise<WhoopRecoveryDay[]> {
  const params = new URLSearchParams();
  params.set("timezone", opts.timezone);
  if (opts.since) params.set("since", opts.since);
  if (opts.until) params.set("until", opts.until);
  const resp = await fetch(`${config.apiUrl}/whoop/recovery?${params.toString()}`, {
    headers: { Authorization: `Bearer ${token}` },
  });
  return unwrap<WhoopRecoveryDay[]>(resp, []);
}
```

- [ ] **Step 4: Run the tests to verify they pass**

Run: `npx vitest run lib/api.test.ts -t listWhoopRecovery`
Expected: PASS (4 tests).

- [ ] **Step 5: Commit**

```bash
git add lib/api.ts lib/api.test.ts
git commit -m "feat(recovery): add listWhoopRecovery api client"
```

---

### Task 2: Promote `SegmentedToggle` to `components/`

**Files:**
- Create: `components/segmented-toggle.tsx`
- Create: `components/segmented-toggle.test.tsx`
- Modify: `app/(app)/settings/_components/primitives.tsx` (remove `SegmentedToggle`)
- Modify: `app/(app)/settings/_components/primitives.test.tsx` (remove its `SegmentedToggle` describe)
- Modify: `app/(app)/settings/page.tsx` (update import)

The recovery page's range toggle is the **second** consumer of `SegmentedToggle`, which currently lives in the settings-page-private `_components/primitives.tsx`. Per AGENTS.md ("anything used by two or more pages lives under `components/`. Promote on second use"), move it to a shared module. Behavior is unchanged — this is a pure move + import rewire.

- [ ] **Step 1: Create the shared component**

Create `components/segmented-toggle.tsx` (verbatim body from the current `primitives.tsx`, with the doc comment):

```tsx
/** Full-pill segmented control: active = accent fill, inactive = muted. */
export function SegmentedToggle<T extends string>({
  value,
  options,
  onChange,
  disabled = false,
}: {
  value: T;
  options: { value: T; label: string }[];
  onChange: (value: T) => void;
  disabled?: boolean;
}) {
  return (
    <div
      role="group"
      className="inline-flex rounded-full border border-[var(--border)] bg-[var(--background)] p-0.5"
    >
      {options.map((opt) => {
        const active = opt.value === value;
        return (
          <button
            key={opt.value}
            type="button"
            disabled={disabled}
            aria-pressed={active}
            onClick={() => {
              if (!active) onChange(opt.value);
            }}
            className={`rounded-full px-3 py-1 text-xs font-medium transition disabled:cursor-not-allowed disabled:opacity-60 ${
              active
                ? "bg-[var(--accent)] text-[var(--accent-fg)]"
                : "text-[var(--muted)] hover:text-[var(--foreground)]"
            }`}
          >
            {opt.label}
          </button>
        );
      })}
    </div>
  );
}
```

- [ ] **Step 2: Create its test** (move the two `SegmentedToggle` cases from `primitives.test.tsx`)

Create `components/segmented-toggle.test.tsx`:

```tsx
/// <reference types="vitest/globals" />
import { render, screen, fireEvent } from "@testing-library/react";
import { SegmentedToggle } from "./segmented-toggle";

describe("SegmentedToggle", () => {
  it("marks the active option and fires onChange for another", () => {
    const onChange = vi.fn();
    render(
      <SegmentedToggle
        value="mi"
        options={[
          { value: "mi", label: "Miles" },
          { value: "km", label: "Kilometers" },
        ]}
        onChange={onChange}
      />,
    );
    expect(screen.getByRole("button", { name: "Miles" })).toHaveAttribute("aria-pressed", "true");
    fireEvent.click(screen.getByRole("button", { name: "Kilometers" }));
    expect(onChange).toHaveBeenCalledWith("km");
  });

  it("does not fire onChange when clicking the already-active option", () => {
    const onChange = vi.fn();
    render(
      <SegmentedToggle
        value="mi"
        options={[
          { value: "mi", label: "Miles" },
          { value: "km", label: "Kilometers" },
        ]}
        onChange={onChange}
      />,
    );
    fireEvent.click(screen.getByRole("button", { name: "Miles" }));
    expect(onChange).not.toHaveBeenCalled();
  });
});
```

- [ ] **Step 3: Remove `SegmentedToggle` from `primitives.tsx`**

Delete the entire `SegmentedToggle` function (the `/** Full-pill segmented control… */` comment + function) from `app/(app)/settings/_components/primitives.tsx`. Leave `inputClass`, `Card`, and `Field` untouched.

- [ ] **Step 4: Remove the moved test from `primitives.test.tsx`**

In `app/(app)/settings/_components/primitives.test.tsx`, delete the whole `describe("SegmentedToggle", …)` block and remove `SegmentedToggle` from the `import { Card, Field, SegmentedToggle } from "./primitives";` line (leave `Card, Field`).

- [ ] **Step 5: Rewire the settings page import**

In `app/(app)/settings/page.tsx`, change line 32:

```tsx
import { Card, Field, inputClass } from "./_components/primitives";
import { SegmentedToggle } from "@/components/segmented-toggle";
```

- [ ] **Step 6: Run the affected tests + typecheck**

Run: `npx vitest run components/segmented-toggle.test.tsx "app/(app)/settings" && npm run typecheck`
Expected: PASS; typecheck clean (settings page still compiles with the new import).

- [ ] **Step 7: Commit**

```bash
git add components/segmented-toggle.tsx components/segmented-toggle.test.tsx "app/(app)/settings/_components/primitives.tsx" "app/(app)/settings/_components/primitives.test.tsx" "app/(app)/settings/page.tsx"
git commit -m "refactor: promote SegmentedToggle to shared components"
```

---

### Task 3: Chart colors for the recovery trends

**Files:**
- Modify: `lib/chart-colors.ts`
- Test: none (constants only; exercised by Task 8's chart tests)

recharts cannot resolve `var(--token)`, so the recovery charts need hex constants mirroring the design-system tokens. The three metric lines read as a **calm neutral foreground** (`--foreground #c8cad0`) — meaning is carried by the score chart's colored band zones and the RHR/HRV average reference lines, not by a series hue (per the design system: the accent stays interactive-only, and status colors are semantic, so the score lines must not read as brand or as an activity). The band fills mirror the semantic status tokens (`--success`/`--warning`/`--danger`), applied at low opacity by the `ReferenceArea` so they read as translucent zones.

- [ ] **Step 1: Append the constants**

Add to the end of `lib/chart-colors.ts`:

```typescript
// --- Recovery (Whoop) page ---------------------------------------------
// The three metric lines are the calm neutral --foreground: meaning comes from
// the score chart's band zones and the RHR/HRV average reference lines, not a
// per-series hue (the accent stays interactive-only; status colors are
// semantic). The band fills mirror --success/--warning/--danger, drawn
// translucent by ReferenceArea (see RECOVERY_BAND_FILL_OPACITY).
export const CHART_RECOVERY_SCORE = "#c8cad0"; // --foreground (score line, over color bands)
export const CHART_RECOVERY_RHR = "#c8cad0"; // --foreground (resting-HR line)
export const CHART_RECOVERY_HRV = "#c8cad0"; // --foreground (HRV line)
export const CHART_RECOVERY_BAND_SUCCESS = "#86b39f"; // --success (score >= 67)
export const CHART_RECOVERY_BAND_WARNING = "#d6b87f"; // --warning (score 34-66)
export const CHART_RECOVERY_BAND_DANGER = "#c79292"; // --danger (score <= 33)
export const RECOVERY_BAND_FILL_OPACITY = 0.1; // translucent zone fill
export const CHART_RECOVERY_AVG = "#565a63"; // --faint (dashed average reference line)
```

- [ ] **Step 2: Verify it compiles**

Run: `npm run typecheck`
Expected: PASS.

- [ ] **Step 3: Commit**

```bash
git add lib/chart-colors.ts
git commit -m "feat(recovery): add recovery chart color tokens"
```

---

### Task 4: Recovery helper module (`lib/recovery.ts`)

**Files:**
- Create: `lib/recovery.ts`
- Test: `lib/recovery.test.ts`

The only real logic on the page: band classification and shaping the fetched rows for the charts. Kept pure and separate so the band boundaries are unit-testable and the chart-data mapping (nulls → gaps) is verified without rendering. Reuse the existing date helpers from `lib/steps-stats.ts` (`isoDate`, `rangeSinceIso`, `parseLocalDate`) at the call sites — do **not** duplicate them here.

- [ ] **Step 1: Write the failing tests**

Create `lib/recovery.test.ts`:

```typescript
import { describe, expect, it } from "vitest";
import {
  recoveryBand,
  recoveryBandColor,
  toChartRows,
  latestForToday,
  type RecoveryBand,
} from "./recovery";
import type { WhoopRecoveryDay } from "./api";

describe("recoveryBand", () => {
  it("maps success at >= 67, including the 67 boundary", () => {
    expect(recoveryBand(100)).toBe("success");
    expect(recoveryBand(67)).toBe("success");
  });

  it("maps warning across 34..66 inclusive", () => {
    expect(recoveryBand(66)).toBe("warning");
    expect(recoveryBand(34)).toBe("warning");
  });

  it("maps danger at <= 33, including the 33 boundary", () => {
    expect(recoveryBand(33)).toBe("danger");
    expect(recoveryBand(0)).toBe("danger");
  });
});

describe("recoveryBandColor", () => {
  it("returns the semantic token var for each band", () => {
    const colors: Record<RecoveryBand, string> = {
      success: "var(--success)",
      warning: "var(--warning)",
      danger: "var(--danger)",
    };
    expect(recoveryBandColor("success")).toBe(colors.success);
    expect(recoveryBandColor("warning")).toBe(colors.warning);
    expect(recoveryBandColor("danger")).toBe(colors.danger);
  });
});

describe("toChartRows", () => {
  const rows: WhoopRecoveryDay[] = [
    { date: "2026-07-02", recovery_score: 72, resting_heart_rate: 54, hrv_rmssd_milli: 88.4 },
    { date: "2026-07-01", recovery_score: null, resting_heart_rate: null, hrv_rmssd_milli: null },
  ];

  it("returns rows ascending by date with null metrics preserved as null (gaps, not zeros)", () => {
    const out = toChartRows(rows);
    expect(out.map((r) => r.date)).toEqual(["2026-07-01", "2026-07-02"]);
    expect(out[0]).toEqual({
      date: "2026-07-01",
      recovery_score: null,
      resting_heart_rate: null,
      hrv_rmssd_milli: null,
    });
    expect(out[1].recovery_score).toBe(72);
  });

  it("does not mutate the input array", () => {
    const copy = [...rows];
    toChartRows(rows);
    expect(rows).toEqual(copy);
  });
});

describe("latestForToday", () => {
  const rows: WhoopRecoveryDay[] = [
    { date: "2026-07-01", recovery_score: 60, resting_heart_rate: 55, hrv_rmssd_milli: 70 },
    { date: "2026-07-02", recovery_score: 72, resting_heart_rate: 54, hrv_rmssd_milli: 88 },
  ];

  it("returns today's row when a row matches today's date", () => {
    expect(latestForToday(rows, "2026-07-02")).toEqual(rows[1]);
  });

  it("returns null when no row matches today (never promotes yesterday)", () => {
    expect(latestForToday(rows, "2026-07-03")).toBeNull();
  });

  it("returns null for an empty list", () => {
    expect(latestForToday([], "2026-07-02")).toBeNull();
  });
});
```

- [ ] **Step 2: Run to verify failure**

Run: `npx vitest run lib/recovery.test.ts`
Expected: FAIL — module `./recovery` not found.

- [ ] **Step 3: Implement `lib/recovery.ts`**

```typescript
import type { WhoopRecoveryDay } from "./api";

/**
 * Recovery zone bands, mirroring Whoop's green/yellow/red convention so the
 * numbers agree with the user's Whoop app: success >= 67, warning 34..66,
 * danger <= 33.
 */
export type RecoveryBand = "success" | "warning" | "danger";

export function recoveryBand(score: number): RecoveryBand {
  if (score >= 67) return "success";
  if (score >= 34) return "warning";
  return "danger";
}

/** The CSS variable for a band's semantic status color (for non-recharts UI). */
export function recoveryBandColor(band: RecoveryBand): string {
  return `var(--${band})`;
}

/**
 * Shape fetched rows for the trend charts: ascending by date, with null metrics
 * left null so recharts renders them as gaps (never zeros). Non-mutating.
 */
export function toChartRows(rows: WhoopRecoveryDay[]): WhoopRecoveryDay[] {
  return [...rows].sort((a, b) => a.date.localeCompare(b.date));
}

/**
 * Today's recovery row, or null when there is none for `today`. Yesterday's row
 * is never promoted — an absent morning webhook must read as "no data yet
 * today", not as stale readiness. `today` is a local YYYY-MM-DD date.
 */
export function latestForToday(
  rows: WhoopRecoveryDay[],
  today: string,
): WhoopRecoveryDay | null {
  return rows.find((r) => r.date === today) ?? null;
}
```

- [ ] **Step 4: Run to verify pass**

Run: `npx vitest run lib/recovery.test.ts`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add lib/recovery.ts lib/recovery.test.ts
git commit -m "feat(recovery): add band + chart-row helpers"
```

---

### Task 5: Sidebar `Recovery` tab

**Files:**
- Modify: `components/sidebar.tsx` (add NAV entry between Bodyweight and Progress; add `RecoveryIcon` at the bottom with the other icons)
- Test: `components/sidebar.test.tsx`

The `NAV` array stays static (Non-Goal: no connection-gated nav). `RecoveryIcon` is a new inline stroke-only SVG matching the existing set: `viewBox="0 0 24 24"`, `width/height={16}`, `fill="none"`, `stroke="currentColor"`, `strokeWidth="1.75"`, rounded caps/joins, `aria-hidden`. Use a heart-with-pulse glyph, visually distinct from `ActivityIcon`'s bare waveform. No `exact` flag (no sibling routes) — active state uses the existing `startsWith` logic untouched.

- [ ] **Step 1: Write the failing tests**

Add these to `components/sidebar.test.tsx` inside a new `describe`, and update the existing "renders all 10 nav destinations" test to expect **11** including "Recovery". Concretely:

1. In the `for (const label of [ … ])` list in the "renders all 10 nav destinations" test, add `"Recovery"` between `"Bodyweight"` and `"Progress"`, and rename the test to "renders all 11 nav destinations including Recovery".
2. Add a new test:

```tsx
describe("Sidebar — Recovery tab", () => {
  it("renders a Recovery link pointing at /recovery, after Bodyweight and before Progress", () => {
    render(<Sidebar />);
    const links = screen.getAllByRole("link");
    const recovery = screen.getByRole("link", { name: "Recovery" });
    expect(recovery).toHaveAttribute("href", "/recovery");
    const bodyweight = screen.getByRole("link", { name: "Bodyweight" });
    const progress = screen.getByRole("link", { name: "Progress" });
    expect(links.indexOf(bodyweight)).toBeLessThan(links.indexOf(recovery));
    expect(links.indexOf(recovery)).toBeLessThan(links.indexOf(progress));
  });

  it("marks Recovery active on /recovery", () => {
    pathnameRef.current = "/recovery";
    render(<Sidebar />);
    expect(screen.getByRole("link", { name: "Recovery" })).toHaveAttribute(
      "aria-current",
      "page",
    );
  });
});
```

- [ ] **Step 2: Run to verify failure**

Run: `npx vitest run components/sidebar.test.tsx`
Expected: FAIL — no Recovery link; and the updated count test fails.

- [ ] **Step 3: Add the NAV entry**

In `components/sidebar.tsx`, insert into the `NAV` array between the Bodyweight and Progress entries:

```tsx
  { href: "/recovery", label: "Recovery", icon: <RecoveryIcon /> },
```

- [ ] **Step 4: Add the `RecoveryIcon`**

Add near the other icon components at the bottom of `components/sidebar.tsx` (match the neighboring icon style exactly):

```tsx
function RecoveryIcon() {
  return (
    <svg
      viewBox="0 0 24 24"
      width={16}
      height={16}
      fill="none"
      stroke="currentColor"
      strokeWidth="1.75"
      strokeLinecap="round"
      strokeLinejoin="round"
      aria-hidden="true"
    >
      <path d="M20.5 8.5a4.5 4.5 0 0 0-8.5-2 4.5 4.5 0 0 0-8.5 2c0 1.6 1 3.2 2.6 4.6H9l1.5-2.5 2 4 1.5-2.5h4.4C19.5 11.7 20.5 10.1 20.5 8.5Z" />
    </svg>
  );
}
```

(A heart outline whose bottom is interrupted by a pulse tick — distinct from `ActivityIcon`'s plain polyline waveform.)

- [ ] **Step 5: Run to verify pass**

Run: `npx vitest run components/sidebar.test.tsx`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add components/sidebar.tsx components/sidebar.test.tsx
git commit -m "feat(recovery): add Recovery sidebar tab"
```

---

### Task 6: Dashboard recovery card deep-link → `/recovery`

**Files:**
- Modify: `app/(app)/dashboard/page.tsx` (`DEEP_LINKS.recovery`)
- Test: `app/(app)/dashboard/page.test.tsx`

Only the deep-link target changes; the card renders unchanged (already gated on the summary carrying the recovery block). `whoop-card.test.tsx` passes its own `href` prop and is unaffected.

- [ ] **Step 1: Update the deep-link test**

In `app/(app)/dashboard/page.test.tsx`, find the `it("links each card into its deep page", …)` test. If the `FULL_SUMMARY`/summary fixture used by that test includes a `recovery` block, add:

```tsx
    expect(hrefFor(/Recovery/i)).toBe("/recovery");
```

If the fixture for that test has `recovery: null` (card not rendered), instead add a dedicated test that ensures a recovery block is present, then asserts the link. Inspect the fixture first; use whichever fits. A self-contained version that does not depend on the shared fixture's recovery field:

```tsx
  it("deep-links the recovery card to /recovery", async () => {
    summaryToReturn = {
      ...FULL_SUMMARY,
      recovery: {
        today: {
          date: "2026-07-02",
          resting_heart_rate: 54,
          recovery_score: 72,
          hrv_rmssd_milli: 88,
        },
        resting_hr_spark: [55, 54, 56, 54],
      },
    };
    render(<DashboardPage />);
    const recovery = await screen.findByRole("link", { name: /Recovery/i });
    expect(recovery).toHaveAttribute("href", "/recovery");
  });
```

Match the fixture/variable names actually used in the file (`summaryToReturn` / `FULL_SUMMARY` in the examples above are from the existing tests — confirm and reuse the real ones).

- [ ] **Step 2: Run to verify failure**

Run: `npx vitest run "app/(app)/dashboard/page.test.tsx" -t recovery`
Expected: FAIL — link still points at `/settings?tab=integrations`.

- [ ] **Step 3: Flip the deep link**

In `app/(app)/dashboard/page.tsx`, change the `DEEP_LINKS` entry:

```tsx
  recovery: "/recovery",
```

- [ ] **Step 4: Run to verify pass**

Run: `npx vitest run "app/(app)/dashboard/page.test.tsx"`
Expected: PASS (all dashboard tests).

- [ ] **Step 5: Commit**

```bash
git add "app/(app)/dashboard/page.tsx" "app/(app)/dashboard/page.test.tsx"
git commit -m "feat(recovery): deep-link dashboard recovery card to /recovery"
```

---

### Task 7: `RecoveryHero` component

**Files:**
- Create: `app/(app)/recovery/_components/recovery-hero.tsx`
- Test: `app/(app)/recovery/_components/recovery-hero.test.tsx`

A score ring in the steps `HeroRing` idiom, filled to `today.recovery_score / 100` and stroked by band color (semantic `--success`/`--warning`/`--danger`, never the accent). The score number is centered; two MiniStats beside it show today's resting HR (bpm) and HRV (ms). **Missing-today rule:** when `today` is null, the ring renders unfilled with a "no data yet today" caption and both MiniStats show em-dashes — yesterday's row is never promoted (the parent passes `today={latestForToday(rows, todayIso)}`, so this component only decides rendering, not selection). The ring + MiniStat markup mirrors the steps-view idiom but is self-contained here (steps-view's `Ring`/`MiniStat` are module-private and not exported).

- [ ] **Step 1: Write the failing tests**

Create `app/(app)/recovery/_components/recovery-hero.test.tsx`:

```tsx
/// <reference types="vitest/globals" />
import { render, screen } from "@testing-library/react";
import { RecoveryHero } from "./recovery-hero";

describe("RecoveryHero", () => {
  it("renders today's score, resting HR, and HRV with a band-colored ring", () => {
    render(
      <RecoveryHero
        today={{
          date: "2026-07-02",
          recovery_score: 72,
          resting_heart_rate: 54,
          hrv_rmssd_milli: 88.4,
        }}
      />,
    );
    expect(screen.getByTestId("recovery-score")).toHaveTextContent("72");
    // success band (>= 67)
    expect(screen.getByTestId("recovery-ring-fill")).toHaveAttribute("stroke", "var(--success)");
    expect(screen.getByText("54")).toBeInTheDocument(); // resting HR bpm
    expect(screen.getByText("88")).toBeInTheDocument(); // HRV ms (rounded)
  });

  it("uses the warning band at 66 and success at 67", () => {
    const { rerender } = render(
      <RecoveryHero
        today={{ date: "d", recovery_score: 66, resting_heart_rate: 50, hrv_rmssd_milli: 60 }}
      />,
    );
    expect(screen.getByTestId("recovery-ring-fill")).toHaveAttribute("stroke", "var(--warning)");
    rerender(
      <RecoveryHero
        today={{ date: "d", recovery_score: 67, resting_heart_rate: 50, hrv_rmssd_milli: 60 }}
      />,
    );
    expect(screen.getByTestId("recovery-ring-fill")).toHaveAttribute("stroke", "var(--success)");
  });

  it("renders a no-data state with em-dashes when today is null (no promotion)", () => {
    render(<RecoveryHero today={null} />);
    expect(screen.getByText(/no data yet today/i)).toBeInTheDocument();
    expect(screen.queryByTestId("recovery-ring-fill")).not.toBeInTheDocument();
    // Both MiniStats show an em-dash.
    expect(screen.getAllByText("—").length).toBeGreaterThanOrEqual(2);
  });

  it("shows em-dash for an individually-null metric even when the score exists", () => {
    render(
      <RecoveryHero
        today={{ date: "d", recovery_score: 40, resting_heart_rate: null, hrv_rmssd_milli: null }}
      />,
    );
    expect(screen.getByTestId("recovery-score")).toHaveTextContent("40");
    expect(screen.getByTestId("recovery-ring-fill")).toHaveAttribute("stroke", "var(--warning)");
    expect(screen.getAllByText("—").length).toBeGreaterThanOrEqual(2);
  });
});
```

- [ ] **Step 2: Run to verify failure**

Run: `npx vitest run "app/(app)/recovery/_components/recovery-hero.test.tsx"`
Expected: FAIL — component not found.

- [ ] **Step 3: Implement `recovery-hero.tsx`**

```tsx
import type { WhoopRecoveryDay } from "@/lib/api";
import { recoveryBand, recoveryBandColor } from "@/lib/recovery";

/**
 * Today's readiness at a glance: a band-colored score ring (filled to
 * score/100) with resting-HR and HRV MiniStats beside it. When `today` is null
 * the ring is unfilled with a "no data yet today" caption and the stats read
 * em-dashes — the parent decides what "today" is (never promoting an older
 * row), this component only renders it.
 */
export function RecoveryHero({ today }: { today: WhoopRecoveryDay | null }) {
  const score = today?.recovery_score ?? null;
  const band = score !== null ? recoveryBand(score) : null;
  const stroke = band !== null ? recoveryBandColor(band) : null;

  return (
    <div className="flex flex-col items-center gap-8">
      <div className="flex flex-col items-center gap-2 pt-2">
        <Ring pct={score} stroke={stroke} />
        <p className="text-center text-[13px] text-[var(--muted)]">
          {score !== null ? "recovery today" : "no data yet today"}
        </p>
      </div>

      <div className="flex w-full max-w-sm items-stretch justify-center gap-px overflow-hidden rounded-[var(--radius-card)] border border-[var(--border)] bg-[var(--border)] text-center">
        <MiniStat label="Resting HR" value={fmt(today?.resting_heart_rate ?? null, "bpm")} />
        <MiniStat label="HRV" value={fmt(today?.hrv_rmssd_milli ?? null, "ms")} />
      </div>
    </div>
  );
}

/** The score ring: fills to score/100 in the band color; unfilled when null. */
function Ring({ pct, stroke }: { pct: number | null; stroke: string | null }) {
  const r = 52;
  const c = 2 * Math.PI * r;
  const frac = pct === null ? 0 : Math.min(Math.max(pct, 0), 100) / 100;
  return (
    <div className="relative grid h-[148px] w-[148px] place-items-center sm:h-[180px] sm:w-[180px]">
      <svg viewBox="0 0 128 128" className="h-full w-full -rotate-90" aria-hidden="true">
        <circle cx="64" cy="64" r={r} fill="none" stroke="var(--surface-3)" strokeWidth="11" />
        {pct !== null && stroke !== null && (
          <circle
            data-testid="recovery-ring-fill"
            cx="64"
            cy="64"
            r={r}
            fill="none"
            stroke={stroke}
            strokeWidth="11"
            strokeLinecap="round"
            strokeDasharray={c}
            strokeDashoffset={c * (1 - frac)}
          />
        )}
      </svg>
      <div className="absolute flex flex-col items-center">
        <span
          data-testid="recovery-score"
          className="text-[34px] font-semibold leading-none tracking-[-0.04em] tabular-nums sm:text-[42px]"
        >
          {pct !== null ? Math.round(pct) : "—"}
        </span>
      </div>
    </div>
  );
}

function MiniStat({ label, value }: { label: string; value: string }) {
  return (
    <div className="flex-1 bg-[var(--background)] px-3 py-3">
      <div className="text-[16px] font-semibold tabular-nums text-[var(--foreground)]">{value}</div>
      <div className="mt-0.5 text-[10px] font-semibold uppercase tracking-[0.14em] text-[var(--faint)]">
        {label}
      </div>
    </div>
  );
}

/** Rounds a nullable metric to a whole number with a unit suffix; null → em-dash. */
function fmt(value: number | null, unit: string): string {
  return value === null ? "—" : `${Math.round(value)} ${unit}`;
}
```

- [ ] **Step 4: Run to verify pass**

Run: `npx vitest run "app/(app)/recovery/_components/recovery-hero.test.tsx"`
Expected: PASS.

Note: the test asserts `getByText("54")` and `getByText("88")` — the values render as `"54 bpm"` / `"88 ms"`, so use a function matcher if exact text fails. If `getByText("54")` does not match the combined string, change those assertions to `screen.getByText(/54\s*bpm/)` and `screen.getByText(/88\s*ms/)`. Prefer the regex form in the implementation of the test to avoid brittleness.

- [ ] **Step 5: Commit**

```bash
git add "app/(app)/recovery/_components/recovery-hero.tsx" "app/(app)/recovery/_components/recovery-hero.test.tsx"
git commit -m "feat(recovery): add RecoveryHero score ring"
```

---

### Task 8: `RecoveryTrends` component

**Files:**
- Create: `app/(app)/recovery/_components/recovery-trends.tsx`
- Test: `app/(app)/recovery/_components/recovery-trends.test.tsx`

Three recharts line panels (the metrics share no axis), each in a `ResponsiveContainer`, stacked on narrow widths and side-by-side where room allows (a responsive grid). All three use `connectNulls={false}` so null days are gaps, not zeros, and `isAnimationActive={false}` (matches every other chart + keeps tests deterministic).

- **Recovery score** — y fixed `[0, 100]`; three translucent `ReferenceArea` band zones (danger 0–33, warning 34–66, success 67–100) using the `CHART_RECOVERY_BAND_*` fills at `RECOVERY_BAND_FILL_OPACITY`; a neutral `CHART_RECOVERY_SCORE` line.
- **Resting HR** — y auto-fit (`["auto", "auto"]`); a dashed `CHART_RECOVERY_AVG` `ReferenceLine` at the range mean of non-null values; neutral `CHART_RECOVERY_RHR` line.
- **HRV** — same treatment as resting HR (dashed average `ReferenceLine`), neutral `CHART_RECOVERY_HRV` line.

The average is computed over non-null values only; when a metric has no non-null values the reference line is omitted.

Follow the recharts import + axis/tooltip idiom from `components/activities/steps-view.tsx` and `app/(app)/running/_components/PaceChart.tsx`: `CartesianGrid stroke={CHART_GRID}`, `XAxis`/`YAxis` `stroke={CHART_AXIS}` with `tick={{ fill: CHART_AXIS, fontSize: 11 }}`, and a tooltip styled with `CHART_TOOLTIP_BG`/`CHART_TOOLTIP_BORDER`/`CHART_TOOLTIP_RADIUS`.

- [ ] **Step 1: Write the failing tests**

recharts is mocked to expose the props under test as data-attributes (mirroring how steps-view tests assert on recharts). Create `app/(app)/recovery/_components/recovery-trends.test.tsx`:

```tsx
/// <reference types="vitest/globals" />
import { render, screen } from "@testing-library/react";

// Mock recharts to inspectable divs. Line/ReferenceArea/ReferenceLine record
// their key props as data-attributes so we can assert config without a real
// SVG layout.
vi.mock("recharts", () => {
  const Pass = ({ children }: { children?: React.ReactNode }) => <div>{children}</div>;
  return {
    ResponsiveContainer: Pass,
    LineChart: ({ children }: { children?: React.ReactNode }) => (
      <div data-recharts="LineChart">{children}</div>
    ),
    Line: ({
      dataKey,
      connectNulls,
      stroke,
    }: {
      dataKey: string;
      connectNulls?: boolean;
      stroke?: string;
    }) => (
      <div
        data-recharts="Line"
        data-key={dataKey}
        data-connect-nulls={String(connectNulls)}
        data-stroke={stroke}
      />
    ),
    ReferenceArea: ({ y1, y2, fill }: { y1: number; y2: number; fill: string }) => (
      <div data-recharts="ReferenceArea" data-y1={y1} data-y2={y2} data-fill={fill} />
    ),
    ReferenceLine: ({ y, stroke }: { y: number; stroke: string }) => (
      <div data-recharts="ReferenceLine" data-y={y} data-stroke={stroke} />
    ),
    XAxis: () => <div data-recharts="XAxis" />,
    YAxis: ({ domain }: { domain?: unknown }) => (
      <div data-recharts="YAxis" data-domain={JSON.stringify(domain)} />
    ),
    CartesianGrid: () => <div data-recharts="CartesianGrid" />,
    Tooltip: () => <div data-recharts="Tooltip" />,
  };
});

import { RecoveryTrends } from "./recovery-trends";
import type { WhoopRecoveryDay } from "@/lib/api";

const rows: WhoopRecoveryDay[] = [
  { date: "2026-07-01", recovery_score: 40, resting_heart_rate: 60, hrv_rmssd_milli: 70 },
  { date: "2026-07-02", recovery_score: null, resting_heart_rate: null, hrv_rmssd_milli: null },
  { date: "2026-07-03", recovery_score: 80, resting_heart_rate: 50, hrv_rmssd_milli: 90 },
];

describe("RecoveryTrends", () => {
  it("renders three line panels, one per metric", () => {
    render(<RecoveryTrends rows={rows} />);
    const keys = screen
      .getAllByTestId ? [] : []; // placeholder to keep TS happy if unused
    const lines = document.querySelectorAll('[data-recharts="Line"]');
    const dataKeys = Array.from(lines).map((l) => l.getAttribute("data-key"));
    expect(dataKeys).toEqual(
      expect.arrayContaining(["recovery_score", "resting_heart_rate", "hrv_rmssd_milli"]),
    );
  });

  it("disables connectNulls on every line (null days are gaps)", () => {
    render(<RecoveryTrends rows={rows} />);
    const lines = document.querySelectorAll('[data-recharts="Line"]');
    for (const line of Array.from(lines)) {
      expect(line.getAttribute("data-connect-nulls")).toBe("false");
    }
  });

  it("draws three score band zones with the semantic fills", () => {
    render(<RecoveryTrends rows={rows} />);
    const areas = Array.from(document.querySelectorAll('[data-recharts="ReferenceArea"]'));
    const fills = areas.map((a) => a.getAttribute("data-fill"));
    expect(fills).toEqual(
      expect.arrayContaining(["#c79292", "#d6b87f", "#86b39f"]),
    );
    // danger 0-33, warning 34-66, success 67-100 (boundaries).
    const danger = areas.find((a) => a.getAttribute("data-fill") === "#c79292")!;
    expect(danger.getAttribute("data-y1")).toBe("0");
    expect(danger.getAttribute("data-y2")).toBe("33");
    const success = areas.find((a) => a.getAttribute("data-fill") === "#86b39f")!;
    expect(success.getAttribute("data-y1")).toBe("67");
    expect(success.getAttribute("data-y2")).toBe("100");
  });

  it("draws an average reference line for resting HR and HRV at the non-null mean", () => {
    render(<RecoveryTrends rows={rows} />);
    const refLines = Array.from(document.querySelectorAll('[data-recharts="ReferenceLine"]'));
    const ys = refLines.map((l) => Number(l.getAttribute("data-y")));
    // resting HR mean of [60, 50] = 55; HRV mean of [70, 90] = 80.
    expect(ys).toEqual(expect.arrayContaining([55, 80]));
  });
});
```

- [ ] **Step 2: Run to verify failure**

Run: `npx vitest run "app/(app)/recovery/_components/recovery-trends.test.tsx"`
Expected: FAIL — component not found.

- [ ] **Step 3: Implement `recovery-trends.tsx`**

```tsx
import {
  CartesianGrid,
  Line,
  LineChart,
  ReferenceArea,
  ReferenceLine,
  ResponsiveContainer,
  Tooltip,
  XAxis,
  YAxis,
} from "recharts";
import type { WhoopRecoveryDay } from "@/lib/api";
import { toChartRows } from "@/lib/recovery";
import {
  CHART_AXIS,
  CHART_GRID,
  CHART_RECOVERY_AVG,
  CHART_RECOVERY_BAND_DANGER,
  CHART_RECOVERY_BAND_SUCCESS,
  CHART_RECOVERY_BAND_WARNING,
  CHART_RECOVERY_HRV,
  CHART_RECOVERY_RHR,
  CHART_RECOVERY_SCORE,
  CHART_TOOLTIP_BG,
  CHART_TOOLTIP_BORDER,
  CHART_TOOLTIP_RADIUS,
  RECOVERY_BAND_FILL_OPACITY,
} from "@/lib/chart-colors";

/**
 * Three trend panels — recovery score, resting HR, HRV — over the fetched
 * range. The metrics share no axis, so each gets its own panel: score is fixed
 * 0-100 with translucent band zones; resting HR and HRV auto-fit with a dashed
 * range-average reference line. Null days are gaps (connectNulls off), never
 * zeros. Responsive grid: stacked narrow, side-by-side where there's room.
 */
export function RecoveryTrends({ rows }: { rows: WhoopRecoveryDay[] }) {
  const data = toChartRows(rows);
  return (
    <div className="grid grid-cols-1 gap-4 lg:grid-cols-3">
      <ScorePanel data={data} />
      <MetricPanel
        title="Resting heart rate"
        data={data}
        dataKey="resting_heart_rate"
        stroke={CHART_RECOVERY_RHR}
        unit="bpm"
      />
      <MetricPanel
        title="HRV"
        data={data}
        dataKey="hrv_rmssd_milli"
        stroke={CHART_RECOVERY_HRV}
        unit="ms"
      />
    </div>
  );
}

function ScorePanel({ data }: { data: WhoopRecoveryDay[] }) {
  return (
    <Panel title="Recovery score">
      <ResponsiveContainer width="100%" height="100%">
        <LineChart data={data} margin={{ top: 12, right: 16, bottom: 8, left: 0 }}>
          <CartesianGrid stroke={CHART_GRID} strokeDasharray="3 3" vertical={false} />
          <ReferenceArea
            y1={0}
            y2={33}
            fill={CHART_RECOVERY_BAND_DANGER}
            fillOpacity={RECOVERY_BAND_FILL_OPACITY}
            ifOverflow="hidden"
          />
          <ReferenceArea
            y1={34}
            y2={66}
            fill={CHART_RECOVERY_BAND_WARNING}
            fillOpacity={RECOVERY_BAND_FILL_OPACITY}
            ifOverflow="hidden"
          />
          <ReferenceArea
            y1={67}
            y2={100}
            fill={CHART_RECOVERY_BAND_SUCCESS}
            fillOpacity={RECOVERY_BAND_FILL_OPACITY}
            ifOverflow="hidden"
          />
          <XAxis
            dataKey="date"
            stroke={CHART_AXIS}
            tick={{ fill: CHART_AXIS, fontSize: 11 }}
            minTickGap={16}
            tickFormatter={formatAxisDate}
          />
          <YAxis
            stroke={CHART_AXIS}
            tick={{ fill: CHART_AXIS, fontSize: 11 }}
            width={36}
            domain={[0, 100]}
          />
          <Tooltip
            cursor={{ stroke: CHART_AXIS, strokeWidth: 1 }}
            wrapperStyle={{ outline: "none" }}
            contentStyle={tooltipStyle}
            labelFormatter={formatTooltipDate}
            formatter={(v) => [String(v), "Score"]}
          />
          <Line
            dataKey="recovery_score"
            stroke={CHART_RECOVERY_SCORE}
            strokeWidth={2}
            dot={false}
            connectNulls={false}
            isAnimationActive={false}
          />
        </LineChart>
      </ResponsiveContainer>
    </Panel>
  );
}

function MetricPanel({
  title,
  data,
  dataKey,
  stroke,
  unit,
}: {
  title: string;
  data: WhoopRecoveryDay[];
  dataKey: "resting_heart_rate" | "hrv_rmssd_milli";
  stroke: string;
  unit: string;
}) {
  const avg = mean(data.map((d) => d[dataKey]));
  return (
    <Panel title={title}>
      <ResponsiveContainer width="100%" height="100%">
        <LineChart data={data} margin={{ top: 12, right: 16, bottom: 8, left: 0 }}>
          <CartesianGrid stroke={CHART_GRID} strokeDasharray="3 3" vertical={false} />
          <XAxis
            dataKey="date"
            stroke={CHART_AXIS}
            tick={{ fill: CHART_AXIS, fontSize: 11 }}
            minTickGap={16}
            tickFormatter={formatAxisDate}
          />
          <YAxis
            stroke={CHART_AXIS}
            tick={{ fill: CHART_AXIS, fontSize: 11 }}
            width={36}
            domain={["auto", "auto"]}
          />
          <Tooltip
            cursor={{ stroke: CHART_AXIS, strokeWidth: 1 }}
            wrapperStyle={{ outline: "none" }}
            contentStyle={tooltipStyle}
            labelFormatter={formatTooltipDate}
            formatter={(v) => [`${v} ${unit}`, title]}
          />
          {avg !== null && (
            <ReferenceLine
              y={avg}
              stroke={CHART_RECOVERY_AVG}
              strokeDasharray="6 4"
              strokeWidth={1}
              ifOverflow="extendDomain"
              label={{
                value: `avg ${Math.round(avg)} ${unit}`,
                position: "right",
                fill: CHART_RECOVERY_AVG,
                fontSize: 10,
              }}
            />
          )}
          <Line
            dataKey={dataKey}
            stroke={stroke}
            strokeWidth={2}
            dot={false}
            connectNulls={false}
            isAnimationActive={false}
          />
        </LineChart>
      </ResponsiveContainer>
    </Panel>
  );
}

function Panel({ title, children }: { title: string; children: React.ReactNode }) {
  return (
    <div className="flex flex-col gap-3 rounded-[var(--radius-card)] border border-[var(--border)] bg-[var(--surface)] p-3 sm:p-4">
      <h3 className="text-sm font-semibold tracking-tight">{title}</h3>
      <div className="h-[220px] w-full">{children}</div>
    </div>
  );
}

const tooltipStyle = {
  backgroundColor: CHART_TOOLTIP_BG,
  border: `1px solid ${CHART_TOOLTIP_BORDER}`,
  borderRadius: CHART_TOOLTIP_RADIUS,
  padding: "8px 10px",
  fontSize: "12px",
} as const;

/** Mean of the non-null numbers, or null when there are none. */
function mean(values: (number | null)[]): number | null {
  const nums = values.filter((v): v is number => v !== null);
  if (nums.length === 0) return null;
  return nums.reduce((a, b) => a + b, 0) / nums.length;
}

function formatAxisDate(date: string): string {
  const [y, m, d] = date.split("-").map(Number);
  return `${m}/${d}`;
}

function formatTooltipDate(date: string | number): string {
  const [y, m, d] = String(date).split("-").map(Number);
  return new Date(y, m - 1, d).toLocaleDateString("en-US", {
    weekday: "short",
    month: "short",
    day: "numeric",
  });
}
```

- [ ] **Step 4: Run to verify pass**

Run: `npx vitest run "app/(app)/recovery/_components/recovery-trends.test.tsx"`
Expected: PASS. (Remove the placeholder `const keys = …` line from the first test if it causes an unused-var lint error — it exists only to illustrate; the real assertion uses `document.querySelectorAll`.)

- [ ] **Step 5: Commit**

```bash
git add "app/(app)/recovery/_components/recovery-trends.tsx" "app/(app)/recovery/_components/recovery-trends.test.tsx"
git commit -m "feat(recovery): add RecoveryTrends charts"
```

---

### Task 9: `RecoveryLog` component

**Files:**
- Create: `app/(app)/recovery/_components/recovery-log.tsx`
- Test: `app/(app)/recovery/_components/recovery-log.test.tsx`

A reverse-chronological (newest first) plain list of the fetched rows: date, the score as a small band-colored chip, resting HR, HRV; em-dash for nulls. No accordion, no pagination (≤ ~90 rows, already fetched). The score chip uses the band's semantic color as a soft tint (mirrors the macro/stat-chip idiom — rounded-full, low-alpha background).

- [ ] **Step 1: Write the failing tests**

Create `app/(app)/recovery/_components/recovery-log.test.tsx`:

```tsx
/// <reference types="vitest/globals" />
import { render, screen, within } from "@testing-library/react";
import { RecoveryLog } from "./recovery-log";
import type { WhoopRecoveryDay } from "@/lib/api";

const rows: WhoopRecoveryDay[] = [
  { date: "2026-07-01", recovery_score: 40, resting_heart_rate: 60, hrv_rmssd_milli: 70 },
  { date: "2026-07-03", recovery_score: 80, resting_heart_rate: 50, hrv_rmssd_milli: 90.2 },
  { date: "2026-07-02", recovery_score: null, resting_heart_rate: null, hrv_rmssd_milli: null },
];

describe("RecoveryLog", () => {
  it("lists rows newest-first", () => {
    render(<RecoveryLog rows={rows} />);
    const items = screen.getAllByRole("listitem");
    expect(within(items[0]).getByText(/Jul 3/)).toBeInTheDocument();
    expect(within(items[1]).getByText(/Jul 2/)).toBeInTheDocument();
    expect(within(items[2]).getByText(/Jul 1/)).toBeInTheDocument();
  });

  it("renders the score as a band-colored chip and rounds metrics", () => {
    render(<RecoveryLog rows={rows} />);
    const top = screen.getAllByRole("listitem")[0];
    const chip = within(top).getByTestId("recovery-log-chip");
    expect(chip).toHaveTextContent("80");
    expect(chip).toHaveStyle({ color: "var(--success)" });
    expect(within(top).getByText(/50/)).toBeInTheDocument(); // resting HR
    expect(within(top).getByText(/90/)).toBeInTheDocument(); // HRV rounded
  });

  it("shows em-dashes for a fully-null day", () => {
    render(<RecoveryLog rows={rows} />);
    const middle = screen.getAllByRole("listitem")[1]; // 2026-07-02
    expect(within(middle).getAllByText("—").length).toBeGreaterThanOrEqual(3);
  });
});
```

- [ ] **Step 2: Run to verify failure**

Run: `npx vitest run "app/(app)/recovery/_components/recovery-log.test.tsx"`
Expected: FAIL — component not found.

- [ ] **Step 3: Implement `recovery-log.tsx`**

```tsx
import type { WhoopRecoveryDay } from "@/lib/api";
import { recoveryBand, recoveryBandColor } from "@/lib/recovery";

/**
 * Reverse-chronological day log of the fetched recovery rows — date, a
 * band-colored score chip, resting HR, HRV; em-dash for nulls. Plain list, no
 * accordion, no pagination (the rows are already fetched for the charts).
 */
export function RecoveryLog({ rows }: { rows: WhoopRecoveryDay[] }) {
  const sorted = [...rows].sort((a, b) => b.date.localeCompare(a.date));
  return (
    <section className="flex flex-col gap-3">
      <h3 className="text-sm font-semibold tracking-tight">Day log</h3>
      <ul className="flex flex-col divide-y divide-[var(--border)] overflow-hidden rounded-[var(--radius-card)] border border-[var(--border)] bg-[var(--surface)]">
        {sorted.map((row) => (
          <li key={row.date} className="flex items-center justify-between gap-3 px-4 py-3">
            <span className="text-sm text-[var(--foreground)]">{formatRowDate(row.date)}</span>
            <div className="flex items-center gap-5">
              <ScoreChip score={row.recovery_score} />
              <Metric value={row.resting_heart_rate} unit="bpm" />
              <Metric value={row.hrv_rmssd_milli} unit="ms" />
            </div>
          </li>
        ))}
      </ul>
    </section>
  );
}

function ScoreChip({ score }: { score: number | null }) {
  if (score === null) {
    return <span className="w-10 text-right text-sm tabular-nums text-[var(--faint)]">—</span>;
  }
  const color = recoveryBandColor(recoveryBand(score));
  return (
    <span
      data-testid="recovery-log-chip"
      className="inline-flex min-w-10 items-center justify-center rounded-full px-2 py-0.5 text-xs font-semibold tabular-nums"
      style={{ color, backgroundColor: `color-mix(in srgb, ${color} 14%, transparent)` }}
    >
      {score}
    </span>
  );
}

function Metric({ value, unit }: { value: number | null; unit: string }) {
  return (
    <span className="w-16 text-right text-sm tabular-nums text-[var(--muted)]">
      {value === null ? "—" : `${Math.round(value)} ${unit}`}
    </span>
  );
}

function formatRowDate(date: string): string {
  const [y, m, d] = date.split("-").map(Number);
  return new Date(y, m - 1, d).toLocaleDateString("en-US", {
    weekday: "short",
    month: "short",
    day: "numeric",
  });
}
```

Note on the chip background: `color-mix` with a `var()` color is used elsewhere for tints; if the code-quality reviewer or a jsdom `toHaveStyle` assertion struggles with `color-mix`, the test asserts only `color` (the `var(--success)` token), which is stable. Keep the tint subtle and token-derived (never a raw hex).

- [ ] **Step 4: Run to verify pass**

Run: `npx vitest run "app/(app)/recovery/_components/recovery-log.test.tsx"`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add "app/(app)/recovery/_components/recovery-log.tsx" "app/(app)/recovery/_components/recovery-log.test.tsx"
git commit -m "feat(recovery): add RecoveryLog day list"
```

---

### Task 10: `/recovery` page — fetch, render-gate, range toggle

**Files:**
- Create: `app/(app)/recovery/page.tsx`
- Test: `app/(app)/recovery/page.test.tsx`

The client page ties it together. On mount and on range change it resolves the browser timezone, computes `since`/`until` local dates for the selected range (default 30d), and runs `Promise.all([getWhoopConnection(token), listWhoopRecovery(token, { timezone, since, until })])`. One fetch feeds all three sections; today's row is `latestForToday(rows, todayIso)`.

**Render gate, in order:**
1. Connection `absent` or `revoked` → empty state: what the integration does + a **Connect Whoop** button linking `/settings?tab=integrations`.
2. Connection `error` → same layout, reconnect phrasing ("Whoop connection needs attention — Reconnect in Settings").
3. Connected, zero rows in range → "your first recovery lands after tonight's sleep."
4. Data → `RecoveryHero` + range toggle + `RecoveryTrends` + `RecoveryLog`.

Reuse the auth/401 idiom from `steps-view.tsx` (`getToken` → `clearToken` + `router.replace("/login")` on a 401). Range toggle is the promoted `SegmentedToggle` (Task 2). Date helpers come from `@/lib/steps-stats` (`isoDate`, `rangeSinceIso`). Selection is page state, not a URL param.

- [ ] **Step 1: Write the failing tests**

Create `app/(app)/recovery/page.test.tsx`:

```tsx
/// <reference types="vitest/globals" />
import { render, screen, fireEvent, waitFor } from "@testing-library/react";

const getTokenMock = vi.hoisted(() => vi.fn(() => "tok"));
const clearTokenMock = vi.hoisted(() => vi.fn());
const replaceMock = vi.hoisted(() => vi.fn());
const getWhoopConnectionMock = vi.hoisted(() => vi.fn());
const listWhoopRecoveryMock = vi.hoisted(() => vi.fn());

vi.mock("next/navigation", () => ({
  useRouter: () => ({ replace: replaceMock, push: vi.fn() }),
}));

vi.mock("@/lib/auth", () => ({
  getToken: getTokenMock,
  clearToken: clearTokenMock,
}));

vi.mock("@/lib/api", () => ({
  getWhoopConnection: getWhoopConnectionMock,
  listWhoopRecovery: listWhoopRecoveryMock,
}));

// recharts → passthrough divs so the trends render without layout.
vi.mock("recharts", () => {
  const Pass = ({ children }: { children?: React.ReactNode }) => <div>{children}</div>;
  return new Proxy(
    {},
    { get: () => Pass },
  );
});

import RecoveryPage from "./page";

const dayRow = {
  date: "2026-07-02",
  recovery_score: 72,
  resting_heart_rate: 54,
  hrv_rmssd_milli: 88,
};

beforeEach(() => {
  vi.useFakeTimers({ shouldAdvanceTime: true });
  vi.setSystemTime(new Date(2026, 6, 2, 12, 0, 0)); // 2026-07-02 local
  getTokenMock.mockReturnValue("tok");
  getWhoopConnectionMock.mockResolvedValue({ status: "connected" });
  listWhoopRecoveryMock.mockResolvedValue([dayRow]);
});

afterEach(() => {
  vi.useRealTimers();
  vi.clearAllMocks();
});

describe("RecoveryPage — render gate", () => {
  it("absent connection → Connect Whoop CTA into Settings", async () => {
    getWhoopConnectionMock.mockResolvedValue({ status: "absent" });
    render(<RecoveryPage />);
    const cta = await screen.findByRole("link", { name: /connect whoop/i });
    expect(cta).toHaveAttribute("href", "/settings?tab=integrations");
  });

  it("revoked connection → Connect Whoop CTA", async () => {
    getWhoopConnectionMock.mockResolvedValue({ status: "revoked" });
    render(<RecoveryPage />);
    expect(await screen.findByRole("link", { name: /connect whoop/i })).toBeInTheDocument();
  });

  it("error connection → reconnect phrasing", async () => {
    getWhoopConnectionMock.mockResolvedValue({ status: "error" });
    render(<RecoveryPage />);
    expect(await screen.findByText(/needs attention/i)).toBeInTheDocument();
    expect(screen.getByRole("link", { name: /reconnect/i })).toHaveAttribute(
      "href",
      "/settings?tab=integrations",
    );
  });

  it("connected but zero rows → first-night copy", async () => {
    listWhoopRecoveryMock.mockResolvedValue([]);
    render(<RecoveryPage />);
    expect(
      await screen.findByText(/your first recovery lands after tonight's sleep/i),
    ).toBeInTheDocument();
  });

  it("connected with data → hero score + all three sections", async () => {
    render(<RecoveryPage />);
    await waitFor(() => expect(screen.getByTestId("recovery-score")).toHaveTextContent("72"));
    expect(screen.getByText("Recovery score")).toBeInTheDocument();
    expect(screen.getByText("Day log")).toBeInTheDocument();
  });
});

describe("RecoveryPage — range + fetch", () => {
  it("fetches with the browser timezone and 30-day default window", async () => {
    render(<RecoveryPage />);
    await waitFor(() => expect(listWhoopRecoveryMock).toHaveBeenCalled());
    const call = listWhoopRecoveryMock.mock.calls[0];
    expect(call[0]).toBe("tok");
    expect(call[1]).toMatchObject({ until: "2026-07-02" });
    expect(typeof call[1].timezone).toBe("string");
    // 30 days back from 2026-07-02 → 2026-06-02.
    expect(call[1].since).toBe("2026-06-02");
  });

  it("refetches with a wider window when 90d is selected", async () => {
    render(<RecoveryPage />);
    await waitFor(() => expect(listWhoopRecoveryMock).toHaveBeenCalled());
    listWhoopRecoveryMock.mockClear();
    fireEvent.click(screen.getByRole("button", { name: "90d" }));
    await waitFor(() => expect(listWhoopRecoveryMock).toHaveBeenCalled());
    expect(listWhoopRecoveryMock.mock.calls[0][1].since).toBe("2026-04-03");
  });

  it("clears the token and redirects on a 401", async () => {
    getWhoopConnectionMock.mockRejectedValue(new Error("HTTP 401 unauthorized"));
    listWhoopRecoveryMock.mockRejectedValue(new Error("HTTP 401 unauthorized"));
    render(<RecoveryPage />);
    await waitFor(() => expect(clearTokenMock).toHaveBeenCalled());
    expect(replaceMock).toHaveBeenCalledWith("/login");
  });
});
```

- [ ] **Step 2: Run to verify failure**

Run: `npx vitest run "app/(app)/recovery/page.test.tsx"`
Expected: FAIL — page not found.

- [ ] **Step 3: Implement `page.tsx`**

```tsx
"use client";

import Link from "next/link";
import { useCallback, useEffect, useMemo, useState } from "react";
import { useRouter } from "next/navigation";
import { clearToken, getToken } from "@/lib/auth";
import {
  getWhoopConnection,
  listWhoopRecovery,
  type WhoopConnection,
  type WhoopRecoveryDay,
} from "@/lib/api";
import { isoDate, rangeSinceIso } from "@/lib/steps-stats";
import { latestForToday } from "@/lib/recovery";
import { SegmentedToggle } from "@/components/segmented-toggle";
import { RecoveryHero } from "./_components/recovery-hero";
import { RecoveryTrends } from "./_components/recovery-trends";
import { RecoveryLog } from "./_components/recovery-log";

type RangeKey = "7" | "30" | "90";
const RANGE_OPTIONS: { value: RangeKey; label: string }[] = [
  { value: "7", label: "7d" },
  { value: "30", label: "30d" },
  { value: "90", label: "90d" },
];

/**
 * Recovery — Whoop's daily readiness given a first-class home. Web-only,
 * read-only: the data is Whoop-owned and arrives each morning. One fetch per
 * range selection feeds the hero (today's score ring), the trend charts, and
 * the day log; a render gate ahead of them handles the no/broken-connection and
 * first-night states. All data flows through the existing GET /whoop/recovery
 * endpoint using the house timezone + local-date convention.
 */
export default function RecoveryPage() {
  const router = useRouter();
  const [range, setRange] = useState<RangeKey>("30");
  const [conn, setConn] = useState<WhoopConnection | null>(null);
  const [rows, setRows] = useState<WhoopRecoveryDay[] | null>(null);
  const [error, setError] = useState<string | null>(null);

  const handleAuthError = useCallback(
    (err: unknown): boolean => {
      const msg = err instanceof Error ? err.message : String(err);
      if (msg.toLowerCase().includes("401")) {
        clearToken();
        router.replace("/login");
        return true;
      }
      return false;
    },
    [router],
  );

  const refetch = useCallback(() => {
    const token = getToken();
    if (!token) {
      router.replace("/login");
      return;
    }
    const timezone = Intl.DateTimeFormat().resolvedOptions().timeZone;
    const days = Number(range);
    const since = rangeSinceIso(days);
    const until = isoDate(new Date());
    setRows(null);
    setError(null);
    Promise.all([
      getWhoopConnection(token),
      listWhoopRecovery(token, { timezone, since, until }),
    ])
      .then(([c, r]) => {
        setConn(c);
        setRows(r);
      })
      .catch((err: unknown) => {
        if (handleAuthError(err)) return;
        setError(err instanceof Error ? err.message : "Failed to load recovery");
      });
  }, [router, handleAuthError, range]);

  useEffect(() => {
    refetch();
  }, [refetch]);

  const todayIso = isoDate(new Date());
  const today = useMemo(
    () => (rows ? latestForToday(rows, todayIso) : null),
    [rows, todayIso],
  );

  return (
    <div className="mx-auto flex w-full max-w-4xl flex-col gap-6 px-4 py-6">
      <header className="flex flex-col gap-1">
        <h1 className="text-2xl font-semibold tracking-tight">Recovery</h1>
        <p className="text-sm text-[var(--muted)]">
          Your daily readiness from Whoop — recovery score, resting heart rate, and HRV.
        </p>
      </header>

      {error && (
        <div className="rounded-md border border-[var(--danger)]/40 bg-[var(--danger)]/10 px-3 py-2 text-sm text-[var(--danger)]">
          {error}
        </div>
      )}

      {conn === null && rows === null && !error && (
        <p className="text-sm text-[var(--muted)]">Loading recovery…</p>
      )}

      {conn !== null &&
        (conn.status === "absent" || conn.status === "revoked" ? (
          <ConnectState variant="connect" />
        ) : conn.status === "error" ? (
          <ConnectState variant="reconnect" />
        ) : rows !== null && rows.length === 0 ? (
          <FirstNightState />
        ) : rows !== null ? (
          <>
            <RecoveryHero today={today} />
            <div className="flex items-center justify-between">
              <h2 className="text-sm font-semibold tracking-tight">Trends</h2>
              <SegmentedToggle value={range} options={RANGE_OPTIONS} onChange={setRange} />
            </div>
            <RecoveryTrends rows={rows} />
            <RecoveryLog rows={rows} />
          </>
        ) : null)}
    </div>
  );
}

/** Empty state for a missing/revoked (connect) or errored (reconnect) link. */
function ConnectState({ variant }: { variant: "connect" | "reconnect" }) {
  return (
    <div className="flex flex-col items-center gap-4 rounded-[var(--radius-card)] border border-[var(--border)] bg-[var(--surface)] px-6 py-12 text-center">
      <p className="text-base font-semibold">Connect Whoop to see your recovery</p>
      <p className="max-w-md text-sm text-[var(--muted)]">
        {variant === "reconnect"
          ? "Whoop connection needs attention — reconnect to resume your daily recovery score, resting heart rate, and HRV."
          : "Whoop sends your recovery score, resting heart rate, and HRV each morning. Connect your account to track them here."}
      </p>
      <Link
        href="/settings?tab=integrations"
        className="inline-flex items-center rounded-full bg-[var(--accent)] px-4 py-2 text-[13px] font-medium text-[var(--accent-fg)] hover:opacity-80"
      >
        {variant === "reconnect" ? "Reconnect in Settings" : "Connect Whoop"}
      </Link>
    </div>
  );
}

/** Connected, but no rows have landed yet. */
function FirstNightState() {
  return (
    <div className="flex flex-col items-center gap-3 rounded-[var(--radius-card)] border border-[var(--border)] bg-[var(--surface)] px-6 py-12 text-center">
      <p className="text-base font-semibold">No recovery data yet</p>
      <p className="max-w-md text-sm text-[var(--muted)]">
        Your first recovery lands after tonight&apos;s sleep.
      </p>
    </div>
  );
}
```

- [ ] **Step 4: Run to verify pass**

Run: `npx vitest run "app/(app)/recovery/page.test.tsx"`
Expected: PASS.

Note on the `error`-state CTA test: it asserts a link named `/reconnect/i` — the "Reconnect in Settings" button matches. It also asserts the `/needs attention/i` copy — the `ConnectState variant="reconnect"` paragraph contains it. Keep both strings intact.

- [ ] **Step 5: Full local gate**

Run: `npm run lint && npm run format:check && npm run typecheck && npm run test`
Expected: all PASS. If `format:check` flags files, run `npx prettier --write` on the created/modified files and re-run. Then:

Run: `npm run build`
Expected: production build succeeds (new route compiles).

- [ ] **Step 6: Commit**

```bash
git add "app/(app)/recovery/page.tsx" "app/(app)/recovery/page.test.tsx"
git commit -m "feat(recovery): add /recovery page with render gate and range toggle"
```

---

## Self-Review

**1. Spec coverage** (checked against `sows/recovery-page.md`):
- Sidebar Recovery item after Bodyweight, house-style SVG icon, `startsWith` active state → Task 5. ✅
- Score ring color-banded (success ≥67 / warning 34–66 / danger ≤33), resting HR + HRV beside it, "no data yet today" before the webhook, no promotion of yesterday → Tasks 4, 7. ✅
- Trend charts (score/RHR/HRV) over 7/30/90 (default 30) + day log → Tasks 8, 9, 10. ✅
- Empty states: absent/revoked → connect CTA; error → reconnect; connected+zero → first-night → Task 10. ✅
- Dashboard deep-link flip → Task 6. ✅
- Existing `GET /whoop/recovery` via timezone+local-date, zero API changes → Task 1. ✅
- Score band `ReferenceArea` zones; RHR/HRV dashed average `ReferenceLine`; null days as gaps (`connectNulls` off); new `CHART_RECOVERY_*` tokens → Tasks 3, 8. ✅
- `WhoopRecoveryDay` type + `listWhoopRecovery`; `getWhoopConnection` reused → Task 1, 10. ✅
- `SegmentedToggle` reused (promoted per convention) → Task 2. ✅
- Tests: sidebar active, render gate, band boundaries (33/34, 66/67), missing-today, client timezone/local-dates, null→gap mapping → Tasks 1, 4, 5, 7, 8, 10. ✅
- Non-goals respected: no sleep/strain, no nav gating, no writes, no pagination, no mobile, no agent/MCP changes. ✅

**2. Placeholder scan:** No TBD/TODO/"handle edge cases" — every code step carries full code. (The one illustrative `const keys = …` line in Task 8's first test is flagged in-step for removal.)

**3. Type consistency:** `WhoopRecoveryDay` fields (`date`, `recovery_score`, `resting_heart_rate`, `hrv_rmssd_milli`) are identical across api.ts, recovery.ts, and all components. `RecoveryBand` union (`success`/`warning`/`danger`) is consistent between `recoveryBand`/`recoveryBandColor` and the band `ReferenceArea` boundaries. `RangeKey` (`"7"|"30"|"90"`) and the `SegmentedToggle<T extends string>` generic line up. Chart constants referenced in Task 8 all exist from Task 3.

## Notes for the executor
- Work in `/workspace/prog-strength-web` on branch `feat/recovery-page` (already created).
- Before each commit the Husky `pre-commit` hook runs lint-staged + `tsc --noEmit`; let it run (never `--no-verify`).
- After all tasks, run the full gate once more: `npm run lint && npm run format:check && npm run typecheck && npm run test && npm run build`.
