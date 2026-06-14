# Mobile Parity Phase 4: Chat Photo, Nutrition, Bodyweight Goal, Exercises Catalog — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement Phase 4 (final parity phase) of `sows/mobile-feature-parity-and-testflight.md`: photo meal logging in chat, nutrition custom meals + direct-log + entry editing, bodyweight goal + entry editing, and an exercises catalog screen.

**Architecture:** Pure-JS phase. `expo-image-picker` and its camera/photo permission strings are already in the binary (added Phase 1 for avatars), so `runtimeVersion` stays `"1"` and every merge ships OTA — the native fingerprint guard (`npm run fingerprint:check`) MUST stay green. Web is the parity reference (`../prog-strength-web`): chat `page.tsx` + `lib/agent.ts`, `components/nutrition/*` + `components/quick-add-modal.tsx` + `log-item-modal.tsx` + `log-entry-edit-modal.tsx`, `app/(app)/bodyweight/page.tsx`, `app/(app)/exercises/page.tsx`.

**Verification policy:** No JS test runner (deliberate). Every task ends with `npm run typecheck && npm run lint` + listed simulator checks. PATH for shell work: `export PATH="$HOME/.nvm/versions/node/v24.15.0/bin:$PATH"`. Hooks run on commit (lint-staged + typecheck + commitlint — conventional, lower-case subjects ≤100 chars including any scope).

**Branch:** `feat/mobile-parity-phase-4` off up-to-date `main` in `prog-strength-mobile`.

**Key facts established during research (don't re-derive):**
- `streamChat(token, body: unknown)` in `lib/stream.ts` passes the body through opaquely; the agent `/chat` accepts `content: string | ContentBlock[]`. So the image turn sends `ContentBlock[]` for the current message while prior turns and the persisted record stay strings.
- `expo-image-picker`'s `launchImageLibraryAsync({ base64: true })` returns `asset.base64` + `asset.mimeType` + `asset.fileSize` — no separate file read needed. Settings avatar flow is the picker reference (`app/settings.tsx`).
- Mobile chat in-state `Message` type is `{ role, content: string, tools? }`; the displayed message for an image turn stores web's `"[image attached] …"` placeholder.
- Modal reference pattern: `components/nutrition/macro-goals-sheet.tsx` (pageSheet Modal + KeyboardAvoidingView + header/scroll/footer). SegmentedControl for meal-type/tab pickers.
- Missing API fetchers (Task 1 adds, all exist on web twin): `createCustomNutritionLogEntry`, `updateBodyweightEntry`, `getBodyweightGoal`, `putBodyweightGoal`. Existing + reused: `updateNutritionLogEntry`, `createNutritionLogEntry`, `listExercises` + `useExerciseCatalog`.

**Deliberate deviations from web (don't "fix"):**
- Exercises catalog is a root-level stack screen pushed from a Settings row + workout-detail link (no sixth tab — five-tab cap), same pattern as Settings. Web has it as a tab.
- "Request an exercise" banner uses a mailto link to the beta-contact email if mobile `config` exposes one; if not, render the copy as plain text without the link and note it (don't invent a config field).

---

### Task 1: API fetchers + types (`lib/api.ts`)

Add the four missing fetchers + the chat `ContentBlock` type. Web twin (`../prog-strength-web/lib/api.ts` + `lib/agent.ts`) is the source of truth — transcribe exact paths, payload shapes, error messages, null/default handling, `encodeURIComponent` usage; follow web on any drift and report it.

**Files:**
- Modify: `lib/api.ts` (append types/fetchers in their respective sections)

- [ ] **Step 1: Custom nutrition log entry** (web `lib/api.ts` `CreateCustomLogEntryPayload` / `createCustomNutritionLogEntry`):

```typescript
export type CreateCustomLogEntryPayload = {
  name: string;
  calories: number;
  protein_g: number;
  fat_g: number;
  carbs_g: number;
  meal: MealType;
  consumed_at?: string; // RFC3339; server defaults to now
};

/** POST /nutrition-log/custom — a one-off meal with typed macros, no pantry item. */
export async function createCustomNutritionLogEntry(
  token: string,
  payload: CreateCustomLogEntryPayload,
): Promise<NutritionLogEntry> { /* fetch + unwrap, throw on null — copy web exactly */ }
```

- [ ] **Step 2: Bodyweight goal** (web `BodyweightGoal` / `getBodyweightGoal` / `putBodyweightGoal`):

```typescript
export type BodyweightGoal = {
  weight: number;
  unit: "lb" | "kg";
  created_at: string | null;
  updated_at: string | null;
};

/** GET /me/bodyweight-goal — empty (weight 0, null timestamps) when unset. */
export async function getBodyweightGoal(token: string): Promise<BodyweightGoal> { /* unwrap with web's exact default */ }

/** PUT /me/bodyweight-goal. */
export async function putBodyweightGoal(
  token: string,
  goal: { weight: number; unit: "lb" | "kg" },
): Promise<BodyweightGoal> { /* fetch + unwrap, throw on null */ }
```

The empty-state convention is `weight === 0` / null timestamps = "no goal set" — mirror web's default object exactly.

- [ ] **Step 3: Bodyweight entry update** (web `updateBodyweightEntry`):

```typescript
/** PUT /bodyweight/{id}. */
export async function updateBodyweightEntry(
  token: string,
  id: string,
  payload: { weight?: number; unit?: "lb" | "kg"; measured_at?: string },
): Promise<BodyweightEntry> { /* fetch + unwrap, throw on null, encodeURIComponent(id) */ }
```

- [ ] **Step 4: Chat ContentBlock** — colocate with the chat types (near `ChatMessage`):

```typescript
/**
 * A block in a multimodal chat turn. Text-only turns send a plain
 * string; a turn carrying a photo sends ContentBlock[] (image first,
 * then text). The agent /chat accepts `content: string | ContentBlock[]`.
 * Image bytes never persist — the stored record uses the
 * "[image attached] …" placeholder. Sibling: web lib/agent.ts.
 */
export type ContentBlock =
  | { type: "text"; text: string }
  | {
      type: "image";
      source: {
        type: "base64";
        media_type: "image/jpeg" | "image/png" | "image/webp";
        data: string;
      };
    };
```

- [ ] **Step 5:** typecheck + lint. Commit: `feat(api): custom-meal, bodyweight goal/update fetchers + chat content block (web twin parity)`

---

### Task 2: Photo meal logging in chat

Web reference: `../prog-strength-web/app/(app)/chat/page.tsx` (image accept/validation, payload construction, `[image attached]` persistence, preview chip). Mobile: `app/(tabs)/chat/index.tsx` (composer ~689–737, send ~390–581).

**Files:**
- Modify: `app/(tabs)/chat/index.tsx`

- [ ] **Step 1: Picker + validation.** Add a `pendingImage` state `{ uri: string; base64: string; mediaType: "image/jpeg" | "image/png" | "image/webp" } | null`. Image-attach `Pressable` (Ionicons `image`) between the mic button and TextInput. Handler:

```tsx
const MAX_IMAGE_BYTES = 5 * 1024 * 1024;
const ALLOWED: Record<string, "image/jpeg" | "image/png" | "image/webp"> = {
  "image/jpeg": "image/jpeg",
  "image/png": "image/png",
  "image/webp": "image/webp",
};

async function attachImage() {
  const res = await ImagePicker.launchImageLibraryAsync({
    mediaTypes: ["images"],
    quality: 0.7,
    base64: true,
  });
  if (res.canceled || !res.assets[0]) return;
  const a = res.assets[0];
  const media = ALLOWED[a.mimeType ?? ""];
  if (!media) {
    Alert.alert("Unsupported image", "Use JPG, PNG, or WebP.");
    return;
  }
  if ((a.fileSize ?? 0) > MAX_IMAGE_BYTES || !a.base64) {
    Alert.alert("Image too large", "Pick an image under 5 MB.");
    return;
  }
  setPendingImage({ uri: a.uri, base64: a.base64, mediaType: media });
}
```

(Confirm `ImagePicker` is imported — it's already a dependency; import as in settings.tsx. If a camera option is desired, an `ActionSheetIOS` Library/Camera chooser like settings' avatar menu is acceptable, but library-only is fine for v1 — note which you did.)

- [ ] **Step 2: Preview chip.** Above the composer row, when `pendingImage`: a small chip with a 40pt thumbnail (`Image source={{ uri: pendingImage.uri }}`), and an ✕ Pressable (hitSlop, ≥44pt zone) that clears `setPendingImage(null)`.

- [ ] **Step 3: Send-flow integration.** In `send()`:
  - Capture `const img = pendingImage;` at the top; allow send when `trimmed.length > 0 || img` (currently requires non-empty text — image-only with a space is valid, like web's `text: trimmed || " "`). Update the send-button `disabled` and TextInput gating accordingly.
  - The in-state displayed user message keeps the placeholder string: `const displayContent = img ? (trimmed ? \`[image attached] ${trimmed}\` : "[image attached]") : trimmed;` → `userMsg = { role: "user", content: displayContent }`.
  - Build the OUTGOING messages array separately: prior `messages` (string content) + the current turn's content. Current turn content is `img ? [{type:"image", source:{type:"base64", media_type: img.mediaType, data: img.base64}}, {type:"text", text: trimmed || " "}] : trimmed`. Pass `messages: outgoingMessages` to `streamChat` (the `body: unknown` signature already accepts mixed string/ContentBlock[] content).
  - Persisted turn (`appendChatTurn` user.content) uses `displayContent` (the placeholder string) — image bytes never reach the API.
  - Clear `setPendingImage(null)` right after building the request (same point text input is cleared), so a failed turn doesn't silently drop the image without feedback — match how the existing flow handles the text on error (if the existing flow restores text on error, restore the image too; otherwise clear and note it).

- [ ] **Step 4: Title generation.** If the turn is image-only (`trimmed` empty), the title call uses `displayContent` — fine (web does the same).

- [ ] **Step 5:** typecheck + lint + simulator (attach a photo → thumbnail chip → send → assistant responds to the image; chip clears; image-only send works; capped gate still blocks). Commit: `feat(chat): photo meal logging — attach image, multimodal turn, placeholder persistence`

---

### Task 3: Nutrition — custom meals + direct-log

Web reference: `../prog-strength-web/components/quick-add-modal.tsx` (custom tab), `log-item-modal.tsx` (direct-log). Mobile: `components/nutrition/quick-add-sheet.tsx`, `today-view.tsx`, `pantry-view.tsx`.

**Files:**
- Modify: `components/nutrition/quick-add-sheet.tsx`, `components/nutrition/today-view.tsx`, `components/nutrition/pantry-view.tsx`

- [ ] **Step 1: Custom-meal tab in quick-add.** The quick-add sheet gains a top SegmentedControl `Saved | Custom` (or "Pantry | Custom" — match web's tab labels). Saved = the existing pantry/recipe picker. Custom = a form: name (required) + calories + protein + carbs + fat (non-negative integers, the macro-goals-sheet's string-input + parse pattern) + the existing meal-type picker. On submit in Custom mode, call `createCustomNutritionLogEntry(token, { name, calories, protein_g, carbs_g, fat_g, meal, consumed_at })` (consumed_at computed the same way the existing quick-add computes it — today → now, else noon of the selected day). Validation copy mirrors web ("Name is required.", "Macros must be non-negative.").

- [ ] **Step 2: Direct-log from pantry/recipe rows.** In `pantry-view.tsx`, each pantry item and recipe row gains a "Log" action (alongside edit/delete) that opens a small log sheet: quantity (servings, > 0) + meal-type picker → `createNutritionLogEntry(token, { pantry_item_id | recipe_id, quantity, meal, consumed_at })`. Reuse/extract a shared `LogItemSheet` if the quantity+meal form would otherwise duplicate the quick-add's saved-mode form — judge; small duplication is acceptable. On success, toast/confirm and (if the today view is visible) it'll reflect on next focus.

- [ ] **Step 3:** typecheck + lint + simulator (Custom tab logs a one-off meal that appears in today's log; pantry/recipe row "Log" adds an entry). Commit: `feat(nutrition): custom one-off meals + direct-log from pantry and recipe rows`

---

### Task 4: Nutrition — log entry editing

Web reference: `../prog-strength-web/components/nutrition/log-entry-edit-modal.tsx`. Mobile: `components/nutrition/today-view.tsx` (entry rows, currently delete-only ~150–163).

**Files:**
- Modify: `components/nutrition/today-view.tsx`
- Create: `components/nutrition/log-entry-edit-sheet.tsx`

- [ ] **Step 1: Edit affordance.** Each log-entry row gains an Edit action (alongside the existing Delete) — an iOS action sheet on the row (Edit / Delete / Cancel) or an explicit edit icon; match the repo's row-action idiom (bodyweight rows use an action sheet — mirror it).
- [ ] **Step 2: Edit sheet** (`log-entry-edit-sheet.tsx`, macro-goals-sheet Modal shell). Two shapes, branched on entry kind (web does this):
  - Pantry/recipe entry: quantity (servings) + meal-type + consumed_at time-of-day (preserve the calendar day — a time picker or keep the existing day, just allow meal/quantity edits for v1 if a time picker is heavy; match web which edits quantity/meal/consumed_at).
  - Custom entry (`custom_meal_name` present / no pantry_item_id/recipe_id): name + four macros + meal.
  - Submit → `updateNutritionLogEntry(token, id, payload)` with only the changed-shape fields. On success close + refresh the day.
- [ ] **Step 3:** typecheck + lint + simulator (edit a pantry entry's quantity → totals update; edit a custom entry's macros → totals update). Commit: `feat(nutrition): edit logged entries — quantity/meal for items, macros for custom`

---

### Task 5: Bodyweight — goal + entry editing

Web reference: `../prog-strength-web/app/(app)/bodyweight/page.tsx` (goal modal, reference line, goal stat tile, entry edit). Mobile: `components/nutrition/bodyweight-view.tsx`, `components/nutrition/bodyweight-chart.tsx`.

**Files:**
- Modify: `components/nutrition/bodyweight-view.tsx`, `components/nutrition/bodyweight-chart.tsx`

- [ ] **Step 1: Goal fetch + modal.** On mount/focus, also `getBodyweightGoal(token)` (a 0/null result = unset). A "Goal" affordance (toolbar button or a goal stat tile that opens on tap) opens a macro-goals-sheet-style Modal: weight input + lb/kg toggle (seed from existing goal, else `profile?.weight_unit ?? "lb"`); submit → `putBodyweightGoal(token, { weight, unit })`, store result, close. Validate weight > 0.
- [ ] **Step 2: Goal on the chart.** Pass the goal (converted to the chart's display unit via `convertWeight`) into `bodyweight-chart.tsx`; render a dashed green (`#10b981`) horizontal reference line at the goal y, labeled `{weight} {unit}` (match the run-metric-chart referenceY rendering style and the in-domain guard — only draw when the goal falls within the padded y-domain, else skip the line so it never draws off-plot). Extend the y-domain to include the goal so it's visible when near the data edge.
- [ ] **Step 3: Goal stat tile.** Add a "Goal" tile to the existing stat tiles: value = goal in display unit or "Not set"; sublabel = `${|goal − currentAvg|} ${unit} to go` when both exist (web's AccentStatTile).
- [ ] **Step 4: Entry editing.** Bodyweight rows currently action-sheet to Delete (or have a delete affordance) — add Edit. Edit sheet (Modal): weight + unit + measured-at (date; a date/time picker or, if heavy, weight+unit edit with the existing date — match web which edits weight/unit/measured_at). Submit → `updateBodyweightEntry(token, id, payload)`; refresh.
- [ ] **Step 5:** typecheck + lint + simulator (set a goal → dashed line + tile appear; flip units → goal line/tile convert; edit an entry's weight → chart + stats update). Commit: `feat(bodyweight): goal line + stat tile, and entry editing`

---

### Task 6: Exercises catalog screen

Web reference: `../prog-strength-web/app/(app)/exercises/page.tsx`. Mobile: `components/exercise-catalog-context.tsx` (already provides `useExerciseCatalog()` → `{ exercises, byID, loading, error, refresh }`, fetched once per session). No screen exists yet.

**Files:**
- Create: `app/exercises.tsx` (root-level stack screen, like `app/settings.tsx`)
- Create: `components/muscle-group-pill.tsx`, `components/equipment-pill.tsx`
- Modify: `app/_layout.tsx` if needed for the route's Stack registration (settings.tsx is a sibling — follow how it's registered); `app/settings.tsx` (add an "Exercise catalog" row); `app/(tabs)/activities/workout/[id].tsx` (make exercise names tappable → `/exercises`, optional but per SOW)

- [ ] **Step 1: Pills.** Tiny presentational components — `MuscleGroupPill` and `EquipmentPill` (rounded bordered chips, `bg-surface border-border text-muted` style matching existing pills; muscle-group can use a subtle accent if web does). Both take a string label.
- [ ] **Step 2: Catalog screen** (`app/exercises.tsx`). Stack.Screen header "Exercises" (dark options like settings). `useExerciseCatalog()` for data. Search `TextInput` filtering on name + muscle_groups + equipment (web's exact predicate — substring match across all three, lower-cased). A–Z grouping (web's exact group/sort: sort by name, group by uppercased first letter, "#" fallback). `SectionList` (letters as section headers) or grouped `ScrollView`. Each row: tappable to expand → description + "Targets" muscle-group pills + equipment pills (web's ExerciseRow). 44pt rows, loading/error/empty states. The "request an exercise" banner: mailto link if mobile `config` exposes a contact email (grep `lib/config.ts`); else plain-text copy, noted.
- [ ] **Step 3: Entry points.** Settings gets an "Exercise catalog" row (in a list-style Pressable matching the settings layout) → `router.push("/exercises")`. In workout detail, make each exercise name a Pressable → `/exercises` (the catalog screen; v1 doesn't deep-link to a specific exercise — note that as a deferral). If typedRoutes flags `/exercises` before the file is registered, ensure the file exists first (it's created this task).
- [ ] **Step 4:** typecheck + lint + simulator (open catalog from Settings; search "barbell" filters; expand a row shows description + pills; workout-detail exercise tap opens catalog). Commit: `feat(exercises): searchable catalog screen with a-z grouping + entry points`

---

### Task 7: Final review + PR

- [ ] **Step 1:** Full `npm run typecheck && npm run lint && npm run format:check && npm run fingerprint:check` (the last MUST pass — Phase 4 is JS-only; a fingerprint change means a native module slipped in).
- [ ] **Step 2:** Final whole-branch review (controller dispatches a reviewer over `main...HEAD`).
- [ ] **Step 3:** Push `feat/mobile-parity-phase-4`, open PR (title ≤100 chars incl. scope). Note: pure JS → ships via OTA on merge; this completes SOW Phases 0–4 (full parity). List the manual smoke checklist.

---

## Self-review notes (applied)

- **SOW Phase 4 coverage:** photo meal logging ✓ (T2), custom meals ✓ (T3), direct-log ✓ (T3), nutrition entry editing ✓ (T4), bodyweight goal ✓ (T5), bodyweight entry editing ✓ (T5), exercises catalog ✓ (T6). The SOW's "edit flows for log entries where missing" → T4; "goal status tile" → T5.
- **Type consistency:** `CreateCustomLogEntryPayload`/`BodyweightGoal`/`ContentBlock` defined T1, consumed T2/T3/T4/T5; `updateBodyweightEntry` T1 → T5.
- **Native-module check:** every feature reuses `expo-image-picker` (already in the binary) or pure JS — fingerprint must not change. T7 gates on `fingerprint:check`. If any task is tempted to add a date-picker native module, prefer a JS approach or defer (note it) rather than breaking the OTA property.
- **Judgment calls:** image attach is library-only for v1 (camera is a small follow-up via ActionSheet, perms already present); log-entry/bodyweight edit may keep the existing date and edit only quantity/meal/macros/weight if a native date-time picker would be required — match web's field set where a JS control suffices, defer the date-time edit with a note otherwise.
