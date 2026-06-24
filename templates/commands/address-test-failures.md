---
id: address-test-failures
type: command
name: /address-test-failures
description: Developer command to address P0 production code failures recorded by /execute-tests — fixes production code, verifies locally, then re-enters the full review cycle
status: active
version: 1
primary_role: developer
links:
  - relation: follows
    target: execute-tests
  - relation: precedes
    target: review-implementation
---

# /address-test-failures

## Purpose

Address production code bugs that caused P0 test failures in milestone MN. This command is the developer's entry point when `/execute-tests` sets `implementation_status: test_failed`. Fix the production code, verify locally, then re-enter the full review cycle. Any code change — however small — goes through `/review-implementation`.

Test code bugs are QE's responsibility and are not addressed here. If the test execution round included both production and test code bugs, QE bundles their test code fixes into the same commit batch so the review cycle covers everything together.

## Usage

```
/address-test-failures PLAN-0001 M1
```

## Pre-flight Check

Read PLAN-NNNN milestone MN's `**Implementation status:**`. Must be `test_failed`. If not, stop:

- `pending` → "MN has not been implemented. Run `/implement-plan PLAN-NNNN MN`."
- `in_progress` → "Review cycle is already open for MN. Check `/review-implementation PLAN-NNNN MN`."
- `approved` → "No test failures recorded. Run `/execute-tests PLAN-NNNN MN TEST-NNNN`."
- `tests_passed` → "All tests passing. Run `/create-pr PLAN-NNNN MN` to open the milestone PR."
- `pr_open` → "MN's PR is open. No test failures to address."
- `pr_merged` → "MN is fully complete — PR has been merged."

## Step 1 — Read the Failure Report

Read PLAN-NNNN milestone MN's latest `### Test Execution — Round N` section. Collect every row marked `Status: FAIL` that was classified as a **production code bug** by QE.

For each failure, read:
- The test case spec in TEST-NNNN — the full Given/When/Then or Scenario/Input/Expected block for that ID
- The test file implementing the test case
- The production code exercised by the test

The failure message in the Notes column is the starting point. The TEST-NNNN spec is the ground truth for what the production code should do.

## Step 2 — Diagnose and Fix

For each production code bug:

1. Identify the exact divergence: what the production code does vs. what TEST-NNNN specifies it should do
2. Fix the production code — do not change test files (test code is QE's domain)
3. Do not expand scope beyond what the failing tests require — if the fix reveals a related issue that wasn't caught by tests, surface it rather than silently expanding the fix

If QE is bundling test code fixes in the same batch: they commit their test file changes separately, or coordinate with the developer to commit together. The review cycle covers all changes in the batch.

## Step 3 — Update Milestone Status

Update PLAN-NNNN milestone MN before committing any fix:

```
**Implementation status:** in_progress
```

Commit this status change:

```bash
git add sdlc/plans/PLAN-NNNN-[slug].md
git commit -m "chore(MN): reopen milestone — addressing P0 production code failures"
```

## Step 4 — Local Verification

After all production code fixes are applied, run the project's verification suite (from `project.yml` stack configuration):

```bash
{{stack.lint}}
{{stack.test.unit}}
```

If `stack.lint` is `none`, skip the lint step. Confirm the specific failing test cases now pass by running only those test files. The invocation syntax for targeting a single test file varies by runner — see ARCH.md for this project's pattern (e.g., Vitest: append the file path; pytest: `pytest path/to/test_file.py`; RSpec: `rspec spec/path_to_spec.rb`).

Read the milestone's tasks in PLAN-NNNN to derive verification steps — what core behaviors does MN implement? Present these to the user as a verification checklist:

> **Production code fixes applied for MN — [milestone name].**
>
> Files changed:
> [list each changed production file]
>
> Local checks pass. Previously failing tests now passing:
> - TC-[ID]: [what was wrong and what changed]
> - TC-[ID]: [what was wrong and what changed]
>
> Please verify the fix locally:
> [Verification steps derived from the milestone's tasks in PLAN-NNNN — start the dev server, exercise the relevant feature paths, confirm the expected behavior]
>
> When satisfied, run: `/review-implementation PLAN-NNNN MN`

Wait for the user to confirm before handing off.

## Step 5 — Handoff to Review

```
/review-implementation PLAN-NNNN MN
```

When handing off, explicitly note:
- Which test cases were failing and why (production code bug)
- Which files were changed to fix them
- Whether QE also bundled test code fixes in this batch

This context helps reviewers focus on the changed paths rather than re-examining the entire milestone.

## Notes

- Fix all production code bugs identified in the failing round before triggering review — do not fix one, re-test, fix another. One fix batch per review round keeps the history clean.
- If the diagnosis reveals a design-level issue (the spec was ambiguous, the implementation followed one reading but the tests assumed another), surface it rather than picking one interpretation silently.
- If any production code fix touches security-critical paths (auth/data access, credential comparison, authorization checks), call this out explicitly in the review handoff — the Security perspective will scrutinize it carefully.
- **Team context — QE coordination:** `/execute-tests` commits QE's test code fixes before setting `test_failed`. Pull the branch before making any production code changes so your working tree includes QE's fixes. This way `/review-implementation` sees both sets of changes in one batch rather than requiring two separate review cycles.
