---
id: create-test-plan
type: command
name: /create-test-plan
description: Quality Engineer command to produce TEST-NNNN — test strategy, AC coverage, happy path trace alignment, observability/security verification, and explicit deferral criteria
status: active
version: 4
primary_role: quality-engineer
reviewing_roles:
  - security
  - sre
  - lead-developer
output_artifact_type: test-plan
changelog:
  - v4 — frontmatter initialization now includes `updated`, which address-test-plan-review.md already assumed existed and updated on every version bump.
  - v3 — added the coordination-repo path: authors SYSTEM-TEST-PLAN.md and cross-repo E2E scenario YAML files for a thread whose dependencies need live, multi-service verification. For a thread participant repo, tags AC-covering test cases with cross_repo_scenarios where applicable.
---

# /create-test-plan

## Purpose

Produce a TEST-PLAN (TEST-NNNN) that maps every acceptance criterion to a concrete test case, aligns test coverage with the PLAN's happy path trace, defines the test strategy (unit / integration / e2e mix), specifies observability verification, includes security test cases for each SEC finding, and explicitly reviews all deferred test cases against production-only deferral criteria. TEST-NNNN is authored and approved before implementation begins — it is the test contract the Developer works against.

## When to Invoke

**`repo_role: code` (including thread participants):**
- PLAN-NNNN `approved`
  If PLAN-NNNN is `revision_requested`: **do not proceed.** "A plan back-cycle is in progress. Wait until PLAN-NNNN returns to `approved` after the review and approve-plan cycle completes."
- SEC-NNNN `completed` (if triggered)
- OBS-NNNN `draft` or `approved`
- **Before `/implement-plan` begins** — TEST-NNNN must be approved before the Developer writes a single line of code

**`repo_role: coordination`:** the thread's DESIGN-NNNN Cross-Repo Contract is approved (Interface Freeze, per `/approve-design`) — this does not wait on any participant's own PLAN/TEST-NNNN, since a thread's live-verification needs are a property of its XDEPs, not of any one repo's implementation plan.

Pass artifact IDs as arguments: `/create-test-plan SPEC-NNNN PLAN-NNNN OBS-NNNN SEC-NNNN`. For a coordination repo authoring the thread-level system test plan, pass the thread instead: `/create-test-plan THREAD-NNNN`.

## Pre-flight — Coordination repo check

Read `project.yml`'s `repo_role`. This step applies only when it's `coordination`; for `repo_role: code` (the default, including a thread participant), skip to "Required Inputs" below — nothing changes there except the one addition noted in Step 3's closing paragraph.

**If `repo_role: coordination`:** this invocation authors a thread-level system test specification, not a per-repo TEST-NNNN.

1. Read the thread's approved DESIGN-NNNN (specifically its Cross-Repo Contract) and every XDEP under `sdlc/dependencies/THREAD-NNNN-XDEP-*.yml`.
2. For each XDEP, determine whether its `verification` layers (schema-registry compatibility, consumer-deserialization, etc. — per [`templates/reference/xdep-schema.md`](../reference/xdep-schema.md), already authored on the XDEP itself in `create-design`) are sufficient on their own, or whether the dependency genuinely needs two or more participants deployed and running together to verify. **Most XDEPs need no scenario at all** — contract verification already covers them. Only `contract`-kind XDEPs with real cross-service behavior (not just a schema shape), or a flow spanning multiple XDEPs end-to-end, are candidates.
3. **If no XDEP needs a live scenario:** state this explicitly in the thread's registry entry and stop — do not create `SYSTEM-TEST-PLAN.md` or any `sdlc/scenarios/` files. This artifact is conditional, the same way OBS/SEC are conditionally triggered rather than mandatory (per [`templates/reference/xdep-schema.md`](../reference/xdep-schema.md)).
4. **If one or more scenarios are needed:** for each, author a scenario YAML at `sdlc/scenarios/THREAD-NNNN-E2E-NN.yml` per the schema in `templates/reference/xdep-schema.md` — entrypoint, expected outcome, `covers_xdeps`, `covers_acceptance_criteria`, an `implementation` block pointing to an executor-owned repo (**never** this coordination repo — no exception for a simple scenario), `owners`/`triage_owner`, environment profile, execution binding (with triggers and the `PASS`/`FAIL`/`BLOCKED_ENVIRONMENT`/`ERROR` result contract), and `status: active`.
5. Author `sdlc/THREAD-NNNN/SYSTEM-TEST-PLAN.md` — prose only: strategy, scope, rationale, which critical cross-repo workflows this thread verifies at the system level and why. Reference scenarios by ID; do not restate their field-level details — the scenario YAML is the sole source of truth for those (a duplicate `timeout` or `entrypoint` value here will eventually drift from the YAML and nothing will say which one is real).
6. For each scenario, note which participant repos it covers — each of those repos' own `create-test-plan` runs (per-repo, below) should reference it via `cross_repo_scenarios`.

Everything else in this command (Steps 1–9, Output Contract, Quality Gate) does not apply to the coordination-repo path — SYSTEM-TEST-PLAN.md has no AC Coverage Map, Happy Path Coverage, or per-milestone structure of its own; those belong to each participant's own TEST-NNNN.

## Required Inputs

- **SPEC-NNNN** — approved; Acceptance Criteria are the primary test-case source
- **PLAN-NNNN** — approved; task structure, milestone grouping, and Happy Path Trace inform test organization
- **OBS-NNNN** — read Validation Checklist section for observability verification steps
- **SEC-NNNN** — (if triggered) read Findings section for security test cases
- **ADR-NNNN** (any ADRs linked from DESIGN-NNNN) — read each ADR's Decision and Consequences sections; these are the source for invariant/boundary test cases in Step 3a
- **ARCH.md** — read for the project's test framework, test patterns, and no-mock rules

## Steps

Adopt the Quality Engineer role. Read ARCH.md for the testing stack — unit runner, integration runner, e2e runner — and any project-specific testing rules (e.g., no DB mocks in integration tests) before drafting test cases.

### Step 1 — Initialize TEST-NNNN

Create the TEST-PLAN file with:
- `status: draft`
- `version: 1`
- `updated: [today's date]` — `/address-test-plan-review` updates this on every version bump; it must exist from creation
- `target_plan_version`: version of PLAN-NNNN being tested against
- `execution_date: null`
- `execution_summary: null`
- `current_review_round: 0` (updated by `/review-test-plan` after each round)
- `schema_version: 1`
- `generation` block: `command: /create-test-plan`, `command_version: [use version from this command's frontmatter]`, `model` [self-identify from system context], `generated_at: [today's date]`, `session_id: null`

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

   **AC descriptions are not included in the Coverage Map.** Reference AC IDs only (e.g., `AC1`, not `AC1 — expires_at nullable TIMESTAMPTZ`). AC text lives in SPEC-NNNN; commands that need it read SPEC directly. Repeating descriptions here creates a second copy that drifts when SPEC is revised.

Guidelines:
- Every AC must have at least one P0 automated test covering the happy path.
- Negative paths (invalid input, unauthorized access, not found) are P1 at minimum for authentication-adjacent ACs; P2 for others. Write a separate Gherkin scenario for each distinct outcome.
- If an AC genuinely cannot be automated, mark it `manual`, write it in Gherkin with a `Procedure` block and a `Reason` line. A P0 manual test without justification is a quality gate failure.

**Thread participants only:** if the coordination repo's thread has a `SYSTEM-TEST-PLAN.md` with a scenario listing this repo in its coverage (per the Pre-flight above) or covering one of this repo's ACs via `covers_acceptance_criteria`, add a `cross_repo_scenarios: [THREAD-NNNN-E2E-NN]` field to TEST-NNNN. This means this repo *participates in* that scenario — it does not mean this repo's own `execute-tests` run decides the scenario's result (see [`templates/reference/xdep-schema.md`](../reference/xdep-schema.md)); the AC itself still needs its own repo-local test case here as normal. Do not treat the cross-repo scenario as a substitute for repo-local AC coverage.

### Step 3a — Write invariant/boundary test cases from ADRs

For each ADR linked to DESIGN-NNNN, write at least one test case that would fail if the ADR's decision were violated — not a test that the happy-path behavior works, a test that specifically exercises the boundary the ADR establishes (e.g. for an ADR stating "the model must never author the final score directly," a test that feeds a crafted model response attempting to influence the score outside the sanctioned input path, and asserts the score is unaffected or the response is rejected). Use the `TC-ADR-[NNNN]-[seq]` ID format. Assign P1 unless the ADR concerns a security-sensitive or data-integrity boundary, in which case P0.

If an ADR's boundary is not independently testable (e.g. a purely organizational/process decision), state that explicitly with a one-line reason rather than omitting it.

### Step 4 — Align test cases with the happy path trace

Read the **Happy Path Trace** section from PLAN-NNNN. For each User Story trace, walk each flow step and identify which test case from Step 3 exercises it. Produce a **Happy Path Coverage** table in TEST-NNNN.

Also read any **Alternate/Failure Path Trace** section(s) in PLAN-NNNN, if present, and produce a corresponding **Alternate Path Coverage** table using the same format and the same verbatim-copy rule below.

Copy each flow step description **verbatim** from PLAN-NNNN's Happy Path Trace — do not paraphrase, shorten, or reword. PLAN is the single source for flow step text. Any deviation makes it impossible to verify that TEST coverage aligns with what was planned.

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

TEST-NNNN must contain: Test Strategy; AC Coverage Map organized by milestone (milestone → AC ID → test cases, each with ID, type, execution mode, priority — AC descriptions are not re-stated here, IDs only); ADR/Invariant Test Cases (one per ADR linked to DESIGN-NNNN, or an explicit not-independently-testable note); Happy Path Coverage (one table per primary User Story); Alternate Path Coverage (one table per Alternate/Failure Path Trace in PLAN-NNNN, if any exist); Observability Verification; Security Test Cases; Test Data Strategy; Execution Results Summary (placeholder). All `deferred-to-release` test cases must have a `Deferral Reason`. Frontmatter: `version`; `target_plan_version`; `execution_date: null`; `execution_summary: null`. For a thread participant with an applicable cross-repo scenario, frontmatter also includes `cross_repo_scenarios: [THREAD-NNNN-E2E-NN, ...]`.

For a `repo_role: coordination` thread with at least one live-verification dependency, `SYSTEM-TEST-PLAN.md` must contain: strategy/scope/rationale prose, and a list of scenario IDs with which participants and XDEPs each covers (field-level detail stays in the scenario YAML, never duplicated here).

## Quality Gate

Before requesting review:
- [ ] Frontmatter complete: `status: draft`, `schema_version: 1`, `version: 1`, `updated`, `target_plan_version`, `execution_date: null`, `generation` block (with `command_version` and `session_id: null`)
- [ ] Every test case has an ID, type, execution mode, and priority
- [ ] Every AC has at least one P0 automated test covering the happy path
- [ ] No P0 test case is `manual` without a `Procedure` block and `Reason` line
- [ ] Integration tests use a real database (no DB mocks, per ARCH.md rules)
- [ ] Happy Path Coverage section present; every flow step mapped or documented
- [ ] Alternate Path Coverage section present for any PLAN-NNNN Alternate/Failure Path Trace
- [ ] Every ADR linked to DESIGN-NNNN has at least one invariant/boundary test case, or an explicit not-independently-testable note
- [ ] Observability verification covers each signal in OBS-NNNN
- [ ] Security test cases cover each finding in SEC-NNNN; Critical/High findings are P0
- [ ] All security test cases test the failing path
- [ ] Test data strategy ensures no shared state between test cases
- [ ] AC Coverage Map organized by milestone matching PLAN structure
- [ ] All `deferred-to-release` test cases have a valid `Deferral Reason`; no P0 deferred
- [ ] Execution Results Summary placeholder present
- [ ] *(thread participants only)* `cross_repo_scenarios` set for any applicable scenario, and the AC it also covers still has its own repo-local test case
- [ ] *(repo_role: coordination only)* Every XDEP needing live verification has a scenario; every scenario's `implementation` points to an executor-owned repo, never this one; scenarios only authored when contract-level verification is insufficient

**Self-check before proceeding:** Re-read each item above. Confirm each is complete, or state what was skipped and why before continuing to Handoff.

## Cross-Repo Scenario Approval Gate (coordination-repo threads only)

`SYSTEM-TEST-PLAN.md` and its scenarios don't fit `/review-test-plan`'s per-repo review shape (no AC Coverage Map, no milestones, no single test-execution owner) — the same reason OBS-NNNN uses an inline Lead Developer Approval Gate rather than a separate review command. Before committing, present the draft to the Architect for explicit approval:

**Blocking (must pass before commit):**
- [ ] Every scenario's `implementation` points to an executor-owned repo, never this coordination repo
- [ ] Every scenario names `owners` and a `triage_owner`
- [ ] Every XDEP genuinely needing live, multi-service verification has a covering scenario — no gap silently left to contract verification that can't actually catch it
- [ ] `SYSTEM-TEST-PLAN.md` references scenarios by ID only — no field-level detail duplicated from the scenario YAML

Ask the Architect: "Do all scenarios meet the specificity and ownership bar? Confirm to commit SYSTEM-TEST-PLAN.md and its scenarios."

Do not commit until the Architect explicitly confirms.

## Handoff

**`repo_role: code` (including thread participants):** Show the user the TEST-PLAN file. Do not commit.

> "Here is the draft TEST-PLAN. Please review — once you confirm, run `/review-test-plan TEST-NNNN SPEC-NNNN OBS-NNNN SEC-NNNN` to run all perspectives in one pass."

The single commit for the test planning phase is owned by `/approve-test-plan`.

**`repo_role: coordination`:** After the Architect confirms the Cross-Repo Scenario Approval Gate above, ask: "Solo project (commit only) or team project (push too)?"
```bash
git add sdlc/THREAD-NNNN/SYSTEM-TEST-PLAN.md sdlc/scenarios/THREAD-NNNN-E2E-*.yml sdlc/registry/index.md
git commit -m "THREAD-NNNN: system test plan — [N] scenarios, Architect approved"
# team only:
git push origin main
```
There is no separate `/review-test-plan`/`/approve-test-plan` cycle for this path — the Architect gate above is the review, and this commit is the approval. If no scenarios were needed (Pre-flight Step 3), skip this Handoff entirely; there is nothing to commit.
