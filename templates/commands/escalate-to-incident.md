---
id: escalate-to-incident
type: command
name: /escalate-to-incident
description: Convert an existing BUG-NNNN to an active incident when the defect is causing active production impact that warrants incident response
status: active
version: 2
primary_role: sre
links:
  - relation: follows
    target: create-bug
  - relation: produces
    target: incidents/INC-NNNN.md
  - relation: precedes
    target: resolve-incident
changelog:
  - v2 — reconciled against create-bug.md's newly canonical status vocabulary (no drift found; incident_active, and the fix_deployed/closed pre-flight check, already matched). Built for the ForgeKit Operator feature (PLAN-0001 T16).
---

# /escalate-to-incident

## Purpose

Escalate an existing bug to an active incident when production impact is confirmed and the normal development pipeline is too slow to respond. Creates INC-NNNN linked to BUG-NNNN and marks the bug `incident_active`.

Use when a bug filed through the normal process turns out to be actively breaking production. If there is no existing BUG-NNNN, use `/create-incident` directly.

## Usage

```
/escalate-to-incident BUG-NNNN
```

## Pre-flight

Read `sdlc/bugs/BUG-NNNN-[slug].md`.

- If `linked_incident` is already set → stop: "BUG-NNNN is already linked to [INC-NNNN]. If a new incident has started, file it separately with `/create-incident`."
- If `status` is `fix_deployed` or `closed` → stop: "BUG-NNNN is already resolved. If a regression occurred, file a new bug with `/create-bug` or declare a new incident with `/create-incident`."

## Step 1 — Confirm escalation details

Display the bug summary:

```
Bug:     BUG-NNNN — [title]
Status:  [status]
Severity: [severity]
```

Ask:

> **What is the current production impact?** (1–3 sentences — what is broken now, who is affected, and at what scale. This becomes the INC-NNNN impact statement.)
>
> **Incident severity?** (assess based on current production impact, not the bug's original severity)
> - **SEV-1** — Full outage, data loss, or active security breach
> - **SEV-2** — Major feature outage or significant degradation, no workaround available
> - **SEV-3** — Partial degradation, workaround available
> - **SEV-4** — Minor impact, most users unaffected

## Step 2 — Create INC-NNNN

Scan `sdlc/incidents/` for the next available INC number.

Write `sdlc/incidents/INC-NNNN-[slug].md`:

```markdown
---
id: INC-NNNN
title: "[title from BUG-NNNN]"
severity: SEV-N
status: active
declared_at: "YYYY-MM-DD HH:MM UTC"
resolved_at: ~
declared_by: sre
responders: []
linked_bug: BUG-NNNN
linked_postmortem: ~
---

# INC-NNNN — [title]

## Impact

[Production impact statement from Step 1]

*Escalated from BUG-NNNN.*

## Timeline

| Time (UTC) | Event |
|---|---|
| HH:MM | Incident declared — escalated from BUG-NNNN |

## Resolution

*(Populated by `/resolve-incident`)*

## Follow-up

- Linked bug: BUG-NNNN
- Linked postmortem: —
```

## Step 3 — Update BUG-NNNN

Update `sdlc/bugs/BUG-NNNN-[slug].md` frontmatter (`incident_active` is create-bug.md's canonical vocabulary; the pre-flight check above against `fix_deployed`/`closed` is also consistent with it — reconciled, no drift found):

```yaml
status: incident_active
linked_incident: INC-NNNN
```

## Step 4 — Confirm and next steps

> **BUG-NNNN escalated to INC-NNNN (SEV-N).**
>
> BUG-NNNN status → `incident_active`
> INC-NNNN created: `sdlc/incidents/INC-NNNN-[slug].md`
>
> **During incident response:**
> - `/update-incident INC-NNNN` — log timeline events as the situation develops
> - Hotfixes or rollbacks can be applied outside the normal pipeline — document each action in the timeline
>
> **When resolved:**
> - `/resolve-incident INC-NNNN` will determine next steps:
>   - If the hotfix permanently resolves the defect → BUG-NNNN marked `fix_deployed`
>   - If the defect still needs a proper fix → BUG-NNNN returned to `open` and goes through the pipeline

Do not commit until after resolving — commit INC-NNNN and the updated BUG-NNNN together via `/resolve-incident`.
