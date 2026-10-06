---
id: PLAN-0001
type: plan
schema_version: 1
status: approved
version: 2
updated: 2026-10-05
target_design_version: 3
includes_observability_tasks: true
includes_security_tasks: true
milestone_count: 6
current_review_round: 2
changelog:
  - v2 — approved after 2 review rounds
approval:
  mode: repo_only
  approved_by: "Eric Pizon"
  approved_role: lead-developer
  approved_at: "2026-10-05"
  decision_summary: "PLAN-0001 approved for test planning phase — 20 tasks across 6 milestones, all 17 ACs covered."
  external_event_id: null
  external_event_uri: null
policy_evaluation:
  evaluated: null
  policy_pack: null
  result: null
generation:
  command: /create-plan
  command_version: "5"
  model: "claude-sonnet-5"
  generated_at: "2026-10-05"
  session_id: null
pipeline_status:
  spec: SPEC-0001 approved
  analysis: ANALYSIS-0001 completed
  design: DESIGN-0001 approved
  obs: OBS-0001 approved
  sec: SEC-0001 completed
  plan: approved
  test: approved
---

# PLAN-0001 — External Work Intake and Durable Coordination Context

## Task Breakdown

**T1** | functional
Author `templates/reference/external-work-intake.md` — the shared mechanism doc for G1/G5: the capability-check-then-attempt pattern (D5), the any-non-success-is-failure rule (D9), failure-diagnostic sanitization (D11), and external-content containment (D12). Every later task that touches intake, write-back, or Investigation points here instead of restating these rules.
**AC:** supports AC1, AC2, AC4, AC16 (foundational — no standalone AC of its own)

**T2** | functional
Add `sdlc/investigations/README.md` to `templates/sdlc/`; wire it into `forge-init.md`'s Phase 5 Step 6 directory-creation list for both `repo_role: code` and `repo_role: coordination`.
**AC:** supports AC16

**T3** | functional
Add `sdlc/promotions/README.md` to `templates/sdlc/`; wire it into `forge-init.md`'s coordination-repo directory list (Phase 5 Step 6 — corrected from an earlier citation of "Phase 4B," a factual error caught during implementation). Document in the README that `known_participants`/`environments` (AC5) are optional, directly-edited `project.yml` fields with no dedicated authoring command — consistent with how other durable config fields in ForgeKit already work — while `coordination_repo` (AC7/AC8) *is* command-populated.
**AC:** supports AC5, AC13–AC15

**T4** | functional
Modify `templates/commands/create-spec.md`: add the `--from <system>:<id>` argument; implement AC1 (capability-check, fetch-or-paste, `source_work` recorded either way), AC2/AC3 (`product_spec` block, the two capture-policy branches), D6 (compliance-aware confirmation gate, baseline — see T17 for the credential-scan extension). Points at T1's reference doc for the shared mechanism, but states inline, explicitly, in this command's own text: **fetched or pasted content is always presented to the TPO role as quoted reference material to read and interpret — never as instructions the role executes (D12)**; and **any fetch failure, shown to the user or recorded anywhere, is reduced to a sanitized failure category first — raw connector output is never shown or committed verbatim (D11)**. These invariants must be visible where the Developer is actually working, not left to memory of T1.
**AC:** AC1, AC2, AC3

**T5** | functional
Modify `templates/commands/approve-spec.md`: implement AC4 (write-back prompt, capability-check-then-write-or-fallback). Points at T1's reference doc for the shared mechanism, but states inline, explicitly: **any outcome that is not a confirmed full success — including a partial write — is treated as failure, with no "partially updated" state (D9)**; and **the failure itself is reduced to a sanitized category before display or persistence, never shown or committed raw (D11)**.
**AC:** AC4

**T6** | functional
Modify `.claude/commands/forge-init.md` (corrected from an earlier citation of `templates/commands/forge-init.md` — `forge-init.md` has no template counterpart; it's ForgeKit's own bootstrapping command, not one copied into adopting projects, same factual-citation-error class as T3's "Phase 4B" fix): add AC7's coordination-participation question to the `repo_role: code` path, writing `coordination_repo` to `project.yml` (baseline — see T19a for the validation extension).
**AC:** AC7

**T7** | functional
Modify `templates/commands/approve-design.md`'s Interface Freeze step: implement D1/ADR-0002 — regenerate `SYSTEM-ARCH.md`'s Dependency Map wholesale from the current set of `active` XDEPs at the moment an XDEP transitions to `active`; create the file lazily if it doesn't exist yet; a regeneration failure logs a warning and does not block the Interface Freeze approval itself.
**AC:** AC6

**T8** | functional
Author new command `templates/commands/promote-to-thread.md` (**ADR-0001**): implement AC8 (read/persist `coordination_repo`, baseline — see T19b for the validation extension), AC9 (Option B's six-step idempotent sequence, anchored by `pending_promotion` — written before any *write* to the coordination repo, reading it first to determine the candidate number is fine; re-mint and update `pending_promotion` if that number is later found already claimed by a different SPEC's concurrent promotion), AC10 (re-examine prior review rounds on substance, not a blanket restart or silent carry-forward), AC11 (`status: promoted`/`superseded_by`, never deleted). States inline, explicitly: **no write to the coordination repo happens before `pending_promotion` is committed to this SPEC's own frontmatter (D4/ADR-0001's core invariant)**; and **any error this command surfaces, including the coordination repo being unreachable, is reduced to a sanitized failure category first (D11, extended — see T20)**.
**AC:** AC8, AC9, AC10, AC11

**T9** | functional
Modify `templates/commands/status.md`: implement AC12 (detect `status: promoted`, redirect to `superseded_by`) and the Round-1 self-caught fix (detect `pending_promotion` set without `status: promoted`, report "promotion in progress — resume via `/promote-to-thread`").
**AC:** AC12

**T10** | functional
Author new command `templates/commands/promote-to-environment.md` (**ADR-0004**): implement AC13 (governance-value reasoning, never a hardcoded environment name), AC14 (`governance: ci` has no command path at all), AC15 (lightweight record — reuses `approve-release-plan.md`'s `candidate_ref` shape — appended to `sdlc/promotions/{environment}.md`, explicitly excluding Gating-criteria/Rollback/Validation-Plan sections).
**AC:** AC13, AC14, AC15

**T11** | functional
Author new command `templates/commands/create-investigation.md` (**ADR-0003**): implement AC16 (Question → Approach → Findings → Decision, four outcomes: no action, ADR hand-off, advisory-only recommend-implementation, investigate further); reuses T1's reference doc for its own `source_work`/capture mechanics when started from an external ticket (baseline — see T18b for the slug-sanitization extension). States inline, explicitly: **a `recommend_implementation` decision is written as plain advisory text only — this command never creates or writes to any external ticket (D12's containment principle applies symmetrically on the output side too)**.
**AC:** AC16

**T12** | functional
Author new command `templates/commands/create-adr.md` (**ADR-0003**): implement AC17 (standalone ADR authorship, same required sections as a DESIGN-nested ADR, no `target_design` pointer, no parent DESIGN or SPEC required) (baseline — see T18b for the slug-sanitization extension).
**AC:** AC17

**T13** | observability | `templates/commands/create-spec.md`
**Source:** OBS-0001 IT-1 — see OBS-0001 for full implementation spec.
**Milestone:** M2

**T14** | observability | `templates/commands/approve-spec.md`
**Source:** OBS-0001 IT-2 — see OBS-0001 for full implementation spec.
**Milestone:** M2

**T15** | observability | `templates/commands/create-spec.md`
**Source:** OBS-0001 IT-3 — see OBS-0001 for full implementation spec.
**Milestone:** M2

**T16** | observability | `templates/commands/promote-to-thread.md`
**Source:** OBS-0001 IT-4 — see OBS-0001 for full implementation spec.
**Milestone:** M4

**T17** | security | `templates/commands/create-spec.md`
**Source:** SEC-0001 SEC-1 — see SEC-0001 for full mitigation spec.
**Milestone:** M2

**T18a** | security | `templates/commands/create-spec.md`
**Source:** SEC-0001 SEC-2 — see SEC-0001 for full mitigation spec.
**Milestone:** M2

**T18b** | security | `templates/commands/create-investigation.md`, `templates/commands/create-adr.md`
**Source:** SEC-0001 SEC-2 — see SEC-0001 for full mitigation spec.
**Milestone:** M6

**T19a** | security | `.claude/commands/forge-init.md` (corrected from `templates/commands/forge-init.md` — same citation-error class as T6/T3)
**Source:** SEC-0001 SEC-3 — see SEC-0001 for full mitigation spec.
**Milestone:** M3

**T19b** | security | `templates/commands/promote-to-thread.md`
**Source:** SEC-0001 SEC-3 — see SEC-0001 for full mitigation spec.
**Milestone:** M4

**T20** | security | `templates/commands/promote-to-thread.md`
**Source:** DESIGN-0001 D11 (scope extended per its own 2026-10-05 amendment, resolving Round 2's non-blocking `[SEC-1]`, picked up as `/review-plan`'s own `[ARCH-1]` finding) — see DESIGN-0001 for full spec.
**Milestone:** M4

*(T18 and T19 are each split across two milestones because their target files span code delivered in different PRs — splitting keeps each milestone's PR self-contained rather than leaving a cross-milestone task dangling.)*

## Sequencing & Dependencies

No database, no infrastructure — "sequencing" here means: shared mechanisms before the commands that depend on them, and every command before the observability/security tasks that instrument or harden it.

1. **Foundation** (T1–T3) — the shared reference doc and directory scaffolding every later task points to.
2. **Commands that depend only on Foundation** (T4–T12) — each is independent of the others at the code level (different command files), but every one of T4, T5, T8, T11, T12 depends on T1 existing first; T7 depends on nothing but the already-existing `/approve-design`; T6/T10 depend on nothing but Foundation.
3. **Observability** (T13–T16) — each depends on its target command (T4/T5/T8) already existing.
4. **Security** (T17–T20) — each depends on its target command(s) already existing; T18a/T19a/T19b additionally depend on T1 (they extend rules T1 already documents). T20 depends on T8 (it extends T8's own error-handling) and on DESIGN-0001's own D11 amendment, not on T1 directly.

## Milestones

### M1 — Foundation: Shared Mechanics & Directory Scaffolding
**Tasks:** T1, T2, T3
**Implementation status:** pr_merged
**AC Coverage:** supports AC1, AC2, AC4, AC5, AC13–AC16 (no standalone AC — pure foundation)
**Depends on:** —
**Rollback:** Safe — no command behavior changes yet; this milestone only adds a new reference doc and two new directory READMEs.

### Test Execution — Round 1
Date: 2026-10-05
Mode: N/A — no test cases scoped to this milestone
Executed by: `/execute-tests PLAN-0001 M1 TEST-0001`

TEST-0001's own AC Coverage Map states explicitly: "M1 — Foundation: No ACs of its own — T1/T2/T3 are infrastructure." This is the authoritative scoping (not this milestone's own looser "supports AC5" line, which `[QE-1]` from implementation review had flagged as a possible scheduling conflict with TEST-0001 assigning AC5's tests to M3) — resolved in practice: zero test cases, automated or manual, are scoped to M1. Nothing to execute; no CI/local path distinction applies.

**P0 results:** 0 of 0 (vacuously passing — no P0 test case scoped to this milestone)
**P1+ results:** 0 of 0

### M2 — External Work Intake (G1)
**Tasks:** T4, T5, T13, T14, T15, T17, T18a
**Implementation status:** pr_merged
**AC Coverage:** AC1, AC2, AC3, AC4 (full)
**Depends on:** M1
**Rollback:** Safe — `--from` is a new optional argument; a SPEC created without it behaves exactly as today. No existing behavior changes.

### Test Execution — Round 1
Date: 2026-10-05
Mode: local (scenario-based — ForgeKit has no compiled code/CI; per TEST-0001's own Test Strategy, each case is a scripted walkthrough of `create-spec.md`/`approve-spec.md`'s actual text against a disposable fixture repo, with a mocked ADO/Confluence connector, checking resulting file/frontmatter state)
Executed by: /execute-tests PLAN-0001 M2 TEST-0001

**Pre-flight gap found and fixed:** `execute-tests.md`'s Test Case Identification step only ever filtered by TEST-NNNN's AC Coverage Map table, which would have silently skipped every `TC-OBS-*`/`TC-SEC-*`/`TC-ADR-*` case (none carry an AC) and `TC-EDGE-1` (cites AC2 only in prose, no table row) at every milestone, forever. Fixed in `execute-tests.md` v4 before running M2's own test cases — see that file's changelog.

| Test Case ID | Priority | Status | Notes |
|---|---|---|---|
| TC-FUNC-1 | P0 | PASS | Working connector simulated; fetched directly; `source_work` recorded. |
| TC-FUNC-2 | P1 | PASS | No connector; fell to paste; `source_work` identical shape to TC-FUNC-1. |
| TC-FUNC-3 | P1 | PASS | Connector attempted, errored (`not_found`); fell back to paste, not blocked. |
| TC-FUNC-4 | P0 | PASS | Fetched path: `product_spec` has `fetched_at`, no companion file exists. |
| TC-FUNC-5 | P0 | PASS | Paste + `compliance.frameworks:[none]`, no credential match: `.source.md` written automatically; `content_digest` verified as the file's real SHA-256. |
| TC-FUNC-6 | P1 | PASS | Paste + non-`[none]` compliance: confirmation required and given before write. |
| TC-FUNC-7 | P1 | PASS | Same as TC-FUNC-6, confirmation declined: `capture_status: declined`, no companion file, SPEC still created. |
| TC-FUNC-8 | P0 | PASS | `product_spec` core fields (`system`/`page_id`/`revision`) identically shaped across TC-FUNC-1 (fetch) and TC-FUNC-2 (paste). |
| TC-FUNC-9 | P0 | PASS | Working write access simulated; exact content shown; confirmed; `write_back.result: updated`. |
| TC-FUNC-10 | P1 | PASS | No write access; `write_back.result: copy_paste_fallback`, `attempted: false`. |
| TC-FUNC-11 | P1 | PASS | Write attempted, confirmed, then failed (`network_error`); sanitized category recorded, never raw output; fell back to copy-paste. |
| TC-FUNC-12 | P2 | PASS | No `source_work` block at all: write-back step skipped entirely, no `write_back` field added. |
| TC-FUNC-13 | P1 | PASS | Simulated `/reopen-spec` → `/revise-spec` → second `/approve-spec`: write-back fired a second time (v2 changelog records a second `updated` event). |
| TC-EDGE-1 | P2 | PASS | `compliance.frameworks: [soc2, gdpr]` (multi-value): gate still fires — triggered by "non-`[none]`", not a single-value match. |
| TC-OBS-source_work_intake_result-01 | P1 | PASS | All three values (`fetched`/`pasted`/`fetch_failed`) each fired only from their actual trigger — verified via grep against the simulated structured log. |
| TC-OBS-source_work_write_back_result-01 | P1 | PASS | All three values (`updated`/`copy_paste_fallback`/`write_failed`) each fired only from their actual trigger, same verification method. |
| TC-OBS-compliance_gate_outcome-01 | P1 | PASS | All three values (`captured`/`declined`/`not_applicable`) each fired only from their actual trigger. |
| TC-SEC-SEC1-01 | P1 | PASS | Credential-shaped string (`sk-[A-Za-z0-9]{20,}`) in pasted content, `compliance.frameworks:[none]`: regex match verified directly; confirmation gate fired; declined → `.source.md` confirmed absent from fixture directory listing. |
| TC-SEC-SEC2-01 | P1 | PASS *(partial — M2 scope)* | Path-traversal-crafted title (`../../etc/passwd...`): naive transform still carried `../`; sanitization rule (strip outside `[a-z0-9-]`) verified via shell transform to neutralize it completely. `/create-investigation`/`/create-adr` portions of this case re-run at M6 (those commands don't exist yet). |

**P0 results:** 5 passed, 0 failed (TC-FUNC-1, 4, 5, 8, 9)
**P1+ results:** 13 passed, 0 failed, 0 deferred-to-release

### M3 — Durable Coordination-Repo Context (G2)
**Tasks:** T6, T7, T19a
**Implementation status:** pr_merged
**AC Coverage:** AC5, AC6, AC7 (full)
**Depends on:** M1
**Rollback:** Safe — the new `forge-init` question defaults to "not yet" (unset); `SYSTEM-ARCH.md` regeneration only fires when an XDEP reaches `active`, and its own failure mode is already fail-open by design.

### Test Execution — Round 1
Date: 2026-10-06
Mode: local (scenario-based, per TEST-0001's own Test Strategy — scripted walkthroughs of `forge-init.md`/`approve-design.md`'s actual text against disposable fixture repos: one `repo_role: coordination` fixture for TC-FUNC-14/15/16 and TC-ADR-0002-01, one `repo_role: code` fixture for TC-FUNC-17/18 and TC-SEC-SEC3-01)
Executed by: /execute-tests PLAN-0001 M3 TEST-0001

| Test Case ID | Priority | Status | Notes |
|---|---|---|---|
| TC-FUNC-14 | P0 | PASS | `known_participants`/`environments` added to a coordination repo's `project.yml`; parsed cleanly via `npx js-yaml` — genuine schema-acceptance check, not just eyeballed. |
| TC-FUNC-14b | P2 | deferred-to-release | Manual — cross-suite confirmation that no other command's behavior silently changes because of these fields' presence; not a single automatable assertion. |
| TC-FUNC-15 | P0 | PASS | Interface Freeze with 2 new `active` XDEPs: `SYSTEM-ARCH.md`'s Dependency Map regenerated wholesale, `dependency_map_generated_at` advanced, prior hand-edited content replaced entirely. |
| TC-FUNC-16 | P1 | PASS | A third, deliberately malformed XDEP file (confirmed via a real YAML parse failure) triggered a logged warning; regeneration proceeded with the 2 valid XDEPs; Interface Freeze approval was not blocked. |
| TC-FUNC-17 | P0 | PASS | `repo_role: code` fixture, Phase 2b answered with `acme/platform-context`: `coordination_repo` recorded in `project.yml`. |
| TC-FUNC-18 | P2 | PASS | Phase 2b answered "not yet": `coordination_repo` key entirely absent; confirmed (via grep across `forge-init.md`) the question appears only in Phase 2b, Option C, and Full redo — never any other re-run path. |
| TC-ADR-0002-01 | P1 | PASS | The pre-existing hand-edited "HAND-EDITED STALE ENTRY" row is confirmed absent from the regenerated Dependency Map — proving the section is genuinely derived, not incrementally preserving manual edits. |
| TC-SEC-SEC3-01 | P2 | PASS *(partial — M3 scope)* | A pasted URL (`https://github.com/acme/platform-context`) verified to fail the `org/repo` pattern via direct regex test; rejected and re-prompted; corrected answer (`acme/platform-context`) verified to pass and was then written. `/promote-to-thread`'s reactive-equivalent portion re-runs at M4 (doesn't exist yet). |

**P0 results:** 3 passed, 0 failed (TC-FUNC-14, 15, 17)
**P1+ results:** 4 passed, 0 failed, 1 deferred-to-release (TC-FUNC-14b)

### M4 — Single-Repo → Cross-Repo Promotion (G3)
**Tasks:** T8, T9, T16, T19b, T20
**Implementation status:** pr_merged
**AC Coverage:** AC8, AC9, AC10, AC11, AC12 (full)
**Depends on:** M1 only. *(Corrected per Round 1's `[SRE-1]`: T8's own AC8 logic already has a reactive fallback for `coordination_repo` being unset — it asks and persists at that point rather than requiring M3 deployed first. M4 needs M3's logic to exist *eventually*, not *first*; this is not a hard deployment-ordering dependency.)*
**Rollback:** Safe — `/promote-to-thread` and `/status`'s new detection branch are both purely additive; a SPEC with no `pending_promotion`/`status: promoted` field behaves exactly as today.

### Test Execution — Round 1
Date: 2026-10-06
Mode: local (scenario-based, per TEST-0001's own Test Strategy — scripted walkthroughs of `promote-to-thread.md`/`status.md`'s actual text against two disposable fixture repos: a promoting `repo_role: code` repo and a `repo_role: coordination` repo, exercised as a real two-repo interaction, not a single-repo simulation)
Executed by: /execute-tests PLAN-0001 M4 TEST-0001

| Test Case ID | Priority | Status | Notes |
|---|---|---|---|
| TC-FUNC-19 | P0 | PASS | `coordination_repo` already set in `project.yml`: reused without asking. |
| TC-FUNC-20 | P1 | PASS | `coordination_repo` unset: asked reactively, persisted — confirmed in the **same commit** as `pending_promotion` (`git show --stat` on the actual fixture commit). |
| TC-FUNC-21 | P0 | PASS | SPEC in `status: draft`: `THREAD-0100` minted, `pending_promotion` committed in the promoting repo (real git SHA `8e1865e`) while the coordination repo's `sdlc/` was still confirmed empty via `ls` — the D4 ordering invariant checked concretely, not just asserted. `thread.yml` written listing the participant. |
| TC-FUNC-22 | P0 | PASS | SPEC in `status: approved` (`SPEC-TC22`): same mechanism succeeded — `THREAD-0101` minted, `thread.yml` written, finalized to `promoted`. |
| TC-FUNC-23 | P1 | PASS | Re-invoking on an already-`promoted` SPEC (`SPEC-TC22`): Step 0 no-op, reported existing `THREAD-0101`; confirmed via directory listing that no `THREAD-0102` was minted. |
| TC-FUNC-24 | P1 | PASS | Re-invoking after `pending_promotion` was committed but before any coordination-repo write: resumed from Step 2 without minting a second `THREAD-0100`. |
| TC-FUNC-25 | P0 | PASS | Prior ANALYSIS Round 1 finding `[LD-1]` re-examined and recorded as **invalidated by the newly-discovered scope**, with reasoning — not a blanket restart or silent carry-forward — in the copied `THREAD-0100/ANALYSIS.md`'s new `## Promotion Disposition` section. |
| TC-FUNC-26 | P0 | PASS | Original `SPEC-TC21` file confirmed still on disk (`ls`), `status: promoted`, `superseded_by: THREAD-0100`. |
| TC-FUNC-27 | P0 | PASS | `/status SPEC-TC22` (promoted): reports the promotion and names `THREAD-0101`. |
| TC-FUNC-28 | P1 | PASS | `/status SPEC-TC28` (`pending_promotion` set, `status: draft`): reports "promotion in progress — resume via `/promote-to-thread`," distinct from both the plain-draft and promoted-redirect reports. |
| TC-ADR-0001-01 | P0 | PASS | The specific interruption point ADR-0001 exists to make safe — `pending_promotion` committed, zero coordination-repo writes yet — was reproduced concretely (verified via `ls` on the coordination fixture immediately after the promoting-repo commit) and resumed correctly without a duplicate mint. |
| TC-OBS-promote_to_thread_outcome-01 | P1 | PASS | `interrupted` logged by the resuming invocation upon discovering `pending_promotion`; `resumed_and_completed` (not plain `completed`) logged separately at that same run's own Step 7 — both recorded in sequence in the fixture's observability log. |
| TC-SEC-SEC3-01 | P2 | PASS *(now fully complete — M3 covered `/forge-init`, this completes `/promote-to-thread`)* | `acme.platform//context` verified via direct regex test to fail the `org/repo` pattern; rejected and re-prompted; `acme/platform-context` verified to pass and was then persisted. |
| TC-EDGE-2 | P2 | PASS | SPEC in `status: revision_requested` (mid-`/reopen-spec`): Step 0 refused cleanly before Step 1 ever ran; frontmatter confirmed unchanged — no `pending_promotion`/`status` corruption. |

**P0 results:** 7 passed, 0 failed (TC-FUNC-19, 21, 22, 25, 26, 27, TC-ADR-0001-01)
**P1+ results:** 7 passed, 0 failed, 0 deferred-to-release

### M5 — Proportional Environment Governance (G4)
**Tasks:** T10
**Implementation status:** approved
**AC Coverage:** AC13, AC14, AC15 (full)
**Depends on:** M1, M3 (reads the `environments` list, documented in M3's T3)
**Rollback:** Safe — `governance: ci` has no command path at all; `governance: release` is untouched; only `governance: promotion` exercises new code.

### M6 — Investigation / Decision Workflow (G5)
**Tasks:** T11, T12, T18b
**Implementation status:** pending
**AC Coverage:** AC16, AC17 (full)
**Depends on:** M1
**Rollback:** Safe — both commands are wholly new; nothing existing changes.

## Happy Path Trace

### US1 — Developer / intake

| Flow Step | Covered by |
|---|---|
| Developer runs `/create-spec --from ADO-14237` | T4 |
| ForgeKit checks for working access, fetches directly | T4, T13 |
| SPEC-NNNN is produced with `source_work`/`product_spec` recorded | T4 |
| ForgeKit later approves the SPEC and offers to update the ticket | T5, T14 |

Gap check: none — all steps covered.

### US2 — Architect

| Flow Step | Covered by |
|---|---|
| Architect runs `/approve-design` on a thread's Interface Freeze | T7 |
| `SYSTEM-ARCH.md`'s Dependency Map regenerates from active XDEPs | T7 |
| Architect reads `SYSTEM-ARCH.md` for the current system topology | T3 (README describing the artifact), T7 (the artifact itself) |

Gap check: none — all steps covered.

### US3 — Lead Developer

| Flow Step | Covered by |
|---|---|
| Lead Developer discovers mid-DESIGN that a feature needs a second repo | *(human judgment — no task; this is the trigger condition, not an implementation step)* |
| Lead Developer runs `/promote-to-thread SPEC-NNNN` | T8 |
| `THREAD-NNNN` is minted, `thread.yml` written, prior reviews re-examined | T8, T16 |
| Original SPEC is marked `status: promoted`/`superseded_by` | T8 |
| `/status SPEC-NNNN` on the original SPEC redirects to the thread | T9 |

Gap check: none — all steps covered.

### US4 — SRE / Lead Developer

| Flow Step | Covered by |
|---|---|
| SRE/Lead Developer runs `/promote-to-environment qa` | T10 |
| A per-participant `candidate_ref` map, promoter, timestamp, and smoke-test result are appended to `sdlc/promotions/qa.md` | T10 |

Gap check: none — all steps covered.

### US5 — Developer / Investigation

| Flow Step | Covered by |
|---|---|
| Developer runs `/create-investigation --from ADO-99001` | T11 |
| ForgeKit walks Question → Approach → Findings → Decision | T11 |
| Decision is recorded (one of four outcomes) | T11 |

Gap check: none — all steps covered.

## Alternate / Failure Path Trace

### US1 — Developer / intake — Alternate Path: no working connector, compliance framework applies, user declines confirmation

| Flow Step | Covered by |
|---|---|
| Fetch attempt fails (no connector) | T4 |
| Developer is prompted to paste content | T4 |
| `compliance.frameworks` is non-`[none]` — confirmation gate fires | T4, T15 |
| Developer declines — SPEC still created, `capture_status: declined`, no `.source.md` written | T4 |

Gap check: none — all steps covered.

### US3 — Lead Developer — Alternate Path: promotion interrupted mid-flight, then resumed

| Flow Step | Covered by |
|---|---|
| `/promote-to-thread` writes `pending_promotion` and is interrupted before completion | T8 |
| `/status SPEC-NNNN` reports "promotion in progress — resume via `/promote-to-thread`" | T9 |
| `/promote-to-thread SPEC-NNNN` is re-invoked, reads `pending_promotion`, resumes from the first incomplete step | T8, T16 (resumed outcome recorded distinctly from a clean run) |

Gap check: none — all steps covered.

### US4 — SRE / Lead Developer — Alternate Path: attempting promotion against a non-`promotion`-governance environment

| Flow Step | Covered by |
|---|---|
| SRE/Lead Developer runs `/promote-to-environment` against an environment whose `governance` is `ci` | T10 (rejects — no command path exists for `ci` at all, per AC14) |
| SRE/Lead Developer runs `/promote-to-environment` against an environment whose `governance` is `release` | T10 (redirects to the existing RELEASE-PLAN commands, unchanged) |

Gap check: none — all steps covered.

### US5 — Developer / Investigation — Alternate Path: decision is "no action"

| Flow Step | Covered by |
|---|---|
| Developer completes Findings, decision is `no_action` | T11 |
| `INV-NNNN` is set `status: concluded`, no ADR, no recommendation, no further command invoked | T11 |

Gap check: none — all steps covered.

(US2 has a single terminal outcome — reading the current topology — with no alternate/failure path distinct enough to warrant tracing; stated explicitly rather than omitted.)

## Out of Scope for This Plan

Per SPEC-0001's own Non-Goals, carried forward unchanged: ADO/Confluence API client implementation (uses whatever access already exists in the environment; ForgeKit builds no client of its own); automatic ADO/Confluence write-back (every write is confirmation-gated, per T5); `EPIC-NNNN`/`FEATURE-NNNN`/`USER-STORY-NNNN` artifact types; a "Workspace" domain model or `workspace.yml`; migrating command distribution to Claude Code plugins; a new QA test-execution framework; a `revision_set` field (reuses `candidate_ref`, per T10); "ForgeKit Operator" (separate milestone, untouched).

Additionally, per DESIGN-0001's own Round 2 non-blocking findings, deliberately not addressed in this PLAN (both are reasonable to defer, per the Address Summary's own disposition): the `[SRE-1]`-adjacent explicit naming of "coordination repo unreachable" as a failure mode (already safe to retry under Option B, just not individually documented); nothing else was deferred — all other Round 1/Round 2 findings are implemented above.

## Definition of Done

| AC | Description | Covered by | Status |
|---|---|---|---|
| AC1 | `/create-spec` accepts an optional external work reference (e.g., an ADO ticket ID). It first checks whether the running environment already has working read access to that system (e.g., a connected MCP server or CLI tool) — if so, it fetches the ticket directly; if not, or if the fetch attempt itself fails (invalid ticket ID, connector error), it prompts the user to paste the content and supply the ticket's identifying fields manually rather than blocking on the failure. Either way, the produced SPEC-NNNN's frontmatter records a `source_work: {system, id, type, parent_epic}` block. | T4 | Covered |
| AC2 | When content is fetched directly (access available), the SPEC's frontmatter records a `product_spec: {system, page_id, revision, fetched_at}` block; no companion file is required, since the connector can independently re-verify that revision later. When content is supplied by copy/paste (no access available), and the initializing project's own `compliance.frameworks` is `[none]`, the SPEC's directory additionally contains an immutable companion file (e.g. `SPEC-NNNN.source.md`), written once at intake and never edited afterward, holding exactly what was supplied; the frontmatter's `product_spec` block adds `captured_at` and `content_digest` fields referencing it, since there is no connector to re-verify against later. When copy/paste is the only path and `compliance.frameworks` is non-`[none]`, ForgeKit warns the user before writing the companion file and requires an explicit "this does not contain regulated data" confirmation before it is created — the companion file is never written automatically in that case, per Security Considerations. | T4, T17 | Covered |
| AC3 | The `product_spec` block's core fields (`system`, `id`/`page_id`, `revision`) are identical regardless of access path — only the presence of `fetched_at` vs. `captured_at`/`content_digest`/a companion file differs, keyed on whether access existed at the moment of intake and, when it didn't, on the compliance confirmation outcome (AC2) — never hardcoded to "no connector exists yet," and never skipping the compliance check when one applies. | T4 | Covered |
| AC4 | When `/approve-spec` approves a SPEC that carries a `source_work` block (AC1), its Handoff step prompts the user: do they want the originating ADO ticket updated to reflect the approved SPEC? This prompt fires on every approval of that SPEC, including a later re-approval after a `/reopen-spec` → `/revise-spec` cycle — not only the first. If yes, ForgeKit checks for working write access the same way AC1 checks read access — if available, it updates the ticket directly (after showing the user what will be written, and only on their explicit confirmation); if not, it generates a plain-text summary for the user to paste in manually. If the write itself fails after confirmation (permission denied, network error, or the ticket having changed since it was last read), ForgeKit reports the failure clearly to the user and falls back to generating the copy-paste summary — it never fails silently. If no `source_work` block is present, this prompt is skipped entirely. | T5 | Covered |
| AC5 | A `repo_role: coordination` repo's `project.yml` may declare `known_participants: [{repo, purpose}]` and `environments: [{name, purpose, governance}]`. Both fields are optional and additive — a coordination repo lacking them is unaffected. | T3 | Covered |
| AC6 | A `repo_role: coordination` repo may hold a durable system-architecture artifact containing: (a) external subsystems the whole system depends on that no single participant's own `ARCH.md` owns, and (b) a Dependency Map section whose content is derived from the current set of `active`-status XDEPs across all threads. | T7 | Covered |
| AC7 | `/forge-init`, when initializing a `repo_role: code` project, asks whether the project participates in cross-repo coordination and, if so, whether to point at an existing coordination repo or defer the decision. An answered value is recorded as `coordination_repo` in the initializing repo's own `project.yml`; "not yet" leaves it unset. | T6 | Covered |
| AC8 | `/promote-to-thread` reads `coordination_repo` from the promoting repo's `project.yml` first. If already set, it is reused without re-asking. If unset, the same question from AC7 is asked at that point instead, and the answer is persisted. | T8 | Covered |
| AC9 | A new command, `/promote-to-thread SPEC-NNNN`, converts a repo-local SPEC into a THREAD-NNNN in the designated coordination repo. Valid from either `status: draft` or `status: approved`. A no-op if already promoted. | T8 | Covered |
| AC10 | Promotion re-examines, on substance, any review round that already completed under the single-repo assumption. | T8 | Covered |
| AC11 | After promotion, the original repo-local SPEC-NNNN is set to `status: promoted`, `superseded_by: THREAD-NNNN` — never deleted. | T8 | Covered |
| AC12 | `/status SPEC-NNNN`, run against a promoted SPEC, reports the promotion explicitly and names the authoritative `THREAD-NNNN`. | T9 | Covered |
| AC13 | Each entry in a coordination repo's `environments` list declares a `governance` value of `ci`, `promotion`, or `release`. Every command reasons about `governance`, never a hardcoded environment name. | T10 | Covered |
| AC14 | A `governance: ci` environment transition requires no new ForgeKit artifact. | T10 | Covered |
| AC15 | A `governance: promotion` environment transition produces a lightweight record — candidate map, who, when, smoke-test result — explicitly excluding Gating-criteria, Rollback plan, or Validation Plan sections. | T10 | Covered |
| AC16 | A command authors a standalone artifact type recording an Investigation's question, approach, findings, and one of four decision outcomes. | T11 | Covered |
| AC17 | A standalone ADR-authoring path exists that does not require a parent `DESIGN-NNNN`. | T12 | Covered |

All 17 ACs fully covered by this milestone set. No deferrals.

## Plan Review — Round 1

**Date:** 2026-10-05
**Status:** Changes requested — 2 blocking, 3 non-blocking

### SRE Findings

- **[SRE-1] non-blocking** — M4's "Depends on: M3" is overstated. T8's own AC8 logic already has a reactive fallback for `coordination_repo` being unset (asks and persists at that point) — M4 doesn't strictly require M3 to be deployed first; it only needs M3's logic *eventually*, not *first*. Worth correcting the dependency note so it doesn't impose an artificial deployment-ordering constraint that doesn't actually exist.

### QE Findings

- **[QE-1] non-blocking** — US5's Alternate Path Trace chose `decision: no_action` as its representative alternate — the least architecturally interesting of the four possible outcomes. `decision: adr` would better exercise the T11→T12 hand-off, which is exactly the kind of cross-task boundary where a planning gap is more likely to hide. The current trace has no internal gap and technically satisfies "at least one representative path," so this is a coverage-quality suggestion, not a missing requirement.

### Architect Findings

- **[ARCH-1] blocking** — DESIGN-0001's Round 2 non-blocking `[SEC-1]` ("D11's sanitization requirement doesn't extend to whatever error `/promote-to-thread` surfaces if the coordination repo itself is unreachable... should extend D11's rule uniformly") was accepted as a developer note at `/approve-design` — which, per that command's own gate, means PLAN is the next and last point where it can still be captured as a real task. No current PLAN task does this. T8's own description covers AC8/AC9/AC10/AC11 and the Round 2 `[LD-1]`/`[LD-2]` wording and re-mint fixes, but never extends D11's sanitization to `/promote-to-thread`'s own coordination-repo-unreachable error path. Left as-is, this finding — real, already identified, already agreed needed uniform extension — would be silently dropped at the one remaining point where ForgeKit's task-based process can still catch it.
- **[ARCH-2] blocking** — Invariant-preservation violation (Architect Perspective check #7): T4, T5, T8, T11, and T12 each only say "points at T1's reference doc" for D9 (any-non-success-is-failure), D11 (sanitize failure diagnostics), and D12 (content containment) — never restating the actual invariant in their own task description. This is exactly the failure mode check #7 exists to catch: a task whose literal wording ("see the reference doc") could be satisfied by a Developer who never actually internalizes the safety property while implementing it. The invariant must be spelled out where the Developer is actually working, not left to a cross-reference.
- **[ARCH-3] non-blocking** — T8, T10, T11, and T12 each implement behavior that was the direct subject of an ADR (ADR-0001, ADR-0004, ADR-0003, ADR-0003 respectively), but each task cites only the corresponding Key Decision letter (D4, D10, D3) — never the ADR ID itself. Add explicit ADR references for traceability.

**Clean perspectives:** Developer, Security

### Contradictions Requiring Arbitration

None.

## Address Summary — Round 1

**Date:** 2026-10-05

### Addressed

- **[ARCH-1]** — Round 2's accepted-as-note DESIGN finding (extend D11's sanitization to `/promote-to-thread`'s own coordination-repo errors) had no PLAN task. Fixed two ways: amended DESIGN-0001 itself (v4) to extend D11's stated scope and update `/promote-to-thread`'s own Interface/API Changes description — not fixing PLAN alone while leaving DESIGN's text narrower than what's now implemented — then added new task **T20** (security, M4) covering it.
- **[ARCH-2]** — T4, T5, T8, T11, and T12 each only pointed at T1's reference doc for D9/D11/D12's invariants instead of stating them inline. Fixed: each task now states its relevant invariant(s) explicitly in its own description, while still pointing at T1 for the full shared mechanism.
- **[SRE-1]** — M4's "Depends on: M3" was overstated given T8's own reactive fallback for `coordination_repo`. Corrected to "M1 only," with the reasoning stated inline in the milestone block — a clear factual correction, not a judgment call, so fixed without being asked per Step 5's own exception.
- **[ARCH-3]** — T8, T10, T11, and T12 cited only their Key Decision letter, not the ADR ID. Added explicit `(ADR-NNNN)` citations alongside each task's command name — mechanical completeness, not a judgment call, so fixed without being asked.

### Not Addressed

- **[QE-1]** — **accepted risk** — US5's Alternate Path Trace uses `decision: no_action` rather than `decision: adr` as its representative path. The current trace has no internal gap and technically satisfies "at least one representative path"; retracing with a different outcome is a coverage-quality judgment call, not a correction, so left as-is per the default non-blocking disposition.

## Plan Review — Round 2

**Date:** 2026-10-05
**Status:** Approved — no blocking findings

`[QE]`: no new findings — domain unaffected by Round 1 address (ARCH-1/ARCH-2/SRE-1/ARCH-3 touched DESIGN fidelity, invariant statements, and a dependency note — none touch AC traceability, testability, or the Happy/Alternate Path Traces; T20 is error-handling, not a new flow step, so the traces don't need revisiting).
`[Security]`: no new findings — domain unaffected (SEC-0001's own findings SEC-1/2/3 weren't touched this round; ARCH-1 is a DESIGN-level finding, not a SEC-0001 one).

### SRE Findings

Re-checked T20's placement (same milestone as T8, so it deploys atomically with the error path it hardens — no transient-vulnerability window) and the corrected M4 dependency note for any new inconsistency. Confirmed clean — no new finding.

### Developer Findings

- **[DEV-1] non-blocking** — T8's "re-mint and update `pending_promotion` if that number is later found already claimed" doesn't state *how* "already claimed" is actually detected. The natural, correct mechanism is the same CI uniqueness-check rejection that THREAD/XDEP ID minting already relies on elsewhere (per `cross-repo-resolution.md`) — not a new content-inspection mechanism comparing `thread.yml`'s contents against the promoting SPEC's own identity, which wouldn't even work (`thread.yml` records participants, not which SPEC originated the promotion). A Developer could resolve this correctly by reading the existing CI-check documentation, but the task description doesn't say so, leaving room to invent a more complex, and potentially wrong, mechanism instead. Worth stating explicitly; the underlying behavior is still correctly resolvable without it.

Milestone sizing re-checked given T20's addition: M4 now has 5 tasks, still well under the 8-10 threshold. No split needed.

### Architect Findings

- **[ARCH-1] non-blocking, flagged for explicit user decision rather than silently resolved** — DESIGN-0001 was already approved and *committed* (as v3) before this Address Summary amended it to v4. There's no established ForgeKit convention for how a post-approval amendment to an already-committed DESIGN gets re-committed: as its own standalone commit now, or folded into whatever commits next (PLAN-0001's eventual `/approve-plan` commit). This is a process question outside this review's own scope to decide — surfaced here, and should be put to the user directly before any further commits happen.

**Clean perspectives:** QE, Security

### Contradictions Requiring Arbitration

None.

## Address Summary — Round 2

**Date:** 2026-10-05

### Not Addressed

- **[DEV-1]** — **accepted** — T8's re-mint-on-collision detection mechanism isn't stated explicitly (it's the existing CI uniqueness-check rejection, not a new mechanism), but the underlying behavior is still correctly resolvable without the clarification. Documentation-clarity improvement, not a factual error — accepted as-is per the default non-blocking disposition.
- **[ARCH-1]** — **accepted, decision recorded** — User's explicit call: DESIGN-0001's v4 amendment rides along with PLAN-0001's eventual `/approve-plan` commit rather than getting its own standalone commit now.
