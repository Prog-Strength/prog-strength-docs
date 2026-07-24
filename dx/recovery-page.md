---
type: dx
status: draft
surface: recovery-page
idioms:
  - readiness-command-hero
  - morning-report
  - banded-trend-canvas
  - split-ledger-control-room
  - banded-week-rail
references:
  - Whoop
  - Oura
  - Apple Health
  - Linear
  - Garmin Connect
scope: in-system
variant_count: 5
repos:
  - prog-strength-web
  - prog-strength-docs
---

# DX: Recovery Page

**Status**: Draft · **Last updated**: 2026-07-23

> A DX (Design Exploration) is the platform's **divergent** work type. Unlike a
> SOW it does not converge on one correct implementation — it produces N
> differentiated visual variants of a single frontend surface, side by side on
> one comparison route, awaiting a human pick. It **never merges** and ships no
> production code; the chosen direction feeds a downstream SOW that builds it
> for real.

## Context

The recovery page (`/recovery`) shipped from
[`sows/recovery-page.md`](../sows/recovery-page.md) as Whoop readiness data's
first-class home: a score ring for today, trend charts over a 7/30/90-day
toggle, and a day log. The plumbing is right and not in question — one fetch
feeds everything, the timezone convention is followed, the connect/empty gates
work. What's unsatisfying is that the page a user actually opens most of the
time is **mostly dead pixels**.

Three things drag it down today:

- **The hero spends most of its life as an em-dash.** The original SOW decided
  that yesterday's row must never be promoted into "today", which was the right
  call about *honesty* — but the implementation of that call is a hollow gray
  ring, "—" in the score slot, "no data yet today", and two more em-dashes for
  resting HR and HRV. Until the morning webhook lands (and all day when a night
  goes unrecorded), the most prominent pixels on a page whose entire job is
  *telling you about your recovery* say nothing at all. A readiness surface
  should always have something true to say — there is a whole window of history
  right below it to summarize.
- **The page is nearly monochrome on data with a canonical color story.**
  Recovery is *the* three-band metric — Whoop trained every one of its users to
  read green/yellow/red at a glance — yet today the band color appears only in
  the tiny log chips and (when today has data) the ring stroke. The trend lines
  are plain white-ish strokes; the score chart's band zones are barely
  perceptible washes. For a page about a color-coded score, it reads gray.
- **The day log doesn't respect its own growth curve.** It's a loose,
  full-height list — one comfortable row per day, unpaginated — and Whoop
  ingestion appends to it every single morning, forever. At 90 days it is
  already a long undifferentiated scroll; it needs to become compact and
  paginated, a ledger you consult rather than a wall you scroll past.

One smaller gap: the page depends entirely on the Whoop connection, but only
the *disconnected* states link to Settings → Integrations. A connected user who
wants to check or manage the link that feeds this page has no path to it from
here — a quiet backlink belongs somewhere on the healthy page too.

This DX exists to find the *direction* for a recovery page that always has
something to say and finally uses its color story — before committing
engineering effort. `scope: in-system`: the foundation is decided (soft
near-black ramp, periwinkle accent, Manrope, 14px hairline panels, and the
desaturated status trio that already encodes the bands), so variants do **not**
re-litigate palette, accent, or type. They diverge on **layout, structure,
density, and composition** — what the hero is when today hasn't landed, how
hard the band color works, and how the log is treated. No production recovery
code changes here.

## The surface

The `/recovery` view (`prog-strength-web/app/(app)/recovery/page.tsx` +
`_components/`). Anatomy of what's on screen today (a variant should account
for all of it, though it may reorganize, demote, or fold chrome together in
service of its idiom):

- **Header** — "Recovery" + a one-line subtitle naming the three metrics.
- **Hero** — an SVG ring filled to `recovery_score`/100, stroked in the band's
  status token, score numeral centered; two MiniStats beside it (Resting HR
  bpm, HRV ms). When today's row is absent: unfilled ring, "—", "no data yet
  today", em-dash MiniStats.
- **Range toggle** — the `SegmentedToggle` primitive, `7d · 30d · 90d`,
  default 30d.
- **Trend charts** — three recharts line panels (score with translucent band
  `ReferenceArea` zones and a fixed 0–100 axis; resting HR and HRV each with a
  dashed range-average `ReferenceLine`). Null-metric days render as gaps.
- **Day log** — reverse-chronological list: date, band-colored score chip,
  resting HR, HRV; em-dash for nulls. No pagination.
- **Render gate** — connection absent/revoked → connect CTA; error → reconnect
  CTA; connected + zero rows → "your first recovery lands after tonight's
  sleep". These states are fine and are **not** part of this exploration.

The data shape (from `prog-strength-web/lib/api.ts`):

```ts
type WhoopRecoveryDay = {
  date: string; // local YYYY-MM-DD
  recovery_score: number | null; // 0–100
  resting_heart_rate: number | null; // bpm
  hrv_rmssd_milli: number | null; // ms
};
```

One call per range selection to `GET /whoop/recovery` (timezone + local-date
convention) feeds every section; averages and band mapping are client-side.
Bands mirror Whoop so numbers agree with the user's Whoop app: **success ≥ 67,
warning 34–66, danger ≤ 33** (`lib/recovery.ts`), rendered with the design
system's desaturated status tokens `--success` `#86b39f` / `--warning`
`#d6b87f` / `--danger` `#c79292` — never the periwinkle accent, so banding
reads as state, not selection.

**Decisions fixed across all variants** (product calls this DX records; every
variant honors them, and the downstream SOW implements them):

1. **No dead hero.** This **supersedes the original SOW's "yesterday's row is
   never promoted"** as implemented. The honesty principle stands — a stale
   number must never masquerade as today — but the answer to "no data yet" is
   now a **representative window value, honestly labeled**, not an em-dash.
   Default: the selected range's averages (score average colored by its band,
   average resting HR, average HRV) under an explicit label like *"30-day
   average · today's recovery hasn't landed yet."* A variant may instead lead
   with the most recent day if it is unmistakably dated ("Yesterday · Wed Jul
   22"). What no variant may do is render a dash where a number could be.
2. **Compact, paginated day log.** This **supersedes the original SOW's
   "no pagination" non-goal.** Rows are ingested daily forever; the log becomes
   a dense ledger — tighter rows, tabular alignment — paginated at 10–15 rows
   with a quiet pager (the bodyweight table's 20-row pagination is the house
   precedent). Every variant renders the pager, not just page one.
3. **Whoop settings backlink.** A quiet "Manage Whoop connection →" affordance
   (to `/settings?tab=integrations`) lives somewhere unintrusive on the
   healthy page — footer of the log, header meta, or equivalent. Accent-colored
   link register, not a button.
4. **Band color works harder, within the system.** More color means deploying
   the *decided* tokens confidently — band tints on chart strokes/fills, banded
   cells, colored deltas — not inventing new hues. The palette available to
   every variant is: neutrals, the periwinkle accent (interactive chrome only),
   and the status trio for bands. That constraint is the exploration: how vivid
   can a calm, desaturated three-tone system feel?

**Visual states the variants must handle** (render them all in the mockup —
the first one is the whole reason this DX exists):

- **No data yet today.** The morning-webhook gap and the unrecorded night. The
  hero shows the representative value per decision 1; the page must feel alive,
  not waiting.
- **Today present, each band.** A green day (93), a yellow day (56), a red day
  (28) — the banding must read instantly on all three without going stoplight-
  loud on the near-black field.
- **Partial-null days.** A day with a score but no HRV renders an em-dash in
  the ledger and a gap in the chart — never a zero.
- **Missing days.** Whoop skips a night entirely: a date gap in the log, a gap
  in the lines. Must not look broken.
- **Sparse 90d.** A month of data on the 90-day range — lines with long empty
  lead-ins, averages computed over what exists.
- **Deep log.** Enough rows that the pager is real (page 1 of 2+), rendered
  mid-state, not just a decorative control.

**Representative fixture** (mirror the screenshot that prompted this — a month
of real ingestion, today not yet landed):

- Range: **30 days**, ending **Thu, Jul 23 2026**. No row for Jul 23 yet.
- Recent rows: **Wed Jul 22 — 56 · 56 bpm · 88 ms** (yellow); **Tue Jul 21 —
  93 · 50 bpm · 129 ms** (green); **Mon Jul 20 — 89 · 46 bpm · 118 ms**
  (green); **Fri Jul 17 — 62 · 55 bpm · — (null HRV)**; **Wed Jul 15 — 28 ·
  61 bpm · 54 ms** (red); **no row at all for Sun Jul 12**.
- Window averages: **score 64** (yellow band — deliberately not green, so the
  fallback hero shows the banding isn't always the happy color), **resting HR
  52 bpm**, **HRV 96 ms**. ~26 rows in the window → the log paginates.

These are mockups: **static fixtures that look real are preferred** — do not
wire the variants to live Whoop services.

## Idioms

Five genuinely distinct compositions of the same near-black / periwinkle /
Manrope surface. Because `scope: in-system`, none re-decides palette, accent,
or type — they diverge along **type scale** (how hard the hero numeral leans
on size against uniform Manrope), **color logic** (where the three band tokens
are allowed to land, and how much surface they cover), and **spacing rhythm**
(airy briefing ↔ dense control room). Each makes a different element the hero
and leans on a different reference. The Fixed Points hold.

- **readiness-command-hero** — The **ring stays the hero, but never empties**.
  Borrowing **Whoop's** today screen: a large band-stroked ring with a big
  numeral, and when today hasn't landed it renders the **window average** at
  reduced emphasis — a softer/dashed stroke in the average's band hue, a small
  "30-DAY AVG" kicker where "recovery today" sits, swapping to full-strength
  the moment the webhook lands. The day's (or average's) band hue propagates
  down the page — chart emphasis, delta chips, log accents — so a green day
  *feels* green everywhere. Dramatic type-scale contrast (huge numeral,
  everything else small); airy top, denser bottom. → the most direct answer to
  the dead hero: same composition, no more dash.

- **morning-report** — Borrowing **Oura's** readiness briefing, the page reads
  as a **daily report, not a dashboard**: a date kicker, a one-line coaching
  verdict in the house voice ("Solid recovery — HRV is 8% below your 30-day
  baseline"), then **contributor rows** — score, resting HR, HRV — each with
  its value, a baseline-delta arrow, and a whisper of sparkline. The no-data
  state becomes a briefing too: "While today's recovery lands: your 30-day
  baseline is 64 · 52 bpm · 96 ms." Editorial rhythm, generous leading, medium
  type scale; color logic is the most restrained of the five — band dots and
  delta arrows only. → answers the dead hero with *language*, and is the only
  idiom that makes the baselines a first-class product voice.

- **banded-trend-canvas** — Borrowing **Apple Health's** single-metric detail
  view, the **score chart is the page**: full-width and tall, the three band
  zones as unmistakable translucent color fields behind the line (the page's
  color arrives as *background*, not marks), the average as a labeled on-plot
  line, the latest point ringed and annotated. Resting HR and HRV become
  small-multiple strips beneath; the hero compresses to a compact header stat
  row (window average + latest, band-tinted). Generous whitespace, minimal
  chrome, quiet uniform type. → answers "nearly monochrome" from the opposite
  end of command-hero: the bands become the canvas itself.

- **split-ledger-control-room** — A denser **two-pane** working surface
  (**Linear's** composure, **Garmin Connect's** respect for tabular history):
  charts and a compact hero strip in one column, the **day ledger permanently
  visible** in the other — dense tabular rows (date · banded score cell · bpm
  · ms), paginated at 15, hovering a row highlighting its point on the chart.
  Crisp small type, gridded rhythm, the densest of the five; color logic is
  structural — band color as a thin row edge or score-cell fill, accent
  reserved for the active row and pager. → the idiom that takes decision 2
  most seriously: the log stops being an afterthought below the fold and
  becomes half the surface.

- **banded-week-rail** — The organizing spine is a **horizontal rail of
  band-colored day cells** spanning the selected window — Whoop's trend
  calendar with GitHub-contribution-cell energy — the page's single densest
  color moment: thirty small cells in greens, ambers, corals, hollow where a
  night is missing, today's cell outlined-empty until data lands. Selecting a
  cell drives the detail panel beneath (that day's three metrics vs baseline);
  the default selection when today is empty is the **window summary**. The log
  compresses to a compact paginated strip below. Uniform compact type, even
  metronomic spacing; color logic concentrates virtually all color into the
  rail while everything else stays neutral. → the most structurally novel
  spread: recovery as a *streak surface* you scrub, not a stack you scroll.

## References

In-system, so "what to take" from each is **structural** — composition,
density, and data legibility, not their palettes or type:

- **Whoop** (today screen + trend calendar) — take the **big banded ring as an
  emotional hero** and the **month-strip of colored day cells**; recovery users
  already speak this language. Drives `readiness-command-hero` and
  `banded-week-rail`.
- **Oura** (readiness briefing) — take the **morning-report framing**:
  contributors with baseline deltas, a verdict sentence, calm confidence that
  a number plus context beats a number alone. Drives `morning-report`.
- **Apple Health** (single-metric detail) — take the **large quiet chart with
  on-plot annotations** carrying its own legend, minimal surrounding chrome.
  Drives `banded-trend-canvas`.
- **Linear** (two-pane density) — take the **side-by-side working-surface
  composition and single-accent restraint** at high information density.
  Drives `split-ledger-control-room`.
- **Garmin Connect** (metric history tables) — take its **respect for dense
  tabular history** — tight rows, aligned numerals, paging that assumes years
  of data — without its visual clutter. Informs `split-ledger-control-room`
  and the ledger treatment everywhere.

## Selection criteria

A note-to-self for the pick, not a rubric the worker optimizes against. When I
compare these I'm trying to decide:

- **Does the page always say something?** Opened at 7am before the webhook, at
  11pm after a skipped night — the surface should read as a report on my
  recovery, never as a form waiting for input. The no-data-today render is the
  first thing I'll compare across variants, not the last.
- **Does the band color make the page feel alive without going stoplight?**
  The status trio is deliberately desaturated; the winner deploys it so a
  green week and a rough week *look different at a squint*, while still
  sitting calmly on the near-black field next to Activities and Bodyweight.
- **Does the log scale?** Compact, paginated, scannable — a year of ingested
  mornings should be a ledger I can consult, and the pager should feel native,
  not bolted on.
- **Is "today vs my baseline" instant?** The averages aren't just hero
  fallback — the real product question is always "is this normal for me?"
  Variants that surface baseline deltas well are doing the job, not decorating.
- **Does it still read as Prog Strength** — native to the near-black /
  periwinkle / Manrope system, accent kept to chrome, bands kept to state —
  rather than a Whoop screenshot re-skin?
- The missing-night gap, the null-HRV day, and the sparse 90d range should all
  survive the treatment without looking broken.

---

> **Lifecycle.** `status:` is editorial — the owner is the dispatch gate. It
> moves `draft` → `exploring` (worker running) → `awaiting_selection` (draft PR
> open, owner deciding) → `selected` / `abandoned`. The worker sets
> `awaiting_selection` on the `dx/recovery-page` branch as it opens the PR; the
> owner sets the terminal value when they close it.
>
> **Handoff.** This DX ends at a *chosen variant*, not merged code. I open the
> draft `[DX — DO NOT MERGE]` PR, compare the variants on the preview deploy,
> pick one, tick its box, set `status: selected` (noting the winning idiom),
> and **close the PR — never merge it.** Then I open a SOW: *"implement
> recovery-page per the `<chosen-idiom>` variant from `dx/recovery-page`,
> production-quality, conforming to the design system"* — carrying the fixed
> decisions above (no dead hero, paginated ledger, settings backlink).
