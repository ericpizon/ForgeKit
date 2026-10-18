---
id: SPEC-0001
type: spec
schema_version: 1
title: "ForgeKit Operator"
status: approved
created: 2026-10-07
updated: 2026-10-07
version: 3
changelog:
  - "v2 — Added Goals, User Stories, Non-Functional Requirements, Constraints & Assumptions, Security Considerations, Observability Considerations, and Open Questions sections. Resolved the execution-model ambiguity (Operator is human-reinvoked, never daemon-triggered) and the human-confirmation-scope ambiguity (formal gates vs. routine draft-commit pauses). Added a fail-closed autonomy default, an invalid/already-terminal work-unit boundary, and a security/policy-exception escalation AC; split AC-15 into sub-criteria. Addresses ANALYSIS-0001 Round 1 findings SC-1, LD-1 through LD-11, QE-1 through QE-4."
  - "v3 — Added the Operator-invocation-recognition prerequisite to §41 and its deferred mechanism to §51 Open Questions (the adjacent gap LD-9's own resolution exposed). Added one clarifying sentence to §18 noting its outcome set is extended by AC-26 and the autonomy-containment cap. Addresses ANALYSIS-0001 Round 2 findings LD-12, QE-5."
  - "v3 — approved after 3 review rounds."
owner: "Eric Pizon"
priority: high
stakeholders:
  - tpo
  - lead-developer
  - security
  - sre
links: []
product_validated: true
validated_by: "Eric Pizon — TPO"
validated_date: 2026-10-07
approval:
  mode: repo_only
  approved_by: "Eric Pizon"
  approved_role: tpo
  approved_at: "2026-10-07"
  decision_summary: "SPEC-0001 (ForgeKit Operator) approved for design phase after 3 review rounds and product validation sign-off."
  external_event_id: null
  external_event_uri: null
policy_evaluation:
  evaluated: null
  policy_pack: null
  result: null
generation:
  command: /create-spec
  command_version: 5
  model: claude-sonnet-5
  generated_at: 2026-10-07
  session_id: null
---

# ForgeKit Operator

## 1. Purpose

ForgeKit provides a governed software delivery lifecycle through specialized commands, artifacts, reviews, approvals, and gates.

Today, a human still acts as the workflow operator. The human determines which ForgeKit command should execute next, observes its result, decides whether another review or revision is required, advances milestones, coordinates repositories, handles failures, and resumes work after interruptions.

The ForgeKit Operator will automate that orchestration.

The Operator must execute as much of the ForgeKit lifecycle autonomously as can be done safely while preserving ForgeKit's existing governance model, adversarial review process, approval boundaries, traceability, and artifact authority.

The Operator must not become a second implementation of ForgeKit's engineering intelligence.

Its responsibility is to operate ForgeKit.

## 2. Product Outcome

A developer should be able to provide an existing ForgeKit work unit and invoke a single Operator entry point.

Conceptually:

```text
/operate SPEC-0042
```

or:

```text
/operate THREAD-0017
```

From that point, the Operator should autonomously advance the work through every currently permitted ForgeKit stage until one of the following occurs:

```text
The work reaches the requested completion boundary.

An accountable human decision is required.

An external action is required.

An environment or dependency prevents progress.

The workflow cannot safely converge.

ForgeKit state is ambiguous or inconsistent.
```

The user should not have to manually determine which ForgeKit command comes next.

## 3. Core Principle

> The Operator orchestrates ForgeKit. It does not replace ForgeKit.

Existing ForgeKit commands remain responsible for engineering activities such as requirements reasoning, design, architecture, security review, observability planning, implementation planning, test planning, implementation, implementation review, release planning, and release validation.

The Operator is responsible for orchestration concerns such as:

```text
state resolution
next-action selection
command sequencing
gate evaluation
review and revision loops
retry behavior
resumption
cross-repo ordering
human escalation
execution history
convergence protection
```

The Operator must invoke existing ForgeKit domain operations rather than reproducing their reasoning internally.

For example, if a SPEC must become cross-repository work, the Operator invokes the ForgeKit promotion mechanism. It does not independently recreate the promotion process.

## 4. Scope

Operator v1 supports an existing ForgeKit engineering work unit.

Supported entry points must include:

```text
SPEC-NNNN

THREAD-NNNN
```

A work unit may have originated directly in ForgeKit or through external work intake.

The Operator must support work that is already partially complete.

For example, if SPEC and DESIGN are complete and PLAN has not been created, Operator must continue from that point rather than restart the lifecycle.

Operator v1 must support both single-repository and multi-repository delivery.

## 5. Non Goals

Operator v1 does not provide product idea ingestion, product discovery, product portfolio management, ADO or Jira work-item creation, Confluence integration, ADO integration, a graphical user interface, the future structured read model, full audit-evidence infrastructure, organizational resource scheduling, or unrestricted autonomous production deployment.

Operator v1 does not replace existing ForgeKit commands.

Operator v1 does not introduce a generic workflow-definition language.

Operator v1 does not introduce an Operator-owned engineering-state database.

Operator v1 does not require parallel coding agents.

Operator v1 does not optimize ForgeKit for maximum autonomy at the expense of governance.

## 6. Authoritative State

Existing ForgeKit artifacts remain the authoritative source of engineering state.

Examples include:

```text
SPEC
ANALYSIS
DESIGN
ADR
OBSERVABILITY-PLAN
SECURITY-REVIEW
PLAN
TEST-PLAN
IMPL-REVIEW
BUG
INCIDENT
POSTMORTEM
RELEASE-PLAN
THREAD
XDEP
cross-repo system-test artifacts
```

The Operator must not maintain a duplicate interpretation of this engineering state.

When persisted Operator state conflicts with authoritative ForgeKit artifacts, ForgeKit artifacts win.

Operator-owned state may contain only orchestration-specific information that cannot safely be reconstructed from existing artifacts.

Examples include:

```text
Operator run ID
attempt count
last attempted action
review-loop count
finding recurrence information
retry history
last successful checkpoint
current escalation
```

## 7. Shared State Resolver

ForgeKit must have one authoritative mechanism for resolving current workflow state and determining the next eligible action.

The Operator must consume the same state-resolution logic exposed through `/status`.

Operator must not implement an independent artifact parser with its own lifecycle rules.

Conceptually:

```text
ForgeKit artifacts
       ↓
ForgeKit state resolver
       ↓
       ├── /status human output
       │
       └── Operator structured result
```

Before Operator autonomous execution is enabled, `/status` must be extended to resolve the full supported lifecycle.

This includes the pre-PLAN stages, implementation milestone states, release planning, and relevant bug or incident conditions.

The resolver must return a small structured result in addition to the existing human-readable output.

Conceptually:

```yaml
resolved_state:
  stage: implementation
  gate: blocked
  blocking_count: 2
  stale: false
  next_command: "/address-implementation-review PLAN-0042 M2"
```

The exact schema may be refined during DESIGN, but it must remain a derived view of authoritative artifacts rather than becoming another source of truth.

## 8. Review Outcome Normalization

The Operator must never determine whether a review passed by parsing arbitrary Markdown sentences.

ForgeKit review artifacts that currently express blocking and non-blocking findings only in prose must expose a consistent structured current-round result.

This applies at minimum to:

```text
DESIGN
PLAN
TEST-PLAN
IMPL-REVIEW
RELEASE-PLAN
```

Conceptually:

```yaml
latest_round:
  round: 3
  blocking: 0
  non_blocking: 2
  verdict: ready
```

Allowed verdicts for Operator purposes must include at least:

```text
ready
needs_revision
```

The structured value is authoritative for machine control.

Human-readable review summaries remain in the artifact but must be a rendering of the same underlying result.

The Operator must never advance through a gate when `blocking > 0`.

## 9. Finding Lineage and Convergence

Review finding IDs may remain local to a review round.

ForgeKit must, however, provide explicit lineage when a finding recurs across rounds.

Conceptually:

```yaml
finding:
  id: LD-2
  recurs_from:
    round: 2
    finding: LD-1
```

Operator must use explicit lineage rather than fuzzy interpretation of prose to determine recurrence.

Operator must detect at least:

```text
the same issue recurring across rounds
a previously resolved issue returning
review oscillation
the configured review-round limit being exceeded
incompatible reviewer requirements that prevent convergence
```

When autonomous convergence cannot be reached, Operator must stop with `HUMAN_DECISION_REQUIRED`.

It must never weaken a review finding, change its severity, or mark it resolved merely to continue.

## 10. ForgeKit Lifecycle

Operator must follow the lifecycle that ForgeKit actually implements.

It must not introduce a competing lifecycle.

The normal lifecycle is conceptually:

```text
SPEC
  ↓
SPEC review / analysis
  ↓
SPEC approval
  ↓
DESIGN
  ↓
DESIGN review and approval
  ↓
OBSERVABILITY planning
  +
SECURITY review when required
  ↓
PLAN
  ↓
PLAN review and approval
  ↓
TEST-PLAN
  ↓
TEST-PLAN review and approval
  ↓
milestone implementation
  ↓
implementation review
  ↓
implementation approval
  ↓
test execution
  ↓
PR
  ↓
milestone completion
  ↓
remaining milestones
  ↓
RELEASE-PLAN
  ↓
release review / approval
  ↓
release readiness
```

Operator must obey existing ForgeKit rules that conditionally omit or add lifecycle activities.

## 11. Milestone State Machine

The existing milestone implementation state machine remains authoritative.

Operator must support:

```text
pending
    ↓
in_progress
    ↓
approved
    ↓
tests_passed
    ↓
pr_open
    ↓
pr_merged
```

with the existing failure path:

```text
approved
    ↓
test_failed
    ↓
failure resolution
    ↓
retest
```

Operator must determine the appropriate next command from the actual milestone state.

It must not infer implementation completion merely because the milestone is `in_progress`.

## 12. Implementation Execution Completeness

ForgeKit currently needs an additional distinction between:

```text
implementation started
```

and:

```text
implementation command completed successfully
```

This is required for autonomous resumption.

If an implementation session terminates while code generation is incomplete, Operator must not automatically proceed to implementation review simply because the milestone is `in_progress`.

ForgeKit must persist enough execution state to distinguish an interrupted implementation from a completed implementation awaiting review.

Conceptually:

```yaml
implementation_execution:
  state: started | completed
  attempt: 2
  started_at: ...
  completed_at: ...
```

The exact representation may change during DESIGN.

The invariant is:

> Operator must never send partially completed implementation work to implementation review merely because a prior implementation command began.

When execution is interrupted, Operator must either safely resume the implementation activity or stop with an explicit recovery state.

## 13. Idempotency and Resume

Operator must be safe to invoke repeatedly.

Every transition must verify authoritative state before performing a side effect.

This applies particularly to artifact creation, review creation, thread promotion, branch creation, implementation attempts, PR creation, environment promotion, and release operations.

If a command completed but the Operator session terminated before recording its own checkpoint, the next invocation must discover the completed work instead of creating a duplicate.

Conversation history must never be required for successful resumption.

A fresh session must be able to reconstruct enough state from repository artifacts to continue.

## 14. Operator Run Identity

Every autonomous execution should have a durable run identity.

Conceptually:

```text
OPRUN-0042
```

An Operator run should capture only orchestration-specific state.

It should be possible to answer:

```text
What action was last attempted?

Did it complete?

How many autonomous review loops have occurred?

Has the same finding recurred?

Why is execution currently stopped?

What action is eligible next?
```

The Operator run record must not duplicate the contents or status of engineering artifacts.

## 15. Command Execution Cycle

For each autonomous step, Operator must:

```text
Resolve authoritative state.

Determine the next permitted action.

Verify policy allows autonomous execution.

Invoke the existing ForgeKit command.

Verify the command outcome from resulting artifacts.

Resolve state again from authoritative sources.
```

Operator must not assume that a command succeeded because it was invoked.

Artifact state after execution determines success.

## 16. Human Decision Boundaries

Human escalation is expected behavior.

Operator must stop when ForgeKit requires accountable human judgment.

Examples include:

```text
risk acceptance
security exception
policy exception
explicit approval
material scope ambiguity
conflicting requirements
irreversible decision without sufficient evidence
production release confirmation
non-converging adversarial review
ambiguous or inconsistent ForgeKit state
```

Operator must distinguish between information that can be safely inferred and decisions that require accountable human ownership.

Operator may reduce unnecessary questions.

Operator may not remove required human gates.

## 17. Human Decision Package

When Operator stops for human judgment, it must provide a concise decision package.

The package must include:

```text
the decision required
why execution cannot continue safely
the relevant artifact or finding
actions already attempted
available alternatives when known
the recommended option when appropriate
the consequences of each option when known
the command or state that will resume after the decision
```

Related decisions should be grouped when practical so that Operator does not repeatedly interrupt the user for closely related issues.

## 18. Operator Outcomes

Operator must distinguish materially different stopping conditions.

The execution model must support outcomes equivalent to:

```text
COMPLETE

HUMAN_DECISION_REQUIRED

EXTERNAL_ACTION_REQUIRED

BLOCKED_ENVIRONMENT

BLOCKED_DEPENDENCY

RETRYABLE_ERROR

TERMINAL_ERROR

AMBIGUOUS_STATE
```

Names may be refined during DESIGN.

A blocked or ambiguous condition must never be represented as success.

This list is extended by two cases introduced elsewhere in this SPEC: the not-found/already-complete case (AC-26) and reaching the autonomy-containment action cap (§47) — both need an outcome equivalent, with exact naming deferred to DESIGN the same as the rest of this list (§51).

## 19. BUG State Normalization

Before Operator autonomously interacts with BUG/back-cycle workflows, ForgeKit must define one canonical BUG status vocabulary.

All commands that write or evaluate BUG state must use that vocabulary consistently.

Operator must never infer that similarly worded statuses such as `reported`, `open`, `fix_deployed`, `verified`, or `closed` are equivalent unless the canonical state machine explicitly defines that relationship.

The existing bug workflow should be normalized rather than replaced.

## 20. Test Failure Classification

When test execution reveals a failure requiring QE or human classification, the resulting decision must be stored structurally.

At minimum, the classification must distinguish:

```text
production_bug
test_bug
```

Operator must consume the stored classification.

Operator must not independently reinterpret test evidence to override a human classification.

A production defect should use the existing ForgeKit bug/back-cycle behavior.

A test defect may proceed through the appropriate test correction and re-execution flow.

## 21. Multi-Repo Support

Multi-repository operation is required in v1.

Operator must understand existing ForgeKit concepts including:

```text
THREAD
participants
XDEPs
Cross-Repo Contract
Interface Freeze
repo-specific PLANs
repo-specific TEST-PLANs
cross-repo scenarios
candidate_ref maps
deployment ordering
release waves
```

Operator must not treat the first completed repository as completion of the THREAD.

## 22. XDEP Satisfaction

XDEP satisfaction remains derived from authoritative participant state.

Operator v1 must compute XDEP satisfaction through live state resolution.

Operator v1 must not introduce a persisted `satisfied: true` cache merely to improve lookup performance.

If caching is introduced later, it must remain advisory and must never override authoritative live state for a blocking decision.

Correctness takes precedence over performance in Operator v1.

## 23. Thread Completion

`thread.yml` does not need a persisted `complete` field.

Thread completion should be derived.

The resolver must consider, as applicable:

```text
participant lifecycle state
required milestone completion
XDEP satisfaction
approved contract revision
required cross-repo verification
release state
environment state
```

A THREAD is complete only when all required thread-wide conditions are complete.

## 24. Cross-Repo Dependency Deadlock

Operator must detect a condition where work remains incomplete but no participant has an eligible action because participants are mutually blocked by dependencies.

For example:

```text
Repo A waits for Repo B.

Repo B waits for Repo A.

Neither has an executable next action.
```

Operator must not repeatedly poll or spin.

It must return `HUMAN_DECISION_REQUIRED` or an equivalent explicit dependency-deadlock outcome.

The decision package must identify the participants and dependencies forming the cycle.

Operator must not break the dependency automatically.

## 25. Single-Repo to THREAD Promotion

If ANALYSIS or DESIGN determines that a second repository is required, Operator must invoke ForgeKit's existing promotion operation.

Conceptually:

```text
SPEC-0042
    ↓
second participant discovered
    ↓
promote-to-thread
    ↓
THREAD-0017
```

Operator must then resolve state again using the newly authoritative THREAD.

It must respect the original SPEC's promoted/superseded state and must not continue executing against the former authority.

## 26. Parallelism

Operator v1 is not required to execute independent repositories or milestones concurrently.

Sequential orchestration is acceptable.

Dependency-aware parallelism may be added later.

Operator v1 architecture must not prevent future parallel execution, but correctness, determinism, and resumability take precedence over throughput.

## 27. Pull Requests

Operator must use ForgeKit's existing PR workflow.

Before creating a PR, Operator must verify that the milestone is in the ForgeKit state that permits PR creation.

Operator must preserve lineage to the appropriate SPEC or THREAD, PLAN, milestone, and external work reference when one exists.

If the PR already exists, Operator must not create another.

If external review or merge is required, Operator must enter an appropriate waiting or external-action state.

## 28. Environment Promotion

Operator must use ForgeKit's existing environment governance model.

Environment governance levels are:

```text
ci
promotion
release
```

For `ci`, external automation may deploy without a ForgeKit approval ceremony.

For `promotion`, Operator may invoke the lightweight promotion workflow using the existing candidate-ref model.

For `release`, existing RELEASE-PLAN governance applies.

Operator must not introduce a separate environment lifecycle.

## 29. Release Approval Integrity

Release approval must be bound to the exact release subject approved.

For multi-repository releases, existing participant and coordination approval behavior remains authoritative.

Single-repository release approval must also be invalidated when material release identity changes after approval.

At minimum, ForgeKit must be able to determine that the approved candidate and release-plan revision still correspond to the release being executed.

Conceptually:

```yaml
approval:
  approved: true
  approved_by: ...
  approved_at: ...
  candidate_ref: ...
  release_plan_revision: ...
```

The exact schema is a DESIGN decision.

The invariant is:

> Operator must never execute a release using approval that applies to an earlier candidate or materially different release plan.

## 30. Release Resumability

If Operator is permitted to execute the existing release workflow, release execution must be resumable without conversation memory.

Operations such as migrations, deployment, tagging, and subsequent release steps must persist progress before later irreversible operations occur.

Conceptually:

```yaml
deployment_progress:
  migrations_applied: true
  deployed: false
  tagged: false
```

The exact implementation belongs in DESIGN.

If autonomous production execution is disabled by policy, Operator may stop at `PRODUCTION_READY`.

Production deployment automation is not required for Operator v1 success.

## 31. Autonomy Policy

Operator behavior must be policy-controlled.

Policy should determine which otherwise-valid ForgeKit actions may execute autonomously.

Conceptually:

```yaml
operator:
  max_review_rounds: 3

  allow:
    create_artifacts: true
    revise_artifacts: true
    implement_code: true
    run_tests: true
    create_pr: true
    nonproduction_promotion: true

  require_human:
    risk_acceptance: true
    production_release: true
```

The exact schema is not defined by this SPEC.

Operator policy may only make execution more restrictive.

It must never override a stricter ForgeKit gate.

## 32. Safety Invariants

Operator must never:

1. Bypass a blocking finding.
2. Mark its own unresolved finding as resolved without the normal review path.
3. Change approval requirements to allow progress.
4. Treat missing evidence as successful evidence.
5. Treat blocked or unavailable state as PASS.
6. Ignore staleness.
7. carry approval forward when ForgeKit requires reapproval.
8. overwrite immutable review history.
9. bypass Interface Freeze.
10. bypass XDEP requirements.
11. accept risk on behalf of a human.
12. silently reduce feature scope to make execution easier.
13. fabricate external evidence.
14. treat conversational context as more authoritative than repository state.
15. continue from an ambiguous state merely because one next action appears plausible.
16. send potentially incomplete implementation work to review after an interrupted implementation run.
17. break a dependency cycle without an explicit authorized decision.
18. treat an unconfigured or malformed Autonomy Policy as granting any autonomous permission — absence of policy must default to requiring human confirmation for every otherwise-autonomous action (see §49 Security Considerations, AC-25).

## 33. Completion Boundaries

Operator must describe completion precisely.

Supported states should include equivalents of:

```text
IMPLEMENTATION_COMPLETE

QA_READY

RELEASE_READY

PRODUCTION_READY

PRODUCTION_COMPLETE
```

A generic `DONE` must not be used when required downstream lifecycle work remains.

For multi-repository work, the completion state must represent thread-wide status.

## 34. Metrics

Operator must emit enough execution data to support later ForgeKit Evaluator work.

At minimum, ForgeKit must be able to derive:

### Automation Coverage

The percentage of eligible lifecycle actions completed without human intervention.

### Human Intervention Count

The number of Operator stops requiring a person.

### Human Escalation Reason

Whether intervention resulted from approval, product ambiguity, risk acceptance, environment failure, dependency deadlock, external action, non-convergence, or Operator limitation.

### Adversarial Convergence

The number of review/revision rounds before a gate passes.

### Non-Convergence Rate

How often review loops reach the configured autonomous limit.

### Gate Compliance

Whether every required gate was satisfied before dependent work began.

### Rework

How frequently execution returns to an earlier lifecycle stage.

### Execution Recovery

How often an Operator run successfully resumes after interruption.

### Operator Failure Rate

Failures caused by Operator orchestration rather than the engineering work itself.

A UI or metrics dashboard is not required.

## 35. Acceptance Criteria

### AC-01: Single-command continuation

Given a valid ForgeKit SPEC with sufficient information, when the user invokes Operator, Operator determines the current state and begins the next permitted ForgeKit activity without requiring the user to identify the next command.

### AC-02: Mid-lifecycle entry

Given work that already has approved upstream artifacts, Operator begins at the first incomplete or invalid downstream stage and does not recreate completed valid artifacts.

### AC-03: Shared state resolution

Operator and `/status` derive lifecycle state from the same resolver and cannot independently disagree about the next valid action.

### AC-04: Pre-PLAN state resolution

`/status` can resolve SPEC, ANALYSIS, DESIGN, OBSERVABILITY, and SECURITY state before PLAN exists.

### AC-05: Full milestone resolution

`/status` and Operator correctly distinguish all existing milestone states including `pending`, `in_progress`, `approved`, `test_failed`, `tests_passed`, `pr_open`, and `pr_merged`.

### AC-06: Structured review gates

Operator determines blocking-finding counts and review verdicts from structured artifact data rather than parsing prose.

### AC-07: Review correction loop

Given a review containing blocking findings that can be resolved autonomously, Operator invokes the existing correction workflow, re-runs review, and advances only after the blocking count reaches zero.

### AC-08: Non-convergence protection

Given a recurring or oscillating blocking finding that reaches the configured autonomous limit, Operator stops with `HUMAN_DECISION_REQUIRED` and does not bypass the gate.

### AC-09: Interrupted Operator session

Given an Operator session terminating between lifecycle commands, a new session can invoke Operator for the same work and resume from authoritative ForgeKit state without conversation history.

### AC-10: Interrupted implementation

Given `/implement-plan` terminating before implementation completes, a resumed Operator does not proceed directly to implementation review until ForgeKit can establish that implementation execution completed.

### AC-11: Idempotent artifact creation

Given a command completed before an Operator interruption, resuming does not duplicate the artifact, review, branch, thread, or PR.

### AC-12: Test failure classification

Given a test failure requiring QE classification, Operator stops until a structured `production_bug` or `test_bug` decision exists and then follows the appropriate existing ForgeKit path.

### AC-13: Human approval preservation

Given an existing ForgeKit human approval gate, Operator cannot satisfy, bypass, or remove the approval itself.

### AC-14: Single-repository release approval integrity

Given a release plan that changes after approval, Operator does not execute release using approval bound to the previous candidate or release-plan revision.

### AC-15: Multi-repository orchestration

Given a THREAD containing multiple participant repositories, Operator resolves participant state, XDEPs, Interface Freeze, plans, milestones, cross-repo validation, and release eligibility without treating an individual participant's completion as thread completion. Split into independently verifiable sub-criteria (per ANALYSIS-0001 `[QE-2]`):

#### AC-15a: Participant and XDEP resolution

Given a THREAD with multiple participants, Operator resolves each participant's lifecycle state and each XDEP's live satisfaction independently, with no reliance on a cached satisfaction value.

#### AC-15b: Interface Freeze respect

Given a THREAD with an unfrozen or revised Cross-Repo Contract, Operator does not advance implementation work that depends on a contract revision that has not been approved.

#### AC-15c: Per-repo PLAN/TEST-PLAN readiness

Given a THREAD, Operator determines each participant's PLAN/TEST-PLAN readiness independently and does not advance one participant's implementation on the basis of another participant's readiness.

#### AC-15d: Cross-repo validation

Given a THREAD requiring cross-repo scenario validation, Operator does not consider thread-wide implementation complete until the required cross-repo scenarios resolve to a PASS-equivalent outcome.

#### AC-15e: Release-eligibility derivation

Given a THREAD, Operator derives thread-wide release eligibility from the combination of AC-15a through AC-15d rather than from any single participant's state.

### AC-16: Live XDEP evaluation

Operator determines blocking XDEP satisfaction from authoritative participant state rather than relying on a persisted satisfaction cache.

### AC-17: Dependency deadlock

Given mutually blocked participants with no eligible action, Operator detects the deadlock, stops, identifies the dependency cycle, and does not spin or arbitrarily break it.

### AC-18: Promotion during discovery

Given a single-repository SPEC whose ANALYSIS or DESIGN discovers another required participant, Operator invokes the existing promotion workflow and continues using the resulting THREAD as authoritative work.

### AC-19: PR waiting

Given a PR requiring external review or merge, Operator enters an appropriate waiting/external-action state and does not create duplicate PRs or assume merge occurred.

### AC-20: Production boundary

Given policy requiring human production approval or execution, Operator advances work to the appropriate production-ready state and stops before the protected operation.

### AC-21: Release resumption

If Operator is configured to invoke release execution, interruption after an irreversible release step can be safely resumed without depending on conversational memory.

### AC-22: Ambiguous state

Given ForgeKit state that does not match a recognized lifecycle transition, Operator stops with an explicit ambiguous-state outcome rather than guessing.

### AC-23: Existing commands remain authoritative

All engineering reasoning and artifact generation continues through existing ForgeKit commands. Operator does not create parallel implementations of DESIGN, PLAN, review, testing, implementation, or release reasoning.

### AC-24: Metrics capture

After an Operator run, sufficient orchestration data exists to calculate autonomous actions, human interventions, review iterations, retries, recovery events, gate compliance, and failure categories.

### AC-25: Fail-closed autonomy default

Given a work unit with no Autonomy Policy configured, Operator treats every otherwise-autonomous action as requiring human confirmation rather than assuming permission. (Resolves ANALYSIS-0001 `[QE-1]`.)

### AC-26: Invalid or already-terminal work unit

Given `/operate` invoked against a work-unit id that does not exist, or one that exists but has no further eligible lifecycle action (e.g., already `promoted` or `shipped` with nothing downstream pending), Operator reports an explicit not-found or already-complete outcome rather than an ambiguous or silent no-op. (Resolves ANALYSIS-0001 `[QE-3]`.)

### AC-27: Security or policy exception escalation

Given a point in the lifecycle where a security exception or policy exception would be required to proceed (e.g., a SEC finding requiring an `accepted_risk` disposition), Operator stops with `HUMAN_DECISION_REQUIRED` and does not record or imply a risk-acceptance decision on its own behalf. (Resolves ANALYSIS-0001 `[QE-4]`.)

## 36. Primary End-to-End Scenario

Given an approved and sufficiently defined ForgeKit SPEC:

```text
User invokes /operate.

Operator resolves current state.

Operator advances through DESIGN and required reviews.

Blocking design findings are corrected and reviewed again.

Operator advances through observability and security requirements.

Operator creates and advances PLAN.

Operator creates and advances TEST-PLAN.

Operator selects the first eligible milestone.

Implementation executes.

Implementation review executes.

Blocking implementation findings are corrected.

Implementation is approved.

Tests execute.

Failed tests follow the proper correction path.

Passing tests allow PR creation.

External PR conditions are respected.

The milestone reaches pr_merged.

Remaining eligible milestones execute.

For multi-repo work, participant ordering and XDEPs are respected.

Required system verification occurs.

RELEASE-PLAN is created and reviewed.

Release readiness is evaluated.

Operator stops at any mandatory human production gate.

After the gate is satisfied, Operator can be invoked again and continue.
```

At no point should the user need to manually determine the next ForgeKit command.

## 37. Recovery Scenario

Given an Operator run that terminates during implementation:

```text
A new Claude session begins.

User invokes /operate for the same work.

Operator reads authoritative ForgeKit state.

Operator discovers that implementation_execution began but did not complete.

Operator does not invoke implementation review.

Operator follows the defined implementation recovery path.

Once implementation is complete, normal review proceeds.
```

No conversation history is required.

## 38. Multi-Repo Scenario

Given a THREAD spanning three repositories:

```text
Operator resolves coordination state.

Operator verifies participants and XDEPs.

Shared design and Interface Freeze satisfy existing ForgeKit requirements.

Repo-specific plans are created and approved.

Operator derives eligible implementation work.

Participant milestones execute in valid dependency order.

XDEP satisfaction is derived from live participant state.

Required cross-repo tests are evaluated.

Exact candidate refs are retained.

Thread-wide release readiness is evaluated.

Completion is not reported until the required thread-wide state is satisfied.
```

## 39. Non-Convergence Scenario

Given a DESIGN review that repeatedly raises the same material finding:

```text
Round 1 raises finding.

Operator invokes the normal revision workflow.

Round 2 identifies the issue as recurring.

Operator revises again if policy permits.

The finding recurs until the configured limit.

Operator stops with HUMAN_DECISION_REQUIRED.

The escalation identifies the finding lineage and prior attempts.

No approval or downstream work occurs.
```

## 40. Dependency Deadlock Scenario

Given two participant repositories where each is blocked by an unresolved dependency on the other:

```text
Operator resolves the THREAD.

No participant has an eligible implementation action.

The unresolved dependency graph contains a cycle.

Operator stops.

The user receives a decision package identifying the cycle.

Operator does not retry indefinitely.

Operator does not choose which dependency to break.
```

## 41. Operator Readiness Prerequisites

Before autonomous `/operate` execution is considered ready, ForgeKit must provide the following control surface:

```text
Structured current review outcome for DESIGN, PLAN, TEST-PLAN,
IMPL-REVIEW, and RELEASE-PLAN.

A /status resolver covering the supported lifecycle.

Structured finding recurrence lineage.

Persistent implementation execution completeness.

Canonical BUG lifecycle states.

Structured test-failure classification.

Release approval binding sufficient to detect invalid approval reuse.

Persisted release execution progress if Operator can invoke execute-release.

A way for an existing ForgeKit command to recognize that it is being driven by
Operator under explicit, policy-permitted autonomy — as distinct from a human
invoking it directly — so that its own embedded "ask the user to confirm"
Handoff step can be satisfied autonomously only where §48's distinction
between formal gates and routine draft-commit confirmations permits it.
```

These are prerequisites inside the ForgeKit Operator feature.

They do not constitute a separate product architecture.

**Resolution of the scope-split question (ANALYSIS-0001 Round 1, `[SC-1]`):** these prerequisites remain part of this SPEC rather than becoming a separate SPEC or thread. They are independently valuable and will be sequenced as the earliest milestones in this feature's own PLAN — before any milestone that implements `/operate` itself — mirroring the sequencing already recommended by the gap analysis this SPEC is based on. Keeping them in one SPEC preserves a single thread of traceability from that gap analysis through to the Operator command surface that depends on it; splitting would require re-deriving the same dependency ordering across two SPECs for no governance benefit, since both pieces are authored, reviewed, and delivered in immediate sequence by the same team.

## 42. Explicit Architectural Decisions

The following decisions are part of this SPEC:

```text
Existing ForgeKit artifacts remain authoritative.

Existing ForgeKit commands remain the engineering workers.

The Operator does not introduce a generic workflow engine.

The Operator does not introduce a parallel engineering-state store.

The Operator uses the same state resolver as /status.

Review outcomes required for control flow must be structured.

Implementation completion must be distinguishable from implementation start.

XDEP satisfaction is live-derived in v1.

THREAD completion is derived rather than persisted.

Sequential multi-repo execution is acceptable in v1.

Human escalation is correct behavior when judgment is materially required.

Production deployment autonomy is not required for v1 success.

Autonomy Policy is configured within each project's existing project.yml, not a new config file.

Operator is invoked only by a human, in a single continuously-running session; no scheduler, daemon, or webhook re-triggers it unattended in v1.
```

## 43. Future Compatibility

The Operator must remain usable as a lower-level capability beneath a future Product Operator.

Future architecture should be able to operate as:

```text
Product / Change Intake
        ↓
Product reasoning
        ↓
approved engineering change
        ↓
ForgeKit Operator
        ↓
governed engineering delivery
```

The ForgeKit Operator must therefore remain independent of how the engineering work originated.

Product discovery, idea ingestion, continuous product learning, portfolio orchestration, and similar higher-level capabilities must compose with Operator rather than require replacing it.

## 44. Success Definition

ForgeKit Operator v1 is successful when a developer can provide a valid ForgeKit engineering work unit, invoke one Operator entry point, and allow ForgeKit to proceed autonomously until either the requested delivery boundary is reached or a genuinely meaningful reason requires human involvement.

The target is not maximum AI activity.

The target is:

> **Maximum safe autonomous lifecycle coverage while preserving ForgeKit's quality, governance, traceability, and accountable human decision boundaries.**

---

*The following sections were added in v2 to satisfy `/create-spec`'s required-section quality bar, in response to ANALYSIS-0001 Round 1. They restate and organize commitments already made in §1–§44 above as discrete, traceable Goals/User Stories/NFRs/Constraints, rather than introducing new scope.*

## 45. Goals

- **G1 — Single-command continuation.** A developer can advance an existing ForgeKit work unit (SPEC-NNNN or THREAD-NNNN) through every ForgeKit-permitted lifecycle stage by invoking a single Operator entry point, without manually identifying which command runs next. → AC-01, AC-02
- **G2 — Shared state resolution.** Operator and `/status` derive lifecycle state from one shared resolver, so they never disagree about current state or the next valid action. → AC-03, AC-04, AC-05
- **G3 — Structured gate evaluation.** Gate evaluation (review verdicts, blocking-finding counts, approval status) is always derived from structured artifact data, never from parsing prose. → AC-06
- **G4 — Bounded autonomous convergence.** Operator drives review/revision loops autonomously up to a configured limit, and detects non-convergence (recurrence, oscillation, limit exceeded) rather than looping indefinitely or bypassing a gate. → AC-07, AC-08
- **G5 — Safe interruption and resumption.** An Operator run is safe to interrupt and resume — a fresh session reconstructs enough state from repository artifacts alone, with no reliance on conversation history, and never duplicates a side effect or advances partially-completed work. → AC-09, AC-10, AC-11
- **G6 — Preserved human-judgment boundaries.** Every human-judgment boundary already present in ForgeKit (approval gates, risk acceptance, security/policy exceptions, non-convergence, production release) remains a hard stop Operator cannot satisfy on its own behalf, and Operator never assumes autonomous permission it was not explicitly granted. → AC-12, AC-13, AC-20, AC-25, AC-27
- **G7 — Correct multi-repo orchestration.** Operator supports multi-repository THREAD work: participant/XDEP/Interface-Freeze state is resolved live, thread completion is never inferred from one participant, and mutual dependency deadlocks are detected and escalated rather than spun on or broken automatically. → AC-15 (and AC-15a–e), AC-16, AC-17, AC-18
- **G8 — Release-approval integrity and resumability.** Release execution, once Operator is permitted to invoke it, remains bound to the exact approved candidate/release-plan revision and is resumable without conversational memory. → AC-14, AC-21
- **G9 — Precise, honest stopping conditions.** Operator never represents a blocked, ambiguous, or incomplete outcome as success, and names its stopping condition precisely enough to resume correctly. → AC-19, AC-22, AC-23, AC-26
- **G10 — Orchestration metrics without duplicated engineering state.** Operator emits enough orchestration data after a run to support later evaluation of automation coverage, escalation causes, convergence behavior, and failure categorization, without duplicating engineering-artifact content. → AC-24

Every AC in §35 maps to at least one Goal above; every Goal maps to at least one AC.

## 46. User Stories

- As a **developer** operating a ForgeKit work unit, I want to invoke one command and have Operator determine and perform the next valid ForgeKit action, so that I don't have to track pipeline state manually across many sessions.
- As a **Lead Developer / reviewer**, I want Operator to stop and hand me a concrete decision package whenever a review cannot converge autonomously, so that I only get pulled in when my judgment is actually required, with enough context to act immediately.
- As an **SRE**, I want Operator to advance a release up to — but never past — a production action without my explicit confirmation, so that autonomous orchestration never exposes me to an unreviewed production change.
- As a **security reviewer**, I want Operator to escalate security and policy exceptions to me rather than resolve them itself, so that risk-acceptance decisions remain accountable to a named person.
- As a **TPO**, I want a SPEC that gets promoted into a THREAD mid-flow to stay correctly superseded, so that Operator never continues executing against a SPEC that is no longer authoritative.

## 47. Non-Functional Requirements

- **Reliability / consistency.** Every Operator state transition must re-resolve authoritative state immediately before and after invoking a ForgeKit command (§15); it must never act on a state snapshot older than the immediately preceding resolution. Verification path: scenario-based fixture testing (real two-repo git fixtures, real interruption/resume), the same methodology already used for `/promote-to-thread`'s ordering invariant — not a numeric threshold.
- **Idempotency.** Every side-effecting transition (artifact creation, review creation, thread promotion, branch creation, implementation attempt, PR creation, environment promotion, release operation) must be safe to re-invoke without duplication (§13). Verification path: scenario-based (re-invoke after simulated interruption, assert no duplicate artifact/branch/PR) — not a numeric threshold.
- **Autonomy containment** *(new — resolves `[LD-10]`)*. An Operator run must enforce a configurable upper bound on total autonomous side-effecting actions taken in a single run, independent of `max_review_rounds`, so that a state-resolution defect cannot produce unbounded real commits, branches, or PRs before a human notices. The requirement that such a bound exists and is enforced is a SPEC-level requirement; the default numeric value is deferred to DESIGN (§51).
- **Forward compatibility.** Operator must remain invocable against a work unit regardless of how that work unit originated (direct ForgeKit authoring or external work intake, per §4), and must compose beneath a future Product Operator without modification (§43).
- **Operability.** Operator introduces no new infrastructure dependency beyond what ForgeKit commands already require (git, optionally `gh`) — see Constraints & Assumptions (§48) for the execution model this implies.
- **Security.** See §49; not restated here to avoid duplication.

## 48. Constraints & Assumptions

- **Execution model** *(resolves `[LD-8]`)*. `/operate` runs as a single, continuously-executing agent session for as long as a human has it running. ForgeKit introduces no scheduler, daemon, or webhook to re-trigger Operator unattended — consistent with Non-Goals (§5) excluding "unrestricted autonomous production deployment" and with ARCH.md's characterization of ForgeKit as having no runtime or deploy target of its own. When an Operator run is interrupted (session closed, crashed, or simply ended for the day), resumption is always via a fresh, manually-initiated `/operate` invocation by a human — never an automatic re-trigger. Unattended re-triggering is explicitly out of scope for v1, not merely unaddressed.
- **Human-confirmation scope** *(resolves `[LD-9]`)*. Two distinct kinds of "pause" exist in current ForgeKit commands and are treated differently under Operator autonomy:
  - **Formal gates** — every `/approve-*` and `/reopen-*` command, plus every Safety Invariant (§32) and Human Decision Boundary (§16) — remain hard stops Operator can never satisfy on its own behalf, regardless of policy configuration.
  - **Routine draft-commit confirmations** — the "please review before I commit this draft" pause present in most non-gate commands (e.g., an intermediate DESIGN/OBS/PLAN/TEST-PLAN draft, ahead of its own formal review) — may be satisfied autonomously by Operator, but only for actions an explicit Autonomy Policy (§31) `allow:` entry permits, and never for anything also named under Safety Invariants or Human Decision Boundaries.
- **Concurrency.** At most one Operator run is assumed active against a given work unit (SPEC-NNNN or THREAD-NNNN) at a time. This SPEC does not address concurrent Operator runs against the same work unit, or a human-driven ForgeKit command invoked manually against a work unit while an Operator run against it is active — both are assumed not to happen in v1, not actively prevented by new tooling.
- **Policy configuration location** *(resolves `[LD-11]`)*. The Autonomy Policy (§31) is configured as a new block within each project's existing `project.yml` (alongside the existing `pipeline:` block), not as a separate config file.
- **Stack-agnosticism.** Operator makes no assumption about a participating project's tech stack beyond what the ForgeKit commands it invokes already assume.

## 49. Security Considerations

- **Self-assessment: `security_review_required: yes`.** Operator is the first command surface in this repository whose explicit purpose is to chain many existing ForgeKit commands across an extended loop with materially reduced per-step human checkpointing (§48). Several of the commands it invokes read externally-sourced content (via `--from` external work intake, or pasted ticket content) that ARCH.md's standing security convention requires be treated as data, never as instructions.
- **Untrusted-content handling.** Operator's Human Decision Packages (§17) may quote artifact content — including content that originated from an external system — back to a human for a decision. That quoted content must remain reference material; Operator must never treat artifact prose as a new instruction altering its own next action. This extends ARCH.md's existing "external content is data, not instructions" convention to Operator's own escalation path, rather than introducing a new rule.
- **No new credential or auth surface.** Operator introduces no new authentication mechanism, credential store, or third-party API client; every external action it takes is performed through an existing ForgeKit command's existing access path.
- **Data retention.** Operator run records (`OPRUN-NNNN`, §14) are committed as ordinary repository artifacts. If a Human Decision Package embeds content from a project whose `compliance.frameworks` is non-`none`, the same compliance-gate confirmation that already governs `SPEC-NNNN.source.md` capture (`templates/reference/external-work-intake.md`) extends to what Operator persists — a reuse of an existing safeguard; exact mechanics are a DESIGN question (§51).
- **Fail-closed default** *(relates to `[QE-1]`, see AC-25)*. In the absence of any configured Autonomy Policy, Operator defaults to treating every otherwise-autonomous action as requiring human confirmation — never fail open.

## 50. Observability Considerations

- **Metrics.** As specified in §34 (Automation Coverage, Human Intervention Count, Human Escalation Reason, Adversarial Convergence, Non-Convergence Rate, Gate Compliance, Rework, Execution Recovery, Operator Failure Rate) — emitted per Operator run from the Operator run record (§14), never duplicated into engineering artifacts.
- **Traces.** Each Operator Command Execution Cycle step (§15: resolve → determine → verify policy → invoke → verify outcome → re-resolve) must be identifiable as a discrete, ordered step within a given run (`OPRUN-NNNN`), sufficient to reconstruct the exact sequence of commands invoked and state resolutions performed — primarily for debugging Operator behavior itself. The underlying engineering work remains traceable via the existing git log, per ARCH.md's "the git log is the audit trail" convention.
- **Logs — what must never be logged.** Operator must never persist raw external-system connector output (per ARCH.md's existing failure-diagnostic sanitization convention — sanitized category only), raw credential-shaped strings, or the full unredacted content of a project's compliance-sensitive artifacts beyond what the invoking ForgeKit command's own output contract already permits. A Human Decision Package may quote the specific finding or artifact excerpt relevant to the decision, but must not become a mechanism for bulk-copying sensitive artifact content into a new, separately-retained location.

## 51. Open Questions

- **Exact `resolved_state` schema** (§7) — field names/types for the structured `/status` output. *Resolved at: DESIGN.*
- **Exact `implementation_execution` representation** (§12) — where and how "started vs. completed" is persisted. *Resolved at: DESIGN.*
- **Exact Operator outcome name set** (§18) — `COMPLETE`/`HUMAN_DECISION_REQUIRED`/etc. may be renamed or extended. *Resolved at: DESIGN.*
- **Exact release-approval binding schema** (§29) — fields proving an approval is bound to the current candidate/release-plan revision. *Resolved at: DESIGN.*
- **Exact `deployment_progress` schema** (§30) — fields tracking migration/deploy/tag progress for release resumability. *Resolved at: DESIGN.*
- **Exact Autonomy Policy schema and autonomy-containment default** (§31, §47) — the `allow`/`require_human` field set, and the default numeric value for the per-run action cap. *Resolved at: DESIGN.*
- **Mechanics of extending the compliance-gate confirmation to Operator-persisted content** (§49) — exact trigger points and file shape. *Resolved at: DESIGN.*
- **Exact mechanism for Operator-driven invocation recognition** (§41, new in v3) — whether a command distinguishes Operator-driven, policy-permitted invocation from direct human invocation via a flag, an environment marker, or another convention. *Resolved at: DESIGN.*

None of the above is fundamental enough to change a Goal or Non-Goal if answered differently — each is a representation detail within an already-decided architectural direction (§42).

---

## Revision Summary — Round 1

**Date:** 2026-10-07

### Addressed
- `[SC-1]`: Scope-split question (prerequisites vs. Operator) — kept as one SPEC; rationale and PLAN-level sequencing commitment added to §41.
- `[LD-1]`: No Goals section — added §45, with full Goal↔AC traceability.
- `[LD-2]`: No User Stories section — added §46.
- `[LD-3]`: No Non-Functional Requirements section — added §47, including the new autonomy-containment NFR.
- `[LD-4]`: No Constraints & Assumptions section — added §48, including execution-model and concurrency assumptions.
- `[LD-5]`: No Security Considerations section — added §49, with `security_review_required: yes` self-assessment and rationale.
- `[LD-6]`: No Observability Considerations section in the required shape — added §50 (metrics/traces/logs, including what must never be logged).
- `[LD-7]`: No Open Questions section — added §51, collecting every item previously deferred inline to DESIGN.
- `[LD-8]`: Execution-model ambiguity — resolved in §48: Operator is a single human-reinvoked session; no daemon/scheduler in v1. Also recorded in §42.
- `[LD-9]`: Ambiguity over which Handoff confirmations survive under autonomy — resolved in §48: formal gates vs. routine draft-commit confirmations now explicitly distinguished.
- `[LD-10]`: No circuit breaker on total autonomous actions — added as a new NFR in §47 (exact default value deferred to DESIGN per §51, which is appropriate since this is a threshold, not a missing requirement).
- `[LD-11]`: Autonomy Policy storage location unstated — resolved in §48 and §42: nests in each project's existing `project.yml`.
- `[QE-1]`: No fail-closed default for unconfigured policy — added AC-25, §49's fail-closed statement, and Safety Invariant #18.
- `[QE-2]`: AC-15 was a compound AC — split into AC-15a through AC-15e.
- `[QE-3]`: No boundary AC for invalid/already-terminal work-unit ids — added AC-26.
- `[QE-4]`: No AC/scenario for security/policy-exception escalation — added AC-27.

### Not Addressed
*(none — every flagged item from ANALYSIS-0001 Round 1 is listed above.)*

## Revision Summary — Round 2

**Date:** 2026-10-07

### Addressed
- `[LD-12]`: §48's resolution of `[LD-9]` (which confirmations survive under autonomy) exposed an adjacent, unstated mechanical precondition — existing commands have no way to recognize Operator-driven, policy-permitted invocation versus direct human invocation, so they cannot know to skip their own embedded confirmation step. Added the requirement to §41's prerequisite list and its deferred mechanism to §51 Open Questions — consistent with how every other mechanism in this SPEC is handled (state the requirement now, defer the representation to DESIGN).
- `[QE-5]`: §18's Operator Outcomes list didn't account for AC-26's not-found/already-complete case or the new autonomy-containment cap (§47). Added one clarifying sentence to §18 noting the list is extended by both, with exact naming deferred to DESIGN the same as the rest of the list.

### Not Addressed
*(none — both flagged items from ANALYSIS-0001 Round 2 are listed above.)*

## Product Validation

**Validated by:** Eric Pizon — TPO
**Date:** 2026-10-07
**Model:** claude-sonnet-5
**Status:** validated

### Acceptance Criteria — Plain Language Review

**AC-01 (original):** Given a valid ForgeKit SPEC with sufficient information, when the user invokes Operator, Operator determines the current state and begins the next permitted ForgeKit activity without requiring the user to identify the next command.
**In plain terms:** When you have a SPEC that's ready to move forward and you tell Operator to take over, it figures out on its own what to do next and starts doing it — you don't have to tell it which command to run.

**AC-02 (original):** Given work that already has approved upstream artifacts, Operator begins at the first incomplete or invalid downstream stage and does not recreate completed valid artifacts.
**In plain terms:** If some of the work is already done and approved, Operator picks up right where that left off instead of redoing anything already finished.

**AC-03 (original):** Operator and `/status` derive lifecycle state from the same resolver and cannot independently disagree about the next valid action.
**In plain terms:** If you ask Operator what's next, and separately ask `/status` what's next, you always get the same answer.

**AC-04 (original):** `/status` can resolve SPEC, ANALYSIS, DESIGN, OBSERVABILITY, and SECURITY state before PLAN exists.
**In plain terms:** Even before a PLAN exists, `/status` tells you exactly where things stand across the earlier stages.

**AC-05 (original):** `/status` and Operator correctly distinguish all existing milestone states including `pending`, `in_progress`, `approved`, `test_failed`, `tests_passed`, `pr_open`, and `pr_merged`.
**In plain terms:** Operator always knows exactly which of the seven possible states a piece of implementation work is in, and never confuses one for another.

**AC-06 (original):** Operator determines blocking-finding counts and review verdicts from structured artifact data rather than parsing prose.
**In plain terms:** Operator decides whether a review passed by reading a clear, labeled data field — never by trying to interpret a written sentence.

**AC-07 (original):** Given a review containing blocking findings that can be resolved autonomously, Operator invokes the existing correction workflow, re-runs review, and advances only after the blocking count reaches zero.
**In plain terms:** If a review comes back with fixable issues, Operator fixes them, gets it reviewed again, and only moves forward once there are zero blocking issues left.

**AC-08 (original):** Given a recurring or oscillating blocking finding that reaches the configured autonomous limit, Operator stops with `HUMAN_DECISION_REQUIRED` and does not bypass the gate.
**In plain terms:** If the same issue keeps coming back and hits the configured limit, Operator stops and asks a human instead of pushing through anyway.

**AC-09 (original):** Given an Operator session terminating between lifecycle commands, a new session can invoke Operator for the same work and resume from authoritative ForgeKit state without conversation history.
**In plain terms:** If Operator gets interrupted, a brand-new session pointed at the same work picks up correctly — it doesn't need to remember the previous conversation.

**AC-10 (original):** Given `/implement-plan` terminating before implementation completes, a resumed Operator does not proceed directly to implementation review until ForgeKit can establish that implementation execution completed.
**In plain terms:** If code-writing gets cut off halfway, Operator won't send that half-finished work to review — it notices it's incomplete and handles that first.

**AC-11 (original):** Given a command completed before an Operator interruption, resuming does not duplicate the artifact, review, branch, thread, or PR.
**In plain terms:** If something finished right before an interruption, resuming won't create a second copy of it.

**AC-12 (original):** Given a test failure requiring QE classification, Operator stops until a structured `production_bug` or `test_bug` decision exists and then follows the appropriate existing ForgeKit path.
**In plain terms:** When a human needs to judge whether a failing test reveals a real bug or a bad test, Operator waits for that judgment call before doing anything else.

**AC-13 (original):** Given an existing ForgeKit human approval gate, Operator cannot satisfy, bypass, or remove the approval itself.
**In plain terms:** Operator can never approve its own work on a step that's supposed to require a real person's sign-off.

**AC-14 (original):** Given a release plan that changes after approval, Operator does not execute release using approval bound to the previous candidate or release-plan revision.
**In plain terms:** If the release plan changes after someone approved it, Operator treats that approval as no longer valid rather than using it anyway.

**AC-15 (original, umbrella):** Given a THREAD containing multiple participant repositories, Operator resolves participant state, XDEPs, Interface Freeze, plans, milestones, cross-repo validation, and release eligibility without treating an individual participant's completion as thread completion.
**In plain terms:** When work spans several repositories, Operator tracks every repo's progress and every cross-repo dependency, and never assumes the whole project is done just because one repo finished.

**AC-15a (original):** Operator resolves each participant's lifecycle state and each XDEP's live satisfaction independently, with no reliance on a cached satisfaction value.
**In plain terms:** Operator checks each repo's real, current status — it never trusts an old, saved "yes, that's done" answer.

**AC-15b (original):** Operator does not advance implementation work that depends on a contract revision that has not been approved.
**In plain terms:** If the shared agreement between repos hasn't been finalized (or just changed), Operator won't let anyone start building against it yet.

**AC-15c (original):** Operator determines each participant's PLAN/TEST-PLAN readiness independently and does not advance one participant's implementation on the basis of another's readiness.
**In plain terms:** Each repo's plan has to be ready on its own — one repo being ready doesn't let a different repo start building.

**AC-15d (original):** Operator does not consider thread-wide implementation complete until required cross-repo scenarios resolve to a PASS-equivalent outcome.
**In plain terms:** If the project needs tests that check multiple repos together, Operator won't call it "done" until those pass.

**AC-15e (original):** Operator derives thread-wide release eligibility from the combination of AC-15a through AC-15d rather than from any single participant's state.
**In plain terms:** Whether the whole multi-repo project can release is decided by looking at everything above together, never just one repo.

**AC-16 (original):** Operator determines blocking XDEP satisfaction from authoritative participant state rather than relying on a persisted satisfaction cache.
**In plain terms:** Operator always double-checks a cross-repo dependency against the real, current state — it doesn't just trust a saved flag.

**AC-17 (original):** Given mutually blocked participants with no eligible action, Operator detects the deadlock, stops, identifies the dependency cycle, and does not spin or arbitrarily break it.
**In plain terms:** If two repos are each waiting on the other, Operator notices the standoff, stops, and explains exactly what's stuck — it doesn't retry forever or force a side through.

**AC-18 (original):** Given a single-repository SPEC whose ANALYSIS or DESIGN discovers another required participant, Operator invokes the existing promotion workflow and continues using the resulting THREAD as authoritative work.
**In plain terms:** If a single-repo feature turns out to need a second repo partway through, Operator converts it into a proper multi-repo project the normal way and keeps going.

**AC-19 (original):** Given a PR requiring external review or merge, Operator enters an appropriate waiting/external-action state and does not create duplicate PRs or assume merge occurred.
**In plain terms:** Once a pull request is open and waiting on a person, Operator waits — it won't open a second one, and won't assume it merged until it actually sees that it did.

**AC-20 (original):** Given policy requiring human production approval or execution, Operator advances work to the appropriate production-ready state and stops before the protected operation.
**In plain terms:** Operator gets everything ready for production, then stops right before actually touching it and waits for a person.

**AC-21 (original):** If Operator is configured to invoke release execution, interruption after an irreversible release step can be safely resumed without depending on conversational memory.
**In plain terms:** If a release gets interrupted after something that can't be undone, you can resume safely in a new session without needing to remember what already happened.

**AC-22 (original):** Given ForgeKit state that does not match a recognized lifecycle transition, Operator stops with an explicit ambiguous-state outcome rather than guessing.
**In plain terms:** If Operator ever hits a situation it doesn't recognize, it says so clearly and stops — it never just guesses.

**AC-23 (original):** All engineering reasoning and artifact generation continues through existing ForgeKit commands. Operator does not create parallel implementations of DESIGN, PLAN, review, testing, implementation, or release reasoning.
**In plain terms:** Operator never does the actual engineering thinking itself — the same ForgeKit commands as today still do design, planning, reviewing, testing, coding, and releasing; Operator just decides when to run them.

**AC-24 (original):** After an Operator run, sufficient orchestration data exists to calculate autonomous actions, human interventions, review iterations, retries, recovery events, gate compliance, and failure categories.
**In plain terms:** After a run, you can see exactly how much Operator did on its own, how often it needed a human, how many review rounds happened, and what went wrong, if anything.

**AC-25 (original):** Given a work unit with no Autonomy Policy configured, Operator treats every otherwise-autonomous action as requiring human confirmation rather than assuming permission.
**In plain terms:** If nobody has set up autonomy rules yet, Operator assumes it can't do anything on its own and asks a human for every step.

**AC-26 (original):** Given `/operate` invoked against a work-unit id that does not exist, or one that exists but has no further eligible lifecycle action, Operator reports an explicit not-found or already-complete outcome rather than an ambiguous or silent no-op.
**In plain terms:** If you point Operator at something that doesn't exist or is already fully finished, it tells you that clearly instead of silently doing nothing.

**AC-27 (original):** Given a point in the lifecycle where a security exception or policy exception would be required to proceed, Operator stops with `HUMAN_DECISION_REQUIRED` and does not record or imply a risk-acceptance decision on its own behalf.
**In plain terms:** If moving forward would require someone to officially accept a security or policy risk, Operator leaves that decision entirely to a human.

All translations above were presented to Eric Pizon (TPO) and confirmed as-is, with no corrections.

### Structural Questions

1. **Problem statement describes pain, not solution?**
   Confirmed — no gap identified by the TPO.

2. **Non-Goals specific enough to use as decision rules?**
   Confirmed — no gap identified by the TPO.

3. **Uncovered user scenario?**
   None identified by the TPO.

### AC-Specific Questions

For every AC above (AC-01 through AC-27, including AC-15a–e): *Is the plain-terms translation the complete definition of done, or is there an implied condition not stated?*

**Answer (covering all ACs):** "No gaps identified. Proceed." — Eric Pizon (TPO) reviewed the full set and did not flag an implied condition on any individual AC.

### Gaps Found

None.

### Sign-off

> Eric Pizon confirms: the plain-language translations are accurate and the questions above are answered. This SPEC is ready for design.
