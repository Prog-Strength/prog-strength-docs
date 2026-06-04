---
status: ready_for_implementation
repos:
  - prog-strength-web
  - prog-strength-docs
---

# Pantry / Recipe Form Polish + Toast Notifications

**Status**: Ready for Implementation · **Last updated**: 2026-06-04

## Introduction

The Add Pantry Item and Add Recipe forms on the web app are the first thing a user sees after they decide to teach Prog Strength about a new food, and they look like the rest of the page hadn't been built yet. Bare inputs, uppercase muted labels stacked above each field, a flat utilitarian grid, and a tiny red one-line error message that lives inside the form's surface and clears the moment the modal closes. There's no positive feedback at all — the user clicks Save, the modal vanishes, and the only signal that anything happened is that the pantry list refetched in the background. If the API rejects the save, the user sees the error briefly, but the moment the modal goes away on a successful retry the slate is wiped.

This SOW does two things at once because they belong together. The first is a visual polish pass on both forms to make them feel like the rest of the page — color-coded macro inputs that tie into the rings palette already shared by the log view, pantry catalog, and recipes catalog (extracted to `lib/macro-colors.ts` in the prior PR); cleaner input styling with accent focus rings; replacement of the bordered control buttons with the ghost-text and accent variants used elsewhere; and the standardized trash icon on recipe-component rows. The second is the introduction of a general-purpose toast notification system — `<ToastProvider>`, `useToast()`, a fixed-top banner stack — that the two modals fire into on save success and on server-side save failure, so the user gets a confirmation banner that reads "Added Greek yogurt to your pantry" or an error banner that reads "Couldn't save: ..." instead of the modal silently disappearing or the inline error vanishing with the close.

Nothing here is API surface or agent surface. The notification system is new infrastructure in `prog-strength-web` and the forms are the first two consumers; subsequent pages can adopt it as natural follow-ups.

## Proposed Solution

A new `components/toast/` module owns the toast infrastructure: a `<ToastProvider>` that wraps the app at the root layout level, a `useToast()` hook returning `{ success, error, info }`, and a `<ToastViewport>` portal that renders the active stack at the top-center of the viewport. The banner style matches the locked-in option C from brainstorming: a muted-fill chip with a colored left border (emerald for success, red for error), a small status icon, the message, and a dismiss `×` on the right. Success banners auto-dismiss after 4 seconds; error banners persist until the user dismisses them, because errors are infrequent and worth keeping on screen. Multiple banners stack vertically with the oldest on top; clicking dismiss removes that one and the rest reflow.

The pantry item form drops its surrounding bordered card and renders directly inside the modal body. The top row is a three-column grid — Name (wide) / Serving size / Serving unit — and below it a thin section divider with a `Per serving` label introduces the four macro inputs in a `grid-cols-4`. Each macro input gains two ties to the palette: a small color dot in its label and a 2px colored left border on the input itself. Protein is emerald, fat is pink, carbs is amber, calories stays neutral (no swatch, no border) because calories has no ring color allocated by deliberate design. The input visuals across the form get a focus ring that uses the page accent and slightly more breathing room. The button row at the bottom replaces the bordered controls: ghost Cancel, danger-ghost Delete (only in edit mode), filled accent Save.

The recipe form follows the same vocabulary. Name input at the top; the live `MacroPreview` tile picks up the macro palette so the four numbers (Cal / Protein / Fat / Carbs) render in their ring colors with a small color dot next to each label, matching the way the catalog rows are now styled. The components list keeps its up/down ordering arrows — recipe component order matters and a drag-handle pattern isn't worth introducing for a max-of-20 list — but the per-row remove control switches from `✕` to the standardized trash icon used everywhere else on the page. The `+ Add component` button drops its border in favor of the ghost text-button pattern (plus icon + label) that the toolbar buttons on the bodyweight and nutrition pages use.

Both modals fire into the toast system at exactly two moments: on `createPantryItem` / `updatePantryItem` / `createRecipe` / `updateRecipe` resolving successfully (`toast.success`), and on the same calls rejecting with a non-401 error from the server (`toast.error`). Client-side validation errors (`Name is required`, `Serving size must be greater than zero`) stay inline inside the form because the user is looking right at the field they need to fix; a banner at the top of the viewport for a missing field would be louder than the situation needs. 401s continue to redirect to login as today.

No new API endpoints. No schema work. No mobile or agent surfaces touched. One PR on `prog-strength-web`.

## Goals and Non-Goals

### Goals

- Add a general-purpose toast notification system in `components/toast/`:
  - `<ToastProvider>` that wraps `{children}` in `app/layout.tsx`.
  - `useToast()` hook returning `{ success, error, info }`. `info` is exposed but unused in this SOW; ships for future use.
  - `<ToastViewport>` portal that renders the stack at the top of the viewport, centered horizontally, max width matching the page content's max-w-md gutter so it doesn't dominate the screen.
  - Success banners auto-dismiss after 4 seconds; error banners persist until the user clicks the dismiss `×`.
  - Multiple banners stack vertically, newest at the bottom of the stack.
  - Banner style: muted-fill background, 3px colored left border (`#6ee7b7` emerald for success, `#f87171` red for error), small status icon (✓ for success, ! for error), the message text, dismiss `×` on the right.
  - Slide-down + fade-in on enter, fade-out on exit. CSS transitions only; no animation library.
  - Hook is safe to call from server components: the provider lives in a client component and the hook errors with a clear "must be used within ToastProvider" message if invoked outside the tree.
- Polish the pantry item form (`components/pantry-item-form.tsx`):
  - Drop the outer `rounded-lg border border-[var(--border)] bg-[var(--surface)] p-4` card; render directly into the modal body so the visual nesting is reduced.
  - Top row layout: `grid grid-cols-1 sm:grid-cols-[2fr_1fr_1fr] gap-3` — Name | Serving size | Serving unit. (Today's nested grid is replaced by a single grid.)
  - Section divider: a horizontal rule with the label `Per serving` as a small uppercase tracking-wider muted heading inline.
  - Four-column macro row: Calories | Protein (g) | Fat (g) | Carbs (g). `MacroField` gains a `tone?: "protein" | "fat" | "carbs"` prop: when set, the label gets a leading color dot and the input gets a 2px left border in that color. Calories (no tone) renders without either.
  - Input styling: `bg-[var(--surface)]`, focus ring (`focus:outline focus:outline-2 focus:outline-[var(--accent)]` or the existing focus convention), padding bumped to `px-3 py-2`.
  - Button row: replace the bordered controls with ghost (Cancel) / danger-ghost (Delete, edit mode) / filled accent (Save). Use the existing button vocabulary the page already standardizes on.
- Polish the recipe form (`components/recipe-form.tsx`):
  - Same flat container + input styling as pantry.
  - `MacroPreview` tile: each number colored to match the palette (calories muted, protein emerald, fat pink, carbs amber); each label gains a leading color dot.
  - `ComponentRow`: keep ↑ / ↓ ordering arrows (recipe order matters and a drag-handle adds complexity not paid back at a max-of-20 list). Replace the `✕` remove button with the standardized trash icon used in `nutrition-log-view.tsx` and the catalog rows.
  - Replace the bordered `+ Add component` button with a ghost text-button using the plus icon + "Add component" label, matching the toolbar buttons on the bodyweight and nutrition pages.
- Wire the modals to the toast system:
  - `PantryItemModal`: on `createPantryItem` resolving, `toast.success("Added \"<name>\" to your pantry.")`. On `updatePantryItem` resolving, `toast.success("Updated \"<name>\".")`. On either rejecting with a non-401 error, `toast.error("Couldn't save: <server message>")` *and* keep the inline form error (the user is at the form; the banner is a notification, the inline error is a correctable hint). The modal still closes on success exactly as today.
  - `RecipeModal`: same pattern. On create, `toast.success("Added \"<name>\" as a recipe.")`. On edit, `toast.success("Updated \"<name>\".")`.
  - 401 errors continue to redirect to `/login` as today; no toast for those.
  - Delete-confirm flows (the existing inline delete inside the edit modal, and the icon-driven delete modals from the prior PR) also fire `toast.success` on success — but those are folded into Goals here only by virtue of being adjacent; the focus of this SOW is the create/edit flows the user requested.
- Wrap `app/layout.tsx` `{children}` in `<ToastProvider>` so any page can call `useToast()`.

### Non-Goals

- **Adopting toasts on every existing form.** Bodyweight log/edit/delete, log-entry edit/delete, macro-goals modal, the chat page — all of these could benefit. They become follow-ups once the infrastructure exists; not in this SOW.
- **An animation library.** CSS `transition` covers the slide-down + fade. No Framer Motion, no react-spring.
- **Server-side toast queue / persistence across navigations.** Toasts are ephemeral client-state; clicking a link clears them. Persisting across navigations adds complexity the use case doesn't ask for.
- **Toast positioning options.** One placement: top-center of the viewport. No per-call override (`toast.success(msg, { position: "bottom-right" })`). YAGNI.
- **`info` / `warning` banner styles.** The hook exposes `info` for API symmetry but the visual style isn't shipped; first caller that needs it can ship the color treatment. `warning` is out entirely.
- **Replacing the modal's inline form-level error.** Both signals fire on server errors. The inline error is correctable; the toast is the notification. Removing the inline error would force the user to scroll up to the banner to read what's wrong with the field they just typed in.
- **Restyling the bodyweight log modal, macro-goals modal, quick-add modal, log-entry-edit modal, or the macro goals page.** Out of scope; their forms will follow this vocabulary in follow-up SOWs.
- **API / MCP / agent / mobile changes.** Pure web polish + new client infrastructure.
- **Form-state library (react-hook-form, formik).** Existing `useState`-per-field pattern stays. The forms are 7 fields max; a library would replace clear code with indirection.

## Implementation Details

### Toast system

New directory `components/toast/` with three files:

`components/toast/types.ts`:

```ts
export type ToastVariant = "success" | "error" | "info";

export type Toast = {
  id: string;
  variant: ToastVariant;
  message: string;
};

export type ToastContextValue = {
  toasts: Toast[];
  success: (message: string) => void;
  error: (message: string) => void;
  info: (message: string) => void;
  dismiss: (id: string) => void;
};
```

`components/toast/toast-provider.tsx` (client component):

```tsx
"use client";
import { createContext, useCallback, useContext, useMemo, useState } from "react";
import { ToastViewport } from "./toast-viewport";
import type { Toast, ToastContextValue, ToastVariant } from "./types";

const ToastContext = createContext<ToastContextValue | null>(null);

const AUTO_DISMISS_MS = { success: 4000, info: 4000, error: 0 } as const;
//                                                ^ 0 = never

export function ToastProvider({ children }: { children: React.ReactNode }) {
  const [toasts, setToasts] = useState<Toast[]>([]);

  const dismiss = useCallback((id: string) => {
    setToasts((prev) => prev.filter((t) => t.id !== id));
  }, []);

  const push = useCallback(
    (variant: ToastVariant, message: string) => {
      const id = crypto.randomUUID();
      setToasts((prev) => [...prev, { id, variant, message }]);
      const ms = AUTO_DISMISS_MS[variant];
      if (ms > 0) {
        setTimeout(() => dismiss(id), ms);
      }
    },
    [dismiss],
  );

  const value = useMemo<ToastContextValue>(
    () => ({
      toasts,
      success: (m) => push("success", m),
      error: (m) => push("error", m),
      info: (m) => push("info", m),
      dismiss,
    }),
    [toasts, push, dismiss],
  );

  return (
    <ToastContext.Provider value={value}>
      {children}
      <ToastViewport toasts={toasts} onDismiss={dismiss} />
    </ToastContext.Provider>
  );
}

export function useToast(): Omit<ToastContextValue, "toasts"> {
  const ctx = useContext(ToastContext);
  if (!ctx) {
    throw new Error("useToast must be used within a ToastProvider");
  }
  const { success, error, info, dismiss } = ctx;
  return { success, error, info, dismiss };
}
```

`components/toast/toast-viewport.tsx`:

```tsx
"use client";
import type { Toast } from "./types";

export function ToastViewport({ toasts, onDismiss }: { toasts: Toast[]; onDismiss: (id: string) => void }) {
  if (toasts.length === 0) return null;
  return (
    <div
      className="pointer-events-none fixed left-1/2 top-4 z-50 flex w-full max-w-md -translate-x-1/2 flex-col gap-2 px-4"
      role="region"
      aria-label="Notifications"
    >
      {toasts.map((t) => (
        <ToastBanner key={t.id} toast={t} onDismiss={() => onDismiss(t.id)} />
      ))}
    </div>
  );
}

function ToastBanner({ toast, onDismiss }: { toast: Toast; onDismiss: () => void }) {
  const palette = TONES[toast.variant];
  return (
    <div
      role={toast.variant === "error" ? "alert" : "status"}
      className={`pointer-events-auto flex items-start gap-3 rounded-md border border-[var(--border)] ${palette.bg} px-3 py-2 text-sm shadow-lg`}
      style={{ borderLeftWidth: "3px", borderLeftColor: palette.accent }}
    >
      <span
        aria-hidden
        className="mt-0.5 inline-flex h-4 w-4 flex-shrink-0 items-center justify-center rounded-full text-[10px] font-bold"
        style={{ background: palette.iconBg, color: palette.accent }}
      >
        {palette.glyph}
      </span>
      <p className="flex-1">{toast.message}</p>
      <button
        type="button"
        onClick={onDismiss}
        aria-label="Dismiss notification"
        className="text-[var(--muted)] transition hover:text-[var(--foreground)]"
      >
        ✕
      </button>
    </div>
  );
}

const TONES = {
  success: { accent: "#6ee7b7", bg: "bg-[var(--surface)]", iconBg: "rgba(110, 231, 183, 0.16)", glyph: "✓" },
  error:   { accent: "#f87171", bg: "bg-[var(--surface)]", iconBg: "rgba(248, 113, 113, 0.16)", glyph: "!" },
  info:    { accent: "var(--accent)", bg: "bg-[var(--surface)]", iconBg: "rgba(59, 130, 246, 0.16)", glyph: "i" },
} as const;
```

`components/toast/index.ts`:

```ts
export { ToastProvider, useToast } from "./toast-provider";
```

### Root layout wiring

`app/layout.tsx`:

```tsx
import { ToastProvider } from "@/components/toast";
// ... existing imports

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        <ToastProvider>{children}</ToastProvider>
      </body>
    </html>
  );
}
```

(Exact JSX adjusts to the existing layout's structure; the only change is the `<ToastProvider>` wrap around `{children}`.)

### Pantry item form

`components/pantry-item-form.tsx` rewrites with the new layout. The key changes:

- Drop the outer `<form className="... rounded-lg border ... bg-[var(--surface)] p-4">` card. The form renders into the modal body directly:

  ```tsx
  <form onSubmit={submit} className="flex flex-col gap-4">
    {/* top row + section divider + macros + buttons */}
  </form>
  ```

- Top row (replaces the nested-grid block at lines 87–128):

  ```tsx
  <div className="grid grid-cols-1 gap-3 sm:grid-cols-[2fr_1fr_1fr]">
    <Field label="Name">
      <input ... />
    </Field>
    <Field label="Serving size">
      <input type="number" ... className="tabular-nums" />
    </Field>
    <Field label="Serving unit">
      <input ... />
    </Field>
  </div>
  ```

  `Field` is a small local wrapper that owns the uppercase muted label + the input slot, so each row stays terse.

- Section divider:

  ```tsx
  <div className="flex items-center gap-3 pt-1">
    <span className="text-[10px] font-semibold uppercase tracking-wider text-[var(--muted)]">
      Per serving
    </span>
    <hr className="flex-1 border-[var(--border)]" />
  </div>
  ```

- Macro row uses an updated `MacroField` that takes `tone?: "protein" | "fat" | "carbs"`:

  ```tsx
  <div className="grid grid-cols-2 gap-3 sm:grid-cols-4">
    <MacroField label="Calories" value={calories} onChange={setCalories} disabled={busy} />
    <MacroField label="Protein (g)" value={proteinG} onChange={setProteinG} disabled={busy} tone="protein" />
    <MacroField label="Fat (g)" value={fatG} onChange={setFatG} disabled={busy} tone="fat" />
    <MacroField label="Carbs (g)" value={carbsG} onChange={setCarbsG} disabled={busy} tone="carbs" />
  </div>
  ```

  `MacroField` internals:

  ```tsx
  function MacroField({ label, value, onChange, disabled, tone }) {
    const color = tone ? MACRO_COLORS[tone] : null;
    return (
      <label className="flex flex-col gap-1 text-xs">
        <span className="inline-flex items-center gap-1.5 font-semibold uppercase tracking-wider text-[var(--muted)]">
          {color && (
            <span
              aria-hidden
              className="inline-block h-2 w-2 rounded-full"
              style={{ background: color }}
            />
          )}
          {label}
        </span>
        <input
          type="number"
          min={0}
          step="any"
          value={value}
          onChange={(e) => onChange(e.target.value)}
          disabled={disabled}
          className="rounded-md border border-[var(--border)] bg-[var(--surface)] px-3 py-2 text-sm tabular-nums focus:outline focus:outline-2 focus:outline-[var(--accent)]"
          style={color ? { borderLeftWidth: "2px", borderLeftColor: color } : undefined}
        />
      </label>
    );
  }
  ```

  Import `MACRO_COLORS` from `@/lib/macro-colors`.

- Inline error (`shownError`) keeps its current style + position above the button row. Don't fire a toast for local validation; the user is at the field.

- Button row:

  ```tsx
  <div className="flex items-center justify-end gap-2 pt-2">
    {onCancel && <button className="ghost-button">Cancel</button>}
    {onDelete && <button className="danger-ghost-button">Delete</button>}
    <button className="accent-button">{busy ? "Saving…" : submitLabel}</button>
  </div>
  ```

  Where `ghost-button`, `danger-ghost-button`, `accent-button` are local utility class compositions matching the existing button vocabulary already used in the bodyweight log modal and quick-add modal.

### Recipe form

`components/recipe-form.tsx` follows the same direction:

- Drop the outer surface card; render the form directly into the modal body.
- Name input matches the pantry-form input styling.
- `MacroPreview` tiles (one per macro) gain color: protein number in emerald, fat in pink, carbs in amber, calories neutral. Add a leading color dot in each tile's label:

  ```tsx
  function Tile({ label, value, tone }: { label: string; value: string; tone?: "protein" | "fat" | "carbs" }) {
    const color = tone ? MACRO_COLORS[tone] : null;
    return (
      <div className="text-center">
        <p className="text-sm font-semibold tabular-nums" style={color ? { color } : undefined}>
          {value}
        </p>
        <p className="inline-flex items-center justify-center gap-1.5 text-[10px] uppercase tracking-wider text-[var(--muted)]">
          {color && <span aria-hidden className="inline-block h-1.5 w-1.5 rounded-full" style={{ background: color }} />}
          {label}
        </p>
      </div>
    );
  }
  ```

- `ComponentRow`: keep the select, the quantity input, and the up/down arrows. Replace the `✕` remove button with the standardized `<TrashIcon />` (the same SVG markup already duplicated in `nutrition-log-view.tsx`, `pantry-view.tsx`, and `recipes-view.tsx`). The button itself uses `text-[var(--danger)]` on a transparent background with a `hover:bg-[var(--background)]` hint, matching the catalog row's trash icon.

- `+ Add component` becomes:

  ```tsx
  <button
    type="button"
    onClick={addComponent}
    disabled={busy || atCap}
    className="inline-flex items-center gap-1.5 text-xs font-medium text-[var(--foreground)] transition hover:opacity-70 disabled:opacity-40"
  >
    <PlusIcon />
    Add component
  </button>
  ```

  `PlusIcon` mirrors the one in `pantry-view.tsx` / `recipes-view.tsx`.

- Button row matches the pantry form.

### Modal wiring

`components/pantry-item-modal.tsx`:

```tsx
import { useToast } from "@/components/toast";

export function PantryItemModal(props: Props) {
  const { mode, token, onClose } = props;
  const toast = useToast();
  // ... existing state

  function handleSubmit(payload: PantryItemPayload) {
    setBusy(true);
    setError(null);
    const op = mode === "create"
      ? createPantryItem(token, payload)
      : updatePantryItem(token, props.initial.id, payload);
    op.then(() => {
      toast.success(
        mode === "create"
          ? `Added "${payload.name}" to your pantry.`
          : `Updated "${payload.name}".`,
      );
      props.onSaved();
      onClose();
    })
      .catch((err: unknown) => {
        const msg = err instanceof Error ? err.message : String(err);
        if (msg.toLowerCase().includes("401")) {
          clearToken();
          router.replace("/login");
          return;
        }
        setError(msg);
        toast.error(`Couldn't save: ${msg}`);
      })
      .finally(() => setBusy(false));
  }
  // ...
}
```

The toast fires alongside (not instead of) the existing inline `setError(msg)` — the inline error is the correctable hint at the form, the toast is the notification.

`components/recipe-modal.tsx`: same pattern using `createRecipe` / `updateRecipe` and recipe-specific success copy ("Added \"<name>\" as a recipe." / "Updated \"<name>\".").

The existing delete flow inside the pantry-item-modal's confirm-delete panel fires `toast.success("Deleted \"<name>\".")` on success and `toast.error(...)` on failure. The icon-driven `PantryItemDeleteModal` and `RecipeDeleteModal` (shipped in the prior nutrition-catalog-polish PR) gain the same wiring: success and error both toast.

### Visual consistency

The button row vocabulary already in use across the page:

- Ghost: `rounded-md border border-[var(--border)] bg-[var(--surface)] px-3 py-1.5 text-xs font-medium transition hover:opacity-80 disabled:opacity-50`.
- Danger-ghost: same with `border-[var(--danger)]/40 bg-[var(--danger)]/10 text-[var(--danger)]`.
- Accent (filled): `rounded-md bg-[var(--accent)] px-3 py-1.5 text-xs font-medium text-[var(--accent-fg)] transition hover:opacity-80 disabled:opacity-50`.

Reuse these verbatim; don't introduce a fourth variant.

The trash and plus icon SVG markup currently lives duplicated in three places (`nutrition-log-view.tsx`, `pantry-view.tsx`, `recipes-view.tsx`). The recipe-form picks up its own local copy too. The extraction-to-shared-module decision (`components/icons/`) is deliberately deferred — there's now a fourth occurrence, the natural threshold for DRY, but the existing four copies are already living shape that the SOW does not need to renegotiate to ship. A follow-up cleanup SOW can extract the icon set when it has another consumer's pull behind it.

### Tests

`prog-strength-web` has no test runner today. Verification:

- `npx tsc --noEmit` — clean
- `npx eslint components/toast components/pantry-item-form.tsx components/pantry-item-modal.tsx components/recipe-form.tsx components/recipe-modal.tsx app/layout.tsx` — clean
- `npx next build` — all 17 routes generate, `/nutrition` prerendered
- Hand-test checklist:
  - **Add pantry item, success:** open `/nutrition?view=pantry`, click Add, fill in valid values, click Save. The modal closes. A banner reads `Added "<name>" to your pantry.` and auto-dismisses after 4 seconds.
  - **Add pantry item, server error:** induce a server error (e.g., set `serving_unit` to a value the server rejects, or take down the API). The inline form error appears at the bottom of the form. A red banner reads `Couldn't save: <server message>` and stays until dismissed.
  - **Add pantry item, client validation:** clear the Name field, click Save. The inline form error appears. **No banner fires.**
  - **Edit pantry item:** click pencil on an existing item, change a macro, click Save changes. Modal closes. Banner reads `Updated "<name>".`
  - **Delete pantry item from inside the edit modal:** open edit, click Delete, confirm. Modal closes. Banner reads `Deleted "<name>".`
  - **Add recipe, success/error/validation:** same three cases against `/nutrition?view=recipes`.
  - **Edit recipe, components list:** reorder components with ↑/↓, remove with trash icon. Save. Banner reads `Updated "<name>".`
  - **Banner stacking:** trigger two saves back-to-back (e.g., dismiss the first banner mid-flight). Banners stack newest-at-bottom; oldest dismissable independently.
  - **Visual polish:** macro inputs show colored dot + colored left border; calories stays neutral. The "Per serving" divider reads as a small uppercase muted heading with a horizontal rule. Buttons match the rest of the page.
  - **Server vs client error split:** a 500 from the server toasts; a missing-name local validation does not.

### Rollout

Single PR on `prog-strength-web`. No coordination window. Merge whenever. Hand-test in the deployed environment after merge.

The toast system is intentionally narrow surface area — `ToastProvider`, `useToast()`, three variants. Once it ships, subsequent SOWs adopt it on a per-page basis without ceremony.
