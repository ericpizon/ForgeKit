---
id: close-milestone
type: command
name: /close-milestone
description: Developer command to record a merged PR and close a milestone — verifies the PR is merged, updates implementation_status to pr_merged on main, and routes to the next milestone or Release Planning
status: active
version: 1
primary_role: developer
links:
  - relation: follows
    target: create-pr
  - relation: precedes
    target: implement-plan
---

# /close-milestone

## Purpose

Record that a milestone's PR has been merged into main. Verifies the merge on GitHub, switches to main, updates `implementation_status: pr_merged` in PLAN-NNNN, and tells the developer what comes next — the next unblocked milestone or Release Planning if all milestones are complete.

Run this after merging the PR on GitHub, before starting the next milestone.

## Usage

```
/close-milestone PLAN-0001 M1
```

## Pre-flight Check

Read PLAN-NNNN milestone MN's `**Implementation status:**`. Must be `pr_open`. If not, stop:

- `pending` → "MN has not been implemented. Run `/implement-plan PLAN-NNNN MN`."
- `in_progress` → "MN is still in the review cycle. Complete it before closing."
- `approved` → "MN is approved but tests have not run. Run `/execute-tests PLAN-NNNN MN TEST-NNNN`."
- `test_failed` → "MN has P0 failures. Run `/address-test-failures PLAN-NNNN MN`."
- `tests_passed` → "MN tests are passing but no PR has been created. Run `/create-pr PLAN-NNNN MN`."
- `pr_merged` → "MN is already closed."

## Step 1 — Verify the PR is merged

```bash
gh pr list --head feature/PLAN-NNNN-MN --state merged
```

If the PR does not appear as merged, stop:

> "The PR for MN (`feature/PLAN-NNNN-MN`) does not appear to be merged yet. Merge it on GitHub, then re-run `/close-milestone PLAN-NNNN MN`."

## Step 2 — Switch to main and pull

```bash
git checkout main
git pull origin main
```

This brings in all merged content — SDLC artifacts, MN's source code, and the `pr_open` status commit from `/create-pr`.

## Step 3 — Update milestone status

In PLAN-NNNN, update milestone MN:

```
**Implementation status:** pr_merged
```

Commit the update:

```bash
git add sdlc/plans/PLAN-NNNN-[slug].md
git commit -m "chore(MN): PR merged — implementation_status pr_merged"
```

**Solo project:** push directly to main:
```bash
git push origin main
```

**Team project:** if branch protection prevents direct pushes to main, skip the push — the status is recorded locally and will be visible when the next PR is opened. Alternatively, push if your team allows direct status-tracking commits on main.

## Step 4 — Determine next step

Read all milestones in PLAN-NNNN. Check their `**Implementation status:**` and `**depends_on_milestones:**` fields.

**If one or more milestones are now unblocked** (all their `depends_on_milestones` are `pr_merged` and their own status is `pending`):

> **MN is closed.** The following milestone(s) are now unblocked:
>
> - MX — [milestone name] (`depends_on_milestones` satisfied)
>
> Run `/implement-plan PLAN-NNNN MX` to begin.

**If all milestones are `pr_merged`:**

> **All milestones are complete — the feature is ready for Release Planning.**
>
> SRE runs `/create-release-plan vX.Y.Z` to begin the release phase.

## Notes

- Do not run this command before the PR is merged — the pre-flight check enforces this, but the `gh` verification is the authoritative gate
- If `gh` is not authenticated or the repo has no remote, skip step 1 and confirm the merge manually before proceeding
- The `pr_merged` status commit on main is a lightweight tracking update — it does not affect the feature code, only the SDLC artifact tracking field
