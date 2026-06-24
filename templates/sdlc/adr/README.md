# sdlc/adr/

Architectural Decision Records (ADRs) produced by `/create-design` when a significant technical choice is made.

## Contents

```
ADR-NNNN-short-title.md    # one ADR per architectural decision
```

## When ADRs are created

`/create-design` prompts the Architect role to create an ADR whenever a decision meets the significance threshold — typically: decisions that constrain future implementation choices, cannot easily be reversed, or would surprise a new contributor who reads only the code.

## Format

Each ADR records:
- **Context** — what situation prompted the decision
- **Decision** — what was chosen
- **Alternatives considered** — what was rejected and why
- **Consequences** — what this decision enables and what it forecloses

## Do not delete ADRs

ADRs are an audit trail. If a decision is later reversed, add a new ADR referencing the original rather than deleting the old one.
