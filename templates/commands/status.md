---
id: status
type: command
name: /status
description: Show current pipeline state for a thread — phase statuses, active review rounds, milestone progress, and suggested next action. Use for cold-start orientation after a session break.
status: active
version: 4
primary_role: any
changelog:
  - v4 — added the general pre-PLAN form (SPEC/ANALYSIS/DESIGN/OBS/SEC orientation, for a SPEC never promoted) where `/status` previously had no resolver at all before a PLAN existed; fixed the milestone Next Action branch, which had no handling for `test_failed`/`tests_passed`/`pr_open` despite those being real states elsewhere in the pipeline; added the structured `resolved_state` output block (with `ambiguous_cause`) alongside the existing human-readable report, and the pre-existing-artifact fallback rule (missing structured field → `ambiguous`, never a prose-parsing fallback). Built for the ForgeKit Operator feature (PLAN-0001 T11/T12/T13) — the single biggest gap the original Operator gap-analysis identified in this command. Amended during M1's own `/review-implementation` Round 1: the pre-PLAN form now checks for ANALYSIS-NNNN's existence before reading it, since it doesn't exist at all immediately after `/create-spec` — the single most common state this form will actually be invoked against ([SRE-1]); the milestone scan now states an explicit output for the case where every remaining milestone is blocked-pending with none actionable, rather than relying on the generic ambiguous-state fallback implicitly ([LD-1]).
  - v3 — added the `/status SPEC-NNNN` form (AC12): detects `status: promoted` and redirects to `superseded_by`, and detects `pending_promotion` set without `status: promoted` (an interrupted `/promote-to-thread` run) and directs the user to resume rather than reporting stale draft/approved status. Valid even before any PLAN exists, since promotion can happen as early as `status: draft`. Built for SPEC-0001 (T9).
  - v2 — added a cross-repo section for a thread participant: thread membership, the Cross-Repo Contract's working/approved state, whether this repo's own inputs are stale, and the resolution status of any XDEPs this repo's current milestone depends on.
---

# /status

## Purpose

Orient quickly to the current state of a thread after a session break or when joining mid-pipeline. Reads only frontmatter and milestone status lines — does not load full artifact content. Emits a compact summary and a suggested next action in under 30 seconds.

`/status PLAN-NNNN` gives full milestone/pipeline detail. `/status SPEC-NNNN` covers both the promotion-state check and, when the SPEC was never promoted, a general pre-PLAN orientation across SPEC/ANALYSIS/DESIGN/OBS/SEC — the same resolver, reading one stage further back than a PLAN-NNNN form needs to.

## Usage

```
/status PLAN-NNNN
/status SPEC-NNNN   ← promotion-state check, then general pre-PLAN orientation — valid even before any PLAN exists
```

## SPEC-NNNN form — promotion state check

*(G3) Invoked directly against a SPEC rather than a PLAN — the only form that makes sense before a PLAN exists at all, since `/promote-to-thread` is valid starting from `status: draft` (during ANALYSIS, before any PLAN) just as much as `status: approved` (during DESIGN).* Read SPEC-NNNN frontmatter only, then branch:

- **`status: promoted`** (AC12) — report the promotion explicitly: "SPEC-NNNN was promoted to **THREAD-NNNN** (`superseded_by`) — THREAD-NNNN is now the authoritative artifact for this work, in the coordination repo. Do not continue the single-repo pipeline on SPEC-NNNN itself." Never show single-repo pipeline status (draft/approved/review-round state) for a promoted SPEC as if it were still authoritative — that status is frozen at whatever it was the moment promotion completed, and reporting it plainly would read as current when it isn't.
- **`pending_promotion` set, `status` not yet `promoted`** (the Round-1 self-caught fix — an interrupted `/promote-to-thread` run) — report: "Promotion in progress — SPEC-NNNN has a pending promotion to **THREAD-NNNN** that didn't complete. Resume via `/promote-to-thread SPEC-NNNN` — it will pick up from the first incomplete step, not mint a second THREAD-NNNN." Do not report ordinary draft/approved status here either; a human reading "status: draft" next to a dangling `pending_promotion` would reasonably assume promotion never started, when in fact it's mid-flight.
- **Neither field set** — this SPEC was never promoted. Fall through to the **general pre-PLAN form** below rather than reporting bare `status` alone.

The promotion check above never loads anything beyond SPEC-NNNN's own frontmatter — it answers exactly one question ("is this SPEC still the authoritative artifact, or did it move") as fast as possible. The general form below is a second, separate pass that only runs once promotion is ruled out.

## General pre-PLAN form — SPEC/ANALYSIS/DESIGN/OBS/SEC status before a PLAN exists

*(Resolves a gap in `/status`'s own original scope: before PLAN-NNNN exists, the pipeline previously had no resolver at all, only "check individual artifact frontmatters directly" — this form closes that for the one thing that actually benefits from a cold-start summary just as much as the PLAN-NNNN form does: a multi-round SPEC/DESIGN review cycle.)*

1. Read SPEC-NNNN frontmatter (`status`, `version`). If `status` is `draft` or `revision_requested`: **check whether ANALYSIS-NNNN exists before reading it** — it does not exist at all until `/review-spec` has run once, which is the normal state immediately after `/create-spec`, not a rare edge case (fixed here — the original form assumed it always existed). If it doesn't exist, the next action is `/review-spec SPEC-NNNN`; stop there. If it exists, read its frontmatter (`readiness_verdict`, `latest_round` if present) — this is as far as the pipeline goes.
2. If SPEC-NNNN `status: approved` or later, check whether DESIGN-NNNN exists. If not: the next action is `/create-design`.
3. If DESIGN-NNNN exists, read its frontmatter (`status`, `latest_round`, `current_review_round`). If a required structured field like `latest_round` is absent on an artifact that predates this field being introduced, do **not** fall back to parsing the artifact's prose `**Status:**` line — see the `ambiguous_cause` rule under Output, below.
4. If DESIGN-NNNN `status: approved`, check OBS-NNNN/SEC-NNNN existence and state (OBS always expected; SEC only if ANALYSIS-NNNN's `security_review_required: yes`).
5. Emit the same `## Thread Status` output shape as the PLAN-NNNN form's Pipeline table, restricted to the rows that exist so far (SPEC/ANALYSIS/DESIGN/OBS/SEC only — no Implementation or Cross-repo sections, since no PLAN exists yet to anchor them), plus the `resolved_state` block and a Next Action line following the same Next Action Inference logic, evaluated against whichever of these stages is the first incomplete one.

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

**Structured output** — appended after the human-readable report above, on every invocation of either form (resolves SPEC-0001 §51 Open Question 1; schema per ADR-0002 of the ForgeKit Operator feature, PLAN-0001 T13):

```yaml
resolved_state:
  stage: spec | analysis | design | obs | sec | plan | test_plan | implementation | release
  gate: ready | blocked | human_decision_required | ambiguous
  subject: <ARTIFACT-NNNN or THREAD-NNNN>
  blocking_count: <int>
  stale: <bool>
  next_command: "<exact command invocation string — same as the human-readable Next action line>"
  ambiguous_cause: missing_structured_field | unrecognized_state_combination   # present only when gate: ambiguous
```

This is a derived view over the same frontmatter fields the human-readable report above already reads — never a second source of truth. `gate` is restricted to exactly the four values listed; every more specific stopping condition (which outcome name applies, if any) is a concern for whatever resolver consumes this, not for `/status` itself.

**Pre-existing-artifact fallback:** when computing `resolved_state` requires a structured field (`latest_round`, `implementation_execution`, etc.) that a given artifact predates and therefore lacks entirely, do **not** fall back to parsing that artifact's prose `**Status:**`/`**Summary:**` line — that would defeat the entire point of reading structured fields instead of prose. Emit `gate: ambiguous` with `ambiguous_cause: missing_structured_field` instead. This is a one-time cost per pre-existing artifact: running the normal `/review-*` command for that stage once backfills the field as a side effect, after which `/status` resolves normally. Reserve `ambiguous_cause: unrecognized_state_combination` for every other case this command cannot resolve cleanly — the distinction matters operationally, since the first is an expected migration cost and the second is a genuine resolver gap worth filing a BUG against.

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

**PLAN and TEST approved — milestone implementation in progress.** Read each milestone's `**Implementation status:**` line in order and resolve the next action from the first one not yet `pr_merged`, branching on its actual state (previously this only ever branched on `in_progress`, with no handling at all for `test_failed`/`tests_passed`/`pr_open` despite those being real, heavily-used states in `execute-tests.md`/`create-pr.md`/`close-milestone.md` — fixed here):
- `pending`, all `depends_on_milestones` now `pr_merged` → "Run `/implement-plan PLAN-NNNN MN`."
- `pending`, a dependency not yet `pr_merged` → note MN as blocked on that dependency; continue checking the next milestone rather than stopping here.
- `in_progress`, no IMPL-REVIEW file → "Run `/review-implementation PLAN-NNNN MN`."
- `in_progress`, IMPL-REVIEW `current_review_round = N` → "IMPL-REVIEW Round N is the latest. Check Round N — run `/address-implementation-review PLAN-NNNN MN` or `/approve-implementation PLAN-NNNN MN` depending on findings."
- `approved` → "Run `/execute-tests PLAN-NNNN MN TEST-NNNN`."
- `test_failed` → "MN has P0 test failures. Run `/address-test-failures PLAN-NNNN MN` — do not re-implement."
- `tests_passed` → "Run `/create-pr PLAN-NNNN MN`."
- `pr_open` → "MN's PR is open. Merge it on GitHub, then run `/close-milestone PLAN-NNNN MN`."
- `pr_merged` → MN is done; continue to the next milestone.

**All milestones `pr_merged`:**
> "Implementation complete. Run `/create-release-plan` (or check release/PR status in the registry)."

**Milestone-level deadlock (every remaining milestone blocked-pending, none actionable):**
If the scan reaches the end of the milestone list with every non-`pr_merged` milestone in the `pending`-blocked state (no dependency chain resolving to anything `in_progress`/`approved`/etc.), report this explicitly rather than falling through silently to the generic ambiguous-state message below:
> "No milestone is currently actionable — [MN] is blocked on [MX], which is blocked on [...]. Check `depends_on_milestones` in PLAN-NNNN for a cycle or a misconfigured dependency."

**Ambiguous state:**
If the current state doesn't match any pattern above, emit: "State is ambiguous — read [ARTIFACT-NNNN] Round [N] directly to determine next step." In the structured output, this is `resolved_state.gate: ambiguous`, with `ambiguous_cause` set per the Output section's rule above.
