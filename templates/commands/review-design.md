---
id: review-design
type: command
name: /review-design
description: Architect command to run all Stage 3 design perspectives in one pass — Options Analysis, DBA, Security, SRE, Lead Developer, plus Cross-Repo Contract Review for coordination-repo threads
status: active
version: 5
primary_role: architect
output_artifact_type: design (reviewed)
changelog:
  - v5 — added a structured `latest_round` frontmatter field (round/blocking/non_blocking/verdict), written alongside the existing `**Status:**` prose line every round, so gate results are readable without parsing prose. Built for the ForgeKit Operator feature (PLAN-0001 T1), closing a gap the Operator gap-analysis identified across every review-*.md command.
  - v4 — added the Cross-Repo Contract Review step for coordination-repo threads: checks XDEP verification appropriateness and cross-side symmetry, and deployment-ordering cycles, ahead of Interface Freeze in approve-design.
---

# /review-design

## Purpose

Run all five Stage 3 reviews of DESIGN-NNNN in a single pass: Options Analysis, DBA, Security, SRE, and Lead Developer — plus a Cross-Repo Contract Review as a sixth, leading pass when DESIGN-NNNN belongs to a `repo_role: coordination` thread. The Architect executes this command immediately after drafting DESIGN, and again after each revision pass in `/address-design-review`. Output is a `## Design Review — Round N` section appended to DESIGN-NNNN.

## When to Invoke

- Immediately after `/create-design` produces the DESIGN draft and the user confirms
- After `/address-design-review` revises the DESIGN (next review round)

Pass SPEC and DESIGN IDs: `/review-design SPEC-NNNN DESIGN-NNNN`.

## On Re-Review

If a `## Design Review — Round N` section already exists in DESIGN-NNNN:

1. Read `current_review_round` from DESIGN-NNNN frontmatter. The new round number is `current_review_round + 1`.
2. Read the `## Address Summary — Round N` section. Note which findings were addressed, accepted risk, deferred, or escalated.
3. Load only the DESIGN sections cited in the address summary — do not re-read the full DESIGN. Sections not referenced by any addressed finding are unchanged by definition.
4. For each perspective: confirm previously-blocking findings are resolved. Do not re-raise findings marked accepted risk or deferred. **If a perspective's domain was not touched by any addressed finding, emit a single line — `[Perspective]: no new findings — domain unaffected by Round N address.` — and skip the full perspective analysis.**
5. Append a new `## Design Review — Round N` section. Do not modify prior round sections — they form an audit trail.

## Steps

Run all steps in sequence. Read ARCH.md before starting — all findings must be grounded in the project's actual patterns, constraints, and tech stack. For a `repo_role: coordination` thread, re-resolve every entry in DESIGN-NNNN's own `inputs.architecture` list (per [`templates/reference/cross-repo-resolution.md`](../reference/cross-repo-resolution.md)) rather than reading a single local ARCH.md — a review grounded in one implicit stack misses whatever the other participants actually expose. If DESIGN-NNNN has a Cross-Repo Contract section, also run Step 0a before the Options Analysis Review.

### 0a. Cross-Repo Contract Review (coordination-repo threads only)

Before reviewing the rest of the DESIGN, review the Cross-Repo Contract sub-section (its XDEPs) on its own — this is the sub-section that can be approved ahead of the rest of DESIGN (see `/approve-design`'s Interface Freeze step), so it needs a review pass with its own blocking bar. Read [`templates/reference/xdep-schema.md`](../reference/xdep-schema.md) before assessing. For each XDEP:

1. **Verification appropriateness.** For a `contract`-kind XDEP, is the chosen `verification` layer set actually appropriate for its `interface_kind` (e.g. an `async-event` relying on schema-registry compatibility alone, with no semantic/behavioral check where meaning can drift without a schema change)? An inappropriate or missing verification layer for the interface type is a blocking finding.
2. **Verification symmetry.** Is the verification rigor comparable on both the provider's and consumer's side, or does one side's own local review quietly carry more weight than the other's? A contract that's rigorously checked by the provider and only loosely checked by the consumer (or vice versa) is a blocking finding — this can't be caught by either participant's own review in isolation, since each only checks its own bar.
3. **Kind-payload correctness.** Does the XDEP's payload match its declared `dependency_kind` (e.g. a `deployment`-kind XDEP has no dangling `contract` block)? Flag any mismatch.
4. **Identity discipline.** Does any XDEP's resolution depend on `related_acceptance_criteria` rather than being referenced directly via `satisfies_xdeps` from a PLAN task? ACs are traceability only — a blocking finding if the design implies otherwise.
5. **Deployment-ordering cycles.** Across all `deployment`-kind (and non-`independent`-`rollout.ordering` `contract`-kind) XDEPs in this thread, confirm no cycle exists. A cycle is blocking — there is no valid deploy order until it's resolved.
6. **Fan-out correctness.** For a provider with multiple consumers on the same capability, confirm each relationship is its own XDEP sharing the same `contract.id` — not one XDEP listing multiple consumers.

### 0b. Options Analysis Review

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

1. **Data model completeness.** List every table/collection and every column/field — including the full nested field list of any jsonb/document-typed column, enumerated with the same rigor as a relational column (a jsonb/document column is not exempt from completeness review just because it is schemaless at the DB layer). Note which fields appear in WHERE clauses or JOIN conditions — these are index candidates.
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
8. **Data-authorship boundary trace.** If the project establishes a boundary between deterministic code and model-authored output (e.g. an ADR stating "the model must never author the final score/decision"), trace every field the model is allowed to return and confirm none of them indirectly determines the protected outcome — a field used as a weight, multiplier, or other input to the protected computation is functionally authoring it, even if it isn't the final value itself. Flag any model-returned field that feeds a downstream deterministic decision without being independently fixed or validated first.
9. **Prompt-injection framing check.** *(Applicable if `project.yml`'s `stack.llm_agent_untrusted_input` is `yes` and this DESIGN builds a prompt from that input.)* For every prompt built from external or user-controlled text, does the DESIGN state how injection risk is bounded — either explicit prompt framing telling the model the content is data to analyze, not instructions to follow, or structural containment (the content can only reach a code path the model doesn't control, per item 8's boundary trace)? A DESIGN that sends untrusted text into a prompt with no stated mitigation of either kind is a blocking finding — this is the input-side counterpart to item 8's output-side check, and neither substitutes for the other.

### 3. SRE Perspective

Adopt the SRE role. Adversarially stress-test DESIGN-NNNN against production failure scenarios. Assess:

1. **Dependency failure modes.** For each new dependency the DESIGN introduces: what happens when it is unavailable? When it is slow? When it returns corrupt data? Missing failure mode analysis for a critical dependency is a finding.
2. **Rollback story.** Can the deployment be rolled back without a data migration? Does the schema allow reverting to a prior deployment? A schema change that cannot be reverted without data loss requires a rollback plan in the DESIGN.
3. **Failure mode distinguishability.** Can an operator tell the difference between distinct failure conditions at 3am? If two distinct failures produce the same observable error, document it — non-blocking but must be recorded.
4. **Operational toil.** Does the DESIGN require manual steps on a recurring basis not acknowledged in the design? Flag unacknowledged toil.
5. **Infrastructure-specific latency.** Does the DESIGN acknowledge latency characteristics of the underlying infrastructure (serverless cold starts, connection pool warm-up, etc.)? Latency-sensitive behavior added without acknowledging known infrastructure delays is a finding.
6. **Monitoring surface.** Can an operator determine system health from the signals described in the DESIGN? Is there a signal that goes to zero when a critical endpoint stops working?
7. **Concurrent-writer safety.** For any resource (DB row, file, cache entry) that can be written by more than one independent code path — e.g. a background/async job's completion write racing a separately-triggered reconciliation or timeout write — confirm the DESIGN states a conditioned write pattern (optimistic locking, conditional UPDATE, compare-and-swap) rather than an unconditional write, and explicitly states what happens when two writers race for the same resource. An unconditional write to a resource with more than one writer path is a blocking finding.

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
- **Structural cross-reference:** For any data structure defined in one section (e.g. an algorithm's stated output shape) and consumed or validated in another section, confirm the consuming section's assumptions about that structure's shape/granularity match the producing section's actual definition — not just its name. A validation check that assumes a coarser or finer granularity than what's actually produced is a contradiction, even if both sections individually read as correct.

## Output

Append a `## Design Review — Round N` section to DESIGN-NNNN. Then update `current_review_round` in DESIGN-NNNN frontmatter to the round number just appended.

Also set a `latest_round: {round: N, blocking: N, non_blocking: N, verdict: ready | needs_revision}` frontmatter field to the same values the round section's own `**Status:**` line states (`verdict: ready` when blocking is 0, `needs_revision` otherwise). This is the structured counterpart to that prose line — written every round alongside it, never instead of it, so a reader or an automated resolver can determine the gate result without parsing the `**Status:**` sentence.

**Null suppression:** Omit any `### [Perspective] Findings` section that has no findings — do not emit the section header or a "None." line. Instead, list all clean perspectives in a single `**Clean perspectives:** [list]` line placed before `### Contradictions Requiring Arbitration`. A fully clean review needs only Date, Status, the clean-perspectives line, and `Contradictions: None.`

```
## Design Review — Round N

**Date:** YYYY-MM-DD
**Status:** Changes requested — N blocking, N non-blocking | Approved — no blocking findings

### Cross-Repo Contract Findings  *(omit section if none, or if not a coordination-repo thread)*
- [XDEP-1] **blocking** — [finding]

### Options Analysis Findings  *(omit section if none)*
- [OPT-1] **blocking** — [finding]
- [OPT-2] **non-blocking** — [finding]

### DBA Findings  *(omit section if none)*
- [DBA-1] **blocking** — [finding]

### Security Findings  *(omit section if none)*
- [SEC-1] **blocking** — [finding]

### SRE Findings  *(omit section if none)*
- [SRE-1] **non-blocking** — [finding]

### Lead Developer Findings  *(omit section if none)*
- [LD-1] **blocking** — [finding]

**Clean perspectives:** [list any perspectives with no findings]

### Contradictions Requiring Arbitration
- [or "None."]
```

## Quality Gate

Before capturing the review output:
- [ ] *(coordination-repo threads only)* Cross-Repo Contract Review completed — all six sub-checks assessed for every XDEP
- [ ] Options Analysis step completed — all six sub-checks assessed
- [ ] All four technical perspectives completed (DBA, Security, SRE, Lead Developer) — no perspective skipped
- [ ] Each finding labeled blocking or non-blocking with a reason
- [ ] Cross-perspective check completed — contradictions identified or confirmed absent
- [ ] Round number is correct

**Self-check before proceeding:** Re-read each item above. Confirm each is complete, or state what was skipped and why before continuing to Handoff.

## Handoff

**If blocking findings exist:**
> "There are blocking findings. Run `/address-design-review DESIGN-NNNN` to address them."

**If no blocking findings:**
> "Review is clean. Run `/approve-design DESIGN-NNNN` to commit and close Stage 3."

**If this is a coordination-repo thread with only the Cross-Repo Contract findings clean (blocking findings remain elsewhere in the DESIGN):**
> "The Cross-Repo Contract has no blocking findings, though the rest of DESIGN-NNNN does. Run `/approve-design DESIGN-NNNN interface-freeze` to approve just the Cross-Repo Contract now — this unblocks every participant's `create-plan` for XDEP-only dependencies — while `/address-design-review DESIGN-NNNN` continues on the remaining findings."
