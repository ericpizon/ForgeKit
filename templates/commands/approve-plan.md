---
id: approve-plan
type: command
name: /approve-plan
description: Lead Developer command to close the plan review loop — confirm all findings resolved, mark PLAN approved, trigger TEST-PLAN authorship
status: active
version: 1
primary_role: lead-developer
output_artifact_type: plan (approved)
---

# /approve-plan

## Purpose

Close the plan review loop. Confirm all required review sections are present and all blocking findings are resolved or accepted as documented risk. Mark the PLAN `approved`, then trigger QE to begin TEST-PLAN authorship.

This is the exit gate for Stage 5. TEST-PLAN authorship does not begin before `/approve-plan` runs. Developer implementation does not begin before TEST-NNNN is approved.

## When to Invoke

- Latest `## Plan Review — Round N` in PLAN-NNNN shows zero blocking findings
- Do not invoke if any blocking finding remains unresolved

Pass the PLAN ID: `/approve-plan PLAN-NNNN`.

## Steps

1. **Final readiness gate.** Read `current_review_round` from PLAN-NNNN frontmatter. Navigate directly to `## Plan Review — Round {current_review_round}` — do not scan for it. Confirm:
   - Zero blocking findings
   - Every non-blocking finding is either addressed or documented as accepted risk with rationale
   - PLAN version is current

   If any blocking finding remains: **do not proceed**. Return to `/address-plan-review`.

2. **Update PLAN status to `approved`.** In PLAN frontmatter:
   - Set `status: approved`
   - Set `approved_date` to today's date
   - Set `approved_by: lead-developer`
   - Add changelog entry: `vN — approved after [N] review rounds`

3. **Update the registry.** In `sdlc/registry/index.md`, update the PLAN-NNNN row: `status: approved`, `approved_date`, and `milestone_count`.

## Quality Gate

Before committing:
- [ ] Latest Plan Review section shows zero blocking findings
- [ ] All non-blocking findings addressed or explicitly accepted as risk
- [ ] PLAN frontmatter: `status: approved`, `approved_date`, `approved_by` set
- [ ] Registry updated for PLAN-NNNN

## Handoff

**Pause for human review:**
> "Ready to commit the entire plan phase — PLAN-NNNN with all review rounds, address summaries, and approval. Please review. Confirm: (1) proceed with the commit? (2) solo project (no push) or team project (push too)?"

**Commit (after user confirms):**
```
git add sdlc/plans/PLAN-NNNN-slug.md sdlc/registry/index.md
git commit -m "PLAN-NNNN: approved — [N] review rounds, all findings resolved"
```

**Push (only if user confirms team context):**
```
git push origin thread/NNNN-slug
```

**Notify:**
- **Quality Engineer** → `/create-test-plan SPEC-NNNN PLAN-NNNN OBS-NNNN SEC-NNNN`

TEST-NNNN must be approved before Developer begins `/implement-plan`.
