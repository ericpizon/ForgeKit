---
id: create-plan
type: command
name: /create-plan
description: Lead Developer command to produce PLAN-NNNN — an ordered, traceable implementation task breakdown from DESIGN, SPEC, OBS, and SEC artifacts; includes happy path trace
status: active
version: 5
primary_role: lead-developer
reviewing_roles:
  - sre
  - developer
  - quality-engineer
output_artifact_type: plan
changelog:
  - v5 — Step 4 now states cross_repo_contract.approved_ref must be copied verbatim from DESIGN's own approved.ref field, not derived from design.ref or any other nearby commit SHA — fixes a real provenance-recording bug caught in customer-api's /review-implementation.
  - v4 — frontmatter initialization now includes `updated`, which address-plan-review.md already assumed existed and updated on every version bump.
  - v3 — added the cross-repo thread path: resolves DESIGN/SEC/OBS from the coordination repo, filters to tasks tagged for this repo, records the inputs block and thread pointer, and tags cross-repo tasks with satisfies_xdeps independently of acceptance_criteria.
---

# /create-plan

## Purpose

Produce an ordered, traceable implementation task breakdown (PLAN-NNNN) from the approved DESIGN, OBS, and SEC artifacts. Every task is tagged by source (functional / observability / security), tied to one or more acceptance criteria or artifact sections, and ordered to respect implementation dependencies. The PLAN is the Developer's primary work artifact.

## When to Invoke

- DESIGN-NNNN `approved`
  If DESIGN-NNNN is `revision_requested`: **do not proceed.** "A design back-cycle is in progress. Wait until DESIGN-NNNN returns to `approved` after the review and approve-design cycle completes."
- OBS-NNNN `draft` or `approved` (instrumentation tasks available)
- SEC-NNNN `completed` (if triggered) — open findings resolved or accepted

Pass artifact IDs as arguments: `/create-plan DESIGN-NNNN SPEC-NNNN OBS-NNNN SEC-NNNN`. For a repo participating in a cross-repo thread, pass the thread and coordination repo instead: `/create-plan THREAD-NNNN org/forge-sdlc`.

## Pre-flight — Thread participation check

This step applies only when this repo participates in a cross-repo thread (invoked with a `THREAD-NNNN` argument, or a prior PLAN in this repo already recorded a `thread` pointer). For a repo with no thread association, skip straight to Required Inputs — nothing changes.

1. Resolve DESIGN-NNNN, SEC-NNNN, and OBS-NNNN from `sdlc/THREAD-NNNN/` in the coordination repo (per [`templates/reference/cross-repo-resolution.md`](../reference/cross-repo-resolution.md)) instead of this repo's own `sdlc/design/`, `sdlc/security/`, `sdlc/observability/` — those directories don't hold this thread's artifacts; the coordination repo does.
2. **Filter, don't transcribe.** DESIGN-NNNN's Cross-Repo Contract and any functional sections apply to every participant, but OBS-NNNN's instrumentation tasks and SEC-NNNN's mitigation tasks were each tagged with an owning `repo` (per `design-observability`/`security-review`'s coordination-repo path) — only derive tasks from entries tagged for this repo. A task tagged for a different participant is not this repo's task, even though this repo can see the whole artifact.
3. Resolve this thread's XDEPs from `sdlc/dependencies/THREAD-NNNN-XDEP-*.yml` in the coordination repo (per [`templates/reference/xdep-schema.md`](../reference/xdep-schema.md)). Keep only the ones where this repo is `provider` or `consumer` — Step 2b below assigns these to tasks.
4. Record in PLAN-NNNN frontmatter: `thread: THREAD-NNNN`, `coordination_repo: org/forge-sdlc` (a lightweight pointer, not a live registry — `project.yml` stays durable config), and an `inputs` block per [`templates/reference/cross-repo-resolution.md`](../reference/cross-repo-resolution.md) covering `design`, `cross_repo_contract` (the approved revision this PLAN was built against), and `sec`/`obs` if applicable. `cross_repo_contract.approved_ref` must be copied **verbatim** from DESIGN-NNNN's own frontmatter `cross_repo_contract.approved.ref` field — it is not `design.ref` (the ref used to fetch DESIGN's content) and not any other nearby commit SHA; see [`templates/reference/cross-repo-resolution.md`](../reference/cross-repo-resolution.md#what-gets-recorded) for the distinction and a worked example.

## Required Inputs

- **DESIGN-NNNN** — approved; read all sections for functional task derivation
- **OBS-NNNN** — read Instrumentation Tasks section; these become `observability`-tagged tasks. For a thread participant, only tasks tagged for this repo (see Pre-flight above)
- **SEC-NNNN** — (if triggered) read Implementation Tasks Derived section; these become `security`-tagged tasks. For a thread participant, only tasks tagged for this repo
- **SPEC-NNNN** — read Acceptance Criteria for the Definition of Done mapping. For a thread participant, also resolved from `sdlc/THREAD-NNNN/SPEC.md` in the coordination repo, not a local file
- **ARCH.md** — read for framework phase ordering, module layout, and implementation patterns

## Task ID Convention

Assign sequential numeric IDs to tasks as they are identified: `T1`, `T2`, `T3`, etc. across all source types (functional, observability, security). IDs are assigned in the order tasks are derived — functional first, then observability, then security — and are stable once assigned. Do not renumber tasks after milestones are defined.

Spike tasks follow the same convention but are prefixed: `T2-spike`. A spike is a timeboxed investigation for an area of technical uncertainty that must be resolved before an implementation task can begin. Format: `T2-spike — [question to answer] — [timebox] — unblocks T3`. Spikes are first-class tasks: they appear in the Task Breakdown, belong to a milestone, and map to the AC or design decision they unblock.

## Steps

1. **Initialize PLAN frontmatter.** Create the PLAN file with:
   - `status: draft`
   - `version: 1`
   - `updated: [today's date]` — `/address-plan-review` updates this on every version bump; it must exist from creation
   - `target_design_version`: version of DESIGN-NNNN being planned against
   - `includes_observability_tasks: false` (update to `true` in Step 3 if OBS-NNNN exists)
   - `includes_security_tasks: false` (update to `true` in Step 4 if SEC-NNNN exists)
   - `milestone_count: TBD` (update in Step 7 after milestones are defined)
   - `current_review_round: 0` (updated by `/review-plan` after each round)
   - `schema_version: 1`
   - `generation` block: `command: /create-plan`, `command_version: [use version from this command's frontmatter]`, `model` [self-identify from system context], `generated_at: [today's date]`, `session_id: null`
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

2. **Derive functional tasks from DESIGN.** For each component or section of the DESIGN (data model, infrastructure, data access layer, API layer, UI layer), identify the concrete implementation tasks. A task is concrete if a Developer can implement it without making a design decision. Assign IDs starting at T1. Read ARCH.md for the canonical phase ordering for this project's stack. If a component involves technical uncertainty, add a spike task immediately before the first implementation task it blocks. **For a thread participant:** a coordination-repo DESIGN's Architecture Overview and Component/Data Model Changes sections may describe several participants' components together — derive tasks only for the components this repo actually implements (unlike OBS/SEC tasks, DESIGN sections carry no explicit per-repo tag; use the DESIGN's own system-boundary language and this repo's ARCH.md to judge which components are this repo's).

2b. **Tag cross-repo tasks with `satisfies_xdeps`** *(thread participants only)* — For each task from Step 2 that implements one of this repo's XDEPs (from Pre-flight Step 3), add a `satisfies_xdeps` field listing the XDEP ID(s) it satisfies:

   ```yaml
   satisfies_xdeps:
     - THREAD-0142-XDEP-01
   ```

   This is independent of, and in addition to, the task's own `acceptance_criteria` — never derive an XDEP's resolution through the ACs it happens to also satisfy (see [`templates/reference/xdep-schema.md`](../reference/xdep-schema.md)). A task can satisfy an XDEP with no AC at all (a pure contract-publishing task with no directly user-facing AC), and a task can satisfy an AC with no XDEP. Every XDEP where this repo is provider or consumer must be satisfied by at least one task — an XDEP with no satisfying task is a planning gap, not a deferred concern.

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

   **Alternate/failure-path trace.** For any primary User Story whose SPEC ACs describe more than one terminal outcome (e.g. a status-bearing entity with failure/alternate states, not just success), trace at least one representative alternate path the same way — from the triggering condition through to the final observable outcome — and identify which task(s) cover each step. This is not exhaustive per-status coverage (the Definition of Done's AC coverage table handles that); it exists to catch the same class of planning gap the Happy Path Trace catches, but for the paths that are easiest to under-plan because they're not the first thing anyone thinks about.

   ```markdown
   ### US1 — [User Story title] — Alternate Path: [failure/alternate condition]

   | Flow Step | Covered by |
   |---|---|
   | [triggering condition] | TN — [task] |
   | [system responds/transitions] | TN — [task] |
   | [user-observable outcome] | TN — [task] |

   Gap check: none — all steps covered.
   ```

   If no primary User Story has more than one terminal outcome, state that explicitly in the PLAN rather than omitting the subsection.

9. **Define Definition of Done.** State explicitly which SPEC acceptance criteria are fully covered by the milestone set, and identify any ACs not yet covered (with a note on why they're deferred or covered by testing).

   Format as a table: `| AC | Description | Covered by | Status |`. Copy each AC description **verbatim** from SPEC-NNNN — do not paraphrase, summarize, or shorten. SPEC is the single source of truth for AC text; the PLAN DoD table is a reference copy only. Any divergence between the PLAN description and SPEC text signals a stale PLAN that was not updated after a SPEC revision.

10. **Identify out-of-scope deferrals.** Explicitly list items considered but deferred to future slices, with the Non-Goal or constraint that justifies deferral.

## Output Contract

PLAN-NNNN must contain: Task Breakdown (ordered; functional tasks have source tag, AC reference, and concrete description; observability and security tasks have source tag, target file, and source artifact ID reference — no description copied from OBS/SEC); Milestones (named, each listing tasks, AC coverage, dependencies, rollback note); Sequencing & Dependencies (phase overview); Happy Path Trace (one table per primary User Story); Alternate/Failure Path Trace (at least one per primary User Story with more than one terminal outcome, or an explicit statement that none apply); Out of Scope for This Plan; Definition of Done (AC coverage map).

For a thread participant, frontmatter also includes `thread`, `coordination_repo`, and an `inputs` block (design/cross_repo_contract/sec/obs, per [`templates/reference/cross-repo-resolution.md`](../reference/cross-repo-resolution.md)); any task satisfying a cross-repo dependency carries `satisfies_xdeps`.

## Quality Gate

Before requesting review:
- [ ] Frontmatter complete: `status: draft`, `schema_version: 1`, `version: 1`, `updated`, `target_design_version`, `milestone_count`, `includes_observability_tasks`, `includes_security_tasks`, `current_review_round: 0`, `generation` block (with `command_version` and `session_id: null`), and `pipeline_status` block all set
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
- [ ] Alternate/Failure Path Trace present for any User Story with more than one terminal outcome, or explicitly stated not applicable
- [ ] *(thread participants only)* Every XDEP where this repo is provider or consumer is satisfied by at least one task's `satisfies_xdeps`
- [ ] *(thread participants only)* `thread`, `coordination_repo`, and `inputs` block set in frontmatter; only tasks tagged for this repo were pulled from OBS/SEC

**Self-check before proceeding:** Re-read each item above. Confirm each is complete, or state what was skipped and why before continuing to Handoff.

## Handoff

Show the user the PLAN file. Do not commit.

> "Here is the draft PLAN. Please review — once you confirm, run `/review-plan PLAN-NNNN SPEC-NNNN DESIGN-NNNN SEC-NNNN` to run all perspectives in one pass."

No commit happens here. The single commit for the entire plan phase is owned by `/approve-plan`.
