---
id: TEST-0001
type: test-plan
schema_version: 1
status: passed
version: 2
updated: 2026-10-06
target_plan_version: 2
execution_date: "2026-10-06"
execution_summary: "All P0 automated tests passed across M1-M6. Manual tests deferred to release phase."
current_review_round: 2
changelog:
  - v2 — approved after 2 review rounds
approval:
  mode: repo_only
  approved_by: "Eric Pizon"
  approved_role: quality-engineer
  approved_at: "2026-10-05"
  decision_summary: "TEST-0001 approved for implementation phase — 38 functional + 4 ADR + 4 observability + 3 security + 3 edge-case test cases, all 17 ACs covered with a P0 happy-path test."
  external_event_id: null
  external_event_uri: null
policy_evaluation:
  evaluated: null
  policy_pack: null
  result: null
generation:
  command: /create-test-plan
  command_version: "4"
  model: "claude-sonnet-5"
  generated_at: "2026-10-05"
  session_id: null
---

# TEST-0001 — External Work Intake and Durable Coordination Context

## Test Strategy

ForgeKit's own repo has no compiled code, no database, and no running service — every capability in PLAN-0001 is new or modified command text (markdown instructions an AI agent interprets), plus new frontmatter fields and markdown artifacts. There is no unit/integration/E2E split in the conventional sense. Verification is **scenario-based**: a scripted walkthrough of a command's stated behavior against a disposable test fixture, checking the resulting artifact content, frontmatter, and files against DESIGN-0001's stated contract — the same methodology that validated the original multi-repo coordination feature end-to-end via ParcelFlow before this engagement's standing gate let it commit.

- **Functional cases (`TC-FUNC-*`):** invoke the modified/new command against a realistic fixture repo; assert on the resulting file/frontmatter state.
- **Invariant cases (`TC-ADR-*`):** craft the specific condition each ADR's decision exists to prevent, and confirm it's actually prevented — not just that the happy path works.
- **Observability cases (`TC-OBS-*`):** confirm each structured log field (OBS-0001's signals) emits with the exact named value set.
- **Security cases (`TC-SEC-*`):** craft the adversarial/accidental-exposure condition each SEC-0001 finding describes, confirm the mitigation actually fires.

**The one place mocking is appropriate:** the external ADO/Confluence connector itself. ForgeKit owns no API client (per SPEC-0001's own Non-Goals) and tests nothing about third-party system behavior — only its own capability-check-then-branch logic (D5). Automated cases simulate the connector's success/failure responses; this is a fundamentally different boundary than "mocking your own database," which doesn't apply here since there is no database.

## AC Coverage Map

### M1 — Foundation
*(No ACs of its own — T1/T2/T3 are infrastructure. Covered implicitly by every milestone below that depends on them.)*

### M2 — External Work Intake (G1)

| AC | Test Cases |
|---|---|
| AC1 | TC-FUNC-1 (integration, automated, P0), TC-FUNC-2 (integration, automated, P1), TC-FUNC-3 (integration, automated, P1) |
| AC2 | TC-FUNC-4 (integration, automated, P0), TC-FUNC-5 (integration, automated, P0), TC-FUNC-6 (integration, automated, P1), TC-FUNC-7 (integration, automated, P1) |
| AC3 | TC-FUNC-8 (integration, automated, P0) |
| AC4 | TC-FUNC-9 (integration, automated, P0), TC-FUNC-10 (integration, automated, P1), TC-FUNC-11 (integration, automated, P1), TC-FUNC-12 (integration, automated, P2), TC-FUNC-13 (integration, automated, P1) |

### M3 — Durable Coordination-Repo Context (G2)

| AC | Test Cases |
|---|---|
| AC5 | TC-FUNC-14 (integration, automated, P0), TC-FUNC-14b (manual, manual, P2) |
| AC6 | TC-FUNC-15 (integration, automated, P0), TC-FUNC-16 (integration, automated, P1) |
| AC7 | TC-FUNC-17 (integration, automated, P0), TC-FUNC-18 (integration, automated, P2) |

### M4 — Single-Repo → Cross-Repo Promotion (G3)

| AC | Test Cases |
|---|---|
| AC8 | TC-FUNC-19 (integration, automated, P0), TC-FUNC-20 (integration, automated, P1) |
| AC9 | TC-FUNC-21 (integration, automated, P0), TC-FUNC-22 (integration, automated, P0), TC-FUNC-23 (integration, automated, P1), TC-FUNC-24 (integration, automated, P1) |
| AC10 | TC-FUNC-25 (integration, automated, P0) |
| AC11 | TC-FUNC-26 (integration, automated, P0) |
| AC12 | TC-FUNC-27 (integration, automated, P0), TC-FUNC-28 (integration, automated, P1) |

### M5 — Proportional Environment Governance (G4)

| AC | Test Cases |
|---|---|
| AC13 | TC-FUNC-29 (integration, automated, P0), TC-FUNC-30 (integration, automated, P2) |
| AC14 | TC-FUNC-31 (integration, automated, P0) |
| AC15 | TC-FUNC-32 (integration, automated, P0), TC-FUNC-33 (integration, automated, P1) |

### M6 — Investigation / Decision Workflow (G5)

| AC | Test Cases |
|---|---|
| AC16 | TC-FUNC-34 (integration, automated, P0), TC-FUNC-35 (integration, automated, P0), TC-FUNC-36 (integration, automated, P1), TC-FUNC-37 (integration, automated, P2) |
| AC17 | TC-FUNC-38 (integration, automated, P0) |

## Test Case Detail

**TC-FUNC-1 (integration, automated, P0):** Given a connected MCP/CLI tool simulating working ADO access, When `/create-spec --from ADO-14237` runs, Then the ticket is fetched directly and the produced SPEC's frontmatter records `source_work: {system, id, type, parent_epic}`.

**TC-FUNC-2 (integration, automated, P1):** Given no working connector, When `/create-spec --from ADO-14237` runs, Then the user is prompted to paste content, and `source_work` is still recorded identically.

**TC-FUNC-3 (integration, automated, P1):** Given a connector that reports a fetch failure (not "absent," an actual error), When the command runs, Then it falls back to the paste prompt rather than blocking.

**TC-FUNC-4 (integration, automated, P0):** Given content fetched directly, When the SPEC is created, Then `product_spec` records `{system, page_id, revision, fetched_at}` and no companion file exists.

**TC-FUNC-5 (integration, automated, P0):** Given content pasted and `compliance.frameworks: [none]`, When the SPEC is created, Then a `.source.md` companion file is written automatically and `product_spec` includes `captured_at`/`content_digest`.

**TC-FUNC-6 (integration, automated, P1):** Given content pasted and `compliance.frameworks` is non-`[none]`, When the SPEC is created, Then the user is warned and must explicitly confirm "this does not contain regulated data" before the companion file is written.

**TC-FUNC-7 (integration, automated, P1):** Given the compliance confirmation from TC-FUNC-6 is declined, When the SPEC is created, Then `capture_status: declined` is recorded, no companion file exists, and SPEC creation still succeeds — not blocked.

**TC-FUNC-8 (integration, automated, P0):** Given a SPEC created via either the fetch or the paste path, Then `product_spec`'s core fields (`system`, `id`/`page_id`, `revision`) are present and identically shaped regardless of path.

**TC-FUNC-9 (integration, automated, P0):** Given a SPEC with `source_work` and working write access, When `/approve-spec` runs, Then the user is shown the exact content to be written and, on confirmation, the ticket is updated directly.

**TC-FUNC-10 (integration, automated, P1):** Given no write access, When `/approve-spec` runs, Then a copy-paste summary is generated instead.

**TC-FUNC-11 (integration, automated, P1):** Given a write attempt that fails after confirmation (including a simulated partial success), Then the failure is reported as a sanitized category — never raw connector output — and falls back to the copy-paste summary.

**TC-FUNC-12 (integration, automated, P2):** Given a SPEC with no `source_work` block, When `/approve-spec` runs, Then the write-back prompt is skipped entirely.

**TC-FUNC-13 (integration, automated, P1):** Given a SPEC re-approved after a `/reopen-spec` → `/revise-spec` cycle, Then the write-back prompt fires again, not only on first approval.

**TC-FUNC-14 (integration, automated, P0):** Given a coordination repo's `project.yml` with `known_participants`/`environments` added, When the file is parsed, Then it accepts the fields without error (schema acceptance) — this part is mechanical and automatable even though the fields have no dedicated authoring command.

**TC-FUNC-14b (manual, manual, P2):** *Procedure:* with `known_participants`/`environments` present per TC-FUNC-14, run the existing command suite against that repo. *Reason:* confirming no *other* command's behavior silently changes because of their presence requires cross-suite manual confirmation, not a single automatable assertion.

**TC-FUNC-15 (integration, automated, P0):** Given a thread's Cross-Repo Contract reaching Interface Freeze with new `active` XDEPs, When `/approve-design` completes that step, Then `SYSTEM-ARCH.md`'s Dependency Map is regenerated wholesale to reflect the current active XDEP set.

**TC-FUNC-16 (integration, automated, P1):** Given a malformed XDEP file encountered during regeneration, Then a warning is logged and the Interface Freeze approval still proceeds — not blocked.

**TC-FUNC-17 (integration, automated, P0):** Given a `repo_role: code` project running `/forge-init`, When asked about cross-repo coordination and an existing coordination repo is named, Then `coordination_repo` is recorded in `project.yml`.

**TC-FUNC-18 (integration, automated, P2):** Given "not yet" is answered, Then `coordination_repo` remains unset and is not proactively re-asked outside of `/forge-init`'s Option C or a full redo.

**TC-FUNC-19 (integration, automated, P0):** Given `coordination_repo` already set, When `/promote-to-thread` runs, Then it's reused without re-asking.

**TC-FUNC-20 (integration, automated, P1):** Given `coordination_repo` unset, When `/promote-to-thread` runs, Then the question is asked reactively and the answer is persisted to `project.yml`.

**TC-FUNC-21 (integration, automated, P0):** Given a repo-local SPEC in `status: draft`, When `/promote-to-thread` runs, Then a `THREAD-NNNN` is minted and `pending_promotion` is written to the SPEC's own frontmatter before any write to the coordination repo, and `sdlc/THREAD-NNNN/thread.yml` is written in the coordination repo listing this repo as a participant.

**TC-FUNC-22 (integration, automated, P0):** Given a repo-local SPEC in `status: approved`, When `/promote-to-thread` runs, Then the same promotion mechanism succeeds — both `draft` and `approved` are valid trigger points.

**TC-FUNC-23 (integration, automated, P1):** Given a SPEC already `status: promoted`, When `/promote-to-thread` is invoked again, Then it's a no-op reporting the existing `THREAD-NNNN` — not a second mint, not an error.

**TC-FUNC-24 (integration, automated, P1):** Given `/promote-to-thread` is interrupted after writing `pending_promotion` but before completing, When it's re-invoked, Then it resumes from the first incomplete step without minting a second `THREAD-NNNN`.

**TC-FUNC-25 (integration, automated, P0):** Given a promoted thread whose SPEC had prior ANALYSIS/DESIGN review rounds, When promotion re-examines them, Then each prior finding's disposition (still valid / invalidated / needs re-review) is recorded explicitly — not a blanket restart, not a silent carry-forward.

**TC-FUNC-26 (integration, automated, P0):** Given a completed promotion, Then the original SPEC is `status: promoted`, `superseded_by: THREAD-NNNN`, and the file still exists.

**TC-FUNC-27 (integration, automated, P0):** Given a promoted SPEC, When `/status SPEC-NNNN` runs, Then it reports the promotion and names the `THREAD-NNNN`.

**TC-FUNC-28 (integration, automated, P1):** Given a SPEC with `pending_promotion` set but not yet `status: promoted`, When `/status` runs, Then it reports "promotion in progress — resume via `/promote-to-thread`."

**TC-FUNC-29 (integration, automated, P0):** Given environments with different `governance` values, When any command reasons about environment progression, Then it branches on `governance`, never a hardcoded environment name.

**TC-FUNC-30 (integration, automated, P2):** Given an arbitrarily-named environment (not "dev"/"qa"/"production"), Then commands still behave correctly based on its `governance` value alone.

**TC-FUNC-31 (integration, automated, P0):** Given an environment with `governance: ci`, When `/promote-to-environment` is attempted against it, Then there is no command path at all.

**TC-FUNC-32 (integration, automated, P0):** Given an environment with `governance: promotion`, When `/promote-to-environment` runs, Then a row is appended to `sdlc/promotions/{environment}.md` with the per-participant candidate map, promoter, timestamp, and smoke-test result.

**TC-FUNC-33 (integration, automated, P1):** Given the same command, Then the record explicitly excludes any Gating-criteria table, Rollback plan, or Validation Plan section.

**TC-FUNC-34 (integration, automated, P0):** Given `/create-investigation` walks Question → Approach → Findings, When a `no_action` decision is recorded, Then `INV-NNNN` is `status: concluded` with no further action taken.

**TC-FUNC-35 (integration, automated, P0):** Given a decision of `adr`, Then `/create-adr` is invoked and `INV-NNNN.linked_adr` is set to the resulting ADR's ID.

**TC-FUNC-36 (integration, automated, P1):** Given a decision of `recommend_implementation`, Then the recommendation is written as plain advisory text only — no external ticket is created or written to.

**TC-FUNC-37 (integration, automated, P2):** Given a decision of `investigate_further`, Then `INV-NNNN` remains `status: open`.

**TC-FUNC-38 (integration, automated, P0):** Given `/create-adr` is invoked with no parent DESIGN or SPEC, Then a standalone `ADR-NNNN` is authored with all required sections and no `target_design` pointer.

## ADR / Invariant Test Cases

**TC-ADR-0001-01 (integration, automated, P0 — data-integrity boundary):** Given `/promote-to-thread` is interrupted immediately after `pending_promotion` is committed but before any coordination-repo write, When it's re-invoked, Then it does **not** mint a second `THREAD-NNNN` — it reads `pending_promotion` and resumes. This is the specific condition ADR-0001's state-inspection decision exists to prevent (distinct from TC-FUNC-24's happy-path framing of the same mechanism).

**TC-ADR-0002-01 (integration, automated, P1):** Given a manual hand-edit is made directly to `SYSTEM-ARCH.md`'s Dependency Map section, When the next Interface Freeze event regenerates it, Then the hand-edit is overwritten by the wholesale regeneration — proving the section is genuinely derived, not incrementally preserving manual edits (which would violate ADR-0002's own decision).

**TC-ADR-0003-01 (integration, automated, P2):** Given `/create-investigation` authors `INV-NNNN`, Then its frontmatter `type` is `investigation` (not `bug` or `spec`) and it lives at `sdlc/investigations/` — confirming it doesn't silently reuse BUG's or SPEC's own artifact shape, which ADR-0003 explicitly rejected.

**TC-ADR-0004-01 (integration, automated, P1):** Given two separate `/promote-to-environment` invocations against the same environment, When both complete, Then `sdlc/promotions/{environment}.md` contains two rows in **one** file — not two separate numbered artifacts — confirming the append-only-log decision holds rather than silently reverting to ForgeKit's usual per-event-numbered-artifact convention.

## Happy Path Coverage

### US1 — Developer / intake

| Flow Step | Test Case(s) |
|---|---|
| Developer runs `/create-spec --from ADO-14237` | TC-FUNC-1 |
| ForgeKit checks for working access, fetches directly | TC-FUNC-1 |
| SPEC-NNNN is produced with `source_work`/`product_spec` recorded | TC-FUNC-1, TC-FUNC-4 |
| ForgeKit later approves the SPEC and offers to update the ticket | TC-FUNC-9 |

Gap check: none — all steps covered.

### US2 — Architect

| Flow Step | Test Case(s) |
|---|---|
| Architect runs `/approve-design` on a thread's Interface Freeze | TC-FUNC-15 |
| `SYSTEM-ARCH.md`'s Dependency Map regenerates from active XDEPs | TC-FUNC-15 |
| Architect reads `SYSTEM-ARCH.md` for the current system topology | *(no test case — reading the artifact is not itself a verifiable behavior; TC-FUNC-15 already verifies the artifact's content is correct)* |

Gap check: none — all steps covered or explicitly justified.

### US3 — Lead Developer

| Flow Step | Test Case(s) |
|---|---|
| Lead Developer discovers mid-DESIGN that a feature needs a second repo | *(no test case — human judgment, not a system behavior, per PLAN-0001's own trace)* |
| Lead Developer runs `/promote-to-thread SPEC-NNNN` | TC-FUNC-21, TC-FUNC-22 |
| `THREAD-NNNN` is minted, `thread.yml` written, prior reviews re-examined | TC-FUNC-21, TC-FUNC-25 |
| Original SPEC is marked `status: promoted`/`superseded_by` | TC-FUNC-26 |
| `/status SPEC-NNNN` on the original SPEC redirects to the thread | TC-FUNC-27 |

Gap check: none — all steps covered or explicitly justified.

### US4 — SRE / Lead Developer

| Flow Step | Test Case(s) |
|---|---|
| SRE/Lead Developer runs `/promote-to-environment qa` | TC-FUNC-32 |
| A per-participant candidate map, promoter, timestamp, and smoke-test result are appended to `sdlc/promotions/qa.md` | TC-FUNC-32 |

Gap check: none — all steps covered.

### US5 — Developer / Investigation

| Flow Step | Test Case(s) |
|---|---|
| Developer runs `/create-investigation --from ADO-99001` | TC-FUNC-34 |
| ForgeKit walks Question → Approach → Findings → Decision | TC-FUNC-34 |
| Decision is recorded (one of four outcomes) | TC-FUNC-34, TC-FUNC-35, TC-FUNC-36, TC-FUNC-37 |

Gap check: none — all steps covered.

## Alternate Path Coverage

### US1 — Alternate Path: no working connector, compliance framework applies, user declines confirmation

| Flow Step | Test Case(s) |
|---|---|
| Fetch attempt fails (no connector) | TC-FUNC-2 |
| Developer is prompted to paste content | TC-FUNC-2 |
| `compliance.frameworks` is non-`[none]` — confirmation gate fires | TC-FUNC-6 |
| Developer declines — SPEC still created, `capture_status: declined`, no `.source.md` written | TC-FUNC-7 |

Gap check: none — all steps covered.

### US3 — Alternate Path: promotion interrupted mid-flight, then resumed

| Flow Step | Test Case(s) |
|---|---|
| `/promote-to-thread` writes `pending_promotion` and is interrupted | TC-FUNC-24, TC-ADR-0001-01 |
| `/status SPEC-NNNN` reports promotion in progress | TC-FUNC-28 |
| `/promote-to-thread` is re-invoked, reads `pending_promotion`, resumes | TC-FUNC-24, TC-ADR-0001-01 |

Gap check: none — all steps covered.

### US4 — Alternate Path: attempting promotion against a non-`promotion`-governance environment

| Flow Step | Test Case(s) |
|---|---|
| Attempt against `governance: ci` | TC-FUNC-31 |
| Attempt against `governance: release` | *(no new test case — redirects to the existing, already-tested RELEASE-PLAN commands, unchanged by this PLAN)* |

Gap check: none — all steps covered or explicitly justified.

### US5 — Alternate Path: decision is "no action"

| Flow Step | Test Case(s) |
|---|---|
| Developer completes Findings, decision is `no_action` | TC-FUNC-34 |
| `INV-NNNN` is `status: concluded`, no ADR, no recommendation, no further command invoked | TC-FUNC-34 |

Gap check: none — all steps covered.

## Edge Cases and Boundary Conditions

**TC-EDGE-1 (integration, automated, P2):** Given `compliance.frameworks: [soc2, gdpr]` (multiple frameworks, not a single value), confirm AC2's confirmation gate still fires correctly — triggered by "non-`[none]`," not by matching one specific framework.

**TC-EDGE-2 (integration, automated, P2):** Given a SPEC in an unexpected status outside `draft`/`approved`/`promoted` (e.g., mid-`/reopen-spec`), When `/promote-to-thread` is invoked, Then it errors clearly rather than silently corrupting `pending_promotion`/`status` state. *(AC9 only names three valid statuses; this confirms the fourth case fails safely rather than being undefined.)*

**TC-EDGE-3 (integration, automated, P2):** Given an `environments` entry with a missing or invalid `governance` value, confirm the relevant commands reject or error clearly rather than silently defaulting to an assumption. *(Neither SPEC-0001 nor DESIGN-0001 explicitly states this behavior — flagging it here for `/review-test-plan` to assess whether it reveals a genuine spec gap.)*

## Observability Verification

**TC-OBS-source_work_intake_result-01 (integration, automated, P1):** Given each of the three outcomes (`fetched`, `pasted`, `fetch_failed`) is triggered in turn, Then the `source_work.intake_result` log field emits with exactly that value each time, and no other value fires for the wrong trigger.

**TC-OBS-source_work_write_back_result-01 (integration, automated, P1):** Same pattern for `source_work.write_back_result`'s three values (`updated`, `copy_paste_fallback`, `write_failed`).

**TC-OBS-compliance_gate_outcome-01 (integration, automated, P1):** Same pattern for `compliance_gate.outcome`'s three values (`captured`, `declined`, `not_applicable`).

**TC-OBS-promote_to_thread_outcome-01 (integration, automated, P1):** Given a clean run, Then `completed` is logged at its final step. Given an interrupted run that is later re-invoked, Then the *resuming* invocation logs `interrupted` (for the prior attempt) immediately upon discovering `pending_promotion` already set — before proceeding — and separately logs `resumed_and_completed` at its own final step. All three values are confirmed to fire only for their actual trigger.

## Security Test Cases

**TC-SEC-SEC1-01 (integration, automated, P1 — SEC-0001 SEC-1, Medium):** Given pasted content containing a credential-shaped string (matching one of the patterns named in SEC-0001, e.g. `sk-[A-Za-z0-9]{20,}`) and `compliance.frameworks: [none]`, When `/create-spec` processes it, Then the "this does not contain regulated data" confirmation prompt is shown before any write, and the `.source.md` file does not exist on disk unless and until that confirmation is given.

**TC-SEC-SEC2-01 (integration, automated, P1 — SEC-0001 SEC-2, Medium):** Given a fetched or pasted ticket title containing path-traversal-shaped characters (`../../etc`), When a slug is derived (in `/create-spec`, `/create-investigation`, or `/create-adr`), Then the resulting slug contains only `[a-z0-9-]` characters.

**TC-SEC-SEC3-01 (integration, automated, P2 — SEC-0001 SEC-3, Low):** Given a malformed `coordination_repo` value (not matching the `org/repo` shape) is entered during `/forge-init`'s Q&A or `/promote-to-thread`'s reactive equivalent, Then the value is rejected and re-prompted rather than written to `project.yml` as-is.

All three test the failing/defensive path directly, per Step 7's requirement — none merely confirm the happy path still works.

## Test Data Strategy

No database exists, so "test data" means disposable fixture repos — a throwaway local git repo per test run (mirroring the `ParcelFlow-brownfield` pattern already used to validate the original multi-repo feature), never shared state between test cases. Each test case creates its own fixture, runs the scenario, and discards it. The one external boundary — the ADO/Confluence connector — is simulated via a mocked connector interface for automated cases (see Test Strategy above); this is the sole place mocking is appropriate, since ForgeKit owns and tests only its own capability-check-then-branch logic, never third-party system behavior.

## Execution Results Summary

**Status:** pending — no milestone has been implemented yet.

## Review and Justification of Deferred Test Cases

**None deferred.** ForgeKit has no production-only constraint analogous to a real OAuth callback URL or a serverless cold start — every capability here is testable in a disposable local git fixture with a mocked external connector, the same way the entire original multi-repo coordination feature was validated via ParcelFlow before it was allowed to commit. No P0 (or any priority) test case requires anything beyond that fixture.

## Test Plan Review — Round 1

**Date:** 2026-10-05
**Status:** Changes requested — 2 blocking, 1 non-blocking

### SRE Findings

- **[SRE-1] blocking, traces back to an already-approved artifact** — `TC-OBS-promote_to_thread_outcome-01` must verify the `interrupted` value, but OBS-0001's own instrumentation task (IT-4) states the log emission happens "as `/promote-to-thread`'s literal last action" — which an interrupted run, by construction, never reaches. Neither OBS-0001 nor DESIGN-0001 states what mechanism actually logs `interrupted`. As currently specified, this test case cannot be written against an implementable behavior. Likely fix: the *resuming* invocation — which discovers `pending_promotion` already set from a prior incomplete run — logs `interrupted` (for that prior attempt) upon making that discovery, early in its own run, then separately logs `completed`/`resumed_and_completed` at its own last action. Both values still come from `/promote-to-thread` itself, just at two different points instead of one. This needs amending in OBS-0001 (and possibly DESIGN-0001's own description of the signal) — the same kind of post-approval amendment situation as `[ARCH-1]` during `/address-plan-review`, and should be surfaced to the user the same way rather than decided here.

### Lead Developer Findings

- **[LD-1] blocking** — US3's Happy Path Coverage row "`THREAD-NNNN` is minted, `thread.yml` written, prior reviews re-examined" cites `TC-FUNC-21` and `TC-FUNC-25` — but `TC-FUNC-21`'s own `Then` clause only asserts `THREAD-NNNN` minting and `pending_promotion`; it never asserts anything about `thread.yml` being written. No other test case asserts on `thread.yml`'s content or existence either. This is false coverage for that specific sub-claim, exactly the failure mode this check exists to catch — a test cited as covering a step that it doesn't actually exercise.

### Security Findings

- **[SEC-1] non-blocking** — `TC-SEC-SEC1-01`'s `Then` clause ("the confirmation gate fires... rather than auto-capturing the companion file") is adequate but could be tightened to name the exact checkable assertion directly — e.g. "the `.source.md` file does not exist on disk unless confirmed" — rather than the somewhat general "the gate fires" framing.

**Clean perspectives:** *(none — all three perspectives produced at least one finding)*

### Contradictions Requiring Arbitration

None.

## Address Summary — Round 1

**Date:** 2026-10-05

### Addressed

- **[SRE-1]** — OBS-0001's `promote_to_thread.outcome` described `interrupted` as logged at the same final step as `completed`/`resumed_and_completed` — unreachable for a run that was actually interrupted. Amended OBS-0001 directly (its Key Signals table, instrumentation task IT-4, and Validation Checklist): `interrupted` is now logged by the *resuming* invocation upon discovering `pending_promotion`, before it proceeds; `completed`/`resumed_and_completed` remain logged at final-step completion. Updated TEST-0001's own `TC-OBS-promote_to_thread_outcome-01` to match. Proceeded with this specific fix without further user confirmation beyond "address-test-plan-review," per the user's own direction to move forward.
- **[LD-1]** — `TC-FUNC-21` was cited as covering "`thread.yml` written" but never asserted on it. Extended `TC-FUNC-21`'s own `Then` clause to assert `thread.yml` is written in the coordination repo, listing this repo as a participant.
- **[SEC-1]** — `TC-SEC-SEC1-01`'s `Then` clause tightened to name the exact checkable assertion (the confirmation prompt, and the `.source.md` file's absence pending confirmation) rather than the more general "the gate fires" framing. Addressed without being asked since it's a clear precision improvement with no judgment call involved.

### Not Addressed

*(None — all three Round 1 findings were addressed.)*

## Test Plan Review — Round 2

**Date:** 2026-10-05
**Status:** Approved — no blocking findings

### SRE Findings

Re-checked the amended `interrupted`-logging mechanism specifically for new failure modes: a promotion interrupted and never resumed (no second invocation ever occurs, so `interrupted` never logs at all) — confirmed acceptable, since that case remains observable through `/status`'s own "promotion in progress" detection (AC12), a different channel for a different purpose; a promotion interrupted twice in succession — confirmed each new invocation correctly logs `interrupted` exactly once for whatever incomplete state it finds, with no double-logging or loss; whether discovering `pending_promotion` on this SPEC could be confused with the unrelated "different SPEC claimed my candidate number" collision scenario — confirmed no conflation, since `pending_promotion` being set on *this* SPEC unambiguously means a prior invocation against *this same SPEC*, a different concern entirely from the coordination-repo-side collision case. No new finding.

### Lead Developer Findings

Re-checked `TC-FUNC-21`'s extended `Then` clause against the Happy Path Coverage row it's cited for — all three sub-claims ("`THREAD-NNNN` minted," "`thread.yml` written," "prior reviews re-examined") are now genuinely exercised by their cited test cases (`TC-FUNC-21` for the first two, `TC-FUNC-25` for the third). No new finding.

### Security Findings

- **[SEC-1] non-blocking** — `TC-SEC-SEC1-01`'s `Given` clause doesn't specify whether the compliance confirmation is given or declined, and its revised `Then` clause now asserts both outcomes at once ("the prompt is shown... and the file does not exist unless and until confirmed") rather than one specific scenario. Not a coverage gap — `TC-FUNC-6`/`TC-FUNC-7` already separately and unambiguously test the confirm/decline branches — but worth splitting into two explicit scenarios (declined → no file; confirmed → file exists) for clarity, the same rigor already applied elsewhere.

**Clean perspectives:** SRE, Lead Developer

### Contradictions Requiring Arbitration

None.

## Address Summary — Round 2

**Date:** 2026-10-05

### Not Addressed

- **[SEC-1]** — **accepted** — `TC-SEC-SEC1-01`'s Given/Then clause covers both the declined and confirmed outcomes in one scenario rather than two separate ones. Not a coverage gap (`TC-FUNC-6`/`TC-FUNC-7` already test both branches unambiguously elsewhere); a clarity improvement, not a correction, so accepted as-is per the default non-blocking disposition.
