---
id: ADR-0003
type: adr
schema_version: 1
status: accepted
target_design: DESIGN-0001
created: 2026-10-05
---

# ADR-0003 — Investigation Is a New Standalone Artifact Type and Command, Not an Extension

## Context

SPEC-0001 (G5) requires support for spike/research work that may conclude without any implementation. Left open (OQ4): does this reuse an existing artifact type and command (`/create-bug`, `/create-spec`) with a mode flag, or does it get its own?

## Decision

A new standalone command, `/create-investigation`, authoring a new artifact type, `INV-NNNN`, at `sdlc/investigations/`. Its decision outcomes branch to either nothing further, a new standalone `/create-adr` command (itself new, reusable beyond Investigations), or back into intake (G1) as ordinary new work if a build is recommended.

## Alternatives Considered

**Extending `/create-bug` with an "investigation mode"** was rejected: a BUG's lifecycle always terminates in a fix (`reported → triaged → fixed → verified`) — there is no BUG state that means "concluded, nothing to fix." Forcing Investigation's actually-different lifecycle (`open → concluded`, with a decision that may be "do nothing") through BUG's state machine would mean either inventing fake terminal states that don't mean what BUG's other states mean, or leaving BUG's own state machine ambiguous about what "fixed" means for a ticket that was never a defect in the first place.

**Extending `/create-spec` with an "investigation mode"** was rejected for the mirror-image reason: a SPEC's entire structure (Goals, Acceptance Criteria, NFRs) presupposes a build is coming. An Investigation that concludes "no action" would leave every one of those sections either empty or fabricated to satisfy a template that doesn't fit what actually happened.

Both alternatives were also weighed against the precedent already set earlier in this same SPEC's own design history: the explicit decision *not* to introduce `EPIC-NNNN`/`FEATURE-NNNN`/`USER-STORY-NNNN` as parallel artifact types, specifically to avoid duplicating ADO's own hierarchy. That precedent argues for reuse-over-new-types in general — but it applies to concepts ADO *already* owns as its source of truth. Investigation's *outcome record* (question, approach, findings, decision) is not something ADO owns or tracks in any structured way; ADO's own Investigation ticket type is just a label with no corresponding content shape. Minting `INV-NNNN` here doesn't duplicate anything ADO already provides — it's new content ForgeKit is the first system to actually structure.

## Consequences

ForgeKit gains its fourth top-level numbered artifact sequence with its own directory (`sdlc/investigations/`, alongside `sdlc/specs/`, `sdlc/bugs/`, `sdlc/incidents/`) — a real, if small, addition to the pipeline's vocabulary. Future proposals for new artifact types should be checked against the same test applied here: does the external tool already own this content's shape (reuse/pointer, no new type), or is this genuinely new structured content ForgeKit is the first to track (a new type may be warranted)?

`/create-adr` becoming a standalone command (not bundled into `/create-investigation`) means any future need for an ADR with no parent DESIGN — not just Investigation's — already has a path, without needing its own design discussion later.

## Status Rationale

Accepted as part of DESIGN-0001's Round 0 draft, resolving SPEC-0001's OQ4.

## Revisit Triggers

- If a future proposal wants to extend BUG or SPEC's lifecycle to cover an "investigation-like" outcome, re-examine whether the lifecycle mismatch reasoning here still holds, or whether that proposal's specific case is different enough to warrant reuse after all.
- If `INV-NNNN` volume in practice turns out tiny enough that a dedicated artifact type feels like overhead rather than clarity, revisit whether it should have been a lighter-weight convention instead (e.g., a tagged section inside an existing artifact type).
