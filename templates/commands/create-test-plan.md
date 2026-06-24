---
id: create-test-plan
type: command
name: /create-test-plan
description: Quality Engineer command to produce TEST-NNNN — test strategy, AC coverage, happy path trace alignment, observability/security verification, and explicit deferral criteria
status: active
version: 1
primary_role: quality-engineer
reviewing_roles:
  - security
  - sre
  - lead-developer
output_artifact_type: test-plan
---

# /create-test-plan

## Purpose

Produce a TEST-PLAN (TEST-NNNN) that maps every acceptance criterion to a concrete test case, aligns test coverage with the PLAN's happy path trace, defines the test strategy (unit / integration / e2e mix), specifies observability verification, includes security test cases for each SEC finding, and explicitly reviews all deferred test cases against production-only deferral criteria. TEST-NNNN is authored and approved before implementation begins — it is the test contract the Developer works against.

## When to Invoke

- PLAN-NNNN `approved`
- SEC-NNNN `completed` (if triggered)
- OBS-NNNN `draft` or `approved`
- **Before `/implement-plan` begins** — TEST-NNNN must be approved before the Developer writes a single line of code

Pass artifact IDs as arguments: `/create-test-plan SPEC-NNNN PLAN-NNNN OBS-NNNN SEC-NNNN`.

## Required Inputs

- **SPEC-NNNN** — approved; Acceptance Criteria are the primary test-case source
- **PLAN-NNNN** — approved; task structure, milestone grouping, and Happy Path Trace inform test organization
- **OBS-NNNN** — read Validation Checklist section for observability verification steps
- **SEC-NNNN** — (if triggered) read Findings section for security test cases
- **ARCH.md** — read for the project's test framework, test patterns, and no-mock rules

## Steps

Adopt the Quality Engineer role. Read ARCH.md for the testing stack — unit runner, integration runner, e2e runner — and any project-specific testing rules (e.g., no DB mocks in integration tests) before drafting test cases.

### Step 1 — Initialize TEST-NNNN

Create the TEST-PLAN file with:
- `status: draft`
- `version: 1`
- `target_plan_version`: version of PLAN-NNNN being tested against
- `execution_date: null`
- `execution_summary: null`

Create the initial document structure with section headings for each required section, including an Execution Results Summary section with `status: pending`.

### Step 2 — Define test strategy

State the unit / integration / e2e split per ARCH.md (Testing Strategy section), and critically **when each type is written during implementation**:

- **Unit tests:** TDD-first for every pure function — failing test before implementation. No database, no network calls, no framework dependencies.
- **Integration tests:** Written alongside each API handler or service — test skeleton before or as the handler is written. Real database — follow ARCH.md's no-mock rule.
- **E2E tests:** Written as pages or flows are built; stubs when scaffolded, completed when the full flow is navigable.

Map the strategy to the specific ACs of this thread.

### Step 3 — Map every AC to test cases

For each AC in SPEC-NNNN, write one or more concrete test cases. Use Gherkin for integration and E2E cases (Given/When/Then); simple format for unit cases. Assign type, execution mode, and priority to each case.

Organize test cases by milestone — group the AC Coverage Map under each milestone heading matching the PLAN structure. This maps test cases to PRs.

Guidelines:
- Every AC must have at least one P0 automated test covering the happy path.
- Negative paths (invalid input, unauthorized access, not found) are P1 at minimum for authentication-adjacent ACs; P2 for others. Write a separate Gherkin scenario for each distinct outcome.
- If an AC genuinely cannot be automated, mark it `manual`, write it in Gherkin with a `Procedure` block and a `Reason` line. A P0 manual test without justification is a quality gate failure.

### Step 4 — Align test cases with the happy path trace

Read the **Happy Path Trace** section from PLAN-NNNN. For each User Story trace, walk each flow step and identify which test case from Step 3 exercises it. Produce a **Happy Path Coverage** table in TEST-NNNN:

```markdown
## Happy Path Coverage

### US1 — [User Story title]

| Flow Step | Test Case(s) |
|---|---|
| User navigates to feature | TC-FUNC-5 — GET returns list |
| User submits form | TC-FUNC-2 — POST happy path |
| Result appears | TC-FUNC-6 — list includes new record |

Gap check: [none — all steps covered | step N has no test case — see note]
```

If a flow step has no test case: either write a new test case for it, or document explicitly why no test is needed. Do not leave flow steps unverified without written justification.

### Step 5 — Identify edge cases and boundary conditions

For each AC, identify inputs at the boundary of valid/invalid for each constraint the AC defines. Write edge cases as Gherkin integration scenarios. Assign P2 unless they map to a security or data integrity path (P1 in that case).

### Step 6 — Specify observability verification *(if OBS-NNNN was produced)*

For each signal in OBS-NNNN, write a test case using the `TC-OBS-[signal]-[seq]` ID format. The `Then` clause must name the expected metric value, label set, or log fields explicitly — not "the metric is emitted." Assign type (`integration` if automated; `manual` if requiring dashboard inspection) and priority (primary signals P1; label correctness and log structure P2). Manual observability cases must include a step-by-step `Procedure` and a `Reason`.

### Step 7 — Specify security test cases *(if SEC-NNNN was triggered)*

For each SEC finding, write a test case using the `TC-SEC-[finding]-[seq]` ID format. Write a separate scenario for each distinct failure path. Assign priority based on finding severity: Critical/High → P0; Medium → P1; Low → P2. Every security scenario must test the failing path, not just the happy path.

### Step 8 — Define test data strategy

How is test data (records, credentials, users) created for each test? Tests must not share state — each test case sets up and tears down its own data. Describe the isolation mechanism for integration tests (e.g., database branches, transaction rollback, unique prefixes).

### Step 9 — Review and justify all deferred test cases

Identify every test case marked `deferred-to-release`. A test case may only be deferred if it **genuinely requires a production environment** to verify:

| Valid deferral reason | Example |
|---|---|
| Requires a real OAuth callback URL | Production OAuth callback |
| Requires a production-only exporter endpoint | OTLP endpoint only configured in production |
| Requires production-specific infrastructure | Serverless cold-start latency, production database |

A test case must **not** be deferred because it is difficult to automate, requires significant setup, or the QE prefers manual verification.

For each deferred test case, add a `Deferral Reason` line citing the production-only constraint.

**Hard limit: no P0 test case may be deferred to release under any circumstances.**

## Output Contract

TEST-NNNN must contain: Test Strategy; AC Coverage Map organized by milestone (milestone → AC → test cases, each with ID, type, execution mode, priority); Happy Path Coverage (one table per primary User Story); Observability Verification; Security Test Cases; Test Data Strategy; Execution Results Summary (placeholder). All `deferred-to-release` test cases must have a `Deferral Reason`. Frontmatter: `version`; `target_plan_version`; `execution_date: null`; `execution_summary: null`.

## Quality Gate

Before requesting review:
- [ ] Frontmatter complete: `status: draft`, `version: 1`, `target_plan_version`, `execution_date: null`
- [ ] Every test case has an ID, type, execution mode, and priority
- [ ] Every AC has at least one P0 automated test covering the happy path
- [ ] No P0 test case is `manual` without a `Procedure` block and `Reason` line
- [ ] Integration tests use a real database (no DB mocks, per ARCH.md rules)
- [ ] Happy Path Coverage section present; every flow step mapped or documented
- [ ] Observability verification covers each signal in OBS-NNNN
- [ ] Security test cases cover each finding in SEC-NNNN; Critical/High findings are P0
- [ ] All security test cases test the failing path
- [ ] Test data strategy ensures no shared state between test cases
- [ ] AC Coverage Map organized by milestone matching PLAN structure
- [ ] All `deferred-to-release` test cases have a valid `Deferral Reason`; no P0 deferred
- [ ] Execution Results Summary placeholder present

## Handoff

Show the user the TEST-PLAN file. Do not commit.

> "Here is the draft TEST-PLAN. Please review — once you confirm, run `/review-test-plan TEST-NNNN SPEC-NNNN OBS-NNNN SEC-NNNN` to run all perspectives in one pass."

The single commit for the test planning phase is owned by `/approve-test-plan`.
