---
id: review-plan
type: command
name: /review-plan
description: Lead Developer command to run all Stage 5 plan perspectives in one pass — SRE, Developer, QE, Security, Architect
status: active
version: 1
primary_role: lead-developer
output_artifact_type: plan (reviewed)
---

# /review-plan

## Purpose

Run all Stage 5 plan reviews in a single pass, adopting each reviewer's perspective in sequence. The Lead Developer executes this command immediately after drafting PLAN, and again after each revision in `/address-plan-review`. Output is a `## Plan Review — Round N` section appended to PLAN-NNNN.

## When to Invoke

- After the user confirms the `/create-plan` draft — no commit required before review
- After `/address-plan-review` appends an Address Summary to PLAN-NNNN (next review round)

Pass the PLAN, SPEC, DESIGN, and (if applicable) SEC IDs: `/review-plan PLAN-NNNN SPEC-NNNN DESIGN-NNNN SEC-NNNN`.

## On Re-Review

If a `## Plan Review — Round N` section already exists in PLAN-NNNN:

1. Read `current_review_round` from PLAN-NNNN frontmatter. The new round number is `current_review_round + 1`.
2. Read the latest `## Address Summary — Round N` section to see which findings were addressed, accepted, or deferred.
3. Load only the PLAN tasks cited in the address summary — do not re-read the full task list. Tasks not referenced by any addressed finding are unchanged.
4. For each perspective: verify previously-blocking findings marked "Addressed" are actually resolved. Do not re-raise findings marked with accepted-risk rationale. **If a perspective's domain was unaffected by the address pass, emit `[Perspective]: no new findings — domain unaffected by Round N address.` and skip the full perspective analysis.**
5. **Happy Path Trace:** if the trace was complete in the prior round (no gaps), skip re-validation. Only re-check rows for tasks that were added or modified during the address pass.
6. Append a new `## Plan Review — Round N` section. Do not modify prior round sections.

## Before Running Perspectives

Verify PLAN-NNNN contains all required sections. If a section is missing, return to `/create-plan` — do not run perspectives against an incomplete artifact.

Required sections:
- [ ] Task Breakdown (ordered tasks with IDs, source tags, AC references)
- [ ] Milestones (each with task list, AC coverage, depends-on, rollback note)
- [ ] Sequencing & Dependencies
- [ ] Happy Path Trace (one table per primary User Story — absence is non-blocking but noted)
- [ ] Definition of Done (AC coverage map)
- [ ] Out of Scope for This Plan

Required frontmatter fields:
- [ ] `status: draft`
- [ ] `target_design_version` set
- [ ] `milestone_count` set (not TBD)
- [ ] `includes_observability_tasks` and `includes_security_tasks` set

## Steps

### 1. SRE Perspective

Adopt the SRE role. Review PLAN-NNNN for operational safety. Assess:

1. **Migration task ordering.** Schema migrations must be applied before the code that depends on them is deployed. If a code task is ordered before its migration task, that is a blocking finding. Migrations must be placed in the earliest milestone that requires them.
2. **Rollback feasibility.** Can each individual PR be reverted without leaving the system in an inconsistent state? A milestone that removes or restructures something introduced by a previous milestone requires a documented rollback procedure.
3. **Ops gaps.** Are there tasks that require manual production steps (env var changes, one-time migration scripts) not represented as explicit PLAN tasks? Any implicit manual step must be made explicit.
4. **Deployment sequence consistency.** Does the deployment sequence ensure critical infrastructure is in place before dependent features go live? Flag any sequence that creates a window of inconsistency.
5. **OBS-NNNN task coverage.** Are all OBS-NNNN instrumentation tasks referenced in the PLAN by source ID (e.g., `OBS-NNNN IT-1`)? A missing OBS task reference is a non-blocking finding. Do not flag an OBS task as missing just because its description was not copied — ID reference is the correct and expected format.

### 2. Developer Perspective

Adopt the Developer role. Review PLAN-NNNN from the implementer's perspective — every task must be specific enough to implement without making a design decision. Assess:

1. **Implementation ambiguity.** Can each task be picked up and implemented without asking a clarifying question? A task that names the specific function, file, and behavior is unambiguous. A task that says only "implement the endpoint" is not — flag it.
2. **Implicit ordering and milestone coherence.** Are there tasks listed as independent that actually depend on an earlier task? Also check milestone coherence: does each milestone represent a coherent, self-contained unit? A milestone that requires types, functions, or schema from an undeployed milestone has an undocumented cross-milestone dependency.
3. **Implicit design decisions.** Does any task require the Developer to make a choice that belongs in the DESIGN? Flag any task that contains a hidden decision.
4. **AC coverage.** Does every task reference the SPEC acceptance criteria it covers? A task that cannot be traced to an AC may be out of scope.
5. **"Wire up X" tasks.** Tasks described as "wire up," "connect," or "integrate" without specifics are too vague — flag them.
6. **Milestone sizing.** Is each milestone scoped for a manageable single PR? More than 8–10 tasks or multiple unrelated capabilities is likely too large — flag as **non-blocking** with a proposed split. Also flag milestones with 1–2 trivial tasks that could merge with an adjacent milestone.
7. **Spike task quality.** For each spike task: (a) Does it state a specific answerable question? (b) Does it include a timebox? (c) Does it explicitly name the task(s) it unblocks? A spike missing any of these is a **non-blocking** finding.

### 3. Quality Engineer Perspective

Adopt the Quality Engineer role. Adversarially review PLAN-NNNN for testability. Assess:

1. **AC traceability.** For each AC in SPEC-NNNN, is there a PLAN task that implements it? If an AC has no corresponding task, that is a blocking finding.
2. **Verifiable outcomes.** Can a test be written that proves each task was completed correctly? Abstract task descriptions produce guess-and-check implementations.
3. **Write task read path coverage.** If a task writes data, is there a task that provides a read path so the test can verify the write?
4. **Decomposition gaps.** Are there compound tasks that combine multiple distinct verifiable behaviors that should be independently testable?
5. **Testability preconditions.** Does a task depend on environment state not set up by an earlier task? Missing prerequisites make tests fragile.
6. **Per-milestone testability.** Can each milestone be tested in isolation after deployment?
7. **Happy path trace validation.** Does PLAN-NNNN contain a Happy Path Trace section? For each User Story trace: are all flow steps accounted for? A **missing Happy Path Trace section** is **non-blocking**. A **gap within an existing trace** is **blocking** — an unimplemented flow step means the user journey is broken.

### 4. Security Perspective *(only if SEC-NNNN exists)*

Adopt the Security role. Verify every SEC-NNNN finding has a corresponding implementation task. Assess:

1. **Complete finding list.** List every finding in SEC-NNNN by ID and severity.
2. **Finding coverage.** For each open or mitigated finding, find the corresponding PLAN task. If no task maps to the mitigation, that is a blocking finding.
3. **Quiet deferral.** Deferral of a High finding requires explicit accepted-risk documentation in SEC-NNNN with a named approver — it cannot happen silently in the PLAN.
4. **Security-sensitive task specificity.** Any task implementing a security mitigation (e.g., timing-safe comparison, input sanitization, auth enforcement) must name the specific mechanism — a generic task description is insufficient.
5. **Deployment window.** Is there a security task that needs to deploy atomically with another to avoid a transient vulnerability window?

### 5. Architect Perspective

Adopt the Architect role. Review PLAN-NNNN for fidelity to DESIGN-NNNN. Assess:

1. **DESIGN fidelity.** For each Key Decision in DESIGN-NNNN, find the corresponding PLAN task. If a Key Decision has no task, that decision was silently deferred to the Developer — blocking finding.
2. **Surface coupling.** Are there tasks that imply inappropriate sharing of state or logic between distinct system boundaries? Flag coupling that contradicts the DESIGN.
3. **Deferred design decisions.** Does any task require the Developer to make a choice that belongs in the DESIGN?
4. **Module layout conformance.** Do the tasks imply creating files in the locations defined in ARCH.md? Deviations must be documented.
5. **ADR traceability.** If any task implements behavior that was the subject of an ADR, the task description should reference that ADR.
6. **Milestone delivery sequence fidelity.** Does the milestone ordering respect DESIGN's module layering?

### 6. Cross-Perspective Check

- **Contradictions:** Does the SRE migration ordering conflict with the Developer's ordering assumptions? Does the Security deployment window conflict with the SRE rollback story? Flag contradictions — the Lead Developer must arbitrate.
- **Cascade:** Did the Developer perspective reveal a decomposed task that changes AC coverage the QE perspective should know about?

## Output

Append a `## Plan Review — Round N` section to PLAN-NNNN. Then update `current_review_round` in PLAN-NNNN frontmatter to the round number just appended.

**Null suppression:** Omit any `### [Perspective] Findings` section that has no findings — do not emit the section header or a "None." line. List all clean perspectives in a single `**Clean perspectives:** [list]` line before `### Contradictions Requiring Arbitration`.

```
## Plan Review — Round N

**Date:** YYYY-MM-DD
**Status:** Changes requested — N blocking, N non-blocking | Approved — no blocking findings

### SRE Findings  *(omit section if none)*
- [SRE-1] **blocking** — [finding]

### Developer Findings  *(omit section if none)*
- [DEV-1] **non-blocking** — [finding]

### QE Findings  *(omit section if none)*
- [QE-1] **blocking** — [finding]

### Security Findings  *(omit if SEC-NNNN does not exist or no findings)*
- [SEC-1] **blocking** — [finding]

### Architect Findings  *(omit section if none)*
- [ARCH-1] **non-blocking** — [finding]

**Clean perspectives:** [list any perspectives with no findings]

### Contradictions Requiring Arbitration
- [or "None"]
```

## Quality Gate

- [ ] All required perspectives completed
- [ ] Milestone sizing checked for each milestone
- [ ] Every spike task checked for question, timebox, and unblock target
- [ ] Happy Path Trace validated
- [ ] Each finding labeled blocking or non-blocking with a reason
- [ ] Cross-perspective check completed

**Self-check before proceeding:** Re-read each item above. Confirm each is complete, or state what was skipped and why before continuing to Handoff.

## Handoff

- If blocking findings exist → "Run `/address-plan-review PLAN-NNNN` to address findings."
- If no blocking findings → "Run `/approve-plan PLAN-NNNN` to close the plan phase with a single commit."
