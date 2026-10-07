---
id: design-observability
type: command
name: /design-observability
description: SRE command to produce OBS-NNNN — signals, SLOs, dashboards, alerts, and instrumentation tasks — from an approved DESIGN
status: active
version: 5
primary_role: sre
reviewing_roles:
  - lead-developer
output_artifact_type: observability-plan
changelog:
  - v5 — `repo_role: coordination` now pushes to `main`, not `thread/NNNN-slug` (Option C of a full-pipeline drift audit — see `create-spec.md`'s v5 note for the full reasoning).
  - v4 — added Step 2 ("Confirm the instrumentation mechanism"), reading ARCH.md's stack.observability before defining any signal, and branching to structured-log-field signals when no metrics library exists. Closes a gap where instrumentation tasks assumed a counter/histogram API the stack didn't have, previously only caught one stage downstream at /review-test-plan.
  - v3 — fixed the commit path for the coordination-repo case, which still hardcoded sdlc/observability/ — OBS-NNNN for a thread lives at sdlc/THREAD-NNNN/OBS.md.
  - v2 — added a coordination-repo path: reads ARCH.md from every thread participant, and tags each instrumentation task with the participant repo it belongs to so create-plan can pull only its own subset.
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
- **ARCH.md** — read for the project's observability stack and instrumentation conventions (from `stack.observability`). For a `repo_role: coordination` thread, re-resolve every entry in DESIGN-NNNN's own `inputs.architecture` list (per [`templates/reference/cross-repo-resolution.md`](../reference/cross-repo-resolution.md)) rather than reading a single local ARCH.md — an observability surface assessment scoped to one implicit stack misses whatever the other participants actually expose. Carry the same list forward as an `inputs.architecture` entry in OBS-NNNN frontmatter.

## Steps

1. **Identify operationally critical paths** — From the DESIGN, identify which request paths are highest-traffic and most operationally critical. These get the most detailed instrumentation.
2. **Confirm the instrumentation mechanism** — Read ARCH.md's `stack.observability` field before defining any signal (for a `repo_role: coordination` thread, check it for every participant whose instrumentation tasks this plan will tag). Branch explicitly:
   - **A real metrics library is named** (OpenTelemetry, prom-client, a Datadog/CloudWatch/etc. SDK) — signals are defined as that library's own API (counter, histogram, gauge, span), as in Step 3 below.
   - **No metrics library — structured logging only** (or `stack.observability: none`) — signals are defined as structured log-line emissions with named fields instead (e.g., `log.info({event: "shipment.registered", shipmentId})`), and every later reference to "the signal" in this plan (SLOs, dashboards, alerts, validation checklist) means "this log field, queried from the log stream," not a counter object. Do not write a signal as a counter/histogram/gauge API call unless ARCH.md actually names a library that provides one — the gap this closes surfaced as a blocking Lead Developer finding one stage downstream, at `/review-test-plan`, when a plan assumed a queryable counter the stack didn't have.
3. **Define key signals** — For each critical component, define: metric type (counter, histogram, gauge — or, per Step 2, a structured log field), metric name (namespaced: `service.component.measurement`), label dimensions, instrumentation point in the code (specific file and function), and the failure mode this signal detects.
4. **Address latency edge cases** — Identify any infrastructure characteristics that could skew latency distributions (e.g., cold starts, connection pooling, external service calls). Label anomalous latency separately so it does not contaminate the normal success-path distribution.
5. **Define SLO posture** — If empirical traffic data does not exist yet, do not commit to specific SLO thresholds. Define the SLO *measurement process* instead (what to collect, for how long, how to derive the threshold). Commit to informal baselines that should not trigger alerts but provide a monitoring reference.
6. **Define dashboards** — Name each dashboard, its purpose, and its panels. At minimum: one dashboard for each operationally critical endpoint (request rate, error rate, latency distribution).
7. **Define alerts** — For each alert: condition (metric threshold + time window), severity (critical/warning/informational), likely cause, and runbook summary. Alerts must be actionable — firing an alert that has no clear next step is worse than no alert.
8. **Define instrumentation tasks** — Translate signals into concrete implementation tasks. Each task names the observability API (counter, histogram, span from the stack's OTel or equivalent library — or, per Step 2, a structured log-line emission with named fields), the metric name, the instrumentation point (specific file), and the labels. These tasks are tagged `observability` in the PLAN. For a `repo_role: coordination` thread, each task also names the `repo` it belongs to — a task with no owning participant can't be pulled into any repo's `create-plan`.
9. **Define validation checklist** — For each signal, specify a manual or automated check that confirms it is emitting correctly after deployment. This becomes a gate in RELEASE-PLAN.

## Output Contract

OBS-NNNN must contain all Required Sections per the OBSERVABILITY-PLAN template: Key Signals (each with instrumentation point and failure mode rationale), SLOs/Thresholds (or SLO definition process if pre-baseline), Dashboards, Alerts (with runbook summary), Instrumentation Tasks (tagged for PLAN — each task also tagged with its owning participant `repo` for a `repo_role: coordination` thread). Frontmatter: `validation_status: not_yet_validated`; `target_design_version`; `schema_version: 1`; `generation` block (`command: /design-observability`, `command_version: [use version from this command's frontmatter]`, `model` [self-identify from system context], `generated_at: [today's date]`, `session_id: null`). For a `repo_role: coordination` thread, frontmatter also includes an `inputs.architecture` list (one entry per participant ARCH.md read).

## Lead Developer Approval Gate

After OBS-NNNN is drafted, present it to the Lead Developer for explicit approval before committing. The Lead Developer checks:

**Blocking (must pass before commit):**
- [ ] Every instrumentation task specifies a mechanism that actually exists in ARCH.md's `stack.observability` — a counter/histogram/span API call only if a metrics library is named, a structured log-line emission with named fields otherwise (per Step 2) — plus the metric/span name (or log field name), the file and function, and the label schema. "Add metrics to the endpoint" is not a task, and neither is a counter/histogram call against a stack with no metrics library.
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
- [ ] `schema_version: 1` present in OBS frontmatter
- [ ] `generation` block present: command, command_version, model self-identified, generated_at set, session_id: null
- [ ] *(coordination-repo threads only)* Every instrumentation task is tagged with its owning participant `repo`

## Handoff

**Before committing — pause for human review:**
> Show the user the OBS file and ask: "Please review the observability plan before I commit. Confirm: (1) proceed with the commit? (2) solo project (no push) or team project (push too)?"

**Commit (after user confirms):**

`repo_role: code`:
```
git add sdlc/observability/OBS-NNNN-slug.md sdlc/registry/index.md
git commit -m "OBS-NNNN: observability plan — LD approved"
```

`repo_role: coordination` (thread-scoped, not the flat `sdlc/observability/` a `repo_role: code` project uses):
```
git add sdlc/THREAD-NNNN/OBS.md sdlc/registry/index.md
git commit -m "THREAD-NNNN: observability plan — LD approved"
```

**Push (only if user confirms team context):**

`repo_role: code`:
```
git push origin thread/NNNN-slug
```

`repo_role: coordination` — there is no thread branch; this commit already landed on `main` (per `create-spec.md`'s `repo_role: coordination` Handoff):
```
git push origin main
```

Lead Developer review must complete and findings resolved before OBS-NNNN instrumentation tasks are included in `/create-plan`.
