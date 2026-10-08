---
id: execute-release
type: command
name: /execute-release
description: SRE command to apply production migrations, deploy to production, and create a release tag — the authorized production deployment step
status: active
version: 4
primary_role: sre
changelog:
  - v4 — added the approval-binding check (candidate_ref/release_plan_revision, per approve-release-plan.md v4) and deployment_progress, written incrementally after Steps 3/4/5 rather than only narrated in Step 6's Deployment Record. Closes a real resumability gap: "re-run /execute-release from Step 3" was previously the only instruction given on migration failure, with no field recording that migrations had already succeeded before a later step — a session interrupted between steps had no way to know it was safe to skip straight ahead rather than re-attempting an already-successful step. Built for the ForgeKit Operator feature (PLAN-0001 T18/T19).
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

**Approval binding check** (per DESIGN-0001 §4/Key Decision 6 — closes a real gap: nothing previously checked whether an approval still applied to the code actually being deployed). Before proceeding past the `status: approved` check above:
- **If `approval.candidate_ref` or `approval.release_plan_revision` is absent entirely** (this RELEASE-PLAN was approved by a version of `/approve-release-plan` predating this binding — a realistic case for any project upgrading its ForgeKit templates after already approving a release): this is not a mismatch, it's an approval that never established binding at all. Stop with a distinct message: "This RELEASE-PLAN was approved before candidate binding existed. Re-run `/approve-release-plan RELEASE-PLAN-vX.Y.Z` to establish `candidate_ref`/`release_plan_revision` before deploying." Do not fall through to the mismatch checks below — comparing an absent field to a real value always "differs," and reporting that as a mismatch would wrongly imply the commit or plan changed when really nothing was ever bound.
- Otherwise, compare `approval.candidate_ref` to the current `HEAD` of the branch this release deploys from. If they differ, stop: "This RELEASE-PLAN's approval was bound to a different commit (`[approval.candidate_ref]`) than what's currently on this branch (`[current HEAD]`). Re-run `/approve-release-plan` against the current commit before deploying — never deploy against a stale approval."
- Compare `approval.release_plan_revision` to this RELEASE-PLAN's own current `version`. If they differ (the plan was edited after approval with no re-approval), stop with the equivalent message naming the stale revision.

**Resume check.** Read `deployment_progress` (initialize it to `{migrations_applied: false, deployed: false, tagged: false, updated_at: null}` if this field doesn't exist yet — a release never attempted before). The first `false` field is exactly where to resume:
- `migrations_applied: false` → start at Step 1. This field being `false` means nothing in this command has succeeded yet — there is no cheaper path than restarting from the top; Steps 1/2 are cheap and idempotent (confirmation, `git checkout`/`pull`), so there's no real cost to re-running them even if they happened moments ago in an interrupted session.
- `migrations_applied: true, deployed: false` → migrations already succeeded; resume at Step 4. Do **not** re-run Step 3 — re-applying an already-successful migration against a database that has already changed is itself a risk this field exists to prevent.
- `deployed: true, tagged: false` → deployment already succeeded; resume at Step 5 only.
- All three `true` → this release already completed; stop and point to `/validate-release` instead of re-deploying.

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
{{stack.deploy.env_pull}}
```

Confirm the production database connection value is the **production** connection string — not the local development value. Then apply migrations:

```bash
{{stack.deploy.migration_command}}
```

Wait for migration output. A successful run ends with all pending migrations listed as applied. If any migration fails:

- **Stop immediately** — do not proceed to Step 4
- Surface the full error output to the user
- Deploying code that depends on schema that does not exist in production will cause a production incident
- Fix the migration and re-run `/execute-release` from Step 3

**Once migrations genuinely succeed** (or are confirmed skipped because `Migration risk: none`), write `deployment_progress.migrations_applied: true` to `RELEASE-PLAN-vX.Y.Z.md` frontmatter immediately — before Step 4 runs, not afterward (per DESIGN-0001 §4/Key Decision 7). This closes a real resumability gap: previously, "re-run `/execute-release` from Step 3" was the only instruction given on migration failure, but nothing recorded whether a *prior* run's migrations had already succeeded — a session interrupted between Step 3 and Step 4 had no field to tell a resumed run it could skip straight to Step 4 rather than re-attempting migrations that already landed.

Note the migration result (migrations applied or skipped) for the Deployment Record in Step 6.

## Step 4 — Deploy to production

```bash
{{stack.deploy.command}}
```

Stream deployment output. Wait for completion and confirm success. If deployment fails, stop immediately — surface the full error output to the user. Do not proceed to Step 5.

**Once deployment genuinely succeeds**, write `deployment_progress.deployed: true` immediately, before Step 5 runs.

## Step 5 — Create release tag

```bash
gh release create vX.Y.Z \
  --title "vX.Y.Z — [feature title from SPEC-NNNN]" \
  --notes "[one-paragraph summary from RELEASE-PLAN scope section]" \
  --latest
```

If not using GitHub, create the equivalent tag in the project's release system.

**Once the tag is genuinely created**, write `deployment_progress.tagged: true`. At this point `deployment_progress` reads `{migrations_applied: true, deployed: true, tagged: true}` — every step this command owns has now actually happened, independent of whatever the Deployment Record in Step 6 goes on to narrate.

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
