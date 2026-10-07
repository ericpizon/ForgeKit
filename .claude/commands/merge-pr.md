---
id: merge-pr
type: command
name: /merge-pr
status: deprecated
deprecated_date: 2026-06-18
replaced_by: approve-implementation
deprecation_note: "PR merge flow replaced by /approve-implementation PLAN-NNNN MN + /execute-tests. Do not invoke."
version: 1
primary_role: lead-developer
---

# /merge-pr

**Deprecated.** This command has been replaced by the two-step sequence: `/approve-implementation PLAN-NNNN MN` followed by `/execute-tests PLAN-NNNN MN TEST-NNNN`.

## What to use instead

```
/approve-implementation PLAN-NNNN MN
```

Then, after QE runs the test suite:

```
/execute-tests PLAN-NNNN MN TEST-NNNN
```

## Why deprecated

The original `/merge-pr` merged the PR immediately after review, bundling code commit and merge into one step with no structured test execution gate. The replacement separates concerns: `/approve-implementation` commits the milestone code after the review cycle closes; `/execute-tests` gates the PR on automated test results before the developer opens it on GitHub.
