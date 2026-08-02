# Recovery Tile Family Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship the five recovery tiles chosen from `dx/recovery-tile` as separate, independently addable dashboard tiles — the `recovery` tile rewritten as `readiness-verdict`, plus four new catalog ids (`hrv_balance`, `morning_vitals`, `recovery_trend`, `recovery_log`) — with the API's recovery section re-gated on any family tile.

**Architecture:** `prog-strength-api` adds four `TileID` constants to `Catalog` (order is load-bearing — it fixes the web tray order) and re-gates `buildRecoverySection` on any recovery-family tile, still emitting exactly one `recovery` section key. `prog-strength-web` mirrors the catalog, builds five production tile components under `app/(app)/dashboard/_components/recovery/` sharing one tested formatting/status module, and rewires `TileCard` with five cases. No new API data, no migration, no design-system change — `scope: in-system` against v0.4; every value needed already ships in `RecoveryView`. `prog-strength-docs` flips the DX to `selected` and the SOW to `shipped`.

**Tech Stack:** Go 1.25 / chi / SQLite (API); Next.js 16 App Router, React 19, TypeScript, Tailwind v4, Vitest + Testing Library (web).

**Source spec:** `/workspace/prog-strength-docs/sows/recovery-tile-family.md` — read it before starting; its "Color logic" and "states" tables are binding. The visual spec is the DX mockup code on the un-merged `origin/dx/recovery-tile` branch of `prog-strength-web` (`app/design-explore/recovery-tile/**`) — reference only, **never merge or copy the MockCard/fixture shell**.

**Binding constraints (from the SOW, apply to every web task):**
1. All charts/leaders read `days` (date-aligned, nulls preserved), never `spark`.
2. Never recompute a server figure. Only permitted client arithmetic: signed delta of a today-value vs a server baseline, and `shortAvg − hrvAvg`.
3. The band is per-user (`balancedLow`/`balancedHigh` as received). No hardcoded "normal HRV range".
4. `days`/`baseline`/`hrv` are typed optional — guard once at the top of each component, render the calibrating state if absent, never `!`-assert.
5. Status→token mapping is single-sourced in `recovery/shared.ts`: `suppressed`→`--warning` (never danger), `elevated`→`--accent` (never a bigger green), `balanced`→`--success`, `unknown`→`--muted`. CSS vars by name, never raw hex.
6. Never bypass hooks (`--no-verify`), never add `//nolint`, never skip/weaken a test to get green.

**Branches:** create `feat/recovery-tile-family` from `main` in each of the three repos. Conventional-commit messages, lowercase subjects.

---

## Repo 1: prog-strength-api (`/workspace/prog-strength-api`)

### Task 1: Catalog — four new TileIDs

**Files:**
- Modify: `internal/dashboard/tiles.go`
- Test: `internal/dashboard/tiles_test.go`

- [ ] **Step 1: Create the branch**

```bash
cd /workspace/prog-strength-api && git checkout -b feat/recovery-tile-family
```

- [ ] **Step 2: Update the catalog tests to expect the four new ids (failing first)**

In `internal/dashboard/tiles_test.go`, replace both the `all` list (in `TestCatalog_EveryConstantAppearsExactlyOnce`) and the `want` list (in `TestCatalog_Order`) with:

```go
	all := []TileID{
		TileRunning, TileWalking, TileCycling, TileHiking, TileLifting,
		TileSteps, TileNutrition, TileBodyweight, TileBloodPressure,
		TileRecovery, TileHRVBalance, TileMorningVitals, TileRecoveryTrend, TileRecoveryLog,
		TileStreak,
	}
```

(same 15-id slice for `want` in `TestCatalog_Order`). Also extend `TestValidTileID`:

```go
	if !ValidTileID("hrv_balance") {
		t.Error("hrv_balance should be valid")
	}
	if !ValidTileID("recovery_log") {
		t.Error("recovery_log should be valid")
	}
```

- [ ] **Step 3: Run to verify it fails**

Run: `go test ./internal/dashboard/ -run TestCatalog -v`
Expected: compile error — `undefined: TileHRVBalance` etc.

- [ ] **Step 4: Add the constants and Catalog entries**

In `internal/dashboard/tiles.go`, the const block becomes (new ids immediately after `TileRecovery`, in this exact order — it fixes the web tray order):

```go
const (
	TileRunning       TileID = "running"
	TileWalking       TileID = "walking"
	TileCycling       TileID = "cycling"
	TileHiking        TileID = "hiking"
	TileLifting       TileID = "lifting"
	TileSteps         TileID = "steps"
	TileNutrition     TileID = "nutrition"
	TileBodyweight    TileID = "bodyweight"
	TileBloodPressure TileID = "blood_pressure"
	TileRecovery      TileID = "recovery"
	TileHRVBalance    TileID = "hrv_balance"
	TileMorningVitals TileID = "morning_vitals"
	TileRecoveryTrend TileID = "recovery_trend"
	TileRecoveryLog   TileID = "recovery_log"
	TileStreak        TileID = "streak"
)
```

and `Catalog` becomes:

```go
var Catalog = []TileID{
	TileRunning, TileWalking, TileCycling, TileHiking, TileLifting,
	TileSteps, TileNutrition, TileBodyweight, TileBloodPressure,
	TileRecovery, TileHRVBalance, TileMorningVitals, TileRecoveryTrend, TileRecoveryLog,
	TileStreak,
}
```

`ValidTileID`/`catalogSet` derive from `Catalog` — no edit. `defaultLayout` (`layout_resolve.go`) is deliberately untouched.

- [ ] **Step 5: Run to verify it passes**

Run: `go test ./internal/dashboard/ -run 'TestCatalog|TestValidTileID' -v`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add internal/dashboard/tiles.go internal/dashboard/tiles_test.go
git commit -m "feat(dashboard): add recovery-family tile ids to the catalog"
```

(If the repo's pre-commit hooks aren't armed on this clone, arm them first: `pre-commit install --install-hooks --hook-type pre-commit --hook-type commit-msg --hook-type pre-push`.)

### Task 2: Re-gate `buildRecoverySection` on the recovery family

**Files:**
- Modify: `internal/dashboard/handler.go:230-232`
- Test: `internal/dashboard/summary_layout_test.go`

- [ ] **Step 1: Write the failing handler tests**

Append to `internal/dashboard/summary_layout_test.go` (the file already imports `context`, `time`, `whoopconn`; `dataEnvelope`, `assertKeysPresent`, `assertKeysAbsent`, `equalStrs`, `testNow` are defined there or in `handler_test.go`):

```go
// seedWhoopConnected upserts a CONNECTED Whoop connection so
// buildRecoverySection passes its gate (a fresh Upsert sets status=connected,
// per the Repository docs).
func seedWhoopConnected(t *testing.T, rp *repos, userID string) {
	t.Helper()
	tokens := whoopconn.TokenBundle{
		AccessTokenEnc:    []byte("a"),
		AccessTokenNonce:  []byte("n"),
		RefreshTokenEnc:   []byte("r"),
		RefreshTokenNonce: []byte("n"),
		ExpiresAt:         testNow.Add(time.Hour),
	}
	if err := rp.whoopConn.Upsert(context.Background(), userID, 12345, tokens, "read:recovery", testNow); err != nil {
		t.Fatalf("whoop upsert: %v", err)
	}
}

// TestSummary_FamilyTileAlone_YieldsRecoverySection pins the SOW's re-gate: a
// layout containing ONLY hrv_balance (no recovery tile) must still produce a
// populated "recovery" section — and no "hrv_balance" key. This is the first
// place the response's section-key set is deliberately not a subset of layout.
func TestSummary_FamilyTileAlone_YieldsRecoverySection(t *testing.T) {
	r, rp, userID := newTestEnv(t)
	seedWhoopConnected(t, rp, userID)

	if err := rp.layout.Upsert(context.Background(), userID, []TileID{TileHRVBalance}); err != nil {
		t.Fatalf("layout upsert: %v", err)
	}

	layout, data := dataEnvelope(t, r, userID, "?timezone=UTC")

	if !equalStrs(layout, []string{"hrv_balance"}) {
		t.Errorf("layout = %v, want [hrv_balance]", layout)
	}
	assertKeysPresent(t, data, "recovery")
	assertKeysAbsent(t, data, "hrv_balance")
	if string(data["recovery"]) == "null" {
		t.Error("recovery = null, want a populated section for a connected user")
	}
}

// TestSummary_MultipleFamilyTiles_OneRecoverySection asserts the section is
// built and emitted exactly once under the "recovery" key no matter how many
// family tiles are on the layout — no per-tile section keys.
func TestSummary_MultipleFamilyTiles_OneRecoverySection(t *testing.T) {
	r, rp, userID := newTestEnv(t)
	seedWhoopConnected(t, rp, userID)

	ids := []TileID{TileRecovery, TileMorningVitals, TileRecoveryLog}
	if err := rp.layout.Upsert(context.Background(), userID, ids); err != nil {
		t.Fatalf("layout upsert: %v", err)
	}

	layout, data := dataEnvelope(t, r, userID, "?timezone=UTC")

	if !equalStrs(layout, []string{"recovery", "morning_vitals", "recovery_log"}) {
		t.Errorf("layout = %v, want [recovery morning_vitals recovery_log]", layout)
	}
	assertKeysPresent(t, data, "recovery")
	assertKeysAbsent(t, data, "morning_vitals", "recovery_log", "hrv_balance", "recovery_trend")
}

// TestSummary_NoFamilyTile_NoRecoverySection asserts the inverse: with no
// recovery-family tile enabled, the "recovery" key is absent entirely.
func TestSummary_NoFamilyTile_NoRecoverySection(t *testing.T) {
	r, rp, userID := newTestEnv(t)
	seedWhoopConnected(t, rp, userID)

	if err := rp.layout.Upsert(context.Background(), userID, []TileID{TileRunning, TileStreak}); err != nil {
		t.Fatalf("layout upsert: %v", err)
	}

	_, data := dataEnvelope(t, r, userID, "?timezone=UTC")

	assertKeysAbsent(t, data, "recovery", "hrv_balance", "morning_vitals", "recovery_trend", "recovery_log")
}
```

- [ ] **Step 2: Run to verify the first test fails**

Run: `go test ./internal/dashboard/ -run 'TestSummary_FamilyTile|TestSummary_MultipleFamily|TestSummary_NoFamily' -v`
Expected: `TestSummary_FamilyTileAlone_YieldsRecoverySection` FAILS (`key "recovery" absent`); the other two pass (they describe existing behavior too — that's fine, they pin it).

- [ ] **Step 3: Implement the family re-gate**

In `internal/dashboard/handler.go`, replace:

```go
	if enabled[TileRecovery] {
		out[string(TileRecovery)] = h.buildRecoverySection(ctx, r, userID, now, loc)
	}
```

with:

```go
	// Every recovery-family tile reads the ONE shared "recovery" section, so it
	// is built once when ANY family tile is enabled and emitted only under the
	// "recovery" key. This is the first place the response's section-key set
	// deliberately diverges from layout: a layout of [hrv_balance] yields a
	// "recovery" key and no "hrv_balance" key (pinned by the summary layout
	// tests). The web adapter already reads it that way.
	recoveryFamily := []TileID{
		TileRecovery, TileHRVBalance, TileMorningVitals, TileRecoveryTrend, TileRecoveryLog,
	}
	for _, id := range recoveryFamily {
		if enabled[id] {
			out[string(TileRecovery)] = h.buildRecoverySection(ctx, r, userID, now, loc)
			break
		}
	}
```

`buildRecoverySection`'s body, `defaultLayout`, and `hasConnectedWhoop` are untouched.

- [ ] **Step 4: Run to verify all pass**

Run: `go test ./internal/dashboard/ -v`
Expected: PASS (whole package).

- [ ] **Step 5: Commit**

```bash
git add internal/dashboard/handler.go internal/dashboard/summary_layout_test.go
git commit -m "feat(dashboard): gate the recovery section on any recovery-family tile"
```

### Task 3: API local CI gate

- [ ] **Step 1: Run the full local gate (same checks as CI)**

```bash
cd /workspace/prog-strength-api
go build ./... && go vet ./...
golangci-lint run --timeout=5m       # local binary is the CI-pinned v2.12.2
go mod tidy && git diff --exit-code go.mod go.sum
go test ./...
```

Expected: all green, no `go.mod`/`go.sum` drift. If lint or a test fails, fix the code (never `//nolint`), re-run, and amend/commit the fix with an appropriate conventional message.

---

## Repo 2: prog-strength-web (`/workspace/prog-strength-web`)

All components live in a new directory `app/(app)/dashboard/_components/recovery/`. `npm ci` has already been run (husky hooks armed). Test runner: `npx vitest run <file>` for focused runs.

**Ordering note:** the `TileId` union must NOT grow until Task 10, because `TileCard`'s `never` default makes an unhandled id a type error and the pre-commit hook runs `tsc --noEmit`. Tasks 4–9 build the components (they only depend on `RecoveryView`); Task 10 lands catalog + renderer + old-card deletion in one commit so every commit typechecks.

### Task 4: `recovery/shared.ts` — single-sourced formatting + status mapping

**Files:**
- Create: `app/(app)/dashboard/_components/recovery/shared.ts`
- Test: `app/(app)/dashboard/_components/recovery/shared.test.ts`

- [ ] **Step 1: Create the branch**

```bash
cd /workspace/prog-strength-web && git checkout -b feat/recovery-tile-family
```

- [ ] **Step 2: Write the failing test**

`app/(app)/dashboard/_components/recovery/shared.test.ts`:

```ts
import { describe, expect, test } from "vitest";
import { hrvStatusColor, signed, signedUnit, statusWord, trendLabel, weekday } from "./shared";

describe("hrvStatusColor — the SOW color contract", () => {
  test("suppressed maps to --warning and never --danger", () => {
    expect(hrvStatusColor("suppressed")).toBe("var(--warning)");
    expect(hrvStatusColor("suppressed")).not.toContain("danger");
  });

  test("elevated maps to --accent and never --success (unusual, not extra good)", () => {
    expect(hrvStatusColor("elevated")).toBe("var(--accent)");
    expect(hrvStatusColor("elevated")).not.toContain("success");
  });

  test("balanced maps to --success", () => {
    expect(hrvStatusColor("balanced")).toBe("var(--success)");
  });

  test("unknown maps to --muted", () => {
    expect(hrvStatusColor("unknown")).toBe("var(--muted)");
  });
});

describe("statusWord", () => {
  test("house words in sentence case", () => {
    expect(statusWord("suppressed")).toBe("Suppressed");
    expect(statusWord("balanced")).toBe("Balanced");
    expect(statusWord("elevated")).toBe("Elevated");
    expect(statusWord("unknown")).toBe("Calibrating");
  });
});

describe("trendLabel", () => {
  test("glyph + word pairs per direction", () => {
    expect(trendLabel("rising")).toEqual({ glyph: "▲", word: "rising this week" });
    expect(trendLabel("falling")).toEqual({ glyph: "▼", word: "falling this week" });
    expect(trendLabel("steady")).toEqual({ glyph: "▬", word: "steady this week" });
    expect(trendLabel("unknown")).toEqual({ glyph: "·", word: "calibrating" });
  });
});

describe("signed", () => {
  test("positive carries a plus", () => {
    expect(signed(3.2)).toBe("+3");
  });

  test("negative carries a unicode minus, not an ascii hyphen", () => {
    expect(signed(-17.2)).toBe("−17");
    expect(signed(-17.2)).not.toBe("-17");
  });

  test("zero is ±0, including a sub-half value that rounds to zero", () => {
    expect(signed(0)).toBe("±0");
    expect(signed(-0.2)).toBe("±0");
  });

  test("respects the digits argument", () => {
    expect(signed(-1.44, 1)).toBe("−1.4");
  });
});

describe("signedUnit", () => {
  test("appends the unit", () => {
    expect(signedUnit(-8.9, "ms", 1)).toBe("−8.9 ms");
    expect(signedUnit(2, "bpm")).toBe("+2 bpm");
  });
});

describe("weekday", () => {
  test("parses a local date with no timezone drift", () => {
    // Parsed as local Y/M/D parts, 2026-08-01 is a Saturday in every timezone.
    expect(weekday("2026-08-01")).toBe("Sat");
    expect(weekday("2026-07-31")).toBe("Fri");
  });

  test("malformed input degrades to an em-dash", () => {
    expect(weekday("nope")).toBe("—");
  });
});
```

- [ ] **Step 3: Run to verify it fails**

Run: `npx vitest run "app/(app)/dashboard/_components/recovery/shared.test.ts"`
Expected: FAIL — cannot resolve `./shared`.

- [ ] **Step 4: Implement `shared.ts`**

```ts
/**
 * Shared formatting and status→token mapping for the five recovery tiles.
 *
 * Single-sourced on purpose: five hand-rolled copies of the status switch is
 * exactly how `elevated` ends up green on one tile. The mapping is the SOW's
 * color contract — `suppressed` reads as warning (a low-HRV morning is
 * information, not an emergency, never danger red), `elevated` reads as accent
 * (unusual, never a bigger green), `balanced` reads as success (the ordinary
 * state), `unknown` reads as muted. Pure functions, no React; reads the v0.4
 * CSS vars by name, never a raw hex.
 */

import type { RecoveryHrvStatus, RecoveryTrendDirection } from "@/lib/dashboard";

/** Map an HRV balance status to the CSS var that carries its color. */
export function hrvStatusColor(status: RecoveryHrvStatus): string {
  switch (status) {
    case "suppressed":
      return "var(--warning)";
    case "elevated":
      return "var(--accent)";
    case "balanced":
      return "var(--success)";
    default:
      return "var(--muted)";
  }
}

/** The house word for each status (sentence case). */
export function statusWord(status: RecoveryHrvStatus): string {
  switch (status) {
    case "suppressed":
      return "Suppressed";
    case "elevated":
      return "Elevated";
    case "balanced":
      return "Balanced";
    default:
      return "Calibrating";
  }
}

/** A small glyph + word pair for a trend direction. */
export function trendLabel(trend: RecoveryTrendDirection): { glyph: string; word: string } {
  switch (trend) {
    case "rising":
      return { glyph: "▲", word: "rising this week" };
    case "falling":
      return { glyph: "▼", word: "falling this week" };
    case "steady":
      return { glyph: "▬", word: "steady this week" };
    default:
      return { glyph: "·", word: "calibrating" };
  }
}

/** Signed delta string, e.g. +3 / −17 / ±0, always with a unicode minus. */
export function signed(n: number, digits = 0): string {
  const r = digits > 0 ? Number(n.toFixed(digits)) : Math.round(n);
  if (r === 0) return "±0";
  return r > 0 ? `+${r}` : `−${Math.abs(r)}`;
}

/** Signed delta with a unit suffix, e.g. "−17 ms". */
export function signedUnit(n: number, unit: string, digits = 0): string {
  return `${signed(n, digits)} ${unit}`;
}

/** "2026-08-01" → "Sat". Parsed as local date parts — no timezone drift. */
export function weekday(iso: string): string {
  const [y, m, d] = iso.split("-").map(Number);
  if (!y || !m || !d) return "—";
  return new Date(y, m - 1, d).toLocaleDateString("en-US", { weekday: "short" });
}
```

- [ ] **Step 5: Run to verify it passes**

Run: `npx vitest run "app/(app)/dashboard/_components/recovery/shared.test.ts"`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add "app/(app)/dashboard/_components/recovery/shared.ts" "app/(app)/dashboard/_components/recovery/shared.test.ts"
git commit -m "feat(dashboard): shared status mapping and formatting for recovery tiles"
```

### Task 5: Test fixtures + connect card + `readiness-verdict`

**Files:**
- Create: `app/(app)/dashboard/_components/recovery/fixtures.ts`
- Create: `app/(app)/dashboard/_components/recovery/connect-card.tsx`
- Create: `app/(app)/dashboard/_components/recovery/readiness-verdict.tsx`
- Test: `app/(app)/dashboard/_components/recovery/readiness-verdict.test.tsx`

- [ ] **Step 1: Create the shared test fixtures**

`app/(app)/dashboard/_components/recovery/fixtures.ts`:

```ts
/**
 * Test fixtures for the recovery tile family — the four states the DX
 * enumerated (suppressed / balanced / calibrating / no-reading-yet), each with
 * a full 31-day date-aligned window and an interior all-null day (2026-07-29,
 * index 27) so the gap contract is exercised by default. Values mirror the
 * DX's headline fixture: baseline 91.2 ± 12.6 ms, band 78.6–103.8. Test-only;
 * never imported by production code.
 */

import type { RecoveryBaselineView, RecoveryDayPoint, RecoveryView } from "@/lib/dashboard";

/** The fixture "today" — the last of the 31 window dates. */
export const FIXTURE_TODAY = "2026-08-01";

// Index 27 (2026-07-29) is the interior gap; index 30 is today.
const HRV_SERIES: (number | null)[] = [
  95, 88, 102, 91, 84, 97, 90, 79, 105, 93, 87, 99, 92, 81, 96, 89, 101, 94, 85, 98, 90, 83, 100,
  92, 89, 95, 86, null, 86, 81, 74,
];

function isoDate(offset: number): string {
  const d = new Date(2026, 6, 2 + offset); // 2026-07-02 + offset, local
  const mm = String(d.getMonth() + 1).padStart(2, "0");
  const dd = String(d.getDate()).padStart(2, "0");
  return `${d.getFullYear()}-${mm}-${dd}`;
}

/** Build the 31-day window from an HRV series; a null HRV ⇒ a fully-null day. */
export function makeDays(hrv: (number | null)[] = HRV_SERIES): RecoveryDayPoint[] {
  return hrv.map((v, i) => ({
    date: isoDate(i),
    hrv: v,
    restingHr: v === null ? null : 49 + (i % 6),
    recoveryScore: v === null ? null : 48 + (i % 30),
  }));
}

function baseline(): RecoveryBaselineView {
  return {
    windowDays: 30,
    restingHrAvg: 52.4,
    restingHrDays: 27,
    hrvAvg: 91.2,
    hrvStdDev: 12.6,
    hrvDays: 26,
    recoveryScoreAvg: 68.1,
    recoveryScoreDays: 27,
  };
}

/** Calibrated + suppressed — the DX's headline fixture. Today 74 ms, z −1.37. */
export function suppressedView(): RecoveryView {
  const days = makeDays();
  days[days.length - 1] = { date: FIXTURE_TODAY, hrv: 74, restingHr: 51, recoveryScore: 58 };
  return {
    restingToday: 51,
    recoveryScore: 58,
    hrvToday: 74,
    spark: [53, 52, 51, 51],
    days,
    baseline: baseline(),
    hrv: {
      status: "suppressed",
      balancedLow: 78.6,
      balancedHigh: 103.8,
      zScore: -1.37,
      trend: "falling",
      shortAvg: 82.3,
    },
  };
}

/** Calibrated + balanced — the boring good day. Today 94 ms, z +0.22. */
export function balancedView(): RecoveryView {
  const days = makeDays();
  days[days.length - 1] = { date: FIXTURE_TODAY, hrv: 94, restingHr: 52, recoveryScore: 71 };
  return {
    restingToday: 52,
    recoveryScore: 71,
    hrvToday: 94,
    spark: [53, 52, 51, 52],
    days,
    baseline: baseline(),
    hrv: {
      status: "balanced",
      balancedLow: 78.6,
      balancedHigh: 103.8,
      zScore: 0.22,
      trend: "steady",
      shortAvg: 92.8,
    },
  };
}

/** Calibrating — 9 of 14 nights in; averages, bounds, z all null. */
export function calibratingView(): RecoveryView {
  const days = makeDays(HRV_SERIES.map((v, i) => (i < 21 ? null : v)));
  return {
    restingToday: 51,
    recoveryScore: 58,
    hrvToday: 74,
    spark: [53, 52, 51],
    days,
    baseline: {
      windowDays: 30,
      restingHrAvg: null,
      restingHrDays: 9,
      hrvAvg: null,
      hrvStdDev: null,
      hrvDays: 9,
      recoveryScoreAvg: null,
      recoveryScoreDays: 9,
    },
    hrv: {
      status: "unknown",
      balancedLow: null,
      balancedHigh: null,
      zScore: null,
      trend: "unknown",
      shortAvg: null,
    },
  };
}

/** No reading yet today — 7am before the webhook; baseline and trend intact. */
export function noReadingView(): RecoveryView {
  const days = makeDays();
  days[days.length - 1] = { date: FIXTURE_TODAY, hrv: null, restingHr: null, recoveryScore: null };
  return {
    restingToday: null,
    recoveryScore: null,
    hrvToday: null,
    spark: [53, 52, 51],
    days,
    baseline: baseline(),
    hrv: {
      status: "unknown",
      balancedLow: 78.6,
      balancedHigh: 103.8,
      zScore: null,
      trend: "falling",
      shortAvg: 82.3,
    },
  };
}

/** A legacy payload with no derived blocks — exercises the top-of-card guard. */
export function legacyView(): RecoveryView {
  return { restingToday: 51, recoveryScore: 58, spark: [53, 52, 51] };
}
```

- [ ] **Step 2: Create the shared connect card**

`app/(app)/dashboard/_components/recovery/connect-card.tsx`:

```tsx
/**
 * RecoveryConnectCard — the shared `present: false` body for every
 * recovery-family tile. One empty grammar, five different headings: each tile
 * passes its own catalog title so an unconnected user still sees which tile
 * they added, over the same connect CTA. Generalizes the old
 * RecoveryCardEmpty (whoop-card.tsx), which hardcoded the "Recovery" title.
 */

import { MiniCard, MiniCardEmpty } from "../mini-card";

export function RecoveryConnectCard({ title, href }: { title: string; href: string }) {
  return (
    <MiniCard title={title} href={href}>
      <MiniCardEmpty cta="Connect Whoop to see recovery" />
    </MiniCard>
  );
}
```

- [ ] **Step 3: Write the failing test**

`app/(app)/dashboard/_components/recovery/readiness-verdict.test.tsx`:

```tsx
/// <reference types="vitest/globals" />

import { render, screen } from "@testing-library/react";
import { RecoveryConnectCard } from "./connect-card";
import { balancedView, calibratingView, legacyView, noReadingView, suppressedView } from "./fixtures";
import { ReadinessVerdictCard } from "./readiness-verdict";

const HREF = "/recovery";

describe("ReadinessVerdictCard", () => {
  it("suppressed: heroes the verdict sentence with the status word colored", () => {
    render(<ReadinessVerdictCard section={suppressedView()} href={HREF} />);
    const word = screen.getByText("Suppressed");
    expect(word).toBeInTheDocument();
    expect(word).toHaveStyle({ color: "var(--warning)" });
    // (74 − 91.2) / 91.2 → −19% vs baseline, spelled out in the sentence.
    expect(screen.getByText(/19% below/)).toBeInTheDocument();
  });

  it("suppressed: three contributor rows carry values and baseline-delta chips", () => {
    render(<ReadinessVerdictCard section={suppressedView()} href={HREF} />);
    expect(screen.getByText("Resting HR")).toBeInTheDocument();
    expect(screen.getByText("−1.4")).toBeInTheDocument(); // 51 vs 52.4
    expect(screen.getByText("−10")).toBeInTheDocument(); // 58 vs 68.1
    expect(screen.getByText("−17")).toBeInTheDocument(); // 74 vs 91.2
  });

  it("balanced: the ordinary day reads calm, with the verdict in success", () => {
    render(<ReadinessVerdictCard section={balancedView()} href={HREF} />);
    const word = screen.getByText("Balanced");
    expect(word).toHaveStyle({ color: "var(--success)" });
    expect(screen.getByText(/3% above/)).toBeInTheDocument();
  });

  it("calibrating: renders honest n-of-14 progress, not an empty frame", () => {
    render(<ReadinessVerdictCard section={calibratingView()} href={HREF} />);
    expect(screen.getByText(/9\s*of 14 nights/)).toBeInTheDocument();
    expect(screen.queryByText("NaN", { exact: false })).not.toBeInTheDocument();
  });

  it("no reading yet: a full true sentence from server baselines, never em-dashes", () => {
    render(<ReadinessVerdictCard section={noReadingView()} href={HREF} />);
    expect(screen.getByText(/No reading yet today/)).toBeInTheDocument();
    expect(screen.getByText("52.4")).toBeInTheDocument();
    expect(screen.getByText(/trending falling/)).toBeInTheDocument();
    // Yesterday is never promoted into today: no today-value figures render.
    expect(screen.queryByText("74")).not.toBeInTheDocument();
  });

  it("guards absent derived blocks with a calibrating body, never a crash", () => {
    render(<ReadinessVerdictCard section={legacyView()} href={HREF} />);
    expect(screen.getByText("Recovery is calibrating.")).toBeInTheDocument();
  });

  it("keeps the whole-card link into the deep page", () => {
    const { container } = render(<ReadinessVerdictCard section={suppressedView()} href={HREF} />);
    expect(container.querySelector("a")).toHaveAttribute("href", HREF);
  });
});

describe("RecoveryConnectCard", () => {
  it("renders the tile's own title over the shared connect CTA", () => {
    const { container } = render(<RecoveryConnectCard title="HRV Balance" href={HREF} />);
    expect(screen.getByRole("heading", { name: "HRV Balance" })).toBeInTheDocument();
    expect(screen.getByText("Connect Whoop to see recovery")).toBeInTheDocument();
    expect(container.querySelector("a")).toHaveAttribute("href", HREF);
  });
});
```

- [ ] **Step 4: Run to verify it fails**

Run: `npx vitest run "app/(app)/dashboard/_components/recovery/readiness-verdict.test.tsx"`
Expected: FAIL — cannot resolve `./readiness-verdict`.

- [ ] **Step 5: Implement `readiness-verdict.tsx`**

```tsx
/**
 * ReadinessVerdictCard — the rewritten `recovery` tile ("Recovery").
 *
 * Heroes the SENTENCE: a verdict line in the house voice over three quiet
 * contributor rows (score / resting HR / HRV), each with its value and a muted
 * baseline-delta chip. No chart, no giant numeral. The status color lands on
 * exactly ONE word (the verdict); the score row stays neutral — painting the
 * score's own green/yellow/red band here too would be the two-traffic-lights
 * failure the DX names.
 *
 * The no-reading branch is where this idiom earns its slot: a full true
 * sentence from the server baselines instead of em-dashes, and yesterday is
 * never promoted into today. The only client-side arithmetic is a signed
 * delta of a today-value against a server baseline (and its percentage) —
 * never a re-averaged series.
 */

import type { RecoveryView } from "@/lib/dashboard";
import { MiniCard } from "../mini-card";
import { hrvStatusColor, signed, statusWord } from "./shared";

const TITLE = "Recovery";

export function ReadinessVerdictCard({ section, href }: { section: RecoveryView; href: string }) {
  const { restingToday, recoveryScore, baseline, hrv } = section;
  const hrvToday = section.hrvToday ?? null;

  // Guard once: the derived blocks are typed optional. Absent blocks read as
  // calibrating — never a `!`-assert, never an empty frame.
  if (!baseline || !hrv) {
    return (
      <MiniCard title={TITLE} href={href}>
        <p className="text-base leading-relaxed text-[var(--muted)]">Recovery is calibrating.</p>
      </MiniCard>
    );
  }

  const calibrating = baseline.hrvAvg === null;
  const noReading = hrvToday === null && recoveryScore === null && restingToday === null;

  return (
    <MiniCard title={TITLE} href={href}>
      <Verdict calibrating={calibrating} noReading={noReading} view={section} />
      <div className="mt-1 flex flex-col divide-y divide-[var(--border)]">
        <Contributor
          label="Recovery"
          value={recoveryScore}
          unit=""
          baseline={baseline.recoveryScoreAvg}
        />
        <Contributor
          label="Resting HR"
          value={restingToday}
          unit="bpm"
          baseline={baseline.restingHrAvg}
          digits={1}
        />
        <Contributor label="HRV" value={hrvToday} unit="ms" baseline={baseline.hrvAvg} />
      </div>
    </MiniCard>
  );
}

/** The verdict line — the hero. Exactly one colored word. */
function Verdict({
  calibrating,
  noReading,
  view,
}: {
  calibrating: boolean;
  noReading: boolean;
  view: RecoveryView;
}) {
  const { baseline, hrv } = view;
  const hrvToday = view.hrvToday ?? null;
  if (!baseline || !hrv) return null;

  if (calibrating) {
    return (
      <p className="text-lg font-medium leading-snug text-[var(--foreground)]">
        <span style={{ color: hrvStatusColor("unknown") }}>Calibrating</span> — {baseline.hrvDays}{" "}
        of 14 nights in. Your baseline lands in a few more mornings.
      </p>
    );
  }

  if (noReading) {
    // A full, true sentence — the baselines are real and printable.
    return (
      <p className="text-lg font-medium leading-snug text-[var(--foreground)]">
        No reading yet today. Your 30-day baseline is{" "}
        <span className="font-mono tabular-nums">
          {baseline.recoveryScoreAvg !== null ? Math.round(baseline.recoveryScoreAvg) : "—"}
        </span>{" "}
        ·{" "}
        <span className="font-mono tabular-nums">
          {baseline.restingHrAvg !== null ? baseline.restingHrAvg.toFixed(1) : "—"} bpm
        </span>{" "}
        ·{" "}
        <span className="font-mono tabular-nums">
          {baseline.hrvAvg !== null ? Math.round(baseline.hrvAvg) : "—"} ms
        </span>
        , trending {hrv.trend === "unknown" ? "flat" : hrv.trend}.
      </p>
    );
  }

  // Calibrated with a reading: the verdict word plus one % figure vs baseline
  // (a signed delta of two server figures — hrvToday vs hrvAvg).
  const pct =
    hrvToday !== null && baseline.hrvAvg
      ? Math.round(((hrvToday - baseline.hrvAvg) / baseline.hrvAvg) * 100)
      : null;
  const tail =
    pct === null
      ? ""
      : pct <= -3
        ? `${Math.abs(pct)}% below your 30-day baseline`
        : pct >= 3
          ? `${pct}% above your 30-day baseline`
          : "right around your 30-day baseline";

  return (
    <p className="text-lg font-medium leading-snug text-[var(--foreground)]">
      <span style={{ color: hrvStatusColor(hrv.status) }}>{statusWord(hrv.status)}</span>
      {tail ? <> — HRV is {tail}.</> : "."}
    </p>
  );
}

/** One contributor row: label, value, and a muted baseline-delta chip. */
function Contributor({
  label,
  value,
  unit,
  baseline,
  digits = 0,
}: {
  label: string;
  value: number | null;
  unit: string;
  baseline: number | null;
  digits?: number;
}) {
  const delta = value !== null && baseline !== null ? value - baseline : null;
  return (
    <div className="flex items-center justify-between py-1.5">
      <span className="text-xs text-[var(--muted)]">{label}</span>
      <div className="flex items-baseline gap-2">
        <span className="font-mono text-sm tabular-nums text-[var(--foreground)]">
          {value !== null ? value : "—"}
          {value !== null && unit ? <span className="text-[var(--muted)]"> {unit}</span> : null}
        </span>
        <span className="rounded-full bg-[var(--surface-2)] px-1.5 py-0.5 font-mono text-[10px] tabular-nums text-[var(--muted)]">
          {delta !== null ? signed(delta, digits) : "vs base —"}
        </span>
      </div>
    </div>
  );
}
```

- [ ] **Step 6: Run to verify it passes**

Run: `npx vitest run "app/(app)/dashboard/_components/recovery/readiness-verdict.test.tsx"`
Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add "app/(app)/dashboard/_components/recovery/fixtures.ts" "app/(app)/dashboard/_components/recovery/connect-card.tsx" "app/(app)/dashboard/_components/recovery/readiness-verdict.tsx" "app/(app)/dashboard/_components/recovery/readiness-verdict.test.tsx"
git commit -m "feat(dashboard): readiness-verdict recovery card and shared connect card"
```

### Task 6: `balance-band` — HRV Balance

**Files:**
- Create: `app/(app)/dashboard/_components/recovery/balance-band.tsx`
- Test: `app/(app)/dashboard/_components/recovery/balance-band.test.tsx`

- [ ] **Step 1: Write the failing test**

`app/(app)/dashboard/_components/recovery/balance-band.test.tsx`:

```tsx
/// <reference types="vitest/globals" />

import { render, screen } from "@testing-library/react";
import { HrvBalanceCard } from "./balance-band";
import { balancedView, calibratingView, noReadingView, suppressedView } from "./fixtures";

const HREF = "/recovery";

describe("HrvBalanceCard", () => {
  it("suppressed: band rect at server bounds, baseline centre line, today in warning", () => {
    const { container } = render(<HrvBalanceCard section={suppressedView()} href={HREF} />);
    expect(screen.getByText("74")).toBeInTheDocument();
    expect(screen.getByText("Suppressed")).toBeInTheDocument();
    // The band — one rect filled with the desaturated success token.
    const rect = container.querySelector("rect");
    expect(rect).not.toBeNull();
    expect(rect?.getAttribute("fill")).toBe("var(--success)");
    // Dashed baseline centre line.
    expect(container.querySelector("line")).not.toBeNull();
    // Today's point carries the status color.
    const circle = container.querySelector("circle");
    expect(circle?.getAttribute("fill")).toBe("var(--warning)");
    // Server bounds spelled out, rounded: 78.6–103.8 → 79–104.
    expect(screen.getAllByText(/79–104 ms/).length).toBeGreaterThan(0);
  });

  it("breaks the polyline around the interior gap instead of interpolating", () => {
    const { container } = render(<HrvBalanceCard section={suppressedView()} href={HREF} />);
    // One null at index 27 splits the 31-day series into exactly two segments.
    expect(container.querySelectorAll("polyline")).toHaveLength(2);
  });

  it("balanced: today's point renders in success", () => {
    const { container } = render(<HrvBalanceCard section={balancedView()} href={HREF} />);
    expect(container.querySelector("circle")?.getAttribute("fill")).toBe("var(--success)");
  });

  it("calibrating: honest progress, no band and no chart frame", () => {
    const { container } = render(<HrvBalanceCard section={calibratingView()} href={HREF} />);
    expect(screen.getByText(/9 of 14/)).toBeInTheDocument();
    expect(container.querySelector("svg")).toBeNull();
    expect(container.querySelector("rect")).toBeNull();
  });

  it("no reading yet: prints the band bounds and still draws the chart, no today point", () => {
    const { container } = render(<HrvBalanceCard section={noReadingView()} href={HREF} />);
    expect(screen.getByText(/No reading yet/)).toBeInTheDocument();
    expect(container.querySelector("rect")).not.toBeNull();
    expect(container.querySelector("circle")).toBeNull();
  });
});
```

- [ ] **Step 2: Run to verify it fails**

Run: `npx vitest run "app/(app)/dashboard/_components/recovery/balance-band.test.tsx"`
Expected: FAIL — cannot resolve `./balance-band`.

- [ ] **Step 3: Implement `balance-band.tsx`**

```tsx
/**
 * HrvBalanceCard — the `hrv_balance` tile ("HRV Balance").
 *
 * Heroes the BAND: the athlete's own balanced zone (server `balancedLow`–
 * `balancedHigh`, baseline ± 1 of their own SDs — never a generic "normal HRV"
 * range) drawn as a filled horizontal zone behind the 30-day series, so "am I
 * inside my normal range?" is a spatial question. The HRV axis carries ALL the
 * color on this tile: band fill in desaturated success, today's point in the
 * status color; the series stays muted ink and the recovery score is absent.
 *
 * The polyline splits into gap-free segments — a null day breaks the line
 * rather than interpolating across it. Bounds null (calibrating) renders the
 * honest n-of-14 progress state: no band drawn at zero, no empty chart frame.
 */

import type { RecoveryView } from "@/lib/dashboard";
import { MiniCard } from "../mini-card";
import { hrvStatusColor, statusWord } from "./shared";

const TITLE = "HRV Balance";
const W = 260;
const H = 92;
const PAD = 4;

export function HrvBalanceCard({ section, href }: { section: RecoveryView; href: string }) {
  const { days, baseline, hrv } = section;

  // Guard once — calibrating until the derived blocks and both bounds exist.
  if (!days || !baseline || !hrv || hrv.balancedLow === null || hrv.balancedHigh === null) {
    return (
      <MiniCard title={TITLE} href={href}>
        <Calibrating nights={baseline?.hrvDays ?? 0} />
      </MiniCard>
    );
  }

  const low = hrv.balancedLow;
  const high = hrv.balancedHigh;
  const base = baseline.hrvAvg;
  const hrvSeries = days.map((d) => d.hrv);
  const known = hrvSeries.filter((v): v is number => v !== null);

  // Scale spans the union of the series and the band, with headroom.
  const lo = Math.min(low, ...known) - 4;
  const hi = Math.max(high, ...known) + 4;
  const span = hi - lo || 1;
  const y = (v: number) => PAD + (1 - (v - lo) / span) * (H - PAD * 2);
  const x = (i: number) => (i / Math.max(1, days.length - 1)) * W;

  // Break the polyline into gap-free segments — never interpolate a null.
  const segments: string[] = [];
  let run: string[] = [];
  hrvSeries.forEach((v, i) => {
    if (v === null) {
      if (run.length > 1) segments.push(run.join(" "));
      run = [];
    } else {
      run.push(`${x(i).toFixed(1)},${y(v).toFixed(1)}`);
    }
  });
  if (run.length > 1) segments.push(run.join(" "));

  const todayIdx = days.length - 1;
  const todayVal = hrvSeries[todayIdx];
  const statusColor = hrvStatusColor(hrv.status);

  return (
    <MiniCard title={TITLE} href={href}>
      {/* Modest headline: today's ms + status word, or the bounds when absent. */}
      <div className="flex items-baseline gap-2">
        {todayVal !== null ? (
          <>
            <span className="font-mono text-xl font-semibold tracking-tight tabular-nums text-[var(--foreground)]">
              {todayVal}
              <span className="ml-0.5 text-xs font-medium text-[var(--muted)]">ms</span>
            </span>
            <span className="text-sm font-medium" style={{ color: statusColor }}>
              {statusWord(hrv.status)}
            </span>
          </>
        ) : (
          <span className="text-sm text-[var(--muted)]">
            No reading yet · band{" "}
            <span className="font-mono tabular-nums text-[var(--foreground)]">
              {Math.round(low)}–{Math.round(high)} ms
            </span>
          </span>
        )}
      </div>

      {/* The chart owns the card. */}
      <svg
        viewBox={`0 0 ${W} ${H}`}
        preserveAspectRatio="none"
        className="h-[92px] w-full"
        role="img"
        aria-label="Thirty-day HRV against your balanced band"
      >
        <rect
          x={0}
          y={y(high)}
          width={W}
          height={Math.max(0, y(low) - y(high))}
          fill="var(--success)"
          fillOpacity={0.12}
        />
        {base !== null && (
          <line
            x1={0}
            x2={W}
            y1={y(base)}
            y2={y(base)}
            stroke="var(--success)"
            strokeOpacity={0.45}
            strokeWidth={1}
            strokeDasharray="3 3"
            vectorEffect="non-scaling-stroke"
          />
        )}
        {segments.map((pts, i) => (
          <polyline
            key={i}
            points={pts}
            fill="none"
            stroke="var(--muted)"
            strokeWidth={1.5}
            strokeLinecap="round"
            strokeLinejoin="round"
            vectorEffect="non-scaling-stroke"
          />
        ))}
        {todayVal !== null && (
          <circle
            cx={x(todayIdx)}
            cy={y(todayVal)}
            r={3.5}
            fill={statusColor}
            stroke="var(--surface)"
            strokeWidth={1.5}
          />
        )}
      </svg>

      {/* Neutral caption — the bounds spelled out; no second color axis. */}
      <p className="text-[11px] text-[var(--faint)]">
        Your balanced range{" "}
        <span className="font-mono tabular-nums text-[var(--muted)]">
          {Math.round(low)}–{Math.round(high)} ms
        </span>{" "}
        · baseline{" "}
        <span className="font-mono tabular-nums text-[var(--muted)]">
          {base !== null ? Math.round(base) : "—"} ms
        </span>
      </p>
    </MiniCard>
  );
}

/** New-user state — no band to draw yet. Honest progress toward 14 nights. */
function Calibrating({ nights }: { nights: number }) {
  return (
    <div className="flex flex-col gap-2 py-1">
      <span className="text-sm font-medium text-[var(--muted)]">Calibrating your band</span>
      <div className="h-1.5 w-full overflow-hidden rounded-full bg-[var(--surface-2)]">
        <div
          className="h-full rounded-full bg-[var(--accent)]"
          style={{ width: `${Math.min(100, (nights / 14) * 100)}%` }}
        />
      </div>
      <p className="text-[11px] text-[var(--faint)]">
        <span className="font-mono tabular-nums text-[var(--muted)]">{nights} of 14</span> nights ·
        your normal range appears once Whoop knows your spread
      </p>
    </div>
  );
}
```

- [ ] **Step 4: Run to verify it passes**

Run: `npx vitest run "app/(app)/dashboard/_components/recovery/balance-band.test.tsx"`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add "app/(app)/dashboard/_components/recovery/balance-band.tsx" "app/(app)/dashboard/_components/recovery/balance-band.test.tsx"
git commit -m "feat(dashboard): hrv balance band tile"
```

### Task 7: `three-dial-vitals` — Morning Vitals

**Files:**
- Create: `app/(app)/dashboard/_components/recovery/three-dial-vitals.tsx`
- Test: `app/(app)/dashboard/_components/recovery/three-dial-vitals.test.tsx`

- [ ] **Step 1: Write the failing test**

`app/(app)/dashboard/_components/recovery/three-dial-vitals.test.tsx`:

```tsx
/// <reference types="vitest/globals" />

import { render, screen } from "@testing-library/react";
import { balancedView, calibratingView, noReadingView, suppressedView } from "./fixtures";
import { MorningVitalsCard } from "./three-dial-vitals";

const HREF = "/recovery";

describe("MorningVitalsCard", () => {
  it("suppressed: three equal cells with values and vs-30d delta captions", () => {
    render(<MorningVitalsCard section={suppressedView()} href={HREF} />);
    expect(screen.getByText("58")).toBeInTheDocument();
    expect(screen.getByText("51")).toBeInTheDocument();
    expect(screen.getByText(/74/)).toBeInTheDocument();
    expect(screen.getByText("vs 30d −10")).toBeInTheDocument();
    expect(screen.getByText("vs 30d −1.4")).toBeInTheDocument();
    expect(screen.getByText("vs 30d −17")).toBeInTheDocument();
  });

  it("exactly one status dot, beside HRV, in the status color", () => {
    const { container } = render(<MorningVitalsCard section={suppressedView()} href={HREF} />);
    const dots = container.querySelectorAll("span.h-1\\.5.w-1\\.5");
    expect(dots).toHaveLength(1);
    expect((dots[0] as HTMLElement).style.backgroundColor).toBe("var(--warning)");
  });

  it("balanced: the dot reads success and stays the only color", () => {
    const { container } = render(<MorningVitalsCard section={balancedView()} href={HREF} />);
    const dots = container.querySelectorAll("span.h-1\\.5.w-1\\.5");
    expect((dots[0] as HTMLElement).style.backgroundColor).toBe("var(--success)");
  });

  it("no reading yet: promotes the 30-day averages to primary figures", () => {
    render(<MorningVitalsCard section={noReadingView()} href={HREF} />);
    expect(screen.getByText(/No reading yet today/)).toBeInTheDocument();
    expect(screen.getByText("68")).toBeInTheDocument(); // recoveryScoreAvg 68.1
    expect(screen.getByText("52.4")).toBeInTheDocument(); // restingHrAvg
    expect(screen.getByText(/91/)).toBeInTheDocument(); // hrvAvg 91.2
    expect(screen.getAllByText("30d avg")).toHaveLength(3);
  });

  it("calibrating: cells show today's readings with calibrating captions, no NaN", () => {
    render(<MorningVitalsCard section={calibratingView()} href={HREF} />);
    expect(screen.getAllByText("calibrating")).toHaveLength(3);
    expect(screen.queryByText(/NaN/)).not.toBeInTheDocument();
  });
});
```

- [ ] **Step 2: Run to verify it fails**

Run: `npx vitest run "app/(app)/dashboard/_components/recovery/three-dial-vitals.test.tsx"`
Expected: FAIL — cannot resolve `./three-dial-vitals`.

- [ ] **Step 3: Implement `three-dial-vitals.tsx`**

```tsx
/**
 * MorningVitalsCard — the `morning_vitals` tile ("Morning Vitals").
 *
 * Heroes the THREE NUMBERS AS EQUALS: score, resting HR, and HRV as three
 * equal tabular cells, each over a small "vs 30d ±n" baseline-delta caption.
 * No hero figure — the grid is the hierarchy. Exactly ONE status dot, beside
 * HRV, is the only interpretation offered; neither three-state color axis is
 * painted at strength.
 *
 * In the no-reading state each cell promotes its server baseline to the
 * primary figure ("30d avg" caption) so the card is three facts, not three
 * em-dashes. Deltas are today-value minus server baseline — nothing is
 * re-averaged client-side.
 */

import type { RecoveryView } from "@/lib/dashboard";
import { MiniCard } from "../mini-card";
import { hrvStatusColor, signed } from "./shared";

const TITLE = "Morning Vitals";

export function MorningVitalsCard({ section, href }: { section: RecoveryView; href: string }) {
  const { restingToday, recoveryScore, baseline, hrv } = section;
  const hrvToday = section.hrvToday ?? null;

  if (!baseline || !hrv) {
    return (
      <MiniCard title={TITLE} href={href}>
        <p className="text-sm text-[var(--muted)]">Vitals are calibrating.</p>
      </MiniCard>
    );
  }

  const noReading = recoveryScore === null && restingToday === null && hrvToday === null;

  return (
    <MiniCard title={TITLE} href={href}>
      {noReading && (
        <p className="text-[11px] text-[var(--muted)]">
          No reading yet today · your 30-day averages
        </p>
      )}
      <div className="grid grid-cols-3 divide-x divide-[var(--border)]">
        <Cell
          label="Score"
          value={recoveryScore}
          baseline={baseline.recoveryScoreAvg}
          noReading={noReading}
        />
        <Cell
          label="Rest HR"
          unit="bpm"
          value={restingToday}
          baseline={baseline.restingHrAvg}
          digits={1}
          noReading={noReading}
        />
        <Cell
          label="HRV"
          unit="ms"
          value={hrvToday}
          baseline={baseline.hrvAvg}
          noReading={noReading}
          dot={hrvStatusColor(hrv.status)}
        />
      </div>
    </MiniCard>
  );
}

/** One equal stat cell. Uniform weight — the grid, not any one figure, leads. */
function Cell({
  label,
  value,
  baseline,
  unit,
  digits = 0,
  noReading,
  dot,
}: {
  label: string;
  value: number | null;
  baseline: number | null;
  unit?: string;
  digits?: number;
  noReading: boolean;
  dot?: string;
}) {
  const calibrating = baseline === null;
  // In the no-reading state the baseline is promoted to the primary figure so
  // the cell stays a fact; otherwise today's value leads.
  const primary = noReading ? baseline : value;
  const delta = value !== null && baseline !== null ? value - baseline : null;

  const primaryText =
    primary === null
      ? "—"
      : digits > 0 && !Number.isInteger(primary)
        ? primary.toFixed(digits)
        : String(Math.round(primary));

  const caption = calibrating
    ? "calibrating"
    : noReading
      ? "30d avg"
      : delta !== null
        ? `vs 30d ${signed(delta, digits)}`
        : "vs 30d —";

  return (
    <div className="flex flex-col items-center gap-0.5 px-1 py-1 text-center">
      <div className="flex items-center gap-1">
        <span className="text-[10px] uppercase tracking-wide text-[var(--faint)]">{label}</span>
        {dot && (
          <span
            aria-hidden="true"
            className="h-1.5 w-1.5 rounded-full"
            style={{ backgroundColor: dot }}
          />
        )}
      </div>
      <span className="font-mono text-xl font-medium tracking-tight tabular-nums text-[var(--foreground)]">
        {primaryText}
        {primary !== null && unit && (
          <span className="ml-0.5 text-[10px] font-medium text-[var(--muted)]">{unit}</span>
        )}
      </span>
      <span className="font-mono text-[10px] tabular-nums text-[var(--muted)]">{caption}</span>
    </div>
  );
}
```

- [ ] **Step 4: Run to verify it passes**

Run: `npx vitest run "app/(app)/dashboard/_components/recovery/three-dial-vitals.test.tsx"`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add "app/(app)/dashboard/_components/recovery/three-dial-vitals.tsx" "app/(app)/dashboard/_components/recovery/three-dial-vitals.test.tsx"
git commit -m "feat(dashboard): morning vitals tile"
```

### Task 8: `trend-rail` — Recovery Trend

**Files:**
- Create: `app/(app)/dashboard/_components/recovery/trend-rail.tsx`
- Test: `app/(app)/dashboard/_components/recovery/trend-rail.test.tsx`

- [ ] **Step 1: Write the failing test**

`app/(app)/dashboard/_components/recovery/trend-rail.test.tsx`:

```tsx
/// <reference types="vitest/globals" />

import { render, screen } from "@testing-library/react";
import type { RecoveryView } from "@/lib/dashboard";
import { calibratingView, FIXTURE_TODAY, noReadingView, suppressedView } from "./fixtures";
import { TrendRailCard } from "./trend-rail";

const HREF = "/recovery";

function railMarks(container: HTMLElement): HTMLElement[] {
  return Array.from(container.querySelectorAll('[role="img"] > span')) as HTMLElement[];
}

describe("TrendRailCard", () => {
  it("heroes the week-vs-baseline delta, derived from server figures only", () => {
    render(<TrendRailCard section={suppressedView()} href={HREF} />);
    // shortAvg 82.3 vs hrvAvg 91.2 → −8.9 ms → −10% (rounded).
    expect(screen.getByText(/10%/)).toBeInTheDocument();
    expect(screen.getByText("−8.9 ms")).toBeInTheDocument();
    expect(screen.getByText(/falling this week/)).toBeInTheDocument();
  });

  it("delta figure carries the status color", () => {
    render(<TrendRailCard section={suppressedView()} href={HREF} />);
    expect(screen.getByText(/10%/)).toHaveStyle({ color: "var(--warning)" });
  });

  it("renders one mark per day with the gap left blank", () => {
    const { container } = render(<TrendRailCard section={suppressedView()} href={HREF} />);
    const marks = railMarks(container);
    expect(marks).toHaveLength(31);
    const blanks = marks.filter((m) => m.style.backgroundColor === "var(--surface-2)");
    expect(blanks).toHaveLength(1); // the interior all-null day
  });

  it("classifies marks in and out of the band", () => {
    const { container } = render(<TrendRailCard section={suppressedView()} href={HREF} />);
    const marks = railMarks(container);
    const inBand = marks.filter((m) => m.style.backgroundColor === "var(--success)");
    const outBand = marks.filter((m) => m.style.backgroundColor === "var(--faint)");
    // Fixture: 28 in-band days; 105 (above) and today's 74 (below, single) out.
    expect(inBand).toHaveLength(28);
    expect(outBand).toHaveLength(2);
  });

  it("promotes a ≥3-day below-band run to warning", () => {
    const view = suppressedView();
    const days = view.days ?? [];
    // Force the last three days below balancedLow (78.6) with no gap.
    days[days.length - 3] = { date: days[days.length - 3].date, hrv: 74, restingHr: 51, recoveryScore: 55 };
    days[days.length - 2] = { date: days[days.length - 2].date, hrv: 72, restingHr: 51, recoveryScore: 54 };
    days[days.length - 1] = { date: FIXTURE_TODAY, hrv: 70, restingHr: 52, recoveryScore: 50 };
    const { container } = render(<TrendRailCard section={view} href={HREF} />);
    const warning = railMarks(container).filter(
      (m) => m.style.backgroundColor === "var(--warning)",
    );
    expect(warning).toHaveLength(3);
  });

  it("no reading today: the headline is unaffected by definition", () => {
    render(<TrendRailCard section={noReadingView()} href={HREF} />);
    expect(screen.getByText(/10%/)).toBeInTheDocument();
    expect(screen.getByText("−8.9 ms")).toBeInTheDocument();
  });

  it("calibrating: honest n-of-14 progress instead of a rail", () => {
    render(<TrendRailCard section={calibratingView()} href={HREF} />);
    expect(screen.getByText("Trend calibrating")).toBeInTheDocument();
    expect(screen.getByText(/9 of 14/)).toBeInTheDocument();
  });

  it("guards a missing days array", () => {
    const view: RecoveryView = { restingToday: 51, recoveryScore: 58, spark: [] };
    render(<TrendRailCard section={view} href={HREF} />);
    expect(screen.getByText("Trend calibrating")).toBeInTheDocument();
  });
});
```

- [ ] **Step 2: Run to verify it fails**

Run: `npx vitest run "app/(app)/dashboard/_components/recovery/trend-rail.test.tsx"`
Expected: FAIL — cannot resolve `./trend-rail`.

- [ ] **Step 3: Implement `trend-rail.tsx`**

```tsx
/**
 * TrendRailCard — the `recovery_trend` tile ("Recovery Trend").
 *
 * Heroes the DIRECTION and deliberately demotes today: one large delta figure
 * — the server 7-day HRV mean (`shortAvg`) against the 30-day baseline
 * (`hrvAvg`), a difference of two server figures, never a re-averaged series —
 * with the trend word beneath and a full-width rail of one mark per day.
 * Marks shade in-band success / out-of-band faint / null blank, and a run of
 * ≥3 consecutive BELOW-band days promotes to warning as a sustained dip
 * (above-band days never warn — elevated is unusual, not alarming). Today is
 * just the last mark, so the headline survives a missing morning webhook.
 */

import type { RecoveryView } from "@/lib/dashboard";
import { MiniCard } from "../mini-card";
import { hrvStatusColor, signedUnit, trendLabel } from "./shared";

const TITLE = "Recovery Trend";
// Consecutive below-band days that read as a sustained dip.
const RUN_THRESHOLD = 3;

type MarkTone = "in" | "below" | "above" | "run" | "blank";

const MARK_COLOR: Record<MarkTone, string> = {
  in: "var(--success)",
  below: "var(--faint)",
  above: "var(--faint)",
  run: "var(--warning)",
  blank: "var(--surface-2)",
};

export function TrendRailCard({ section, href }: { section: RecoveryView; href: string }) {
  const { days, baseline, hrv } = section;

  if (!days || !baseline || !hrv || baseline.hrvAvg === null || hrv.shortAvg === null) {
    return (
      <MiniCard title={TITLE} href={href}>
        <div className="flex flex-col gap-2 py-1">
          <span className="text-sm font-medium text-[var(--muted)]">Trend calibrating</span>
          <p className="text-[11px] text-[var(--faint)]">
            <span className="font-mono tabular-nums text-[var(--muted)]">
              {baseline?.hrvDays ?? 0} of 14
            </span>{" "}
            nights · the week-over-baseline read needs a baseline first
          </p>
        </div>
      </MiniCard>
    );
  }

  const deltaMs = hrv.shortAvg - baseline.hrvAvg;
  const deltaPct = Math.round((deltaMs / baseline.hrvAvg) * 100);
  const { glyph, word } = trendLabel(hrv.trend);
  const statusColor = hrvStatusColor(hrv.status);

  const marks = days.map((d) => classify(d.hrv, hrv.balancedLow, hrv.balancedHigh));
  promoteSustainedDips(marks, RUN_THRESHOLD);

  return (
    <MiniCard title={TITLE} href={href}>
      {/* The hero: one big delta figure. Today is NOT here. */}
      <div className="flex items-baseline gap-2">
        <span
          className="font-mono text-3xl font-semibold tracking-tight tabular-nums"
          style={{ color: statusColor }}
        >
          {glyph} {Math.abs(deltaPct)}%
        </span>
        <span className="font-mono text-sm tabular-nums text-[var(--muted)]">
          {signedUnit(deltaMs, "ms", 1)}
        </span>
      </div>
      <p className="-mt-1 text-xs text-[var(--muted)]">
        7-day HRV {word} · vs 30-day baseline{" "}
        <span className="font-mono tabular-nums">{Math.round(baseline.hrvAvg)} ms</span>
      </p>

      {/* Full-width rail of day marks — spatial consistency, gaps blank. */}
      <div
        className="mt-1 flex items-end gap-[2px]"
        role="img"
        aria-label="Thirty days in or out of your balanced band"
      >
        {marks.map((tone, i) => (
          <span
            key={i}
            className="h-4 min-w-[2px] flex-1 rounded-[1px]"
            style={{ backgroundColor: MARK_COLOR[tone], opacity: tone === "blank" ? 1 : 0.9 }}
          />
        ))}
      </div>
      <p className="text-[10px] text-[var(--faint)]">
        <span className="inline-flex items-center gap-1">
          <Dot c={MARK_COLOR.in} /> in band
        </span>
        <span className="mx-1.5 inline-flex items-center gap-1">
          <Dot c={MARK_COLOR.below} /> outside
        </span>
        <span className="inline-flex items-center gap-1">
          <Dot c={MARK_COLOR.run} /> sustained dip
        </span>
      </p>
    </MiniCard>
  );
}

function classify(hrv: number | null, low: number | null, high: number | null): MarkTone {
  if (hrv === null) return "blank";
  if (low === null || high === null) return "above";
  if (hrv < low) return "below";
  if (hrv > high) return "above";
  return "in";
}

/** Promote runs of ≥threshold consecutive below-band marks to "run". */
function promoteSustainedDips(marks: MarkTone[], threshold: number) {
  let start = -1;
  for (let i = 0; i <= marks.length; i++) {
    if (i < marks.length && marks[i] === "below") {
      if (start === -1) start = i;
    } else {
      if (start !== -1 && i - start >= threshold) {
        for (let j = start; j < i; j++) marks[j] = "run";
      }
      start = -1;
    }
  }
}

function Dot({ c }: { c: string }) {
  return (
    <span aria-hidden="true" className="h-1.5 w-1.5 rounded-[1px]" style={{ backgroundColor: c }} />
  );
}
```

- [ ] **Step 4: Run to verify it passes**

Run: `npx vitest run "app/(app)/dashboard/_components/recovery/trend-rail.test.tsx"`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add "app/(app)/dashboard/_components/recovery/trend-rail.tsx" "app/(app)/dashboard/_components/recovery/trend-rail.test.tsx"
git commit -m "feat(dashboard): recovery trend rail tile"
```

### Task 9: `morning-ledger` — Recovery Log

**Files:**
- Create: `app/(app)/dashboard/_components/recovery/morning-ledger.tsx`
- Test: `app/(app)/dashboard/_components/recovery/morning-ledger.test.tsx`

- [ ] **Step 1: Write the failing test**

`app/(app)/dashboard/_components/recovery/morning-ledger.test.tsx`:

```tsx
/// <reference types="vitest/globals" />

import { render, screen } from "@testing-library/react";
import { calibratingView, noReadingView, suppressedView } from "./fixtures";
import { MorningLedgerCard } from "./morning-ledger";

const HREF = "/recovery";

describe("MorningLedgerCard", () => {
  it("header prints the server baselines as the yardstick", () => {
    render(<MorningLedgerCard section={suppressedView()} href={HREF} />);
    // 91.2 ms · 52.4 bpm · 68.1 → rounded.
    expect(screen.getByText(/91 ms · 52 bpm · 68/)).toBeInTheDocument();
  });

  it("rows are newest-first with Today leading", () => {
    render(<MorningLedgerCard section={suppressedView()} href={HREF} />);
    const labels = screen.getAllByText(/^(Today|Fri|Thu|Wed)$/).map((el) => el.textContent);
    expect(labels).toEqual(["Today", "Fri", "Thu", "Wed"]);
  });

  it("a fully-null morning renders a 'no reading' row, not a vanished one", () => {
    // The fixture's interior gap (2026-07-29, Wed) falls inside the last four days.
    render(<MorningLedgerCard section={suppressedView()} href={HREF} />);
    expect(screen.getByText("no reading")).toBeInTheDocument();
  });

  it("today's row carries the HRV delta glyph as its only color", () => {
    const { container } = render(<MorningLedgerCard section={suppressedView()} href={HREF} />);
    // 74 vs 91.2 → below by more than 3 → ▼ in warning.
    const glyphs = Array.from(container.querySelectorAll("span")).filter(
      (s) => s.textContent === "▼",
    ) as HTMLElement[];
    expect(glyphs.length).toBeGreaterThan(0);
    expect(glyphs[0].style.color).toBe("var(--warning)");
  });

  it("no reading yet today: today appears as a no-reading row", () => {
    render(<MorningLedgerCard section={noReadingView()} href={HREF} />);
    expect(screen.getByText("Today")).toBeInTheDocument();
    expect(screen.getAllByText("no reading").length).toBeGreaterThan(0);
  });

  it("calibrating: the header shows honest n-of-14 progress", () => {
    render(<MorningLedgerCard section={calibratingView()} href={HREF} />);
    expect(screen.getByText(/9 of 14 nights/)).toBeInTheDocument();
  });
});
```

- [ ] **Step 2: Run to verify it fails**

Run: `npx vitest run "app/(app)/dashboard/_components/recovery/morning-ledger.test.tsx"`
Expected: FAIL — cannot resolve `./morning-ledger`.

- [ ] **Step 3: Implement `morning-ledger.tsx`**

```tsx
/**
 * MorningLedgerCard — the `recovery_log` tile ("Recovery Log").
 *
 * Heroes the LOG: the last four mornings as dated rows (weekday · HRV with a
 * baseline-delta glyph · resting HR · score) under a quiet server-baseline
 * header. Rows read from `days` (date-aligned) newest-first; a fully-null
 * morning renders an italic "no reading" row rather than vanishing — the gap
 * is data. The ONLY color on a row is the HRV delta glyph vs the server
 * baseline; dates, figures, and the header stay neutral ink.
 */

import type { RecoveryDayPoint, RecoveryView } from "@/lib/dashboard";
import { MiniCard } from "../mini-card";
import { weekday } from "./shared";

const TITLE = "Recovery Log";
const ROWS = 4;

export function MorningLedgerCard({ section, href }: { section: RecoveryView; href: string }) {
  const { days, baseline } = section;

  if (!days || !baseline) {
    return (
      <MiniCard title={TITLE} href={href}>
        <p className="text-sm text-[var(--muted)]">Log is calibrating.</p>
      </MiniCard>
    );
  }

  const calibrating = baseline.hrvAvg === null;
  const recent = days.slice(-ROWS);

  return (
    <MiniCard title={TITLE} href={href}>
      {/* Quiet baseline header — the yardstick every row is read against. */}
      <div className="flex items-center justify-between border-b border-[var(--border)] pb-1.5">
        <span className="text-[10px] uppercase tracking-wide text-[var(--faint)]">
          {calibrating ? "calibrating" : "30-day baseline"}
        </span>
        <span className="font-mono text-[11px] tabular-nums text-[var(--muted)]">
          {calibrating ? (
            <>{baseline.hrvDays} of 14 nights</>
          ) : (
            <>
              {baseline.hrvAvg !== null ? Math.round(baseline.hrvAvg) : "—"} ms ·{" "}
              {baseline.restingHrAvg !== null ? Math.round(baseline.restingHrAvg) : "—"} bpm ·{" "}
              {baseline.recoveryScoreAvg !== null ? Math.round(baseline.recoveryScoreAvg) : "—"}
            </>
          )}
        </span>
      </div>

      {/* Dense blotter of dated mornings, newest first. */}
      <div className="flex flex-col divide-y divide-[var(--border)]">
        {[...recent].reverse().map((d, i) => (
          <LedgerRow key={d.date} day={d} baselineHrv={baseline.hrvAvg} isToday={i === 0} />
        ))}
      </div>
    </MiniCard>
  );
}

/** One morning row. The only color on the row is the HRV delta glyph. */
function LedgerRow({
  day,
  baselineHrv,
  isToday,
}: {
  day: RecoveryDayPoint;
  baselineHrv: number | null;
  isToday: boolean;
}) {
  const label = isToday ? "Today" : weekday(day.date);
  const missing = day.hrv === null && day.restingHr === null && day.recoveryScore === null;

  if (missing) {
    return (
      <div className="flex items-center justify-between py-1.5">
        <span className="w-12 font-mono text-xs text-[var(--muted)]">{label}</span>
        <span className="text-xs italic text-[var(--faint)]">no reading</span>
      </div>
    );
  }

  // Delta glyph vs the server baseline — the row's only splash of color.
  let sign: { glyph: string; color: string } | null = null;
  if (day.hrv !== null && baselineHrv !== null) {
    const d = day.hrv - baselineHrv;
    if (d <= -3) sign = { glyph: "▼", color: "var(--warning)" };
    else if (d >= 3) sign = { glyph: "▲", color: "var(--success)" };
    else sign = { glyph: "▬", color: "var(--muted)" };
  }

  return (
    <div className="flex items-center justify-between py-1.5">
      <span className="w-12 font-mono text-xs text-[var(--muted)]">{label}</span>
      <div className="flex items-center gap-3 font-mono text-xs tabular-nums text-[var(--foreground)]">
        <span className="inline-flex w-16 items-center justify-end gap-1">
          {day.hrv !== null ? `${day.hrv} ms` : "— ms"}
          {sign && (
            <span aria-hidden="true" style={{ color: sign.color }}>
              {sign.glyph}
            </span>
          )}
        </span>
        <span className="w-14 text-right text-[var(--muted)]">
          {day.restingHr !== null ? `${day.restingHr} bpm` : "—"}
        </span>
        <span className="w-6 text-right">
          {day.recoveryScore !== null ? day.recoveryScore : "—"}
        </span>
      </div>
    </div>
  );
}
```

- [ ] **Step 4: Run to verify it passes**

Run: `npx vitest run "app/(app)/dashboard/_components/recovery/morning-ledger.test.tsx"`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add "app/(app)/dashboard/_components/recovery/morning-ledger.tsx" "app/(app)/dashboard/_components/recovery/morning-ledger.test.tsx"
git commit -m "feat(dashboard): recovery log ledger tile"
```

### Task 10: Catalog mirror + renderer rewire + retire `whoop-card`

**Files:**
- Modify: `lib/dashboard-tiles.ts`
- Modify: `lib/dashboard-tiles.test.ts`
- Modify: `app/(app)/dashboard/_components/tile-renderer.tsx`
- Modify: `app/(app)/dashboard/_components/tile-renderer.test.tsx`
- Delete: `app/(app)/dashboard/_components/whoop-card.tsx`, `app/(app)/dashboard/_components/whoop-card.test.tsx`

This lands in ONE commit — adding to the `TileId` union makes the renderer's `never` default a type error until its cases exist, and the pre-commit hook typechecks.

- [ ] **Step 1: Update the catalog tests (failing first)**

In `lib/dashboard-tiles.test.ts`:
- `expect(TILE_CATALOG.length).toBe(11)` → `toBe(15)` (rename the test to `"has exactly 15 tiles"`).
- The order array becomes:

```ts
    expect(TILE_CATALOG.map((t) => t.id)).toEqual([
      "running",
      "walking",
      "cycling",
      "hiking",
      "lifting",
      "steps",
      "nutrition",
      "bodyweight",
      "blood_pressure",
      "recovery",
      "hrv_balance",
      "morning_vitals",
      "recovery_trend",
      "recovery_log",
      "streak",
    ]);
```

- `ALL_TILE_IDS` gains, after `recovery: true,`:

```ts
    hrv_balance: true,
    morning_vitals: true,
    recovery_trend: true,
    recovery_log: true,
```

- Add a test pinning the family descriptions:

```ts
  test("the five recovery-family tiles have distinct titles and descriptions", () => {
    const family = ["recovery", "hrv_balance", "morning_vitals", "recovery_trend", "recovery_log"];
    const titles = new Set(family.map((id) => tileEntry(id as TileId).title));
    const descriptions = new Set(family.map((id) => tileEntry(id as TileId).description));
    expect(titles.size).toBe(5);
    expect(descriptions.size).toBe(5);
    for (const id of family) {
      expect(tileEntry(id as TileId).href).toBe("/recovery");
    }
    // The rewritten recovery entry no longer describes the retired card.
    expect(tileEntry("recovery").description).not.toContain("resting HR");
  });
```

Run: `npx vitest run lib/dashboard-tiles.test.ts` — expected: FAIL (count 11, unknown ids).

- [ ] **Step 2: Update `lib/dashboard-tiles.ts`**

The union gains four members after `"recovery"`:

```ts
export type TileId =
  | "running"
  | "walking"
  | "cycling"
  | "hiking"
  | "lifting"
  | "steps"
  | "nutrition"
  | "bodyweight"
  | "blood_pressure"
  | "recovery"
  | "hrv_balance"
  | "morning_vitals"
  | "recovery_trend"
  | "recovery_log"
  | "streak";
```

The `recovery` entry's description is rewritten and the four new entries follow it (before `streak`):

```ts
  {
    id: "recovery",
    title: "Recovery",
    href: "/recovery",
    description: "Today's readiness as a sentence, with baseline deltas.",
  },
  {
    id: "hrv_balance",
    title: "HRV Balance",
    href: "/recovery",
    description: "Today's HRV against your own balanced range.",
  },
  {
    id: "morning_vitals",
    title: "Morning Vitals",
    href: "/recovery",
    description: "Score, resting HR, and HRV vs your 30-day averages.",
  },
  {
    id: "recovery_trend",
    title: "Recovery Trend",
    href: "/recovery",
    description: "Which way your HRV is heading this week.",
  },
  {
    id: "recovery_log",
    title: "Recovery Log",
    href: "/recovery",
    description: "Your last few mornings as dated readings.",
  },
```

Run: `npx vitest run lib/dashboard-tiles.test.ts` — expected: PASS. (`npm run typecheck` now fails on the renderer switch — fixed next step.)

- [ ] **Step 3: Rewire `tile-renderer.tsx`**

Replace the import of `whoop-card`:

```tsx
import { RecoveryConnectCard } from "./recovery/connect-card";
import { ReadinessVerdictCard } from "./recovery/readiness-verdict";
import { HrvBalanceCard } from "./recovery/balance-band";
import { MorningVitalsCard } from "./recovery/three-dial-vitals";
import { TrendRailCard } from "./recovery/trend-rail";
import { MorningLedgerCard } from "./recovery/morning-ledger";
```

Replace the `recovery` case with the five family cases (each falls to its own titled connect CTA when `!data.recovery.present`; the `never` default is untouched):

```tsx
    case "recovery":
      return data.recovery.present ? (
        <ReadinessVerdictCard section={data.recovery} href={href} />
      ) : (
        <RecoveryConnectCard title="Recovery" href={href} />
      );
    case "hrv_balance":
      return data.recovery.present ? (
        <HrvBalanceCard section={data.recovery} href={href} />
      ) : (
        <RecoveryConnectCard title="HRV Balance" href={href} />
      );
    case "morning_vitals":
      return data.recovery.present ? (
        <MorningVitalsCard section={data.recovery} href={href} />
      ) : (
        <RecoveryConnectCard title="Morning Vitals" href={href} />
      );
    case "recovery_trend":
      return data.recovery.present ? (
        <TrendRailCard section={data.recovery} href={href} />
      ) : (
        <RecoveryConnectCard title="Recovery Trend" href={href} />
      );
    case "recovery_log":
      return data.recovery.present ? (
        <MorningLedgerCard section={data.recovery} href={href} />
      ) : (
        <RecoveryConnectCard title="Recovery Log" href={href} />
      );
```

Also update the file's header comment paragraph about recovery: the whole recovery FAMILY (five ids) reads the one `recovery` section, and each id renders its own titled `RecoveryConnectCard` when enabled-but-absent.

- [ ] **Step 4: Delete the retired card**

```bash
git rm "app/(app)/dashboard/_components/whoop-card.tsx" "app/(app)/dashboard/_components/whoop-card.test.tsx"
```

(Its behaviors live on: the empty CTA in `recovery/connect-card.tsx` + its test in `readiness-verdict.test.tsx`; the present-card coverage is superseded by the five tile test files.)

- [ ] **Step 5: Update `tile-renderer.test.tsx`**

Replace the old `"renders the live recovery card when present"` test (which asserted `bpm resting` from the retired card) and add family coverage. Add imports:

```tsx
import { suppressedView } from "./recovery/fixtures";
```

New/replacement tests inside the `describe`:

```tsx
  const FAMILY: [TileId, string][] = [
    ["recovery", "Recovery"],
    ["hrv_balance", "HRV Balance"],
    ["morning_vitals", "Morning Vitals"],
    ["recovery_trend", "Recovery Trend"],
    ["recovery_log", "Recovery Log"],
  ];

  it.each(FAMILY)("renders the %s card from the shared recovery section", (id, title) => {
    render(<TileCard id={id} data={fixture({ recovery: { present: true, ...suppressedView() } })} />);
    expect(screen.getByRole("heading", { name: title })).toBeInTheDocument();
    expect(screen.queryByText("Connect Whoop to see recovery")).not.toBeInTheDocument();
  });

  it.each(FAMILY)("renders the titled connect CTA for %s when recovery is absent", (id, title) => {
    render(<TileCard id={id} data={fixture({ recovery: { present: false } })} />);
    expect(screen.getByRole("heading", { name: title })).toBeInTheDocument();
    expect(screen.getByText("Connect Whoop to see recovery")).toBeInTheDocument();
  });
```

Import `TileId` for the tuple type: `import { type TileId } from "@/lib/dashboard-tiles";` (merge with any existing import from that module). Keep the existing "Connect Whoop CTA" test (still valid) and drop only the `bpm resting` one. The `RecoveryView` import stays only if still used; remove it if unused.

- [ ] **Step 6: Run the full local verification**

```bash
npx vitest run
npm run typecheck
npm run lint
```

Expected: all PASS.

- [ ] **Step 7: Commit**

```bash
git add lib/dashboard-tiles.ts lib/dashboard-tiles.test.ts "app/(app)/dashboard/_components/tile-renderer.tsx" "app/(app)/dashboard/_components/tile-renderer.test.tsx"
git commit -m "feat(dashboard): five recovery-family tiles in the catalog and renderer"
```

### Task 11: Web local CI gate

- [ ] **Step 1: Run every CI check locally**

```bash
cd /workspace/prog-strength-web
npm run lint
npm run format:check
npm run typecheck
npm run test
npm run build
```

Expected: all green. If `format:check` flags files, run `npx prettier --write` on them, re-run, and commit as `style(dashboard): format recovery tile sources`. Any other failure: fix the code, never weaken the check.

---

## Repo 3: prog-strength-docs (`/workspace/prog-strength-docs`)

### Task 12: Flip the DX to selected and the SOW to shipped

**Files:**
- Modify: `dx/recovery-tile.md` (frontmatter + body header)
- Modify: `sows/recovery-tile-family.md` (frontmatter + body header)

- [ ] **Step 1: Create the branch**

```bash
cd /workspace/prog-strength-docs && git checkout -b feat/recovery-tile-family
```

- [ ] **Step 2: Flip `dx/recovery-tile.md`**

Frontmatter: `status: draft` → `status: selected`, and add (mirroring `dx/steps-tile.md`'s `selected_idiom` convention, plural here because all five won):

```yaml
status: selected
selected_idioms:
  - balance-band
  - readiness-verdict
  - three-dial-vitals
  - trend-rail
  - morning-ledger
```

Body header line becomes:

```markdown
**Status**: Selected (all five idioms) · **Last updated**: 2026-08-02
```

And add a selection note directly under it (matching steps-tile's pattern):

```markdown
> **Selected:** all five idioms — `readiness-verdict`, `balance-band`,
> `three-dial-vitals`, `trend-rail`, `morning-ledger` (DX comparison PR
> Prog-Strength/prog-strength-web#143, closed un-merged). The
> no-two-variants-hero-the-same-figure constraint held on the spread, so all
> five ship as separate, independently addable catalog tiles. Built for real by
> [`sows/recovery-tile-family.md`](../sows/recovery-tile-family.md).
```

- [ ] **Step 3: Flip `sows/recovery-tile-family.md`**

Frontmatter: `status: ready_for_implementation` → `status: shipped`.
Body header: `**Status**: Ready for implementation · **Last updated**: 2026-08-01` → `**Status**: Shipped · **Last updated**: 2026-08-02`.

- [ ] **Step 4: Commit**

```bash
git add dx/recovery-tile.md sows/recovery-tile-family.md
git commit -m "docs: mark recovery-tile-family as shipped"
```

---

## Self-review checklist (spec coverage)

- Four `TileID`s after `TileRecovery` in Go, byte-identical order in TS mirror — Tasks 1, 10.
- Family re-gate, one `recovery` key, handler tests incl. `[hrv_balance]`-alone / three-family / no-family — Task 2.
- `defaultLayout` and `buildRecoverySection` body untouched — Tasks 1–2 (explicit non-edits).
- Five production components + titled empty variants + `whoop-card` retired — Tasks 5–10.
- Shared module single-sources status→token; `elevated`→accent and `suppressed`→warning asserted — Task 4.
- All charts read `days`; no `spark` reads in any new tile; polyline splits; rail gaps blank; ledger no-reading rows — Tasks 6, 8, 9.
- No client re-averaging: only `today − baseline` deltas and `shortAvg − hrvAvg` — Tasks 5–8.
- Four fixture states + interior gap + not-connected per tile — fixtures in Task 5, per-tile tests Tasks 5–9, renderer Task 10.
- Catalog 11→15, order test, `ALL_TILE_IDS`, rewritten `recovery` description — Task 10.
- CI green both repos, no `//nolint`, no `--no-verify` — Tasks 3, 11.
- DX `selected` (all five), SOW `shipped` — Task 12.
