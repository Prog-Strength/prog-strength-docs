# Live Workout Logging Session — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development to implement this plan task-by-task. Every task is implemented by an implementer subagent, then spec-reviewed and code-quality-reviewed before moving on. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let a lifter log a workout *as they lift* — start with one tap, land on a dedicated live-session screen, add exercises and sets in real time (supersets logged a round at a time), navigate the rest of the app freely while the session stays alive, then review-and-save with a single `POST /workouts`. Build the same experience on both clients (Expo mobile + Next.js web) on a shared draft model and save payload. **No API or schema changes** — the existing `Workout`/`WorkoutExercise`/`Set` contract and `POST /workouts` are sufficient.

**SOW:** `sows/live-workout-logging-session.md`

**Architecture:** Each client gets a root-mounted `ActiveWorkoutSessionProvider` that owns a single local draft (`ActiveSession`), mirrors every mutation to device storage (`localStorage` on web, `AsyncStorage` on mobile), and survives both in-app navigation and same-device restart. The draft is local-only until the user confirms the review screen, where a single `POST /workouts` fires. A `session !== null` boolean drives an app-wide "workout in progress" indicator that routes back to the live screen. The draft model, the pure mutation/serialization logic, and the save payload are **identical across clients** (the canonical TypeScript is embedded below in Task W1/M1) so the two clients can never drift into different workout records.

**Tech stack:**
- **Web** (`/workspace/prog-strength-web`): Next.js 16 App Router (client components), React 19, Tailwind v4 (CSS variables for theme), TanStack Query available but most pages use `fetch` + `useState`. Test suite is **Vitest + Testing Library (jsdom)**, co-located `*.test.tsx`. Verify with `npm run typecheck`, `npm run lint`, `npm test`, `npm run build`.
- **Mobile** (`/workspace/prog-strength-mobile`): Expo Router, React Native, NativeWind (Tailwind v3, dark-only tokens: `background #0a0a0b`, `surface #18181b`, `border #27272a`, `foreground #fafafa`, `muted #a1a1aa`, `accent #3b82f6`, `danger #ef4444`). **No test runner exists**; verify with `npm run typecheck` (`tsc --noEmit`), `npm run lint` (`expo lint`), and `npm run format:check`. Pure model logic is shared verbatim with web (where it *is* unit-tested), so mobile relies on typecheck + the web tests for the shared logic.

## Working directories

Web tasks (`W*`) run in `/workspace/prog-strength-web`. Mobile tasks (`M*`) run in `/workspace/prog-strength-mobile`. Each is on its own `feat/live-workout-logging-session` branch against `main`. Run all `npm` commands from the respective repo root. Both repos use `package-lock.json` (`npm ci` already run by the orchestrator).

## Shared facts (both clients — gathered from the codebases)

- **Types live in `lib/api.ts`** on both clients (mirrored, "edit twice" discipline). The save payload type already exists on both:
  ```ts
  export type WorkoutSet = { reps: number; weight: number; unit: "lb" | "kg" };
  export type WorkoutPayload = {
    name?: string;
    performed_at: string;   // RFC3339, required
    ended_at?: string;      // RFC3339, optional
    notes?: string;
    exercises: { exercise_id: string; superset_group?: number | null; notes?: string; sets: WorkoutSet[] }[];
  };
  ```
- **`createWorkout` does NOT exist on either client yet** — only `listWorkouts`, `getWorkout`, `updateWorkout`, `deleteWorkout`. It must be added (POST /workouts), mirroring `updateWorkout` exactly but with `method: "POST"` and no id in the path.
- **Response** of create is a `Workout` (id, name?, performed_at, ended_at?, notes?, exercises, …, `personal_records_set: PersonalRecordEvent[]`). Surface PRs from `personal_records_set`.
- **Profile** exposes `weight_unit: "lb" | "kg"` via `useProfile()` (`lib/profile-context.tsx`) on both clients.
- **Exercise catalog item**: `{ id: string /* slug */, name, description?, muscle_groups: string[], equipment: string[] }`. Fetched via `listExercises()` (public, no auth). Web has **no catalog provider** (callers call `listExercises()` directly, e.g. `components/activities/workouts-view.tsx`, `app/(app)/workouts/[id]/page.tsx`). Mobile **has** `ExerciseCatalogProvider` (`components/exercise-catalog-context.tsx`) mounted in `app/_layout.tsx`, exposing `{ exercises, byID, loading, error, refresh }` via `useExerciseCatalog()`.
- **Auth**: web `getToken()` (sync, `lib/auth.ts`, localStorage); mobile `getToken()` (async, `lib/auth.ts`, Keychain). 401 → clear token + route to `/login`.
- **`listWorkouts(token, { limit })`** returns `{ items: Workout[], total, limit, offset, has_more }`. Used for last-time prefill.
- **Web WorkoutModal** (`components/workout-modal.tsx`) is the canonical reference for draft→payload conversion, datetime-local ↔ RFC3339 helpers, exercise/set editing UI, and validation. Reuse its `rfc3339ToLocalInput`/`localInputToRFC3339` patterns and its ExerciseCard layout.
- **Dumbbell weight is per-dumbbell**, not pair (web convention) — no special handling needed, just don't double anything.

---

# WEB

## Task W1: Shared draft model + `createWorkout` API (web, tested)

**Why first:** every later web task imports the draft model and the create function. The pure logic lives in one module and is unit-tested with Vitest; the provider and screens build on it.

**Files:**
- Create: `lib/workout-draft.ts`
- Create: `lib/workout-draft.test.ts`
- Modify: `lib/api.ts` (add `createWorkout`)

- [ ] **Step 1: Add `createWorkout` to `lib/api.ts`.** Place it directly above `updateWorkout`. Mirror `updateWorkout` exactly but POST to `/workouts` (no id), same `unwrap<Workout | null>` + null guard:
  ```ts
  /**
   * POST /workouts. Creates a workout from the full draft payload. Returns
   * the created Workout (including any personal_records_set the save
   * triggered) so the caller can route to it and surface PRs without a
   * follow-up fetch. Throws the API's `error` envelope on non-2xx.
   */
  export async function createWorkout(token: string, payload: WorkoutPayload): Promise<Workout> {
    const resp = await fetch(`${config.apiUrl}/workouts`, {
      method: "POST",
      headers: { "Content-Type": "application/json", Authorization: `Bearer ${token}` },
      body: JSON.stringify(payload),
    });
    const created = await unwrap<Workout | null>(resp, null);
    if (!created) throw new Error("API did not return the created workout");
    return created;
  }
  ```

- [ ] **Step 2: Create `lib/workout-draft.ts`** — the canonical shared model + pure logic. **This exact content is the single source of truth; the mobile task M1 copies it verbatim (only the import path for `WorkoutSet`/`WorkoutPayload` is identical since both live in `lib/api.ts`).**

  ```ts
  import type { WorkoutPayload, WorkoutSet, Workout } from "@/lib/api";

  export type DraftSet = { reps: number; weight: number; unit: "lb" | "kg" };

  export type DraftExercise = {
    exercise_id: string; // slug from the catalog
    superset_group: number | null;
    sets: DraftSet[];
    notes?: string;
  };

  export type ActiveSession = {
    name: string;
    performed_at: string; // RFC3339, stamped at start
    notes?: string;
    exercises: DraftExercise[]; // order is array order
    // client-only, never sent on save:
    restTimer?: { startedAt: string; durationSec: number } | null;
    restDefaultSec: number; // user-configurable default rest duration
  };

  /** "Workout — Sat, Jun 14" — mirrors the server's default-naming convention. */
  export function defaultSessionName(now: Date): string {
    const weekday = now.toLocaleDateString("en-US", { weekday: "short" });
    const mon = now.toLocaleDateString("en-US", { month: "short" });
    return `Workout — ${weekday}, ${mon} ${now.getDate()}`;
  }

  export function createSession(now: Date): ActiveSession {
    return {
      name: defaultSessionName(now),
      performed_at: now.toISOString(),
      exercises: [],
      restTimer: null,
      restDefaultSec: 90,
    };
  }

  export function defaultSet(unit: "lb" | "kg"): DraftSet {
    return { reps: 0, weight: 0, unit };
  }

  // --- pure mutators (return a new session; never mutate the input) ---

  export function addExercise(s: ActiveSession, exercise_id: string, firstSet?: DraftSet): ActiveSession {
    return { ...s, exercises: [...s.exercises, { exercise_id, superset_group: null, sets: firstSet ? [firstSet] : [] }] };
  }

  export function removeExercise(s: ActiveSession, idx: number): ActiveSession {
    return { ...s, exercises: s.exercises.filter((_, i) => i !== idx) };
  }

  export function reorderExercise(s: ActiveSession, from: number, to: number): ActiveSession {
    if (from === to || from < 0 || to < 0 || from >= s.exercises.length || to >= s.exercises.length) return s;
    const next = [...s.exercises];
    const [moved] = next.splice(from, 1);
    next.splice(to, 0, moved);
    return { ...s, exercises: next };
  }

  export function addSet(s: ActiveSession, exIdx: number, set: DraftSet): ActiveSession {
    return mapExercise(s, exIdx, (ex) => ({ ...ex, sets: [...ex.sets, set] }));
  }

  export function updateSet(s: ActiveSession, exIdx: number, setIdx: number, patch: Partial<DraftSet>): ActiveSession {
    return mapExercise(s, exIdx, (ex) => ({ ...ex, sets: ex.sets.map((st, i) => (i === setIdx ? { ...st, ...patch } : st)) }));
  }

  export function removeSet(s: ActiveSession, exIdx: number, setIdx: number): ActiveSession {
    return mapExercise(s, exIdx, (ex) => ({ ...ex, sets: ex.sets.filter((_, i) => i !== setIdx) }));
  }

  /** Assign a fresh monotonic superset_group to the given exercise indices (2+). */
  export function createSuperset(s: ActiveSession, idxs: number[]): ActiveSession {
    if (idxs.length < 2) return s;
    const group = nextSupersetGroup(s);
    const set = new Set(idxs);
    return { ...s, exercises: s.exercises.map((ex, i) => (set.has(i) ? { ...ex, superset_group: group } : ex)) };
  }

  /** Clear superset_group back to null for all members of a group. */
  export function ungroupSuperset(s: ActiveSession, group: number): ActiveSession {
    return { ...s, exercises: s.exercises.map((ex) => (ex.superset_group === group ? { ...ex, superset_group: null } : ex)) };
  }

  /** Append one set to each member of the group (a "round"). Defaults each member's set from its previous set. */
  export function logSupersetRound(s: ActiveSession, group: number, sets: Record<number, DraftSet>): ActiveSession {
    return {
      ...s,
      exercises: s.exercises.map((ex, i) => {
        if (ex.superset_group !== group) return ex;
        const next = sets[i] ?? ex.sets[ex.sets.length - 1] ?? defaultSet(ex.sets[0]?.unit ?? "lb");
        return { ...ex, sets: [...ex.sets, next] };
      }),
    };
  }

  export function setName(s: ActiveSession, name: string): ActiveSession { return { ...s, name }; }
  export function setNotes(s: ActiveSession, notes: string): ActiveSession { return { ...s, notes }; }
  export function setExerciseNotes(s: ActiveSession, exIdx: number, notes: string): ActiveSession {
    return mapExercise(s, exIdx, (ex) => ({ ...ex, notes }));
  }

  /** Monotonic within the session: 1 + current max group, or 1 if none. */
  export function nextSupersetGroup(s: ActiveSession): number {
    const groups = s.exercises.map((e) => e.superset_group).filter((g): g is number => g !== null);
    return groups.length ? Math.max(...groups) + 1 : 1;
  }

  /**
   * Serialize a session to the create-workout body. `endedAt` is stamped at
   * save unless the user overrode it on the review screen (pass the override
   * RFC3339 string, or undefined to omit). Drops empty optional fields like
   * the WorkoutModal does, and skips exercises with zero sets.
   */
  export function sessionToPayload(s: ActiveSession, endedAt: string | undefined): WorkoutPayload {
    const payload: WorkoutPayload = {
      performed_at: s.performed_at,
      exercises: s.exercises
        .filter((ex) => ex.sets.length > 0 && ex.exercise_id)
        .map((ex) => ({
          exercise_id: ex.exercise_id,
          ...(ex.superset_group !== null && { superset_group: ex.superset_group }),
          ...(ex.notes && { notes: ex.notes }),
          sets: ex.sets.map((st) => ({ reps: st.reps, weight: st.weight, unit: st.unit }) as WorkoutSet),
        })),
    };
    if (s.name) payload.name = s.name;
    if (endedAt) payload.ended_at = endedAt;
    if (s.notes) payload.notes = s.notes;
    return payload;
  }

  export function isSessionSaveable(s: ActiveSession): boolean {
    const exs = s.exercises.filter((ex) => ex.sets.length > 0 && ex.exercise_id);
    if (exs.length === 0) return false;
    for (const ex of exs) {
      for (const st of ex.sets) {
        if (!Number.isFinite(st.reps) || st.reps <= 0) return false;
        if (!Number.isFinite(st.weight) || st.weight < 0) return false;
      }
    }
    return true;
  }

  /**
   * Build a map of exercise_id → most-recent logged set, from a page of recent
   * workouts (newest first by performed_at). Used for last-time prefill. Read-
   * only convenience; callers degrade silently if the source fetch failed.
   */
  export function buildPrefillMap(workouts: Workout[]): Map<string, DraftSet> {
    const sorted = [...workouts].sort((a, b) => b.performed_at.localeCompare(a.performed_at));
    const map = new Map<string, DraftSet>();
    for (const w of sorted) {
      for (const ex of w.exercises) {
        if (map.has(ex.exercise_id)) continue;
        const last = ex.sets[ex.sets.length - 1];
        if (last) map.set(ex.exercise_id, { reps: last.reps, weight: last.weight, unit: last.unit });
      }
    }
    return map;
  }

  function mapExercise(s: ActiveSession, idx: number, fn: (ex: DraftExercise) => DraftExercise): ActiveSession {
    return { ...s, exercises: s.exercises.map((ex, i) => (i === idx ? fn(ex) : ex)) };
  }
  ```

- [ ] **Step 3: Create `lib/workout-draft.test.ts`** — Vitest unit tests covering: `defaultSessionName` format; `createSession` stamps performed_at and default name; add/remove/reorder exercise; add/update/remove set; `createSuperset` assigns a shared monotonic group and `nextSupersetGroup` increments; `ungroupSuperset` clears; `logSupersetRound` appends one set per member and tolerates differing counts; `sessionToPayload` derives order from array index, drops empty optionals, omits zero-set exercises, includes `ended_at` only when passed; `isSessionSaveable` rejects empty/invalid; `buildPrefillMap` picks the most-recent set per exercise across workouts.

- [ ] **Step 4: Verify and commit.**
  ```bash
  npm run typecheck && npm run lint && npm test
  git add lib/workout-draft.ts lib/workout-draft.test.ts lib/api.ts
  git commit -m "feat(workout): shared live-session draft model + createWorkout API"
  ```

## Task W2: `ActiveWorkoutSessionProvider` + persistence + mount + start/indicator (web)

**Why:** the provider is the root of the feature — it owns the draft, persists to `localStorage`, exposes the action surface, and drives the in-progress indicator. Mounting it and wiring a start entry point makes the feature reachable.

**Files:**
- Create: `lib/active-workout-session.tsx`
- Create: `lib/active-workout-session.test.tsx`
- Modify: `app/(app)/layout.tsx` (mount provider above the app shell)
- Create: `components/live-workout/in-progress-banner.tsx`
- Modify: `app/(app)/layout.tsx` or `components/sidebar.tsx` to render the banner app-wide
- Modify: `components/activities/workouts-view.tsx` (add a "Start live workout" button in the workouts view header)

- [ ] **Step 1: Provider.** `ActiveWorkoutSessionProvider` holds `session: ActiveSession | null` in state, hydrated once from `localStorage` key `ps_active_workout_session` on mount (guard `typeof window`). Every state change writes through to `localStorage` (and removes the key when session becomes null). Expose via `useActiveWorkoutSession()`:
  - `session`
  - `start()` → `createSession(new Date())`, then **kick off last-time prefill**: `listWorkouts(token, { limit: 50 })` → `buildPrefillMap`, stored in a ref/state on the provider and exposed as `prefill: Map<string, DraftSet>` (degrade silently on failure).
  - action methods that wrap the pure mutators from `lib/workout-draft.ts`: `setName`, `addExercise` (seeds first set from `prefill` when present, else `defaultSet(profile unit)`), `removeExercise`, `reorderExercise`, `addSet`, `updateSet`, `removeSet`, `createSuperset`, `ungroupSuperset`, `logSupersetRound`, `setNotes`, `setExerciseNotes`, `setTimes(performed_at?, ended_at?)`, `startRestTimer()`, `clearRestTimer()`, `setRestDefault(sec)`.
  - `discard()` → clear session + storage.
  - `save(endedAt?)` → `createWorkout(token, sessionToPayload(session, endedAt ?? new Date().toISOString()))`; on success clear the draft and return the `Workout`; on failure **throw** (caller surfaces error, draft preserved).
  - Throw a clear error if `useActiveWorkoutSession` is used outside the provider (match `useProfile` pattern).
  - Get the weight unit from `useProfile()` for new-set defaults; tolerate `profile === null` (fall back to `"lb"`).
- [ ] **Step 2: Persistence test.** Vitest: mutating the session writes JSON to `localStorage`; a fresh provider hydrates from `localStorage`; `discard`/successful `save` removes the key. Mock `lib/api` (`createWorkout`, `listWorkouts`) and `lib/auth` (`getToken`) following the `profile-context.test.tsx` `vi.hoisted`/`vi.mock` pattern.
- [ ] **Step 3: Mount.** In `app/(app)/layout.tsx`, wrap the authenticated shell with `<ActiveWorkoutSessionProvider>` (inside `ProfileProvider` so it can read the unit).
- [ ] **Step 4: In-progress banner.** `components/live-workout/in-progress-banner.tsx`: a client component that reads `useActiveWorkoutSession()`; when `session !== null` and the current route is not the live screen, render a persistent pill/banner ("Workout in progress · <duration>") that links to `/workout/live`. Render it in the authenticated layout so it shows on every authed page. Use `usePathname()` to hide it on the live/review routes.
- [ ] **Step 5: Start entry point.** Add a primary "Start live workout" button to the workouts view header (`components/activities/workouts-view.tsx`) that calls `start()` then `router.push("/workout/live")`. If a session already exists, the button instead says "Resume workout" and just navigates.
- [ ] **Step 6: Verify and commit.**
  ```bash
  npm run typecheck && npm run lint && npm test && npm run build
  git add -A && git commit -m "feat(workout): active-session provider, persistence, start + in-progress banner"
  ```

## Task W3: Live-session screen (web)

**Why:** the core logging surface — add exercises via catalog search, append/edit/remove sets, superset round cards, duration clock, rest timer, discard.

**Files:**
- Create: `app/(app)/workout/live/page.tsx`
- Create: `components/live-workout/exercise-search.tsx`
- Create: `components/live-workout/exercise-card.tsx`
- Create: `components/live-workout/superset-card.tsx`
- Create: `components/live-workout/rest-timer.tsx`
- Create: `components/live-workout/duration-clock.tsx`
- Optionally: `components/live-workout/use-now.ts` (a `useNow(intervalMs)` ticking hook for the clock/timer)

- [ ] **Step 1: Live page.** Client component at `/workout/live`. On mount: if `session === null`, redirect to `/activities?view=workouts` (nothing to log). Fetch the catalog once via `listExercises()` (mirror `workouts-view.tsx`). Header: inline-editable name (`setName`), a `DurationClock` ticking from `session.performed_at`, and a "Finish" button → `router.push("/workout/review")`. Body: an `ExerciseSearch` to add exercises, then the ordered list of exercise/superset cards. Footer: "Discard" (behind a confirm) → `discard()` + route away.
- [ ] **Step 2: ExerciseSearch.** Text input that client-side filters the catalog by name (case-insensitive; optionally muscle group/equipment like the mobile exercises screen). Selecting a result calls `addExercise(exercise_id)` (provider seeds the prefilled first set) and clears the query.
- [ ] **Step 3: ExerciseCard.** Renders one non-superset exercise: name (from catalog), per-set rows (reps / weight / unit) editable and removable (reuse the WorkoutModal ExerciseCard grid + `inputClasses`), "+ Add set" (copies last set's weight/unit as default, then `startRestTimer()`), exercise notes input, remove-exercise, and a "Make superset" affordance (select this + another to group). Reorder via up/down buttons calling `reorderExercise` (keep it simple — no DnD library).
- [ ] **Step 4: SupersetCard.** Renders a grouped card for exercises sharing a `superset_group`: shows each member and the current round index; a "Log round" button calls `logSupersetRound(group, perMemberSets)` appending one set per member and advancing, then `startRestTimer()`. Members may end with differing set counts (partial rounds) — that's fine. An "Ungroup" button calls `ungroupSuperset(group)`.
- [ ] **Step 5: RestTimer.** Reads `session.restTimer`; when set, shows a live countdown (`useNow`) from `startedAt + durationSec`; at zero, shows "Rest complete" (a simple visual/optional `Notification` if permission already granted — do not prompt). Lets the user adjust the default duration (`setRestDefault`) and dismiss (`clearRestTimer`). Web is foreground-only; no background scheduling.
- [ ] **Step 6: DurationClock.** Formats `now - performed_at` as `H:MM:SS` / `MM:SS`, ticking each second via `useNow(1000)`.
- [ ] **Step 7: Verify and commit.**
  ```bash
  npm run typecheck && npm run lint && npm test && npm run build
  git add -A && git commit -m "feat(workout): live-session screen — search, sets, supersets, clock, rest timer"
  ```

## Task W4: Review + save screen (web)

**Why:** closes the loop — full edit of every field incl. `performed_at`/`ended_at`, single `POST /workouts`, PR surfacing, route to the saved workout.

**Files:**
- Create: `app/(app)/workout/review/page.tsx`

- [ ] **Step 1: Review page.** Client component at `/workout/review`. If `session === null`, redirect to `/activities?view=workouts`. Render the full draft editable: name, notes, every exercise and set (reuse the live ExerciseCard editing), and crucially **`performed_at` and `ended_at` as `datetime-local` inputs** (default `ended_at` to "now"; convert with the WorkoutModal `rfc3339ToLocalInput`/`localInputToRFC3339` helpers — extract them to `lib/workout-draft.ts` or a small `lib/datetime.ts` and have WorkoutModal import them too, to avoid duplication). A "Save workout" button (disabled unless `isSessionSaveable`) calls `setTimes` for any timestamp edits then `save(endedAtRFC3339)`.
- [ ] **Step 2: Save outcome.** On success: the draft is cleared by the provider; route to the saved workout (`/workouts/<id>` if that route renders a single workout, else `/activities?view=workouts`) and surface any `created.personal_records_set` (toast via the existing `ToastProvider`, e.g. "New PR: <exercise> <weight><unit> × <reps>"). On failure: show the error inline and keep the draft (no navigation).
- [ ] **Step 3: Verify and commit.**
  ```bash
  npm run typecheck && npm run lint && npm test && npm run build
  git add -A && git commit -m "feat(workout): review-and-save screen with editable timestamps + PR surfacing"
  ```

---

# MOBILE

## Task M1: Shared draft model + `createWorkout` API + AsyncStorage dep (mobile)

**Why first:** mirrors W1 so the two clients share an identical model and payload. Also adds the `@react-native-async-storage/async-storage` dependency the provider needs.

**Files:**
- Modify: `package.json` + `package-lock.json` (add async-storage via `npx expo install`)
- Create: `lib/workout-draft.ts` (verbatim copy of the web canonical module from Task W1 Step 2)
- Modify: `lib/api.ts` (add `createWorkout`)

- [ ] **Step 1: Add dependency.** `npx expo install @react-native-async-storage/async-storage` (uses the Expo-compatible version and updates the lockfile). This is a JS/native-config dependency; note in the eventual PR whether it changes the native fingerprint (`npm run fingerprint:check`). If a config-plugin/rebuild is required, the PR description must call it out (it likely is **not** OTA-able if the native module is new — flag this honestly).
- [ ] **Step 2: Add `createWorkout` to `lib/api.ts`** directly above `updateWorkout`, mirroring the existing mobile `updateWorkout` (async token already passed in by callers) with `method: "POST"` to `${config.apiUrl}/workouts` and the same `unwrap` + null guard. Match mobile's existing style.
- [ ] **Step 3: Create `lib/workout-draft.ts`** — copy the exact module from Task W1 Step 2. The `@/lib/api` import path resolves on mobile too (tsconfig paths). Do not add tests (no runner); the web copy is the tested source of truth. Add a top-of-file comment: `// MIRRORS prog-strength-web/lib/workout-draft.ts — keep identical.`
- [ ] **Step 4: Verify and commit.**
  ```bash
  npm run typecheck && npm run lint && npm run format:check
  git add -A && git commit -m "feat(workout): shared live-session draft model + createWorkout API + async-storage"
  ```

## Task M2: `ActiveWorkoutSessionProvider` + AsyncStorage + mount + start/indicator (mobile)

**Why:** root-mounted provider with device persistence + the in-progress pill + one-tap start.

**Files:**
- Create: `lib/active-workout-session.tsx`
- Modify: `app/_layout.tsx` (mount alongside `ProfileProvider`, `UsageProvider`, `ExerciseCatalogProvider`)
- Create: `components/active-workout-banner.tsx` (persistent pill)
- Modify: a sensible start entry point — the Activities hub (`app/(tabs)/activities/index.tsx`) or workouts view (`components/activities/workouts-view.tsx`)

- [ ] **Step 1: Provider.** Same action surface and semantics as web W2 Step 1, but persistence uses `AsyncStorage` (async get/set/remove, key `ps_active_workout_session`). Hydrate on mount via an effect (AsyncStorage is async — hold a `hydrated` boolean and render children immediately; the session populates once read). Write through on every change. `start()` runs last-time prefill via `listWorkouts(await getToken(), { limit: 50 })` → `buildPrefillMap`. `save()` calls `createWorkout(await getToken(), …)`. Read weight unit from `useProfile()`. `useActiveWorkoutSession()` throws outside the provider.
- [ ] **Step 2: Mount.** Add `<ActiveWorkoutSessionProvider>` to `app/_layout.tsx` nested with the other root providers (inside `ProfileProvider` so it can read the unit; the live screen is a root route like `/exercises`, so the provider must be at root, not inside `(tabs)`).
- [ ] **Step 3: In-progress pill.** `components/active-workout-banner.tsx`: when `session !== null` and not on the live/review screen, render a persistent pill (absolute, above the tab bar) "Workout in progress · <duration>" that routes to the live screen on tap. Mount it in `app/_layout.tsx` (or the tabs layout) so it shows app-wide. Use `usePathname()`/segments to hide on the live & review routes.
- [ ] **Step 4: Start entry point.** Add a primary "Start live workout" button to the Activities workouts view; calls `start()` then `router.push("/workout/live")`. Shows "Resume workout" when a session already exists.
- [ ] **Step 5: Verify and commit.**
  ```bash
  npm run typecheck && npm run lint && npm run format:check
  git add -A && git commit -m "feat(workout): active-session provider, AsyncStorage persistence, start + in-progress pill"
  ```

## Task M3: Live-session screen (mobile)

**Why:** the core logging surface on phone — the primary case.

**Files:**
- Create: `app/workout/_layout.tsx` (Stack for the workout flow, root-level)
- Create: `app/workout/live.tsx`
- Create: `components/live-workout/exercise-search.tsx`
- Create: `components/live-workout/exercise-card.tsx`
- Create: `components/live-workout/superset-card.tsx`
- Create: `components/live-workout/rest-timer.tsx`
- Create: `components/live-workout/duration-clock.tsx`

- [ ] **Step 1: Route + screen.** Root-level `app/workout/live.tsx` (Stack screen, header hidden or titled). If `session === null`, `router.replace` back to activities. Use `useExerciseCatalog()` for the catalog (already provided at root). Header: inline-editable name (`TextInput` → `setName`), `DurationClock`, and a "Finish" button → `router.push("/workout/review")`. Body: a `FlatList`/`ScrollView` of exercise/superset cards with an `ExerciseSearch` at top. Footer: "Discard" behind an `Alert.alert` confirm → `discard()` + route away. NativeWind styling with the dark tokens.
- [ ] **Step 2: ExerciseSearch.** `TextInput` filtering `useExerciseCatalog().exercises` by name/muscle group/equipment (reuse logic from the existing exercises screen). Tapping a result calls `addExercise(id)` and clears the query.
- [ ] **Step 3: ExerciseCard.** Native card: catalog name; per-set rows with numeric `TextInput`s for reps/weight and a unit toggle (default from profile); add/remove set; "+ Add set" copies last set + `startRestTimer()`; exercise notes; remove-exercise; reorder up/down via `reorderExercise`; "Make superset" affordance.
- [ ] **Step 4: SupersetCard.** Grouped card; current round index; "Log round" → `logSupersetRound` + `startRestTimer()`; "Ungroup" → `ungroupSuperset`. Tolerates partial rounds.
- [ ] **Step 5: RestTimer.** Foreground countdown using a 1s interval hook from `session.restTimer`. **Scope boundary (per SOW):** in the foregrounded app it's a live on-screen countdown; leaving the app schedules a single local notification at the timer's end (only if `expo-notifications` is already available + permission granted — do **not** add background-execution complexity, and do not block the feature on notification permission). If `expo-notifications` is not already a dependency, ship foreground-only countdown and note the notification piece as a follow-up in the PR (adding a native module changes the fingerprint; prefer not to expand native surface in this task). Configurable default via `setRestDefault`; dismiss via `clearRestTimer`.
- [ ] **Step 6: DurationClock.** Ticks each second; formats elapsed from `performed_at`.
- [ ] **Step 7: Verify and commit.**
  ```bash
  npm run typecheck && npm run lint && npm run format:check
  git add -A && git commit -m "feat(workout): live-session screen — search, sets, supersets, clock, rest timer"
  ```

## Task M4: Review + save screen (mobile)

**Why:** closes the loop on mobile.

**Files:**
- Create: `app/workout/review.tsx`

- [ ] **Step 1: Review screen.** If `session === null`, route back to activities. Render the full draft editable: name, notes, every exercise/set, and `performed_at`/`ended_at` via a date/time picker. **Use only an already-available picker** — if `@react-native-community/datetimepicker` is not already a dependency, do not add a native module here; instead provide editable native text fields validated to RFC3339/local format (or use the platform's existing patterns) and note the nicer picker as a follow-up. Default `ended_at` to "now". "Save workout" (disabled unless `isSessionSaveable`) calls `setTimes` then `save(endedAt)`.
- [ ] **Step 2: Save outcome.** On success: provider clears the draft; route to the saved workout detail (`/(tabs)/activities/workout/<id>`) and surface `created.personal_records_set` (Alert or the app's existing toast pattern). On failure: surface the error and keep the draft.
- [ ] **Step 3: Verify and commit.**
  ```bash
  npm run typecheck && npm run lint && npm run format:check
  git add -A && git commit -m "feat(workout): review-and-save screen with editable timestamps + PR surfacing"
  ```

---

## Cross-client consistency check (final, before PRs)

- [ ] Diff `prog-strength-web/lib/workout-draft.ts` against `prog-strength-mobile/lib/workout-draft.ts` — they must be identical except the mirror comment. The save payload (`sessionToPayload`) and draft model must match field-for-field.
- [ ] Both clients: start → add exercise (prefilled first set) → add sets → group a superset → log a round → navigate away and back (session intact) → finish → review (edit times) → save → PRs surfaced → draft cleared.

## Rollout / deployment notes (for the docs PR)

This is a **client-only** feature — no API, schema, or migration changes (`POST /workouts` already exists and is unchanged). The web and mobile PRs are **mutually independent** and can merge in parallel; neither depends on the other and neither depends on any backend deploy. Web ships on the next deploy; mobile ships as an OTA update **only if** no new native module was added (async-storage is a native module — if it isn't already in the binary, mobile needs a new build, which the mobile PR must call out via `fingerprint:check`).

### Hand-test after rollout
- **Web:** Start a live workout from Activities → add an exercise (confirm last-time prefill) → log straight sets → create a superset and log a round → navigate to Chat and back (banner returns you, session intact) → Finish → edit `ended_at` → Save → confirm the workout appears with correct sets/supersets and any PRs are surfaced.
- **Mobile:** Same flow on the phone; additionally force-quit and relaunch mid-session to confirm AsyncStorage rehydration; confirm the in-progress pill routes back to the live screen; confirm the rest timer counts down in the foreground.
