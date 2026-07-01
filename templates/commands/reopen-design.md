---
id: reopen-design
type: command
name: /reopen-design
description: Architect command to reopen the design phase when a discovery invalidates the approved DESIGN — captures discovery record, runs impact analysis, updates pipeline_status
status: active
version: 1
primary_role: architect
---

# /reopen-design

## Purpose

Re-enter the design phase after approval when a discovery — a new technical constraint, an unforeseen integration problem, a production finding — invalidates the current DESIGN. Records what was discovered, surfaces blast radius before making changes, and resets downstream pipeline status so PLAN and TEST are not used against stale design.

## Usage

```
/reopen-design DESIGN-NNNN
```

## Role Gate

This command is restricted to the Architect. Confirm before proceeding:

> "Are you acting as the Architect for this thread? Confirm to proceed."

Do not proceed without explicit confirmation.

## Steps

### Step 1 — Read current state

Read DESIGN-NNNN frontmatter. Note current `status`.

Check the registry for downstream artifacts on this thread:
- Does PLAN-NNNN exist? Read its `pipeline_status` and milestone statuses.
- Does TEST-NNNN exist? What is its status?
- Are any milestones `in_progress`?
- Are there any open PRs in the registry?

### Step 2 — Discovery intake

Ask the Architect for four pieces of information:

> **To open a back-cycle, provide:**
>
> 1. **Discovered by:** Your name and role.
> 2. **What was discovered:** What technical constraint, integration problem, or finding was uncovered? (1–3 sentences)
> 3. **Why this invalidates the DESIGN:** Which sections are affected — API shapes, data model, module layout, Key Decisions, ADRs? Why do these need to change?
> 4. **Blast radius:** Which downstream artifacts (OBS, SEC, PLAN, TEST, in-progress milestones) are likely affected and why?

Wait for all four answers before proceeding.

### Step 3 — Impact analysis (read-only)

Emit the impact analysis:

```
## Impact Analysis — DESIGN-NNNN Back-cycle

DESIGN-NNNN: approved → revision_requested

Downstream artifacts:
  OBS-NNNN ([status]): [affected if observability design changes] | [not affected]
  SEC-NNNN ([status]): [affected if security surface changes] | [not affected]
  PLAN-NNNN ([status]): pipeline_status.design will be updated to revision_requested
  TEST-NNNN ([status]): [affected if task scope changes] | [not affected]
  MN in_progress: [risk noted — discovery may affect ongoing implementation]
  Open PRs: [warn if any found — must be reviewed after revision]

Note: the TPO should be informed that a design back-cycle is in progress.
```

Then ask:

> "This back-cycle will set DESIGN-NNNN to `revision_requested` and update PLAN's pipeline_status. Confirm to proceed."

Do not make any file changes until confirmed.

### Step 4 — Apply changes

After confirmation:

**DESIGN-NNNN frontmatter:**
- Set `status: revision_requested`
- Increment `version`, update `updated` date
- Add changelog entry: `vN — revision_requested [date] — discovery: [one-line summary]`

**DESIGN-NNNN body:**
- Append `## Discovery — [date]` section (see Discovery Record format below)

**PLAN-NNNN frontmatter (if PLAN exists):**
- Update `pipeline_status.design: revision_requested`

### Step 5 — Discovery Record format

Append to DESIGN-NNNN:

```markdown
## Discovery — YYYY-MM-DD

**Discovered by:** [name] — [role]
**Stage triggered:** design

**What was discovered:**
[Architect's answer from Step 2]

**Why this invalidates the DESIGN:**
[Architect's answer from Step 2 — specific sections named]

**Blast radius:**
- OBS-NNNN: [affected / not affected — reason]
- SEC-NNNN: [affected / not affected]
- PLAN-NNNN: [affected / not affected]
- TEST-NNNN: [affected / not affected]
- MN in_progress: [risk noted / not affected — can continue]

**Status change:** DESIGN-NNNN `approved` → `revision_requested`
```

## Quality Gate

Before closing:
- [ ] Discovery intake complete — all four fields answered
- [ ] Impact analysis shown to Architect and confirmed before any file changes
- [ ] DESIGN-NNNN status set to `revision_requested`
- [ ] `## Discovery — [date]` section appended to DESIGN-NNNN
- [ ] PLAN-NNNN `pipeline_status.design` updated to `revision_requested` (if PLAN exists)
- [ ] DESIGN-NNNN `version` incremented, `updated` date current, changelog entry added

**Self-check before proceeding:** Re-read each item above. Confirm each is complete, or state what was skipped and why before continuing to Handoff.

## Handoff

> **DESIGN-NNNN is now `revision_requested`.**
>
> **Re-run sequence:**
> 1. Edit DESIGN-NNNN to address the discovery — revise the affected sections (API shapes, data model, Key Decisions, ADRs as needed). Increment version and update `updated` date.
> 2. Run `/review-design SPEC-NNNN DESIGN-NNNN` — all four perspectives assess the revision.
> 3. Once clean: run `/approve-design DESIGN-NNNN`.
> 4. Assess PLAN impact: run `/review-plan PLAN-NNNN SPEC-NNNN DESIGN-NNNN` to determine whether tasks or milestones need revision. If yes: begin the address-plan-review → approve-plan cycle. TEST-NNNN may also need `/address-test-plan-review TEST-NNNN` if task scope changed.
>
> Inform the TPO and Lead Developer that a design back-cycle is in progress.
