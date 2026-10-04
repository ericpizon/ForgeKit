---
id: execute-tests
type: command
name: /execute-tests
description: QE command to run automated tests for a milestone's test cases — CI-first with local fallback; records results in PLAN-NNNN; QE classifies failures and routes to QE fix path (test code) or developer path (production code)
status: active
version: 3
primary_role: quality-engineer
changelog:
  - v2 — failure routing redesigned: QE classifies each failure as production code bug (test_failed → developer) or test code bug (QE fixes and reruns, no review cycle); mixed failures handled as a batch
  - v3 — CI detection generalized from a hardcoded `.github/workflows/` check to `stack.ci_trigger_command`, a plain runnable command — the old check silently ran tests locally for any non-GitHub-Actions CI system instead of triggering it
links:
  - relation: follows
    target: approve-implementation
  - relation: precedes
    target: create-pr
  - relation: precedes
    target: address-test-failures
---

# /execute-tests

## Purpose

Run all automated test cases for a milestone's ACs and record the results. When P0 tests fail, QE classifies each failure and routes accordingly: test code bugs are fixed by QE and retested immediately; production code bugs set `test_failed` and are handed to the developer via `/address-test-failures`. Manual test cases are listed as `deferred-to-release`.

## Usage

```
/execute-tests PLAN-0001 M1 TEST-0001
```

## Pre-flight Check

Read PLAN-NNNN milestone MN's `**Implementation status:**`.

- `approved` → proceed
- `test_failed` → also proceed (this is a re-run after fixes)
- Anything else → stop: "MN must be `approved` before test execution. Run `/approve-implementation PLAN-NNNN MN` first."

## Test Case Identification

Read TEST-NNNN. Filter to test cases whose ACs map to MN's `**AC Coverage:**` field.

Separate into two buckets:

**Automated** (`execution: automated`) — run these.

**Manual** (`execution: manual`) — do not run; list as `deferred-to-release` in the results table with the reason from their `Reason:` block.

## CI Detection

Read `project.yml`'s `stack.ci_trigger_command`. This is a plain runnable command, not an assumption about which CI system produced it — do not detect CI by checking for `.github/workflows/` specifically, since that only ever matches GitHub Actions and silently falls through to the Local Path for any other CI system (Jenkins, CircleCI, GitLab CI, etc.), running tests locally instead of actually triggering that repo's real pipeline.

### CI Path (`stack.ci_trigger_command` is set and not `none`)

1. Run `stack.ci_trigger_command` for the current branch (e.g. `gh workflow run test.yml`, a Jenkins CLI invocation, or whatever the project's own CI needs to be triggered)
2. Stream CI output if the command supports it; otherwise poll for completion via the project's own CI tooling
3. Parse results: collect pass/fail per test case from the test runner output

### Local Path (`stack.ci_trigger_command` is `none` or unset)

Run the project's unit and e2e test commands (from `project.yml` stack configuration):

```bash
{{stack.test.unit}} 2>&1 | tee .test-output-unit.txt
```

If `stack.test.e2e` is not `none`:

```bash
{{stack.test.e2e}} 2>&1 | tee .test-output-e2e.txt
```

Parse each output file for test case PASS / FAIL results. Match test names to TEST-NNNN test case IDs where possible.

## Results Recording

Append to PLAN-NNNN milestone MN section, after any existing content:

```markdown
### Test Execution — Round N
Date: [today]
Mode: local | CI
Executed by: /execute-tests PLAN-NNNN MN TEST-NNNN

| Test Case ID | Priority | Status | Notes |
|---|---|---|---|
| TC-AC1-1 | P0 | PASS | |
| TC-AC2-1 | P0 | PASS | |
| TC-AC11-2 | P0 | FAIL | Expected 401, got 404 |
| TC-OBS-counter-1 | P1 | deferred-to-release | Requires production deployment + telemetry backend |

**P0 results:** N passed, M failed
**P1+ results:** X passed, Y failed, Z deferred-to-release
```

Record all results before routing. Do not route failures before the full table is written.

## If All P0 Tests Pass

1. Update PLAN-NNNN milestone MN: `**Implementation status:** tests_passed`
2. Commit:
   ```bash
   git add sdlc/plans/PLAN-NNNN-[slug].md
   git commit -m "test(MN): P0 tests pass — implementation_status tests_passed"
   ```
3. Notify:

   > **MN — all P0 tests passing.** The developer can now open the milestone PR: `/create-pr PLAN-NNNN MN`

   **If this is the last milestone** (all others are already `pr_open` or `pr_merged`): update TEST-NNNN before handing off so it goes to main with the final milestone's PR. Commit the TEST-NNNN status update to the current milestone branch:
   ```bash
   git add sdlc/test-plans/TEST-NNNN-[slug].md
   git commit -m "test: all P0 tests passed — TEST-NNNN status passed"
   ```
   Update TEST-NNNN frontmatter: `status: passed`, `execution_date: [today]`, `execution_summary: "All P0 automated tests passed across M1–MN. Manual tests deferred to release phase."`

## If Any P0 Tests Fail

Adopt the Quality Engineer role (`.claude/roles/quality-engineer.md`).

After recording all results, present each failing P0 test case to QE for classification. For each failure:

> **TC-[ID] failed** (P0)
> Failure: [failure message from test output]
> Test spec (TEST-NNNN): [the Given/When/Then or Scenario/Input/Expected block for this ID]
>
> Classify this failure:
> 1. **Production code bug** — the test assertion is correct; the implementation is wrong (wrong status code, wrong response body, wrong logic, wrong sequence)
> 2. **Test code bug** — the implementation is correct; the test assertion or setup is wrong (wrong expected value, missing Given condition, incorrect test data)

Wait for QE to classify **all** failing P0s before taking any action.

---

### Path A — All failures are test code bugs (QE path)

Production code is correct. QE owns the fix.

1. Milestone status stays `approved` — do not change it
2. For each test code bug, QE:
   - Reads the TEST-NNNN spec for the test case to confirm the correct assertion
   - Fixes the test file — corrects the assertion, Given conditions, or test data setup — so it matches the TEST-NNNN spec exactly
   - Does not modify production code
3. Confirm each fix passes by running only the affected test file. Invocation syntax for targeting a single file varies by runner — see ARCH.md for this project's pattern (e.g., Vitest: append the file path; pytest: `pytest path/to/test_file.py`).
4. Run the full suite to confirm no regressions:
   ```bash
   {{stack.test.unit}}
   ```
5. Commit all test file fixes:
   ```bash
   git add [affected test file(s)]
   git commit -m "fix(MN-tests): correct test assertion(s) — TC-[ID], TC-[ID]"
   ```
6. Re-run: `/execute-tests PLAN-NNNN MN TEST-NNNN`

The new round records the corrected results. No review cycle is required — QE self-verifies that fixed assertions match TEST-NNNN.

---

### Path B — Any failures are production code bugs (developer path)

1. Record the classification in the test execution table — add a `Classification` column or note in the Notes field for each failure:
   - `production code bug` — developer must fix
   - `test code bug` — QE will fix

2. **If the round includes test code bugs (mixed or test-only within Path B context):** QE fixes and commits those test files **now**, before setting `test_failed`. This way the developer pulls a branch that already has the test code corrections and only needs to address production code. For each fix, confirm it passes by targeting the specific file (syntax varies by runner — see ARCH.md), then confirm no regressions:
   ```bash
   {{stack.test.unit}}
   git add [affected test file(s)]
   git commit -m "fix(MN-tests): correct test assertion(s) — TC-[ID]"
   ```

3. Update PLAN-NNNN milestone MN:
   ```
   **Implementation status:** test_failed
   ```

4. Commit the results and status update:
   ```bash
   git add sdlc/plans/PLAN-NNNN-[slug].md
   git commit -m "test(MN): P0 failure(s) — test_failed [TC-ID, TC-ID]"
   ```

5. Hand off to developer:

   > **P0 production code failure(s) recorded for MN.**
   >
   > Production code bugs (developer):
   > - TC-[ID]: [failure description]
   >
   > QE has already committed test code fixes for any test code bugs in this round.
   > **Team:** pull the branch before starting `/address-test-failures` to get QE's fixes.
   > **Solo:** test code fixes are already committed — only production code remains.
   >
   > Run `/address-test-failures PLAN-NNNN MN` to address the production code failures.

---

## P1+ Failures

P1 and lower failures do not block `tests_passed`. Record them in the results table and surface them clearly:

> **P1 failures (non-blocking):**
> - TC-[ID]: [failure message]
>
> These do not block the next milestone. Fix before the release phase or document as known gaps in PLAN-NNNN.
