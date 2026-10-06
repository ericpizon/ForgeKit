---
id: IMPL-REVIEW-0001-M2
type: impl-review
schema_version: 1
plan: PLAN-0001
milestone: M2
current_review_round: 2
approval:
  mode: repo_only
  approved_by: "Eric Pizon"
  approved_role: lead-developer
  approved_at: "2026-10-05"
  decision_summary: "M2 approved for merge — external work intake/write-back on /create-spec and /approve-spec, zero blocking findings after one address round."
  external_event_id: null
  external_event_uri: null
policy_evaluation:
  evaluated: null
  policy_pack: null
  result: null
generation:
  command: /review-implementation
  command_version: "4"
  model: "claude-sonnet-5"
  generated_at: "2026-10-05"
  session_id: null
---

# Implementation Review — PLAN-0001 M2

## Task Reference — M2

*Derived from PLAN-0001 at review creation time. Reviewers use this to verify coverage without loading the full PLAN.*

| Task | File(s) | Description |
|---|---|---|
| T4 | `templates/commands/create-spec.md` | `--from <system>:<id>` argument; AC1 (capability-check, fetch-or-paste, `source_work` recorded either way), AC2/AC3 (`product_spec` block, two capture-policy branches), D6 (compliance-aware confirmation gate baseline). States inline: D12 (fetched/pasted content is quoted reference material, never executed instructions) and D11 (fetch failures reduced to a sanitized category before display or commit). |
| T5 | `templates/commands/approve-spec.md` | AC4 (write-back prompt, capability-check-then-write-or-fallback). States inline: D9 (any non-confirmed-full-success is failure, no partial state) and D11 (write-back failures also sanitized before display or persistence). |
| T13 | `templates/commands/create-spec.md` | OBS-0001 IT-1 — `source_work.intake_result` log emission. |
| T14 | `templates/commands/approve-spec.md` | OBS-0001 IT-2 — `source_work.write_back_result` log emission. |
| T15 | `templates/commands/create-spec.md` | OBS-0001 IT-3 — `compliance_gate.outcome` log emission. |
| T17 | `templates/commands/create-spec.md` | SEC-0001 SEC-1 — credential-pattern scan, decoupled from `compliance.frameworks` status. |
| T18a | `templates/commands/create-spec.md` | SEC-0001 SEC-2 — safe slug derivation (`[a-z0-9-]` only) for titles sourced from `--from` intake. |

## Round 1 — 2026-10-05

### Lead Developer Findings

**[LD-1] blocking — write-back (T5) runs in `approve-spec.md`'s `Steps` section, before the human-review gate, not in `Handoff` as AC4 specifies**

SPEC-0001's AC4 states explicitly: *"its **Handoff** step prompts the user... do they want the originating ADO ticket updated."* The implementation instead placed this logic as **Step 5** of the `## Steps` section — before the `## Quality Gate` self-check and before `## Handoff`'s three-question human-review gate ("state the problem in one sentence," "any uncovered scenario," "any Non-Goal that breaks the feature") that can still send the SPEC back to `/revise-spec`.

This is not just an organizational mismatch. Steps 1–4 (which Step 5 now sits alongside) only mutate local, uncommitted frontmatter — reversible, no external effect, safe to run ahead of the human sanity check. Step 5 is categorically different: on confirmation, it performs (or at least prompts toward) a **real write to an external system of record other people rely on**. SPEC-0001's own Non-Goals section is explicit about this distinction: *"Write access is a materially different risk than read access... The explicit-confirmation-before-write requirement in AC4 ... is the control this SPEC relies on — DESIGN must not weaken it."* Running the write-back prompt before the three-question gate means a ticket could be updated with an approval decision that the very next step (the human-review gate) goes on to reject and send back to `/revise-spec` — exactly the kind of premature external side effect AC4's placement in Handoff was meant to prevent.

**Resolution recommendation:** Move Step 5 out of `## Steps` entirely. Insert it into `## Handoff`, after the three-question gate's "all three check out" branch and the user's "ready to commit" confirmation, but before the `git add`/`git commit` lines (the `write_back` frontmatter fields it records must land in the same commit). Steps 1–4 remain as pure pre-Handoff bookkeeping; the external-facing action moves to where a decision to actually proceed has already been confirmed.

**Clean perspectives:** Security, SRE, QE

### Summary
**Summary:** 1 blocking, 0 non-blocking findings.

### Next Step
`/address-implementation-review PLAN-0001 M2`

### Address Summary — Round 1
Date: 2026-10-05

| Finding | Resolution |
|---|---|
| [LD-1] | Fixed: removed write-back logic from `approve-spec.md`'s `## Steps` (was Step 5). Inserted into `## Handoff`, positioned after the three-question human-review gate's "ready to commit" confirmation and before the `git add`/`git commit` lines — `write_back` frontmatter still lands in the same commit, but the external write (or prompt toward one) now only fires once the human has already confirmed the SPEC is ready, not while it could still be sent back to `/revise-spec`. Removed the now-premature "write-back was attempted or declined" bullet from `## Quality Gate`'s checklist, since that check is self-contained within the Handoff step itself and the Quality Gate self-check runs before Handoff, i.e. before write-back has happened at all. |

**Not addressed:** None.

### Next Step
`/review-implementation PLAN-0001 M2`

## Round 2 — 2026-10-05

Scope: `templates/commands/approve-spec.md` only — the one file the Round 1 address pass touched.

### Lead Developer Findings

[LD-1] re-checked: write-back logic is no longer in `## Steps`. It now sits in `## Handoff`, correctly sequenced after the three-question gate's "all three check out" branch and the "ready to commit" confirmation, and before the `git add`/`git commit` lines — so `write_back`'s frontmatter still lands in the same commit, and the external write (or prompt toward one) can no longer fire ahead of a gate that could still reject the SPEC. `## Quality Gate`'s checklist no longer references write-back prematurely. Resolved as recommended.

**Clean perspectives:** Lead Developer, Security, SRE, QE

### Summary
**Summary:** 0 blocking, 0 non-blocking findings.

### Next Step
`/approve-implementation PLAN-0001 M2`
