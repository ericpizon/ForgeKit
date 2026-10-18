---
id: execute-tests
type: command
name: /execute-tests
description: QE command to run automated tests for a milestone's test cases — CI-first with local fallback; records results in PLAN-NNNN; QE classifies failures and routes to QE fix path (test code) or developer path (production code)
status: active
version: 6
primary_role: quality-engineer
changelog:
  - v6 — added the Post-Merge Validation Path (`pipeline.test_gate: post_merge`): a milestone reached via `/approve-implementation`'s own merge step (already `pr_merged`, `post_merge_test: pending`) validates the merged/deployed state instead of a feature branch — `stack.qa_trigger_command` if configured, otherwise a path-aware local sync (`git pull` only on the team path, never solo) and local test run. Writes `post_merge_test.status` (P0-only trigger for `failed`, never overwritten once set) and routes a P0 failure to `/create-bug`/`/escalate-to-incident` instead of `test_failed`, since there is no feature branch left to fix and re-test. Built for the Configurable Post-Merge Test Gate feature (PLAN-0002 T5, OBS-0002).
  - v5 — the test-execution table gained a standing `Classification` column (always present, not an ad-hoc Notes addition on failure) using the exact literal values QE's own classification prompt produces (`production_bug`/`test_bug`) — resolves a gap where this decision was readable only as free-text prose. Built for the ForgeKit Operator feature (PLAN-0001 T17).
  - v4 — Test Case Identification now also pulls in TC-OBS-*/TC-SEC-*/TC-ADR-* cases by cross-referencing MN's observability/security tasks against the signal/finding ID each test case names, since these carry no AC at all and filtering only by the AC Coverage Map silently never executed them at any milestone; and now also pulls in Edge/Alternate-path cases that cite an in-scope AC only in prose, with no row in the formal table. Caught running `/execute-tests PLAN-0001 M2 TEST-0001` — M2 has OBS/SEC tasks (T13-T15, T17-T18a) and TEST-0001 has a `TC-EDGE-1` citing AC2 only in prose, both of which the old table-only filter would have skipped entirely at every milestone.
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

- `approved` → proceed via the ordinary (`pre_merge`) path below
- `test_failed` → also proceed via the ordinary path (this is a re-run after fixes)
- `pr_merged` **with a `post_merge_test` entry present on MN** → this is `pipeline.test_gate: post_merge`'s own route into this command, reached directly from `/approve-implementation`'s merge step, never from an `approved` status. Proceed via the **Post-Merge Validation Path** section below instead of the ordinary path.
  - If `post_merge_test.status` is already `passed` or `failed` (not `pending`): stop — "MN's post-merge test outcome is already recorded (`[status]`). A post-merge regression is addressed as a new milestone or hotfix, never by re-running this command against the same milestone."
- Anything else → stop: "MN must be `approved` before test execution. Run `/approve-implementation PLAN-NNNN MN` first."

## Test Case Identification

Read TEST-NNNN. Two independent sources feed MN's test scope — a TEST-NNNN with only an AC Coverage Map (no OBS/SEC/ADR sections) only ever has the first:

1. **AC-mapped cases:** test cases whose ACs appear in MN's `**AC Coverage:**` field (per TEST-NNNN's AC Coverage Map).
2. **Task-sourced cases** (`TC-OBS-*`, `TC-SEC-*`, `TC-ADR-*`): these are not tied to an AC at all — they're tied to an OBS-NNNN signal, a SEC-NNNN finding, or an ADR, named directly in their own test case title or description (e.g. `TC-OBS-source_work_intake_result-01` ↔ OBS-0001's `source_work.intake_result` signal; `TC-SEC-SEC1-01` ↔ SEC-0001's SEC-1). Cross-reference against MN's own task list in PLAN-NNNN: an `observability`/`security` task whose description cites a signal or finding ID pulls in every test case naming that same ID. A test case naming a command MN's tasks don't touch (e.g. a `/promote-to-thread` signal when MN only modifies `/create-spec`) is out of scope for MN even if the signal/finding ID itself is in TEST-NNNN. **Filtering only by the AC Coverage Map misses this entire category — OBS/SEC/ADR test cases carry no AC at all, so a milestone with observability or security tasks and no corresponding AC-mapped case for them would otherwise never get these tests executed at any milestone.**
3. **Edge/Alternate-path cases that cite an in-scope AC only in prose**, not in the formal AC Coverage Map table (e.g. a `TC-EDGE-N` whose description says "confirm AC2's gate still fires correctly" for a variant input, with no corresponding row in the table). Scan the Edge Cases and Alternate Path sections for any case naming an AC that is in MN's `**AC Coverage:**` field — include it even though the table itself doesn't list it there.

Separate the combined set into two buckets:

**Automated** (`execution: automated`) — run these.

**Manual** (`execution: manual`) — do not run; list as `deferred-to-release` in the results table with the reason from their `Reason:` block.

## Post-Merge Validation Path (`pipeline.test_gate: post_merge`)

*(Applies only when the Pre-flight Check above routed here — a milestone already `pr_merged` with `post_merge_test: pending`. Skip this entire section for the ordinary path; continue at CI Detection below instead.)*

There is no feature branch left to test against — MN already merged to `main` via `/approve-implementation`'s own merge step. Everything here validates the already-merged/deployed state instead.

### Validation Target

Read `project.yml`'s `stack.qa_trigger_command`.

**Configured (not `none`):** run it directly — this is the project's own QA-environment health/deployment check. It is never conflated with `stack.ci_trigger_command`, which stays branch-scoped and is only ever used by the ordinary `pre_merge` path below. Parse its output for pass/fail per test case the same way the CI Path parses CI output.

**Not configured (`none`):** fall back to local validation against `main` itself. First sync the local checkout — this differs by which merge path `/approve-implementation` actually took for MN:
- **Team path** (MN merged via a real PR, `gh pr merge`): `git checkout main && git pull` — the merge happened on GitHub; the local checkout is stale and genuinely needs fetching.
- **Solo path** (MN merged via a direct local fast-forward): `git checkout main` only, no pull — local `main` is already current from that fast-forward moments earlier, and a bare `git pull` would error on a repo with no remote configured.

**If either sync command fails** (`git checkout` blocked by uncommitted local changes, `git pull` failing on a network error or a genuine conflict) — stop and surface it to the user directly as an infrastructure/environment problem. Do **not** proceed to run tests anyway: per `DESIGN-0002` §3.4, `post_merge_test.status` is never overwritten once written, so a result computed against an unexpectedly half-synced checkout would become a permanent, uncorrectable record for this milestone. Resolve the sync problem first, then re-run this command — it does not require re-merging MN.

Then run:
```bash
{{stack.test.unit}} 2>&1 | tee .test-output-unit.txt
```
If `stack.test.e2e` is not `none`:
```bash
{{stack.test.e2e}} 2>&1 | tee .test-output-e2e.txt
```

**Infra-failure distinction:** if `stack.qa_trigger_command` itself fails to run at all (a network/auth error, not a test result), surface this directly to the user as an infrastructure problem and stop — never write anything to `post_merge_test.status`, which must mean "tests ran and found a real problem," not "the trigger mechanism was broken." Re-running this command retries the trigger; it does not require re-merging MN.

### Results Recording (post-merge)

Append the same `### Test Execution — Round N` table format as Results Recording below, with `Mode: post_merge-qa_trigger | post_merge-local`.

### Writing `post_merge_test`

Update MN's entry in PLAN-NNNN:
```yaml
post_merge_test:
  status: passed | failed
  updated_at: "[now]"
```
**Only a P0 failure sets `status: failed`** — exactly the same P0-only trigger `test_failed` already uses under `pre_merge`. If every P0 case passes, set `status: passed` — any P1+ failures are still recorded in the table but never flip this field away from `passed`. This write happens exactly once per milestone; the Pre-flight Check above already refuses to re-enter this command once `post_merge_test.status` is no longer `pending`.

### Routing a P0 Failure (post-merge)

Adopt the Quality Engineer role. Unlike the ordinary path, there is no feature branch left to fix and re-test, so `test_failed` and `/address-test-failures` never apply here:

- **Environment is not shared/blocking:** run `/create-bug`, naming MN and the failing test case(s).
- **Environment is shared/blocking** (other work depends on it being healthy): run `/escalate-to-incident` instead.

`**Implementation status:**` stays `pr_merged` throughout — it was already terminal before this command ran, and neither a passing nor a failing post-merge outcome changes it. Commit the results and the `post_merge_test` write together:
```bash
git add sdlc/plans/PLAN-NNNN-[slug].md
git commit -m "test(MN): post-merge validation — post_merge_test.status [passed|failed]"
```

---

## CI Detection

*(Ordinary `pre_merge` path, continued from the Pre-flight Check above.)*

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

| Test Case ID | Priority | Status | Classification | Notes |
|---|---|---|---|---|
| TC-AC1-1 | P0 | PASS | — | |
| TC-AC2-1 | P0 | PASS | — | |
| TC-AC11-2 | P0 | FAIL | production_bug | Expected 401, got 404 |
| TC-OBS-counter-1 | P1 | deferred-to-release | — | Requires production deployment + telemetry backend |

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

1. Record the classification in the test execution table's `Classification` column (now a standing column on every round, not an ad-hoc addition — resolves a gap where this decision previously lived only in free-text Notes, unreadable without parsing prose) using exactly the same literal values QE just chose from above:
   - `production_bug` — developer must fix
   - `test_bug` — QE will fix

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
