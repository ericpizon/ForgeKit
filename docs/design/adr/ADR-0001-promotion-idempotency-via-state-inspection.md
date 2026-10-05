---
id: ADR-0001
type: adr
schema_version: 1
status: accepted
target_design: DESIGN-0001
created: 2026-10-05
---

# ADR-0001 — Promotion Idempotency via State-Inspection, Not a Checkpoint File

## Context

`/promote-to-thread` (SPEC-0001, G3) is ForgeKit's first multi-step command that mutates state across two repos (the promoting repo and the coordination repo) and must survive being interrupted partway through. SPEC-0001's own Reliability NFR requires it be "safely resumable" and that a partial promotion "never leave the original SPEC ambiguous between promoted and not promoted."

Three options were considered (full detail in DESIGN-0001's Design Options):
- **Option A** — a local checkpoint file tracking which internal step last completed, resumed from that point on retry.
- **Option B** — every step checks its own completion condition against actual committed state before acting; no side file.
- **Option C** — no resumability at all; a retry starts over with a fresh THREAD-NNNN, abandoning any partial one.

## Decision

Option B. Every step of `/promote-to-thread` is idempotent by construction: it checks whether its own effect already exists in committed state before acting, and skips if so. The original SPEC's `status: promoted`/`superseded_by` transition is the *last* step, only performed once every other step is confirmed complete — guaranteeing an interrupted run never shows `promoted` with an incomplete thread behind it.

## Alternatives Considered

**Option A (checkpoint file)** was rejected despite offering more precise failure diagnostics. It introduces a wholly new class of state ForgeKit has never needed — a side file that itself could desync from reality (e.g. if someone deletes it, or two people run the command concurrently against the same SPEC). Option B has no equivalent failure mode: the committed state itself is unambiguous and shared.

**Option C (no resumability)** was rejected outright — it directly contradicts SPEC-0001's own approved Reliability NFR, and would leave orphaned `THREAD-NNNN` allocations behind over time, wasting IDs and confusing anyone who encounters them.

## Consequences

`/promote-to-thread` is the first ForgeKit command built on "check real state before acting" as an explicit, named resumability strategy — not just an incidental property. This establishes a reusable precedent: it extends the same philosophy already behind THREAD/XDEP ID collision checks (does this number already exist at HEAD before allocating) and the self-referential-ref pattern (committed state as truth, not a side mechanism). Any future ForgeKit command that needs to be interruption-safe should default to this pattern before considering a checkpoint-file approach.

The cost: each step's implementation is slightly more verbose (an existence check before the action, not just a sequential script), and if something does go wrong, diagnosing exactly which step failed requires inspecting actual repo state rather than reading a log. Accepted — `/promote-to-thread` is human-invoked and infrequent, not a high-volume automated path where that diagnostic gap would matter more.

## Status Rationale

Accepted as part of DESIGN-0001's Round 0 draft, following the Architect's confirmed choice of Option B during Phase 2 (Options Analysis) of `/create-design`.

## Revisit Triggers

- If a future ForgeKit command needs multi-step resumability across more than two repos, or across a step that cannot be made idempotent by a simple existence check (e.g., a step with an unavoidable external side effect that can't be safely repeated), revisit whether this pattern still applies or whether a checkpoint-style mechanism becomes justified for that specific case.
- If `/promote-to-thread` in practice is invoked at high frequency (contradicting its expected human-invoked, infrequent usage pattern), revisit whether the diagnostic cost accepted here becomes a real operational problem.
