---
id: review-release-plan
type: command
name: /review-release-plan
description: Lead Developer + Security + QE inline review of RELEASE-PLAN — scope accuracy, security resolution, and validation readiness; appends review round directly to RELEASE-PLAN
status: active
version: 1
primary_role: lead-developer
reviewing_roles:
  - security
  - quality-engineer
---

# /review-release-plan

## Purpose

Review the RELEASE-PLAN from three perspectives: Lead Developer (scope accuracy, completeness, rollback viability), Security (all findings resolved or explicitly accepted), and Quality Engineer (validation plan coverage, deferred test case appropriateness, OBS signals, rollback triggers). Appends a `## Release Review — Round N` section directly to `RELEASE-PLAN-vX.Y.Z.md`. Blocking findings must be resolved before `/approve-release-plan`.

## Usage

```
/review-release-plan RELEASE-PLAN-vX.Y.Z
```

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

## Quality Engineer Perspective

Adopt the Quality Engineer role (`.claude/roles/quality-engineer.md`). Read TEST-NNNN alongside the RELEASE-PLAN.

- **Smoke test coverage:** does the Validation Plan section list smoke test steps that cover all primary User Stories from SPEC-NNNN? A smoke test plan that omits a primary User Story is a **blocking** finding.
- **Deferred test case appropriateness:** for each test case in TEST-NNNN marked `deferred-to-release`, confirm the Validation Plan accounts for it. A P0 deferred test case is a **blocking** finding.
- **OBS signal adequacy:** the Validation Plan must include at least one OBS verification step per primary signal. A plan with no OBS validation step is **non-blocking** on low-risk deployments; **blocking** on first production deployment or High migration risk.
- **Rollback trigger definition:** the Rollback Plan section must name a specific, observable trigger condition — not just "if something breaks." An unobservable trigger is a **non-blocking** finding.
- **Known-issues completeness:** if known issues are listed, each must state an owner and a target resolution date or milestone. An open known issue with no owner is a **non-blocking** finding.

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

Each finding:

```
**[ROLE-N] blocking | non-blocking — [title]**
[description of the issue]
[resolution recommendation]
```

Nothing is committed here.
