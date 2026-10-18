---
id: IMPL-REVIEW-0001-M1
type: impl-review
schema_version: 1
plan: PLAN-0001
milestone: M1
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
  approved_at: "2026-10-07"
  decision_summary: "M1 (Structured Review Outcomes & Status Resolver) approved for merge after 2 review rounds — all findings resolved, zero remaining."
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
  generated_at: "2026-10-07"
  session_id: null
---

# Implementation Review — PLAN-0001 M1

## Task Reference — M1

*Derived from PLAN-0001 at review creation time.*

| Task | File(s) | Description |
|---|---|---|
| T1 | `templates/commands/review-design.md` | Add `latest_round` frontmatter field, written alongside the existing prose `**Status:**` line, per DESIGN-0001 §4/Key Decision 2 |
| T2 | `templates/commands/review-plan.md` | Same `latest_round` field addition as T1 |
| T3 | `templates/commands/review-test-plan.md` | Same `latest_round` field addition as T1 |
| T4 | `templates/commands/review-implementation.md` | Same `latest_round` field addition as T1, replacing the existing `**Summary:**` prose line's sole role as the gate signal |
| T5 | `templates/commands/review-release-plan.md` | Same `latest_round` field addition as T1, replacing the bare bold-line prose's sole role as the gate signal |
| T6 | `templates/commands/address-design-review.md` | Add the `recurs_from: {round, finding}` finding-lineage convention to the "On Re-Review" section |
| T7 | `templates/commands/address-plan-review.md` | Same `recurs_from` convention as T6 |
| T8 | `templates/commands/address-test-plan-review.md` | Same `recurs_from` convention as T6 |
| T9 | `templates/commands/address-implementation-review.md` | Same `recurs_from` convention as T6 |
| T10 | `templates/commands/address-release-plan-review.md` | Same `recurs_from` convention as T6 |
| T11 | `templates/commands/status.md` | Add the general pre-PLAN form (reads SPEC/ANALYSIS/DESIGN/OBS/SEC frontmatter directly) |
| T12 | `templates/commands/status.md` | Fix the Next Action Inference's milestone branch (explicit handling for `test_failed`/`tests_passed`/`pr_open`) |
| T13 | `templates/commands/status.md` | Add the `resolved_state` structured output block (schema per ADR-0002, including `ambiguous_cause`); add the pre-existing-artifact fallback rule |

## Round 1 — 2026-10-07

`git status` confirmed exactly 11 command template files modified, each attributable to one or more M1 tasks above, plus `sdlc/plans/PLAN-0001-forgekit-operator.md`'s own `in_progress` status flip (expected PLAN bookkeeping, not a code change). No out-of-scope file found.

### Lead Developer Findings

- **[LD-1] non-blocking — `status.md`'s milestone scan has no stated output for "every remaining milestone is pending-and-blocked, none actionable"**
`status.md`'s rewritten Next Action Inference (T12) correctly handles all seven milestone states for the *first* milestone not yet `pr_merged`, including "note MN as blocked on that dependency; continue checking the next milestone." But it never states what to emit if the scan reaches the end of the milestone list with every remaining one in that blocked-pending state (a true dependency deadlock at the milestone level — which shouldn't occur in a correctly-ordered PLAN like PLAN-0001's own strictly linear M1→M2/M3→M4→M5 chain, but is a real gap for `status.md` as a general-purpose template other projects will also use). The existing generic "Ambiguous state" fallback at the bottom of Next Action Inference would likely catch this by default, but it's worth stating explicitly rather than relying on that fallback implicitly.
Recommendation: add one explicit line — "If every milestone is either `pr_merged` or blocked-pending with no dependency chain resolving, report this explicitly as a milestone-level deadlock, naming the blocking dependencies" — rather than letting it fall through silently to the generic ambiguous-state message.

### SRE Findings

- **[SRE-1] blocking — `status.md`'s general pre-PLAN form (T11) assumes ANALYSIS-NNNN exists whenever SPEC-NNNN is `draft`, with no existence check**
Step 1 of the new General pre-PLAN form reads: "If `status` is `draft` or `revision_requested`, read ANALYSIS-NNNN frontmatter..." — but ANALYSIS-NNNN does not exist at all immediately after `/create-spec`, before `/review-spec` has ever run once. This is not a rare edge case — it is the *normal* state of a freshly-drafted SPEC, arguably the single most common state this new form will actually be invoked against (a developer running `/status SPEC-0042` right after drafting it, exactly the "cold-start orientation" use case this command exists for). As written, the instruction has no defined behavior for this case — it just says to read a file that may not exist yet.
Recommendation: add an explicit existence check before Step 1's ANALYSIS read: "If ANALYSIS-NNNN does not exist yet, the next action is `/review-spec SPEC-NNNN` — do not attempt to read it." This is a one-line fix, not a design change — DESIGN-0001 §3.5 never actually specified this edge case either way, so there's no DESIGN conflict in fixing it here.

### QE Findings

- **[QE-1] non-blocking — TEST-0001 has no test case exercising the exact scenario `[SRE-1]` identifies**
`TC-FUNC-2` (the test case covering `AC-04`, pre-PLAN resolution) uses a fixture with SPEC-NNNN already `status: approved`, which never touches the `draft`-with-no-ANALYSIS-yet branch `[SRE-1]` found broken. This means `[SRE-1]`'s gap could ship without any test case catching it, even after this milestone's own test cases are run. Not blocking this round — it's a TEST-0001 coverage gap, not an M1 implementation gap — but worth a TEST-0001 addendum (a new P1 case under `AC-04`) once `[SRE-1]` is fixed, so the fix is actually verified rather than just asserted.

**Clean perspectives:** Security.

### Contradictions Requiring Arbitration

None. One cascade already noted above: `[QE-1]` is downstream of `[SRE-1]` — the same gap, from the test-coverage angle rather than the implementation angle.

### Summary
**1 blocking, 2 non-blocking findings.**

### Next Step
`/address-implementation-review PLAN-0001 M1`

### Address Summary — Round 1
Date: 2026-10-07

| Finding | Resolution |
|---|---|
| [SRE-1] | Fixed: `status.md`'s general pre-PLAN form now checks whether ANALYSIS-NNNN exists before reading it; if absent, reports `/review-spec SPEC-NNNN` as the next action instead of attempting an undefined read. |
| [LD-1] | Fixed: added an explicit "milestone-level deadlock" output for the case where every remaining milestone is blocked-pending with none actionable, rather than relying on the generic ambiguous-state fallback implicitly. |
| [QE-1] | Fixed: added `TC-FUNC-43` to TEST-0001 (AC-04), exercising the exact draft-SPEC-with-no-ANALYSIS-yet scenario `[SRE-1]` fixed — so the fix is now verified, not just asserted. TEST-0001 bumped to v3. |

**Not addressed:** None — all three findings from Round 1 were fixed.

## Round 2 — 2026-10-07

Scoped to the two files Round 1's address summary touched: `status.md` and `TEST-0001-forgekit-operator.md`. Verified each fix against its actual text, not just the Address Summary's claim:

- `[SRE-1]`: confirmed `status.md`'s Step 1 now checks existence before reading `ANALYSIS-NNNN`, and correctly falls through to reading its frontmatter when it *does* exist — including the real case of `status: draft` with `ANALYSIS-NNNN` present carrying `readiness_verdict: needs_revision` (a state the original fix's own finding didn't name but the actual edit handles correctly anyway, since the existence check is the right general fix, not a narrow patch for one sub-case).
- `[LD-1]`: confirmed the new milestone-deadlock branch is textually consistent with every other branch in Next Action Inference — none of them explicitly annotate their `resolved_state.gate` mapping either, so this one's silence on that point is consistent with the existing pattern, not a new gap.
- `[QE-1]`: confirmed `TC-FUNC-43`'s Given/When/Then directly exercises the fixed scenario and is correctly formatted (`integration | automated | P1`, proper Gherkin), and that it's correctly placed under `AC-04` in the M1 coverage table.

**Clean perspectives:** Lead Developer, Security, SRE, QE.

### Summary
**0 blocking, 0 non-blocking findings.**

### Next Step
`/approve-implementation PLAN-0001 M1`
