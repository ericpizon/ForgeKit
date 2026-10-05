---
id: ADR-0002
type: adr
schema_version: 1
status: accepted
target_design: DESIGN-0001
created: 2026-10-05
---

# ADR-0002 — SYSTEM-ARCH.md's Dependency Map Is Derived, Regenerated Wholesale at Interface Freeze

## Context

SPEC-0001 (G2) requires a coordination repo's `SYSTEM-ARCH.md` to hold a Dependency Map "derived from the current set of active-status XDEPs across all threads — never a hand-maintained, independently-drifting duplicate of XDEP content" (AC6). Left open (OQ1): what actually triggers that derivation, and how.

Three real alternatives exist:
- **A new, separate sync command**, invoked on demand whenever someone wants a fresh view.
- **Full regeneration at `/approve-design`'s Interface Freeze step** — the exact moment an XDEP transitions to `active`.
- **Incremental patching** — each Interface Freeze updates only the specific XDEP rows that changed, never a full rebuild.

## Decision

Full regeneration, triggered automatically at `/approve-design`'s Interface Freeze step. The Dependency Map is rebuilt wholesale from the complete current set of `active`-status XDEPs every time an Interface Freeze happens — never incrementally patched, and never requiring a separate command to stay current.

## Alternatives Considered

**A separate sync command** was rejected because it reintroduces exactly the staleness problem this feature exists to solve: a view that's only as fresh as the last time someone remembered to run the sync command is a hand-maintained view with extra automation, not a genuinely derived one. This is the same failure mode already diagnosed and fixed once in this engagement for `ARCH.md`'s own Module Layout/Testing Strategy/Observability sections (which went stale because no command wrote drift back automatically) — the fix there was to attach the sync to an event that already happens (a milestone's `/approve-implementation`), not to add a new command nobody remembers to run. The same reasoning applies here.

**Incremental patching** was rejected as unnecessary complexity for no real benefit: Interface Freeze events are infrequent (one per cross-repo dependency change, not a high-frequency path), so a full rebuild's cost is negligible, and a full rebuild is structurally simpler and less error-prone than tracking which specific rows need patching.

## Consequences

`SYSTEM-ARCH.md`'s Dependency Map can only ever be as current as the most recent Interface Freeze — if a long stretch of time passes with cross-repo dependencies changing but no DESIGN ever reaching Interface Freeze for them, the map would still reflect an older state. This is accepted: Interface Freeze is the point at which an XDEP's status actually becomes authoritative (`active`), so the map is never "wrong" relative to what's actually been approved — only potentially behind what's still in draft.

A regeneration failure (e.g., a malformed XDEP file) must not block the Interface Freeze approval itself — `SYSTEM-ARCH.md` is a derived convenience view, not a gate on the thing that actually matters (the Cross-Repo Contract's own approval). `/approve-design` logs a warning and proceeds.

## Status Rationale

Accepted as part of DESIGN-0001's Round 0 draft, resolving SPEC-0001's OQ1.

## Revisit Triggers

- If Interface Freeze events become frequent enough that full-rebuild cost becomes measurable (unlikely given the nature of cross-repo contract changes, but worth re-checking if a team's usage pattern differs sharply from what was assumed here), revisit incremental patching.
- If a team needs the Dependency Map to reflect `draft`/`in_review` XDEPs as well as `active` ones (e.g., for forward-looking planning), that's a different artifact or a different field, not a reason to change when this one regenerates.
