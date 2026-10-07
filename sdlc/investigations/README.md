# sdlc/investigations/

Investigation and decision records produced by `/create-investigation` — spike or research work that may legitimately conclude without any implementation.

## Status lifecycle

```
open → concluded
```

An Investigation is `open` while the Question → Approach → Findings walk is still in progress, and `concluded` once a decision is recorded. There is no "in between" status — the decision is the thing that closes it.

## Decision outcomes

Exactly one of four, recorded in the `decision` frontmatter field:

| Decision | Meaning |
|---|---|
| `no_action` | The question was answered; nothing needs to be built. |
| `adr` | The finding is itself an architectural decision worth preserving — hands off to `/create-adr`, which authors a standalone `ADR-NNNN` with no parent DESIGN required. |
| `recommend_implementation` | The finding recommends building something. This is advisory text only — `/create-investigation` never creates or writes to any external ticket. A human acts on the recommendation; the resulting work re-enters ForgeKit as ordinary new intake if and when it does. |
| `investigate_further` | The question isn't answerable yet; the Investigation stays `open` for a follow-up round. |

## Why this isn't a BUG or a SPEC

A BUG's lifecycle always terminates in a fix. A SPEC's structure (Goals, Acceptance Criteria, NFRs) presupposes a build is coming. An Investigation may conclude "do nothing" — forcing that outcome through either of those templates means either inventing a fake terminal state or leaving a filled-in template that doesn't describe what actually happened.

## Naming convention

`INV-NNNN-[kebab-case-title].md`
