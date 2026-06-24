---
id: review-design
type: command
name: /review-design
description: Architect command to run all Stage 3 design perspectives in one pass — Options Analysis, DBA, Security, SRE, Lead Developer
status: active
version: 1
primary_role: architect
output_artifact_type: design (reviewed)
---

# /review-design

## Purpose

Run all five Stage 3 reviews of DESIGN-NNNN in a single pass: Options Analysis, DBA, Security, SRE, and Lead Developer. The Architect executes this command immediately after drafting DESIGN, and again after each revision pass in `/address-design-review`. Output is a `## Design Review — Round N` section appended to DESIGN-NNNN.

## When to Invoke

- Immediately after `/create-design` produces the DESIGN draft and the user confirms
- After `/address-design-review` revises the DESIGN (next review round)

Pass SPEC and DESIGN IDs: `/review-design SPEC-NNNN DESIGN-NNNN`.

## On Re-Review

If a `## Design Review — Round N` section already exists in DESIGN-NNNN:

1. Identify the current round number (prior round + 1).
2. Read the `## Address Summary — Round N` section. Note which findings were addressed, accepted risk, deferred, or escalated.
3. For each perspective: confirm previously-blocking findings are resolved. Do not re-raise findings marked accepted risk or deferred.
4. Append a new `## Design Review — Round N` section. Do not modify prior round sections — they form an audit trail.

## Steps

Run all five steps in sequence. Read ARCH.md before starting — all findings must be grounded in the project's actual patterns, constraints, and tech stack.

### 0. Options Analysis Review

Before reviewing the technical spec, validate the options analysis that produced it. This step has a high blocking bar — only challenge the chosen option if you can make a concrete case for a different one.

Assess:

1. **Options present.** Were at least 2 options generated? If the Design Options section is absent or contains only one option, that is a blocking finding.
2. **Options are genuinely distinct.** Do the options represent meaningfully different architectural approaches? Two options that differ only in naming are a blocking finding.
3. **Recommended option reasoning quality.** Does the recommendation explicitly cite SPEC constraints, ANALYSIS risk flags, or ARCH patterns? A recommendation justified only by "simpler" or "cleaner" is a non-blocking finding requiring stronger justification.
4. **Wrong option chosen.** Is there a concrete case that a non-recommended option would better satisfy the SPEC ACs, ANALYSIS risk flags, or ARCH constraints? If yes, this is a blocking finding — state the specific constraint the recommended option fails to satisfy.
5. **Missing option.** Is there an obviously viable approach that was not considered — particularly in brownfield codebases where extending vs. introducing new structure is a genuine choice? If a missing option would materially change the recommendation, blocking. If worth noting but unlikely to change the outcome, non-blocking.
6. **ADR linkage.** For each Key Decision ADR, does its `Alternatives Considered` section reference the options analysis? Omitting the options entirely is non-blocking but worth flagging.

### 1. DBA Perspective

Adopt the DBA role. Adversarially review DESIGN-NNNN's data model, schema decisions, and query patterns — identify problems before they are locked into a PLAN. Consult ARCH.md for the project's database, ORM, and migration tool. Assess:

1. **Data model completeness.** List every table/collection and every column/field. Note which fields appear in WHERE clauses or JOIN conditions — these are index candidates.
2. **Index coverage on hot paths.** Any field queried on every request must be indexed. List any WHERE-clause field with no corresponding index.
3. **Migration safety.** For each DDL operation implied: is it safe under concurrent production traffic? Name any operation that could lock tables or require downtime.
4. **Connection pool implications.** Under concurrent load, does the design stay within the project's database connection limits? Read ARCH.md for the connection model.
5. **Audit fields.** Every mutable table should have `updatedAt` (or equivalent). A mutable table without an audit timestamp is a finding.
6. **Data types.** Are types appropriate for the values they store? Flag any mismatch between stored data and column type.
7. **ORM patterns.** Are the proposed query patterns achievable with the project's ORM without raw SQL? If a decision implies a query pattern that requires raw SQL, flag it.

### 2. Security Perspective

Adopt the Security role. Review DESIGN-NNNN's trust boundary placement, auth surface accuracy, and credential handling — this is not the full Stage 4b security review, but confirming architectural soundness before DESIGN approval. Read ARCH.md for the project's auth model. Assess:

1. **Trust boundary map.** Confirm the DESIGN's auth model matches ARCH.md's authentication model exactly.
2. **Auth-before-lookup enforcement.** Every external endpoint must check auth before touching the database. Any code path that implies a data lookup before auth verification is a blocking finding.
3. **Credential comparison safety.** Any credential or key comparison must use a timing-safe comparison mechanism. If the DESIGN specifies naive string equality for secret comparison, that is a blocking finding.
4. **Plaintext-never-persisted rule.** Sensitive values (API keys, passwords, tokens) must never be stored or logged in plaintext. If the DESIGN implies logging or persisting a plaintext credential, that is a blocking finding.
5. **Auth middleware scope.** If the project uses auth middleware, confirm its matcher scope is correct — particularly for public endpoints that must not trigger auth redirects.
6. **Implicit scope beyond Non-Goals.** If the DESIGN implies authorization logic beyond what SPEC-NNNN's Non-Goals allow, flag it as a scope question.
7. **`security_review_required` consistency.** Confirm the DESIGN's surface is consistent with the verdict from ANALYSIS-NNNN.

### 3. SRE Perspective

Adopt the SRE role. Adversarially stress-test DESIGN-NNNN against production failure scenarios. Assess:

1. **Dependency failure modes.** For each new dependency the DESIGN introduces: what happens when it is unavailable? When it is slow? When it returns corrupt data? Missing failure mode analysis for a critical dependency is a finding.
2. **Rollback story.** Can the deployment be rolled back without a data migration? Does the schema allow reverting to a prior deployment? A schema change that cannot be reverted without data loss requires a rollback plan in the DESIGN.
3. **Failure mode distinguishability.** Can an operator tell the difference between distinct failure conditions at 3am? If two distinct failures produce the same observable error, document it — non-blocking but must be recorded.
4. **Operational toil.** Does the DESIGN require manual steps on a recurring basis not acknowledged in the design? Flag unacknowledged toil.
5. **Infrastructure-specific latency.** Does the DESIGN acknowledge latency characteristics of the underlying infrastructure (serverless cold starts, connection pool warm-up, etc.)? Latency-sensitive behavior added without acknowledging known infrastructure delays is a finding.
6. **Monitoring surface.** Can an operator determine system health from the signals described in the DESIGN? Is there a signal that goes to zero when a critical endpoint stops working?

### 4. Lead Developer Perspective

Adopt the Lead Developer role. Adversarially stress-test DESIGN-NNNN for implementability. Read ARCH.md for framework conventions and known stack constraints before assessing. Assess:

1. **Key Decision implementability.** Can each Key Decision be directly translated into a PLAN task? Abstract decisions like "the endpoint should be fast" are not implementable — flag them for concretization.
2. **Framework correctness.** Does the DESIGN assume any framework behavior that is incorrect or inconsistent with ARCH.md conventions? Common traps include implicit behavior assumptions about middleware, server-side rendering, and lifecycle hooks.
3. **Hidden stack dependencies.** Does any design decision require a library not in the current stack, an ORM feature that doesn't exist, or a framework capability not established in the architecture? Name the dependency and confirm it is available.
4. **Module layout against ARCH.** Does the DESIGN's implied file structure match the canonical layout in ARCH.md? Intentional deviations must be documented in the DESIGN.
5. **Data access layer pattern.** Does the DESIGN imply inline data queries in route/request handlers? That may contradict the ARCH pattern (read ARCH.md to verify). Flag pattern violations.
6. **Decision testability.** A Developer writing tests against this DESIGN needs to write a failing test before implementing. If any decision is too abstract to test, it will produce a guess-and-check implementation.

### 5. Cross-Perspective Check

After completing all four perspectives:

- **Contradictions:** Does any DBA finding contradict a Lead Developer finding? Does any Security finding conflict with the SRE rollback story? Flag contradictions explicitly — the Architect must arbitrate before addressing.
- **Cascade:** Did DBA findings reveal schema implications with security consequences? Did a Key Decision clarification create new failure modes the SRE should know about? Add to the appropriate perspective's findings.

## Output

Append a `## Design Review — Round N` section to DESIGN-NNNN:

```
## Design Review — Round N

**Date:** YYYY-MM-DD
**Status:** Changes requested — N blocking, N non-blocking | Approved — no blocking findings

### Options Analysis Findings
- [OPT-1] **blocking** — [finding]
- [OPT-2] **non-blocking** — [finding]
- [or "None."]

### DBA Findings
- [DBA-1] **blocking** — [finding]
- [DBA-2] **non-blocking** — [finding]

### Security Findings
- [SEC-1] **blocking** — [finding]

### SRE Findings
- [SRE-1] **non-blocking** — [finding]

### Lead Developer Findings
- [LD-1] **blocking** — [finding]

### Contradictions Requiring Arbitration
- [or "None."]
```

## Quality Gate

Before capturing the review output:
- [ ] Options Analysis step completed — all six sub-checks assessed
- [ ] All four technical perspectives completed (DBA, Security, SRE, Lead Developer) — no perspective skipped
- [ ] Each finding labeled blocking or non-blocking with a reason
- [ ] Cross-perspective check completed — contradictions identified or confirmed absent
- [ ] Round number is correct

## Handoff

**If blocking findings exist:**
> "There are blocking findings. Run `/address-design-review DESIGN-NNNN` to address them."

**If no blocking findings:**
> "Review is clean. Run `/approve-design DESIGN-NNNN` to commit and close Stage 3."
