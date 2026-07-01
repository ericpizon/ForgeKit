---
id: approve-implementation
type: command
name: /approve-implementation
description: Lead Developer command to close the implementation phase for a milestone — zero blocking findings gate, updates SEC findings status, OBS instrumentation log, and setup documentation; commits all milestone artifacts together
status: active
version: 1
primary_role: lead-developer
---

# /approve-implementation

## Purpose

Close the implementation phase for a milestone. This is the first and only git commit that captures the milestone's source code — all generated and revised files have lived in the working tree (uncommitted) since `/implement-plan` started. This commit bundles everything: production code, test code, migrations, the review artifact, the updated PLAN status, any SEC findings now mitigated, any OBS tasks now implemented, and the final status update.

## Usage

```
/approve-implementation PLAN-NNNN M1
```

## Gate — Zero Blocking Findings Required

Read `sdlc/reviews/IMPL-REVIEW-NNNN-MN.md`. Read `current_review_round` from frontmatter and navigate directly to `## Round {current_review_round}`. Check its `**Summary:**` line.

If blocking > 0:
> "Approval blocked: N blocking finding(s) remain in Round X. Run `/address-implementation-review PLAN-NNNN MN` to resolve them."

Do not proceed until the latest round summary shows `0 blocking`.

## Actions

### 1. Update milestone status

In PLAN-NNNN, update milestone MN:

```
**Implementation status:** approved
```

### 2. Pre-commit checks

Run the project's lint and unit/integration tests as a safety net. Read project.yml for the commands:

```bash
{{stack.lint}}
{{stack.test.unit}}
```

If `stack.lint` is `none`, skip the lint step. If any fail, fix before proceeding.

### 3. Update BUG-NNNN status *(only if PLAN-NNNN has `linked_bug`)*

If `linked_bug` is set in PLAN-NNNN frontmatter, update `sdlc/bugs/BUG-NNNN.md` frontmatter:

```yaml
status: fix_merged
```

This records that the fix has passed implementation review — the code is in the branch, ready for merge. The status advances to `pending_verification` in `/execute-release`, after the fix is deployed to production.

If PLAN-NNNN has no `linked_bug`, skip this step.

### 4. Update SEC-NNNN findings_status

Read PLAN-NNNN milestone MN's task list. Identify any task that implements a SEC-NNNN finding mitigation.

If SEC-NNNN exists for this thread and MN has security tasks:

1. Read `sdlc/security/SEC-NNNN-[slug].md`
2. For each finding ID referenced by MN's tasks, if `open` and the implementation review confirmed the mitigation is in place, update the frontmatter entry to `mitigated`:

```yaml
findings_status:
  F1: mitigated    # implemented in MN
  F2: open         # not in scope for MN — leave unchanged
```

If no MN tasks reference SEC findings, skip this step.

### 5. Update OBS-NNNN instrumentation log

Read PLAN-NNNN milestone MN's task list. Identify any observability tasks.

If OBS-NNNN exists and MN has OBS tasks, append (or update) an `## Instrumentation Implementation Log` section:

```markdown
## Instrumentation Implementation Log

| Task | Milestone | Status | Date |
|---|---|---|---|
| [OBS task description] | MN | implemented | [today] |
```

If the section already exists (from a prior milestone), append new rows — do not overwrite prior milestone rows.

### 6. Update project setup documentation for new environment variables or tooling

Scan MN's task descriptions, OBS-NNNN signal configuration, and SEC-NNNN findings implemented in MN for new environment variables, new packages, or new CLI tooling prerequisites.

**If new environment variables are found:** update the project's setup documentation (SETUP.md, README.md, or equivalent — check ARCH.md for where setup instructions live) with the new variables, their descriptions, and example values. Update the `.env` example block and production environment section if applicable.

**If new CLI tooling prerequisites are introduced:** document them in the prerequisites section of the project's setup documentation.

If MN introduces nothing new, skip this step.

### 7. Pause for user review

Show the user the full diff (all files changed since the `in_progress` commit). Ask:

> **MN — [milestone name] is ready to commit.**
>
> All checks pass. Zero blocking findings in the implementation review.
> [If SEC updated:] SEC-NNNN: N finding(s) marked `mitigated`.
> [If OBS updated:] OBS-NNNN: N instrumentation task(s) logged as implemented.
> [If setup docs updated:] Setup documentation: N new environment variable(s) documented.
>
> Please review the diff above.
> Confirm: (1) proceed with commit? (2) solo project (commit only, no push) or team project (push too)?

Wait for confirmation before committing.

### 8. Commit (after user confirms)

Run `git status` to see all modified and new files. Stage files that belong to this milestone — do not use `git add -A`:

```bash
git status

# Stage source code, tests, migrations, and SDLC artifacts for this milestone
git add [source code files from git status]
git add [test files from git status]
git add [migration files, if any]
git add sdlc/plans/PLAN-NNNN-[slug].md
git add sdlc/reviews/IMPL-REVIEW-NNNN-MN.md
git add sdlc/bugs/BUG-NNNN.md                  # only if linked_bug is set (step 3)
git add sdlc/security/SEC-NNNN-[slug].md       # only if updated in step 4
git add sdlc/observability/OBS-NNNN-[slug].md  # only if updated in step 5
git add SETUP.md                                # only if updated in step 6 (or equivalent setup doc)

git commit -m "feat(MN): [milestone name] — [brief summary] ([AC list])"
# for hotfix plans (type: hotfix): git commit -m "fix(BUG-NNNN): [milestone name] — [brief summary]"
```

**Push only if the user confirms team context:**

```bash
git push -u origin feature/PLAN-NNNN-MN
```

### 9. Implementation phase complete

> **Implementation phase complete — MN is committed and the branch is updated.**
>
> **Testing phase:** When QE is ready, run `/execute-tests PLAN-NNNN MN TEST-NNNN` to execute automated tests and record results.
>
> Note: the next milestone cannot start until MN reaches `pr_merged`. `approved` does not unlock dependencies — tests must pass and the PR must be merged.
