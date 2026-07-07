---
id: design-observability
type: command
name: /design-observability
description: SRE command to produce OBS-NNNN — signals, SLOs, dashboards, alerts, and instrumentation tasks — from an approved DESIGN
status: active
version: 1
primary_role: sre
reviewing_roles:
  - lead-developer
output_artifact_type: observability-plan
---

# /design-observability

## Purpose

Produce an Observability Plan (OBS-NNNN) that defines what "healthy" looks like for a feature before it is implemented. Instrumentation is a code task — it must be designed here so it appears in the PLAN and is built in, not added after. Output gates PLAN-NNNN: instrumentation tasks from this plan become plan tasks.

## When to Invoke

- DESIGN-NNNN is `approved`

This command runs in parallel with `/security-review` (if triggered). Both must complete before `/create-plan`.

Pass the DESIGN artifact ID as argument: `/design-observability DESIGN-NNNN`.

## Required Inputs

- **DESIGN-NNNN** — approved; read the Architecture Overview, Component/Data Model Changes, and Observability Implementation sections
- **ADR(s)** referenced in the DESIGN — read caching/failure-mode decisions that affect what signals are needed
- **SPEC-NNNN** — read Observability Considerations (OC items) as requirements inputs
- **ARCH.md** — read for the project's observability stack and instrumentation conventions (from `stack.observability`)

## Steps

1. **Identify operationally critical paths** — From the DESIGN, identify which request paths are highest-traffic and most operationally critical. These get the most detailed instrumentation.
2. **Define key signals** — For each critical component, define: metric type (counter, histogram, gauge), metric name (namespaced: `service.component.measurement`), label dimensions, instrumentation point in the code (specific file and function), and the failure mode this signal detects.
3. **Address latency edge cases** — Identify any infrastructure characteristics that could skew latency distributions (e.g., cold starts, connection pooling, external service calls). Label anomalous latency separately so it does not contaminate the normal success-path distribution.
4. **Define SLO posture** — If empirical traffic data does not exist yet, do not commit to specific SLO thresholds. Define the SLO *measurement process* instead (what to collect, for how long, how to derive the threshold). Commit to informal baselines that should not trigger alerts but provide a monitoring reference.
5. **Define dashboards** — Name each dashboard, its purpose, and its panels. At minimum: one dashboard for each operationally critical endpoint (request rate, error rate, latency distribution).
6. **Define alerts** — For each alert: condition (metric threshold + time window), severity (critical/warning/informational), likely cause, and runbook summary. Alerts must be actionable — firing an alert that has no clear next step is worse than no alert.
7. **Define instrumentation tasks** — Translate signals into concrete implementation tasks. Each task names the observability API (counter, histogram, span from the stack's OTel or equivalent library), the metric name, the instrumentation point (specific file), and the labels. These tasks are tagged `observability` in the PLAN.
8. **Define validation checklist** — For each signal, specify a manual or automated check that confirms it is emitting correctly after deployment. This becomes a gate in RELEASE-PLAN.

## Output Contract

OBS-NNNN must contain all Required Sections per the OBSERVABILITY-PLAN template: Key Signals (each with instrumentation point and failure mode rationale), SLOs/Thresholds (or SLO definition process if pre-baseline), Dashboards, Alerts (with runbook summary), Instrumentation Tasks (tagged for PLAN). Frontmatter: `validation_status: not_yet_validated`; `target_design_version`; `generation` block (`command: /design-observability`, `model` [self-identify from system context], `generated_at: [today's date]`).

## Lead Developer Approval Gate

After OBS-NNNN is drafted, present it to the Lead Developer for explicit approval before committing. The Lead Developer checks:

**Blocking (must pass before commit):**
- [ ] Every instrumentation task specifies: the exact API call (counter, histogram, span), the metric/span name, the file and function, and the label schema. "Add metrics to the endpoint" is not a task.
- [ ] Every alert has a stated likely cause and a runbook summary — not just a threshold condition
- [ ] Every operationally critical path from the DESIGN has at least one signal
- [ ] Validation checklist entries are testable (specific check, not "verify it works")

**Non-blocking (note and continue):**
- Latency edge cases not yet addressed (flag for PLAN task)
- SLO thresholds not committed (acceptable pre-baseline — mark as informational)

Ask the Lead Developer:
> "Do all instrumentation tasks meet the specificity bar? Confirm to commit OBS-NNNN."

Do not commit until the Lead Developer explicitly confirms.

## Quality Gate

Before committing OBS-NNNN:
- [ ] Lead Developer has explicitly confirmed instrumentation task specificity
- [ ] Every operationally critical path has at least one signal
- [ ] Every alert has a likely cause and runbook summary
- [ ] Instrumentation tasks are specific enough to implement without clarification
- [ ] Validation checklist is present and testable
- [ ] `generation` block present: command, model self-identified, generated_at set

## Handoff

**Before committing — pause for human review:**
> Show the user the OBS file and ask: "Please review the observability plan before I commit. Confirm: (1) proceed with the commit? (2) solo project (no push) or team project (push too)?"

**Commit (after user confirms):**
```
git add sdlc/observability/OBS-NNNN-slug.md sdlc/registry/index.md
git commit -m "OBS-NNNN: observability plan — LD approved"
```

**Push (only if user confirms team context):**
```
git push origin thread/NNNN-slug
```

Lead Developer review must complete and findings resolved before OBS-NNNN instrumentation tasks are included in `/create-plan`.
