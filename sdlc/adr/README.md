# sdlc/adr/

Architectural Decision Records (ADRs) — produced either by `/create-design` when a significant technical choice is made during a feature's design, or standalone by `/create-adr` when a decision needs recording with no parent DESIGN or SPEC at all.

## Contents

```
ADR-NNNN-short-title.md    # one ADR per architectural decision
```

**In a `repo_role: coordination` repo:** this directory holds only *standalone* ADRs (via `/create-adr`, directly or via `/create-investigation`'s `decision: adr` hand-off) — a decision not tied to any one cross-repo thread. An ADR about a specific thread's own cross-repo contract lives thread-scoped instead, at `sdlc/THREAD-NNNN/adr/`, authored by that thread's own `/create-design`.

## When ADRs are created

- **`/create-design`** prompts the Architect role to create an ADR whenever a decision meets the significance threshold — typically: decisions that constrain future implementation choices, cannot easily be reversed, or would surprise a new contributor who reads only the code. (In a coordination repo, this path is thread-scoped — see above.)
- **`/create-adr`** authors one directly, with no parent DESIGN or SPEC required — most commonly reached via `/create-investigation` concluding `decision: adr`, but usable any time a durable decision needs recording independent of a specific build.

## Format

Each ADR records:
- **Context** — what situation prompted the decision
- **Decision** — what was chosen
- **Alternatives considered** — what was rejected and why
- **Consequences** — what this decision enables and what it forecloses

## Do not delete ADRs

ADRs are an audit trail. If a decision is later reversed, add a new ADR referencing the original rather than deleting the old one.
