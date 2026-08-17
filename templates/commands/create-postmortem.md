---
id: create-postmortem
type: command
name: /create-postmortem
description: SRE command to author a postmortem for a resolved incident — root cause, contributing factors, and structured action items; offers to create BUG-NNNN inline for code-fix items
status: active
version: 1
primary_role: sre
links:
  - relation: follows
    target: resolve-incident
  - relation: produces
    target: postmortems/PM-NNNN.md
---

# /create-postmortem

## Purpose

Author PM-NNNN from a resolved incident. The SRE role leads. Guides through root cause analysis, contributing factors, and structured action items. For `code-fix` action items, offers to create BUG-NNNN inline so follow-up work is traceable from the start. Blameless throughout — name systems, decisions, and processes, not people.

## Usage

```
/create-postmortem INC-NNNN
```

## Pre-flight

Read `sdlc/incidents/INC-NNNN-[slug].md`.

- If `status` is not `resolved` → stop: "INC-NNNN is not yet resolved. Run `/resolve-incident INC-NNNN` first."
- If `linked_postmortem` is already set → stop: "A postmortem already exists for INC-NNNN: [PM-NNNN]. Edit that file directly."

## Step 1 — Determine next PM number

Scan `sdlc/postmortems/` for existing PM-NNNN files. Use the next available four-digit number.

## Step 2 — Root cause and contributing factors

Load the INC-NNNN timeline, impact, and resolution sections as context.

**Root cause — apply the 5 Whys:**

Start from the failure symptom in INC-NNNN's impact statement and ask "why?" repeatedly until reaching a systemic cause. Five is a guideline — stop earlier if a genuine root is reached, go deeper if the answer is still a symptom.

Walk the SRE through each step:

> **Why did [symptom from INC-NNNN impact] happen?** (Why 1)

> **Why did [Why 1 answer] happen?** (Why 2)

> **Why did [Why 2 answer] happen?** (Why 3)

> **Why did [Why 3 answer] happen?** (Why 4)

> **Why did [Why 4 answer] happen?** (Why 5)

> **Root cause statement:** What is the systemic issue at the bottom of this chain? (1–2 sentences)

Rules for the 5 Whys:
- "Human error" or "someone forgot" is never a final answer — always ask why the system put a person in a position to make that error
- Stop when the answer points to a process, system, or structural gap that can actually be changed
- If two Whys produce different branches, follow both — contributing factors, not a second root cause

> **Contributing factors?**
> What conditions allowed the root cause to have this impact? List each one. Examples: no alert existed for this error path; rollback procedure was undocumented; the change had no staging equivalent of production load; on-call rotation hadn't been updated after a team change.

## Step 3 — Response retrospective

> **What went well during the response?**
> (Examples: on-call was paged within 2 minutes; rollback was clean and took under 5 minutes; cross-team communication was fast)

> **What went poorly during the response?**
> (Examples: root cause took 45 minutes to identify because logs lacked the request ID; no runbook existed for this failure mode; severity was initially underestimated)

## Step 4 — Action items

For each action item, collect description, type, and owner. Ask for all items before offering to create artifacts.

> **Action item description?** (what needs to be done — be specific enough that someone can act on it)
>
> **Type?**
> - `code-fix` — a code defect that needs a proper fix through the pipeline
> - `feature` — new capability needed to prevent recurrence (e.g., circuit breaker, rate limiter, feature flag)
> - `observability` — missing alert, metric, or log signal
> - `infrastructure` — operational improvement (deployment tooling, rollback automation, scaling)
> - `process` — documentation, runbook, on-call process, or team convention change
>
> **Owner?** (role or name — leave blank if unassigned)

<!-- TODO(forgekit): `process` currently conflates two different scopes — a fix to *this project's* runbook/conventions vs. a gap in a *ForgeKit pipeline command or role file itself* (a review checklist that should have caught this class of issue, applicable to every project using ForgeKit, not just this one). If a `process` item's root cause traces back to a command/role definition, ask which command or role file it belongs in and what checklist addition would have caught it, distinct from ordinary project-level `process` items. Not implemented yet — revisit when a real postmortem actually surfaces this case. See ForgeKit session notes 2026-08-11. -->

Repeat until the SRE indicates all action items are captured.

## Step 5 — Create downstream artifacts for code-fix items

For each action item typed `code-fix`:

> "Action item [PM-NNNN-AI-N] is a code fix: '[description]'. Create BUG-NNNN now? (yes / no)"

If yes: invoke `/create-bug` inline with the action item description pre-filled as the bug title. Set `linked_postmortem: PM-NNNN` in the resulting bug's frontmatter. Record the assigned BUG-NNNN in the action item's `Linked` column.

If no: the item remains unlinked. It will appear as an open action item in `/sync-registry` output and in release plan risk checks until a BUG-NNNN is linked.

For `feature` items: note after postmortem creation — "Run `/create-spec` to start a feature thread for [AI-N]. Include `linked_postmortem: PM-NNNN` in the SPEC frontmatter." Do not invoke `/create-spec` inline.

`observability`, `infrastructure`, and `process` items stay in PM-NNNN. No downstream pipeline artifact is created — they are tracked and closed by the owner directly in this file.

## Step 6 — Write PM-NNNN

Write `sdlc/postmortems/PM-NNNN-[slug].md`:

```markdown
---
id: PM-NNNN
schema_version: 1
title: "[title from INC-NNNN]"
incident: INC-NNNN
severity: [severity from INC-NNNN]
date: [today]
status: complete
authors:
  - sre
generation:
  command: /create-postmortem
  command_version: "[use version from this command's frontmatter]"
  model: "[self-identify from system context]"
  generated_at: "[today's date]"
  session_id: null
---

# PM-NNNN — [title]

## Incident summary

**Declared:** [declared_at from INC-NNNN] | **Resolved:** [resolved_at] | **Duration:** [duration]
**Severity:** [SEV-N] | **Mitigation:** [resolution summary from INC-NNNN]

[2–3 sentence summary of what broke, how long it lasted, and the scale of impact]

## Timeline

[Full timeline table from INC-NNNN, expanded with any post-resolution detail]

| Time (UTC) | Event |
|---|---|
| HH:MM | [event] |

## Root cause

**5 Whys analysis:**

| # | Why? | Answer |
|---|---|---|
| 1 | Why did [symptom]? | [answer] |
| 2 | Why did [answer to 1]? | [answer] |
| 3 | Why did [answer to 2]? | [answer] |
| 4 | Why did [answer to 3]? | [answer] |
| 5 | Why did [answer to 4]? | [answer] |

**Root cause:** [The systemic issue at the bottom of the chain — the gap in process, system, or structure that made the incident possible]

## Contributing factors

[From Step 2 — each factor on its own line]

## What went well

[From Step 3]

## What went poorly

[From Step 3]

## Action items

| ID | Description | Type | Owner | Status | Linked |
|---|---|---|---|---|---|
| PM-NNNN-AI-1 | [description] | [type] | [owner or —] | open | [BUG-NNNN or —] |

## Lessons learned

[Key insights this incident produced for the team — the broader understanding, not a restatement of action items. What would someone new to the team need to know after reading this?]
```

## Step 7 — Link postmortem back to incident

Update `sdlc/incidents/INC-NNNN-[slug].md` frontmatter:

```yaml
linked_postmortem: PM-NNNN
```

Also update the `## Follow-up` section in INC-NNNN:

```markdown
- Linked postmortem: PM-NNNN
```

## Step 8 — Commit

```bash
git add sdlc/postmortems/PM-NNNN-[slug].md
git add sdlc/incidents/INC-NNNN-[slug].md
git add sdlc/bugs/        # only if BUG-NNNNs were created in Step 5
git commit -m "postmortem(PM-NNNN): [title] — N action item(s)"
```

## Step 9 — Summary

> **PM-NNNN complete.**
>
> Action items: N total
> - [X] code-fix — [N linked to BUG-NNNNs, M unlinked]
> - [X] feature — run `/create-spec` with `linked_postmortem: PM-NNNN`
> - [X] observability / infrastructure / process — tracked in PM-NNNN
>
> Open action items will appear in `/sync-registry` output and surface as named risks in release plans until resolved or linked.

## Quality bar (blameless postmortem)

- Root cause is derived via 5 Whys — not stated as a bare assertion without a chain of reasoning
- "Human error," "someone forgot," or "implementation mistake" is never a final Why — always one more: why did the system allow that error to have this impact?
- If the 5 Whys chain stops at a person, keep asking
- Root cause names a system, decision, or process failure — not a person
- Contributing factors are specific and actionable — not "human error"
- Every `code-fix` action item either has a BUG-NNNN linked or is explicitly left unlinked with a note why
- Action items are concrete: someone reading this 6 months from now can act on each item without asking for clarification
