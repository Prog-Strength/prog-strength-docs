---
type: dx
status: selected
selected_idiom: aligned-deck
surface: recovery-page-refresh
idioms:
  - aligned-deck
  - metric-focus
  - ledger-first
  - season-rail
references:
  - Apple Health
  - Garmin Connect
  - Oura
  - Whoop
  - Linear
scope: in-system
variant_count: 4
repos:
  - prog-strength-web
  - prog-strength-docs
---

# DX: Recovery Page (refresh)

**Status**: Selected — `aligned-deck` · **Last updated**: 2026-08-12

> **Selected: `aligned-deck`.** Comparison PR
> [prog-strength-web#167](https://github.com/Prog-Strength/prog-strength-web/pull/167)
> closed un-merged. The winner does **not** carry prerequisite P2, so resting HR
> ships answered by the trailing average that already exists and
> [`dx/resting-hr-tile.md`](resting-hr-tile.md)'s Open Question 3 is settled in
> the negative for now. The two downstream SOWs, in order:
>
> 1. [`sows/recovery-history-endpoint.md`](../sows/recovery-history-endpoint.md)
>    — prerequisite **P1**, the enriched recovery view at page scale.
> 2. [`sows/recovery-page-refresh.md`](../sows/recovery-page-refresh.md) — the
>    page itself, per the `aligned-deck` variant.

> A DX (Design Exploration) is the platform's **divergent** work type. Unlike a
> SOW it does not converge on one correct implementation — it produces N
> differentiated visual variants of a single frontend surface, side by side on
> one comparison route, awaiting a human pick. It **never merges** and ships no
> production code; the chosen direction feeds a downstream SOW that builds it
> for real.

> A second pass at `/recovery`. The first exploration
> ([`dx/recovery-page.md`](recovery-page.md)) ran on 2026-07-23 and built five
> variants on `dx/recovery-page`; its draft PR
> ([prog-strength-web#207](https://github.com/Prog-Strength/prog-strength-web/pull/207))
> sat unselected for three weeks and is **abandoned** rather than picked. Not
> because the variants were bad — because the ground moved underneath them. See
> *Why the first pass is stale*.

## Why the first pass is stale

The July DX was written when the dashboard had **one** recovery tile. Since it
ran, an entire recovery tile family shipped:

| Shipped | Tile | Idiom it established |
| --- | --- | --- |
| 2026-08-02 | `recovery`, `morning_vitals` | Verdict sentence with contributor rows; three-dial vitals |
| 2026-08-10 | `hrv_balance` | **Banded ribbon + rolling curve + gauge**, two views on one card |
| 2026-08-12 | `recovery_log` | Band rail over three mornings |
| 2026-08-12 | `resting_hr` | **Rank strip** — position within the athlete's own month |

Two consequences, and together they are why a second exploration is cheaper than
picking a stale winner.

**First, the page now has a house vocabulary to inherit, and didn't before.**
The July variants each invented their own chart treatment because there was
nothing to inherit. Picking one now would ship a page that ignores four tiles
built since — and in particular ignores the HRV chart, which is the best thing
in the recovery surface and which the owner explicitly wants carried forward.
`hrv-chart.ts`'s own header anticipated this: its machinery is *"the natural
import for the `/recovery` page when that chart follows."*

**Second, the page's job changed.** In July, `/recovery` was the only place
Whoop data was properly rendered, so "be the today surface" was a defensible
brief. Five tiles now answer *how am I today?* on the dashboard, at a glance,
without a click. A page whose hero is a bigger version of the score ring sitting
one scroll above is answering a question that has already been answered. **This
DX reassigns the page's job to history and depth** — see Fixed Decision 1.

What carries forward from the July DX, unchanged and not re-litigated: the
no-dead-hero principle, the paginated ledger, the Whoop settings backlink, and
the observation that the page is nearly monochrome on data with a canonical
colour story. Those were right. They are restated below as fixed decisions
rather than explored again.

## Context

The page today (the screenshot that prompted this): a hollow score ring at
`13`, two mini-stats, a `7d · 30d · 90d` toggle, three recharts line panels, and
a day log showing the three most recent mornings. Three specific problems, all
of which the tile family has since solved *at tile scale* and none of which the
page has inherited:

- **The three charts are the same chart three times.** Recovery score, resting
  HR, and HRV are rendered as identical white polylines with a dashed average
  and a fixed-or-auto y-axis. They are visually interchangeable, which is a
  strange thing to do to three metrics that mean completely different things and
  have completely different baselines. Meanwhile the HRV tile draws the same
  metric with a drifting band ribbon, per-night status marks that change *shape*
  outside the band, and a gauge — and it is legible in a third of the space.
- **The day log shows three rows.** It is reverse-chronological, unpaginated,
  and effectively truncated; a user with a year of ingested mornings can reach
  none of it. For a page whose job is now history, the ledger is the feature,
  not the footer.
- **The colour story is unused.** Recovery is *the* three-band metric and Whoop
  trained every user to read it at a glance. The page spends band colour on the
  log chips and (when today has data) the ring stroke, and nowhere else.

`scope: in-system`: the foundation is decided — near-black ramp, periwinkle
accent, Manrope, 14px hairline panels, and the desaturated status trio
(`--success` `#86b39f` / `--warning` `#d6b87f` / `--danger` `#c79292`).
Variants do **not** re-litigate palette, accent, or type. They diverge on
**layout, structure, density, navigation, and chart form**.

## The surface

`prog-strength-web/app/(app)/recovery/page.tsx` + `_components/`
(`recovery-hero.tsx`, `recovery-trends.tsx`, `recovery-log.tsx`).

The render gates — connection absent/revoked → connect CTA; error → reconnect
CTA; connected with zero rows → "your first recovery lands after tonight's
sleep" — are **fine and out of scope**, exactly as in the July DX.

### The data asymmetry — the crux of this exploration

The three metrics do **not** have equal machinery behind them, and this is the
single most important fact for anyone designing this page.
`internal/recoverytrend` emits a `Baseline` carrying `RestingHRAvg`, `HRVAvg`,
and `RecoveryScoreAvg` — three trailing averages. Beyond that:

| Metric | Trailing avg | Spread / band | Per-day classification | Baseline drift | Canonical bands |
| --- | --- | --- | --- | --- | --- |
| **HRV** | ✅ | ✅ `HRVStdDev`, `BalancedLow/High`, z-score | ✅ `DayResult` per day | ✅ `BaselineTrend` | — |
| **Recovery score** | ✅ | ❌ | ❌ | ❌ | ✅ **Whoop's 0–33 / 34–66 / 67–100** (`lib/recovery.ts`) |
| **Resting HR** | ✅ | ❌ | ❌ | ❌ | ❌ |

Read the row that matters: **the HRV tile's idiom is not portable, because it is
built on machinery that exists for HRV alone.** The drifting ribbon needs a
per-day band; the shape-changing marks need a per-day classification; the gauge
needs bounds. None of that exists for score or resting HR.

This is not an oversight — it is why the shipped `resting_hr` tile chose a
**rank strip**: a rank needs no baseline, only a distribution. And it is exactly
the question [`dx/resting-hr-tile.md`](resting-hr-tile.md) / its SOW left open
(*"Should the server compute a resting-HR band?"*, Open Question 3, lean: ship
and judge).

**Recovery score is the interesting middle case.** It has no server band, but it
has something better for charting purposes: **fixed, canonical, user-recognised
bands** that need no computation at all. A banded score chart is available
today, for free, and would agree with the user's Whoop app.

So the owner's ask — *better graphs for score and resting HR, inspired by the
HRV chart but not copying it* — is answered differently per metric, and the
variants are required to take a position rather than paint three ribbons and
hope.

### Prerequisites — read before estimating any variant

**P1 — hard, and it blocks every variant.** The page cannot draw the HRV chart
today, at all. `GET /whoop/recovery` returns bare rows:

```ts
type WhoopRecoveryDay = {
  date: string;                    // local YYYY-MM-DD
  recovery_score: number | null;   // 0–100
  resting_heart_rate: number | null;
  hrv_rmssd_milli: number | null;
};
```

No baselines, no per-day bands, no drift, no z-scores. Every figure
`prepareHrvChart` consumes arrives via `/dashboard/summary`'s recovery section,
which is dashboard-scoped and windowed for a tile. **Promoting the HRV chart to
the page therefore requires an API SOW** that serves the enriched recovery view
at page scale (longer windows, the same `RecoveryView` shape). This is a
prerequisite of the *winner*, whichever it is, and it should be scoped as its
own SOW before the page SOW.

**P2 — soft, and variant-dependent.** Resting-HR spread + per-day banding, so
RHR can be drawn with ribbon-class machinery. **Only some variants below need
it, and each one says so.** Selecting a variant that carries P2 is a decision to
fund that API work; selecting one that doesn't is a decision to answer resting
HR with what exists. **That trade is deliberately part of this pick** — it is
the cleanest opportunity the product has had to settle `resting-hr-tile`'s open
question with a picture rather than an argument.

**P3 — not required.** Paginating the full ledger needs no API change. Whoop
ingests one row per day, so three years of history is ~1,100 rows of four
fields; `GET /whoop/recovery` with a wide `since` returns the lot in tens of
kilobytes and the client pages it. Variants should assume full history is
available and page it client-side. (`GET /whoop/recovery` takes only
`timezone`, `since`, `until` — there is no cursor, and it does not need one.)

## Fixed decisions

Product calls this DX records. Every variant honours them; the downstream SOW
implements them. They are not part of the spread.

1. **The page's job is history and depth, not today.** The dashboard's five
   recovery tiles own *how am I this morning?*. The page owns what a tile cannot
   hold: long windows, per-metric detail, the full ledger, patterns across
   weeks. The hero is a **window orientation** — what this stretch of time has
   been like — not a restatement of the score ring one scroll up. A variant may
   still surface today, but today must not be the page's centre of gravity.
2. **The HRV panel inherits the shipped tile idiom. No variant redesigns it.**
   The banded ribbon, the 7-day rolling curve, the shape-changing out-of-band
   marks, and the gauge are decided and liked. Variants render them at page
   scale via `prepareHrvChart` and diverge only on *how much room it gets and
   what sits around it*. **This is the one element deliberately held constant
   across the spread** — the exploration is everything else. Nothing may
   re-derive a figure `prepareHrvChart` already returns.
3. **Score and resting HR get genuinely different charts — from each other, and
   from HRV.** Three interchangeable polylines is the defect this DX exists to
   fix. Inspiration from the HRV chart is welcome; a third and fourth ribbon is
   not. Each variant states its score treatment and its RHR treatment explicitly.
4. **The ledger is paginated over all stored history**, not over the selected
   range. A user opens the page to reach March; they must be able to. Dense
   tabular rows, tabular numerals, a quiet pager (the bodyweight table's
   20-row pagination is the house precedent). Every variant renders a real
   mid-state page, not page one.
5. **No dead hero.** Carried from the July DX. The honesty principle stands — a
   stale number must never masquerade as today — but the answer to "today hasn't
   landed" is a representative, explicitly-labelled window figure, never an
   em-dash where a number could be.
6. **Band colour works harder, within the system.** More colour means deploying
   the *decided* tokens confidently, not inventing hues. Available: neutrals,
   periwinkle accent (interactive chrome only), and the status trio for bands.
7. **A quiet "Manage Whoop connection →"** to `/settings?tab=integrations` lives
   somewhere unintrusive on the healthy page. Accent link register, not a button.
8. **Nothing recomputes a server figure.** The standing rule from
   `hrv-chart.ts`. Client arithmetic maps an already-computed value onto a pixel
   or a percentage; it does not invent a baseline, a band, or a verdict. A
   variant that wants a resting-HR band declares P2 — it does not compute one on
   the client.

## Visual states the variants must render

- **Today hasn't landed.** The 7am gap and the unrecorded night. Per decision 5,
  and per decision 1 the page should barely flinch — its subject is the window.
- **Each band present.** A green stretch, a mixed stretch, a rough stretch. The
  banding must read at a squint without going stoplight-loud on near-black.
- **Partial-null days.** A score with no HRV: an em-dash in the ledger, a **gap**
  in the chart, never a zero.
- **Missing nights.** A date absent entirely: a gap in the line, a hollow cell
  in any rail. Must not look broken.
- **Calibrating.** Fewer than `min_baseline_days` samples: no HRV band, no
  averages, honest counts. The HRV panel already handles this — the rest of the
  page must not render a confident number beside it.
- **Deep ledger.** Enough rows that pagination is real, rendered **mid-state**
  (page 4 of 37), not page one.
- **Sparse long window.** A month of data on a 90-day/1-year range: long empty
  lead-ins, averages over what exists.

## Representative fixture

Mirror the screenshot that prompted this, extended to the history the page now
owns. Static fixtures that look real are strongly preferred — **do not wire
variants to live Whoop services.**

- **Today: Wed, Aug 12 2026 — score 13 (red) · 69 bpm · 45 ms.** A genuinely bad
  morning, deliberately: the page must be legible when the news is bad, and a
  fixture full of green days proves nothing.
- Recent: **Tue Aug 11 — 78 · 50 bpm · 106 ms** (green); **Mon Aug 10 — 52 ·
  47 bpm · 96 ms** (yellow); **Sun Aug 9 — null score · 51 bpm · 88 ms**
  (partial); **no row at all for Fri Aug 7**.
- **History depth: 14 months of rows**, so the ledger pages to ~30 pages and a
  1-year range is populated. The last 30 days average **score 61 · 54 bpm ·
  89 ms**; four weeks ago the HRV baseline was **83 ms**, so baseline drift is
  **▲ +6 ms · 4w** and the HRV panel has a real drifting ribbon rather than a
  flat one.
- A **suppressed run**: five consecutive below-band HRV nights in late July, so
  the ribbon's out-of-band marks and any rail-based idiom have something true to
  show.

## Idioms

Four compositions of the same near-black / periwinkle / Manrope surface. Because
`scope: in-system`, none re-decides palette, accent, or type — and because of
Fixed Decision 2, none re-decides the HRV chart. They diverge on **how history
is navigated**, **how the three metrics relate spatially**, and **what form the
score and RHR charts take**.

Each idiom names its score treatment and its RHR treatment, and declares whether
it carries prerequisite **P2**.

---

### `aligned-deck` — three charts, one time axis

**Reference: Apple Health / trading terminals.** The page is a vertical deck of
three full-width panels sharing **one x-axis and one synchronized crosshair** —
drag anywhere and all three read out the same morning. The argument is that the
real question on a history page is *what happened to all three at once*, and
three panels with independent axes make that comparison impossible today.

- **HRV**: the tile chart at full width, ribbon and marks intact, given the most
  vertical room of the three.
- **Score**: a **banded column chart** — one column per morning, coloured by its
  canonical Whoop band, with the three zone boundaries as hairlines. Columns,
  not a line, because score is a daily verdict rather than a continuous
  quantity, and because a run of red columns is the single most legible thing
  this page could show.
- **Resting HR**: a **diverging bar chart around the trailing average** — the
  baseline is the zero line, each morning a bar above or below it. Uses only
  `RestingHRAvg`, which exists. It reads as *"how far off my normal was I"*
  without claiming a band nobody computed.
- **Ledger**: full-width beneath the deck, paginated, with the crosshair's
  selected date highlighted.
- **P2: not required.**

→ The most direct answer to "three identical polylines": keep three panels, make
each one the right *form* for its metric, and bind them to one axis so the deck
answers a question no single panel can.

---

### `metric-focus` — one metric at a time, in full

**Reference: Apple Health single-metric detail, Oura trends.** The page shows
**one large chart at a time** with a metric switcher (`Score · Resting HR ·
HRV`); the other two collapse to thin summary strips that promote on click. The
argument is that depth and comparison are different jobs, and the tiles already
do comparison — the page should do depth, giving one metric the whole width, a
real y-axis, annotations, and per-metric chrome that would be clutter if
tripled.

- **HRV** (focused): the tile chart at page scale plus what the tile has no room
  for — the gauge, the drift tag, band bounds printed on-plot, and the
  suppressed-run stretch annotated.
- **Score** (focused): a **banded area with an on-plot band legend** and each
  band's day-count printed in the margin (`green 11 · yellow 14 · red 5` for the
  window) — the distribution stated as text beside the chart that shows it.
- **Resting HR** (focused): the **`resting_hr` rank strip promoted to page
  scale** — the athlete's mornings sorted low to high across the full width,
  today's tick filled and labelled, the trailing average as a dashed tick — with
  a dated line chart *beneath* it. The rank answers *is this good for me?*; the
  line answers *what has it been doing?*. This is the idiom that takes the
  shipped tile's reasoning most seriously and extends it rather than replacing
  it.
- **Ledger**: beneath, paginated, filtered to the focused metric's column
  emphasised.
- **P2: not required** — deliberately. This variant's thesis is that resting HR
  never needed a band.

→ The variant that proves the page can be *deeper* than the dashboard rather
than merely bigger, and the one that answers the RHR question with no API work
at all.

---

### `ledger-first` — the record is the page

**Reference: Garmin Connect metric history, Linear.** Inverts the page. The
**paginated ledger is the primary column**, permanently visible and dense —
date · banded score cell · bpm · ms, one tight row per morning, 20 to a page —
and selecting a row drives a **detail panel** beside it showing that morning's
three metrics against the baseline as it stood *that day*. Charts live in the
panel, sized to it, not spread across the page.

- **HRV**: in the detail panel, the tile chart windowed around the selected
  morning, so the ribbon shows where that day sat in its own context.
- **Score**: **the ledger cell itself is the chart** — each row's score cell
  filled to its value and tinted by its band, so scanning the column *is*
  reading a horizontal bar chart of the month. Plus a compact banded column
  strip in the panel header.
- **Resting HR**: a **column of micro-sparklines**, one per row, each showing
  that morning against the trailing 7 — a "how did this land" glance at row
  scale. In the panel, the same diverging-bar treatment as `aligned-deck`.
- **P2: not required.**

→ Takes Fixed Decisions 1 and 4 most literally: if the page's job is history,
the record should be the page and the charts should serve it, rather than the
record being a footer under three charts.

---

### `season-rail` — scrub the whole history

**Reference: Whoop's trend calendar, GitHub contribution graph.** The organizing
spine is a **horizontal rail of band-coloured cells, one per stored morning**,
spanning months and scrubbable across the athlete's entire history — the page's
single densest colour moment and its navigation at the same time. Brushing a
stretch of the rail drives the three charts beneath it; the rail replaces the
`7d · 30d · 90d` toggle with a continuous one.

- **HRV**: beneath the rail, the tile chart over the brushed window.
- **Score**: the **rail itself is the score chart** — every morning a cell in its
  canonical band, hollow where a night is missing, so a rough fortnight is
  visible as a band of colour from across the room. A conventional chart beneath
  is unnecessary and deliberately omitted.
- **Resting HR**: **a banded ribbon matching HRV's form**, drawn against an RHR
  baseline and per-day band. **This variant carries P2** and is the one that
  makes the case for it: if the answer to "the page should feel coherent" is
  that all three metrics are drawn the same way, this is what that looks like,
  and the API work is the price.
- **Ledger**: a compact paginated strip below, its page following the brush.
- **P2: REQUIRED.**

→ The most structurally novel of the four, the only one that treats *all* stored
history as a navigable surface rather than a paged list, and the explicit
counter-proposal on resting HR. It is also the most expensive: it is the one
variant whose selection commits an API SOW.

---

## References

In-system, so what to take is **structural** — composition, density, and data
legibility, not palettes or type:

- **Apple Health** (single-metric detail) — the large quiet chart carrying its
  own on-plot legend and annotations, minimal surrounding chrome. Drives
  `metric-focus`, informs `aligned-deck`.
- **Garmin Connect** (metric history tables) — respect for dense tabular
  history: tight rows, aligned numerals, paging that assumes years of data,
  without its clutter. Drives `ledger-first`. Its HRV Status card is already the
  structural source of the shipped HRV tile, so `ledger-first` is closest to the
  existing house lineage.
- **Oura** (trends) — a metric switcher that makes one thing at a time feel
  complete rather than partial. Drives `metric-focus`.
- **Whoop** (trend calendar) — the month-strip of band-coloured day cells; users
  already speak this language. Drives `season-rail`.
- **Linear** (two-pane density) — side-by-side working-surface composition and
  single-accent restraint at high information density. Informs `ledger-first`.

**Not a reference: the shipped HRV tile.** It is not something to take from — it
is the fixed element every variant carries (Fixed Decision 2).

## Selection criteria

A note-to-self for the pick, not a rubric the worker optimizes against. When
comparing these I want to decide:

- **Does the page earn its click, given the dashboard?** The tiles answer today.
  If a variant's strongest moment is a bigger score ring, it has failed the
  brief regardless of how well it is drawn.
- **Do the score and resting-HR charts feel as considered as the HRV one?** This
  is the whole ask. The HRV panel will look good in every variant because it is
  the same panel — I should be comparing the *other two*, and I should
  deliberately look at them with the HRV panel covered up.
- **Is resting HR answered, or dodged?** Two variants say it never needed a
  band; one says all three should match and asks for API work to get there. Both
  are legitimate. This pick settles `resting-hr-tile`'s Open Question 3, so it
  deserves more than a glance.
- **Can I get to March?** Fixed Decision 4 is a requirement, but the *feel* of
  reaching deep history differs enormously between a pager and a scrub rail.
  Which one would I actually use.
- **Does the bad morning read well?** The fixture opens on a 13. A recovery page
  that is beautiful on green weeks and alarming on red ones is worse than one
  that is calm throughout.
- **Does the colour finally do something, without going stoplight?** A green
  stretch and a rough stretch should look different at a squint while still
  sitting calmly beside Activities and Bodyweight.
- **Does it still read as Prog Strength** — near-black, accent kept to chrome,
  bands kept to state — rather than a Whoop or Garmin re-skin?
- The missing night, the null-HRV day, the calibrating state, and the sparse
  1-year range should all survive the treatment without looking broken.

## Cost note for the pick

Three of the four variants are buildable on prerequisite **P1 alone** (serve the
enriched recovery view at page scale). `season-rail` additionally requires
**P2** (resting-HR spread and per-day banding in `recoverytrend`). That is a
real difference in downstream cost and it should be weighed **as a design
question, not a budget one** — if all-three-drawn-alike is genuinely the right
page, the API work is worth it, and if it isn't, no saving justifies picking a
worse page.

---

> **Lifecycle.** `status:` is editorial — the owner is the dispatch gate. It
> moves `draft` → `exploring` (worker running) → `awaiting_selection` (draft PR
> open, owner deciding) → `selected` / `abandoned`. The worker sets
> `awaiting_selection` on the `dx/recovery-page-refresh` branch as it opens the
> PR; the owner sets the terminal value when they close it.
>
> **Handoff.** This DX ends at a *chosen variant*, not merged code. I open the
> draft `[DX — DO NOT MERGE]` PR, compare on the preview deploy, pick one, tick
> its box, set `status: selected` (noting the winning idiom), and **close the PR
> — never merge it.** Then I open **two** SOWs, in order: first the API
> prerequisite (P1, plus P2 if the winner carries it), then *"implement
> recovery-page-refresh per the `<chosen-idiom>` variant, production-quality,
> conforming to the design system"* — carrying the fixed decisions above.
