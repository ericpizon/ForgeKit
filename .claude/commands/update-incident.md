---
id: update-incident
type: command
name: /update-incident
description: Append a timestamped event to an active incident timeline; optionally update severity or status
status: active
version: 1
primary_role: sre
links:
  - relation: follows
    target: create-incident
  - relation: precedes
    target: resolve-incident
---

# /update-incident

## Purpose

Append a timestamped timeline entry to an active incident. Run this throughout the incident as new information arrives — every significant development, hypothesis tested, action taken, or status change should be logged. The running timeline is the primary source for the postmortem.

## Usage

```
/update-incident INC-NNNN
```

## Pre-flight

Read `sdlc/incidents/INC-NNNN-[slug].md`. Check `status`.

- `active` or `monitoring` → proceed
- `resolved` → stop: "INC-NNNN is already resolved. No further updates needed."

## Step 1 — What happened?

Ask:

> **What happened?** (be specific: what was observed, what action was taken, or what was discovered — this goes verbatim into the timeline)
>
> **Has severity changed?** (current: SEV-N — enter new severity or leave blank to keep)
>
> **Has status changed?** (current: [status] — enter `monitoring` if the incident is contained but not fully resolved, or leave blank to keep)

## Step 2 — Update INC-NNNN

1. Append a new row to the `## Timeline` table with the current UTC timestamp:

   ```markdown
   | HH:MM | [event description] |
   ```

2. If severity changed: update `severity` in frontmatter and include the change in the timeline event text (e.g., "Severity upgraded to SEV-1 — confirmed data loss").

3. If status changed to `monitoring`: update `status: monitoring` in frontmatter and note in the timeline.

## Step 3 — Confirm

Show the last three timeline entries and note:

> **INC-NNNN updated.**
>
> [Last 3 timeline entries]
>
> Continue updating: `/update-incident INC-NNNN`
> When mitigated: `/resolve-incident INC-NNNN`
