---
type: dx
status: draft
surface: timeline
idioms:
  - strava-social-dashboard
  - garmin-feed-leaderboard
  - editorial-milestone-journal
  - terminal-activity-ledger
  - soft-coaching-community
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

# DX: Timeline

**Status**: Draft · **Last updated**: 2026-06-16

## Context

The Timeline (`/timeline`) is Prog Strength's social surface — the reverse-chronological feed where training shows up as it happens: runs, lifts, personal records, best efforts, each with reactions and comments. The plumbing is social-capable already (every post has an author and a `/u/<username>` profile, reactions, comment threads), but the feed today reads as a stack of near-identical dark cards with small muted stat chips — it doesn't feel alive, social, or motivating, and it's a long way from the feeds people actually scroll. Garmin Connect and Strava — where this user lives — make a training feed feel like a community: bold activity cards, route maps, kudos, a weekly leaderboard, a sense of other people training alongside you.

This DX explores the timeline as a **social / following feed** — posts from the people you follow *and* your own, with the community and competitive elements (kudos, a weekly leaderboard, discovery) that make a fitness feed worth opening daily. The current account simply has no one followed yet, so a first-class concern here is that the feed feels good across the whole range: a rich multi-person feed for someone with an active network, *and* the **no-followers onboarding state** that nudges discovery without feeling empty. It's a visual/layout exploration: the post types, reactions, and profiles that exist stay as they are; what's explored is how a social training feed should look and what social scaffolding (rails, leaderboard, discovery) frames it.

## The surface

The timeline route (`app/(app)/timeline/page.tsx`) and its feed components (`app/(app)/timeline/_components/*`), inside the dark app shell. A variant should render a **populated following feed** (multiple authors), the rich card types, and the social scaffolding it chooses. The parts:

- **The feed** — a list of post cards, each (`TimelinePostCard`):
  - **Author** — avatar + display name (links to `/u/<username>`). In a social feed these are different people, not just you.
  - **Source-type badge** — one of `run` 🏃, `workout` 🏋️, `pr` 🏆 (Personal Record), `best_effort` ⚡ — plus a relative/absolute timestamp.
  - **Title + body** — e.g. "W7 D1 - Easy Run", "barbell-military-press PR", "Recovery Week Lift 2 - Shoulders & Arms", with an optional free-text `notes` paragraph the athlete wrote.
  - **Stat chips / projection** — type-specific: runs show distance + time (+ pace); PRs show the lift (`155 lb × 5`); workouts show `6 exercises · 26 sets · 17,875 lb` and carry an exercise breakdown that today feeds a small **radar** (`WorkoutTimelineSummary`).
  - **Route map (runs)** — runs carry `trackpoints` (sampled GPS from the TCX import), so a run card *can* show a **route map** the way Strava/Garmin do. Treat it as an element a variant may include (the feed projection would need to expose trackpoints; for mockups a static route is fine).
  - **Reactions** (`ReactionBar`) — four types in order: 👍 like, 💪 strong, 🔥 fire, 🎉 celebrate — each with a live count and an active state for the viewer's own reaction (kudos, essentially).
  - **Comments** (`CommentThread`) — a per-post count badge that expands a flat, oldest-first thread.
- **Social scaffolding** (idiom-dependent, drawn from the references) — a feed can be flanked by rails: a **weekly leaderboard** (e.g. steps or volume ranked among the people you follow, your row highlighted — Garmin), a **your-week summary / streak** panel (Strava's left rail), and **discovery** ("find people to follow" / suggested athletes). These are part of the social direction being explored; treat them as design candidates, not a fixed spec.
- **Empty / no-followers state** — the current account follows no one. The feed must still feel intentional and alive here: surface your own training, and make **discovery the hero** (a clear, inviting "find people to follow" path) rather than showing a barren list. This state is as important to get right as the populated one.
- **Header** — "Timeline" + the "react and comment on your milestones" subtitle, and a Refresh affordance.

**Data shape** (for realistic fixtures — the feed projection):

```ts
type TimelineSourceType = "workout" | "run" | "pr" | "best_effort";
type ReactionType = "like" | "strong" | "fire" | "celebrate";
type TimelinePost = {
  author: { display_name: string; avatar_url?: string; username?: string };
  source_type: TimelineSourceType;
  occurred_at: string;           // ISO; rendered relative or absolute
  title: string;
  notes?: string;                // optional athlete-written body
  // type-specific projection: run → {distance, duration, pace}; pr → {lift, weight, reps};
  // workout → {exercises, sets, volume, breakdown for the radar}
  reactions: { summary: Record<ReactionType, number>; mine: ReactionType[] };
  comment_count: number;
};
// runs additionally have trackpoints (RunningTrackpoint[]) at the activity level → optional route map.
```

**Representative fixture** — a **multi-person** following feed (the screenshots show only the solo account; invent a plausible network so variants render socially). Mix:

- A followed athlete's **run** with a route map, stat row (distance · time · pace), and a few kudos + a comment.
- Your own **PR** ("barbell-military-press PR · 155 lb × 5") — celebratory, fresh, no reactions yet.
- A followed athlete's **workout** ("Recovery Week Lift 2 - Shoulders & Arms · 6 exercises · 26 sets · 17,875 lb") with a notes paragraph and the radar breakdown.
- Your own **run** ("W7 D1 - Easy Run · 4.2 mi · 42:51") with mixed reactions.
- A **best_effort** ⚡ milestone from someone you follow.
- A **weekly leaderboard** fixture (steps or volume, ~8 people, your row highlighted mid-pack — like the Garmin screenshot).

Also render the **no-followers empty state** as its own panel. These are mockups — **static fixtures that look real are preferred**; do not wire to the live feed.

**States the variants must handle** (where lazy social feeds fall apart): a populated multi-author feed; **your own** post vs. **someone else's**; a run **with a route map**; a workout with **stat chips + radar**; a **PR / best-effort** milestone (these should feel celebratory, not like every other row); reactions with counts **and** the viewer's active kudos; comments collapsed vs. expanded; the **no-followers onboarding** state; and the **leaderboard rail** with the viewer highlighted.

## Idioms

Five genuinely distinct directions. All stay **dark** (`scope: greenfield` means exploring social structure and layout beyond today's single-column feed, not changing the base theme). Each must diverge along **type scale**, **color logic**, and **spacing rhythm** — not just accent color. Fixed points: the Prog Strength identity, and that the rich card data (stats, route maps, the radar, reactions, the celebratory PR/milestone treatment) stays legible.

- **strava-social-dashboard** — Strava's three-column dashboard. A left rail (your profile + **streak** + a this-week mini-chart), a center **following feed** of bold, confident activity cards (author, title, a horizontal stat row with labels under big values, a prominent **route map**, kudos + comment affordances), and a right rail (discovery / suggested athletes / challenges). Type: confident athletic sans, strong title hierarchy. Color: photographic, the map and one energetic accent carry it. Spacing: generous, card-forward, social-first.

- **garmin-feed-leaderboard** — Garmin's two-column, competition-forward layout. A center news feed of **metric-dense** activity cards (work-time / sets / calories / distance as labeled stat blocks) beside a prominent right **weekly leaderboard** rail (ranked among the people you follow, your row highlighted, a metric switcher). Type: small utilitarian sans + tabular numerals. Color: functional, category-coded, restrained. Spacing: tight, data-forward — the feed as a competitive dashboard.

- **editorial-milestone-journal** — the feed as a curated training story. Dramatic type scale (a display face for titles and **oversized numerals** for the stat that matters); PRs, best efforts, and others' achievements get a **celebratory editorial moment** rather than a uniform row; single-column, art-directed, generous leading, minimal rails. Color: mostly-neutral with **one editorial accent** for milestones. Spacing: columnar, magazine-like — the opposite of the dense leaderboard idiom.

- **terminal-activity-ledger** — the following feed as a precise activity stream. **Monospace metadata** (author handle, type glyph, timestamp, stats), tight log-like rows, kudos and type shown by **glyphs** not chrome, a compact ranked sidebar; keyboard-navigable, high density, single sharp accent on graphite. Type: uniform mono, hierarchy by weight/indentation. Spacing: dense, ledgered — maximum activity per screen.

- **soft-coaching-community** — a warm, encouraging community feed. Soft **rounded cards**, friendly kudos, **streak / consistency** emphasis, a warm "your people this week" leaderboard, gentle coaching microcopy, and a brand accent. Type: rounded, comfortable, mid-contrast. Color: warm tonal hues on dark + one clay-ish accent. Spacing: generous and rounded — motivating and approachable, the community as a supportive crew rather than a competition.

## References

- **Strava** (Dashboard / following feed) — take its **three-column social dashboard**, the bold activity card with a labeled stat row + route map, the left-rail **streak/this-week**, and kudos. Drives `strava-social-dashboard`.
- **Garmin Connect** (News Feed) — take its **metric-dense activity cards** and especially the **Weekly Leaderboard** rail (ranked among your people, your row highlighted, metric switcher). Drives `garmin-feed-leaderboard`.
- **Linear** — take its **crisp dense typography, single-accent restraint, and structured panes**. Drives `terminal-activity-ledger` and the precision of `editorial-milestone-journal`.
- **Whoop** — take its **warm single-accent treatment, big confident metrics, and coaching tone**. Drives `soft-coaching-community`.

## Selection criteria

A note-to-self for the pick, not a rubric the worker optimizes against. When I compare these I'm weighing:

- **Does it make the feed feel alive and social** — like other people are training alongside me — rather than a quiet list of my own cards? That's the reason for this exploration.
- **Do the activity cards have real craft** — Strava/Garmin-level — with the run map, the stat presentation, and the workout radar all legible and good-looking?
- **Does the no-followers state feel intentional and inviting?** Right now I follow no one; the feed has to be great empty, nudging discovery, not a dead end. This is the state I'll actually see first.
- **Do milestones (PRs, best efforts) feel celebratory**, standing out from routine posts?
- **Does the competitive layer (leaderboard, kudos) motivate without turning the feed into a noisy scoreboard?**
- **Does it still read as Prog Strength**, not a reskin of Strava or Garmin?

---

> **Lifecycle.** `status:` is editorial — the owner is the dispatch gate. It moves `draft` → `exploring` (worker running) → `awaiting_selection` (draft PR open, owner deciding) → `selected` / `abandoned`.
>
> **Handoff.** This DX ends at a *chosen variant*, not merged code. I open the draft `[DX — DO NOT MERGE]` PR, compare the variants on the preview deploy, pick one, tick its box, set `status: selected` (noting the winning idiom), and **close the PR — never merge it.** Then I open a SOW: *"implement timeline per the `<chosen-idiom>` variant from `dx/timeline`, production-quality, conforming to the design system."*
