---
id: create-design
type: command
name: /create-design
description: Architect command to produce DESIGN-NNNN — options analysis with recommendation, then full technical spec for the confirmed option
status: active
version: 1
primary_role: architect
reviewing_roles:
  - dba
  - security
output_artifact_type: design
---

# /create-design

## Purpose

Produce the technical architecture for a feature: options analysis, recommended approach with explicit reasoning, component boundaries, data model, API surface, key decisions, and any ADRs warranted by cross-cutting choices. The output is DESIGN-NNNN (and zero or more ADR-NNNN files) that unblock parallel Stage 4 work (Observability Design, Security Review) and ultimately PLAN-NNNN.

The options analysis runs before the full spec is written — the user confirms the recommended option before the Architect invests in the detailed design. This prevents re-design loops caused by reviewers asking "did you consider X?"

## When to Invoke

- SPEC-NNNN is `approved` and ANALYSIS-NNNN is `completed` with `readiness_verdict: ready`
- SPEC-NNNN `product_validated: true`

If `product_validated` is `false` or missing: **do not proceed.** "Product validation not complete. Run `/validate-spec SPEC-NNNN` first."
If `product_validated: pending`: warn the user — "Validation is recorded but sign-off is still pending from [name]. Confirm explicitly that you want to proceed before starting design." Do not proceed without explicit confirmation.

## Required Inputs

- **SPEC-NNNN** — approved
- **ANALYSIS-NNNN** — completed; read ambiguities and risk flags — each must be resolved or documented in the design
- **`sdlc/architecture/ARCH.md`** — read first; patterns established there are not up for re-decision without a new ADR

Pass artifact IDs as arguments: `/create-design SPEC-NNNN ANALYSIS-NNNN`.

---

## Phase 1 — Foundation

### Step 1 — Read ARCH

Confirm the current system baseline before designing. Any pattern already established in ARCH is a constraint, not a choice. Note established patterns for data access, module layout, API conventions, and auth — these scope what is genuinely open for decision in the options analysis.

### Step 2 — Resolve ANALYSIS ambiguities

For each ambiguity listed in ANALYSIS-NNNN, make an explicit decision. Document the decision and rationale — these feed into the Key Decisions section or an ADR if cross-cutting. Do not defer ambiguities into the design.

---

## Phase 2 — Options Analysis

### Step 3 — Assess scope and option count

Read ANALYSIS-NNNN's risk flags, complexity signals, and existing-system integration points to determine how many options to generate:

| Scope | Signals | Options |
|---|---|---|
| **Simple** | Follows an established ARCH pattern; no risk flags; no significant existing code integration | 2 — obvious approach + one alternative |
| **Moderate** | New data model, new API surface, new external dependency, or 1–2 risk flags | 2 — full trade-off analysis on both |
| **Complex** | Touches existing systems, legacy integration, multiple risk flags, `security_review_required: yes`, or multiple viable architectures with meaningfully different tradeoffs | 3 |

For a brownfield or large codebase, pay particular attention to: whether existing code can be extended vs. wrapped vs. replaced, whether existing tables can absorb the new data or require a new table, and whether a new service boundary is warranted vs. extending the monolith.

### Step 4 — Generate options

For each option, produce all five fields. Thin options with one-line pros/cons are a quality gate failure.

**Name** — concise label that distinguishes the approach (e.g., "Extend existing users table" not "Option A")

**Approach** — 1–2 sentences: how it solves the problem and what makes it architecturally distinct from the alternatives

**Existing code integration** — what this option extends, wraps, replaces, or works around in the current codebase; for new projects with no existing code, state "greenfield — no existing code integration"

**Pros** — 2–4 bullets; be specific (e.g., "no schema migration needed" not "simpler")

**Cons** — 2–4 bullets; be honest about real costs, not hypothetical ones

**Risk** — Low / Medium / High — assess migration risk, rollback complexity, and security surface change

### Step 5 — State recommendation and get confirmation

After all options are drafted, state the recommendation:

- Name the recommended option
- Explain *why* with explicit references to SPEC ACs or Non-Goals, ANALYSIS risk flags, and ARCH patterns — "this is simpler" is not sufficient reasoning
- Name the trade-offs accepted by not choosing the alternatives
- If two options are genuinely equivalent, say so and state the tiebreaker

Then present to the user before writing any further:

> **Design Options — [N options]**
>
> **Option A: [Name]**
> Approach: ...
> Existing code integration: ...
> Pros: ...
> Cons: ...
> Risk: ...
>
> **Option B: [Name]**
> ...
>
> **Recommended: Option [X]**
> Reasoning: [explicit citation of SPEC/ANALYSIS/ARCH constraints]
> Trade-offs accepted: [what we give up by not choosing the alternatives]
>
> Confirm to proceed with the full spec for Option [X]? Or specify a different option.

Wait for user input. If the user selects a different option, update the recommendation and re-present. Do not write the full spec until an option is confirmed.

---

## Phase 3 — Full Design Spec (confirmed option only)

### Step 6 — Define system boundaries

Identify the distinct request surfaces and their authentication models. Verify they share only the appropriate shared resources (database, cache) and not a security boundary.

### Step 7 — Design the data model

Define tables/collections, columns/fields, types, constraints. Identify the queries that will run on each path (especially hot paths) and confirm indexes support them.

### Step 8 — Design the API surface

Define endpoints or service interfaces, HTTP methods (or equivalent), request/response shapes, and status codes for each outcome. External API surfaces must include a version prefix. Read ARCH.md for the project's API conventions.

### Step 9 — Identify ADR candidates

Any decision that: (a) constrains future design choices, (b) chooses between real alternatives with different tradeoffs, or (c) answers a question that will be asked again. Author ADR-NNNN for each.

The options analysis from Phase 2 feeds directly into ADR content — non-recommended options become the `Alternatives Considered` section of the ADR.

### Step 10 — Enumerate failure modes

For each component, state what happens when it fails. Particularly: what happens when the database or a critical external dependency is unavailable? Document the fail-closed vs. fail-open decision explicitly.

### Step 11 — Consider forward compatibility

Does this design make future work harder than necessary? If so, either adjust or document the constraint in an ADR.

### Step 12 — Identify ARCH updates

Note any cross-cutting patterns introduced by this DESIGN that must be reflected in ARCH upon approval. For each pattern, state: (a) does it extend an existing ARCH pattern, replace one, or introduce something entirely new? (b) If it replaces an existing pattern, name the specific ARCH section that will be superseded — this is required for the ARCH update in `/approve-design`.

---

## Output Contract

DESIGN-NNNN must contain all Required Sections in this order:

1. **Design Options** — all options from Phase 2 (minimum 2; each with all five fields)
2. **Recommended Option** — stated recommendation with explicit reasoning and accepted trade-offs
3. **Architecture Overview** — full spec for the confirmed option
4. **Component / Data Model Changes**
5. **Interface / API Changes**
6. **Key Decisions** (with ADR pointers)
7. **Alternatives Considered** — may reference Design Options for decisions captured in ADRs
8. **Impact on Existing Systems**

Plus optional sections as warranted: Performance Considerations, Observability Implementation.

DESIGN-NNNN frontmatter must include: `schema_version: 1`, and a `generation` block: `command: /create-design`, `command_version: [use version from this command's frontmatter]`, `model` [self-identify from system context], `generated_at: [today's date]`, `session_id: null`.

Each ADR-NNNN must contain: Context, Decision, Consequences, Status Rationale, Revisit Triggers.

## Quality Gate

**End of Phase 2 (before showing options to user):**
- [ ] Minimum 2 options generated
- [ ] Each option has all five fields with substantive content — not one-liners
- [ ] Recommended option reasoning explicitly cites SPEC constraints, ANALYSIS risk flags, or ARCH patterns
- [ ] Trade-offs accepted are named explicitly

**End of Phase 3 (before showing full spec to user):**
- [ ] DESIGN frontmatter initialized: `current_review_round: 0` (updated by `/review-design` after each round)
- [ ] `schema_version: 1` present in DESIGN frontmatter
- [ ] `generation` block present in DESIGN frontmatter: command, command_version, model self-identified, generated_at set, session_id: null
- [ ] Every ANALYSIS ambiguity is explicitly resolved
- [ ] External API surfaces have version prefixes
- [ ] All failure modes for the hot path are enumerated
- [ ] Every Key Decision that constrains future work has an ADR
- [ ] Cross-cutting patterns this DESIGN introduces to ARCH are noted

**Self-check before proceeding:** Re-read each item above. Confirm each is complete, or state what was skipped and why before continuing to Handoff.

## Handoff

### After options confirmation

User has confirmed the option. Proceed immediately to Phase 3 — write the full design spec for the confirmed option.

### After full spec is drafted

Show the complete DESIGN and any ADR files to the user:

> **DESIGN-NNNN drafted.**
> Options analyzed: [N]. Recommended and confirmed: Option [X].
> Please review the full spec. Confirm when ready to proceed.

Once the user confirms:
→ `/review-design SPEC-NNNN DESIGN-NNNN`

Nothing is committed yet. `/review-design` runs all reviewer perspectives, captures results, and handles the commit once the review comes back clean.
