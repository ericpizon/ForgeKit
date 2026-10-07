---
id: review-implementation
type: command
name: /review-implementation
description: Multi-role review of a milestone's implementation — Lead Developer, Security, SRE, QE perspectives in one pass; produces IMPL-REVIEW artifact
status: active
version: 4
primary_role: lead-developer
reviewing_roles:
  - security
  - sre
  - quality-engineer
output_artifact_type: impl-review
changelog:
  - v4 — added a staleness check for PLANs with an inputs.cross_repo_contract block, the last of the five gate points from templates/reference/cross-repo-resolution.md's staleness rule.
---

# /review-implementation

## Purpose

Review the implementation of one milestone from four perspectives: Lead Developer, Security, SRE, and QE. Each perspective adopts the corresponding role's mental model and quality bar. Produces or appends to `sdlc/reviews/IMPL-REVIEW-NNNN-MN.md`. Blocking findings must be addressed before `/approve-implementation` can proceed.

## Usage

```
/review-implementation PLAN-NNNN M1
```

## Required Inputs

- **PLAN-NNNN** — read milestone MN's task list and AC coverage
- **TEST-NNNN** — read the AC coverage map for MN's ACs; QE review checks test implementation against this contract
- **SEC-NNNN** — filter findings applicable to MN's tasks *(skip if PLAN frontmatter shows `includes_security_tasks: false`)*
- **OBS-NNNN** — filter observability tasks for MN *(skip if PLAN frontmatter shows `includes_observability_tasks: false` and MN has no OBS tasks listed)*
- **DESIGN-NNNN** — read API shapes, data model, module layout, and Key Decisions for MN
- **ARCH.md** — read for coding conventions, module layout patterns, auth model, and testing rules
- **Source code** — read all files created or modified for MN. Derive the file list from MN's task descriptions in PLAN-NNNN.
- **`sdlc/reviews/IMPL-REVIEW-NNNN-MN.md`** — read if it exists (re-review context)

## Staleness check

*(Only if PLAN-NNNN has an `inputs.cross_repo_contract` block.)* Before reviewing, re-resolve the coordination repo's current `approved` Cross-Repo Contract revision (per [`templates/reference/cross-repo-resolution.md`](../reference/cross-repo-resolution.md)) and compare to `inputs.cross_repo_contract.revision`. If they differ, the code under review may have been generated against a now-superseded dependency: flag this at the top of the review round as a blocking finding under Lead Developer Findings rather than silently reviewing as normal — the fix may be a code change, or may just be confirming MN's specific tasks are unaffected, but that determination has to happen before approval, not skipped.

## On Re-Review

If `sdlc/reviews/IMPL-REVIEW-NNNN-MN.md` already exists:

1. Read `current_review_round` from IMPL-REVIEW-NNNN-MN.md frontmatter. The new round number is `current_review_round + 1`. After appending the new round section, update `current_review_round` in frontmatter to the new round number.
2. Read the latest `### Address Summary — Round N` to understand which findings were addressed and which were deferred with documented reasons.
3. Load artifacts lazily — read the address summary first, then load only the source files, PLAN tasks, DESIGN sections, and TEST cases referenced by addressed findings. Do not reload ARCH, DESIGN, or TEST in full if their domain was unaffected by the address pass.
4. Scope this round's review to changed files only. Examine unchanged files only if a change in one file has plausible cascade impact on another. Do not re-review the entire milestone.
5. Append a new `## Round N` section — do not modify prior round content.

## Review Output File

**File:** `sdlc/reviews/IMPL-REVIEW-NNNN-MN.md`

**If the file does not exist:** create it (frontmatter + Task Reference table + Round 1 section).
**If the file already exists:** follow On Re-Review steps above, then append the new round section. Do not regenerate the Task Reference table.

**Null suppression:** Omit any `### [Perspective] Findings` section that has no findings — do not emit the section header or "None." List clean perspectives in a single `**Clean perspectives:** [list]` line before `### Summary`.

```markdown
---
id: IMPL-REVIEW-NNNN-MN
type: impl-review
schema_version: 1
plan: PLAN-NNNN
milestone: MN
current_review_round: 1
generation:
  command: /review-implementation
  command_version: "[use version from this command's frontmatter]"
  model: "[self-identify from system context]"
  generated_at: "[today's date]"
  session_id: null
---

# Implementation Review — PLAN-NNNN MN

## Task Reference — MN

*Derived from PLAN-NNNN at review creation time. Reviewers use this to verify coverage without loading the full PLAN.*

| Task | File(s) | Description |
|---|---|---|
| T1 | src/path/to/file.ts | [task description from PLAN-NNNN] |
| T2 | src/path/to/other.ts | [task description from PLAN-NNNN] |

## Round 1 — [date]

### Lead Developer Findings  *(omit section if none)*
[findings]

### Security Findings  *(omit section if none)*
[findings]

### SRE Findings  *(omit section if none)*
[findings]

### QE Findings  *(omit section if none)*
[findings]

**Clean perspectives:** [list any perspectives with no findings]

### Summary
N blocking, M non-blocking findings.

### Next Step
[/address-implementation-review PLAN-NNNN MN | /approve-implementation PLAN-NNNN MN]
```

Each finding:
```
**[ROLE-N] blocking | non-blocking — [title]**
[description of the issue]
[resolution recommendation]
```

---

## Lead Developer Perspective

Adopt the Lead Developer role (`.claude/roles/lead-developer.md`). Read ARCH.md before reviewing code — all code quality judgments must be grounded in the project's established conventions, not generic best practices.

**If creating IMPL-REVIEW-NNNN-MN.md for the first time:** populate the `## Task Reference — MN` table from MN's task list in PLAN-NNNN before writing any findings. Include every task in MN: task ID, the file(s) it creates or modifies, and its description. OBS and SEC tasks: use the source ID (e.g., `OBS-NNNN IT-N`) as the description. This table is written once and never modified.

**Before reading individual files**, run `git status` to list every new and modified file in the working tree. Cross-reference this list against MN's task descriptions in PLAN-NNNN — each task names the file(s) it creates or modifies. Any file that appears in `git status` but is not attributable to a task in MN is an out-of-scope change and must be flagged as blocking.

**If PLAN-NNNN has `linked_bug` set:** read `sdlc/bugs/BUG-NNNN.md` `## Triage` section before reading any source files. Note the confirmed root cause and proposed fix. After reviewing the diff, confirm:
- The diff targets the documented root cause — not a workaround that masks the symptom
- If the fix diverged from what was described in triage, the reason must be documented in PLAN-NNNN or BUG-NNNN

If the diff addresses a different issue than what was triaged, flag as **blocking**.

Then read every source file in MN's task list. Check:

- All MN tasks are implemented and match DESIGN-NNNN specs (API shapes, function signatures, data model, module layout)
- Type safety: no unsafe type casts or `any`-equivalent escapes in typed languages
- Input validation: all external input validated at the boundary before being passed to internal functions (read ARCH.md for the project's validation approach)
- Module layout matches DESIGN-NNNN and ARCH.md conventions (files in correct paths; no logic that belongs in shared modules leaked into handler/controller files)
- Auth guard pattern is correct per ARCH.md: auth is checked before any database access in external-facing handlers
- No implementation decisions were made silently — any task that required a judgment call should have surfaced as a blocker during `/implement-plan`; surface anything that slipped through
- **Invariant/boundary enforcement.** For each ADR establishing a deterministic/model-authorship boundary (or any other explicit invariant) relevant to MN's tasks, confirm the actual code enforces it — e.g. if a model response type is destructured or mapped into a downstream computation, confirm no forbidden field crosses that boundary. This is distinct from "matches DESIGN-NNNN specs": a developer can match DESIGN's stated shape while still leaving the invariant unenforced if DESIGN didn't fully anticipate every code path — check against the ADR's stated intent directly, not just the DESIGN section that cites it.

---

## Security Perspective

Adopt the Security role (`.claude/roles/security.md`). Read SEC-NNNN's findings applicable to MN before reviewing code. Apply only checks relevant to MN's tasks.

For each SEC finding scoped to MN:
- Verify the specific mitigation is implemented — not a general similar pattern, the exact mechanism specified
- Confirm the mitigation is in the right location (the file and function named in the finding)

Project-agnostic security checks applicable to every milestone:
- Auth is checked **before** any data access — no data lookup precedes auth verification in any external-facing handler
- Sensitive values (credentials, tokens, API keys) are never logged — only non-sensitive identifiers (prefixes, hashed values, user IDs) may appear in log statements
- Error responses for security-sensitive paths do not leak internal state, stack traces, or distinction between "not found" and "unauthorized" when enumeration prevention is required
- External input is validated before use
- **Prompt construction, if `project.yml`'s `stack.llm_agent_untrusted_input` is `yes`.** For any code in MN that builds a prompt from external or user-controlled text — whether or not that code was the subject of a specific SEC finding — confirm the prompt explicitly frames that content as data to analyze, not instructions to follow, per ARCH.md's standing security convention. This is a standing check, not one gated on a SEC finding existing: a new prompt-building path added in MN that was never covered by SEC-NNNN's scope still needs it.

Read ARCH.md for the project's auth model and credential handling conventions. Findings must cite specific files and lines, not generic concerns.

---

## SRE Perspective

Adopt the SRE role (`.claude/roles/sre.md`). Check:

- **Error handling:** Critical paths (database calls, external service calls) have appropriate error handling. Failure modes match what was specified in DESIGN-NNNN — fail-closed vs fail-open decisions from ADRs are implemented as documented.
- **OBS tasks implemented:** for MN's observability tasks from OBS-NNNN, verify metric names, label dimensions, and instrumentation points match OBS-NNNN exactly. A misnamed metric or missing label will silently drop data.
- **Log format:** structured logging per ARCH.md conventions; no string concatenation; no sensitive values in any field.
- **Error response format:** error responses consistently return the expected content type; no framework default error pages leak into API responses.

---

## QE Perspective

Adopt the Quality Engineer role (`.claude/roles/quality-engineer.md`). Read TEST-NNNN's AC coverage map for MN's ACs. For each test case listed, locate the corresponding test file and verify:

- **AC coverage:** every AC in MN's `AC Coverage` field has at least one test case in the code — cross-reference TEST-NNNN's coverage map against the actual test files.
- **Unit vs. integration boundary per ARCH.md:** unit tests have no database access, no network calls, no framework dependencies. Integration tests use a real database — no mocks per ARCH.md testing rules.
- **Data isolation:** tests set up and tear down their own data. No shared fixtures that persist across tests. Use unique identifiers to prevent collision between concurrent or sequential test runs.
- **P0 test cases present and automated:** none of the P0 test cases from TEST-NNNN were downgraded to manual without a documented reason. Any such demotion is a blocking finding.
- **Test implementations match TEST-NNNN specifications:** correct assertion values, correct expected status codes, correct response body shapes. A test that passes while asserting the wrong thing is worse than no test.
- **Security test cases use separate scenarios for distinct failure paths** — multiple failure conditions must not be merged into a single test with OR logic.

---

## Cross-Perspective Check

After completing all perspectives:

- **Contradictions:** Does a finding from one perspective conflict with another? Flag contradictions — the developer must arbitrate before addressing.
- **Cascade:** Did one perspective's finding change scope in a way another perspective should know about? Add a note under the affected perspective's findings.

## Summary Line

```
**Summary:** N blocking, M non-blocking findings.
```

## Next Step

- If blocking > 0: `/address-implementation-review PLAN-NNNN MN`
- If blocking == 0: `/approve-implementation PLAN-NNNN MN`

State the next step explicitly at the end of the round section.
