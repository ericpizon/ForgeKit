---
id: approve-test-plan
type: command
name: /approve-test-plan
description: Quality Engineer command to close the test plan review loop — confirm all findings resolved, mark TEST-PLAN approved, trigger implementation
status: active
version: 1
primary_role: quality-engineer
output_artifact_type: test-plan (approved)
links:
  - relation: relates_to
    target: create-test-plan
  - relation: relates_to
    target: review-test-plan
  - relation: relates_to
    target: address-test-plan-review
---

# /approve-test-plan

## Purpose

Close the test plan review loop. Confirm the latest review section shows zero blocking findings. Mark TEST-NNNN `approved` and trigger Developer implementation. TEST-NNNN approval is the gate before a single line of implementation code is written — the AC coverage map is the Developer's test contract.

## When to Invoke

- Latest `## Test Plan Review — Round N` in TEST-NNNN shows zero blocking findings
- Do not invoke if any blocking finding remains unresolved

Pass the TEST-PLAN ID: `/approve-test-plan TEST-0001`.

## Required Inputs

- **TEST-NNNN** — latest Test Plan Review section must show zero blocking findings; all Address Summaries present
- **PLAN-NNNN** — `pipeline_status.test` will be updated to `approved` in Step 3
- **SEC-NNNN** — (if triggered) determines whether Security perspective findings are required to be resolved

## Steps

Adopt the Quality Engineer role. Apply the perspective and stack knowledge from [`.claude/roles/quality-engineer.md`](.claude/roles/quality-engineer.md) throughout these steps.

1. **Final readiness gate.** Read `current_review_round` from TEST-NNNN frontmatter. Navigate directly to `## Test Plan Review — Round {current_review_round}` — do not scan for it. Confirm:
   - Zero blocking findings
   - Every non-blocking finding is either addressed in the Address Summary or documented as accepted risk with rationale
   - Every AC from SPEC-NNNN has at least one P0 automated test case covering the happy path
   - Execution Results Summary placeholder is present (`status: pending`)

   If any blocking finding remains: **do not proceed**. Return to `/address-test-plan-review`.

2. **Update TEST-NNNN status to `approved`.** In TEST frontmatter:
   - Set `status: approved`
   - Add `approval:` block:
     ```yaml
     approval:
       mode: repo_only
       approved_by: "[Quality Engineer name]"
       approved_role: quality-engineer
       approved_at: "[today's date]"
       decision_summary: "[one sentence confirming TEST-NNNN approved for implementation phase]"
       external_event_id: null
       external_event_uri: null
     policy_evaluation:
       evaluated: null
       policy_pack: null
       result: null
     ```
   - Add changelog entry: `vN — approved after [N] review rounds`

3. **Update PLAN-NNNN pipeline_status.** In PLAN-NNNN frontmatter, set `pipeline_status.test: approved`.

4. **Update the registry.** In `sdlc/registry/index.md`, update the TEST-NNNN row: set `status: approved`.

## Quality Gate

Before committing:
- [ ] Latest Test Plan Review section shows zero blocking findings
- [ ] All non-blocking findings addressed or explicitly accepted as risk with documented rationale
- [ ] Every AC has at least one P0 automated test case covering the happy path
- [ ] No P0 test case is manual-only without a `Procedure` block and a `Reason` line
- [ ] All security test cases for Critical/High SEC findings are P0
- [ ] AC Coverage Map is organized by milestone matching the PLAN structure
- [ ] Execution Results Summary placeholder present with `status: pending`
- [ ] TEST frontmatter: `status: approved`; `approval:` block set (`approved_by`, `approved_role: quality-engineer`, `approved_at`, `decision_summary` populated)
- [ ] PLAN-NNNN `pipeline_status.test` updated to `approved`
- [ ] Registry updated for TEST-NNNN

**Self-check before proceeding:** Re-read each item above. Confirm each is complete, or state what was skipped and why before continuing to Handoff.

## Handoff

Show the user the approved TEST-PLAN. This is the single commit for the entire test planning phase.

**Pause for human review:**

Ask:

> **Before we commit — three questions:**
> 1. Pick the P0 test case you are least confident about. Can it run in CI as written, or does it have hidden dependencies or environment assumptions?
> 2. Is there a failure path — unauthorized access, resource not found, invalid input — that has no corresponding test case in the coverage map?
> 3. Review any deferred test cases. Is anything deferred for convenience rather than a genuine production-environment constraint?
>
> If any answer reveals a gap: return to `/address-test-plan-review TEST-NNNN`.
>
> If all three check out: "Ready to commit the entire test planning phase — TEST-NNNN with all review rounds, address summaries, and approval. Confirm: (1) proceed with the commit? (2) solo project (no push) or team project (push too)?"

**Commit (after user confirms):**
```
git add sdlc/test-plans/TEST-NNNN-slug.md sdlc/registry/index.md
git commit -m "TEST-NNNN: test planning phase approved — [N] review rounds"
```

**Push (only if user confirms team context):**
```
git push origin thread/NNNN-slug
```

**Notify:**
- **Developer** → `/implement-plan PLAN-NNNN` — TEST-NNNN is the test contract; the AC coverage map defines what test cases to write alongside each milestone task

**After implementation and CI (separate commit, not part of this phase):**
The QE records test execution results directly in TEST-NNNN — updating the Execution Results Summary section and setting `execution_date` and `execution_summary` in frontmatter. A `failed` or unexplained `partial` result blocks RELEASE-PLAN approval.
