---
id: resolve-incident
type: command
name: /resolve-incident
description: Mark an incident resolved, record the mitigation, determine postmortem requirement from severity threshold, and prompt for follow-up actions
status: active
version: 3
primary_role: sre
changelog:
  - v3 — Step 4's "permanent fix still needed" path now reverts BUG-NNNN to `accepted`, not `open` — `open` was never part of the canonical BUG status vocabulary `create-bug.md` declares, and nothing else in the pipeline ever wrote or read it. Built for the ForgeKit Operator feature (PLAN-0001 T16).
  - v2 — Handoff now asks solo-vs-team before pushing, like every other commit-producing command in the pipeline. Found during a full-pipeline drift audit: the entire Bug/Incident/Postmortem family committed locally but never pushed regardless of team context.
links:
  - relation: follows
    target: update-incident
  - relation: precedes
    target: create-postmortem
---

# /resolve-incident

## Purpose

Close an active incident. Records the mitigation, calculates duration, determines whether a postmortem is required based on the severity threshold in `project.yml`, and prompts for follow-up actions — postmortem authoring and permanent bug fix if needed.

## Usage

```
/resolve-incident INC-NNNN
```

## Pre-flight

Read `sdlc/incidents/INC-NNNN-[slug].md`. Check `status`.

- `active` or `monitoring` → proceed
- `resolved` → stop: "INC-NNNN is already resolved."

## Step 1 — Resolution details

> **How was the incident mitigated?** (e.g., rolled back to v1.2.3, applied hotfix commit abc1234, reverted config change, restarted service — be specific so the postmortem timeline is complete)
>
> **Is a permanent code fix still needed?** (yes — the mitigation is a workaround and the underlying defect remains; no — the fix is permanent)

## Step 2 — Update INC-NNNN

1. Set `status: resolved` in frontmatter
2. Set `resolved_at: [current UTC timestamp]` in frontmatter
3. Append final timeline entry:

   ```markdown
   | HH:MM | Incident resolved — [brief mitigation summary] |
   ```

4. Populate the `## Resolution` section:

   ```markdown
   ## Resolution

   **Resolved at:** [timestamp]
   **Duration:** [calculated: resolved_at − declared_at, rounded to nearest minute]
   **Mitigation:** [resolution description from Step 1]
   **Permanent fix needed:** [yes / no]
   ```

## Step 3 — Determine postmortem requirement

Read `pipeline.postmortem` from `project.yml`:

| Value | Meaning |
|---|---|
| `always` or `true` | Postmortem required for all incidents |
| `SEV-1` | Required only for SEV-1 |
| `SEV-2` | Required for SEV-1 and SEV-2 |
| `SEV-3` | Required for SEV-1, SEV-2, and SEV-3 |
| `SEV-4` | Required for all incidents (same as `always`) |
| `never` or `false` | Postmortem always optional |

Determine outcome: **required** / **optional** / **not needed**

## Step 4 — Handle BUG-NNNN if this incident was escalated from one

Check INC-NNNN frontmatter for `linked_bug`. If set:

- If permanent fix still needed: update `sdlc/bugs/BUG-NNNN-[slug].md` status back to `accepted` — the defect still needs to go through the pipeline, the same status it already carried before escalation (not `open`, which isn't part of the canonical vocabulary `create-bug.md` declares — fixed here, since nothing else in the pipeline ever wrote or read `open`).
- If mitigation is permanent (no further fix needed): update `sdlc/bugs/BUG-NNNN-[slug].md` status to `fix_deployed`.

## Step 5 — Commit

```bash
git add sdlc/incidents/INC-NNNN-[slug].md
git add sdlc/bugs/BUG-NNNN-[slug].md   # only if linked_bug is set (Step 4)
git commit -m "incident(INC-NNNN): resolved — [title] ([duration])"
```

**Ask before pushing:**
> "Solo project (commit only, no push) or team project (push too)?"

```bash
# team only:
git push origin main
```

## Step 6 — Next steps

> **INC-NNNN resolved.** Duration: [duration]
>
> [If postmortem required:]
> **Postmortem required** (SEV-N meets the threshold configured in project.yml).
> Run: `/create-postmortem INC-NNNN`
>
> [If postmortem optional:]
> **Postmortem optional** at this severity. Run `/create-postmortem INC-NNNN` if warranted.
>
> [If permanent fix needed and no linked_bug:]
> **Permanent fix needed.** File the underlying defect:
> Run `/create-bug` — set `linked_incident: INC-NNNN` in the bug frontmatter so the connection is traceable.
>
> [If linked_bug was set and fix is permanent:]
> BUG-NNNN marked `fix_deployed` — the hotfix permanently resolved the defect.
>
> [If linked_bug was set and fix still needed:]
> BUG-NNNN returned to `open` — the permanent fix still needs to go through the pipeline.
