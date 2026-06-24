---
id: create-bug
type: command
name: /create-bug
description: Quality Engineer command to file a defect report — capture reproduction steps, severity estimate, affected version, and linked artifacts; creates BUG-NNNN in reported status
status: active
version: 1
primary_role: quality-engineer
output_artifact_type: bug
---

# /create-bug

## Purpose

Capture a defect report as BUG-NNNN. Anyone can file a bug — a QE, developer, TPO, or end user report passed through the team. `/create-bug` is purely capture: it records the symptom, reproduction steps, severity estimate, and known context. It does not confirm whether the issue is actually a defect — that is the job of `/triage-bug`.

## Usage

```
/create-bug
/create-bug INC-NNNN     # when spawned from a production incident
```

---

## Step 1 — Gather defect information

Prompt the user for the following:

> **Title:** (one-line description of the defect — what is wrong, not what was expected)
>
> **Affected version:** (which release or branch exhibits this behavior — e.g., v1.0.0, main, feature/PLAN-NNNN-M2)
>
> **Environment:** (production / staging / local development / CI)
>
> **Severity estimate:** (Critical / High / Medium / Low — see table below; triage will confirm)
>
> **Reproduction steps:** (numbered steps from a clean state that reliably produce the defect)
>
> **Actual behavior:** (what happens)
>
> **Expected behavior:** (what should happen — cite the relevant AC from SPEC-NNNN if known)
>
> **Is this reproducible?** (always / intermittent / reproduced once)
>
> **Linked artifacts:** (SPEC-NNNN, PLAN-NNNN, or INC-NNNN if known — leave blank if unknown)

Severity estimate guidelines — the reporter's initial read; `/triage-bug` confirms:

| Severity | Criteria |
|---|---|
| **Critical** | Core functionality completely broken; data loss or corruption; security vulnerability; no workaround |
| **High** | Major feature broken for most users; workaround exists but is not acceptable for production |
| **Medium** | Feature partially broken; workaround available and acceptable short-term |
| **Low** | Minor issue; cosmetic defect; edge case with minimal user impact |

---

## Step 2 — Determine next BUG ID

Check `sdlc/bugs/` for existing BUG files and take the next sequential ID. If the directory does not exist, create it and start at BUG-0001.

---

## Step 3 — Create BUG-NNNN

Create `sdlc/bugs/BUG-NNNN.md`:

```markdown
---
id: BUG-NNNN
status: reported
severity: [Critical | High | Medium | Low]
priority: null
reported_at: [date]
reported_by: [name or role]
affected_version: [version or branch]
environment: [production | staging | local | ci]
reproducibility: [always | intermittent | once]
linked_incident: [INC-NNNN | null]
linked_spec: [SPEC-NNNN | null]
linked_plan: [PLAN-NNNN | null]
fix_plan: null
fix_milestone: null
rejection_reason: null
resolved_at: null
verified_at: null
---

# BUG-NNNN — [title]

## Summary

[One paragraph describing the defect — what is wrong, in what context, and what the impact is.]

## Reproduction Steps

**Environment:** [production | staging | local | ci]
**Affected version:** [version or branch]
**Reproducibility:** [always | intermittent | once]

1. [Step 1 — start from a clean state]
2. [Step 2]
3. [Step N]

**Actual behavior:** [what happens]

**Expected behavior:** [what should happen — cite SPEC-NNNN AC-N if applicable]

## Severity Estimate

**[Critical | High | Medium | Low]** — [one-line reason]

## Linked Artifacts

- Incident: [INC-NNNN | none]
- Spec: [SPEC-NNNN | unknown]
- Plan: [PLAN-NNNN | unknown]

## Triage

*Pending — see `/triage-bug BUG-NNNN`.*

## Fix

*Pending triage.*

## Verification

*Pending fix.*
```

---

## Quality Gate

- [ ] Reproduction steps are numbered, start from a clean state, and are specific enough for another engineer to follow
- [ ] Actual behavior and expected behavior are both present and distinct
- [ ] Severity estimate is provided with a one-line reason
- [ ] Reproducibility field is set
- [ ] All known linked artifacts recorded; unknown fields set to `null`

## Handoff

Do not commit. BUG-NNNN is not committed until `/triage-bug` accepts or rejects it.

> "BUG-NNNN filed. Run `/triage-bug BUG-NNNN` to confirm whether this is an accepted defect and assign it to a fix milestone."
