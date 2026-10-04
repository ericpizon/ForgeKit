---
id: approve-release-plan
type: command
name: /approve-release-plan
description: SRE command to approve the RELEASE-PLAN — zero blocking findings gate, commits the release planning phase, authorizes production deployment via /execute-release
status: active
version: 3
primary_role: sre
changelog:
  - v3 — release_plan_ref now uses the documented self-referential-ref pattern (PENDING_SELF_COMMIT sentinel + immediate follow-up commit) instead of silently expecting the field to contain its own containing commit's SHA.
  - v2 — added the coordination-repo approval model: per-participant sign-off bound to the exact candidate/environment, plus a separate thread-level coordination approval — not one blanket approver and not a pure rollup.
---

# /approve-release-plan

## Purpose

Close the release planning phase. Gate: zero blocking findings in the latest review round. Updates the RELEASE-PLAN status to `approved` and commits it. An approved RELEASE-PLAN is the authorization for `/execute-release` to deploy to production.

## Usage

```
/approve-release-plan RELEASE-PLAN-vX.Y.Z
```

## Gate — Zero Blocking Findings Required

Read `RELEASE-PLAN-vX.Y.Z.md` (or, for a `repo_role: coordination` thread release, `RELEASE-PLAN-vX.Y.Z-{environment}.md` under `sdlc/THREAD-NNNN/`). Find the latest `## Release Review — Round N` section. Check its `**Summary:**` line.

If blocking > 0:
> "Approval blocked: N blocking finding(s) remain in Round X. Run `/address-release-plan-review RELEASE-PLAN-vX.Y.Z` to resolve them."

Do not proceed until the latest round shows `0 blocking`.

## Coordination-repo approval model (thread releases only)

For `repo_role: code`, skip to Actions below — a single SRE approval is the whole gate, as today.

For a `repo_role: coordination` thread release, approval is two-dimensional, not a single blanket sign-off and not a pure rollup over participants — the same convoy problem that keeps one repo's Cross-Repo Contract approval separate from the rest of DESIGN cuts both ways here: one person shouldn't approve readiness for repos they don't own, but deployment-wave ordering, rollback strategy, and cross-repo evidence requirements are feature-wide concerns no participant repo owns either.

```yaml
approvals:
  participants:
    transformer-service:
      approved: true
      candidate_ref: a81f32
      release_plan_ref: PENDING_SELF_COMMIT    # this RELEASE-PLAN-vX.Y.Z-{env}.md's own commit
      environment: production
      approved_by: ...
    document-worker:
      approved: false
      candidate_ref: f92e10
      release_plan_ref: PENDING_SELF_COMMIT
      environment: production

  coordination:
    approved: false               # the shared plan itself — wave ordering, rollback
    release_plan_ref: PENDING_SELF_COMMIT  # strategy, evidence requirements — approved by
    approved_by: ...              # whoever owns coordination for this thread (SRE/Architect)

  overall: false                  # true only once every participant AND coordination are true
```

`release_plan_ref` points at this same approval commit, which doesn't have a SHA yet when the file is written — use the self-referential-ref pattern (see [`templates/reference/cross-repo-resolution.md`](../reference/cross-repo-resolution.md#the-self-referential-ref-pattern)): write `PENDING_SELF_COMMIT` on every row that needs it now, make the approval commit (Action 3 below), then make an immediate one-line follow-up commit replacing every `PENDING_SELF_COMMIT` with that commit's real SHA.

Each participant's own owner approves that row, bound to the exact `candidate_ref`/`release_plan_ref`/`environment` reviewed — the same "approval outlives content" bug already fixed once for Interface Freeze (a bare `approved: true` with no bound candidate says nothing about what was actually reviewed). Changing a participant's candidate invalidates only that participant's row; changing a system-wide property (wave ordering, verification requirements) invalidates the `coordination` row. The `coordination` approver isn't approving another team's code — they're approving the coordination plan itself, the same way `create-release-plan` is already an SRE-authored artifact today.

**Do not proceed to Actions below until `overall: true`** — every participant row `approved: true` for the current `candidate_ref`/`release_plan_ref`/`environment`, and `coordination.approved: true` for the current `release_plan_ref`.

## Actions

### 1. Update status

**`repo_role: code`:** in `RELEASE-PLAN-vX.Y.Z.md` frontmatter:

```yaml
status: approved
approval:
  mode: repo_only
  approved_by: "[SRE name]"
  approved_role: sre
  approved_at: "[today's date]"
  decision_summary: "RELEASE-PLAN-vX.Y.Z approved — production deployment authorized"
  external_event_id: null
  external_event_uri: null
policy_evaluation:
  evaluated: null
  policy_pack: null
  result: null
```

**`repo_role: coordination`:** write the `approvals` block from above into `RELEASE-PLAN-vX.Y.Z-{environment}.md` frontmatter, with every participant row `approved: true` and `coordination.approved: true`, then set `approvals.overall: true` and top-level `status: approved`.

### 2. Pause for confirmation

Show the user the full RELEASE-PLAN and ask:

> **RELEASE-PLAN-vX.Y.Z is ready to approve.**
>
> Zero blocking findings. All readiness checklist items verified.
>
> Approving this plan authorizes production deployment via `/execute-release`.
>
> Confirm to proceed?

**`repo_role: coordination`:** additionally confirm each participant's own owner has actually signed off their row — do not set a participant's `approved: true` on the coordination approver's say-so alone; that defeats the point of splitting the two dimensions.

Wait for explicit confirmation before committing.

### 3. Commit

**`repo_role: code`:**
```bash
git add sdlc/release-plans/RELEASE-PLAN-vX.Y.Z.md
git commit -m "release(vX.Y.Z): RELEASE-PLAN approved — production deployment authorized"
```

**`repo_role: coordination`:**
```bash
git add sdlc/THREAD-NNNN/RELEASE-PLAN-vX.Y.Z-{environment}.md
git commit -m "THREAD-NNNN: RELEASE-PLAN-vX.Y.Z-{environment} approved — production deployment authorized"
```

Then resolve every `PENDING_SELF_COMMIT` sentinel written in Action 1 with the SHA this commit just produced, and make the immediate one-line follow-up commit:
```bash
git add sdlc/THREAD-NNNN/RELEASE-PLAN-vX.Y.Z-{environment}.md
git commit -m "THREAD-NNNN: record RELEASE-PLAN-vX.Y.Z-{environment}'s own approval commit ref"
```

**Push (ask before pushing):**
> "Solo project (commit only) or team project (push to main too)?"

```bash
# team only:
git push origin main
```

### 4. Handoff

**`repo_role: code`:**
> **Release planning complete — RELEASE-PLAN-vX.Y.Z is approved.**
>
> Run `/execute-release RELEASE-PLAN-vX.Y.Z` when ready to deploy to production.

**`repo_role: coordination`:**
> **Release planning complete — RELEASE-PLAN-vX.Y.Z-{environment} is approved for THREAD-NNNN.**
>
> Each participant runs its own `/execute-release` independently, scoped to its own readiness — this coordination-repo approval authorizes the thread-wide plan; it does not deploy anything itself. If this org has no existing cross-repo deployment orchestration, consider the optional wave pre-flight in each participant's `/execute-release`.
