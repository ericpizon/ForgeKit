---
id: implement-plan
type: command
name: /implement-plan
description: Developer command to implement all tasks in one PLAN milestone — generating code, tests, and migrations, then prompting for local verification before review
status: active
version: 1
primary_role: developer
output_artifact_type: source-code
---

# /implement-plan

## Purpose

Generate all production code, test code, and migration files for one milestone in PLAN-NNNN. After generation, prompt the user to verify the implementation locally before triggering the implementation review.

## Usage

```
/implement-plan PLAN-NNNN M1
```

Each milestone is implemented independently. Milestones with `depends_on_milestones` can only start once their dependencies are `pr_merged`.

## Pre-flight Checks

Before generating any code, verify all of the following. Surface a blocker and stop if any fail.

1. **TEST-NNNN is `approved`** — the test contract must exist before code is written
2. **PLAN-NNNN is `approved`**
3. **Milestone `implementation_status` is `pending`** — read PLAN-NNNN milestone MN's `**Implementation status:**` field. Stop with a specific message for each non-pending state:
   - `in_progress` → "MN is already in progress. Check the current implementation or run `/review-implementation PLAN-NNNN MN`."
   - `approved` → "MN implementation is approved. Run `/execute-tests PLAN-NNNN MN TEST-NNNN` to begin the testing phase."
   - `test_failed` → "MN has P0 test failures. Run `/address-test-failures PLAN-NNNN MN` to address them — do not re-implement."
   - `tests_passed` → "MN tests are passing but no PR has been created. Run `/create-pr PLAN-NNNN MN`."
   - `pr_open` → "MN's PR is open. Merge it on GitHub before starting the next milestone."
   - `pr_merged` → "MN is fully complete — PR has been merged into main."
   - If the user intends to restart from scratch, ask explicitly before overwriting any status.
4. **All `depends_on_milestones` are `pr_merged`** — for each listed dependency milestone, read its `**Implementation status:**`:
   - `pr_merged` → OK, proceed
   - `pr_open` → stop — "MX's PR is open but not yet closed. Merge the PR on GitHub, then run `/close-milestone PLAN-NNNN MX`."
   - `tests_passed` → stop — "MX tests are passing but no PR has been created. Run `/create-pr PLAN-NNNN MX` first."
   - Any other status → stop — "MX must reach `pr_merged` before MN can start. Current status: [status]."
5. **For M1 specifically — SETUP prerequisites:** Read ARCH.md for required environment variables and local dev setup expectations. Confirm dev server is running, database is connected, required env vars are set. Surface any missing prerequisites as blockers before proceeding.

## Create Milestone Branch

After all pre-flight checks pass, create the milestone's dedicated branch before touching any code.

**M1** — branches from the thread branch (which carries all SDLC artifacts committed during planning):

```bash
git checkout thread/NNNN-slug
git pull origin thread/NNNN-slug
git checkout -b feature/PLAN-NNNN-M1
```

**M2+** — branches from `main` (which has SDLC artifacts + all prior milestones' code after their PRs merged):

```bash
git checkout main
git pull origin main
git checkout -b feature/PLAN-NNNN-MN
```

Branch naming is always `feature/PLAN-NNNN-MN` (example: `feature/PLAN-0001-M2`). One branch per milestone; each PR targets `main` via `/create-pr`.

**Solo project:** the branch creation still applies — it keeps milestone history readable even without a remote.

## Required Inputs

- **PLAN-NNNN** — approved; provides task list, AC coverage, and implementation notes for MN
- **DESIGN-NNNN** — read Key Decisions, data model, API shapes, and module layout for exact implementation specs
- **TEST-NNNN** — approved; filter test cases for MN's ACs — these define the test files to write alongside production code
- **SEC-NNNN** — filter findings applicable to MN's tasks; implement mitigations in the tasks that own them *(skip if PLAN frontmatter shows `includes_security_tasks: false`)*
- **OBS-NNNN** — filter observability tasks for MN; these must be built in, not added after *(skip if PLAN frontmatter shows `includes_observability_tasks: false` and MN has no OBS tasks listed)*
- **ARCH.md** — read for framework conventions, tech stack specifics, and patterns to follow

## Update Milestone Status

Before generating any code, update PLAN-NNNN milestone MN:

```
**Implementation status:** in_progress
```

**If PLAN-NNNN has `linked_bug` set** (this is a bug fix plan): also update `sdlc/bugs/BUG-NNNN.md` frontmatter:

```yaml
status: in_progress
```

**Important — code is not committed during the review cycle.** All generated source files live in the working tree, uncommitted, through every review round and address cycle. Only PLAN-NNNN's status field (and BUG-NNNN if applicable) is committed now. `/approve-implementation` is the single commit that captures all milestone code, tests, migrations, and the review artifact together.

**Team project** — commit the status change immediately so the milestone is visibly claimed on the shared branch:

```bash
git add sdlc/plans/PLAN-NNNN-[slug].md
git add sdlc/bugs/BUG-NNNN.md   # only if linked_bug is set
git commit -m "chore(MN): mark implementation in_progress"
# for hotfix plans (type: hotfix): git commit -m "fix(BUG-NNNN): mark implementation in_progress"
```

**Solo project** — this commit is optional. The `in_progress` status is already written to disk and will be committed alongside the code in `/approve-implementation`. Skip if working alone.

## Code Generation

Generate all production code and test code for the milestone's tasks. Derive the file list from the task descriptions in PLAN-NNNN — each task names the file it creates or modifies.

Read ARCH.md for:
- Framework conventions and file structure
- Database and migration approach (schema file locations, migration tool)
- Auth layer patterns (how auth is enforced in route handlers)
- Observability patterns (instrumentation approach)
- Test framework and patterns (unit vs integration vs e2e split, no-mock rules)

Follow this phase order within the milestone (adapt to what tasks MN actually contains):

**Phase 1 — Schema (if MN has schema tasks)**
Write schema definitions per the project's data layer conventions. Generate and review migrations before applying — verify column types, constraints, and indexes are correct. Apply migration to local dev database.

**Phase 2 — Infrastructure (if MN has infra tasks)**
Database client singleton, auth configuration, instrumentation setup. Unit test any non-trivial configuration logic.

**Phase 3 — Data Access Layer — TDD for pure functions first**
For every pure function (validation, hashing, computation with no I/O):
1. Write the failing unit test first
2. Run it — confirm it fails for the right reason
3. Implement the function to pass
4. Refactor if needed; test must still pass

Key distinction: pure functions are unit-testable; database-coupled wrappers are integration-tested.

**Phase 4 — API layer — integration tests alongside**
For each endpoint or server action:
- Write the integration test skeleton before or as you write the handler
- Test cases come directly from TEST-NNNN's AC coverage map for that endpoint
- Use real database — no mocks unless ARCH.md explicitly specifies otherwise
- Every distinct response code has a corresponding test case

**Phase 5 — UI layer (if MN has UI tasks)**
Components and pages. Write E2E test stubs per TEST-NNNN as pages are built.

**Phase 6 — Observability (if MN has OBS tasks)**
Per OBS-NNNN: instrumentation with exact metric names and label schema.

**Phase 7 — Security mitigations (if MN has SEC tasks)**
Per SEC-NNNN findings applicable to MN. Security test cases in TEST-NNNN validate these.

For each task, state: which PLAN task and AC(s) it implements, which TEST-NNNN test cases validate it.

## Blocker Protocol

If a task is ambiguous — can be implemented two ways with observably different outcomes — stop, surface the blocker, reference the task ID, state both interpretations, propose a resolution. Do not implement a guess.

If a task requires touching something outside the milestone's stated scope, call it out and note it as a follow-up. Do not expand silently.

## Pre-verification Checks

After generating all code, run locally before showing the diff. Use the commands specified in project.yml:

```bash
{{stack.lint}}
{{stack.test.unit}}
```

If `stack.lint` is `none`, skip the lint step. All checks must pass. Fix any failures before proceeding.

## Local Verification Prompt

After checks pass, show the user the diff and all generated/modified files, then say:

> **MN — [milestone name] implementation ready for local verification.**
>
> Files created/modified:
> [list each file]
>
> Type check, lint, and unit/integration tests pass locally.
>
> **Please verify the implementation:**
>
> Read the milestone's tasks in PLAN-NNNN to understand what needs to be verified. For each major capability this milestone adds:
> 1. Confirm the feature works end-to-end per the user story
> 2. Confirm any error paths behave as specified in the ACs
> 3. Confirm any security boundaries hold (e.g., unauthorized access returns the correct error)
>
> When satisfied, run: `/review-implementation PLAN-NNNN MN`

## Handoff

After the user confirms local verification, they run:
→ `/review-implementation PLAN-NNNN MN`
