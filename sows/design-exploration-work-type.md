---
status: draft
repos:
  - prog-strength-developer
  - prog-strength-docs
---

# Design Exploration (DX) — A Divergent, Decision-Gated Work Type

**Status**: Draft · **Last updated**: 2026-06-16

## Introduction

The autonomous developer platform is built around one shape of work: the **SOW**. A SOW is convergent — one spec, one correct implementation, reviewed and merged. That shape is right for almost everything the platform does, because almost everything the platform does *has* a correct answer: an endpoint either returns the right macros or it doesn't, a migration either backfills cleanly or it doesn't. The worker's job is to converge on that answer, and the docs PR is the signal that it has.

Design is the exception, and it is a sharp one. A frontend surface does not have *a* correct look — it has a space of valid ones, and the value is in the *spread*: seeing a data-dense terminal treatment next to a warm editorial one next to a brutalist monospace one, and recognizing which fits the product. Forcing that through the SOW container destroys exactly what makes it valuable. A SOW asks the worker to converge, so it converges — on the model's median taste. The result is the generic, soulless frontend that every unguided AI produces: technically fine, spacing plausible, and completely interchangeable with every other AI's output. Convergence is the wrong objective function for design, and the SOW pipeline can only converge.

The problem is not that the model has bad taste. It is that **human taste is entering at the wrong gate.** In a SOW it enters at code review — by which point the surface already looks like one thing, and the reviewer can only nudge that one thing. The decision that actually matters — *which of several genuinely different directions is right* — was never offered. There were never several directions. The pipeline flattened them before the human ever saw them.

This SOW introduces a second work type, **DX (Design Exploration)**, whose mechanics are deliberately divergent and decision-gated. A DX produces **N differentiated visual variants of a single frontend surface, rendered side by side on one screen, awaiting a human pick.** It does not converge, it does not produce production code, and — the load-bearing constraint — **it never merges.** Convergence is the human's job, and it happens at *selection*, not at code review. The human opens the comparison, picks the direction that fits, and that pick becomes the input to a normal SOW that implements it properly. Taste enters at the gate where it changes the outcome.

After this ships, the platform can dispatch a DX the same way it dispatches a SOW — same Actions button, same fleet lock, same dashboards — by pointing it at a `dx/<surface>.md` ticket instead of a `sows/<feature>.md` one. The worker reads the ticket's enumerated design directions, builds one disposable variant per direction behind a flag on a throwaway branch, and opens a draft pull request — explicitly marked never-to-merge — whose body is a selection sheet backed by a live preview deploy. The owner compares, picks, closes the PR, and writes the SOW that builds the winner for real.

## Proposed Solution

A "work type" in this platform is three things and nothing more: **(1)** the ticket schema the worker parses from frontmatter, **(2)** which prompt template the worker renders and feeds to Claude, and **(3)** the branch / PR / merge contract that prompt enforces. Everything else — the dispatch workflow, the DynamoDB SOW lock, the userdata bootstrap, the launch template, the Grafana dashboards, the CloudWatch streams — is type-agnostic plumbing that DX reuses unchanged. This is what makes DX a small, additive change rather than a parallel pipeline.

DX is selected by a new **`type:` frontmatter field** on the ticket: `sow` (the default when absent, so every existing SOW keeps working untouched) or `dx`. The worker's bootstrap already loads the ticket's YAML frontmatter to read `repos:`; it now also reads `type` and **routes on it** — choosing `bootstrap/prompt-dx.md.tpl` over `bootstrap/prompt.md.tpl`, and substituting the DX ticket's fields (`surface`, `idioms`, `references`, `scope`, `variant_count`) into that prompt. The routing-and-validation logic does **not** go into userdata as more inline bash; following the precedent set by the `fleet` control-plane package, it lives in a small, unit-tested Python module (`bootstrap/ticket.py`) that userdata calls.

The DX prompt enforces the divergent contract. The worker builds a **single comparison route** — `/design-explore/<surface>` — that renders all N variants on one screen, behind a feature flag, on a throwaway `dx/<surface>` branch. Each variant realizes one of the ticket's enumerated **idioms** (distinct design directions), so the output is a forced spread, not N near-duplicates. The code is disposable scaffolding — rough is fine, because the winner gets reimplemented properly by a downstream SOW. The worker opens a **draft pull request titled `[DX — DO NOT MERGE]`** whose body is the selection artifact: a link to the surface's live preview deploy, one section per idiom, and a selection checklist. The worker never flips a status to "shipped," never picks a winner, and never merges. That is the human's job at the selection gate.

The selected variant then feeds a normal SOW — "implement `<surface>` per the `<chosen-idiom>` variant from `dx/<surface>`, production-quality" — and the divergent exploration converges back into the existing convergent pipeline.

## Goals and Non-Goals

### Goals

- A second autonomous **work type, `DX`**, dispatchable through the *existing* "Dispatch SOW" workflow, fleet lock, and dashboards — by pointing it at a `dx/<surface>.md` ticket.
- A **`type:` frontmatter discriminator** (`sow` | `dx`, default `sow`) that routes the worker to the right prompt template. Existing SOWs, which have no `type:`, continue to behave exactly as before.
- A **DX ticket schema** (`templates/dx-template.md`) with the mandatory `surface`, `idioms`, `references`, `scope`, and `variant_count` fields, plus a body that grounds each idiom and the selection criteria.
- A **DX prompt template** (`prompt-dx.md.tpl`) that produces **N differentiated variants on one comparison route**, behind a flag, on a throwaway branch — leaning on the `frontend-design` skill already installed on the worker.
- The **never-merge contract, enforced**: a DX opens a draft `[DX — DO NOT MERGE]` PR as its selection artifact, flips no status to shipped, and picks no winner.
- **Fail-fast validation at boot**: a `type: dx` ticket whose `idioms` is missing or shorter than `variant_count` terminates with a clear message before Claude runs — because without enumerated idioms the variants collapse into near-duplicates and the whole exercise is wasted compute.
- Routing + validation logic in a **unit-tested Python module** (`bootstrap/ticket.py`), not inline userdata bash, consistent with the `fleet` package precedent.
- The `prog-strength-developer` **README** documents the DX work type alongside the SOW one.

### Non-Goals

- **Automating the selection.** The human pick *is* the feature. A "pick the best one for me" step rebuilds the SOW pipeline's convergence and regresses straight back to median taste. The worker must never choose a winner.
- **Promoting DX mockup code to production.** Variants are disposable scaffolding. The winner is reimplemented properly via a downstream SOW that conforms to the design system; mockup code is never merged or promoted as-is.
- **The Design System (DS) work type.** DX produces selections; codifying recurring selections into `design-system.md` + tokens is the convergent counterpart, deferred to a follow-up SOW (see "Design System" below). Nothing to harvest until several DXs have run.
- **Mobile DX.** v1 targets `prog-strength-web` (Next.js) surfaces, because the draft-PR handoff relies on the per-PR **preview deploy** to render the comparison. `prog-strength-mobile` (TestFlight, no per-PR web preview) needs a different handoff and is a noted future extension.
- **A new dispatch workflow, lock table, or dashboard.** DX rides the existing ones. The SOW lock keys on the ticket path regardless of type; `dx/runs-list.md` locks exactly like `sows/foo.md`.
- **Changing the SOW work type's behavior.** The SOW prompt, its `feat/<slug>` branches, its docs-PR "ready" signal, and the status-flip flow are untouched.

## Implementation Details

Subsections follow the lifecycle: the ticket (docs) → the routing the worker does (developer) → the prompt contract (developer) → the operator-facing surfaces (README, validation/tests).

### DX ticket schema and template (`prog-strength-docs`)

DX tickets live in a new top-level directory, **`dx/`**, as `dx/<surface>.md` — distinct from `sows/`, because a DX is not a statement of work and never ships code of its own. A new `templates/dx-template.md` mirrors the house style of `templates/sow-template.md` (blockquote prompts the author deletes as they fill sections in).

Frontmatter:

```yaml
---
type: dx
status: draft            # draft | exploring | awaiting_selection | selected | abandoned
surface: runs-list       # the component/view being explored
idioms:                  # MANDATORY. Enumerated, distinct design directions.
  - editorial-data-journalism   # Each must differ in type scale, color logic,
  - brutalist-monospace         # and spacing rhythm — not just accent color.
  - warm-organic                # len(idioms) must be >= variant_count.
  - terminal-dense
  - linear-minimal
references:              # North-star products that ground "good" concretely.
  - Strava               # Bare adjectives ("professional", "clean", "modern")
  - Whoop                # are banned — name products, not vibes.
  - Linear
scope: in-system         # in-system (refine within the design system)
                         #   | greenfield (explore beyond it)
variant_count: 5         # default 5
repos:
  - prog-strength-web    # the single surface repo whose UI is explored
  - prog-strength-docs   # the DX ticket itself (status updates)
---
```

Body sections (each a template blockquote prompt):

- **Context** — what surface this is, where it lives today, and why it's worth exploring now. Product-focused, like a SOW intro.
- **The surface** — the concrete component/view, its data, and its current treatment (if any). Enough that a worker can build a faithful variant without guessing the data shape.
- **Idioms** — one short paragraph per idiom grounding the direction: what type scale, color logic, and spacing rhythm it implies, and which reference product it leans on. This is what forces the spread; vague idioms produce duplicate variants.
- **References** — the north-star products and *what specifically* to take from each ("Whoop's recovery-ring density", not "Whoop").
- **Selection criteria** — what the owner will weigh when picking. Not a rubric the worker optimizes against (that re-introduces convergence) — a note-to-self for the human at the gate.

`status:` is editorial, like the SOW `status:` — the owner is the dispatch gate. Its DX-specific lifecycle is `draft` → `exploring` (worker running) → `awaiting_selection` (PR open, owner deciding) → `selected` / `abandoned`. The worker sets `awaiting_selection` on the `dx/<surface>` branch as part of opening the PR; the owner sets the terminal value when they close it.

The `repos:` list works exactly as it does for SOWs — it is the worker's clone list. For a web DX that is the surface repo (`prog-strength-web`) plus `prog-strength-docs` (so the worker can push the ticket's `status` update).

`prog-strength-docs/AGENTS.md` § Frontmatter and `README.md` gain a short note that `type: dx` tickets live in `dx/` and follow `templates/dx-template.md`.

### Worker routing and validation (`prog-strength-developer`)

Today `bootstrap/userdata.sh.tpl` parses the ticket frontmatter inline (a Python heredoc) only to extract `repos:`. That inline parse grows awkward the moment routing and validation join it, and the repo has already made the call — in the `fleet` SOW — that control logic belongs in a tested module, not workflow/userdata bash. So this work introduces **`bootstrap/ticket.py`**, a small module that owns everything type-related:

- `parse(ticket_path) -> Ticket` — load and validate the frontmatter. `Ticket` carries `type` (default `sow`), `repos`, and, for DX, `surface`, `idioms`, `references`, `scope`, `variant_count`.
- **Validation** (raises with a clear, single-line message that userdata surfaces before terminating):
  - `repos` non-empty (the existing guard, moved here).
  - For `type: dx`: `surface` present; `idioms` present and `len(idioms) >= variant_count`; `references` present and non-empty. These are the conditions under which a DX is worth running at all — enforcing them at boot turns a wasted six-hour run that produces five identical cards into an instant, legible failure.
- `prompt_template(ticket) -> str` — returns `bootstrap/prompt-dx.md.tpl` for `dx`, `bootstrap/prompt.md.tpl` for `sow`.
- `substitutions(ticket, ...) -> dict` — the `__KEY__` → value map for the `sed` render. Both types share the path/slug/org/today tokens (`__SOW_PATH__`, `__SOW_SLUG__`, `__GITHUB_ORG__`, `__TODAY__` — kept under those names so the render mechanism and CloudWatch stream slug are identical regardless of type); DX additionally emits `__SURFACE__`, `__IDIOMS__` (a readable list), `__REFERENCES__`, `__SCOPE__`, `__VARIANT_COUNT__`.

`userdata.sh.tpl` changes minimally: where it currently parses `repos:` and unconditionally renders `prompt.md.tpl`, it instead calls `ticket.py` to (a) get the repo list, (b) get the chosen template path, and (c) get the substitution pairs, then renders that template. The clone loop, the `frontend-design`/superpowers plugin install, the JSONL renderer sidecar, the lock release, and self-termination are all unchanged and type-agnostic.

The **dispatch workflow needs no functional change**: its `sow_path` input is already just a path within `prog-strength-docs`, and `dx/runs-list.md` flows through acquire-lock → render-userdata → run-instance → attach identically to `sows/foo.md`. The SOW slug derivation (`basename … .md`) and the CloudWatch stream prefix work for DX paths as-is. The input's *description* is updated to say "path to a SOW (`sows/…`) or DX (`dx/…`) ticket," and the workflow display name can broaden to "Dispatch ticket," but the mechanics stay put. (Renaming the input key is deliberately avoided — it's a hand-invoked workflow and a key rename breaks muscle memory and any saved dispatches for no functional gain.)

### The DX prompt template (`bootstrap/prompt-dx.md.tpl`)

A sibling to `prompt.md.tpl`, rendered with the same `sed` mechanism. It instructs an unattended Claude as follows (paraphrased; the file is the source of truth):

- **Task.** Read the DX ticket at `/workspace/prog-strength-docs/__SOW_PATH__` (the shared path token, here pointing at the `dx/<surface>.md` ticket). Produce `__VARIANT_COUNT__` visual variants of the `__SURFACE__` surface, **one per idiom** in `__IDIOMS__`, grounded in `__REFERENCES__`, within `__SCOPE__`.
- **Invoke the `frontend-design` skill** and follow it. This is a design exploration; quality and *differentiation* are the whole point.
- **Build one comparison route** at `/design-explore/<surface>` in `prog-strength-web` that renders every variant on a single screen (labeled with its idiom), **behind a feature flag** so it is never reachable in normal navigation. Variants are self-contained throwaway components — duplication between them is fine, shared abstraction is *not* the goal, divergence is.
- **Each variant must realize a different idiom** along type scale, color logic, and spacing rhythm — not a re-skinned accent color. The worker maps variant → idiom explicitly in code comments and in the PR body.
- **Disposable on purpose.** Rough code is acceptable; the winner is reimplemented properly by a downstream SOW. Do not over-engineer, do not write tests for mockups, do not wire variants to real data services unless trivial — static fixtures that look real are fine and preferred.
- **Branch and PR contract — the hard constraints:**
  - Work on a throwaway branch **`dx/<surface>`** (never `feat/…`, never `main`).
  - Open a **draft** pull request titled **`[DX — DO NOT MERGE] <surface> — N design variants`**.
  - **Never merge. Never pick a winner. Touch no production routes, no production code paths, no shipped components.** The comparison route is additive and flag-gated.
  - Do **not** flip any `status:` to `shipped` and do **not** open the SOW-style docs "ready to ship" PR — a DX has no shippable state. The worker does update the DX ticket's own `status:` to `awaiting_selection` on the `dx/<surface>` branch (in `prog-strength-docs`), which is what the draft PR's docs change carries.
- **The PR body is the selection artifact** (required template, below): the live preview-deploy URL of `/design-explore/<surface>`, one section per idiom (name, the reference it draws on, a one-line "what makes this one distinct," and an anchor/screenshot), and a **selection checklist** the owner ticks. It closes with a plain statement that this PR is never merged — the owner picks a variant, closes it, and opens a SOW to build the winner.
- **Constraints** (shared with the SOW prompt): running under `--dangerously-skip-permissions`, no destructive git, no force-push, modify only repos in `repos:`. Plus the DX-specific: produce no production-bound diff, and if the ticket is ambiguous enough to block (e.g. fewer idioms than variants slipped past validation), open the draft PR with whatever variants are done rather than hitting the runtime backstop empty.

Required DX PR-body template (analogous to the SOW docs-PR template, adapted for selection):

```markdown
## Design Exploration: {{ surface }} — DO NOT MERGE

{{ one_sentence: what surface this explores and why }}

**Ticket**: [`dx/{{ surface }}.md`](…)
**Preview**: {{ live preview-deploy URL of /design-explore/{{ surface }} }}
**Scope**: {{ in-system | greenfield }}

## Variants

{{ for each idiom, in ticket order: }}
### {{ idiom }}
- **Draws on**: {{ reference product + what specifically }}
- **Distinct because**: {{ type scale / color logic / spacing rhythm in one line }}
- **See**: {{ anchor in the comparison route / screenshot }}
{{ end for }}

## Selection

Pick the direction that fits the product, then **close this PR** (never merge)
and open a SOW: "implement {{ surface }} per the <chosen-idiom> variant from
dx/{{ surface }}, production-quality, conforming to the design system."

- [ ] {{ idiom_1 }}
- [ ] {{ idiom_2 }}
- [ ] … one box per idiom

---

This PR is a disposable exploration on a throwaway branch. It is never merged;
the chosen variant is reimplemented by a downstream SOW.
```

### Feature-flag and route placement (`prog-strength-web`)

The comparison route is additive and isolated: a new route segment under `app/` (e.g. `app/design-explore/[surface]/`) gated by an existing-style flag/env check so it is dead in production. Because the branch is never merged, this code never reaches `main` — the gate is belt-and-suspenders so that even if someone deploys the preview, the route is not linked from anywhere. The DX prompt is responsible for following `prog-strength-web`'s existing routing and flag conventions (the worker reads its `AGENTS.md`).

### README (`prog-strength-developer`)

A new **"Work types"** section documents the two types side by side: SOW (convergent, one change, `feat/<slug>`, merges after review, docs PR is the ready signal) and DX (divergent, N variants, `dx/<surface>`, never merges, draft PR is the selection artifact). It states the routing mechanism (`type:` frontmatter → prompt template, via `bootstrap/ticket.py`), the boot-time DX validation, the web-only v1 scope and why (preview-deploy handoff), and a one-line forward pointer to the deferred DS work type.

### Tests

The divergence is genuinely testable where it matters — in `ticket.py`, the routing/validation seam:

- `parse` accepts a real SOW frontmatter (no `type:`) and defaults `type == sow`.
- `parse` accepts a valid DX ticket and exposes `surface`/`idioms`/`references`/`scope`/`variant_count`.
- **Validation rejects** a `type: dx` ticket with missing `idioms`, with `len(idioms) < variant_count`, with empty `references`, or with missing `surface` — each with a distinct message (the core guarantee that a malformed DX fails fast, not after six hours).
- Validation rejects empty `repos:` for both types (the moved-in guard).
- `prompt_template` returns the DX template for `dx`, the SOW template for `sow`.
- `substitutions` emits the expected `__KEY__` set for each type.

These run in the existing `tests/` suite (pytest), alongside the `fleet` tests. Userdata itself stays shell and is exercised only end-to-end by a real dispatch; keeping its new logic in `ticket.py` is what makes the behavior unit-testable at all.

## Handoff to SOW

A DX ends at a *chosen variant*, not at merged code. The selection gate is the human:

1. The owner opens the draft PR, follows the preview-deploy link, and compares the N variants on `/design-explore/<surface>`.
2. The owner picks the direction that fits, ticks its box, sets the DX ticket `status: selected` (noting the winning idiom), and **closes the PR — never merges it.** The `dx/<surface>` branch is left for reference or deleted; either way nothing reaches `main`.
3. The owner opens a normal **SOW**: *"Implement `<surface>` per the `<chosen-idiom>` variant from `dx/<surface>`, production-quality, conforming to the design system."* That SOW runs through the existing convergent pipeline — `feat/<slug>` branch, review, docs-PR ready signal, merge.

The mockup code is never promoted straight to production. The DX gave the *direction*; the SOW builds it for real.

## Design System (DS) — deferred, but here is the seam

DX is divergent and never converges by design. Across many DXs, the owner will keep making the *same* underlying decisions (a tighter type scale, a particular spacing rhythm, a recurring color logic). The convergent counterpart that captures those is a **DS (Design System)** work type: it harvests recurring decisions out of completed DXs into a durable `design-system.md` + design tokens, which DX tickets then reference via `scope: in-system` and which downstream SOWs conform to.

DS is **deliberately out of scope here**, for two reasons. First, there is nothing to harvest on day one — a DS pass needs several completed DXs to find what actually recurs, and this SOW ships the first DX capability. Second, a DS update may turn out to be *just a normal SOW* that edits `design-system.md` and the token files — convergent, merges, reviewed — in which case it needs no distinct mechanics at all. Which of those it is becomes answerable only after a few DXs have run. The `scope` field is included now so DX tickets are forward-compatible with a design system that does not yet exist (`in-system` simply means "there isn't one yet, stay conventional" until there is). DS gets its own SOW once the input set exists.

## Rollout

No infrastructure, no migration, no coordination window — this is prompt, bootstrap, docs, and a flag-gated route.

1. **`prog-strength-developer`** — add `bootstrap/ticket.py` + its tests; add `bootstrap/prompt-dx.md.tpl`; switch `userdata.sh.tpl` to route via `ticket.py`; update the README "Work types" section. Merge. The next worker boot uses the new path for both types; existing SOWs (no `type:`) are unaffected because `type` defaults to `sow`.
2. **`prog-strength-docs`** — add `templates/dx-template.md` and the `dx/` directory (with a first real ticket if one is ready); note `type: dx` in `AGENTS.md` § Frontmatter and `README.md`. Merge.
3. **First DX dispatch** — Actions → "Dispatch" → paste `dx/<surface>.md`. The worker builds the variants on `dx/<surface>`, opens the `[DX — DO NOT MERGE]` draft PR, and the preview deploy renders the comparison. The owner selects and opens the follow-up SOW.

### Verification after rollout

- Dispatch an existing `sows/*.md` ticket and confirm it behaves exactly as before (SOW prompt, `feat/<slug>`, docs-PR signal) — the regression guard that `type` defaulting works.
- Dispatch a valid `dx/<surface>.md` ticket and confirm: a `dx/<surface>` branch and a **draft** `[DX — DO NOT MERGE]` PR; a flag-gated `/design-explore/<surface>` route rendering `variant_count` variants, one per idiom; the preview-deploy link resolves; no `status: shipped` flip and no production diff.
- Dispatch a deliberately malformed DX ticket (idioms shorter than `variant_count`) and confirm the worker **terminates fast** with the validation message, having launched no exploration.

## Open Questions

- **Idioms vs. variant_count mismatch** — validation requires `len(idioms) >= variant_count`, and the worker builds one variant per idiom up to the count. If `idioms` exceeds the count, does the worker pick the first N, or is `variant_count` redundant once idioms are enumerated? Leaning: drop `variant_count` and let the idiom list *be* the count, with a default template carrying five idioms. Cheap to settle at implementation; kept as a field for now so the default is explicit.
- **Preview-deploy URL capture** — the worker needs the surface repo's preview URL to put in the PR body. Confirm the cleanest source (the provider's PR comment/deployment status vs. constructing the URL from the branch name), since the draft PR is opened by the worker and the deploy may not have finished at PR-creation time. A "preview link will appear in the deploy check below" line is an acceptable fallback if capture is racy.
- **Branch cleanup** — `dx/<surface>` branches accumulate as throwaways. Auto-delete on PR close, leave them, or a periodic sweep? Out of band from this SOW, but worth a convention.
- **One DX per surface** — the SOW lock keys on ticket path, so re-dispatching the same `dx/<surface>.md` is blocked while one is in flight (correct). Re-exploring a surface later means a new dispatch against the same ticket after the prior PR is closed; confirm that reads naturally or whether re-runs want a versioned ticket path.
