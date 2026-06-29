# Plan: Running Heart-Rate Zones — Ranked Bars

Implements [`sows/running-heart-rate-zones-ranked-bars.md`](../sows/running-heart-rate-zones-ranked-bars.md).

Display-only refactor of the running-detail HR-zones widget from a single 100%-stacked
bar + legend into a ranked list of five per-zone horizontal bars (high→low), plus a
retune of the `--zone-1..5` token scale to a separable, calm, cool→warm palette. No
backend, API, or data-model changes.

## Repos

- **prog-strength-web** — the component, the tokens, the tests.
- **prog-strength-docs** — design-system v0.4.2 changelog + Color note, and the SOW status flip.

## Task 1 — Retune `--zone-1..5` in `prog-strength-web/app/globals.css`

Replace the five periwinkle hex values (lines ~75-79) with the separable cool→warm
scale from the SOW table, and rewrite the comment block above them.

| Token | Value |
| --- | --- |
| `--zone-1` | `#6b7280` |
| `--zone-2` | `#6f93c2` |
| `--zone-3` | `#6fb39a` |
| `--zone-4` | `#d0a86e` |
| `--zone-5` | `#cc8077` |

- Keep token names and the `--color-zone-1..5` Tailwind aliases unchanged — only hex
  values and the comment change.
- Comment block: describe the new separable cool→warm scale (slate → dusty blue →
  muted teal-green → muted amber → dusty coral), drop the "Z5 == --accent" note, and
  note it sits at the status-tone saturation register, distinct from the accent.

**Verify**: `grep` confirms `--zone-*` is referenced only in `globals.css` and
`HeartRateZones.tsx` (blast radius contained).

## Task 2 — Rebuild `HeartRateZones.tsx` as ranked bars

Keep: prop contract `{ zones: HeartRateZones | null | undefined }`, the early return
(`null` when `!zones || zones.zones.length === 0`), card chrome, the uppercase
`HEART RATE ZONES` heading, the `calibrating` banner, the `zoneVar()` helper.

Replace the stacked bar + legend `<ul>` with a ranked bar list:

- Iterate zones **descending by `zone`** (5→1) — sort a copy, do not mutate the prop array.
- Each zone is one row containing:
  - **Label line**: zone `name` in `--foreground` medium weight + `{min_bpm}–{max_bpm} bpm`
    in `--muted`. (Optional faint `Z{n}` prefix — implementer discretion; keep it legible
    at rail width.)
  - **Bar line**: a `flex` row — a `flex-1` full-width track
    (`bg-[var(--surface-2)]`, `rounded-full`, `h-2.5`) containing a fill of
    `width: ${z.time_pct * 100}%` and `backgroundColor: zoneVar(z.zone)` via inline style,
    `aria-hidden`. A fixed-width figures column pinned right: time
    (`formatDuration(z.time_seconds)`) in `--muted` and percent
    (`formatPercent(z.time_pct)`) in `--foreground`, both `tabular-nums`.
  - 0% fill renders an empty track; the row still renders (fixed five-zone axis).
- No horizontal overflow at rail width (track flexes, figures column fixed-width).
- Update the file header comment to describe the ranked-bars treatment and reference
  design-system v0.4.2.

## Task 3 — Update `HeartRateZones.test.tsx`

Keep the existing `zones()` fixture factory and the null/calibrating/calibrated cases.
Add coverage for the new structure:

- Renders one row per zone (five) with each `name`, bpm range, time, and percent present.
- Rows render in **descending zone order** (VO2max first, Recovery last) — assert DOM order.
- Each fill node carries the right inline `width` (from `time_pct`) and
  `backgroundColor: var(--zone-N)`.
- Zero-time zone fixture (`time_seconds: 0`, `time_pct: 0`) still renders that row with
  `0:00` / `0%`.
- Extreme-skew fixture (Recovery `0:37`/1%, Aerobic `0:24`/1%, Tempo `3:01`/5%, Threshold
  `13:34`/22%, VO2max `43:34`/71%) renders all five rows with their figures.
- Calibrating banner present when calibrating, absent when calibrated.
- Renders nothing when null / empty.

## Task 4 — design-system.md (prog-strength-docs)

- Bump header to **v0.4.2** / last-updated 2026-06-29.
- Under **Color**, add a short note: the HR-zone `--zone-1..5` scale is now a separable
  calm five-tone scale (cool→warm, status-tone saturation register, distinct from the
  accent), superseding the periwinkle ramp.
- Add a **Changelog** entry **v0.4.2** (2026-06-29) recording the separable zone scale,
  with provenance pointing at this SOW.
- Add the SOW to the **Provenance** list.

## Task 5 — SOW status flip

Frontmatter `status: shipped`; body `**Status**: Shipped`; `**Last updated**: 2026-06-29`.

## Verification gate (prog-strength-web)

`npm run lint` → `npm run format:check` → `npm run typecheck` → `npm run test` →
`npm run build`, all green before push. Commit in Conventional Commits format (Husky
hooks enforce). No `--no-verify`, no lint-disable, no test skips.
