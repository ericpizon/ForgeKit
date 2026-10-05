---
id: ANALYSIS-0001
type: analysis
schema_version: 1
status: completed
target_spec_version: 3
readiness_verdict: ready
security_review_required: yes
generation:
  command: /review-spec
  command_version: "6"
  model: "claude-sonnet-5"
  generated_at: "2026-10-05"
  session_id: null
---

# ANALYSIS-0001 — Review of SPEC-0001 (Round 1)

## Review Limitation — No ARCH.md for ForgeKit's Own Repo

`c:/GitHub/ForgeKit` has never been through its own `/forge-init` — no `project.yml`, no `ARCH.md`. Per the explicit decision made before drafting SPEC-0001 ("Option 2"), this review substitutes `README.md` and the existing `templates/commands/*.md` files for the Lead Developer's feasibility check and the Security Perspective's `project.yml`-keyed checks (`stack.llm_agent_untrusted_input`, `compliance.frameworks`). Those two Security checks are marked "not independently assessable" below rather than forced into yes/no — this is a structural limitation of reviewing ForgeKit's own meta-development, not a gap in SPEC-0001 itself.

## Analysis Round 1

### Scope Coherence Check

- **Independent deliverability:** Yes. Four largely independent slices exist: **{G1}** (external intake), **{G2, G3}** (durable coordination context + the promotion mechanism that depends on having a coordination repo to promote into), **{G4}** (environment governance — no dependency on G1/G2/G3/G5), **{G5}** (Investigation workflow — no dependency on anything else). Each could ship and deliver value without the others being live.
- **AC partitionability:** Yes, cleanly — the Acceptance Criteria are already grouped exactly along these lines (Intake / Coordination context / Promotion / Environment governance / Investigation), with no cross-group data or state dependency except the intentional, already-discussed AC7↔AC8 coupling between G2 and G3.
- **Risk profile divergence:** Yes, and this sharpened during drafting: G1 now carries a live write-back path and self-assesses `security_review_required: yes`, while G2, G4, and G5 are pure governance/documentation mechanics with no comparable external surface, and G3 is a repo-internal command with no external integration at all.

```
[SC-1] non-blocking — Scope coherence: G1 is independently deliverable from {G2,G3}, G4, and G5 — and now carries a materially different risk profile (security_review_required: yes) than the other three slices.
{G2,G3}, G4, and G5 could each ship as separate SPECs without G1 being live, each delivering its own stated value independently.
Recommended split: SPEC-0001 covers G1 alone; separate SPECs cover {G2,G3}, G4, and G5 — or document the rationale for keeping all five together despite the risk-profile divergence.
TPO decision required: split or document rationale for keeping together.
```

### Lead Developer Perspective

**Section presence:** All required sections present — Problem Statement, Background, Goals, Non-Goals, User Stories, Acceptance Criteria, NFRs, Constraints & Assumptions, Security Considerations, Observability Considerations, Open Questions. No blocking gap here.

**Completeness check:** Every Goal maps to AC(s) (G1→AC1-4, G2→AC5-7, G3→AC7-12, G4→AC13-15, G5→AC16-17); every User Story maps to AC(s) (US1→AC1-4, US2→AC5-6, US3→AC9-12, US4→AC13-15, US5→AC16-17). No orphaned Goal, User Story, or AC.

**Scope consistency check:** No AC was found to imply functionality a stated Non-Goal excludes. AC4's direct-write path was checked specifically against the "no API client of its own" and "no automated write" Non-Goals — consistent, since both require externally-provided access and explicit per-SPEC confirmation.

**Ambiguity identification / completeness findings:**

```
[LD-1] blocking — AC8's logic breaks its own stated guarantee.
AC8 says the coordination-repo question is "asked exactly once, never both proactively and reactively, and never neither." But AC8 only describes reading coordination_repo when already set, or asking (per AC7's question) when unset — it never states that a reactively-obtained answer is written back into the promoting repo's own project.yml. Without that persistence, a second, later promotion of a different SPEC in the same repo would find coordination_repo still unset and ask again — directly violating the "asked exactly once" claim across the repo's lifetime, not just within one promotion.
Fix: AC8 must state that an answer obtained reactively is persisted to project.yml's coordination_repo field, identically to how AC7's proactive answer is persisted.
```

```
[LD-2] non-blocking — AC11 doesn't address the original SPEC's own git branch (created per create-spec.md's Handoff convention: `thread/NNNN-slug`) once promotion sets it to status: promoted. Left unspecified whether it's merged, abandoned, or untouched. A DESIGN-level detail, not required to resolve in the SPEC itself.
```

```
[LD-3] non-blocking — AC4 doesn't state whether the "offer to update ADO" prompt re-fires on a later re-approval of the same SPEC (via /reopen-spec → /revise-spec → /approve-spec again), or only on first approval. The underlying intent (keep the ADO ticket in sync with the current approved state) already implies "yes, re-fire" — this just isn't spelled out. DESIGN can resolve without a SPEC reword.
```

```
[LD-4] non-blocking — AC7's "not yet" answer: not stated whether this is permanently sticky or re-askable on a later /forge-init full-redo. DESIGN-level detail.
```

**Implementation feasibility check:** Checked against README.md and the existing command files (substituting for ARCH.md, per the noted limitation above). Every capability in this SPEC is new/modified command text and new frontmatter fields — nothing requires infrastructure ForgeKit's actual nature (markdown command definitions executed by an AI agent, no running service) can't support. No feasibility gap found.

**Risk flag identification:** The live write-back path (AC4) is the dominant risk — covered in depth under Security Perspective below. No additional operational or scope risk found beyond what's already named in Non-Goals/Constraints.

**NFR completeness and specificity:** All five NFRs present. Performance and Security have no numeric threshold but state why (no runtime path; confirmation-gated, not threshold-gated). Reliability, Forward compatibility, and Operability each name the downstream stage (DESIGN) where their open mechanism gets resolved, per OQ1/OQ2/OQ6. No missing NFR.

**Observability Considerations completeness:** All three concerns (metrics, traces, logs) addressed, including the "none needed and why" case and what must never be logged. No gap.

### Quality Engineer Perspective

**AC testability:**

```
[QE-1] blocking — AC16 names no command.
Every other new capability in this SPEC names at least a command-level entry point: /create-spec (G1), /forge-init's new question (G2/G3), /promote-to-thread (G3). AC16 describes what the new Investigation/decision artifact contains, but never states whether it's authored by a new standalone command or an extension of an existing one. Without a named entry point, no failing test can be written against AC16 before DESIGN — there's nothing to invoke.
Fix: state at minimum that a command exists for this (new or extended); the exact command name can still be deferred to DESIGN (consistent with OQ4 already deferring the artifact's ID sequence), but "a command exists" cannot be deferred.
```

**Missing negative and boundary ACs:**

```
[QE-2] blocking — AC4 has no write-failure path.
AC4's entire safety property rests on "show the user exactly what will be written, then write only on their confirmation." But AC4 says nothing about what happens if the write itself fails after confirmation (permission denied, network error, the ticket having changed since it was last read). A silent failure with no detection or fallback could leave the user believing the ADO ticket was updated when it wasn't — undermining the exact safety property this SPEC's own Security Considerations section relies on (see SEC-1 area below for the related compliance angle).
Fix: AC4 needs an explicit failure-path statement — report the failure clearly to the user and fall back to the copy-paste summary; never fail silently.
```

```
[QE-3] non-blocking — AC1 has no stated behavior for intake failure (invalid ticket ID, connector error mid-fetch). Presumably falls back to the copy-paste path, but this isn't stated. Lower severity than QE-2 — no write/safety property is at risk on the read side.
```

```
[QE-5] non-blocking — AC9 doesn't address double-promotion: invoking /promote-to-thread on a SPEC already status: promoted. Should be a clear no-op or explicit error, not undefined behavior.
```

**State/lifecycle coverage check:**

```
[QE-4] blocking — Status × operation coverage gap on SPEC-NNNN.
G3 states promotion can be discovered "during ANALYSIS or DESIGN." ANALYSIS runs as part of /review-spec, before a SPEC is ever approved (status: draft). DESIGN only runs after approval (status: approved). AC9 through AC12 never state whether /promote-to-thread is valid against a draft SPEC (promotion discovered during ANALYSIS, pre-approval) as well as an approved one (promotion discovered during DESIGN) — or only one of the two. This is exactly the {status} × {operation} cross-product this check exists to catch, and the cell is currently unaddressed for at least one real, reachable status value.
Fix: AC9 (or a new AC) must state explicitly whether promotion is valid from draft, approved, or both statuses, and what (if anything) differs in each case.
```

**Compound ACs:** No AC found bundling independently-failing outcomes that should be split — each multi-part AC (AC1, AC9, AC15) describes sequential facets of one coherent command behavior, consistent with how prior SPECs in this engagement have written multi-part ACs for a single command's contract.

**Test preconditions:** No AC found assuming unestablished environment state — AC8/AC9's dependency on AC7's resolved `coordination_repo`, and AC13's dependency on AC5's `environments` list, are both already correctly sequenced and cross-referenced.

**NFR testability:** All five NFRs have at least an implied verification path (interrupt-and-resume testing for Reliability; confirmation-bypass testing for Security; field-presence checks for Forward compatibility; architectural review for Operability; N/A-with-reason for Performance). No untestable NFR.

### Security Perspective

**Security Review Trigger Assessment:**

| Criterion | Yes/No | Rationale |
|---|---|---|
| Authentication/authorization | No | ForgeKit implements no auth of its own; any live access relies entirely on externally-configured credentials (per Non-Goals). |
| PII/sensitive data | **Yes** | A `.source.md` companion file (AC2) can inherit PII from the underlying PRD it captures. See SEC-1. |
| Secrets/credential management | **Yes** | Live access, when available, depends on externally-configured credentials ForgeKit doesn't manage — but whose scope and failure modes now directly affect this SPEC's behavior (AC1, AC4). |
| Third-party integrations | **Yes** | AC1/AC4's capability-aware access to ADO/Confluence is a third-party integration touching an external system of record, even when confirmation-gated. |
| Data retention/compliance | **Yes** | See SEC-1 — captured content may carry retention/erasure obligations this SPEC doesn't yet resolve. |
| LLM/agent input from untrusted/third-party text | Not independently assessable | No `project.yml` exists for ForgeKit's own repo to check `stack.llm_agent_untrusted_input` against (see Review Limitation above). |
| Regulatory/compliance framework scope | Not independently assessable | Same limitation — any adopting team's own `compliance.frameworks` would need checking at their own adoption time, not at ForgeKit's meta-level. |

**Findings:**

```
[SEC-1] blocking — Security Considerations names the PII-inheritance risk of .source.md but not the property that makes it categorically different from ordinary risk.
The current text says committing a .source.md companion file "inherits" whatever risk the underlying PRD carries, and that this "should be flagged explicitly... not assumed away" — but names no actual mitigation. The missing property: once committed, a file is effectively impossible to reliably un-commit from git history in practice (rewriting history after the fact doesn't retract what's already been cloned, mirrored, or cached elsewhere). This directly conflicts with any "right to erasure" a compliance framework (e.g. GDPR) might require if regulated data ends up captured this way. This is a materially different, more severe risk than "could inherit PII" alone — it's a potential irreversible exposure, not a correctable one.
Fix: Security Considerations must name this irreversibility explicitly and state a concrete mitigation direction for DESIGN to resolve — e.g., warn the user before capture whenever the adopting project's own compliance.frameworks is non-[none], and require an explicit "this does not contain regulated data" confirmation before the companion file is written, not merely offered as a vague future caveat.
```

**Combined `security_review_required`: yes** — matches SPEC-0001's own self-assessment; confirmed independently via the trigger checklist above (three criteria fire independently: PII, credential dependency, third-party integration).

## Readiness Verdict

**`needs_revision`.** Five blocking findings — `[LD-1]`, `[QE-1]`, `[QE-2]`, `[QE-4]`, `[SEC-1]` — each requires adding or rewording specific Acceptance Criteria or the Security Considerations section itself, not merely a DESIGN-level implementation choice. Per the combined-verdict rule, this cannot be `ready` until each is addressed, accepted as risk with rationale, or deferred with documented reasoning.

**Non-blocking, carried forward for the TPO's awareness (not required before `/revise-spec`):** `[SC-1]`, `[LD-2]`, `[LD-3]`, `[LD-4]`, `[QE-3]`, `[QE-5]`.

## Analysis Round 2

**Scope Coherence:** Not re-litigated — Round 1's `[SC-1]` was addressed, not deferred, via the TPO's explicit "keep together" decision now documented in SPEC-0001's Background. A documented decision, per the On Re-Review process, is not re-raised.

**Confirmed resolved per SPEC-0001 v2's Revision Summary — Round 1:** `[LD-1]` (AC8 persistence — now correctly required), `[LD-3]` (AC4 re-fire — now stated), `[QE-1]`-in-spirit (see new `[LD-1]` below — the fix introduced a wording self-contradiction, not a failure to address the underlying gap), `[QE-2]` (AC4 write-failure path — now stated), `[QE-3]` (AC1 intake-failure fallback — now stated), `[QE-4]`/`[QE-5]` (AC9 status coverage and double-promotion — both now stated).

**New findings — introduced by the Round 1 revision itself, not present before it:**

### Lead Developer Perspective

```
[LD-1] blocking — AC16's own wording is now self-contradictory.
AC16 opens with "A new command authors a standalone artifact type..." — presupposing the command is new. The very next sentence says "Whether this command is wholly new or an extension of an existing one... are deferred to DESIGN." These two statements can't both be true as written: either it's new (first sentence) or that's still an open question (second sentence). The Round 1 fix correctly added "a command exists" but accidentally over-committed to "new" in the same breath.
Fix: reword the opening to not presuppose "new" — e.g., "A command authors a standalone artifact type..." — consistent with the second sentence's actual deferral.
```

```
[LD-2] non-blocking — Non-Goals' "every write (AC4) requires the user's explicit, per-SPEC confirmation" could now be misread as "confirmed once per SPEC's entire lifetime," given AC4 was just revised to explicitly fire on every re-approval (potentially multiple times across reopen/revise/reapprove cycles for the same SPEC). AC4's own text is unambiguous; this is a cross-reference clarity nit in Non-Goals, not a behavioral gap. Suggest "per-write confirmation" or "per-approval confirmation" if revised.
```

### Security Perspective

```
[SEC-1] blocking — AC2 doesn't implement Security Considerations' own Round 1 mitigation.
Security Considerations (SEC-1, Round 1) now requires: "when the adopting project's own compliance.frameworks is non-[none], warn the user before writing a .source.md file and require an explicit 'this does not contain regulated data' confirmation before capture — never capture automatically in that case." But AC2 — the Acceptance Criterion that actually defines capture behavior — was not updated to reflect this. As written, AC2 still describes unconditional capture whenever no connector access exists, with no compliance check anywhere in its text. A mitigation stated only in prose, with no corresponding AC a test could be written against, is not enforceable — exactly the AC-testability bar this review applies everywhere else.
Fix: AC2 must incorporate the compliance-aware confirmation gate from Security Considerations directly, not leave it as an unreferenced parallel requirement.
```

**Security Review Trigger Assessment:** Unchanged from Round 1 — PII/sensitive data, secrets/credential management, third-party integrations, and data retention/compliance all still fire `yes`, for the same reasons already stated. `security_review_required: yes` stands.

**Readiness Verdict:** `needs_revision`. Two blocking findings (`[LD-1]`, `[SEC-1]`), both introduced by the Round 1 fix itself rather than carried over — the underlying gaps Round 1 targeted are genuinely closed; these are new, narrower issues the fix itself created. One non-blocking item (`[LD-2]`) carried forward for awareness, not required before the next revision.

## Analysis Round 3

Re-reviewed SPEC-0001 v3 against Round 2's Revision Summary. Checked specifically for the failure mode Round 2 itself caught in Round 1's fix: a repair that closes the named gap but introduces a new one nearby.

**Confirmed resolved:** `[LD-1]` (AC16's self-contradiction — now reads "A command authors..." without presupposing new vs. extended) and `[SEC-1]` (AC2 now directly incorporates the compliance-aware confirmation gate, not left as unenforced prose) are both genuinely closed — re-checked AC16 and AC2's literal text, not just the revision summary's description of them. `[LD-2]` (Non-Goals wording) is also resolved — "per-approval confirmation" with an explicit pointer to AC4 removes the "once ever" misreading.

**Checked the self-caught AC3 fix for correctness too** (not required, since it wasn't a flagged finding, but it changed AC text): AC3 now names both keying factors (access existence, and compliance confirmation outcome when access doesn't exist) and remains consistent with AC2's actual three-way branch, Security Considerations, and both prior Revision Summaries. No drift found between them.

**Hunted specifically for a third-order repeat of the Round 1→Round 2 pattern** (a fix to AC2 breaking something else nearby): checked AC1 (still consistent — governs obtaining content, not capturing it, no overlap with AC2's new branch), AC4 (writing an approved SPEC's own authored summary back out carries a different risk profile than capturing raw external content in — correctly does not need the same compliance gate, and doesn't claim to), the Forward Compatibility NFR (still holds — the schema's core fields are unaffected by the compliance branch), and Constraints & Assumptions (no stale reference to the pre-Round-2 capture model). No new issue found.

**Lead Developer, Quality Engineer, Security Perspectives:** No new findings this round.

**Security Review Trigger Assessment:** Unchanged — `security_review_required: yes` stands, same reasoning as Rounds 1 and 2.

**Readiness Verdict:** `ready`. All blocking findings across all three rounds are resolved; remaining non-blocking items (`[SC-1]`'s disposition, `[LD-2]`/`[LD-4]` from Round 1, OQ1–OQ9) are either already addressed via explicit TPO decision or deferred to DESIGN with no remaining impact on this SPEC's Goals, Non-Goals, or Acceptance Criteria.
