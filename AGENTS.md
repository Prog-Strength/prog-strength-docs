# AGENTS.md — prog-strength-docs

Guidance for AI coding agents (Claude Code, Copilot, Codex, etc.) working in this repo.

## What this repo is

`prog-strength-docs` is the long-form documentation home for **Prog Strength**. It does **not** contain application code — code lives in the sibling service repos (`prog-strength-api`, `prog-strength-mcp`, `prog-strength-agent`, `prog-strength-web`, `prog-strength-mobile`, `prog-strength-infra`, `prog-strength-sdk`, `prog-strength-developer`).

This repo's job is to capture the *why* behind the work: statements of work (SOWs), implementation plans, and design notes that wouldn't otherwise survive in commit history or scattered conversations.

```
sows/         Statements of work — one markdown file per non-trivial feature.
plans/        Implementation plans derived from SOWs (typically YYYY-MM-DD-<slug>.md).
templates/    Boilerplate for new docs. Always start a new SOW from templates/sow-template.md.
```

## Statements of Work (SOWs) — the primary artifact

A SOW is the unit of cross-cutting feature work. **Prog Strength** is decentralized — most features touch 3+ repos — and GitHub issues break down at that boundary, so SOWs in this repo are the natural place to specify cross-repo work end-to-end.

### Authorship

SOWs are written by a **human developer**, with AI assistance. The owner has had good results drafting SOWs in Claude Code with the `superpowers` plugin — in particular `superpowers:brainstorming` to pressure-test the problem and `superpowers:writing-plans` if a downstream implementation plan is needed. AI-only SOWs without a human in the loop tend to drift toward plausible-but-wrong scope; don't ship one.

### Frontmatter (REQUIRED for every new SOW — non-negotiable)

> Skipping this is the single most common bug in agent-drafted SOWs. The `dispatch-sow.yml` worker in `prog-strength-developer` parses the `repos:` list with `yq` to decide which repos to clone. A SOW without frontmatter fails dispatch immediately — the worker has no way to know which repos to check out, so it can't run at all. Add the frontmatter from the start; do not wait until the SOW is "ready for dispatch."

Every new SOW MUST begin with YAML frontmatter at the very top of the file — before the H1 title, before anything else:

```yaml
---
status: <draft|ready_for_implementation|in_review|shipped>
repos:
  - prog-strength-api
  - prog-strength-web
  - prog-strength-docs   # always include if the SOW itself will be edited
---
```

- `repos` is **load-bearing**. It's the clone-list the autonomous developer reads to decide which repos to check out before invoking Claude Code. Each entry must be the bare repository name in the `Prog-Strength/<name>` namespace. Inclusion is not an obligation to modify — Claude may decide a referenced repo doesn't need changes — but omission means the repo won't be available at all.
- `status` is **informational and editorial**. `prog-strength-developer` does NOT gate dispatch on it — the owner is the gate. The body's `**Status**:` line can complement or temporarily disagree with the frontmatter during transitions.
- Existing SOWs are **not** retroactively annotated. Add frontmatter only when a SOW is scheduled for an autonomous re-run, or when a fresh SOW is written. **A "fresh SOW" means any new file under `sows/` — there is no draft-vs-ready distinction here. Frontmatter goes in on the first commit.**

Do not copy older SOWs that lack frontmatter as a template — many pre-convention SOWs are still in the repo for historical reasons; copying one is the recurring trap. Always start from `templates/sow-template.md`, which includes the frontmatter block.

See `sows/prog-strength-developer.md` for the SOW that introduced this convention — it's the authoritative reference for how the autonomous-developer pipeline consumes these files.

### Format

Every new SOW MUST be copied from `templates/sow-template.md` and follow its section order (Introduction → Proposed Solution → Goals and Non-Goals → Implementation Details → Open Questions). The template's blockquote prompts are deletable instructions — read each, write the section, then delete the prompt. Leftover prompts in a submitted SOW signal "unfinished" during review.

## How this repo plugs into the automated workflow

The end-to-end loop:

1. A human (with AI assistance) drafts a SOW under `sows/`, using the template and adding frontmatter.
2. The SOW is reviewed and merged to `main`.
3. The owner manually triggers the [`dispatch-sow.yml`](https://github.com/Prog-Strength/prog-strength-developer/actions/workflows/dispatch-sow.yml) workflow in [`prog-strength-developer`](https://github.com/Prog-Strength/prog-strength-developer), passing the SOW path (e.g. `sows/my-feature.md`).
4. That workflow spins up an ephemeral EC2 worker which:
   - Clones this repo and reads the SOW's `repos:` frontmatter.
   - Clones each listed repo into `/workspace/<repo>`.
   - Runs Claude Code non-interactively against the SOW.
   - Opens pull requests against each affected repo on a `feat/<sow-slug>` branch.
   - Self-terminates.
5. The owner reviews and merges the resulting PRs at their own pace.

There is **no** auto-discovery — the owner explicitly dispatches a SOW by path. Concurrency is serialized to one worker at a time; the workflow's pre-flight check fails fast if a worker is already running.

## Guidance for agents editing this repo

- **Always add YAML frontmatter to a new SOW.** This is the #1 source of dispatch failures and the #1 thing agents forget. Every new file under `sows/` MUST start with the `---\nstatus: ...\nrepos:\n  - ...\n---` block before the H1. No exceptions for "draft" or "still figuring it out" SOWs. See the Frontmatter section above for the exact shape.
- **Don't invent SOWs unprompted.** Drafting a SOW is a deliberate human act, sometimes with AI help. If a user asks you to draft one, use the template; if they ask you to *suggest* features, propose them in chat, not as committed SOW files.
- **Preserve frontmatter format exactly.** The `dispatch-sow.yml` worker parses it with `yq`; structural changes can break dispatch silently.
- **Don't retroactively add frontmatter** to shipped SOWs unless the user explicitly asks. The README and the developer SOW both spell out that backfill is opt-in.
- **Plans (`plans/`) are downstream of SOWs.** If a SOW has a corresponding plan, name it `YYYY-MM-DD-<sow-slug>.md` so the autonomous developer can find it via glob.
- **No code in this repo.** If you find yourself wanting to add a script or a config file, you're probably in the wrong repo — most likely `prog-strength-developer` or one of the service repos.
- **Use the `superpowers` skills when drafting.** `superpowers:brainstorming` for early-stage SOWs; `superpowers:writing-plans` if a SOW needs an accompanying plan in `plans/`; `superpowers:writing-skills` is irrelevant here (no skills live in this repo).

## Reference

- SOW template: `templates/sow-template.md`
- SOW that introduces the autonomous-developer pipeline and frontmatter convention: `sows/prog-strength-developer.md`
- Trivial end-to-end smoke-test SOW: `sows/test-developer-bootstrap.md`
- Autonomous developer repo: <https://github.com/Prog-Strength/prog-strength-developer>
- Dispatch workflow: <https://github.com/Prog-Strength/prog-strength-developer/actions/workflows/dispatch-sow.yml>
