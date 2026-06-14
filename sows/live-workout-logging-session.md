---
status: shipped
repos:
  - prog-strength-mobile
  - prog-strength-web
  - prog-strength-docs
---

# Live Workout Logging Session

**Status**: Shipped · **Last updated**: 2026-06-14

## Introduction

Today a lifter records a workout *after* it happens. The web client's `WorkoutModal` is a post-hoc editor: you finish your session, drive home, and then sit down to reconstruct what you did — which exercises, how many sets, what weight, in what order. That reconstruction is exactly the friction that kills logging habits. It is extra work tacked onto a session that already felt complete, it relies on memory that has already started to fade, and it is the single most common reason a tracked workout never gets entered at all.

This feature removes the post-hoc step by letting the user log *as they lift*. From either the phone (the primary case — you have it in your hand at the gym) or the mobile browser, the user starts a session with one tap, lands on a dedicated live-session screen, and adds exercises and sets in real time as they complete them. Supersets are first-class: the user groups exercises and logs a round at a time. When the session is done, a review screen lets them verify and fix everything — including the start and end times, for the inevitable "I forgot to hit end and finished this at home" case — before it is saved to their profile.

Two constraints shape the whole design. First, the session must not trap the user: they can navigate anywhere in the app (chat with the agent, check progress, log nutrition) mid-workout and return with the session intact, because pausing your workout to use the app is its own kind of friction. Second, this is almost entirely a *client* feature. The Go API already models the full domain — a `Workout` owns ordered `WorkoutExercise`s, each carrying a nullable `superset_group` and a list of `Set`s with reps, weight, and unit, plus `performed_at`, `ended_at`, and `notes`. The work is building the live-capture experience on top of that contract on both clients, with no schema or endpoint changes.

## Proposed Solution

Each client gains an **active workout session** that lives in a root-level context provider and is mirrored to local persistence (`AsyncStorage` on mobile, `localStorage` on web). The draft holds everything needed to eventually `POST /workouts`: name, `performed_at`, notes, and an ordered list of exercises, each with its sets and an optional superset grouping. Because the provider is mounted above the navigator, the session survives both in-app navigation and a same-device app restart. The draft is purely local until the user confirms the review screen; a single `POST /workouts` at that moment is the only network write the session makes. There is no server-side "active session" concept and therefore no cross-device resume — a deliberate v1 simplification (see Non-Goals).

The user flow is: **start → log → (navigate freely) → finish → review → save.**

**Start** is one tap. It creates a session pre-named with a sensible default (`Workout — <Weekday>, <Mon D>`, mirroring the server's existing default-naming convention) and routes straight to the live screen. No form stands between intent and logging; the name is editable inline at any time.

**Log** happens on the live-session screen. The user adds exercises one at a time through a search field that filters the already-cached exercise catalog. Each exercise renders as a card to which the user appends sets (weight + reps; unit comes from their profile) as they finish them. The first set of a newly added exercise is prefilled from the user's most recent logged performance of that exercise ("last-time prefill") to cut keystrokes. A running session-duration clock sits in the header.

**Supersets** are created by grouping two or more exercises into a single card. The card logs a *round* at a time — one set per member exercise — then advances to the next round. On save this maps cleanly onto the existing data model: every member `WorkoutExercise` shares the same `superset_group` integer and carries its own per-exercise sets.

**Navigate freely**: the root-level provider means leaving the live screen never discards the session. An app-wide "workout in progress" indicator (a persistent banner/pill) gives a one-tap route back into the live screen from anywhere.

**Finish → Review → Save**: "Finish" routes to a review screen that presents the entire captured session, fully editable — name, notes, every exercise and set, and crucially the `performed_at` / `ended_at` timestamps. Confirming fires `POST /workouts`, clears the local draft, and lands the user on the saved workout (surfacing any personal records the response reports). A **discard** affordance is available throughout the flow, behind a confirm, to abandon a session and clear its draft without saving.

## Goals and Non-Goals

### Goals

- A root-mounted `ActiveWorkoutSession` provider on each client, persisted to device storage, holding the full draft and surviving navigation and same-device app restart.
- One-tap start with a sensible default name and immediate routing to the live-session screen; name editable inline.
- A live-session screen: add exercises via catalog search, append sets (weight/reps, profile unit) per exercise, reorder/remove exercises and sets.
- **Last-time prefill**: a newly added exercise's first set defaults to the user's most recent logged values for that exercise.
- **Superset round logging**: group 2+ exercises into one card that logs a round at a time and advances; persists as shared `superset_group` ints with per-exercise sets.
- **Live session duration** clock in the header, derived from `performed_at`.
- **Navigate-during-session**: full app navigation while a session is active, with an app-wide "in progress" indicator that routes back to the live screen.
- **Rest timer**: a countdown that starts when a set is logged (scope boundary defined in Implementation Details).
- **Review-before-save** screen with full edit of all fields including `performed_at` and `ended_at`; confirm performs a single `POST /workouts`.
- **Discard/abandon** with confirmation, clearing the local draft.
- Feature parity in behavior and data contract across mobile (Expo) and web (Next.js), built on a shared draft model and save payload.

### Non-Goals

- **Server-backed / cross-device active sessions.** The draft is local-only; a session started on the phone does not resume on the laptop. Revisiting this would add a session status field, a migration, and sync logic — out of scope for a solo pre-launch app. The review screen's editable timestamps already cover the dominant "finished it later" pain point.
- **Real-time multi-device sync** of an in-progress session. No push, no subscription. Out of scope as a corollary of local-only state.
- **No API or schema changes.** The existing `Workout` / `WorkoutExercise` / `Set` model and `POST /workouts` are sufficient. If something forces a contract change, that is a separate decision, not a silent expansion of this SOW.
- **Programmed or templated workouts** ("apply my 5/3/1 day", "repeat last Chest & Back"). Adjacent and appealing, but a separate feature with its own template model and UI. This SOW is freeform live capture only.
- **Agent-initiated live sessions.** The AI agent can already create completed workouts via the existing MCP `create_workout` tool; driving the *live* session UI from the agent is out of scope here.
- **Editing a previously saved workout** through this flow. The web `WorkoutModal` continues to own post-save editing. This feature ends at first save.
- **Plate math / barbell loading calculators, RPE/RIR capture, and per-set rest logging.** Not in v1; can be layered onto the set row later without changing the session architecture.

## Implementation Details

### Shared draft model

Both clients model the draft on the existing save payload so the final `POST /workouts` is a near-direct serialization. The TypeScript shape (defined once per client, mirrored exactly between them):

```ts
type DraftSet = { reps: number; weight: number; unit: "lb" | "kg" };

type DraftExercise = {
  exercise_id: string;       // slug from the catalog
  superset_group: number | null;
  sets: DraftSet[];
  notes?: string;
};

type ActiveSession = {
  name: string;
  performed_at: string;      // RFC3339, stamped at start
  notes?: string;
  exercises: DraftExercise[]; // order is array order
  // client-only, not sent on save:
  restTimer?: { startedAt: string; durationSec: number } | null;
};
```

On save, the client emits the existing create-workout body: `name`, `performed_at`, `ended_at` (stamped at the moment of save unless the user overrode it on the review screen), `notes`, and `exercises` with `order` derived from array index, each carrying `superset_group`, `notes`, and `sets`. This is exactly the shape the API and the MCP `create_workout` tool already accept.

### Session provider and persistence

A root-mounted provider (`ActiveWorkoutSessionProvider`) owns the draft and exposes actions: `start`, `setName`, `addExercise`, `removeExercise`, `reorderExercise`, `addSet`, `updateSet`, `removeSet`, `createSuperset`, `logSupersetRound`, `setNotes`, `setTimes`, `discard`, and `save`. Every mutation writes through to device storage (`AsyncStorage` / `localStorage`) so a reload or cold start rehydrates the in-progress session. On mobile the provider mounts in `app/_layout.tsx` alongside `ProfileProvider` and `ExerciseCatalogProvider`; on web it mounts in the authenticated layout above the app router. A single boolean (`session !== null`) drives the app-wide "in progress" indicator.

### Live-session screen

- **Exercise search** filters the in-memory catalog already provided by `ExerciseCatalogProvider` (mobile) / the catalog query (web) — client-side filtering, no new endpoint. Selecting a result appends a `DraftExercise`.
- **Set entry** appends a `DraftSet`; weight unit defaults from the user's profile. Sets are individually editable and removable.
- **Last-time prefill** resolves a newly added exercise's most recent prior set. Implementation: on session start (or lazily on first add), fetch a bounded page of recent workouts via `GET /workouts` and build an in-memory map of `exercise_id → most-recent set`. A newly added exercise seeds its first set from that map when present; absent any history, the set starts empty. This is a read-only convenience and degrades silently if the fetch fails.
- **Duration clock** ticks from `performed_at` in the header.

### Supersets

`createSuperset` assigns a fresh `superset_group` integer (monotonic within the session) to two or more selected exercises and renders them as a single grouped card. The card tracks a current round index; `logSupersetRound` appends one set to each member exercise and advances the round. Members may have differing set counts if the user logs partial rounds — the persisted shape tolerates this since sets live per exercise. Ungrouping clears the members' `superset_group` back to `null`.

### Rest timer

A foreground countdown that starts when a set is logged, with a user-configurable default duration. **Scope boundary**: on web and in the foregrounded mobile app the timer is a live on-screen countdown; on mobile, leaving the app schedules a single local notification at the timer's end rather than guaranteeing precise background tick execution. The timer is a convenience nudge, not a real-time guarantee, and timer state is client-only (never part of the saved workout). This keeps the feature off the path of background-execution and notification-permission complexity while still being useful in the common case.

### Review and save

"Finish" routes to a review screen rendering the full draft with every field editable, including `performed_at` and `ended_at` (datetime pickers) so a user who forgot to end the session can correct both ends. Confirming calls the existing create-workout client function with the serialized body, then `discard`s the local draft on success and routes to the saved workout, surfacing any personal records returned in the response. On failure the draft is preserved and the error surfaced so nothing is lost. **Discard** (available from the live screen and the review screen, behind a confirm) clears the draft without a network call.

### Cross-client consistency

Mobile and web ship the same draft model, the same action surface, and the same save payload. The two screens differ only in presentation primitives (Expo Router screen + native inputs vs. Next.js route + web inputs) and persistence backend (`AsyncStorage` vs. `localStorage`). Keeping the model and payload identical is what prevents the two clients from drifting into subtly different workout records.

## Resolutions

*(Open questions resolved during design; recorded here as they were settled.)*

1. **Where the in-progress session lives.** Resolved: local-only draft in a root context with device persistence. Survives navigation and same-device restart; no server-side active session and no cross-device resume. Chosen for simplicity and zero API/cost surface on a solo pre-launch app; the editable review-screen timestamps absorb the main "finished later" use case that a server session would otherwise justify.

2. **SOW scope across surfaces.** Resolved: one SOW covering mobile and web together, sharing a single draft model and save payload, with per-surface implementation sections. Keeps the two clients from diverging.

3. **Superset logging interaction.** Resolved: grouped "round" logging — one card per superset, log a round (a set per member), advance. Maps to shared `superset_group` ints with per-exercise sets. Chosen over a flat cosmetic-tag approach because round-based logging is the actual reason to support supersets.

4. **Optional conveniences in v1.** Resolved: last-time prefill, live session duration, discard/abandon, and the rest timer are all in scope. The rest timer ships with the foreground-plus-single-local-notification boundary described above rather than guaranteed background execution.
