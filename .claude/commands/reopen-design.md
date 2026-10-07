---
id: reopen-design
type: command
name: /reopen-design
description: Architect command to reopen the design phase when a discovery invalidates the approved DESIGN — captures discovery record, runs impact analysis, updates pipeline_status
status: active
version: 2
primary_role: architect
changelog:
  - v2 — added the coordination-repo path: a thread-wide DESIGN back-cycle flags every participant's own PLAN, not just one.
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

**`repo_role: code` (default):** check the registry for downstream artifacts on this thread:
- Does PLAN-NNNN exist? Read its `pipeline_status` and milestone statuses.
- Does TEST-NNNN exist? What is its status?
- Are any milestones `in_progress`?
- Are there any open PRs in the registry?

**`repo_role: coordination` (this DESIGN belongs to a cross-repo thread):** read `sdlc/THREAD-NNNN/thread.yml` for the participant list. For **every** participant, resolve its own PLAN-NNNN (if any) via [`templates/reference/cross-repo-resolution.md`](../reference/cross-repo-resolution.md) — `pipeline_status`, milestone statuses, and open PRs. Also note whether this DESIGN's Cross-Repo Contract has any `approved` revision already relied on by a participant — reopening the DESIGN does not automatically retract an already-approved Interface Freeze (see Step 4 below).

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

**`repo_role: coordination`:** replace the single `PLAN-NNNN` line with one row per participant, and add an explicit Cross-Repo Contract disposition — an already-approved Interface Freeze does not automatically retract just because something else in DESIGN reopens:

```
Downstream artifacts:
  OBS-NNNN ([status]): [affected if observability design changes] | [not affected]
  SEC-NNNN ([status]): [affected if security surface changes] | [not affected]
  Participants:
    transformer-service — PLAN-NNNN ([status]): pipeline_status.design will be updated to revision_requested
    document-worker — PLAN-NNNN ([status]): pipeline_status.design will be updated to revision_requested
  Cross-Repo Contract (approved rev [M], if any): [unaffected — the discovery doesn't touch any XDEP, stays approved]
    | [affected — the discovery changes a dependency; bump working.revision and re-run the Interface Freeze
       review before treating rev M as still current]
  MN in_progress (any participant): [risk noted — discovery may affect ongoing implementation]
  Open PRs (any participant): [warn if any found — must be reviewed after revision]

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

**`repo_role: coordination` — every participant's PLAN-NNNN (if it exists):**
- Update `pipeline_status.design: revision_requested` in each one
- Apply directly if the participant repo is reachable as a working directory this session; otherwise hand off explicitly, naming the repo and field change
- Record which outcome happened, per participant, in the Discovery Record's Blast radius line (`applied` or `handed off — pending confirmation`) — a hand-off that exists only in this session's chat output is invisible once the session ends. `/sync-registry` run later from the coordination repo can then catch a hand-off that never actually got carried out.

**`repo_role: coordination` — Cross-Repo Contract disposition (per the Step 3 determination):**
- If the discovery does not touch any XDEP: leave `cross_repo_contract.approved` untouched — it stays valid, and participants relying only on XDEPs may continue.
- If the discovery does touch an XDEP: bump `cross_repo_contract.working.revision`, set `working.status: draft`, and note in the Discovery Record that the Interface Freeze must be re-reviewed before the new `approved` revision can be trusted — do not silently leave the old `approved` revision looking current while the underlying XDEP has actually changed.

### Step 5 — Discovery Record format

Append to DESIGN-NNNN:

```markdown
## Discovery — YYYY-MM-DD

**Discovered by:** [name] — [role]
**Stage triggered:** design
**Generated with:** [your current model identifier]

**What was discovered:**
[Architect's answer from Step 2]

**Why this invalidates the DESIGN:**
[Architect's answer from Step 2 — specific sections named]

**Blast radius:**
- OBS-NNNN: [affected / not affected — reason]
- SEC-NNNN: [affected / not affected]
- PLAN-NNNN: [affected / not affected] *(repo_role: coordination — list every participant with its outcome: "transformer-service — applied", "document-worker — handed off, pending confirmation")*
- TEST-NNNN: [affected / not affected]
- MN in_progress: [risk noted / not affected — can continue]
- Cross-Repo Contract *(repo_role: coordination only)*: [unaffected — approved rev [M] stands] | [affected — working.revision bumped, Interface Freeze re-review required before rev [M+1] can be trusted]

**Status change:** DESIGN-NNNN `approved` → `revision_requested`
```

## Quality Gate

Before closing:
- [ ] Discovery intake complete — all four fields answered
- [ ] Impact analysis shown to Architect and confirmed before any file changes
- [ ] DESIGN-NNNN status set to `revision_requested`
- [ ] `## Discovery — [date]` section appended to DESIGN-NNNN
- [ ] PLAN-NNNN `pipeline_status.design` updated to `revision_requested` (if PLAN exists)
- [ ] *(repo_role: coordination)* Every participant's PLAN-NNNN updated, applied directly or explicitly handed off — none silently skipped
- [ ] *(repo_role: coordination)* Cross-Repo Contract disposition explicitly determined (unaffected vs. working.revision bumped) — never left ambiguous
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
>
> **`repo_role: coordination`:** inform every participant's Lead Developer, not just one. If the Cross-Repo Contract was affected, run `/approve-design DESIGN-NNNN interface-freeze` again once the new working revision clears `/review-design`'s Cross-Repo Contract Review — no participant should treat the prior `approved` revision as current in the meantime.
