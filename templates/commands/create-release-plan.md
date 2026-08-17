---
id: create-release-plan
type: command
name: /create-release-plan
description: SRE command to assess release readiness and author the RELEASE-PLAN — gating criteria, rollback plan, migration steps, and open risk acknowledgment including postmortem action items
status: active
version: 2
primary_role: sre
links:
  - relation: produces
    target: release-plans/RELEASE-PLAN-vX.Y.Z.md
  - relation: precedes
    target: execute-release
---

# /create-release-plan

## Purpose

Assess readiness for a production release and produce the RELEASE-PLAN. Covers gating criteria (tests, security, observability), migration sequencing, rollback path, and named risks — including open postmortem action items that are surfaced (not blocked). The RELEASE-PLAN must be approved before `/execute-release` proceeds.

## Usage

```
/create-release-plan v1.2.3
```

The version argument becomes the RELEASE-PLAN identifier and is carried into the release notes.

## Step 1 — Identify release scope

Ask:

> **What features or milestones are included in this release?**
> (list SPEC-NNNN / PLAN-NNNN IDs, or describe the scope in plain language)
>
> **What is the target release date?** (or "as soon as ready")

For each referenced PLAN-NNNN, read its milestone statuses to confirm all milestones are `pr_merged`.

## Step 2 — Gating criteria assessment

Read ARCH.md for the deployment model. Read all referenced PLAN, TEST, SEC, and OBS artifacts. Assess each gate:

**Tests:**
- [ ] All TEST-NNNN artifacts in scope have `status: passed`
- [ ] No P0 test failures outstanding in any in-scope PLAN milestone
- [ ] Any P1 failures are documented with named approver sign-off in PLAN-NNNN

**Security:**
- [ ] All in-scope SEC-NNNN findings are `mitigated` or explicitly `accepted_risk` with documented rationale
- [ ] No `open` Critical or High findings remain without accepted_risk documentation

**Observability:**
- [ ] OBS-NNNN instrumentation tasks are logged as `implemented` in the observability log
- [ ] Key signals defined in OBS-NNNN are verified to emit in the target environment (or deferred to post-deploy monitoring with explicit note)

**Migrations:**
- [ ] If the release adds or modifies schema: migration has been tested against a production-representative dataset
- [ ] Migration command confirmed: `{{stack.deploy.migration_command}}`
- [ ] Migration is a pre-deploy step (runs before new application code is live)

**Environment:**
- [ ] Required environment variables for this release are documented and confirmed present in production
- [ ] Environment pull verified: `{{stack.deploy.env_pull}}`

**Deployment Protection:**
- [ ] Read ARCH.md's Authentication model. If this release deploys to a publicly reachable environment for the first time, or the deploy target has changed since the last release: platform-level deployment protection (e.g. Vercel preview/production access protection) or application-level authentication is confirmed active. A release that makes previously local-only data reachable from the public internet without this is a **blocking** gate failure, not a note — this is exactly the class of gap that survives every other review because it's a process/deployment fact, not a code change.
- [ ] If ARCH.md's Authentication model states "no auth required" specifically because the deployment is local-only: confirm this release's actual deploy target is still local-only. If the target has changed to anything publicly reachable, that stated assumption is now stale and must be revisited before proceeding — do not let a stale "no auth needed" carry forward silently into a public release.
- [ ] If this environment already has deployment protection from a prior release: confirm it is still active (not disabled during debugging and left off) rather than assuming carryover.

## Step 3 — Rollback plan

Ask:

> **What is the rollback mechanism if the deploy fails or causes a regression?**
> (e.g., redeploy previous version via `vercel --prod --force`, revert the merge commit, re-run prior migration — be specific about commands and who can execute them)
>
> **How long does rollback take?** (estimate)
>
> **Does rolling back require reversing any schema migrations?** (yes / no — if yes, describe the approach)

A vague rollback plan ("revert the PR") is not acceptable. Name the exact command and the person or role who can run it.

## Step 4 — Open postmortem action items (non-blocking)

Scan `sdlc/postmortems/` for PM-NNNN files with action items where `status: open` and `type` is `code-fix` or `feature` and `Linked` is empty (no BUG-NNNN or SPEC created yet).

If any are found, include them as named risks — not a gate:

```
## Open postmortem action items

The following postmortem action items have not yet been converted to pipeline artifacts.
This does not block the release but must be acknowledged by the approver.

| PM | AI | Description | Type | Age |
|---|---|---|---|---|
| PM-NNNN | PM-NNNN-AI-N | [description] | code-fix | [days since PM date] |
```

If none: omit this section.

## Step 5 — Write RELEASE-PLAN

Write `sdlc/release-plans/RELEASE-PLAN-vX.Y.Z.md`:

```markdown
---
id: RELEASE-PLAN-vX.Y.Z
type: release-plan
schema_version: 1
version: "X.Y.Z"
status: draft
created: [today]
target_date: [target date or "as soon as ready"]
scope:
  - SPEC-NNNN
  - PLAN-NNNN
generation:
  command: /create-release-plan
  command_version: "[use version from this command's frontmatter]"
  model: "[self-identify from system context]"
  generated_at: "[today's date]"
  session_id: null
---

# RELEASE-PLAN — vX.Y.Z

## Scope

[List of features and milestones included in this release]

## Gating criteria

| Gate | Status | Notes |
|---|---|---|
| All P0 tests passing | [pass / fail / partial] | [details] |
| P1 failures acknowledged | [pass / n/a] | [approver if applicable] |
| SEC findings resolved or accepted | [pass / fail] | [open findings if any] |
| OBS instrumentation verified | [pass / deferred] | [deferred signals if any] |
| Migration tested | [pass / n/a] | |
| Environment variables confirmed | [pass] | |
| Deployment protection confirmed | [pass / n/a — still local-only / fail] | [mechanism: platform protection / app auth, or why n/a] |

**Overall gate status:** [ready / not ready — reason]

## Migration plan

[If no migrations: "No schema migrations in this release."]

[If migrations:]
1. Pull production environment: `{{stack.deploy.env_pull}}`
2. Run migration (pre-deploy): `{{stack.deploy.migration_command}}`
3. Verify migration completed successfully before deploying application code
4. Deploy: `{{stack.deploy.command}}`

## Rollback plan

**Mechanism:** [exact command or steps]
**Estimated time:** [duration]
**Schema rollback:** [yes — approach / no]
**Who can execute:** [role or names]

## Open items

[Non-blocking items acknowledged for this release:]

| Item | Risk level | Mitigation / Rationale |
|---|---|---|
| [P1 test failure if any] | Medium | [approver sign-off] |
| [Deferred OBS signal] | Low | [will verify post-deploy] |

[If no open items: "None."]

## Open postmortem action items

[Section included only if Step 4 found unlinked items — see Step 4 output]

## Approvals

| Role | Approved | Date |
|---|---|---|
| Lead Developer | | |
| SRE | | |
| Security | *(if SEC findings present)* | |

**Status:** draft → [approved on approval]
```

## Step 6 — Confirm and next steps

> **RELEASE-PLAN-vX.Y.Z created.**
>
> Gate status: [ready / not ready]
>
> [If not ready:] Resolve the failing gates before seeking approval.
> [If open postmortem items:] N unlinked postmortem action item(s) surfaced — acknowledged, not blocking.
>
> Next: share RELEASE-PLAN with Lead Developer and Security for approval signatures.
> Once all approvals are in place, update `status: approved` and run `/execute-release v X.Y.Z`.
