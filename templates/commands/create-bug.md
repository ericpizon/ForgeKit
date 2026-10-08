---
id: create-bug
type: command
name: /create-bug
description: Quality Engineer command to file a defect report — capture reproduction steps, severity estimate, affected version, and linked artifacts; creates BUG-NNNN in reported status
status: active
version: 3
primary_role: quality-engineer
output_artifact_type: bug
changelog:
  - v3 — declared the canonical `status` vocabulary once, in one place, since it was previously inconsistent across every command that touches it (`resolve-incident.md` wrote a value, `open`, that wasn't in anyone's enum; `verified`/`closed` were checked for but never written by anything). Built for the ForgeKit Operator feature (PLAN-0001 T15).
  - v2 — added an optional related_xdep field for a bug that's actually a cross-repo contract violation rather than a defect local to one repo.
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
>
> **Related cross-repo dependency, if applicable:** (THREAD-NNNN-XDEP-NN — only if this defect is actually a contract violation between this repo and another participant, not a defect local to this repo's own code; leave blank otherwise)

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
schema_version: 1
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
related_xdep: [THREAD-NNNN-XDEP-NN | null]
fix_plan: null
fix_milestone: null
rejection_reason: null
resolved_at: null
verified_at: null
generation:
  command: /create-bug
  command_version: "[use version from this command's frontmatter]"
  model: "[self-identify from system context]"
  generated_at: "[today's date]"
  session_id: null
---

## Canonical `status` vocabulary

Declared once here — every command that reads or writes BUG-NNNN's `status` field (`/triage-bug`, `/escalate-to-incident`, `/resolve-incident`, `/execute-release`) must use exactly these values, never a similarly-worded substitute:

```
reported → accepted | rejected | needs_info
accepted → pending_verification                  (fix deployed via the normal PLAN/release pipeline)
pending_verification → verified | reopened
reopened → accepted                               (loops back for a new fix attempt)
(any non-terminal status) → incident_active       (production impact warrants incident response)
incident_active → accepted | fix_deployed          (fix still needed, vs. the incident's own mitigation is permanent)
fix_deployed → verified                            (same confirmation step pending_verification leads to)
rejected → closed                                  (terminal, no fix — `rejection_reason` names why)
verified → closed                                  (terminal)
```

`rejected` and `closed` are terminal — no command transitions out of them. `verified` and `closed` currently have no command that writes them (no `/verify-bug-fix`-shaped command exists yet in this pipeline) — this is a known, explicitly-acknowledged gap, not silently hidden; `/triage-bug`, `/escalate-to-incident`, and `/resolve-incident` are reconciled against this list as written (see each command's own changelog), but creating the missing verification step is out of scope for this reconciliation.

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
- Cross-repo dependency: [THREAD-NNNN-XDEP-NN | none]

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
- [ ] If this defect is actually a cross-repo contract violation, `related_xdep` names the specific XDEP rather than being left blank
- [ ] `schema_version: 1` present in frontmatter
- [ ] `generation` block present: command, command_version, model self-identified, generated_at set, session_id: null

## Handoff

Do not commit. BUG-NNNN is not committed until `/triage-bug` accepts or rejects it.

> "BUG-NNNN filed. Run `/triage-bug BUG-NNNN` to confirm whether this is an accepted defect and assign it to a fix milestone."
