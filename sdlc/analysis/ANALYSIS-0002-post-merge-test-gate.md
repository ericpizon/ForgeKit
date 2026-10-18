---
id: ANALYSIS-0002
type: analysis
schema_version: 1
spec_id: SPEC-0002
status: completed
target_spec_version: 3
readiness_verdict: ready
security_review_required: no
generation:
  command: /review-spec
  command_version: "6"
  model: "claude-sonnet-5"
  generated_at: "2026-10-09"
  session_id: null
---

# ANALYSIS-0002 — Review of SPEC-0002

## Analysis Round 1 — 2026-10-09

### Scope Coherence Check

- **Independent deliverability:** No subset ships independently with real value. A config field with no consuming logic (G1 alone) is useless; G3's failure-response only exists within the mode G1 defines; G4's clean-closure behavior only matters once G1's merge-now path exists.
- **AC partitionability:** No clean partition — AC-01/AC-02 are sequentially dependent within the same workflow; AC-06/AC-07 depend on AC-01 having actually happened.
- **Risk profile divergence:** AC-05 (a config question) is trivial; AC-01-03/AC-06-07 (rerouting a safety-relevant pipeline sequence) carry real risk. This divergence exists but doesn't warrant a split — it's inherent complexity variance within one feature, not two separable products.

**[SC-1] non-blocking — Scope coherence confirmed.** Goals are interdependent and cannot ship independently; no split recommended.

### Completeness Assessment

All required sections present. Every Goal maps to at least one AC (G1→01/02/04/05, G2→04, G3→03, G4→06/07); every AC maps back to a Goal; no orphans. Every User Story maps to an AC (Story 1→AC-01, Story 2→AC-02, Story 3→AC-04, Story 4→AC-03). No scope-consistency contradiction found between any AC and the stated Non-Goals.

### Ambiguities & Open Questions

**[LD-1] blocking — AC-02 bundles two different validation targets into one AC without picking one.** "`/execute-tests` ... validates the state now on `main` (or an externally-deployed environment sourced from it)" — these are observably different implementations: re-running tests against the merged *commit* (no live environment needed, closer to today's local/CI path just re-pointed at `main`) versus validating an actual *running* QA deployment (requiring the project to expose some reachable target — a URL, a deploy-status check, something). OQ-3 defers the *trigger mechanism*, but not this more basic question of *what is actually being validated*. Needs: AC-02 reworded to pick one (or explicitly state both are valid sub-modes DESIGN must enumerate separately, with each given its own AC).

**[QE-2] non-blocking — No stated behavior for a project switching `pipeline.test_gate` mid-project, with milestones already in flight under the old mode.** Doesn't need an AC — but deserves at least an Open Question naming it, the same way OQ-1/OQ-2 name other deferred edge cases. As written, a reader can't tell whether this was considered and judged out of scope, or simply not thought about.

### Risk Flags

**[LD-2] blocking — The inherent operational risk of `post_merge` mode (code on `main`/QA that hasn't passed tests yet, for some window) is never explicitly named or accepted anywhere in the SPEC.** The Reliability NFR addresses only the merge *action* being human-confirmed — it says nothing about the residual risk the *mode itself* trades in for its enterprise benefit. This is exactly the kind of thing this codebase's own prior security reviews (SEC-0001's `RET-1`/`RET-2`) treat as something to name and accept explicitly, not leave implicit — a reader of this SPEC should not have to infer "oh, I guess `main` can be briefly broken under this mode" from first principles. Needs: an explicit statement under Constraints & Assumptions (or a new NFR) naming this as an accepted, inherent trade-off of choosing `post_merge` — not a defect to fix, just something this SPEC should say out loud.

No new security, credential, or data-sensitivity risk beyond what's already addressed in Security Considerations.

### Quality Engineer Findings

**[QE-1] blocking — No defined behavior for a declined merge offer (AC-01's missing negative case).** AC-01 says `/approve-implementation`'s Handoff "offers to merge... directly" — what happens if the human declines? Stays at `approved`, same as today's confirmation pattern? Something else? This is a real, reachable state with no stated behavior at all, not a hypothetical edge case.

**[QE-3] blocking — The milestone status enum has no way to represent "merged, post-merge test outcome still pending/in-flight," and AC-02/AC-03's own described behavior has nothing to write their result into once AC-06 fires.** Per AC-06, the milestone reaches `pr_merged` *immediately* at merge — a terminal status in today's enum. But AC-02 says `/execute-tests` then runs *afterward*, and AC-03 says a failure routes to `/create-bug`/`/escalate-to-incident`. Once the milestone is already at the terminal `pr_merged` status, there is no field anywhere that distinguishes "tests haven't run yet," "tests passed," and "tests failed, BUG-NNNN filed" — they all read identically as `pr_merged`. This is a state/lifecycle coverage gap in the literal sense this review's Step 10 checks for (a missing cell in the {status} × {operation} cross-product), not just a missing test case. Needs: either a new sub-field tracking post-merge test outcome (distinct from the milestone's own terminal implementation status), or an explicit statement that this is tracked entirely in the BUG/incident record instead, with the milestone intentionally staying `pr_merged` throughout regardless of test outcome — either is acceptable, but the SPEC needs to say which, since AC-06 and AC-02/03 currently read as mutually unaware of this exact tension.

AC testability: all 7 ACs are individually testable (none describe internal-only behavior). No other missing negative/boundary AC found beyond `[QE-1]`. No compound ACs found — AC-02's and AC-06's multi-clause phrasing each describes one causally-coupled outcome, not independently-verifiable facets that should be split. No unstated test preconditions found. All NFRs have a verification path or an explicit DESIGN deferral.

### Security Review Trigger Assessment

| Criterion | Result | Rationale |
|---|---|---|
| Authentication/authorization | no | No new auth surface — reuses existing git/merge mechanics |
| PII/sensitive data | no | No new data handled |
| Secrets/credential management | no | No new credential handling |
| Third-party integrations | no | `post_merge`'s QA-trigger mechanism (OQ-3) reuses the shape of the existing `stack.ci_trigger_command` integration point, not a new one ForgeKit itself owns — re-assess at DESIGN only if OQ-3 resolves to something genuinely new |
| Data retention/compliance | no | No new data retained |
| LLM/agent untrusted input | n/a | `stack.llm_agent_untrusted_input` is `n/a` for this repo, unchanged |
| Regulatory/compliance framework scope | n/a | `compliance.frameworks: [none]` for this repo, unchanged |

`security_review_required: no` — confirmed, no change from the SPEC's own self-assessment.

### Readiness Verdict

**`needs_revision`** — 4 blocking findings (`[LD-1]`, `[LD-2]`, `[QE-1]`, `[QE-3]`), 2 non-blocking (`[SC-1]`, `[QE-2]`). All four blocking findings require a Goal/AC/NFR/Constraints change, not a Design-level resolution alone — per this command's own readiness rule.

### Suggested Spec Revisions

- Rework AC-02 to pick one validation target (or split into two ACs, one per sub-mode, with DESIGN enumerating which applies when).
- Add an explicit, named acceptance of `post_merge`'s residual risk (code on `main`/QA before tests confirm it) under Constraints & Assumptions.
- Add a negative-case AC (or extend AC-01) covering a declined merge offer.
- Add a new field or explicit statement resolving how post-merge test outcome is tracked once the milestone itself reaches the terminal `pr_merged` status.
- Add an Open Question naming the mid-project mode-switch case (`[QE-2]`), even if its resolution is deferred to DESIGN.

## Analysis Round 2 — 2026-10-09

Scope: re-read SPEC-0002 v2 in full against the Revision Summary — Round 1. Confirmed resolved on substance: `[LD-1]` (AC-02 now states the invariant, no longer bundles two targets), `[QE-1]` (AC-01's declined-merge case is explicit), `[QE-2]` (OQ-5 added), `[QE-3]` (AC-08 added, correctly resolves the tension with AC-06 — AC-06 governs the milestone's own terminal status, AC-08 governs a separate field, no remaining contradiction between them).

### Lead Developer Findings

**[LD-1] blocking — the new Constraints bullet imposes a requirement on AC-05 that AC-05's own text doesn't commit to.** The `[LD-2]`-fix bullet under Constraints & Assumptions says "the `/forge-init` question (AC-05) should say so plainly, not just offer the two mode names" — but AC-05 itself was never updated and still only says "the human is asked which `test_gate` mode to use, with `pre_merge` as the default." A Constraint cannot impose a testable requirement an AC doesn't itself state — this is the same class of inconsistency Step 2's scope-consistency check exists to catch, just in the Constraint→AC direction rather than the AC→Non-Goal direction it's usually phrased for. Needs: AC-05 itself extended to require the risk trade-off be communicated as part of the question, not left as an unenforced aspiration sitting one section away.

### Quality Engineer Findings

**[QE-1] non-blocking — `AC-08`'s new post-merge test-outcome field raises a question `AC-07`'s dependency gating doesn't address: should a *failed* post-merge test re-block a dependent milestone that already started under `AC-07`'s "recognizes the prior milestone as unlocking work once merged"?** This genuinely didn't exist as a question in Round 1 — it's a direct consequence of adding `AC-08`'s tracking field. Not blocking: whether to block dependent work on a post-merge failure is a legitimate policy trade-off (an enterprise might deliberately want unrelated downstream work to continue), not something this SPEC needs to mandate one way. Needs: a new Open Question naming it, the same treatment `[QE-2]` got in Round 1.

**Clean perspectives:** Security (confirmed unaffected — no new field carries sensitive data; `security_review_required: no` still holds), Scope Coherence (unaffected by this round's changes).

### Readiness Verdict

**`needs_revision`** — 1 blocking (`[LD-1]`), 1 non-blocking (`[QE-1]`).

### Suggested Spec Revisions

- Extend AC-05 itself to require the `post_merge` trade-off be communicated as part of the `/forge-init` question, not left as an unenforced Constraints-only aspiration.
- Add an Open Question naming whether a post-merge test failure should affect `AC-07`'s dependency-gating for already-unlocked downstream milestones.

## Analysis Round 3 — 2026-10-09

Scope: re-read SPEC-0002 v3 in full against the Revision Summary — Round 2, then a holistic pass across the whole document (traceability, scope consistency, remaining ambiguity, NFR/Observability/Security completeness) given this is the third round.

Confirmed resolved on substance: `[LD-1]` (`AC-05` itself now states the trade-off plainly, no longer an unenforced Constraints-only aspiration), `[QE-1]` (`OQ-6` added, correctly scoped as a DESIGN-level policy trade-off rather than a mandated AC).

No new findings. Bidirectional traceability still holds (unaffected by this round — no AC added or removed). No remaining ambiguity found across any AC. None of `OQ-1` through `OQ-6` are fundamental enough to change a Goal or Non-Goal depending on how DESIGN resolves them — each is an implementation-shape question within already-fixed scope. NFR, Observability, and Security sections unaffected and still complete.

**Clean perspectives:** Lead Developer, Quality Engineer, Security, Scope Coherence.

### Readiness Verdict

**`ready`**

### Suggested Spec Revisions

None — no findings this round.
