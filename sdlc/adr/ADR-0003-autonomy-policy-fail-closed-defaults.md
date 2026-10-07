---
id: ADR-0003
type: adr
schema_version: 1
title: "Autonomy Policy defaults fail closed; formal-gate vs. routine-confirmation is classified per Handoff step, not per command"
status: accepted
created: 2026-10-07
links:
  - DESIGN-0001
generation:
  command: /create-design
  command_version: 3
  model: claude-sonnet-5
  generated_at: 2026-10-07
  session_id: null
---

# ADR-0003 — Autonomy Policy Defaults and Gate Classification Granularity

## Context

SPEC-0001 §48/AC-25/Safety Invariant #18 require a fail-closed default when no Autonomy Policy is configured. §48 also distinguishes "formal gates" (always a hard stop) from "routine draft-commit confirmations" (autonomy-eligible under policy). Both needed a concrete mechanism before `operate.md`'s categorization table (ADR-0001) could be written correctly.

## Decision

1. **Fail-closed defaults.** Every `project.yml` `operator.allow.*` key defaults to `false`. Absence of the entire `operator:` block, or absence of a specific key within it, both mean "false" — never implicitly `true`. `require_human.risk_acceptance` and `require_human.production_release` are always enforced `true` by `/operate` regardless of what a project's `project.yml` says or omits; a project cannot configure around Safety Invariants #11/#20.

2. **Per-Handoff-step classification, not per-command.** Whether a given pause is a "formal gate" or a "routine confirmation" is decided at the level of the individual Handoff pause, not the whole command. A single command can contain both kinds — `security-review.md`'s routine draft-commit pause (is the SEC draft ready to commit?) is autonomy-eligible under `allow.create_artifacts`, while its `accepted_risk` disposition is always a formal gate per AC-27, regardless of policy. The same split applies to `execute-release.md` (non-prod steps vs. the production-release confirmation) and `promote-to-environment.md` (non-prod vs. `release`-governance promotions).

## Alternatives Considered

**Classifying at the whole-command level.** Rejected — `security-review.md`, `execute-release.md`, and `promote-to-environment.md` each genuinely mix both kinds of pause. A whole-command classification would either over-restrict routine drafting (treating the whole command as a hard gate because of one risk-acceptance step inside it) or under-protect a real risk-acceptance decision (treating the whole command as autonomy-eligible because most of its pauses are routine).

**A fail-open default with an explicit opt-out.** Not a live alternative — ruled out directly by SPEC-0001 AC-25 and Safety Invariant #18.

## Consequences

`operate.md`'s categorization table (ADR-0001, DESIGN-0001 §3.3) must be precise at the per-Handoff-step level for any command that mixes both kinds, not merely per-command — called out explicitly in the table's own notes column for `security-review.md`, `execute-release.md`, and `promote-to-environment.md`, the three commands most likely to need this distinction in practice.

## Status Rationale

Accepted — directly implements constraints SPEC-0001 already states explicitly; this ADR supplies the concrete mechanism, not a new policy decision.

## Revisit Triggers

If a command is found whose pauses resist clean per-step classification even with this finer granularity, revisit the classification model itself rather than forcing that command into either bucket.
