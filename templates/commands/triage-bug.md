---
id: triage-bug
type: command
name: /triage-bug
description: Lead Developer command to accept or reject a filed bug — confirm it is a real defect, classify severity and priority, assign to a fix milestone, or close with a documented rejection reason
status: active
version: 1
primary_role: lead-developer
---

# /triage-bug

## Purpose

Formally accept or reject BUG-NNNN. Before triage, a filed bug is an unverified claim. After triage, it is either an accepted defect with a confirmed severity, priority, and fix assignment — or a closed record with a documented rejection reason. The Lead Developer owns this decision.

## Usage

```
/triage-bug BUG-NNNN
```

---

## Step 1 — Read BUG-NNNN

Read `sdlc/bugs/BUG-NNNN.md` in full. Then read the linked artifacts:
- If `linked_spec` is set: read the relevant AC from SPEC-NNNN to confirm what behavior was specified
- If `linked_incident` is set: read INC-NNNN for additional context on blast radius and timeline

---

## Step 2 — Attempt reproduction

Present the reproduction steps to the user and ask them to attempt reproduction in the stated environment:

> **Reproduction check for BUG-NNNN:**
>
> Environment: [from BUG-NNNN]
> Affected version: [from BUG-NNNN]
>
> [Numbered reproduction steps from BUG-NNNN]
>
> Were you able to reproduce the defect? (yes — always / yes — intermittent / no)

If the user cannot reproduce: see **Reject — Cannot Reproduce** below.

---

## Step 3 — Accept or reject

Based on the reproduction result and the linked SPEC AC (if available), make the triage decision.

### Accept

The behavior is reproducible AND diverges from what the SPEC specifies (or from reasonable expected behavior if no SPEC AC covers it).

Prompt the user to confirm:
1. **Confirmed severity** — re-evaluate using the same table from `/create-bug`; override the reporter's estimate if warranted; state the reason
2. **Priority**:

| Priority | Meaning |
|---|---|
| **P0** | Fix before next release; blocks production |
| **P1** | Fix in the next planned release cycle |
| **P2** | Fix when capacity allows; does not block releases |
| **P3** | Nice to have; may be deferred indefinitely |

3. **Fix path**:
   - **New milestone on existing PLAN-NNNN** — the fix is small and belongs with the original feature thread
   - **New PLAN-NNNN** — the fix is substantial enough to warrant its own plan
   - **Fast-track** — Critical or P0 severity; use `/release-hotfix` pattern

Update BUG-NNNN frontmatter:
```yaml
status: accepted
priority: [P0 | P1 | P2 | P3]
severity: [confirmed severity]
fix_plan: [PLAN-NNNN — to be created | existing PLAN-NNNN]
fix_milestone: [MN — to be assigned | pending plan creation]
```

Append to the `## Triage` section:

```markdown
## Triage

**Decision:** Accepted
**Date:** [date]
**Triaged by:** [Lead Developer name]

**Confirmed severity:** [Critical | High | Medium | Low] — [reason]
**Priority:** [P0 | P1 | P2 | P3] — [reason]
**Fix path:** [new milestone on PLAN-NNNN | new PLAN-NNNN | fast-track hotfix]
**Root cause hypothesis:** [initial hypothesis — to be confirmed during implementation]
```

### Reject — Not a Bug

The behavior is reproducible but matches what the SPEC specifies, or is otherwise intentional.

```yaml
status: rejected
rejection_reason: not-a-bug
```

```markdown
## Triage

**Decision:** Rejected — Not a Bug
**Date:** [date]
**Triaged by:** [Lead Developer name]

**Reason:** The observed behavior matches [SPEC-NNNN AC-N | the intended design].
**Action:** [Close — no further action | Flag SPEC-NNNN AC-N for clarification]
```

### Reject — Duplicate

```yaml
status: rejected
rejection_reason: duplicate
```

```markdown
## Triage

**Decision:** Rejected — Duplicate
**Date:** [date]
**Triaged by:** [Lead Developer name]

**Duplicate of:** BUG-NNNN
**Action:** Track fix progress in BUG-NNNN.
```

### Reject — Cannot Reproduce

```yaml
status: needs_info
rejection_reason: cannot-reproduce
```

```markdown
## Triage

**Decision:** Needs Info — Cannot Reproduce
**Date:** [date]
**Triaged by:** [Lead Developer name]

**Attempted in:** [environment, version]
**Result:** Could not reproduce using the steps as written.
**Request:** [specific information needed]
**Auto-close:** If no response within 14 days, BUG-NNNN will be closed as cannot-reproduce.
```

### Reject — Won't Fix

```yaml
status: rejected
rejection_reason: wont-fix
```

```markdown
## Triage

**Decision:** Rejected — Won't Fix
**Date:** [date]
**Triaged by:** [Lead Developer name]

**Reason:** [specific rationale]
**Known issue:** [yes — will appear in release notes | no — too minor to surface]
```

### Reject — Out of Scope

```yaml
status: rejected
rejection_reason: out-of-scope
```

```markdown
## Triage

**Decision:** Rejected — Out of Scope
**Date:** [date]
**Triaged by:** [Lead Developer name]

**Reason:** [what is out of scope and why]
**Action:** [Close | Spawn SPEC-NNNN — this is a valid feature gap worth addressing]
```

---

## Quality Gate

- [ ] Reproduction attempted and result documented
- [ ] Decision is one of: accepted, not-a-bug, duplicate, needs-info, wont-fix, out-of-scope
- [ ] Accepted bugs have confirmed severity, priority, and fix path assigned
- [ ] Rejected bugs have a documented reason specific enough for the reporter to understand
- [ ] BUG-NNNN frontmatter `status` updated
- [ ] `## Triage` section appended to BUG-NNNN

---

## Handoff

Commit BUG-NNNN — the triage decision is the first stable state worth persisting:

```bash
git add sdlc/bugs/BUG-NNNN.md
git commit -m "bug(BUG-NNNN): triage — [accepted | rejected: reason] — [title]"
```

### Accepted — any priority

> "All bug fixes follow the full pipeline — plan → implement → review → test → release → verify. Priority determines urgency and PLAN size, not which steps run.
>
> - **P0/Critical:** run `/release-hotfix BUG-NNNN` — creates a minimal PLAN and TEST-NNNN, routes through the full pipeline, ships as a patch release.
> - **P1:** create a new PLAN-NNNN (one milestone) or add a milestone to the existing PLAN-NNNN.
> - **P2/P3:** add a task to the next planned PLAN-NNNN milestone; ships with the next feature release."

### Rejected (any reason)

> "BUG-NNNN closed. No further action required."

### Needs info

> "BUG-NNNN is on hold. Follow up with the reporter. Auto-close in 14 days if no response."
