---
id: create-incident
type: command
name: /create-incident
description: Declare a new production incident — three questions, then the INC-NNNN record is live; optimized for speed during active response
status: active
version: 1
primary_role: sre
links:
  - relation: produces
    target: incidents/INC-NNNN.md
  - relation: precedes
    target: update-incident
  - relation: precedes
    target: resolve-incident
---

# /create-incident

## Purpose

Declare a new production incident and open the INC-NNNN record. Three questions, then the record is live. Add detail throughout the incident with `/update-incident`. Resolve with `/resolve-incident`.

## Incident criteria

Declare an incident — not a bug — when at least one of the following is true **and** the normal development pipeline is too slow to respond:

- Service is unavailable or error-rate is elevated for users (not just in testing or staging)
- A data integrity issue is affecting live data
- A security breach or unauthorized access is active or suspected
- An SLO violation is in progress (latency or availability threshold breached)
- A bug that appeared benign in staging is critical at production scale

If the issue has no active production impact, file a BUG-NNNN instead — `/create-bug`.

## Usage

```
/create-incident
```

## Step 1 — Determine next INC number

Scan `sdlc/incidents/` for existing INC-NNNN files. Use the next available four-digit number.

## Step 2 — Three questions

Ask only these — do not ask for more upfront:

> **Severity?**
> - **SEV-1** — Full outage, data loss, or active security breach
> - **SEV-2** — Major feature outage or significant degradation, no workaround available
> - **SEV-3** — Partial degradation, workaround available
> - **SEV-4** — Minor impact, most users unaffected
>
> **Short title?** (≤60 characters — becomes the incident name and filename slug)
>
> **What is currently broken and who is affected?** (1–3 sentences — the initial impact statement)

## Step 3 — Create INC-NNNN

Write `sdlc/incidents/INC-NNNN-[slug].md`:

```markdown
---
id: INC-NNNN
schema_version: 1
title: "[title]"
severity: SEV-N
status: active
declared_at: "YYYY-MM-DD HH:MM UTC"
resolved_at: ~
declared_by: sre
responders: []
linked_bug: ~
linked_postmortem: ~
generation:
  command: /create-incident
  command_version: "[use version from this command's frontmatter]"
  model: "[self-identify from system context]"
  generated_at: "[today's date]"
  session_id: null
---

# INC-NNNN — [title]

## Impact

[Initial impact statement from Step 2]

## Timeline

| Time (UTC) | Event |
|---|---|
| HH:MM | Incident declared — [title] |

## Resolution

*(Populated by `/resolve-incident`)*

## Follow-up

- Linked bug: —
- Linked postmortem: —
```

## Step 4 — Confirm and next steps

> **INC-NNNN declared (SEV-N).**
>
> Record: `sdlc/incidents/INC-NNNN-[slug].md`
>
> **During the incident:**
> - `/update-incident INC-NNNN` — append timeline events as the situation develops
> - Severity can be updated at any time with `/update-incident`
>
> **When resolved:**
> - `/resolve-incident INC-NNNN` — records the mitigation and determines next steps

Do not commit the INC file during an active incident — commit after resolution via `/resolve-incident`.
