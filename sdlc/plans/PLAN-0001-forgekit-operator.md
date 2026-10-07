---
id: PLAN-0001
type: plan
schema_version: 1
title: "ForgeKit Operator — Implementation Plan"
status: approved
version: 2
updated: 2026-10-07
target_design_version: 6
includes_observability_tasks: true
includes_security_tasks: true
milestone_count: 5
current_review_round: 2
generation:
  command: /create-plan
  command_version: 5
  model: claude-sonnet-5
  generated_at: 2026-10-07
  session_id: null
pipeline_status:
  spec: SPEC-0001 approved
  analysis: ANALYSIS-0001 completed
  design: DESIGN-0001 approved
  obs: OBS-0001 approved
  sec: SEC-0001 completed
  plan: approved
  test: approved
approval:
  mode: repo_only
  approved_by: "Eric Pizon"
  approved_role: lead-developer
  approved_at: "2026-10-07"
  decision_summary: "PLAN-0001 (ForgeKit Operator) approved for test planning phase after 2 review rounds — all blocking findings resolved, all non-blocking findings documented."
  external_event_id: null
  external_event_uri: null
policy_evaluation:
  evaluated: null
  policy_pack: null
  result: null
changelog:
  - "v2 — Addressed Plan Review Round 1: T25 rewritten to compose existing mechanisms instead of gesturing at concepts (DEV-1); T22 gained the AC-10 read-path invariant (QE-1) and the lookup-before-create invariant stated explicitly (ARCH-2); SEC-0001 SEC-2 and ADR citations added (SEC-1, ARCH-1). AC-18 moved from T22 to T25's coverage."
  - "v2 — approved after 2 review rounds"
---

# PLAN-0001 — ForgeKit Operator

Sequences SPEC-0001's §41 "Operator Readiness Prerequisites" as the earliest milestones (M1–M3), per the TPO decision already recorded there resolving `[SC-1]`, before the Operator command surface itself (M4–M5).

## Task Breakdown

### M1 — Structured Review Outcomes & Status Resolver

**T1** | functional | `templates/commands/review-design.md`
Add the `latest_round: {round, blocking, non_blocking, verdict}` frontmatter field, written alongside the existing prose `**Status:**` line, per DESIGN-0001 §4/Key Decision 2.
**AC:** AC-06

**T2** | functional | `templates/commands/review-plan.md`
Same `latest_round` field addition as T1.
**AC:** AC-06

**T3** | functional | `templates/commands/review-test-plan.md`
Same `latest_round` field addition as T1.
**AC:** AC-06

**T4** | functional | `templates/commands/review-implementation.md`
Same `latest_round` field addition as T1, replacing the existing `**Summary:**` prose line's sole role as the gate signal.
**AC:** AC-06

**T5** | functional | `templates/commands/review-release-plan.md`
Same `latest_round` field addition as T1, replacing the bare bold-line prose's sole role as the gate signal.
**AC:** AC-06

**T6** | functional | `templates/commands/address-design-review.md`
Add the `recurs_from: {round, finding}` finding-lineage convention to the "On Re-Review" section, per DESIGN-0001 §4.
**AC:** AC-08

**T7** | functional | `templates/commands/address-plan-review.md`
Same `recurs_from` convention as T6.
**AC:** AC-08

**T8** | functional | `templates/commands/address-test-plan-review.md`
Same `recurs_from` convention as T6.
**AC:** AC-08

**T9** | functional | `templates/commands/address-implementation-review.md`
Same `recurs_from` convention as T6.
**AC:** AC-08

**T10** | functional | `templates/commands/address-release-plan-review.md`
Same `recurs_from` convention as T6.
**AC:** AC-08

**T11** | functional | `templates/commands/status.md`
Add the general pre-PLAN form: reads SPEC/ANALYSIS/DESIGN/OBS/SEC frontmatter directly (the same way the existing PLAN-and-later form reads PLAN/TEST/milestone frontmatter) per DESIGN-0001 §3.5.
**AC:** AC-04

**T12** | functional | `templates/commands/status.md`
Fix the Next Action Inference's milestone branch to add explicit handling for `test_failed`/`tests_passed`/`pr_open` (currently absent) per DESIGN-0001 §3.5.
**AC:** AC-05

**T13** | functional | `templates/commands/status.md`
Add the `resolved_state` structured output block (schema per ADR-0002, including `ambiguous_cause`) appended to the existing human-readable report; add the pre-existing-artifact fallback rule (missing structured field forces `gate: ambiguous`, never a prose-parsing fallback) per DESIGN-0001 §3.5.
**ADR:** ADR-0002.
**AC:** AC-03, AC-22

### M2 — Implementation Tracking, BUG Canonicalization & Test Classification

**T14** | functional | `templates/commands/implement-plan.md`
Write `implementation_execution: {state, attempt, started_at, completed_at}` on the milestone entry: `state: started` at invocation start, `state: completed` at successful completion, per DESIGN-0001 §4.
**AC:** AC-10

**T15** | functional | `templates/commands/create-bug.md`
Declare the canonical BUG status vocabulary once in this command's Output Contract: `reported → accepted | rejected | needs_info → pending_verification → verified | reopened → incident_active → fix_deployed → closed`, per DESIGN-0001 §4.
**AC:** AC-12 *(supporting — canonicalizes the vocabulary AC-12's classification result flows into)*

**T16** | functional | `templates/commands/triage-bug.md`, `templates/commands/resolve-incident.md`, `templates/commands/escalate-to-incident.md`
Reconcile each command's BUG `status` reads/writes against T15's canonical vocabulary — fix any value referenced that isn't in the canonical list, or any value the canonical list expects that these commands never actually write.
**AC:** AC-12 *(supporting)*

**T17** | functional | `templates/commands/execute-tests.md`
Add the `classification: production_bug | test_bug` column to the test-execution table, recorded structurally at the point QE answers, per DESIGN-0001 §4.
**AC:** AC-12

### M3 — Release Approval Binding & Resumability

**T18** | functional | `templates/commands/create-release-plan.md`, `templates/commands/approve-release-plan.md`
Extend the existing `approval:` block with `candidate_ref`/`release_plan_revision` fields; `/operate` (built in M4) checks both against the current candidate/RELEASE-PLAN version before executing, per DESIGN-0001 §4/Key Decision 6.
**AC:** AC-14

**T19** | functional | `templates/commands/execute-release.md`
Write `deployment_progress: {migrations_applied, deployed, tagged, updated_at}` incrementally after each existing numbered step, not only at the end, per DESIGN-0001 §4/Key Decision 7.
**AC:** AC-21

### M4 — Operator Core: Command Execution Cycle & Safety Mechanisms

**T20** | functional | `templates/project.yml.template`
Add the `operator:` policy block (`max_review_rounds`, `max_actions_per_run`, `allow.*`, `require_human.*`) with every `allow.*` key defaulting to `false` — absence of the block or a specific key both mean "false" — per DESIGN-0001 §4.
**ADR:** ADR-0003.
**AC:** AC-25

**T21** | functional | `sdlc/operator-runs/README.md` *(new file, mirroring every other `sdlc/` subdirectory's README convention)*
Document the `OPRUN-NNNN` artifact type: naming convention, lifecycle (terminal vs. non-terminal `status` values per DESIGN-0001 §4), and when it's created/read.
**AC:** *(supporting — no direct AC; infrastructure for AC-09/AC-11)*

**T22** | functional | `templates/commands/operate.md` *(new command)*
Implement the Command Execution Cycle (resolve → determine → verify policy → invoke → verify outcome → re-resolve, DESIGN-0001 §3.2/§15) and the full `OPRUN-NNNN` schema (DESIGN-0001 §4). Specifically includes:
- **Lookup-before-create** (DESIGN-0001 §3.4, resolves `[DBA-1]`): before creating a new `OPRUN-NNNN`, scan `sdlc/operator-runs/` for an existing record matching this `work_unit` with a non-terminal `status`. **Invariant: never create a second `OPRUN-NNNN` for a `work_unit` that already has a non-terminal one — resume the existing record instead, every time, with no exception.**
- **Interrupted-implementation recovery routing** (DESIGN-0001 §3.4/§12/§37): on resume, before invoking implementation review for any milestone, check that milestone's `implementation_execution.state`. **Invariant: if `state: started` with no `completed_at`, Operator must route to the implementation recovery path and must never invoke implementation review directly — this is the read side of the guarantee T14 writes.**
- The three-stage gate-to-outcome resolution procedure (DESIGN-0001 §3.6).
**ADR:** ADR-0001 (orchestration mechanism), ADR-0002 (`resolved_state`/`OPRUN-NNNN.status` relationship this procedure implements).
**AC:** AC-01, AC-02, AC-09, AC-10, AC-11, AC-13, AC-16, AC-19, AC-20, AC-23, AC-26

**T23** | functional | `templates/commands/operate.md`
Implement the Autonomy Overlay Rule's formal-gate/routine-confirmation classification (DESIGN-0001 §3.2) and the command → policy-category table (§3.3), including the `security-review.md` special case (any `accepted_risk` entry in `findings_status` forces a hard stop, regardless of policy) and the explicit fail-closed tiebreak for unrecognized Handoff phrasing or a malformed `operator.allow.*` value.
**Source:** SEC-0001 SEC-1 (fail-closed tiebreak) and SEC-0001 SEC-2 (the `findings_status` content-check implemented here is SEC-2's mitigation mechanism) — see SEC-0001 for both findings; SEC-1's mitigation text is DESIGN-0001 §3.2's existing `[LD-1]` disposition, now made literal here rather than left as an un-implemented intention. **ADR:** ADR-0001 (orchestration mechanism), ADR-0003 (fail-closed defaults / per-step classification).
**AC:** AC-07, AC-13, AC-27

**T24** | functional | `templates/commands/operate.md`
Implement resumption containment: every re-read of an `OPRUN-NNNN`'s `history:` list or decision-package body — including Operator's own prior writes — is treated under the same data-not-instruction framing as a first-time external read, per DESIGN-0001 §3.4.
**Source:** SEC-0001 SEC-3 — see SEC-0001 for the full finding.
**AC:** AC-09 *(supporting — hardens the resumption guarantee against adversarial content)*

**T25** | functional | `templates/commands/operate.md`
Implement multi-repo orchestration by composing already-specified mechanisms, not new ones (revised during Plan Review Round 1, `[DEV-1]`, to remove the implementation-ambiguity the original wording left open):
- **Participant/XDEP resolution (AC-15a):** for each participant, apply T22's own single-repo Command Execution Cycle against that participant's `work_unit`. For each XDEP where this thread's repos are provider/consumer, satisfaction is: does any participant's PLAN carry a task with `satisfies_xdeps` naming this XDEP whose milestone has reached `pr_merged`? (Per `templates/reference/xdep-schema.md` — no cached `satisfied` field exists or is introduced; this is a live check every time, per SPEC AC-16.)
- **Interface Freeze (AC-15b):** compare `cross_repo_contract.working.revision` to `approved.revision` (existing fields, `templates/reference/cross-repo-resolution.md`); block implementation work depending on a revision where these differ.
- **Per-repo readiness (AC-15c):** apply AC-15a's resolution independently per participant; never let one participant's readiness justify advancing another's.
- **Cross-repo validation (AC-15d):** check for a `PASS`-equivalent result (per `templates/reference/xdep-schema.md`'s existing 4-value scenario-result enum) on every required cross-repo scenario before considering thread-wide implementation complete.
- **Dependency-deadlock detection (AC-17):** walk the XDEP ordering-dependency graph for a cycle, reusing the same cycle-detection check `create-design.md`'s own Step 9b already performs at design time against a thread's XDEP set — applied here at runtime against the current live set instead. A cycle found → stop with `HUMAN_DECISION_REQUIRED`, naming the specific participants/dependencies in the cycle; never break it automatically.
- **Thread-wide release eligibility (AC-15e):** the logical AND of AC-15a through AC-15d holding across every participant — never derived from any single participant's state alone.
- **Promotion during discovery (AC-18):** invoke the existing `/promote-to-thread` directly; do not reimplement its logic.
**AC:** AC-15, AC-15a, AC-15b, AC-15c, AC-15d, AC-15e, AC-17, AC-18

### M5 — Operator Observability & Supporting Tooling

**T26** | observability | `templates/commands/operate.md`
**Source:** OBS-0001 IT-1 — see OBS-0001 for full implementation spec.
**Milestone:** M5

**T27** | observability | `templates/commands/operate.md`
**Source:** OBS-0001 IT-2 — see OBS-0001 for full implementation spec.
**Milestone:** M5

**T28** | observability | `templates/commands/operate.md`
**Source:** OBS-0001 IT-3 — see OBS-0001 for full implementation spec.
**Milestone:** M5

**T29** | observability | `templates/commands/status.md`
**Source:** OBS-0001 IT-4 — see OBS-0001 for full implementation spec.
**Milestone:** M5

**T30** | observability | `templates/commands/operate.md`
**Source:** OBS-0001 IT-5 — see OBS-0001 for full implementation spec.
**Milestone:** M5

**T31** | observability | `templates/commands/operate.md`
**Source:** OBS-0001 IT-6 — see OBS-0001 for full implementation spec.
**Milestone:** M5

**T32** | observability | `templates/commands/operate.md`
**Source:** OBS-0001 IT-7 — see OBS-0001 for full implementation spec.
**Milestone:** M5

**T33** | functional | `.claude/commands/review-forgekit-pr.md`
Extend Tier 1 structural checks to verify every command with a confirmation Handoff step appears in `operate.md`'s categorization table (§3.3), per DESIGN-0001 §8/ADR-0001's Consequences — catches categorization-table drift at review time.
**AC:** *(supporting — no direct AC; closes ADR-0001's accepted trade-off)*

**T34** | functional | `templates/PIPELINE.md`
Document `/operate` in the pipeline overview, per DESIGN-0001 §8.
**AC:** *(supporting — no direct AC; documentation)*

## Milestones

### M1 — Structured Review Outcomes & Status Resolver
**Tasks:** T1, T2, T3, T4, T5, T6, T7, T8, T9, T10, T11, T12, T13
**Implementation status:** pending
**AC Coverage:** AC-03, AC-04, AC-05, AC-06, AC-08 (partial — lineage convention only; full AC-08 behavior lands at M4), AC-22
**Depends on:** —
**Rollback:** Safe — every change is an additive frontmatter field or an additive output block; no existing field is removed or renamed, no existing command behavior changes for a human invoking these commands directly.

### M2 — Implementation Tracking, BUG Canonicalization & Test Classification
**Tasks:** T14, T15, T16, T17
**Implementation status:** pending
**AC Coverage:** AC-10, AC-12
**Depends on:** —
**Rollback:** Safe — additive fields and a vocabulary reconciliation (correcting which values are written, not changing the field itself); no schema removal.

### M3 — Release Approval Binding & Resumability
**Tasks:** T18, T19
**Implementation status:** pending
**AC Coverage:** AC-14, AC-21
**Depends on:** —
**Rollback:** Safe — additive fields on the existing `approval:` block and a new `deployment_progress` field; `execute-release.md`'s actual production-deploy steps are unchanged.

### M4 — Operator Core: Command Execution Cycle & Safety Mechanisms
**Tasks:** T20, T21, T22, T23, T24, T25
**Implementation status:** pending
**AC Coverage:** AC-01, AC-02, AC-07, AC-09, AC-11, AC-13, AC-15 (+ AC-15a–e), AC-16, AC-17, AC-18, AC-19, AC-20, AC-23, AC-25, AC-26, AC-27
**Depends on:** M1, M2, M3 (reads `resolved_state`, `implementation_execution`, BUG vocabulary, `classification`, and the release-approval binding fields each of those milestones produces)
**Rollback:** Safe but higher-impact than M1–M3 — `operate.md` and the `operator:` policy block are both new; rolling back means simply not shipping/enabling `/operate` (every `allow.*` key already defaults to `false`), with zero effect on any existing command's behavior when invoked directly by a human.

### M5 — Operator Observability & Supporting Tooling
**Tasks:** T26, T27, T28, T29, T30, T31, T32, T33, T34
**Implementation status:** pending
**AC Coverage:** AC-24
**Depends on:** M4 (every instrumentation task adds a field write at a point M4's logic already reaches)
**Rollback:** Safe — purely additive logging/instrumentation and documentation; no change to Operator's actual decision logic.

## Sequencing & Dependencies

1. **M1** must land first — every later milestone's `/operate` logic (M4) depends on `latest_round`, `recurs_from`, and `/status`'s `resolved_state` already existing and being readable; without them, `/operate` would have nothing non-prose to resolve state from (DESIGN-0001 §3.5's own fallback rule exists specifically for artifacts that predate this work, not as a substitute for it).
2. **M2** and **M3** have no dependency on each other or on M1 beyond ARCH.md's general conventions — both could in principle run in parallel, but are sequenced after M1 since M1 establishes the `resolved_state`/gate-reading pattern M2/M3's own fields are consumed through.
3. **M4** depends on M1 (reads `resolved_state`), M2 (reads `implementation_execution`, BUG vocabulary, `classification`), and M3 (reads release-approval binding) — it is the first milestone where all the prerequisite control-surface fields get consumed by actual orchestration logic, not just produced.
4. **M4's own internal order**: T20 (policy block) and T21 (OPRUN README) have no code dependency and can be done first or in parallel; T22 (core cycle + OPRUN schema) must land before T23 (overlay rule, which records into OPRUN) and T24 (resumption containment, which re-reads OPRUN); T25 (multi-repo) depends on T22's core cycle existing to extend.
5. **M5** depends entirely on M4 — every instrumentation task (T26–T32) adds a field write at a point M4's logic already reaches; none of them are meaningful before M4 exists. T33/T34 have no code dependency on M4 and could technically move earlier, but are grouped with M5 since they're both "supporting the finished Operator," not blocking anything.

This order follows ARCH.md's general phase guidance (schema/infrastructure before consuming logic) adapted to a markdown-command system: here "schema" means the structured frontmatter fields (M1–M3), and "consuming logic" is `operate.md` itself (M4–M5).

## Happy Path Trace

### User Story — "As a developer operating a ForgeKit work unit, I want to invoke one command and have Operator determine and perform the next valid ForgeKit action..."

| Flow Step | Covered by |
|---|---|
| Developer invokes `/operate SPEC-0042` | T22 — Command Execution Cycle entry point |
| Operator resolves current state | T13 — `/status`'s `resolved_state` block; T11/T12 — pre-PLAN and milestone-branch coverage |
| Operator determines the next permitted action from `resolved_state.next_command` | T22 — Stage A/B of the gate-to-outcome procedure |
| Operator checks whether this action is autonomy-eligible | T23 — Autonomy Overlay Rule + categorization table |
| Operator invokes the sub-command (e.g. `/create-design`) and it reaches its own Handoff pause | T23 — self-answers if `operator.allow.create_artifacts: true`, else stops |
| Operator verifies the outcome from resulting artifacts, re-resolves state | T22 — Stage C of the gate-to-outcome procedure |
| Operator records the step in `OPRUN-NNNN`'s `history:` | T22 (core write) + T26/T27/T28 (the `actor`/`escalation.reason`/`oprun_lookup_result` enrichments) |
| Cycle repeats until a stopping condition is reached | T22 (Stage A/B/C classify every stopping case) |

Gap check: none — all steps covered.

## Alternate/Failure Path Trace

### User Story — same as above — Alternate Path: Non-Convergence (SPEC-0001 §39 Scenario)

| Flow Step | Covered by |
|---|---|
| A review round raises a blocking finding | T1–T5 — `latest_round` makes the finding count structurally readable |
| Operator invokes the normal revision workflow (`/address-*-review`) if policy permits | T23 — Autonomy Overlay Rule, `revise_artifacts` category |
| The same finding recurs in a later round | T6–T10 — `recurs_from` lineage makes recurrence detectable without prose interpretation |
| The recurrence count reaches `operator.max_review_rounds` | T22 — `review_loop_counts` tracking in `OPRUN-NNNN` |
| Operator stops with `HUMAN_DECISION_REQUIRED` rather than bypassing the gate | T22's gate-to-outcome procedure, Stage B |
| The decision package identifies the finding lineage and prior attempts | T22 (core Human Decision Package rendering) |

Gap check: none — all steps covered.

## Out of Scope for This Plan

- **Unattended/scheduler-triggered Operator re-invocation** — excluded per DESIGN-0001 §48/Non-Goals; `/operate` remains human-reinvoked only.
- **A literal `satisfied: true` XDEP cache** — SPEC §22/AC-16 explicitly keep XDEP satisfaction live-derived in v1; no caching task exists in this plan.
- **Production deployment automation beyond what `execute-release.md` already does** — SPEC §30/§33 don't require this for v1 success; T19 only adds resumability tracking to the existing command, nothing new in the production-execution path itself.
- **A dedicated AC for the Operator-invocation-recognition mechanism** (ANALYSIS-0001 `[LD-13]`, non-blocking) — deferred; T22/T23 already implement the mechanism SPEC §41 requires, and this plan's own Happy Path/Alternate Path traces exercise it without a dedicated AC.
- **An audit of all ~50 commands for the single-combined-pause-with-gate-content pattern** (Design Review Round 4 `[LD-3]`, non-blocking) — out of scope for this plan; `security-review.md` (the one instance found) is fixed by T23. A future audit, if warranted, is a separate thread of work.

## Definition of Done

| AC | Description | Covered by | Status |
|---|---|---|---|
| AC-01 | Given a valid ForgeKit SPEC with sufficient information, when the user invokes Operator, Operator determines the current state and begins the next permitted ForgeKit activity without requiring the user to identify the next command. | T22 | Covered |
| AC-02 | Given work that already has approved upstream artifacts, Operator begins at the first incomplete or invalid downstream stage and does not recreate completed valid artifacts. | T22 | Covered |
| AC-03 | Operator and `/status` derive lifecycle state from the same resolver and cannot independently disagree about the next valid action. | T13 | Covered |
| AC-04 | `/status` can resolve SPEC, ANALYSIS, DESIGN, OBSERVABILITY, and SECURITY state before PLAN exists. | T11 | Covered |
| AC-05 | `/status` and Operator correctly distinguish all existing milestone states including `pending`, `in_progress`, `approved`, `test_failed`, `tests_passed`, `pr_open`, and `pr_merged`. | T12 | Covered |
| AC-06 | Operator determines blocking-finding counts and review verdicts from structured artifact data rather than parsing prose. | T1–T5 | Covered |
| AC-07 | Given a review containing blocking findings that can be resolved autonomously, Operator invokes the existing correction workflow, re-runs review, and advances only after the blocking count reaches zero. | T23 | Covered |
| AC-08 | Given a recurring or oscillating blocking finding that reaches the configured autonomous limit, Operator stops with `HUMAN_DECISION_REQUIRED` and does not bypass the gate. | T6–T10, T22 | Covered |
| AC-09 | Given an Operator session terminating between lifecycle commands, a new session can invoke Operator for the same work and resume from authoritative ForgeKit state without conversation history. | T22, T24 | Covered |
| AC-10 | Given `/implement-plan` terminating before implementation completes, a resumed Operator does not proceed directly to implementation review until ForgeKit can establish that implementation execution completed. | T14, T22 | Covered |
| AC-11 | Given a command completed before an Operator interruption, resuming does not duplicate the artifact, review, branch, thread, or PR. | T22 | Covered |
| AC-12 | Given a test failure requiring QE classification, Operator stops until a structured `production_bug` or `test_bug` decision exists and then follows the appropriate existing ForgeKit path. | T15–T17 | Covered |
| AC-13 | Given an existing ForgeKit human approval gate, Operator cannot satisfy, bypass, or remove the approval itself. | T22, T23 | Covered |
| AC-14 | Given a release plan that changes after approval, Operator does not execute release using approval bound to the previous candidate or release-plan revision. | T18 | Covered |
| AC-15 | Given a THREAD containing multiple participant repositories, Operator resolves participant state, XDEPs, Interface Freeze, plans, milestones, cross-repo validation, and release eligibility without treating an individual participant's completion as thread completion. | T25 | Covered |
| AC-15a | Given a THREAD with multiple participants, Operator resolves each participant's lifecycle state and each XDEP's live satisfaction independently, with no reliance on a cached satisfaction value. | T25 | Covered |
| AC-15b | Given a THREAD with an unfrozen or revised Cross-Repo Contract, Operator does not advance implementation work that depends on a contract revision that has not been approved. | T25 | Covered |
| AC-15c | Given a THREAD, Operator determines each participant's PLAN/TEST-PLAN readiness independently and does not advance one participant's implementation on the basis of another participant's readiness. | T25 | Covered |
| AC-15d | Given a THREAD requiring cross-repo scenario validation, Operator does not consider thread-wide implementation complete until the required cross-repo scenarios resolve to a PASS-equivalent outcome. | T25 | Covered |
| AC-15e | Given a THREAD, Operator derives thread-wide release eligibility from the combination of AC-15a through AC-15d rather than from any single participant's state. | T25 | Covered |
| AC-16 | Operator determines blocking XDEP satisfaction from authoritative participant state rather than relying on a persisted satisfaction cache. | T22 | Covered |
| AC-17 | Given mutually blocked participants with no eligible action, Operator detects the deadlock, stops, identifies the dependency cycle, and does not spin or arbitrarily break it. | T25 | Covered |
| AC-18 | Given a single-repository SPEC whose ANALYSIS or DESIGN discovers another required participant, Operator invokes the existing promotion workflow and continues using the resulting THREAD as authoritative work. | T25 | Covered |
| AC-19 | Given a PR requiring external review or merge, Operator enters an appropriate waiting/external-action state and does not create duplicate PRs or assume merge occurred. | T22 | Covered |
| AC-20 | Given policy requiring human production approval or execution, Operator advances work to the appropriate production-ready state and stops before the protected operation. | T22 | Covered |
| AC-21 | If Operator is configured to invoke release execution, interruption after an irreversible release step can be safely resumed without depending on conversational memory. | T19 | Covered |
| AC-22 | Given ForgeKit state that does not match a recognized lifecycle transition, Operator stops with an explicit ambiguous-state outcome rather than guessing. | T13 | Covered |
| AC-23 | All engineering reasoning and artifact generation continues through existing ForgeKit commands. Operator does not create parallel implementations of DESIGN, PLAN, review, testing, implementation, or release reasoning. | T22 | Covered |
| AC-24 | After an Operator run, sufficient orchestration data exists to calculate autonomous actions, human interventions, review iterations, retries, recovery events, gate compliance, and failure categories. | T26–T32 | Covered |
| AC-25 | Given a work unit with no Autonomy Policy configured, Operator treats every otherwise-autonomous action as requiring human confirmation rather than assuming permission. | T20 | Covered |
| AC-26 | Given `/operate` invoked against a work-unit id that does not exist, or one that exists but has no further eligible lifecycle action (e.g., already `promoted` or `shipped` with nothing downstream pending), Operator reports an explicit not-found or already-complete outcome rather than an ambiguous or silent no-op. | T22 | Covered |
| AC-27 | Given a point in the lifecycle where a security exception or policy exception would be required to proceed (e.g., a SEC finding requiring an `accepted_risk` disposition), Operator stops with `HUMAN_DECISION_REQUIRED` and does not record or imply a risk-acceptance decision on its own behalf. | T23 | Covered |

Every AC is covered by this milestone set. No AC is deferred.

## Plan Review — Round 1

**Date:** 2026-10-07
**Status:** Changes requested — 3 blocking, 8 non-blocking

### SRE Findings

- [SRE-1] **non-blocking** — T16 (BUG vocabulary reconciliation) corrects what `triage-bug.md`/`resolve-incident.md`/`escalate-to-incident.md` write *going forward*; pre-existing `BUG-NNNN` files already using an inconsistent value aren't addressed. Unlike DESIGN-0001 §3.5's explicit fallback for pre-existing artifacts missing a *field*, there's no equivalent note here for pre-existing artifacts with an *inconsistent value already written*. Likely low-impact (BUG/INCIDENT aren't part of `resolved_state` computation), but worth a one-line acknowledgment rather than silence.
- [SRE-2] **non-blocking** — T20 adds a new `operator:` block to `project.yml.template`, but nothing in this PLAN addresses how an *already-initialized* project (including this very ForgeKit repo) gets that block added to its own `project.yml`. `/upgrade-pipeline`'s documented scope (per `forge-init.md`) is limited to `.claude/commands/`/`.claude/roles/` refresh — it does not mention `project.yml.template` propagation at all. Not a safety gap (every `allow.*` key already defaults to `false` when the block is absent entirely, so fail-closed holds either way) — but it is a usability gap with no documented path today. Recommend either a task extending `/upgrade-pipeline` to also diff `project.yml.template`, or an explicit `Out of Scope` entry acknowledging this is deferred.

### Developer Findings

- [DEV-1] **blocking** — T25 (multi-repo orchestration) fails the implementation-ambiguity bar: its own description says "per DESIGN-0001 §21–§24 *concepts*, carried from SPEC §21–§24" — concepts, not a specified procedure. Unlike T22's gate-to-outcome mapping (§3.6 gives an exact three-stage procedure) or T23's overlay rule (exact classification text), DESIGN never gave multi-repo resolution/deadlock-detection the same mechanical treatment — it states the invariants (SPEC §21–25) without a "how." A developer picking up T25 would have to make real design-level judgment calls about how participant/XDEP/deadlock resolution actually works mechanically. This was already self-flagged in `/create-plan`'s own Quality Gate self-check; raising it here as the formal finding it deserves.
- [DEV-2] **non-blocking** — M5 bundles two distinct capabilities: Operator observability instrumentation (T26–T32, 7 tasks, one coherent concern) and unrelated supporting tooling (T33 Tier 1 review extension, T34 PIPELINE.md docs). 9 tasks total is borderline on size, but the bigger issue is the two capabilities aren't actually one thing — recommend splitting into M5 (observability, T26–T32) and M6 (supporting tooling, T33–T34).
- [DEV-3] **non-blocking** — M1's 13 tasks (T1–T13) are well over the usual 8–10 guideline, though `/create-plan`'s own Step 7 guidance explicitly exempts M1 as "the foundation... does not slice cleanly." Even with that exemption, 13 tasks in one PR is a lot to review — the milestone does cleanly decompose into three sub-concerns (`latest_round` × 5 commands, `recurs_from` × 5 commands, `/status` extension × 3 tasks). Not requiring a split given the stated exemption, but worth confirming that exemption is actually the intended reading rather than an oversized milestone slipping through under its cover.

### QE Findings

- [QE-1] **blocking** — AC-10 ("a resumed Operator does not proceed directly to implementation review until ForgeKit can establish that implementation execution completed") is a write-then-read guarantee: T14 covers the *write* path (`implement-plan.md` setting `implementation_execution`), but the Definition of Done maps AC-10 to T14 alone — no task is cited for the *read* path (Operator checking `implementation_execution.state`/`completed_at` on resume and routing to recovery instead of review, per DESIGN-0001 §3.4's "Interrupted-implementation recovery logging" paragraph). T22's description doesn't name this behavior explicitly either. Per the write/read-path rule: a test can verify the field gets written, but nothing currently named verifies Operator actually *acts* on it. Recommend citing T22 (or a new sub-task) against AC-10 and naming the recovery-routing behavior explicitly in its description.
- [QE-2] **non-blocking** — T22 is a compound task bundling several independently-testable behaviors (core cycle orchestration, full `OPRUN-NNNN` read/write mechanics, lookup-before-create, the three-stage gate-to-outcome procedure). All four are tightly coupled in the same file (`operate.md`), so this is a style judgment, not a hard requirement — but splitting would make each independently verifiable rather than requiring one large fixture to exercise all of T22 at once.
- [QE-3] **non-blocking** — The Alternate/Failure Path Trace satisfies `/create-plan`'s "at least one" requirement (non-convergence, SPEC §39), but this feature has at least two other distinct terminal-outcome mechanisms worth equal confidence: dependency deadlock (AC-17, SPEC §40) and the production-boundary stop (AC-20). Given `[DEV-1]`'s finding that T25 (which covers the deadlock case) is under-specified, a second alternate-path trace through the deadlock scenario specifically would likely have surfaced `[DEV-1]` even without the Developer perspective's separate catch — recommend adding it.

### Security Findings

- [SEC-1] **non-blocking** — T23 implements both `SEC-0001 SEC-1`'s fix (cited) and `SEC-0001 SEC-2`'s fix (the `findings_status` content-check mechanism — not cited). The mechanism is genuinely present in T23's description, so this isn't "no task maps to the mitigation" in the strict blocking sense, but `SEC-2`'s citation is missing entirely from the PLAN — a future audit scanning for which task resolved `SEC-2` would find nothing. Recommend adding `SEC-0001 SEC-2` to T23's Source line.

### Architect Findings

- [ARCH-1] **non-blocking** — None of T13/T20/T22/T23 cite their corresponding ADR (`ADR-0002`, `ADR-0003`, `ADR-0001`, `ADR-0001`/`ADR-0003` respectively) by ID — only DESIGN section numbers. Equally precise in practice, but Rule 5's explicit guidance is to cite the ADR directly when a task implements its subject matter. Cheap to add.
- [ARCH-2] **blocking** — T22's "including lookup-before-create (resolves `[DBA-1]`)" names the mechanism but never states the actual invariant it protects (never create a second `OPRUN-NNNN` for a `work_unit` that already has a non-terminal one) in the task's own words — unlike T20, T23, and T24, which all state their invariants explicitly inline. Per the Architect Perspective's own bar: "a task whose literal wording could be satisfied while violating the invariant is a blocking finding." A developer could satisfy "implement lookup-before-create" with a lookup that occasionally races or misses, without the task text itself making the actual guarantee clear.

**Clean perspectives:** none this round — every perspective raised at least one finding.

### Contradictions Requiring Arbitration

None. One cascade: `[DEV-1]`'s finding (T25 under-specified) is the same root cause `[QE-3]` independently arrived at via the alternate-path-trace angle — both point at the same gap from different directions, reinforcing rather than contradicting each other.

## Address Summary — Round 1

**Date:** 2026-10-07

### Addressed
- `[DEV-1]` — T25 rewritten to compose already-specified mechanisms instead of gesturing at "concepts": each AC-15 sub-criterion now names the exact existing field/mechanism it reduces to (T22's own resolver reused per-participant, the existing XDEP satisfaction check, the existing `cross_repo_contract` revision fields, the existing 4-value scenario-result enum), and AC-17's deadlock detection is resolved by reusing `create-design.md`'s own existing design-time cycle-detection check, applied at runtime instead — no new DESIGN decision was needed; this was a PLAN-level specificity gap, not an actual missing design decision, once traced back to what already exists.
- `[QE-1]` — T22 now explicitly names the read-side behavior AC-10 requires (checking `implementation_execution.state`/`completed_at` on resume, routing to recovery instead of review) as its own stated invariant, and the DoD table's AC-10 row now cites T14 (write) and T22 (read + routing) together.
- `[ARCH-2]` — T22's lookup-before-create bullet now states the actual invariant explicitly ("never create a second `OPRUN-NNNN` for a `work_unit` that already has a non-terminal one") rather than only naming the mechanism.
- `[SEC-1]` — fixed directly as a clear citation addition, no judgment involved: T23's Source line now also cites `SEC-0001 SEC-2`.
- `[ARCH-1]` — fixed directly as a clear citation addition: T13, T20, T22, and T23 now each cite their corresponding ADR(s) alongside the existing DESIGN section references.

A consequence of resolving `[DEV-1]`: AC-18 (promotion during discovery) moved from T22's AC list to T25's, since promotion-on-discovery is thematically part of T25's multi-repo scope and T25's revision now explicitly names it — the Definition of Done's AC-18 row updated to cite T25 instead of T22.

### Not Addressed
- `[SRE-1]` — **accepted** — pre-existing BUG files with inconsistent values aren't migrated; low-impact since BUG/INCIDENT state isn't part of `resolved_state` computation, and no field is missing, just a value divergence a human could spot if it ever mattered.
- `[SRE-2]` — **accepted** — `/upgrade-pipeline`'s scope not extended to `project.yml.template` propagation; not a safety gap (fail-closed holds via the absent-block default regardless), and extending that command's scope is a separate concern from this feature.
- `[DEV-2]` — **accepted** — M5's two capabilities (observability instrumentation, supporting tooling) stay bundled; splitting is a reasonable alternative but not required, and 9 tasks is only borderline over guideline.
- `[DEV-3]` — **accepted** — M1's 13 tasks stand; `/create-plan`'s own stated exemption for foundation milestones applies, and the three sub-concerns within it are independently coherent even if bundled into one PR.
- `[QE-2]` — **accepted** — T22 stays as one compound task; splitting is a style preference for a single tightly-coupled file, not required for testability.
- `[QE-3]` — **accepted** — only one Alternate/Failure Path Trace is required by `/create-plan`'s own "at least one" rule; adding a second (dependency deadlock) would be nice-to-have, not addressed this round. Partially moot in substance, since `[DEV-1]`'s fix to T25 already makes the deadlock mechanism concrete even without its own trace table.

## Plan Review — Round 2

**Date:** 2026-10-07
**Status:** Changes requested — 0 blocking, 1 non-blocking

**SRE:** no new findings — domain unaffected by Round 1's address (`[SRE-1]`/`[SRE-2]` were both accepted as-is; no section SRE would re-examine was touched).

Confirmed resolved on substance: `[DEV-1]` (T25's rewrite genuinely composes existing mechanisms — the "reuse `create-design.md`'s own cycle-check, applied at runtime" choice was checked against ARCH.md's own ADR-worthiness criteria and doesn't rise to a Key Decision requiring DESIGN sign-off: it's traceable, easily reversible, and constrains nothing downstream), `[QE-1]` (T22's recovery-routing invariant is now stated and testable), `[ARCH-2]` (T22's lookup-before-create invariant is now explicit), `[SEC-1]` and `[ARCH-1]` (citations confirmed present and correct).

### QE Findings

- [QE-4] **non-blocking** — T25's rewrite, while now concrete, surfaced the same compound-task shape `[QE-2]` already flagged for T22: it bundles a one-time state-transition trigger (AC-18, promotion during discovery) with ongoing multi-repo state resolution (AC-15/16/17). Not blocking for the same reason `[QE-2]` wasn't — tightly coupled within one file, each AC still independently testable even under one task ID — but worth noting alongside `[QE-2]` as the same pattern recurring.

**Clean perspectives:** Developer, Security, Architect.

### Contradictions Requiring Arbitration

None.

## Address Summary — Round 2

**Date:** 2026-10-07

### Not Addressed
- `[QE-4]` — **accepted** — same compound-task shape already accepted for `[QE-2]` (T22); not required to split, tightly coupled within one file, each AC independently testable even under one task ID.
