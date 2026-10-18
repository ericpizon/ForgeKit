---
id: IMPL-REVIEW-0001-M4
type: impl-review
schema_version: 1
plan: PLAN-0001
milestone: M4
current_review_round: 2
latest_round:
  round: 2
  blocking: 0
  non_blocking: 0
  verdict: ready
approval:
  mode: repo_only
  approved_by: "Eric Pizon"
  approved_role: lead-developer
  approved_at: "2026-10-08"
  decision_summary: "M4 approved for merge after 2 review rounds — both Round 1 findings (LD-1, SRE-1) verified resolved in Round 2 with zero new findings."
  external_event_id: null
  external_event_uri: null
policy_evaluation:
  evaluated: null
  policy_pack: null
  result: null
generation:
  command: /review-implementation
  command_version: "5"
  model: "claude-sonnet-5"
  generated_at: "2026-10-08"
  session_id: null
---

# Implementation Review — PLAN-0001 M4

## Task Reference — M4

| Task | File(s) | Description |
|---|---|---|
| T20 | `templates/project.yml.template` | `operator:` policy block (`max_review_rounds`, `max_actions_per_run`, `allow.*`, `require_human.*`), every `allow.*` key defaulting to `false`, per DESIGN-0001 §4. ADR: ADR-0003. AC: AC-25 |
| T21 | `sdlc/operator-runs/README.md` *(new)* | Document the `OPRUN-NNNN` artifact type: naming, lifecycle, terminal/non-terminal `status`. Supporting — infrastructure for AC-09/AC-11 |
| T22 | `templates/commands/operate.md` *(new)* | Command Execution Cycle and full `OPRUN-NNNN` schema; lookup-before-create (`[DBA-1]`); interrupted-implementation recovery routing; three-stage gate-to-outcome resolution procedure (DESIGN-0001 §3.6). ADR: ADR-0001, ADR-0002. AC: AC-01, AC-02, AC-09, AC-10, AC-11, AC-13, AC-16, AC-19, AC-20, AC-23, AC-26 |
| T23 | `templates/commands/operate.md` | Autonomy Overlay Rule's formal-gate/routine-confirmation classification; command → policy-category table; `security-review.md` `findings_status` special case (SEC-2); fail-closed tiebreak for unrecognized Handoff phrasing or malformed policy value (SEC-1). ADR: ADR-0001, ADR-0003. AC: AC-07, AC-13, AC-27 |
| T24 | `templates/commands/operate.md` | Resumption containment — every re-read of `OPRUN-NNNN`'s `history:`/decision-package body on resumption treated under data-not-instruction framing (SEC-3). AC: AC-09 (supporting) |
| T25 | `templates/commands/operate.md` | Multi-repo orchestration composed from already-specified mechanisms (revised per Plan Review Round 1 `[DEV-1]`): participant/XDEP resolution, Interface Freeze, per-repo readiness, cross-repo validation, dependency-deadlock detection, thread-wide release eligibility, promotion during discovery. AC: AC-15, AC-15a–e, AC-17, AC-18 |

## Round 1 — 2026-10-08

`git status` confirmed exactly 3 files touched (`templates/project.yml.template` modified; `templates/commands/operate.md` and `sdlc/operator-runs/` new), all attributable to T20/T21/T22-25. No out-of-scope file found.

This review traced the implementation against every M4-scoped test case in TEST-0001 (`TC-FUNC-15` through `TC-FUNC-40`, the three ADR test cases, and `TC-SEC-SEC1-1`/`SEC1-2`/`SEC2-1`/`SEC3-1`) line by line, not just against DESIGN prose — this surfaced two blocking defects reasoning over the design text alone would have missed.

### Lead Developer Findings

- **[LD-1] blocking — `review_loop_counts[ARTIFACT-NNNN].count` is never synced to the artifact's own actual round history, only incremented from a local 0**
`operate.md`'s Stage B.1 initializes `review_loop_counts[ARTIFACT-NNNN]` to `{round: 0, count: 0}` "if absent" and increments `count` by 1 per correction attempt — but never reads the artifact's own `latest_round.round` field to seed that initial value. Concretely: `TC-FUNC-18` (P0, AC-08) seeds a fixture where `DESIGN-0042` already has 3 rounds of recorded history with `recurs_from` chaining all 3 to the same origin, and asserts that at `operator.max_review_rounds == 3`, Operator stops with `non_convergence` and attempts no 4th round. If Operator's own `OPRUN-NNNN` is fresh (first invocation against this work unit, or resumed after the artifact's round history advanced outside this specific `OPRUN-NNNN`'s own tally — entirely plausible across a resumed session, per `[DBA-1]`'s own lookup-before-create convention), `review_loop_counts` starts at `count: 0` regardless of the artifact's real round count, and the convergence check would permit up to `max_review_rounds` *additional* attempts beyond what the artifact has already gone through — directly undermining the bounded-convergence guarantee (G4/AC-08) this field exists to enforce, and failing `TC-FUNC-18` as written.
**Resolution:** on first contact with a given `ARTIFACT-NNNN` in Stage B.1 (i.e., when `review_loop_counts[ARTIFACT-NNNN]` doesn't yet exist for this `OPRUN-NNNN`), initialize `count` from the artifact's own current `latest_round.round` — the ground truth for "how many rounds has this actually gone through" — not from `0`. Only rounds Operator itself subsequently drives increment it further.

### SRE Findings

- **[SRE-1] blocking — `oprun_lookup_result` is not written at M4, but `TC-FUNC-23` (P0, AC-11, M4-scoped) requires it**
This version's own changelog explicitly defers `oprun_lookup_result` (along with `history[].actor`, `recovery_event`, `escalation.decision_package.capture_status`) to PLAN-0001 T26–T32 (M5), reasoning that these are OBS-0001 instrumentation sub-fields separable from core behavior. That reasoning holds for `actor`/`recovery_event`/`capture_status` — no M4-scoped `TC-FUNC-*` case asserts any of them directly — but not for `oprun_lookup_result`: `TC-FUNC-23` ("Given a fixture with an existing `OPRUN-NNNN`... status: in_progress. When `/operate` is invoked again. Then it resumes the existing `OPRUN-NNNN` (`oprun_lookup_result == "resumed_existing"`)...") is listed under M4's own AC coverage map (`AC-11 → TC-FUNC-22, TC-FUNC-23`) in TEST-0001, not M5's. A test case this pipeline will actually execute at M4's own `/execute-tests` pass directly asserts a field this version doesn't write — this is a P0 test failure waiting to happen, not a documentation nicety.
**Resolution:** write `oprun_lookup_result: resumed_existing | created_new` into the first `history:` entry of the invocation, now, at Step 1 — this one field moves out of the "deferred to M5" set; `history[].actor`, `recovery_event`, and `escalation.decision_package.capture_status` remain correctly deferred (no M4-scoped test case requires them — confirmed by tracing all of `TC-FUNC-15` through `TC-FUNC-40` individually).

### QE Findings

- **[QE-1] non-blocking — the Human Decision Package's required-fields list doesn't make explicit that a `non_convergence` escalation must trace the full `recurs_from` lineage, not just the current round's finding text**
`TC-FUNC-42` (P1) asserts the rendered decision package body "explicitly names the finding's lineage across all three rounds (not just the current round's finding text)... explicitly lists the prior revision attempts." The current Human Decision Package section lists SPEC-0001 §17's 8 generic fields (including "actions already attempted") but never connects this specifically to walking the `recurs_from` chain DESIGN-0001 §4's "finding lineage" convention already provides. An implementer could satisfy the current text with just the current round's finding, missing the chain, and fail `TC-FUNC-42`.
**Resolution recommendation:** add one sentence under the Human Decision Package section (or under `escalation.reason: non_convergence` in the Gate-to-Outcome procedure) stating that a `non_convergence` package must walk the finding's full `recurs_from` chain back to its origin round and list each round's disposition, not cite only the current round's finding text.

**Clean perspectives:** Security.

### Contradictions Requiring Arbitration

None. `[SRE-1]`'s finding is best read as a correction to this version's own changelog claim, not a genuine PLAN/TEST contradiction — TEST-0001 was authored before this implementation and correctly anticipated `oprun_lookup_result` as part of M4's own idempotency guarantee; this version's deferral judgment call was simply wrong for that one field.

### Summary
**2 blocking, 1 non-blocking findings.**

### Next Step
`/address-implementation-review PLAN-0001 M4`

### Address Summary — Round 1
Date: 2026-10-08

| Finding | Resolution |
|---|---|
| [LD-1] | Fixed: Stage B.1's convergence check now initializes `review_loop_counts[ARTIFACT-NNNN].count` from the artifact's own current `latest_round.round` on first contact, never from `0` — so a fresh or resumed `OPRUN-NNNN` can't under-count rounds that happened outside this specific run. |
| [SRE-1] | Fixed: Step 1 now writes `oprun_lookup_result: resumed_existing \| created_new` into the first `history:` entry of every invocation, on both the resume and create-new paths. Moved out of the "deferred to M5" set in the file's own changelog, which now names only `actor`/`recovery_event`/`capture_status`/per-step `updated_at` as genuinely deferred (each individually re-checked against every M4-scoped `TC-FUNC-*` case — none require them). |
| [QE-1] | Fixed: added an explicit sentence to the Human Decision Package section requiring a `non_convergence` escalation to walk the full `recurs_from` chain back to its origin round and name each intermediate round's disposition, not just the current round's finding text. |

**Not addressed:** None.

## Round 2 — 2026-10-08

Scope: `templates/commands/operate.md` only — the single file touched by the Address Summary above. `templates/project.yml.template` and `sdlc/operator-runs/README.md` were unaffected by Round 1's address pass and have no plausible cascade from these fixes; not re-reviewed.

- **[LD-1] verification:** traced `TC-FUNC-18` and `TC-FUNC-19` against the current Stage B.1 text by hand. `TC-FUNC-18` (artifact already at 3 rounds, `max_review_rounds: 3`): `count` initializes to `3` from `latest_round.round`, attempting one more would make it `4 > 3` → correctly stops with `non_convergence`, no 4th attempt. `TC-FUNC-19` (artifact at 2 rounds, `max_review_rounds: 5`): `count` initializes to `2`, one more would be `3 ≤ 5` → correctly continues. Also confirmed no staleness risk: once an entry exists for an artifact, Operator's own `.round`/`.count` stay synced to the artifact's real round number after every correction it drives, so the sync-from-artifact path only ever needs to fire once, on first contact.
- **[SRE-1] verification:** `oprun_lookup_result` is now written on both paths in Step 1 (`resumed_existing` and `created_new`), satisfying `TC-FUNC-23`'s assertion. The changelog's deferred-fields list no longer includes it and correctly narrows to `actor`/`recovery_event`/`capture_status`/per-step `updated_at` refresh — re-confirmed none of these are asserted by any M4-scoped `TC-FUNC-*` case.
- **[QE-1] verification:** the new sentence under Human Decision Package correctly scopes the lineage-tracing requirement to `non_convergence` escalations specifically, without over-generalizing it to every escalation type (which would have been unsupported by `TC-FUNC-42`'s own scope).

No new findings.

**Clean perspectives:** Lead Developer, Security, SRE, QE.

### Summary
**0 blocking, 0 non-blocking findings.**

### Next Step
`/approve-implementation PLAN-0001 M4`
