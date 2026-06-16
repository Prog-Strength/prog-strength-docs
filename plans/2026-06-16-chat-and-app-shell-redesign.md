# Chat & App Shell Redesign Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Restyle the app shell and rebuild the `/chat` surface to the `soft-modern-messenger` direction — introducing a slate neutral ramp, a violet accent (replacing blue), Nunito type, and rounded/soft-shadow depth as named, reusable design tokens — while preserving all existing chat behavior.

**Architecture:** Three layers, built bottom-up. (1) Foundation tokens in `app/globals.css` + Nunito font in the root layout — the single source of truth. (2) Restyle the shared `components/sidebar.tsx` (visual only; same 12 destinations + IA). (3) Rebuild the chat page's render tree as a three-column messenger (nav rail · persistent conversation-list pane that retires the History drawer · thread), extracting the presentational pieces into focused `components/chat/*` components while the 1600-line page's logic stays intact. Presentation-only: no behavior, agent, tool, routing, or data changes.

**Tech Stack:** Next.js 16 App Router, React 19, TypeScript, Tailwind v4 (CSS variables + `@theme inline`), `next/font/google`, Vitest + Testing Library, `react-markdown` + `remark-gfm`.

---

## Constraints (read first)

- **Presentation only.** Do not change chat logic: streaming, voice (mic + TTS), image attach (5 MB), per-turn model routing, daily-usage cap, telemetry, session persistence/resume/delete. A behavior change is a regression.
- **Tokens, not hex.** Components reference CSS variables (`var(--accent)`, `var(--surface)`, …), never raw hex. The blue→violet flip happens once, at the token layer.
- **IA unchanged.** Same 12 nav destinations, same routes, same grouping.
- **No new lint warnings, no `--no-verify`, no `//eslint-disable` to dodge a rule, no skipped/weakened tests.** Existing pre-existing warnings (React 19 hooks rules) stay as-is; don't add new ones and don't mass-fix old ones.
- **The `design-explore` route is the visual spec, not code to copy.** It is not present in this checkout; the SOW's text is the spec. Reimplement against the real chat page, `Message`/`ToolCall` model, and `components/sidebar.tsx`.
- Run the gate locally before pushing: `npm run lint && npm run format:check && npm run typecheck && npm run test && npm run build`.

## File structure

**Modified:**
- `app/globals.css` — the token layer: slate ramp, violet accent (repoint), macro tints, radii, shadows, font var wiring.
- `app/layout.tsx` — swap Geist sans → Nunito via `next/font/google`; keep a mono.
- `components/sidebar.tsx` — restyle the rail (brand lockup, nav pills, user card). Visual only.
- `components/sidebar.test.tsx` — assert restyled rail: 12 destinations, active/idle, user card.
- `app/(app)/chat/page.tsx` — replace the render tree with the three-column messenger; preserve all hooks/logic; retire the drawer.
- `app/(app)/chat/page.test.tsx` — keep behavior tests (capped, 429, profile payload); adapt selectors to the new DOM.

**Created (chat presentational components, under `components/chat/`):**
- `components/chat/assistant-markdown.tsx` — the `react-markdown` renderer (moved from page) with a macro-table interception hook.
- `components/chat/macro-card.tsx` — parse a nutrition markdown table → styled card (per-item rows, Total row, P/F/C tinted chips). NEW logic; unit-tested.
- `components/chat/message-bubble.tsx` — rounded user (violet) / assistant (raised slate) bubbles; carries the model pill, tool chips, markdown/macro card, inline meal-photo.
- `components/chat/model-pill.tsx` — "via Sonnet 4.6".
- `components/chat/tool-pill.tsx` — running/ok/error tool-call chip.
- `components/chat/conversation-list.tsx` — the persistent "Chats" pane (search, new-chat, session list with active highlight; resume + delete) that replaces `ChatHistoryDrawer`.
- `components/chat/conversation-list.test.tsx` — sessions render; resume + delete still work.
- `components/chat/composer.tsx` — rounded composer (mic / attach / textarea / violet Send) with disabled-in-flight + capped states.
- `components/chat/chat-empty-state.tsx` — polished first-run state.
- `components/chat/icons.tsx` — shared inline SVG icons used by the chat components (moved from page).

**Deleted:**
- `components/chat/chat-history-drawer.tsx` — superseded by `conversation-list.tsx`.

---

## Task 1: Foundation tokens + Nunito

**Files:**
- Modify: `app/globals.css`
- Modify: `app/layout.tsx`

The token layer is the source of truth. Repointing `--accent` to violet flips every current consumer (user bubbles, Send, active nav, links, focus rings, the `border-[var(--accent)]/NN` opacity-modifier consumers) in one place. Keep `--accent` a **solid hex** so existing `bg-[var(--accent)]/15`-style opacity modifiers keep working; add `--accent-soft` / `--accent-line` as named tints for new code. Keep `--warning`, `--danger`, `--success`, `--warm-accent*`, and the `--discipline-*` tokens as-is (the shipped calendar consumes them; re-toning them is out of scope).

- [ ] **Step 1: Rewrite the `:root` block + `@theme inline` in `app/globals.css`.**

Replace the neutrals and accent; add macro tints, faint text, strong border, accent variants, radii, and shadows. Keep status + warm + discipline tokens.

```css
@import "tailwindcss";

/* Force dark. Soft-modern-messenger slate-dark system — the app's
   foundational visual language (see sows/chat-and-app-shell-redesign.md).
   One palette, no `dark:` variants. */
:root {
  /* Neutrals — slate ramp */
  --background: #15171c; /* base */
  --surface: #1e2128; /* panels / rails / bubbles */
  --surface-2: #272b33; /* raised — hover, chips */
  --surface-3: #2e333c; /* raised-2 */
  --foreground: #eef0f4; /* primary text */
  --muted: #9aa1ad; /* secondary text */
  --faint: #6b7280; /* tertiary text */
  /* Hairline borders — white-alpha so they read on any slate surface */
  --border: rgba(255, 255, 255, 0.07);
  --border-strong: rgba(255, 255, 255, 0.1);

  /* Accent — violet (replaces blue). Solid hex so `/NN` opacity
     modifiers on existing consumers keep working; soft/line are
     named tints for new code. */
  --accent: #8b7cf6;
  --accent-dark: #7765ec;
  --accent-fg: #ffffff;
  --accent-soft: rgba(139, 124, 246, 0.14);
  --accent-line: rgba(139, 124, 246, 0.3);

  /* Macro tints — macro card P/F/C chips (each ~13%-alpha bg) */
  --macro-protein: #34d399;
  --macro-protein-bg: rgba(52, 211, 153, 0.13);
  --macro-fat: #fbbf24;
  --macro-fat-bg: rgba(251, 191, 36, 0.13);
  --macro-carb: #60a5fa;
  --macro-carb-bg: rgba(96, 165, 250, 0.13);

  /* Status */
  --warning: #fbbf24;
  --danger: #ef4444;
  --success: #34d399;

  /* Radii + soft depth */
  --radius-card: 1rem; /* rounded-2xl */
  --radius-card-lg: 1.5rem; /* rounded-3xl */
  --radius-pill: 9999px;
  --shadow-soft: 0 1px 2px rgba(0, 0, 0, 0.3), 0 8px 24px rgba(0, 0, 0, 0.22);
  --shadow-raised: 0 1px 2px rgba(0, 0, 0, 0.35), 0 2px 6px rgba(0, 0, 0, 0.25);

  /* Warm-organic coaching tokens (shipped calendar). Unchanged — the
     accent flip is blue→violet only; warm-accent is a separate token. */
  --warm-accent: #e08a5e;
  --warm-accent-fg: #1b1206;

  --discipline-run-bg: #3a241a;
  --discipline-run-fg: #f0b48f;
  --discipline-run-dot: #e08a5e;
  --discipline-lift-bg: #332a16;
  --discipline-lift-fg: #e8c878;
  --discipline-lift-dot: #d6ab54;
  --discipline-mobility-bg: #1f3330;
  --discipline-mobility-fg: #8fd6c4;
  --discipline-mobility-dot: #4fbfa3;
  --discipline-core-bg: #2c2440;
  --discipline-core-fg: #c2adf0;
  --discipline-core-dot: #9a7fe0;
}

@theme inline {
  --color-background: var(--background);
  --color-foreground: var(--foreground);
  --color-surface: var(--surface);
  --color-surface-2: var(--surface-2);
  --color-surface-3: var(--surface-3);
  --color-border: var(--border);
  --color-border-strong: var(--border-strong);
  --color-muted: var(--muted);
  --color-faint: var(--faint);
  --color-accent: var(--accent);
  --color-accent-dark: var(--accent-dark);
  --color-accent-fg: var(--accent-fg);
  --color-accent-soft: var(--accent-soft);
  --color-accent-line: var(--accent-line);
  --color-macro-protein: var(--macro-protein);
  --color-macro-protein-bg: var(--macro-protein-bg);
  --color-macro-fat: var(--macro-fat);
  --color-macro-fat-bg: var(--macro-fat-bg);
  --color-macro-carb: var(--macro-carb);
  --color-macro-carb-bg: var(--macro-carb-bg);
  --color-warning: var(--warning);
  --color-danger: var(--danger);
  --color-success: var(--success);
  --color-warm-accent: var(--warm-accent);
  --color-warm-accent-fg: var(--warm-accent-fg);
  --color-discipline-run-bg: var(--discipline-run-bg);
  --color-discipline-run-fg: var(--discipline-run-fg);
  --color-discipline-run-dot: var(--discipline-run-dot);
  --color-discipline-lift-bg: var(--discipline-lift-bg);
  --color-discipline-lift-fg: var(--discipline-lift-fg);
  --color-discipline-lift-dot: var(--discipline-lift-dot);
  --color-discipline-mobility-bg: var(--discipline-mobility-bg);
  --color-discipline-mobility-fg: var(--discipline-mobility-fg);
  --color-discipline-mobility-dot: var(--discipline-mobility-dot);
  --color-discipline-core-bg: var(--discipline-core-bg);
  --color-discipline-core-fg: var(--discipline-core-fg);
  --color-discipline-core-dot: var(--discipline-core-dot);
  --font-sans: var(--font-nunito);
  --font-mono: var(--font-geist-mono);
}

html,
body {
  background: var(--background);
  color: var(--foreground);
  font-family: var(--font-nunito), ui-rounded, system-ui, sans-serif;
}
```

- [ ] **Step 2: Swap Geist sans → Nunito in `app/layout.tsx`.**

Nunito is a variable font on `next/font/google` (no explicit weight array needed — weights 600/700/800 resolve from the variable axis). Keep `Geist_Mono` for code.

```tsx
import type { Metadata } from "next";
import { Nunito, Geist_Mono } from "next/font/google";
import "./globals.css";
import { Providers } from "./providers";
import { ToastProvider } from "@/components/toast";
import { DistanceUnitProvider } from "@/lib/distance-unit-context";

const nunito = Nunito({
  variable: "--font-nunito",
  subsets: ["latin"],
});

const geistMono = Geist_Mono({
  variable: "--font-geist-mono",
  subsets: ["latin"],
});

export const metadata: Metadata = {
  title: "Prog Strength",
  description: "Natural-language strength training tracker.",
};

export default function RootLayout({
  children,
}: Readonly<{
  children: React.ReactNode;
}>) {
  return (
    <html
      lang="en"
      className={`${nunito.variable} ${geistMono.variable} h-full antialiased dark`}
    >
      <body className="min-h-full flex flex-col bg-[var(--background)] text-[var(--foreground)]">
        <Providers>
          <ToastProvider>
            <DistanceUnitProvider>{children}</DistanceUnitProvider>
          </ToastProvider>
        </Providers>
      </body>
    </html>
  );
}
```

- [ ] **Step 3: Verify the gate.**

Run: `npm run typecheck && npm run lint && npm run build`
Expected: typecheck clean, no NEW lint warnings, build succeeds. (No test for pure token/CSS values.)

- [ ] **Step 4: Commit.**

```bash
git add app/globals.css app/layout.tsx
git commit -m "feat(design-system): soft-modern-messenger foundation tokens + Nunito"
```

---

## Task 2: Shared shell restyle

**Files:**
- Modify: `components/sidebar.tsx`
- Modify: `components/sidebar.test.tsx`

Visual only. Same `NAV` array (12 destinations), same routes, same collapse behavior, same `AccountAnchor` menu logic. Restyle:
- **Brand badge lockup**: the `BrandMark` inside a rounded violet-tinted badge (`bg-[var(--accent-soft)]`, `text-[var(--accent)]`, `rounded-xl`) next to a two-line lockup — "Prog Strength" (semibold) over "AI Coach" (muted, `text-[10px]`). "AI Coach" is a static product descriptor (not invented user status) — keep it. Collapsed mode keeps the badge-as-toggle.
- **Nav pills**: `rounded-[var(--radius-pill)]`-ish (`rounded-full` or `rounded-xl` per messenger feel — use `rounded-xl`), idle = `text-[var(--muted)]` with `hover:bg-[var(--surface-2)]`; **active = `bg-[var(--accent-soft)]` + `border border-[var(--accent-line)]` + `text-[var(--accent)]` (violet glyph + label)**. Keep `aria-current="page"` on active.
- **User card**: pinned at bottom (already is). Keep the initials/avatar + display name. Replace any invented "Free plan" line with a **real** secondary line — the user's email (`profile?.email`), muted, truncated. Drop it gracefully when absent. Keep the popover menu (Sign out) and all its a11y wiring.

Keep all existing inline SVG icon components in this file (the sidebar's icons are sidebar-private).

- [ ] **Step 1: Update `sidebar.test.tsx` expectations FIRST (TDD), then implement.**

The existing tests already assert display name, initials "SL", avatar img, menu open/close, sign-out, collapse. Keep all of them. ADD:
- the rail renders all 12 destinations (assert each label is present),
- the active entry (`/chat`, since `usePathname` is mocked to `/chat`) carries `aria-current="page"`,
- the user card shows the email secondary line (`lifter@example.com`).

```tsx
it("renders all 12 nav destinations", () => {
  render(<Sidebar />);
  for (const label of [
    "Chat", "Timeline", "Search", "Requests", "Activities", "Exercises",
    "Calendar", "Nutrition", "Bodyweight", "Progress", "Personal Records", "Settings",
  ]) {
    expect(screen.getByRole("link", { name: label })).toBeInTheDocument();
  }
});

it("marks the active route with aria-current", () => {
  render(<Sidebar />);
  expect(screen.getByRole("link", { name: "Chat" })).toHaveAttribute("aria-current", "page");
});

it("shows the user email as the account secondary line", () => {
  render(<Sidebar />);
  expect(screen.getByText("lifter@example.com")).toBeInTheDocument();
});
```

(The existing `profileCtx()` already provides `email: "lifter@example.com"`. Note the active-link assertion: a collapsed-sidebar test must not run before the restore effect; the new tests run in the default expanded state.)

- [ ] **Step 2: Run the new tests to confirm they fail.**

Run: `npm run test -- components/sidebar.test.tsx`
Expected: the new assertions FAIL (email line / aria-current not yet present in restyled form), existing ones still pass.

- [ ] **Step 3: Implement the restyle in `components/sidebar.tsx`.**

Brand lockup (expanded state) replaces the wordmark `<Link>`:
```tsx
<Link
  href="/chat"
  aria-label="Prog Strength home"
  className="flex min-w-0 items-center gap-2.5 transition hover:opacity-90"
>
  <span className="flex h-9 w-9 shrink-0 items-center justify-center rounded-xl bg-[var(--accent-soft)] text-[var(--accent)]">
    <BrandMark size={20} />
  </span>
  <span className="flex min-w-0 flex-col leading-tight">
    <span className="truncate text-sm font-extrabold text-[var(--foreground)]">Prog Strength</span>
    <span className="truncate text-[10px] font-semibold text-[var(--muted)]">AI Coach</span>
  </span>
</Link>
```
Nav pill className:
```tsx
className={`flex items-center gap-3 rounded-xl px-2.5 py-2 text-sm font-semibold transition ${
  active
    ? "border border-[var(--accent-line)] bg-[var(--accent-soft)] text-[var(--accent)]"
    : "border border-transparent text-[var(--muted)] hover:bg-[var(--surface-2)] hover:text-[var(--foreground)]"
}`}
```
User card: in `AccountAnchor`, when expanded, render the display name over the muted email line:
```tsx
{!collapsed && (
  <span className="flex min-w-0 flex-col leading-tight text-left">
    <span className="truncate text-sm font-semibold text-[var(--foreground)]" title={displayName}>
      {displayName}
    </span>
    {email && <span className="truncate text-[11px] text-[var(--muted)]">{email}</span>}
  </span>
)}
```
where `const email = profile?.email ?? null;`. Keep the avatar, the menu, Escape/outside-click, sign-out exactly as-is. Restyle the aside container with `bg-[var(--surface)]` (now slate) and keep the hairline border.

- [ ] **Step 4: Run sidebar tests + typecheck.**

Run: `npm run test -- components/sidebar.test.tsx && npm run typecheck`
Expected: all sidebar tests PASS, typecheck clean.

- [ ] **Step 5: Commit.**

```bash
git add components/sidebar.tsx components/sidebar.test.tsx
git commit -m "feat(shell): restyle sidebar rail to soft-modern-messenger"
```

---

## Task 3: Chat presentational components

**Files:**
- Create: `components/chat/icons.tsx`
- Create: `components/chat/model-pill.tsx`
- Create: `components/chat/tool-pill.tsx`
- Create: `components/chat/macro-card.tsx`
- Create: `components/chat/macro-card.test.tsx`
- Create: `components/chat/assistant-markdown.tsx`
- Create: `components/chat/message-bubble.tsx`
- Create: `components/chat/chat-empty-state.tsx`
- Create: `components/chat/composer.tsx`
- Create: `components/chat/conversation-list.tsx`
- Create: `components/chat/conversation-list.test.tsx`

Extract the presentational pieces currently inlined in `app/(app)/chat/page.tsx` into focused components, restyled to the new language, **with identical rendered behavior** except where the SOW asks for new visuals (rounded bubbles, macro card). The page (Task 4) imports these. Shared types `Message` and `ToolCall` move to a small module the components and page both import.

### Shared types

Put `ToolCall` and `Message` in `components/chat/types.ts` (re-exported by the page if convenient), so components type their props without importing from the page:
```ts
import type { ContentBlock } from "@/lib/agent";

export type ToolCall = { name: string; state: "running" | "ok" | "error" };

export type Message = {
  role: "user" | "assistant";
  content: string | ContentBlock[];
  tools?: ToolCall[];
  model?: string;
};
```

- [ ] **Step 1: `components/chat/icons.tsx`** — move the chat icon components (`MicIcon`, `PlusIcon`, `SendIcon`, `PaperclipIcon`, `SpeakerIcon`, `DotsIcon`, `CheckIcon`, `XIcon`, plus a `SearchIcon` for the pane search and a `TrashIcon` for the session row) out of the page/drawer verbatim, exported individually. No visual change to the glyphs themselves.

- [ ] **Step 2: `components/chat/model-pill.tsx`** — move `ModelLabel` + `humanizeModelName` here, exported as `ModelPill`. Same muted `via {name}` text, tuned to the new tokens (`text-[var(--muted)]`).

- [ ] **Step 3: `components/chat/tool-pill.tsx`** — move `ToolPill` + `humanizeToolName`. Restyle chips to rounded-pill on slate: running = `animate-pulse` + `bg-[var(--surface-2)]` + muted; ok = `bg-[var(--accent-soft)]` + `border-[var(--accent-line)]` + `text-[var(--accent)]` + check; error = danger-tinted + x. Same three states, same `humanizeToolName`.

- [ ] **Step 4 (TDD): `components/chat/macro-card.test.tsx`** — write failing tests for a `parseMacroTable(headers, rows)` pure helper and the `MacroCard` render.

`MacroCard` renders a styled nutrition card from a GFM table. The helper detects a macro-shaped table (headers, lowercased, include at least two of `protein`/`carb`/`fat`, or the `p`/`f`/`c` short forms) and returns `null` for non-macro tables (so the markdown renderer falls back to a plain table). Numbers parse leniently (strip non-numeric except `.`; `"32g"` → 32).

```tsx
import { parseMacroTable } from "./macro-card";

it("parses a standard nutrition table", () => {
  const parsed = parseMacroTable(
    ["Item", "Calories", "Protein", "Carbs", "Fat"],
    [
      ["Chicken breast", "165", "31g", "0g", "3.6g"],
      ["Rice", "205", "4g", "45g", "0.4g"],
      ["Total", "370", "35g", "45g", "4g"],
    ],
  );
  expect(parsed).not.toBeNull();
  expect(parsed!.items).toHaveLength(2);
  expect(parsed!.total).toEqual(
    expect.objectContaining({ label: "Total", protein: 35, carbs: 45, fat: 4 }),
  );
});

it("returns null for a non-macro table", () => {
  expect(parseMacroTable(["Exercise", "Sets", "Reps"], [["Squat", "3", "5"]])).toBeNull();
});

it("synthesizes a total when no Total row is present", () => {
  const parsed = parseMacroTable(
    ["Food", "Protein", "Carbs", "Fat"],
    [
      ["Eggs", "12", "1", "10"],
      ["Toast", "6", "24", "2"],
    ],
  );
  expect(parsed!.total).toEqual(expect.objectContaining({ protein: 18, carbs: 25, fat: 12 }));
});
```

- [ ] **Step 5: Run macro-card tests to confirm they fail.**

Run: `npm run test -- components/chat/macro-card.test.tsx`
Expected: FAIL ("parseMacroTable is not a function").

- [ ] **Step 6: Implement `components/chat/macro-card.tsx`.**

Define the parsed shape + helper + component:
```tsx
export type MacroRow = {
  label: string;
  calories: number | null;
  protein: number | null;
  carbs: number | null;
  fat: number | null;
};
export type ParsedMacros = { items: MacroRow[]; total: MacroRow };

// Lenient numeric parse: "32g" -> 32, "1,015" -> 1015, "—" -> null.
function num(cell: string): number | null {
  const cleaned = cell.replace(/[^0-9.]/g, "");
  if (!cleaned) return null;
  const n = Number(cleaned);
  return Number.isFinite(n) ? n : null;
}

// Column index for the first header matching any alias (case-insensitive).
function colIndex(headers: string[], aliases: string[]): number {
  const lower = headers.map((h) => h.trim().toLowerCase());
  return lower.findIndex((h) => aliases.some((a) => h === a || h.startsWith(a)));
}

export function parseMacroTable(headers: string[], rows: string[][]): ParsedMacros | null {
  const pIdx = colIndex(headers, ["protein", "p"]);
  const cIdx = colIndex(headers, ["carb", "carbs", "carbohydrate", "c"]);
  const fIdx = colIndex(headers, ["fat", "f"]);
  // Need at least two of the three macro columns to call it a macro table.
  const present = [pIdx, cIdx, fIdx].filter((i) => i >= 0).length;
  if (present < 2) return null;
  const calIdx = colIndex(headers, ["calorie", "calories", "kcal", "cal"]);
  const labelIdx = 0;

  const toRow = (cells: string[]): MacroRow => ({
    label: (cells[labelIdx] ?? "").trim(),
    calories: calIdx >= 0 ? num(cells[calIdx] ?? "") : null,
    protein: pIdx >= 0 ? num(cells[pIdx] ?? "") : null,
    carbs: cIdx >= 0 ? num(cells[cIdx] ?? "") : null,
    fat: fIdx >= 0 ? num(cells[fIdx] ?? "") : null,
  });

  const all = rows.map(toRow);
  const isTotal = (r: MacroRow) => /^(total|totals|sum|daily total)$/i.test(r.label.trim());
  const totalRow = all.find(isTotal);
  const items = all.filter((r) => !isTotal(r));
  if (items.length === 0) return null;

  const sum = (key: "calories" | "protein" | "carbs" | "fat") => {
    const vals = items.map((r) => r[key]).filter((v): v is number => v != null);
    return vals.length ? vals.reduce((a, b) => a + b, 0) : null;
  };
  const total: MacroRow = totalRow ?? {
    label: "Total",
    calories: sum("calories"),
    protein: sum("protein"),
    carbs: sum("carbs"),
    fat: sum("fat"),
  };
  return { items, total };
}
```
Then a `MacroCard({ parsed }: { parsed: ParsedMacros })` component: a `rounded-[var(--radius-card)]` card on `bg-[var(--surface-2)]` with a hairline border; per-item rows (label + small macro figures); a divider; a **Total** row; and three tinted **P / F / C chips** using `--macro-protein*`, `--macro-fat*`, `--macro-carb*` (e.g. `bg-[var(--macro-protein-bg)] text-[var(--macro-protein)]`). Show grams; show calories when present.

- [ ] **Step 7: Run macro-card tests to confirm they pass.**

Run: `npm run test -- components/chat/macro-card.test.tsx`
Expected: PASS.

- [ ] **Step 8: `components/chat/assistant-markdown.tsx`** — move `AssistantMarkdown` here. Keep every element mapping; restyle to new tokens. **Add macro interception:** the `table` component reads its `node` (hast) to extract header cells and body rows; call `parseMacroTable(headers, rows)`; if non-null render `<MacroCard parsed={…} />`, else render the existing styled `<table>`. Helper to walk the hast node:
```tsx
function extractTable(node: unknown): { headers: string[]; rows: string[][] } | null {
  // node is a hast <table> element; walk thead/tbody → tr → th/td,
  // concatenating text node values. Defensive: return null on any
  // shape surprise so we fall back to the plain table renderer.
}
```
Keep `react-markdown` safe-by-default (no raw HTML). If table-node walking is awkward, the acceptable fallback is to keep the plain table renderer and detect macros one level up in `message-bubble` from the raw `content` string — but prefer the in-renderer interception so non-macro tables are untouched.

- [ ] **Step 9: `components/chat/message-bubble.tsx`** — move `MessageBubble` + `ImageMessage` here, importing `ModelPill`, `ToolPill`, `AssistantMarkdown`, and `Message`/`ToolCall` types. Restyle: user bubble `bg-[var(--accent)] text-[var(--accent-fg)] rounded-2xl rounded-br-md`; assistant bubble `bg-[var(--surface-2)] text-[var(--foreground)] rounded-2xl rounded-bl-md shadow-[var(--shadow-soft)]`. Keep the typing-placeholder (`…` animate-pulse), the metadata row (model pill + tool chips), the multimodal `ImageMessage` (inline meal photo from base64 data URL), and the string/markdown branches **identical in logic**.

- [ ] **Step 10: `components/chat/chat-empty-state.tsx`** — a polished first-run state: brand badge, "Ask about your training.", and 2–3 example-prompt chips. Pure presentational; accept an optional `onPickExample?: (text: string) => void` so Task 4 can wire the chips to the composer (if wiring is non-trivial, render them as static styled chips — no behavior change required by the SOW beyond the empty state existing).

- [ ] **Step 11: `components/chat/composer.tsx`** — extract the footer composer as a controlled presentational component. Props mirror the page's existing handlers/state so **no logic moves** — the page still owns state:
```tsx
type ComposerProps = {
  input: string;
  onInputChange: (v: string) => void;
  onSend: () => void;
  onKeyDown: (e: React.KeyboardEvent<HTMLTextAreaElement>) => void;
  onAttachClick: () => void;
  onPaste: (e: React.ClipboardEvent<HTMLTextAreaElement>) => void;
  fileInputRef: React.RefObject<HTMLInputElement | null>;
  onFileChange: (e: React.ChangeEvent<HTMLInputElement>) => void;
  pendingImage: { previewUrl: string; filename: string } | null;
  onDismissImage: () => void;
  speechSupported: boolean;
  listening: boolean;
  onMicDown: () => void;
  onMicUp: () => void;
  capped: boolean;
  cappedTooltip: string;
  streaming: boolean;
  loading: boolean;
  sessionId: string | null;
  placeholder: string;
};
```
Render the rounded composer: mic (44×44), attach (44×44), the auto-sizing textarea on `bg-[var(--surface-2)]` with `rounded-2xl` + violet focus ring (`focus:border-[var(--accent)]`), and the violet Send. Keep the staged-image chip, the disabled-in-flight + capped logic, the drag/drop on the outer footer (the page keeps the `onDragOver`/`onDrop` since it owns `onSelectImage`). The mic must render only when `speechSupported`.

- [ ] **Step 12 (TDD): `components/chat/conversation-list.test.tsx`** — failing tests for the new pane (it replaces the drawer). Model the existing drawer behavior tests. Mock `@/lib/api` (`listChatSessions`, `deleteChatSession`) and `next/navigation`.

```tsx
it("renders the sessions returned by the API", async () => {
  listChatSessionsMock.mockResolvedValue([
    { id: "s1", title: "Leg day plan", message_count: 4, last_message_at: new Date().toISOString(), /* …ChatSession fields */ },
  ]);
  render(<ConversationList activeSessionId={null} />);
  expect(await screen.findByText("Leg day plan")).toBeInTheDocument();
});

it("resumes a session on click", async () => {
  // …click the row → router.push("/chat?session=s1")
});

it("deletes a session (optimistic) after confirm", async () => {
  // window.confirm stubbed true → row disappears, deleteChatSession called
});
```

- [ ] **Step 13: Run conversation-list tests to confirm they fail.**

Run: `npm run test -- components/chat/conversation-list.test.tsx`
Expected: FAIL (component not implemented).

- [ ] **Step 14: Implement `components/chat/conversation-list.tsx`.**

A persistent left-of-thread column ("Chats"). Reuse the drawer's data logic (`listChatSessions`, `deleteChatSession`, optimistic remove, 401 handling, `formatRelative`) — but as an always-mounted pane, not a slide-in drawer. Props:
```tsx
type ConversationListProps = {
  activeSessionId: string | null;
  onNewChat?: () => void; // optional; page wires to router.push("/chat")
  className?: string;
};
```
- Header: "Chats" title + a new-chat (`+`) button (calls `onNewChat`).
- A search pill (`SearchIcon` + text input) that filters the rendered sessions client-side by title (case-insensitive). Local state only — no API change.
- Session list: each row = brand-mark avatar + title + a muted line (`{message_count} messages · {relative time}`), active row highlighted with `bg-[var(--accent-soft)]` + `border-[var(--accent-line)]`. Row click resumes via `router.push("/chat?session=…")`; trash deletes (confirm → optimistic remove → if active, `router.push("/chat")`). Fetch the list on mount and expose no `open` prop (always live). Loading + empty + error states like the drawer.

- [ ] **Step 15: Run all new component tests + typecheck.**

Run: `npm run test -- components/chat && npm run typecheck`
Expected: all PASS, typecheck clean. (Note: the page is not yet wired; `chat-history-drawer.tsx` still exists and still compiles. Don't delete it until Task 4.)

- [ ] **Step 16: Commit.**

```bash
git add components/chat/
git commit -m "feat(chat): extract restyled presentational components + macro card"
```

---

## Task 4: Rebuild chat page three-column

**Files:**
- Modify: `app/(app)/chat/page.tsx`
- Modify: `app/(app)/chat/page.test.tsx`
- Delete: `components/chat/chat-history-drawer.tsx`

Replace the page's **render tree** with the three-column messenger, wiring the Task 3 components. **Keep every hook, effect, ref, and the entire `send` callback byte-for-byte** (state, SSE loop, voice, telemetry, cap, persistence). Only JSX in the `return (...)` changes, plus: import the extracted components instead of the inline definitions (remove the now-moved inline component/helper/icon definitions from the page), and replace `ChatHistoryDrawer` usage with the persistent `ConversationList` pane.

Layout target (desktop ≥ `lg`): a flex row inside `<main>` — the `ConversationList` pane (fixed width, e.g. `w-72`, `border-r`, `bg-[var(--surface)]`) on the left of the thread; the thread column (header + scroller + composer) fills the rest. On narrow screens (`< lg`): the pane is hidden by default and toggled open as an overlay sheet by a "Chats" button in the chat header (reuse the drawer's overlay/Escape pattern, now driven by a local `paneOpen` state that only matters below `lg`). This answers the SOW's responsive open question: persistent on desktop, sheet on mobile — the pane never crowds the thread.

Header: assistant avatar (brand mark) with a small presence dot, "Chat" title + a muted model/status line, and the existing Voice on/off + New chat controls. The old "History" pill is removed (the pane replaces it); on mobile add a "Chats" toggle button instead.

- [ ] **Step 1: Update `app/(app)/chat/page.test.tsx` for the new DOM (keep all behavior assertions).**

The three existing suites — capped state (banner + disabled Send/mic/voice), 429 convergence (toast + refresh + optimistic strip), profile payload (display_name/height_cm, and the stale-closure regression) — MUST still pass. Adapt only selectors that the restyle changes:
- The composer/Send/mic/voice `aria-label`s are preserved verbatim by Task 3's composer, so `getByLabelText("Send message")`, `getByLabelText(/voice input/i)`, `getByLabelText("Turn voice mode on")`, `getByPlaceholderText("Message Prog Strength…")` keep working — verify and keep.
- If the test mounts the page and the new `ConversationList` fires `listChatSessions` on mount, add it to the `@/lib/api` mock (return `[]`) so the test doesn't hit a real fetch. Add `listChatSessions: vi.fn(async () => [])` and `deleteChatSession: vi.fn()` to the existing `vi.mock("@/lib/api", …)` factory.

- [ ] **Step 2: Run the chat page tests to see the current state.**

Run: `npm run test -- "app/(app)/chat/page.test.tsx"`
Expected: They may fail to mount until the new imports/mocks line up — that's the integration signal for Step 3.

- [ ] **Step 3: Rebuild the page render tree.**

- Remove the inline `MessageBubble`, `ImageMessage`, `ModelLabel`, `ToolPill`, `AssistantMarkdown`, the icon components, and the `humanize*` helpers now living in `components/chat/*`. Keep `titleAndPatch`, `fallbackTitle`, `persistedToUI`, `replaceLast`, and `acceptImageFile` in the page (they're logic, not presentation) — or move the pure ones to a `lib`/`components/chat` helper only if cleaner; default to leaving them.
- Import `MessageBubble`, `Composer`, `ConversationList`, `ChatEmptyState`, and the shared `Message`/`ToolCall` types.
- Keep `const [historyOpen, setHistoryOpen]` repurposed as `const [paneOpen, setPaneOpen]` for the mobile sheet (or add a new one and drop `historyOpen`).
- New `return`:
```tsx
return (
  <div className="flex flex-1 overflow-hidden">
    <ConversationList
      activeSessionId={sessionId}
      onNewChat={startNewChat}
      className="hidden lg:flex"
    />
    {/* mobile sheet: same component in an overlay when paneOpen */}
    <main className="flex flex-1 flex-col overflow-hidden">
      <ChatHeader … voiceMode … onToggleVoice … onNewChat … onOpenPane={() => setPaneOpen(true)} capped … />
      <div ref={scrollerRef} className="flex-1 overflow-y-auto px-4 py-6 sm:px-6" aria-live="polite">
        <div className="mx-auto flex max-w-2xl flex-col gap-4">
          {messages.length === 0 && <ChatEmptyState />}
          {messages.map((m, i) => (
            <MessageBubble key={i} role={m.role} content={m.content} tools={m.tools} model={m.model} />
          ))}
          {error && <div className="rounded-[var(--radius-card)] border border-[var(--danger)]/40 bg-[var(--danger)]/10 px-3 py-2 text-sm text-[var(--danger)]">{error}</div>}
        </div>
      </div>
      <footer onDragOver={…} onDrop={…} className="border-t border-[var(--border)] px-3 py-3 sm:px-6 sm:py-4">
        {capped && <…cap banner…/>}
        {voiceForcedOff && <…note…/>}
        <Composer … all the props … />
      </footer>
    </main>
  </div>
);
```
The `ChatHeader` can be a small inline subcomponent in the page or a `components/chat/chat-header.tsx` — either is fine; keep Voice/New chat handlers wired to the existing callbacks. Preserve `aria-label`s ("Turn voice mode on/off", "Start a new chat", "Send message", "Start/Stop voice input", "Attach image").
- Delete the `<ChatHistoryDrawer …/>` element and its import.

- [ ] **Step 4: Delete the retired drawer.**

```bash
git rm components/chat/chat-history-drawer.tsx
```
Confirm nothing else imports it: `grep -rn "chat-history-drawer\|ChatHistoryDrawer" app components lib` returns nothing.

- [ ] **Step 5: Run the full gate.**

Run: `npm run typecheck && npm run lint && npm run test && npm run build`
Expected: typecheck clean, no NEW lint warnings, all tests pass (including the adapted chat page tests + new component tests), build succeeds.

- [ ] **Step 6: Commit.**

```bash
git add -A
git commit -m "feat(chat): three-column messenger surface, retire history drawer"
```

---

## Task 5: Docs updates + ship flip (prog-strength-docs)

**Files (in `/workspace/prog-strength-docs`, branch `feat/chat-and-app-shell-redesign`):**
- Modify: `sows/chat-and-app-shell-redesign.md` — frontmatter `status: shipped`; body `**Status**: Shipped`; `**Last updated**: 2026-06-16`.
- Modify: `dx/chat-and-app-shell.md` — frontmatter `status: selected` and record the selected idiom `soft-modern-messenger` (per the SOW Rollout step 2). Only if not already `selected`.
- Modify: `sows/calendar-view-redesign.md` — update the **Design tokens / palette** section to reference the shared violet-on-slate tokens instead of introducing a clay accent (per the SOW's "Interactions" + Goals). Keep the calendar's structure/streak/coaching feel; re-point the accent + surfaces to the shared tokens.

This is a docs-only repo change; no code gate. Commit:
```bash
git commit -m "docs: mark chat-and-app-shell-redesign as shipped"
```
(If multiple doc edits, they can be one commit or split: the SOW status flip commit subject is the required one.)

---

## Self-Review (controller checklist after writing)

**Spec coverage:** tokens ✔ (Task 1), shared shell ✔ (Task 2), three-column messenger + conversation pane retiring the drawer ✔ (Tasks 3–4), bubbles/model pill/tool chips/macro card/meal photo/empty/typing/error/composer ✔ (Tasks 3–4), behavior preserved ✔ (constraint + Task 4 keeps logic), accent flip tokenized ✔ (Task 1), calendar SOW + docs record ✔ (Task 5), CI gate ✔ (each task + final). Responsive pane + "AI Coach"/"Free plan" open questions answered (Tasks 2, 4).

**Placeholders:** none — code shown for the novel pieces (tokens, macro parser, component shapes). The page rebuild deliberately preserves existing logic rather than restating 1600 lines.

**Type consistency:** `Message`/`ToolCall` defined once (`components/chat/types.ts`), `ParsedMacros`/`MacroRow` from macro-card, `ConversationListProps`/`ComposerProps` as specified.
