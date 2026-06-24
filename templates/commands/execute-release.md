---
id: execute-release
type: command
name: /execute-release
description: SRE command to apply production migrations, deploy to production, and create a release tag — the authorized production deployment step
status: active
version: 1
primary_role: sre
---

# /execute-release

## Purpose

Deploy the approved release to production and create the release tag. This is the command that puts code in front of users. Requires deployment tooling installed and authenticated per the project's SETUP.md.

## Usage

```
/execute-release RELEASE-PLAN-vX.Y.Z
```

## Pre-flight Check

Read `RELEASE-PLAN-vX.Y.Z.md` frontmatter `status`. Must be `approved`. If not, stop: "RELEASE-PLAN must be `approved` before deployment. Run `/approve-release-plan RELEASE-PLAN-vX.Y.Z` first."

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

Read the **Risk Assessment** section of `RELEASE-PLAN-vX.Y.Z.md`. Check the **Migration risk** line.

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

Note the migration result (migrations applied or skipped) for the Deployment Record in Step 6.

## Step 4 — Deploy to production

```bash
{{stack.deploy.command}}
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
git push origin main
```

## Step 7 — Handoff

> **vX.Y.Z is live in production.**
>
> Production URL: [URL]
> Release tag: vX.Y.Z
>
> Run `/validate-release RELEASE-PLAN-vX.Y.Z TEST-NNNN OBS-NNNN` to verify the release in production.
