# prog-strength-docs

Long-form documentation for **Prog Strength** — statements of work, design notes, and anything else that helps maintain and continue development on the project.

Code lives in the service repos (`prog-strength-api`, `prog-strength-mcp`, `prog-strength-agent`, `prog-strength-web`, `prog-strength-infra`); this repo is for the docs that explain *why* the code is the way it is.

## Layout

```
sows/        Statements of work — one per non-trivial feature.
templates/   Boilerplate for new docs. Start here when adding a new SoW.
```

## Adding a statement of work

1. Copy `templates/sow-template.md` to `sows/<feature-slug>.md`.
2. Fill in each section in order, deleting the blockquote prompts as you go. Leftover prompts in a submitted SoW are a visible "this is unfinished" signal during review.
3. Commit and open a PR. The `Status` field at the top of the SoW progresses `Draft → In Review → Approved → Shipped` as the doc evolves and the feature ships.
