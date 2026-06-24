---
id: approve-release-plan
type: command
name: /approve-release-plan
description: SRE command to approve the RELEASE-PLAN — zero blocking findings gate, commits the release planning phase, authorizes production deployment via /execute-release
status: active
version: 1
primary_role: sre
---

# /approve-release-plan

## Purpose

Close the release planning phase. Gate: zero blocking findings in the latest review round. Updates the RELEASE-PLAN status to `approved` and commits it. An approved RELEASE-PLAN is the authorization for `/execute-release` to deploy to production.

## Usage

```
/approve-release-plan RELEASE-PLAN-vX.Y.Z
```

## Gate — Zero Blocking Findings Required

Read `RELEASE-PLAN-vX.Y.Z.md`. Find the latest `## Release Review — Round N` section. Check its `**Summary:**` line.

If blocking > 0:
> "Approval blocked: N blocking finding(s) remain in Round X. Run `/address-release-plan-review RELEASE-PLAN-vX.Y.Z` to resolve them."

Do not proceed until the latest round shows `0 blocking`.

## Actions

### 1. Update status

In `RELEASE-PLAN-vX.Y.Z.md` frontmatter:

```yaml
status: approved
```

### 2. Pause for confirmation

Show the user the full RELEASE-PLAN and ask:

> **RELEASE-PLAN-vX.Y.Z is ready to approve.**
>
> Zero blocking findings. All readiness checklist items verified.
>
> Approving this plan authorizes production deployment via `/execute-release`.
>
> Confirm to proceed?

Wait for explicit confirmation before committing.

### 3. Commit

```bash
git add sdlc/release-plans/RELEASE-PLAN-vX.Y.Z.md
git commit -m "release(vX.Y.Z): RELEASE-PLAN approved — production deployment authorized"
```

**Push (ask before pushing):**
> "Solo project (commit only) or team project (push to main too)?"

```bash
# team only:
git push origin main
```

### 4. Handoff

> **Release planning complete — RELEASE-PLAN-vX.Y.Z is approved.**
>
> Run `/execute-release RELEASE-PLAN-vX.Y.Z` when ready to deploy to production.
