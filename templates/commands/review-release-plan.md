---
id: review-release-plan
type: command
name: /review-release-plan
description: Lead Developer + Security + QE inline review of RELEASE-PLAN — scope accuracy, security resolution, and validation readiness; appends review round directly to RELEASE-PLAN
status: active
version: 4
primary_role: lead-developer
reviewing_roles:
  - security
  - quality-engineer
changelog:
  - v4 — added a structured `latest_round` frontmatter field (round/blocking/non_blocking/verdict), written alongside the existing bare `**N blocking, M non-blocking findings.**` prose line every round. Built for the ForgeKit Operator feature (PLAN-0001 T5).
  - v3 — added a coordination-repo path: reads every participant's PLAN/TEST-NNNN, and validates or overrides the proposed deployment/rollback wave ordering per participant/XDEP rather than accepting the derived order automatically.
---

# /review-release-plan

## Purpose

Review the RELEASE-PLAN from three perspectives: Lead Developer (scope accuracy, completeness, rollback viability), Security (all findings resolved or explicitly accepted), and Quality Engineer (validation plan coverage, deferred test case appropriateness, OBS signals, rollback triggers). Appends a `## Release Review — Round N` section directly to `RELEASE-PLAN-vX.Y.Z.md`. Blocking findings must be resolved before `/approve-release-plan`.

## Usage

```
/review-release-plan RELEASE-PLAN-vX.Y.Z
```

## Pre-flight — Coordination repo check

Read `project.yml`'s `repo_role`. This step applies only when it's `coordination` and RELEASE-PLAN-vX.Y.Z-{environment}.md is a thread-wide release; for `repo_role: code`, skip — nothing changes.

**If `repo_role: coordination`:** resolve every participant's PLAN-NNNN and TEST-NNNN, and the shared SEC/OBS findings scoped to this release, via [`templates/reference/cross-repo-resolution.md`](../reference/cross-repo-resolution.md) — the same "read every participant, not one" extension already applied in `create-design`/`create-plan`. Each perspective below runs once per participant where its checks are participant-scoped (e.g. rollback plan viability), and once at the thread level where they're system-scoped (e.g. deployment wave ordering).

**Deployment/rollback wave ordering (new check, thread releases only).** The Lead Developer must explicitly validate or override the Deployment wave order and Rollback plan sections `create-release-plan` derived — the derived order is a *proposal*, never accepted automatically:
- Confirm the wave order actually reflects real `deployment`-kind XDEP constraints, not an artifact of file ordering.
- For the rollback plan's reverse-wave default: for each participant/XDEP, confirm reverse rollback is actually safe (schema changes in particular — a forward-only migration must be named explicitly as an exception, never silently reverse-rolled-back). An unexamined reverse-wave default for a participant with a forward-only migration is a **blocking** finding.
- If the org has real cross-repo deployment orchestration (Argo CD, Spinnaker, Harness, etc.), confirm the derived order was actually translated into that tool's own configuration rather than left as a document nobody consumes — non-blocking if unconfirmed, but worth naming explicitly either way.

## On Re-Review

If `## Release Review — Round 1` already exists:

1. Identify the current round number (latest section + 1)
2. Read the latest `### Address Summary — Round N` to identify what was changed
3. Scope this round to changed sections only
4. Append a new `## Release Review — Round N` section

## Lead Developer Perspective

Adopt the Lead Developer role (`.claude/roles/lead-developer.md`).

- **Scope accuracy:** does the Scope section accurately reflect what was implemented? Cross-reference PLAN-NNNN milestones — no AC should appear in scope if its milestone is not `pr_merged`; no `pr_merged` milestone should be absent from scope
- **Readiness checklist completeness:** all items that can be verified now are checked; deferred items (production validation) are explicitly marked as deferred with the right command
- **Rollback plan viability:** is the rollback plan executable as written? If schema migrations are included, is there explicit rollback SQL or a safe migration reversal path?
- **Risk assessment accuracy:** does the risk level reflect the actual change set — schema changes, first-time deployment, or live traffic?

## Security Perspective

Adopt the Security role (`.claude/roles/security.md`).

- **All findings resolved:** for every finding in SEC-NNNN, confirm `findings_status` is `mitigated` or `accepted_risk` — any `open` finding blocks release regardless of severity
- **Accepted risk documentation:** for any `accepted_risk` finding, confirm the RELEASE-PLAN names a specific approver and a revisit condition — undocumented acceptance is a blocking finding
- **No new security surface in scope:** confirm the scope section does not include capabilities not covered by SEC-NNNN's original assessment; if it does, a new Security Review is required before release
- **Deployment protection actually assessed, not just checked off:** if the Gating criteria table's "Deployment protection confirmed" row is `pass`, confirm the Notes column names the actual mechanism (which platform-protection setting, or which auth layer) — a `pass` with no named mechanism is a **blocking** finding, treated the same as an unresolved SEC finding. If the row is `n/a — still local-only`, confirm that's consistent with the actual deploy target in this release's scope, not stale from a prior release.

## Quality Engineer Perspective

Adopt the Quality Engineer role (`.claude/roles/quality-engineer.md`). Read TEST-NNNN alongside the RELEASE-PLAN.

- **Smoke test coverage:** does the Validation Plan section list smoke test steps that cover all primary User Stories from SPEC-NNNN? A smoke test plan that omits a primary User Story is a **blocking** finding.
- **Deferred test case appropriateness:** for each test case in TEST-NNNN marked `deferred-to-release`, confirm the Validation Plan accounts for it. A P0 deferred test case is a **blocking** finding.
- **OBS signal adequacy:** the Validation Plan must include at least one OBS verification step per primary signal. A plan with no OBS validation step is **non-blocking** on low-risk deployments; **blocking** on first production deployment or High migration risk.
- **Rollback trigger definition:** the Rollback Plan section must name a specific, observable trigger condition — not just "if something breaks." An unobservable trigger is a **non-blocking** finding.
- **Known-issues completeness:** if known issues are listed, each must state an owner and a target resolution date or milestone. An open known issue with no owner is a **non-blocking** finding.
- **Required cross-repo evidence (thread releases only):** for each scenario in the Required cross-repo evidence section, confirm its execution binding and triggers are actually configured (not just declared) — a scenario listed as `required: release` with no real trigger wired up is a **blocking** finding, since `validate-release` will have nothing eligible to consume.

## Review Output

Append to `RELEASE-PLAN-vX.Y.Z.md`:

```markdown
## Release Review — Round N — [date]

### Lead Developer Findings
[findings or "None."]

### Security Findings
[findings or "None."]

### Quality Engineer Findings
[findings or "None."]

### Summary
**N blocking, M non-blocking findings.**

### Next Step
[/address-release-plan-review RELEASE-PLAN-vX.Y.Z | /approve-release-plan RELEASE-PLAN-vX.Y.Z]
```

Also set RELEASE-PLAN-vX.Y.Z.md's frontmatter `latest_round: {round: N, blocking: N, non_blocking: N, verdict: ready | needs_revision}` matching the Summary line exactly (`verdict: ready` when blocking is 0, `needs_revision` otherwise) — the structured counterpart to that prose line, written every round alongside it.

Each finding:

```
**[ROLE-N] blocking | non-blocking — [title]**
[description of the issue]
[resolution recommendation]
```

Nothing is committed here.
