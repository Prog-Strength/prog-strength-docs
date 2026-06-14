# Mobile Parity Phase 2: Activities & Running — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement Phase 2 of `sows/mobile-feature-parity-and-testflight.md`: the Workouts tab becomes an Activities tab with Overview | Workouts | Running segments, run list + detail with pace/HR/elevation charts, TCX import, and runs on the calendar.

**Architecture:** Port the web's `/activities` hub and `/running/[id]` surfaces (reference repo: `../prog-strength-web`) into the existing Expo patterns: `SegmentedControl` for the three segments, `react-native-svg` charts following `bodyweight-chart.tsx` conventions, `lib/api.ts` web-twin fetchers, distance/pace formatting driven by `profile.distance_unit` (server-backed — mobile has no localStorage layer). Route group `app/(tabs)/workouts/` is renamed to `app/(tabs)/activities/` with `workout/[id]` and `run/[id]` detail screens.

**Tech Stack:** Expo SDK 55 / RN 0.83, expo-router (typedRoutes ON), NativeWind, react-native-svg, expo-document-picker (NEW native module — native rebuild required at merge).

**Verification policy:** No JS test runner in this repo (SOW decision — manual on-device smoke is the bar). Every task ends with `npm run typecheck && npm run lint` plus listed manual checks. PATH note for all shell work: `export PATH="$HOME/.nvm/versions/node/v24.15.0/bin:$PATH"`.

**Branch:** `feat/mobile-parity-phase-2` off up-to-date `main` in `prog-strength-mobile`.

**Deliberate deviations from web (do not "fix" these):**
- The Workouts segment keeps the existing weekly SectionList + duration chart unchanged (no timeframe filtering); the hub's timeframe pills drive Overview and Running only. The web filters workouts too — deferred until the segment needs it.
- Overview ports the stat tiles and adds a recent combined list, but NOT the web's weekly combined chart (the duration chart and calendar already cover that read).
- Old `/workouts/[id]` deep links are not redirected — every internal reference is updated in Task 3 and no external links exist (single-user app, no universal links).

---

### Task 1: Running API fetchers + types (`lib/api.ts`)

Mirror the web twin exactly (read `../prog-strength-web/lib/api.ts` running section). One RN adaptation: `importRunningTcx` takes a `{uri, name, mimeType}` file descriptor (RN FormData), same pattern as `uploadAvatar`.

**Files:**
- Modify: `lib/api.ts` (append; current ~1328 lines)

- [ ] **Step 1: Append** — section divider `// --- Running activities ---` (match file style), then:

```typescript
export type ActivityType = "running" | "walking" | "cycling" | "other";
export type IngestSource = "manual_tcx" | "garmin_api";

export type RunningSession = {
  id: string;
  activity_type: ActivityType;
  ingest_source: IngestSource;
  source_activity_id: string;
  name: string | null;
  start_time: string; // RFC3339
  distance_meters: number;
  duration_seconds: number;
  avg_pace_sec_per_km: number | null;
  best_pace_sec_per_km: number | null;
  avg_heart_rate_bpm: number | null;
  max_heart_rate_bpm: number | null;
  total_calories: number | null;
  elevation_gain_meters: number | null;
  created_at: string;
  // Present on detail GET only.
  trackpoints?: RunningTrackpoint[];
};

export type RunningTrackpoint = {
  sequence: number;
  elapsed_seconds: number;
  distance_meters: number;
  heart_rate_bpm: number | null;
  pace_sec_per_km: number | null;
  elevation_meters: number | null;
};

export type RunningSessionsPage = {
  activities: RunningSession[];
  next_before: string | null;
};

export type RunningMetrics = {
  current_week: {
    distance_meters: number;
    run_count: number;
    delta_pct_vs_prior_week: number | null;
  };
  current_month: { distance_meters: number; run_count: number };
  recent_avg_pace_sec_per_km: number | null;
  all_time: { distance_meters: number; run_count: number };
};

/**
 * GET /activities. Range mode (since/until) and cursor mode
 * (limit/before) are mutually exclusive — the API rejects mixing them.
 */
export async function listRunningSessions(
  token: string,
  opts: { limit?: number; before?: string; since?: string; until?: string } = {},
): Promise<RunningSessionsPage> {
  const params = new URLSearchParams();
  if (opts.limit !== undefined) params.set("limit", String(opts.limit));
  if (opts.before) params.set("before", opts.before);
  if (opts.since) params.set("since", opts.since);
  if (opts.until) params.set("until", opts.until);
  const qs = params.toString();
  const resp = await fetch(`${config.apiUrl}/activities${qs ? `?${qs}` : ""}`, {
    headers: { Authorization: `Bearer ${token}` },
  });
  return unwrap<RunningSessionsPage>(resp, { activities: [], next_before: null });
}

/** GET /activities/{id} — includes trackpoints. */
export async function getRunningSession(
  token: string,
  id: string,
): Promise<RunningSession> {
  const resp = await fetch(`${config.apiUrl}/activities/${id}`, {
    headers: { Authorization: `Bearer ${token}` },
  });
  const got = await unwrap<RunningSession | null>(resp, null);
  if (!got) throw new Error("activity not found");
  return got;
}

/** GET /activities/running-metrics — fixed buckets, timeframe-independent. */
export async function getRunningMetrics(
  token: string,
  timezone: string,
): Promise<RunningMetrics> {
  const resp = await fetch(
    `${config.apiUrl}/activities/running-metrics?timezone=${encodeURIComponent(timezone)}`,
    { headers: { Authorization: `Bearer ${token}` } },
  );
  const got = await unwrap<RunningMetrics | null>(resp, null);
  if (!got) throw new Error("running metrics unavailable");
  return got;
}

/** PATCH /activities/{id} — rename. */
export async function renameRunningSession(
  token: string,
  id: string,
  name: string,
): Promise<RunningSession> {
  const resp = await fetch(`${config.apiUrl}/activities/${id}`, {
    method: "PATCH",
    headers: {
      "Content-Type": "application/json",
      Authorization: `Bearer ${token}`,
    },
    body: JSON.stringify({ name }),
  });
  const got = await unwrap<RunningSession | null>(resp, null);
  if (!got) throw new Error("API did not return the updated activity");
  return got;
}

/** DELETE /activities/{id}. */
export async function deleteRunningSession(
  token: string,
  id: string,
): Promise<void> {
  const resp = await fetch(`${config.apiUrl}/activities/${id}`, {
    method: "DELETE",
    headers: { Authorization: `Bearer ${token}` },
  });
  await unwrap<unknown>(resp, null);
}

/**
 * Thrown by importRunningTcx on a 409 — the activity was already
 * imported. Carries the existing activity's id so the UI can link to
 * it ("already in your log → View run").
 */
export class DuplicateRunError extends Error {
  existingActivityId: string;
  constructor(message: string, existingActivityId: string) {
    super(message);
    this.name = "DuplicateRunError";
    this.existingActivityId = existingActivityId;
  }
}

/** A local .tcx file picked for import (RN FormData part). */
export type PickedFile = {
  uri: string;
  name: string;
  mimeType: string;
};

/**
 * POST /activities/tcx as multipart/form-data under field `file`.
 * Server caps at 10 MB. 409 → DuplicateRunError. No Content-Type
 * header (fetch sets the multipart boundary; same rule as uploadAvatar).
 */
export async function importRunningTcx(
  token: string,
  file: PickedFile,
): Promise<RunningSession> {
  const form = new FormData();
  form.append("file", {
    uri: file.uri,
    name: file.name,
    type: file.mimeType,
  } as unknown as Blob);
  const resp = await fetch(`${config.apiUrl}/activities/tcx`, {
    method: "POST",
    headers: { Authorization: `Bearer ${token}` },
    body: form,
  });
  if (resp.status === 409) {
    let body: { error?: string; existing_activity_id?: string } = {};
    try {
      body = await resp.json();
    } catch {
      // fall through to defaults
    }
    throw new DuplicateRunError(
      body.error || "This activity is already in your log.",
      body.existing_activity_id ?? "",
    );
  }
  const got = await unwrap<RunningSession | null>(resp, null);
  if (!got) throw new Error("API did not return the imported activity");
  return got;
}
```

- [ ] **Step 2:** `npm run typecheck && npm run lint` → pass.
- [ ] **Step 3:** Commit: `feat(api): running activities fetchers — list/detail/metrics/rename/delete/tcx (web twin parity)`

---

### Task 2: Distance/pace/duration helpers (`lib/units.ts`)

Port the web's formatting exactly (`../prog-strength-web/lib/distance-unit-context.tsx` + `lib/format.ts`). Mobile reads the unit from `profile.distance_unit` at call sites — these are pure functions.

**Files:**
- Modify: `lib/units.ts` (append)

- [ ] **Step 1: Append:**

```typescript
// --- Distance / pace / run duration (running features) ---------------

export const METERS_PER_MILE = 1609.344;
export const METERS_PER_KM = 1000;
export const KM_PER_MILE = 1.609344;

export type DistanceUnit = "mi" | "km";

/** Meters → "5.0" in the given unit (always 1 decimal, no unit suffix). */
export function formatDistance(meters: number, unit: DistanceUnit): string {
  if (!Number.isFinite(meters)) return "—";
  const divisor = unit === "mi" ? METERS_PER_MILE : METERS_PER_KM;
  return (meters / divisor).toFixed(1);
}

/** sec/km → "m:ss" pace per the given unit; "—" for null/invalid. */
export function formatPace(secPerKm: number | null, unit: DistanceUnit): string {
  if (secPerKm == null || !Number.isFinite(secPerKm) || secPerKm <= 0) return "—";
  const secPerUnit = unit === "mi" ? secPerKm * KM_PER_MILE : secPerKm;
  const totalSeconds = Math.round(secPerUnit);
  const minutes = Math.floor(totalSeconds / 60);
  const seconds = totalSeconds % 60;
  return `${minutes}:${String(seconds).padStart(2, "0")}`;
}

/** Seconds → "m:ss" or "h:mm:ss" (zero-padded). Web lib/format.ts twin. */
export function formatRunDuration(seconds: number): string {
  if (!Number.isFinite(seconds) || seconds < 0) return "—";
  const total = Math.round(seconds);
  const h = Math.floor(total / 3600);
  const m = Math.floor((total % 3600) / 60);
  const s = total % 60;
  if (h > 0) return `${h}:${String(m).padStart(2, "0")}:${String(s).padStart(2, "0")}`;
  return `${m}:${String(s).padStart(2, "0")}`;
}

/** Fallback display name for an unnamed run, from its start time. */
export function runFallbackName(startTime: string): string {
  const d = new Date(startTime);
  if (Number.isNaN(d.getTime())) return "Run";
  return `${d.toLocaleDateString(undefined, { month: "short", day: "numeric" })} run`;
}
```

(Check the web's `runFallbackName` — if its format differs, match web and note it.)

- [ ] **Step 2:** typecheck + lint → pass. Commit: `feat(units): distance, pace, and run-duration formatting (web twin parity)`

---

### Task 3: Route restructure — Workouts tab → Activities tab

**Files:**
- Rename dir: `app/(tabs)/workouts/` → `app/(tabs)/activities/` (`git mv`)
- Move: `app/(tabs)/activities/[id].tsx` → `app/(tabs)/activities/workout/[id].tsx`
- Modify: `app/(tabs)/activities/_layout.tsx`, `app/(tabs)/_layout.tsx`, `app/index.tsx`, `app/login.tsx`, `app/(tabs)/calendar.tsx`, `app/(tabs)/activities/index.tsx`, `components/progress/progress-view.tsx`, `components/progress/prs-view.tsx`

- [ ] **Step 1:** `git mv "app/(tabs)/workouts" "app/(tabs)/activities"` then `mkdir` + `git mv` the detail screen to `workout/[id].tsx`.
- [ ] **Step 2:** Update the stack layout (`activities/_layout.tsx`): screens `index` (title "Activities"), `workout/[id]` (title "Workout"), and pre-register `run/[id]` (title "Run") — the screen file arrives in Task 6; expo-router tolerates declaring options for not-yet-created routes via the file's own Stack.Screen instead, so ONLY register `run/[id]` here if a placeholder file exists; otherwise skip and let Task 6's screen own its options. Keep the existing dark screenOptions and the index `headerRight: () => <AvatarButton />`.
- [ ] **Step 3:** Tab registration in `app/(tabs)/_layout.tsx`: `name="workouts"` → `name="activities"`, title "Activities", icon stays `barbell-outline` (recognizable; revisit later).
- [ ] **Step 4:** Update every hardcoded route ref (grep `"/workouts"`):
  - `app/index.tsx`: redirect → `/activities`
  - `app/login.tsx` (3×): `router.replace("/activities")`
  - `app/(tabs)/calendar.tsx`: `router.push(\`/activities/workout/${id}\`)`
  - `app/(tabs)/activities/index.tsx`: `router.push(\`/activities/workout/${item.id}\`)`
  - `components/progress/progress-view.tsx` (2×) and `components/progress/prs-view.tsx`: → `/activities/workout/${...}`
- [ ] **Step 5:** `npx expo customize tsconfig.json`? No — just run `npm run typecheck`; typedRoutes regenerate via `npx expo start` type generation or `npx expo export --no-build` is unnecessary: `expo lint`/`tsc` consume `.expo/types`, regenerate with `npx expo config --type prebuild >/dev/null 2>&1 || true` then `npx tsc --noEmit` — if route types are stale, run `npx expo start --no-dev --max-workers 1` briefly OR simply `npx expo-router typegen` if available (check `npx expo-router --help`). Whichever regenerates, typecheck must pass with the new routes.
- [ ] **Step 6:** Manual check in simulator: app boots to Activities tab, workout list renders, tapping a workout opens detail, calendar + progress links navigate.
- [ ] **Step 7:** Commit: `refactor(nav): workouts tab becomes activities — route group rename + ref updates`

---

### Task 4: Activities hub — segments + timeframe scaffold

**Files:**
- Modify: `app/(tabs)/activities/index.tsx` (becomes the hub shell)
- Create: `components/activities/workouts-view.tsx` (extraction of the current list)

- [ ] **Step 1:** Extract everything the current `index.tsx` renders (DurationChart header + weekly SectionList + pull-to-refresh + its `groupByWeek`/date helpers) into `components/activities/workouts-view.tsx` exporting `WorkoutsView()` — behavior-identical, including `router.push` to `/activities/workout/[id]`.
- [ ] **Step 2:** Rewrite `index.tsx` as the hub:

```tsx
// Activities hub — Overview | Workouts | Running, mirroring the web
// /activities page. The segment state is local (no URL backing on
// mobile); the timeframe pills drive Overview + Running (Workouts
// keeps its own weekly view — deliberate deviation from web).
import { useState } from "react";
import { View } from "react-native";
import { SegmentedControl } from "@/components/segmented-control";
import { WorkoutsView } from "@/components/activities/workouts-view";
import { OverviewView } from "@/components/activities/overview-view";
import { RunningView } from "@/components/activities/running-view";
import { TimeframePills, type Timeframe } from "@/components/activities/timeframe-pills";

type ActivityView = "overview" | "workouts" | "running";

export default function ActivitiesScreen() {
  const [view, setView] = useState<ActivityView>("overview");
  const [timeframe, setTimeframe] = useState<Timeframe>("30d");

  return (
    <View className="flex-1 bg-background">
      <View className="gap-3 px-4 pt-3">
        <SegmentedControl
          value={view}
          onChange={setView}
          segments={[
            { value: "overview", label: "Overview" },
            { value: "workouts", label: "Workouts" },
            { value: "running", label: "Running" },
          ]}
        />
        {view !== "workouts" && (
          <TimeframePills value={timeframe} onChange={setTimeframe} />
        )}
      </View>
      {view === "overview" && <OverviewView timeframe={timeframe} />}
      {view === "workouts" && <WorkoutsView />}
      {view === "running" && <RunningView timeframe={timeframe} />}
    </View>
  );
}
```

- [ ] **Step 3:** Create `components/activities/timeframe-pills.tsx`:

```tsx
// 7d/30d/90d/All pills shared by Overview + Running. Timeframe →
// half-open [since, until) RFC3339 bounds, matching the web hub.
import { Pressable, Text, View } from "react-native";

export type Timeframe = "7d" | "30d" | "90d" | "all";

const OPTIONS: readonly { value: Timeframe; label: string; days: number | null }[] = [
  { value: "7d", label: "7d", days: 7 },
  { value: "30d", label: "30d", days: 30 },
  { value: "90d", label: "90d", days: 90 },
  { value: "all", label: "All", days: null },
];

/** Bounds for API calls; both undefined for "all". */
export function timeframeBounds(tf: Timeframe): { since?: string; until?: string } {
  const days = OPTIONS.find((o) => o.value === tf)?.days ?? null;
  if (days === null) return {};
  return {
    since: new Date(Date.now() - days * 24 * 60 * 60 * 1000).toISOString(),
    until: new Date().toISOString(),
  };
}

export function TimeframePills({
  value,
  onChange,
}: {
  value: Timeframe;
  onChange: (v: Timeframe) => void;
}) {
  return (
    <View className="flex-row gap-2">
      {OPTIONS.map((o) => {
        const active = o.value === value;
        return (
          <Pressable
            key={o.value}
            onPress={() => onChange(o.value)}
            disabled={active}
            accessibilityRole="button"
            accessibilityState={{ selected: active }}
            hitSlop={6}
            className={`rounded-full border px-3 py-1.5 ${
              active ? "border-accent bg-accent/15" : "border-border bg-surface active:opacity-80"
            }`}
          >
            <Text className={`text-xs font-medium ${active ? "text-accent" : "text-muted"}`}>
              {o.label}
            </Text>
          </Pressable>
        );
      })}
    </View>
  );
}
```

For this task only, create minimal `overview-view.tsx` and `running-view.tsx` stubs (centered muted "Loading…"-free placeholder text) so the hub compiles — Tasks 5/8 replace them. (Check `text-accent` exists in tailwind config; otherwise use the accent text class the repo uses.)

- [ ] **Step 4:** typecheck + lint + simulator check (three segments switch; Workouts segment identical to the old tab). Commit: `feat(activities): hub shell — segments + timeframe pills, workouts view extracted`

---

### Task 5: Running view — metrics tiles + run list

**Files:**
- Replace stub: `components/activities/running-view.tsx`
- Reference: web `components/activities/running-view.tsx`

- [ ] **Step 1:** Implement. Requirements (full fidelity list — write idiomatic RN against the patterns in `workouts-view.tsx`):
  - Props: `{ timeframe: Timeframe }`; bounds via `timeframeBounds(timeframe)`.
  - On mount + timeframe change + pull-to-refresh: fetch `getRunningMetrics(token, Intl timezone)` and `listRunningSessions(token, bounds)` in parallel (`Promise.all`); 401 → clearToken + replace /login (copy the pattern from workouts-view).
  - Metrics banner: 2×2 grid of tiles — "This week" (`formatDistance` + unit label + delta% line, tone: green when > 0, muted when 0/null, red when < 0), "This month" (distance + "N runs"), "Avg pace (30d)" (`formatPace` + `/mi|/km`), "All time" (distance + "N runs"). Unit from `useProfile().profile?.distance_unit ?? "mi"`.
  - Run list: FlatList of rows sorted by start_time desc — each row: name (or `runFallbackName`), date (e.g. "Thu, Jun 11"), then `5.2 mi · 8:42 /mi · 45:12 · 152 bpm` (omit null HR). Row press → `router.push(\`/activities/run/${id}\`)` (route exists after Task 6; use the same `as never` cast trick from Phase 1 ONLY if typecheck fails, with a TODO).
  - "Import .tcx" button in a list header row (UI only this task — disabled with a "soon" caption OR omit entirely; Task 7 adds the live button. Prefer omitting).
  - Empty state: "No runs in this window. Import a .tcx from the Running view." Loading: ActivityIndicator. Error: inline danger box.
- [ ] **Step 2:** typecheck + lint + simulator (metrics tiles render against prod data; rows navigate — to a 404 until Task 6, that's fine). Commit: `feat(running): running view — metrics tiles + run list`

---

### Task 6: Run detail screen + charts

**Files:**
- Create: `components/charts/ticks.ts` — extract `niceYTicks`/`niceXTicks` from `components/nutrition/bodyweight-chart.tsx`; refactor bodyweight-chart to import them (no behavior change).
- Create: `components/activities/run-metric-chart.tsx` — ONE parameterized SVG chart used three times.
- Create: `app/(tabs)/activities/run/[id].tsx`

- [ ] **Step 1: ticks extraction.** Move the two helpers verbatim into `components/charts/ticks.ts`, export them, import in bodyweight-chart. Typecheck must stay green.
- [ ] **Step 2: RunMetricChart.** Props:

```tsx
export function RunMetricChart({
  points,            // { x: number; y: number }[] — x = distance in DISPLAY unit
  height = 160,
  color,             // line/area color
  yFormat,           // (y: number) => string for axis + reference labels
  xLabel,            // "mi" | "km"
  yDomainPadding = 0.08,
  referenceY,        // optional horizontal dashed line (e.g. avg HR)
  referenceLabel,
  invertY = false,   // pace: lower is better → smaller values at top
}: { ... })
```

Implementation mirrors bodyweight-chart's structure: measure width via onLayout, PADDING constants, xScale/yScale, `niceYTicks(yMin, yMax, 4)` grid lines + labels, 2–3 x ticks via `niceXTicks`, a single Polyline for the series (skip null gaps by splitting into segments — points arrive pre-filtered), optional dashed reference Line + label. `invertY` flips the y mapping: `yScale = PADDING_TOP + ((v - yMin) / (yMax - yMin)) * plotH` when inverted. No tooltips/cursor in v1 (web's synced cursor is a desktop affordance — note as deviation).

- [ ] **Step 3: run/[id].tsx.** Requirements:
  - `useLocalSearchParams` id; fetch `getRunningSession`; loading/error states; Stack.Screen title = run name or fallback.
  - Stats grid (2 cols × 4 rows): Distance (`formatDistance` + unit), Avg Pace (`formatPace` + `/unit`), Avg HR ("N bpm"/"—"), Calories, Duration (`formatRunDuration`), Best Pace, Max HR, Elev Gain ("N m"/"—").
  - Charts from `trackpoints ?? []`, x = `distance_meters / (unit === "mi" ? METERS_PER_MILE : METERS_PER_KM)`:
    - Pace: y = `pace_sec_per_km * (unit === "mi" ? KM_PER_MILE : 1)`, filter null pace, `yFormat` = m:ss via formatPace-style math, `invertY` true, blue (#3b82f6).
    - Heart rate: filter null HR, referenceY = avg_heart_rate_bpm (when present) with label `Avg N bpm`, red-ish (#f87171), yFormat = `${Math.round(y)}`.
    - Elevation: filter null elevation, green (#34d399), yFormat = `${Math.round(y)} m`.
    - Each chart in a titled card; "No data" placeholder when its series is empty.
  - Header actions (headerRight in Stack.Screen options): an ellipsis button → `ActionSheetIOS` with Rename / Delete (destructive) / Cancel. Rename: `Alert.prompt` (iOS) prefilled with current name → `renameRunningSession`, update local state. Delete: confirm `Alert.alert` two-button → `deleteRunningSession` → `router.back()`.
- [ ] **Step 4:** typecheck + lint + simulator (open a run with trackpoints: three charts render, stats correct in mi; flip distance unit in Settings → km everywhere; rename + delete work). Commit: `feat(running): run detail — stats grid, pace/HR/elevation charts, rename/delete`

---

### Task 7: TCX import (expo-document-picker — NATIVE MODULE)

**Files:**
- Modify: `package.json`/lockfile (`npx expo install expo-document-picker`), `components/activities/running-view.tsx`

- [ ] **Step 1:** `npx expo install expo-document-picker` (no config plugin needed — it has no permissions on iOS for the system picker; verify with `npx expo config --type prebuild` that nothing new is required).
- [ ] **Step 2:** Add an "Import .tcx" toolbar button at the top of RunningView:

```tsx
async function importTcx() {
  const res = await DocumentPicker.getDocumentAsync({
    type: ["application/octet-stream", "application/xml", "text/xml", "*/*"],
    copyToCacheDirectory: true,
  });
  if (res.canceled || !res.assets[0]) return;
  const asset = res.assets[0];
  if (asset.size != null && asset.size > 10 * 1024 * 1024) {
    setImportError("File is larger than the 10 MB limit.");
    return;
  }
  setImporting(true);
  setImportError(null);
  setDuplicateOf(null);
  try {
    const token = await getToken();
    if (!token) throw new Error("not signed in");
    const session = await importRunningTcx(token, {
      uri: asset.uri,
      name: asset.name ?? "activity.tcx",
      mimeType: asset.mimeType ?? "application/xml",
    });
    router.push(`/activities/run/${session.id}`);
    void load({ refreshing: false }); // refresh list + metrics behind the nav
  } catch (err) {
    if (err instanceof DuplicateRunError) {
      setDuplicateOf(err.existingActivityId);
      setImportError("This run is already in your log.");
    } else {
      setImportError(err instanceof Error ? err.message : String(err));
    }
  } finally {
    setImporting(false);
  }
}
```

Inline error box under the button; when `duplicateOf` is non-empty render a "View run →" link to `/activities/run/${duplicateOf}`. Button shows spinner while importing.
- [ ] **Step 3:** typecheck + lint. Simulator note: the system document picker works in the iOS simulator (Files app) — drag a .tcx onto the simulator to test, or defer the end-to-end import to on-device smoke. Commit: `feat(running): TCX import via document picker — duplicate handling, 10MB guard`

---

### Task 8: Overview segment

**Files:**
- Replace stub: `components/activities/overview-view.tsx`
- Reference: web `components/activities/activities-overview-view.tsx`

- [ ] **Step 1:** Implement. Props `{ timeframe: Timeframe }`. Fetch in parallel with the timeframe bounds: `listWorkouts(token, {...bounds, limit: 100})` + `listRunningSessions(token, bounds)`. Compute (client-side, mirroring web):
  - Hero tiles (2×2): Total time (workout spans where ended_at present + run duration_seconds, rendered "Xh Ym"), Sessions (count both), Workouts (count), Runs (count).
  - Secondary tiles (2×2): Volume (sum of `workoutVolume`-equivalent: Σ reps×weight per set converted to preferred weight unit — reuse `convertWeight`; label with unit), Distance (Σ distance_meters via formatDistance + unit), PRs (Σ personal_records_set.length), Avg session (total time / sessions, "Xm").
  - Recent list: merge workouts + runs, sort by performed_at/start_time desc, take 10; workout rows → `/activities/workout/[id]`, run rows → `/activities/run/[id]`; distinguish with the calendar's color convention (accent for lifts, teal for runs — small colored dot or left bar).
  - Pull-to-refresh, loading, error, empty states per repo patterns.
- [ ] **Step 2:** typecheck + lint + simulator (numbers sanity-check against the web Overview for the same window). Commit: `feat(activities): overview segment — combined stats + recent activity list`

---

### Task 9: Runs on the calendar

**Files:**
- Modify: `app/(tabs)/calendar.tsx`

- [ ] **Step 1:** Alongside `listWorkouts`, fetch `listRunningSessions(token, { since, until })` for the same grid bounds; bucket into `runsByDay` keyed by the existing `localDateKey` helper (key on `start_time` local day).
- [ ] **Step 2:** DayCell: render a second dot when the day has runs — teal (#2dd4bf / `bg-teal-400` if the token exists, else inline color) next to the existing workout dot. Both dots when both exist.
- [ ] **Step 3:** Agenda for the selected day: render run rows after workout rows — teal left bar, name/fallback, `start time · distance unit · pace /unit`; press → `/activities/run/[id]`.
- [ ] **Step 4:** typecheck + lint + simulator (a month with runs shows teal dots; agenda lists the run; navigation works). Commit: `feat(calendar): runs on the month grid + day agenda`

---

### Task 10: Final review + PR

- [ ] **Step 1:** Full `npm run typecheck && npm run lint`; `grep -rn '"/workouts' app components` returns nothing.
- [ ] **Step 2:** Final whole-branch review (controller dispatches reviewer over `main...HEAD`).
- [ ] **Step 3:** Manual smoke checklist (owner, on device after native rebuild): Activities tab boots to Overview; segments switch; timeframe pills refetch; run list + detail charts against real TCX data; import a .tcx from Files/Garmin share; duplicate import shows "View run"; calendar teal dots; unit flip mi↔km updates distance/pace everywhere.
- [ ] **Step 4:** Push `feat/mobile-parity-phase-2`, open PR. PR body MUST flag: **expo-document-picker is a new native module → after merge, run the release workflow with `build=true, profile=production` and update via TestFlight before expecting the new code on-device** (OTA alone will not deliver a bundle targeting the new runtime).

---

## Self-review notes (applied)

- **SOW Phase 2 coverage:** Activities hub w/ segments ✓ (T4), Overview ✓ (T8), run list + metrics ✓ (T5), run detail charts ✓ (T6), TCX import + 409 UX ✓ (T7), calendar runs ✓ (T9), timeframe pills ✓ (T4). SOW's "/running/[id]" route became `/activities/run/[id]` (route group keeps detail screens under their tab stack so the tab bar stays visible — same reasoning as the existing workout detail).
- **Type consistency:** `Timeframe`/`timeframeBounds` defined T4, consumed T5/T8; `PickedFile`/`DuplicateRunError` defined T1, consumed T7; `RunMetricChart` props defined and consumed in T6 only; `formatDistance/formatPace/formatRunDuration/runFallbackName` defined T2, consumed T5/T6/T8/T9.
- **Known judgment calls:** Workouts segment unfiltered by timeframe (deviation noted); no chart cursor/tooltips on mobile v1; no run-list cursor pagination (range mode only, matching web's timeframe behavior).
