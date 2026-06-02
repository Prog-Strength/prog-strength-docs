# prog-strength-docs

Long-form documentation for **Prog Strength** — statements of work, design notes, and anything else that helps maintain and continue development on the project.

Code lives in the service repos (`prog-strength-api`, `prog-strength-mcp`, `prog-strength-agent`, `prog-strength-web`, `prog-strength-infra`); this repo is for the docs that explain *why* the code is the way it is.

## Layout

```
sows/        Statements of work — one per non-trivial feature.
plans/       Implementation plans derived from SoWs.
templates/   Boilerplate for new docs. Start here when adding a new SoW.
```

## Adding a statement of work

1. Copy `templates/sow-template.md` to `sows/<feature-slug>.md`.
2. Fill in each section in order, deleting the blockquote prompts as you go. Leftover prompts in a submitted SoW are a visible "this is unfinished" signal during review.
3. Commit and open a PR. The `Status` field at the top of the SoW progresses `Draft → In Review → Approved → Shipped` as the doc evolves and the feature ships.

## SOW frontmatter

SOWs that are candidates for autonomous implementation by `prog-strength-developer` SHOULD include YAML frontmatter at the very top of the file:

```yaml
---
status: <draft|ready_for_implementation|in_review|shipped>
repos:
  - prog-strength-api
  - prog-strength-agent
  - prog-strength-infra
  - prog-strength-docs   # always include when the SOW itself will be edited
---
```

`status` is informational and editorial — `prog-strength-developer` does not gate dispatch on it. The owner is the gate by choosing when to click "Run workflow" in the developer repo. Status values complement the in-body `Status:` line; either or both can be present during a transition period.

`repos:` is load-bearing for `prog-strength-developer`. Each entry must be a repository name in the `Prog-Strength/<name>` namespace. The autonomous developer reads this list to decide which repos to clone before invoking Claude Code. Inclusion is a clone-list, not a touch-list — Claude may decide a referenced repo doesn't need changes.

Existing SOWs are not retroactively annotated; only SOWs scheduled for autonomous re-runs need frontmatter. Anything already `Status: Shipped` can stay as-is.

See `sows/prog-strength-developer.md` for the SOW that introduces this convention.
