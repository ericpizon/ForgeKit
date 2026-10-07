---
id: execute-release
type: command
name: /execute-release
description: SRE command to apply production migrations, deploy to production, and create a release tag — the authorized production deployment step
status: active
version: 3
primary_role: sre
changelog:
  - v3 — Step 6's deployment-record commit now asks solo-vs-team before pushing, like every other commit-producing command in the pipeline. Found during a full-pipeline drift audit: this was the one command that pushed to `main` unconditionally with no question asked at all — inconsistent with every sibling approve-*/create-adr/promote-*/create-investigation command. The production deploy itself (Steps 3–5) is unaffected; only the tracking-record push gained the gate.
  - v2 — added an optional deployment-wave pre-flight check for a repo participating in a cross-repo thread release, for orgs with no existing cross-repo deployment orchestration. This repo's own readiness gate is otherwise unchanged — cross-repo sequencing is never a mandatory second gate on top of an org's own orchestration.
---

# /execute-release

## Purpose

Deploy the approved release to production and create the release tag. This is the command that puts code in front of users. Requires deployment tooling installed and authenticated per the project's SETUP.md.

## Usage

```
/execute-release RELEASE-PLAN-vX.Y.Z
```

For a release that specifically deploys this repo's part of a cross-repo thread's feature (as opposed to this repo's own unrelated release): `/execute-release THREAD-NNNN vX.Y.Z-{environment}`.

## Pre-flight Check

**This repo's own release (default):** read `RELEASE-PLAN-vX.Y.Z.md` frontmatter `status`. Must be `approved`. If not, stop: "RELEASE-PLAN must be `approved` before deployment. Run `/approve-release-plan RELEASE-PLAN-vX.Y.Z` first."

**A cross-repo thread's release:** there is no local `RELEASE-PLAN-vX.Y.Z.md` for a release that only exists as part of the thread — resolve `RELEASE-PLAN-vX.Y.Z-{environment}.md` from the coordination repo instead (per [`templates/reference/cross-repo-resolution.md`](../reference/cross-repo-resolution.md), using this repo's PLAN-NNNN `thread`/`coordination_repo` pointer). Check the top-level `status: approved` (set only once `approvals.overall: true` — every participant *and* the thread-level coordination approval are both in place, per `approve-release-plan`) and this repo's own row in `approvals.participants` is `approved: true` bound to the `candidate_ref` actually being deployed. If either isn't satisfied, stop with the equivalent message pointed at the coordination repo's RELEASE-PLAN.

**Optional wave pre-flight** *(only if this repo's PLAN-NNNN carries a `thread`/`coordination_repo` pointer, and only for orgs with no existing cross-repo deployment orchestration — Argo CD, Spinnaker, Harness, or equivalent already own this sequencing and should never get a second, competing gate from ForgeKit)*: using that pointer, resolve the thread's `RELEASE-PLAN-vX.Y.Z-{environment}.md` from the coordination repo (per [`templates/reference/cross-repo-resolution.md`](../reference/cross-repo-resolution.md)) and check this repo's own Deployment wave order entry. If an earlier wave's participant isn't yet `deployment: complete`, ask before proceeding — this is a manual-coordination aid, not a hard block, since ForgeKit doesn't own execution: "This repo is in Wave [N]; Wave [N-1] participant(s) [list] are not yet deployed. Proceed anyway, or wait?" This check is scoped entirely to reading the thread's own release plan — it never provisions, schedules, or triggers anything in another repo.

## Step 1 — Final deployment confirmation

Present a pre-deployment summary and require explicit confirmation:

> **About to deploy to production.**
>
> Version: vX.Y.Z
> Branch: main (all milestones merged)
>
> Readiness:
> - All milestones `pr_merged` ✓
> - TEST-NNNN `passed` ✓
> - Security findings resolved ✓
> - RELEASE-PLAN `approved` ✓
>
> This will deploy to the public production URL. There is no dry-run — once confirmed, deployment begins.
>
> Confirm to proceed?

Wait for explicit confirmation. Do not run any deployment command without it.

## Step 2 — Ensure on main and up to date

```bash
git checkout main
git pull origin main
```

## Step 3 — Apply production database migrations

Read the **Risk Assessment** section of `RELEASE-PLAN-vX.Y.Z.md`. Check the **Migration risk** line. **For a cross-repo thread release:** the coordination repo's RELEASE-PLAN tracks gate status (Tests/Security/Observability pass/fail) per participant, not migration mechanics — those stay this repo's own concern, exactly as in a local release. Read migration risk from this repo's own PLAN-NNNN and ARCH.md instead.

**If `Migration risk: none` (no schema changes in this release):** skip this step and note "No migrations — skipped" in the Deployment Record.

**If migrations exist (Low or High risk):**

Migrations must be applied **before** deploying code for additive changes (new tables, new nullable columns). For destructive changes, follow the sequence stated in the RELEASE-PLAN's rollback plan — some destructive migrations must land **after** the old code is no longer serving traffic.

Pull production environment variables:

```bash
none
```

Confirm the production database connection value is the **production** connection string — not the local development value. Then apply migrations:

```bash
none
```

Wait for migration output. A successful run ends with all pending migrations listed as applied. If any migration fails:

- **Stop immediately** — do not proceed to Step 4
- Surface the full error output to the user
- Deploying code that depends on schema that does not exist in production will cause a production incident
- Fix the migration and re-run `/execute-release` from Step 3

Note the migration result (migrations applied or skipped) for the Deployment Record in Step 6.

## Step 4 — Deploy to production

```bash
none
```

Stream deployment output. Wait for completion and confirm success. If deployment fails, stop immediately — surface the full error output to the user. Do not proceed to Step 5.

## Step 5 — Create release tag

```bash
gh release create vX.Y.Z \
  --title "vX.Y.Z — [feature title from SPEC-NNNN]" \
  --notes "[one-paragraph summary from RELEASE-PLAN scope section]" \
  --latest
```

If not using GitHub, create the equivalent tag in the project's release system.

## Step 6 — Record the deployment

Update the `## Deployment Record` section in `RELEASE-PLAN-vX.Y.Z.md`:

```markdown
## Deployment Record

- Production URL: [production URL]
- Deployed at: [timestamp]
- Release tag: vX.Y.Z
- Deployed by: /execute-release
- Migrations: [applied — list migration files | none — no schema changes]
```

**If RELEASE-PLAN frontmatter has `linked_bug` set:** update `sdlc/bugs/BUG-NNNN.md` frontmatter:

```yaml
status: pending_verification
```

Commit the deployment record:

```bash
git add sdlc/release-plans/RELEASE-PLAN-vX.Y.Z.md
git add sdlc/bugs/BUG-NNNN.md   # only if linked_bug is set
git commit -m "release(vX.Y.Z): deployed to production"
```

**Ask before pushing** — the deploy itself (Steps 3–5) already happened regardless; this is only the SDLC tracking record:
> "Solo project (commit only, no push) or team project (push too)?"

```bash
# team only:
git push origin main
```

**If this repo participates in a cross-repo thread release:** also update this repo's row in the coordination repo's `RELEASE-PLAN-vX.Y.Z-{environment}.md` (`deployment: complete`, and `deployment_validation` if this repo ran its own smoke check) — that shared record is how other participants and `execute-release`'s optional wave pre-flight know this wave is done. Commit that update in the coordination repo, separately from this repo's own commit above.

## Step 7 — Handoff

> **vX.Y.Z is live in production.**
>
> Production URL: [URL]
> Release tag: vX.Y.Z
>
> Run `/validate-release RELEASE-PLAN-vX.Y.Z TEST-NNNN OBS-NNNN` to verify the release in production.
