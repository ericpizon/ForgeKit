---
id: ADR-0004
type: adr
schema_version: 1
status: accepted
target_design: DESIGN-0001
created: 2026-10-05
---

# ADR-0004 — Promotion Records Use an Append-Only Per-Environment Log, Not a Numbered Per-Event Artifact

## Context

SPEC-0001 (G4, AC15) requires a lightweight record for every `governance: promotion` environment transition — a candidate map, who, when, and a smoke-test result. Every other numbered ForgeKit artifact type (`SPEC-NNNN`, `BUG-NNNN`, `RELEASE-PLAN-vX.Y.Z`, and now `INV-NNNN`) gets its own file, minted with its own sequential ID. Promotions are explicitly meant to be routine and frequent — potentially many times per day across active work.

## Decision

Promotion records do not get their own numbered artifact. Instead, each environment with `governance: promotion` has one file — `sdlc/promotions/{environment}.md` — holding an append-only table, one row per promotion event.

## Alternatives Considered

**A numbered per-event artifact (`PROMOTION-NNNN`)**, matching every other ForgeKit artifact type's convention, was rejected specifically because of the frequency mismatch this feature itself exists to solve. SPEC-0001's whole motivation for G4 was "a routine promotion to a Dev or QA environment has no lightweight path — only the full RELEASE-PLAN ceremony... or nothing at all." Minting a new numbered file, in its own registry entry, for every single QA promotion would recreate exactly the ceremony-for-routine-events problem this goal was written to avoid — just with a smaller file instead of a smaller process.

## Consequences

This is ForgeKit's first artifact-storage pattern that isn't "one file per event, sequentially numbered." Promotion history for a given environment is read by opening one file and scanning rows, not by listing a directory of many small files. This trades away per-event addressability (there's no `PROMOTION-0047` to link to from elsewhere) for proportionality to the actual event frequency — consistent with SPEC-0001's own stated goal of ceremony proportional to risk.

Any future lightweight, high-frequency event this pipeline needs to track should be checked against this same reasoning before defaulting to "mint a new NNNN sequence": is the event frequent enough that per-event files would themselves become the ceremony problem being solved?

## Status Rationale

Accepted as part of DESIGN-0001's Round 0 draft, resolving part of SPEC-0001's G4 scope (the record shape for `governance: promotion`).

## Revisit Triggers

- If a future need arises to reference one specific promotion event from elsewhere (e.g., a RELEASE-PLAN citing exactly which QA promotion served as its evidence), revisit whether rows need their own stable identifier within the log (a row-level ID, not necessarily a full separate file).
- If promotion logs grow large enough that scanning one file becomes unwieldy in practice, revisit whether per-environment-per-period files (e.g. monthly) would restore proportionality without reintroducing per-event files.
