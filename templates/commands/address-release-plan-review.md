---
id: address-release-plan-review
type: command
name: /address-release-plan-review
description: SRE command to address findings from a RELEASE-PLAN review round — updates the plan, appends address summary, re-enters review cycle
status: active
version: 2
primary_role: sre
links:
  - relation: follows
    target: review-release-plan
  - relation: precedes
    target: review-release-plan
changelog:
  - v2 — added the `recurs_from: {round, finding}` finding-lineage convention for a fix that resolves the same underlying issue as a prior round's finding. Built for the ForgeKit Operator feature (PLAN-0001 T10).
---

# /address-release-plan-review

## Purpose

Address all findings from the latest review round of `/review-release-plan`. Update the relevant sections of the RELEASE-PLAN document, then append an address summary. Re-enters the review cycle.

## Usage

```
/address-release-plan-review RELEASE-PLAN-v1.0.0
```

## Process

### Step 1 — Read findings

Open `RELEASE-PLAN-vX.Y.Z.md`. Find the latest `## Release Review — Round N` section. Collect all blocking and non-blocking findings.

### Step 2 — Address findings

For each blocking finding, update the relevant section of the RELEASE-PLAN:
- Scope corrections — add missing milestones, remove out-of-scope items
- Rollback plan gaps — add explicit rollback steps or step-by-step reversal instructions
- Risk level corrections — update the risk assessment to reflect the actual change set
- SEC finding documentation — add named approver and revisit condition for `accepted_risk` findings

**If a finding is the same underlying issue as one from a prior round** (your own judgment — a finding ID is round-scoped and does not persist automatically), note `recurs_from: {round: [prior round], finding: [prior finding ID]}` in this finding's Address Summary table row.

For non-blocking findings: fix if straightforward and in-scope; if deferring, document the reason.

### Step 3 — Append address summary

Append immediately after the latest review section:

```markdown
### Address Summary — Round N
Date: [today]

| Finding | Resolution |
|---|---|
| [ROLE-N] | Fixed: [one-line description of what changed] (+ `recurs_from: {round, finding}` if this recurs a prior round's finding) |
| [ROLE-N] | Deferred: [reason] |

**Not addressed:** [any explicitly deferred findings with reason — or "None."]
```

### Step 4 — Handoff

```
/review-release-plan RELEASE-PLAN-vX.Y.Z
```

Nothing is committed here. `/approve-release-plan` is the single commit for the entire release planning phase.
