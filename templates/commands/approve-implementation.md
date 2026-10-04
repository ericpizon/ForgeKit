---
id: approve-implementation
type: command
name: /approve-implementation
description: Lead Developer command to close the implementation phase for a milestone — zero blocking findings gate, updates SEC findings status, OBS instrumentation log, setup documentation, and ARCH.md structural drift; commits all milestone artifacts together
status: active
version: 3
primary_role: lead-developer
changelog:
  - v2 — added the thread-participant path for Steps 4/5/8 — SEC-NNNN and OBS-NNNN live in the coordination repo (sdlc/THREAD-NNNN/SEC.md, OBS.md) for a thread, not this repo's own sdlc/security/, sdlc/observability/; updates to those shared files are applied directly if reachable this session, otherwise handed off, and always committed separately from this repo's own milestone commit.
  - v3 — Step 6 now also syncs ARCH.md's Module layout/Testing strategy/Observability sections against what the milestone actually built — no command in the pipeline previously wrote this back (implement-plan only reads ARCH.md), which let these sections drift from reality over a project's life. Scoped to the current milestone's own changes; `/update-arch` remains the tool for a full re-scan or catching up pre-existing drift.
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

Update `IMPL-REVIEW-NNNN-MN.md` frontmatter to record the approval:

```yaml
approval:
  mode: repo_only
  approved_by: "[Lead Developer name]"
  approved_role: lead-developer
  approved_at: "[today's date]"
  decision_summary: "[one sentence confirming milestone MN approved for merge]"
  external_event_id: null
  external_event_uri: null
policy_evaluation:
  evaluated: null
  policy_pack: null
  result: null
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

1. **`repo_role: code` (default):** read `sdlc/security/SEC-NNNN-[slug].md`. **Thread participant (PLAN-NNNN has a `thread`/`coordination_repo` pointer):** SEC-NNNN lives at `sdlc/THREAD-NNNN/SEC.md` in the coordination repo instead — resolve it there (per [`templates/reference/cross-repo-resolution.md`](../reference/cross-repo-resolution.md)).
2. For each finding ID referenced by MN's tasks, if `open` and the implementation review confirmed the mitigation is in place, update the frontmatter entry to `mitigated`:

```yaml
findings_status:
  F1: mitigated    # implemented in MN
  F2: open         # not in scope for MN — leave unchanged
```

**Thread participant:** this is a write to a file shared by every participant. Apply it directly if the coordination repo is reachable as a working directory this session (same as `approve-design`'s ARCH.md handling); otherwise hand it off explicitly, naming the exact finding ID and status change — never silently skip it because the coordination repo isn't locally reachable. Either way, this update is committed separately, in the coordination repo, not as part of this milestone's commit (Step 8).

If no MN tasks reference SEC findings, skip this step.

### 5. Update OBS-NNNN instrumentation log

Read PLAN-NNNN milestone MN's task list. Identify any observability tasks.

If OBS-NNNN exists and MN has OBS tasks: **`repo_role: code` (default)** reads/updates `sdlc/observability/OBS-NNNN-[slug].md` directly. **Thread participant:** OBS-NNNN lives at `sdlc/THREAD-NNNN/OBS.md` in the coordination repo instead — the same apply-directly-or-hand-off rule from Step 4 applies, and this update is likewise committed separately in the coordination repo. Append (or update) an `## Instrumentation Implementation Log` section:

```markdown
## Instrumentation Implementation Log

| Task | Milestone | Repo | Status | Date |
|---|---|---|---|---|
| [OBS task description] | MN | [this repo, for a thread] | implemented | [today] |
```

If the section already exists (from a prior milestone), append new rows — do not overwrite prior milestone rows. For a thread's shared OBS-NNNN, the log accumulates rows from every participant, so the `Repo` column is what tells you which one did which — omit that column entirely for a `repo_role: code` project with no thread.

### 6. Update project setup documentation, and sync ARCH.md structural drift

Scan MN's task descriptions, OBS-NNNN signal configuration, and SEC-NNNN findings implemented in MN for new environment variables, new packages, or new CLI tooling prerequisites.

**If new environment variables are found:** update the project's setup documentation (SETUP.md, README.md, or equivalent — check ARCH.md for where setup instructions live) with the new variables, their descriptions, and example values. Update the `.env` example block and production environment section if applicable.

**If new CLI tooling prerequisites are introduced:** document them in the prerequisites section of the project's setup documentation.

**ARCH.md structural sync.** `/implement-plan` only ever reads ARCH.md — nothing in the pipeline writes back what a milestone actually built, which is why ARCH.md's Module layout, Testing strategy, and Observability sections drift from reality over the life of a project. This step closes that gap at the one point that already has the full picture of what changed. Compare the files actually created or modified in MN (from `git status`, cross-referenced against MN's task list) against ARCH.md's own sections:

- **`## Module layout`:** any new top-level file or directory under the project's source root not already represented in the current tree — add it, matching the existing tree's own level of detail (file name plus a one-line purpose comment; do not add a comment for every function).
- **`## Testing strategy`:** any new test file that exercises a module category not already named in the Unit/Integration/E2E bullets — add a one-line mention of what it covers.
- **`## Observability`:** cross-reference against this same step's own OBS-NNNN instrumentation log update for MN (Step 5 above) — any signal now `implemented` for this repo that ARCH.md's `Key signals` line doesn't list — add it there too.

This is a structural sync scoped to what MN's own tasks actually touched — not a full project re-scan. For a large refactor, or to catch up an ARCH.md that drifted before this step existed, use `/update-arch` separately instead.

Draft the proposed ARCH.md changes now, but do not treat them as pre-approved — they are shown to the user as part of MN's full diff in Step 7 below, under the same explicit-confirmation gate as everything else in that diff. Never write them unreviewed.

If MN introduces no structural change beyond what ARCH.md already reflects, state that explicitly and skip the ARCH.md diff (the env-var/tooling checks above still apply independently).

If MN introduces nothing new on any of the above, skip this step entirely.

### 7. Pause for user review

Show the user the full diff (all files changed since the `in_progress` commit). Then ask:

> **MN — [milestone name] is ready to commit.**
>
> All checks pass. Zero blocking findings in the implementation review.
> [If SEC updated:] SEC-NNNN: N finding(s) marked `mitigated`.
> [If OBS updated:] OBS-NNNN: N instrumentation task(s) logged as implemented.
> [If setup docs updated:] Setup documentation: N new environment variable(s) documented.
> [If ARCH.md updated:] ARCH.md: Module layout/Testing strategy/Observability sections updated to reflect MN's actual changes.

> **Before you confirm — three questions:**
> 1. Which part of this implementation is most complex? Is that part adequately tested by the milestone's test cases?
> 2. Is there anything in this diff you would be uncomfortable defending at 3am during an incident? If yes, say so now.
> 3. Does every changed file match what DESIGN-NNNN specified, or were implementation-time judgment calls made that DESIGN did not address?
>
> If any answer reveals a gap: resolve it before confirming.
>
> If all three check out: "Confirm: (1) proceed with commit? (2) solo project (commit only, no push) or team project (push too)?"

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
git add SETUP.md                                # only if updated in step 6 (or equivalent setup doc)
git add sdlc/architecture/ARCH.md               # only if updated in step 6 (structural sync)

# repo_role: code (default) only — thread participants never have these locally:
git add sdlc/security/SEC-NNNN-[slug].md       # only if updated in step 4
git add sdlc/observability/OBS-NNNN-[slug].md  # only if updated in step 5

git commit -m "feat(MN): [milestone name] — [brief summary] ([AC list])"
# for hotfix plans (type: hotfix): git commit -m "fix(BUG-NNNN): [milestone name] — [brief summary]"
```

**Thread participant:** SEC-NNNN/OBS-NNNN updates from Steps 4/5 are never part of this commit — they belong to the coordination repo's own history. If applied directly this session, commit them there separately (`git add sdlc/THREAD-NNNN/SEC.md` / `sdlc/THREAD-NNNN/OBS.md`, in the coordination repo's working tree); if handed off, the coordination-repo commit happens whenever that hand-off is carried out, not now.

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
