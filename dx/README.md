# Design Exploration (DX) tickets

DX tickets live here as `dx/<surface>.md` — one per frontend surface being
explored. A DX is the platform's **divergent** work type: it produces N
differentiated visual variants of a single surface, side by side on one
flag-gated comparison route, awaiting a human pick. It **never merges** and
ships no production code — the chosen direction feeds a downstream SOW that
builds it production-quality.

This is distinct from `sows/`: a DX is not a statement of work and has no
shippable state of its own.

## Adding a DX ticket

1. Copy [`../templates/dx-template.md`](../templates/dx-template.md) to
   `dx/<surface>.md`.
2. Fill in each section in order, deleting the blockquote prompts as you go.
   The `idioms:` list is mandatory and must enumerate at least
   `variant_count` genuinely distinct directions (differing in type scale,
   color logic, and spacing rhythm) — the dispatch worker validates this at
   boot and fails fast otherwise.
3. Commit and open a PR. Once merged, dispatch it the same way as a SOW:
   the [`Dispatch ticket`](https://github.com/Prog-Strength/prog-strength-developer/actions/workflows/dispatch-sow.yml)
   workflow in `prog-strength-developer`, passing the `dx/<surface>.md` path.

See [`AGENTS.md` § Frontmatter](../AGENTS.md) for the `type:` discriminator
and the SOW that introduced this work type,
[`sows/design-exploration-work-type.md`](../sows/design-exploration-work-type.md).

v1 targets `prog-strength-web` surfaces only — the draft-PR handoff relies
on the per-PR preview deploy to render the comparison.
