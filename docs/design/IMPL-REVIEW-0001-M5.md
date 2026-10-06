---
id: IMPL-REVIEW-0001-M5
type: impl-review
schema_version: 1
plan: PLAN-0001
milestone: M5
current_review_round: 1
approval:
  mode: repo_only
  approved_by: "Eric Pizon"
  approved_role: lead-developer
  approved_at: "2026-10-06"
  decision_summary: "M5 approved for merge — /promote-to-environment authored (AC13-AC15), zero blocking findings."
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

# Implementation Review — PLAN-0001 M5

## Task Reference — M5

*Derived from PLAN-0001 at review creation time. Reviewers use this to verify coverage without loading the full PLAN.*

| Task | File(s) | Description |
|---|---|---|
| T10 | `templates/commands/promote-to-environment.md` (new) | AC13 (governance-value reasoning, never a hardcoded environment name), AC14 (`governance: ci` has no command path at all), AC15 (lightweight append-only record, reusing `approve-release-plan.md`'s `candidate_ref` shape, explicitly excluding Gating-criteria/Rollback/Validation-Plan). See ADR-0004. |

## Round 1 — 2026-10-06

### Lead Developer Findings

*(none — the command's own Step 1 resolves `governance` before anything else and branches only on that value; `TC-FUNC-30`'s requirement, an arbitrarily-named environment behaving correctly on `governance` alone, holds by construction since the literal name is never inspected after Step 1's lookup)*

### Security Findings

**[SEC-1] non-blocking — self-caught during authoring: `environments[].name` needed point-of-use validation this command is the one place that actually owes it**

SEC-0001's SEC-3 resolution (recorded during M3's implementation review) explicitly named `/promote-to-environment` as the future point-of-use validator for `environments[].name`, since that field has no writing command of its own and was accepted as residual risk specifically on the premise that *some* consumer would validate it before use. This command interpolates the environment name directly into a file path (`sdlc/promotions/{environment}.md`) — an unsanitized value could otherwise escape that directory. Added a validation step (reject anything outside `[a-z0-9-]`, refuse rather than silently sanitize, since this is a human-edited config value, not a derived slug) immediately after Step 1's governance lookup, before the name is used anywhere.

**Resolution recommendation:** Already fixed, inline, before this review round — flagging per this engagement's standing practice of surfacing every self-caught gap explicitly rather than letting it disappear into a clean-looking diff. `known_participants[].repo` remains unaddressed by any command yet (no command currently reads that field in a way that interpolates it anywhere sensitive) — not this task's scope, and still covered by the same general residual-risk acceptance for whichever future command becomes its actual point of use.

### SRE Findings

*(none — no OBS tasks in this milestone's scope; `sdlc/promotions/{environment}.md`'s lazy creation mirrors the already-reviewed `SYSTEM-ARCH.md` pattern from M3, no new failure mode introduced)*

### QE Findings

*(none — AC13/TC-FUNC-29/30, AC14/TC-FUNC-31, AC15/TC-FUNC-32/33, and TC-ADR-0004-01 each map directly onto a step: governance resolution (Step 1), the `ci`/`release` refusals (Step 1), the four-field row and append-only file (Step 3))*

**Clean perspectives:** Lead Developer, SRE, QE

### Summary
**Summary:** 0 blocking, 1 non-blocking findings.

### Next Step
`/approve-implementation PLAN-0001 M5`

### Addendum — found during `/execute-tests`, 2026-10-06

`TC-EDGE-3` (TEST-0001's own self-flagged open question: "missing or invalid `governance` value — confirm the relevant commands reject or error clearly rather than silently defaulting to an assumption... flagging here for `/review-test-plan` to assess whether it reveals a genuine spec gap") was never actually picked up by either of TEST-0001's two review rounds — the question was raised and then never answered before TEST-0001's own approval. Running it against the real implementation during M5's test execution surfaced that Step 1's governance branch had no explicit case for anything other than `ci`/`promotion`/`release` — a missing or malformed value would have fallen through undefined. Fixed in `promote-to-environment.md` directly: an explicit fourth branch refuses and directs to fix `project.yml`, never silently defaulting. Not re-opening this review round for a one-line addition; documented here for the audit trail, consistent with how this engagement has handled every other gap caught after its originating review already closed.
