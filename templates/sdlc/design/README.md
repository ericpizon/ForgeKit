# sdlc/design/

Technical design artifacts. Each DESIGN-NNNN maps to a SPEC-NNNN and captures the technical approach — data model changes, component design, failure modes, and architectural decisions — before implementation planning begins.

## Commands

| Command | Role | What it does |
|---|---|---|
| `/create-design SPEC-NNNN` | Architect | Author technical design from approved spec |
| `/review-design DESIGN-NNNN` | Architect | Multi-perspective review: DBA, Security, SRE, Lead Dev viewpoints |
| `/address-design-review DESIGN-NNNN` | Architect | Address findings; increments version in frontmatter |
| `/approve-design DESIGN-NNNN` | Lead Dev | Formal approval; unlocks `/create-plan` and parallel OBS/SEC stages |

## Sub-artifacts

Architectural Decision Records (ADRs) are created for significant decisions made during design. ADRs live in `sdlc/adr/`, not in this directory.

## Status lifecycle

```
draft → in_review → approved
             ↘ needs_revision → draft (after /address-design-review)
```

## Naming convention

`DESIGN-NNNN-[kebab-case-title].md`
