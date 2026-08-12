# Recovery Log Tile Redesign — `band-rail-and-recent` Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace `MorningLedgerCard`'s body with the `band-rail-and-recent` two-register composition — a 14-day recovery-score rail read against a dashed baseline tick, a seam carrying the 30-day baseline as `score · bpm · ms`, and the three most recent mornings in full detail — correcting the four DX binding defects and the five mockup defects the SOW names.

**Architecture:** All three registers stay in the one component file (`recovery/morning-ledger.tsx`); the rail is ~30 lines and is not reusable until `/recovery` wants it. The band vocabulary (`recoveryBand` / `recoveryBandWord` / `recoveryBandColor`) and one display formatter (`round`) graduate into `recovery/shared.ts` beside `hrvStatusColor` / `statusWord`, strictly additively. `railY` — the 0–100 score → pixel map shared by the bars **and** the tick — is exported from the component so a test can pin the two to each other. Score-bearing fixtures are added to `recovery/fixtures.ts`; existing exports are untouched because four other tiles' suites read them.

**Tech Stack:** Next.js 16 App Router, React 19, TypeScript, Tailwind v4 (CSS-variable tokens), Vitest + Testing Library (jsdom).

---

## Source documents (read before any task)

- SOW: `sows/recovery-log-tile-redesign.md` in `prog-strength-docs`.
- DX: `dx/recovery-log-tile.md` — already `status: selected` with
  `selected_idioms: [band-rail-and-recent]` and the selection note in place.
  **No DX edit is needed by this plan.**
- Design system: `design-system.md` **v0.4.4**.

## Design-system & SOW constraints (binding on every task)

- **Tokens only, no raw hex.** The tile may use `--foreground`, `--muted`, `--faint`,
  `--surface-2`, `--border`, `--success`, `--warning`, `--danger`. `scope: in-system` —
  no new token, no palette or type divergence.
- **`--accent` never appears on this tile.** The periwinkle is app chrome and, via
  `hrvStatusColor`, the *elevated HRV* colour. This tile paints **no HRV status at all**,
  so `--accent` in its markup is a bug and a test asserts its absence.
- **`--danger` at text and mark weight only** — a red word, a red bar. Never a filled row.
- **Type**: Manrope (app default). Nothing on the card above **13px**. `tabular-nums` on
  every figure; `font-mono` on the `bpm · ms` group. Numerals carry tight tracking
  (`tracking-[-0.03em]`), per the design system.
- **`MiniCard`'s panel, radius, and padding are not overridden.**
- **Nothing recomputes a server figure.** The only arithmetic in the tile is `railY`
  (score → pixels) and `Math.round` for display. No deltas, no averages, no z-scores.
- **Never draw from `spark`** — it omits missing days and destroys the date alignment the
  rail depends on. Always `days`.
- **Additive only in `shared.ts` and `fixtures.ts`.** No existing export changes signature
  or value. `dashboard/page.test.tsx`, `tile-renderer.test.tsx`, `hrv-tile.test.tsx`,
  `balance-band.test.tsx`, `trend-rail.test.tsx`, `readiness-verdict.test.tsx`,
  `three-dial-vitals.test.tsx` and `fixtures.test.ts`'s existing blocks must pass
  **unmodified**. If one needs editing, something in scope has been disturbed — report it
  rather than editing it.

## Decisions resolved from the SOW's Open Questions and corrections

These are settled. Do not re-litigate them inside a task.

1. **Detail register on `sparse`** (SOW Open Question 1): **keep the last three calendar
   days**, even when all three are empty. Both registers then cover the same window; the
   rail is already carrying the "there is history here" message.
2. **`MIN_BASELINE_DAYS = 14` stays a client-side literal** (SOW Open Question 2), but as
   a named constant with a comment saying the real value is
   `config.Recovery.MinBaselineDays` on the API.
3. **`recoveryBand*` is not applied to `recovery` or `morning_vitals`** (SOW Open
   Question 3). Those tiles are not touched.
4. **The baseline tick is `--muted` at 50% opacity, not `--border-strong`.** The SOW
   pre-authorises this step-up and makes it conditional on the tick reading over a
   full-strength `--success` bar. It does not: `--border-strong` is
   `rgba(255,255,255,0.1)`, which composited over `--success` `#86b39f` yields `#92bba9`
   — a **1.10:1** contrast ratio against the bar, i.e. invisible, and invisible exactly at
   the moment the fortnight is good. `--muted` `#7d818c` at 0.5 composites to `#829a96`
   (**1.28:1** against the bar, **2.09:1** against `--surface`), which reads on both. The
   SOW's own words: *"a structural datum is not chrome."* Put those numbers in a code
   comment so nobody "restores" the hairline.
5. **An absent morning is a full-rail-height ghost column**, `--surface-2` at 60% opacity,
   never a 2px stub. It differs from a low score in **kind**, not in height.
6. **The calibrating gate is `baseline.recoveryScoreAvg === null`** and the progress line
   counts `baseline.recoveryScoreDays` and says **mornings**, not nights.
7. **A missing score is handled at the field level**, not the row level: the row is a
   *no reading* row only when all three readings are null; otherwise a null score prints
   `—` and the **band word is omitted entirely**.

## File Structure

| File | Change | Responsibility after this plan |
| --- | --- | --- |
| `app/(app)/dashboard/_components/recovery/shared.ts` | **Extended (additive)** | Gains `RecoveryBand`, `recoveryBand`, `recoveryBandWord`, `recoveryBandColor`, `round`. Still pure, still no React. |
| `app/(app)/dashboard/_components/recovery/shared.test.ts` | **Extended (additive)** | Gains four `describe` blocks pinning the cut points, the words, the tokens, and the formatter. |
| `app/(app)/dashboard/_components/recovery/fixtures.ts` | **Extended (additive)** | Gains `RECOVERY_LOG_TODAY`, `bandedDays`, `recoveryLogView`, `sparseView`, `partialMorningView`. |
| `app/(app)/dashboard/_components/recovery/fixtures.test.ts` | **Extended (additive)** | Gains one `describe` pinning the new generator and views. |
| `app/(app)/dashboard/_components/recovery/morning-ledger.tsx` | **Rewritten** | The rail, the seam, the detail rows, the guard, the calibrating state. Exports `MorningLedgerCard` and `railY`. |
| `app/(app)/dashboard/_components/recovery/morning-ledger.test.tsx` | **Rewritten** | Every current assertion describes behaviour this plan deletes. |
| `lib/dashboard-tiles.ts` | **One line** | The `recovery_log` tray description. |

Nothing else in either repo is touched. In particular: no `tile-renderer.tsx` change, no
`TileId` change, no `href` change, no `/recovery` page change, no `app/design-explore/**`.

---

## Task 1: The band vocabulary and the display formatter in `shared.ts`

**Files:**
- Modify: `app/(app)/dashboard/_components/recovery/shared.ts` (append; touch nothing above)
- Test: `app/(app)/dashboard/_components/recovery/shared.test.ts` (append; touch nothing above)

Four new exports, single-sourced here for the reason the file's own header already gives:
three hand-rolled copies of a threshold switch is how `52` ends up green on one tile and
yellow on another. This is a **labelling**, not a recomputation — naming which third of
Whoop's fixed, published 0–100 scale a score falls in introduces no statistics, exactly as
`weekday()` introduces no calendar.

- [ ] **Step 1: Write the failing tests**

Append to `app/(app)/dashboard/_components/recovery/shared.test.ts`, and add
`recoveryBand`, `recoveryBandColor`, `recoveryBandWord` and `round` to the **existing**
import list from `"./shared"` (keep it alphabetised, as it is today):

```ts
describe("recoveryBand — Whoop's published cut points, single-sourced", () => {
  test("green at and above 67", () => {
    expect(recoveryBand(100)).toBe("green");
    expect(recoveryBand(67)).toBe("green");
  });

  test("yellow from 34 to 66 — the boundaries are the whole point of this function", () => {
    expect(recoveryBand(66)).toBe("yellow");
    expect(recoveryBand(34)).toBe("yellow");
  });

  test("red at and below 33", () => {
    expect(recoveryBand(33)).toBe("red");
    expect(recoveryBand(0)).toBe("red");
  });

  test("no score is no band, never a red zero", () => {
    expect(recoveryBand(null)).toBe("none");
  });
});

describe("recoveryBandWord", () => {
  test("the words Whoop trained its users to read beside the score", () => {
    expect(recoveryBandWord("green")).toBe("Recovered");
    expect(recoveryBandWord("yellow")).toBe("Adequate");
    expect(recoveryBandWord("red")).toBe("Low");
    expect(recoveryBandWord("none")).toBe("No reading");
  });
});

describe("recoveryBandColor — the one deliberate exception to the no-danger-red contract", () => {
  test("red is --danger and must not be 'fixed' to warning", () => {
    // A sub-33 recovery score is Whoop's own red shown in Whoop's own app.
    // Softening it means the log disagrees with the device the number came from.
    expect(recoveryBandColor("red")).toBe("var(--danger)");
  });

  test("green is --success and yellow is --warning", () => {
    expect(recoveryBandColor("green")).toBe("var(--success)");
    expect(recoveryBandColor("yellow")).toBe("var(--warning)");
  });

  test("no band takes muted ink, so an absence is never a colour", () => {
    expect(recoveryBandColor("none")).toBe("var(--muted)");
  });

  test("every band is a token, never a raw hex", () => {
    for (const band of ["green", "yellow", "red", "none"] as const) {
      expect(recoveryBandColor(band)).toMatch(/^var\(--[a-z-]+\)$/);
    }
  });

  test("no band borrows the accent — this family paints no HRV status", () => {
    for (const band of ["green", "yellow", "red", "none"] as const) {
      expect(recoveryBandColor(band)).not.toBe("var(--accent)");
    }
  });
});

describe("round — a server figure rounded for display, never re-derived", () => {
  test("the Whoop RMSSD float that wraps the shipped row becomes an integer", () => {
    expect(round(96.10188)).toBe("96");
    expect(round(112.44031)).toBe("112");
  });

  test("a non-integral recovery score rounds too — it is a float on the wire", () => {
    expect(round(52.4)).toBe("52");
  });

  test("absent is an em dash, not a zero", () => {
    expect(round(null)).toBe("—");
  });
});
```

- [ ] **Step 2: Run the tests and verify they fail**

Run: `npx vitest run "app/(app)/dashboard/_components/recovery/shared.test.ts"`
Expected: FAIL — `recoveryBand is not a function` / the new imports do not resolve.

- [ ] **Step 3: Append the implementation to `shared.ts`**

Append at the end of `app/(app)/dashboard/_components/recovery/shared.ts` (after
`weekday`). Do not modify anything above it:

```ts
/** Which third of Whoop's published 0–100 recovery scale a score falls in. */
export type RecoveryBand = "green" | "yellow" | "red" | "none";

/**
 * Which third of Whoop's published 0–100 recovery scale a score falls in, at
 * Whoop's own published cut points (67 and 33). This is a LABELLING, not a
 * recomputation: the house rule forbids re-deriving a server figure — an
 * average, a band bound, a z-score — and naming a third of a fixed scale
 * introduces no statistics. Single-sourced here for the reason this file
 * already exists: three hand-rolled copies of a threshold switch is how `52`
 * ends up green on one tile and yellow on another.
 */
export function recoveryBand(score: number | null): RecoveryBand {
  if (score === null) return "none";
  if (score >= 67) return "green";
  if (score <= 33) return "red";
  return "yellow";
}

/** The word Whoop trained every one of its users to read beside the score. */
export function recoveryBandWord(band: RecoveryBand): string {
  switch (band) {
    case "green":
      return "Recovered";
    case "yellow":
      return "Adequate";
    case "red":
      return "Low";
    default:
      return "No reading";
  }
}

/**
 * Re-toned to v0.4, never Whoop's saturated traffic light. Red really is
 * `--danger`, and the recovery SCORE is the one place in this family that is
 * allowed: this file's contract that a suppressed HRV morning reads `--warning`
 * ("information, not an emergency") still holds for HRV, but a sub-33 recovery
 * score is Whoop's own red shown in Whoop's own app, and softening it means the
 * log disagrees with the device the number came from. Text and mark weight
 * only — a red word or a red bar, never a red row.
 */
export function recoveryBandColor(band: RecoveryBand): string {
  switch (band) {
    case "green":
      return "var(--success)";
    case "yellow":
      return "var(--warning)";
    case "red":
      return "var(--danger)";
    default:
      return "var(--muted)";
  }
}

/**
 * A server figure rounded for display, or `—`. Never re-derived, only rounded.
 *
 * Every figure on the recovery log goes through this one formatter. `hrv` is a
 * raw Whoop RMSSD float (`96.10188`) and `recoveryScore` is a nullable float on
 * the wire, so printing either raw is what breaks a row onto two lines.
 */
export function round(v: number | null): string {
  return v === null ? "—" : String(Math.round(v));
}
```

- [ ] **Step 4: Run the tests and verify they pass**

Run: `npx vitest run "app/(app)/dashboard/_components/recovery/shared.test.ts"`
Expected: PASS, including every pre-existing block in the file.

- [ ] **Step 5: Run the whole recovery suite for regressions**

Run: `npx vitest run "app/(app)/dashboard/_components/recovery/"`
Expected: PASS. The four sibling tiles read `shared.ts`; nothing they use has changed.

- [ ] **Step 6: Commit**

```bash
git add "app/(app)/dashboard/_components/recovery/shared.ts" \
        "app/(app)/dashboard/_components/recovery/shared.test.ts"
git commit -m "feat(dashboard): single-source the whoop recovery band vocabulary"
```

---

## Task 2: Score-bearing fixtures

**Files:**
- Modify: `app/(app)/dashboard/_components/recovery/fixtures.ts` (append; existing exports byte-for-byte unchanged)
- Test: `app/(app)/dashboard/_components/recovery/fixtures.test.ts` (append one `describe`)

The gap: `makeDays` derives `recoveryScore` as `48 + (i % 30)`, a sawtooth spanning 48–77.
It produces yellows and greens and **never a red**, so the existing fixtures cannot express
the default fixture's story — a bad weekend — at all.

These fixtures anchor their window to a **new** `RECOVERY_LOG_TODAY` (`2026-08-11`, a
Tuesday) rather than the family's `FIXTURE_TODAY` (`2026-08-01`, a Saturday), because the
DX's story is told in weekday labels: Sat 35 → Sun 29 → Mon 52 → today, no reading. On a
Saturday "today" the tests would assert on a weekend that is not the one the story is
about. `FIXTURE_TODAY` and every existing view keep their dates exactly.

- [ ] **Step 1: Write the failing tests**

Append to `app/(app)/dashboard/_components/recovery/fixtures.test.ts`, adding
`bandedDays`, `partialMorningView`, `recoveryLogView`, `RECOVERY_LOG_TODAY` and
`sparseView` to the **existing** import list from `"./fixtures"`:

```ts
/**
 * The score-bearing fixtures, added for the recovery-log rail. They exist
 * because `makeDays` derives its score as `48 + (i % 30)` and therefore never
 * produces a red morning — the one thing the log's headline fixture is about.
 * Pinned here for the same reason `driftingDays` is: a generator that
 * hand-authors a series is a thing the tile should not have to discover.
 */
describe("the score-bearing fixtures", () => {
  it("dates a banded series so its last day is the log fixtures' today", () => {
    const days = bandedDays([61, 52, 44]);
    expect(days.map((d) => d.date)).toEqual(["2026-08-09", "2026-08-10", RECOVERY_LOG_TODAY]);
  });

  it("keeps a null score as a fully absent morning", () => {
    const [day] = bandedDays([null]);
    expect(day.recoveryScore).toBeNull();
    expect(day.restingHr).toBeNull();
    expect(day.hrv).toBeNull();
  });

  it("leaves every per-day band field unset — the log reads none of them", () => {
    for (const day of bandedDays([61, null, 44])) {
      expect(day.baselineAvg).toBeNull();
      expect(day.balancedLow).toBeNull();
      expect(day.balancedHigh).toBeNull();
      expect(day.zScore).toBeNull();
      expect(day.status).toBe("unknown");
    }
  });

  it("tells the DX's story: a red weekend, a rebound Monday, no reading yet today", () => {
    const days = recoveryLogView().days!;
    expect(days).toHaveLength(31);
    expect(days.slice(-4).map((d) => d.recoveryScore)).toEqual([35, 29, 52, null]);
    expect(days.slice(-4).map((d) => d.date)).toEqual([
      "2026-08-08",
      "2026-08-09",
      "2026-08-10",
      "2026-08-11",
    ]);
  });

  it("carries the deliberately ugly Whoop floats verbatim — they are the wrap", () => {
    const hrv = recoveryLogView().days!.map((d) => d.hrv);
    expect(hrv).toContain(96.10188);
    expect(hrv).toContain(112.44031);
    expect(hrv).toContain(83.242966);
  });

  it("puts exactly two ghost slots in the log fixture's fortnight", () => {
    // The strap-off day and this morning, which has not landed yet. The rail's
    // own test computes this from the payload; pinned here so a fixture tweak
    // that quietly removes the gap is caught in the file that caused it.
    const window = recoveryLogView().days!.slice(-14);
    expect(window.filter((d) => d.recoveryScore === null)).toHaveLength(2);
    expect(window.filter((d) => d.recoveryScore !== null)).toHaveLength(12);
  });

  it("gives the sparse view three readings in its last eight days", () => {
    const days = sparseView().days!;
    expect(days.slice(-8).filter((d) => d.recoveryScore !== null)).toHaveLength(3);
    // The three most recent calendar days are all empty — the detail register's
    // worst state, and the one the idiom was selected on.
    expect(days.slice(-3).every((d) => d.recoveryScore === null)).toBe(true);
    // A week ago there IS history, so the rail is visibly not an empty tile.
    expect(days.slice(-14, -8).every((d) => d.recoveryScore !== null)).toBe(true);
  });

  it("gives the partial morning a resting HR and an HRV but no score", () => {
    const today = partialMorningView().days!.at(-1)!;
    expect(today.recoveryScore).toBeNull();
    expect(today.restingHr).toBe(54);
    expect(today.hrv).toBe(90.5127);
  });
});
```

- [ ] **Step 2: Run the tests and verify they fail**

Run: `npx vitest run "app/(app)/dashboard/_components/recovery/fixtures.test.ts"`
Expected: FAIL — the new imports do not resolve.

- [ ] **Step 3: Append the fixtures**

Append at the end of `app/(app)/dashboard/_components/recovery/fixtures.ts`. Do not modify
anything above it — `makeDays`, `isoDate`, `baseline()`, `driftingDays` and all fourteen
existing views stay byte-for-byte:

```ts
/**
 * The recovery-log fixtures' "today" — 2026-08-11, a TUESDAY, and the DX's own
 * headline date. These views are anchored here rather than on `FIXTURE_TODAY`
 * (2026-08-01, a Saturday) because the log's story is told in weekday labels:
 * a bad Saturday and Sunday, a rebound Monday, and no reading yet today. On a
 * Saturday "today" the tests would assert on the wrong weekend.
 */
export const RECOVERY_LOG_TODAY = "2026-08-11";

/** `RECOVERY_LOG_TODAY` minus `back` days, as YYYY-MM-DD, from local date parts. */
function logIsoDate(back: number): string {
  const [y, m, d] = RECOVERY_LOG_TODAY.split("-").map(Number);
  const date = new Date(y, m - 1, d - back);
  const mm = String(date.getMonth() + 1).padStart(2, "0");
  const dd = String(date.getDate()).padStart(2, "0");
  return `${date.getFullYear()}-${mm}-${dd}`;
}

/**
 * A date-aligned day series taking EXPLICIT recovery scores, oldest→newest and
 * ending on `RECOVERY_LOG_TODAY`, so a test can state the band sequence it is
 * asserting on rather than reading it out of a modulus. `makeDays`'s
 * `48 + (i % 30)` sawtooth never produces a red morning, which is precisely the
 * state the recovery log exists to make legible.
 *
 * A null score is a fully absent morning — no resting HR, no HRV — which is
 * what a strap-off day looks like on the wire. The per-day band fields are left
 * null / "unknown" for the same reason `makeDays` leaves them so: a day's own
 * trailing band cannot be known without recomputing it, and this tile reads
 * none of them anyway.
 */
export function bandedDays(scores: (number | null)[]): RecoveryDayPoint[] {
  const last = scores.length - 1;
  return scores.map((score, i) => ({
    date: logIsoDate(last - i),
    hrv: score === null ? null : 80 + (i % 11),
    restingHr: score === null ? null : 49 + (i % 6),
    recoveryScore: score,
    baselineAvg: null,
    balancedLow: null,
    balancedHigh: null,
    zScore: null,
    status: "unknown",
  }));
}

/** The 23 days of history behind the DX fixture's eight on-screen mornings. */
const LOG_HISTORY: (number | null)[] = [
  62, 58, 71, 49, 55, 64, 70, 44, 59, 66, 53, 61, 74, 47, 57, 63, 51, 68, 60, 45, 56, 72, 54,
];

/**
 * The DX's eight on-screen mornings, verbatim. The deliberately ugly floats
 * (`112.44031`, `83.242966`, `96.10188`) are real Whoop values and they are
 * here on purpose: a tile that wraps on them has failed before it is compared.
 *
 * Read as: Sunday's 29 is the story. A red morning, resting HR six beats above
 * a 53 baseline, HRV ordinary — the classic under-recovered-but-not-sick
 * signature — and Saturday's 35 says it was two days, not one. Monday's 52 with
 * a 47 bpm resting HR is the rebound, and today has not landed yet.
 */
const LOG_TAIL: ReadonlyArray<Pick<RecoveryDayPoint, "restingHr" | "recoveryScore" | "hrv">> = [
  { restingHr: 61, recoveryScore: 41, hrv: 61.0 },
  { restingHr: 51, recoveryScore: 78, hrv: 112.44031 },
  { restingHr: null, recoveryScore: null, hrv: null }, // strap off
  { restingHr: 54, recoveryScore: 66, hrv: 90.5127 },
  { restingHr: 57, recoveryScore: 35, hrv: 83.242966 }, // Sat
  { restingHr: 59, recoveryScore: 29, hrv: 77.39185 }, // Sun — the story
  { restingHr: 47, recoveryScore: 52, hrv: 96.10188 }, // Mon — the rebound
  { restingHr: null, recoveryScore: null, hrv: null }, // today, 7am
];

/** The baseline behind the log fixtures — the DX's own figures. */
function logBaseline(): RecoveryBaselineView {
  return {
    windowDays: 30,
    restingHrAvg: 53.4,
    restingHrDays: 28,
    hrvAvg: 88.2,
    hrvStdDev: 20.1,
    hrvDays: 27,
    recoveryScoreAvg: 57.6,
    recoveryScoreDays: 28,
  };
}

/**
 * Assemble a log view around a day series, deriving the scalar today-figures
 * from its last day so the two cannot disagree.
 */
function logView(days: RecoveryDayPoint[]): RecoveryView {
  const last = days[days.length - 1];
  return {
    restingToday: last.restingHr,
    recoveryScore: last.recoveryScore,
    hrvToday: last.hrv,
    // Legacy and gap-omitting — the rail must never draw from this.
    spark: [61, 51, 54, 57, 59, 47],
    days,
    baseline: logBaseline(),
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
 * The DX's default fixture and the recovery log's headline state: 31 days
 * ending in a red weekend (35, 29), a rebound Monday (52), and no reading yet
 * today. This is what the tile looks like most mornings — nothing dramatic,
 * nothing broken — and it is what the redesign is judged on.
 */
export function recoveryLogView(): RecoveryView {
  const days = bandedDays([...LOG_HISTORY, ...LOG_TAIL.map((m) => m.recoveryScore)]);
  const from = days.length - LOG_TAIL.length;
  LOG_TAIL.forEach((morning, i) => {
    days[from + i] = { ...days[from + i], ...morning };
  });
  return logView(days);
}

/**
 * Three readings in the last eight days — travel, or the strap on the charger.
 * **This is the fixture the idiom was selected on.** The three most recent
 * calendar days are all empty, so the detail register is in its worst state,
 * while the rail still shows a week of readings behind them: the tile has to
 * look intentional here rather than broken.
 */
export function sparseView(): RecoveryView {
  return logView(bandedDays([...LOG_HISTORY, 58, null, 41, null, 63, null, null, null]));
}

/**
 * A morning with a resting HR and an HRV but NO recovery score — the three
 * readings are independently nullable and Whoop does produce this state. The
 * row must print `—` for the score and NO band word, rather than the words
 * "No reading" beside two perfectly good figures.
 */
export function partialMorningView(): RecoveryView {
  const days = [...recoveryLogView().days!];
  days[days.length - 1] = {
    ...days[days.length - 1],
    restingHr: 54,
    recoveryScore: null,
    hrv: 90.5127,
  };
  return logView(days);
}
```

- [ ] **Step 4: Run the tests and verify they pass**

Run: `npx vitest run "app/(app)/dashboard/_components/recovery/fixtures.test.ts"`
Expected: PASS, including every pre-existing block.

`RecoveryBaselineView`, `RecoveryDayPoint` and `RecoveryView` are already imported at the
top of `fixtures.ts` — do not duplicate the import.

- [ ] **Step 5: Run the whole recovery suite for regressions**

Run: `npx vitest run "app/(app)/dashboard/_components/recovery/"`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add "app/(app)/dashboard/_components/recovery/fixtures.ts" \
        "app/(app)/dashboard/_components/recovery/fixtures.test.ts"
git commit -m "test(dashboard): add score-bearing recovery fixtures with a red weekend"
```

---

## Task 3: Rewrite `MorningLedgerCard` as the two-register rail

**Files:**
- Rewrite: `app/(app)/dashboard/_components/recovery/morning-ledger.tsx`
- Rewrite: `app/(app)/dashboard/_components/recovery/morning-ledger.test.tsx`

This is the substance of the SOW. **Rewrite both files wholesale** — every current
assertion in the test file describes behaviour this task deletes (the `ms · bpm · score`
header order, the four-row window, the `▼` delta glyph and its `--warning` colour). Do not
adapt them.

Depends on Task 1 (`recoveryBand*`, `round`) and Task 2 (the fixtures).

### Two things to know before writing the test file

1. **`textContent` concatenates sibling elements with no separator.** The row's visual gaps
   come from `gap-2`, not from text, so `row.textContent` is `"Mon52Adequate47 bpm · 96 ms"`.
   The tests therefore read a row as its **fields, in DOM order** (`fields()` below), which
   is both accurate and a stronger assertion — it pins the reading order the SOW cares
   about (Recovery → resting HR → HRV) rather than a whitespace accident.
2. **jsdom's CSSOM parses longhands reliably and shorthands with `var()` unreliably.** The
   bar's fill is set as `backgroundColor`, not `background`, matching `trend-rail.tsx`'s
   idiom, so `el.style.backgroundColor` reads back as `"var(--danger)"`. Pixel values are
   compared with `Number.parseFloat` + `toBeCloseTo`, never as strings, because
   `railY(57.6)` is `23.040000000000003` and nothing guarantees how the CSSOM serialises it.

### Layout budget

The card body lands at ~154px — 40 rail + 8 + ~21 seam + 8 + ~77 detail — against the
shipped tile's ~180px and the DX's 260px ceiling, so this redesign **costs the dashboard
row no height**. `MiniCard` lays its children out with `gap-3` (12px), so the three
registers are wrapped in **one** `flex flex-col gap-2` child; that is what buys the 8px
seams the budget assumes, and it is why `MiniCard` gets one child rather than three.

- [ ] **Step 1: Write the failing test file**

Replace the entire contents of
`app/(app)/dashboard/_components/recovery/morning-ledger.test.tsx` with:

```tsx
/// <reference types="vitest/globals" />

import { render, screen, within } from "@testing-library/react";
import type { RecoveryView } from "@/lib/dashboard";
import {
  calibratingView,
  legacyView,
  partialBandView,
  partialMorningView,
  recoveryLogView,
  sparseView,
} from "./fixtures";
import { MorningLedgerCard, railY } from "./morning-ledger";

const HREF = "/recovery";
/** The rail's contract, mirrored — not an implementation detail. */
const RAIL_H = 40;
const RAIL_DAYS = 14;

function renderCard(section: RecoveryView) {
  return render(<MorningLedgerCard section={section} href={HREF} />);
}

/** Collapsed text of a whole subtree, for "does this string appear at all". */
function text(el: Element | null): string {
  return (el?.textContent ?? "").replace(/\s+/g, " ").trim();
}

/**
 * A detail row read as its FIELDS in DOM order. `textContent` would run the
 * siblings together (`Mon52Adequate…`) because the gaps are flex, not text —
 * and reading the fields is the stronger assertion anyway, since it pins the
 * row's reading order: Recovery → resting HR → HRV.
 */
function fields(row: Element): string[] {
  return Array.from(row.children).map((child) => text(child));
}

function bars(container: HTMLElement): HTMLElement[] {
  return Array.from(container.querySelectorAll('[data-testid="rail-bar"]'));
}
function ghosts(container: HTMLElement): HTMLElement[] {
  return Array.from(container.querySelectorAll('[data-testid="rail-ghost"]'));
}
function tick(container: HTMLElement): HTMLElement | null {
  return container.querySelector('[data-testid="rail-tick"]');
}
function rows(container: HTMLElement): HTMLElement[] {
  return Array.from(container.querySelectorAll('[data-testid="detail-row"]'));
}
/** The calibrating fixture with the two day-counts deliberately disagreeing. */
function calibratingMismatched(): RecoveryView {
  const view = calibratingView();
  // 9 scored mornings behind the rail; 22 HRV nights behind a metric this tile
  // does not draw. The two are emitted independently and can differ by weeks.
  return { ...view, baseline: { ...view.baseline!, recoveryScoreDays: 9, hrvDays: 22 } };
}

describe("MorningLedgerCard — the seam", () => {
  it("prints the baselines in score · bpm · ms order, rounded", () => {
    renderCard(recoveryLogView());
    // 57.6 · 53.4 bpm · 88.2 ms → rounded. A change from the shipped ms · bpm · score.
    expect(screen.getByText("58 · 53 bpm · 88 ms")).toBeInTheDocument();
  });

  it("dashes each baseline figure independently and keeps the line's shape", () => {
    const view = recoveryLogView();
    renderCard({ ...view, baseline: { ...view.baseline!, hrvAvg: null } });
    expect(screen.getByText("58 · 53 bpm · — ms")).toBeInTheDocument();
  });

  it("counts MORNINGS, not nights — the gate and the progress line are one metric", () => {
    renderCard(calibratingMismatched());
    expect(screen.getByText("calibrating")).toBeInTheDocument();
    expect(screen.getByText("9 of 14 mornings")).toBeInTheDocument();
  });

  it("never prints the HRV night count or calls a morning a night", () => {
    const { container } = renderCard(calibratingMismatched());
    expect(text(container)).not.toMatch(/night/i);
    expect(text(container)).not.toContain("22");
  });
});

describe("MorningLedgerCard — the rail", () => {
  it("renders one position per calendar day in the window", () => {
    const view = recoveryLogView();
    const { container } = renderCard(view);
    const window = view.days!.slice(-RAIL_DAYS);
    expect(bars(container).length + ghosts(container).length).toBe(RAIL_DAYS);
    expect(bars(container)).toHaveLength(window.filter((d) => d.recoveryScore !== null).length);
    expect(ghosts(container)).toHaveLength(window.filter((d) => d.recoveryScore === null).length);
  });

  it("paints each bar in its own morning's band colour", () => {
    const { container } = renderCard(recoveryLogView());
    const fills = bars(container).map((b) => b.style.backgroundColor);
    // Sunday's 29 is a --danger bar in a rail that is otherwise warm.
    expect(fills).toContain("var(--danger)");
    expect(fills).toContain("var(--warning)");
    expect(fills).toContain("var(--success)");
    expect(fills).not.toContain("var(--accent)");
  });

  it("draws an absent morning as a full-height ghost, differing in KIND not height", () => {
    const { container } = renderCard(sparseView());
    const ghost = ghosts(container)[0];
    const bar = bars(container)[0];
    // A ghost runs the whole rail and carries no inline height cap...
    expect(ghost.className).toContain("h-full");
    expect(ghost.style.height).toBe("");
    // ...while a scored bar is capped at its own score's pixel height. A 2px
    // stub at the foot of the rail would fail both halves of this.
    expect(Number.parseFloat(bar.style.height)).toBeGreaterThan(2);
    expect(Number.parseFloat(bar.style.height)).toBeLessThan(RAIL_H);
  });

  it("maps every bar through railY, so a score of 100 is the whole rail", () => {
    expect(railY(0)).toBe(0);
    expect(railY(100)).toBe(RAIL_H);
    const view = recoveryLogView();
    const { container } = renderCard(view);
    const scored = view.days!.slice(-RAIL_DAYS).filter((d) => d.recoveryScore !== null);
    bars(container).forEach((bar, i) => {
      expect(Number.parseFloat(bar.style.height)).toBeCloseTo(
        Math.max(2, railY(scored[i].recoveryScore!)),
        5,
      );
    });
  });

  it("pins the tick to the SAME map as the bars", () => {
    const view = recoveryLogView();
    const { container } = renderCard(view);
    // A bar whose score IS the baseline average terminates exactly at the tick.
    // Rescaling the rail to the observed data range would desynchronise the two
    // silently, because the tick would still be drawn on the 0-100 scale.
    expect(Number.parseFloat(tick(container)!.style.bottom)).toBeCloseTo(
      railY(view.baseline!.recoveryScoreAvg!),
      5,
    );
  });

  it("draws no tick while the score baseline is still calibrating", () => {
    const { container } = renderCard(calibratingView());
    expect(tick(container)).toBeNull();
    // The scores exist even though the normal does not, so the bars still draw.
    expect(bars(container).length).toBeGreaterThan(0);
  });

  it("carries a text alternative rather than vanishing from the accessibility tree", () => {
    renderCard(recoveryLogView());
    const rail = screen.getByRole("img");
    expect(rail).toHaveAccessibleName(/14 days of recovery score/);
    expect(rail).toHaveAccessibleName(/12 with readings/);
    expect(rail).toHaveAccessibleName(/30-day average of 58/);
  });

  it("says so when there is no average to be read against", () => {
    renderCard(calibratingView());
    expect(screen.getByRole("img")).toHaveAccessibleName(/no 30-day average yet/);
  });

  it("sizes every mark by flex, so both breakpoints are the same DOM", () => {
    const { container } = renderCard(recoveryLogView());
    for (const mark of [...bars(container), ...ghosts(container)]) {
      expect(mark.className).toContain("flex-1");
      expect(mark.className).not.toMatch(/\bw-\d/);
    }
  });
});

describe("MorningLedgerCard — the detail rows", () => {
  it("shows exactly three mornings, newest first, with Today leading", () => {
    const { container } = renderCard(recoveryLogView());
    const detail = rows(container);
    expect(detail).toHaveLength(3);
    expect(detail.map((r) => fields(r)[0])).toEqual(["Today", "Mon", "Sun"]);
  });

  it("reads Recovery -> resting HR -> HRV, hero to footnote", () => {
    const { container } = renderCard(recoveryLogView());
    // Monday's rebound: 52, 47 bpm, 96.10188 ms.
    expect(fields(rows(container)[1])).toEqual(["Mon", "52", "Adequate", "47 bpm · 96 ms"]);
  });

  it("never prints a raw Whoop float — this is the wrap, and it is a bug", () => {
    const { container } = renderCard(recoveryLogView());
    expect(text(container)).not.toContain("96.10188");
    expect(text(container)).not.toContain("77.39185");
    expect(text(container)).toContain("96 ms");
  });

  it("rounds a non-integral recovery score for the same reason", () => {
    const view = recoveryLogView();
    const days = [...view.days!];
    days[days.length - 2] = { ...days[days.length - 2], recoveryScore: 52.4 };
    const { container } = renderCard({ ...view, days });
    expect(fields(rows(container)[1]).slice(0, 3)).toEqual(["Mon", "52", "Adequate"]);
    expect(text(container)).not.toContain("52.4");
  });

  it("carries no HRV delta glyph and no HRV status colour — that idiom is gone", () => {
    const { container } = renderCard(recoveryLogView());
    expect(text(container)).not.toMatch(/[▼▲▬]/);
    // `--accent` is the token only `hrvStatusColor` produces. This tile paints
    // no HRV status at all, so it must not appear anywhere in the markup.
    expect(container.innerHTML).not.toContain("var(--accent)");
  });

  it("renders a fully-null morning as one 'no reading' row", () => {
    const { container } = renderCard(recoveryLogView());
    expect(fields(rows(container)[0])).toEqual(["Today", "no reading"]);
    expect(screen.getAllByText("no reading")).toHaveLength(1);
  });

  it("keeps the three most recent calendar days even when all three are empty", () => {
    const { container } = renderCard(sparseView());
    expect(rows(container)).toHaveLength(3);
    expect(screen.getAllByText("no reading")).toHaveLength(3);
  });

  it("prints a dash and NO band word for a morning with readings but no score", () => {
    const { container } = renderCard(partialMorningView());
    const today = rows(container)[0];
    // `Today  No reading  54 bpm · 90 ms` is false: a missing score has no
    // band, and the honest rendering of that is silence, not a label.
    expect(fields(today)).toEqual(["Today", "—", "54 bpm · 91 ms"]);
    expect(text(today)).not.toMatch(/no reading/i);
  });

  it("sizes the figure group to its content, so it cannot wrap", () => {
    const { container } = renderCard(recoveryLogView());
    const groups = Array.from(container.querySelectorAll('[data-testid="detail-figures"]'));
    expect(groups.length).toBeGreaterThan(0);
    for (const group of groups) {
      expect(group.className).toContain("ml-auto");
      expect(group.className).not.toMatch(/\bw-\d/);
    }
  });
});

describe("MorningLedgerCard — the states it has to survive", () => {
  it("full week: every position is a bar, and the tick still crosses them", () => {
    const view = recoveryLogView();
    // Today has landed and nothing in the window is missing — the clean case,
    // and the fixture the tick's contrast is judged on.
    const days = view.days!.map((d) =>
      d.recoveryScore === null ? { ...d, recoveryScore: 71, restingHr: 49, hrv: 94 } : d,
    );
    const { container } = renderCard({ ...view, days });
    expect(bars(container)).toHaveLength(RAIL_DAYS);
    expect(ghosts(container)).toHaveLength(0);
    expect(text(container)).not.toMatch(/no reading/i);
    expect(tick(container)).not.toBeNull();
  });

  it("red morning: today at 22 is one red bar and one red word, never a red row", () => {
    const view = recoveryLogView();
    const days = [...view.days!];
    days[days.length - 1] = {
      ...days[days.length - 1],
      restingHr: 63,
      recoveryScore: 22,
      hrv: 58,
    };
    const { container } = renderCard({ ...view, days });
    const today = rows(container)[0];
    expect(fields(today)).toEqual(["Today", "22", "Low", "63 bpm · 58 ms"]);
    expect(within(today).getByText("Low").style.color).toBe("var(--danger)");
    // Text and mark weight only: the row itself carries no fill of any kind.
    expect(today.getAttribute("style")).toBeNull();
  });

  it("is indifferent to the per-day band fields the sibling tiles care about", () => {
    const view = partialBandView();
    const stripped = {
      ...view,
      days: view.days!.map((d) => ({
        ...d,
        baselineAvg: null,
        balancedLow: null,
        balancedHigh: null,
        zScore: null,
        status: "unknown" as const,
      })),
    };
    const banded = render(<MorningLedgerCard section={view} href={HREF} />);
    const unbanded = render(<MorningLedgerCard section={stripped} href={HREF} />);
    // An uncalibrated day is an ORDINARY day here. This variant reads no per-day
    // band field, so stripping every one of them changes nothing on the card.
    expect(unbanded.container.innerHTML).toBe(banded.container.innerHTML);
  });

  it("reads as calibrating on a legacy payload with no derived blocks", () => {
    renderCard(legacyView());
    expect(screen.getByText("Log is calibrating.")).toBeInTheDocument();
  });
});
```

- [ ] **Step 2: Run the tests and verify they fail**

Run: `npx vitest run "app/(app)/dashboard/_components/recovery/morning-ledger.test.tsx"`
Expected: FAIL — `railY` is not exported, and every rail/seam assertion fails against the
shipped four-row ledger.

- [ ] **Step 3: Rewrite the component**

Replace the entire contents of
`app/(app)/dashboard/_components/recovery/morning-ledger.tsx` with:

```tsx
/**
 * MorningLedgerCard — the `recovery_log` tile ("Recovery Log").
 *
 * Two unequal registers over a seam — the `band-rail-and-recent` idiom from
 * `dx/recovery-log-tile`:
 *
 *   1. THE RAIL — fourteen days as vertical bars, each one's height
 *      proportional to that morning's recovery score and its fill that
 *      morning's Whoop band colour, over a dashed neutral tick at the 30-day
 *      score average. A fortnight read against its own normal, so a bad weekend
 *      is a notch you see before you read a figure.
 *   2. THE SEAM — the 30-day baseline as `score · bpm · ms` on a hairline,
 *      doubling as the register divider.
 *   3. THE DETAIL ROWS — the three most recent mornings in full
 *      (weekday · score · band word · bpm · ms). This is where the tile's
 *      distinct job happens: the cross-metric reading — *the score was low and
 *      the resting HR was up* — is a sentence only this tile can make.
 *
 * Fourteen on the rail and three in detail is the idiom's argument, not a
 * budget. Seven identical rows give you a week you must read row by row; a rail
 * gives you a fortnight you read at a glance and detail only where detail is
 * used. It is also what survives a sparse fortnight, where a row-per-day tile
 * degrades into a column of *no reading*.
 *
 * An absent morning is a FULL-HEIGHT ghost column, never a short bar: a gap and
 * a catastrophic score differ in kind, not in degree, and a stub at the foot of
 * the rail sits a pixel away from a real score of 5.
 *
 * The tile paints NO HRV status. Its colour budget is spent, once, on the
 * recovery band — the rail's fill and one small word per row — which is the
 * resolution to the tension `readiness-verdict.tsx` names in its own docstring:
 * two traffic lights on one card is confusion.
 *
 * Nothing here recomputes a server figure. Naming which third of Whoop's fixed,
 * published 0–100 scale a score falls in is display formatting, and the only
 * other arithmetic is mapping an already-computed score onto a pixel height,
 * which is what a bar chart is.
 */

import type { RecoveryDayPoint, RecoveryView } from "@/lib/dashboard";
import { MiniCard } from "../mini-card";
import { recoveryBand, recoveryBandColor, recoveryBandWord, round, weekday } from "./shared";

const TITLE = "Recovery Log";
/** Days on the rail, and mornings in the detail register. */
const RAIL_DAYS = 14;
const DETAIL_ROWS = 3;
/** The rail's height in px. `railY` maps a 0–100 score onto it. */
const RAIL_H = 40;
/**
 * How many scored mornings the server needs before it will emit an average.
 * The real value is `config.Recovery.MinBaselineDays` on the API; this is a
 * client-side copy, as it also is on the HRV balance tile.
 */
const MIN_BASELINE_DAYS = 14;

/**
 * Score (0–100) → bar height in px. The baseline TICK uses this same map, and
 * that is load-bearing: the rail's whole claim is that a fortnight reads
 * against its own normal, which is true only while a bar at the baseline's
 * value terminates exactly at the tick. The obvious future "improvement" —
 * rescaling the rail to the observed data range so a flat fortnight uses the
 * full height — breaks that silently, because the tick would still be drawn on
 * the 0–100 scale. Hence one exported, tested function rather than two inline
 * expressions.
 *
 * The 0–100 scale is also right on its own terms: the recovery score is a
 * percentage with meaningful absolute values, so a 29 should look short in
 * absolute terms and not merely short relative to a bad fortnight.
 */
export function railY(score: number): number {
  return (score / 100) * RAIL_H;
}

/**
 * The rail's text alternative, e.g. "14 days of recovery score, 12 with
 * readings, against a 30-day average of 58." Every bar and the tick are
 * `aria-hidden`, so without this the first register would be absent from the
 * accessibility tree with nothing standing in for it — and fourteen unlabelled
 * divs in the tree would be worse than one labelled group.
 */
function railLabel(rail: RecoveryDayPoint[], avg: number | null): string {
  const readings = rail.filter((d) => d.recoveryScore !== null).length;
  const window = `${rail.length} days of recovery score, ${readings} with readings`;
  return avg === null
    ? `${window}, with no 30-day average yet.`
    : `${window}, against a 30-day average of ${Math.round(avg)}.`;
}

export function MorningLedgerCard({ section, href }: { section: RecoveryView; href: string }) {
  const { days, baseline } = section;

  if (!days || !baseline) {
    return (
      <MiniCard title={TITLE} href={href}>
        <p className="text-sm text-[var(--muted)]">Log is calibrating.</p>
      </MiniCard>
    );
  }

  // Gate on the metric this card actually DRAWS. The score average is the
  // tick's own input, and without it the rail has no normal to be read against.
  // (The shipped tile gated on `hrvAvg`, which was right for a tile whose only
  // coloured element was an HRV delta and is wrong for this one.)
  const calibrating = baseline.recoveryScoreAvg === null;
  // `days` is date-aligned with nulls preserved, so the slice is a true calendar
  // fortnight and the bar pitch is a day. Never `spark` — it omits missing days
  // and destroys the alignment the rail's whole shape depends on.
  const rail = days.slice(-RAIL_DAYS);
  const recent = [...days.slice(-DETAIL_ROWS)].reverse();

  return (
    <MiniCard title={TITLE} href={href}>
      {/* One child, not three: MiniCard lays its children out on gap-3 and the
          two seams here are 8px. The whole body is ~154px, under the shipped
          tile's ~180 — this redesign costs the dashboard row no height. */}
      <div className="flex flex-col gap-2">
        <div
          role="img"
          aria-label={railLabel(rail, baseline.recoveryScoreAvg)}
          className="relative flex items-end gap-[2px]"
          style={{ height: RAIL_H }}
        >
          {baseline.recoveryScoreAvg !== null && (
            /* Drawn ABOVE the bars, by DOM order, and in `--muted` at half
               strength rather than the `--border-strong` hairline: composited
               over a full-strength `--success` bar that hairline reaches only
               1.10:1 contrast — invisible, and invisible exactly when the
               fortnight is good, which is when the register's "against its own
               normal" argument matters most. `--muted` at 0.5 reads 1.28:1 over
               the bar and 2.09:1 over `--surface`. A structural datum is not
               chrome. */
            <div
              aria-hidden="true"
              data-testid="rail-tick"
              className="pointer-events-none absolute inset-x-0 border-t border-dashed border-[var(--muted)] opacity-50"
              style={{ bottom: railY(baseline.recoveryScoreAvg) }}
            />
          )}
          {rail.map((d) => (
            <RailBar key={d.date} day={d} />
          ))}
        </div>

        <div className="flex items-baseline justify-between border-t border-[var(--border)] pt-1.5 text-[10px] uppercase tracking-[0.08em] text-[var(--faint)]">
          <span>{calibrating ? "calibrating" : "30-day base"}</span>
          {/* Each figure guards on its own: `restingHrAvg` or `hrvAvg` may be
              null while `recoveryScoreAvg` is not, and the seam keeps its shape
              rather than collapsing when one average is missing. */}
          <span className="tabular-nums normal-case tracking-[-0.01em]">
            {calibrating
              ? `${baseline.recoveryScoreDays} of ${MIN_BASELINE_DAYS} mornings`
              : `${round(baseline.recoveryScoreAvg)} · ${round(baseline.restingHrAvg)} bpm · ${round(baseline.hrvAvg)} ms`}
          </span>
        </div>

        <div className="-mt-0.5 flex flex-col divide-y divide-[var(--border)]">
          {recent.map((d, i) => (
            <DetailRow key={d.date} day={d} isToday={i === 0} />
          ))}
        </div>
      </div>
    </MiniCard>
  );
}

/** One day's mark on the rail. */
function RailBar({ day }: { day: RecoveryDayPoint }) {
  const score = day.recoveryScore;

  // An absent morning is a ghost slot running the full height of the rail — an
  // empty position in a rack, not a very short bar. Differing in KIND rather
  // than in height is what keeps a strap-off day from reading as a score of 2,
  // and it is what makes a sparse fortnight look intentional rather than broken.
  if (score === null) {
    return (
      <div
        aria-hidden="true"
        data-testid="rail-ghost"
        className="h-full flex-1 rounded-[1px] bg-[var(--surface-2)] opacity-60"
      />
    );
  }

  return (
    <div
      aria-hidden="true"
      data-testid="rail-bar"
      className="flex-1 rounded-[1px]"
      style={{
        // The 2px floor bites only below a score of 5, where it can never
        // approach a plausible baseline tick; it is there so a genuine
        // near-zero morning still renders a mark.
        height: Math.max(2, railY(score)),
        backgroundColor: recoveryBandColor(recoveryBand(score)),
      }}
    />
  );
}

/** One morning in full: recovery, then resting HR, then HRV. */
function DetailRow({ day, isToday }: { day: RecoveryDayPoint; isToday: boolean }) {
  // Positional, by the payload's contract that `days` ends on the local today.
  const label = isToday ? "Today" : weekday(day.date);
  const missing = day.recoveryScore === null && day.restingHr === null && day.hrv === null;

  return (
    <div data-testid="detail-row" className="flex items-baseline gap-2 py-1">
      {/* A width on the LABEL, which holds "Today" and cannot wrap, so the three
          scores line up in a column. The fixed widths this redesign deletes were
          on the VALUE cells (w-16 / w-14 / w-6), which is what made a five-decimal
          millisecond reading break onto a second line. */}
      <span className="w-10 shrink-0 text-[10px] text-[var(--faint)]">{label}</span>
      {missing ? (
        /* A morning the webhook never delivered. At 7am this is today's own row,
           and "Today · no reading" is the true and useful answer to "has my
           morning landed?" — so it is never suppressed. */
        <span className="text-[11px] italic text-[var(--faint)]">no reading</span>
      ) : (
        <>
          <span className="text-[13px] tabular-nums tracking-[-0.03em] text-[var(--foreground)]">
            {round(day.recoveryScore)}
          </span>
          {/* A missing score has no band, and the honest rendering of that is
              silence — not the words "No reading" beside two live figures. */}
          {day.recoveryScore !== null && (
            <span
              className="text-[10px] uppercase tracking-[0.08em]"
              style={{ color: recoveryBandColor(recoveryBand(day.recoveryScore)) }}
            >
              {recoveryBandWord(recoveryBand(day.recoveryScore))}
            </span>
          )}
          {/* No fixed width: the group sizes to its content, so a three-digit
              figure plus its unit cannot wrap. */}
          <span
            data-testid="detail-figures"
            className="ml-auto whitespace-nowrap font-mono text-[11px] tabular-nums text-[var(--foreground)]"
          >
            {round(day.restingHr)}
            <span className="text-[var(--faint)]"> bpm</span>
            {" · "}
            {round(day.hrv)}
            <span className="text-[var(--faint)]"> ms</span>
          </span>
        </>
      )}
    </div>
  );
}
```

- [ ] **Step 4: Run the tests and verify they pass**

Run: `npx vitest run "app/(app)/dashboard/_components/recovery/morning-ledger.test.tsx"`
Expected: PASS.

If a `fields()` assertion fails, the mismatch is telling you the row's DOM order or its
separators are wrong — fix the **component**, not the expectation. Do not weaken an
assertion to make it pass, and do not add a test-only element to satisfy one.

- [ ] **Step 5: Run the whole dashboard suite for regressions**

Run: `npx vitest run "app/(app)/dashboard/"`
Expected: PASS, with `page.test.tsx`, `tile-renderer.test.tsx` and the other four recovery
tiles' suites **unmodified**. If one of them needs editing, either the catalog or
`shared.ts` has been disturbed and that is out of scope — report it rather than editing.

- [ ] **Step 6: Commit**

```bash
git add "app/(app)/dashboard/_components/recovery/morning-ledger.tsx" \
        "app/(app)/dashboard/_components/recovery/morning-ledger.test.tsx"
git commit -m "feat(dashboard): rebuild the recovery log as a 14-day band rail"
```

---

## Task 4: The tray description

**Files:**
- Modify: `lib/dashboard-tiles.ts:137`

The catalog is unchanged but for the tray copy, which currently claims "your last few
mornings" and is now wrong in both halves — the tile shows a fortnight, and it shows three
mornings in full, not "a few".

- [ ] **Step 1: Make the edit**

In `lib/dashboard-tiles.ts`, in the `recovery_log` entry, replace:

```ts
    description: "Your last few mornings as dated readings.",
```

with:

```ts
    description: "Two weeks of recovery score — your last three mornings in full.",
```

Change nothing else in the entry — same `id`, same `title`, same `href`.

> **Revised during review** from the SOW's suggested *"A fortnight of recovery, and your
> last three mornings in full."* — which the SOW offers as "something like", not as fixed
> copy. Three reasons, all checkable against the catalog: the rail plots only
> `recoveryScore`, and bare "recovery" reads first as *rest/deload* in a strength app;
> "fortnight" was the sole Britishism in any user-facing string, and would have had the
> tile call its own window "a fortnight" to sighted users while `railLabel` says "14 days"
> to screen readers; and the em dash is the house shape for a
> primary-register-then-secondary-register description (`hrv_balance`, `weather`), which is
> exactly what this tile is.

- [ ] **Step 2: Verify nothing asserted the old string**

Run: `grep -rn "last few mornings" --include=*.ts --include=*.tsx . | grep -v node_modules`
Expected: no output.

Run: `npx vitest run lib/ "app/(app)/dashboard/"`
Expected: PASS.

- [ ] **Step 3: Commit**

```bash
git add lib/dashboard-tiles.ts
git commit -m "docs(dashboard): describe the recovery log as a fortnight plus three mornings"
```

---

## Task 5: The full local CI gate

**Files:** none — this task changes nothing unless a check fails.

CI (`.github/workflows/ci.yml`) runs `npm run lint` → `npm run format:check` →
`npm run typecheck` → `npm run test` → `npm run build`. A PR that fails CI costs a manual
round-trip, so the whole gate runs locally first. The Husky `pre-commit` hook (lint-staged
+ `tsc --noEmit`) and `commit-msg` hook (commitlint) run on every commit above; **never**
bypass either with `--no-verify` or `HUSKY=0`.

- [ ] **Step 1: Lint**

Run: `npm run lint`
Expected: no errors. **Never** silence a finding with an eslint-disable comment; fix the
code.

- [ ] **Step 2: Format**

Run: `npm run format:check`
If it fails, run `npm run format` and commit the result as `style: prettier`. Prettier owns
wrapping — do not hand-fight it.

- [ ] **Step 3: Types**

Run: `npm run typecheck`
Expected: no errors.

- [ ] **Step 4: Tests**

Run: `npm run test`
Expected: the whole suite green.

- [ ] **Step 5: Build**

Run: `npm run build`
Expected: success.

- [ ] **Step 6: Confirm the diff is exactly seven files**

Run: `git diff --stat main...HEAD`
Expected: `morning-ledger.tsx`, `morning-ledger.test.tsx`, `shared.ts`, `shared.test.ts`,
`fixtures.ts`, `fixtures.test.ts`, `lib/dashboard-tiles.ts` — and nothing else. In
particular, no change under `app/design-explore/`, no change to `tile-renderer.tsx`, and no
change to the other four recovery tiles.

---

## Verification against the SOW's state list

Each of these is covered by a named test in Task 3; this table is the cross-check, not
extra work.

| SOW state | Fixture | Assertion |
| --- | --- | --- |
| Default (red weekend, rebound Monday, no reading today) | `recoveryLogView()` | rail counts, band colours, row order, `Today · no reading` |
| `full-week` | `recoveryLogView()` with its two gaps filled | 14 bars, 0 ghosts, no *no reading*, tick present over a fully coloured rail |
| `red-morning` | `recoveryLogView()` with today at 22 | a `--danger` bar and a `Low` word; the row itself carries no fill |
| `sparse` | `sparseView()` | ghost-vs-bar differs in kind; three *no reading* rows |
| `calibrating` | `calibratingView()` | no tick, bars still draw, `9 of 14 mornings`, no `NaN` |
| `partial-band` | `partialBandView()` | stripping every per-day band field changes nothing |
| Partial morning | `partialMorningView()` | `Today · — · 54 bpm · 91 ms`, no band word |
| No reading yet today | `recoveryLogView()` | `Today · no reading` leads, ghost last on the rail |
| Legacy payload | `legacyView()` | *"Log is calibrating."* verbatim |
| Both breakpoints | `recoveryLogView()` | `flex-1` marks, no `w-*` on any value cell |
</content>
