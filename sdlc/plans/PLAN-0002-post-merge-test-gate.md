---
id: PLAN-0002
type: plan
schema_version: 1
title: "Configurable Post-Merge Test Gate — Implementation Plan"
status: approved
version: 3
updated: 2026-10-17
changelog:
  - "v3 — approved after 3 review rounds"
target_design_version: 2
includes_observability_tasks: true
includes_security_tasks: false
milestone_count: 1
current_review_round: 3
latest_round:
  round: 3
  blocking: 0
  non_blocking: 0
  verdict: ready
generation:
  command: /create-plan
  command_version: "5"
  model: "claude-sonnet-5"
  generated_at: "2026-10-17"
  session_id: null
pipeline_status:
  spec: SPEC-0002 approved
  analysis: ANALYSIS-0002 completed
  design: DESIGN-0002 approved
  obs: OBS-0002 approved
  sec: not triggered
  plan: approved
  test: approved
approval:
  mode: repo_only
  approved_by: "Eric Pizon"
  approved_role: lead-developer
  approved_at: "2026-10-17"
  decision_summary: "PLAN-0002 (Configurable Post-Merge Test Gate) approved for test planning phase after 3 review rounds — all blocking findings resolved."
  external_event_id: null
  external_event_uri: null
policy_evaluation:
  evaluated: null
  policy_pack: null
  result: null
---

# PLAN-0002 — Configurable Post-Merge Test Gate

ANALYSIS-0002's own Scope Coherence Check (`[SC-1]`) confirmed no subset of this feature ships independently with real value — a config field with no consuming logic is useless, and the failure-response/clean-closure behaviors only matter once the merge-now path itself exists. This PLAN is therefore a single milestone, not a vertical-slice sequence.

## Task Breakdown

**T1** | functional | `templates/project.yml.template`
Add `pipeline.test_gate: pre_merge | post_merge` (default `pre_merge`; absent or malformed → treated as `pre_merge`, fail-closed per ADR-0003's own established convention, DESIGN-0002 §3.4/§3.6) and `stack.qa_trigger_command: "<command>" | "none"` (default `"none"`, only meaningful under `post_merge`) to the template.
**AC:** AC-04 *(supporting — establishes the fail-closed-default schema every other task's `post_merge` branch is conditioned on; AC-01/AC-05/AC-08 each depend on this field existing)*

**T2** | functional | `templates/commands/forge-init.md`
Add the `test_gate` workflow question (DESIGN-0002 §3.5): asks which mode to use, states the `post_merge` trade-off plainly — code reaches `main`/QA before `/execute-tests` confirms it — not just the two mode names, defaults to `pre_merge` if the human has no strong preference.
**AC:** AC-05

**T3** | functional | `templates/commands/approve-implementation.md`
Add the `post_merge` branch to the Handoff (DESIGN-0002 §3.3, resolves Design Review Round 1 `[LD-1]`; mechanism per ADR-0004):
- Under `pipeline.test_gate: post_merge`, the Handoff offers to merge the milestone branch to `main` directly, instead of pointing to `/execute-tests`. If declined, the milestone stays at `approved`, re-offered next time — the same recoverable pattern as any other declined confirmation in the pipeline.
- **Team path, retry-safe (resolves Plan Review Round 1 `[ARCH-1]`):** before calling `gh pr create`, check for an existing open PR for this milestone branch (e.g. `gh pr list --head <branch>` or `gh pr view <branch>`). If one already exists (a retry after a prior blocked attempt), reuse it and attempt `gh pr merge` on it directly — never call `gh pr create` again for a branch that already has an open PR. If none exists, `gh pr create`, then attempt `gh pr merge` immediately in the same step.
- On confirm: **solo** → direct fast-forward merge to `main` (identical mechanics to `create-pr.md`'s existing local-only path). **Team** → the retry-safe sequence above.
- **Verify the merge actually completed** (`gh pr view --json state`) before writing anything — never trust `gh pr merge`'s exit code alone.
- **Confirmed merged:** set `**Implementation status:** pr_merged` and `post_merge_test: {status: pending, updated_at: <now>}` in the same step.
- **Not merged** (a required status check or additional approval blocked it): leave the PR open, keep `**Implementation status:** approved`. Tell the human what blocked it **as a sanitized category** (resolves Plan Review Round 1 `[ARCH-2]`) — e.g. `required_status_check_pending`, `required_review_missing`, `branch_protection_rule` — derived from `gh`'s output, never that raw CLI output itself, per ARCH.md's "Failure-diagnostic sanitization" convention (raw connector output is never shown or committed verbatim). A re-run finds the existing open PR via the same check above and retries rather than creating a duplicate.
**AC:** AC-01, AC-06, AC-08 *(partial — writes the initial `pending` value; the terminal `passed`/`failed` write is T5)*

**T4** | observability | `templates/commands/approve-implementation.md`
**Source:** OBS-0002 IT-1 — see OBS-0002 for full implementation spec. `post_merge_merge_attempt.blocked_reason` is written as the same sanitized category T3 defines above (resolves Plan Review Round 1 `[ARCH-2]`) — never `gh`'s raw CLI output.
**Milestone:** M1

**T5** | functional | `templates/commands/execute-tests.md`
Add the post-merge validation branch (DESIGN-0002 §3.4/§3.5/§3.6):
- **Pre-flight:** if the target milestone already has a `post_merge_test` block present (merged under `post_merge` — checked before reading the project's *current* `pipeline.test_gate` setting, per OQ-5's per-milestone-authority resolution), skip the branch-scoped Pre-flight entirely and validate the merged state instead.
- **Validation target:** run `stack.qa_trigger_command` if configured. Otherwise, fall back to local validation against `main` itself — first sync the local checkout, branched by which path T3 took (resolves Plan Review Round 1 `[SRE-1]` and Round 2 `[SRE-3]`): **team path** → `git checkout main && git pull` (`gh pr merge` happened on GitHub; local `main` is stale and genuinely needs fetching). **Solo path** → `git checkout main` only, no pull (T3's own fast-forward merge already brought local `main` fully current moments earlier; a bare `git pull` would error on a repo with no remote configured). Then run `stack.test.unit`/`stack.test.e2e` against that now-synced checkout.
- **Result write:** set `post_merge_test: {status: passed | failed, updated_at: <now>}`. Only a P0 failure sets `failed` (resolves Design Review Round 1 `[SRE-1]`) — a P1-or-lower failure is recorded in the test execution table as today, without flipping this status. Once written, `status`/`updated_at` are never overwritten for that milestone again — a fix for a post-merge regression lands as a new milestone or hotfix, tracked on its own entry.
- **Failure routing:** a P0 failure routes to `/create-bug` (or `/escalate-to-incident` if the affected environment is shared/blocking) — never to `test_failed`, which assumes an unmerged, revertible feature branch that no longer exists at this point.
- **Infra-failure distinction:** a `stack.qa_trigger_command` invocation failure (network/auth, not a test result) is surfaced directly to the user as an infrastructure problem and never written to `post_merge_test.status` — that field must mean "tests ran and found a real problem," not "the trigger mechanism was broken."
**AC:** AC-02, AC-03, AC-08 *(terminal `passed`/`failed` write — the other half of AC-08, started by T3)*

## Milestones

### M1 — Configurable Post-Merge Test Gate
**Tasks:** T1, T2, T3, T4, T5
**Implementation status:** tests_passed
**implementation_execution:** {state: completed, attempt: 1, started_at: "2026-10-18", completed_at: "2026-10-18"}
**AC Coverage:** AC-01, AC-02, AC-03, AC-04, AC-05, AC-06, AC-07 (no task — see Definition of Done), AC-08
**Depends on:** —
**Rollback:** Safe — every task's change is additive and conditional on `pipeline.test_gate: post_merge`; the `pre_merge` path (today's only behavior, and the default) is never touched by any task's own bullets, confirmed individually in T1/T3/T5's text. Rolling back means simply not shipping/enabling the `post_merge` branch, with zero effect on any project that hasn't opted in.

### Test Execution — Round 1
Date: 2026-10-18
Mode: local (fixture trace — no test runner exists for this project; `stack.test.unit`/`stack.ci_trigger_command` are both `none`. Per ARCH.md Tier 2, each case is traced directly against the actual committed command text in `dd1acdc`, with git-state cases — `TC-FUNC-13`'s `implement-plan.md` non-change — confirmed via `git diff` rather than by inspection alone.)
Executed by: /execute-tests PLAN-0002 M1 TEST-0002

| Test Case ID | Priority | Status | Classification | Notes |
|---|---|---|---|---|
| TC-FUNC-1 | P0 | PASS | — | Step 9's `post_merge` branch offers the merge, never mentions `/execute-tests` first; solo confirm runs the fast-forward mechanics exactly as specified. |
| TC-FUNC-2 | P0 | PASS | — | Declined offer leaves `**Implementation status:**` at `approved`; same offer re-made next Handoff — confirmed verbatim in Step 9. |
| TC-FUNC-3 | P0 | PASS | — | `qa_trigger_command` configured → run directly, per Validation Target's first branch. |
| TC-FUNC-4 | P0 | PASS | — | Team path local fallback: `git checkout main && git pull` runs before the test commands. |
| TC-FUNC-5 | P0 | PASS | — | Solo path local fallback: `git checkout main` only, no pull — text explicitly states why (already current, no remote guaranteed). |
| TC-FUNC-6 | P0 | PASS | — | Non-shared/blocking P0 failure routes to `/create-bug`, never `test_failed`. |
| TC-FUNC-7 | P0 | PASS | — | Shared/blocking P0 failure routes to `/escalate-to-incident` instead. |
| TC-FUNC-8 | P0 | PASS | — | Confirmed via `git diff dd1acdc~1 dd1acdc` — the `pre_merge` branch's message text, CI Detection, Results Recording, and pass/fail sections are all byte-unchanged; only new sections were inserted. |
| TC-FUNC-9 | P0 | PASS | — | Phase 4C: "Before merging *(default)*" — no-preference path writes `pre_merge`. |
| TC-FUNC-10 | P1 | deferred-to-release | — | Manual — requires reading the actual rendered question text for plain-language clarity; not mechanically checkable (per its own `Reason:` line). |
| TC-FUNC-11 | P0 | PASS | — | Solo: `pr_merged` set directly in the merge step; `/create-pr` never invoked. |
| TC-FUNC-12 | P0 | PASS | — | Team: same, plus `/close-milestone` also confirmed never invoked. |
| TC-FUNC-13 | P0 | PASS | — | `git diff dd1acdc~1 dd1acdc -- templates/commands/implement-plan.md` returns empty — zero lines changed, confirming the "no change required" claim directly rather than assuming it. |
| TC-FUNC-14 | P0 | PASS | — | `**Implementation status:** pr_merged` and `post_merge_test: {status: pending, ...}` written together under "Once the merge is confirmed." |
| TC-FUNC-15 | P0 | PASS | — | All-P0-pass sets `status: passed`, per Writing `post_merge_test`. |
| TC-FUNC-16 | P0 | PASS | — | P1-only failure leaves `status: passed`; a P0 failure sets `status: failed` — both stated explicitly. |
| TC-FUNC-17 | P0 | PASS | — | Pre-flight Check stops re-entry once `post_merge_test.status` is no longer `pending`. |
| TC-FUNC-18 | P0 | PASS | — | Both `project.yml.template`'s field comment and Step 9's branch condition state the fail-closed default for absent/malformed values. |
| TC-FUNC-19 | P0 | PASS | — | Pre-flight Check routes on MN's own `post_merge_test` presence, checked ahead of the project's current `pipeline.test_gate` value — per-milestone authority holds by construction. |
| TC-FUNC-20 | P0 | PASS | — | Phase 4C's "After merging" branch is selectable and recorded as `pipeline.test_gate: post_merge`. |
| TC-ADR-0004-1 | P0 | PASS | — | Merge verification via `gh pr view --json state`; a non-`MERGED` result leaves `**Implementation status:**` at `approved` with the PR open — `post_merge_test` is never reached in this branch. |
| TC-ADR-0004-2 | P0 | PASS | — | `gh pr list` lookup runs before any `gh pr create` call; an existing open PR is reused, never duplicated. |
| TC-ADR-0004-3 | P0 | PASS | — | `blocked_reason` is one of three named sanitized categories, explicitly never `gh`'s raw CLI text — stated identically in both `approve-implementation.md` (T3) and the OBS task (T4). |
| TC-OBS-pipeline_test_gate-1 | P1 | PASS | — | The two modes' exact branch behaviors are both named explicitly in Step 9 and the Pre-flight Check, not merely asserted as "consistent." |
| TC-OBS-post_merge_test_status-1 | P1 | PASS | — | All three values (`pending`/`passed`/`failed`) confirmed via `TC-FUNC-14`/`15`/`16` above. |
| TC-OBS-post_merge_test_updated_at-1 | P2 | PASS | — | Set at both the `pending` write (Step 9) and the terminal write (execute-tests.md) — two real, distinct timestamps by construction. |
| TC-OBS-merge_attempt-1 | P1 | PASS | — | Shape confirmed via `TC-ADR-0004-1`/`-3`; superseded (no separate cleanup write) once a retry succeeds, per Step 9's own note. |
| TC-OBS-qa_trigger_infra_failure-1 | P1 | PASS | — | Infra-failure distinction stops before writing `post_merge_test.status`, surfacing the problem directly instead. |

**P0 results:** 22 passed, 0 failed
**P1+ results:** 5 passed, 0 failed, 1 deferred-to-release

## Sequencing & Dependencies

1. **T1** must land first — every other task reads or writes `pipeline.test_gate` or `stack.qa_trigger_command`, both introduced here. This follows ARCH.md's general phase guidance (schema before consuming logic) adapted to a markdown-command system: here "schema" means these two `project.yml.template` fields, and "consuming logic" is the command-surface changes in T2/T3/T5.
2. **T2** depends on T1's field existing to set via the new workflow question — no code dependency on T3/T5.
3. **T3** depends on T1's schema (reads/writes `pipeline.test_gate`, writes `post_merge_test`'s initial value). No dependency on T2 — a project could have `pipeline.test_gate: post_merge` set directly in `project.yml` even before an adopting project re-runs `/forge-init`.
4. **T4** depends on T3 — it extends the exact merge-verification step T3 implements, in the same file, with the same "Not merged" branch as its write point. These two land together in practice even though they're tagged separately per source (functional vs. observability).
5. **T5** depends on T3 having already written `post_merge_test: {status: pending, ...}` and `**Implementation status:** pr_merged` for a milestone — T5's Pre-flight branch is conditioned on that block's presence.

## Happy Path Trace

### US1 — "As a Lead Developer at an enterprise with a shared QA environment, I want milestone code to merge to `main` as soon as it's reviewed and approved..."

| Flow Step | Covered by |
|---|---|
| Milestone reaches `approved` under `pipeline.test_gate: post_merge` | T1 — schema; T2 — the mode was configured at `/forge-init` |
| `/approve-implementation`'s Handoff offers to merge directly | T3 |
| Human confirms; merge executes (solo fast-forward or team PR+merge) | T3 |
| Merge verified actually completed | T3 |
| `**Implementation status:** pr_merged` and `post_merge_test: {status: pending}` written | T3 |
| Adopting project's own CI/CD deploys from `main` (outside ForgeKit's control, per Non-Goal #2) | — |
| QE runs `/execute-tests`; result ties to the merged commit, never the deleted feature branch | T5 |
| `post_merge_test.status` set to `passed` | T5 |

Gap check: none — all steps covered.

## Alternate/Failure Path Trace

### US1 — Alternate Path: Post-Merge P0 Test Failure (User Story 4)

| Flow Step | Covered by |
|---|---|
| Milestone already merged under `post_merge` (`post_merge_test: pending`) | T3 |
| `/execute-tests` runs against the merged state and finds a P0 failure | T5 |
| Failure routes to `/create-bug` (or `/escalate-to-incident` if shared/blocking) — never `test_failed` | T5 |
| `post_merge_test.status` set to `failed`, `updated_at` set, never later overwritten | T5 |

Gap check: none — all steps covered.

### US1 — Alternate Path: Team-Path Merge Blocked by Branch Protection (ADR-0004)

| Flow Step | Covered by |
|---|---|
| Human confirms the offered merge under `post_merge` (team path) | T3 |
| `gh pr create` succeeds; `gh pr merge` fails or stalls (a required check/approval blocks it) | T3 |
| Merge-verification confirms the PR is not actually merged | T3 |
| `**Implementation status:**` stays at `approved` (never prematurely advanced); PR left open; `post_merge_merge_attempt.blocked` recorded with a reason | T3 + T4 (OBS) |
| Human resolves the blocking requirement and re-runs `/approve-implementation`; the existing open PR is found and retried, not duplicated | T3 |

Gap check: none — all steps covered.

## Out of Scope for This Plan

- **Building an actual QA-environment deployment mechanism** (SPEC Non-Goal #2) — `stack.qa_trigger_command` assumes the adopting project's own CI/CD already deploys from `main`; ForgeKit provisions nothing.
- **A third, hybrid test-gate mode** (SPEC Non-Goal #4) — exactly two modes, no partial variant.
- **Any change to the RELEASE-PLAN/production-release flow** (SPEC Non-Goal #3) — scoped entirely to the per-milestone implementation phase.
- **A per-milestone `test_gate` override** — DESIGN-0002 §3.2 (OQ-1) resolves this as project-level only for v1; the milestone-entry field design leaves room for a future override without restructuring, but no task implements one now.
- **Automatic re-blocking of dependent milestones on a post-merge failure** (OQ-6) — a deliberate human judgment call exercised through the BUG/incident record, not a pipeline gate; no task changes `implement-plan.md`'s dependency check.
- **Coordination-repo-specific handling beyond OQ-2's "per-repo setting" resolution** — ForgeKit's own dogfooding is `repo_role: code`; no thread-level task is needed for this plan.

## Definition of Done

| AC | Description | Covered by | Status |
|---|---|---|---|
| AC-01 | Given a project with `pipeline.test_gate: post_merge`, when a milestone reaches `approved` via `/approve-implementation`, the Handoff offers to merge the milestone branch to `main` directly, without requiring `/execute-tests` first. If the human declines the offer, the milestone remains at `approved` — re-offered the next time. | T3 | Covered |
| AC-02 | Given `pipeline.test_gate: post_merge` and a milestone branch already merged, when `/execute-tests` subsequently runs for that milestone, every result it reports is tied to the merged commit now on `main`, via whatever specific target mechanism the project configures — never a result that could be satisfied merely by the already-deleted feature branch's prior state. | T5 | Covered |
| AC-03 | Given `pipeline.test_gate: post_merge`, when `/execute-tests` finds a P0 failure after the milestone has already merged, the failure routes to `/create-bug` (or `/escalate-to-incident` if the affected environment is shared/blocking) — never to the `test_failed` milestone status. | T5 | Covered |
| AC-04 | Given a project with no `pipeline.test_gate` configured, or explicitly `pre_merge`, every command in the implementation/testing/PR sequence behaves identically to its behavior before this feature existed. | T1 (+ every task's `post_merge`-conditional framing, individually confirmed) | Covered |
| AC-05 | Given `/forge-init` running, the human is asked which `test_gate` mode to use, with `pre_merge` as the default. The question states the `post_merge` trade-off plainly, not just the two mode names. | T2 | Covered |
| AC-06 | Given `pipeline.test_gate: post_merge`, once a milestone merges via `/approve-implementation`'s offered merge, `/create-pr` and `/close-milestone` are never invoked for that milestone — `**Implementation status:**` reaches `pr_merged` directly from the merge step itself. | T3 | Covered |
| AC-07 | Given `pipeline.test_gate: post_merge`, the next milestone's `depends_on_milestones` gate still correctly recognizes the prior milestone as unlocking downstream work once merged — the dependency-gating mechanism must not implicitly assume the `pre_merge`-only status sequence. | *(no task)* — `implement-plan.md`'s existing Pre-flight Check #4 already checks only `pr_merged`, which `post_merge`'s own merge step still sets; confirmed sufficient, with zero change required, in DESIGN-0002 §3.5/§8. | Covered |
| AC-08 | Given `pipeline.test_gate: post_merge`, the milestone's `**Implementation status:**` reaching the terminal `pr_merged` at merge time does not prevent tracking what happens next — a separate, explicit field distinguishes post-merge testing `pending`, `passed`, and `failed`, independently of the already-terminal implementation status. | T3 (writes `pending`), T5 (writes terminal `passed`/`failed`) | Covered |

Every AC is covered by this milestone set. No AC is deferred.

## Plan Review — Round 1

**Date:** 2026-10-17
**Status:** Changes requested — 3 blocking, 2 non-blocking

### SRE Findings

- [SRE-1] **blocking** — T5's fallback validation path ("running `stack.test.unit`/`stack.test.e2e` directly against `main`'s current checked-out state") never states the git-sync step that makes "`main`'s current checked-out state" actually true. Under the team path, after `gh pr merge` succeeds, the *local working tree* is still on the feature branch — nothing in T3 or T5 ever runs `git checkout main && git pull`. As written, a developer implementing T5 literally could run the fallback test commands against a stale checkout and get a result that has nothing to do with what's actually on `main`. This is not cosmetic: per DESIGN-0002 §3.4, once `post_merge_test.status` is written it is *never overwritten* — a false result from a stale checkout becomes a permanent, uncorrectable record for that milestone. Needs: T5 extended to state the explicit `git checkout main && git pull` (or equivalent) step before invoking the fallback test commands.
- [SRE-2] **non-blocking** — T4's `Source:` line cites `OBS-0002` rather than `OBS-0002 IT-1`, the format `create-plan.md`'s own Step 3 requires (and `OBS-0001`/`PLAN-0001`'s own established precedent uses). No real ambiguity results, since `OBS-0002` has only the one instrumentation task, but the citation format itself is inconsistent with convention.

### QE Findings

- [QE-1] **non-blocking** — confirms `AC-07` traceability was deliberately checked, not silently skipped: `implement-plan.md`'s existing `depends_on_milestones` gate already checks only `pr_merged`, which `post_merge`'s own merge step (T3) still sets, so no PLAN task is needed — exactly as DESIGN-0002 §3.5/§8 confirms explicitly. No gap.

### Architect Findings

- [ARCH-1] **blocking** — Invariant-preservation gap. `ADR-0004`'s Decision explicitly establishes that a retried merge "finds the existing open PR and retries rather than creating a duplicate one" — an invariant, not a nice-to-have. T3's task text asserts this same outcome ("a re-run finds the existing open PR and retries rather than creating a duplicate") but never states the actual mechanism that makes it true — there is no instruction to check for an existing open PR for this branch (e.g. `gh pr list --head <branch>` or `gh pr view <branch>`) before calling `gh pr create` again. A developer implementing T3 literally as written could call `gh pr create` unconditionally on every re-run, producing exactly the duplicate `ADR-0004` exists to prevent. Needs: T3 extended to state the explicit existing-PR check, performed before any `gh pr create` call on a retry.
- [ARCH-2] **blocking** — Invariant-preservation gap. ARCH.md's "Failure-diagnostic sanitization" convention (Security conventions) requires that any failure surfaced from an external system is reduced to a sanitized category before display or persistence — raw connector/git output is never shown or committed verbatim, "since it can echo credential fragments." T3/T4 write `post_merge_merge_attempt.blocked_reason` directly into a committed PLAN-NNNN milestone entry, with no stated sanitization step — as written, this field would carry `gh`'s own raw CLI output straight into permanent git history, precisely the scenario this convention exists to prevent. This wasn't addressed in DESIGN-0002 either, so it isn't a case of PLAN silently dropping something DESIGN already resolved — it's a genuine gap neither artifact closed. Needs: T3/T4 state that `blocked_reason` is written as a sanitized category (e.g. `required_status_check_pending`, `required_review_missing`, `branch_protection_rule`) derived from `gh`'s output, never that output verbatim.

**Clean perspectives:** Developer.

### Contradictions Requiring Arbitration

None. Two cascades worth noting once the blocking findings above are addressed:
- The Alternate/Failure Path Trace's "Team-Path Merge Blocked" row ("Human resolves the blocking requirement and re-runs... existing open PR found and retried, not duplicated") currently describes behavior `ARCH-1`'s fix is what will actually make true — re-validate that row once T3 is revised.
- `SRE-1`'s fix (an explicit git-sync step) introduces a new precondition that a future `TEST-PLAN` test case for T5's fallback path will need to set up explicitly — worth flagging for QE's own Testability Preconditions check when `TEST-NNNN` is authored.

## Address Summary — Round 1

**Date:** 2026-10-17

### Addressed
- `[SRE-1]` — T5 now states the explicit `git checkout main && git pull` step before running the fallback `stack.test.unit`/`stack.test.e2e` commands, so "`main`'s current checked-out state" is actually guaranteed true before those commands run.
- `[SRE-2]` — T4's `Source:` line now reads `OBS-0002 IT-1`, matching `create-plan.md`'s required citation format. Addressed directly as a non-judgment formatting correction, per this command's own exception for clear factual/formatting fixes.
- `[ARCH-1]` — T3 now states the explicit existing-PR lookup (`gh pr list --head <branch>` / `gh pr view <branch>`) that must run before any `gh pr create` call, so a retry reuses the existing open PR instead of creating a duplicate — the invariant `ADR-0004`'s Decision establishes is now spelled out where the Developer will actually be working, not left to memory of having read the ADR.
- `[ARCH-2]` — T3 and T4 both now state that `blocked_reason` is written as a sanitized category (e.g. `required_status_check_pending`, `required_review_missing`, `branch_protection_rule`) derived from `gh`'s output, never `gh`'s raw CLI text, per ARCH.md's "Failure-diagnostic sanitization" convention.

### Not Addressed
- `[QE-1]` — **no action needed**: this was QE's own explicit confirmation that `AC-07` correctly needs no PLAN task, not a flagged gap requiring a change.

## Plan Review — Round 2

**Date:** 2026-10-17
**Status:** Changes requested — 1 blocking, 0 non-blocking

**Developer:** no new findings — domain unaffected by Round 1's address (the fixes added mechanism specificity to T3/T4/T5's existing tasks; no new ambiguity, milestone restructuring, or AC-coverage change resulted).

Confirmed resolved on substance: `[SRE-2]` (T4's `Source:` line now reads `OBS-0002 IT-1`), `[ARCH-1]` (T3's existing-PR lookup, via `gh pr list --head`/`gh pr view`, now runs before any `gh pr create` call — a retry reuses the open PR rather than duplicating it), `[ARCH-2]` (T3 and T4 both now specify a sanitized category for `blocked_reason`, never `gh`'s raw CLI output). The Round 1 cascade note on the Alternate/Failure Path Trace's "Team-Path Merge Blocked" row is also resolved — that row's claimed behavior is now genuinely backed by T3's stated mechanism, not just asserted.

### SRE Findings

- [SRE-3] **blocking** — `[SRE-1]`'s own fix (T5's `git checkout main && git pull` before the fallback test run) assumes a remote always exists to pull from. Under the **solo** path, T3's merge is a local-only fast-forward — there is no remote `gh pr merge` step for local `main` to be stale against, and a bare `git pull` with no upstream configured errors rather than silently succeeding (this entire repo's own dogfooding has run every milestone so far with no remote configured at all). The fix correctly closes the staleness gap for the **team** path, where `gh pr merge` happens on GitHub and local `main` genuinely needs syncing — but introduces exactly the "fix closes one gap, opens an adjacent one" pattern this project's own history has hit before (`PLAN-0001` M2's `TC-FUNC-9`). Needs: T5 branch the sync step by path — **team:** `git checkout main && git pull`. **Solo:** `git checkout main` only — local `main` is already current from T3's own fast-forward merge moments earlier; no remote fetch is needed or guaranteed to even be possible.

**Clean perspectives:** QE, Architect.

### Contradictions Requiring Arbitration

None.

## Address Summary — Round 2

**Date:** 2026-10-17

### Addressed
- `[SRE-3]` — T5's sync step is now branched by which path T3 took: **team** → `git checkout main && git pull` (remote is authoritative — local `main` is genuinely stale after a `gh pr merge` on GitHub). **Solo** → `git checkout main` only, no pull (local `main` is already current from T3's own fast-forward merge; a bare `git pull` with no remote configured would error rather than succeed).

### Not Addressed
*(none — the one Round 2 finding is addressed above.)*

## Plan Review — Round 3

**Date:** 2026-10-17
**Status:** Approved — no blocking findings

Scope: T5 (the only task Round 2's address touched) plus a holistic re-pass over T1–T5 and both trace sections, given this closes the review.

**Developer:** no new findings — domain unaffected by Round 2's address (a path-branching refinement to an existing step; no new ambiguity, milestone restructuring, or AC-coverage change).

**Architect:** no new findings — domain unaffected by Round 2's address (T5's sync-step branching touches neither an ADR nor a DESIGN Key Decision invariant; `[ARCH-1]`/`[ARCH-2]` were both about T3/T4, untouched this round).

**SRE:** `[SRE-3]` confirmed resolved on substance — the sync step now correctly distinguishes team (`git checkout main && git pull`, remote is authoritative) from solo (`git checkout main` only, already current locally, no remote guaranteed to exist). Holistic re-check of T1–T5 found no further migration-ordering, rollback, ops-gap, deployment-sequence, or OBS-coverage issues.

**QE:** Happy Path Trace and both Alternate/Failure Path Trace tables re-validated against the final T1–T5 text — no gap in either; every row's claimed behavior is now backed by an explicit mechanism in its cited task, not just asserted.

**Clean perspectives:** Developer, QE, Architect, SRE.

### Contradictions Requiring Arbitration

None.
