# <Feature Title>

**Status**: Draft · **Last updated**: YYYY-MM-DD

> Replace the title above with the feature name. Update `Status` (`Draft` → `In Review` → `Approved` → `Shipped`) and `Last updated` as the document evolves. Each blockquote below is a prompt — read it, write the section, then delete the prompt.

## Introduction

> What problem does this feature solve and why does it matter to a **Prog Strength** user? Keep this section product-focused. Lead with the gap in the current product, explain why the gap exists, and end with what changes for the user once it ships. Avoid implementation details — those go below.

## Proposed Solution

> High-level description of the approach. The reader should finish this section knowing the *shape* of the solution and its key abstractions, not the columns of any table. Implementation specifics go in the next section.

## Goals and Non-Goals

### Goals

> The contract this work commits to delivering. One bullet per goal — short, scannable, testable.

- 

### Non-Goals

> The things this work explicitly does not deliver, even if they seem adjacent. Non-goals are load-bearing — they prevent scope creep during review and give reviewers a clean way to say "out of scope, file separately."

- 

## Implementation Details

> Subsection per moving part. The headings below are common patterns from past SOWs in this repo; keep the ones that apply, delete the rest, and add new subsections as needed. Subsection order should follow the lifecycle of the feature (data → write → read → migration).

### Data Model

> Schema or type changes. New tables go in a markdown table of columns, types, and descriptions. Note any required indexes after the table, and call out foreign keys and cascade behavior explicitly.

| Column | Type | Description |
| --- | --- | --- |
| `id` | text | Primary key. |

### Write Path

> When and how state is written, updated, and deleted. Spell out cascade behavior and consistency invariants so reviewers can spot drift hazards. Bullet per lifecycle event.

- **Action** — what happens.

### Algorithms

> Any non-trivial math or logic. Use fenced code blocks for formulas, and explain in prose what is being computed and why. Where multiple approaches exist, show the meaningful alternatives and state the rationale for the choice — future readers benefit more from "we considered X and rejected it because Y" than from a bare decision.

```
formula = …
```

### API Surface

> New or changed endpoints: path, method, request shape, response shape, auth requirements. Skip this section if the feature is internal-only with no external consumers.

### Backfill or Migration

> How existing data is populated or transformed when the feature ships. Cover three things:
>
> 1. **Mechanism** — migration step, separate CLI command, lazy on-read, etc.
> 2. **Recoverability** — what happens if it fails partway through. Truncate-and-rerun is usually the cheapest story for derived tables.
> 3. **Scale boundary** — at what data volume the chosen strategy would need to change.

## Open Questions

> Decisions deliberately deferred to a follow-up. Each item should state the question, the options considered, and a tentative lean — not just an unanswered question. Reviewers can then either accept the lean or push back, instead of being asked to design from scratch.

1. **Question**. Options. Tentative lean.
