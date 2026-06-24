---
id: generate-release-notes
type: command
name: /generate-release-notes
description: TPO command to produce RELEASE-NOTES-vX.Y.Z and formally mark the feature as delivered — final commit of the release phase
status: active
version: 1
primary_role: tpo
output_artifact_type: release-notes
links:
  - relation: follows
    target: validate-release
---

# /generate-release-notes

## Purpose

Produce the RELEASE-NOTES artifact and formally close the feature thread. Documents what shipped, the deployment record, validation results, and any known gaps. Marks SPEC-NNNN as `shipped`. This is the final commit of the release phase.

## Usage

```
/generate-release-notes RELEASE-PLAN-v1.0.0
```

## Pre-flight Check

Read `RELEASE-PLAN-vX.Y.Z.md` Validation Results section. The smoke test must be `PASS`. If not, stop: "Production validation is not complete — smoke test must pass before signing off the release. Run `/validate-release` first."

## Step 1 — Collect content

Read:
- **RELEASE-PLAN-vX.Y.Z.md** — scope, readiness checklist, deployment record, validation results
- **PLAN-NNNN** — milestone names and AC coverage per milestone
- **SPEC-NNNN** — feature title, problem statement, goals (for human-readable summary)
- **TEST-NNNN** — execution results (any P1+ failures recorded in production round)

## Step 2 — Draft RELEASE-NOTES

Adopt the TPO role (`.claude/roles/tpo.md`).

Produce `sdlc/releases/RELEASE-NOTES-vX.Y.Z.md`:

**Frontmatter:**

```yaml
---
id: RELEASE-NOTES-vX.Y.Z
type: release-notes
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

### Known issues / follow-ups

Any P1+ test failures from the production validation round, deferred manual tests that did not pass, or accepted-risk SEC findings that require a future revisit. If none: "None."

### Deviations from release plan

Any differences between what RELEASE-PLAN-vX.Y.Z defined and what actually happened during deployment or validation. If none: "None."

## Step 3 — Mark SPEC as shipped

Update SPEC-NNNN frontmatter:

```yaml
status: shipped
```

## Step 4 — Pause for review

Show the draft to the user:

> **RELEASE-NOTES-vX.Y.Z is ready.**
>
> [full draft above]
>
> Confirm to commit and close the feature thread?

Wait for confirmation.

## Step 5 — Commit everything

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
