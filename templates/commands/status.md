---
id: status
type: command
name: /status
description: Show current pipeline state for a thread — phase statuses, active review rounds, milestone progress, and suggested next action. Use for cold-start orientation after a session break.
status: active
version: 1
primary_role: any
---

# /status

## Purpose

Orient quickly to the current state of a thread after a session break or when joining mid-pipeline. Reads only frontmatter and milestone status lines — does not load full artifact content. Emits a compact summary and a suggested next action in under 30 seconds.

Use once PLAN-NNNN exists. Before PLAN is created, the pipeline is short enough that checking individual artifact frontmatters directly is sufficient.

## Usage

```
/status PLAN-NNNN
```

## Steps

1. **Read PLAN-NNNN frontmatter only.** Extract: `pipeline_status` block, `current_review_round`, `milestone_count`. Do not read the full PLAN body yet.

2. **Read PLAN-NNNN milestone status lines.** For each milestone, read only its `**Implementation status:**` line. Do not load task descriptions.

3. **Read active artifact frontmatters** — only for stages not yet complete:
   - If `pipeline_status.test` is not `approved`: read TEST-NNNN frontmatter (`status`, `current_review_round`).
   - If any milestone is `in_progress`: read `sdlc/reviews/IMPL-REVIEW-NNNN-MN.md` frontmatter (`current_review_round`) for each in-progress milestone.
   - If any `pipeline_status` field shows `revision_requested`: note the stage and the artifact involved.

4. **Emit the status summary** — see Output format below.

5. **Infer and state the next action** — see Next Action Inference below.

## Output

```
## Thread Status — PLAN-NNNN
Date: YYYY-MM-DD

### Pipeline
| Stage    | Artifact        | Status                        |
|----------|-----------------|-------------------------------|
| SPEC     | SPEC-NNNN       | approved                      |
| ANALYSIS | ANALYSIS-NNNN   | completed                     |
| DESIGN   | DESIGN-NNNN     | approved                      |
| OBS      | OBS-NNNN        | approved / not produced       |
| SEC      | SEC-NNNN        | completed / not triggered     |
| PLAN     | PLAN-NNNN       | draft (Round N) / approved    |
| TEST     | TEST-NNNN       | pending / draft (Round N) / approved |

### Implementation
*(Omit this section if TEST is not yet approved.)*

| Milestone | Name             | Status      | IMPL-REVIEW   |
|-----------|------------------|-------------|---------------|
| M1        | Foundation       | approved    | —             |
| M2        | Flag CRUD        | in_progress | Round 2       |
| M3        | Dashboard        | pending     | —             |

### Next action
[single line — exact next command to run and why]
```

## Next Action Inference

Read pipeline_status from the bottom up. The first incomplete stage determines the next action.

**Back-cycle in progress (check first):**
If any `pipeline_status` field is `revision_requested`:
> "Back-cycle in progress: [stage] is under revision. Resolve the [ARTIFACT-NNNN] revision before running downstream commands."

**PLAN not yet approved:**
- `current_review_round = 0` → "No plan review has run. Run `/review-plan PLAN-NNNN SPEC-NNNN DESIGN-NNNN`."
- `current_review_round > 0` → "Plan review Round N is the latest. Check Round N for outstanding findings — run `/address-plan-review PLAN-NNNN` or `/approve-plan PLAN-NNNN` depending on findings."

**PLAN approved, TEST not yet approved:**
- TEST does not exist → "Run `/create-test-plan SPEC-NNNN PLAN-NNNN OBS-NNNN SEC-NNNN`."
- TEST `current_review_round = 0` → "TEST draft exists. Run `/review-test-plan TEST-NNNN SPEC-NNNN OBS-NNNN`."
- TEST `current_review_round > 0` → "Test plan review Round N is the latest. Check Round N — run `/address-test-plan-review TEST-NNNN` or `/approve-test-plan TEST-NNNN` depending on findings."

**PLAN and TEST approved, milestones in progress:**
- Milestone `in_progress`, no IMPL-REVIEW file → "Run `/review-implementation PLAN-NNNN MN`."
- Milestone `in_progress`, IMPL-REVIEW `current_review_round = N` → "IMPL-REVIEW Round N is the latest. Check Round N — run `/address-implementation-review PLAN-NNNN MN` or `/approve-implementation PLAN-NNNN MN` depending on findings."

**All milestones approved:**
> "Implementation complete. Run `/create-pr PLAN-NNNN MN` for each milestone not yet PRed, or check PR status in the registry."

**Ambiguous state:**
If the current state doesn't match any pattern above, emit: "State is ambiguous — read [ARTIFACT-NNNN] Round [N] directly to determine next step."
