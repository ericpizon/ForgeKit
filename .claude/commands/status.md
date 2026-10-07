---
id: status
type: command
name: /status
description: Show current pipeline state for a thread — phase statuses, active review rounds, milestone progress, and suggested next action. Use for cold-start orientation after a session break.
status: active
version: 3
primary_role: any
changelog:
  - v3 — added the `/status SPEC-NNNN` form (AC12): detects `status: promoted` and redirects to `superseded_by`, and detects `pending_promotion` set without `status: promoted` (an interrupted `/promote-to-thread` run) and directs the user to resume rather than reporting stale draft/approved status. Valid even before any PLAN exists, since promotion can happen as early as `status: draft`. Built for SPEC-0001 (T9).
  - v2 — added a cross-repo section for a thread participant: thread membership, the Cross-Repo Contract's working/approved state, whether this repo's own inputs are stale, and the resolution status of any XDEPs this repo's current milestone depends on.
---

# /status

## Purpose

Orient quickly to the current state of a thread after a session break or when joining mid-pipeline. Reads only frontmatter and milestone status lines — does not load full artifact content. Emits a compact summary and a suggested next action in under 30 seconds.

Use once PLAN-NNNN exists. Before PLAN is created, the pipeline is short enough that checking individual artifact frontmatters directly is sufficient.

## Usage

```
/status PLAN-NNNN
/status SPEC-NNNN   ← promotion-state check, see below — valid even before any PLAN exists
```

## SPEC-NNNN form — promotion state check

*(G3) Invoked directly against a SPEC rather than a PLAN — the only form that makes sense before a PLAN exists at all, since `/promote-to-thread` is valid starting from `status: draft` (during ANALYSIS, before any PLAN) just as much as `status: approved` (during DESIGN).* Read SPEC-NNNN frontmatter only, then branch:

- **`status: promoted`** (AC12) — report the promotion explicitly: "SPEC-NNNN was promoted to **THREAD-NNNN** (`superseded_by`) — THREAD-NNNN is now the authoritative artifact for this work, in the coordination repo. Do not continue the single-repo pipeline on SPEC-NNNN itself." Never show single-repo pipeline status (draft/approved/review-round state) for a promoted SPEC as if it were still authoritative — that status is frozen at whatever it was the moment promotion completed, and reporting it plainly would read as current when it isn't.
- **`pending_promotion` set, `status` not yet `promoted`** (the Round-1 self-caught fix — an interrupted `/promote-to-thread` run) — report: "Promotion in progress — SPEC-NNNN has a pending promotion to **THREAD-NNNN** that didn't complete. Resume via `/promote-to-thread SPEC-NNNN` — it will pick up from the first incomplete step, not mint a second THREAD-NNNN." Do not report ordinary draft/approved status here either; a human reading "status: draft" next to a dangling `pending_promotion` would reasonably assume promotion never started, when in fact it's mid-flight.
- **Neither field set** — this SPEC was never promoted. Report its plain `status` (draft/approved/etc.) — nothing cross-repo to add; fall through to the ordinary single-repo orientation a `/status` call on any other SPEC would give.

This form never loads a PLAN, a THREAD-NNNN's own artifacts, or anything beyond SPEC-NNNN's own frontmatter — it answers exactly one question ("is this SPEC still the authoritative artifact, or did it move") as fast as the file-level `/status` check this command already promises elsewhere.

## PLAN-NNNN form — full pipeline/milestone status

## Steps

1. **Read PLAN-NNNN frontmatter only.** Extract: `pipeline_status` block, `current_review_round`, `milestone_count`. Do not read the full PLAN body yet.

2. **Read PLAN-NNNN milestone status lines.** For each milestone, read only its `**Implementation status:**` line. Do not load task descriptions.

3. **Read active artifact frontmatters** — only for stages not yet complete:
   - If `pipeline_status.test` is not `approved`: read TEST-NNNN frontmatter (`status`, `current_review_round`).
   - If any milestone is `in_progress`: read `sdlc/reviews/IMPL-REVIEW-NNNN-MN.md` frontmatter (`current_review_round`) for each in-progress milestone.
   - If any `pipeline_status` field shows `revision_requested`: note the stage and the artifact involved.

4. **Cross-repo check** *(only if PLAN-NNNN frontmatter has a `thread` and `coordination_repo` pointer)* — this is the fact most easily lost track of when working alone or async: "am I blocked on another repo, and is anyone waiting on me?"
   - Re-resolve the coordination repo's current Cross-Repo Contract `working`/`approved` state (per [`templates/reference/cross-repo-resolution.md`](../reference/cross-repo-resolution.md)).
   - Compare it to this PLAN's own `inputs.cross_repo_contract.revision` — if they differ, this PLAN is stale against the current approved contract.
   - Resolve every XDEP where this repo is provider or consumer (from `sdlc/dependencies/THREAD-NNNN-XDEP-*.yml` in the coordination repo). For each, note its `status` (`active`/`superseded`/`retired`) and, if this repo's current milestone has a task with `satisfies_xdeps` referencing it, whether that task is done.
   - Note any XDEP where this repo is the *consumer* and the *provider*'s side isn't done yet (this repo is blocked), and any XDEP where this repo is the *provider* and the *consumer* hasn't started (someone may be waiting on this repo).

5. **Emit the status summary** — see Output format below.

6. **Infer and state the next action** — see Next Action Inference below.

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

### Cross-repo
*(Omit this entire section if PLAN-NNNN has no `thread` pointer.)*

**Thread:** THREAD-NNNN (coordination repo: org/forge-sdlc)
**Cross-Repo Contract:** working rev [N] ([draft/in_review/approved]) — approved rev [M] | **STALE** if this PLAN's `inputs.cross_repo_contract.revision` doesn't match [M]

| XDEP | Role | Status | This repo's task | Blocking? |
|---|---|---|---|---|
| THREAD-0142-XDEP-01 | consumer | active | satisfied (T7) | — |
| THREAD-0142-XDEP-02 | provider | active | not yet satisfied | someone may be waiting on this repo |

### Next action
[single line — exact next command to run and why]
```

## Next Action Inference

Read pipeline_status from the bottom up. The first incomplete stage determines the next action.

**Back-cycle in progress (check first):**
If any `pipeline_status` field is `revision_requested`:
> "Back-cycle in progress: [stage] is under revision. Resolve the [ARTIFACT-NNNN] revision before running downstream commands."

**Cross-Repo Contract stale (check second, thread participants only):**
If the Cross-Repo Contract's approved revision no longer matches this PLAN's `inputs.cross_repo_contract.revision`:
> "This PLAN is stale against the Cross-Repo Contract (built against rev [N], approved is now rev [M]). Re-run `/create-plan` before proceeding with implementation."

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
