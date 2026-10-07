# sdlc/specs/

Feature specifications. Each SPEC-NNNN captures the user-facing requirement for a feature thread — the problem, goals, acceptance criteria, and non-goals — before any design or implementation begins.

## Commands

| Command | Role | What it does |
|---|---|---|
| `/create-spec` (or `--from <system>:<id>` to start from an existing external ticket) | TPO | Author SPEC from a problem description, postmortem action item, or external ticket |
| `/review-spec SPEC-NNNN` | Lead Dev | Multi-perspective review: feasibility, scope, edge cases, security surface |
| `/revise-spec SPEC-NNNN` | TPO | Address review findings; increments version in frontmatter |
| `/validate-spec SPEC-NNNN` | TPO | Product validation sign-off — confirms the spec represents real product intent |
| `/approve-spec SPEC-NNNN` | TPO | Formal approval; unlocks `/create-design`. Offers to write the decision back to the originating ticket if the SPEC came from one. |
| `/reopen-spec SPEC-NNNN` | TPO | Back-cycle: a downstream discovery invalidated this SPEC — captures a discovery record, assesses blast radius |
| `/promote-to-thread SPEC-NNNN` | Any | Cross-repo scope discovered mid-flow — converts this SPEC into a `THREAD-NNNN` in a coordination repo |

## Status lifecycle

```
draft → in_review → approved
             ↘ needs_revision → draft (after /revise-spec)
             ↘ promoted (via /promote-to-thread — never deleted; superseded_by names the THREAD-NNNN)
```

`promoted` is terminal and reachable from either `draft` or `approved` — once cross-repo scope is discovered, the SPEC stays as a historical record and the `THREAD-NNNN` it names becomes the authoritative artifact. `/status SPEC-NNNN` reports this redirect explicitly rather than showing stale single-repo status.

## Naming convention

`SPEC-NNNN-[kebab-case-title].md`
