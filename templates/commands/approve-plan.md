---
id: approve-plan
type: command
name: /approve-plan
description: Lead Developer command to close the plan review loop — confirm all findings resolved, mark PLAN approved, trigger TEST-PLAN authorship
status: active
version: 2
primary_role: lead-developer
output_artifact_type: plan (approved)
changelog:
  - v2 — added a staleness re-check before final approval for PLANs with an inputs.cross_repo_contract block.
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

   **Staleness re-check** *(only if PLAN-NNNN has an `inputs.cross_repo_contract` block)*: re-resolve the coordination repo's current `approved` revision one more time (per [`templates/reference/cross-repo-resolution.md`](../reference/cross-repo-resolution.md)) — a clean review round does not guarantee the Cross-Repo Contract hasn't advanced in the time since. If it has, this PLAN is `stale`: **do not approve**. Direct the user back to `/create-plan` (or a fresh `/review-plan` pass, if the divergence is scoped enough to address without a full re-derivation) before proceeding.

2. **Update PLAN status to `approved`.** In PLAN frontmatter:
   - Set `status: approved`
   - Add `approval:` block:
     ```yaml
     approval:
       mode: repo_only
       approved_by: "[Lead Developer name]"
       approved_role: lead-developer
       approved_at: "[today's date]"
       decision_summary: "[one sentence confirming PLAN-NNNN approved for test planning phase]"
       external_event_id: null
       external_event_uri: null
     policy_evaluation:
       evaluated: null
       policy_pack: null
       result: null
     ```
   - Add changelog entry: `vN — approved after [N] review rounds`
   - Set `pipeline_status.plan: approved`

3. **Update the registry.** In `sdlc/registry/index.md`, update the PLAN-NNNN row: `status: approved` and `milestone_count`.

## Quality Gate

Before committing:
- [ ] Latest Plan Review section shows zero blocking findings
- [ ] All non-blocking findings addressed or explicitly accepted as risk
- [ ] PLAN frontmatter: `status: approved`; `approval:` block set (`approved_by`, `approved_role: lead-developer`, `approved_at`, `decision_summary` populated)
- [ ] Registry updated for PLAN-NNNN
- [ ] *(thread participants only)* Staleness re-check passed — the Cross-Repo Contract's approved revision still matches `inputs.cross_repo_contract.revision`

**Self-check before proceeding:** Re-read each item above. Confirm each is complete, or state what was skipped and why before continuing to Handoff.

## Handoff

**Pause for human review:**

Ask:

> **Before we commit — three questions:**
> 1. If the delivery timeline compressed by 30%, which milestone would you cut or defer? Does this reveal anything about whether the milestone structure is right?
> 2. Walk the Happy Path Trace for the primary User Story. Is every step covered by a named task, or is there a step with no task assigned?
> 3. Is there an AC in the Definition of Done that doesn't feel fully covered by the task set?
>
> If any answer reveals a gap: return to `/address-plan-review PLAN-NNNN`.
>
> If all three check out: "Ready to commit the entire plan phase — PLAN-NNNN with all review rounds, address summaries, and approval. Confirm: (1) proceed with the commit? (2) solo project (no push) or team project (push too)?"

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
