---
id: SPEC-0002
type: spec
schema_version: 1
title: "Configurable Post-Merge Test Gate"
status: approved
created: 2026-10-09
updated: 2026-10-17
version: 3
changelog:
  - "v3 — approved after 3 review rounds"
product_validated: true
validated_by: "Eric Pizon — TPO"
validated_date: 2026-10-17
owner: "Eric Pizon"
priority: Medium
stakeholders:
  - Eric Pizon (TPO)
links:
  - relation: derived_from
    target: INV-0001
generation:
  command: /create-spec
  command_version: "5"
  model: "claude-sonnet-5"
  generated_at: "2026-10-09"
  session_id: null
approval:
  mode: repo_only
  approved_by: "Eric Pizon"
  approved_role: tpo
  approved_at: "2026-10-17"
  decision_summary: "SPEC-0002 (Configurable Post-Merge Test Gate) approved for design phase after 3 review rounds and product validation sign-off."
  external_event_id: null
  external_event_uri: null
policy_evaluation:
  evaluated: null
  policy_pack: null
  result: null
---

# SPEC-0002 — Configurable Post-Merge Test Gate

## Scope intent and Non-Goals (Pre-flight answer)

Self-answered from INV-0001's own recommendation rather than a separate round-trip, since the Investigation already reasoned through both — stated explicitly here so it can be corrected before drafting proceeds, not silently assumed:

**In scope:**
1. A new `project.yml` field (`pipeline.test_gate: pre_merge | post_merge`, default `pre_merge`).
2. A new `/forge-init` workflow question configuring it.
3. `/approve-implementation` offering to merge the milestone branch to `main` directly under `post_merge`, instead of pointing to `/execute-tests`.
4. `/execute-tests` validating the merged/deployed state instead of the feature branch, under `post_merge`.
5. A defined response when a P0 failure is found post-merge (routing into the existing BUG/incident model).
6. `/create-pr`/`/close-milestone` becoming unreachable (by design) under `post_merge`, with the milestone's own closure happening at the merge step instead.

**Out of scope (Non-Goals):**
1. Any change to default behavior for a project that doesn't opt in — `pre_merge` stays byte-identical to today.
2. Building an actual QA-environment deployment mechanism — ForgeKit still provisions nothing; `post_merge` assumes the adopting project's own CI/CD already deploys from `main`.
3. Any change to the RELEASE-PLAN/production-release flow (Stage 9-10) — scoped entirely to the per-milestone implementation phase (Stage 6-7).
4. A third, hybrid test-gate mode — exactly two modes, no partial variant.
5. Resolving how multi-repo coordination threads interact with this setting (named as an Open Question, not decided here).

## Problem Statement

In a trunk-based enterprise workflow, QE validates milestone code in a shared QA environment that continuously deploys from `main` — not from an unmerged feature branch. ForgeKit's pipeline currently supports only one sequencing: `/execute-tests` must pass against the feature branch before any merge to `main` can happen at all (`/approve-implementation` commits to the feature branch; `/create-pr` is the only thing that ever merges, and only after `tests_passed`). A team whose actual QA process can only ever observe what's deployed from `main` has no way to get code there for QE to test it, without either bypassing ForgeKit's own sequencing entirely or accepting that `/execute-tests` can never reach their real QA environment.

## Goals

- **G1 — Configurable test-gate mode.** A project can configure, at `/forge-init` time, whether milestone code must pass `/execute-tests` before merging (`pre_merge`, today's only behavior) or merges immediately upon `/approve-implementation`, with testing happening afterward against the merged/deployed state (`post_merge`). → AC-01, AC-02, AC-04, AC-05
- **G2 — No behavior change for existing/default projects.** Every project that doesn't explicitly opt into `post_merge` behaves identically to today. → AC-04
- **G3 — Defined post-merge failure response.** A P0 test failure discovered after merge under `post_merge` has an explicit, defined response — not an undefined gap. → AC-03, AC-08
- **G4 — Clean milestone closure under `post_merge`.** The milestone's closure happens at the merge step, with no orphaned, never-reachable `/create-pr`/`/close-milestone` step hanging off the end, and no break in the next milestone's dependency gating. → AC-06, AC-07

## Non-Goals

See "Scope intent and Non-Goals" above — restated here per the standard Output Contract location:
1. Changing default (`pre_merge`) behavior for any project.
2. Building a QA-environment deployment mechanism.
3. Changing the RELEASE-PLAN/production-release flow.
4. A third/hybrid test-gate mode.
5. Resolving multi-repo/coordination-repo interaction (Open Question, not a Non-Goal exactly — named below).

## User Stories

- As a Lead Developer at an enterprise with a shared QA environment, I want milestone code to merge to `main` as soon as it's reviewed and approved, so that our existing CI/QA pipeline — which only ever tests what's actually on `main` — can pick it up immediately, instead of waiting on a ForgeKit-specific test step that can never reach our real QA environment.
- As a QE at that same enterprise, I want to validate against the real, deployed QA environment after merge, so that my results reflect what users will actually be served, not an un-deployed feature branch no one else can see.
- As a solo developer or small team, I want the default behavior to stay exactly as it is today, so that this change costs me nothing if I never opt in.
- As a Developer whose milestone just merged under `post_merge` and QE found a P0 defect afterward, I want a clear, defined next step, so that I'm not left choosing between revert/hotfix/escalate on my own judgment with no guidance from the pipeline.

## Acceptance Criteria

- **AC-01:** Given a project with `pipeline.test_gate: post_merge`, when a milestone reaches `approved` via `/approve-implementation`, the Handoff offers to merge the milestone branch to `main` directly, without requiring `/execute-tests` first. If the human declines the offer, the milestone remains at `approved` — the same recoverable state as before the offer, re-offered the next time `/approve-implementation`'s Handoff (or an equivalent re-invocation) is reached, exactly like any other declined confirmation elsewhere in the pipeline.
- **AC-02:** Given `pipeline.test_gate: post_merge` and a milestone branch already merged, when `/execute-tests` subsequently runs for that milestone, every result it reports is tied to the merged commit now on `main` — via whatever specific target mechanism the project configures (the exact mechanism — testing `main`'s own state directly vs. a deployed QA environment sourced from it — is a DESIGN-level choice, OQ-3) — and never a result that could be satisfied merely by the already-deleted feature branch's prior state.
- **AC-03:** Given `pipeline.test_gate: post_merge`, when `/execute-tests` finds a P0 failure after the milestone has already merged, the failure routes to `/create-bug` (or `/escalate-to-incident` if the affected environment is shared/blocking) — never to the `test_failed` milestone status, which assumes an unmerged, revertible-without-consequence feature branch that no longer exists at this point.
- **AC-04:** Given a project with no `pipeline.test_gate` configured, or explicitly `pre_merge`, every command in the implementation/testing/PR sequence behaves identically to its behavior before this feature existed.
- **AC-05:** Given `/forge-init` running (first-run or re-configuring), the human is asked which `test_gate` mode to use, with `pre_merge` as the default if they have no strong preference. The question itself states the `post_merge` trade-off plainly (code reaches `main`/QA before `/execute-tests` confirms it) — not just the two mode names — so the choice is informed, not just a label pick.
- **AC-06:** Given `pipeline.test_gate: post_merge`, once a milestone merges via `/approve-implementation`'s offered merge, `/create-pr` and `/close-milestone` are never invoked for that milestone — `**Implementation status:**` reaches `pr_merged` directly from the merge step itself.
- **AC-07:** Given `pipeline.test_gate: post_merge`, the next milestone's `depends_on_milestones` gate (in `/implement-plan`'s Pre-flight Check) still correctly recognizes the prior milestone as unlocking downstream work once merged — the dependency-gating mechanism must not implicitly assume the `pre_merge`-only status sequence.
- **AC-08:** Given `pipeline.test_gate: post_merge`, the milestone's `**Implementation status:**` reaching the terminal `pr_merged` at merge time (AC-06) does not prevent tracking what happens next — a separate, explicit field (exact shape resolved at DESIGN) distinguishes post-merge testing `pending`, `passed`, and `failed` for that milestone, independently of the already-terminal implementation status. `pr_merged` alone must never be the only signal available once AC-02/AC-03's own described behavior needs somewhere to record its result.

## Non-Functional Requirements

- **Performance:** N/A — a workflow-sequencing change, not a new latency-sensitive runtime path.
- **Reliability:** the offered merge must remain a human-confirmed action under every circumstance — this feature must never make merging to `main` silent or automatic, consistent with every other commit/merge gate in the pipeline.
- **Security:** no new credential or auth surface — reuses existing git/merge mechanics and the existing BUG/incident artifacts; no new external system integration.
- **Forward compatibility:** must not assume `/execute-tests`'s existing "CI Path" (triggering `stack.ci_trigger_command` "for the current branch") is reusable as-is for post-merge validation — a `post_merge`-mode test run needs its own clearly-stated target (main, or a project-configured QA trigger). Exact mechanism deferred to DESIGN (Open Question below).
- **Operability:** no new infrastructure dependency introduced by ForgeKit itself — `post_merge` assumes the adopting project's own CI/CD already deploys from `main`; ForgeKit provisions nothing.

## Constraints & Assumptions

- ForgeKit itself (this repo) has no stack or deploy target of its own — any dogfood test of this feature exercises the config/workflow-branching logic only, never an actual QA deployment.
- Assumes the adopting project's QA environment deploys from `main` (or an equivalent shared integration branch) continuously or on a schedule, outside ForgeKit's control.
- Single-repo scope for this SPEC — multi-repo/coordination-repo interaction is an Open Question, not resolved here.
- **Accepted residual risk of `post_merge` mode, stated explicitly rather than left implicit:** for some window between merge and `/execute-tests` reporting a result, `main` (and whatever QA environment deploys from it) can be carrying code that hasn't yet been confirmed to pass. This is not a defect this SPEC needs to close — it is the actual trade-off a project accepts by choosing `post_merge` over `pre_merge` in exchange for matching their real QA process, the same way `pre_merge` accepts the cost of `/execute-tests` being unable to validate against a real deployed environment. A project should understand this trade-off before configuring `post_merge` — the `/forge-init` question (AC-05) should say so plainly, not just offer the two mode names.

## Security Considerations

**`security_review_required`: no.** No new credential or auth surface; no new external system integration — this feature reuses existing git/merge mechanics and the existing BUG/incident artifacts exactly as they are today.

## Observability Considerations

- **Metrics:** N/A — ForgeKit has no metrics library (markdown-instruction-based, per ARCH.md's own "What this repo actually is"); not applicable, consistent with OBS-0001's framing for this entire repo.
- **Traces:** N/A, same reasoning.
- **Logs:** the milestone's own structured fields (`**Implementation status:**`, and the project's configured `pipeline.test_gate`) are themselves the audit trail of which mode a given milestone ran under. DESIGN should confirm whether the milestone record should explicitly note which mode applied to it specifically, for later legibility if a project changes this setting between milestones. **What must never be logged:** N/A — this feature introduces no new field that could ever carry a credential, token, or PII; every field it adds (`pipeline.test_gate`, the merge confirmation) is a plain workflow-mode marker with nothing sensitive to redact.

## Open Questions

- **OQ-1:** Does `pipeline.test_gate` apply uniformly across an entire project, or could/should it be set per-milestone? *Resolved at: DESIGN.*
- **OQ-2:** For a `repo_role: coordination` multi-repo thread, does each participant repo configure `test_gate` independently, or does the thread-wide policy need to account for mixed modes across participants? *Resolved at: DESIGN.*
- **OQ-3:** Exact mechanism by which `post_merge`-mode `/execute-tests` reaches the deployed QA state — a new `stack.qa_trigger_command`-style field, a repurposed `stack.ci_trigger_command` with different semantics, or something else. *Resolved at: DESIGN.*
- **OQ-4:** Should the offered merge under `post_merge` go directly to a fast-forward merge (mirroring `create-pr.md`'s existing local-only path), or should it still support opening a real PR for async review before merging — just without requiring tests to pass first? *Resolved at: DESIGN.*
- **OQ-5:** What happens when a project switches `pipeline.test_gate` mid-project, with milestones already in flight under the old mode? Does the new setting apply only to milestones not yet started, or can an in-progress milestone's mode change mid-stream? *Resolved at: DESIGN.*
- **OQ-6:** Should a post-merge test *failure* (AC-08's `failed` outcome) re-block a dependent milestone that `AC-07`'s dependency gate already recognized as unlocked once the prior milestone merged — or is that a deliberate policy trade-off a project can choose not to enforce? *Resolved at: DESIGN.*

## Revision Summary — Round 1

**Date:** 2026-10-09

### Addressed
- `[LD-1]`: AC-02 reworded from "validates `main` (or a deployed environment)" to an invariant — every result ties to the merged commit, with the specific target mechanism left to DESIGN (OQ-3), removing the ambiguous bundling of two literal targets into one AC.
- `[LD-2]`: added an explicit "Accepted residual risk of `post_merge` mode" bullet under Constraints & Assumptions, naming the window where `main`/QA can carry unconfirmed code as a stated trade-off, not an implicit cost — and requiring `/forge-init`'s question (AC-05) to state it plainly.
- `[QE-1]`: AC-01 extended with the declined-merge case — milestone stays `approved`, re-offered next time, same as any other declined confirmation in the pipeline.
- `[QE-2]`: added `OQ-5`, naming the mid-project mode-switch question explicitly rather than leaving it unaddressed.
- `[QE-3]`: added `AC-08`, requiring a separate field (shape resolved at DESIGN) to track post-merge test outcome independently of the milestone's own terminal `pr_merged` status — closing the gap where AC-02/AC-03's described behavior had nowhere to record its result. Mapped to G3.

### Not Addressed
- `[SC-1]` — **no action needed**: this is the Scope Coherence Check's own confirmation that no split is warranted, not a flagged gap requiring a change.

## Revision Summary — Round 2

**Date:** 2026-10-09

### Addressed
- `[LD-1]`: `AC-05` itself now requires the `post_merge` trade-off be stated plainly as part of the question — no longer an unenforced Constraints-only aspiration one section away.
- `[QE-1]`: added `OQ-6`, naming whether a post-merge test failure should re-block a dependent milestone `AC-07` already unlocked, deferred to DESIGN as a policy trade-off rather than mandated here.

### Not Addressed
*(none — both Round 2 findings addressed above.)*

## Product Validation

**Validated by:** Eric Pizon — TPO
**Date:** 2026-10-17
**Model:** claude-sonnet-5
**Status:** validated

### Acceptance Criteria — Plain Language Review

**AC-01 (original):** Given a project with `pipeline.test_gate: post_merge`, when a milestone reaches `approved` via `/approve-implementation`, the Handoff offers to merge the milestone branch to `main` directly, without requiring `/execute-tests` first. If the human declines the offer, the milestone remains at `approved` — the same recoverable state as before the offer, re-offered the next time `/approve-implementation`'s Handoff (or an equivalent re-invocation) is reached, exactly like any other declined confirmation elsewhere in the pipeline.
**In plain terms:** If you've turned on this mode, once your code is reviewed and approved, the system asks if you want to merge it right away — you don't have to wait for tests to pass first. If you say no, nothing happens and it'll ask again next time.

**AC-02 (original):** Given `pipeline.test_gate: post_merge` and a milestone branch already merged, when `/execute-tests` subsequently runs for that milestone, every result it reports is tied to the merged commit now on `main` — via whatever specific target mechanism the project configures (the exact mechanism is a DESIGN-level choice, OQ-3) — and never a result that could be satisfied merely by the already-deleted feature branch's prior state.
**In plain terms:** After your code merges in this mode, running tests checks the real, merged version of your code — never leftover results from the branch that no longer exists.

**AC-03 (original):** Given `pipeline.test_gate: post_merge`, when `/execute-tests` finds a P0 failure after the milestone has already merged, the failure routes to `/create-bug` (or `/escalate-to-incident` if the affected environment is shared/blocking) — never to the `test_failed` milestone status, which assumes an unmerged, revertible-without-consequence feature branch that no longer exists at this point.
**In plain terms:** If a serious test failure turns up after the merge already happened, the system files it as a bug (or an incident if it's blocking others) instead of pretending you can still just fix-and-retest like nothing was ever merged.

**AC-04 (original):** Given a project with no `pipeline.test_gate` configured, or explicitly `pre_merge`, every command in the implementation/testing/PR sequence behaves identically to its behavior before this feature existed.
**In plain terms:** If you never turn this on, nothing about your day-to-day workflow changes at all.

**AC-05 (original):** Given `/forge-init` running (first-run or re-configuring), the human is asked which `test_gate` mode to use, with `pre_merge` as the default if they have no strong preference. The question itself states the `post_merge` trade-off plainly — not just the two mode names — so the choice is informed, not just a label pick.
**In plain terms:** When you set up a new project, you'll be asked to pick a mode, and you'll be told honestly what you're trading off before you have to decide — if you're unsure, the safer option is picked for you automatically.

**AC-06 (original):** Given `pipeline.test_gate: post_merge`, once a milestone merges via `/approve-implementation`'s offered merge, `/create-pr` and `/close-milestone` are never invoked for that milestone — `**Implementation status:**` reaches `pr_merged` directly from the merge step itself.
**In plain terms:** In this mode, once your code merges, you don't go through a separate "open a PR" step — the system just marks the work as done right then.

**AC-07 (original):** Given `pipeline.test_gate: post_merge`, the next milestone's `depends_on_milestones` gate still correctly recognizes the prior milestone as unlocking downstream work once merged — the dependency-gating mechanism must not implicitly assume the `pre_merge`-only status sequence.
**In plain terms:** Turning this mode on doesn't break the rule that later work has to wait for earlier work to finish — that still works correctly.

**AC-08 (original):** Given `pipeline.test_gate: post_merge`, the milestone's `**Implementation status:**` reaching the terminal `pr_merged` at merge time does not prevent tracking what happens next — a separate, explicit field (exact shape resolved at DESIGN) distinguishes post-merge testing `pending`, `passed`, and `failed` for that milestone, independently of the already-terminal implementation status.
**In plain terms:** Even after the system says your work is "done" (merged), there's still a separate, visible way to tell whether the after-the-fact testing has happened yet, and whether it passed.

### Structural Questions

1. **Problem statement describes pain, not solution?**
   Confirmed — describes pain, no design option foreclosed.

2. **Non-Goals specific enough to use as decision rules?**
   Confirmed — all five specific enough to quote directly.

3. **Uncovered user scenario?**
   None identified.

### AC-Specific Questions

**AC-01:** Is "If you've turned on this mode... it'll ask again next time" the complete definition of done, or is there an implied condition not stated?
Complete, no gap.

**AC-02:** Is "running tests checks the real, merged version... never leftover results" the complete definition of done, or is there an implied condition not stated?
Complete, no gap.

**AC-03:** Is "the system files it as a bug... instead of pretending nothing was merged" the complete definition of done, or is there an implied condition not stated?
Complete, no gap.

**AC-04 through AC-08:** Is each translation the complete definition of done, or is there an implied condition not stated?
All five confirmed complete, no gaps.

### Gaps Found

None.

### Sign-off

> Eric Pizon confirms: the plain-language translations are accurate and the questions above are answered. This SPEC is ready for design.
