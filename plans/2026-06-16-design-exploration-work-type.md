# Implementation Plan — Design Exploration (DX) Work Type

Derived from `sows/design-exploration-work-type.md`. Two repos are
touched (the SOW `repos:` list): `prog-strength-developer` and
`prog-strength-docs`. The `prog-strength-web` `/design-explore/<surface>`
route is **not** built here — it is the contract a *future* DX dispatch
executes per `bootstrap/prompt-dx.md.tpl`; `prog-strength-web` is
deliberately absent from this SOW's clone list.

A "work type" is three things: the ticket schema parsed from frontmatter,
the prompt template rendered for Claude, and the branch/PR/merge contract
that prompt enforces. DX adds a second value of each; everything else
(dispatch workflow, fleet lock, userdata bootstrap, dashboards) is
type-agnostic plumbing reused unchanged.

## prog-strength-developer

### 1. `bootstrap/ticket.py` (new) + `tests/test_ticket.py` (new) — TDD

A small, unit-tested module owning everything type-related, mirroring the
`fleet` package precedent (control logic in tested code, not inline
userdata bash). Must run on **Python 3.9** (AL2023 worker system python),
so `from __future__ import annotations`, no `match`, no runtime `X | Y`.

Public surface:

- `parse(ticket_path) -> Ticket` — read file, extract the YAML
  frontmatter block, validate, return a `Ticket`. `Ticket` carries
  `type` (default `sow`), `repos`, and for DX `surface`, `idioms`,
  `references`, `scope`, `variant_count`.
- Validation (raises `TicketError` with a clear single-line message that
  userdata surfaces before terminating):
  - no frontmatter block;
  - `type` not in `{sow, dx}`;
  - `repos` empty (the existing guard, moved in) — both types;
  - DX only: `surface` missing; `idioms` missing/empty;
    `len(idioms) < variant_count`; `references` missing/empty.
- `prompt_template(ticket) -> str` — `prompt-dx.md.tpl` for `dx`,
  `prompt.md.tpl` for `sow` (basename; caller joins with templates dir).
- `substitutions(ticket, *, sow_path, sow_slug, github_org, today) -> dict`
  — the `__KEY__` → value map. Shared: `__SOW_PATH__`, `__SOW_SLUG__`,
  `__GITHUB_ORG__`, `__TODAY__` (names kept so the render mechanism and
  CloudWatch stream slug are identical regardless of type). DX adds
  `__SURFACE__`, `__IDIOMS__`, `__REFERENCES__`, `__SCOPE__`,
  `__VARIANT_COUNT__`. Multi-value fields render as single-line readable
  lists (sed-safe).
- A thin `argparse` CLI (`__main__`) with two subcommands userdata calls:
  - `repos --ticket PATH` — parse (full validation, fail-fast) and print
    the repo list, one per line.
  - `render --ticket PATH --sow-path … --sow-slug … --github-org …
    --today … --templates-dir … --out …` — parse, pick template,
    substitute, write the rendered prompt.

Tests (pytest, alongside the `fleet` tests; fixtures written to a
`tmp_path`): SOW frontmatter defaults `type == sow`; valid DX exposes its
fields; validation rejects missing `idioms` / `idioms < variant_count` /
empty `references` / missing `surface` / empty `repos` (distinct
messages); `prompt_template` routes per type; `substitutions` emits the
expected `__KEY__` set per type.

Declare `pyyaml>=6` in `pyproject.toml` (ticket.py's hard dependency;
previously only transitive via dev deps).

### 2. `bootstrap/prompt-dx.md.tpl` (new)

Sibling to `prompt.md.tpl`, same `sed`/replace render. Instructs an
unattended Claude to: read the DX ticket; invoke `frontend-design`;
build `__VARIANT_COUNT__` variants, one per idiom in `__IDIOMS__`,
grounded in `__REFERENCES__`, within `__SCOPE__`, on one flag-gated
comparison route `/design-explore/<surface>` in `prog-strength-web`; work
on a throwaway `dx/<surface>` branch; open a **draft** PR titled
`[DX — DO NOT MERGE] <surface> — N design variants` whose body is the
selection sheet (preview URL, one section per idiom, selection checklist).
Hard constraints: never merge, never pick a winner, no production diff,
no `status: shipped` flip; the worker only sets the DX ticket
`status: awaiting_selection`.

### 3. `bootstrap/userdata.sh.tpl` (edit, minimal)

- Move the `prog-strength-developer` clone (to
  `/opt/prog-strength-developer-repo`) **earlier** — before the ticket
  parse — so `ticket.py` is on disk when it's needed.
- Replace the inline `repos:` python heredoc with
  `python3 -m bootstrap.ticket repos` (PYTHONPATH at the cloned repo). A
  non-zero exit surfaces the validation message and terminates — this is
  the fail-fast guard.
- Replace the `sed` prompt render with `python3 -m bootstrap.ticket
  render`, which routes the template by type.
- Everything else (clone loop, plugin install, JSONL renderer, lock
  release, self-terminate) unchanged and type-agnostic.

### 4. `.github/workflows/dispatch-sow.yml` (edit, copy-only)

Broaden the `sow_path` input *description* to "path to a SOW (`sows/…`)
or DX (`dx/…`) ticket" and the workflow `name` to "Dispatch ticket". The
input **key stays `sow_path`** (renaming breaks muscle memory / saved
dispatches for no functional gain). No mechanics change.

### 5. `README.md` (edit)

New "Work types" section: SOW (convergent, `feat/<slug>`, merges, docs PR
is the ready signal) vs DX (divergent, N variants, `dx/<surface>`, never
merges, draft PR is the selection artifact); routing via `type:`
frontmatter → `bootstrap/ticket.py`; boot-time DX validation; web-only v1
scope (preview-deploy handoff); forward pointer to the deferred DS type.

## prog-strength-docs

### 6. `templates/dx-template.md` (new) + `dx/` directory

Mirror the house style of `templates/sow-template.md` (blockquote prompts
the author deletes). Frontmatter per SOW: `type: dx`, `status` (DX
lifecycle), `surface`, `idioms` (mandatory, `len >= variant_count`),
`references`, `scope`, `variant_count`, `repos`. Body sections: Context,
The surface, Idioms, References, Selection criteria. Create the `dx/`
directory with a short `dx/README.md` (no real ticket is ready; do not
fabricate a surface that may not exist in `prog-strength-web`).

### 7. `AGENTS.md` + `README.md` (edit)

Short note in AGENTS.md § Frontmatter and README that `type: dx` tickets
live in `dx/` and follow `templates/dx-template.md`.

### 8. Status flip (step 4 of the autonomous workflow)

In `sows/design-exploration-work-type.md`: frontmatter `status: shipped`;
body `**Status**: Shipped`; `**Last updated**: 2026-06-16`. Commit
`docs: mark design-exploration-work-type as shipped`.

## Verification (local, before push)

- `uv run pytest -q` green (new `test_ticket.py` + existing 51).
- `bash -n bootstrap/userdata.sh.tpl` clean.
- Render `prompt-dx.md.tpl` and `prompt.md.tpl` through `ticket.py render`
  against a SOW fixture and a DX fixture; confirm token substitution and
  template routing.
- No drift in `go`/lint gates — N/A (no Go, no lint config in this repo;
  the only gate is pytest).

## Rollout (for the docs PR body)

No infra, no migration, no coordination window. Merge
`prog-strength-developer` first (the next worker boot routes both types;
existing SOWs unaffected because `type` defaults to `sow`), then
`prog-strength-docs` (the template + `dx/` dir + status flip). They are
effectively independent, but developer-first means the moment a `dx/`
ticket exists it is already dispatchable.
