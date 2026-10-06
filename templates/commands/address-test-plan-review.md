---
id: address-test-plan-review
type: command
name: /address-test-plan-review
description: Quality Engineer command to address findings from TEST-PLAN review sections and prepare for re-review or approval
status: active
version: 2
primary_role: quality-engineer
output_artifact_type: test-plan (revised)
changelog:
  - v2 — the zero-blocking exit path no longer bypasses disposition of a new non-blocking finding from a re-review round; it now appends a minimal Address Summary before handing off to /approve-test-plan, which already required every non-blocking finding to carry a documented disposition. Same gap class, and same fix, as address-plan-review.md v2 and address-design-review.md's non-blocking handling — found when TEST-0001's own Round 2 actually hit it.
links:
  - relation: relates_to
    target: create-test-plan
  - relation: relates_to
    target: review-test-plan
  - relation: relates_to
    target: approve-test-plan
---

# /address-test-plan-review

## Purpose

Address findings from the Stage 5b TEST-PLAN review sections (Security, SRE, Lead Developer). Work through every blocking finding, revise the TEST-PLAN accordingly, and trigger re-review. Non-blocking findings are addressed if straightforward or documented as accepted risk. The loop repeats until all blocking findings are resolved, at which point `/approve-test-plan` is triggered.

## When to Invoke

After `/review-test-plan` appends a `## Test Plan Review — Round N` section with blocking findings.

Pass the TEST-PLAN ID: `/address-test-plan-review TEST-0001`.

## Required Inputs

- **TEST-NNNN** — read the latest `## Test Plan Review — Round N` section for all findings
- **SEC-NNNN** — (if triggered) keep in view; security test coverage must be maintained
- **OBS-NNNN** — (if produced) keep in view; observability verification coverage must be maintained

## Steps

Adopt the Quality Engineer role. Apply the perspective and stack knowledge from [`.claude/roles/quality-engineer.md`](.claude/roles/quality-engineer.md) throughout these steps — particularly the test case format conventions and the quality bar for security and observability coverage.

1. **Read `current_review_round` from TEST-NNNN frontmatter.** Navigate directly to `## Test Plan Review — Round {current_review_round}` — do not scan for it. If the status shows zero blocking findings:
   - If every non-blocking finding in this round already has a disposition recorded in a prior `## Address Summary` (true on a first clean round with no findings at all) — there is nothing to address. **Exit here.** Trigger `/approve-test-plan TEST-NNNN`. Do not modify anything.
   - If this round contains any non-blocking finding with no recorded disposition yet (a re-review round can legitimately surface a new one even with zero blocking findings) — do not exit yet. Append a minimal `## Address Summary — Round N` containing only a `### Not Addressed` list, with each new non-blocking finding marked `accepted` (Step 5's default-acceptance rule applies here too — only mark otherwise if the user has explicitly requested addressing it). Then trigger `/approve-test-plan TEST-NNNN`. `/approve-test-plan`'s own gate requires every non-blocking finding to carry a documented disposition — this is what satisfies it without forcing a full address-and-re-review round-trip over a trivial finding.

   If `current_review_round` is 0 or no review section exists yet — run `/review-test-plan` first.

2. **List every finding by perspective** (Security if applicable, SRE, Lead Developer) and blocking status before making any changes.

3. **Triage.** Separate into three buckets:
   - **Must resolve now:** blocking findings from any perspective
   - **Accept by default:** non-blocking findings are accepted as-is unless the user has explicitly requested addressing specific ones
   - **Accept or defer:** findings the QE accepts as risk (document rationale) or explicitly defers

4. **Address blocking findings.** For each blocking finding:
   - State the perspective, finding ID, and the test case(s) it targets
   - Revise only the test cases the finding references
   - If a test case needs to be added, add it in the appropriate section
   - If fixing the finding requires a scope change beyond the SPEC or SEC, surface to the appropriate role (a missing AC goes back to TPO; a missing SEC finding goes back to Security), document the escalation in the Address Summary, and continue addressing the remaining findings. Do not block the address cycle on upstream resolution.

5. **Non-blocking findings:** Accept as-is by default — list them in the Not Addressed section as `accepted`. Only address a non-blocking finding if the user has explicitly requested it, or if it is a correction of a clear factual error or typo with no judgment involved.

6. **Increment TEST-NNNN version in frontmatter.** Increment `version` by 1 and update `updated` to today's date.

7. **Append `## Address Summary — Round N` to TEST-NNNN.** Use the same round number as the review section being addressed. Format:

   ```
   ## Address Summary — Round N

   **Date:** YYYY-MM-DD

   ### Addressed
   - [SEC-1] — [what was changed and where]
   - [SRE-2] — [what was changed and where]

   ### Not Addressed
   - [SRE-1] — accepted risk: [rationale]
   ```

   Every finding from the Test Plan Review must appear in exactly one of the two lists. Do not omit findings silently.

## Quality Gate

Before showing the diff to the user:
- [ ] Every blocking finding from the latest Test Plan Review section is resolved or documented as accepted risk with rationale
- [ ] No security test case quietly dropped or scope reduced to cover fewer failure paths without documented justification
- [ ] No observability verification step removed without documented justification
- [ ] TEST-NNNN `version` incremented, `updated` date current
- [ ] `## Address Summary — Round N` appended with every finding listed in Addressed or Not Addressed

**Self-check before proceeding:** Re-read each item above. Confirm each is complete, or state what was skipped and why before continuing to Handoff.

## Handoff

Show the user the TEST-NNNN diff and the Address Summary. Do not commit.

> "Here is the revised test plan and the address summary for Round N. Once you confirm, run `/review-test-plan TEST-NNNN SPEC-NNNN OBS-NNNN SEC-NNNN` for the next review round."

No commit happens here. The single commit for the entire test planning phase is owned by `/approve-test-plan`.
