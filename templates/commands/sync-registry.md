---
id: sync-registry
type: command
name: /sync-registry
description: Scans all SDLC artifacts, groups them by Thread ID, and regenerates the global registry index and per-thread FEATURE dashboards
status: active
version: 3
primary_role: tpo
links:
  - relation: produces
    target: registry/index.md
  - relation: produces
    target: features/FEATURE-NNNN.md
changelog:
  - v3 — added `sdlc/investigations/` to the single-repo scan table (INV-NNNN), and `sdlc/investigations/`/`sdlc/adr/`/`sdlc/promotions/` to the coordination-repo path. Found during a full-pipeline drift audit: these artifact types were completely invisible to the registry despite `forge-init.md` explicitly creating directories for them in both repo roles.
  - v2 — added a coordination-repo path producing a thread-level portfolio view across every participant, and a one-line thread pointer for a participant repo's own local registry.
---

# /sync-registry

## Purpose

TPO command. Regenerate the feature registry from the current state of all SDLC artifacts. Produces two outputs:

1. **`sdlc/registry/index.md`** — flat global index of all feature threads: stage, status, and last updated at a glance
2. **`sdlc/features/FEATURE-NNNN.md`** — per-thread dashboard: artifact index, lifecycle progress, milestone status, open items

This is a cross-cutting utility — it does not belong to a single pipeline phase. Run it after any major status transition to keep the registry current:

- After `/close-milestone` — a milestone moved to `pr_merged`; implementation progress updated
- After `/generate-release-notes` — a feature shipped; thread closes out
- After `/create-spec` — a new thread starts; registry gains a new entry
- Before starting a new feature — TPO checks what is currently in flight before committing new scope

FEATURE files are always regenerated from source — never hand-edited.

## Usage

```
/sync-registry
```

No arguments. Scans the entire `sdlc/` directory.

## Pre-flight — Coordination repo check

Read `project.yml`'s `repo_role`. For `repo_role: code` (the default), continue to Step 1 below with one addition: **if a thread pointer exists** (any PLAN in this repo has a `thread`/`coordination_repo` frontmatter pair), add a one-line pointer to each such thread's entry in `sdlc/registry/index.md` — thread ID and coordination repo — rather than attempting to reproduce that thread's full cross-repo picture locally. That stays the coordination repo's job.

**If `repo_role: coordination`:** this invocation produces a thread-level portfolio view across every participant — the value this command's design always pointed at — instead of scanning a flat `sdlc/specs/`, `sdlc/design/`, etc. layout that a coordination repo doesn't have.

1. Scan `sdlc/THREAD-NNNN/` for each thread directory: read `thread.yml` (participants), `SPEC.md`, `DESIGN.md` (including its `cross_repo_contract` working/approved block), `SECURITY-REVIEW.md`/`OBSERVABILITY-PLAN.md` if present, any `RELEASE-PLAN-vX.Y.Z-{environment}.md` files, `SYSTEM-TEST-PLAN.md` if present, and any thread-scoped `adr/`.
2. Scan the top-level `sdlc/dependencies/` and `sdlc/scenarios/` directories, associating each `THREAD-NNNN-XDEP-NN.yml` / `THREAD-NNNN-E2E-NN.yml` file with its thread by the `THREAD-NNNN` ID prefix in the filename.
3. For each participant in `thread.yml`, resolve its PLAN-NNNN frontmatter (`pipeline_status`, milestone implementation statuses) via [`templates/reference/cross-repo-resolution.md`](../reference/cross-repo-resolution.md) — a lightweight read, frontmatter and milestone status lines only, the same discipline `/status` already applies locally.
4. Also scan the top-level `sdlc/investigations/` and `sdlc/adr/` directories — standalone ADRs and Investigations not tied to any one thread (per `forge-init.md`'s coordination-repo directory list) — and list them in the registry index as their own flat section, not folded into any thread's entry. Also scan `sdlc/promotions/`: for each `{environment}.md` file, surface its most recent row (candidate refs, promoted by, promoted at, smoke-test result) as a one-line "last promotion" summary per environment — this is a log to summarize, not an artifact to index by ID the way SPEC/DESIGN/etc. are.
5. Skip Steps 1–3 below (they describe the flat single-repo scan and stage-precedence table) and proceed to the coordination-repo output format (below) instead of the single-repo FEATURE-NNNN.md format.

## Step 1 — Scan artifacts

Read the frontmatter of every artifact file in these directories:

| Directory | Artifact type | ID format |
|---|---|---|
| `sdlc/specs/` | SPEC | `SPEC-NNNN` |
| `sdlc/analysis/` | ANALYSIS | `ANALYSIS-NNNN` |
| `sdlc/design/` | DESIGN | `DESIGN-NNNN` |
| `sdlc/adr/` | ADR | `ADR-NNNN` (independent numbering) |
| `sdlc/observability/` | OBSERVABILITY-PLAN | `OBS-NNNN` |
| `sdlc/security/` | SECURITY-REVIEW | `SEC-NNNN` |
| `sdlc/plans/` | PLAN | `PLAN-NNNN` |
| `sdlc/test-plans/` | TEST-PLAN | `TEST-NNNN` |
| `sdlc/reviews/` | IMPL-REVIEW | `IMPL-REVIEW-NNNN-MN` |
| `sdlc/release-plans/` | RELEASE-PLAN | `RELEASE-PLAN-vX.Y.Z` |
| `sdlc/releases/` | RELEASE-NOTES | `RELEASE-NOTES-vX.Y.Z` |
| `sdlc/incidents/` | INCIDENT | `INC-NNNN` (independent) |
| `sdlc/postmortems/` | POSTMORTEM | `PM-NNNN` (independent) |
| `sdlc/investigations/` | INVESTIGATION | `INV-NNNN` (independent — not necessarily tied to any thread; `decision: adr` investigations link to their `linked_adr` via `links`) |

For each file, collect: `id`, `type`, `status`, `updated`, and any `links` entries.

## Step 2 — Group into threads

Group artifacts by shared Thread ID (the NNNN number common to SPEC, ANALYSIS, DESIGN, PLAN, TEST, OBS, SEC, and IMPL-REVIEW for the same feature).

Cross-cutting artifacts (ADRs, INC, PM, RELEASE-PLAN, RELEASE-NOTES) are associated with threads via `links` in their frontmatter (`relates_to`, `delivers`, `documents`).

Each thread's artifact set:

```
Thread NNNN:
  core chain:       SPEC-NNNN, ANALYSIS-NNNN, DESIGN-NNNN, PLAN-NNNN, TEST-NNNN
  cross-cutting:    OBS-NNNN, SEC-NNNN (if present)
  implementation:   IMPL-REVIEW-NNNN-M1 … IMPL-REVIEW-NNNN-MN
  release:          RELEASE-PLAN-vX.Y.Z, RELEASE-NOTES-vX.Y.Z (linked via `delivers: SPEC-NNNN`)
  operational:      INC-NNNN, PM-NNNN (linked via `relates_to`)
  adrs:             ADR-NNNN (linked via `referenced_by` or `relates_to`)
```

## Step 3 — Determine stage and status per thread

For each thread, read artifact statuses and apply this precedence (highest complete stage wins):

| Condition | Stage | Status label |
|---|---|---|
| RELEASE-NOTES exists | 10 — Released | `shipped` |
| RELEASE-PLAN `approved` + deployment record populated | 10 — Releasing | `deploying` |
| RELEASE-PLAN `approved` | 9 — Release Planning | `release approved` |
| RELEASE-PLAN `draft` | 9 — Release Planning | `release planning` |
| All PLAN milestones `pr_merged` and TEST `passed` | 8 — Ready for Release | `ready for release` |
| Any PLAN milestone `pr_open` | 7 — Testing/PR | `PR open — MN` |
| Any PLAN milestone `tests_passed` | 7 — Testing | `testing — MN` |
| Any PLAN milestone `test_failed` | 7 — Testing | `test failures — MN` |
| Any PLAN milestone `approved` | 7 — Testing | `awaiting QE — MN` |
| Any PLAN milestone `in_progress` | 6 — Implementation | `implementing — MN` |
| TEST `approved` and PLAN `approved` | 6 — Ready for Implementation | `ready for implementation` |
| PLAN `approved` | 5b — Test Planning | `test planning` |
| PLAN `draft` | 5 — Implementation Planning | `plan in progress` |
| DESIGN `approved` | 5 — Ready for Planning | `ready for planning` |
| DESIGN `draft` or DESIGN `needs_revision` | 3 — Architecture & Design | `design in progress` |
| SPEC `approved` and ANALYSIS `completed` | 3 — Ready for Design | `ready for design` |
| SPEC `approved` | 1 — Specification | `spec approved` |
| SPEC `draft` or `needs_revision` | 1 — Specification | `spec in progress` |

For milestones — when the status references "MN", identify the active milestone (the lowest-numbered milestone that is not yet `pr_merged`).

## Step 4 — Generate FEATURE-NNNN.md files

For each thread, write `sdlc/features/FEATURE-NNNN.md` (overwrite if it exists — this file is always regenerated):

```markdown
---
id: FEATURE-NNNN
type: feature
thread_id: "NNNN"
rollup_generated_at: [ISO timestamp]
---

# Feature NNNN — [title from SPEC-NNNN]

## Thread Overview

**Summary:** [one-line problem statement from SPEC-NNNN]
**Stage:** [current stage label]
**Status:** [status label]

## Lifecycle Progress

- [x] Stage 1 — Specification (SPEC-NNNN: [status])
- [x] Stage 2 — Specification Analysis (ANALYSIS-NNNN: [status])
- [x] Stage 3 — Architecture & Design (DESIGN-NNNN: [status])
- [x] Stage 4a — Observability Design (OBS-NNNN: [status] — or N/A)
- [x] Stage 4b — Security Review (SEC-NNNN: [status] — or N/A)
- [x] Stage 5 — Implementation Planning (PLAN-NNNN: [status])
- [x] Stage 5b — Test Planning (TEST-NNNN: [status])
- [ ] Stage 6/7 — Implementation & Testing (milestones: see below)
- [ ] Stage 9 — Release Planning
- [ ] Stage 10 — Release

[Use [x] for completed stages, [ ] for incomplete, based on artifact statuses]

## Artifact Index

| Artifact | Status | Updated |
|---|---|---|
| SPEC-NNNN | [status] | [updated date] |
| ANALYSIS-NNNN | [status] | [updated date] |
| DESIGN-NNNN | [status] | [updated date] |
| OBS-NNNN | [status] | [updated date] — or N/A |
| SEC-NNNN | [status] | [updated date] — or N/A |
| PLAN-NNNN | [status] | [updated date] |
| TEST-NNNN | [status] | [updated date] |
| RELEASE-PLAN-vX.Y.Z | [status] | [updated date] — or not yet created |
| RELEASE-NOTES-vX.Y.Z | — | [updated date] — or not yet created |

## Implementation Status

| Milestone | Name | Status |
|---|---|---|
| M1 | [name from PLAN-NNNN] | [implementation_status] |
| M2 | [name from PLAN-NNNN] | [implementation_status] |
| MN | [name from PLAN-NNNN] | [implementation_status] |

[Only present if PLAN-NNNN exists and has milestones]

## Linked Operational Artifacts

| Artifact | Type | Summary |
|---|---|---|
| INC-NNNN | Incident | [title] |
| PM-NNNN | Postmortem | [title] |
| ADR-NNNN | ADR | [title] |

[Only list artifacts that explicitly link to this thread. If none: "None."]

## Open Items

[List any of the following if present:]
- Open SEC findings (status `open` in SEC-NNNN)
- Active incidents linked to this thread
- Postmortem action items not yet converted to artifacts (`status: pending` in PM-NNNN action_items)

[If none: "None."]
```

## Step 5 — Generate sdlc/registry/index.md

Read `project.yml` for the project name. Write `sdlc/registry/index.md` (overwrite entirely — always regenerated):

```markdown
# [project.name] — Feature Registry

**Last synced:** [ISO timestamp]
**Threads:** N

| Thread | Feature | Stage | Status | Updated |
|---|---|---|---|---|
| [0001](../features/FEATURE-0001.md) | [title from SPEC-0001] | [stage] | [status] | [date] |
| [0002](../features/FEATURE-0002.md) | [title from SPEC-0002] | [stage] | [status] | [date] |

---

## Open postmortem action items

[Scan all PM-NNNN files in sdlc/postmortems/ for action items with status: open.
Include only code-fix and feature type items here — observability/infrastructure/process
items are owned entirely within their PM and do not surface in the registry.]

[If any open code-fix or feature items exist:]

| PM | AI | Description | Type | Owner | Linked artifact |
|---|---|---|---|---|---|
| [PM-NNNN](../postmortems/PM-NNNN-[slug].md) | PM-NNNN-AI-N | [description] | [type] | [owner or —] | [BUG-NNNN or —] |

[If none: "None."]

---

*Generated by `/sync-registry`. Do not edit manually — run `/sync-registry` to refresh.*
```

Sort threads by Thread ID ascending. Link each thread ID to its FEATURE-NNNN.md file.

For the open postmortem action items section: scan every file in `sdlc/postmortems/`, read each `## Action items` table, and collect rows where `Status` is `open` and `Type` is `code-fix` or `feature`. Include the PM link, AI ID, description, type, owner, and linked artifact (or `—` if unlinked). Sort by PM ID ascending.

## Step 4b/5b — Coordination-repo output (coordination-repo threads only)

For each thread found in Pre-flight, write `sdlc/features/FEATURE-THREAD-NNNN.md` instead of the single-repo FEATURE-NNNN.md format:

```markdown
---
id: FEATURE-THREAD-NNNN
type: feature
thread_id: "THREAD-NNNN"
rollup_generated_at: [ISO timestamp]
---

# Thread NNNN — [title from SPEC.md]

## Participants

| Repo | Implementation | Deployment (latest release) |
|---|---|---|
| transformer-service | [pipeline_status.plan / milestone rollup] | [from latest RELEASE-PLAN, if any] |
| document-worker | [pipeline_status.plan / milestone rollup] | [from latest RELEASE-PLAN, if any] |

## Cross-Repo Contract

**Working:** rev [N] ([draft/in_review/approved])   **Approved:** rev [M]

| XDEP | Provider | Consumer | Status |
|---|---|---|---|
| THREAD-NNNN-XDEP-01 | transformer-service | document-worker | active |

## System test coverage

*(omit if the thread has no SYSTEM-TEST-PLAN.md)*

| Scenario | Covers | Status |
|---|---|---|
| THREAD-NNNN-E2E-01 | XDEP-01, XDEP-02 | active |

## Releases

| Version — environment | Status | Deployment ready | Feature verification |
|---|---|---|---|
| v1.2.3 — production | approved | true | passed |

[If no RELEASE-PLAN exists yet: "No releases yet."]
```

Then write `sdlc/registry/index.md` with a thread-level portfolio table in place of the single-repo Feature Registry:

```markdown
# [project.name] — Cross-Repo Thread Registry

**Last synced:** [ISO timestamp]
**Threads:** N

| Thread | Feature | Participants | Contract | Stage | Updated |
|---|---|---|---|---|---|
| [THREAD-0142](../features/FEATURE-THREAD-0142.md) | [title] | transformer-service, document-worker | approved rev 4 | design in progress | [date] |

---

*Generated by `/sync-registry`. Do not edit manually — run `/sync-registry` to refresh.*
```

A participant's PLAN that could not be resolved (repo unreachable, network/auth issue) is shown as `unresolved` in its Implementation column rather than silently omitted — a portfolio view that quietly drops a participant it couldn't reach is worse than one that shows the gap.

**Stale hand-off check.** If SPEC.md or DESIGN.md has a `## Discovery` section recording a participant as `handed off — pending confirmation` (per `reopen-spec`/`reopen-design`), cross-check that participant's actual resolved `pipeline_status` — if it doesn't yet show the expected `revision_requested`, the hand-off was never carried out. Surface this explicitly in the thread's FEATURE-THREAD-NNNN.md (a **Pending hand-offs** section) rather than letting a stale hand-off go unnoticed simply because the session that created it ended.

## Step 6 — Show and confirm

Present a summary to the user:

> **Registry sync complete.**
>
> Threads found: N
> FEATURE files generated: N
> Registry index updated: sdlc/registry/index.md
>
> [Table preview of sdlc/registry/index.md]
>
> Confirm to commit?

Wait for confirmation.

## Step 7 — Commit

```bash
git add sdlc/registry/index.md
git add sdlc/features/
git commit -m "chore: sync feature registry — N thread(s)"
git push origin main
```

## Notes

- Run `/sync-registry` any time the registry feels out of date — after closing a milestone, approving a release plan, shipping a feature, or linking an incident
- FEATURE files are always overwritten — never edit them manually; any manual edits will be lost on the next sync
- Cross-cutting artifacts (ADRs, incidents, postmortems) appear in the Open Items and Linked Operational Artifacts sections of the thread they relate to; they do not get their own FEATURE entry
- If `sdlc/registry/` or `sdlc/features/` directories do not exist, create them before writing files
