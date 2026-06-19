---
type: dx
status: draft
surface: running-activity-detail
idioms:
  - interval-rep-blocks
  - splits-ledger-spine
  - effort-trace-hero
  - plan-adherence-headline
  - editorial-run-recap
references:
  - TrainingPeaks
  - Runalyze
  - Whoop
  - MacroFactor
  - The Athletic
scope: in-system
variant_count: 5
repos:
  - prog-strength-web
  - prog-strength-docs
---

# DX: Running Activity Detail

**Status**: Draft · **Last updated**: 2026-06-18

> A DX (Design Exploration) is the platform's **divergent** work type. Unlike a
> SOW it does not converge on one correct implementation — it produces N
> differentiated visual variants of a single frontend surface, side by side on
> one comparison route, awaiting a human pick. It **never merges** and ships no
> production code; the chosen direction feeds a downstream SOW that builds it for
> real.

## Context

The **Running Activity Detail** page (`/running/[id]`) is where a runner lands
after tapping a single run — the page that's supposed to answer *"how did that
run actually go?"* It's reached from the Runs list (the `← Runs` back-link),
shows one imported TCX activity, and carries a Delete action. The **functionality
is solid and not in question here**: the activity loads, the eight summary tiles
compute, the trackpoint series plot against distance, the plan-linkage lookup
works, and Unlink/Delete do their jobs.

What's unsatisfying is that the page renders **every run the same way — as an
undifferentiated metrics grid** — when the run on screen is plainly *not* a
generic run. The linkage banner gives it away: *"✓ Completes your planned W7 D2 —
Interval Run."* This was a **structured interval session**, and the page shows
**none of that structure**. It's eight equal stat tiles, two lookalike line
charts (heart rate and pace, distinguishable only by their axis), and an empty
elevation box — the exact same treatment a flat easy run or a tempo run would
get. Three things get lost in that flatness:

- **The session's structure is invisible.** An interval run *is* its reps — the
  surges and the recoveries. Those intervals are sitting right there in the pace
  trace (each work bout is a dip toward fast pace, each recovery a rise), but the
  page draws them as one continuous line, so the shape of the workout — the thing
  that defines it — never reads. The runner can't see how many reps they ran, how
  each one landed, or where they faded.
- **The planned-vs-actual story is a one-line banner.** The most interesting fact
  on the screen — *this run completed a prescribed workout* — is a thin strip with
  an Unlink link. Whether the runner actually *hit the prescription* (the right
  number of reps, at the right pace) is the emotional core of a planned session,
  and the page says nothing about it beyond "✓".
- **A data artifact dominates the charts.** There's a GPS/HR dropout around mile 2
  — heart rate plunges to ~120 and pace spikes to a nonsensical 30:04 /mi for one
  sample. Because the charts auto-scale to raw extremes, that single glitch
  **owns the pace y-axis** and squashes every real data point into the bottom
  eighth of the plot. A surface that treats a dropout as signal can't tell the
  run's story.

This DX exists to find a direction that treats this surface as **what it is — a
single training session with shape, structure, and (often) a prescription it was
meant to fulfill** — before committing engineering effort. **`scope:
in-system`**: the foundation is decided (design-system v0.4, oura-calm), so
variants do **not** re-decide palette, accent, type, or the run discipline hue —
they diverge on **how they compose and *encode* the run**: which element is the
hero, how session structure is surfaced, and how the plan-vs-actual story is told.
No production run-detail code changes here.

> **Design-system baseline.** This DX is in-system against design-system **v0.4**
> (oura-calm, app-wide): soft near-black field, desaturated periwinkle accent
> (**app chrome only — never an activity hue**), Manrope, hairline panels,
> delicate single-stroke charts. Running carries its **discipline hue** —
> green-teal (`--discipline-run-*`: bg `#16302a`, fg `#82d3b8`, dot `#46b893`) via
> `activityColors("running")`. Variants inherit these tokens; they diverge on
> composition and density, not color. The periwinkle accent stays selection /
> app-chrome; **lifecycle status (the plan ✓) is shape + badge, never the accent
> slot.** Status tones (success `#86b39f`, danger `#c79292`) are the only signal
> colors a variant should reach for to encode hit/miss.

## The surface

The `/running/[id]` detail page (a variant should account for all of it, though it
may reorganize, demote, segment, or fold chrome together in service of its idiom):

- **Header** — the `← Runs` back-link, the run **name** (`Afternoon Run`), a
  date·time·duration subtitle (`Thu, Jun 18, 2026 · 12:10 PM · 53:04`), and a
  **Delete** button (top-right, destructive).
- **Plan-linkage banner** — present only when this run fulfilled a planned
  workout: `✓ Completes your planned {plan.name}` with an **Unlink** action. The
  linked plan carries `run_type` (`easy` | `threshold` | `intervals`) and a
  free-text `run_details` prescription — **this is the only structure the plan
  itself provides** (a run plan has no machine-readable reps; see the data note
  below). Absent for an unlinked run.
- **Summary tiles** (today eight equal cards in two rows) — **Distance** (`5.2
  mi`); **Avg Pace** (`10:11 /mi`); **Avg HR** (`169 bpm`); **Calories** (`806`);
  **Duration** (`53:04`); **Best Pace** (`8:21 /mi`); **Max HR** (`191 bpm`);
  **Elev Gain** (`—` when null).
- **Heart-rate chart** — `heart_rate_bpm` per trackpoint vs distance, with an
  `Avg {n} bpm` reference line (`169`).
- **Pace chart** — `pace_sec_per_km` per trackpoint vs distance (rendered in the
  user's unit). **Lower is faster** — the y-axis is inverted-meaning.
- **Elevation chart** — `elevation_meters` vs distance, full-width; degrades to a
  **`No elevation data`** panel when the device logged none (the representative
  case — TCX from a watch without a barometric altimeter).

The data shape (from `prog-strength-web/lib/api.ts`):

```ts
// The activity, via getRunningSession(token, id). trackpoints is present
// ONLY on the per-id detail GET (the list omits it to keep payloads small).
type RunningSession = {
  id: string;
  activity_type: "running" | "walking" | "cycling" | "other";
  name: string | null;                 // "Afternoon Run"
  start_time: string;                  // RFC3339
  distance_meters: number;             // stored metric; converted at render
  duration_seconds: number;
  avg_pace_sec_per_km: number | null;  // null for non-running activities
  best_pace_sec_per_km: number | null;
  avg_heart_rate_bpm: number | null;
  max_heart_rate_bpm: number | null;
  total_calories: number | null;
  elevation_gain_meters: number | null; // null → "—" / "No elevation data"
  trackpoints?: RunningTrackpoint[];   // detail GET only
};

type RunningTrackpoint = {
  sequence: number;
  elapsed_seconds: number;
  distance_meters: number;
  heart_rate_bpm: number | null;       // null per-sample → gap, not zero
  pace_sec_per_km: number | null;
  elevation_meters: number | null;
};

// The fulfilled plan, via getPlannedWorkoutBySession(token, id, "activity").
// null when the run isn't linked to a plan.
type PlannedWorkout = {
  id: string;
  name: string | null;                 // "W7 D2 - Interval Run"
  activity_kind: "lift" | "run";
  status: "planned" | "completed" | "skipped";
  completed_session_id: string | null;
  run_type: "easy" | "threshold" | "intervals" | null;
  run_details: string | null;          // free text, e.g. the interval prescription
  // ...lift agenda + calendar-sync fields omitted (irrelevant to a run)
};
```

**Data note that shapes the structure idioms — read this before designing.** A
planned **run** does **not** carry a structured rep list the way a planned lift
does (lifts have `exercises[].sets[]`; runs have only `run_type` +
free-text `run_details`). So a variant that wants to show "8 × 400m" has two
honest sources, and should use both: (1) the **prescription text** from
`run_details` (display it; do not try to parse it into guaranteed-correct
structured targets), and (2) the **intervals latent in the pace trace** —
segmenting the trackpoints into work/recovery bouts by pace is where the *actual*
structure lives. The interesting variants reconstruct the session's shape **from
the trace itself** and align it against the prescription text as context — they do
not invent plan fields that don't exist.

Times render `mm:ss` (and `h:mm:ss` past an hour); distance/pace honor the user's
display-unit setting (`useDistanceUnit` → `/mi` or `/km`). Distances and paces are
stored metric server-side and converted at render.

**Visual states the variants must handle** (this is where lazy designs fall apart,
so render them all in the mockup):

- **The dropout artifact.** One trackpoint near mile 2 has a garbage spike (pace
  ~`30:04 /mi`, HR plunging to ~`120`). A variant must **not let the glitch own
  the axis** — clamp/winsorize the plotted range, or break the line at the gap —
  so the real signal isn't squashed. This is the single most common failure on the
  current page; a good variant fixes it visibly.
- **No elevation data.** `elevation_gain_meters === null` and every
  `elevation_meters` is null — the representative case. The elevation slot must
  degrade to an inviting empty state, not a flat-line chart or a broken axis. A
  variant may **drop the elevation panel entirely** when there's nothing to show
  rather than reserve dead space.
- **Unlinked run.** No fulfilled plan (`getPlannedWorkoutBySession` → null) — the
  banner and any plan-vs-actual framing must vanish gracefully, and the page must
  still feel complete as a pure activity readout. (This is the common case; most
  runs aren't planned.)
- **Run type that isn't intervals.** A linked `easy` or `threshold` run has no
  meaningful rep structure — the structure-oriented variants must degrade to
  splits/segments without manufacturing fake intervals.
- **Sparse vs dense trace.** A short run with few trackpoints vs a long one with
  hundreds — both the charts and any splits/rep breakdown have to survive both.
- **Missing HR.** `avg_heart_rate_bpm === null` / per-sample HR nulls — the HR
  tiles and chart degrade without leaving holes; a variant leaning on HR zones
  needs a no-HR fallback.
- **"Lower is faster" stays intuitive.** Pace is inverted-meaning; whatever
  encoding a variant picks must never make a *rising* pace line look like
  improvement.

**Representative fixture** (the screenshot's run — use something like it so
variants render realistically: a completed interval session, no elevation, the
mile-2 dropout, an interval pace pattern that's *present in the trace*):

- **Activity**: `Afternoon Run`, Thu Jun 18 2026 · 12:10 PM, **53:04**, **5.2
  mi**. Avg pace `10:11 /mi`, best pace `8:21 /mi`, avg HR `169`, max HR `191`,
  calories `806`, **elevation null** (`No elevation data`).
- **Linked plan**: `W7 D2 - Interval Run`, `status: completed`, `run_type:
  intervals`, `run_details: "Warm-up 1 mi · 8 × 400m @ 5K effort (~6:30/mi) · 200m
  jog recovery · cool-down 1 mi"`.
- **Trace shape (craft this so structure is visible — the current flat line is the
  problem being solved)**: a ~1 mi warm-up around `9:30/mi`, then **8 work/recovery
  oscillations** — surges toward `6:30/mi` (HR climbing into the 180s) alternating
  with recoveries near `9:30/mi` (HR sagging) — then a cool-down. HR drifts upward
  across the session (cardiac drift, `~150 → ~190`). The avg-pace `10:11` is
  dragged slow by recoveries + the dropout, which is the point.
- **The dropout**: one sample at ~mile 2 reads `pace 30:04 /mi`, `HR 120` — a
  device glitch, not a real recovery.

These are mockups: **static fixtures that look real are preferred** — do not wire
the variants to the live activity service. A crafted fixture that *contains* the
interval pattern is essential, because the structure idioms have nothing to show
against a featureless trace.

## Idioms

Five genuinely distinct **compositions** of the *same* oura-calm surface. Because
`scope: in-system`, none re-decides palette, accent, or type — they diverge along
**type scale** (one editorial headline vs uniform tabular figures vs one hero
chart), **color logic** (how the fixed run green-teal hue and the desaturated
success/danger status tones are deployed to encode reps, hit/miss, and effort),
and **spacing rhythm** (airy recap ↔ dense analytical ledger ↔ rhythmic block
stack). Each makes a *different element the hero* and leans on a different
reference.

- **interval-rep-blocks** — The **workout's structure is the page**. The pace
  trace is **segmented into its work/recovery bouts** and the reps are stacked as a
  sequence of blocks (the warm-up, 8 work intervals, their recoveries, the
  cool-down), each block a small card: its avg pace, HR, and duration, sized or
  tinted by effort. The `run_details` prescription rides above as the *target* the
  blocks are read against. Summary tiles compress to a thin header strip. Mid type
  scale; rhythmic, repeating vertical cadence (the blocks *are* the spacing).
  Color logic: green-teal for the work bouts, calm neutral for recoveries, the
  status tones only where a rep clearly over/under-ran the target. → makes "this is
  a structured interval session" **literal**, answering *the structure is
  invisible*. (TrainingPeaks workout-structure / interval laps.)

- **splits-ledger-spine** — Treats the run as **a log of segments** and makes the
  **per-distance splits table the backbone**. The page reads top-to-bottom as a
  ledger — each split (per mile, or per detected interval) a row: distance · time ·
  pace · avg HR · elevation Δ — with the fastest/slowest split marked. Summary
  tiles fold into a compact header; the charts demote to a supporting strip beside
  or below the table. Uniform, fine, tabular type; hierarchy from **alignment, not
  size**. Color logic: restrained, **color-as-state only** — green-teal accenting
  the split column, status tone on the standout splits. Spacing: dense, gridded,
  maximally informative. → for the runner who reads their run as numbers in rows.
  (Runalyze / Garmin Connect splits transparency.)

- **effort-trace-hero** — The **trace is the hero**. One enlarged, full-bleed
  effort chart dominates, with **HR and pace fused** (or precisely stacked and
  synced) so the surges and recoveries read as the *shape* of the session, and the
  **interval boundaries drawn as bands** behind the line. This is the variant that
  most directly **tames the dropout** — a clamped axis and broken line so one
  glitch never squashes the plot. The eight stat tiles compress to axis
  annotations / a caption rail. Big-chart hierarchy — the plot is the page, numbers
  become marginalia. Color logic: a single delicate stroke per series, green-teal
  interval bands, the accent reserved for the hovered point. Spacing: generous,
  the chart claims the page. → makes the **run-as-one-picture** narrative the
  story, the most visual take. (Whoop's single strain/effort trace.)

- **plan-adherence-headline** — Frames the page around **did you hit the
  prescription**. The thin linkage banner is **promoted to the hero**: a verdict
  block — `W7 D2 · Interval Run — completed` with a **prescribed-vs-actual
  readout** (target pace/reps from `run_details` set against what the trace
  delivered, e.g. "7 of 8 reps on pace · avg work `6:34` vs target `6:30`"). The
  raw metrics and chart support beneath. Mid type scale; a balanced two-column
  *target | actual* rhythm. Color logic: **adherence is the encoded thing** —
  desaturated success on hit, danger on miss, green-teal for the discipline; the
  verdict carries the color budget. → drags the **buried planned-vs-actual story**
  to the front and makes it the headline; degrades cleanly to a plain readout for
  unlinked runs. (MacroFactor's predicted-vs-actual framing.)

- **editorial-run-recap** — Tells the run as **a short story**. A real headline +
  a one-or-two-sentence recap leads (`Your W7 D2 intervals — 5.2 mi in 53:04, work
  bouts at 6:30 pace, strong through rep 6 then a fade`), with a few **standout
  figures pulled large** as supporting stats, the effort chart as a single
  supporting figure, and the splits/reps as a quiet appendix below the fold.
  Editorial type scale (genuine headline / body contrast within Manrope); airy,
  reading-document spacing. Color logic: calm and near-monochrome, green-teal and a
  single status tone reserved for the one fact worth emphasizing. → the calm
  "here's how it went" take, answering *the page has no point of view*. (The
  Athletic's race-recap voice / Oura's narrative day cards.)

## References

In-system, so "what to take" from each is **structural** — composition, encoding,
and information design, not their palettes or type:

- **TrainingPeaks** (workout structure) — take the way it renders a **session as
  its prescribed-and-executed segments**, work and recovery as distinct stacked
  blocks. Drives `interval-rep-blocks`.
- **Runalyze** (running analysis) — take its **splits-as-evidence ledger**: a dense
  per-segment table where the run's story is legible in aligned rows. Drives
  `splits-ledger-spine`.
- **Whoop** (single effort/strain trace) — take its **one dominant trace** that
  makes effort over time the whole readout, and its discipline of a clamped,
  readable axis. Drives `effort-trace-hero`.
- **MacroFactor** (predicted vs actual) — take its **expectation-vs-reality
  verdict** as the headline and the signed target-vs-actual framing. Drives
  `plan-adherence-headline`.
- **The Athletic** (race recap) — take its **editorial recap voice**: a headline
  and a sentence that *say what happened* before the numbers. Drives
  `editorial-run-recap`.

## Selection criteria

A note-to-self for the pick, not a rubric the worker optimizes against. When I
compare these I'm trying to decide:

- **Does the session's structure finally read?** For the interval run, can I see
  the reps — how many, how each landed, where I faded — instead of one flat line?
  That's the whole reason this DX exists.
- **Is the planned-vs-actual relationship felt**, when there is one — did I hit the
  prescription — without making an *unlinked* run feel broken or empty?
- **Is the dropout handled?** A variant that still lets the mile-2 glitch own the
  pace axis has failed the most basic test on this page.
- **Does "lower is faster" stay intuitive** so a rising pace line never looks like
  progress?
- **Does it degrade gracefully** to no-elevation, no-HR, unlinked, and
  non-interval runs without looking half-built?
- **Does it still feel like the calm instrument** — native to oura-calm v0.4 and
  the run discipline hue, not a louder re-skin or a generic fitness-app dashboard?

---

> **Lifecycle.** `status:` is editorial — the owner is the dispatch gate. It moves
> `draft` → `exploring` (worker running) → `awaiting_selection` (draft PR open,
> owner deciding) → `selected` / `abandoned`. The worker sets `awaiting_selection`
> on the `dx/running-activity-detail` branch as it opens the PR; the owner sets the
> terminal value when they close it.
>
> **Handoff.** This DX ends at a *chosen variant*, not merged code. I open the
> draft `[DX — DO NOT MERGE]` PR, compare the variants on the preview deploy
> (`/design-explore/running-activity-detail`), pick one, tick its box, set `status:
> selected` (noting the winning idiom), and **close the PR — never merge it.** Then
> I open a SOW: *"implement running-activity-detail per the `<chosen-idiom>`
> variant from `dx/running-activity-detail`, production-quality, conforming to the
> design system."* The mockup code is never promoted as-is.
