# Mobile Parity Phase 0+1: TestFlight Repo Work + Settings/Units/Usage Foundations — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement Phases 0–1 of `sows/mobile-feature-parity-and-testflight.md`: splash/runbook/release-pipeline repo work, then the settings screen, profile/units/usage contexts, header avatar entry point, and capped-aware chat composer in `prog-strength-mobile`.

**Architecture:** Port the web app's `ProfileProvider`/usage-context patterns to React Native (async Keychain token instead of localStorage, expo-router instead of next/navigation). Providers mount in the thin root layout and fetch lazily; the existing `(tabs)` auth gate triggers the first fetch. Settings is a root-level Stack screen pushed from a header avatar button, so the tab bar's five slots stay untouched.

**Tech Stack:** Expo SDK 55 / RN 0.83 / React 19, expo-router, NativeWind (tokens: `bg-background`, `bg-surface`, `border-border`, `text-foreground`, `text-muted`, `text-danger`, `bg-accent`), expo-secure-store, expo-image-picker (new), expo-splash-screen (new).

**Verification policy:** This repo has no JS test runner (deliberate — the SOW sets manual on-device smoke as the bar; automated test infra is a non-goal). Every task therefore ends with `npm run typecheck && npm run lint` plus a listed manual simulator check, instead of unit-test steps. Do not add a test framework in this plan.

**Branch:** Work in `prog-strength-mobile` on branch `feat/mobile-parity-phase-1` (create from up-to-date `main`). The repo root below is `prog-strength-mobile/` unless prefixed otherwise.

**Owner-only prerequisites (NOT agent tasks — listed for context):** `npx eas-cli login`, `npx eas-cli credentials` (Apple sign-in), App Store Connect app record, `EXPO_TOKEN` repo secret, first `eas build --platform ios --profile preview`. The agent tasks below are everything that lives in the repo.

---

### Task 1: Splash screen configuration (Phase 0)

The app icon (`assets/icon.png`, verified 1024×1024) exists and is wired in `app.json`, but there is no splash configuration — TestFlight builds would launch on a white flash, jarring in a forced-dark app.

**Files:**
- Modify: `app.json` (plugins array)
- Modify: `package.json` (via expo install)

- [ ] **Step 1: Install the plugin package**

Run: `npx expo install expo-splash-screen`
Expected: adds `expo-splash-screen` at the SDK-55-compatible version to `package.json`.

- [ ] **Step 2: Add the plugin to `app.json`**

In the `plugins` array, after `"expo-audio"`, add:

```json
[
  "expo-splash-screen",
  {
    "backgroundColor": "#0a0a0b",
    "image": "./assets/icon.png",
    "imageWidth": 160
  }
]
```

`#0a0a0b` matches the `background` token in `tailwind.config.js` and the header/tab-bar colors in `app/(tabs)/_layout.tsx`.

- [ ] **Step 3: Verify config parses**

Run: `npx expo config --type prebuild | head -40`
Expected: JSON output includes the splash plugin block; no schema errors.

- [ ] **Step 4: Typecheck, lint, commit**

Run: `npm run typecheck && npm run lint`
Expected: both pass.

```bash
git add app.json package.json package-lock.json
git commit -m "feat(release): dark splash screen via expo-splash-screen"
```

---

### Task 2: Release runbook in README (Phase 0)

**Files:**
- Modify: `README.md` (append a new `## Releasing` section at the end)

- [ ] **Step 1: Append the runbook section**

Read `.github/workflows/release.yml` first and adjust the wording below ONLY if the workflow's trigger/input names differ from what's described (the workflow predates this plan: OTA `eas update` on push to `main`; manual `workflow_dispatch` with `build=true` and `profile` for native builds).

```markdown
## Releasing

Two release paths, both driven by `.github/workflows/release.yml`:

### JS-only changes (the common case)

Merge to `main`. CI runs `eas update --branch production` and the new
bundle is fetched on next app launch (~30s). No build, no review, no
version bump. Roll back by re-publishing the previous update:
`npx eas-cli update:republish --branch production`.

### Native changes (new native module, app.json plugin, SDK upgrade)

OTA cannot ship these — `runtimeVersion.policy: appVersion` pins JS
updates to the installed binary. Bump `version` in `app.json`, then run
the release workflow manually (Actions → release → Run workflow →
`build=true`, `profile=production`). EAS builds in the cloud and
submits to TestFlight Internal Testing (no App Review for internal
testers). Install the new build from the TestFlight app.

### One-time setup (already done — recorded for disaster recovery)

1. `npx eas-cli login` (Expo account `jwallace145`)
2. `npx eas-cli credentials` → EAS-managed iOS cert + provisioning
   profile against the Apple Developer account; App Store Connect app
   record for `fitness.progstrength.app`
3. Expo access token → `EXPO_TOKEN` secret on this repo
4. First build: `eas build --platform ios --profile preview`, install
   via the EAS link, smoke test; then `--profile production` to reach
   TestFlight
```

- [ ] **Step 2: Commit**

```bash
git add README.md
git commit -m "docs: release runbook — OTA vs native build paths, one-time setup"
```

---

### Task 3: Profile + usage API fetchers (`lib/api.ts`)

Mirror the web app's `/me` surface (`prog-strength-web/lib/api.ts:293-426`) into the mobile twin, adapting `uploadAvatar` for React Native's FormData (no `File` in RN — multipart parts are `{uri, name, type}` objects).

**Files:**
- Modify: `lib/api.ts` (append at end of file)
- Reference: `../prog-strength-web/lib/api.ts:293-426`

- [ ] **Step 1: Append types and fetchers**

```typescript
/* ------------------------------------------------------------------ */
/* Profile + usage (/me)                                              */
/* ------------------------------------------------------------------ */

/**
 * The resolved profile returned by the four /me profile endpoints
 * (GET/PATCH /me, POST/DELETE /me/avatar). `avatar_url` is resolved
 * server-side (presigned S3 GET, OAuth fallback, or null); `height_cm`
 * is the canonical centimeter value. Sibling: prog-strength-web
 * lib/api.ts ResolvedProfile.
 */
export type ResolvedProfile = {
  id: string;
  email: string;
  display_name: string;
  weight_unit: "lb" | "kg";
  distance_unit: "mi" | "km";
  height_cm: number | null;
  avatar_url: string | null;
};

/** Partial profile update for PATCH /me; omit a field to leave it unchanged. */
export type ProfilePatch = {
  display_name?: string;
  // Canonical centimeters; pass `null` to clear a previously-set height.
  height_cm?: number | null;
  weight_unit?: "lb" | "kg";
  distance_unit?: "mi" | "km";
};

/** GET /me. Throws if the response carries no user payload. */
export async function getMe(token: string): Promise<ResolvedProfile> {
  const resp = await fetch(`${config.apiUrl}/me`, {
    headers: { Authorization: `Bearer ${token}` },
  });
  const got = await unwrap<ResolvedProfile | null>(resp, null);
  if (!got) throw new Error("user not found");
  return got;
}

/** PATCH /me. Returns the updated profile so callers can re-seed state. */
export async function updateMe(
  token: string,
  patch: ProfilePatch,
): Promise<ResolvedProfile> {
  const resp = await fetch(`${config.apiUrl}/me`, {
    method: "PATCH",
    headers: {
      "Content-Type": "application/json",
      Authorization: `Bearer ${token}`,
    },
    body: JSON.stringify(patch),
  });
  const updated = await unwrap<ResolvedProfile | null>(resp, null);
  if (!updated) throw new Error("API did not return the updated user");
  return updated;
}

/**
 * A local image picked for avatar upload. RN's FormData takes a
 * {uri, name, type} part instead of a browser File — this is the
 * deliberate divergence from the web twin.
 */
export type PickedImage = {
  uri: string;
  // e.g. "image/jpeg" — the server accepts image/png, image/jpeg,
  // image/webp and enforces the 2 MB cap authoritatively.
  mimeType: string;
  fileName: string;
};

/**
 * POST /me/avatar as multipart/form-data under the field `file`.
 * No Content-Type header — fetch fills in the multipart boundary
 * itself, same as the web twin.
 */
export async function uploadAvatar(
  token: string,
  image: PickedImage,
): Promise<ResolvedProfile> {
  const form = new FormData();
  form.append("file", {
    uri: image.uri,
    name: image.fileName,
    type: image.mimeType,
  } as unknown as Blob);
  const resp = await fetch(`${config.apiUrl}/me/avatar`, {
    method: "POST",
    headers: { Authorization: `Bearer ${token}` },
    body: form,
  });
  const updated = await unwrap<ResolvedProfile | null>(resp, null);
  if (!updated) throw new Error("API did not return the updated profile");
  return updated;
}

/** DELETE /me/avatar. avatar_url falls back to the OAuth photo or null. */
export async function deleteAvatar(token: string): Promise<ResolvedProfile> {
  const resp = await fetch(`${config.apiUrl}/me/avatar`, {
    method: "DELETE",
    headers: { Authorization: `Bearer ${token}` },
  });
  const updated = await unwrap<ResolvedProfile | null>(resp, null);
  if (!updated) throw new Error("API did not return the updated profile");
  return updated;
}

/**
 * The authed user's daily AI-usage snapshot. Percent only — the API
 * deliberately omits dollar figures. `capped` gates the chat composer;
 * `resets_at` (RFC3339) is the user's next local midnight.
 */
export type UsageData = {
  percent_used: number;
  capped: boolean;
  resets_at: string;
};

/** GET /me/usage. `tz` is the user's IANA timezone for rollover anchoring. */
export async function getMyUsage(token: string, tz: string): Promise<UsageData> {
  const resp = await fetch(
    `${config.apiUrl}/me/usage?tz=${encodeURIComponent(tz)}`,
    { headers: { Authorization: `Bearer ${token}` } },
  );
  return unwrap<UsageData>(resp, {
    percent_used: 0,
    capped: false,
    resets_at: "",
  });
}
```

- [ ] **Step 2: Typecheck, lint, commit**

Run: `npm run typecheck && npm run lint`
Expected: both pass (the `as unknown as Blob` cast is what keeps RN FormData happy under DOM lib types — if lint flags it, prefer a single `// eslint-disable-next-line` over loosening tsconfig).

```bash
git add lib/api.ts
git commit -m "feat(api): /me profile, avatar, and usage fetchers (web twin parity)"
```

---

### Task 4: Profile context (`lib/profile-context.tsx`)

Port of `prog-strength-web/lib/profile-context.tsx` with three RN adaptations: `getToken()` is async (Keychain), 401 handling routes via expo-router, and `uploadAvatar` takes a `PickedImage`.

**Files:**
- Create: `lib/profile-context.tsx`
- Reference: `../prog-strength-web/lib/profile-context.tsx`

- [ ] **Step 1: Create the file**

```tsx
/**
 * Shared resolved-profile state for the authed app. The header avatar
 * button, the Settings screen, and unit-aware displays all read one
 * `GET /me` result instead of fetching independently — editing the
 * display name in Settings updates the header instantly.
 *
 * Mounted once in app/_layout.tsx. The provider does NOT fetch on
 * mount (the root layout renders before auth is known); the (tabs)
 * auth gate calls `refresh()` once a token is confirmed. Mirrors
 * prog-strength-web/lib/profile-context.tsx — keep the two in sync.
 */
import {
  createContext,
  useCallback,
  useContext,
  useState,
  type ReactNode,
} from "react";
import { useRouter } from "expo-router";
import {
  deleteAvatar,
  getMe,
  updateMe,
  uploadAvatar,
  type PickedImage,
  type ProfilePatch,
  type ResolvedProfile,
} from "@/lib/api";
import { clearToken, getToken } from "@/lib/auth";

type ProfileContextValue = {
  profile: ResolvedProfile | null;
  loading: boolean;
  error: string | null;
  refresh: () => Promise<void>;
  update: (patch: ProfilePatch) => Promise<ResolvedProfile>;
  uploadAvatar: (image: PickedImage) => Promise<ResolvedProfile>;
  removeAvatar: () => Promise<ResolvedProfile>;
};

const ProfileContext = createContext<ProfileContextValue | null>(null);

export function ProfileProvider({ children }: { children: ReactNode }) {
  const router = useRouter();
  const [profile, setProfile] = useState<ResolvedProfile | null>(null);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  // On a 401 from any profile call, drop the token and bounce to
  // /login. Returns true when it handled an auth failure.
  const handleAuthError = useCallback(
    async (err: unknown): Promise<boolean> => {
      const msg = err instanceof Error ? err.message : String(err);
      if (msg.toLowerCase().includes("401")) {
        await clearToken();
        router.replace("/login");
        return true;
      }
      return false;
    },
    [router],
  );

  const refresh = useCallback(async (): Promise<void> => {
    const token = await getToken();
    if (!token) return; // stay idle; the (tabs) auth gate owns routing
    setLoading(true);
    try {
      const data = await getMe(token);
      setProfile(data);
      setError(null);
    } catch (err: unknown) {
      if (await handleAuthError(err)) return;
      setError(err instanceof Error ? err.message : String(err));
    } finally {
      setLoading(false);
    }
  }, [handleAuthError]);

  // Shared write path: run the call, store the returned profile,
  // rethrow so the caller can surface the error inline.
  const mutate = useCallback(
    async (
      op: (token: string) => Promise<ResolvedProfile>,
    ): Promise<ResolvedProfile> => {
      const token = await getToken();
      if (!token) {
        router.replace("/login");
        throw new Error("not authenticated");
      }
      try {
        const next = await op(token);
        setProfile(next);
        setError(null);
        return next;
      } catch (err: unknown) {
        await handleAuthError(err);
        throw err;
      }
    },
    [handleAuthError, router],
  );

  const update = useCallback(
    (patch: ProfilePatch) => mutate((token) => updateMe(token, patch)),
    [mutate],
  );
  const doUploadAvatar = useCallback(
    (image: PickedImage) => mutate((token) => uploadAvatar(token, image)),
    [mutate],
  );
  const removeAvatar = useCallback(
    () => mutate((token) => deleteAvatar(token)),
    [mutate],
  );

  return (
    <ProfileContext.Provider
      value={{
        profile,
        loading,
        error,
        refresh,
        update,
        uploadAvatar: doUploadAvatar,
        removeAvatar,
      }}
    >
      {children}
    </ProfileContext.Provider>
  );
}

/** Throws outside <ProfileProvider> so a missing provider fails loudly. */
export function useProfile(): ProfileContextValue {
  const ctx = useContext(ProfileContext);
  if (!ctx) throw new Error("useProfile must be used within a ProfileProvider");
  return ctx;
}
```

- [ ] **Step 2: Typecheck, lint, commit**

Run: `npm run typecheck && npm run lint`
Expected: pass.

```bash
git add lib/profile-context.tsx
git commit -m "feat(profile): shared resolved-profile context (web port)"
```

---

### Task 5: Usage context (`lib/usage-context.tsx`)

**Files:**
- Create: `lib/usage-context.tsx`

- [ ] **Step 1: Create the file**

```tsx
/**
 * Daily AI-usage state (GET /me/usage). The Settings usage section and
 * the capped-aware chat composer read one snapshot. Same lazy-fetch
 * contract as profile-context: the (tabs) auth gate triggers the first
 * refresh; the chat screen re-refreshes after each completed turn so
 * the cap engages without an app restart.
 */
import {
  createContext,
  useCallback,
  useContext,
  useState,
  type ReactNode,
} from "react";
import { getMyUsage, type UsageData } from "@/lib/api";
import { getToken } from "@/lib/auth";

type UsageContextValue = {
  usage: UsageData | null;
  refresh: () => Promise<void>;
};

const UsageContext = createContext<UsageContextValue | null>(null);

export function UsageProvider({ children }: { children: ReactNode }) {
  const [usage, setUsage] = useState<UsageData | null>(null);

  const refresh = useCallback(async (): Promise<void> => {
    const token = await getToken();
    if (!token) return;
    try {
      const tz = Intl.DateTimeFormat().resolvedOptions().timeZone;
      setUsage(await getMyUsage(token, tz));
    } catch {
      // Usage is advisory UI — never block the app on it. A failed
      // fetch leaves the previous snapshot (or null = uncapped).
    }
  }, []);

  return (
    <UsageContext.Provider value={{ usage, refresh }}>
      {children}
    </UsageContext.Provider>
  );
}

export function useUsage(): UsageContextValue {
  const ctx = useContext(UsageContext);
  if (!ctx) throw new Error("useUsage must be used within a UsageProvider");
  return ctx;
}
```

- [ ] **Step 2: Typecheck, lint, commit**

Run: `npm run typecheck && npm run lint`

```bash
git add lib/usage-context.tsx
git commit -m "feat(usage): daily AI-usage context"
```

---

### Task 6: Mount providers; trigger first fetch from the auth gate

**Files:**
- Modify: `app/_layout.tsx`
- Modify: `app/(tabs)/_layout.tsx`

- [ ] **Step 1: Wrap the root Stack**

In `app/_layout.tsx`, add imports and wrap the existing `<Stack …/>`:

```tsx
import { ProfileProvider } from "@/lib/profile-context";
import { UsageProvider } from "@/lib/usage-context";
```

```tsx
<GestureHandlerRootView className="flex-1">
  <SafeAreaProvider>
    <ProfileProvider>
      <UsageProvider>
        <StatusBar style="light" />
        <Stack screenOptions={{ headerShown: false }} />
      </UsageProvider>
    </ProfileProvider>
  </SafeAreaProvider>
</GestureHandlerRootView>
```

Also update the file's header comment: the providers are mounted here (not in `(tabs)`) because the Settings screen lives outside the tab group; they stay idle until `refresh()` is called, so the layout keeps its "no SecureStore round trip at mount" property.

- [ ] **Step 2: Trigger the first fetch in the (tabs) auth gate**

In `app/(tabs)/_layout.tsx`, inside `TabsLayout`, extend the existing token-check effect (which sets `ready`) to kick both contexts exactly once when a token is confirmed:

```tsx
import { useProfile } from "@/lib/profile-context";
import { useUsage } from "@/lib/usage-context";
```

```tsx
const { profile, refresh: refreshProfile } = useProfile();
const { refresh: refreshUsage } = useUsage();

useEffect(() => {
  getToken().then((t) => {
    if (!t) {
      router.replace("/login");
    } else {
      setReady(true);
      // First fetch for the shared contexts. Guarded so a re-mount
      // of the tab tree doesn't double-fetch a warm profile.
      if (!profile) {
        void refreshProfile();
        void refreshUsage();
      }
    }
  });
  // eslint-disable-next-line react-hooks/exhaustive-deps -- run once per mount
}, [router]);
```

- [ ] **Step 3: Verify in simulator**

Run: `npm run ios` (or `npx expo start` + simulator).
Expected: app boots to tabs as before; no redbox; network inspector (or API logs) shows exactly one `GET /me` and one `GET /me/usage` after launch.

- [ ] **Step 4: Typecheck, lint, commit**

```bash
git add app/_layout.tsx "app/(tabs)/_layout.tsx"
git commit -m "feat(profile): mount profile/usage providers; first fetch from auth gate"
```

---

### Task 7: Install expo-image-picker (native module boundary)

**Files:**
- Modify: `package.json`, `app.json`

- [ ] **Step 1: Install**

Run: `npx expo install expo-image-picker`

- [ ] **Step 2: Add the plugin with permission copy**

In `app.json` plugins (after the splash entry from Task 1):

```json
[
  "expo-image-picker",
  {
    "photosPermission": "Prog Strength uses your photo library to set a profile picture and to log meals from photos.",
    "cameraPermission": "Prog Strength uses the camera to log meals from photos."
  }
]
```

The camera string lands now (Phase 4's photo meal logging reuses this module — that's the SOW's "one native rebuild" plan), but Phase 1 only uses the library picker.

- [ ] **Step 3: Verify, commit**

Run: `npx expo config --type prebuild | grep -A3 image-picker` — plugin present.
Run: `npm run typecheck && npm run lint`

```bash
git add app.json package.json package-lock.json
git commit -m "feat(native): expo-image-picker plugin (avatar now, photo meals in phase 4)"
```

**Note for the executor:** from this commit on, a custom dev client / new native build is required to exercise the picker on device (`eas build --profile development` or simulator dev client). Pure-JS work in later tasks still runs in the existing client.

---

### Task 8: Avatar header button (`components/avatar-button.tsx`)

**Files:**
- Create: `components/avatar-button.tsx`
- Modify: `app/(tabs)/_layout.tsx` (Tabs screenOptions)
- Modify: `app/(tabs)/workouts/_layout.tsx`, `app/(tabs)/chat/_layout.tsx` (stack headers)

- [ ] **Step 1: Create the component**

```tsx
// Header avatar → Settings. 28pt circle showing the uploaded/OAuth
// avatar, or the user's initials as fallback. Rendered as headerRight
// on every tab header — the iOS-conventional account entry point,
// mirroring the web sidebar's account anchor (Settings gets no tab;
// the five slots are taken).
import { Image, Pressable, Text } from "react-native";
import { useRouter } from "expo-router";
import { useProfile } from "@/lib/profile-context";

function initials(name: string): string {
  const parts = name.trim().split(/\s+/).filter(Boolean);
  if (parts.length === 0) return "?";
  const first = parts[0][0] ?? "";
  const last = parts.length > 1 ? (parts[parts.length - 1][0] ?? "") : "";
  return (first + last).toUpperCase();
}

export function AvatarButton() {
  const router = useRouter();
  const { profile } = useProfile();

  return (
    <Pressable
      onPress={() => router.push("/settings")}
      accessibilityRole="button"
      accessibilityLabel="Settings"
      // 28pt visual + hitSlop ≥ the SOW's 44pt touch-target floor.
      hitSlop={10}
      className="mr-4 h-7 w-7 items-center justify-center overflow-hidden rounded-full border border-border bg-surface active:opacity-80"
    >
      {profile?.avatar_url ? (
        <Image
          source={{ uri: profile.avatar_url }}
          className="h-7 w-7"
          accessibilityIgnoresInvertColors
        />
      ) : (
        <Text className="text-[10px] font-semibold text-muted">
          {profile ? initials(profile.display_name) : "…"}
        </Text>
      )}
    </Pressable>
  );
}
```

- [ ] **Step 2: Wire into the three Tabs-owned headers**

In `app/(tabs)/_layout.tsx`, import `AvatarButton` and add to the shared `screenOptions`:

```tsx
headerRight: () => <AvatarButton />,
```

(Chat and Workouts hide the Tabs header, so this lands on Calendar, Nutrition, Progress.)

- [ ] **Step 3: Wire into the two stack-owned headers**

In `app/(tabs)/workouts/_layout.tsx` and `app/(tabs)/chat/_layout.tsx`, import `AvatarButton` and add `headerRight: () => <AvatarButton />` to the **index** screen's options only (detail/history screens keep their back-button-focused headers; chat's layout already composes headerRight buttons — append the avatar after the existing New chat/History buttons inside the same fragment).

- [ ] **Step 4: Verify in simulator**

Expected: every tab's header shows the avatar circle (initials until an avatar exists); tapping it navigates to an (as-yet-missing) `/settings` route — a 404/unmatched screen is expected until Task 9; navigation itself must not crash.

- [ ] **Step 5: Typecheck, lint, commit**

```bash
git add components/avatar-button.tsx "app/(tabs)/_layout.tsx" "app/(tabs)/workouts/_layout.tsx" "app/(tabs)/chat/_layout.tsx"
git commit -m "feat(settings): header avatar button on all tab headers"
```

---

### Task 9: Settings screen (`app/settings.tsx`)

The big one. Root-level Stack screen (outside `(tabs)`, so it presents full-screen over the tab bar with a back button). Sections: Profile (display name, height, avatar), Units (distance, weight), Usage.

**Files:**
- Create: `app/settings.tsx`
- Create: `components/settings/unit-toggle.tsx`
- Reference: `../prog-strength-web/app/(app)/settings/page.tsx` (validation + copy), `components/nutrition/macro-goals-sheet.tsx` (form patterns)

- [ ] **Step 1: Create the unit-toggle component**

```tsx
// Two-option unit selector rendered as joined buttons (lb|kg, mi|km).
// A radio group on web; on mobile a segmented pair is the idiom and
// keeps each target comfortably above 44pt.
import { Pressable, Text, View } from "react-native";

export function UnitToggle<T extends string>({
  options,
  value,
  onChange,
  disabled,
}: {
  options: readonly { value: T; label: string }[];
  value: T;
  onChange: (v: T) => void;
  disabled?: boolean;
}) {
  return (
    <View className="flex-row overflow-hidden rounded-md border border-border">
      {options.map((opt, i) => {
        const active = opt.value === value;
        return (
          <Pressable
            key={opt.value}
            onPress={() => onChange(opt.value)}
            disabled={disabled || active}
            accessibilityRole="button"
            accessibilityState={{ selected: active }}
            className={`min-h-11 flex-1 items-center justify-center px-4 py-2 ${
              active ? "bg-accent" : "bg-surface active:opacity-80"
            } ${i > 0 ? "border-l border-border" : ""}`}
          >
            <Text
              className={`text-sm font-medium ${
                active ? "text-accent-fg" : "text-foreground"
              }`}
            >
              {opt.label}
            </Text>
          </Pressable>
        );
      })}
    </View>
  );
}
```

(If `text-accent-fg` is not in `tailwind.config.js`, use the same class the macro-goals-sheet save button uses for its label — keep token usage consistent with the existing sheet.)

- [ ] **Step 2: Create the settings screen**

```tsx
// Settings — profile, units, usage. Parity with web /settings, laid
// out as grouped cards. Pushed from the header AvatarButton; lives
// outside (tabs) so it presents full-screen with a back affordance.
import { useEffect, useState } from "react";
import {
  ActionSheetIOS,
  ActivityIndicator,
  Alert,
  Image,
  Platform,
  Pressable,
  ScrollView,
  Text,
  TextInput,
  View,
} from "react-native";
import { Stack } from "expo-router";
import * as ImagePicker from "expo-image-picker";
import { useProfile } from "@/lib/profile-context";
import { useUsage } from "@/lib/usage-context";

// Mirrors the API's display-name cap; server is authoritative.
const MAX_DISPLAY_NAME = 60;
const CM_PER_INCH = 2.54;

export default function SettingsScreen() {
  const { profile, loading, error, update, uploadAvatar, removeAvatar } =
    useProfile();
  const { usage, refresh: refreshUsage } = useUsage();

  const [name, setName] = useState("");
  const [height, setHeight] = useState(""); // display unit (in or cm)
  const [saving, setSaving] = useState(false);
  const [formError, setFormError] = useState<string | null>(null);
  const [avatarBusy, setAvatarBusy] = useState(false);

  const heightUnit = profile?.distance_unit === "km" ? "cm" : "in";

  // Seed the form whenever the resolved profile (re)arrives.
  useEffect(() => {
    if (!profile) return;
    setName(profile.display_name);
    if (profile.height_cm === null) {
      setHeight("");
    } else {
      setHeight(
        heightUnit === "cm"
          ? String(Math.round(profile.height_cm))
          : (profile.height_cm / CM_PER_INCH).toFixed(1),
      );
    }
  }, [profile, heightUnit]);

  useEffect(() => {
    void refreshUsage();
  }, [refreshUsage]);

  async function saveProfile() {
    const trimmed = name.trim();
    if (!trimmed) {
      setFormError("Display name is required.");
      return;
    }
    if (trimmed.length > MAX_DISPLAY_NAME) {
      setFormError(`Display name must be ≤ ${MAX_DISPLAY_NAME} characters.`);
      return;
    }
    let height_cm: number | null = null;
    if (height.trim() !== "") {
      const n = Number(height);
      if (!Number.isFinite(n) || n <= 0) {
        setFormError("Height must be a positive number.");
        return;
      }
      height_cm = heightUnit === "cm" ? Math.round(n) : Math.round(n * CM_PER_INCH);
    }
    setSaving(true);
    setFormError(null);
    try {
      await update({ display_name: trimmed, height_cm });
    } catch (err) {
      setFormError(err instanceof Error ? err.message : String(err));
    } finally {
      setSaving(false);
    }
  }

  async function changeAvatar() {
    const result = await ImagePicker.launchImageLibraryAsync({
      mediaTypes: ["images"],
      allowsEditing: true,
      aspect: [1, 1],
      quality: 0.8,
    });
    if (result.canceled || !result.assets[0]) return;
    const asset = result.assets[0];
    setAvatarBusy(true);
    try {
      await uploadAvatar({
        uri: asset.uri,
        mimeType: asset.mimeType ?? "image/jpeg",
        fileName: asset.fileName ?? "avatar.jpg",
      });
    } catch (err) {
      Alert.alert("Avatar", err instanceof Error ? err.message : String(err));
    } finally {
      setAvatarBusy(false);
    }
  }

  function avatarMenu() {
    const doRemove = async () => {
      setAvatarBusy(true);
      try {
        await removeAvatar();
      } catch (err) {
        Alert.alert("Avatar", err instanceof Error ? err.message : String(err));
      } finally {
        setAvatarBusy(false);
      }
    };
    if (Platform.OS === "ios") {
      const hasUpload = Boolean(profile?.avatar_url);
      ActionSheetIOS.showActionSheetWithOptions(
        {
          options: hasUpload
            ? ["Choose photo", "Remove photo", "Cancel"]
            : ["Choose photo", "Cancel"],
          destructiveButtonIndex: hasUpload ? 1 : undefined,
          cancelButtonIndex: hasUpload ? 2 : 1,
        },
        (i) => {
          if (i === 0) void changeAvatar();
          if (hasUpload && i === 1) void doRemove();
        },
      );
    } else {
      void changeAvatar();
    }
  }

  function resetCountdown(): string | null {
    if (!usage?.resets_at) return null;
    const ms = Date.parse(usage.resets_at) - Date.now();
    if (!Number.isFinite(ms) || ms <= 0) return null;
    const h = Math.floor(ms / 3_600_000);
    const m = Math.floor((ms % 3_600_000) / 60_000);
    return h > 0 ? `${h}h ${m}m` : `${m}m`;
  }

  if (loading && !profile) {
    return (
      <View className="flex-1 items-center justify-center bg-background">
        <Stack.Screen options={{ title: "Settings", headerShown: true }} />
        <ActivityIndicator />
      </View>
    );
  }

  const countdown = resetCountdown();

  return (
    <ScrollView
      className="flex-1 bg-background"
      contentContainerClassName="gap-6 px-4 py-4 pb-12"
    >
      <Stack.Screen
        options={{
          title: "Settings",
          headerShown: true,
          headerStyle: { backgroundColor: "#0a0a0b" },
          headerTitleStyle: { color: "#fafafa" },
          headerTintColor: "#fafafa",
          headerShadowVisible: false,
        }}
      />

      {/* ---- Profile ---- */}
      <Section title="Profile">
        <Pressable
          onPress={avatarMenu}
          disabled={avatarBusy}
          accessibilityRole="button"
          accessibilityLabel="Change profile photo"
          className="min-h-11 flex-row items-center gap-3 active:opacity-80"
        >
          <View className="h-14 w-14 items-center justify-center overflow-hidden rounded-full border border-border bg-background">
            {avatarBusy ? (
              <ActivityIndicator />
            ) : profile?.avatar_url ? (
              <Image
                source={{ uri: profile.avatar_url }}
                className="h-14 w-14"
                accessibilityIgnoresInvertColors
              />
            ) : (
              <Text className="text-lg font-semibold text-muted">
                {profile?.display_name?.[0]?.toUpperCase() ?? "?"}
              </Text>
            )}
          </View>
          <View>
            <Text className="text-sm text-foreground">Profile photo</Text>
            <Text className="text-xs text-muted">
              {profile?.avatar_url ? "Tap to change or remove" : "Tap to add"}
            </Text>
          </View>
        </Pressable>

        <Field label="Display name">
          <TextInput
            value={name}
            onChangeText={setName}
            maxLength={MAX_DISPLAY_NAME}
            autoCapitalize="words"
            editable={!saving}
            className="min-h-11 rounded-md border border-border bg-background px-3 py-2 text-sm text-foreground"
          />
        </Field>

        <Field label={`Height (${heightUnit})`}>
          <TextInput
            value={height}
            onChangeText={setHeight}
            keyboardType="decimal-pad"
            editable={!saving}
            placeholder="Not set"
            placeholderTextColor="#a1a1aa"
            className="min-h-11 rounded-md border border-border bg-background px-3 py-2 text-sm tabular-nums text-foreground"
          />
        </Field>

        {(formError ?? error) && (
          <View className="rounded-md border border-danger/40 bg-danger/10 px-3 py-2">
            <Text className="text-xs text-danger">{formError ?? error}</Text>
          </View>
        )}

        <Pressable
          onPress={saveProfile}
          disabled={saving}
          accessibilityRole="button"
          className="min-h-11 items-center justify-center rounded-md bg-accent px-4 py-2 active:opacity-80 disabled:opacity-50"
        >
          {saving ? (
            <ActivityIndicator color="#fff" />
          ) : (
            <Text className="text-sm font-medium text-accent-fg">Save</Text>
          )}
        </Pressable>
      </Section>

      {/* ---- Units ---- */}
      <Section title="Units">
        <Field label="Distance">
          <UnitToggle
            options={[
              { value: "mi", label: "Miles" },
              { value: "km", label: "Kilometers" },
            ]}
            value={profile?.distance_unit ?? "mi"}
            onChange={(v) => void update({ distance_unit: v })}
          />
        </Field>
        <Field label="Weight">
          <UnitToggle
            options={[
              { value: "lb", label: "Pounds" },
              { value: "kg", label: "Kilograms" },
            ]}
            value={profile?.weight_unit ?? "lb"}
            onChange={(v) => void update({ weight_unit: v })}
          />
        </Field>
      </Section>

      {/* ---- Usage ---- */}
      <Section title="Daily AI usage">
        <View className="h-2 overflow-hidden rounded-full bg-background">
          <View
            className={`h-2 rounded-full ${usage?.capped ? "bg-danger" : "bg-accent"}`}
            style={{ width: `${Math.min(100, usage?.percent_used ?? 0)}%` }}
          />
        </View>
        <Text className="text-xs text-muted">
          {usage
            ? `${Math.round(usage.percent_used)}% of today's allowance used` +
              (countdown ? ` · resets in ${countdown}` : "")
            : "Usage unavailable"}
        </Text>
        {usage?.capped && (
          <Text className="text-xs text-danger">
            Daily allowance reached — chat is paused until reset.
          </Text>
        )}
      </Section>

      <Text className="text-center text-xs text-muted">{profile?.email}</Text>
    </ScrollView>
  );
}

function Section({
  title,
  children,
}: {
  title: string;
  children: React.ReactNode;
}) {
  return (
    <View className="gap-3 rounded-lg border border-border bg-surface px-4 py-4">
      <Text className="text-[10px] font-semibold uppercase tracking-wider text-muted">
        {title}
      </Text>
      {children}
    </View>
  );
}

function Field({
  label,
  children,
}: {
  label: string;
  children: React.ReactNode;
}) {
  return (
    <View className="gap-1">
      <Text className="text-xs text-muted">{label}</Text>
      {children}
    </View>
  );
}
```

Import `UnitToggle` from `@/components/settings/unit-toggle`.

- [ ] **Step 3: Cross-check validation against the web settings page**

Read `../prog-strength-web/app/(app)/settings/page.tsx` and confirm: display-name cap (60), height bounds, and any copy worth mirroring. Adjust the constants/copy above if web differs — web is the parity source of truth.

- [ ] **Step 4: Verify in simulator**

Expected: avatar button → Settings pushes with back button; name + height edit and Save round-trips (header avatar initials update instantly); unit toggles persist across app restart (server-backed); usage bar renders; email shows at the foot.

- [ ] **Step 5: Typecheck, lint, commit**

```bash
git add app/settings.tsx components/settings/unit-toggle.tsx
git commit -m "feat(settings): settings screen — profile, units, usage"
```

---

### Task 10: Capped-aware chat composer

**Files:**
- Modify: `app/(tabs)/chat/index.tsx` (composer TextInput ≈ line 728; send-gating state ≈ line 101)

- [ ] **Step 1: Read the composer region**

Read `app/(tabs)/chat/index.tsx` around the send handler and the `TextInput` (the file is ~1065 lines; the composer gating comment "composer is gated on `!loading`" near line 101 and the `<TextInput` near line 728 are the anchors).

- [ ] **Step 2: Gate on `capped`**

Add to the component body:

```tsx
import { useUsage } from "@/lib/usage-context";
```

```tsx
const { usage, refresh: refreshUsage } = useUsage();
const capped = usage?.capped ?? false;
```

- Extend the existing send-enabled condition (wherever `loading` already gates sending) with `&& !capped`, and set `editable={!capped && /* existing condition */}` on the composer `TextInput`.
- Immediately above the composer row, render a banner when capped (reuse the danger-box pattern from macro-goals-sheet):

```tsx
{capped && (
  <View className="mx-4 mb-2 rounded-md border border-danger/40 bg-danger/10 px-3 py-2">
    <Text className="text-xs text-danger">
      Daily AI allowance reached — chat resumes after midnight.
    </Text>
  </View>
)}
```

- In the send flow's completion path (where the assistant turn finishes streaming and state settles — success **and** error paths), add `void refreshUsage();` so a turn that crosses the cap disables the composer for the next message.

- [ ] **Step 3: Verify in simulator**

Normal case: chat works as before, one extra `GET /me/usage` after each completed turn. Capped case (if the API exposes no test hook, temporarily hardcode `capped = true` locally to verify the banner + disabled input render, then revert): input disabled, banner visible, mic/send unresponsive.

- [ ] **Step 4: Typecheck, lint, commit**

```bash
git add "app/(tabs)/chat/index.tsx"
git commit -m "feat(chat): capped-aware composer — disable input at daily allowance"
```

---

### Task 11: Weight-unit display preference

Entries store their as-logged unit; the preference converts **at render time only** (the web rule — server data is never rewritten). Wire the existing displays to `profile.weight_unit`.

**Files:**
- Create: `lib/units.ts`
- Modify: `components/workout-row.tsx`, `app/(tabs)/workouts/[id].tsx`, `components/nutrition/bodyweight-view.tsx`, `components/nutrition/bodyweight-chart.tsx`
- Reference: `../prog-strength-web/lib/format.ts` (web's converters — mirror naming where it has equivalents)

- [ ] **Step 1: Create the helper**

```typescript
/**
 * Render-time weight conversion. Sets and bodyweight entries carry the
 * unit they were logged in; the user's preferred unit (profile
 * weight_unit) converts at display only — stored data is never
 * reinterpreted. Mirrors the web app's conversion rule in lib/format.ts.
 */
export const KG_PER_LB = 0.45359237;

export function convertWeight(
  value: number,
  from: "lb" | "kg",
  to: "lb" | "kg",
): number {
  if (from === to) return value;
  return from === "lb" ? value * KG_PER_LB : value / KG_PER_LB;
}

/** "225 lb" / "102.1 kg" in the preferred unit; ≤1 decimal, no trailing zero. */
export function formatWeight(
  value: number,
  unit: "lb" | "kg",
  preferred: "lb" | "kg",
): string {
  const converted = convertWeight(value, unit, preferred);
  const rounded = Math.round(converted * 10) / 10;
  const text = Number.isInteger(rounded) ? String(rounded) : rounded.toFixed(1);
  return `${text} ${preferred}`;
}
```

- [ ] **Step 2: Apply at the four display sites**

In each file, get the preference via `const { profile } = useProfile()` and `const preferred = profile?.weight_unit;`, then replace direct `"{weight} {unit}"` renderings with `formatWeight(weight, unit, preferred ?? unit)` (falling back to the as-logged unit while the profile loads). For the bodyweight chart, convert each point's y-value with `convertWeight` before charting and label the axis with the preferred unit. Read each file before editing; keep changes display-only — payloads sent to the API keep the unit the user logged in.

- [ ] **Step 3: Cross-check against web**

Read how `../prog-strength-web` renders workout set weights (search `weight_unit` usage in `components/workout-details.tsx` and the bodyweight page) and match its rounding/edge behavior if it differs from `formatWeight` above.

- [ ] **Step 4: Verify in simulator**

Flip Weight to kg in Settings → workout list/detail and bodyweight chart re-render in kg without refetch (context-driven); flip back to lb → original numbers return exactly (no drift from double conversion).

- [ ] **Step 5: Typecheck, lint, commit**

```bash
git add lib/units.ts components/workout-row.tsx "app/(tabs)/workouts/[id].tsx" components/nutrition/bodyweight-view.tsx components/nutrition/bodyweight-chart.tsx
git commit -m "feat(units): render-time weight conversion driven by profile preference"
```

---

### Task 12: Final verification + PR

- [ ] **Step 1: Full pass**

Run: `npm run typecheck && npm run lint`
Expected: clean.

- [ ] **Step 2: Manual smoke checklist (simulator)**

1. Cold launch → tabs render, exactly one `GET /me` + `GET /me/usage`.
2. Avatar button on all five tab headers → Settings.
3. Edit display name → Save → header initials change instantly.
4. Set height in `in`, flip distance to km → height field relabels `cm` with the converted value.
5. Flip weight to kg → workout detail + bodyweight chart convert; flip back, values restore.
6. Chat still streams; usage refreshes after a turn.
7. Splash: dark background, centered mark (dev client rebuild needed to see it).

- [ ] **Step 3: Push and open PR**

```bash
git push -u origin feat/mobile-parity-phase-1
gh pr create --title "feat: settings, profile/units/usage foundations + TestFlight repo work (parity phase 0+1)" --body "Implements Phases 0-1 of sows/mobile-feature-parity-and-testflight.md ..."
```

PR body should list the SOW reference, the native-module boundary (image-picker + splash → native rebuild required before the next TestFlight build), and the smoke checklist results.

---

## Self-review notes (already applied)

- **Spec coverage:** SOW Phase 0 repo work (splash ✓ Task 1, runbook ✓ Task 2, release.yml verified inside Task 2; icon already 1024×1024 — nothing to do). Phase 1 (settings ✓ T9, profile context ✓ T4, units contexts ✓ T4+T11 — `distance_unit` lives on the profile, so no separate distance context is needed until running views land in Phase 2, usage ✓ T5+T10, capped composer ✓ T10, avatar via image-picker ✓ T7+T9).
- **Type consistency:** `PickedImage`/`ProfilePatch` defined in T3, consumed in T4/T9; `UsageData` defined in T3, consumed in T5/T9/T10; `UnitToggle` defined in T9 step 1, used in T9 step 2.
- **Known judgment call:** providers fetch lazily (gate-triggered) rather than on-mount like web — RN's root layout renders pre-auth, and this preserves the root layout's documented "no SecureStore at mount" property.
