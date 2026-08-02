---
type: sow
status: ready_for_implementation
repos:
  - prog-strength-api
  - prog-strength-web
  - prog-strength-docs
---

# Recovery Tile Family — Five Dashboard Tiles from `dx/recovery-tile`

**Status**: Ready for implementation · **Last updated**: 2026-08-01

> Frontend SOW with a small API companion. It implements the chosen DX variants and
> therefore inherits that DX's `scope` — here **`in-system`**. The visual foundation
> (design-system **v0.4**, oura-calm) is already decided, so this SOW **conforms** to
> it; it does **not** re-tone the system or touch shared tokens. `prog-strength-web`
> does the rendering work, `prog-strength-api` does the catalog + section-gating work,
> and `prog-strength-docs` only flips the DX to `selected` and marks this SOW shipped.

## Introduction

The **Recovery mini-card** on the dashboard (`RecoveryCard` in
`app/(app)/dashboard/_components/whoop-card.tsx`) wears the grid's generic
furniture: a `BigNum` resting-HR headline, a min/max-normalized `Spark` of recent
resting HR, and a `MetaRow` carrying the recovery score as a footnote. The Design
Exploration [`dx/recovery-tile.md`](../dx/recovery-tile.md)
(PR Prog-Strength/prog-strength-web#143) named why that furniture doesn't just
underserve this metric but **actively misinforms**:

- **A normalized sparkline of resting HR answers a question nobody asks.** Resting
  HR is a narrow-range metric where the *size* of the deviation is the entire
  signal, and min/max normalization is precisely what destroys it — a 1 bpm wobble
  and a 9 bpm spike draw the same picture.
- **The headline number has nothing to be measured against.** "51 bpm" is a number;
  "51 bpm against your 30-day average of 52.4" is a fact about the athlete.
- **HRV — the metric a recovery surface exists to talk about — is not on the tile
  at all.** [`sows/dashboard-recovery-metrics-payload.md`](./dashboard-recovery-metrics-payload.md)
  closed that data gap (`prog-strength-web` #142): the tile now receives a 31-day
  date-aligned history, per-user baselines, a balanced band, and an HRV status. The
  tile still doesn't show any of it.

This DX differed structurally from every prior one, and that difference is the
point. Because [`sows/customizable-dashboard-tiles.md`](./customizable-dashboard-tiles.md)
made the grid user-composed, the selection gate was **not** "which one variant
replaces the card" but **"which of these ship as separate, independently addable
tiles."** The five variants were therefore drafted as **complementary tiles with
non-overlapping jobs** — each heroes a *different* figure (band / sentence /
three-equal-numbers / direction / log), each carries its own title and tray
description, and each lets exactly **one** of the two three-state color axes carry
color.

**The owner selected all five.** That assignment holds precisely because the
no-two-variants-hero-the-same-figure constraint was binding on the spread: five
recovery tiles on one grid still never print the same number twice. The catalog
goes **11 → 15 tiles**.

This SOW reimplements all five **production-quality** against the real tile and
real data. Because `scope: in-system`, there is **no palette, accent, type, or
design-system change**. There is **no new backend data and no new endpoint** —
every value all five compositions need already ships in `RecoveryView`; the API
work here is catalog registration and section gating only.

## Proposed Solution

### The shipped `recovery` tile is rewritten, not preserved

`readiness-verdict`'s proposed title is `Recovery` — the same title the existing
tile carries. Rather than mint a sixth id and leave the misinforming card addable
forever, **the `recovery` id keeps its id and its catalog slot and its body becomes
`readiness-verdict`.** Its tray description is rewritten to match. The other four
ship as new ids:

| Tile id | Variant | Title | Heroes |
| --- | --- | --- | --- |
| `recovery` *(rewritten)* | `readiness-verdict` | `Recovery` | the sentence |
| `hrv_balance` *(new)* | `balance-band` | `HRV Balance` | the band |
| `morning_vitals` *(new)* | `three-dial-vitals` | `Morning Vitals` | three equal numbers |
| `recovery_trend` *(new)* | `trend-rail` | `Recovery Trend` | the direction |
| `recovery_log` *(new)* | `morning-ledger` | `Recovery Log` | the log |

**This is deliberately migration-free.** Every stored layout containing `"recovery"`
keeps working and silently renders the new verdict card; no layout row is rewritten,
no read-path filter silently drops a tile a user asked for, and every existing
Whoop-connected user is upgraded off the resting-HR sparkline without action.

### All five read one section

Every variant reads the **same** `recovery` section — there is no per-tile payload.
The four new ids therefore get **no section of their own**; the API keeps emitting
exactly one `recovery` key and the web adapter keeps reading `summary.recovery`
unchanged. The only API change of consequence is **gating**: `buildRecoverySection`
is currently `if enabled[TileRecovery]`, so a user who adds only *HRV Balance* would
get a nil section and a blank tile. It must be re-gated on **any** recovery-family
tile being enabled.

This makes the response's section-key set intentionally *not* a subset of `layout`
— a layout of `[hrv_balance]` yields a `recovery` section and no `hrv_balance` key.
That is correct and already compatible with the web adapter (`recovery:
summary.recovery ? { present: true, ... } : { present: false }`), but it is a new
property of the contract and is covered by a handler test below.

### Everything the variants need already ships

From `prog-strength-web/lib/dashboard.ts`, after the payload SOW:

```ts
type RecoveryView = {
  restingToday: number | null;    // BigNum-free now; a contributor row / cell
  recoveryScore: number | null;
  hrvToday?: number | null;
  spark: number[];                // LEGACY 7d resting-HR spark, gap-OMITTING
  days?: RecoveryDayPoint[];      // 31 date-aligned days, oldest→newest, nulls preserved
  baseline?: RecoveryBaselineView;// windowDays, restingHrAvg, hrvAvg, hrvStdDev, counts…
  hrv?: RecoveryHrvView;          // status, balancedLow/High, zScore, trend, shortAvg
};
```

Four properties of that shape are **binding on every variant**, carried over from
the DX verbatim:

1. **`days` is the honest series; `spark` is not.** `spark` omits days without a
   reading, shortening the array and destroying date alignment. **All five charts
   and the ledger read `days`.** No new tile reads `spark`; it stays on the type only
   because it is still serialized.
2. **Never recompute a server figure.** `hrvAvg`, `restingHrAvg`,
   `recoveryScoreAvg`, `balancedLow`, `balancedHigh`, and `zScore` are computed
   server-side over a window that *excludes today*. Display as received. The only
   client-side arithmetic permitted is a **signed delta of a today-value against a
   server baseline** (`hrvToday − hrvAvg`) and `trend-rail`'s `shortAvg − hrvAvg`
   percentage — both differences of two server figures, never a re-averaged series.
3. **The band is per-user.** `balancedLow`/`balancedHigh` are this athlete's own
   baseline ± this athlete's own SD. No variant may hardcode a "normal HRV range".
4. **`days` / `baseline` / `hrv` are typed optional.** Guard once at the top of each
   variant and render its calibrating state if absent — **never `!`-assert.**

### Color logic — the trap, restated as a per-tile contract

Recovery carries **two three-state axes that must not fight**: the Whoop recovery
score band (`lib/recovery.ts`: success ≥ 67 / warning 34–66 / danger ≤ 33) and the
HRV balance status, which says whether today is *typical for this athlete*, not
whether it is *good*. Painting both at strength inside a 180px card produces two
competing traffic lights.

Per-tile assignment (binding, from the DX):

| Tile | What carries color | What stays neutral |
| --- | --- | --- |
| `Recovery` | status on **exactly one word** (the verdict) | figures, delta chips |
| `HRV Balance` | the **HRV axis entirely** — band fill, today's point | score absent; series in muted ink |
| `Morning Vitals` | **one status dot** beside HRV | all three figures, all captions |
| `Recovery Trend` | the **delta figure** + per-mark in/out-of-band shading | everything else |
| `Recovery Log` | the **per-row HRV delta signs** only | dates, figures, header |

Status→token mapping is shared and single-sourced:

- **`suppressed` → `--warning`** (`#d6b87f`), **never `--danger`**. A low-HRV
  morning is information, not an emergency.
- **`balanced` → `--success`** (`#86b39f`) or calm neutral. Balanced is the
  *ordinary* state — roughly two-thirds of days — and must feel unremarkable.
- **`elevated` → `--accent`** (periwinkle). HRV well above baseline is *unusual*,
  not "extra good". **It must never render as a bigger green** — this is the single
  most common way an implementation gets the semantics wrong.
- **`unknown` → `--muted`**, and never an em-dash alone.
- Recovery has **no `--discipline-*` hue** and none is invented.

## Goals and Non-Goals

### Goals

**`prog-strength-api`**

- **Add four `TileID` constants** — `TileHRVBalance` (`hrv_balance`),
  `TileMorningVitals` (`morning_vitals`), `TileRecoveryTrend` (`recovery_trend`),
  `TileRecoveryLog` (`recovery_log`) — and place them in `Catalog` immediately after
  `TileRecovery`, in the order above. `Catalog` order fixes the web tray order and
  must stay byte-identical to the TS mirror.
- **Re-gate `buildRecoverySection`** on *any* recovery-family tile being enabled,
  building and emitting the section **exactly once** under the `"recovery"` key
  regardless of how many family tiles are on the layout. No second read, no
  per-tile section.
- **Leave `defaultLayout` unchanged** — a Whoop-connected user still gets exactly
  one recovery tile (`TileRecovery`) by default; the other four are opt-in from the
  tray. `hasConnectedWhoop` is untouched.
- **Update `tiles_test.go`** (both the `all` and `want` lists) and add a handler
  test asserting: a layout of `[hrv_balance]` alone yields a populated `recovery`
  section and **no** `hrv_balance` key; a layout with three family tiles yields one
  `recovery` section; a layout with **no** family tile yields **no** `recovery` key.
- **Go CI green** (build, vet, test).

**`prog-strength-web`**

- **Add the four `TileId`s and their catalog entries** in `lib/dashboard-tiles.ts`,
  in the same position and order as the Go mirror, each `href: "/recovery"`, each
  with its own one-line tray description. **Rewrite the `recovery` entry's
  description** — `"Whoop recovery and resting HR."` no longer describes the card.
  Proposed copy:
  - `recovery` — *"Today's readiness as a sentence, with baseline deltas."*
  - `hrv_balance` — *"Today's HRV against your own balanced range."*
  - `morning_vitals` — *"Score, resting HR, and HRV vs your 30-day averages."*
  - `recovery_trend` — *"Which way your HRV is heading this week."*
  - `recovery_log` — *"Your last few mornings as dated readings."*
- **Build five production tile components** under
  `app/(app)/dashboard/_components/recovery/`, each owning the `MiniCard` body
  (title, `p-4`, the whole-card link into `/recovery` stay chrome it keeps
  functional) and each exporting a **titled empty variant** for the
  enabled-but-not-connected case.
- **Single-source the shared formatting and status mapping** in one small tested
  module (`recovery/shared.ts`): `hrvStatusColor`, `statusWord`, `trendLabel`,
  `signed` / `signedUnit`, `weekday`. Five hand-rolled copies of the status→token
  switch is exactly how `elevated` ends up green on one tile.
- **Rewire `TileCard`** (`tile-renderer.tsx`) with five cases, each threading
  `data.recovery` and each falling to its own empty variant when
  `!data.recovery.present`. The `never` default keeps its compile-time
  exhaustiveness guarantee.
- **Handle every state the DX enumerated, on all five**, with no `NaN`, no band
  drawn at zero, no empty chart frame, and **no degrading to em-dashes**:
  - **Calibrated + suppressed** — the headline case; true and slightly concerning
    without alarming.
  - **Calibrated + balanced** — the boring good day, which is most days. A tile that
    only looks good in the dramatic state is a failed tile.
  - **Calibrating** (`hrvDays < 14`) — averages, both band bounds and `zScore` are
    **null**; status and trend are `unknown`. Every new Whoop user lives here for two
    weeks. Show honest progress from the counts ("9 of 14 nights"), never an empty
    frame.
  - **No reading yet today** (`restingToday` / `hrvToday` / `recoveryScore` all null,
    baseline present) — **the state every user is in each morning until the webhook
    lands.** The baseline and the trend are still true and still printable. Yesterday
    is **never** promoted into today. `Recovery` degrades to a full true sentence;
    `Morning Vitals` promotes the 30-day averages to the primary figures;
    `Recovery Trend`'s headline is unaffected by definition; `HRV Balance` prints the
    band bounds; `Recovery Log` shows the morning as a `no reading` row.
  - **Interior gaps** — null runs mid-window (a strap left on the charger). Charts
    **break the polyline** and leave rail marks blank. Never interpolate across a
    gap, never plot it as zero, never let a gap vanish and shift the dates.
  - **Not connected** — `!present` renders a titled connect CTA per tile
    (`MiniCardEmpty`, "Connect Whoop to see recovery"). The existing behavior must
    not break for `recovery`.
  - **Both breakpoints** — full-width single-column on mobile, one-third-width on
    desktop, within the **~180px** content budget so any two can sit on one grid.
- **Tests** — priority on the state matrix and the color contract:
  - `recovery/shared.test.ts` — the status→token mapping asserted explicitly,
    including **`elevated` → `--accent` and not `--success`**, and
    **`suppressed` → `--warning` and not `--danger`**; `signed` produces `+3` / `−17`
    / `±0` with a unicode minus; `weekday` parses a local date with no timezone
    drift.
  - One test file per tile, each driven across the **four fixture states**
    (suppressed / balanced / calibrating / no-reading-yet) plus the
    **interior-gap** and **not-connected** cases, asserting the tile's own hero
    figure is present and that no state renders a bare `—` as its whole body.
  - `balance-band` — band rect drawn at the *server* bounds; baseline centre line at
    `hrvAvg`; the series **breaks** into multiple polylines around a null; today's
    point in the status color; **no band and no chart frame** when bounds are null.
  - `trend-rail` — delta derived from `shortAvg` vs `hrvAvg` (not a re-average);
    per-mark in/out classification; a ≥3-day below-band run promoted to warning;
    gaps blank; the headline still renders when today is null.
  - `morning-ledger` — a missing morning renders a `no reading` **row**, not a
    vanished one; rows newest-first; the header prints server baselines.
  - `tile-renderer.test.tsx` — all five ids render their card when
    `recovery.present`, their titled empty CTA when not.
  - `lib/dashboard-tiles.test.ts` — count `11 → 15`, the order array, and the
    `ALL_TILE_IDS` exhaustiveness record all updated.
  - **CI green** (lint/format/typecheck/test/build).

**`prog-strength-docs`**

- Flip [`dx/recovery-tile.md`](../dx/recovery-tile.md) to `status: selected`, noting
  **all five idioms** as the winning set; mark this SOW `shipped` on merge.

### Non-Goals

- **Any new API data, endpoint, payload field, or fetch.** All five compositions are
  derived entirely from the `RecoveryView` the tile already receives. The API change
  is **catalog registration + section gating only** — no change to
  `buildRecoverySection`'s body, to the whoop repo layer, or to any DTO.
- **Any design-system or shared-token change.** `scope: in-system` against v0.4 —
  conform only. No token/accent/type edit, no `design-system.md` change, **no
  invented recovery hue**, and no re-use of the recovery-score green/yellow/red for
  the HRV axis.
- **Any layout migration or stored-layout rewrite.** The `recovery` id is preserved
  precisely so none is needed. No SQL, no backfill, no `CHECK` constraint (there
  isn't one — migration 049; the write path validates and the read path filters).
- **Changing the default layout.** A connected user still gets one recovery tile;
  the `hasConnectedWhoop` gate and `defaultLayout`'s composition are untouched.
- **Touching the other ten tiles** or the shared `Spark` / `BigNum` / `MiniCard` /
  `MetaRow` / `MacroBar` primitives — they keep serving the rest of the grid
  unchanged. `Spark` is not modified; the recovery tiles simply stop using it.
- **The deep `/recovery` page** and `lib/recovery.ts`'s score bands — out of scope
  and unchanged. These tiles link into that page; they do not restyle it.
- **`prog-strength-mobile`.** Mobile carries no tile-catalog mirror and does not
  consume the dashboard layout; it is not in `repos:`.
- **Promoting the DX mockup code.** `app/design-explore/recovery-tile/**` is the
  **visual spec, not code to copy** — the throwaway `MockCard` shell, the fixture
  file, and the `_variants/*` sources stay on the unmerged DX branch. The
  `design-explore` route stays gated behind `NEXT_PUBLIC_ENABLE_DESIGN_EXPLORE` and
  never ships. **The DX PR is closed, never merged.**

## Implementation Details

### API — catalog (`internal/dashboard/tiles.go`)

Four constants added after `TileRecovery`, and the same four appended to `Catalog`
in that position. Order is load-bearing: it fixes the web tray order, and
`tiles_test.go`'s `TestCatalog_Order` plus the TS mirror test both assert it.

```go
TileRecovery      TileID = "recovery"
TileHRVBalance    TileID = "hrv_balance"
TileMorningVitals TileID = "morning_vitals"
TileRecoveryTrend TileID = "recovery_trend"
TileRecoveryLog   TileID = "recovery_log"
TileStreak        TileID = "streak"
```

`ValidTileID` and `catalogSet` derive from `Catalog` and need no edit, so the layout
write-path validation picks the new ids up for free.

### API — section gating (`internal/dashboard/handler.go`)

Replace the single `if enabled[TileRecovery]` with a family predicate, keeping the
`"recovery"` key and building the section **once**:

```go
recoveryFamily := []TileID{
    TileRecovery, TileHRVBalance, TileMorningVitals, TileRecoveryTrend, TileRecoveryLog,
}
anyRecovery := false
for _, id := range recoveryFamily {
    if enabled[id] {
        anyRecovery = true
        break
    }
}
if anyRecovery {
    out[string(TileRecovery)] = h.buildRecoverySection(ctx, r, userID, now, loc)
}
```

The surrounding `defer1` / nil-typed-pointer-marshals-to-null convention is
unchanged: a family tile enabled with no Whoop data still serializes `"recovery":
null`, which the web adapter reads as `present: false` and renders as the connect
CTA. The `out` map is deliberately keyed by section, not by tile — this is the first
place those two sets diverge, and the handler test pins it.

### Web — catalog mirror (`lib/dashboard-tiles.ts`)

The `TileId` union and `TILE_CATALOG` gain the four ids in the Go order, all with
`href: "/recovery"`, and the `recovery` entry's `description` is rewritten. The
file's header comment already states the mirror invariant; no structural change.

### Web — the five tiles (`app/(app)/dashboard/_components/recovery/`)

`whoop-card.tsx` is retired into this directory: `RecoveryCard` becomes
`readiness-verdict.tsx`'s export and `RecoveryCardEmpty` is generalized to take the
tile's `title`, so all five share one empty body with five different headings.
`whoop-card.test.tsx` follows it.

- **`shared.ts`** — the status→token switch, the house status words, the trend
  glyph/word pairs, signed-delta formatting with a unicode minus, and local-date
  weekday parsing. Pure, no React, tested directly. Reads the v0.4 CSS vars by name
  (`--warning`, `--success`, `--accent`, `--muted`) and **never a raw hex**.
- **`readiness-verdict.tsx`** (`Recovery`) — a verdict line in the house voice at
  the top in generous leading, over three divided contributor rows (score / resting
  HR / HRV), each with its value and a muted baseline-delta chip. No chart, no giant
  numeral. Status color lands on **one word**. The no-reading branch renders a
  complete sentence built from the server baselines; the calibrating branch renders
  the `n of 14 nights` sentence.
- **`balance-band.tsx`** (`HRV Balance`) — a modest headline (today's ms + status
  word) over an SVG that owns the card. The band is a `rect` between `y(balancedHigh)`
  and `y(balancedLow)` filled `--success` at low opacity; the baseline is a dashed
  centre line at `hrvAvg`; the 30-day series is a **muted** polyline **split into
  gap-free segments** so a null breaks the line rather than interpolating across it;
  today's point is a circle in the status color. The scale spans the union of the
  series and the band with headroom, computed once. Guards on
  `balancedLow`/`balancedHigh` being null and renders the calibrating progress state
  instead of an empty frame.
- **`three-dial-vitals.tsx`** (`Morning Vitals`) — a `grid-cols-3` of equal cells
  divided by hairlines, each cell a label, a tabular figure, and a `vs 30d ±n`
  caption. Uniform weight; the grid is the hierarchy. Exactly **one status dot**,
  beside HRV. In the no-reading state each cell promotes its server baseline to the
  primary figure and the caption becomes `30d avg`, so the card is three facts rather
  than three em-dashes.
- **`trend-rail.tsx`** (`Recovery Trend`) — one large delta figure (`shortAvg` vs
  `hrvAvg`, as `▼ 10%` with `−8.9 ms` beside it) in the status color, the trend word
  beneath, then a full-width flex rail of one mark per day in `days`. Marks classify
  in-band → `--success`, out-of-band → `--faint`, null → `--surface-2` (blank), and a
  run of ≥3 consecutive out-of-band days promotes to `--warning` as a sustained dip.
  Today is just the last mark. Guards on `hrvAvg`/`shortAvg` null.
- **`morning-ledger.tsx`** (`Recovery Log`) — a quiet baseline header
  (`91 ms · 52 bpm · 68`, or `n of 14 nights` while calibrating) over the last four
  `days` entries reversed to newest-first, each a fixed-width weekday (`Today` for
  the last), then HRV / resting HR / score in aligned tabular columns. A fully-null
  morning renders an italic `no reading` row. The **only** color on a row is the HRV
  delta glyph against the server baseline.

### Web — tile renderer (`tile-renderer.tsx`)

Five cases replacing the one, each with the same present/empty shape:

```tsx
case "hrv_balance":
  return data.recovery.present
    ? <HrvBalanceCard section={data.recovery} href={href} />
    : <RecoveryConnectCard title="HRV Balance" href={href} />;
```

No other case in the switch is touched, and the `never` default still makes a
missing case a **type error**.

## Rollout

1. **`prog-strength-api`** — the four `TileID`s, the `Catalog` insertion, the
   family re-gate in `handler.go`, the `tiles_test.go` list updates, and the handler
   gating tests, in one PR. **Merges and deploys first**: the web tray must not offer
   a tile the API's write path will reject as an invalid id.
2. **`prog-strength-web`** — the catalog mirror, the five tile components, the
   shared module, the renderer rewire, and the tests, in one PR. Vercel preview to
   verify each tile against a suppressed morning, an ordinary balanced Tuesday, a
   calibrating new user, a pre-webhook 7am, a strap-on-the-charger interior gap, and
   an unconnected user — at both breakpoints, and with **two and three recovery
   tiles on one grid** beside Steps and Blood Pressure.
3. **`prog-strength-docs`** — flip `dx/recovery-tile.md` to `status: selected` (all
   five idioms), mark this SOW `shipped`.

### Verification after rollout

- The **add-tile tray offers five recovery tiles** with five distinct titles and
  five distinct descriptions, in catalog order after Blood Pressure. Adding each one
  persists, survives reload, and renders its own card.
- **A user who adds only *HRV Balance*** — with no `recovery` tile on their layout —
  gets a **populated band chart**, not a blank card. (This is the bug the re-gate
  exists to prevent; check it explicitly.)
- **Existing Whoop users were upgraded silently**: a stored layout containing
  `recovery` now renders the verdict card, with no layout edit and no lost tile.
- **No two tiles on one grid print the same figure as their hero** — the pair reads
  as two facts, not one fact twice.
- **Today vs baseline is instant on every tile.** No tile shows a today-value
  without its server baseline; no tile's figure disagrees with the deep `/recovery`
  page, because nothing is re-averaged client-side.
- **The per-user band comes through** — `HRV Balance` visibly draws *this athlete's*
  range, not a generic "normal HRV" band.
- **Nothing is mostly em-dashes at 7am.** Every tile says something true in the
  no-reading-yet state; yesterday is never promoted into today.
- **The calibrating state looks intentional** on all five — honest `n of 14`
  progress, no `NaN`, no band at zero, no empty chart frame.
- **Interior gaps break honestly** — the polyline splits, the rail mark is blank, the
  ledger shows a `no reading` row; no interpolation, no zero-plot, no date shift.
- **The two color axes stay out of each other's way**: `suppressed` is warning and
  never danger red, `balanced` is calm, and **`elevated` reads as unusual, not as a
  bigger green**.
- **They stay calm dashboard tiles** — each within ~180px, quiet enough to sit beside
  Steps and Blood Pressure, still reading as v0.4 (near-black, periwinkle as meaning,
  desaturated status, no invented recovery hue) and previewing the `/recovery` page
  they link into.
- `design-system.md` unchanged; the `design-explore/recovery-tile` route stays gated
  and 404s in production; **no DX mockup code shipped**; the DX PR is **closed, never
  merged**.

## Open Questions

1. **Tray density — five of fifteen tiles are now recovery.** A flat tray puts a
   third of the catalog under one domain, which is a lot of scrolling for a user
   looking for Nutrition. **Lean:** ship the tray **flat** in catalog order (the
   recovery ids are contiguous, so they already read as a group) and revisit grouping
   or section headings as a separate tray-UX change if it actually feels unwieldy in
   preview. Do **not** grow this SOW into a tray redesign.
2. **`trend-rail`'s 30 marks at one-third width.** On desktop a third-width card is
   ~250px inside padding; 30 flex marks with 2px gaps leaves each mark ~6px, which is
   fine — but at the *narrowest* mobile breakpoint minus padding it can drop below a
   comfortable tap/read size. **Lean:** keep all 30 marks with `flex-1` and a
   `min-width`, confirm at preview; if it reads as mush, shorten to the trailing 21
   days rather than shrinking the marks. Confirm at review.
3. **`balance-band`'s height budget.** The mockup's chart is 92px; plus headline,
   caption, `MiniCard` title and `gap-3` gutters that lands right at the ~180px
   ceiling. **Lean:** keep the chart at 92px and tighten the caption to a single line;
   if it overruns, the caption is the first thing to go (the band bounds are already
   spatially legible from the chart). Measure both breakpoints at preview.
4. **Section-key vs layout-id divergence.** After the re-gate, the summary response
   can carry a `recovery` section that is not in `layout`. The web adapter already
   handles it, and mobile does not consume the layout — but this is a new contract
   property. **Lean:** pin it with the handler test called out above and note it in
   the handler's comment; no consumer change expected. Flag at review if any other
   client turns out to assert `sections ⊆ layout`.
5. **The legacy `spark` field.** No production tile reads `RecoveryView.spark` after
   this ships — it survives only because the API still serializes
   `resting_hr_spark`. **Lean:** leave it in place; removing it is a payload change
   and belongs to a separate cleanup with the API in `repos:`. Out of scope here.
6. **`Recovery`'s recovery-score contributor row and the score's own color band.**
   `readiness-verdict` prints the score as a neutral contributor row while
   `lib/recovery.ts` has a well-established green/yellow/red for it, which users are
   trained on from Whoop. **Lean:** keep the row **neutral** as the DX specifies — the
   verdict word is that tile's single color, and painting the score too is exactly the
   two-traffic-lights failure. Confirm at review that the uncolored score doesn't read
   as a regression against the deep page.
