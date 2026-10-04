---
id: generate-release-notes
type: command
name: /generate-release-notes
description: TPO command to produce RELEASE-NOTES-vX.Y.Z and formally mark the feature as delivered — final commit of the release phase
status: active
version: 2
primary_role: tpo
output_artifact_type: release-notes
links:
  - relation: follows
    target: validate-release
changelog:
  - v2 — added a coordination-repo path: aggregates every participant's shipped changes into one thread-wide RELEASE-NOTES in the coordination repo, rather than one participant's notes standing in for the whole thread.
---

# /generate-release-notes

## Purpose

Produce the RELEASE-NOTES artifact and formally close the feature thread. Documents what shipped, the deployment record, validation results, and any known gaps. Marks SPEC-NNNN as `shipped`. This is the final commit of the release phase.

## Usage

```
/generate-release-notes RELEASE-PLAN-v1.0.0
```

For a `repo_role: coordination` thread release: `/generate-release-notes THREAD-NNNN RELEASE-PLAN-v1.0.0-production`.

## Pre-flight Check

Read `RELEASE-PLAN-vX.Y.Z.md` Validation Results section. The smoke test must be `PASS`. If not, stop: "Production validation is not complete — smoke test must pass before signing off the release. Run `/validate-release` first."

**`repo_role: coordination` (thread releases only):** read `sdlc/THREAD-NNNN/RELEASE-PLAN-vX.Y.Z-{environment}.md`'s Validation Results and Participants table instead. `system.feature_verification` must be `passed` (every required cross-repo scenario has eligible evidence, per `/validate-release` Track 4) and every participant's own `deployment_validation` must be `passed` (propagated there by each participant's own `/validate-release` Tracks 1–3 — no need to resolve each participant's local RELEASE-PLAN separately). If either is missing, stop with the equivalent message, naming which participant or which required scenario is not yet validated.

## Step 1 — Collect content

**`repo_role: code`:** read:
- **RELEASE-PLAN-vX.Y.Z.md** — scope, readiness checklist, deployment record, validation results
- **PLAN-NNNN** — milestone names and AC coverage per milestone
- **SPEC-NNNN** — feature title, problem statement, goals (for human-readable summary)
- **TEST-NNNN** — execution results (any P1+ failures recorded in production round)

**`repo_role: coordination`:** read `sdlc/THREAD-NNNN/RELEASE-PLAN-vX.Y.Z-{environment}.md` (scope, participants table, deployment record, validation results including cross-repo scenario evidence), `sdlc/THREAD-NNNN/SPEC.md`, and `sdlc/THREAD-NNNN/OBS.md` (a single shared document for the whole thread — read it once, not per participant), then resolve every participant's own PLAN-NNNN and TEST-NNNN (per [`templates/reference/cross-repo-resolution.md`](../reference/cross-repo-resolution.md)) — the release notes aggregate every participant's changes, not one participant's notes standing in for the whole thread.

## Step 2 — Draft RELEASE-NOTES

Adopt the TPO role (`.claude/roles/tpo.md`).

Produce `sdlc/releases/RELEASE-NOTES-vX.Y.Z.md`:

**Frontmatter:**

```yaml
---
id: RELEASE-NOTES-vX.Y.Z
type: release-notes
schema_version: 1
title: "[Feature title] — vX.Y.Z"
released_at: [timestamp from RELEASE-PLAN Deployment Record]
target_version: vX.Y.Z
deployment_url: [from RELEASE-PLAN]
deviations_from_plan: false
links:
  - relation: documents
    target: RELEASE-PLAN-vX.Y.Z
  - relation: delivers
    target: SPEC-NNNN
generation:
  command: /generate-release-notes
  command_version: "[use version from this command's frontmatter]"
  model: "[self-identify from system context]"
  generated_at: "[today's date]"
  session_id: null
---
```

**Required sections:**

### Summary

One paragraph: what this release delivers, who it's for, what they can now do. Derived from SPEC-NNNN's problem statement and goals. Human-readable — not a task list, not a milestone list.

### What shipped

Per milestone — brief, human-readable description of the capability each milestone contributed:

```markdown
**M1 — [Milestone name]**
[2–3 sentences describing the user-visible or system capability this milestone delivered]

**M2 — [Milestone name]**
...
```

**`repo_role: coordination`:** group by participant repo first, then by that participant's own milestones within it — a reader needs to know which repo delivered what, not just a flattened milestone list with no ownership:

```markdown
### transformer-service
**M1 — [Milestone name]**
...

### document-worker
**M1 — [Milestone name]**
...
```

### Acceptance criteria delivered

| AC | Description |
|---|---|
| AC-1 | [text from SPEC-NNNN] |
| AC-2 | ... |

### Deployment

```markdown
- **Version:** vX.Y.Z
- **Deployed at:** [timestamp]
- **Production URL:** [URL]
- **GitHub Release:** [URL]
```

**`repo_role: coordination`:** one row per participant, plus the system-level verification result:

```markdown
| Repo | Deployed at | Production URL | GitHub Release |
|---|---|---|---|
| transformer-service | [timestamp] | [URL] | [URL] |
| document-worker | [timestamp] | [URL] | [URL] |

**Feature verification (cross-repo):** passed — [N] required scenario(s) eligible
```

### Known issues / follow-ups

Any P1+ test failures from the production validation round, deferred manual tests that did not pass, or accepted-risk SEC findings that require a future revisit. If none: "None."

### Deviations from release plan

Any differences between what RELEASE-PLAN-vX.Y.Z defined and what actually happened during deployment or validation. If none: "None."

## Step 3 — Mark SPEC as shipped

**`repo_role: code`:** update SPEC-NNNN frontmatter:

```yaml
status: shipped
```

**`repo_role: coordination`:** update `sdlc/THREAD-NNNN/SPEC.md` frontmatter the same way — there is one SPEC per thread, not one per participant, so this is a single update regardless of how many repos participated.

## Step 4 — Pause for review

Show the draft to the user:

> **RELEASE-NOTES-vX.Y.Z is ready.**
>
> [full draft above]
>
> Confirm to commit and close the feature thread?

Wait for confirmation.

## Step 5 — Commit everything

**`repo_role: code`:**
```bash
git add sdlc/releases/RELEASE-NOTES-vX.Y.Z.md
git add sdlc/specs/SPEC-NNNN-[slug].md
git add sdlc/observability/OBS-NNNN-[slug].md
git add sdlc/release-plans/RELEASE-PLAN-vX.Y.Z.md
git add sdlc/test-plans/TEST-NNNN-[slug].md
git add sdlc/registry/index.md
git commit -m "release(vX.Y.Z): RELEASE-NOTES published — [feature title] delivered"
git push origin main
```

**`repo_role: coordination`:**
```bash
git add sdlc/releases/RELEASE-NOTES-vX.Y.Z.md
git add sdlc/THREAD-NNNN/SPEC.md
git add sdlc/THREAD-NNNN/RELEASE-PLAN-vX.Y.Z-{environment}.md
git add sdlc/THREAD-NNNN/OBS.md            # only if updated (e.g. Instrumentation Implementation Log entries)
git add sdlc/registry/index.md
git commit -m "THREAD-NNNN: RELEASE-NOTES published — [feature title] delivered"
git push origin main
```
OBS-NNNN is a thread-wide shared artifact (one document, tasks tagged per participant) — it lives here, in the coordination repo, not in any one participant's own `sdlc/observability/`. TEST-NNNN, by contrast, really is per-participant (each repo has its own) and stays in that participant's repo, not part of this commit.

## Step 6 — Final confirmation

> **[Feature title] — vX.Y.Z is delivered.**
>
> Production URL: [URL]
> GitHub Release: [URL]
> SPEC-NNNN: shipped
>
> The feature thread is complete.
>
> Operations phase: monitor production via OBS-NNNN signals.
> If an incident occurs: `/create-incident`
> If a postmortem action item requires a new spec: `/create-spec`
