---
id: IMPL-REVIEW-0001-M3
type: impl-review
schema_version: 1
plan: PLAN-0001
milestone: M3
current_review_round: 2
latest_round:
  round: 2
  blocking: 0
  non_blocking: 0
  verdict: approved
approval:
  mode: repo_only
  approved_by: "Eric Pizon"
  approved_role: lead-developer
  approved_at: "2026-10-08"
  decision_summary: "M3 approved for merge after 2 review rounds — all Round 1 findings (SRE-1, LD-1, QE-1) verified resolved in Round 2 with zero new findings."
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

# Implementation Review — PLAN-0001 M3

## Task Reference — M3

| Task | File(s) | Description |
|---|---|---|
| T18 | `templates/commands/create-release-plan.md`, `templates/commands/approve-release-plan.md` | Extend the existing `approval:` block with `candidate_ref`/`release_plan_revision`; `/execute-release` checks both before executing, per DESIGN-0001 §4/Key Decision 6 |
| T19 | `templates/commands/execute-release.md` | Write `deployment_progress: {migrations_applied, deployed, tagged, updated_at}` incrementally after each existing numbered step, per DESIGN-0001 §4/Key Decision 7 |

## Round 1 — 2026-10-08

`git status` confirmed exactly 3 command template files modified, each attributable to T18/T19, plus `sdlc/plans/PLAN-0001-forgekit-operator.md`'s own `in_progress` status flip. No out-of-scope file found.

### Lead Developer Findings

- **[LD-1] non-blocking — the resume-check's Step 1/2 parenthetical introduces an ungrounded condition**
`execute-release.md`'s new resume-check says: "`migrations_applied: false` → start at Step 1 (or Step 3 directly if Step 1/2 are confirmed already done this session)." Nothing tracks whether Step 1 (confirmation) or Step 2 (checkout/pull `main`) happened — only `migrations_applied`/`deployed`/`tagged` are tracked, and by construction, `migrations_applied: false` means nothing in this run has succeeded yet. The parenthetical's "confirmed... this session" has no actual mechanism behind it and just adds confusing, unfounded optionality. Recommend removing it — Step 1/2 are cheap and idempotent; there's no real cost to always restarting from Step 1 when all three fields are `false`.

### SRE Findings

- **[SRE-1] blocking — the approval-binding check has no handling for a RELEASE-PLAN approved before `candidate_ref`/`release_plan_revision` existed**
A project that approved a RELEASE-PLAN before upgrading to this version of ForgeKit's templates (a realistic scenario — release approvals are rare, high-stakes events, not something re-run casually) would have `approval.candidate_ref` simply absent. The new check as written — "compare `approval.candidate_ref` to the current `HEAD`. If they differ, stop" — treats absent the same as mismatched, producing a confusing false-positive error ("approval was bound to a different commit (`null`/`undefined`)") that would block a perfectly legitimate, already-approved deployment. This is exactly the class of gap DESIGN-0001 §3.5 already established a pattern for elsewhere (missing structured field ≠ a parseable mismatch) — that pattern was never extended here.
Recommendation: if `approval.candidate_ref` is absent entirely, stop with a distinct message — "This RELEASE-PLAN was approved before candidate binding existed. Re-run `/approve-release-plan` to establish it before deploying." — rather than the mismatch message, which implies something changed when really nothing was ever bound.

**Clean perspectives:** Security.

### QE Findings

- **[QE-1] non-blocking — `TC-FUNC-12`/`TC-FUNC-13` only test the already-bound case, not the pre-existing-approval case `[SRE-1]` identifies**
Both test cases' `Given` clauses assume `approval.candidate_ref`/`approval.release_plan_revision` already exist. Once `[SRE-1]` is fixed, a new test case covering the absent-field path should be added so the fix is verified, not just asserted — same pattern as `[QE-1]` in M2's own review.

### Contradictions Requiring Arbitration

None.

### Summary
**1 blocking, 2 non-blocking findings.**

### Next Step
`/address-implementation-review PLAN-0001 M3`

### Address Summary — Round 1
Date: 2026-10-08

| Finding | Resolution |
|---|---|
| [SRE-1] | Fixed: `execute-release.md`'s approval-binding check now checks for `approval.candidate_ref`/`approval.release_plan_revision` being absent entirely *before* the mismatch comparison, stopping with a distinct message ("approved before candidate binding existed — re-run `/approve-release-plan`") instead of a false-positive mismatch. |
| [LD-1] | Fixed: removed the ungrounded "(or Step 3 directly if Step 1/2 are confirmed already done this session)" parenthetical from the resume check — nothing tracks Step 1/2 completion, and `migrations_applied: false` already unambiguously means restart from Step 1; replaced with a one-line explanation of why that's always safe (Steps 1/2 are cheap and idempotent). |
| [QE-1] | Fixed: added `TC-FUNC-44` (P1) to TEST-0001, covering the absent-`candidate_ref`/`release_plan_revision` path `[SRE-1]` fixes, distinct from TC-FUNC-12/13's mismatch-path coverage. Added to AC-14's coverage row. TEST-0001 bumped to v5 — same pattern as M1's TC-FUNC-43 and M2's TC-FUNC-10 additions. |

**Not addressed:** None.

## Round 2 — 2026-10-08

Scope: changed files only, per the Address Summary above — `templates/commands/execute-release.md` (SRE-1, LD-1) and `sdlc/test-plans/TEST-0001-forgekit-operator.md` (QE-1). `create-release-plan.md` and `approve-release-plan.md` were untouched by Round 1's address pass and have no plausible cascade from these fixes (the absent-field case only arises from a *pre*-v4 approval; `approve-release-plan.md` v4 already always writes both fields going forward) — not re-reviewed.

- **[SRE-1] verification:** the absent-field check (new line 34) is ordered before the mismatch comparisons (lines 35-36) and explicitly states "do not fall through" — confirmed the logic is sound: by the time the mismatch checks run, neither field can be absent, so a real mismatch and a never-bound approval can no longer be confused with each other.
- **[LD-1] verification:** the ungrounded parenthetical is gone; the resume check now states plainly why restarting from Step 1 is always correct and costless when `migrations_applied: false`.
- **[QE-1] verification:** `TC-FUNC-44` is present, mapped into AC-14's coverage row alongside TC-FUNC-12/13, and its Given/When/Then is scoped to the absent-field path specifically (distinct from TC-FUNC-12/13's stale-value path) — not a duplicate of existing coverage.

No new findings. All three Round 1 findings confirmed resolved against the actual file content.

**Clean perspectives:** Lead Developer, Security, SRE, QE.

### Summary
**0 blocking, 0 non-blocking findings.**

### Next Step
`/approve-implementation PLAN-0001 M3`
