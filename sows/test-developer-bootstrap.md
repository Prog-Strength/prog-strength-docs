---
status: ready_for_implementation
repos:
  - prog-strength-docs
---

# Test: Developer Bootstrap

**Status:** Test only · **Last updated:** 2026-06-02

This SOW exists solely to validate the `prog-strength-developer` autonomous-developer pipeline end-to-end. It is intentionally minimal so that a failed run unambiguously indicates a problem with the developer infrastructure (Terraform, userdata, GitHub App auth, Claude Code invocation) rather than the SOW content.

## Task

Append a single line to the bottom of `README.md` in this repository (`prog-strength-docs`):

```
Tested by prog-strength-developer on 2026-06-02.
```

Open a pull request titled `test: developer bootstrap smoke check` against `main`.

## Acceptance criteria

- The line above appears at the bottom of `README.md` on a new feature branch.
- A pull request exists with the specified title.
- No other files are modified.

## Cleanup

After the smoke test passes, this SOW and the resulting line in `README.md` can be deleted in a follow-up PR.
