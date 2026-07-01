---
id: approve-spec
type: command
name: /approve-spec
description: TPO command to close the spec phase — confirm review is ready, mark SPEC approved, commit all spec-phase artifacts
status: active
version: 1
primary_role: tpo
output_artifact_type: spec (approved)
---

# /approve-spec

## Purpose

Close the spec phase. Confirm the latest review shows `readiness_verdict: ready`, mark the SPEC `approved`, and commit all spec-phase artifacts in a single commit. Nothing in Stage 3 (Design) begins before `/approve-spec` runs.

## When to Invoke

- `/review-spec` outputs `readiness_verdict: ready`
- Do not invoke if any open question is unresolved or `readiness_verdict: needs_revision`

Pass SPEC and ANALYSIS IDs: `/approve-spec SPEC-NNNN ANALYSIS-NNNN`.

## Required Inputs

- **SPEC-NNNN** — latest version; all revision rounds complete
- **ANALYSIS-NNNN** — must show `readiness_verdict: ready`

## Steps

1. **Final readiness gate.** Read ANALYSIS-NNNN. Confirm:
   - `readiness_verdict: ready`
   - All open questions in SPEC-NNNN are resolved or explicitly deferred with documented rationale
   - SPEC version is current

   If `readiness_verdict: needs_revision`: **do not proceed.** Return to `/revise-spec SPEC-NNNN ANALYSIS-NNNN`.

2. **Mark SPEC approved.** In SPEC-NNNN frontmatter:
   - Set `status: approved`
   - Set `approved_date` to today's date
   - Add changelog entry: `vN — approved after [N] review rounds`

3. **Mark ANALYSIS completed.** In ANALYSIS-NNNN frontmatter:
   - Set `status: completed`
   - Confirm `readiness_verdict: ready` and `security_review_required` reflect the final round's verdict

4. **Update registry.** In `sdlc/registry/index.md`:
   - Add or update SPEC-NNNN entry: `status: approved`
   - Add or update ANALYSIS-NNNN entry: `status: completed`

## Quality Gate

Before committing:
- [ ] ANALYSIS shows `readiness_verdict: ready`
- [ ] SPEC `status: approved`, `approved_date` set
- [ ] ANALYSIS `status: completed`
- [ ] Registry updated for both artifacts
- [ ] No open question remains unresolved without documented rationale

**Self-check before proceeding:** Re-read each item above. Confirm each is complete, or state what was skipped and why before continuing to Handoff.

## Handoff

**Before committing — pause for human review:**

Show the user the approved SPEC and ANALYSIS. Then ask:

> **Before we commit — three questions:**
> 1. State the problem this SPEC solves in one sentence, without using any implementation terms. Does that sentence accurately describe the user need?
> 2. Is there a user scenario or edge case not covered by any AC that a developer might reasonably assume is in scope?
> 3. Read the Non-Goals. Is anything listed as out of scope that would make the feature incomplete or unusable without it?
>
> If any answer reveals a gap: return to `/revise-spec SPEC-NNNN ANALYSIS-NNNN`.
>
> If all three check out: "Ready to commit and move to design. Confirm: (1) proceed with the commit? (2) solo project (no push) or team project (push too)?"

**Commit (after user confirms):**
```
git add sdlc/specs/SPEC-NNNN-slug.md sdlc/analysis/ANALYSIS-NNNN-slug.md sdlc/registry/index.md
git commit -m "SPEC-NNNN: approved — [N review rounds]"
```

**Push (only if user confirms team context):**
```
git push -u origin thread/NNNN-slug
```

**Note `security_review_required` for the Architect:**
> State the value from ANALYSIS frontmatter. If `yes`: `/security-review` will be triggered after DESIGN is approved — the Architect should be aware before starting `/create-design`.

→ **Architect** → `/create-design SPEC-NNNN ANALYSIS-NNNN`
