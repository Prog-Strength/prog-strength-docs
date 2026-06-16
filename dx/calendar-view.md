---
type: dx
status: selected
selected_idiom: warm-organic-coaching
surface: calendar-view
idioms:
  - strava-airy-minimal
  - garmin-data-dense
  - editorial-data-journalism
  - terminal-dense-mono
  - warm-organic-coaching
references:
  - Strava
  - Garmin Connect
  - Linear
  - Whoop
scope: greenfield
variant_count: 5
repos:
  - prog-strength-web
  - prog-strength-docs
---

# DX: Calendar View

**Status**: Selected · **Last updated**: 2026-06-16

> **Selected: `warm-organic-coaching`.** Picked at the selection gate from
> [prog-strength-web#56](https://github.com/Prog-Strength/prog-strength-web/pull/56)
> (the `[DX — DO NOT MERGE]` draft PR). Implementation is carried by the
> downstream SOW [`sows/calendar-view-redesign.md`](../sows/calendar-view-redesign.md),
> which adopts the variant's structure and coaching feel re-toned onto the
> existing dark theme. The DX branch and PR #56 are disposable and can be
> closed/deleted now that the decision lives in the SOW.

## Context

The monthly training calendar (`/calendar`) is where a Prog Strength user sees their training month at a glance — what they lifted, what they ran, what's planned, and how each week is shaping up. The **functionality is solid and not in question here**: the grid renders logged lifts, logged runs, planned sessions, and the merged "planned-then-completed" case correctly; weekly rollups compute; the day digest works. What's unsatisfying is the **look**. The current treatment is a dark, low-contrast grid of small text pills that reads as generic and a little flat — it doesn't feel like a training tool you'd be proud to open every morning, and it doesn't hold a candle to the calendars in the apps this user actually lives in (Garmin Connect, Strava). The calendar is one of the most-visited surfaces in the app, so its visual quality sets the tone for the whole product.

This DX exists to find the *direction* for a calendar that feels purpose-built for training — before committing engineering effort to building one. It explores genuinely different visual languages side by side so the owner can pick the one that fits, then hand that pick to a SOW that implements it for real. No production calendar code changes here.

## The surface

The monthly calendar view. Anatomy of what's on screen today (all of which a variant should account for, though a variant may reorganize or drop chrome in service of its idiom):

- **Header** — month + year title ("June 2026"), a `Today` button, a primary `Plan a workout` action, and prev/next month chevrons.
- **Month summary stat row** — six hero stats for the visible month: **Lift Time** (`7h 20m`), **Run Time** (`1h 55m`), **Run Distance** (`11.7 mi`), **Avg Pace** (`9:49 /mi`), **Longest Run** (`4.2 mi`), **Activities** (`9`).
- **Month grid** — a Mon→Sun, up-to-6-row grid of day cells. Each cell shows the day number and zero or more **event pills**. A **7th "Week" column** on the right of every row summarizes that week: **sessions** count, **lift** time, **run** distance, and **steps** when present.
- **Day digest** — below the grid, the selected day expands to a dated list of its activities (e.g. "Tuesday, June 16, 2026 · 2 activities"), each row showing the session name, a `PLANNED` tag where applicable, scheduled time, and type.

The events come from a discriminated union (`components/calendar/types.ts`):

```ts
type CalendarEvent =
  | { kind: "workout";  startMs; workout }          // a logged lift
  | { kind: "run";      startMs; run }              // a logged run
  | { kind: "planned";  startMs; planned }          // forward-looking scheduled intent
  | { kind: "completed-planned"; startMs; planned; logged }  // planned + the session that fulfilled it, merged into one "done" entry
```

Weekly rollups (`components/calendar/weekly-overview.tsx`):

```ts
type WeeklyStat = { weekStart: Date; activities: number; liftMinutes: number; runMeters: number; steps: number };
```

**Visual states the variants must handle** (these are where lazy designs fall apart, so render them all in the mockup):

- **Completed lift** vs **completed run** vs **planned** must be instantly distinguishable. Today: completed lifts are a filled teal pill, completed runs a filled green pill, planned sessions a dashed blue outline pill with a clock glyph (and a recurring glyph for repeats). A variant needs its *own* legible encoding of these three+ states — not necessarily color-only.
- **Dense day** — a single day can stack 3–4 pills (e.g. June 16 carries four planned sessions). The treatment must not break or truncate into uselessness.
- **Empty days and sparse weeks** — most days are empty; some weeks have `0 activities`. The empty state should feel intentional, not broken.
- **Leading/trailing days** — days from the adjacent months are shown greyed/de-emphasized.
- **Current day** — today is highlighted.
- **Recovery weeks** — sessions named "Recovery Week …" are common; no special data, but the spread of intensity across a month is part of the story a good calendar tells.

**Representative fixture** (use something like this so variants render realistically — a month with a completed first half, a planned second half, a dense day, a recovery week, and empty stretches):

- Week of Jun 1: 5 activities — completed lifts Mon/Wed/Sat/Sun, a completed easy run Tue. Week summary: 5 sessions · lift 4h 48m · run 3.9 mi.
- Week of Jun 8: 3 activities (a "Recovery Week") — Wed lift, Sat lift, Sun run. Summary: 3 sessions · lift 2h 32m · run 3.6 mi · 11,000 steps.
- Week of Jun 15: today is Mon Jun 15 (completed easy run ✓). **Jun 16 is dense**: four planned sessions stacked (W7 D2 Interval Run, W5 D1 Chest, plus two more). Thu/Sat/Sun carry planned sessions. Summary: 1 activity so far · run 4.2 mi · 10,000 steps.
- Weeks of Jun 22 and Jun 29: empty (0 activities).

Activities mix lifts (with a set count, e.g. "29 Sets", and a duration) and runs (with distance + duration + pace, e.g. "Base · 4.05 mi · 0:37:29"). Names look like "W7 D2 - Interval Run", "Recovery Week - Run 1", "Week 4 Workout 1 - Arms".

These are mockups: **static fixtures that look real are preferred** — do not wire the variants to live data services.

## Idioms

Five genuinely distinct visual languages. Each must diverge along **type scale**, **color logic**, and **spacing rhythm** — not just accent color. `scope: greenfield`, so a variant is free to leave the current dark, low-contrast system behind (the user's reference apps are light); the only fixed points are the Prog Strength identity (the "P" wordmark/logo and product name) and that all the states above remain legible.

- **strava-airy-minimal** — Leans on **Strava's** training calendar. Light canvas, *generous* whitespace, hairline (1px) cell borders, and **text-first** activity rows: a session is a single line of restrained sans-serif text with a thin colored category bar at its left edge — no boxed pills. Modest, tight type scale with strong reliance on weight (not size) for hierarchy. Color logic: near-monochrome neutrals with a single category-coded accent per activity (run / lift / other). Add a small **30-day volume sparkline** + a compact summary strip up top, Strava-style. Spacing rhythm: loose and breathable; the month feels calm, not packed.

- **garmin-data-dense** — Leans on **Garmin Connect**. Information-dense and functional: every activity is a compact bordered card carrying an icon + duration + its key metric (sets for lifts, distance/pace for runs), with a **category-colored left border** (lift = one hue, run = another, hike/other = a third). A **persistent "Weekly Totals" rail** runs down the right of each week row (distance / time / a calories-or-steps figure). Tight, utilitarian sans type at a small scale; a Week/Month toggle affordance. Color logic: strictly **category-coded and functional**, never decorative. Spacing rhythm: tight, gridded, maximally informative per square inch — the opposite of the airy idiom.

- **editorial-data-journalism** — Treats the training month as a **data story**, à la a well-set magazine spread or a NYT-graphics piece. A large display heading and **oversized, high-contrast display numerals** for the month's hero stats (think a serif or grotesque display face); the weekly summary reads like a sidebar pull-quote rather than a stat tile. Dramatic type scale — big-to-small contrast carries the hierarchy. Color logic: a mostly-neutral page with **one editorial accent** used deliberately for emphasis. Spacing rhythm: columnar and editorial, with intentional rhythm and generous leading. The grid is still a grid, but it's *art-directed*.

- **terminal-dense-mono** — A **developer-terminal** aesthetic (the **Linear**/command-line end of the spectrum). **Monospace throughout.** High-contrast (dark or light) grid where session state is shown with **status glyphs** — `✓` done, `○` planned, `⟳` recurring — instead of colored pills, and ASCII-flavored dividers/rules. A single accent color on an otherwise neutral field; size is uniform, so hierarchy comes from weight, glyphs, and indentation. Feels keyboard-navigable and precise. Spacing rhythm: dense and uniform, monospaced cadence, zero decoration. Maximum signal, minimum chrome.

- **warm-organic-coaching** — The human, encouraging end. A **warm palette** (amber / clay / sand on a soft off-white or warm-dark), **rounded** pill chips and rounded cell corners, larger touch targets, and a friendly sans with rounded terminals. Emphasizes **consistency and streaks** — a gentle "you trained 5 of 7 days this week" tone — and renders activity types as soft tonal hues rather than saturated category colors. Type scale: comfortable and rounded, mid-contrast. Spacing rhythm: generous and soft, lots of rounded breathing room. Feels like a supportive coach, not a data console — closer to **Whoop's** warmth than Garmin's density.

## References

- **Strava** (Training Calendar) — take its **text-first activity rows** (thin category bar + a single line of text, no heavy pills), its **30-day volume sparkline + summary stat strip**, and its overall airy, low-chrome restraint. Drives `strava-airy-minimal`.
- **Garmin Connect** (Calendar) — take its **per-activity metadata density** (icon + duration + sets/distance on every entry), its **category-colored left borders**, and especially its **"Weekly Totals" right rail**. Drives `garmin-data-dense`.
- **Linear** — take its **crisp typographic hierarchy, restrained single-accent palette, and keyboard-first density**. Informs `terminal-dense-mono` (and the precision of `editorial-data-journalism`).
- **Whoop** — take its **single-accent-on-a-warm/charcoal field and big, confident hero metrics with a coaching tone**. Informs `warm-organic-coaching` and the hero-stat treatment in `editorial-data-journalism`.

## Selection criteria

A note-to-self for the pick, not a rubric for the worker. When I compare these I'm trying to decide:

- **Does it make the training month legible at a glance?** The whole point of the calendar is that I open it and instantly know what I did, what's coming, and whether the week is on track. Density that buries that fails; airiness that hides it also fails.
- **Does the planned-vs-completed-vs-merged distinction stay obvious** without me squinting? That tri-state is the heart of this surface.
- **Does it feel like a *training* product I'm proud of** — closer to Garmin/Strava than to a generic admin calendar — while still reading as Prog Strength?
- **Does the dense day (4 stacked sessions) survive** the treatment, and do the empty weeks still feel intentional?
- I'm explicitly open to **leaving the current dark theme** if a lighter or warmer direction simply works better — I like how Garmin and Strava feel in daylight.

---

> **Lifecycle.** `status:` is editorial — the owner is the dispatch gate. It moves `draft` → `exploring` (worker running) → `awaiting_selection` (draft PR open, owner deciding) → `selected` / `abandoned`.
>
> **Handoff.** This DX ends at a *chosen variant*, not merged code. I open the draft `[DX — DO NOT MERGE]` PR, compare the variants on the preview deploy, pick one, tick its box, set `status: selected` (noting the winning idiom), and **close the PR — never merge it.** Then I open a SOW: *"implement calendar-view per the `<chosen-idiom>` variant from `dx/calendar-view`, production-quality, conforming to the design system."*
