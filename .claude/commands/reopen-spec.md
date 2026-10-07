---
id: reopen-spec
type: command
name: /reopen-spec
description: TPO command to reopen the spec phase when a discovery invalidates the approved SPEC — captures discovery record, runs impact analysis, resets product_validated, updates pipeline_status
status: active
version: 2
primary_role: tpo
changelog:
  - v2 — added the coordination-repo path: a thread-wide SPEC back-cycle flags every participant's own PLAN, not just one — silently leaving other participants building against an invalidated SPEC is exactly the failure mode this command exists to prevent.
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

**`repo_role: code` (default):** check the registry for downstream artifacts on this thread:
- Does DESIGN-NNNN exist? What is its status?
- Does PLAN-NNNN exist? Read its `pipeline_status` and milestone statuses.
- Does TEST-NNNN exist? What is its status?
- Are any milestones `in_progress`?
- Are there any open PRs in the registry?

**`repo_role: coordination` (this SPEC belongs to a cross-repo thread):** read `sdlc/THREAD-NNNN/thread.yml` for the participant list. Does DESIGN-NNNN exist locally? What is its status? For **every** participant, resolve its own PLAN-NNNN (if any) via [`templates/reference/cross-repo-resolution.md`](../reference/cross-repo-resolution.md) — `pipeline_status`, milestone statuses, and any open PRs. A thread-wide SPEC invalidates the foundation every participant's PLAN was built on, not just one; silently leaving other participants unaware is exactly the failure mode this command exists to prevent.

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

**`repo_role: coordination`:** replace the single `PLAN-NNNN` line with one row per participant — every one of them was built against the SPEC now being reopened:

```
Downstream artifacts:
  DESIGN-NNNN ([status]): [affected — reason] | [likely unaffected — reason]
  Participants:
    transformer-service — PLAN-NNNN ([status]): pipeline_status.spec will be updated to revision_requested
    document-worker — PLAN-NNNN ([status]): pipeline_status.spec will be updated to revision_requested
    search-indexer — no PLAN yet: nothing to flag
  MN in_progress (any participant): [risk noted per participant]
  Open PRs (any participant): [warn if any found — must be reviewed after revision]
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

**`repo_role: coordination` — every participant's PLAN-NNNN (if it exists):**
- Update `pipeline_status.spec: revision_requested` in each one, not just the first found
- If a participant repo is reachable as one of this session's own working directories, apply the update directly (same as `approve-design`'s ARCH.md handling); otherwise, hand the update off explicitly to that participant — name the repo and the exact field change, never silently skip a participant just because it isn't locally reachable
- **Record which outcome happened, per participant, in the Discovery Record itself** (not just in this session's chat output) — a hand-off that only exists as a spoken instruction is invisible the moment this session ends. Use `applied` or `handed off — pending confirmation` in the Blast radius line below. A future `/sync-registry` run from the coordination repo can then cross-check: a participant still marked `handed off` whose own PLAN doesn't actually show `revision_requested` yet is a hand-off that never got carried out, not a false alarm.

### Step 5 — Discovery Record format

Append to SPEC-NNNN:

```markdown
## Discovery — YYYY-MM-DD

**Discovered by:** [name] — [role]
**Stage triggered:** spec
**Generated with:** [your current model identifier]

**What was discovered:**
[TPO's answer from Step 2]

**Why this invalidates the SPEC:**
[TPO's answer from Step 2 — specific sections named]

**Blast radius:**
- DESIGN-NNNN: [affected / not affected — reason]
- OBS-NNNN: [affected / not affected]
- SEC-NNNN: [affected / not affected]
- PLAN-NNNN: [affected / not affected] *(repo_role: coordination — list every participant with its outcome: "transformer-service — applied", "document-worker — handed off, pending confirmation", "search-indexer — no PLAN yet", etc.)*
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
- [ ] *(repo_role: coordination)* Every participant's PLAN-NNNN (not just one) has `pipeline_status.spec` updated, applied directly or explicitly handed off — none silently skipped
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
>
> **`repo_role: coordination`:** inform every participant's Lead Developer, not just one — each participant's PLAN was flagged above, and each needs to independently decide whether to pause `in_progress` work.
