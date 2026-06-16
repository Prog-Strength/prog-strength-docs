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

# DX: <Surface Name>

**Status**: Draft · **Last updated**: YYYY-MM-DD

> A DX (Design Exploration) is the platform's **divergent** work type. Unlike a SOW it does not converge on one correct implementation — it produces N differentiated visual variants of a single frontend surface, side by side on one comparison route, awaiting a human pick. It **never merges** and ships no production code; the chosen direction feeds a downstream SOW that builds it for real. Replace the title above with the surface name. Keep the frontmatter exactly — the `dispatch` worker parses it (see prog-strength-developer `bootstrap/ticket.py`), and `idioms` shorter than `variant_count` fails dispatch at boot. Each blockquote below is a prompt — read it, write the section, then delete the prompt.

## Context

> What surface is this, where does it live in the product today, and why is it worth exploring now? Product-focused, like a SOW intro. Lead with what's unsatisfying about the surface today (or that it doesn't exist yet) and what a better one would unlock for a **Prog Strength** user. Avoid implementation detail.

## The surface

> The concrete component/view: its data, its current treatment (if any), and the states it has to handle (empty, loading, error, dense, sparse). Give enough that a worker can build a faithful variant without guessing the data shape — name the fields, show a realistic fixture. The variants are mockups, so static fixtures that look real are fine and preferred.

## Idioms

> One short paragraph **per idiom** grounding the direction concretely: what **type scale**, **color logic**, and **spacing rhythm** it implies, and which reference product it leans on. This is what forces the spread — vague idioms produce duplicate variants. There must be at least `variant_count` of them, and each must be distinguishable from the others along those three axes, not just by accent color.
>
> - **editorial-data-journalism** — …
> - **brutalist-monospace** — …
> - **warm-organic** — …
> - **terminal-dense** — …
> - **linear-minimal** — …

## References

> The north-star products from the frontmatter, and **what specifically** to take from each — "Whoop's recovery-ring density and single-accent-on-charcoal palette", not "Whoop". Name products, not adjectives.

## Selection criteria

> What you (the owner) will weigh when picking at the selection gate. This is a **note-to-self for the human**, not a rubric the worker optimizes against — a rubric re-introduces convergence and regresses to median taste. Capture what "fits the product" means for this surface so future-you remembers the call you were trying to make.

---

> **Lifecycle.** `status:` is editorial — the owner is the dispatch gate. It moves `draft` → `exploring` (worker running) → `awaiting_selection` (draft PR open, owner deciding) → `selected` / `abandoned`. The worker sets `awaiting_selection` on the `dx/<surface>` branch as it opens the PR; the owner sets the terminal value when they close it.
>
> **Handoff.** A DX ends at a *chosen variant*, not merged code. The owner opens the draft `[DX — DO NOT MERGE]` PR, compares the variants on the preview deploy, picks one, ticks its box, sets `status: selected` (noting the winning idiom), and **closes the PR — never merges it.** Then they open a normal SOW: *"implement `<surface>` per the `<chosen-idiom>` variant from `dx/<surface>`, production-quality, conforming to the design system."* The mockup code is never promoted as-is.
