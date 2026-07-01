---
id: validate-spec
type: command
name: /validate-spec
description: TPO command to validate product intent before design begins — translates ACs to plain language, asks structural and AC-specific questions, records named sign-off, gates create-design
status: active
version: 1
primary_role: tpo
output_artifact_type: spec (validation section appended)
links:
  - relation: follows
    target: review-spec
  - relation: precedes
    target: approve-spec
---

# /validate-spec

## Purpose

Confirm that the SPEC accurately captures the intended product before the Architect begins design. Design decisions made against a misunderstood SPEC are expensive to unwind — this command surfaces that misunderstanding now.

This is a human-in-the-loop step. The AI produces plain-language translations of every AC and asks two tiers of questions. A named human must answer. The sign-off is recorded in the SPEC and gates both `/approve-spec` and `/create-design`.

## When to Invoke

After the last round of `/review-spec` shows `readiness_verdict: ready`. Before `/approve-spec`.

Usage: `/validate-spec SPEC-NNNN`

## Required Inputs

- **SPEC-NNNN** — all revision rounds complete; latest ANALYSIS shows `readiness_verdict: ready`

## Steps

### Step 1 — Read SPEC-NNNN

Read the full SPEC. Extract:
- Problem Statement
- Goals list
- Non-Goals list
- All Acceptance Criteria (AC text verbatim)

### Step 2 — Translate each AC to plain language

For every AC, produce a paired block. The original is shown first and never omitted:

```
**AC-N (original):** [verbatim text from SPEC-NNNN]
**In plain terms:** [translation — describe the user outcome; no implementation terms, no technical jargon, no reference to system internals]
```

A good plain-language translation answers: "When a user does X in situation Y, they expect Z to happen." If the AC is too ambiguous to translate cleanly, flag it — that is itself a finding.

### Step 3 — Show translations and ask for correction

Present all AC translations to the user. Ask:

> **Please review each AC translation below.**
>
> These are the acceptance conditions in plain language — verify that each translation accurately describes the intended user outcome. If any translation is wrong or incomplete, correct it now. We will use these translations as the basis for validation questions.

Wait for the user to confirm or correct. Update any translations the user corrects before proceeding. Do not move to questions until all translations are confirmed.

### Step 4 — Tier 1: Structural questions (mandatory)

Ask the following three questions. These apply to every spec regardless of content:

> **Structural questions — answer each one:**
>
> 1. Read the Problem Statement. Is it describing user pain, or has it already prescribed a solution? (A solution-prescribing problem statement forecloses design options unnecessarily.)
>
> 2. Read the Non-Goals. Could a developer quote any of them and use them as a decision rule during implementation? If a Non-Goal is too vague to apply without asking the TPO, it needs to be more specific.
>
> 3. Is there a user scenario not covered by any AC that a user of this feature would reasonably expect to work? Think about error paths, edge cases, and multi-step flows — not just the happy path.

Wait for answers to all three before proceeding.

### Step 5 — Tier 2: AC-specific questions (one per AC)

For each AC, ask one targeted question derived from the plain-language translation:

> **AC-N:** Is "[plain-language translation]" the complete definition of done — or is there an implied condition that isn't stated? (For example: a response time requirement, a data persistence guarantee, a specific failure behavior, or a permission boundary.)

Present all Tier 2 questions together. Wait for the full set of answers before proceeding.

### Step 6 — Surface gaps

If any answer reveals a gap:
- State the gap clearly
- Ask: "Does this gap require returning to `/revise-spec SPEC-NNNN ANALYSIS-NNNN`, or do you want to accept it as documented risk with a rationale?"
- If revision: stop here. Do not record sign-off. Direct to revise-spec.
- If accepted risk: record the rationale in the Gaps Found section of the Product Validation output.

### Step 7 — Record sign-off

If no gaps require revision, ask:

> **Who is signing off on this validation?**
> Provide: name and role (e.g., "Alex Chen — Product Owner").
>
> If the product owner is not available right now: provide "async — [name]" and we will record this as pending. Note: pending validation blocks `/approve-spec`.

### Step 8 — Update SPEC-NNNN

**Frontmatter** — add or update these fields:

```yaml
product_validated: true        # or: pending
validated_by: "[name] — [role]"  # or: "pending — [name]"
validated_date: YYYY-MM-DD     # or: null (if pending)
```

**Body** — append a `## Product Validation` section:

```markdown
## Product Validation

**Validated by:** [name] — [role]
**Date:** YYYY-MM-DD
**Status:** validated | pending sign-off from [name]

### Acceptance Criteria — Plain Language Review

**AC-1 (original):** [verbatim]
**In plain terms:** [confirmed translation]

[repeat for each AC]

### Structural Questions

1. **Problem statement describes pain, not solution?**
   [answer]

2. **Non-Goals specific enough to use as decision rules?**
   [answer]

3. **Uncovered user scenario?**
   [answer or "None identified."]

### AC-Specific Questions

**AC-1:** Is "[plain terms]" the complete definition of done, or is there an implied condition not stated?
[answer]

[repeat for each AC]

### Gaps Found

[List any gaps surfaced and resolution — revised, accepted risk with rationale, or deferred. If none: "None."]

### Sign-off

> [name] confirms: the plain-language translations are accurate and the questions above are answered. This SPEC is ready for design.
```

For pending sign-off, replace the Sign-off block with:

```
> Validation pending sign-off from [name]. The Q&A above is recorded. Once [name] confirms, update SPEC frontmatter: `product_validated: true`, `validated_date: YYYY-MM-DD`.
```

## Quality Gate

Before closing:
- [ ] All AC translations shown alongside original verbatim text — no original omitted
- [ ] Tier 1 structural questions answered by a human, not assumed by the AI
- [ ] Tier 2 per-AC question answered for every AC in the SPEC
- [ ] All gaps either returned to revise-spec, accepted as risk with rationale, or deferred with rationale
- [ ] Sign-off name and role recorded (or `pending — [name]` with explicit acknowledgment)
- [ ] SPEC frontmatter: `product_validated`, `validated_by`, `validated_date` all set
- [ ] `## Product Validation` section appended to SPEC-NNNN with complete Q&A record

**Self-check before proceeding:** Re-read each item above. Confirm each is complete, or state what was skipped and why before continuing to Handoff.

## Handoff

**If gaps were found that require revision:**
> "Returning to revise-spec. Run `/revise-spec SPEC-NNNN ANALYSIS-NNNN` to address the gap. After the next review round is clean, re-run `/validate-spec SPEC-NNNN`."

**If `product_validated: true`:**
> "Product validation complete. Signed off by [name] on [date]. Run `/approve-spec SPEC-NNNN ANALYSIS-NNNN` to commit and close the spec phase."

**If `product_validated: pending`:**
> "Validation recorded, pending sign-off from [name]. Once confirmed, update SPEC frontmatter and run `/approve-spec SPEC-NNNN ANALYSIS-NNNN`. Note: approve-spec will block until product_validated is true."
