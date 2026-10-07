---
id: ADR-0002
type: adr
schema_version: 1
title: "/status's resolved_state is the single structured state-resolution output; Operator consumes it rather than parsing artifacts independently"
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

# ADR-0002 — `resolved_state` Schema

## Context

SPEC-0001 §7/AC-03 require Operator and `/status` to derive lifecycle state from one shared resolver and never independently disagree. The exact structured schema `/status` should emit alongside its existing human-readable report was an open question (§51, Open Question 1).

## Decision

```yaml
resolved_state:
  stage: <string>                                              # e.g. spec | design | obs | sec | plan | test_plan | implementation | release
  gate: ready | blocked | human_decision_required | ambiguous
  subject: <ARTIFACT-NNNN or THREAD-NNNN>
  blocking_count: <int>
  stale: <bool>
  next_command: "<exact command invocation string>"
  ambiguous_cause: missing_structured_field | unrecognized_state_combination   # present
                    # only when gate: ambiguous (added per a Design Review Round 2
                    # downstream discovery, OBS-0001 S8 — distinguishes the expected,
                    # self-healing pre-existing-artifact case, DESIGN-0001 §3.5, from
                    # a genuine resolver gap worth filing a BUG against)
```

`/status` is extended, not replaced, to emit this alongside its existing output, and to cover the full lifecycle: a new general pre-PLAN form (reading SPEC/ANALYSIS/DESIGN/OBS/SEC frontmatter directly, the same way its existing PLAN-and-later form reads PLAN/TEST/milestone frontmatter), and a corrected milestone branch in its Next Action Inference — closing the gap where it previously had no branch at all for `test_failed`/`tests_passed`/`pr_open`.

`gate` is restricted to exactly these four values. Every stopping condition in SPEC §18's Operator Outcomes enum reduces to one of these four at the gate-resolution level; finer distinctions (which specific outcome name applies) are computed by `/operate` itself via the concrete three-stage procedure in DESIGN-0001 §3.6, and recorded in `OPRUN-NNNN`'s own `status` field, not pushed into `resolved_state`, which only needs to answer "can Operator proceed on its own." (§3.6 was added during Design Review Round 1 — `[LD-2]` — which found this computation stated as existing but not actually specified anywhere.)

## Alternatives Considered

**A separate, Operator-owned artifact parser.** Rejected — SPEC §7 explicitly forbids a duplicate interpretation of engineering state; it would also directly reintroduce the "second implementation of ForgeKit's engineering intelligence" risk the Core Principle rules out.

**A richer gate enum with more than four values** (e.g., one value per Operator Outcome). Rejected — conflates two different questions. `resolved_state.gate` answers "is this a case `/operate` may proceed on autonomously," which only ever has four shapes; "which specific outcome name applies when it can't" is a separate, `/operate`-internal classification that belongs in `OPRUN-NNNN`.

## Consequences

Every future ForgeKit stage/gate must be expressible as one of `{ready, blocked, human_decision_required, ambiguous}` at the `resolved_state` level, or extending this enum requires a new ADR. The §41 structured-field additions (`latest_round`, `implementation_execution`, canonical BUG status, test `classification`, release-approval binding, `deployment_progress`) all exist specifically so `/status` has non-prose data to read when computing this value — `resolved_state` is a derived view over them, never a sixth source of truth.

## Status Rationale

Accepted — directly resolves SPEC-0001 Open Question 1, adopting the schema shape the original Operator gap analysis already validated as sufficient, with the gate enum narrowed to the four values that are actually load-bearing for autonomous-vs-not decisions.

## Revisit Triggers

If a future lifecycle stage produces a stopping condition that genuinely doesn't fit any of the four gate values without losing information `/operate` needs to act correctly.
