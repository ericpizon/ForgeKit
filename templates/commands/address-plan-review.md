---
id: address-plan-review
type: command
name: /address-plan-review
description: Lead Developer command to address findings from PLAN review sections and prepare the PLAN for re-review or approval
status: active
version: 1
primary_role: lead-developer
output_artifact_type: plan (revised)
links:
  - relation: relates_to
    target: create-plan
  - relation: relates_to
    target: review-plan
  - relation: relates_to
    target: approve-plan
---

# /address-plan-review

## Purpose

Address findings from the Stage 5 PLAN review sections (SRE, Developer, Quality Engineer, Security, Architect). Work through every blocking finding, revise the PLAN accordingly, and trigger re-review. Non-blocking findings are addressed if straightforward or documented as accepted risk. The loop repeats until all required reviewers pass with no blocking findings, at which point `/approve-plan` is triggered.

## When to Invoke

After `/review-plan` appends a `## Plan Review — Round N` section with blocking findings.

Pass the PLAN ID: `/address-plan-review PLAN-0001`.

## Required Inputs

- **PLAN-NNNN** — read the latest `## Plan Review — Round N` section for all findings
- **DESIGN-NNNN** — keep in view; revisions must stay consistent with approved Key Decisions and module layout
- **SEC-NNNN** — (if triggered) keep in view; security task coverage must be maintained

## Steps

1. **Read `current_review_round` from PLAN-NNNN frontmatter.** Navigate directly to `## Plan Review — Round {current_review_round}` — do not scan for it. If the status shows zero blocking findings — there is nothing to address. **Exit here.** Trigger `/approve-plan PLAN-NNNN`. Do not commit anything.

   If `current_review_round` is 0 or no review section exists yet — the review has not been run. Run `/review-plan PLAN-NNNN SPEC-NNNN DESIGN-NNNN SEC-NNNN` first.

2. **List every finding by perspective** (SRE, Developer, QE, Security, Architect) and blocking status before making any changes. Do not start revising until the full picture is in view.

3. **Triage.** Separate into three buckets:
   - **Must resolve now:** blocking findings from any perspective
   - **Accept by default:** non-blocking findings are accepted as-is unless the user has explicitly requested addressing specific ones
   - **Accept or defer:** findings the Lead Developer accepts as risk (document rationale) or explicitly defers

4. **Address blocking findings in order.** For each blocking finding:
   - State the perspective, finding, and the task(s) it targets
   - Revise only the tasks the finding references — do not restructure the PLAN beyond what is needed
   - If a task needs to be split, split only that task; maintain all other task ordering
   - If the fix requires a new task (e.g., a missing security mitigation task), add it in the correct phase position
   - If a fix requires a design decision not yet in DESIGN-NNNN: stop — see Blocker Protocol

5. **Non-blocking findings:** Accept as-is by default — list them in the Not Addressed section as `accepted`. Only address a non-blocking finding if the user has explicitly requested it, or if it is a correction of a clear factual error or typo with no judgment involved.

   If a blocking finding required adding or splitting a task, update only the affected rows in the Happy Path Trace — do not regenerate the full trace. Existing rows for unchanged tasks are stable.

6. **Update the Milestones section if task revisions affected milestone membership.** If any task was split, added, moved, or removed during Steps 4–5, update the affected milestone's `Tasks:` list to reflect the new set. Verify that every task still belongs to exactly one milestone and that every milestone still maps to at least one AC. Update `milestone_count` in frontmatter if the number of milestones changed.

7. **Increment PLAN version in frontmatter.** Increment `version` by 1 and update `updated` to today's date.

8. **Append `## Address Summary — Round N` to PLAN-NNNN.** Use the same round number as the review section being addressed. Format:

   ```
   ## Address Summary — Round N

   **Date:** YYYY-MM-DD

   ### Addressed
   - [SRE-1] — [what was changed and where]
   - [QE-2] — [what was changed and where]

   ### Not Addressed
   - [DEV-1] — accepted risk: [rationale]
   - [ARCH-2] — deferred: [rationale and condition for revisiting]
   ```

   Every finding from the Plan Review must appear in exactly one of the two lists. Do not omit findings silently.

## Blocker Protocol

If a finding requires a task change that contradicts DESIGN-NNNN Key Decisions or module boundaries: stop. Surface to Architect — either the DESIGN must be updated first, or the finding must be documented as accepted drift with Architect approval.

If a finding requires adding a security task not backed by SEC-NNNN: stop. Surface to Security — the SEC finding may need to be opened or reopened before the PLAN task can be added.

If two perspective findings directly contradict each other — resolving one requires violating the other: stop. This should have been flagged in `/review-plan`'s Cross-Perspective Check. Arbitrate now: document both positions and the decision before revising.

## Quality Gate

Before showing the diff to the user:
- [ ] Every blocking finding from the latest Plan Review section is resolved or documented as accepted risk with rationale
- [ ] No revision introduced tasks outside the scope DESIGN-NNNN supports
- [ ] No security task quietly dropped or deferred without SEC-NNNN documentation
- [ ] Milestones section updated if any task was added, split, moved, or removed
- [ ] Every task still belongs to exactly one milestone; every milestone still maps to at least one AC
- [ ] `milestone_count` in frontmatter updated if milestone count changed
- [ ] PLAN `version` incremented, `updated` date current
- [ ] `## Address Summary — Round N` appended with every finding listed in Addressed or Not Addressed

**Self-check before proceeding:** Re-read each item above. Confirm each is complete, or state what was skipped and why before continuing to Handoff.

## Handoff

Show the user the PLAN diff and the Address Summary. Do not commit.

> "Here is the revised plan and the address summary for Round N. Once you confirm, run `/review-plan PLAN-NNNN SPEC-NNNN DESIGN-NNNN SEC-NNNN` for the next review round."

No commit happens here. The single commit for the entire plan phase is owned by `/approve-plan`.
