---
id: create-plan
type: command
name: /create-plan
description: Lead Developer command to produce PLAN-NNNN — an ordered, traceable implementation task breakdown from DESIGN, SPEC, OBS, and SEC artifacts; includes happy path trace
status: active
version: 1
primary_role: lead-developer
reviewing_roles:
  - sre
  - developer
  - quality-engineer
output_artifact_type: plan
---

# /create-plan

## Purpose

Produce an ordered, traceable implementation task breakdown (PLAN-NNNN) from the approved DESIGN, OBS, and SEC artifacts. Every task is tagged by source (functional / observability / security), tied to one or more acceptance criteria or artifact sections, and ordered to respect implementation dependencies. The PLAN is the Developer's primary work artifact.

## When to Invoke

- DESIGN-NNNN `approved`
  If DESIGN-NNNN is `revision_requested`: **do not proceed.** "A design back-cycle is in progress. Wait until DESIGN-NNNN returns to `approved` after the review and approve-design cycle completes."
- OBS-NNNN `draft` or `approved` (instrumentation tasks available)
- SEC-NNNN `completed` (if triggered) — open findings resolved or accepted

Pass artifact IDs as arguments: `/create-plan DESIGN-NNNN SPEC-NNNN OBS-NNNN SEC-NNNN`.

## Required Inputs

- **DESIGN-NNNN** — approved; read all sections for functional task derivation
- **OBS-NNNN** — read Instrumentation Tasks section; these become `observability`-tagged tasks
- **SEC-NNNN** — (if triggered) read Implementation Tasks Derived section; these become `security`-tagged tasks
- **SPEC-NNNN** — read Acceptance Criteria for the Definition of Done mapping
- **ARCH.md** — read for framework phase ordering, module layout, and implementation patterns

## Task ID Convention

Assign sequential numeric IDs to tasks as they are identified: `T1`, `T2`, `T3`, etc. across all source types (functional, observability, security). IDs are assigned in the order tasks are derived — functional first, then observability, then security — and are stable once assigned. Do not renumber tasks after milestones are defined.

Spike tasks follow the same convention but are prefixed: `T2-spike`. A spike is a timeboxed investigation for an area of technical uncertainty that must be resolved before an implementation task can begin. Format: `T2-spike — [question to answer] — [timebox] — unblocks T3`. Spikes are first-class tasks: they appear in the Task Breakdown, belong to a milestone, and map to the AC or design decision they unblock.

## Steps

1. **Initialize PLAN frontmatter.** Create the PLAN file with:
   - `status: draft`
   - `version: 1`
   - `target_design_version`: version of DESIGN-NNNN being planned against
   - `includes_observability_tasks: false` (update to `true` in Step 3 if OBS-NNNN exists)
   - `includes_security_tasks: false` (update to `true` in Step 4 if SEC-NNNN exists)
   - `milestone_count: TBD` (update in Step 7 after milestones are defined)
   - `current_review_round: 0` (updated by `/review-plan` after each round)
   - `generation` block: `command: /create-plan`, `model` [self-identify from system context], `generated_at: [today's date]`
   - `pipeline_status`: snapshot of all upstream artifact statuses at plan creation time — updated by downstream approval commands:

   ```yaml
   pipeline_status:
     spec: SPEC-NNNN approved
     analysis: ANALYSIS-NNNN completed
     design: DESIGN-NNNN approved
     obs: OBS-NNNN approved          # 'not produced' if OBS-NNNN does not exist
     sec: SEC-NNNN completed         # 'not triggered' if SEC-NNNN was not run
     plan: draft
     test: pending
   ```

2. **Derive functional tasks from DESIGN.** For each component or section of the DESIGN (data model, infrastructure, data access layer, API layer, UI layer), identify the concrete implementation tasks. A task is concrete if a Developer can implement it without making a design decision. Assign IDs starting at T1. Read ARCH.md for the canonical phase ordering for this project's stack. If a component involves technical uncertainty, add a spike task immediately before the first implementation task it blocks.

3. **Derive observability tasks from OBS** *(if OBS-NNNN was produced)* — Add one entry per OBS instrumentation task as an `observability`-tagged task. Continue ID sequence from Step 2. Set `includes_observability_tasks: true` in frontmatter. **Do not copy the task description from OBS-NNNN** — reference the OBS task ID instead. Implementation spec lives in OBS-NNNN; copying it creates a second source that drifts when OBS is revised. Format:

   ```
   **TN** | observability | [target file from OBS task]
   **Source:** OBS-NNNN IT-N — see OBS-NNNN for full implementation spec.
   **Milestone:** MN
   ```

4. **Derive security tasks from SEC** *(if SEC-NNNN was produced)* — Add one entry per open SEC finding that requires a code change as a `security`-tagged task. Continue ID sequence. Set `includes_security_tasks: true` in frontmatter. **Do not copy the finding or mitigation description from SEC-NNNN** — reference the SEC finding ID instead. Format:

   ```
   **TN** | security | [target file from SEC finding]
   **Source:** SEC-NNNN F-N — see SEC-NNNN for full mitigation spec.
   **Milestone:** MN
   ```

5. **Order tasks by dependency and document as Sequencing & Dependencies.** The correct phase order depends on the project's stack (read ARCH.md), but generally: (1) database schema + migrations, (2) infrastructure setup (db client, auth config), (3) data access layer, (4) API/service layer, (5) UI layer, (6) observability instrumentation, (7) security mitigations, (8) tests. Tasks that skip this order create implementation blockers. Write this ordering as the **Sequencing & Dependencies** section in the PLAN.

6. **Map tasks to acceptance criteria.** Each functional task must reference at least one AC from SPEC-NNNN. Tasks with no AC mapping are either scope creep or indicate a missing AC. Spike tasks map to the AC or design decision they unblock.

7. **Group tasks into milestones.** A milestone corresponds to one PR. Group the ordered, AC-mapped tasks into milestones using vertical slice decomposition. Set `milestone_count` in frontmatter to the final count. Guidelines:
   - **M1 is always the foundation.** Gather all tasks everything else depends on: schema migrations, infrastructure setup, shared types. These do not slice cleanly — they go first as a standalone milestone.
   - **Subsequent milestones are vertical slices.** Each slice delivers end-to-end behavior for one coherent capability: schema + data access + API + UI + tests together — not a horizontal layer.
   - **Observability and security tasks go in the milestone that introduces the code they cover.**
   - **Spike tasks go in the milestone they unblock.**
   - **Every milestone must map to at least one AC.**
   - **Name milestones descriptively.** `M2 — Flag CRUD & Dashboard` not `M2 — PR 2`.

   Output format for each milestone:
   ```
   ### M1 — Foundation: Schema & Infrastructure
   **Tasks:** T1, T2, T3
   **Implementation status:** pending
   **AC Coverage:** AC1 (partial)
   **Depends on:** —
   **Rollback:** Safe — no UI surface; schema change is additive
   ```

8. **Trace the happy path end-to-end.** For each primary User Story in SPEC-NNNN, walk the complete flow step-by-step through the task list — from the first user action to the final observable outcome. For each step of the flow, identify which task(s) cover it. A step with no corresponding task is a gap: either add a missing task or document why no implementation task is needed.

   Document the trace as a **Happy Path Trace** section in the PLAN:

   ```markdown
   ## Happy Path Trace

   ### US1 — [User Story title]

   | Flow Step | Covered by |
   |---|---|
   | User navigates to feature | T5 — page route |
   | User submits form | T7 — POST handler |
   | Result appears in list | T8 — DAL insert + T9 — list component |

   Gap check: none — all steps covered.
   ```

   If a gap is found, add the missing task to the Task Breakdown before proceeding.

9. **Define Definition of Done.** State explicitly which SPEC acceptance criteria are fully covered by the milestone set, and identify any ACs not yet covered (with a note on why they're deferred or covered by testing).

   Format as a table: `| AC | Description | Covered by | Status |`. Copy each AC description **verbatim** from SPEC-NNNN — do not paraphrase, summarize, or shorten. SPEC is the single source of truth for AC text; the PLAN DoD table is a reference copy only. Any divergence between the PLAN description and SPEC text signals a stale PLAN that was not updated after a SPEC revision.

10. **Identify out-of-scope deferrals.** Explicitly list items considered but deferred to future slices, with the Non-Goal or constraint that justifies deferral.

## Output Contract

PLAN-NNNN must contain: Task Breakdown (ordered; functional tasks have source tag, AC reference, and concrete description; observability and security tasks have source tag, target file, and source artifact ID reference — no description copied from OBS/SEC); Milestones (named, each listing tasks, AC coverage, dependencies, rollback note); Sequencing & Dependencies (phase overview); Happy Path Trace (one table per primary User Story); Out of Scope for This Plan; Definition of Done (AC coverage map).

## Quality Gate

Before requesting review:
- [ ] Frontmatter complete: `status: draft`, `version: 1`, `target_design_version`, `milestone_count`, `includes_observability_tasks`, `includes_security_tasks`, `current_review_round: 0`, `generation` block, and `pipeline_status` block all set
- [ ] Every task has an ID; spike tasks use T[N]-spike format
- [ ] Every task can be implemented without making a design decision
- [ ] Schema tasks precede all tasks that depend on the schema
- [ ] Sequencing & Dependencies section present
- [ ] Instrumentation tasks present (if OBS-NNNN was produced)
- [ ] Security mitigation tasks present for all open SEC findings (if SEC-NNNN was produced)
- [ ] Every functional task references at least one AC
- [ ] Every task belongs to exactly one milestone
- [ ] Every milestone maps to at least one AC
- [ ] M1 contains all foundational tasks
- [ ] Happy Path Trace section present; no gaps left unresolved

**Self-check before proceeding:** Re-read each item above. Confirm each is complete, or state what was skipped and why before continuing to Handoff.

## Handoff

Show the user the PLAN file. Do not commit.

> "Here is the draft PLAN. Please review — once you confirm, run `/review-plan PLAN-NNNN SPEC-NNNN DESIGN-NNNN SEC-NNNN` to run all perspectives in one pass."

No commit happens here. The single commit for the entire plan phase is owned by `/approve-plan`.
