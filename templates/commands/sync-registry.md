---
id: sync-registry
type: command
name: /sync-registry
description: Scans all SDLC artifacts, groups them by Thread ID, and regenerates the global registry index and per-thread FEATURE dashboards
status: active
version: 1
primary_role: tpo
links:
  - relation: produces
    target: registry/index.md
  - relation: produces
    target: features/FEATURE-NNNN.md
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
