---
type: dx
status: draft
surface: running-max-effort-estimate
idioms:
  - single-number-hero
  - projection-chart-forward
  - estimate-vs-actual-gap
  - confidence-ledger
  - trend-narrative
references:
  - Whoop
  - FiveThirtyEight
  - MacroFactor
  - Runalyze
  - Oura
scope: in-system
variant_count: 5
repos:
  - prog-strength-web
  - prog-strength-docs
---

# DX: Running Max-Effort Estimate

**Status**: Draft · **Last updated**: 2026-06-17

> A DX (Design Exploration) is the platform's **divergent** work type. Unlike a
> SOW it does not converge on one correct implementation — it produces N
> differentiated visual variants of a single frontend surface, side by side on
> one comparison route, awaiting a human pick. It **never merges** and ships no
> production code; the chosen direction feeds a downstream SOW that builds it for
> real.

## Context

The **Max-Effort Estimate** page (`/progress/running/[distanceKey]`) answers a
question a runner actually cares about: *"if I went all-out at this distance right
now, what could I run?"* The model fits the user's recent best-effort samples
across distances into a projected time per standard distance (1 mile … marathon),
with a confidence interval, a comparison to their real current best, a trend, and
the attempts that fed the fit. The **functionality is solid and not in question
here** — the estimate computes, the chart plots the projection over time with its
confidence band, and the attempts table lists the efforts.

What's unsatisfying is that the page presents a **statistical estimate as if it
were a plain dashboard metric**. It's four equal-weight stat tiles, a line chart,
and a table — the same treatment as steps or volume — when the thing on screen is
fundamentally *a prediction with uncertainty and a direction*. Three things get
lost in that flatness:

- **The uncertainty is invisible.** `53:42` reads as a hard fact; the confidence
  band exists on the chart but the headline number gives no sense of its range.
- **The estimate-vs-reality story is buried.** The most interesting fact on the
  screenshot — the runner's *actual best* (41:35) is **faster** than their current
  *estimate* (53:42), because recent efforts were slow long-runs that dragged the
  fit — is reduced to a small "+12:07 vs estimate" caption. That tension is the
  whole point of the page and it whispers.
- **The trend has no weight.** "↑ +11:05 slower" is a small red caption, not a
  narrative; whether you're getting faster or slower is the emotional core of a
  progress surface.

This DX exists to find a direction that treats this surface as **what it is — a
prediction, with a range, a comparison to reality, and a trajectory** — before
committing engineering effort. **`scope: in-system`**: the foundation is decided,
so variants do **not** re-decide palette, accent, or type — they diverge on
**how they compose and *encode* the estimate**: which element is the hero, how
uncertainty is shown, and how the estimate-vs-actual-vs-trend story is told. No
production estimate code changes here.

> **Design-system baseline & dispatch sequencing.** This DX is in-system against
> the **oura-calm-minimal** language that `sows/activities-page-redesign.md`
> adopts app-wide (`design-system.md` v0.3: soft near-black field, desaturated
> periwinkle/sage accents, Manrope, hairline panels, delicate single-stroke
> charts). **Dispatch this DX only after that re-tone PR
> (Prog-Strength/prog-strength-docs#113) merges** — otherwise the worker renders
> the variants in the old slate/violet/Nunito palette. The variants inherit the
> new tokens; they diverge on composition, not color.

## The surface

The `/progress/running/[distanceKey]` detail page (a variant should account for all
of it, though it may reorganize, demote, or fold chrome together in service of its
idiom):

- **Header** — a back-link to Progress and the title `Max-Effort Estimate
  · {distance}`.
- **Distance switcher** — a pill row across the standard distances (`1 Mile · 2
  Mile · 5K · 10K · Half Marathon · Marathon`); the active distance is filled. It
  navigates between distance keys (each its own estimate).
- **Stat tiles** (today four equal cards) — **Estimated max-effort time**
  (`53:42`); **Current best** (`41:35`, with the gap caption `+12:07 vs
  estimate`); **Confidence** (`High`, with the basis `75 efforts across 4
  distances`); **Trend** (`↑ +11:05 slower`, colored by direction — slower is the
  concerning/red case).
- **Estimate-over-time chart** — the projection (`estimate_history`) as a line
  with a **confidence band** (lower/upper bounds), the real **attempts** overlaid
  as dots, a `"lower is faster"` axis note, and a time y-axis (`0:00 … 56:41`).
- **Attempts table** — the efforts that fed the fit: `Date · Time · Pace ·
  Source`, each date linking to its activity; source is how the window was found
  (`Long-run window`, `Race-like`).

The data shape (`RunningMaxEffortDetail`, from `prog-strength-web/lib/api.ts`, via
`getRunningMaxEffort(token, distanceKey)`):

```ts
type RunningMaxEffortDetail = {
  distance_key: string; distance_label: string; distance_meters: number;
  estimator_version: string;
  estimate: {                       // null when basis === "insufficient_data"
    seconds: number;
    lower_seconds: number; upper_seconds: number; // the confidence interval
    basis: string;                  // e.g. "multi_distance_fit"
    confidence: string;             // "low" | "medium" | "high"
    n_points: number;               // best-effort samples that fed the fit (75)
    n_distances: number;            // distinct distances that fed it (4)
  } | null;
  actual_best: {                    // null when the user has never covered it
    seconds: number; activity_id: string; achieved_at: string;
  } | null;
  estimate_history: {               // ascending by as_of — the band over time
    as_of: string; seconds: number; lower_seconds: number; upper_seconds: number;
  }[];
  attempts: {                       // real efforts overlaid + listed
    activity_id: string; achieved_at: string;
    duration_seconds: number; pace_sec_per_km: number;
    source: string;                 // "race_like" | "long_run" | …
  }[];
  stats: {
    estimated_max_effort_seconds: number | null;
    current_best_seconds: number | null;
    gap_seconds: number | null;     // best − estimate (sign carries the story)
    confidence: string; data_summary: string;
  };
};
```

Times render `mm:ss` (and `h:mm:ss` for the marathon); pace is `/mi` or `/km` per
the user's display-unit setting (`useDistanceUnit`). The estimate is **null** when
there isn't enough data — the page shows an empty hint instead of a fabricated
number.

**Visual states the variants must handle** (this is where lazy designs fall apart,
so render them all in the mockup):

- **Estimate slower than actual best.** The representative case: estimate `53:42`
  but actual best `41:35` (best is *faster*). The gap sign is the story — a variant
  must make "your proof beats your current projection" read honestly, not as a
  contradiction or an error.
- **Uncertainty width.** A wide confidence band (low confidence, few samples) vs a
  tight one (high confidence) must look *different* — the headline can't read as
  equally precise in both. Don't present a point estimate as a hard fact.
- **Trend direction.** Slower (`↑ +11:05`, concerning) vs faster (improving) vs
  flat — each must read at a glance, and *slower* must inform without demoralizing.
- **Insufficient data.** `estimate === null` (`basis: "insufficient_data"`) — the
  page degrades to an inviting "log a few efforts" hint, not a row of blanks or
  `0:00`.
- **Never-covered distance.** `actual_best === null` — the Current-best tile/compare
  has nothing to show; the surface should stay intact (estimate without a real
  comparison).
- **Sparse vs dense attempts.** As few as the 3 shown, or many — the chart overlay
  and the table both have to survive both.
- **Confidence levels.** `low` / `medium` / `high` (with the `n_points` /
  `n_distances` basis) should be legible as a trust signal, not just a word.
- **"Lower is faster" stays intuitive.** The y-axis is inverted-meaning (a lower
  time is better); whatever encoding a variant picks must not make a *rising* line
  look like progress.

**Representative fixture** (the screenshot's 10K — use something like it so
variants render realistically: estimate slower than best, high confidence, a
slowing trend, mixed sources):

- **Distance**: 10K. **Estimate**: `53:42` (band ≈ `53:10 – 54:18`), basis
  `multi_distance_fit`, **confidence High** (`75 efforts across 4 distances`).
- **Actual best**: `41:35`, achieved Apr 11, 2026 → **gap `+12:07` (best faster
  than estimate)**.
- **Trend**: `↑ +11:05 slower` (the estimate has risen — gotten slower — over the
  window).
- **Estimate history**: a band rising from ~`45:00` (Apr 11) to ~`53:42` (May 22),
  lower-is-faster.
- **Attempts**: Apr 11, 2026 — `41:35` · `6:41 /mi` · *Long-run window*; May 8,
  2026 — `53:03` · `8:32 /mi` · *Long-run window*; May 22, 2026 — `56:41` · `9:07
  /mi` · *Race-like*.

These are mockups: **static fixtures that look real are preferred** — do not wire
the variants to the live estimate service.

## Idioms

Five genuinely distinct **compositions** of the *same* oura-calm surface. Because
`scope: in-system`, none re-decides palette, accent, or type — they diverge along
**type scale** (how hard they lean on one hero numeral vs uniform figures),
**color logic** (how the fixed periwinkle/sage accents and the desaturated
positive/negative tones are deployed to encode uncertainty, the gap, and the
trend), and **spacing rhythm** (reductive and airy ↔ dense analytical ledger).
Each makes a *different element the hero* and leans on a different reference.

- **single-number-hero** — The **estimate is the page**. One oversized time
  dominates, with the **confidence interval rendered as the subtitle** (`53:10 –
  54:18`, not just `53:42`) so uncertainty is part of the headline, not hidden in
  a chart. The trend sits as a small directional tag beside it; the chart and
  attempts demote to a calm strip below. Dramatic type-scale contrast — hero
  numeral huge, everything else fine. Color logic: near-monochrome, the accent
  reserved for the one number and a single trend tint. Spacing: generous, the page
  opens with "here's your number, and how sure we are." → answers *uncertainty is
  invisible* by putting the range in the headline. (Whoop's single big metric.)

- **projection-chart-forward** — The **estimate-over-time chart is the hero**,
  enlarged and full-bleed, with the confidence band drawn as a prominent **cone of
  uncertainty** that visibly widens with doubt, and the real attempts as dots
  scattered against the projection. The stat tiles compress to a thin caption strip
  above or below. Chart-scale hierarchy — the plot dominates, numbers become axis
  and annotation. Color logic: the accent line + a soft band fill, attempts in the
  secondary (sage) hue. Spacing: the chart claims the page. → makes **trajectory and
  uncertainty** the story; the most visual take. (FiveThirtyEight forecast cones.)

- **estimate-vs-actual-gap** — Frames the page around the **gap between estimate
  and reality**. A side-by-side **"potential vs proof"** hero: the projected
  `53:42` and the actual best `41:35` set against each other with the **`+12:07`
  delta as the centerpiece**, the sign and color carrying whether your proof is
  ahead of or behind your projection. Mid type scale, a balanced two-column rhythm.
  Color logic: the gap is *the* encoded thing — a desaturated positive/negative
  tone on the delta. The chart and attempts support beneath. → drags the **buried
  estimate-vs-reality tension** to the front and makes it the headline.
  (MacroFactor's predicted-vs-actual.)

- **confidence-ledger** — Treats the number as a **claim that must show its work**.
  A dense, analytical, report-like layout that foregrounds **how the estimate was
  derived**: the confidence level, `n_points` / `n_distances` basis, the
  `basis`/`estimator_version`, and the **attempts table promoted to primary
  evidence** with its sources broken out (race-like efforts weighted differently
  from long-run windows). Uniform, fine, tabular type; hierarchy from alignment, not
  size. Color logic: restrained, **color-as-state only** (confidence level, source
  type, gap sign). Spacing: dense, gridded, maximally informative. → for the runner
  who only trusts the number once they can **see its basis**. (Runalyze's running
  prognosis transparency.)

- **trend-narrative** — Organizes the surface as **the story of how the estimate
  has moved**. The **trend is the hero** (`↑ +11:05 slower` becomes a stated
  headline, not a caption), the `estimate_history` reads as a progression/regression
  narrative, and each attempt is an **annotated milestone** on that arc (the slow
  long-runs that dragged it, the race-like effort). Editorial-leaning scale within
  the calm system; a sequential, timeline rhythm. Color logic: trend **direction**
  is the signal (improving vs declining), applied calmly. → answers *the trend has
  no weight* and makes the surface motivating: **are you getting faster?** (Oura's
  calm trend cards.)

## References

In-system, so "what to take" from each is **structural** — composition, encoding,
and information design, not their palettes or type:

- **Whoop** (single hero metric) — take its **one big confident number** and the
  first-glance readout, here extended so the **confidence range rides in the
  headline**. Drives `single-number-hero`.
- **FiveThirtyEight** (election forecast) — take its **cone of uncertainty**: a band
  that visibly widens with doubt, with real observations dotted against the
  projection. Drives `projection-chart-forward`.
- **MacroFactor** (predicted vs actual) — take its **expectation-vs-reality
  comparison** as the headline and the signed-delta framing. Drives
  `estimate-vs-actual-gap`.
- **Runalyze** (running prognosis) — take its **derivation transparency**: showing
  the samples, distances, and confidence behind a predicted time so the number is
  trustable. Drives `confidence-ledger`.
- **Oura** (trend cards) — take its **calm "here's where this is heading" narrative**
  and milestone framing of a moving metric. Drives `trend-narrative`.

## Selection criteria

A note-to-self for the pick, not a rubric the worker optimizes against. When I
compare these I'm trying to decide:

- **Does the uncertainty read honestly?** The headline should never feel like false
  precision — the range/confidence has to be felt, not buried in the chart.
- **Is the estimate-vs-actual-best relationship clear**, *especially* in the weird
  case where my real best is faster than the estimate? That tension is the most
  interesting thing on the page and it should read as insight, not as a bug.
- **Does the trend land** — slower as genuinely concerning, faster as earned —
  without being demoralizing on a bad block?
- **Does "lower is faster" stay intuitive** so a rising line never looks like
  progress?
- **Does it degrade gracefully** to the insufficient-data and never-covered states
  without looking broken?
- **Does it still feel like the calm instrument** — native to the oura-calm system,
  not a louder re-skin?

---

> **Lifecycle.** `status:` is editorial — the owner is the dispatch gate. It moves
> `draft` → `exploring` (worker running) → `awaiting_selection` (draft PR open,
> owner deciding) → `selected` / `abandoned`. The worker sets `awaiting_selection`
> on the `dx/running-max-effort-estimate` branch as it opens the PR; the owner sets
> the terminal value when they close it.
>
> **Handoff.** This DX ends at a *chosen variant*, not merged code. I open the draft
> `[DX — DO NOT MERGE]` PR, compare the variants on the preview deploy, pick one,
> tick its box, set `status: selected` (noting the winning idiom), and **close the
> PR — never merge it.** Then I open a SOW: *"implement running-max-effort-estimate
> per the `<chosen-idiom>` variant from `dx/running-max-effort-estimate`,
> production-quality, conforming to the design system."* Dispatch this DX **after**
> the oura-calm re-tone (docs#113) merges so the variants render in the new system.
