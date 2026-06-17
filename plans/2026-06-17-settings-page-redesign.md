# Settings Page Redesign — Grouped Save-Bar Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Rebuild `prog-strength-web`'s authed `/settings` page from a flat run of per-field-Save rows + silently-optimistic toggles into a Vercel/GitHub-style grouped-card layout with a single page-level draft and one sticky "Save changes" bar.

**Architecture:** A page-level `draft` object holds every editable profile field, diffed against an immutable `initial` baseline captured from `useProfile()`. Editing any field mutates `draft` only — nothing saves on its own. A sticky `SaveBar` appears when `dirtyCount > 0`, showing the dirty count, **Discard**, and a validation-gated **Save changes** that issues one `updateMe(patch)` of only the changed keys, then flashes "All changes saved ✓" and retracts. Avatar upload/remove and calendar connect/disconnect stay as immediate out-of-band actions (unchanged). No backend change; reuses the existing data layer (`useProfile`, `checkUsernameAvailable`, `getCalendarConnection`/`disconnectCalendar`, `useUsage`/`UsageBar`, `useDistanceUnit`, `useToast`).

**Tech Stack:** Next.js 16 App Router, React 19, TypeScript, Tailwind v4 with CSS-variable theme tokens (`var(--surface)`, `var(--accent)`, `var(--accent-fg)`, `var(--warning)`, `var(--danger)`, …), Vitest + Testing Library (jsdom).

---

## Design system conformance (read before any task)

The page conforms to [`design-system.md`](../design-system.md). Use **only** the existing CSS-variable tokens — never new hex:

- **Cards/panels:** `rounded-2xl`, hairline `border border-[var(--border)]`, `bg-[var(--surface)]`, soft shadow (`shadow-sm` / `shadow-lg` for the save bar). Bold section title over a divider (`border-b border-[var(--border)]`).
- **Accent (`var(--accent)` = `#8b7cf6`):** concentrated on the **Save changes** button and **active toggle pills** only. `var(--accent-fg)` for text on accent fills. `var(--accent-line)` for the save bar's top border, `var(--accent-soft)` where a soft fill is wanted.
- **Field surface:** rounded slate `bg-[var(--surface-2)]` or `bg-[var(--surface)]` fill with hairline `border-[var(--border)]`, `--foreground` text, `--muted` placeholder, `rounded-lg`. Focus: `outline-none` + accent border + accent-line ring (reuse the existing `inputClass` idiom from the current page).
- **Labels:** quiet field labels above the control — uppercase, `text-[11px]`, `font-semibold`, `tracking-wide`, `text-[var(--faint)]`.
- **Segmented toggle:** full-pill track `bg-[var(--surface)]`/`bg-[var(--background)]` hairline border; active segment `bg-[var(--accent)] text-[var(--accent-fg)]`; inactive `text-[var(--muted)]` → `hover:text-[var(--foreground)]`.
- **Usage thresholds:** accent (0–79%) → `var(--warning)` (≥80%) → `var(--danger)` (100%). Already implemented by `UsageBar`; reuse it verbatim.

## File structure

All paths under `prog-strength-web/`. Page-private code co-locates in `app/(app)/settings/_components/` (lowercase `.ts` = pure helper module, `.tsx` = component; tests co-locate as `*.test.ts(x)`), matching the timeline route's `statRow.ts` + `StatRow.tsx` convention.

- **Create** `app/(app)/settings/_components/draft.ts` — the `Draft` type + pure helpers: `initialsOf`, `runeLength`, `clampRunes`, `BIO_MAX_RUNES`, `CM_PER_INCH`, `heightToDisplay`, `displayToCm`, `USERNAME_RE`, `draftFromProfile`, `dirtyKeys`, `patchFromDraft`. One responsibility: the draft shape + the pure transforms between profile ⇄ draft ⇄ patch. No React.
- **Create** `app/(app)/settings/_components/draft.test.ts` — unit tests for those helpers.
- **Create** `app/(app)/settings/_components/primitives.tsx` — presentational `Card`, `Field`, `SegmentedToggle`. No data logic.
- **Create** `app/(app)/settings/_components/primitives.test.tsx` — render tests for the primitives.
- **Create** `app/(app)/settings/_components/SaveBar.tsx` — the sticky save bar (presentational; receives `dirtyCount`, `canSave`, `blockReason`, `savedFlash`, `saving`, `onSave`, `onDiscard`).
- **Create** `app/(app)/settings/_components/SaveBar.test.tsx` — render/interaction tests.
- **Create** `app/(app)/settings/_components/UsernameField.tsx` — the `@`-prefixed username input owning the debounced availability probe; reports its block state up via an `onBlockChange`/`onStatus` callback.
- **Create** `app/(app)/settings/_components/UsernameField.test.tsx` — probe-transition + block tests.
- **Rewrite** `app/(app)/settings/page.tsx` — the grouped-savebar page: draft state, save handler, the four cards, wiring the primitives + SaveBar + UsernameField, keeping avatar/calendar as immediate actions.
- **Rewrite** `app/(app)/settings/page.test.tsx` — the new save-model tests.

## Conventions every task must follow

- **Run from** `/workspace/prog-strength-web`.
- **TDD:** write the failing test first, watch it fail, implement, watch it pass.
- **Commit** at the end of each task in Conventional Commits format (`feat(settings): …`), scope `settings`. Do **not** use `--no-verify`. The husky `pre-commit` hook runs lint-staged (eslint --fix + prettier) and `tsc --noEmit`; let it run.
- **Token discipline:** no raw hex; only the CSS-variable tokens listed above.
- **Don't** touch `lib/api.ts`, the contexts, or any other route. No new endpoints/SDK methods.
- **React-hooks lint posture:** `purity`/`set-state-in-effect`/`immutability` are warnings; don't introduce *new* warnings. Effects that re-seed from props or run debounced probes mirror the existing page's patterns (the current `UsernameRow`/`HeightRow` use `// eslint-disable-next-line react-hooks/exhaustive-deps` where a deliberate partial dep list is needed — reuse that idiom rather than fighting the linter).
- **Per-task gate:** `npm run typecheck && npm run lint && npm run test -- <the touched test file>` green before committing. The final task runs the full `npm run test` + `npm run build`.

---

### Task 1: Draft helpers + types (`draft.ts`)

**Files:**
- Create: `app/(app)/settings/_components/draft.ts`
- Test: `app/(app)/settings/_components/draft.test.ts`

The pure core of the new save model. The `Draft` is a flat, all-string-or-enum projection of the editable profile fields, so diffing is a trivial `!==` per key. Height is held as the **display string** in the user's current unit (so typing round-trips without re-deriving), converted to canonical cm only at the patch boundary.

- [ ] **Step 1: Write the failing tests**

```ts
// app/(app)/settings/_components/draft.test.ts
/// <reference types="vitest/globals" />
import {
  initialsOf,
  runeLength,
  clampRunes,
  heightToDisplay,
  displayToCm,
  draftFromProfile,
  dirtyKeys,
  patchFromDraft,
  USERNAME_RE,
  BIO_MAX_RUNES,
  type Draft,
} from "./draft";
import type { ResolvedProfile } from "@/lib/api";

function profile(over: Partial<ResolvedProfile> = {}): ResolvedProfile {
  return {
    id: "u1",
    email: "lifter@example.com",
    display_name: "Sam",
    weight_unit: "lb",
    distance_unit: "mi",
    height_cm: 180,
    avatar_url: null,
    timezone: "America/Denver",
    calendar_default_detail: "time_block",
    username: "sam",
    bio: null,
    ...over,
  };
}

describe("initialsOf", () => {
  it("takes the first two letters of a single-word name", () => {
    expect(initialsOf("Sam")).toBe("SA");
  });
  it("takes first+last initials for a multi-word name", () => {
    expect(initialsOf("Sam Stone")).toBe("SS");
  });
  it("returns ? for an empty name", () => {
    expect(initialsOf("   ")).toBe("?");
  });
});

describe("runeLength / clampRunes", () => {
  it("counts code points, not UTF-16 units", () => {
    expect(runeLength("😀😀")).toBe(2);
  });
  it("clamps by runes without splitting a surrogate pair", () => {
    const out = clampRunes("😀".repeat(200), BIO_MAX_RUNES);
    expect(runeLength(out)).toBe(160);
  });
  it("leaves a short string untouched", () => {
    expect(clampRunes("hi", BIO_MAX_RUNES)).toBe("hi");
  });
});

describe("height conversion", () => {
  it("shows cm in inches when unit is in (180cm → 70.9)", () => {
    expect(heightToDisplay(180, "in")).toBe("70.9");
  });
  it("shows cm as-is when unit is cm", () => {
    expect(heightToDisplay(180, "cm")).toBe("180");
  });
  it("shows an empty string for a null height", () => {
    expect(heightToDisplay(null, "in")).toBe("");
  });
  it("converts an inches display back to cm rounded to 0.1 (72 → 182.9)", () => {
    expect(displayToCm("72", "in")).toBe(182.9);
  });
  it("passes a cm display through", () => {
    expect(displayToCm("180", "cm")).toBe(180);
  });
  it("maps an empty display to null", () => {
    expect(displayToCm("", "in")).toBe(null);
  });
  it("throws RangeError on a non-positive or non-finite height", () => {
    expect(() => displayToCm("0", "cm")).toThrow(RangeError);
    expect(() => displayToCm("abc", "cm")).toThrow(RangeError);
  });
});

describe("draftFromProfile / dirtyKeys / patchFromDraft", () => {
  it("projects a profile into a draft", () => {
    const d = draftFromProfile(profile());
    expect(d).toEqual({
      display_name: "Sam",
      username: "sam",
      bio: "",
      height: "70.9", // mi → inches
      distance_unit: "mi",
      weight_unit: "lb",
      calendar_default_detail: "time_block",
    });
  });
  it("reports no dirty keys for an unedited draft", () => {
    const base = draftFromProfile(profile());
    expect(dirtyKeys(base, base)).toEqual([]);
  });
  it("reports only the changed keys", () => {
    const base = draftFromProfile(profile());
    const next: Draft = { ...base, display_name: "Sammy", bio: "hi" };
    expect(dirtyKeys(base, next).sort()).toEqual(["bio", "display_name"]);
  });
  it("builds a patch of only the dirty keys, mapping height→height_cm and username lowercased", () => {
    const base = draftFromProfile(profile());
    const next: Draft = { ...base, display_name: "Sammy", username: "NewSam", height: "72" };
    expect(patchFromDraft(base, next)).toEqual({
      display_name: "Sammy",
      username: "newsam",
      height_cm: 182.9,
    });
  });
  it("maps a cleared height to height_cm null", () => {
    const base = draftFromProfile(profile());
    const next: Draft = { ...base, height: "" };
    expect(patchFromDraft(base, next)).toEqual({ height_cm: null });
  });
  it("maps a cleared bio to an empty string", () => {
    const base = draftFromProfile(profile({ bio: "old" }));
    const next: Draft = { ...base, bio: "" };
    expect(patchFromDraft(base, next)).toEqual({ bio: "" });
  });
});

describe("USERNAME_RE", () => {
  it("accepts a valid handle", () => {
    expect(USERNAME_RE.test("sam_99")).toBe(true);
  });
  it("rejects a leading digit / uppercase / too-short handle", () => {
    expect(USERNAME_RE.test("1bad")).toBe(false);
    expect(USERNAME_RE.test("Bad")).toBe(false);
    expect(USERNAME_RE.test("ab")).toBe(false);
  });
});
```

- [ ] **Step 2: Run the tests, verify they fail**

Run: `npm run test -- app/\(app\)/settings/_components/draft.test.ts`
Expected: FAIL — module `./draft` not found.

- [ ] **Step 3: Implement `draft.ts`**

```ts
// app/(app)/settings/_components/draft.ts
import type { ResolvedProfile } from "@/lib/api";

export const CM_PER_INCH = 2.54;
export const BIO_MAX_RUNES = 160;
// Mirrors the server's username validator: 3–30 chars, lead letter, then
// lowercase letters / digits / underscores. The server stays authoritative.
export const USERNAME_RE = /^[a-z][a-z0-9_]{2,29}$/;

/** The editable profile fields, projected into one flat diff-able shape. */
export type Draft = {
  display_name: string;
  username: string;
  bio: string;
  // Height as the DISPLAY string in the current distance-derived unit; "" = unset.
  height: string;
  distance_unit: "mi" | "km";
  weight_unit: "lb" | "kg";
  calendar_default_detail: "time_block" | "full_agenda";
};

export type DraftKey = keyof Draft;

export type HeightUnit = "in" | "cm";

/** "mi" distance → height in inches; "km" → centimeters. */
export function heightUnitFor(distance: "mi" | "km"): HeightUnit {
  return distance === "km" ? "cm" : "in";
}

/** Avatar initials: first two of a single name, else first+last initials. */
export function initialsOf(name: string): string {
  const parts = name.trim().split(/\s+/).filter(Boolean);
  if (parts.length === 0) return "?";
  if (parts.length === 1) return parts[0].slice(0, 2).toUpperCase();
  return (parts[0][0] + parts[parts.length - 1][0]).toUpperCase();
}

/** Count by code points (not UTF-16 units) so emoji count as one. */
export const runeLength = (value: string): number => [...value].length;

/** Trim to at most `max` runes without splitting a surrogate pair. */
export const clampRunes = (value: string, max: number): string =>
  runeLength(value) <= max ? value : [...value].slice(0, max).join("");

/** Canonical cm → display string in the given unit; null → "". */
export function heightToDisplay(cm: number | null, unit: HeightUnit): string {
  if (cm == null) return "";
  const v = unit === "in" ? cm / CM_PER_INCH : cm;
  return String(Math.round(v * 10) / 10);
}

/**
 * Display string in the given unit → canonical cm rounded to 0.1; "" → null.
 * Throws RangeError on a non-finite or non-positive value so the save handler
 * can surface "enter a valid height".
 */
export function displayToCm(display: string, unit: HeightUnit): number | null {
  const raw = display.trim();
  if (raw === "") return null;
  const n = Number(raw);
  if (!Number.isFinite(n) || n <= 0) {
    throw new RangeError("Enter a valid height, or leave blank to clear.");
  }
  const cm = unit === "in" ? n * CM_PER_INCH : n;
  return Math.round(cm * 10) / 10;
}

/** Capture an immutable draft baseline from the resolved profile. */
export function draftFromProfile(p: ResolvedProfile): Draft {
  return {
    display_name: p.display_name ?? "",
    username: p.username ?? "",
    bio: p.bio ?? "",
    height: heightToDisplay(p.height_cm, heightUnitFor(p.distance_unit)),
    distance_unit: p.distance_unit,
    weight_unit: p.weight_unit,
    calendar_default_detail: p.calendar_default_detail,
  };
}

/** Keys whose draft value differs from the baseline. */
export function dirtyKeys(initial: Draft, draft: Draft): DraftKey[] {
  return (Object.keys(draft) as DraftKey[]).filter((k) => draft[k] !== initial[k]);
}

/** The PATCH body — only the dirty keys, mapped to the API's shape. */
export function patchFromDraft(
  initial: Draft,
  draft: Draft,
): {
  display_name?: string;
  username?: string;
  bio?: string;
  height_cm?: number | null;
  distance_unit?: "mi" | "km";
  weight_unit?: "lb" | "kg";
  calendar_default_detail?: "time_block" | "full_agenda";
} {
  const patch: ReturnType<typeof patchFromDraft> = {};
  for (const k of dirtyKeys(initial, draft)) {
    switch (k) {
      case "display_name":
        patch.display_name = draft.display_name.trim();
        break;
      case "username":
        patch.username = draft.username.trim().toLowerCase();
        break;
      case "bio":
        patch.bio = draft.bio.trim();
        break;
      case "height":
        patch.height_cm = displayToCm(draft.height, heightUnitFor(draft.distance_unit));
        break;
      case "distance_unit":
        patch.distance_unit = draft.distance_unit;
        break;
      case "weight_unit":
        patch.weight_unit = draft.weight_unit;
        break;
      case "calendar_default_detail":
        patch.calendar_default_detail = draft.calendar_default_detail;
        break;
    }
  }
  return patch;
}
```

- [ ] **Step 4: Run the tests, verify they pass**

Run: `npm run test -- app/\(app\)/settings/_components/draft.test.ts`
Expected: PASS (all cases).

- [ ] **Step 5: Gate + commit**

```bash
npm run typecheck && npm run lint
git add "app/(app)/settings/_components/draft.ts" "app/(app)/settings/_components/draft.test.ts"
git commit -m "feat(settings): add draft helpers and types for the save model"
```

---

### Task 2: Presentational primitives (`primitives.tsx`)

**Files:**
- Create: `app/(app)/settings/_components/primitives.tsx`
- Test: `app/(app)/settings/_components/primitives.test.tsx`

Three dumb building blocks. `Card` = a `rounded-2xl` titled panel. `Field` = uppercase-faint label over a control, optional muted hint. `SegmentedToggle` = the full-pill accent-active toggle (a typed, reusable lift of the current page's `SegmentedControl`).

- [ ] **Step 1: Write the failing tests**

```tsx
// app/(app)/settings/_components/primitives.test.tsx
/// <reference types="vitest/globals" />
import { render, screen, fireEvent } from "@testing-library/react";
import { Card, Field, SegmentedToggle } from "./primitives";

describe("Card", () => {
  it("renders the title and children", () => {
    render(
      <Card title="Profile">
        <p>body</p>
      </Card>,
    );
    expect(screen.getByText("Profile")).toBeInTheDocument();
    expect(screen.getByText("body")).toBeInTheDocument();
  });
});

describe("Field", () => {
  it("renders the label, hint, and control", () => {
    render(
      <Field label="Display name" hint="What your coach calls you">
        <input aria-label="x" />
      </Field>,
    );
    expect(screen.getByText("Display name")).toBeInTheDocument();
    expect(screen.getByText("What your coach calls you")).toBeInTheDocument();
    expect(screen.getByLabelText("x")).toBeInTheDocument();
  });
});

describe("SegmentedToggle", () => {
  it("marks the active option and fires onChange for another", () => {
    const onChange = vi.fn();
    render(
      <SegmentedToggle
        value="mi"
        options={[
          { value: "mi", label: "Miles" },
          { value: "km", label: "Kilometers" },
        ]}
        onChange={onChange}
      />,
    );
    expect(screen.getByRole("button", { name: "Miles" })).toHaveAttribute("aria-pressed", "true");
    fireEvent.click(screen.getByRole("button", { name: "Kilometers" }));
    expect(onChange).toHaveBeenCalledWith("km");
  });

  it("does not fire onChange when clicking the already-active option", () => {
    const onChange = vi.fn();
    render(
      <SegmentedToggle
        value="mi"
        options={[
          { value: "mi", label: "Miles" },
          { value: "km", label: "Kilometers" },
        ]}
        onChange={onChange}
      />,
    );
    fireEvent.click(screen.getByRole("button", { name: "Miles" }));
    expect(onChange).not.toHaveBeenCalled();
  });
});
```

- [ ] **Step 2: Run the tests, verify they fail**

Run: `npm run test -- app/\(app\)/settings/_components/primitives.test.tsx`
Expected: FAIL — module `./primitives` not found.

- [ ] **Step 3: Implement `primitives.tsx`**

```tsx
// app/(app)/settings/_components/primitives.tsx
import type { ReactNode } from "react";

/** Shared input surface — rounded slate fill, hairline border, accent focus ring. */
export const inputClass =
  "rounded-lg border border-[var(--border)] bg-[var(--surface-2)] px-3 py-2 text-sm text-[var(--foreground)] transition placeholder:text-[var(--muted)] focus:border-[var(--accent)] focus:outline-none focus:ring-1 focus:ring-[var(--accent-line)] disabled:opacity-60";

/** A titled, rounded card: bold section title over a divider, padded body. */
export function Card({ title, children }: { title: string; children: ReactNode }) {
  return (
    <section className="rounded-2xl border border-[var(--border)] bg-[var(--surface)] shadow-sm">
      <h2 className="border-b border-[var(--border)] px-5 py-3 text-sm font-bold tracking-tight">
        {title}
      </h2>
      <div className="flex flex-col gap-5 px-5 py-5">{children}</div>
    </section>
  );
}

/** A quiet uppercase-faint label over a control, with an optional muted hint. */
export function Field({
  label,
  hint,
  htmlFor,
  children,
}: {
  label: string;
  hint?: ReactNode;
  htmlFor?: string;
  children: ReactNode;
}) {
  return (
    <div className="flex flex-col gap-1.5">
      <label
        htmlFor={htmlFor}
        className="text-[11px] font-semibold uppercase tracking-wide text-[var(--faint)]"
      >
        {label}
      </label>
      {children}
      {hint != null && <p className="text-xs text-[var(--muted)]">{hint}</p>}
    </div>
  );
}

/** Full-pill segmented control: active = accent fill, inactive = muted. */
export function SegmentedToggle<T extends string>({
  value,
  options,
  onChange,
  disabled = false,
}: {
  value: T;
  options: { value: T; label: string }[];
  onChange: (value: T) => void;
  disabled?: boolean;
}) {
  return (
    <div
      role="group"
      className="inline-flex rounded-full border border-[var(--border)] bg-[var(--background)] p-0.5"
    >
      {options.map((opt) => {
        const active = opt.value === value;
        return (
          <button
            key={opt.value}
            type="button"
            disabled={disabled}
            aria-pressed={active}
            onClick={() => {
              if (!active) onChange(opt.value);
            }}
            className={`rounded-full px-3 py-1 text-xs font-medium transition disabled:cursor-not-allowed disabled:opacity-60 ${
              active
                ? "bg-[var(--accent)] text-[var(--accent-fg)]"
                : "text-[var(--muted)] hover:text-[var(--foreground)]"
            }`}
          >
            {opt.label}
          </button>
        );
      })}
    </div>
  );
}
```

- [ ] **Step 4: Run the tests, verify they pass**

Run: `npm run test -- app/\(app\)/settings/_components/primitives.test.tsx`
Expected: PASS.

- [ ] **Step 5: Gate + commit**

```bash
npm run typecheck && npm run lint
git add "app/(app)/settings/_components/primitives.tsx" "app/(app)/settings/_components/primitives.test.tsx"
git commit -m "feat(settings): add card, field, and segmented-toggle primitives"
```

---

### Task 3: The sticky save bar (`SaveBar.tsx`)

**Files:**
- Create: `app/(app)/settings/_components/SaveBar.tsx`
- Test: `app/(app)/settings/_components/SaveBar.test.tsx`

A presentational sticky bar driven entirely by props (the page owns the state). It renders only when there's something to show (`dirtyCount > 0 || savedFlash`). Left side shows, in priority order: the saved-flash, else the block reason when `!canSave`, else the dirty count. Right side: a text **Discard** and the violet **Save changes** (disabled unless `canSave`).

- [ ] **Step 1: Write the failing tests**

```tsx
// app/(app)/settings/_components/SaveBar.test.tsx
/// <reference types="vitest/globals" />
import { render, screen, fireEvent } from "@testing-library/react";
import { SaveBar } from "./SaveBar";

const base = {
  dirtyCount: 0,
  canSave: false,
  blockReason: null as string | null,
  savedFlash: false,
  saving: false,
  onSave: () => {},
  onDiscard: () => {},
};

describe("SaveBar", () => {
  it("renders nothing when clean and not flashing", () => {
    const { container } = render(<SaveBar {...base} />);
    expect(container).toBeEmptyDOMElement();
  });

  it("shows a singular count for one change", () => {
    render(<SaveBar {...base} dirtyCount={1} canSave />);
    expect(screen.getByText("1 unsaved change")).toBeInTheDocument();
  });

  it("shows a plural count for many changes", () => {
    render(<SaveBar {...base} dirtyCount={3} canSave />);
    expect(screen.getByText("3 unsaved changes")).toBeInTheDocument();
  });

  it("shows the block reason instead of the count when blocked", () => {
    render(<SaveBar {...base} dirtyCount={2} canSave={false} blockReason="fix username first" />);
    expect(screen.getByText("fix username first")).toBeInTheDocument();
    expect(screen.queryByText("2 unsaved changes")).not.toBeInTheDocument();
    expect(screen.getByRole("button", { name: "Save changes" })).toBeDisabled();
  });

  it("enables Save and fires onSave when allowed", () => {
    const onSave = vi.fn();
    render(<SaveBar {...base} dirtyCount={1} canSave onSave={onSave} />);
    const btn = screen.getByRole("button", { name: "Save changes" });
    expect(btn).toBeEnabled();
    fireEvent.click(btn);
    expect(onSave).toHaveBeenCalled();
  });

  it("fires onDiscard from the Discard button", () => {
    const onDiscard = vi.fn();
    render(<SaveBar {...base} dirtyCount={1} canSave onDiscard={onDiscard} />);
    fireEvent.click(screen.getByRole("button", { name: "Discard" }));
    expect(onDiscard).toHaveBeenCalled();
  });

  it("shows the saved flash and hides the action buttons", () => {
    render(<SaveBar {...base} dirtyCount={0} savedFlash />);
    expect(screen.getByText("All changes saved ✓")).toBeInTheDocument();
    expect(screen.queryByRole("button", { name: "Save changes" })).not.toBeInTheDocument();
  });

  it("shows a saving label on the Save button while saving", () => {
    render(<SaveBar {...base} dirtyCount={1} canSave saving />);
    expect(screen.getByRole("button", { name: "Saving…" })).toBeDisabled();
  });
});
```

- [ ] **Step 2: Run the tests, verify they fail**

Run: `npm run test -- app/\(app\)/settings/_components/SaveBar.test.tsx`
Expected: FAIL — module `./SaveBar` not found.

- [ ] **Step 3: Implement `SaveBar.tsx`**

```tsx
// app/(app)/settings/_components/SaveBar.tsx

export function SaveBar({
  dirtyCount,
  canSave,
  blockReason,
  savedFlash,
  saving,
  onSave,
  onDiscard,
}: {
  dirtyCount: number;
  canSave: boolean;
  blockReason: string | null;
  savedFlash: boolean;
  saving: boolean;
  onSave: () => void;
  onDiscard: () => void;
}) {
  // Render only when there's something to say.
  if (dirtyCount === 0 && !savedFlash) return null;

  const left = savedFlash ? (
    <span className="text-sm font-medium text-[var(--success)]">All changes saved ✓</span>
  ) : !canSave && blockReason ? (
    <span className="text-sm text-[var(--muted)]">{blockReason}</span>
  ) : (
    <span className="text-sm font-medium text-[var(--foreground)]">
      {dirtyCount} unsaved {dirtyCount === 1 ? "change" : "changes"}
    </span>
  );

  return (
    <div
      role="region"
      aria-label="Unsaved changes"
      className="sticky bottom-4 z-10 mx-auto flex w-full items-center justify-between gap-4 rounded-2xl border border-[var(--accent-line)] bg-[var(--surface)] px-5 py-3 shadow-lg"
    >
      {left}
      {!savedFlash && (
        <div className="flex shrink-0 items-center gap-2">
          <button
            type="button"
            onClick={onDiscard}
            disabled={saving}
            className="rounded-full px-3 py-1.5 text-sm font-medium text-[var(--muted)] transition hover:text-[var(--foreground)] disabled:opacity-50"
          >
            Discard
          </button>
          <button
            type="button"
            onClick={onSave}
            disabled={!canSave || saving}
            className="rounded-full bg-[var(--accent)] px-4 py-1.5 text-sm font-semibold text-[var(--accent-fg)] transition hover:opacity-90 disabled:opacity-50"
          >
            {saving ? "Saving…" : "Save changes"}
          </button>
        </div>
      )}
    </div>
  );
}
```

- [ ] **Step 4: Run the tests, verify they pass**

Run: `npm run test -- app/\(app\)/settings/_components/SaveBar.test.tsx`
Expected: PASS.

- [ ] **Step 5: Gate + commit**

```bash
npm run typecheck && npm run lint
git add "app/(app)/settings/_components/SaveBar.tsx" "app/(app)/settings/_components/SaveBar.test.tsx"
git commit -m "feat(settings): add sticky save bar component"
```

---

### Task 4: Username field with live availability (`UsernameField.tsx`)

**Files:**
- Create: `app/(app)/settings/_components/UsernameField.tsx`
- Test: `app/(app)/settings/_components/UsernameField.test.tsx`

The `@`-prefixed handle input. It owns the debounced (~400 ms) availability probe (`checkUsernameAvailable`), fired only when the handle is **dirty** (differs from the original) and charset-valid, and it renders the inline status line (idle / checking / available / taken / error / charset hint). It reports two things up to the page: the current value (`onChange`) and whether the handle currently **blocks** Save (`onBlockedChange(true)` while checking / taken / charset-invalid-and-dirty). The handle no longer has its own Save button — it gates the one save bar.

Context for the implementer: the current `UsernameRow` (in the existing `page.tsx`, lines ~386–519) is the reference for the probe effect, the `USERNAME_RE`, the normalization, and the status-line copy. Lift that logic; drop the per-field Save button and the `onSave`; add the `onBlockedChange` callback. Import `USERNAME_RE` from `./draft`. Reuse `inputClass` from `./primitives`.

- [ ] **Step 1: Write the failing tests**

```tsx
// app/(app)/settings/_components/UsernameField.test.tsx
/// <reference types="vitest/globals" />
import { render, screen, fireEvent, waitFor } from "@testing-library/react";

const checkUsernameAvailableMock = vi.hoisted(() => vi.fn());
vi.mock("@/lib/api", async (orig) => ({
  ...(await orig<typeof import("@/lib/api")>()),
  checkUsernameAvailable: checkUsernameAvailableMock,
}));
vi.mock("@/lib/auth", () => ({ getToken: () => "test-token" }));

import { UsernameField } from "./UsernameField";

function setup(over: Partial<React.ComponentProps<typeof UsernameField>> = {}) {
  const onChange = vi.fn();
  const onBlockedChange = vi.fn();
  render(
    <UsernameField
      value="sam"
      original="sam"
      disabled={false}
      onChange={onChange}
      onBlockedChange={onBlockedChange}
      {...over}
    />,
  );
  return { onChange, onBlockedChange };
}

beforeEach(() => {
  vi.clearAllMocks();
  checkUsernameAvailableMock.mockResolvedValue(true);
});

describe("UsernameField", () => {
  it("renders the @-prefixed current value", () => {
    setup();
    expect(screen.getByLabelText("Username")).toHaveValue("sam");
  });

  it("lowercases input through onChange", () => {
    const { onChange } = setup();
    fireEvent.change(screen.getByLabelText("Username"), { target: { value: "NewSam" } });
    expect(onChange).toHaveBeenCalledWith("newsam");
  });

  it("shows the charset hint and blocks for an invalid dirty handle, skipping the probe", async () => {
    const { onBlockedChange } = setup({ value: "1bad", original: "sam" });
    await waitFor(() =>
      expect(screen.getByText(/3–30 characters: start with a letter/)).toBeInTheDocument(),
    );
    expect(checkUsernameAvailableMock).not.toHaveBeenCalled();
    expect(onBlockedChange).toHaveBeenCalledWith(true);
  });

  it("probes and shows available for a free valid handle, unblocking", async () => {
    const { onBlockedChange } = setup({ value: "newhandle", original: "sam" });
    await waitFor(() =>
      expect(checkUsernameAvailableMock).toHaveBeenCalledWith("test-token", "newhandle"),
    );
    await waitFor(() => expect(screen.getByText("@newhandle is available.")).toBeInTheDocument());
    await waitFor(() => expect(onBlockedChange).toHaveBeenLastCalledWith(false));
  });

  it("shows taken and blocks when the probe returns taken", async () => {
    checkUsernameAvailableMock.mockResolvedValue(false);
    const { onBlockedChange } = setup({ value: "occupied", original: "sam" });
    await waitFor(() => expect(screen.getByText("@occupied is taken.")).toBeInTheDocument());
    await waitFor(() => expect(onBlockedChange).toHaveBeenLastCalledWith(true));
  });

  it("is idle and unblocked when the handle equals the original", () => {
    const { onBlockedChange } = setup({ value: "sam", original: "sam" });
    expect(checkUsernameAvailableMock).not.toHaveBeenCalled();
    expect(onBlockedChange).toHaveBeenLastCalledWith(false);
  });
});
```

- [ ] **Step 2: Run the tests, verify they fail**

Run: `npm run test -- app/\(app\)/settings/_components/UsernameField.test.tsx`
Expected: FAIL — module `./UsernameField` not found.

- [ ] **Step 3: Implement `UsernameField.tsx`**

```tsx
// app/(app)/settings/_components/UsernameField.tsx
"use client";

import { useEffect, useState } from "react";
import { checkUsernameAvailable } from "@/lib/api";
import { getToken } from "@/lib/auth";
import { USERNAME_RE } from "./draft";
import { inputClass } from "./primitives";

const DEBOUNCE_MS = 400;

type Availability =
  | { kind: "idle" }
  | { kind: "checking" }
  | { kind: "available" }
  | { kind: "taken" }
  | { kind: "error"; message: string };

/**
 * The @-prefixed handle input. Owns the debounced availability probe and the
 * inline status line; reports its value up (lowercased) and whether it
 * currently blocks Save (checking / taken / charset-invalid while dirty).
 * It has no Save button — it gates the page's single save bar.
 */
export function UsernameField({
  value,
  original,
  disabled,
  onChange,
  onBlockedChange,
}: {
  value: string;
  original: string;
  disabled: boolean;
  onChange: (next: string) => void;
  onBlockedChange: (blocked: boolean) => void;
}) {
  const [availability, setAvailability] = useState<Availability>({ kind: "idle" });

  const normalized = value.trim().toLowerCase();
  const charsetOk = USERNAME_RE.test(normalized);
  const dirty = normalized !== original.trim().toLowerCase();

  // Debounced probe: only for a dirty, charset-valid handle. Cleanup cancels
  // the in-flight timer/result so we don't fire a request per keystroke.
  useEffect(() => {
    if (!dirty || !charsetOk) {
      setAvailability({ kind: "idle" });
      return;
    }
    const token = getToken();
    if (!token) {
      setAvailability({ kind: "idle" });
      return;
    }
    setAvailability({ kind: "checking" });
    let cancelled = false;
    const handle = window.setTimeout(() => {
      checkUsernameAvailable(token, normalized)
        .then((free) => {
          if (cancelled) return;
          setAvailability(free ? { kind: "available" } : { kind: "taken" });
        })
        .catch((err: unknown) => {
          if (cancelled) return;
          setAvailability({
            kind: "error",
            message: err instanceof Error ? err.message : "Couldn't check availability",
          });
        });
    }, DEBOUNCE_MS);
    return () => {
      cancelled = true;
      window.clearTimeout(handle);
    };
  }, [normalized, dirty, charsetOk]);

  // Block Save while the handle is dirty and not known-good: charset-invalid,
  // mid-probe, or taken. (An idle/available/error-on-unchanged handle is fine.)
  const blocked = dirty && (!charsetOk || availability.kind === "checking" || availability.kind === "taken");
  useEffect(() => {
    onBlockedChange(blocked);
  }, [blocked, onBlockedChange]);

  const showCharsetHint = dirty && value.trim() !== "" && !charsetOk;

  return (
    <div className="flex flex-col gap-1.5">
      <div className="flex items-center gap-2">
        <span className="shrink-0 text-sm text-[var(--muted)]">@</span>
        <input
          id="settings-username"
          type="text"
          aria-label="Username"
          value={value}
          maxLength={30}
          autoCapitalize="none"
          autoCorrect="off"
          spellCheck={false}
          disabled={disabled}
          onChange={(e) => onChange(e.target.value.toLowerCase())}
          className={`${inputClass} min-w-0 flex-1`}
        />
      </div>
      {showCharsetHint ? (
        <p className="text-xs text-[var(--danger)]">
          3–30 characters: start with a letter, then lowercase letters, numbers, or underscores.
        </p>
      ) : availability.kind === "checking" ? (
        <p className="text-xs text-[var(--muted)]">Checking availability…</p>
      ) : availability.kind === "available" ? (
        <p className="text-xs text-[var(--success)]">@{normalized} is available.</p>
      ) : availability.kind === "taken" ? (
        <p className="text-xs text-[var(--danger)]">@{normalized} is taken.</p>
      ) : availability.kind === "error" ? (
        <p className="text-xs text-[var(--muted)]">{availability.message}</p>
      ) : null}
    </div>
  );
}
```

Note on the `onChange` lowercasing: the page stores whatever the field reports; lowercasing on the way out keeps `draft.username` canonical so the diff and the eventual patch agree. The test `lowercases input through onChange` asserts this.

- [ ] **Step 4: Run the tests, verify they pass**

Run: `npm run test -- app/\(app\)/settings/_components/UsernameField.test.tsx`
Expected: PASS.

- [ ] **Step 5: Gate + commit**

```bash
npm run typecheck && npm run lint
git add "app/(app)/settings/_components/UsernameField.tsx" "app/(app)/settings/_components/UsernameField.test.tsx"
git commit -m "feat(settings): add username field with live availability probe"
```

---

### Task 5: Rebuild the page + rewrite its test (`page.tsx`, `page.test.tsx`)

**Files:**
- Rewrite: `app/(app)/settings/page.tsx`
- Rewrite: `app/(app)/settings/page.test.tsx`

The integration task. Compose the four grouped cards, the page-level draft, the save handler, and the sticky bar. Avatar and calendar stay as immediate out-of-band actions (lift the existing `AvatarRow` + initials/preview logic and `GoogleCalendarConnectionRow` essentially unchanged, restyled into cards). This is a judgment/integration task — use a capable model.

**Behavioral spec (the page must satisfy all of this):**

1. **Baseline + draft.** On profile load, `initial = draftFromProfile(profile)` and `draft` starts equal to it. While `profile` is null, render a muted "Loading…"/disabled state (the existing page passed `disabled={!profile}`; here, simply gate the cards' editable controls — show the shell). When the profile context updates after a successful save, re-baseline `initial` (and `draft`) from the returned profile so the bar retracts.
   - Re-seeding pattern: keep `initial`/`draft` in state seeded from `profile`; an effect re-seeds **both** when the profile **identity** changes (e.g. key off `profile?.id` plus a saved-generation counter you bump after save) — but must NOT clobber in-progress edits on unrelated context re-renders. Simplest correct approach: re-seed in an effect keyed on a `baselineProfile` reference that you only update (a) on first load and (b) right after a successful save. Mirror the care the old per-field rows took with their re-seed effects; an `eslint-disable-next-line react-hooks/exhaustive-deps` on the re-seed effect is acceptable, matching the existing code.
2. **`set(key, value)`** updates `draft[key]`. `dirty = dirtyKeys(initial, draft)`, `dirtyCount = dirty.length`.
3. **Distance unit drives height display.** When the user flips the distance toggle, the height field's unit label and the displayed value must follow. Because `draft.height` is stored as a display string in the *current* unit, flipping distance must **re-express** the height string into the new unit so the underlying cm is preserved (convert the current `draft.height` via cm: `displayToCm(old) → heightToDisplay(cm, newUnit)`), and do the same for the `initial.height` baseline so flipping distance back and forth doesn't spuriously mark height dirty. Implement this in the distance `set` path (a small helper `reexpressHeight`). Guard the conversion: if `draft.height` is currently invalid, leave it as typed.
4. **Validation gate.** `nameEmpty = draft.display_name.trim() === ""`. `handleBlocked` comes from `UsernameField`'s `onBlockedChange`. `canSave = dirtyCount > 0 && !nameEmpty && !handleBlocked && !saving`. `blockReason`: `nameEmpty ? "Add a display name to save" : handleBlocked ? "Fix the username to save" : null`.
5. **Save handler.** Build `patch = patchFromDraft(initial, draft)`. If empty, no-op. Call `update(patch)`:
   - On success: re-baseline from the returned profile, `setSavedFlash(true)`, clear it after ~1800 ms (store the timeout id; clear on unmount). The bar shows the flash then retracts.
   - Also write the distance unit through `useDistanceUnit().setUnit(draft.distance_unit)` when `distance_unit` is dirty (keeps the localStorage context in sync, matching today's behavior; `setUnit` itself fire-and-forgets a `updateMe`, which is harmless/idempotent next to the explicit patch — but to avoid a double PATCH, call `setUnit` for the localStorage+context effect and rely on the single `update(patch)` for persistence; if simpler, call `setUnit` after the successful `update`). Keep it simple and correct: after a successful `update`, if distance changed, call `setUnit(draft.distance_unit)` so localStorage + context converge.
   - On error: keep the bar; surface the message via `toast.error` and an inline reason in the bar (set `blockReason` to the error or show a toast — a toast plus leaving the bar up is sufficient; the existing tests use toasts).
   - `displayToCm` can throw on an invalid height; wrap the patch build in try/catch and, on a height error, toast the message and abort the save (don't call `update`).
6. **Cards (top to bottom):**
   - **Profile** (`Card title="Profile"`): avatar block (Upload/Change/Remove → `uploadAvatar`/`removeAvatar` immediately, keep the 2 MB + png/jpeg/webp guard + toast; initials via `initialsOf`), display name `Field` (input, `maxLength={60}`, inline "Display name is required." when empty-and-touched), `UsernameField`, bio `Field` (textarea, `clampRunes(.,160)` on change, `{runeLength}/160` counter in `tabular-nums`).
   - **Daily AI allowance** (`Card title="Daily AI allowance"`): render `<UsageBar />` unchanged; copy that it's read-only; excluded from draft/dirty.
   - **Units** (`Card title="Units"`): distance `SegmentedToggle` (mi/km) and weight `SegmentedToggle` (lb/kg) bound to `draft`. Height belongs logically with the body metrics — place the height `Field` in the Profile card per the SOW's Profile composition (display name, username, bio) **and** a height input; SOW lists height under Profile card ("display name; username; bio") — re-reading the SOW: height is part of the draft but the SOW's Profile card bullet lists avatar/name/username/bio, and Units lists distance+weight, with "Distance also drives the height unit." Height was historically in Profile. **Decision:** keep the height `Field` in the **Profile** card (as today), with its unit label following the distance toggle. Document this in a comment.
   - **Google Calendar** (`Card title="Google Calendar"`): `getCalendarConnection()` on mount (connected/absent/checking) with Connect (OAuth redirect to `${config.apiUrl}/auth/google/calendar/connect?return_to=${origin}/settings`) or Disconnect (`disconnectCalendar` + reload of the connection), both immediate/out-of-band; default-event-detail `SegmentedToggle` (time_block/full_agenda) bound to `draft`.
7. **Layout:** `main` with a header ("Settings"), a scrollable body, a centered `max-w-2xl` column of cards with `gap` between them, and **bottom padding reserved** (e.g. `pb-24`) so the sticky `SaveBar` never covers the last card. Single column at narrow width (the cards are already full-width; ensure nothing forces a min-width that breaks mobile). Render `<SaveBar .../>` at the foot of the column.

**The new test file** must cover the SOW's test list. Reuse the existing mock scaffolding (next/navigation, `@/lib/api` partial mock for calendar+username, `@/lib/auth`, `@/lib/distance-unit-context`, `@/components/toast`, `@/lib/usage-context`, `@/lib/profile-context`) and the `profile()`/`snapshot()` helpers from the current test. Cases:

- **No bar when clean:** initial render shows no "unsaved" region.
- **Editing shows the bar with the right count:** change display name → "1 unsaved change"; also edit bio → "2 unsaved changes".
- **Discard reverts and hides the bar:** after editing name+bio, click Discard → inputs return to "Sam"/"" and the bar is gone.
- **Save issues one `update` with only the changed keys + flash + re-baseline:** edit name and bio, click Save changes → `updateMe`/`update` called once with `{ display_name, bio }` (no other keys); "All changes saved ✓" appears; then (mock `update` resolves to the updated profile) the bar retracts.
- **Save disabled on empty name, with reason:** clear the name → bar shows, Save changes disabled, reason "Add a display name to save".
- **Save disabled while username checking / taken / charset-invalid, with reason:** type a new handle → while checking, Save disabled + "Fix the username to save"; with `checkUsernameAvailable→false`, "@x is taken." and Save stays disabled; type "1bad" → charset hint and Save disabled.
- **Username probe transitions:** checking → available (`true`) / taken (`false`).
- **Bio rune counter caps at 160 incl. emoji:** `"😀".repeat(200)` → value 160 runes, "160/160".
- **Height converts in↔cm and unset round-trips:** mi → "Height (in)" shows 70.9; with `distance_unit:"km"` → "Height (cm)" shows 180; `height_cm:null` → empty; editing inches and saving sends `height_cm` converted.
- **Avatar upload/remove are immediate and out of the dirty count:** uploading a valid png calls `uploadAvatar` and does **not** show the save bar; remove calls `removeAvatar`; oversized/non-image rejected with the existing toasts and no upload.
- **Calendar connect/disconnect out of the dirty count:** absent → "Connect Google Calendar"; connected → "Disconnect" calls `disconnectCalendar`; default-detail toggle edits `draft` (so it *does* show the bar — it's a draftable field — assert that flipping it shows "1 unsaved change" rather than calling `update` immediately).
- **Usage colors at 50/80/100%:** reuse the existing UsageBar assertions (accent / warning / danger).

> Note: the old tests asserted the default-detail and weight toggles called `update()` immediately. Under the new model those are draftable — flipping them shows the save bar and persists only on Save. Update those assertions accordingly (this is the intended behavior change).

- [ ] **Step 1: Rewrite `page.test.tsx`** for the new model (cases above). Keep the module mocks; adapt the helpers. Add a `vi.useFakeTimers()` or `waitFor` strategy for the 400 ms username debounce and the 1800 ms flash as the existing tests do (they rely on `waitFor`; the debounce resolves under real timers within `waitFor`).

- [ ] **Step 2: Run the test, watch it fail** against the old page.

Run: `npm run test -- app/\(app\)/settings/page.test.tsx`
Expected: FAIL (old page has per-field Save buttons; new assertions don't match).

- [ ] **Step 3: Rewrite `page.tsx`** per the behavioral spec. Compose `Card`/`Field`/`SegmentedToggle` from `./primitives`, `SaveBar` from `./SaveBar`, `UsernameField` from `./UsernameField`, and the helpers from `./draft`. Keep `AvatarRow`/`AvatarPreview` and `GoogleCalendarConnectionRow` as local components (restyled), immediate. Keep the 401/error surfacing that the profile context provides; surface per-field/save errors via `toast` + the bar.

- [ ] **Step 4: Run the settings test, iterate to green**

Run: `npm run test -- app/\(app\)/settings/page.test.tsx`
Expected: PASS.

- [ ] **Step 5: Full gate**

```bash
npm run typecheck && npm run lint && npm run format:check && npm run test && npm run build
```
Expected: all green. If `format:check` flags files, run `npm run format` and re-stage.

- [ ] **Step 6: Commit**

```bash
git add "app/(app)/settings/page.tsx" "app/(app)/settings/page.test.tsx"
git commit -m "feat(settings): rebuild settings page into grouped save-bar layout"
```

---

## Self-review checklist (run after all tasks)

- **Spec coverage:** grouped cards (Profile · Daily AI allowance · Units · Google Calendar) ✓; single draft + one sticky bar ✓; appear-on-dirty + count + Discard + gated Save + saved flash ✓; one `update(patch)` of dirty keys only ✓; validation gate (empty name / username states) with reason ✓; reused data layer unchanged ✓; preserved field semantics (username probe, bio runes, height cm↔display + unset, avatar image/initials + 2 MB guard, calendar states, usage thresholds, read-only allowance) ✓; avatar + calendar immediate/out-of-band ✓; design-system tokens only ✓; narrow/mobile reflow + bottom-padding reservation ✓; 401 handling preserved (context) ✓; test file rewritten ✓.
- **No backend change**, no new endpoints/SDK methods, no DX mockup code imported, `design-explore` route untouched.
- **Placeholder scan:** no TBD/TODO; every helper/type referenced across tasks is defined in Task 1 / Task 2.
- **Type consistency:** `Draft`, `DraftKey`, `draftFromProfile`, `dirtyKeys`, `patchFromDraft`, `heightToDisplay`, `displayToCm`, `initialsOf`, `runeLength`, `clampRunes`, `USERNAME_RE`, `inputClass`, `Card`, `Field`, `SegmentedToggle`, `SaveBar`, `UsernameField` names match across tasks.

## CI gate before pushing (per the SOW Rollout)

From `/workspace/prog-strength-web`: `npm run lint && npm run format:check && npm run typecheck && npm run test && npm run build` — all green. Do not bypass husky hooks; no `--no-verify`. Then push `feat/settings-page-redesign` and open the PR.
