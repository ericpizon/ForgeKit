---
id: create-pr
type: command
name: /create-pr
description: Developer command to close out a completed milestone — opens a GitHub PR (team/remote) or fast-forward merges directly to main (solo/no remote), generating a PR description from PLAN, IMPL-REVIEW, and test results either way
status: active
version: 2
primary_role: developer
changelog:
  - v2 — added the local-only path (no `gh`, no remote, or declined ceremony): fast-forward merge straight from `tests_passed` to `pr_merged`, skipping `pr_open` and `/close-milestone` entirely. Found during a full-pipeline drift audit — this command had no fallback at all, unlike its sibling `/close-milestone`'s documented "if `gh` isn't authenticated or there's no remote" escape hatch, despite this exact gap being worked around ad hoc (undocumented) on every solo ForgeKit milestone to date.
links:
  - relation: follows
    target: execute-tests
  - relation: precedes
    target: close-milestone   # real-PR path only
  - relation: precedes
    target: implement-plan
---

# /create-pr

## Purpose

Close out a completed milestone, from its feature branch (`feature/PLAN-NNNN-MN`) to `main`. Generates a title and description from the PLAN task list, IMPL-REVIEW history, and test execution results either way. **Team/remote context:** opens a real GitHub PR and sets `implementation_status: pr_open`, pending a human merge on GitHub. **Solo or no-remote context:** fast-forward merges directly into `main` and sets `implementation_status: pr_merged` in one step — there's no PR to review asynchronously when one person wrote and is also closing the milestone.

The next milestone cannot start until MN reaches `pr_merged` — either via a merged PR (`/close-milestone` records it) or directly on the local-only path. `/implement-plan` detects a merged PR automatically when it runs the pre-flight for the next milestone; the local-only path sets `pr_merged` itself, so there's nothing further to detect.

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
> Confirm: (1) proceed? (2) push and open a real GitHub PR, or local-only (no `gh`/no remote, or you simply don't want the ceremony) — fast-forward merge directly into `main` and skip straight to `pr_merged`?

Wait for confirmation before proceeding. If `gh` is not authenticated or the repo has no remote at all, say so explicitly and default to recommending local-only rather than letting Step 4 fail opaquely.

## Step 4 — Create the PR *(real-PR path only)*

After the user confirms the real-PR path:

```bash
gh pr create \
  --title "feat(MN): [milestone name] — PLAN-NNNN" \
  --body "[generated description]" \
  --base main \
  --head feature/PLAN-NNNN-MN
```

## Step 5 — Update milestone status *(real-PR path only)*

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

## Step 4b/5b — Local-only path (no `gh`, no remote, or declined ceremony)

Skip Step 4/5 entirely — there is no PR to open, so `pr_open` is skipped too; go straight from `tests_passed` to `pr_merged` in one move, mirroring what the real-PR path accomplishes in two:

```bash
git checkout main
git merge --ff-only feature/PLAN-NNNN-MN
```

If the merge isn't a clean fast-forward (main has diverged — unusual for a solo/local workflow but possible), stop and surface it rather than forcing a merge commit silently; ask the user how they want to reconcile it.

Update PLAN-NNNN milestone MN directly to the final status:

```
**Implementation status:** pr_merged
```

Commit on `main` (no feature-branch status commit needed — there's no PR history to preserve it against):

```bash
git add sdlc/plans/PLAN-NNNN-[slug].md
git commit -m "chore(MN): mark pr_merged - no remote PR ceremony (local-only, no push)"
```

Skip straight to Step 6's Handoff below — there is no `/close-milestone` step on this path either (that command verifies a *merged PR*, which doesn't exist here); this single commit *is* the milestone's closure.

## Step 6 — Handoff

**Real-PR path:** output the PR URL returned by `gh pr create`, then:

> **MN PR is open:** [PR URL]
>
> Merge the PR on GitHub when ready, then run `/close-milestone PLAN-NNNN MN` to record it and advance.

**Local-only path:**

> **MN is merged into `main` and closed** — `pr_merged`, no remote PR ceremony.

**Either path, once MN is `pr_merged`:**

> [If more milestones remain:]
> Run `/implement-plan PLAN-NNNN M(N+1)` — it will detect MN's `pr_merged` status automatically.
>
> [If this is the last milestone:]
> All milestones are complete. The feature is ready for Release Planning — SRE runs `/create-release-plan vX.Y.Z`.

## Notes

- Do not proceed (either path) until the milestone is `tests_passed` — a diff that hasn't passed its own tests shouldn't be merged, reviewed or not
- The milestone branch is `feature/PLAN-NNNN-MN` — never a generic `feature/PLAN-NNNN` branch; one PR (or one local merge) per milestone keeps diffs small and reviewable
- **Real-PR path:** GitHub PR reviewers see the code diff; the PR description gives them the "why" without needing to read the SDLC artifacts directly. The test results and review round count give reviewers confidence the code has already been rigorously reviewed by four role perspectives.
- **Local-only path:** the same generated description is still worth showing the user before merging, even with no external reviewer — it's the last checkpoint before the milestone's diff becomes part of `main`'s permanent history.
- Ask explicitly rather than assuming: a repo with a configured remote and working `gh` auth might still be a solo project where the user prefers the lighter local-only path (no PR ceremony for its own sake); don't force the real-PR path just because it's technically possible.
