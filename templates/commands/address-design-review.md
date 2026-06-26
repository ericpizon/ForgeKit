---
id: address-design-review
type: command
name: /address-design-review
description: Architect command to address findings from DESIGN review sections and prepare the DESIGN for re-review or approval
status: active
version: 1
primary_role: architect
output_artifact_type: design (revised)
links:
  - relation: relates_to
    target: create-design
  - relation: relates_to
    target: review-design
  - relation: relates_to
    target: approve-design
---

# /address-design-review

## Purpose

Address findings from the Stage 3 design review. Work through every blocking finding, revise the DESIGN accordingly, and document what was and was not addressed. The artifact is updated in place — no commit. The updated DESIGN feeds the next round of `/review-design`. The loop repeats until the review is clean, at which point `/approve-design` is triggered.

## When to Invoke

After `/review-design` outputs a `## Design Review — Round N` section with blocking findings.

Pass the DESIGN ID: `/address-design-review DESIGN-0001`.

## Required Inputs

- **DESIGN-NNNN** — read the latest `## Design Review — Round N` section for all findings
- **SPEC-NNNN** — keep in view; revisions must not violate scope or Non-Goals
- **ARCH** (`sdlc/architecture/ARCH.md`) — revisions must stay consistent with established patterns

## Steps

1. **Read the latest `## Design Review — Round N` section.** If the status shows zero blocking findings — there is nothing to address. **Exit here.** Trigger `/approve-design DESIGN-NNNN`. Do not commit anything.

   If no review section exists yet — the review has not been run. Run `/review-design SPEC-NNNN DESIGN-NNNN` first.

2. **List every finding by perspective** (DBA, Security, SRE, Lead Developer) and blocking status before making any changes.

3. **Triage.** Separate into three buckets:
   - **Must resolve now:** blocking findings from any perspective
   - **Accept by default:** non-blocking findings are accepted as-is unless the user has explicitly requested addressing specific ones
   - **Accept or defer:** findings the Architect accepts as risk (document rationale) or defers to a future DESIGN

4. **Address blocking findings in order.** For each blocking finding:
   - State the perspective, finding ID, and the DESIGN section it targets
   - Revise that specific section — do not rewrite sections the finding does not reference
   - If a revision creates a contradiction in another section, resolve both before moving on
   - If a fix requires a new ADR, author it now

5. **Non-blocking findings:** Accept as-is by default — list them in the Not Addressed section as `accepted`. Only address a non-blocking finding if the user has explicitly requested it, or if it is a correction of a clear factual error or typo with no judgment involved.

6. **Append an address summary to the DESIGN artifact.** After all revisions, append an `## Address Summary — Round N` section (matching the round number of the review just addressed):

   ```
   ## Address Summary — Round N

   **Date:** YYYY-MM-DD

   ### Addressed
   - [DBA-1] — [what was changed in DESIGN to resolve this]
   - [SEC-1] — [what was changed]

   ### Not Addressed
   - [SRE-1] — **accepted risk** — [rationale]
   - [LD-2] — **deferred** — [reason and conditions for revisit]
   - [DBA-2] — **escalated** — [SPEC or ARCH conflict blocking resolution]
   ```

   Every finding from the review must appear in one of the two lists. No finding may be silently dropped.

## Blocker Protocol

If addressing a blocking finding requires a scope change (adds a Goal, removes a Non-Goal, contradicts an AC): stop. Surface to TPO and Lead Developer — the SPEC may need revision before the DESIGN can proceed.

If a blocking finding cannot be resolved without contradicting ARCH or an existing ADR: stop. The Architect must propose a new ADR or request an ARCH update before revising the DESIGN.

If two findings from different perspectives contradict each other — resolving one requires violating the other: stop. The contradiction should have been flagged in `/review-design`'s Cross-Perspective Check. Arbitrate now: document both positions and the decision before revising.

## Quality Gate

Before showing the user the results:
- [ ] Every blocking finding is either resolved in the DESIGN or documented in the address summary with rationale
- [ ] No revision introduced scope that SPEC does not support
- [ ] No revision contradicts ARCH without a corresponding ADR
- [ ] DESIGN version incremented, `updated` date current
- [ ] `## Address Summary — Round N` section appended — every finding accounted for

## Handoff

**Self-check before showing the user:**
Re-read the DESIGN with fresh eyes. Does the revised DESIGN still make sense as a whole? Did resolving one finding create an inconsistency another perspective would catch?

**Show the user the results:**
> Display the DESIGN diff and the `## Address Summary — Round N` section. Show what was addressed, what was not, and why.

**Run the next review round:**
→ Tell the user: "Run `/review-design SPEC-NNNN DESIGN-NNNN` for the next review round."

`/review-design` reads the updated DESIGN and the address summary, runs all four perspectives against the revision, and produces a new `## Design Review — Round N+1` section. The cycle repeats until the review is clean, at which point `/approve-design` handles the single commit for the entire stage.
