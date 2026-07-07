---
id: reopen-plan
type: command
name: /reopen-plan
description: Lead Developer command to reopen the implementation planning phase when a discovery invalidates the approved PLAN — captures discovery record, handles in-progress milestone risk, sets TEST to revision_requested, updates pipeline_status
status: active
version: 1
primary_role: lead-developer
---

# /reopen-plan

## Purpose

Re-enter the implementation planning phase after approval when a discovery — a technical blocker found during implementation, an integration gap, a scope clarification from the TPO — invalidates the current PLAN. Records the discovery, assesses in-progress milestone risk, resets TEST-NNNN status so it cannot proceed against a stale plan, and provides a clear re-run sequence.

## Usage

```
/reopen-plan PLAN-NNNN
```

## Role Gate

This command is restricted to the Lead Developer. Confirm before proceeding:

> "Are you acting as the Lead Developer for this thread? Confirm to proceed."

Do not proceed without explicit confirmation.

## Steps

### Step 1 — Read current state

Read PLAN-NNNN frontmatter. Note current `status`, `pipeline_status`, `milestone_count`.

For each milestone, read its `**Implementation status:**` line.

Check:
- Does TEST-NNNN exist? What is its status?
- Are any milestones `in_progress`? List which ones.
- Are there any open PRs in the registry?

### Step 2 — In-progress milestone check

If any milestone is `in_progress`, ask before proceeding to discovery intake:

> "Milestone MN is currently in_progress. Before continuing, answer:
>
> 1. Is this milestone affected by the discovery that's driving this back-cycle?
> 2. Can in-progress work on MN continue, or should it be paused until the plan revision is complete?"

Record both answers. They will be included in the discovery record. Do not block the back-cycle based on the answers — but surface the risk explicitly so the team can decide.

### Step 3 — Discovery intake

Ask the Lead Developer for four pieces of information:

> **To open a back-cycle, provide:**
>
> 1. **Discovered by:** Your name and role.
> 2. **What was discovered:** What implementation finding, blocker, or scope clarification was uncovered? (1–3 sentences)
> 3. **Why this invalidates the PLAN:** Which tasks, milestones, or AC coverage entries are affected? Why do these need to change?
> 4. **Blast radius:** Which downstream artifacts (TEST-NNNN, in-progress milestones, open PRs) are affected and how?

Wait for all four answers before proceeding.

### Step 4 — Impact analysis (read-only)

Emit the impact analysis:

```
## Impact Analysis — PLAN-NNNN Back-cycle

PLAN-NNNN: approved → revision_requested
PLAN-NNNN pipeline_status.plan: approved → revision_requested

Downstream artifacts:
  TEST-NNNN ([status]): will be set to revision_requested — tasks referenced
    in the AC coverage map may change; run /address-test-plan-review after
    plan revision is complete
  MN in_progress: [risk noted per Step 2 answers — affected / can continue]
  Open PRs: [warn if any found — must be reviewed after revision]

Note: the TPO should be informed that a plan back-cycle is in progress.
```

Then ask:

> "This back-cycle will set PLAN-NNNN to `revision_requested` and set TEST-NNNN to `revision_requested` if it exists. Confirm to proceed."

Do not make any file changes until confirmed.

### Step 5 — Apply changes

After confirmation:

**PLAN-NNNN frontmatter:**
- Set `status: revision_requested`
- Set `pipeline_status.plan: revision_requested`
- Increment `version`, update `updated` date
- Add changelog entry: `vN — revision_requested [date] — discovery: [one-line summary]`

**PLAN-NNNN body:**
- Append `## Discovery — [date]` section (see Discovery Record format below)

**TEST-NNNN frontmatter (if TEST exists and status is `approved`):**
- Set `status: revision_requested`
- Increment `version`, update `updated` date
- Add changelog entry: `vN — revision_requested [date] — plan back-cycle: [one-line summary]`
- Append to TEST-NNNN: `*Test plan set to revision_requested by PLAN-NNNN back-cycle on [date]. Re-run /address-test-plan-review after plan revision is complete.*`

### Step 6 — Discovery Record format

Append to PLAN-NNNN:

```markdown
## Discovery — YYYY-MM-DD

**Discovered by:** [name] — [role]
**Stage triggered:** plan
**Generated with:** [your current model identifier]

**What was discovered:**
[Lead Developer's answer from Step 3]

**Why this invalidates the PLAN:**
[Lead Developer's answer from Step 3 — specific tasks and milestones named]

**In-progress milestones:**
- MN: [affected — pausing recommended] | [not affected — can continue]
  Rationale: [answer from Step 2]

**Blast radius:**
- TEST-NNNN: [affected / not affected — reason]
- Open PRs: [warn if any]

**Status change:** PLAN-NNNN `approved` → `revision_requested`; TEST-NNNN `approved` → `revision_requested` (if applicable)
```

## Quality Gate

Before closing:
- [ ] Discovery intake complete — all four fields answered
- [ ] In-progress milestone risk assessed and recorded (if any milestones were in_progress)
- [ ] Impact analysis shown and confirmed before any file changes
- [ ] PLAN-NNNN status set to `revision_requested`
- [ ] PLAN-NNNN `pipeline_status.plan` updated to `revision_requested`
- [ ] `## Discovery — [date]` section appended to PLAN-NNNN
- [ ] TEST-NNNN status set to `revision_requested` (if TEST existed and was `approved`)
- [ ] PLAN-NNNN `version` incremented, `updated` date current, changelog entry added

**Self-check before proceeding:** Re-read each item above. Confirm each is complete, or state what was skipped and why before continuing to Handoff.

## Handoff

> **PLAN-NNNN is now `revision_requested`.**
>
> **Re-run sequence:**
> 1. Edit PLAN-NNNN to address the discovery — revise the affected tasks, milestones, AC coverage, or milestone dependencies as needed. Increment version and update `updated` date.
> 2. Run `/review-plan PLAN-NNNN SPEC-NNNN DESIGN-NNNN` — all perspectives assess the revision.
> 3. Once clean: run `/approve-plan PLAN-NNNN`.
> 4. Sync the test plan: run `/address-test-plan-review TEST-NNNN` to bring it in line with the revised PLAN. Once clean: `/approve-test-plan TEST-NNNN`.
> 5. Resume or reassess any `in_progress` milestones per the risk assessment recorded in the discovery section.
>
> Inform the TPO that a plan back-cycle is in progress.
