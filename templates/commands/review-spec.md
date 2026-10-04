---
id: review-spec
type: command
name: /review-spec
description: Lead Developer command to review a SPEC for scope coherence, completeness, ambiguity, risk, and implementation feasibility; produces ANALYSIS-NNNN
status: active
version: 6
primary_role: lead-developer
reviewing_roles:
  - security
output_artifact_type: analysis
changelog:
  - v6 — Output Contract now requires every finding to carry a bracketed, role-prefixed ID ([LD-N]/[QE-N]/[SEC-N]/[SC-N]), matching every other review command and resolving a contradiction with revise-spec.md's prior "no numeric IDs" instruction.
  - v5 — added a coordination-repo check: the Lead Developer perspective reads ARCH.md from every thread participant (not one) when reviewing a coordination-repo SPEC.
---

# /review-spec

## Purpose

Review a SPEC for completeness, internal consistency, ambiguity, and implementation feasibility. Produce ANALYSIS-NNNN with a `readiness_verdict` and a `security_review_required` decision. A `needs_revision` verdict sends the SPEC back to the TPO via `/revise-spec`. A `ready` verdict unblocks `/approve-spec`.

## When to Invoke

- Immediately after `/create-spec` produces the SPEC draft and the user confirms
- After `/revise-spec` revises the SPEC (next review round)

Pass the SPEC ID: `/review-spec SPEC-NNNN`.

## Pre-flight — Coordination repo check

Read `project.yml`'s `repo_role`. This step applies only when it's `coordination`; for `repo_role: code` (the default), skip straight to "On Re-Review" / "Steps" below — nothing changes.

**If `repo_role: coordination`:** SPEC-NNNN describes a cross-repo thread, so "implementation feasibility relative to the project's actual stack" has no single answer — re-resolve every entry in SPEC-NNNN's own `inputs.architecture` list (per [`templates/reference/cross-repo-resolution.md`](../reference/cross-repo-resolution.md)) rather than reading a single local ARCH.md. Carry the same list forward as an `inputs.architecture` entry in ANALYSIS-NNNN frontmatter. The Lead Developer Perspective's feasibility check (below) is assessed against every participant's stack — a goal or NFR only infeasible for one participant is still a feasibility finding.

## On Re-Review

If an `## Analysis Round N` section already exists in ANALYSIS-NNNN:

1. Identify the current round number (prior round + 1).
2. Read the `## Revision Summary — Round N` section appended to SPEC-NNNN by `/revise-spec`. Note which findings were addressed, which were accepted risk, which were deferred to design, and which were deferred with rationale.
3. Read the previous `## Analysis Round N` findings in ANALYSIS-NNNN.
4. For each perspective: confirm previously-flagged items are resolved per the revision summary. Do not re-raise items marked accepted risk or deferred — those are documented decisions. Check whether the revision introduced new issues or changed scope consistency.
5. Append a new `## Analysis Round N+1` section to ANALYSIS-NNNN. Do not modify prior round sections — they form the audit trail.

## Steps

Run all three perspectives in sequence. The combined verdict is produced after all three.

### Lead Developer Perspective

Adopt the Lead Developer role as defined in `.claude/roles/lead-developer.md`. Read ARCH.md before assessing feasibility — implementation feasibility is relative to the project's actual stack, not a generic baseline. For a coordination-repo thread, this means every participant's ARCH.md (see Pre-flight above), not one.

**Before reviewing content — check section presence:**
Verify all required sections exist in SPEC-NNNN: Problem Statement, Goals, Non-Goals, User Stories, Acceptance Criteria, Non-Functional Requirements, Constraints & Assumptions, Security Considerations, Observability Considerations, Open Questions. A missing section is a blocking finding — do not proceed with content review for that area until the section exists.

**0. Scope Coherence Check** — Evaluate whether the spec's Goals represent a single coherent user outcome or multiple independent ones. Ask three questions:

- **Independent deliverability:** Can any subset of Goals ship and provide user value without the rest being live? If yes, name the subset and what value it delivers alone.
- **AC partitionability:** Can the Acceptance Criteria be cleanly divided into two groups with no data, flow, or state dependencies between them? If yes, describe the partition.
- **Risk profile divergence:** Do parts of the spec have significantly different complexity, risk level, or delivery timeline? If yes, identify which parts and how they differ.

If the answer to any of the above is yes, surface a **non-blocking** split recommendation:

```
[SC-1] non-blocking — Scope coherence: Goals [X] and [Y] are independently deliverable.
[Y] could ship as a separate SPEC without [X] being live, delivering [stated value].
Recommended split: SPEC-NNNN covers [X]; a new SPEC covers [Y].
TPO decision required: split or document rationale for keeping together.
```

If the Goals are interdependent, note: "Scope coherence confirmed — Goals are interdependent and cannot ship independently."

This check is always **non-blocking** — splitting is a TPO product decision.

1. **Completeness check** — Verify every Goal maps to at least one AC; every User Story maps to at least one AC. Identify any Goal or User Story with no AC coverage.
2. **Scope consistency check** — Verify no AC implies functionality that a stated Non-Goal excludes. Identify contradictions.
3. **Ambiguity identification** — For each AC, identify whether it can be implemented two different ways that would produce observably different outcomes. If yes, flag it — the AC must pick one interpretation.
4. **Implementation feasibility check** — For each goal, NFR, and constraint, confirm it is achievable within the project's tech stack (from ARCH.md). Flag any goal or NFR that implies infrastructure the stack cannot support.
5. **Risk flag identification** — Security risks (auth surface, credential handling, data sensitivity), operational risks (failure modes, availability coupling), and scope risks (features that will be harder to add later due to this slice's decisions).
6. **NFR completeness and specificity** — Are NFRs present? Is each NFR specific enough to constrain a DESIGN decision? NFRs with no numeric threshold must name the stage where the threshold will be resolved. A missing NFR section is a finding.
7. **Observability Considerations completeness** — Are all three concerns addressed: metrics, traces, and logs (including what must never be logged)? Any logging requirement specifying credential or PII handling is a security-adjacent finding if absent.

### Quality Engineer Perspective

Adopt the Quality Engineer role as defined in `.claude/roles/quality-engineer.md`.

8. **AC testability** — Can each AC be verified with an automated test written before implementation? An AC that cannot produce a failing test is not a testable AC. Flag any AC that describes internal behavior rather than an observable outcome.
9. **Missing negative and boundary ACs** — For each positive AC, is there a corresponding AC for the failure or boundary case? Missing error-path ACs are findings.
10. **State/lifecycle coverage check** — For any entity with a status or lifecycle (e.g. an evaluation, order, or job with multiple states), enumerate the full cross-product of {status} × {operation the spec describes} (view, retry, decide, cancel, etc.) and confirm every cell has a stated, intentional behavior — not just the common-path cells. A missing cell (e.g. "can a decision be recorded against a `HARD_GATE_SKIPPED` entity, not just a `COMPLETED` one?") is a finding distinct from a simple missing negative/boundary AC — it's a systemic gap in state-machine coverage, not a single missing test case.
12. **Compound ACs** — Does any AC bundle multiple independently verifiable outcomes? If so, flag — it should be split so each can fail independently.
13. **Test preconditions** — Does any AC assume environment state not established by another AC in this SPEC? Unstated prerequisites make tests fragile or impossible to isolate.
14. **NFR testability** — For each NFR, is there a verification path? An NFR with no verification path produces untestable behavior — flag it.

### Security Perspective

Adopt the Security role as defined in `.claude/roles/security.md`. Assess the `security_review_required` flag:

15. **Security Review Trigger Assessment** — Evaluate against the checklist: authentication/authorization, PII/sensitive data, secrets/credential management, third-party integrations, data retention/compliance, LLM/agent input from untrusted or third-party text (check `project.yml`'s `stack.llm_agent_untrusted_input` — if `yes` and this SPEC's feature touches that call path, this criterion alone is sufficient to trigger, independent of whether any other criterion also fires), regulatory/compliance framework scope (check `project.yml`'s `compliance.frameworks` — if not `[none]` and this SPEC's feature touches data or flows in scope for any listed framework, this criterion alone is sufficient to trigger). Each criterion: `yes` or `no` with one-sentence rationale. Emit `security_review_required: yes` or `no`.

### Combined Verdict

16. **Readiness verdict** — `needs_revision` if any LD or QE finding requires adding, removing, or rewording a Goal, Non-Goal, User Story, AC, NFR, or Observability Consideration. `ready` if all remaining gaps are Design-level resolutions that do not require spec changes.

## Output Contract

Append an `## Analysis Round N` section to ANALYSIS-NNNN (create the file on Round 1). Each round section contains: Completeness Assessment, Ambiguities & Open Questions, Risk Flags, Security Review Trigger Assessment (with checklist), Readiness Verdict. Optional: Suggested Spec Revisions (if `needs_revision`).

Every finding (Scope Coherence, Completeness, Ambiguity, Risk Flag, QE, Security — any item raised in any perspective) must carry a bracketed, role-prefixed, per-round-numbered ID: `[LD-N]` for Lead Developer findings, `[QE-N]` for Quality Engineer findings, `[SEC-N]` for Security findings, `[SC-N]` for Scope Coherence findings. Number each prefix independently starting at 1 within each round (a new round restarts numbering). This matches the ID convention used by every other review command in the pipeline (`review-design.md`, `review-plan.md`, etc.) and is required so `/revise-spec` and later `/review-spec` rounds can reference findings unambiguously.

Update ANALYSIS-NNNN frontmatter each round: `readiness_verdict` (`ready` | `needs_revision`); `security_review_required` (`yes` | `no`); `target_spec_version`. On Round 1 creation: also initialize a `generation` block — `command: /review-spec`, `command_version: [use version from this command's frontmatter]`, `model` [self-identify from system context], `generated_at: [today's date]`, `session_id: null` — and set `schema_version: 1` in ANALYSIS frontmatter. Do not update `generation` or `schema_version` on re-reviews.

## Quality Gate

Before capturing review output:
- [ ] Scope Coherence Check completed — all three sub-questions assessed; result stated explicitly
- [ ] All three perspectives completed — Lead Developer, QE, Security
- [ ] Every Goal is checked for AC coverage; every gap is documented
- [ ] Every AC is checked for ambiguity; ambiguous ACs are named with both interpretations
- [ ] Every AC is checked for testability; untestable ACs named with reason
- [ ] Missing negative/boundary ACs identified or confirmed absent
- [ ] For any entity with a status/lifecycle, the status × operation cross-product is checked for coverage gaps
- [ ] NFRs checked for presence, specificity, and design-constraining value
- [ ] Every NFR checked for a verification path; untestable NFRs flagged
- [ ] Observability Considerations checked for metrics, traces, and logging coverage
- [ ] `security_review_required` verdict has a rationale
- [ ] Readiness verdict accounts for LD, QE, and any NFR/observability findings
- [ ] ANALYSIS-NNNN `schema_version: 1` present (Round 1 only)
- [ ] ANALYSIS-NNNN `generation` block present (Round 1 only): command, command_version, model self-identified, generated_at set, session_id: null

**Self-check before proceeding:** Re-read each item above. Confirm each is complete, or state what was skipped and why before continuing to Handoff.

## Handoff

**If `needs_revision`:**
> "The spec needs revision. Run `/revise-spec SPEC-NNNN ANALYSIS-NNNN` to address the flagged items."

**If `ready`:**
> "The spec is ready. Run `/approve-spec SPEC-NNNN ANALYSIS-NNNN` to commit and close the spec phase."
