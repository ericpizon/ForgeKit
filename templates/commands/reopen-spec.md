---
id: reopen-spec
type: command
name: /reopen-spec
description: TPO command to reopen the spec phase when a discovery invalidates the approved SPEC — captures discovery record, runs impact analysis, resets product_validated, updates pipeline_status
status: active
version: 1
primary_role: tpo
---

# /reopen-spec

## Purpose

Re-enter the spec phase after approval when a discovery — a requirement change, a revealed user assumption, a post-design finding — invalidates the current SPEC. This is a controlled back-cycle: it records what was discovered, surfaces blast radius before making changes, and resets downstream status so no artifact proceeds on stale foundations.

## Usage

```
/reopen-spec SPEC-NNNN
```

## Role Gate

This command is restricted to the TPO. Confirm before proceeding:

> "Are you acting as the TPO (Technical Product Owner) for this thread? Confirm to proceed."

Do not proceed without explicit confirmation.

## Steps

### Step 1 — Read current state

Read SPEC-NNNN frontmatter. Note:
- Current `status` (should be `approved` or `revision_requested`)
- `product_validated` value
- `validated_by`, `validated_date`

Check the registry for downstream artifacts on this thread:
- Does DESIGN-NNNN exist? What is its status?
- Does PLAN-NNNN exist? Read its `pipeline_status` and milestone statuses.
- Does TEST-NNNN exist? What is its status?
- Are any milestones `in_progress`?
- Are there any open PRs in the registry?

### Step 2 — Discovery intake

Ask the TPO for four pieces of information:

> **To open a back-cycle, provide:**
>
> 1. **Discovered by:** Your name and role.
> 2. **What was discovered:** What changed or what was found? (1–3 sentences — be specific)
> 3. **Why this invalidates the SPEC:** Which sections are affected — Goals, Non-Goals, specific ACs, Constraints? Why do these need to change?
> 4. **Blast radius:** Based on what you know, which downstream artifacts (DESIGN, OBS, SEC, PLAN, TEST, in-progress milestones) are likely affected and why?

Wait for all four answers before proceeding.

### Step 3 — Impact analysis (read-only)

Based on Step 1 state and Step 2 answers, emit the impact analysis:

```
## Impact Analysis — SPEC-NNNN Back-cycle

SPEC-NNNN: approved → revision_requested

Product validation:
  product_validated will be reset to false.
  Prior sign-off by [name] on [date] will be preserved as prior_validated_by / prior_validated_date.

Downstream artifacts:
  DESIGN-NNNN ([status]): [affected — reason] | [likely unaffected — reason]
  OBS-NNNN ([status]): [affected if Goals or observability requirements change] | [not affected]
  SEC-NNNN ([status]): [affected if security scope changes] | [not affected]
  PLAN-NNNN ([status]): pipeline_status.spec will be updated to revision_requested
  TEST-NNNN ([status]): [affected if ACs change] | [not affected]
  MN in_progress: [risk noted — discovery may affect ongoing implementation]
  Open PRs: [warn if any found — must be reviewed after revision]
```

Then ask:

> "This back-cycle will set SPEC-NNNN to `revision_requested` and reset product validation. The impact analysis above shows downstream effects — these are notes, not automatic changes. Confirm to proceed."

Do not make any file changes until confirmed.

### Step 4 — Apply changes

After confirmation:

**SPEC-NNNN frontmatter:**
- Set `status: revision_requested`
- Set `product_validated: false`
- Rename `validated_by` → `prior_validated_by` (if set)
- Rename `validated_date` → `prior_validated_date` (if set)
- Increment `version`, update `updated` date
- Add changelog entry: `vN — revision_requested [date] — discovery: [one-line summary]`

**SPEC-NNNN body:**
- If `## Product Validation` section exists: add this note at the top of the section:
  `*Validation invalidated by back-cycle on [date] — re-run /validate-spec after revision.*`
- Append `## Discovery — [date]` section (see Discovery Record format below)

**PLAN-NNNN frontmatter (if PLAN exists):**
- Update `pipeline_status.spec: revision_requested`

### Step 5 — Discovery Record format

Append to SPEC-NNNN:

```markdown
## Discovery — YYYY-MM-DD

**Discovered by:** [name] — [role]
**Stage triggered:** spec

**What was discovered:**
[TPO's answer from Step 2]

**Why this invalidates the SPEC:**
[TPO's answer from Step 2 — specific sections named]

**Blast radius:**
- DESIGN-NNNN: [affected / not affected — reason]
- OBS-NNNN: [affected / not affected]
- SEC-NNNN: [affected / not affected]
- PLAN-NNNN: [affected / not affected]
- TEST-NNNN: [affected / not affected]
- MN in_progress: [risk noted / not affected — can continue]

**Status change:** SPEC-NNNN `approved` → `revision_requested`
```

## Quality Gate

Before closing:
- [ ] Discovery intake complete — all four fields answered
- [ ] Impact analysis shown to TPO and confirmed before any file changes
- [ ] SPEC-NNNN status set to `revision_requested`
- [ ] `product_validated` reset to false; prior fields renamed to `prior_*`
- [ ] `## Product Validation` annotated if section exists
- [ ] `## Discovery — [date]` section appended to SPEC-NNNN
- [ ] PLAN-NNNN `pipeline_status.spec` updated to `revision_requested` (if PLAN exists)
- [ ] SPEC-NNNN `version` incremented, `updated` date current, changelog entry added

**Self-check before proceeding:** Re-read each item above. Confirm each is complete, or state what was skipped and why before continuing to Handoff.

## Handoff

> **SPEC-NNNN is now `revision_requested`.**
>
> **Re-run sequence:**
> 1. Edit SPEC-NNNN to address the discovery — revise the affected sections (Goals, ACs, Non-Goals, Constraints as needed). Increment version and update `updated` date. Append a `## Revision Summary — Discovery [date]` section listing what changed and why.
> 2. Run `/review-spec SPEC-NNNN` — a fresh review round assesses whether the revision resolves the discovery.
> 3. Once clean: run `/validate-spec SPEC-NNNN` → `/approve-spec SPEC-NNNN ANALYSIS-NNNN`.
> 4. Assess DESIGN impact: run `/review-design SPEC-NNNN DESIGN-NNNN` to determine whether DESIGN changes are needed. If yes: begin the address-design-review → approve-design cycle.
>
> Note: no downstream artifact is safe to use until SPEC-NNNN is back to `approved`. Inform the Architect and Lead Developer that a spec back-cycle is in progress.
