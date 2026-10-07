---
id: address-pr-review
type: command
name: /address-pr-review
status: deprecated
deprecated_date: 2026-06-18
replaced_by: address-implementation-review
deprecation_note: "PR-centric address flow replaced by /address-implementation-review PLAN-NNNN MN. Do not invoke."
version: 1
primary_role: developer
---

# /address-pr-review

**Deprecated.** This command has been replaced by `/address-implementation-review PLAN-NNNN MN`.

## What to use instead

```
/address-implementation-review PLAN-NNNN MN
```

This command resolves reviewer findings from the structured implementation review (`sdlc/reviews/IMPL-REVIEW-NNNN-MN.md`) rather than from GitHub PR comments. It reads each blocking finding, applies targeted fixes, and updates the review artifact round count — keeping the full address history in the SDLC artifact rather than in PR thread comments.

## Why deprecated

The original `/address-pr-review` operated on free-form PR review comments with no structured blocking/non-blocking classification. The replacement introduces explicit round tracking, severity classification, and a zero-blocking gate before `/approve-implementation` is allowed to proceed.
