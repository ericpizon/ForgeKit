# sdlc/specs/

Feature specifications. Each SPEC-NNNN captures the user-facing requirement for a feature thread — the problem, goals, acceptance criteria, and non-goals — before any design or implementation begins.

## Commands

| Command | Role | What it does |
|---|---|---|
| `/create-spec` | TPO | Author SPEC from a problem description or postmortem action item |
| `/review-spec SPEC-NNNN` | Lead Dev | Multi-perspective review: feasibility, scope, edge cases, security surface |
| `/revise-spec SPEC-NNNN` | TPO | Address review findings; increments version in frontmatter |
| `/approve-spec SPEC-NNNN` | TPO | Formal approval; unlocks `/create-design` |

## Status lifecycle

```
draft → in_review → approved
             ↘ needs_revision → draft (after /revise-spec)
```

## Naming convention

`SPEC-NNNN-[kebab-case-title].md`
