---
type: dx
status: awaiting_selection
surface: recovery-tile
idioms:
  - balance-band
  - readiness-verdict
  - three-dial-vitals
  - trend-rail
  - morning-ledger
references:
  - Whoop
  - Oura
  - Garmin Connect
  - The Athletic
  - Robinhood
scope: in-system
variant_count: 5
repos:
  - prog-strength-web
  - prog-strength-docs
---

# DX: Recovery Tile

**Status**: Awaiting selection · **Last updated**: 2026-08-02

> A DX (Design Exploration) is the platform's **divergent** work type. Unlike a
> SOW it does not converge on one correct implementation — it produces N
> differentiated visual variants of a single frontend surface, side by side on
> one comparison route, awaiting a human pick. It **never merges** and ships no
> production code; the chosen direction feeds a downstream SOW that builds it for
> real.

## Context

The dashboard shipped from [`dx/dashboard`](./dashboard.md) as a grid of
per-domain mini-cards, each wearing the same furniture: a `BigNum` headline, a
normalized `Spark` line, a `MetaRow` of facts. [`dx/steps-tile`](./steps-tile.md)
was the first admission that the furniture doesn't fit every metric — it shipped
`micro-bars-goal-line`, and the Steps tile now draws goal-relative bars where a
sparkline used to be. **Recovery is the next tile where the generic grammar
actively misinforms**, and it has further to fall than Steps did.

The current tile renders a resting-HR headline, a min/max-normalized sparkline of
recent resting HR, and the recovery score as a footnote. Three things are wrong
with that, and they compound:

- **A sparkline of resting HR answers a question nobody asks.** Normalized
  across seven days, the line shows "up-ish" or "down-ish" with no scale — a
  1 bpm wobble and a 9 bpm spike draw the same picture. Resting HR is a
  narrow-range metric where the *size* of the deviation is the entire signal, and
  normalization is precisely what destroys it.
- **The headline number has nothing to be measured against.** "51 bpm" is a
  number. "51 bpm against your 30-day average of 52.4" is a fact about the
  athlete. Until [`sows/dashboard-recovery-metrics-payload.md`](../sows/dashboard-recovery-metrics-payload.md)
  landed, the second one was not expressible; the tile showed the first and left
  the arithmetic to the user's memory.
- **HRV — the metric a recovery surface exists to talk about — was not on the
  tile at all.** It was fetched, stored, serialized, and dropped by the web
  adapter. That is now fixed, and the tile still doesn't show it.

The payload SOW closed the data gap deliberately so this exploration could
happen: the tile can now draw a 31-day date-aligned history, place today against
the user's own baseline, and label HRV `balanced` / `elevated` / `suppressed`
using a band computed from that athlete's own night-to-night variability. A
single HRV reading is close to meaningless — 74 ms is a good morning for one
person and an alarm for another — and the band is what turns it into
information.

**This exploration differs from every prior DX in one structural way, and it is
the point of the exercise.** [`sows/customizable-dashboard-tiles.md`](../sows/customizable-dashboard-tiles.md)
made the grid user-composed: tiles are added from a tray and ordered by the user.
That means the selection gate here is **not** "which one variant replaces the
card" — it is **"which two or three of these ship as separate, independently
addable tiles."** A user who cares about recovery should be able to put *HRV
Balance* and *Morning Vitals* on their dashboard together, and a user who
doesn't should be able to have neither. The variants below are therefore drafted
as **complementary tiles with non-overlapping jobs**, not as five renderings of
the same idea. See *Composition* below — it is a hard constraint on the spread,
not a nice-to-have.

`scope: in-system`: the foundation is decided (see
[`../design-system.md`](../design-system.md) **v0.4**, oura-calm) — soft
near-black ramp, the single **periwinkle** accent (`#9aa6d6`, app-chrome and
meaning only), **Manrope** with tight numeric tracking, tabular figures, 14px
hairline panels, desaturated status colors. Variants do **not** re-litigate
palette, accent, or type. They diverge on **layout, structure, density, and
composition** inside a mini-card.

## The surface

The **Recovery mini-card** on `app/(app)/dashboard/_components/whoop-card.tsx`
(`RecoveryCard`), one cell of the responsive
`grid-cols-1 sm:grid-cols-2 lg:grid-cols-3` `CardGrid`. A variant owns everything
inside the `MiniCard` shell — the uppercase title, the `p-4` padding, and the
whole-card `next/link` into `/recovery` are chrome it keeps functional — and
composes the card body.

**The space budget is the hard constraint, and it is tighter here than it looks.**
Today the body is `BigNum` (~32px) + `Spark` (`h-7` = 28px) + `MetaRow` (~16px)
with `gap-3` gutters — roughly **150–170px** of content. Treat **~180px total**
as the ceiling. Because two of these may sit on one grid, a variant that only
works at 240px is not a candidate; it is a deep-page component.

**Neighbors.** The tile sits beside ten others — Running, Walking, Cycling,
Hiking, Lifting, Steps, Nutrition, Bodyweight, Blood Pressure, Streak — under one
grid. Two are worth studying before designing: **Steps** already broke from the
sparkline (`StepsGoalBars`, goal-relative bars) so the grid's furniture is
provably not uniform, and **Blood Pressure** is the closest neighbor in kind — a
clinical metric with a dual value, a categorical band, and a 30-day average in
its meta row. A recovery tile should look like it belongs to that family.

### The data

**Everything below already ships** — no backend change for this round. From
`prog-strength-web/lib/dashboard.ts`, after
[`sows/dashboard-recovery-metrics-payload.md`](../sows/dashboard-recovery-metrics-payload.md):

```ts
type RecoveryView = {
  restingToday: number | null;    // today's resting HR, bpm
  recoveryScore: number | null;   // today's Whoop score, 0–100
  hrvToday?: number | null;       // today's HRV (RMSSD), ms
  spark: number[];                // LEGACY 7d resting-HR spark, gap-OMITTING
  days?: RecoveryDayPoint[];      // 31 date-aligned days, oldest→newest, nulls preserved
  baseline?: RecoveryBaselineView;
  hrv?: RecoveryHrvView;
};

type RecoveryDayPoint = {
  date: string;                   // YYYY-MM-DD, local
  restingHr: number | null;
  recoveryScore: number | null;
  hrv: number | null;             // ms
};

type RecoveryBaselineView = {
  windowDays: number;             // 30
  restingHrAvg: number | null;    // trailing mean resting HR — "average heart rate"
  restingHrDays: number;          // sample count behind it
  hrvAvg: number | null;          // trailing mean HRV — the baseline
  hrvStdDev: number | null;       // the athlete's own spread
  hrvDays: number;
  recoveryScoreAvg: number | null;
  recoveryScoreDays: number;
};

type RecoveryHrvView = {
  status: "balanced" | "elevated" | "suppressed" | "unknown";
  balancedLow: number | null;     // baseline − 1 SD  (drawable)
  balancedHigh: number | null;    // baseline + 1 SD  (drawable)
  zScore: number | null;          // (today − baseline) / SD
  trend: "rising" | "falling" | "steady" | "unknown";
  shortAvg: number | null;        // 7-day mean HRV
};
```

Four things about this shape that variants must respect:

1. **`days` is the honest series; `spark` is not.** `spark` omits days without a
   reading, which shortens the array and destroys date alignment — a per-day
   chart drawn from it misplaces every point. `days` includes every date in the
   window with null metrics where there is no reading. **Use `days`.** `spark`
   exists only because the shipped card reads it.
2. **Never recompute a server figure.** `hrvAvg`, `restingHrAvg`, `balancedLow`,
   `balancedHigh`, and `zScore` are computed server-side over a window that
   *excludes today*. Display them as received. A variant that averages `days`
   itself will disagree with the deep page and with itself.
3. **The band is per-user, not per-population.** `balancedLow`/`balancedHigh` are
   the athlete's own baseline ± 1 of the athlete's own standard deviations. A
   variant that draws the band is drawing something true about *this* user; a
   variant that hardcodes a "normal HRV range" is lying.
4. **The new fields are typed optional.** `adaptRecovery` always populates them
   when the section is present, but `RecoveryView` marks `days?`/`baseline?`/
   `hrv?` optional, so TypeScript requires a guard. Guard once at the top of the
   variant and render the calibrating state if absent — do not `!`-assert.

### Color logic — the trap on this surface

Recovery already has a color story, and it has **two three-state axes that must
not fight**:

- The **Whoop recovery score** band (`lib/recovery.ts`): success ≥ 67, warning
  34–66, danger ≤ 33. Users are trained on green/yellow/red here, and the deep
  page already uses it.
- The **HRV balance status**, which is a *different axis entirely* — it says
  whether today's HRV is typical for this athlete, not whether it is good.

Painting both at full strength inside a 180px card produces two competing
traffic lights and reads as noise. **Pick one axis to carry color in each
variant**; the other stays neutral ink. Beyond that:

- **`suppressed` reads as warning** (`#d6b87f`) — never danger red. A low-HRV
  morning is information, not an emergency, and the app does not alarm a normal
  bad night.
- **`balanced` reads as success** (`#86b39f`) or as calm neutral. Balanced is
  the *ordinary* state — roughly two-thirds of days — so it should feel
  unremarkable, not celebratory.
- **`elevated` must not automatically read as "better than balanced."** HRV well
  above baseline is unusual, not necessarily good. Render it as accent or
  neutral-informational, not a bigger green. This is the single most common way
  a variant will get the semantics wrong.
- **`unknown` is muted** (`#5b6168`) and never an em-dash alone — see the states
  below.
- Recovery has **no `--discipline-*` hue** (the system enumerates run green-teal
  and lift steel-blue only). Do not invent one.

### States every variant must render

This is where a lazy tile design falls apart. The mockup must show all of these:

- **Calibrated + suppressed** — the headline case. Today's HRV below the band,
  trend falling. Should read as "something true and slightly concerning" without
  alarming.
- **Calibrated + balanced** — the boring good day, which is most days. If a
  variant only looks good in the dramatic state, it is the wrong variant.
- **Calibrating** (`hrvDays < 14`) — `hrvAvg`, `hrvStdDev`, both band bounds and
  `zScore` are **null**; status and trend are `unknown`. **Every new Whoop user
  lives here for two weeks.** No `NaN`, no band drawn at zero, no empty chart
  frame. Use the counts (`hrvDays`, `restingHrDays`) to show honest progress —
  "9 of 14 days" — the way the HR-zone widget shows "calibrating".
- **No reading yet today** (`restingToday`/`hrvToday`/`recoveryScore` all null,
  baseline present) — **the state every user is in each morning until the Whoop
  webhook lands, and all day when a night goes unrecorded.** The baseline and
  the trend are still true and still worth showing.
  [`dx/recovery-page.md`](./recovery-page.md) names the deep page's version of
  this its worst failure: a hollow ring and four em-dashes on a surface whose
  entire job is telling you about your recovery. **A variant that degrades to
  em-dashes here has failed the brief.** Yesterday is never promoted into today —
  an absent webhook must not read as stale readiness — but "your 30-day baseline
  is 91 ms and this week is trending down" is true, available, and printable.
- **Interior gaps** — `days` with null runs mid-window (a strap left on the
  charger). Charts must break the line or skip the point. Never interpolate
  across a gap, never plot it as zero.
- **Not connected** — the section is absent entirely and the existing
  `RecoveryCardEmpty` CTA renders ("Connect Whoop to see recovery"). Unchanged;
  no variant needs to redesign it, but every variant must not break it.
- **Both breakpoints** — full-width single-column on mobile, one-third-width on
  desktop.

### Representative fixture

The headline fixture. `days` is abridged for legibility — **build it out to the
full 31 entries** (30 baseline dates + today), drawing HRV from roughly a
mean-91 / SD-12.6 distribution, resting HR 49–56, score 45–85, with **at least
one interior all-null day** so the gap case is visible in the default view:

```ts
const recovery: RecoveryView = {
  restingToday: 51,
  recoveryScore: 58,
  hrvToday: 74,
  spark: [53, 52, 51, 51],        // legacy, gap-omitting — do not draw from this
  days: [
    // …extend to 31 entries, oldest→newest, ending today…
    { date: "2026-07-26", restingHr: 52, recoveryScore: 70, hrv: 89 },
    { date: "2026-07-27", restingHr: 53, recoveryScore: 71, hrv: 92 },
    { date: "2026-07-28", restingHr: 52, recoveryScore: 74, hrv: 95 },
    { date: "2026-07-29", restingHr: null, recoveryScore: null, hrv: null }, // gap
    { date: "2026-07-30", restingHr: 52, recoveryScore: 69, hrv: 86 },
    { date: "2026-07-31", restingHr: 51, recoveryScore: 66, hrv: 81 },
    { date: "2026-08-01", restingHr: 51, recoveryScore: 58, hrv: 74 },      // today
  ],
  baseline: {
    windowDays: 30,
    restingHrAvg: 52.4, restingHrDays: 27,
    hrvAvg: 91.2, hrvStdDev: 12.6, hrvDays: 26,
    recoveryScoreAvg: 68.1, recoveryScoreDays: 27,
  },
  hrv: {
    status: "suppressed",
    balancedLow: 78.6, balancedHigh: 103.8,
    zScore: -1.37,
    trend: "falling",
    shortAvg: 82.3,
  },
};
```

Read as: today's 74 ms sits 1.37 of this athlete's own standard deviations below
their 91.2 ms baseline — outside their balanced band of 78.6–103.8 — and the week
(82.3 ms) is trending down, not just this morning. Derived figures a variant may
show: **−17.2 ms vs baseline** (−19%), **−1.4 bpm resting HR vs average**,
**−10 score vs average**, **the week's mean 8.9 ms (−10%) under baseline**, and
a count of recent days sitting below the band once `days` is built out.

**Also build**: a `balanced`/`steady` fixture (today 94 ms, z ≈ +0.22), a
**calibrating** fixture (`hrvDays: 9`, all averages and bounds null, status and
trend `unknown`), and a **no-reading-yet-today** fixture (today's three metrics
null, baseline and trend intact). These are mockups — **static fixtures that look
real are preferred**; do not wire variants to live recovery services.

### Composition — these are candidate tiles, not candidate redesigns

The likely outcome is that **two or three of these ship as separate catalog
tiles**, so the spread is constrained accordingly:

- **No two variants may hero the same figure.** If two variants both put today's
  HRV in 32px type, keeping both means printing the same number twice on one
  grid. Each variant below is assigned a distinct hero; that assignment is
  binding.
- **Each variant carries its own title and its own tray description** — not five
  cards all titled `RECOVERY`. Proposed titles are given per idiom. The
  comparison route must render each variant with its own title so the pairing
  can actually be judged.
- **The comparison route must include at least one pair-in-grid mock** — two
  different recovery variants side by side in a real `CardGrid` with two or three
  neighbor tiles (Steps, Blood Pressure) around them. Judging these one at a time
  in isolation is exactly the mistake this DX exists to avoid.
- **A kept variant costs**: one `TileId` in both mirrors (`lib/dashboard-tiles.ts`
  and the Go `Catalog` in `internal/dashboard/tiles.go`), a catalog entry, a
  `TileCard` case, and a contract-test update. **No new API work** — every
  variant reads the same `recovery` section. One implementation note for the
  downstream SOW: `buildRecoverySection` is currently gated on
  `enabled[TileRecovery]`, so it must be re-gated on *any* recovery-family tile
  being enabled, or a user who adds only *HRV Balance* gets a nil section.

## Idioms

Five compositions of the same near-black / periwinkle / Manrope mini-card, each
heroing a **different** element and leaning on a different reference. They
diverge along **type scale** (one giant figure vs even three-column tabular vs
small mono rows), **color logic** (which of the two three-state axes carries
color, and where), and **spacing rhythm** (chart-dominant vs editorial vs gridded
vs dense ledger). All five fit ~180px.

- **balance-band** — *Proposed title: `HRV Balance`.* **Heroes the band.** The
  most literal answer to the brief: a compact 30-day HRV chart where
  `balancedLow`–`balancedHigh` is drawn as a filled horizontal zone behind the
  series, so "am I inside my normal range?" is a spatial question, not an
  arithmetic one. Today's point is marked and carries the status color; the
  baseline is a faint center line through the band. A small headline gives
  today's HRV in ms with the status word beside it. Type scale: modest headline
  over a chart that owns most of the card — the chart is the hierarchy. Color
  logic: the **HRV axis carries all the color** (band fill in desaturated
  success, today's point in warning when suppressed, accent when elevated); the
  recovery score is absent or a neutral caption. Spacing: chart-dominant, tight
  gutters, near-full-bleed within the padding. Borrowing **Oura**'s "your normal
  range" band treatment. → *"is today normal for me?"* — the tile that makes the
  per-user threshold visible rather than asserted.

- **readiness-verdict** — *Proposed title: `Recovery`.* **Heroes the sentence.**
  No chart at all. A verdict line in the house voice — *"Suppressed — HRV is 19%
  below your 30-day baseline"* — sits at the top in generous leading, followed by
  three quiet contributor rows (score, resting HR, HRV), each with its value and
  a baseline-delta chip (`58 · −10`, `51 bpm · −1.4`, `74 ms · −17`). The
  no-reading-yet state is where this idiom earns its place: it degrades to *"No
  reading yet today. Your 30-day baseline is 68 · 52 bpm · 91 ms"* — a full,
  true sentence where every other treatment shows em-dashes. Type scale:
  dramatic big/small editorial contrast, the verdict large-ish, everything else
  small; no giant numeral anywhere. Color logic: near-monochrome ink, **status
  color on exactly one word** (the verdict), delta chips in muted/success.
  Spacing: airy, editorial, generous line height — the calmest card on the grid.
  Borrowing **The Athletic**'s headline-sentence framing. → *"just tell me what
  today means"* — and the only variant that is never mostly em-dashes.

- **three-dial-vitals** — *Proposed title: `Morning Vitals`.* **Heroes the three
  numbers as equals.** A 3-up row of compact stat cells — score, resting HR, HRV
  — each with its value over a tiny baseline-delta caption (`vs 30d avg −10`).
  No hero figure; the grid *is* the hierarchy, and the answer to "which number
  matters today" is deliberately "all three." A single status dot beside the HRV
  cell is the only interpretation offered. Type scale: even Manrope, three equal
  tabular columns, uniform weight — the most restrained type on the grid. Color
  logic: near-monochrome with **one status dot**; neither three-state axis is
  painted at strength. Spacing: gridded, symmetric, dense, three equal columns
  with hairline dividers. Borrowing **Garmin Connect**'s morning-snapshot stat
  grid. → *"give me the panel"* — the compact everything-tile, and the natural
  partner for any of the interpretive ones.

- **trend-rail** — *Proposed title: `Recovery Trend`.* **Heroes the direction,
  and deliberately demotes today.** One large delta figure — `▼ 10%` or
  `−8.9 ms` — for `shortAvg` against `hrvAvg`, with the trend word beneath
  (*falling this week*). Under it, a full-width rail of 30 small day marks, each
  shaded by whether that day sat inside or outside the band, gaps left blank.
  Today is just the last mark on the rail, not the headline. This is the only
  variant that answers "which way am I heading" rather than "where am I now,"
  and the only one whose primary figure is still meaningful when today has no
  reading. Type scale: one big delta numeral over a tiny rail — the strongest
  size contrast in the spread. Color logic: status on the delta figure; per-mark
  in-band/out-of-band shading in success/muted with warning for sustained runs
  below. Spacing: one figure stacked over a single full-width horizontal rail,
  no columns. Borrowing the **GitHub** contribution row's spatial consistency
  read, with **Robinhood**'s delta-figure headline. → *"which way am I
  heading?"* — the direct complement to `balance-band`.

- **morning-ledger** — *Proposed title: `Recovery Log`.* **Heroes the log.** A
  compact stack of the 3–4 most recent mornings as dated rows:
  `Fri · 74 ms ▼ · 51 bpm · 58`, each row a weekday, the three metrics, and a
  delta sign against baseline on the HRV figure, under a quiet
  `baseline 91 ms · 52 bpm · 68` header. Missing mornings appear as a row reading
  `no reading` rather than vanishing — the gap is data. Type scale: small
  functional tabular rows, `Geist_Mono` permitted for the figures, no headline
  numeral; the column alignment does the work. Color logic: accent and status
  spent **entirely on the per-row delta signs**; rows otherwise neutral ink.
  Spacing: dense blotter, tight vertical rhythm, hairline row separation.
  Borrowing **Robinhood**'s sparkline list rows and **Whoop**'s own daily-log
  density. → *"show me the last few mornings as discrete facts"* — for the user
  who wants dated readings, not an interpretation.

## References

In-system, so what to take from each is **structural** — composition, density,
and data legibility, not their palettes or type:

- **Whoop** — take the **daily-log density** and the fact that its users are
  already trained to read a 0–100 score in three bands. Reinforces
  `morning-ledger`, and it is the reason the score's green/yellow/red must not
  be casually re-used for a different axis.
- **Oura** — take the **"your normal range" band**: a personal baseline drawn as
  a zone the day sits inside or outside, rather than a number to compare
  mentally. Drives `balance-band`; it is the single clearest precedent for
  rendering a per-user threshold.
- **Garmin Connect** — take the **morning-snapshot stat grid**: several vitals
  as equal, compact, tabular cells with a small delta caption each, no hero.
  Drives `three-dial-vitals`.
- **The Athletic** — take the **authored headline sentence**: a verdict in prose
  with the supporting figures demoted to small captioned rows. Drives
  `readiness-verdict`.
- **Robinhood** — take the **delta-figure headline** and the **inline per-row
  delta** that turns a short list into a glanceable trend. Drives `trend-rail`'s
  headline and `morning-ledger`'s rows.

## Selection criteria

A note-to-self for the pick, not a rubric the worker optimizes against. When I
compare these I am trying to decide:

- **Which two or three compose?** This is the real question. Do the pair mocks
  read as two facts or as one fact printed twice? Is there a pairing where the
  interpretive tile and the raw-numbers tile genuinely complement each other?
- **Does the per-user band actually come through** — can I see that this
  threshold is *mine*, derived from my own variability, rather than a generic
  "normal HRV" range? That was the entire point of the payload work.
- **Is "today vs my baseline" instant?** The averages are the product, not
  decoration. A variant that shows today's numbers without their baselines has
  regressed to the current tile with extra steps.
- **How does it look on an ordinary balanced Tuesday?** Most days are
  unremarkable. A tile that is only compelling in the suppressed state will be
  boring or, worse, alarming-by-absence 250 days a year.
- **What does it say before the webhook lands?** I open the dashboard in the
  morning. If a variant is mostly em-dashes at 7am, it fails the same way the
  deep page's hero does — and I already know that failure annoys me.
- **Does the calibrating state look intentional?** Two weeks is a long first
  impression for a new Whoop user.
- **Does it stay a calm dashboard tile** — within ~180px, quiet enough to sit
  beside Steps and Blood Pressure without shouting, and honest about the fact
  that two of these may be on the grid at once?
- **Do the two color axes stay out of each other's way** — is `suppressed`
  legible without being alarming, and is `elevated` clearly "unusual" rather
  than "extra good"?
- **Does it still read as Prog Strength v0.4** — near-black, periwinkle as
  meaning not chrome, desaturated status, no invented recovery hue — and does it
  preview the deep `/recovery` page it links into rather than looking like
  Whoop's widget dropped into my grid?

---

> **Lifecycle.** `status:` is editorial — the owner is the dispatch gate. It moves
> `draft` → `exploring` (worker running) → `awaiting_selection` (draft PR open,
> owner deciding) → `selected` / `abandoned`. The worker sets `awaiting_selection`
> on the `dx/recovery-tile` branch as it opens the PR; the owner sets the terminal
> value when they close it.
>
> **Handoff.** This DX ends at *chosen variants* — plural is expected here — not
> merged code. I open the draft `[DX — DO NOT MERGE]` PR, compare the variants on
> the preview deploy (`/design-explore/recovery-tile`, flag-gated behind
> `NEXT_PUBLIC_ENABLE_DESIGN_EXPLORE`, driven across the suppressed / balanced /
> calibrating / no-reading-yet fixtures **and the pair-in-grid mock**), pick the
> two or three that compose, tick their boxes, set `status: selected` (noting the
> winning idioms), and **close the PR — never merge it.** Then I open a SOW:
> *"implement the `<chosen-idiom>` recovery tiles from `dx/recovery-tile` as
> separate catalog tiles, production-quality, conforming to the design system"* —
> which adds one `TileId` per kept tile to both catalog mirrors, a `TileCard` case
> each, and re-gates `buildRecoverySection` on any recovery-family tile being
> enabled. The mockup code is never promoted as-is.
