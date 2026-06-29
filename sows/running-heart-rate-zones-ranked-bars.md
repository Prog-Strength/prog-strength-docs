---
status: draft
repos:
  - prog-strength-web
  - prog-strength-docs
---

# Running Heart-Rate Zones — Ranked Bars

**Status**: Draft · **Last updated**: 2026-06-29

## Introduction

The heart-rate zones widget on the running detail page (`/running/[id]`) answers
a question runners genuinely care about — *how hard was this run, and where did
the effort live?* — but its current presentation buries the answer. The widget
([`sows/running-heart-rate-zones.md`](running-heart-rate-zones.md) shipped it) is a
single thin (12px) **horizontal stacked time-in-zone bar** above a five-row
legend. Three things break down in that treatment, all of them about *display*,
not data:

- **The zones can't be told apart by color.** The segments are filled from the
  `--zone-1..5` ramp, which is **five steps of the same desaturated periwinkle**
  (`#444a66 → #9aa6d6`, with Z5 == the app accent). It was chosen in-system to
  avoid the cliché green→red heat map, but adjacent zones are only a few percent of
  lightness apart — in a 12px bar they read as one smear, and the 10px legend
  swatches don't disambiguate them either.
- **Duration doesn't read from the visual.** A 100%-stacked bar encodes *share of
  a whole* as segment width, so a zone you spent 43 minutes in and one you spent 24
  seconds in differ only by sliver width — and a 1% sliver is invisible. The only
  place time-in-zone is legible is the legend's number column, divorced from the
  graphic that's meant to dramatize it.
- **A stacked bar is the wrong primitive.** The runner's question is comparative —
  *which zone dominated, and by how much?* — and a 100%-stacked bar is built for
  parts-of-a-whole, not comparison.

Garmin Connect's "Time in Zones" view is the established answer to exactly this:
**one horizontal bar per zone**, each bar's length proportional to time, distinct
per-zone colors, and the time and percent on the right. This SOW rebuilds the
**web widget** in that ranked-bars form — adapted to **Prog Strength's** calm
dark theme, not Garmin's saturated stoplight. It is a display-only change in
service of making the running detail page more useful and more legible; **no
backend, data-model, or API changes** — the `heart_rate_zones` block the backend
already returns is exactly the right data.

This is a refinement of a prior **design** effort, so it must not regress on
style or theme: the widget stays native to oura-calm (design-system v0.4.1) — the
soft near-black field, Manrope, hairline 14px panels — and the one place it
*deliberately extends* the system is the zone color scale, which moves from the
indistinguishable periwinkle ramp to a calm-but-separable five-tone scale (see
Implementation Details).

## Proposed Solution

Replace the stacked-bar-plus-legend body of `HeartRateZones.tsx` with a **ranked
list of five horizontal zone bars**, one per zone, rendered in the existing
hand-rolled token-only idiom (no charting dependency, no raw hex), and **retune
the `--zone-1..5` tokens** to a separable, dark-native scale.

The shape of each row mirrors Garmin's Time-in-Zones, re-toned for oura-calm:

- Rows are ordered **high→low** (Zone 5 / VO2max on top, Zone 1 / Recovery at the
  bottom), so the hardest effort leads and the eye reads the run's ceiling first.
- Each row carries a **label** (zone name + bpm range), a **full-width track** with
  a **fill whose length = `time_pct`** in that zone's color, and the **time and
  percent right-aligned** in a fixed column so the figures align straight down the
  stack.
- The bar *is* the legend — the separate swatch grid goes away, which is what
  reunites the graphic with its number and fixes the "duration doesn't read"
  problem.

The zone color scale becomes the canonical encoding of HR-zone intensity:
**distinct hues per zone, but each desaturated to the same calm register as the
existing status tones** — a cool→warm progression (slate → dusty blue → muted
teal-green → muted amber → dusty coral) that is recognizably the Garmin mental
model (easy is cool, max is warm) without its saturation. This replaces the
`--zone-1..5` values in `globals.css` and is recorded in `design-system.md` as a
deliberate, version-bumped extension. Because those tokens are consumed **only**
by this widget, the change is fully contained.

The calibrating banner, the absent-block (no-HR) behavior, and the card chrome
are preserved.

## Goals and Non-Goals

### Goals

- Rebuild the running detail HR-zones widget as **five ranked horizontal bars**
  (one per zone), bar length proportional to time-in-zone, replacing the stacked
  bar and separate legend.
- Make zones **visually distinguishable at a glance** via a retuned, separable
  `--zone-1..5` scale that stays calm and dark-native (no saturated heat map).
- Make **time-in-zone read from the graphic itself** — each bar's length sits
  beside its own time/percent, no cross-referencing a legend.
- Order rows **high→low** so the dominant/hardest zone leads.
- Preserve every current behavior: the calibrating banner, the no-HR "render
  nothing" case, zero-time zones, the card heading and chrome.
- Record the new zone scale in `design-system.md` (version bump + provenance) and
  update `globals.css`.
- Keep the component **token-only and dependency-free**, conforming to
  design-system v0.4.1.
- Update `HeartRateZones.test.tsx` to cover the new structure and states.

### Non-Goals

- **No backend / API / data-model changes.** The `heart_rate_zones` response block
  is unchanged; this is purely a `prog-strength-web` presentation change.
- **No change to zone math, boundaries, names, or the estimation engine.** Those
  live in `internal/hrzones` and are out of scope.
- **No broader running-detail-page redesign.** The page's other elements (header,
  splits, pace strip, charts) are addressed by their own efforts
  ([`sows/running-activity-detail.md`](running-activity-detail.md)); this SOW
  changes only the HR-zones widget.
- **No new charting dependency.** Hand-rolled in the `PaceStrip` idiom.
- **No mobile change.** `prog-strength-mobile` has its own HR-zones treatment to
  consider later.
- **No interactivity beyond what exists** (no hover tooltips, no zone drill-down,
  no animation) — a calm static readout.
- **No user-customizable zone colors.**

## Implementation Details

This is a frontend SOW; it **conforms to** [`design-system.md`](../design-system.md)
and **deliberately extends** it in exactly one place — the zone color scale,
called out below.

### Zone color scale (design-system extension)

The current `--zone-1..5` ramp is five near-identical periwinkles; it is the root
of the "can't tell zones apart" complaint and **Z5 collides with the app accent**
(`#9aa6d6`), which the design system reserves for selection/app-chrome. The new
scale gives each zone a **distinct hue** while holding the **same desaturated,
dark-native register as the decided status tones** (success `#86b39f`, danger
`#c79292`, warning `#d6b87f`) so it reads as part of oura-calm, not a louder
re-skin. The progression is cool→warm (low intensity reads cool, high reads warm),
which matches the Garmin mental model without the saturation:

| Token | Zone | Proposed value | Hue intent |
| --- | --- | --- | --- |
| `--zone-1` | Recovery | `#6b7280` | calm slate — neutral, lowest |
| `--zone-2` | Aerobic | `#6f93c2` | dusty blue |
| `--zone-3` | Tempo | `#6fb39a` | muted teal-green (leans run discipline) |
| `--zone-4` | Threshold | `#d0a86e` | muted amber (status-warning register) |
| `--zone-5` | VO2max | `#cc8077` | dusty coral (status-danger register) |

Notes for implementation:

- These are a **proposed** scale; final values may be nudged ±a few points during
  implementation so each tone sits legibly on `--surface` (`#15171b`) and
  `--surface-2` (the track fill behind it) and so adjacent tones (esp. Z4/Z5) stay
  distinct at small fill widths. The **constraints are fixed**: distinct hues,
  status-tone-level saturation (calm), cool→warm order, and **Z5 must not be the
  accent**.
- The fills do not encode value judgment — coral on VO2max is "hottest zone," not
  "bad." This is consistent with the design-system rule that *activity/measurement
  color is distinct from status*; we are reusing the status tones' **saturation
  register**, not their semantics, and giving zones their own dedicated tokens (we
  do **not** alias `--success`/`--danger`/`--warning`).
- `--zone-1..5` and the `--color-zone-1..5` Tailwind aliases in `globals.css` keep
  their names and structure; **only the hex values change.** The comment block
  above them is updated to describe the new separable scale and to drop the
  "Z5 == --accent" note.
- **Blast radius is contained:** `--zone-*` is referenced only by
  `globals.css` (definition) and `HeartRateZones.tsx` (consumption) — verified —
  so retuning the values affects this widget alone.

`design-system.md` is updated: a new **Changelog** entry (v0.4.2) and a short note
under **Color** recording that the HR-zone scale is now a separable calm five-tone
scale (cool→warm, status-tone register, distinct from the accent), with provenance
pointing at this SOW. This supersedes the v0.x rationale that justified the
periwinkle ramp.

### Widget structure (`HeartRateZones.tsx`)

The component keeps its prop contract (`{ zones: HeartRateZones | null | undefined }`),
its early return (`null` when `!zones || zones.zones.length === 0`), the card
chrome (`rounded-[var(--radius-card)] border border-[var(--border)]
bg-[var(--surface)] px-4 py-3`), and the uppercase `HEART RATE ZONES` heading. The
body changes from `stacked bar + legend` to a **ranked bar list**.

Render order: iterate the zones **descending by `zone`** (5 → 1) so VO2max is on
top. Each zone is one row:

- **Label line** — the zone **name** in `--foreground` medium weight, followed by
  the **bpm range** (`{min_bpm}–{max_bpm} bpm`) in `--muted`. (Optionally prefix
  the zone number, e.g. `Z5`, in `--faint` for a Garmin-like `Zone 5` cue — minor,
  implementer's discretion.)
- **Bar line** — a full-width **track** (`bg-[var(--surface-2)]`, `rounded-full`,
  a real height — ~`h-2.5`, taller than today's `h-3` shared bar since each zone
  now gets its own) containing a **fill** of width `${time_pct * 100}%` and
  `backgroundColor: var(--zone-${zone})`, applied via inline style (matching the
  existing dynamic-fill approach). The fill is `aria-hidden`. A fill of 0% renders
  an empty track (the row still shows, so the fixed five-zone axis is preserved).
- **Figures** — on the same line as (or trailing) the bar, the **time**
  (`formatDuration(z.time_seconds)`) and **percent** (`formatPercent(z.time_pct)`)
  right-aligned with `tabular-nums` in a fixed-width column so they align down the
  stack. Time in `--muted`, percent in `--foreground` (emphasis on the share),
  mirroring today's legend weights.

Layout target (rail-width safe): label line on top, then a row with the track
taking remaining width and the time/percent pinned right — e.g. the track in a
`flex-1` container with a fixed-width figures column. The widget lives in the
narrow right rail under `PaceStrip`, so the bars must stay legible at constrained
width (no horizontal overflow; the track flexes, the figures column is fixed).

The values shown are **real text** (name, bpm, time, percent), so the widget is
self-describing for screen readers; only the decorative fill is `aria-hidden`.
This is an improvement over today, where the visual bar was the only encoding of
share and was fully `aria-hidden`.

### States to preserve

- **Calibrating** — when `reference_confidence !== "calibrated"`, keep the existing
  banner verbatim (*"Calibrating — zones will sharpen as Prog Strength learns your
  heart rate."*) in the accent-soft strip below the bars.
- **No HR** — when the `heart_rate_zones` block is absent (the common case for runs
  without per-point HR), the component renders nothing (`null`). Unchanged.
- **Zero-time zone** — a zone with `time_seconds: 0` / `time_pct: 0` renders its
  row with an empty track and `0:00 · 0%`; it is **not** dropped (keeps the fixed
  five-zone axis and lets the runner see a zone went untouched).
- **Extreme skew** — the representative case (one zone ~71%, two zones ~1%) reads
  correctly: the dominant bar is plainly longest, and the 1% zones still show a
  short-but-present fill and their own legible `0:24 · 1%` figures (the failure
  mode the stacked bar had — a vanishing sliver — is gone because each zone owns a
  full-width track).
- **Balanced run** — several zones of similar length read as similar bar lengths;
  no fake "winner" is manufactured.

### Tests (`HeartRateZones.test.tsx`)

Extend the existing test file (keep the calibrated/calibrating/null fixtures):

- Renders one bar row per zone (five rows) with each zone **name**, **bpm range**,
  **time**, and **percent** present.
- Rows render in **descending zone order** (VO2max first, Recovery last) — assert
  DOM order.
- Each fill's width corresponds to `time_pct` and uses the zone's `--zone-N` fill
  (assert the inline `width` / `backgroundColor` style on the fill node).
- **Zero-time zone**: a fixture with a `time_seconds: 0` zone still renders that
  zone's row with `0:00` / `0%`.
- **Extreme-skew fixture** (the screenshot numbers: Recovery `0:37`/1%, Aerobic
  `0:24`/1%, Tempo `3:01`/5%, Threshold `13:34`/22%, VO2max `43:34`/71%) renders
  all five rows with their figures.
- Calibrating banner shows when not calibrated; absent when calibrated.
- Renders nothing when `zones` is null / empty.

### Conformance & provenance

- The widget references design-system tokens only (`--surface`, `--surface-2`,
  `--border`, `--foreground`, `--muted`, `--faint`, `--radius-card`,
  `--accent-soft`/`--accent-line`, and the retuned `--zone-1..5`). No raw hex in
  the component.
- `design-system.md` Changelog gains a **v0.4.2** entry recording the separable
  zone scale; the **Provenance** list links this SOW.

## Open Questions

1. **Row ordering — high→low vs low→high.** Garmin (and the lean here) is high→low
   so the hardest zone leads. The prior widget legend listed low→high (Recovery
   first). *Tentative lean: high→low*, matching the reference the owner chose;
   trivially flippable if it reads worse for easy runs.
2. **Show the zone number prefix?** Garmin shows both `Zone 5` and a descriptive
   name (`Maximum`); our names *are* descriptive (`VO2max`). Options: name only, or
   a faint `Z5` prefix for scannability. *Tentative lean: name + bpm range only,
   with a faint number prefix if it reads cleanly at rail width.*
3. **Exact zone hexes.** The table proposes a scale; final values get nudged
   against the real surface during implementation under the fixed constraints
   (distinct hues, calm/status-tone saturation, cool→warm, Z5 ≠ accent). *Lean:
   ship the proposed values unless on-device contrast says otherwise.*
4. **Time vs percent emphasis.** Both are shown; which gets `--foreground` weight?
   *Tentative lean: percent emphasized (the share is the headline), time in
   `--muted`* — matching the current legend's weights.
