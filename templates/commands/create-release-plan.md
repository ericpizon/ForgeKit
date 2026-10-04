---
id: create-release-plan
type: command
name: /create-release-plan
description: SRE command to assess release readiness and author the RELEASE-PLAN — gating criteria, rollback plan, migration steps, and open risk acknowledgment including postmortem action items
status: active
version: 4
primary_role: sre
links:
  - relation: produces
    target: release-plans/RELEASE-PLAN-vX.Y.Z.md
  - relation: precedes
    target: execute-release
changelog:
  - v4 — added Step 3b and a Validation Plan section (both template variants) covering smoke test steps, deferred test case handling, and OBS verification — review-release-plan.md's Quality Engineer Perspective already checked for this section, which this command's own template never produced.
  - v3 — added the coordination-repo thread-release path: authors RELEASE-PLAN-vX.Y.Z-{environment}.md, resolving every participant's PLAN, tracking per-participant vs system-level state, and deriving deployment wave order from deployment-kind XDEPs.
---

# /create-release-plan

## Purpose

Assess readiness for a production release and produce the RELEASE-PLAN. Covers gating criteria (tests, security, observability), migration sequencing, rollback path, and named risks — including open postmortem action items that are surfaced (not blocked). The RELEASE-PLAN must be approved before `/execute-release` proceeds.

## Usage

```
/create-release-plan v1.2.3
```

The version argument becomes the RELEASE-PLAN identifier and is carried into the release notes.

## Pre-flight — Coordination repo check

Read `project.yml`'s `repo_role`. This step applies only when it's `coordination` and the release covers a cross-repo thread; for `repo_role: code` (the default, including one participant's own unrelated releases), skip to Step 1 below — nothing changes. A participant's own local releases are unaffected even while its thread has an in-flight coordination-repo release; only a release that specifically claims the thread's feature goes through this path.

**If `repo_role: coordination`:** this invocation authors a thread-wide release plan, not one repo's release.

1. Ask for the target environment in addition to version and scope: `/create-release-plan THREAD-NNNN v1.2.3 production`. The output file is `sdlc/THREAD-NNNN/RELEASE-PLAN-vX.Y.Z-{environment}.md` — one file per version-and-environment attempt, never mutated; a retried attempt at the same version gets its own suffixed file (e.g. a second production attempt is a separate, later-dated file, not an overwrite of the failed one).
2. Read `sdlc/THREAD-NNNN/thread.yml` for the participant list. Resolve every participant's own PLAN-NNNN and TEST-NNNN, plus this thread's single shared SEC-NNNN and OBS-NNNN (`sdlc/THREAD-NNNN/SEC.md`/`OBS.md`, read once — not per participant), all via [`templates/reference/cross-repo-resolution.md`](../reference/cross-repo-resolution.md) instead of reading local `sdlc/plans/`. For each participant, run the same Step 2 gating criteria assessment below (Tests/Security/Observability/Migrations/Environment/Deployment Protection), scoping the Security/Observability checks to that participant's own tagged findings/tasks within the shared documents, and confirm its scoped milestones are `pr_merged`.
3. **Staleness check (release readiness — the last of the five gate points).** Re-resolve the coordination repo's current `approved` Cross-Repo Contract revision and confirm every participant's PLAN was built against it (`inputs.cross_repo_contract.revision` matches). A participant whose PLAN is stale cannot be included in scope until it's re-derived — do not silently release against a superseded dependency.
4. **Derive deployment wave order.** Build rollout edges from this thread's `deployment`-kind XDEPs (and `contract`-kind XDEPs with a non-`independent` `rollout.ordering`). If a cycle exists among those edges, stop — this should have been caught at DESIGN review time (`create-design`'s Step 9b); a cycle surfacing here means a new deployment-kind XDEP was added without a DESIGN re-review. Where acyclic, derive waves via topological sort (e.g. Wave 1: transformer-service; Wave 2: document-worker, search-indexer). This is always computed and recorded — whether anything *enforces* it is a separate, per-org choice made in `execute-release` (see that command), never assumed here.
5. Track **participant state** (per participant: `implementation`, `deployment`, optional `deployment_validation`) separately from **system state** (`deployment_ready` — every deployment-kind XDEP's ordering constraint is satisfiable; `feature_verification` — pending until `validate-release` runs, including any required cross-repo scenario evidence). `feature_verified` is never a per-participant field — a cross-repo scenario either held across the actual composition of services or it didn't, and no single participant can answer that alone.
6. If the thread has a `SYSTEM-TEST-PLAN.md` with required scenarios, list them by ID under a **Required cross-repo evidence** section — `validate-release` resolves and checks eligibility for each.
7. Write `sdlc/THREAD-NNNN/RELEASE-PLAN-vX.Y.Z-{environment}.md` using the structure below, substituting a `participants` table for the single-repo Gating criteria table, and adding `system` and `Deployment wave order` sections. Frontmatter gains `thread: THREAD-NNNN`, `target_environment`, and an `inputs` block listing every participant PLAN resolved.

Everything from Step 1 onward below describes the single-repo path; a coordination-repo release reuses the same rollback-plan, open-postmortem-item, and confirmation structure but scoped per participant, per the schema in [`templates/reference/cross-repo-resolution.md`](../reference/cross-repo-resolution.md) and this thread's own XDEPs.

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

## Step 3b — Validation plan

Derive, do not re-ask — read the inputs that already exist:

1. **Smoke test steps.** From SPEC-NNNN's User Stories and TEST-NNNN's Happy Path Coverage, list the concrete manual or automated steps that confirm each primary User Story actually works post-deploy. Every primary User Story must have at least one step — a smoke test plan that omits one is a downstream blocking finding in `/review-release-plan`.
2. **Deferred test case handling.** From TEST-NNNN, list every test case marked `deferred-to-release` with its `Deferral Reason` and how this release's validation covers it (a manual check, a monitoring window, or an explicit acceptance of residual risk). A `deferred-to-release` P0 test case with no validation coverage here is a downstream blocking finding.
3. **OBS verification steps.** From OBS-NNNN's Key Signals, list at least one verification step per primary signal (reuse OBS-NNNN's own Validation Checklist where it already covers this). A plan with no OBS verification step is a downstream blocking finding on a first production deployment or a High migration-risk release.

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

## Validation Plan

**Smoke test steps (by primary User Story):**

| User Story | Smoke test step | Result |
|---|---|---|
| [US-N] | [concrete step] | [pass / fail / n/a — pre-deploy] |

**Deferred test case coverage:**

| Test case | Deferral reason | This release's coverage |
|---|---|---|
| [TC-N, P0/P1] | [from TEST-NNNN] | [manual check / monitoring window / accepted risk] |

[If no deferred test cases: "None."]

**OBS verification steps (by primary signal):**

| Signal | Verification step | Result |
|---|---|---|
| [signal name] | [from OBS-NNNN's Validation Checklist, or a new step] | [pass / pending] |

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

## Step 5b — Coordination-repo RELEASE-PLAN structure (coordination-repo threads only)

Write `sdlc/THREAD-NNNN/RELEASE-PLAN-vX.Y.Z-{environment}.md`. Frontmatter as in Step 5, plus `thread: THREAD-NNNN`, `target_environment`, and an `inputs` block (one entry per participant PLAN resolved). Replace the single-repo Gating criteria table with a **per-participant** version of the same gates, plus deployment-orchestration state — losing per-participant visibility into tests/security/observability would be a real regression from the single-repo rigor above, not something the participant/system state split was meant to replace:

```markdown
## Participants

| Repo | Tests | Security | Observability | Implementation | Deployment | Deployment validation |
|---|---|---|---|---|---|---|
| transformer-service | pass | pass | pass | complete | pending | — |
| document-worker | pass | pass | pass | complete | pending | — |
| search-indexer | pass | pass | pass | complete | pending | — |

Tests/Security/Observability columns apply the same criteria as the single-repo Gating criteria assessment (Step 2 above), evaluated per participant — but the source documents aren't symmetric: **Tests** comes from that participant's own local TEST-NNNN, while **Security** and **Observability** come from the *shared*, thread-wide SEC-NNNN/OBS-NNNN in this coordination repo (filtered to the findings/tasks tagged for that participant, per `security-review`/`design-observability`'s per-repo tagging) — there is no separate "that participant's own SEC-NNNN/OBS-NNNN," only one document per thread. Implementation/Deployment/Deployment validation are the deployment-orchestration state each participant's own `execute-release` updates. A participant with any gate not `pass` (or documented accepted-risk, per the single-repo rules) blocks this release the same way it would in a single-repo release — the multi-repo structure adds coordination on top of these gates, it doesn't relax them.

## System

- **Deployment ready:** [true/false — every deployment-kind XDEP's ordering constraint is satisfiable across all participants]
- **Feature verification:** pending — see Required cross-repo evidence below

## Deployment wave order

Derived from this thread's deployment-kind (and non-independent-ordering contract-kind) XDEPs — a documented artifact, not an enforced gate (see `execute-release`):

1. **Wave 1:** transformer-service
2. **Wave 2:** document-worker, search-indexer

## Required cross-repo evidence

*(omit this section if the thread has no SYSTEM-TEST-PLAN.md)*

| Scenario | Required for | Status |
|---|---|---|
| THREAD-NNNN-E2E-01 | release | pending |

## Validation Plan

Same three checks as the single-repo Validation Plan (Step 3b above), applied thread-wide — derive from each participant's own TEST-NNNN and this coordination repo's shared OBS-NNNN, not re-asked:

**Smoke test steps (by primary User Story, across all participants):**

| User Story | Owning repo | Smoke test step | Result |
|---|---|---|---|
| [US-N] | [repo] | [concrete step] | [pass / fail / n/a — pre-deploy] |

**Deferred test case coverage (across all participants):**

| Test case | Owning repo | Deferral reason | This release's coverage |
|---|---|---|---|
| [TC-N, P0/P1] | [repo] | [from that repo's TEST-NNNN] | [manual check / monitoring window / accepted risk] |

[If no deferred test cases in any participant: "None."]

**OBS verification steps (by primary signal, from the shared OBS-NNNN):**

| Signal | Owning repo | Verification step | Result |
|---|---|---|---|
| [signal name] | [repo] | [from OBS-NNNN's Validation Checklist, or a new step] | [pass / pending] |

The Required cross-repo evidence section above covers multi-service E2E scenario verification; this section covers each participant's own smoke/deferred/OBS validation — both are required, neither substitutes for the other.

## Rollback plan

Reverse-wave order is the default proposal — `review-release-plan` validates or overrides it per participant/XDEP; a forward-only migration in an earlier wave is named explicitly as an exception, never silently reverse-rolled-back:

1. **Reverse Wave 2:** document-worker, search-indexer
2. **Reverse Wave 1:** transformer-service — *(exception: forward-only if applicable, state why)*

## Approvals

Per-participant sign-off, bound to the exact candidate and environment, plus a separate thread-level coordination approval — see `/approve-release-plan`. Left blank here; populated on approval.
```

Everything else (Open postmortem action items, Step 6's confirmation) applies the same way, scoped to the thread.
