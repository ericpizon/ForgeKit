---
id: release-hotfix
type: command
name: /release-hotfix
description: Lead Developer command to deliver a Critical or P0 bug fix through the full pipeline with minimal artifacts — one-task PLAN, one-test TEST-NNNN, implement, review, test, release
status: active
version: 1
primary_role: lead-developer
---

# /release-hotfix

## Purpose

Deliver a targeted fix for a Critical or P0 bug through the full pipeline with minimal artifacts. The pipeline is the same as any fix — plan, implement, review, test, release — but the artifacts are stripped to the minimum: one milestone, one task, one test case. Priority determines urgency and timeline, not which steps run.

No steps are skipped. A rushed fix that bypasses review or testing is a liability, not a solution.

## Usage

```
/release-hotfix BUG-NNNN
```

BUG-NNNN must be `accepted` with `severity: Critical` or `priority: P0`. If neither condition is met, stop: "Hotfix path is reserved for Critical severity or P0 bugs. For lower severity/priority, use the standard fix path from `/triage-bug`."

---

## Step 1 — Read BUG-NNNN and confirm scope

Read `sdlc/bugs/BUG-NNNN.md`. Confirm `status: accepted` and `severity: Critical` OR `priority: P0`. Root cause hypothesis must be present in the `## Triage` section.

Present a fix scope summary for the user to confirm:

> **Fix scope for BUG-NNNN — [title]**
>
> Root cause: [from triage section]
> Proposed fix: [one sentence — what changes and why it resolves the root cause]
> Files likely affected: [list — be specific]
> Risk: [what adjacent behavior could be affected]
>
> Confirm to proceed?

Do not proceed without explicit confirmation.

---

## Step 2 — Create minimal PLAN

Determine the next PLAN ID from `sdlc/plans/`. Create `sdlc/plans/PLAN-NNNN.md`:

```markdown
---
id: PLAN-NNNN
type: hotfix
status: approved
linked_bug: BUG-NNNN
version: 1
created: [date]
generation:
  command: /release-hotfix
  model: "[self-identify from system context]"
  generated_at: "[today's date]"
---

# PLAN-NNNN — Hotfix: [BUG-NNNN title]

## Context

Hotfix plan for BUG-NNNN. Root cause: [from triage]. Minimal plan — one milestone, one task. Full SPEC/ANALYSIS/DESIGN phases not required; the bug report and triage decision establish the scope.

## Milestone 1 — Fix [root cause summary]

**Implementation status:** pending
**Branch:** feature/PLAN-NNNN-M1

### Tasks

- T1: [specific code change — what file, what logic, why this fixes the root cause]
- T2: [schema migration — only if required; otherwise omit]

### Acceptance Criteria

- AC1: Reproduction steps from BUG-NNNN no longer produce the defect
- AC2: Test case TC-BUG-NNNN passes in CI
- AC3: No regression in adjacent paths confirmed by review
```

The PLAN is pre-approved — the triage decision already authorized the fix scope. No separate `/approve-plan` step.

---

## Step 3 — Create minimal TEST-NNNN

Create `sdlc/test-plans/TEST-NNNN.md` with one test case that proves the defect existed and confirms the fix:

```markdown
---
id: TEST-NNNN
status: approved
linked_plan: PLAN-NNNN
linked_bug: BUG-NNNN
version: 1
generation:
  command: /release-hotfix
  model: "[self-identify from system context]"
  generated_at: "[today's date]"
---

# TEST-NNNN — Hotfix test: [BUG-NNNN title]

## Test Case

**ID:** TC-BUG-NNNN-1
**Type:** [integration | e2e — match the layer where the defect manifests]
**Execution:** automated
**Priority:** P0

**Scenario:** [BUG-NNNN title]

Given [state that sets up the defect condition — from BUG-NNNN reproduction steps]
When [the action that triggers the defect]
Then [the correct behavior — from BUG-NNNN expected behavior]

## Regression Check

[1–2 additional test cases for adjacent behavior most at risk from this fix]
```

The TEST-NNNN is pre-approved. Write the test case first — it must fail before the fix is applied. This is the minimum TDD gate for a hotfix.

---

## Step 4 — Implement

Run `/implement-plan PLAN-NNNN M1`.

`/implement-plan` creates the branch (`feature/PLAN-NNNN-M1`) and generates the fix code.

---

## Step 4b — Run tests

Run `/execute-tests PLAN-NNNN M1`.

The failing test from Step 3 (TC-BUG-NNNN-1) must pass after the fix. The regression check cases must also pass. CI must be green before proceeding to review.

---

## Step 5 — Review

Run the standard implementation review pipeline:

- `/review-implementation PLAN-NNNN M1` — Lead Developer reviews the diff for correctness, scope creep, and regression risk
- If the fix touches auth, authorization, credential handling, input validation, or data access paths: the Security perspective in `/review-implementation` must be applied with extra scrutiny

Address any blocking findings before merging. Do not skip review to save time — a review miss on a hotfix can introduce a second incident.

---

## Step 6 — Merge and release

After review is approved, run `/approve-implementation PLAN-NNNN M1` to commit the fix.

Open a PR and merge it:
1. `/create-pr PLAN-NNNN M1` — open a PR for the hotfix branch
2. Merge the PR on GitHub
3. `/close-milestone PLAN-NNNN M1` — record the merge; advances status to `pr_merged`

Then run `/create-release-plan vX.Y.Z` to create a minimal RELEASE-PLAN. The hotfix release plan references BUG-NNNN as the scope and notes the patch version increment (e.g., v1.2.3 → v1.2.4).

Then: `/review-release-plan` → `/approve-release-plan` → `/execute-release`.

---

## Step 7 — Update BUG-NNNN

Append to the `## Fix` section in BUG-NNNN:

```markdown
## Fix

**Fix type:** Hotfix
**Plan:** PLAN-NNNN
**Test:** TEST-NNNN
**Branch:** feature/PLAN-NNNN-M1
**Released in:** [vX.Y.Z]
**Fix description:** [root cause confirmed, what was changed, why the fix is correct]
```

Update BUG-NNNN frontmatter:
```yaml
status: pending_verification
fix_plan: PLAN-NNNN
fix_milestone: M1
```

---

## Quality Gate

- [ ] Fix scope confirmed by user before any artifact was created
- [ ] Minimal PLAN-NNNN created with `type: hotfix` and one milestone, `status: approved`
- [ ] Test case TC-BUG-NNNN-1 written before fix; confirmed failing before fix, passing after `/execute-tests`
- [ ] Regression check cases pass
- [ ] CI green — no regressions
- [ ] Implementation review completed; Security perspective applied if fix touches auth/data paths
- [ ] All blocking review findings resolved
- [ ] RELEASE-PLAN created, reviewed, and approved before `/execute-release`
- [ ] BUG-NNNN updated to `pending_verification`

---

## Handoff

> "Hotfix deployed as [vX.Y.Z]. BUG-NNNN is now `pending_verification`."
>
> Run `/verify-bug BUG-NNNN` after confirming the fix is live in production to close the bug.
