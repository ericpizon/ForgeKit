---
id: review-test-plan
type: command
name: /review-test-plan
description: Quality Engineer command to run all Stage 5b test plan perspectives in one pass — Security, SRE, Lead Developer
status: active
version: 2
primary_role: quality-engineer
output_artifact_type: test-plan (reviewed)
changelog:
  - v2 — added Lead Developer perspective (Step 3): validates test case implementability with the stack, happy path trace alignment, and deferral criteria; output format and quality gate updated
links:
  - relation: relates_to
    target: create-test-plan
  - relation: relates_to
    target: address-test-plan-review
  - relation: relates_to
    target: approve-test-plan
---

# /review-test-plan

## Purpose

Run all Stage 5b test plan reviews in a single pass, adopting each reviewer's perspective in sequence. The Quality Engineer executes this command immediately after drafting TEST-NNNN, and again after each revision in `/address-test-plan-review`. Output is a `## Test Plan Review — Round N` section appended to TEST-NNNN. The QE then reads the findings and runs `/address-test-plan-review` or `/approve-test-plan` depending on whether blocking findings exist.

## When to Invoke

- After the user confirms the `/create-test-plan` draft — no commit required before review
- After `/address-test-plan-review` appends an Address Summary to TEST-NNNN (next review round)

Pass the TEST, SPEC, OBS, and (if applicable) SEC IDs: `/review-test-plan TEST-0001 SPEC-0001 OBS-0001 SEC-0001`.

## Before Running Perspectives

Verify TEST-NNNN contains all required sections before adopting any reviewer perspective. If a section is missing, stop and return to `/create-test-plan` to complete the draft.

Required sections:
- [ ] Test Strategy (unit/integration/e2e split, when each type is written)
- [ ] Acceptance Criteria Coverage Map organized by milestone (every AC from SPEC-NNNN mapped to test cases)
- [ ] Happy Path Coverage (one table per User Story from PLAN's Happy Path Trace — absence is noted but not a blocker here; gaps within the table are reviewed by Lead Developer)
- [ ] Observability Verification (if OBS-NNNN was produced)
- [ ] Security Test Cases (if SEC-NNNN was triggered)
- [ ] Test Data Strategy
- [ ] Execution Results Summary (placeholder — `status: pending`)

Required frontmatter fields:
- [ ] `status: draft`
- [ ] `target_plan_version` set
- [ ] `execution_date: null`
- [ ] `execution_summary: null`

Test case format: every test case must have an ID, type (`unit`/`integration`/`e2e`/`manual`), execution mode (`automated`/`manual`), and priority (`P0`–`P3`). Integration, E2E, and manual test cases must use Gherkin (Given/When/Then). Unit test cases use the simple Scenario/Input/Expected format. Manual test cases must include a `Procedure` block and a `Reason` line. If any test case is missing these fields or uses the wrong format for its type, stop and return to `/create-test-plan`.

## On Re-Review

If a `## Test Plan Review — Round N` section already exists in TEST-NNNN:

1. Read `current_review_round` from TEST-NNNN frontmatter. The new round number is `current_review_round + 1`.
2. Read the latest `## Address Summary — Round N` section in TEST-NNNN to see which findings were addressed and which were accepted or deferred.
3. Load only the TEST-NNNN sections cited in the address summary — do not re-read the full test plan. Sections not referenced by any addressed finding are unchanged.
4. For each perspective: verify that previously-blocking findings marked "Addressed" are actually resolved. Do not re-raise findings marked "Not Addressed" with documented accepted-risk rationale. **If a perspective's domain was unaffected by the address pass, emit `[Perspective]: no new findings — domain unaffected by Round N address.` and skip the full perspective analysis.**
5. Append a new `## Test Plan Review — Round N` section. Do not modify prior round sections.

## Steps

Adopt the Quality Engineer role. Apply the perspective and stack knowledge from [`.claude/roles/quality-engineer.md`](.claude/roles/quality-engineer.md) throughout these steps — particularly the test case format conventions (Gherkin for integration/E2E/manual; simple for unit), priority levels (P0–P3), and the quality bar for security and observability coverage.

### 1. Security Perspective *(only if SEC-NNNN exists)*

Adopt the Security role. Review every SEC finding in SEC-NNNN against the test cases in TEST-NNNN. Assess:

- **Finding coverage.** Every open or mitigated SEC finding has at least one test case. A finding with no test case is a blocking gap.
- **Gherkin scenario separation.** Enumeration prevention requires two separate Gherkin scenarios — one where the resource exists and one where it does not — both returning the same response body. A single scenario that only tests one case does not prove enumeration prevention — blocking finding.
- **Gherkin `Then` clause precision.** The `Then` clause must assert the exact response status and body, not just "the request fails." `Then the response status is 401 / And the response body is { error: "unauthorized" }` is sufficient. `Then the request is rejected` is not — blocking finding.
- **Priority compliance.** Every security test case for a Critical or High SEC finding must be P0. A High finding with a P1 or lower security test is a blocking finding.
- **No manual-only P0 security tests.** A P0 security test case marked `manual` must have a `Reason` line explaining why automation is not feasible. A manual P0 with no reason is a blocking finding.
- **Negative path coverage.** Every security Gherkin scenario must test the failing path. A scenario whose `Then` only asserts success for a security control is a blocking finding.

### 2. SRE Perspective

Adopt the SRE role. Review every signal in OBS-NNNN against the observability test cases in TEST-NNNN. Assess:

- **Signal coverage.** Every signal in OBS-NNNN has at least one verification test case. A missing primary signal is a blocking finding; a missing secondary signal is non-blocking.
- **Gherkin `Then` specificity.** Every observability test case's `Then` clause must name the expected value, label, or field — not just assert emission. `Then [metric-name]{label=value} increments by 1` is specific. `Then the metric is emitted` is not — non-blocking finding.
- **Priority compliance.** Primary signals (request counter, latency histogram, primary span) must be P1 or better. A primary signal verified only at P2 or lower is a non-blocking finding. Label correctness and log format tests may be P2.
- **Manual observability tests.** A manual observability verification step must include a step-by-step `Procedure` and a `Reason`. "Will check in dashboard" is not a procedure — flag it as non-blocking.

### 3. Lead Developer Perspective

Adopt the Lead Developer role. Read ARCH.md before reviewing — test case implementability depends on the project's actual testing stack, framework conventions, and DB isolation strategy.

Review TEST-NNNN for implementability — can the test cases as written actually be implemented with the project's testing stack? Assess:

1. **Stack feasibility.** For each test case, confirm it is implementable with the available tools (read ARCH.md for the test framework and testing conventions). Flag any test case that:
   - Requires a testing capability the project's test framework does not provide
   - Assumes a test helper, fixture, or factory that is not described in the Test Data Strategy
   - Requires intercepting or mocking something that the project's DB isolation rule prohibits (read ARCH.md for the team's testing conventions)
   - Assumes framework-specific behavior that is difficult to exercise in a test environment (read ARCH.md for framework and auth model)

2. **Given block completeness.** Every integration and E2E test case's `Given` block must fully establish the state it needs without relying on state left by another test. Flag any test where the `Given` block implies pre-existing data it does not create (e.g., "Given a resource exists" without a creation step). These will be flaky in CI.

3. **Correct test layer.** Tests must assert observable behavior — API response status, response body, UI state — not internal implementation details. Flag any test case that:
   - Queries the database directly to assert an outcome that should be verified via API
   - Asserts on internal function calls or module state
   - Tests implementation choice rather than specified behavior

4. **Happy path trace alignment.** Read the **Happy Path Coverage** section in TEST-NNNN. For each User Story trace:
   - Is every flow step mapped to a test case? An unmapped step is a **blocking** gap — a step with no test case means that part of the user journey is unverified even if all ACs pass.
   - Does the referenced test case actually exercise the flow step it claims to cover? A test case cited for "user submits form" that only validates client-side schema without touching the server endpoint is false coverage — **blocking**.
   - Do the flow step descriptions match PLAN-NNNN's Happy Path Trace exactly? Any reworded or paraphrased step is a **non-blocking** finding — PLAN is the single source for flow step text and divergence signals drift.
   - If the Happy Path Coverage section is absent, that is a **non-blocking** finding — surface it as missing and note the specific flow steps at risk.

5. **Deferral criteria compliance.** Review all test cases marked `deferred-to-release`. For each one:
   - Does it have a `Deferral Reason` line? Missing reason is a **blocking** finding.
   - Does the reason cite a genuine production-only constraint (OAuth callback, external telemetry endpoint, production infrastructure)? "Hard to automate" or "requires setup" is **not** a valid deferral reason — **blocking** if that is the only justification.
   - Is any deferred test case P0? A deferred P0 is always a **blocking** finding regardless of reason.

### 4. Cross-Perspective Check *(only if two or more perspectives ran)*

After completing all applicable perspectives:

- **Contradictions:** Does a Security test case require a specific response body that conflicts with what the SRE observability test expects to observe? Does a Lead Developer finding about test layer correctness affect a security test's validity? Flag contradictions.
- **Gaps:** Did the Security perspective reveal an outcome path (e.g., `unauthorized`) that the SRE perspective should also verify has correct metric labels? Did the Lead Developer find a Given block gap that creates a precondition the Security test also relies on? Add to the appropriate perspective's findings.

## Output

Append a `## Test Plan Review — Round N` section to TEST-NNNN. Then update `current_review_round` in TEST-NNNN frontmatter to the round number just appended.

**Null suppression:** Omit any `### [Perspective] Findings` section that has no findings — do not emit the section header or a "None." line. List clean perspectives in a single `**Clean perspectives:** [list]` line before `### Contradictions Requiring Arbitration`.

```
## Test Plan Review — Round N

**Date:** YYYY-MM-DD
**Status:** Changes requested — N blocking, N non-blocking | Approved — no blocking findings

### Security Findings  *(omit if SEC-NNNN does not exist or no findings)*
- [SEC-1] **blocking** — [finding]

### SRE Findings  *(omit section if none)*
- [SRE-1] **non-blocking** — [finding]

### Lead Developer Findings  *(omit section if none)*
- [LD-1] **blocking** — [finding]
- [LD-2] **non-blocking** — [finding]

**Clean perspectives:** [list any perspectives with no findings]

### Contradictions Requiring Arbitration
- [or "None."]
```

## Quality Gate

Before appending the review section:
- [ ] All three perspectives completed — Security (if SEC-NNNN exists), SRE, Lead Developer
- [ ] Lead Developer perspective: stack feasibility, Given block completeness, test layer correctness, happy path trace alignment, and deferral criteria all assessed
- [ ] Each finding labeled blocking or non-blocking with a reason
- [ ] Cross-perspective check completed
- [ ] Round number is correct

**Self-check before proceeding:** Re-read each item above. Confirm each is complete, or state what was skipped and why before continuing to Handoff.

## Handoff

Append the `## Test Plan Review — Round N` section to TEST-NNNN. Do not commit.

Show the user the review findings, then:

- If blocking findings exist → "Run `/address-test-plan-review TEST-NNNN` to address findings."
- If no blocking findings → "Run `/approve-test-plan TEST-NNNN` to close the test planning phase with a single commit."
