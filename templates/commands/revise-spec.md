---
id: revise-spec
type: command
name: /revise-spec
description: TPO command to revise a SPEC based on ANALYSIS findings when readiness_verdict is needs_revision
status: active
version: 1
primary_role: tpo
output_artifact_type: spec
links:
  - relation: relates_to
    target: create-spec
  - relation: relates_to
    target: review-spec
---

# /revise-spec

## Purpose

Revise SPEC-NNNN in response to an ANALYSIS that returned `needs_revision`. The TPO addresses each flagged gap — ambiguous ACs, missing Non-Goals, contradictory scope — without introducing new scope. Output is an incremented SPEC version that unblocks a clean second pass through `/review-spec`.

## When to Invoke

- ANALYSIS-NNNN `readiness_verdict: needs_revision`

Do not invoke to make cosmetic edits or respond to Design-level questions — those are resolved in DESIGN, not SPEC.

Pass both artifact IDs: `/revise-spec SPEC-0001 ANALYSIS-0001`.

## Required Inputs

- **SPEC-NNNN** — the current version being revised
- **ANALYSIS-NNNN** — read the Suggested Spec Revisions section and every flagged item in Completeness Assessment and Ambiguities; these are the only things to address

## Steps

1. **List every flagged item** from ANALYSIS — completeness gaps, ambiguous ACs, Non-Goal contradictions, missing constraints. Do not start writing until the full list is in view.
2. **Address each item** — revise the specific section. Do not rewrite sections the ANALYSIS did not flag.
3. **Check bidirectional traceability** after each AC change — every Goal maps to at least one AC; every AC maps to at least one Goal.
4. **Verify Non-Goals** are still specific enough to use as a decision rule. A Non-Goal that says "we won't build X" must still say "X" precisely enough that a developer can apply it mid-implementation.
5. **Increment `version`** in frontmatter and update `updated` date.
6. **Self-check** — re-read the ANALYSIS flagged items against the revised SPEC. Every item must be either resolved or explicitly noted as a remaining open question with a rationale for deferral.
7. **Append a revision summary to SPEC-NNNN.** After all revisions, append a `## Revision Summary — Round N` section (matching the round number of the ANALYSIS just addressed):

   ```
   ## Revision Summary — Round N

   **Date:** YYYY-MM-DD

   ### Addressed
   - [Section: brief description of finding] — [what was changed in SPEC to resolve this]

   ### Not Addressed
   - [Section: brief description of finding] — **accepted risk** — [rationale]
   - [Section: brief description of finding] — **deferred to design** — [this is a Design-level resolution, not a Spec change]
   - [Section: brief description of finding] — **deferred** — [reason and conditions for revisit]
   ```

   Reference findings by their ANALYSIS section and a brief description (e.g., `Completeness Assessment: Goal G3 has no AC`; `Ambiguities: AC7 can be interpreted two ways`). ANALYSIS findings have no numeric IDs — do not invent them.

   Every flagged item from the ANALYSIS must appear in one of the two lists. No finding may be silently dropped.

## Quality Gate

Before handing off:
- [ ] Every ANALYSIS flagged item is addressed or documented in the revision summary with rationale
- [ ] No new scope was introduced during revision
- [ ] Bidirectional Goal ↔ AC traceability holds after changes
- [ ] Non-Goals are still specific enough to use as decision rules
- [ ] If NFRs were added or modified: each is specific enough to constrain DESIGN or names the stage where the threshold will be resolved
- [ ] If Observability Considerations were added or modified: all three concerns present (metrics, traces, logs) including what must never be logged
- [ ] `version` incremented, `updated` date current
- [ ] `## Revision Summary — Round N` appended to SPEC-NNNN — every flagged item accounted for

## Handoff

**Show the revised spec to the user:**
> Display the SPEC diff. Ask: "Please review the revision. Confirm when ready to proceed."

**Once user confirms — run the next review round:**
→ `/review-spec SPEC-NNNN`

Nothing is committed yet. The cycle repeats until `/review-spec` returns `ready`, at which point `/approve-spec` handles the single commit for the entire spec phase.
