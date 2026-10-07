---
id: review-pr
type: command
name: /review-pr
status: deprecated
deprecated_date: 2026-06-18
replaced_by: review-implementation
deprecation_note: "PR-centric review replaced by milestone-centric /review-implementation. Do not invoke."
version: 1
primary_role: lead-developer
---

# /review-pr

**Deprecated.** This command has been replaced by `/review-implementation PLAN-NNNN MN`.

## What to use instead

```
/review-implementation PLAN-NNNN MN
```

## Why deprecated

The original `/review-pr` operated on an open GitHub PR without structured multi-perspective review or blocking/non-blocking finding classification. The replacement drives a milestone-level review before the PR is even opened — covering Architect, Lead Developer, Security, and SRE perspectives — and produces a structured `IMPL-REVIEW-NNNN-MN.md` artifact that tracks findings across multiple address rounds. The PR is opened only after all blocking findings are resolved and tests pass.
