---
id: create-spec
type: command
name: /create-spec
description: TPO command to author a SPEC artifact from a problem description or Postmortem action item
status: active
version: 3
primary_role: tpo
output_artifact_type: spec
changelog:
  - v3 — frontmatter initialization now includes `changelog: []`, which approve-spec.md and revise-spec.md both already assumed existed and appended to on every version bump. Same gap class as ARCH.md.template (changelog) and create-plan.md/create-test-plan.md (updated), caught during SPEC-0001's own first real /approve-spec run.
  - v2 — added the coordination-repo path (`repo_role: coordination`): declares thread participants, allocates THREAD-NNNN, writes thread.yml, and outputs SPEC.md under sdlc/THREAD-NNNN/ instead of sdlc/specs/. No change for repo_role: code.
---

# /create-spec

## Purpose

Author a new SPEC artifact from a problem description. The TPO applies the spec quality bar: clean problem framing, scope-disciplined non-goals, bidirectional goal-to-AC traceability, and testable acceptance criteria. Output is SPEC-NNNN in `sdlc/specs/`.

## When to Invoke

- A new feature or capability needs to be specified (human-initiated)
- A Postmortem action item generates a new Spec requirement (via `derived_from` link to PM-NNNN)
- A previous SPEC was superseded and a replacement is needed

## Required Inputs

- **Problem description:** A clear statement of the pain being addressed — who experiences it, under what conditions, what the current workaround costs. Does not need to be a formal artifact; a conversation or a Postmortem action item is sufficient.
- **Rough scope intent:** What the requestor believes is in scope vs. out of scope. The TPO will sharpen this, but needs a starting position.
- **Stack context:** Read ARCH.md for the project's tech stack, constraints, and conventions. These are the implementation boundaries a spec must respect.

## Pre-flight — Coordination repo check

Read `project.yml`'s `repo_role`. This step applies only when it's `coordination`; for `repo_role: code` (every existing project, and the default), skip straight to "Pre-flight — Gather inputs" below — nothing changes.

**If `repo_role: coordination`:** this invocation authors SPEC.md for a new cross-repo thread, not a local project spec. Before gathering the usual inputs:

1. Ask for the participant repos:
   > **Which repos does this thread span?** List each one (e.g., `org/transformer-service`, `org/document-worker`). At least two — a thread with only one participant belongs in that repo's own `create-spec`, not here.
2. Determine the next available `THREAD-NNNN` — scan `sdlc/` for existing `THREAD-NNNN/` directories and take the next sequential number. This is a convenience allocation, not a collision guarantee; a CI check on this repo validates uniqueness against HEAD at merge time and requires the later of two concurrent allocations to reallocate — the same shape of problem as two migration files landing on the same number.
3. Create `sdlc/THREAD-NNNN/` and write `thread.yml`:
   ```yaml
   thread: THREAD-NNNN
   participants:
     - repo: org/transformer-service
     - repo: org/document-worker
   ```
4. The SPEC this command produces is written to `sdlc/THREAD-NNNN/SPEC.md` — not `sdlc/specs/SPEC-NNNN-slug.md`. The thread number is the identifying number; there is no separate SPEC sequence inside a coordination repo.
5. Read `ARCH.md` from every participant repo (not this repo's own — a coordination repo has none) before drafting, since Constraints & Assumptions and NFRs need to respect every participant's actual stack, not a single implicit one. Record each as an `inputs.architecture` entry (repo/ref/path, per [`templates/reference/cross-repo-resolution.md`](../reference/cross-repo-resolution.md)) in SPEC-NNNN frontmatter — every downstream command in this thread re-resolves and extends this same list rather than re-deriving participants from `thread.yml` each time.

Everything else in this command proceeds unchanged, with `sdlc/THREAD-NNNN/SPEC.md` as the output path instead of `sdlc/specs/SPEC-NNNN-slug.md`.

## Pre-flight — Gather inputs

Before drafting, confirm the two required inputs that cannot be inferred from the problem description alone. Ask explicitly and wait for answers before proceeding to the steps:

> **1. Scope intent — what is in scope for this spec?**
> List the top 3–6 capabilities or outcomes you expect this spec to cover. These become the starting point for Goals. You don't need to be exhaustive — the TPO will sharpen them.
>
> **2. Out of scope — what should this spec explicitly exclude?**
> List things a developer might reasonably assume are included but should not be built in this thread. These become Non-Goals. If you're unsure, list anything you'd want to defer to a future spec.

Do not proceed to Step 1 until both answers are provided. If the user says "I'll define it later" or defers, ask again:
> "I need your scope intent before I can draft — even a rough list helps me avoid Goals that are too broad or Non-Goals that are too vague. What should be in and out of scope?"

## Steps

1. **Frame the problem** — Restate the pain in one paragraph: who, what conditions, what cost. Confirm the problem statement does not describe a solution.
2. **Identify scope-creep vectors and check goal independence** — List the natural extensions a developer would be tempted to add. These become Non-Goals. Also ask: are all in-scope Goals interdependent, or could any subset ship and deliver user value independently? If Goals are independently deliverable, note the question explicitly in the draft — the TPO decides whether to split or consciously keep them together with a stated reason. Do not make the split decision silently.
3. **Draft Goals** — State outcomes (not tasks). Every Goal must be testable via an Acceptance Criterion.
4. **Draft Non-Goals** — State what is explicitly excluded, specifically enough to use as a decision rule mid-implementation.
5. **Draft User Stories** — For each role that interacts with the feature, state the capability and the "so that" value clause. The "so that" must be non-tautological.
6. **Draft Acceptance Criteria** — One or more testable criteria per Goal. Check bidirectional traceability: every Goal maps to at least one AC; every AC maps back to at least one Goal.
7. **Draft Non-Functional Requirements** — Capture constraints on system qualities: performance (latency targets or explicit deferral to Observability Planning), reliability (consistency guarantees, fail-open vs. fail-closed behavior), security constraints (credential handling, plaintext-never-persisted), forward compatibility (what must not be assumed), and operability (new infrastructure dependencies must be called out explicitly). Each NFR must be specific enough to constrain DESIGN. If no numeric threshold can be stated, name the downstream stage where it will be resolved.
8. **Draft Constraints & Assumptions** — Tech stack constraints (from ARCH.md), environmental assumptions (single environment, greenfield vs. existing data, user base size and trust level).
9. **Draft Security Considerations** — Identify sensitive surfaces (credentials, auth, PII). Include a self-assessment of `security_review_required`.
10. **Draft Observability Considerations** — Three distinct concerns:
    - **Metrics:** What counters, histograms, or gauges must this feature emit?
    - **Traces:** Which critical request paths need distributed tracing for latency diagnosis?
    - **Logs:** What events must be logged for operations, debugging, or audit purposes? What must never be logged (credentials, plaintext keys, PII)? If an audit trail is required, state it here even if the user-facing view is a Non-Goal.
11. **Draft Open Questions** — List only genuinely open items. Each must name a downstream stage for resolution.
12. **Self-check against TPO quality bar** — Problem framing, non-goal specificity, AC testability, bidirectional traceability, NFR specificity, observability concerns captured. Revise before handing off.

## Output Contract

The produced SPEC must contain all of the following Required Sections: Problem Statement, Background (if needed), Goals, Non-Goals, User Stories, Acceptance Criteria, Non-Functional Requirements, Constraints & Assumptions, Security Considerations, Observability Considerations, Open Questions.

Frontmatter must include: `id`, `type: spec`, `schema_version: 1`, `title`, `status: draft`, `created`, `updated`, `version: 1`, `changelog: []` (`approve-spec.md` and `revise-spec.md` both append entries to this on every version bump; it must exist from creation, not be added ad hoc at first approval), `owner`, `priority`, `stakeholders`, `links: []`, `generation` block (`command: /create-spec`, `command_version: [use version from this command's frontmatter]`, `model` [self-identify from system context — e.g., claude-sonnet-4-6], `generated_at: [today's date]`, `session_id: null`). For a `repo_role: coordination` thread, frontmatter also includes `thread: THREAD-NNNN` and an `inputs.architecture` list (one entry per participant ARCH.md read).

## Quality Gate

Before handing off to `/review-spec`:
- [ ] Problem statement describes pain, not solution
- [ ] Every Non-Goal is specific enough to apply as a decision rule without asking the TPO
- [ ] Every Goal maps to at least one AC
- [ ] Every AC maps to at least one Goal (no orphaned requirements)
- [ ] Every AC is specific enough to write a failing test for
- [ ] Every NFR is specific enough to constrain DESIGN or names the stage where the threshold will be resolved
- [ ] Observability Considerations cover metrics, traces, and logging — including what must never be logged
- [ ] `security_review_required` self-assessment is present and reasoned
- [ ] No open question is so fundamental it could change a Goal or Non-Goal if answered differently
- [ ] `schema_version: 1` present in frontmatter
- [ ] `changelog: []` present in frontmatter
- [ ] `generation` block present: command, command_version, model self-identified, generated_at set to today's date, session_id: null

**Self-check before proceeding:** Re-read each item above. Confirm each is complete, or state what was skipped and why before continuing to Handoff.

## Handoff

**Create branch:**
Derive a 2-4 word slug from the SPEC title (lowercase, hyphens, drop filler words).
```
git checkout -b thread/NNNN-slug
```

**Show the draft to the user:**
> Display the SPEC file. Ask: "Please review the draft spec. Confirm when ready to proceed."

**Once user confirms — run the review:**
→ `/review-spec SPEC-NNNN`

Nothing is committed yet. `/review-spec` runs the analysis, captures results, and the loop continues until `/approve-spec` handles the single commit for the entire spec phase.
