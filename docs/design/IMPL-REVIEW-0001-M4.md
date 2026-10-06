---
id: IMPL-REVIEW-0001-M4
type: impl-review
schema_version: 1
plan: PLAN-0001
milestone: M4
current_review_round: 1
approval:
  mode: repo_only
  approved_by: "Eric Pizon"
  approved_role: lead-developer
  approved_at: "2026-10-06"
  decision_summary: "M4 approved for merge — /promote-to-thread authored (AC8-AC11), /status's SPEC-NNNN form (AC12), zero blocking findings."
  external_event_id: null
  external_event_uri: null
policy_evaluation:
  evaluated: null
  policy_pack: null
  result: null
generation:
  command: /review-implementation
  command_version: "4"
  model: "claude-sonnet-5"
  generated_at: "2026-10-06"
  session_id: null
---

# Implementation Review — PLAN-0001 M4

## Task Reference — M4

*Derived from PLAN-0001 at review creation time. Reviewers use this to verify coverage without loading the full PLAN.*

| Task | File(s) | Description |
|---|---|---|
| T8 | `templates/commands/promote-to-thread.md` (new) | AC8 (reactive `coordination_repo` resolution), AC9 (Option B's idempotent six-step sequence, anchored by `pending_promotion`), AC10 (re-examine prior review rounds on substance), AC11 (`status: promoted`/`superseded_by`, never deleted). States inline: no coordination-repo write before `pending_promotion` is committed (D4/ADR-0001); any error, including coordination-repo-unreachable, sanitized first (D11, extended). |
| T9 | `templates/commands/status.md` | New `/status SPEC-NNNN` form (AC12): detects `status: promoted` and redirects to `superseded_by`; detects `pending_promotion` set without `status: promoted` and directs to resume `/promote-to-thread`. |
| T16 | `templates/commands/promote-to-thread.md` | OBS-0001 IT-4 — `promote_to_thread.outcome` (`completed`/`resumed_and_completed` at Step 7; `interrupted`, logged by the resuming invocation, at Step 2). |
| T19b | `templates/commands/promote-to-thread.md` | SEC-0001 SEC-3 — `coordination_repo`'s reactive-ask path (Step 1) validated against `^[\w.-]+/[\w.-]+$`, same as `/forge-init`'s Phase 2b. |
| T20 | `templates/commands/promote-to-thread.md` | DESIGN-0001 D11, extended scope — this command's own coordination-repo and participant-repo errors sanitized before display/commit (Steps 3, 4, 5). |

## Round 1 — 2026-10-06

### Lead Developer Findings

**[LD-1] non-blocking — two gaps in DESIGN-0001's own six-step mechanical sequence, found and resolved during authoring**

DESIGN-0001's Interface/API Changes section spells out `/promote-to-thread`'s sequence in exactly six numbered steps. Two things this command needs are real acceptance criteria but don't correspond to any of those six steps:

1. **AC10 (re-examine prior review rounds)** has no step at all in DESIGN's list. Resolved by inserting it as this command's own Step 6, between cross-repo scope discovery (Step 5) and finalization (Step 7) — the first point at which "is this finding still valid under the newly-discovered scope" is actually answerable, and necessarily before the SPEC is committed to `promoted`.
2. **Copying the promoted SPEC's own content into `THREAD-NNNN`'s directory** is also absent from DESIGN's six steps — "mints the `THREAD-NNNN` directory" is described, but not that the directory needs `SPEC.md`/`ANALYSIS.md`/`DESIGN.md` actually written into it to be useful (DESIGN's own rejected Option A, by contrast, *did* list "copy SPEC content" as one of its checkpoint steps). Resolved by folding it into this command's Step 3, immediately after the directory-creation/collision-check logic, since it shares that step's idempotency shape and failure mode.

Neither is a design error — DESIGN-0001's own `/create-design` process never claimed exhaustive step-by-step mechanical detail was in scope for every AC, and both gaps are resolved here with reasoning stated inline in the command text itself, not silently patched. Flagging per this engagement's standing practice of surfacing every implementation-time gap rather than letting a clean-looking diff hide where a judgment call was made.

**Resolution recommendation:** No further action — both are resolved inline, with the reasoning visible in `promote-to-thread.md` itself (Step 3 and Step 6's own opening paragraphs). Worth noting for `/create-design`'s own future quality bar: a "spell out the exact step sequence" Design Option, as DESIGN-0001's Option B description attempts, should be checked against the full AC list it's implementing, not just reviewed for internal consistency.

### Security Findings

*(none — SEC-3/T19b's validation and D11/T20's sanitization scope were both checked directly against the command text and are present at every coordination-repo and participant-repo touchpoint: Steps 1, 3, 4, 5)*

### SRE Findings

*(none — OBS-0001 IT-4's two instrumentation points match exactly (Step 2's `interrupted`, Step 7's `completed`/`resumed_and_completed`); DESIGN-0001 Round 2's own SRE re-check already confirmed the `pending_promotion` concurrent-write race is ordinary git concurrent-commit handling, not a new failure mode this implementation needed to solve itself)*

### QE Findings

*(none — AC8 through AC12 each have a corresponding step; TC-ADR-0001-01's specific interruption condition (Step 2 committed, before any coordination-repo write) is the exact ordering Step 2/Step 3 enforce; TC-EDGE-2's "unexpected SPEC status" case is handled explicitly in Step 0)*

**Clean perspectives:** Security, SRE, QE

### Summary
**Summary:** 0 blocking, 1 non-blocking findings.

### Next Step
`/approve-implementation PLAN-0001 M4`
