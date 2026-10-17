---
id: DESIGN-0002
type: design
schema_version: 1
title: "Configurable Post-Merge Test Gate — Technical Design"
status: approved
version: 2
updated: 2026-10-17
current_review_round: 2
latest_round:
  round: 2
  blocking: 0
  non_blocking: 0
  verdict: ready
target_spec_version: 3
changelog:
  - "v2 — approved after 2 review rounds"
generation:
  command: /create-design
  command_version: "3"
  model: "claude-sonnet-5"
  generated_at: "2026-10-17"
  session_id: null
approval:
  mode: repo_only
  approved_by: "Eric Pizon"
  approved_role: architect
  approved_at: "2026-10-17"
  decision_summary: "DESIGN-0002 (Configurable Post-Merge Test Gate) approved for Stage 4 after 2 review rounds — all blocking findings resolved."
  external_event_id: null
  external_event_uri: null
policy_evaluation:
  evaluated: null
  policy_pack: null
  result: null
---

# DESIGN-0002 — Configurable Post-Merge Test Gate

## 1. Design Options

### Option A: Milestone-entry field, consistent with the `implementation_execution` precedent

**Approach:** `pipeline.test_gate: pre_merge | post_merge` in `project.yml`; a `post_merge_test: {status, updated_at}` block directly on the milestone's own PLAN entry, mirroring `implementation_execution`'s own `{state, ...timestamps}` shape exactly, written by `/execute-tests`; conditional branching added directly into `approve-implementation.md`/`execute-tests.md`'s existing text.

**Existing code integration:** extends the same files PLAN-0001-style work already touches; mirrors DESIGN-0001's own Key Decision 4 exactly — `implementation_execution` lives on the milestone entry, not a separate artifact, for the structurally identical reason (state meaningless outside its one milestone's context).

**Pros:**
- Directly consistent with an already-established, directly-analogous precedent.
- No new artifact type or propagation ceremony.
- Minimal schema surface; keeps BUG/incident as the one place broader judgment happens.

**Cons:**
- Adds a conditionally-relevant field to PLAN's schema (same minor cost `implementation_execution` already accepted).
- `/execute-tests`/`/approve-implementation` each gain some new conditional branching rather than delegating cleanly elsewhere.

**Risk:** Low.

### Option B: Dedicated sub-artifact, like `OPRUN-NNNN`'s own precedent for run-tracking

**Approach:** a new artifact type (`POSTMERGE-NNNN`) written at merge time, cross-referenced from the milestone's PLAN entry by ID rather than an inline field.

**Existing code integration:** a new `sdlc/post-merge-checks/` directory — new README, `forge-init.md` provisioning, the full propagation ceremony.

**Pros:** keeps PLAN-NNNN's schema completely untouched.

**Cons:**
- Directly contradicts Key Decision 4's own rationale without new justification — structurally the same problem, already solved.
- Repeats the exact new-artifact-type propagation gap `BUG-0005` already found going wrong once this session (`forge-init.md` never provisioning `sdlc/operator-runs/`).
- Unnecessary indirection for state that's 1:1 with a single milestone and never queried independently of it.

**Risk:** Medium — specifically the propagation-ceremony risk `BUG-0005` already demonstrated.

## 2. Recommended Option

**Recommended: Option A.** Directly consistent with Key Decision 4's precedent for the structurally identical problem; avoids repeating `BUG-0005`'s own gap class; satisfies `AC-02`/`AC-03` (naming `/execute-tests` as the command) and `AC-08` ("a separate, explicit field") most directly. Trade-off accepted: some new conditional branching inside existing command files rather than delegating to a cleanly separate artifact — acceptable since the behavior is a simple three-value status marker.

Confirmed by the user before this full spec was written.

## 3. Architecture Overview

### 3.1 System boundaries

No new request surface or auth model — ForgeKit has none to begin with (per ARCH.md's "What this repo actually is"). This feature stays entirely within the existing "AI agent reads and follows markdown command text" execution model; no new security boundary is crossed.

### 3.2 Resolved ambiguities from ANALYSIS-0002 / SPEC-0002's Open Questions

- **OQ-1 (uniform vs. per-milestone):** `pipeline.test_gate` is project-level only, matching every other `project.yml` workflow setting (`artifact_mode`, `pipeline.security_review`, `pipeline.evidence_mode`). No established need for per-milestone granularity; the milestone-entry field design (§3.4) leaves room for a future per-milestone override without restructuring, if one is ever justified.
- **OQ-2 (multi-repo):** per-repo setting, same as `stack.*` and every other per-repo config within a thread (PIPELINE.md's own "Per-repo commands still run per-repo"). No coordination-repo-level unification needed.
- **OQ-3 (QA-trigger mechanism):** a new optional `stack.qa_trigger_command` field, same shape as the existing `stack.ci_trigger_command`. `stack.ci_trigger_command`'s own meaning is unchanged (still branch-scoped, used under `pre_merge`) — the two fields are never conflated.
- **OQ-4 (merge mechanism) — revised during this same Design phase, not carried forward unresolved (see §3.3 below for why the obvious answer didn't survive contact with `AC-06`).**
- **OQ-5 (mid-project switch):** `pipeline.test_gate` is read live at point-of-use (same as `stack.lint`/`stack.test.unit` already are). The one exception: once a specific milestone has merged under `post_merge` (evidenced by its own `post_merge_test` block being present), `/execute-tests` follows the post-merge path for *that milestone* regardless of what the project's *current* setting now says — a milestone's own already-recorded state is authoritative for itself, never overridden by a later global config change (same "artifacts are authoritative" principle DESIGN-0001 §3.4 already established for Operator).
- **OQ-6 (failure re-blocking dependents):** no automatic re-block. `/implement-plan`'s `depends_on_milestones` gate checks only `pr_merged`, exactly as it does today — never `post_merge_test.status`. A post-merge failure's broader impact is a human judgment call exercised through the BUG/incident record (`AC-03`), not an automatic pipeline gate.

### 3.3 Key Decision — how the merge actually happens under `post_merge` (resolves OQ-4, revised)

The straightforward reading of OQ-4 ("reuse `create-pr.md`'s existing solo/team choice") breaks on `AC-06`'s literal requirement: "`/create-pr` and `/close-milestone` are never invoked... `pr_merged` directly from the merge step itself." A real PR normally sits at `pr_open` awaiting async human merge — which would require `/close-milestone` to eventually record it, contradicting `AC-06` outright for the team path.

**Decision:** `/approve-implementation`'s offered merge, under `post_merge`, handles both sub-paths itself, in the same step, every time:
- **Solo:** direct fast-forward merge to `main` (identical mechanics to `create-pr.md`'s existing local-only path).
- **Team:** opens a real PR (`gh pr create`) *and* attempts to immediately merge it (`gh pr merge`) in the same step — since the review and approval a PR would normally gate already happened inside `/approve-implementation` itself (its own Step 7 three-question gate, full diff review). The PR still exists in git history for branch-protection compliance and audit trail; it is just never left open awaiting a second round of async review that already happened, *when the merge succeeds*.

**Merge verification, never assumed (resolves Round 1 `[LD-1]`).** `gh pr merge`'s own exit code is not sufficient — confirm the PR's actual merged state (e.g. `gh pr view --json state`) before writing anything to `**Implementation status:**` or `post_merge_test`, the same "verify the outcome from resulting artifacts, never assume success from invocation alone" discipline DESIGN-0001 §15 already established for Operator, applied here to a plain markdown-instruction sequence:
- **Confirmed merged:** proceed — `**Implementation status:** pr_merged`, `post_merge_test: {status: pending, updated_at: <now>}` (§3.4).
- **Not merged** (a required status check or additional approval blocked the immediate merge — realistic for exactly the branch-protected-enterprise persona this mode targets): the PR is left open, `**Implementation status:**` stays at `approved` (never prematurely advanced), and the Handoff tells the human precisely what blocked it — resolve the requirement, then re-run `/approve-implementation`'s merge step, which will find the PR already open and retry the merge rather than creating a second one.

Either way, once the merge is confirmed, the milestone reaches `pr_merged` directly within this one step — `create-pr.md`/`close-milestone.md` are never invoked for it, genuinely satisfying `AC-06` for both sub-paths, not just the solo one. See **ADR-0004** for the full alternatives considered and rationale — this decision is durable and will be asked again if post-merge-style modes are ever added elsewhere.

### 3.4 Data model

**`project.yml` additions:**
```yaml
pipeline:
  test_gate: pre_merge | post_merge   # default: pre_merge; absent or malformed → treated as pre_merge (fail-closed, ADR-0003's own precedent)

stack:
  qa_trigger_command: "<command>" | "none"   # only meaningful when pipeline.test_gate: post_merge; default "none"
```

**PLAN-NNNN milestone entry — new field**, written alongside the existing `**Implementation status:**` line, present only once a milestone has actually merged under `post_merge` — absent entirely under `pre_merge`, exactly like `implementation_execution`'s own "absent means not applicable yet" convention. **Shaped to directly mirror `implementation_execution`'s own `{state, ...timestamps}` block (resolves Round 1 `[DBA-1]` — the original bare-enum shape had no timestamp, inconsistent with the exact precedent it was justified by):**
```yaml
post_merge_test:
  status: pending | passed | failed
  updated_at: <timestamp>
```
- Set to `{status: pending, updated_at: <now>}` by `/approve-implementation`'s merge step, the same moment `**Implementation status:**` is set to `pr_merged` (§3.3).
- Set to `{status: passed | failed, updated_at: <now>}` by `/execute-tests`, once it actually runs against the merged/deployed state. **Only a P0 failure sets `status: failed` (resolves Round 1 `[SRE-1]`) — exactly the same P0-only trigger `test_failed` already uses under `pre_merge`; a P1-or-lower failure is recorded in the test execution table as today, but does not flip `post_merge_test.status` away from `passed`.**
- **Once written, `status`/`updated_at` are never overwritten by a later run for the same milestone** — a `failed` result is a permanent historical record (consistent with "the git log is the audit trail"); a fix for a post-merge regression lands as a *new* milestone or hotfix (via the existing `/release-hotfix` path), tracked on its own entry, never by mutating the original failed milestone's record.

### 3.5 Interface / command changes

- **`forge-init.md`:** new workflow question (per `AC-05`) — asks which `test_gate` mode, states the `post_merge` trade-off plainly, defaults to `pre_merge`.
- **`approve-implementation.md`:** Handoff branches on `pipeline.test_gate`. Under `post_merge`: offers the merge directly (§3.3's mechanics), writes `post_merge_test: {status: pending, updated_at: <now>}`, sets `**Implementation status:** pr_merged` directly — **but only after confirming the merge itself actually completed (resolves Round 1 `[LD-1]` — see §3.3's revised mechanics below)** — skips straight to telling the user to run `/execute-tests` next, never mentioning `/create-pr`.
- **`execute-tests.md`:** Pre-flight gains a check: if the target milestone already has `post_merge_test` set (merged under `post_merge`), skip the branch-scoped Pre-flight entirely and validate the merged state instead — via `stack.qa_trigger_command` if configured, falling back to running `stack.test.unit`/`stack.test.e2e` directly against `main`'s current checked-out state if not. Writes `post_merge_test.status`/`updated_at` per the P0-only rule above. A P0 failure routes to `/create-bug` (or `/escalate-to-incident` if the environment is shared/blocking) instead of setting `test_failed` — `test_failed` is structurally meaningless here, since there is no unmerged branch left to fix and re-test.
- **`create-pr.md`/`close-milestone.md`:** unchanged — genuinely never reached for a milestone merged under `post_merge` (§3.3).
- **`implement-plan.md`:** Pre-flight Check #4 (`depends_on_milestones`) unchanged — already checks only `pr_merged`, which is sufficient per OQ-6's resolution; no edit needed, confirmed explicitly rather than assumed.

### 3.6 Failure modes

- **`stack.qa_trigger_command` itself fails to run** (network/auth, not a test result): treated as an infrastructure problem, surfaced directly to the user — never silently recorded as `post_merge_test.status: failed`, which must mean "tests ran and found a real problem," not "the trigger mechanism was broken."
- **`gh pr merge` fails after `gh pr create` succeeded** (resolves Round 1 `[LD-1]`) — see §3.3's revised mechanics: the PR is left open, `**Implementation status:**` stays at `approved` (never prematurely set to `pr_merged`), and the human is told exactly what blocked the merge (a required status check, a required additional approval) so they can resolve it and re-run.
- **Malformed or absent `pipeline.test_gate`:** defaults to `pre_merge` — fail-closed to the already-proven, safer behavior, consistent with ADR-0003's own established convention for this exact class of decision.
- **Mid-project mode switch:** see OQ-5's resolution above (§3.2) — a milestone's own already-recorded state wins over whatever the project's current setting says.

## 4. Component / Data Model Changes

See §3.4 above — the complete schema is stated there; not repeated here.

## 5. Interface / API Changes

No network-facing API exists in this repo (per ARCH.md). See §3.5 for the complete command-surface changes.

## 6. Key Decisions

1. **Milestone-entry field over a new artifact type (Option A).** → resolves the Phase 2 options analysis; no dedicated ADR — directly consistent with DESIGN-0001's already-accepted Key Decision 4 precedent, not a new cross-cutting choice.
2. **`post_merge`'s merge mechanism handles both solo and team paths itself, verifying the merge actually completed before advancing status — never left open for async review when it succeeds, left genuinely open only if branch protection blocks it.** → **ADR-0004**.
3. **`pipeline.test_gate` is project-level, read live at point-of-use, with per-milestone authority once a specific milestone has already merged under a given mode.** → resolves OQ-1/OQ-5; no dedicated ADR — directly follows the existing precedent every other `project.yml` pipeline toggle already sets.
4. **A post-merge test failure never automatically re-blocks dependent milestones.** → resolves OQ-6; no dedicated ADR — a direct, narrow application of "the BUG/incident model is where human judgment calls about broader impact happen," not a new principle.

## 7. Alternatives Considered

Covered in full in §1 (Design Options, carried into this DESIGN) and in **ADR-0004** (the merge-mechanism alternatives specifically: always-local-only, real-PR-with-async-review, and the chosen real-PR-with-immediate-merge).

## 8. Impact on Existing Systems

**Commands requiring modification:** `forge-init.md`, `approve-implementation.md`, `execute-tests.md`, `project.yml.template`.

**Commands requiring no modification:** `create-pr.md`, `close-milestone.md` (genuinely never reached under `post_merge` — confirmed, not assumed), `implement-plan.md` (its dependency gate already checks only `pr_merged`, confirmed explicitly in §3.5).

**ARCH.md updates required at `/approve-design`** (not applied now, per existing convention): note `pipeline.test_gate` joins the small set of existing `pipeline.*` behavioral toggles (`security_review`, `evidence_mode`) already documented — purely additive, no existing ARCH.md section superseded.

## 9. Observability Implementation

No metrics library exists in this repo (per ARCH.md). `post_merge_test` and `pipeline.test_gate` are themselves the complete audit trail of which mode a milestone ran under and what its post-merge outcome was — readable directly from PLAN-NNNN's own frontmatter, never duplicated elsewhere. Nothing here must never be logged — no field this feature adds can ever carry a credential, token, or PII (same conclusion SPEC-0002's own Observability Considerations already reached).

## Design Review — Round 1

**Date:** 2026-10-17
**Status:** Changes requested — 2 blocking, 1 non-blocking

### Options Analysis Findings

Clean. Both options are genuinely distinct architectural approaches; the recommendation cites a specific, directly-applicable precedent (DESIGN-0001 Key Decision 4) and a specific, already-observed failure class (`BUG-0005`), not generic "simpler/cleaner" reasoning. No concrete case found that Option B would better satisfy any SPEC AC or ARCH constraint. No missing option found that would change the recommendation — an alternative considered and rejected during this review itself (inferring post-merge status from BUG-NNNN presence rather than an explicit field) would actually reintroduce the exact `pending`-vs-`passed` ambiguity `AC-08` exists to close, confirming rather than undermining the chosen approach.

### DBA Findings

**[DBA-1] blocking — `post_merge_test_status` has no companion timestamp, inconsistent with the exact precedent this design claims to mirror.** §3.1 (and Key Decision 1) explicitly justifies this field's shape by citing `implementation_execution`'s precedent — but `implementation_execution` carries `started_at`/`completed_at` timestamps, and `post_merge_test_status` as designed carries none. A mutable field recording a state transition should have a timestamp (this review's own audit-fields check, and the exact standard the cited precedent itself sets) — without one, nobody can tell *when* a milestone's post-merge tests passed or failed, only that they eventually did. Needs: add `post_merge_test_status_updated_at` (or fold the status into a small nested block mirroring `implementation_execution`'s own `{state, ...timestamps}` shape) so the field is actually consistent with the precedent cited to justify it.

### Lead Developer Findings

**[LD-1] blocking — §3.3/ADR-0004's team-path mechanism (`gh pr create` then `gh pr merge` in the same step) has no stated behavior for `gh pr merge` failing after the PR was successfully created.** A real branch-protection configuration (required status checks, required additional approvals) could easily cause the immediate `gh pr merge` to fail or stall even though the PR itself was created successfully — plausible precisely because this mode targets the branch-protected-enterprise persona SPEC-0002's own User Stories describe. As designed, there's no stated verification step before `**Implementation status:**` is set to `pr_merged` — DESIGN-0001's own standing principle ("Operator must not assume a command succeeded because it was invoked... verify the outcome from resulting artifacts") applies exactly as much here, to a plain markdown-instruction sequence as to Operator's own cycle. Needs: state explicitly that `**Implementation status:** pr_merged` is only written after confirming the merge actually completed (checking the PR's real merged state, not assuming `gh pr merge`'s exit code alone) — and state what happens if it doesn't: the PR is left open, `**Implementation status:**` stays at `approved`, and the human is told to resolve the blocking branch-protection requirement before re-running.

### SRE Findings

**[SRE-1] non-blocking — P1+ test failure handling under `post_merge` is left unstated.** Today's `pre_merge` model is explicit that P1 and lower failures don't block `tests_passed`. Under `post_merge`, does a P1 failure flip `post_merge_test_status` to `failed`, or does `failed` stay reserved for P0 only (consistent with `test_failed` today also being P0-only-triggered)? Reasonably inferable by analogy, but DESIGN should state it explicitly rather than leave an implementer to guess — the same "abstract decisions aren't implementable" standard this review applies to Key Decisions generally.

**Clean perspectives:** Security.

### Contradictions Requiring Arbitration

None. One cascade worth noting: once `[LD-1]` is resolved (explicit merge-verification + failure path), SRE's own dependency-failure-mode coverage (§3.6) should be extended to name this specific failure mode too, alongside the already-covered `stack.qa_trigger_command` infrastructure-failure case — the same class of "verify before trusting success" gap, just on the merge side rather than the test-execution side.

## Address Summary — Round 1

**Date:** 2026-10-17

### Addressed
- `[DBA-1]` — `post_merge_test_status` (bare enum) replaced with `post_merge_test: {status, updated_at}`, directly mirroring `implementation_execution`'s own `{state, ...timestamps}` shape, in §3.1/3.4/3.5/3.6.
- `[LD-1]` — §3.3 and ADR-0004 both revised: the team-path merge is now explicitly verified (`gh pr view --json state`, not `gh pr merge`'s exit code alone) before `**Implementation status:**` advances; an unmerged PR (blocked by branch protection) leaves the milestone at `approved` with the PR open, re-run finds and retries it rather than duplicating. §3.6 gained a matching Failure Modes entry. The cascade noted in Round 1's Cross-Perspective Check (extending SRE's own failure-mode coverage) is addressed by this same §3.6 addition.
- `[SRE-1]` — §3.4 now states explicitly: only a P0 failure sets `post_merge_test.status: failed`, exactly mirroring `test_failed`'s existing P0-only trigger under `pre_merge`; a P1-or-lower failure is recorded in the test table as today without flipping this status. Addressed directly (clear completeness gap, the answer was unambiguous by direct analogy to an already-established convention) rather than accepted as non-blocking risk, since leaving it unstated would have meant an implementer guessing the one genuinely safety-relevant branch (P0 vs. not) of this entire field's write logic.

### Not Addressed
*(none — all three Round 1 findings addressed above.)*

## Design Review — Round 2

**Date:** 2026-10-17
**Status:** Approved — no blocking findings

Scope: §3.1/3.3/3.4/3.5/3.6, Key Decision 2, and ADR-0004 — the sections Round 1's address summary touched. Options Analysis and Security: no new findings — domain unaffected by Round 1's address.

**DBA:** `[DBA-1]` confirmed resolved — `post_merge_test: {status, updated_at}` now genuinely mirrors `implementation_execution`'s audit-field pattern. Considered whether full field-for-field parity (an `attempt` counter, a separate `started_at`) was also needed — no: unlike `implementation_execution`, there is no partial/interrupted-write scenario here (`/execute-tests` writes `status`+`updated_at` together in one invocation, never a bare "started" state), so a single timestamp is the right amount of mirroring, not an incomplete one.

**Lead Developer:** `[LD-1]` confirmed resolved — merge verification is explicit (`gh pr view --json state`), status is never advanced on `gh pr merge`'s exit code alone, and the not-merged path correctly leaves the milestone at `approved` with the PR open for a non-duplicating retry.

**SRE:** `[SRE-1]` confirmed resolved — the P0-only trigger for `status: failed` is now explicit and correctly mirrors `test_failed`'s existing convention. The Round 1 cascade (extending §3.6's failure-mode coverage to the merge-verification case) is confirmed present.

No new findings from this round's own re-examination of the revised sections.

**Clean perspectives:** Options Analysis, Security, DBA, SRE, Lead Developer.

### Contradictions Requiring Arbitration

None.
