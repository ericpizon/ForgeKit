---
id: approve-design
type: command
name: /approve-design
description: Architect command to close the design review loop — confirm all findings resolved, mark DESIGN approved, update ARCH, trigger Stage 4
status: active
version: 1
primary_role: architect
output_artifact_type: design (approved)
---

# /approve-design

## Purpose

Close the design review loop. Confirm all review sections are present and all blocking findings are resolved or accepted as documented risk. Mark the DESIGN `approved`, update ARCH with any cross-cutting patterns introduced, then trigger Stage 4 (Observability Design + conditional Security Review).

This is the exit gate for Stage 3. Nothing in Stage 4 begins before `/approve-design` runs.

## When to Invoke

- `/review-design` outputs a clean `## Design Review — Round N` section (zero blocking findings)
- Do not invoke if any blocking finding remains unresolved

Pass the DESIGN ID: `/approve-design DESIGN-NNNN`.

## Before Starting — Sync Your Branch

**Team context:** Pull before reading — all reviewer sections must be on the branch.
```
git pull origin thread/NNNN-slug
```
**Solo context:** Skip — already local.

## Required Inputs

- **DESIGN-NNNN** — latest `## Design Review — Round N` section must show zero blocking findings
- **ANALYSIS-NNNN** — read `security_review_required` to determine whether Stage 4b is triggered
- **`sdlc/architecture/ARCH.md`** — will be updated if this DESIGN introduces cross-cutting patterns

## Steps

1. **Final review gate.** Read `current_review_round` from DESIGN-NNNN frontmatter. Navigate directly to `## Design Review — Round {current_review_round}` — do not scan for it. Confirm:
   - Blocking finding count is zero
   - DESIGN version is current

   If any blocking finding remains: **do not proceed**. Direct the user to run `/address-design-review DESIGN-NNNN` to resolve them first.

   Non-blocking findings do not block approval. Accept them as developer notes — do **not** address them inline here. If the user wants to address non-blocking findings, they run `/address-design-review DESIGN-NNNN` separately, before or after approval.

2. **Update DESIGN status to `approved`.** In DESIGN frontmatter:
   - Set `status: approved`
   - Set `approved_date` to today's date
   - Set `approved_by: architect`
   - Add changelog entry: `vN — approved after [N] review rounds`

3. **Update ARCH.** Apply any cross-cutting patterns introduced by this DESIGN that belong in the system-wide baseline:
   - New data access patterns (new table shapes, new query conventions)
   - New API conventions (new route patterns, new response shapes)
   - New auth patterns (new trust boundary decisions)
   - New module layout conventions
   
   In ARCH: increment version, add changelog entry referencing this DESIGN, update the relevant section. If a foundational section of ARCH changed — trigger ARCH review before proceeding. If only a new section was appended, no ARCH review needed.

## Quality Gate

Before committing:
- [ ] Latest Design Review section shows zero blocking findings
- [ ] Non-blocking findings accepted as-is (no inline resolution required)
- [ ] DESIGN frontmatter: `status: approved`, `approved_date`, `approved_by` set
- [ ] ARCH updated to reflect this DESIGN's cross-cutting contributions
- [ ] ARCH version incremented, changelog entry added

## Handoff

**Before committing — pause for human review:**
> Show the user the approved DESIGN and the ARCH diff. Ask: "Ready to mark DESIGN approved and trigger Stage 4. Please review. Confirm: (1) proceed with the commit? (2) solo project (no push) or team project (push too)?"

**Commit (after user confirms):**
```
git add sdlc/design/DESIGN-NNNN-slug.md sdlc/adr/ADR-NNNN-slug.md sdlc/architecture/ARCH.md sdlc/registry/index.md
git commit -m "DESIGN-NNNN: approved — [N review rounds]"
```

**Push (only if user confirms team context):**
```
git push origin thread/NNNN-slug
```

**Trigger Stage 4 (both run in parallel):**
- **SRE** → `/design-observability DESIGN-NNNN` (Stage 4a — always)
- Check ANALYSIS-NNNN `security_review_required`: if `yes` → **Security** → `/security-review SPEC-NNNN DESIGN-NNNN ANALYSIS-NNNN` (Stage 4b — conditional)

Both must complete before Lead Developer begins `/create-plan`.
