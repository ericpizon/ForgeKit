---
id: create-pr
type: command
name: /create-pr
description: Developer command to open a GitHub PR for a completed milestone — generates PR description from PLAN, IMPL-REVIEW, and test results; sets implementation_status to pr_open
status: active
version: 1
primary_role: developer
links:
  - relation: follows
    target: execute-tests
  - relation: precedes
    target: implement-plan
---

# /create-pr

## Purpose

Open a GitHub PR from the milestone's feature branch (`feature/PLAN-NNNN-MN`) to `main`. Generates the PR title and description from the PLAN task list, IMPL-REVIEW history, and test execution results so reviewers have full context without reading the raw artifacts. Sets `implementation_status: pr_open`.

The next milestone cannot start until this PR is merged and the status advances to `pr_merged`. `/implement-plan` detects the merge automatically when it runs the pre-flight for the next milestone.

## Usage

```
/create-pr PLAN-0001 M1
```

## Pre-flight Check

Read PLAN-NNNN milestone MN's `**Implementation status:**`. Must be `tests_passed`. If not, stop:

- `pending` → "MN has not been implemented. Run `/implement-plan PLAN-NNNN MN`."
- `in_progress` → "MN is still in progress. Complete the review cycle: `/review-implementation PLAN-NNNN MN`."
- `approved` → "MN is approved but tests have not run. Run `/execute-tests PLAN-NNNN MN TEST-NNNN`."
- `test_failed` → "MN has P0 failures. Run `/address-test-failures PLAN-NNNN MN` to resolve them."
- `pr_open` → "MN's PR is already open. Merge it on GitHub to proceed."
- `pr_merged` → "MN is fully complete — PR has been merged. Nothing to do."

## Step 1 — Collect PR content

Read the following to build the PR description:

**From PLAN-NNNN milestone MN:**
- Task list (T-IDs and descriptions)
- `**AC Coverage:**` field
- `**depends_on_milestones:**`
- Latest `### Test Execution — Round N` section (P0 results, date, mode)

**From `sdlc/reviews/IMPL-REVIEW-NNNN-MN.md`:**
- Count of review rounds (number of `## Round N` sections)
- Final round summary line (`**Summary:** N blocking, M non-blocking findings`)
- Count of resolved blocking findings across all Address Summaries

**From TEST-NNNN:**
- AC coverage map entries for MN's ACs (test case IDs and their descriptions)

## Step 2 — Generate PR description

Build the PR body using the content collected above:

```markdown
## Summary

[2–4 bullet points drawn from MN's task descriptions — what was built, not how]

## Acceptance criteria

| AC | Description | Test cases |
|---|---|---|
| AC-N | [AC text from SPEC] | TC-ACN-1, TC-ACN-2 |

## Implementation review

- **Rounds:** N round(s) of `/review-implementation PLAN-NNNN MN`
- **Findings resolved:** N blocking findings resolved across all rounds
- **Artifact:** `sdlc/reviews/IMPL-REVIEW-NNNN-MN.md`

## Test results

- **P0:** N passed, 0 failed — [date], [local | CI]
- **P1+:** [N passed / Z deferred-to-release]
- **Detail:** `sdlc/plans/PLAN-NNNN-[slug].md` > MN > Test Execution — Round N

## Artifacts

- Plan: `sdlc/plans/PLAN-NNNN-[slug].md` (MN section)
- Review: `sdlc/reviews/IMPL-REVIEW-NNNN-MN.md`
- Test plan: `sdlc/test-plans/TEST-NNNN-[slug].md`
```

## Step 3 — Show and confirm

Present the generated title and description to the user:

> **PR ready to create:**
>
> **Title:** `feat(MN): [milestone name] — PLAN-NNNN`
>
> **Branch:** `feature/PLAN-NNNN-MN` → `main`
>
> [full description as generated above]
>
> Confirm to create the PR?

Wait for confirmation before creating.

## Step 4 — Create the PR

After user confirms:

```bash
gh pr create \
  --title "feat(MN): [milestone name] — PLAN-NNNN" \
  --body "[generated description]" \
  --base main \
  --head feature/PLAN-NNNN-MN
```

## Step 5 — Update milestone status

Update PLAN-NNNN milestone MN:

```
**Implementation status:** pr_open
```

Commit the status update on the milestone branch:

```bash
git add sdlc/plans/PLAN-NNNN-[slug].md
git commit -m "chore(MN): PR open — implementation_status pr_open"
git push origin feature/PLAN-NNNN-MN
```

## Step 6 — Handoff

Output the PR URL returned by `gh pr create`, then:

> **MN PR is open:** [PR URL]
>
> Merge the PR on GitHub when ready.
>
> [If more milestones remain:]
> Once merged, run `/implement-plan PLAN-NNNN M(N+1)` — it will detect the merge and update MN's status to `pr_merged` automatically.
>
> [If this is the last milestone:]
> Once merged, all milestones are complete. The feature is ready for Release Planning — SRE runs `/create-release-plan vX.Y.Z`.

## Notes

- Do not create the PR until the milestone is `tests_passed` — reviewers should see a diff that is already verified
- The PR branch is `feature/PLAN-NNNN-MN` — never a generic `feature/PLAN-NNNN` branch; one PR per milestone keeps diffs small and reviewable
- GitHub PR reviewers see the code diff; the PR description gives them the "why" without needing to read the SDLC artifacts directly
- The PR description's test results and review round count give reviewers confidence the code has already been rigorously reviewed by four role perspectives
