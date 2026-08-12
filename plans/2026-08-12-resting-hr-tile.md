# Resting HR Tile — `sorted-strip` Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship a new tray-only `resting_hr` dashboard tile rendering the `sorted-strip` composition — today's bpm over a rank-ordered strip of the last thirty mornings, a dashed 30-day-average tick, the caption `4th lowest of your last 30`, and the three most recent mornings newest-first — plus the server-owned catalog and routing changes that make the tile addable at all.

**Architecture:** Two repos. `prog-strength-api` gains a `resting_hr` catalog constant (immediately after `recovery_log`), a `recoveryFamily` membership so a lone `resting_hr` layout still builds the shared `recovery` section, and the four tests that pin all of it — **no payload change**. `prog-strength-web` gains a pure ranking/geometry module (`recovery/resting-strip.ts`), the card that renders it (`recovery/resting-rank.tsx`), a fourth generation of recovery fixtures, and the catalog/renderer wiring. `ordinal` and `MIN_BASELINE_DAYS` graduate into `recovery/shared.ts` following the `recoveryBand*` precedent.

**Tech Stack:** Go 1.25 + chi (API); Next.js 16 App Router, React 19, TypeScript, Tailwind v4 (CSS-variable tokens), Vitest + Testing Library (jsdom) (web).

---

## Source documents (read before any task)

- **SOW**: `sows/resting-hr-tile.md` in `prog-strength-docs`. Binding.
- **DX**: `dx/resting-hr-tile.md` — already `status: selected` with
  `selected_idioms: [sorted-strip]`, the selection note, **and** the amended
  "no prerequisite" blockquote recording that the catalog is server-owned.
  **No DX edit is needed by this plan.** Read it for the fixture series and the
  colour contract only; where the DX and the SOW disagree, **the SOW wins**.
- **Design system**: `design-system.md` **v0.4.4**.

## Binding constraints (every task)

- **Tokens only, no raw hex.** This card may use `--foreground`, `--muted`,
  `--faint`, `--border`, `--border-strong`, `--surface-2`, `--warning`.
  `scope: in-system` — no new token, no palette or type divergence, and **no
  `design-system.md` change**.
- **`--accent` never appears on this card.** The periwinkle carries *elevated
  HRV* in this family; there is no equivalent state here and none is invented.
  A test asserts its absence.
- **`--danger` never appears on this card.** Red is licensed in this family for
  exactly one thing (a sub-33 Whoop recovery score). A resting HR has no
  published threshold. A test asserts its absence.
- **A low morning is never painted green.** No `--success` on this card.
- **Colour is gated on the 30-day average, at the rounded delta** — never the
  mockup's "upper third". When `restingHrAvg` is null the card spends **no
  colour at all**.
- **Nothing recomputes a server figure.** The card's only arithmetic is
  `Math.round` for display and counting how many of the athlete's own mornings
  sat below today's. No client-side average, band, z-score, status, or
  threshold.
- **Never draw from `spark`** — it omits missing mornings. Always `days`.
- **bpm renders as an integer everywhere**, and the *ranking* runs on the same
  rounded values.
- **Calibrating gates on `restingHrAvg` / `restingHrDays` and nothing else** —
  never `hrvDays`.
- **`MiniCard`'s panel, radius, and padding are not overridden.**
- **Additive only in `shared.ts` and `fixtures.ts`.** No existing export changes
  signature or value. `morning-ledger.test.tsx`, `hrv-tile.test.tsx`,
  `balance-band.test.tsx`, `trend-rail.test.tsx`, `readiness-verdict.test.tsx`,
  `three-dial-vitals.test.tsx`, `shared.test.ts`, `fixtures.test.ts` and
  `dashboard/page.test.tsx` must all pass **unmodified** except where a task
  below explicitly says otherwise.
- **No API data change.** `RecoveryView`, `RecoveryDayPoint`,
  `RecoveryBaselineView` and the recovery section builder are untouched.
- **No default-layout change.** The tile is tray-only.

## Local check gate (run before every commit and before any push)

**API** (`/workspace/prog-strength-api`) — this environment has no
`libsqlite3-dev`, so cgo needs the header shipped inside the `go-sqlite3`
module. Export this once per shell:

```bash
export CGO_CFLAGS="-I/tmp/sqliteinc"   # sqlite3.h copied from go-sqlite3's sqlite3-binding.h
go build ./...
go vet ./...
go run github.com/golangci/golangci-lint/v2/cmd/golangci-lint@v2.12.2 run --timeout=5m
go mod tidy && git diff --exit-code go.mod go.sum
go test ./...
```

If `/tmp/sqliteinc/sqlite3.h` does not exist, create it:

```bash
mkdir -p /tmp/sqliteinc
cp "$(go env GOMODCACHE)"/github.com/mattn/go-sqlite3@v1.14.44/sqlite3-binding.h /tmp/sqliteinc/sqlite3.h
cp "$(go env GOMODCACHE)"/github.com/mattn/go-sqlite3@v1.14.44/sqlite3ext.h /tmp/sqliteinc/
chmod +w /tmp/sqliteinc/*.h
```

**Web** (`/workspace/prog-strength-web`):

```bash
npm run lint && npm run format:check && npm run typecheck && npm run test && npm run build
```

Commits are Conventional Commits, lowercase subject, no trailing period.
**Never** `--no-verify`, never `//nolint`, never skip or weaken a test.

## File structure

### `prog-strength-api`

| File | Responsibility |
| --- | --- |
| `internal/dashboard/tiles.go` | **Modify.** `TileRestingHR` constant + `Catalog` entry after `TileRecoveryLog`. |
| `internal/dashboard/handler.go` | **Modify.** `resting_hr` joins `recoveryFamily`. |
| `internal/dashboard/tiles_test.go` | **Modify.** `all` list, `TestCatalog_Order` `want`, `TestValidTileID` case. |
| `internal/dashboard/summary_layout_test.go` | **Modify.** Default-layout pin + `[resting_hr]`-alone-yields-`recovery` test. |
| `internal/dashboard/layout_handler_test.go` | **Modify.** `resting_hr` validates on write. |

### `prog-strength-web`

| File | Responsibility |
| --- | --- |
| `app/(app)/dashboard/_components/recovery/shared.ts` | **Extend.** `MIN_BASELINE_DAYS`, `ordinal`. |
| `app/(app)/dashboard/_components/recovery/morning-ledger.tsx` | **Touch.** Private `MIN_BASELINE_DAYS` → import. |
| `app/(app)/dashboard/_components/recovery/hrv-tile.tsx` | **Touch.** Two literal `14`s → the same import. |
| `app/(app)/dashboard/_components/recovery/resting-strip.ts` | **New.** Pure ranking + geometry. No React. |
| `app/(app)/dashboard/_components/recovery/resting-strip.test.ts` | **New.** Unit tests for the above. |
| `app/(app)/dashboard/_components/recovery/fixtures.ts` | **Extend.** `restingDays` + the RHR views. |
| `app/(app)/dashboard/_components/recovery/fixtures.test.ts` | **Extend.** The true-mean invariant. |
| `app/(app)/dashboard/_components/recovery/resting-rank.tsx` | **New.** `RestingRankCard` — four registers. |
| `app/(app)/dashboard/_components/recovery/resting-rank.test.tsx` | **New.** One test per state. |
| `lib/dashboard-tiles.ts` | **Modify.** `TileId` union + `TILE_CATALOG` entry. |
| `lib/dashboard-tiles.test.ts` | **Modify.** 20 → 21, order list, `ALL_TILE_IDS`. |
| `app/(app)/dashboard/_components/tile-renderer.tsx` | **Modify.** `case "resting_hr"`. |
| `app/(app)/dashboard/_components/tile-renderer.test.tsx` | **Modify.** Coverage for the new case. |

## Deviation from the SOW's file names (recorded, deliberate)

The SOW's file-layout table names the pure module
`recovery/resting-rank.ts` beside the component `recovery/resting-rank.tsx`.
**Those two cannot coexist.** Vite's resolver (and `tsconfig`'s
`moduleResolution: "bundler"`) tries `.ts` before `.tsx`, so `./resting-rank`
always resolves to the `.ts` file and the component becomes unimportable;
`allowImportingTsExtensions` is not enabled, so `./resting-rank.tsx` is not a
legal specifier either. This was verified empirically in this repo before the
plan was written, not assumed.

The pure module is therefore **`recovery/resting-strip.ts`**. That is the same
shape as the precedent the SOW itself cites — `hrv-tile.tsx` pairs with
`hrv-chart.ts`, the pure module named after the graphic — and it costs the SOW
nothing it argued for: the SOW's naming argument is entirely about the
*component* being `RestingRankCard` / `resting-rank.tsx` rather than
`SortedStripCard`, and that is honoured exactly. Say so in the web PR body.

## Task order

API first (1–2), then web (3–7). The two repos deploy independently and the web
tile degrades to a connect CTA until the API ships, so the API PR merges first.

---

## Task 1: API — the catalog entry

**Repo:** `prog-strength-api`, branch `feat/resting-hr-tile`.

**Files:**
- Modify: `internal/dashboard/tiles.go`
- Modify: `internal/dashboard/tiles_test.go`
- Modify: `internal/dashboard/layout_handler_test.go`

**Why this is in the API at all:** the tile catalog is server-owned.
`layout_handler.go` validates every incoming tile id against `ValidTileID` and
422s an unknown one; `Layout.Normalize` drops it on read. A web-only catalog
entry could never be added by a user. The DX's "do not add `prog-strength-api`"
blockquote is wrong and the SOW deliberately overrides it.

- [ ] **Step 1: Write the failing tests**

In `internal/dashboard/tiles_test.go`, add `TileRestingHR` to the `all` slice in
`TestCatalog_EveryConstantAppearsExactlyOnce` and to the `want` slice in
`TestCatalog_Order`, in both cases **immediately after `TileRecoveryLog` and
before `TileSleep`**, so the recovery run stays contiguous:

```go
	TileRecovery, TileHRVBalance, TileMorningVitals, TileRecoveryLog, TileRestingHR,
	TileSleep, TileStreak, TileQuote, TileWeather,
```

(Apply that same replacement to **both** slices — they are byte-identical
today and must stay so.)

Add to `TestValidTileID`, beside the other family cases:

```go
	if !ValidTileID("resting_hr") {
		t.Error("resting_hr should be valid")
	}
```

In `internal/dashboard/layout_handler_test.go`, add — this is the test that
would have caught the whole API-scope question:

```go
// TestPutLayout_RestingHrAccepted is the test that makes the catalog change
// load-bearing rather than cosmetic. Before resting_hr entered Catalog,
// ValidTileID rejected it and this write 422'd — so a user who added the tile
// from the tray got an error toast and the tile never persisted. A web-only
// catalog entry cannot fix that, which is why this SOW touches the API.
func TestPutLayout_RestingHrAccepted(t *testing.T) {
	r, rp, userID := newTestEnv(t)

	rec := putLayout(t, r, userID, `{"sections":[{"id":"a","tile_ids":["resting_hr"]}]}`)
	if rec.Code != http.StatusNoContent {
		t.Fatalf("status = %d, want 204; body=%s", rec.Code, rec.Body.String())
	}

	got, err := rp.layout.Get(context.Background(), userID)
	if err != nil {
		t.Fatalf("layout Get: %v", err)
	}
	assertSections(t, got.Sections, []Section{
		{ID: "a", TileIDs: []TileID{TileRestingHR}},
	})
}
```

- [ ] **Step 2: Run the tests to verify they fail**

```bash
export CGO_CFLAGS="-I/tmp/sqliteinc"
go test ./internal/dashboard/ -run 'TestCatalog|TestValidTileID|TestPutLayout_RestingHr'
```

Expected: FAIL to **compile** — `undefined: TileRestingHR`.

- [ ] **Step 3: Add the constant and the catalog entry**

In `internal/dashboard/tiles.go`, after the `TileRecoveryLog` constant and
before the `TileSleep` block:

```go
	// TileRestingHR ranks this morning's resting heart rate within the
	// athlete's own last thirty, rather than dating it. It reads the SAME
	// shared recovery section its four siblings read and needs no new field —
	// it is in the catalog because the catalog is server-owned, not because
	// the payload changed.
	TileRestingHR TileID = "resting_hr"
```

In `Catalog`, extend the recovery run:

```go
var Catalog = []TileID{
	TileRunning, TileRunningLog, TileRunningEffort, TileRunningVertical,
	TileWalking, TileCycling, TileHiking, TileLifting,
	TileSteps, TileNutrition, TileBodyweight, TileBloodPressure,
	TileRecovery, TileHRVBalance, TileMorningVitals, TileRecoveryLog,
	TileRestingHR, TileSleep, TileStreak, TileQuote, TileWeather,
}
```

Position is load-bearing: catalog order fixes the add-tile tray order, so the
tray reads `… recovery_log, resting_hr, sleep …` and the family stays
contiguous with `sleep` keeping its place after it.

- [ ] **Step 4: Run the tests to verify they pass**

```bash
go test ./internal/dashboard/ -run 'TestCatalog|TestValidTileID|TestPutLayout'
```

Expected: PASS.

- [ ] **Step 5: Run the full gate**

```bash
go build ./... && go vet ./... && \
  go run github.com/golangci/golangci-lint/v2/cmd/golangci-lint@v2.12.2 run --timeout=5m && \
  go mod tidy && git diff --exit-code go.mod go.sum && go test ./...
```

Expected: all green, no `go.mod`/`go.sum` drift.

- [ ] **Step 6: Commit**

```bash
git add internal/dashboard/tiles.go internal/dashboard/tiles_test.go \
        internal/dashboard/layout_handler_test.go
git commit -m "feat(dashboard): add the resting_hr tile to the catalog"
```

---

## Task 2: API — `resting_hr` joins the recovery family

**Repo:** `prog-strength-api`, branch `feat/resting-hr-tile`.

**Files:**
- Modify: `internal/dashboard/handler.go` (the `recoveryFamily` slice, ~line 281)
- Modify: `internal/dashboard/summary_layout_test.go`

**Why this is the easy-to-miss one:** the recovery section is built once, when
*any* family tile is enabled, and emitted under the single `recovery` key.
Without this entry a user whose only recovery tile is `resting_hr` gets a
response with **no** `recovery` section, `data.recovery.present` is false, and
the tile renders the connect CTA forever — to a user who is already connected.
There is no type error and no existing test failure to catch it.

- [ ] **Step 1: Write the failing tests**

In `internal/dashboard/summary_layout_test.go`, beside
`TestSummary_FamilyTileAlone_YieldsRecoverySection`:

```go
// TestSummary_RestingHrTileAlone_YieldsRecoverySection pins the membership that
// has no type error behind it: resting_hr reads the shared recovery section, so
// a layout whose ONLY recovery tile is resting_hr must still get one built.
// Without the recoveryFamily entry this returns no "recovery" key and the tile
// shows a connect CTA to an already-connected user.
func TestSummary_RestingHrTileAlone_YieldsRecoverySection(t *testing.T) {
	r, rp, userID := newTestEnv(t)
	seedWhoopConnected(t, rp, userID)

	if err := rp.layout.Upsert(context.Background(), userID, SingleSection([]TileID{TileRestingHR})); err != nil {
		t.Fatalf("layout upsert: %v", err)
	}

	layout, data := dataEnvelope(t, r, userID, "?timezone=UTC")

	if !equalStrs(layout, []string{"resting_hr"}) {
		t.Errorf("layout = %v, want [resting_hr]", layout)
	}
	assertKeysPresent(t, data, "recovery")
	assertKeysAbsent(t, data, "resting_hr")
	if string(data["recovery"]) == "null" {
		t.Error("recovery = null, want a populated section for a connected user")
	}
}

// TestSummary_DefaultLayoutHasNoRestingHrTile pins the rollout, mirroring the
// sleep tile: resting_hr is a catalog tile but NOT part of the default layout.
// Three tiles already print today's resting HR, so a fourth arriving unbidden
// on everyone's dashboard is a bigger product decision than this SOW's remit.
func TestSummary_DefaultLayoutHasNoRestingHrTile(t *testing.T) {
	r, rp, userID := newTestEnv(t)
	seedWhoopConnected(t, rp, userID)

	layout, data := dataEnvelope(t, r, userID, "?timezone=UTC")

	if indexOf(layout, string(TileRestingHR)) >= 0 {
		t.Errorf("default layout must not contain %q; got %v", TileRestingHR, layout)
	}
	assertKeysAbsent(t, data, "resting_hr")
}
```

- [ ] **Step 2: Run the tests to verify the family one fails**

```bash
export CGO_CFLAGS="-I/tmp/sqliteinc"
go test ./internal/dashboard/ -run 'TestSummary_RestingHrTileAlone|TestSummary_DefaultLayoutHasNoRestingHrTile' -v
```

Expected: `TestSummary_DefaultLayoutHasNoRestingHrTile` PASSES already (the
default layout is untouched — that is the point of pinning it), and
`TestSummary_RestingHrTileAlone_YieldsRecoverySection` FAILS with a missing
`recovery` key.

- [ ] **Step 3: Add the family member**

In `internal/dashboard/handler.go`:

```go
	recoveryFamily := []TileID{
		TileRecovery, TileHRVBalance, TileMorningVitals, TileRecoveryLog, TileRestingHR,
	}
```

Nothing else changes — no new section, no new field, no payload consequence.

- [ ] **Step 4: Run the tests to verify they pass**

```bash
go test ./internal/dashboard/ -run TestSummary -v 2>&1 | tail -40
```

Expected: PASS, including the pre-existing
`TestSummary_NoFamilyTile_NoRecoverySection` (a layout of `[running, streak]`
still yields no `recovery` key).

- [ ] **Step 5: Run the full gate**

```bash
go build ./... && go vet ./... && \
  go run github.com/golangci/golangci-lint/v2/cmd/golangci-lint@v2.12.2 run --timeout=5m && \
  go mod tidy && git diff --exit-code go.mod go.sum && go test ./...
```

- [ ] **Step 6: Commit**

```bash
git add internal/dashboard/handler.go internal/dashboard/summary_layout_test.go
git commit -m "feat(dashboard): build the recovery section for a lone resting_hr tile"
```

---

## Task 3: Web — `MIN_BASELINE_DAYS` and `ordinal` move into `shared.ts`

**Repo:** `prog-strength-web`, branch `feat/resting-hr-tile`.

**Files:**
- Modify: `app/(app)/dashboard/_components/recovery/shared.ts`
- Modify: `app/(app)/dashboard/_components/recovery/morning-ledger.tsx:50-55`
- Modify: `app/(app)/dashboard/_components/recovery/hrv-tile.tsx:178-194`
- Modify: `app/(app)/dashboard/_components/recovery/shared.test.ts`

**Context:** the SOW single-sources two things into `shared.ts` "following the
`recoveryBand*` precedent". `MIN_BASELINE_DAYS` is already written out three
times across this family and a fourth copy is how the tiles start disagreeing
about what "calibrating" means. Both edits are **constant substitutions with no
behaviour change**: `morning-ledger.test.tsx` and `hrv-tile.test.tsx` must pass
**unmodified**. If either needs editing, something other than the constant moved
— stop and report that.

- [ ] **Step 1: Write the failing tests**

Append to `app/(app)/dashboard/_components/recovery/shared.test.ts` (match the
file's existing `describe`/`it` or `test` style — read the top of the file
first and follow it):

```ts
describe("MIN_BASELINE_DAYS", () => {
  it("is the server's MinBaselineDays", () => {
    expect(MIN_BASELINE_DAYS).toBe(14);
  });
});

describe("ordinal", () => {
  it("suffixes the ordinary cases", () => {
    expect(ordinal(1)).toBe("1st");
    expect(ordinal(2)).toBe("2nd");
    expect(ordinal(3)).toBe("3rd");
    expect(ordinal(4)).toBe("4th");
    expect(ordinal(30)).toBe("30th");
  });

  // The teens are the whole reason this is a function and not a lookup on the
  // last digit: 11/12/13 take "th" even though 1/2/3 do not.
  it("gives the teens 'th'", () => {
    expect(ordinal(11)).toBe("11th");
    expect(ordinal(12)).toBe("12th");
    expect(ordinal(13)).toBe("13th");
  });

  it("resumes the pattern above the teens", () => {
    expect(ordinal(21)).toBe("21st");
    expect(ordinal(22)).toBe("22nd");
    expect(ordinal(23)).toBe("23rd");
    expect(ordinal(111)).toBe("111th");
  });
});
```

Add `MIN_BASELINE_DAYS` and `ordinal` to the file's existing import from
`./shared`.

- [ ] **Step 2: Run the test to verify it fails**

```bash
npx vitest run "app/(app)/dashboard/_components/recovery/shared.test.ts"
```

Expected: FAIL — `MIN_BASELINE_DAYS`/`ordinal` are not exported.

- [ ] **Step 3: Add both to `shared.ts`**

Append to `app/(app)/dashboard/_components/recovery/shared.ts`:

```ts
/**
 * How many mornings the server needs behind a metric before it emits an average
 * (`internal/recoverytrend`'s `MinBaselineDays`). A client-side copy of a server
 * constant, single-sourced here because it was already written out three times
 * across this family and a fourth copy is how the four tiles start disagreeing
 * about what "calibrating" means.
 *
 * It is deliberately per-METRIC on the server: `restingHrDays`, `hrvDays` and
 * `recoveryScoreDays` are separate samples measured against this same 14, so a
 * tile gates on the count for the metric it actually draws — never on another's.
 */
export const MIN_BASELINE_DAYS = 14;

/** 1 → "1st", 4 → "4th", 11 → "11th". For the rank caption. */
export function ordinal(n: number): string {
  const tens = n % 100;
  // 11th, 12th, 13th are the exceptions the last digit alone gets wrong.
  if (tens >= 11 && tens <= 13) return `${n}th`;
  switch (n % 10) {
    case 1:
      return `${n}st`;
    case 2:
      return `${n}nd`;
    case 3:
      return `${n}rd`;
    default:
      return `${n}th`;
  }
}
```

Also extend the file's header docstring with one sentence recording the new
remit, in the style of the two paragraphs already there — the file documents
each time its scope grows.

- [ ] **Step 4: Substitute the constant at both call sites**

In `morning-ledger.tsx`, **delete** the private declaration (the comment block
and `const MIN_BASELINE_DAYS = 14;`) and add `MIN_BASELINE_DAYS` to the existing
`./shared` import:

```ts
import {
  MIN_BASELINE_DAYS,
  recoveryBand,
  recoveryBandColor,
  recoveryBandWord,
  round,
  weekday,
} from "./shared";
```

In `hrv-tile.tsx`, import it and interpolate it in place of the **two** literal
`14`s inside `Calibrating`:

```tsx
import { MIN_BASELINE_DAYS } from "./shared";
```

```tsx
/** New-user state — no band to draw yet. Honest progress toward 14 nights. */
function Calibrating({ nights }: { nights: number }) {
  return (
    <div className="flex flex-col gap-2 py-1">
      <span className="text-sm font-medium text-[var(--muted)]">Calibrating your band</span>
      <div className="h-1.5 w-full overflow-hidden rounded-full bg-[var(--surface-2)]">
        <div
          className="h-full rounded-full bg-[var(--accent)]"
          style={{ width: `${Math.min(100, (nights / MIN_BASELINE_DAYS) * 100)}%` }}
        />
      </div>
      <p className="text-[11px] text-[var(--faint)]">
        <span className="font-mono tabular-nums text-[var(--muted)]">
          {nights} of {MIN_BASELINE_DAYS}
        </span>{" "}
        nights · your normal range appears once Whoop knows your spread
      </p>
    </div>
  );
}
```

**Watch the whitespace.** The original renders `9 of 14 nights · …` with a
single space between the `</span>` and `nights`. Splitting the JSX across lines
can eat it — the `{" "}` above is what preserves it, and `hrv-tile.test.tsx`
asserts on that string. If the test fails on spacing, fix the JSX, not the test.

- [ ] **Step 5: Run the affected suites to verify they pass unmodified**

```bash
npx vitest run "app/(app)/dashboard/_components/recovery/"
```

Expected: PASS, including `morning-ledger.test.tsx` and `hrv-tile.test.tsx`
with **zero edits** to either file.

- [ ] **Step 6: Run the full gate and commit**

```bash
npm run lint && npm run format:check && npm run typecheck && npm run test && npm run build
git add "app/(app)/dashboard/_components/recovery/shared.ts" \
        "app/(app)/dashboard/_components/recovery/shared.test.ts" \
        "app/(app)/dashboard/_components/recovery/morning-ledger.tsx" \
        "app/(app)/dashboard/_components/recovery/hrv-tile.tsx"
git commit -m "refactor(dashboard): single-source MIN_BASELINE_DAYS and add ordinal to recovery shared"
```

---

## Task 4: Web — `resting-strip.ts`, the pure ranking and geometry module

**Repo:** `prog-strength-web`, branch `feat/resting-hr-tile`.

**Files:**
- Create: `app/(app)/dashboard/_components/recovery/resting-strip.ts`
- Create: `app/(app)/dashboard/_components/recovery/resting-strip.test.ts`

**Context:** the module is separated from the component for the same reason
`hrv-chart.ts` was: the ranking is where every one of the SOW's corrections
lives, it is where the tests earn their keep, and it is the natural import if
the strip is ever promoted to `/recovery`. **No React in this file.**

The two corrections that live here:

1. **Ranking runs on integers.** The mockup ranked raw floats, which orders two
   mornings the card both prints as `50` and then prints an ordinal counting a
   difference the user cannot see. Round first, then sort and rank.
2. **Labels are anchored and de-conflicted**, so today's label can neither leave
   the card nor collide with an endpoint label — and the avg *tick* keeps its
   true position even when its *label* is nudged.

- [ ] **Step 1: Write the failing tests**

Create `app/(app)/dashboard/_components/recovery/resting-strip.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import type { RecoveryDayPoint } from "@/lib/dashboard";
import {
  AVG_LABEL_MAX_PCT,
  AVG_LABEL_MIN_PCT,
  avgInsertPct,
  endpointRow,
  labelAnchor,
  rankOf,
  sortedMornings,
  STRIP_WINDOW,
  tickPct,
} from "./resting-strip";

/** A minimal day series — this module reads `restingHr` and nothing else. */
function days(values: (number | null)[]): RecoveryDayPoint[] {
  return values.map((restingHr, i) => ({
    date: `2026-07-${String(i + 1).padStart(2, "0")}`,
    restingHr,
    recoveryScore: null,
    hrv: null,
    baselineAvg: null,
    balancedLow: null,
    balancedHigh: null,
    zScore: null,
    status: "unknown" as const,
  }));
}

describe("sortedMornings", () => {
  // CORRECTION 1. This test must FAIL against the mockup's raw-float formula.
  it("rounds before sorting, so two mornings the card both prints as 50 sort as one value", () => {
    expect(sortedMornings(days([49.6, 50]), STRIP_WINDOW)).toEqual([50, 50]);
  });

  it("sorts ascending", () => {
    expect(sortedMornings(days([59, 47, 53, 48]), STRIP_WINDOW)).toEqual([47, 48, 53, 59]);
  });

  it("drops nulls rather than zero-filling", () => {
    expect(sortedMornings(days([50, null, 48]), STRIP_WINDOW)).toEqual([48, 50]);
  });

  it("returns an empty array for an all-null window", () => {
    expect(sortedMornings(days([null, null, null]), STRIP_WINDOW)).toEqual([]);
  });

  it("takes only the last `window` mornings", () => {
    // 31 entries in, 30 out — the oldest is excluded, as the payload's own
    // 31-day window means `days` carries one more than the strip draws.
    const thirtyOne = Array.from({ length: 31 }, (_, i) => 40 + i);
    const got = sortedMornings(days(thirtyOne), STRIP_WINDOW);
    expect(got).toHaveLength(30);
    expect(got[0]).toBe(41);
  });
});

describe("rankOf", () => {
  it("is 1-based from the lowest", () => {
    expect(rankOf([47, 48, 49, 50], 47)).toBe(1);
    expect(rankOf([47, 48, 49, 50], 50)).toBe(4);
  });

  // Ties share the LOWER rank — two identical 48s are both "1st lowest" rather
  // than arbitrarily ordered.
  it("gives tied values the same rank", () => {
    expect(rankOf([47, 48, 48, 48, 50], 48)).toBe(2);
  });

  it("ranks a float exactly as its printed integer does", () => {
    expect(rankOf([47, 50, 50, 59], 49.6)).toBe(rankOf([47, 50, 50, 59], 50));
    expect(rankOf([47, 50, 50, 59], 49.6)).toBe(2);
  });

  it("is null when there is no reading yet today", () => {
    expect(rankOf([47, 48, 49], null)).toBeNull();
  });
});

describe("tickPct", () => {
  it("centres each tick in its own share of the strip", () => {
    expect(tickPct(0, 4)).toBe(12.5);
    expect(tickPct(3, 4)).toBe(87.5);
  });
});

describe("avgInsertPct", () => {
  // The average is NOT one of the athlete's mornings, so it sits at the
  // BOUNDARY between two ticks — never at a tick centre.
  it("places the average at the boundary between two ticks", () => {
    expect(avgInsertPct([47, 48, 49, 50], 48.5)).toBe(50);
  });

  it("computes against the raw average, not a rounded one", () => {
    // 53.4 genuinely sits above every 53: the tick is a position, not a value.
    expect(avgInsertPct([53, 53, 54, 54], 53.4)).toBe(50);
  });

  it("is null when there is no average yet", () => {
    expect(avgInsertPct([47, 48], null)).toBeNull();
  });
});

describe("labelAnchor", () => {
  it("anchors left near the low end so the label stays on the card", () => {
    expect(labelAnchor(1.7)).toEqual({ left: "0%", transform: "none" });
  });

  it("anchors right near the high end", () => {
    expect(labelAnchor(98.3)).toEqual({ left: "100%", transform: "translateX(-100%)" });
  });

  it("centres the label over its tick everywhere between", () => {
    expect(labelAnchor(50)).toEqual({ left: "50%", transform: "translateX(-50%)" });
  });
});

describe("endpointRow", () => {
  it("shows both endpoint labels when there is no average to conflict with", () => {
    expect(endpointRow(null)).toEqual({
      avgLabelPct: null,
      showLowest: true,
      showHighest: true,
    });
  });

  it("suppresses neither endpoint when the average sits mid-strip", () => {
    expect(endpointRow(50)).toEqual({ avgLabelPct: 50, showLowest: true, showHighest: true });
  });

  it("clamps the avg LABEL to the card, leaving the caller's tick at its true position", () => {
    expect(endpointRow(2).avgLabelPct).toBe(AVG_LABEL_MIN_PCT);
    expect(endpointRow(97).avgLabelPct).toBe(AVG_LABEL_MAX_PCT);
  });

  // The avg label wins against the endpoints: it is the more informative
  // figure, and the extremes stay visible as the outermost ticks regardless.
  it("drops the lowest label when the avg label crowds it", () => {
    expect(endpointRow(20).showLowest).toBe(false);
    expect(endpointRow(20).showHighest).toBe(true);
  });

  it("drops the highest label when the avg label crowds it", () => {
    expect(endpointRow(90).showHighest).toBe(false);
    expect(endpointRow(90).showLowest).toBe(true);
  });

  it("keeps both at the clearance boundaries", () => {
    expect(endpointRow(34).showLowest).toBe(true);
    expect(endpointRow(66).showHighest).toBe(true);
  });
});

describe("the polarity pin", () => {
  // Reversing the sort to descending would invert the card's ONLY statement of
  // polarity while leaving both endpoint labels technically correct — the
  // quietest possible way to break this tile. Pinned here and again in the DOM.
  it("sorts so that index 0 is the lowest", () => {
    const sorted = sortedMornings(days([59, 47, 53, 48]), STRIP_WINDOW);
    expect(sorted[0]).toBeLessThanOrEqual(sorted[sorted.length - 1]);
    expect(sorted[0]).toBe(47);
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

```bash
npx vitest run "app/(app)/dashboard/_components/recovery/resting-strip.test.ts"
```

Expected: FAIL — cannot resolve `./resting-strip`.

- [ ] **Step 3: Write the module**

Create `app/(app)/dashboard/_components/recovery/resting-strip.ts`:

```ts
/**
 * Ranking and strip geometry for the `resting_hr` tile — pure, no React.
 *
 * The tile's claim is that a resting heart rate is the metric where an absolute
 * number is least interpretable: a 50 is excellent for one athlete and elevated
 * for another. So it does not date the morning, it RANKS it — the last thirty
 * mornings sorted low to high as a strip of ticks, today's filled, under the
 * caption `4th lowest of your last 30`. Everything that decides where a tick
 * goes and which morning outranks which lives here, because that is where the
 * corrections below can be tested rather than eyeballed.
 *
 * INTEGER-FIRST RANKING is the load-bearing one, and it is the one most likely
 * to be undone by a well-meaning simplification. Every figure on this card
 * prints as an integer, so ranking on the raw floats would order two mornings
 * the card both calls `50` — and then print an ordinal counting a difference
 * the user cannot see. It is the same lesson the `flat-month` fixture taught
 * about colour, applied to order instead of ink: *a difference the card does
 * not print is a difference the card does not rank*. Round first and the
 * strip's order, the endpoint labels and the caption all describe the same
 * numbers.
 *
 * The strip encodes ORDER, not magnitude — ticks are evenly spaced by rank, and
 * magnitude lives in the printed extremes. That is what makes a flat month
 * legible without a chart: `48 lowest` / `50 highest` reads as *nothing is
 * happening*, with no auto-scale to lie about it.
 */

import type { RecoveryDayPoint } from "@/lib/dashboard";

/**
 * The strip's window: the last thirty mornings INCLUDING today. `days` carries
 * 31 entries, so the oldest is excluded.
 */
export const STRIP_WINDOW = 30;

/**
 * The athlete's last `window` mornings as INTEGER bpm, sorted ascending.
 *
 * Rounding before sorting is not cosmetic — see the file header.
 *
 * Nulls are dropped, not zero-filled: a strap-off morning is an absent reading,
 * not a heart rate of zero, and the caller's `n` (this array's length) reports
 * how many mornings are actually behind the rank. On a sparse month that is
 * visibly not thirty, and saying so is the honest caption.
 */
export function sortedMornings(days: RecoveryDayPoint[], window: number): number[] {
  return days
    .slice(-window)
    .map((d) => d.restingHr)
    .filter((v): v is number => v !== null)
    .map((v) => Math.round(v))
    .sort((a, b) => a - b);
}

/**
 * Today's rank among them, 1-based, with ties sharing the LOWER rank — two
 * identical 48s are both "1st lowest" rather than arbitrarily ordered. Null when
 * there is no reading yet today.
 *
 * `today` is rounded here for the same reason the series is: a 49.6 that prints
 * as `50` must rank exactly where a 50 ranks.
 */
export function rankOf(sorted: number[], today: number | null): number | null {
  if (today === null) return null;
  const t = Math.round(today);
  return sorted.filter((v) => v < t).length + 1;
}

/** Tick centres, evenly spaced by rank. */
export const tickPct = (i: number, n: number): number => ((i + 0.5) / n) * 100;

/**
 * Where the 30-day average inserts. Deliberately at the BOUNDARY between two
 * ticks — `k / n`, exactly halfway between tick k−1 and tick k — because the
 * average is not one of the athlete's mornings and must not appear to be one.
 * Computed against the raw average, not a rounded one: the tick is a position,
 * and 53.4 genuinely sits above every 53.
 *
 * The average's own window (30 days trailing, EXCLUDING today) is not the same
 * set as the strip's thirty (including today, excluding the oldest). That is
 * not a defect and must not be "fixed" by re-averaging the strip's values,
 * which is the forbidden operation. The tick claims a position, not membership
 * — and the dash the caller draws it with is what says so.
 *
 * Callers guard `sorted.length > 0` before the strip renders at all, so the
 * division is safe by construction.
 */
export function avgInsertPct(sorted: number[], avg: number | null): number | null {
  if (avg === null) return null;
  return (sorted.filter((v) => v < avg).length / sorted.length) * 100;
}

/**
 * left / centre / right anchoring, so a label near an edge stays on the card
 * AND stays adjacent to the tick it names.
 *
 * Switching anchor rather than clamping is the point: `tickPct(0, 30)` is 1.7%,
 * and a centred label there sits mostly outside the panel. Clamping the
 * POSITION would drag the label away from its tick; switching the anchor keeps
 * it over the tick and inside the card at the same time.
 */
export function labelAnchor(pct: number): { left: string; transform: string } {
  if (pct < 6) return { left: "0%", transform: "none" };
  if (pct > 94) return { left: "100%", transform: "translateX(-100%)" };
  return { left: `${pct}%`, transform: "translateX(-50%)" };
}

export const AVG_LABEL_MIN_PCT = 18;
export const AVG_LABEL_MAX_PCT = 82;
/** Clearance either side of the avg label before an endpoint label is dropped. */
export const ENDPOINT_CLEARANCE_PCT = 16;

/**
 * The endpoint register's layout: where the avg label sits, and whether the
 * `lowest` / `highest` captions survive beside it.
 *
 * The avg label yields to the card edges and WINS against the endpoints. It is
 * the more informative figure, and the extreme values remain visible as the
 * outermost ticks even without their captions.
 *
 * Only the LABEL is clamped — the caller draws the avg tick at its true
 * `avgPct`, so the geometry never lies even when the caption is nudged. On the
 * shipped fixtures `avgPct` lands mid-strip and nothing is suppressed; this
 * machinery exists for the athlete the fixtures do not cover, which is why it
 * is unit-tested rather than eyeballed.
 */
export function endpointRow(avgPct: number | null): {
  avgLabelPct: number | null;
  showLowest: boolean;
  showHighest: boolean;
} {
  if (avgPct === null) return { avgLabelPct: null, showLowest: true, showHighest: true };
  const avgLabelPct = Math.min(AVG_LABEL_MAX_PCT, Math.max(AVG_LABEL_MIN_PCT, avgPct));
  return {
    avgLabelPct,
    showLowest: avgLabelPct >= AVG_LABEL_MIN_PCT + ENDPOINT_CLEARANCE_PCT, // ≥ 34
    showHighest: avgLabelPct <= AVG_LABEL_MAX_PCT - ENDPOINT_CLEARANCE_PCT, // ≤ 66
  };
}
```

- [ ] **Step 4: Run the test to verify it passes**

```bash
npx vitest run "app/(app)/dashboard/_components/recovery/resting-strip.test.ts"
```

Expected: PASS, all cases.

- [ ] **Step 5: Run the full gate and commit**

```bash
npm run lint && npm run format:check && npm run typecheck && npm run test && npm run build
git add "app/(app)/dashboard/_components/recovery/resting-strip.ts" \
        "app/(app)/dashboard/_components/recovery/resting-strip.test.ts"
git commit -m "feat(dashboard): add integer-first resting HR ranking and strip geometry"
```

---

## Task 5: Web — a fourth generation of recovery fixtures

**Repo:** `prog-strength-web`, branch `feat/resting-hr-tile`.

**Files:**
- Modify: `app/(app)/dashboard/_components/recovery/fixtures.ts` (append only)
- Modify: `app/(app)/dashboard/_components/recovery/fixtures.test.ts` (append only)

**Context:** `fixtures.ts` already carries three generations (`makeDays`,
`driftingDays`, `bandedDays`), each documented at the top of the file with why
it exists. This adds a fourth, `restingDays`, for the six RHR states. **Every
existing export keeps its series byte-for-byte** — four other tiles' suites read
them.

**The invariant this generation owns:** each view's `restingHrAvg` is the **true
mean of its own thirty pre-today readings**, rounded to 1dp, so a card that
prints the baseline and a card that positions a tick against it stay consistent.
It is derived by the builder *and* asserted by a test — "by construction" is a
property of today's builder, not a law of nature, which is the argument the file
already makes for the drift views.

- [ ] **Step 1: Write the failing test**

Append to `app/(app)/dashboard/_components/recovery/fixtures.test.ts` (add the
new symbols to the existing `./fixtures` import; the file already defines a
local `round(n, dp)` helper mirroring the generator's rounding — reuse it):

```ts
describe("the resting-HR fixtures", () => {
  const views = {
    restingHrView: restingHrView(),
    creepingUpView: creepingUpView(),
    flatMonthView: flatMonthView(),
    restingNoReadingView: restingNoReadingView(),
    restingSparseView: restingSparseView(),
    restingCalibratingView: restingCalibratingView(),
    noMorningsView: noMorningsView(),
  };

  it("each carries a full 31-day date-aligned window ending on RESTING_HR_TODAY", () => {
    for (const [name, view] of Object.entries(views)) {
      expect(view.days, name).toHaveLength(31);
      expect(view.days![30].date, name).toBe(RESTING_HR_TODAY);
    }
  });

  // The property most likely to rot when a series is edited: the tile positions
  // the average tick against the strip's own values, so a fixture whose stated
  // baseline is not the mean of its own history would render a plausible card
  // asserting an impossible payload.
  it("restingHrAvg is the true mean of its own thirty pre-today readings", () => {
    for (const [name, view] of Object.entries(views)) {
      const pre = view
        .days!.slice(0, 30)
        .map((d) => d.restingHr)
        .filter((v): v is number => v !== null);
      if (view.baseline!.restingHrAvg === null) continue; // calibrating: no mean to state
      const mean = round(pre.reduce((a, b) => a + b, 0) / pre.length, 1);
      expect(view.baseline!.restingHrAvg, name).toBe(mean);
    }
  });

  it("restingHrDays is the count behind that mean", () => {
    for (const [name, view] of Object.entries(views)) {
      const readings = view
        .days!.slice(0, 30)
        .filter((d) => d.restingHr !== null).length;
      expect(view.baseline!.restingHrDays, name).toBe(readings);
    }
  });

  it("restingToday agrees with the last day, so hero and rank cannot disagree", () => {
    for (const [name, view] of Object.entries(views)) {
      expect(view.restingToday, name).toBe(view.days![30].restingHr);
    }
  });

  // The DX put 49.6 on the 11th on purpose: a card that prints `49.6 bpm` has
  // failed before it is compared, and a card that RANKS it apart from a 50 has
  // failed just as badly.
  it("the default view carries the DX's float on the 11th", () => {
    expect(restingHrView().days![29].restingHr).toBe(49.6);
  });

  it("the calibrating view has no average and nine mornings behind it", () => {
    const view = restingCalibratingView();
    expect(view.baseline!.restingHrAvg).toBeNull();
    expect(view.baseline!.restingHrDays).toBe(9);
    // Gating on hrvDays instead is the exact bug the recovery_log SOW had to
    // correct, so the two counts are deliberately different here.
    expect(view.baseline!.hrvDays).not.toBe(9);
  });

  it("the sparse view has three readings in its last eight mornings", () => {
    const last8 = restingSparseView()
      .days!.slice(-8)
      .filter((d) => d.restingHr !== null);
    expect(last8).toHaveLength(3);
  });

  it("the no-mornings view has a window but not one reading in it", () => {
    const view = noMorningsView();
    expect(view.days).toHaveLength(31);
    expect(view.days!.every((d) => d.restingHr === null)).toBe(true);
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

```bash
npx vitest run "app/(app)/dashboard/_components/recovery/fixtures.test.ts"
```

Expected: FAIL — the new exports do not exist.

- [ ] **Step 3: Append the fourth generation to `fixtures.ts`**

First extend the file's header docstring with a fourth numbered entry, in the
style of the three already there:

```
 * 4. `restingDays` + the resting-HR views — a window taking an EXPLICIT resting
 *    heart-rate series, anchored on `RESTING_HR_TODAY` (2026-08-12, a
 *    Wednesday, so the recent rows read Today / Tue / Mon), built to the DX's
 *    own numbers so the tile and the DX preview cannot disagree. Each view's
 *    `restingHrAvg` is DERIVED as the true mean of its own thirty pre-today
 *    readings, because the tile positions the average tick against the strip's
 *    own values and a stated-but-wrong baseline would render a plausible card
 *    asserting an impossible payload. Per-day HRV band fields are null as in
 *    (1) and (3) — this tile reads none of them.
```

Then append:

```ts
/**
 * The resting-HR fixtures' "today" — 2026-08-12, a WEDNESDAY and the DX's own
 * headline date, so the tile's three recent rows read `Today / Tue / Mon`.
 */
export const RESTING_HR_TODAY = "2026-08-12";

/** `RESTING_HR_TODAY` minus `back` days, as YYYY-MM-DD, from local date parts. */
function restingIsoDate(back: number): string {
  const [y, m, d] = RESTING_HR_TODAY.split("-").map(Number);
  const date = new Date(y, m - 1, d - back);
  const mm = String(date.getMonth() + 1).padStart(2, "0");
  const dd = String(date.getDate()).padStart(2, "0");
  return `${date.getFullYear()}-${mm}-${dd}`;
}

/**
 * A date-aligned 31-day window taking EXPLICIT resting heart rates,
 * oldest→newest and ending on `RESTING_HR_TODAY`.
 *
 * Explicit rather than generated because the resting-HR tile's whole subject is
 * the SHAPE of a month — flat, climbing, sparse — and a modulus sawtooth cannot
 * state any of them. `makeDays`' `49 + (i % 6)` is a fine filler for a tile that
 * reads resting HR incidentally and useless for the tile that ranks it.
 *
 * A null resting HR is a fully absent morning (no score, no HRV), which is what
 * a strap-off day looks like on the wire. Per-day HRV band fields are left
 * null / "unknown" for the reason `bandedDays` states: a day's own trailing
 * band cannot be known without recomputing it, and this tile reads none of them.
 */
export function restingDays(restingHr: (number | null)[]): RecoveryDayPoint[] {
  const last = restingHr.length - 1;
  return restingHr.map((v, i) => ({
    date: restingIsoDate(last - i),
    hrv: v === null ? null : 80 + (i % 11),
    restingHr: v,
    recoveryScore: v === null ? null : 48 + (i % 30),
    baselineAvg: null,
    balancedLow: null,
    balancedHigh: null,
    zScore: null,
    status: "unknown",
  }));
}

/**
 * The 23 mornings of history behind the DX's own eight recent ones. Hand-
 * authored rather than generated so the default view's baseline lands on the
 * DX's 53.4 and its extremes on the DX's 47 / 59.
 */
const RESTING_HISTORY: number[] = [
  55, 54, 52, 57, 51, 56, 53, 48, 55, 58, 50, 54, 57, 52, 49, 56, 51, 55, 53, 54, 56, 52, 53,
];

/**
 * The DX's eight recent mornings, verbatim. Sunday's 59 is six beats over the
 * average and Monday's 47 is the rebound; today's 50 is unremarkable, which is
 * the point — this is what the tile looks like most days and it should read
 * calm.
 *
 * `49.6` on the 11th is the DX's deliberate float. A card that prints
 * `49.6 bpm` has failed before it is compared, and a card that ranks it apart
 * from the 50 beside it has failed just as badly.
 */
const RESTING_TAIL: (number | null)[] = [51, null, 54, 57, 59, 47, 49.6, 50];

/**
 * Assemble a resting-HR view around a day series, DERIVING `restingHrAvg` and
 * `restingHrDays` from the thirty pre-today mornings so the two cannot disagree
 * with the series the strip is drawn from — the invariant `fixtures.test.ts`
 * pins. `calibrating` forces the average null while leaving the count honest,
 * which is the one state where the server has a sample but no mean.
 *
 * Every non-RHR block is the same athlete as the log fixtures and is spread
 * rather than retyped: this tile reads none of them, and a second set of
 * invented HRV figures is fixture fiction with no reader.
 */
function restingView(days: RecoveryDayPoint[], calibrating = false): RecoveryView {
  const pre = days
    .slice(0, 30)
    .map((d) => d.restingHr)
    .filter((v): v is number => v !== null);
  const last = days[days.length - 1];
  return {
    restingToday: last.restingHr,
    recoveryScore: last.recoveryScore,
    hrvToday: last.hrv,
    // Legacy and gap-omitting — the strip must never draw from this.
    spark: [51, 54, 57, 59, 47, 50],
    days,
    baseline: {
      ...logBaseline(),
      restingHrAvg:
        calibrating || pre.length === 0
          ? null
          : round(pre.reduce((a, b) => a + b, 0) / pre.length, 1),
      restingHrDays: pre.length,
    },
    hrv: {
      status: "unknown",
      balancedLow: 68.1,
      balancedHigh: 108.3,
      zScore: null,
      trend: "steady",
      shortAvg: 86.7,
    },
    baselineTrend: { direction: "steady", deltaMs: 1.2, fromAvg: 87.0, overDays: 28 },
  };
}

/**
 * **default** — an ordinary week with one bump, and the state the tile is
 * judged calm on. Baseline 53.4 over 29 mornings, today 50, extremes 47 and 59.
 * Today ranks 4th lowest of 29 and the card spends NO colour: 50 against a 53.4
 * average rounds to −3, and colour is licensed only upward.
 */
export function restingHrView(): RecoveryView {
  return restingView(restingDays([...RESTING_HISTORY, ...RESTING_TAIL]));
}

/**
 * **creeping-up** — the fixture the tile exists for. Three weeks flat around 48,
 * then 54 / 56 / 57 / 58, and the 30-day average still reads 49 because the
 * climb has not yet dragged it. Today's 58 is the HIGHEST tick: hard against the
 * right end, well right of the dashed average tick, warm, and captioned
 * `30th lowest of your last 30`. The recent rows beneath — 58 / 57 / 56 — supply
 * the "and it has been climbing" that a rank alone cannot.
 */
export function creepingUpView(): RecoveryView {
  const flat = [
    48, 47, 49, 48, 50, 47, 48, 49, 48, 47, 49, 48, 48, 50, 47, 49, 48, 48, 49, 47, 50, 48, 49,
    48, 47, 49, 48,
  ];
  return restingView(restingDays([...flat, 54, 56, 57, 58]));
}

/**
 * **flat-month** — 48–50 for the whole window, and the variant's structural
 * advantage. There is no axis to auto-scale, so the endpoints read `48 lowest`
 * / `50 highest` and two beats of range are instantly readable as *nothing is
 * happening*. Today's 49 is exactly the average, so no colour is spent.
 */
export function flatMonthView(): RecoveryView {
  const month = [
    49, 48, 50, 48, 50, 49, 48, 50, 48, 50, 49, 48, 50, 48, 50, 49, 48, 50, 48, 50, 49, 48, 50,
    48, 50, 49, 48, 50, 49, 49,
  ];
  return restingView(restingDays([...month, 49]));
}

/**
 * **no-reading-yet** — 7am, before the morning webhook lands. The default month
 * with today's reading missing and NOTHING else changed: the baseline stands,
 * the strip and its average tick still draw from the remaining mornings, and
 * yesterday is never promoted into today — the recent rows carry it explicitly
 * one register below, which is where it belongs.
 */
export function restingNoReadingView(): RecoveryView {
  return restingView(restingDays([...RESTING_HISTORY, ...RESTING_TAIL.slice(0, 7), null]));
}

/**
 * **sparse** — three readings in the last eight days, plus one older gap:
 * travel, or the strap on the charger. `n` drops to 24, so the caption must say
 * `of your last 24` rather than claiming thirty, the tick pitch widens
 * accordingly, and two of the three recent rows print `no reading`. Gaps must
 * read as gaps.
 */
export function restingSparseView(): RecoveryView {
  const history = RESTING_HISTORY.map((v, i) => (i === 10 ? null : v));
  return restingView(restingDays([...history, null, 52, null, null, 48, null, null, 50]));
}

/**
 * **calibrating** — `restingHrAvg: null`, `restingHrDays: 9`, and the state this
 * variant was partly selected on: a rank needs no baseline, only a
 * distribution, so the STRIP RENDERS INTACT. No average tick, no average label,
 * no colour, and the caption's reserved second line reads
 * `no avg yet, 9 of 14 mornings`.
 *
 * `hrvDays` is deliberately left at the shared 27. The counts are separate
 * samples on the wire and a tile that borrows the wrong one is the exact bug
 * the recovery_log SOW had to correct in its mockup.
 */
export function restingCalibratingView(): RecoveryView {
  const series = [...RESTING_HISTORY, ...RESTING_TAIL].map((v, i) => (i < 20 ? null : v));
  return restingView(restingDays(series), true);
}

/**
 * **no mornings at all** — `days` present and every `restingHr` null. There is
 * no distribution to rank within and no `n` to divide by, so the card renders a
 * one-line muted body rather than a zero-width strip.
 */
export function noMorningsView(): RecoveryView {
  return restingView(restingDays(Array.from({ length: 31 }, () => null)));
}
```

**Note on placement:** `logBaseline()` and `round()` are already defined in this
file — `logBaseline` near the log fixtures, `round` near the top. Append the new
generation **after** the log fixtures so `logBaseline` is in scope, and do not
move or re-declare either helper.

- [ ] **Step 4: Run the test to verify it passes**

```bash
npx vitest run "app/(app)/dashboard/_components/recovery/"
```

Expected: PASS, including every pre-existing fixture assertion unchanged.

Sanity-check these derived figures appear (they are what Task 6's tests assert):

| View | `restingHrAvg` | `restingHrDays` | `n` | today | rank | lowest / highest | warm? |
| --- | --- | --- | --- | --- | --- | --- | --- |
| `restingHrView` | 53.4 | 29 | 29 | 50 | 4 | 47 / 59 | no |
| `creepingUpView` | 49 | 30 | 30 | 58 | 30 | 47 / 58 | **yes** |
| `flatMonthView` | 49 | 30 | 30 | 49 | 12 | 48 / 50 | no |
| `restingNoReadingView` | 53.4 | 29 | 28 | — | — | 47 / 59 | no |
| `restingSparseView` | 53.4 | 24 | 24 | 50 | 4 | 48 / 58 | no |
| `restingCalibratingView` | null | 9 | 10 | 50 | 2 | 47 / 59 | no |
| `noMorningsView` | null | 0 | 0 | — | — | — | no |

- [ ] **Step 5: Run the full gate and commit**

```bash
npm run lint && npm run format:check && npm run typecheck && npm run test && npm run build
git add "app/(app)/dashboard/_components/recovery/fixtures.ts" \
        "app/(app)/dashboard/_components/recovery/fixtures.test.ts"
git commit -m "test(dashboard): add resting-HR fixtures for the six tile states"
```

---

## Task 6: Web — `RestingRankCard`, the tile itself

**Repo:** `prog-strength-web`, branch `feat/resting-hr-tile`.

**Files:**
- Create: `app/(app)/dashboard/_components/recovery/resting-rank.tsx`
- Create: `app/(app)/dashboard/_components/recovery/resting-rank.test.tsx`

**Context:** four registers inside the standard `MiniCard` shell — hero, strip,
caption, recent mornings. The component name is `RestingRankCard`, **not**
`SortedStripCard`: this family names its files after what the card *says*
(`readiness-verdict`, `morning-ledger`, `balance-band`), and `sorted-strip` is
the DX's vocabulary and should not outlive the DX.

**The card's body height must be constant across all six states.** The caption's
second line is reserved unconditionally, so calibrating does not grow the card
by 12px exactly when the user's dashboard is newest — the same class of defect
as an unreserved chart height, and it moves an entire dashboard row because
`TileGrid` has no span support.

- [ ] **Step 1: Write the failing tests**

Create `app/(app)/dashboard/_components/recovery/resting-rank.test.tsx`:

```tsx
import { render, screen } from "@testing-library/react";
import { describe, expect, it } from "vitest";
import type { RecoveryView } from "@/lib/dashboard";
import { RestingRankCard } from "./resting-rank";
import {
  creepingUpView,
  flatMonthView,
  legacyView,
  noMorningsView,
  restingCalibratingView,
  restingHrView,
  restingNoReadingView,
  restingSparseView,
} from "./fixtures";

const HREF = "/recovery";

function draw(section: RecoveryView) {
  return render(<RestingRankCard section={section} href={HREF} />);
}

/** The card's whole text, whitespace-normalised so JSX line breaks don't bite. */
function text(el: Element): string {
  return (el.textContent ?? "").replace(/\s+/g, " ").trim();
}

/**
 * The same, but joining each direct child's text with a space. `textContent`
 * concatenates adjacent spans with nothing between them, so a row rendering
 * `<span>Tue</span><span>50 bpm</span>` reads "Tue50 bpm" — which is a fact
 * about jsdom, not about the card, and asserting on it would be asserting on
 * the markup rather than on what the user reads.
 */
function parts(el: Element): string {
  return Array.from(el.childNodes)
    .map((n) => (n.textContent ?? "").trim())
    .filter(Boolean)
    .join(" ")
    .replace(/\s+/g, " ");
}

describe("RestingRankCard — default", () => {
  it("prints today's bpm as the hero, with its unit", () => {
    const { container } = draw(restingHrView());
    expect(parts(screen.getByTestId("rhr-hero"))).toBe("50 bpm");
    expect(container.querySelector("a")).toHaveAttribute("href", HREF);
  });

  // An ordinary morning is not an event. The whole colour budget stays unspent.
  it("spends no colour, and keeps the ordinal in --foreground", () => {
    const { container } = draw(restingHrView());
    expect(container.innerHTML).not.toContain("var(--warning)");
    expect(screen.getByTestId("rhr-rank-phrase")).toHaveStyle({
      color: "var(--foreground)",
    });
  });

  it("captions the rank against the honest count of mornings behind it", () => {
    draw(restingHrView());
    expect(text(screen.getByTestId("rhr-caption"))).toBe("4th lowest of your last 29");
  });

  it("prints the extremes at the ends of the strip", () => {
    draw(restingHrView());
    expect(text(screen.getByTestId("rhr-lowest-label"))).toBe("47 lowest");
    expect(text(screen.getByTestId("rhr-highest-label"))).toBe("59 highest");
  });

  it("draws the 30-day average as its own dashed tick, labelled", () => {
    draw(restingHrView());
    expect(screen.getByTestId("rhr-avg-tick")).toBeInTheDocument();
    expect(text(screen.getByTestId("rhr-avg-label"))).toBe("53 avg");
  });

  // The DX's float. A card that prints `49.6 bpm` has failed before it is
  // compared, and one that ranks it apart from the 50 beside it has too.
  it("renders 49.6 as 50 and never prints the float", () => {
    const { container } = draw(restingHrView());
    expect(text(container)).not.toContain("49.6");
    expect(parts(screen.getAllByTestId("rhr-recent-row")[1])).toBe("Tue 50 bpm");
  });

  // Adjacency, not preference: recovery_log prints the same three mornings
  // newest-first in the same register and may sit directly beside this tile.
  it("lists the recent mornings newest-first", () => {
    draw(restingHrView());
    const rows = screen.getAllByTestId("rhr-recent-row");
    expect(rows).toHaveLength(3);
    expect(parts(rows[0])).toBe("Today 50 bpm");
    expect(parts(rows[2])).toBe("Mon 47 bpm");
  });

  // POLARITY PIN. Reversing the sort to descending inverts the card's only
  // statement of "lower is better" while leaving both labels technically
  // correct — the quietest possible way to break this tile.
  it("draws the strip ascending, with `lowest` on the left", () => {
    const { container } = draw(restingHrView());
    const ticks = Array.from(container.querySelectorAll("[data-tick-value]"));
    const values = ticks.map((t) => Number(t.getAttribute("data-tick-value")));
    expect(values).toHaveLength(29);
    expect([...values].sort((a, b) => a - b)).toEqual(values);
    expect(text(screen.getByTestId("rhr-lowest-label"))).toContain("lowest");
  });

  it("describes itself in one sentence for a screen reader", () => {
    draw(restingHrView());
    expect(screen.getByTestId("rhr-strip")).toHaveAttribute(
      "aria-label",
      "Today's resting heart rate of 50 bpm is the 4th lowest of your last 29 mornings, which ranged from 47 to 59 bpm, against a 30-day average of 53.",
    );
  });
});

describe("RestingRankCard — creeping-up", () => {
  it("ranks today at the top of its own month and colours it", () => {
    draw(creepingUpView());
    expect(text(screen.getByTestId("rhr-caption"))).toBe("30th lowest of your last 30");
    expect(screen.getByTestId("rhr-tick-today")).toHaveStyle({
      backgroundColor: "var(--warning)",
    });
    expect(screen.getByTestId("rhr-rank-phrase")).toHaveStyle({ color: "var(--warning)" });
  });

  it("puts today's value at the high end of the strip", () => {
    draw(creepingUpView());
    expect(text(screen.getByTestId("rhr-highest-label"))).toBe("58 highest");
  });

  // A rank alone cannot say "and it has been climbing". The rows can.
  it("shows the climb in the recent rows", () => {
    draw(creepingUpView());
    const rows = screen.getAllByTestId("rhr-recent-row").map(parts);
    expect(rows).toEqual(["Today 58 bpm", "Tue 57 bpm", "Mon 56 bpm"]);
  });
});

describe("RestingRankCard — the colour gate", () => {
  /**
   * CORRECTION 2, and the regression test for it. Today is 55 against a 53.4
   * average — above it, and warm under the SOW's contract — but ranks 19th of
   * 29, which is BELOW the mockup's `rank > (n * 2) / 3` upper-third boundary
   * of 19.33. This test must FAIL against the mockup's rule.
   *
   * The reason the contract wins: the card DRAWS the average tick. Under the
   * upper-third rule today's tick sits visibly right of the dashed average tick
   * and is still painted neutral ink — the card contradicting its own graphic.
   */
  function aboveAverageButMidStrip(): RecoveryView {
    const base = restingHrView();
    const days = [...base.days!];
    days[days.length - 1] = { ...days[days.length - 1], restingHr: 55 };
    return { ...base, days, restingToday: 55 };
  }

  it("colours a morning above the average even when it is not in the upper third", () => {
    draw(aboveAverageButMidStrip());
    expect(text(screen.getByTestId("rhr-caption"))).toBe("19th lowest of your last 29");
    expect(screen.getByTestId("rhr-tick-today")).toHaveStyle({
      backgroundColor: "var(--warning)",
    });
  });

  // isAbove is defined on the PRINTED delta: a difference the card does not
  // print is a difference the card does not colour.
  it("spends no colour when today is exactly the average", () => {
    const { container } = draw(flatMonthView());
    expect(container.innerHTML).not.toContain("var(--warning)");
  });
});

describe("RestingRankCard — flat-month", () => {
  // The variant's structural advantage: no axis to auto-scale, so two beats of
  // range read as "nothing is happening" rather than as a mountain range.
  it("states the whole month's range in two labels", () => {
    draw(flatMonthView());
    expect(text(screen.getByTestId("rhr-lowest-label"))).toBe("48 lowest");
    expect(text(screen.getByTestId("rhr-highest-label"))).toBe("50 highest");
  });
});

describe("RestingRankCard — no reading yet today", () => {
  it("prints an em-dash hero and says so, rather than promoting yesterday", () => {
    draw(restingNoReadingView());
    expect(parts(screen.getByTestId("rhr-hero"))).toBe("— bpm");
    expect(text(screen.getByTestId("rhr-caption"))).toBe("No reading yet today");
    expect(screen.queryByTestId("rhr-tick-today")).toBeNull();
  });

  it("keeps the strip and its average tick drawn from the remaining mornings", () => {
    draw(restingNoReadingView());
    expect(screen.getAllByTestId("rhr-tick")).toHaveLength(28);
    expect(screen.getByTestId("rhr-avg-tick")).toBeInTheDocument();
  });

  it("prints yesterday's value only in the recent-rows register", () => {
    const { container } = draw(restingNoReadingView());
    expect(parts(screen.getAllByTestId("rhr-recent-row")[1])).toBe("Tue 50 bpm");
    for (const row of screen.getAllByTestId("rhr-recent-row")) row.remove();
    expect(text(container)).not.toContain("50");
  });
});

describe("RestingRankCard — sparse", () => {
  it("reports the true number of mornings behind the rank", () => {
    draw(restingSparseView());
    expect(text(screen.getByTestId("rhr-caption"))).toBe("4th lowest of your last 24");
  });

  it("reads a strap-off morning as a gap, in words", () => {
    draw(restingSparseView());
    const rows = screen.getAllByTestId("rhr-recent-row").map(parts);
    expect(rows).toEqual(["Today 50 bpm", "Tue no reading", "Mon no reading"]);
  });
});

describe("RestingRankCard — calibrating", () => {
  // The state this variant was partly selected on: a rank needs no baseline,
  // only a distribution, so the main graphic survives intact.
  it("still draws the strip", () => {
    draw(restingCalibratingView());
    expect(screen.getAllByTestId("rhr-tick").length).toBeGreaterThan(0);
    expect(text(screen.getByTestId("rhr-caption"))).toContain("2nd lowest of your last 10");
  });

  it("draws no average tick, no average label, and no colour", () => {
    const { container } = draw(restingCalibratingView());
    expect(screen.queryByTestId("rhr-avg-tick")).toBeNull();
    expect(screen.queryByTestId("rhr-avg-label")).toBeNull();
    expect(container.innerHTML).not.toContain("var(--warning)");
  });

  // Gated on restingHrDays, never hrvDays — a different sample with a
  // different size.
  it("reports its own metric's progress toward a baseline", () => {
    draw(restingCalibratingView());
    expect(text(screen.getByTestId("rhr-caption"))).toContain("no avg yet, 9 of 14 mornings");
  });

  // jsdom has no layout, so the reserved class IS the assertion.
  it("reserves the caption's second line so the card cannot change height", () => {
    const { unmount } = draw(restingCalibratingView());
    const calibrating = screen.getByTestId("rhr-caption").className;
    unmount();
    draw(restingHrView());
    expect(screen.getByTestId("rhr-caption").className).toBe(calibrating);
    expect(calibrating).toContain("min-h-[28px]");
  });
});

describe("RestingRankCard — degenerate payloads", () => {
  it("renders a muted line, not a zero-width strip, when no morning has a reading", () => {
    const { container } = draw(noMorningsView());
    expect(screen.queryByTestId("rhr-strip")).toBeNull();
    expect(text(container)).toContain("No resting heart rate readings yet");
  });

  it("renders the calibrating body for a legacy payload with no days or baseline", () => {
    const { container } = draw(legacyView());
    expect(screen.queryByTestId("rhr-strip")).toBeNull();
    expect(text(container)).toContain("Resting HR is calibrating");
  });
});

describe("RestingRankCard — the design system", () => {
  // --accent carries `elevated` HRV elsewhere in this family; --danger is
  // licensed for one thing only (a sub-33 Whoop score); and a low morning is
  // never painted green, because most mornings are ordinary.
  it.each([
    ["restingHrView", restingHrView],
    ["creepingUpView", creepingUpView],
    ["flatMonthView", flatMonthView],
    ["restingNoReadingView", restingNoReadingView],
    ["restingSparseView", restingSparseView],
    ["restingCalibratingView", restingCalibratingView],
  ])("spends no accent, danger, or success on %s", (_name, build) => {
    const { container } = draw(build());
    expect(container.innerHTML).not.toContain("var(--accent)");
    expect(container.innerHTML).not.toContain("var(--danger)");
    expect(container.innerHTML).not.toContain("var(--success)");
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

```bash
npx vitest run "app/(app)/dashboard/_components/recovery/resting-rank.test.tsx"
```

Expected: FAIL — cannot resolve `./resting-rank`.

- [ ] **Step 3: Write the component**

Create `app/(app)/dashboard/_components/recovery/resting-rank.tsx`:

```tsx
/**
 * RestingRankCard — the `resting_hr` tile ("Resting HR").
 *
 * Three tiles on this dashboard already print today's resting heart rate. None
 * of them answers *"is this a good morning, for me?"*, because the longest
 * resting-HR history anywhere else in the product is three days — and a resting
 * heart rate that has climbed from 48 to 56 and stayed there for five days is
 * invisible in any window that narrow. This card answers it by RANKING the
 * morning instead of dating it: a user does not actually know whether 50 is good
 * for them (a 50 is excellent for one athlete and elevated for another), and no
 * amount of dated history answers that as directly as showing them where today
 * falls in their own month.
 *
 * Four registers, top to bottom:
 *
 *   1. THE HERO — today's bpm at 20px, repeated at 9px over its own tick below.
 *      Deliberately modest: `morning_vitals` and `recovery` already own today's
 *      number, and if the biggest element here were today's figure the dashboard
 *      would carry a third today-card. This card's job is the MONTH, so the
 *      strip beneath is what the eye should land on.
 *   2. THE STRIP — the last thirty mornings as thin ticks, sorted ascending and
 *      evenly spaced BY RANK, so the strip encodes order and magnitude lives in
 *      the printed extremes. That is what makes a flat month legible without a
 *      chart: `48 lowest` / `50 highest` and no auto-scale to lie about it.
 *   3. THE CAPTION — `4th lowest of your last 30`, the line that makes the whole
 *      card make sense, with the calibrating progress on a permanently reserved
 *      second line.
 *   4. RECENT MORNINGS — the last three newest-first, so chronology is demoted
 *      but not discarded. A rank alone cannot say "and it has been climbing";
 *      `58 / 57 / 56` can.
 *
 * WHY RANKING IS POSITION AND NOT CLASSIFICATION. Resting HR has no band, no
 * z-score, no status and no trend on the wire — `recoverytrend.Compute` derives
 * all of that for HRV only. So this card may not classify a bpm value, and it
 * does not: its only arithmetic is counting how many of the athlete's own
 * mornings sat below today's, which is a POSITION. Nothing here re-averages a
 * series or invents a threshold.
 *
 * WHY THE AVERAGE TICK IS DASHED. It is a statistic, not a morning. It is also
 * computed over a different window from the strip's (30 days trailing excluding
 * today, against the strip's thirty including it), so it claims a position and
 * not membership — and the dash is what says so. Do not "fix" the discrepancy by
 * re-averaging the strip's values; that is the forbidden operation.
 *
 * WHY COLOUR IS GATED ON THE AVERAGE AND NOT THE UPPER THIRD. The DX's idiom
 * description says `--warning` appears only in the athlete's top third; its own
 * colour contract says a reading above the 30-day average is warm. They disagree
 * in a real band, and the contract wins here for one reason above the others:
 * THIS CARD DRAWS THE AVERAGE TICK. Under the upper-third rule today's tick can
 * sit visibly to the right of the dashed average tick and still be painted
 * neutral ink — the card contradicting its own graphic, which is the worst
 * failure available to a tile whose entire claim is that position is the
 * meaning. When there is no average, the card spends no colour at all rather
 * than inventing the verdict the server never made.
 *
 * The colour budget is two elements: today's tick and the caption's ordinal
 * phrase. No `--danger` (a resting HR has no published threshold and inventing
 * one is the forbidden classification), no green for a low morning (most
 * mornings are ordinary), no `--accent` (the periwinkle carries `elevated` HRV
 * elsewhere in this family and there is no equivalent state here).
 *
 * POLARITY. Every other tile on this dashboard means up-is-good; this one
 * inverts it, and it says so SPATIALLY and once — the strip is sorted ascending,
 * so left is better, stated by the endpoint labels and then true everywhere on
 * the card. The caption says `lowest`, never `best`: *lowest* is a fact about
 * the number, *best* is a judgement the server never made.
 */

import type { RecoveryDayPoint, RecoveryView } from "@/lib/dashboard";
import { MiniCard } from "../mini-card";
import { MIN_BASELINE_DAYS, ordinal, round, weekday } from "./shared";
import {
  avgInsertPct,
  endpointRow,
  labelAnchor,
  rankOf,
  sortedMornings,
  STRIP_WINDOW,
  tickPct,
} from "./resting-strip";

const TITLE = "Resting HR";
/** Mornings in the recent register. Matches `recovery_log`, which may sit beside this tile. */
const RECENT_ROWS = 3;
/** The strip's height in px. Fixed, so the card's height cannot move with the data. */
const STRIP_H = 28;

/**
 * The one question colour is allowed to ask: is this morning above the
 * athlete's own 30-day mean? A DIRECTION, never a verdict — `+1` is coloured
 * exactly like `+9`, because both mean the one thing this card may say.
 *
 * Tested on the ROUNDED departure. Every figure here prints as an integer, so a
 * difference the card does not print is a difference the card does not colour.
 */
function isAbove(value: number | null, avg: number | null): boolean {
  if (value === null || avg === null) return false;
  return Math.round(value - avg) > 0;
}

/**
 * The strip's text alternative. Every tick is `aria-hidden` — they are
 * decoration, and the meaning is in the caption and the recent rows, which are
 * real text — so without this the register would be absent from the
 * accessibility tree with nothing standing in for it. Follows `morning-ledger`'s
 * `railLabel` precedent.
 *
 * The two degenerate forms are spelled out rather than interpolated, because a
 * sentence with an em-dash in the middle of it is not a sentence.
 */
function stripLabel(
  today: number | null,
  rank: number | null,
  n: number,
  lowest: number,
  highest: number,
  avg: number | null,
): string {
  const against =
    avg === null ? "with no 30-day average yet." : `against a 30-day average of ${round(avg)}.`;
  if (today === null || rank === null) {
    return `No resting heart rate reading yet today. Your last ${n} mornings ranged from ${lowest} to ${highest} bpm, ${against}`;
  }
  return `Today's resting heart rate of ${round(today)} bpm is the ${ordinal(rank)} lowest of your last ${n} mornings, which ranged from ${lowest} to ${highest} bpm, ${against}`;
}

/** The card body used whenever there is no distribution to rank within. */
function MutedBody({ href, message }: { href: string; message: string }) {
  return (
    <MiniCard title={TITLE} href={href}>
      <p className="text-sm text-[var(--muted)]">{message}</p>
    </MiniCard>
  );
}

export function RestingRankCard({ section, href }: { section: RecoveryView; href: string }) {
  const { days, baseline } = section;

  // One guard at the top for the legacy payload with no derived blocks, exactly
  // as the other four family tiles do. Never `!`-assert the optionals.
  if (!days || !baseline || days.length === 0) {
    return <MutedBody href={href} message="Resting HR is calibrating." />;
  }

  // Today comes from the same array the rank is computed over, so the hero, the
  // filled tick and the caption cannot end up describing different mornings.
  // `days` is date-aligned with nulls preserved and ends on the local today;
  // `spark` omits missing mornings and must never be drawn from.
  const todayValue = days[days.length - 1].restingHr;
  const sorted = sortedMornings(days, STRIP_WINDOW);
  const recent = days.slice(-RECENT_ROWS).reverse();

  // Guard before dividing by `n`. A month with no readings at all is a real
  // payload, and a zero-width strip is not the honest rendering of it.
  if (sorted.length === 0) {
    return <MutedBody href={href} message="No resting heart rate readings yet." />;
  }

  const n = sorted.length;
  const avg = baseline.restingHrAvg;
  const rank = rankOf(sorted, todayValue);
  const avgPct = avgInsertPct(sorted, avg);
  const { avgLabelPct, showLowest, showHighest } = endpointRow(avgPct);
  const warm = isAbove(todayValue, avg);
  const todayPct = rank === null ? null : tickPct(rank - 1, n);
  const lowest = sorted[0];
  const highest = sorted[n - 1];

  return (
    <MiniCard title={TITLE} href={href}>
      {/* One child, not four: MiniCard lays its children out on gap-3, and these
          registers want a tighter 6px seam. The whole body is ~183px, inside the
          DX's ~180px budget and well under its 260px whole-card ceiling. */}
      <div className="flex flex-col gap-1.5">
        <p
          data-testid="rhr-hero"
          className="text-[20px] leading-[22px] tabular-nums tracking-[-0.03em] text-[var(--foreground)]"
        >
          {round(todayValue)}
          <span className="ml-1 text-[10px] tracking-normal text-[var(--muted)]">bpm</span>
        </p>

        <div className="flex flex-col">
          {/* Today's value, repeated over its own tick. Its anchor SWITCHES near
              the ends rather than clamping, so it stays both on the card and
              adjacent to the tick it names. */}
          <div className="relative h-[12px]">
            {todayPct !== null && (
              <span
                data-testid="rhr-today-label"
                className="absolute bottom-0 whitespace-nowrap text-[9px] leading-[11px] tabular-nums text-[var(--foreground)]"
                style={labelAnchor(todayPct)}
              >
                {round(todayValue)}
              </span>
            )}
          </div>

          <div
            role="img"
            aria-label={stripLabel(todayValue, rank, n, lowest, highest, avg)}
            data-testid="rhr-strip"
            className="relative"
            style={{ height: STRIP_H }}
          >
            {sorted.map((value, i) => {
              const isToday = rank !== null && i === rank - 1;
              return (
                <div
                  key={i}
                  aria-hidden="true"
                  data-testid={isToday ? "rhr-tick-today" : "rhr-tick"}
                  data-tick-value={value}
                  className="absolute bottom-0 w-px"
                  style={{
                    left: `${tickPct(i, n)}%`,
                    transform: "translateX(-50%)",
                    // Today is full-height and filled; every other morning is a
                    // shorter hairline, so the strip reads as one marked
                    // position in a field rather than as thirty equal bars.
                    height: isToday ? "100%" : "62%",
                    backgroundColor: isToday
                      ? warm
                        ? "var(--warning)"
                        : "var(--foreground)"
                      : "var(--border-strong)",
                  }}
                />
              );
            })}
            {avgPct !== null && (
              /* DASHED, and drawn at its TRUE position even when its label is
                 clamped — the geometry never lies just because the caption was
                 nudged. Dashed because the average is a statistic and not one of
                 the athlete's mornings. */
              <div
                aria-hidden="true"
                data-testid="rhr-avg-tick"
                className="absolute bottom-0 h-full w-px"
                style={{
                  left: `${avgPct}%`,
                  transform: "translateX(-50%)",
                  background:
                    "repeating-linear-gradient(to bottom, var(--muted) 0 2px, transparent 2px 4px)",
                }}
              />
            )}
          </div>

          {/* The endpoint register. The avg label yields to the card edges and
              wins against the endpoint captions: it is the more informative
              figure, and the extremes stay visible as the outermost ticks even
              without their words. */}
          <div className="relative h-[12px] text-[9px] leading-[11px] tabular-nums text-[var(--faint)]">
            {showLowest && (
              <span data-testid="rhr-lowest-label" className="absolute left-0 top-0">
                {lowest} lowest
              </span>
            )}
            {avgLabelPct !== null && avg !== null && (
              <span
                data-testid="rhr-avg-label"
                className="absolute top-0 whitespace-nowrap text-[var(--muted)]"
                style={{ left: `${avgLabelPct}%`, transform: "translateX(-50%)" }}
              >
                {round(avg)} avg
              </span>
            )}
            {showHighest && (
              <span data-testid="rhr-highest-label" className="absolute right-0 top-0">
                {highest} highest
              </span>
            )}
          </div>
        </div>

        {/* The caption's second line is reserved UNCONDITIONALLY. Appending the
            calibrating progress inline wraps at a one-third desktop cell and
            grows the card by a line exactly when the user's dashboard is
            newest; an empty reserved line is invisible and a 12px jump is not.
            `TileGrid` has no span support, so a tile that changes height moves
            its whole row, including unrelated tiles. */}
        <div data-testid="rhr-caption" className="flex min-h-[28px] flex-col gap-[2px]">
          <p className="text-[10px] leading-[13px] text-[var(--muted)]">
            {rank === null ? (
              // Yesterday is never promoted into today. The recent rows carry
              // the last actual reading one register below, which is where it
              // belongs — a hero that silently changes what it means is how a
              // user learns to distrust a figure.
              "No reading yet today"
            ) : (
              <>
                {/* `lowest`, never `best`: one is a fact about the number, the
                    other is a judgement the server never made. */}
                <span
                  data-testid="rhr-rank-phrase"
                  style={{ color: warm ? "var(--warning)" : "var(--foreground)" }}
                >
                  {ordinal(rank)} lowest
                </span>{" "}
                of your last {n}
              </>
            )}
          </p>
          <p className="min-h-[13px] text-[10px] leading-[13px] text-[var(--faint)]">
            {avg === null
              ? `no avg yet, ${baseline.restingHrDays} of ${MIN_BASELINE_DAYS} mornings`
              : ""}
          </p>
        </div>

        <div className="flex flex-col divide-y divide-[var(--border)] border-t border-[var(--border)]">
          {recent.map((day, i) => (
            <RecentRow key={day.date} day={day} isToday={i === 0} />
          ))}
        </div>
      </div>
    </MiniCard>
  );
}

/** One recent morning. Newest-first, matching `recovery_log`'s same register. */
function RecentRow({ day, isToday }: { day: RecoveryDayPoint; isToday: boolean }) {
  // Positional, by the payload's contract that `days` ends on the local today.
  const label = isToday ? "Today" : weekday(day.date);
  return (
    <div data-testid="rhr-recent-row" className="flex items-baseline justify-between py-1">
      <span className="text-[10px] text-[var(--faint)]">{label}</span>
      {day.restingHr === null ? (
        // The row has space for the word, and a sparse month should read as
        // intentional rather than as a column of em-dashes.
        <span className="text-[10px] italic text-[var(--faint)]">no reading</span>
      ) : (
        <span className="text-[10px] tabular-nums tracking-[-0.01em] text-[var(--foreground)]">
          {round(day.restingHr)}
          <span className="text-[var(--faint)]"> bpm</span>
        </span>
      )}
    </div>
  );
}
```

- [ ] **Step 4: Run the tests to verify they pass**

```bash
npx vitest run "app/(app)/dashboard/_components/recovery/"
```

Expected: PASS. If a figure disagrees, **check the fixture table in Task 5
before changing an assertion** — the numbers there were computed from the
series, and a mismatch means the series was mistyped, not that the test is wrong.

- [ ] **Step 5: Run the full gate and commit**

```bash
npm run lint && npm run format:check && npm run typecheck && npm run test && npm run build
git add "app/(app)/dashboard/_components/recovery/resting-rank.tsx" \
        "app/(app)/dashboard/_components/recovery/resting-rank.test.tsx"
git commit -m "feat(dashboard): add the resting HR rank card"
```

---

## Task 7: Web — the catalog entry and the renderer case

**Repo:** `prog-strength-web`, branch `feat/resting-hr-tile`.

**Files:**
- Modify: `lib/dashboard-tiles.ts`
- Modify: `lib/dashboard-tiles.test.ts`
- Modify: `app/(app)/dashboard/_components/tile-renderer.tsx`
- Modify: `app/(app)/dashboard/_components/tile-renderer.test.tsx`

**Context:** this is the mirror of Task 1. `lib/dashboard-tiles.ts` says in its
own header that the Go `Catalog` and this `TILE_CATALOG` must stay identical in
id set and order, and both sides' contract tests enforce it — so the id goes in
**the same position**, immediately after `recovery_log` and before `sleep`.
`tile-renderer.tsx`'s `never` default is the compile-time guard the SOW relies
on: adding the id to the union without adding the case is a type error.

- [ ] **Step 1: Write the failing tests**

In `lib/dashboard-tiles.test.ts`:

- change `expect(TILE_CATALOG.length).toBe(20)` to `21`, and rename the test to
  `"has exactly 21 tiles"`;
- insert `"resting_hr"` into the order list between `"recovery_log"` and
  `"sleep"`;
- insert `resting_hr: true,` into `ALL_TILE_IDS` in the same position;
- extend the family test — it currently asserts four distinct titles and
  descriptions, and there are now five:

```ts
  test("the five recovery-family tiles have distinct titles and descriptions", () => {
    const family = ["recovery", "hrv_balance", "morning_vitals", "recovery_log", "resting_hr"];
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

- add the tray-order pin, because catalog order **is** the add-tile tray order
  and the family staying contiguous is the reason this id sits where it does:

```ts
  test("the recovery family is a contiguous run ending in resting_hr, before sleep", () => {
    const ids = TILE_CATALOG.map((t) => t.id);
    const family = ["recovery", "hrv_balance", "morning_vitals", "recovery_log", "resting_hr"];
    const at = ids.indexOf("recovery");
    expect(ids.slice(at, at + family.length)).toEqual(family);
    expect(ids[at + family.length]).toBe("sleep");
  });
```

- and pin the title's storage form, since `MiniCardTitle` is what shouts it:

```ts
  test("the resting_hr title is stored in sentence-plus-initialism case", () => {
    // MiniCardTitle uppercases for display, so the catalog never stores a
    // shouted string — the same way every other entry is stored.
    expect(tileEntry("resting_hr").title).toBe("Resting HR");
  });
```

In `app/(app)/dashboard/_components/tile-renderer.test.tsx`, add the id to the
existing `FAMILY` array so both the present and absent cases cover it:

```tsx
  const FAMILY: [TileId, string][] = [
    ["recovery", "Recovery"],
    ["hrv_balance", "HRV Balance"],
    ["morning_vitals", "Morning Vitals"],
    ["recovery_log", "Recovery Log"],
    ["resting_hr", "Resting HR"],
  ];
```

- [ ] **Step 2: Run the tests to verify they fail**

```bash
npx vitest run lib/dashboard-tiles.test.ts "app/(app)/dashboard/_components/tile-renderer.test.tsx"
```

Expected: FAIL — 20 ≠ 21, `resting_hr` is not in the union, no card renders.

- [ ] **Step 3: Add the catalog entry**

In `lib/dashboard-tiles.ts`, add to the `TileId` union between `"recovery_log"`
and `"sleep"`:

```ts
  | "recovery_log"
  | "resting_hr"
  | "sleep"
```

and the entry to `TILE_CATALOG` in the same position:

```ts
  {
    id: "resting_hr",
    title: "Resting HR",
    href: "/recovery",
    description: "Where this morning's resting heart rate ranks in your own month.",
  },
```

- [ ] **Step 4: Add the renderer case**

In `tile-renderer.tsx`, import the card beside its siblings:

```ts
import { RestingRankCard } from "./recovery/resting-rank";
```

and add the case, following the family pattern exactly, after `recovery_log`:

```tsx
    case "resting_hr":
      return data.recovery.present ? (
        <RestingRankCard section={data.recovery} href={href} />
      ) : (
        <RecoveryConnectCard title="Resting HR" href={href} />
      );
```

Update the file's header docstring: the recovery family paragraph currently
names four tiles ("recovery, hrv_balance, morning_vitals, recovery_log") and
says "All four read the one shared `recovery` section". Make it five, and note
that `resting_hr` ships tray-only rather than in the default layout.

- [ ] **Step 5: Run the tests to verify they pass**

```bash
npx vitest run lib/dashboard-tiles.test.ts "app/(app)/dashboard/_components/tile-renderer.test.tsx"
```

Expected: PASS. `app/(app)/dashboard/page.test.tsx` must also pass
**unmodified** — run the whole suite to confirm.

- [ ] **Step 6: Run the full gate and commit**

```bash
npm run lint && npm run format:check && npm run typecheck && npm run test && npm run build
git add lib/dashboard-tiles.ts lib/dashboard-tiles.test.ts \
        "app/(app)/dashboard/_components/tile-renderer.tsx" \
        "app/(app)/dashboard/_components/tile-renderer.test.tsx"
git commit -m "feat(dashboard): add resting_hr to the tile catalog and renderer"
```

---

## Final verification (after all seven tasks)

- [ ] **API**: the full gate, green, from a clean tree.

```bash
cd /workspace/prog-strength-api
export CGO_CFLAGS="-I/tmp/sqliteinc"
go build ./... && go vet ./... && \
  go run github.com/golangci/golangci-lint/v2/cmd/golangci-lint@v2.12.2 run --timeout=5m && \
  go mod tidy && git diff --exit-code go.mod go.sum && go test ./...
```

- [ ] **Web**: the full gate, green, from a clean tree.

```bash
cd /workspace/prog-strength-web
npm run lint && npm run format:check && npm run typecheck && npm run test && npm run build
```

- [ ] **Cross-repo contract**: the Go `Catalog` and the TS `TILE_CATALOG` are
  identical in id set and order. Confirm by eye:

```bash
grep -A 7 '^var Catalog' /workspace/prog-strength-api/internal/dashboard/tiles.go
grep -n 'id: "' /workspace/prog-strength-web/lib/dashboard-tiles.ts
```

Both must read `… recovery, hrv_balance, morning_vitals, recovery_log,
resting_hr, sleep, streak, quote, weather`, 21 entries.

- [ ] **Non-goals audit** — confirm none of these was touched:
  `RecoveryView` / `RecoveryDayPoint` / `RecoveryBaselineView` in
  `lib/dashboard.ts`; `internal/dashboard/dto.go`; `internal/recoverytrend/**`;
  `layout_resolve.go`'s `defaultLayout`; `app/design-explore/**`;
  `prog-strength-mobile`; `design-system.md`; the other four recovery tiles'
  behaviour; `/recovery` deep-page components.

```bash
git diff --stat main...HEAD
cd /workspace/prog-strength-api && git diff --stat main...HEAD
```

- [ ] **Docs**: flip `sows/resting-hr-tile.md` to shipped on
  `prog-strength-docs`' `feat/resting-hr-tile` branch — frontmatter
  `status: shipped`, body `**Status**: Shipped`, `**Last updated**: 2026-08-12`.
  The DX needs **no** edit; it is already `selected` with the amended
  "no prerequisite" blockquote.

- [ ] **Open PRs** — API first in the merge order, then web, then docs.
