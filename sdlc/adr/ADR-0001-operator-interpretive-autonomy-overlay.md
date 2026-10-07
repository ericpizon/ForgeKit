---
id: ADR-0001
type: adr
schema_version: 1
title: "Operator orchestrates via an Interpretive Autonomy Overlay, not per-command Operator-mode branches"
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

# ADR-0001 — Operator Orchestration Mechanism

## Context

`/operate` (SPEC-0001) must autonomously advance work through existing ForgeKit commands, each of which currently pauses for human confirmation before committing routine drafts. Three architectures were considered for how an AI agent following `operate.md`'s own instructions can legitimately skip a sub-command's "ask the user" pause when policy permits it — this is the mechanism ANALYSIS-0001 `[LD-12]` (Round 2) identified as missing, and SPEC-0001 §41/§51 deferred its exact resolution to DESIGN.

## Decision

Option A — Interpretive Autonomy Overlay. `/operate` is one new command. While the agent follows any sub-command's own instructions on Operator's behalf, it applies one general interpretive rule: a Handoff pause matching a formal `/approve-*`/`/reopen-*` gate, or anything named in SPEC-0001 §16/§32, is always a hard stop; any other ("routine draft-commit") pause may be self-answered if the invoking command's corresponding `operator.allow.*` policy key is `true`. The command-to-policy-category mapping is a small, fixed table stated once inside `operate.md` (DESIGN-0001 §3.3). No existing command template is edited.

## Alternatives Considered

**Option B — Operator-aware Handoff mode in every command.** Add an explicit `--operator-run <id>` convention to every command with a confirmation Handoff; each command checks the policy itself and declares its own autonomy behavior. Rejected: a ~50-file edit surface for one feature, directly against the "orchestrates, doesn't replace" framing, and a much larger-scale version of the cross-file drift risk ARCH.md's own Known Tech Debt already documents as real (the solo/team confirmation-phrasing drift).

**Option C — External Operator runtime outside the markdown-instruction model.** A script/process manipulating artifacts and git directly via its own coded logic. Rejected outright: directly contradicts the Core Principle ("must not become a second implementation of ForgeKit's engineering intelligence") and two explicit Non-Goals (no generic workflow-definition language, no Operator-owned engineering-state database); introduces exactly the runtime ARCH.md says this repo doesn't have.

## Consequences

`operate.md` must maintain the categorization table (DESIGN-0001 §3.3) mapping each command to an Autonomy Policy `allow.*` category. A future command added to the pipeline needs a human to remember to add it to this table — unlike Option B, which would force the declaration structurally. `/review-forgekit-pr.md`'s Tier 1 structural checks are extended to verify every command with a confirmation Handoff step appears in this table, catching drift at review time.

## Status Rationale

Accepted — smallest footprint of the three options; directly satisfies SPEC-0001 §3 (Core Principle) and §5 (Non-Goals) with no new invention beyond one command and one small table.

## Revisit Triggers

If `operate.md`'s categorization table drift becomes a recurring real problem (caught by Tier 1 review more than once or twice for the same command), revisit Option B for the specific commands that keep drifting — a partial move is possible without abandoning the overlay for the rest of the pipeline.
