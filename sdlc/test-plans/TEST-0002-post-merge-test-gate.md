---
id: TEST-0002
type: test-plan
schema_version: 1
title: "Configurable Post-Merge Test Gate — Test Plan"
status: passed
version: 2
updated: 2026-10-18
target_plan_version: 3
execution_date: "2026-10-18"
execution_summary: "All P0 automated tests passed across M1 (PLAN-0002's only milestone). One P1 manual case (TC-FUNC-10) deferred to release phase."
current_review_round: 2
latest_round:
  round: 2
  blocking: 0
  non_blocking: 0
  verdict: ready
approval:
  mode: repo_only
  approved_by: "Eric Pizon"
  approved_role: quality-engineer
  approved_at: "2026-10-18"
  decision_summary: "TEST-0002 (Configurable Post-Merge Test Gate) approved for implementation phase after 2 review rounds — all blocking findings resolved."
  external_event_id: null
  external_event_uri: null
policy_evaluation:
  evaluated: null
  policy_pack: null
  result: null
changelog:
  - "v2 — approved after 2 review rounds"
generation:
  command: /create-test-plan
  command_version: "4"
  model: "claude-sonnet-5"
  generated_at: "2026-10-17"
  session_id: null
---

# TEST-0002 — Configurable Post-Merge Test Gate

## Test Strategy

Same reframing TEST-0001 established for this stack (ARCH.md's "What this repo actually is" — no compiler, no test runner, no running service): "correctness" means an AI agent, following a command's text, produces the specified behavior.

- **Unit-equivalent — Structural/static checks (ARCH.md Tier 1):** frontmatter/cross-reference/schema consistency for `project.yml.template`, `forge-init.md`, `approve-implementation.md`, `execute-tests.md`. Written as `/review-forgekit-pr` rule checks, not as test cases here.
- **Integration-equivalent — Scenario-based fixture testing (ARCH.md Tier 2):** the primary tier for this plan. A single command's actual text walked against a disposable git fixture repo (scratchpad directory), asserting on real resulting frontmatter and git state. Most of this plan's cases are this type, since PLAN-0002's own tasks are scoped to 2–3 command files, not a multi-command orchestrator.
- **E2E-equivalent — Multi-command fixture sequences:** used specifically for the merge-then-test sequence (`/approve-implementation` → `/execute-tests`) and the retry-after-blocked-merge scenario, where a single command in isolation can't prove the end-to-end guarantee.

**Execution mode:** `automated` (an AI agent mechanically verifies fixture state against a stated assertion) or `manual` (requires human judgment — e.g., reading rendered question text for plain-language clarity). No test case requires network calls or a database; every fixture is local git state, with `gh` CLI calls simulated via fixture output rather than live GitHub calls.

## AC Coverage Map

### M1 — Configurable Post-Merge Test Gate

| AC | Test Case(s) |
|---|---|
| AC-01 | TC-FUNC-1, TC-FUNC-2 |
| AC-02 | TC-FUNC-3, TC-FUNC-4, TC-FUNC-5 |
| AC-03 | TC-FUNC-6, TC-FUNC-7 |
| AC-04 | TC-FUNC-8 |
| AC-05 | TC-FUNC-9, TC-FUNC-10, TC-FUNC-20 |
| AC-06 | TC-FUNC-11, TC-FUNC-12 |
| AC-07 *(no PLAN task — confirming regression case for the existing-gate claim)* | TC-FUNC-13 |
| AC-08 | TC-FUNC-14, TC-FUNC-15, TC-FUNC-16, TC-FUNC-17 |

**TC-FUNC-1** | integration | automated | P0
```gherkin
Given a fixture project with pipeline.test_gate: post_merge and a milestone at Implementation status: approved
When /approve-implementation's Handoff is reached
Then it offers to merge the milestone branch to main directly
And the Handoff never points to /execute-tests first
When the human confirms (solo path)
Then a direct fast-forward merge to main occurs
```

**TC-FUNC-2** | integration | automated | P0
```gherkin
Given the same fixture as TC-FUNC-1, at the merge offer
When the human declines
Then Implementation status remains "approved" — unchanged from before the offer
And no merge occurs, no post_merge_test field is written
When /approve-implementation is invoked again for this milestone
Then the merge is offered again, identically
```

**TC-FUNC-3** | integration | automated | P0
```gherkin
Given pipeline.test_gate: post_merge, stack.qa_trigger_command: "./scripts/qa-deploy-check.sh", and a milestone already merged (post_merge_test: pending)
When /execute-tests runs for that milestone
Then stack.qa_trigger_command is invoked as the validation target
And the result is written to post_merge_test.status, never derived from the already-deleted feature branch
```

**TC-FUNC-4** | integration | automated | P0
```gherkin
Given pipeline.test_gate: post_merge, stack.qa_trigger_command: "none", a milestone merged via the team path (gh pr merge, not a local fast-forward), and post_merge_test: pending
When /execute-tests runs the local fallback validation
Then git checkout main && git pull runs first
And stack.test.unit/stack.test.e2e then run against that freshly-synced checkout, never the stale pre-pull state
```

**TC-FUNC-5** | integration | automated | P0
```gherkin
Given the same fixture shape as TC-FUNC-4, but the milestone merged via the solo fast-forward path
When /execute-tests runs the local fallback validation
Then only git checkout main runs — no git pull is attempted
And the fixture confirms no remote is configured for this repo, so a pull would have errored had one been attempted
```

**TC-FUNC-6** | integration | automated | P0
```gherkin
Given a milestone merged under post_merge (post_merge_test: pending), and stack.qa_trigger_command targets a non-shared environment
When /execute-tests finds a P0 failure
Then /create-bug is invoked
And Implementation status is never set to test_failed
And post_merge_test.status is set to "failed"
```

**TC-FUNC-7** | integration | automated | P0
```gherkin
Given the same fixture as TC-FUNC-6, but the affected environment is marked shared/blocking
When /execute-tests finds a P0 failure
Then /escalate-to-incident is invoked instead of /create-bug
And Implementation status is never set to test_failed
```

**TC-FUNC-8** | integration | automated | P0
```gherkin
Given a fixture project with pipeline.test_gate absent entirely (no opt-in)
When the full implement → review → approve → execute-tests → create-pr sequence runs for a milestone
Then every command's behavior is byte-identical to its pre-feature text
And no post_merge branch in approve-implementation.md or execute-tests.md is ever reached
And no post_merge_test field is ever written
```

**TC-FUNC-9** | integration | automated | P0
```gherkin
Given a fresh /forge-init run with no prior project.yml
When the test_gate question is reached and the human expresses no strong preference
Then pipeline.test_gate is written as "pre_merge"
```

**TC-FUNC-10** | manual | manual | P1
```gherkin
Given /forge-init's test_gate question as rendered to a human
When the question text is read
Then it states the post_merge trade-off plainly (code reaches main/QA before /execute-tests confirms it) — not just the two mode names
```
**Procedure:** Read the actual rendered question text from a fixture `/forge-init` run; confirm the trade-off sentence is present and in plain language, not just the two literal mode names with no explanation.
**Reason:** "Plain language" is a human-judgment quality bar, not a mechanically checkable string match — a keyword-presence check could pass text that's technically compliant but not actually informative.

**TC-FUNC-11** | integration | automated | P0
```gherkin
Given a milestone merging under post_merge via the solo path
When the merge is confirmed and completes
Then Implementation status reaches "pr_merged" directly from the merge step
And /create-pr is never invoked for this milestone
```

**TC-FUNC-12** | e2e | automated | P0
```gherkin
Given a milestone merging under post_merge via the team path
And gh pr create succeeds, gh pr merge then reports success, and gh pr view --json state independently confirms the PR is genuinely merged
When the merge step evaluates this verification result
Then Implementation status reaches "pr_merged" directly — because verification independently confirmed the merge, not merely because gh pr merge's exit code was 0
And neither /create-pr nor /close-milestone is ever invoked for this milestone
```

**TC-FUNC-13** | integration | automated | P0
```gherkin
Given milestone M1 merged under pipeline.test_gate: post_merge (Implementation status: pr_merged), and milestone M2 with depends_on_milestones: [M1]
When /implement-plan M2's Pre-flight Check #4 evaluates the dependency
Then it correctly recognizes M1 as unlocking M2 (pr_merged is pr_merged, regardless of which path produced it)
And no change to implement-plan.md's existing gate was required to make this true
```

**TC-FUNC-14** | integration | automated | P0
```gherkin
Given a milestone whose merge under post_merge is just confirmed
When Implementation status is set to "pr_merged"
Then post_merge_test: {status: pending, updated_at: <now>} is written in the same step
And pr_merged being terminal does not prevent this field from existing alongside it
```

**TC-FUNC-15** | integration | automated | P0
```gherkin
Given a milestone with post_merge_test: pending
When /execute-tests runs and all P0 cases pass
Then post_merge_test.status is set to "passed", updated_at refreshed
```

**TC-FUNC-16** | integration | automated | P0
```gherkin
Given a milestone with post_merge_test: pending
When /execute-tests finds a P1 failure only (no P0 failures)
Then post_merge_test.status is set to "passed" — a P1-or-lower failure never flips this field to "failed"
When a separate fixture finds a P0 failure
Then post_merge_test.status is set to "failed"
```

**TC-FUNC-17** | integration | automated | P0
```gherkin
Given a milestone whose post_merge_test.status was already set to "failed" by a prior /execute-tests run
When /execute-tests is invoked again for the same milestone (e.g. by mistake, or a re-run)
Then post_merge_test.status and updated_at are not overwritten
And any fix for the regression is expected to land as a new milestone or hotfix, not a mutation of this record
```

## Edge Cases and Boundary Conditions

**TC-FUNC-18** | integration | automated | P0
```gherkin
Given a project.yml with pipeline.test_gate either absent, or set to a malformed value (e.g. "Post-Merge", a typo, or a non-string)
When any command reads pipeline.test_gate
Then it is treated as "pre_merge" — fail-closed, per ADR-0003's established convention
And no post_merge-specific behavior is ever reached from a malformed or absent value
```

**TC-FUNC-19** | integration | automated | P0
```gherkin
Given milestone MA merged under pipeline.test_gate: post_merge (its own post_merge_test block is present)
And the project's pipeline.test_gate is subsequently switched to pre_merge before /execute-tests runs for MA
When /execute-tests MA runs
Then it still follows the post-merge validation path for MA specifically — the presence of MA's own post_merge_test block takes authority over the project's current setting
And a separate milestone MB, started fresh after the switch, correctly follows the pre_merge path instead
```

**TC-FUNC-20** | integration | automated | P0
```gherkin
Given a fresh /forge-init run with no prior project.yml
When the test_gate question is reached and the human explicitly selects post_merge
Then pipeline.test_gate is written as "post_merge"
```

## ADR/Invariant Test Cases

**TC-ADR-0004-1** | e2e | automated | P0
```gherkin
Given ADR-0004's decision: the merge is verified, never assumed — gh pr merge's exit code alone is not sufficient
And a fixture where gh pr merge reports success (exit code 0) but gh pr view --json state shows the PR is not actually merged (a required check landed after the merge attempt)
When /approve-implementation's post_merge merge step evaluates the outcome
Then Implementation status is NOT set to pr_merged
And it remains "approved", with the PR left open
And post_merge_test is not written — there is nothing to track yet, since no merge actually happened
```

**TC-ADR-0004-2** | e2e | automated | P0
```gherkin
Given ADR-0004's decision: a retry never creates a duplicate PR
And a fixture where a prior /approve-implementation attempt left an open PR for this milestone's branch (blocked merge)
When /approve-implementation is re-invoked for the same milestone
Then it checks for an existing open PR for this branch before calling gh pr create
And it finds the existing PR and attempts gh pr merge on it directly
And gh pr create is never called a second time for this branch
```

**TC-ADR-0004-3** | integration | automated | P0
```gherkin
Given ADR-0004's team-path blocked-merge scenario, and ARCH.md's Failure-diagnostic sanitization convention (raw connector output is never shown or committed verbatim)
And gh's own raw failure output contains implementation-specific detail (e.g. a verbose branch-protection API response)
When post_merge_merge_attempt.blocked_reason is written
Then it contains only a sanitized category (e.g. required_status_check_pending, required_review_missing, branch_protection_rule)
And gh's raw CLI output never appears verbatim in any field committed to PLAN-NNNN
```

## Happy Path Coverage

### US1 — "As a Lead Developer at an enterprise with a shared QA environment, I want milestone code to merge to `main` as soon as it's reviewed and approved..."

| Flow Step | Test Case(s) |
|---|---|
| Milestone reaches `approved` under `pipeline.test_gate: post_merge` | TC-FUNC-20 — the mode was explicitly configured as `post_merge` at `/forge-init` |
| `/approve-implementation`'s Handoff offers to merge directly | TC-FUNC-1 |
| Human confirms; merge executes (solo fast-forward or team PR+merge) | TC-FUNC-1 (solo), TC-FUNC-12 (team) |
| Merge verified actually completed | TC-FUNC-11 (solo), TC-FUNC-12 (team) |
| `**Implementation status:** pr_merged` and `post_merge_test: {status: pending}` written | TC-FUNC-14 |
| Adopting project's own CI/CD deploys from `main` (outside ForgeKit's control, per Non-Goal #2) | — |
| QE runs `/execute-tests`; result ties to the merged commit, never the deleted feature branch | TC-FUNC-3, TC-FUNC-4, TC-FUNC-5 |
| `post_merge_test.status` set to `passed` | TC-FUNC-15 |

Gap check: none — all steps covered.

## Alternate Path Coverage

### US1 — Alternate Path: Post-Merge P0 Test Failure (User Story 4)

| Flow Step | Test Case(s) |
|---|---|
| Milestone already merged under `post_merge` (`post_merge_test: pending`) | TC-FUNC-14 |
| `/execute-tests` runs against the merged state and finds a P0 failure | TC-FUNC-6, TC-FUNC-7 |
| Failure routes to `/create-bug` (or `/escalate-to-incident` if shared/blocking) — never `test_failed` | TC-FUNC-6, TC-FUNC-7 |
| `post_merge_test.status` set to `failed`, `updated_at` set, never later overwritten | TC-FUNC-16, TC-FUNC-17 |

Gap check: none — all steps covered.

### US1 — Alternate Path: Team-Path Merge Blocked by Branch Protection (ADR-0004)

| Flow Step | Test Case(s) |
|---|---|
| Human confirms the offered merge under `post_merge` (team path) | TC-ADR-0004-1 *(its own fixture precondition presupposes this confirm already happened, leading into the blocked-merge sequence it tests)* |
| `gh pr create` succeeds; `gh pr merge` fails or stalls | TC-ADR-0004-1 |
| Merge-verification confirms the PR is not actually merged | TC-ADR-0004-1 |
| `**Implementation status:**` stays at `approved`; PR left open; `post_merge_merge_attempt.blocked` recorded with a reason | TC-ADR-0004-1, TC-OBS-merge_attempt-1, TC-ADR-0004-3 *(sanitized reason)* |
| Human resolves the blocking requirement and re-runs; the existing open PR is found and retried, not duplicated | TC-ADR-0004-2 |

Gap check: none — all steps covered.

## Observability Verification

**TC-OBS-pipeline_test_gate-1** | integration | automated | P1
```gherkin
Given a fixture project.yml with pipeline.test_gate set to each of pre_merge and post_merge in turn
When /approve-implementation and /execute-tests each read this field at their respective decision points
Then under post_merge, /approve-implementation's Handoff offers the direct merge and /execute-tests takes the post-merge validation branch
And under pre_merge, /approve-implementation never offers the merge and /execute-tests runs its ordinary branch-scoped Pre-flight instead
```

**TC-OBS-post_merge_test_status-1** | integration | automated | P1
```gherkin
Given three fixtures: (1) a milestone just merged under post_merge, (2) a merged milestone whose post-merge tests all pass, (3) a merged milestone with a P0 post-merge failure
When each committed milestone entry is inspected
Then scenario (1) reads post_merge_test.status == "pending", (2) reads "passed", and (3) reads "failed" — three distinct, explicitly-paired values, not merely three values drawn from the set
```

**TC-OBS-post_merge_test_updated_at-1** | integration | automated | P2
```gherkin
Given a milestone's post_merge_test block written at merge time (status: pending) and later updated by /execute-tests (status: passed)
When updated_at is inspected at both points
Then its value changes between the pending write and the terminal write — it is not a single value fixed at first write
```

**TC-OBS-merge_attempt-1** | integration | automated | P1
```gherkin
Given a fixture blocked-merge scenario (ADR-0004's "Not merged" branch)
When post_merge_merge_attempt is inspected
Then it reads {blocked: true, blocked_reason: <sanitized category>, attempted_at: <timestamp>}
And a subsequent successful retry's post_merge_test: {status: pending, ...} write supersedes it without a separate cleanup write
```

**TC-OBS-qa_trigger_infra_failure-1** | integration | automated | P1
```gherkin
Given stack.qa_trigger_command configured to a command that fails for an infrastructure reason unrelated to test outcomes (e.g. simulated auth failure)
When /execute-tests invokes it under post_merge
Then the failure is surfaced directly to the user as an infrastructure problem
And post_merge_test.status is left unchanged — never set to "failed" for an infra failure
```

## Security Test Cases

N/A — `SEC-0002` was never triggered (`ANALYSIS-0002`'s `security_review_required: no`). `TC-ADR-0004-3` above covers the one security-adjacent concern this feature introduces (ARCH.md's Failure-diagnostic sanitization convention applied to `blocked_reason`), filed under ADR/Invariant Test Cases since it traces to `ADR-0004` rather than a SEC finding.

## Test Data Strategy

Every test case in this plan runs against a disposable git fixture repository created fresh in a scratchpad directory (never this repo), per ARCH.md's Tier 2 methodology:

- Each test case constructs its own fixture `project.yml`/`PLAN-NNNN` state to the exact precondition its `Given` clause specifies — never reused wholesale from another test case's output.
- `gh` CLI behavior (`gh pr create`, `gh pr merge`, `gh pr view`, `gh pr list`) is simulated via fixture command output/exit codes, not live GitHub API calls — this plan tests ForgeKit's own command text and resulting git/frontmatter state, not GitHub's own behavior.
- The solo-path fixtures (`TC-FUNC-5`, `TC-FUNC-11`) use a repo with no remote configured at all, to make the "a pull would error here" assertion genuine rather than assumed.
- The team-path fixtures use a fixture repo with a simulated remote and simulated `gh` output for each PR state transition needed (open, blocked, merged).
- Teardown: fixture repos are deleted after each test case completes; nothing persists between runs.

## Execution Results Summary

**Status:** pending

*(Populated by `/execute-tests` once implementation begins. No test case in this plan has been executed yet — TEST-0002 is still in its authoring/review phase.)*

## Test Plan Review — Round 1

**Date:** 2026-10-18
**Status:** Changes requested — 3 blocking, 2 non-blocking

### SRE Findings

- [SRE-1] **non-blocking** — `TC-OBS-pipeline_test_gate-1`'s `Then` clause ("both commands branch consistently on the same live-read value") doesn't name the two expected behaviors per value — it asserts consistency between the two readers, not what either one should actually do. Needs: state directly, e.g. "under `post_merge`, `/approve-implementation`'s Handoff offers the merge and `/execute-tests` takes the post-merge branch; under `pre_merge`, neither does."
- [SRE-2] **non-blocking** — `TC-OBS-post_merge_test_status-1`'s `Then` clause ("status reads exactly one of {pending, passed, failed} per the specific trigger exercised") doesn't pair each trigger to its exact expected value explicitly — the same imprecision `TEST-0001`'s own `[SRE-1]` flagged for its directly analogous `TC-OBS-status-1` case. This plan's own `TC-FUNC-14`/`TC-FUNC-15`/`TC-FUNC-16` already pair trigger→value explicitly elsewhere, so this case reads as less precise than this plan's own established bar.

### Lead Developer Findings

- [LD-1] **blocking** — Happy Path Coverage's "Human confirms; merge executes (solo fast-forward or team PR+merge)" row cites `TC-ADR-0004-2` for the team-path half. `TC-ADR-0004-2`'s own `Given` starts from a *prior blocked-merge retry* scenario — it never exercises a first-time, successful team-path confirm-and-merge at all; that's what `TC-FUNC-12` actually does. This is false coverage: a reader trusting this row would believe the happy-path team mechanics are proven by a test case that is actually about the alternate (blocked) path. Needs: re-cite the team-path half to `TC-FUNC-12`.
- [LD-2] **blocking** — Happy Path Coverage's "Merge verified actually completed" row cites `TC-ADR-0004-1`. That test case's own `Given`/`Then` exercises exactly the *failure* path — `gh pr merge` reports success but the PR is **not** actually merged, and status correctly does **not** advance. It is already correctly cited under Alternate Path Coverage's "Merge-verification confirms the PR is not actually merged" row. The happy-path row needs a case proving verification correctly confirms a *genuinely successful* merge — `TC-FUNC-11` (solo) and `TC-FUNC-12` (team) both already exercise this. Needs: re-cite to `TC-FUNC-11`/`TC-FUNC-12`.
- [LD-3] **blocking** — Alternate Path Coverage's "Human confirms the offered merge under `post_merge` (team path)" row cites `TC-FUNC-1`, annotated "(same confirm step, team branch)." `TC-FUNC-1`'s own `Given`/`When`/`Then` is explicitly and only the **solo** path ("When the human confirms (solo path)... Then a direct fast-forward merge to main occurs") — it never exercises a team-path confirm at all; the annotation is simply wrong. Needs: re-cite to `TC-ADR-0004-1`, whose own fixture precondition already presupposes a team-path confirm leading into the blocked-merge sequence it tests.

**Clean perspectives:** *(Security not applicable — `SEC-0002` was never triggered)*

### Contradictions Requiring Arbitration

None. All three Lead Developer findings are the same underlying mistake (citing a test case whose actual `Given`/`Then` doesn't match the trace row's claimed step) recurring across both trace tables — worth double-checking every citation in both tables together when addressing, not just the three named rows, in case a fourth instance was missed.

## Address Summary — Round 1

**Date:** 2026-10-18

### Addressed
- `[LD-1]` — Happy Path Coverage's "Human confirms; merge executes" row now cites `TC-FUNC-12` for the team-path half instead of `TC-ADR-0004-2` (a retry/dedup scenario, not a first-time successful merge).
- `[LD-2]` — Happy Path Coverage's "Merge verified actually completed" row now cites `TC-FUNC-11` (solo)/`TC-FUNC-12` (team) instead of `TC-ADR-0004-1` (the failure path). `TC-FUNC-12` itself was also strengthened: verification (`gh pr view --json state`) is now an explicit, actively-exercised `When`/`Then` step rather than an assumed `Given` precondition, so it genuinely proves the happy-path claim rather than just being consistent with it.
- `[LD-3]` — Alternate Path Coverage's "Human confirms the offered merge under `post_merge` (team path)" row now cites `TC-ADR-0004-1` instead of `TC-FUNC-1`, which is solo-only by its own text — the prior annotation was factually wrong.
- `[SRE-1]` — `TC-OBS-pipeline_test_gate-1`'s `Then` clause now states the two exact expected behaviors per mode value, rather than asserting generic "consistency."
- `[SRE-2]` — `TC-OBS-post_merge_test_status-1`'s `Then` clause now pairs each of the three named fixture scenarios to its exact expected value explicitly, matching this plan's own established precision bar elsewhere.

**Additional fix found during the "double-check for a fourth instance" self-check the Round 1 Contradictions note called for:** Happy Path Coverage's first row ("Milestone reaches `approved` under `pipeline.test_gate: post_merge`") cited `TC-FUNC-9` — but `TC-FUNC-9` actually tests the *default-to-`pre_merge`* behavior (no preference given), the opposite of what the row needs. No existing test case covered a human explicitly choosing `post_merge` at `/forge-init`. Added `TC-FUNC-20` for this case, added it to `AC-05`'s AC Coverage Map entry, and re-cited the row to it.

### Not Addressed
*(none — all findings addressed above.)*

## Test Plan Review — Round 2

**Date:** 2026-10-18
**Status:** Approved — no blocking findings

Scope: both trace tables, `TC-FUNC-12`, `TC-FUNC-20`, `AC-05`'s coverage row, and the two revised `TC-OBS-*` cases — the sections Round 1's address pass touched, plus a full re-read of both Happy Path and Alternate Path Coverage tables end to end (per Round 1's own note to double-check for a missed instance).

**Lead Developer:** `[LD-1]`/`[LD-2]`/`[LD-3]` all confirmed resolved — every row in both trace tables now cites a test case whose actual `Given`/`When`/`Then` genuinely exercises the claimed step. `TC-FUNC-12`'s strengthening (verification as an active assertion, not an assumed precondition) holds up under re-reading. The additional self-caught fix (`TC-FUNC-20` replacing the mismatched `TC-FUNC-9` citation) is also confirmed correct — `TC-FUNC-20` explicitly exercises a human choosing `post_merge`, matching the row's claim. No further citation mismatch found anywhere in either table.

**SRE:** `[SRE-1]`/`[SRE-2]` confirmed resolved — both `TC-OBS-*` `Then` clauses now name exact expected values per scenario, matching this plan's own precision bar elsewhere. Signal coverage unaffected (still all 5 `OBS-0002` signals covered; no signal added or removed this round).

**Clean perspectives:** *(Security not applicable — `SEC-0002` was never triggered)*

### Contradictions Requiring Arbitration

None.
