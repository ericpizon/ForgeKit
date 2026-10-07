# sdlc/design/

Technical design artifacts. Each DESIGN-NNNN maps to a SPEC-NNNN and captures the technical approach — data model changes, component design, failure modes, and architectural decisions — before implementation planning begins.

## Commands

| Command | Role | What it does |
|---|---|---|
| `/create-design SPEC-NNNN` | Architect | Author technical design from approved spec |
| `/review-design DESIGN-NNNN` | Architect | Multi-perspective review: DBA, Security, SRE, Lead Dev viewpoints |
| `/address-design-review DESIGN-NNNN` | Architect | Address findings; increments version in frontmatter |
| `/approve-design DESIGN-NNNN` | Architect | Formal approval; updates ARCH.md with any cross-cutting patterns; unlocks `/create-plan` and parallel OBS/SEC stages |
| `/approve-design DESIGN-NNNN interface-freeze` | Architect | Coordination-repo threads only — approves just the Cross-Repo Contract sub-section ahead of the rest of DESIGN, so one unresolved question elsewhere doesn't block every participant's `/create-plan` |
| `/reopen-design DESIGN-NNNN` | Architect | Back-cycle: a downstream discovery invalidated this DESIGN — captures a discovery record, assesses blast radius |

## Sub-artifacts

Architectural Decision Records (ADRs) are created for significant decisions made during design. ADRs live in `sdlc/adr/`, not in this directory — thread-scoped at `sdlc/THREAD-NNNN/adr/` for a coordination-repo thread, flat at `sdlc/adr/` otherwise (or, for a standalone decision not tied to this design at all, via `/create-adr` directly — see `sdlc/adr/README.md`).

## Status lifecycle

```
draft → in_review → approved
             ↘ needs_revision → draft (after /address-design-review)
```

## Naming convention

`DESIGN-NNNN-[kebab-case-title].md`
