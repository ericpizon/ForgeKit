---
id: ANALYSIS-0001
type: analysis
schema_version: 1
title: "ForgeKit Operator — Spec Analysis"
status: completed
target_spec_version: 3
readiness_verdict: ready
security_review_required: yes
generation:
  command: /review-spec
  command_version: 6
  model: claude-sonnet-5
  generated_at: 2026-10-07
  session_id: null
---

# ANALYSIS-0001 — ForgeKit Operator

## Analysis Round 1

### Section Presence Check

Required sections per the SPEC quality bar: Problem Statement, Goals, Non-Goals, User Stories, Acceptance Criteria, Non-Functional Requirements, Constraints & Assumptions, Security Considerations, Observability Considerations, Open Questions.

Present (under different headings, content judged equivalent): **Problem Statement** (§1 Purpose), **Non-Goals** (§5), **Acceptance Criteria** (§35, 24 ACs — strong).

**Missing or not in the required shape:** Goals, User Stories, Non-Functional Requirements, Constraints & Assumptions, Security Considerations, Observability Considerations, Open Questions. Per `/review-spec`'s own rule, a missing required section is a blocking finding in its own right — findings `[LD-1]` through `[LD-7]` below cover each one individually, since each has a distinct, substantive gap rather than being a single uniform omission.

### Scope Coherence Check

`[SC-1]` **non-blocking** — Scope coherence: §41 ("Operator Readiness Prerequisites") names eight control-surface changes to *existing* ForgeKit artifacts (structured review outcome, `/status` extension, finding lineage, implementation-execution completeness, BUG canonicalization, test classification, release-approval binding, release resumability) as things that must exist "before autonomous `/operate` execution is considered ready." These are independently deliverable and independently valuable: every one of them makes `/status` and the existing review/approval gates more reliable for a human operator today, with zero dependency on `/operate` itself existing. They also have a materially different risk profile — small, additive fields on artifacts ForgeKit already produces — versus `/operate` itself, which is a new, higher-autonomy command surface with its own governance stakes (policy schema, run identity, escalation model). Recommended split: a SPEC covering §41's prerequisites only (shippable and useful standalone), with this SPEC narrowed to the Operator command surface that consumes them. TPO decision required: split, or document the rationale for keeping them in one SPEC (e.g., if the prerequisites are intended to ship as PLAN tasks *within* this same thread rather than as their own thread).

### Lead Developer Perspective

`[LD-1]` **blocking** — No **Goals** section exists. §2 (Product Outcome), §3 (Core Principle), and §4 (Scope) together describe outcome and principle but are not stated as discrete, numbered Goal statements. Without an explicit Goals list, the required bidirectional traceability check (every Goal → at least one AC, every AC → at least one Goal) cannot be performed at all — not just incompletely. The 24 ACs in §35 are strong, but they are currently unanchored.

`[LD-2]` **blocking** — No **User Stories** section exists anywhere in the SPEC. At minimum, "developer operating a work unit," "Lead Developer/reviewer being interrupted for a decision," and "SRE approving a release Operator advanced to" are distinct roles with distinct capabilities and "so that" value this SPEC implies but never states explicitly.

`[LD-3]` **blocking** — No **Non-Functional Requirements** section exists. NFR-shaped content is scattered (§13 Idempotency and Resume, §26 Parallelism, §30 Release Resumability) but none are organized under the required NFR framing (performance, reliability/consistency, security, forward compatibility, operability), and none state a numeric threshold or explicitly name the downstream stage where one will be resolved, as the quality bar requires.

`[LD-4]` **blocking** — No **Constraints & Assumptions** section exists. Material assumptions are never stated: does Operator assume a human remains reachable to manually re-invoke it after every stop (today's model), or that something else re-triggers it? Does Operator assume no human-driven ForgeKit command runs concurrently against the same work unit while an Operator run is active? Does it assume exactly one Operator run per work unit at a time? These are exactly the kind of environmental assumptions this section exists to pin down before DESIGN.

`[LD-5]` **blocking** — No **Security Considerations** section exists. §32 (Safety Invariants) and §16 (Human Decision Boundaries) cover security-adjacent governance concerns (risk acceptance, policy exception) but never address credential handling, auth surface, or PII in the form this section requires — see also the Security Perspective findings below, which identify a real trigger.

`[LD-6]` **blocking** — No **Observability Considerations** section exists in the required three-part shape (metrics, traces, logs — including what must never be logged). §34 (Metrics) addresses Operator's own evaluator-facing metrics only, and is framed as a feature capability (feeding AC-24) rather than this required SPEC section. Traces are not mentioned at all. Log redaction is not mentioned at all, which matters given the Security Perspective finding below about decision packages potentially embedding externally-sourced content.

`[LD-7]` **blocking** — No **Open Questions** section exists, despite at least six places in the SPEC explicitly deferring a decision to DESIGN without collecting them: §7 ("exact schema may be refined during DESIGN"), §12 ("exact representation may change during DESIGN"), §18 ("names may be refined during DESIGN"), §29 ("exact schema is a DESIGN decision"), §30 ("exact implementation belongs in DESIGN"), §31 ("exact schema is not defined by this SPEC"). These are genuine open questions and the quality bar requires them named in one place, each with the stage that resolves it — which, here, is consistently DESIGN, but that itself should be stated once rather than six times implicitly.

`[LD-8]` **blocking** — The SPEC never states the execution model `/operate` actually runs under, and this is a real feasibility question, not a stylistic gap. ARCH.md explicitly characterizes this repo as having "no language runtime... no deploy target, no CI," and every existing ForgeKit command's Handoff ends with an explicit human-confirmation pause before any commit. §37 (Recovery Scenario) and AC-09 describe resumption where "a new Claude session begins" — implying execution may span multiple, separately-invoked sessions — but the SPEC never states whether (a) a human is expected to notice an interruption and manually re-invoke `/operate` each time (fits today's all-human-triggered model, requires no new infrastructure), or (b) something is expected to detect interruption and automatically re-trigger Operator unattended (a scheduler/daemon/webhook ForgeKit has never had, and which ARCH.md's own "no CI, no deploy target" framing argues against introducing silently). This materially changes feasibility and governance posture and must be resolved at the SPEC level, not left implicit for DESIGN to discover.

`[LD-9]` **blocking** — The SPEC never states which of ForgeKit's existing "please review before I commit" Handoff pauses — present in nearly every command (`create-spec`, `design-observability`, `create-plan`, etc.) for ordinary draft review, distinct from the formal `/approve-*`/`/reopen-*` gates — remain mandatory under autonomous Operator policy versus become autonomy-skippable. §16 lists categories like "explicit approval," which clearly covers the formal gates (and AC-13 confirms those are preserved), but says nothing about the softer, nearly-universal "confirm to proceed with this commit" interaction point that is not itself a named ForgeKit gate anywhere else. An implementer could reasonably read this SPEC either way, and the two readings carry very different risk profiles — this needs an explicit position, likely in the new Constraints & Assumptions or Safety Invariants section.

`[LD-10]` **non-blocking** — Risk flag: no circuit breaker is defined on *total* autonomous side effects per run — only `max_review_rounds` (§31) is mentioned. A state-resolution bug could otherwise produce unbounded real commits, branches, or PRs before a human notices. Recommend a global per-run action cap and/or a first-run dry-run/preview mode as a DESIGN-level safeguard; not necessarily a SPEC change, but worth noting as a Risk Flag for DESIGN to inherit.

`[LD-11]` **non-blocking** — §31 introduces a new Autonomy Policy block but doesn't say where it's configured. ForgeKit already has one per-project config file (`project.yml`, which already carries a `pipeline:` block for comparable pipeline-wide settings) — worth confirming at DESIGN that this nests there rather than becoming a second config file, purely for consistency with existing convention. Not blocking since it's explicitly deferred to DESIGN already.

### Quality Engineer Perspective

`[QE-1]` **blocking** — No AC or stated invariant covers the *absence* or malformed state of the Autonomy Policy itself. §31 states policy "may only make execution more restrictive" and "must never override a stricter ForgeKit gate," but never states the default behavior when no policy has been configured yet — e.g., the very first `/operate` invocation on a project that has never set one up. A fail-open default (treat an unset `require_human.*` as `false`, i.e., not required) would be a materially dangerous default. The SPEC should state a fail-closed default explicitly — no policy configured means everything defaults to requiring human confirmation — and this needs its own AC alongside AC-13/AC-20, not just an implication.

`[QE-2]` **non-blocking** — AC-15 (Multi-repository orchestration) is a compound AC: it bundles participant-state resolution, XDEP resolution, Interface Freeze respect, per-repo PLAN/TEST-PLAN readiness, cross-repo validation, and release-eligibility derivation into a single criterion. Recommend splitting into independently verifiable sub-ACs so each can be fixture-tested and fail independently — the SPEC-0001 (external-intake) precedent found real bugs specifically because multi-repo invariants were tested at this level of granularity, not as one bundled scenario.

`[QE-3]` **non-blocking** — Missing boundary AC: no AC states the behavior when `/operate` is invoked against a `SPEC-NNNN`/`THREAD-NNNN` id that does not exist, or one that exists but is already terminal (`promoted`, `shipped`) with no further eligible action. This should resolve to a stated outcome (plausibly `AMBIGUOUS_STATE`, or a distinct not-found/already-complete outcome) rather than being unspecified.

`[QE-4]` **non-blocking** — No AC or scenario exercises the "security exception" / "policy exception" escalation categories named in §16, unlike test-classification (AC-12) and release-approval integrity (AC-14), which each get a worked AC and scenario. Recommend at least one concrete AC for a security/policy-exception stop, since §16 names these as hard human-decision boundaries but nothing in §35–§40 demonstrates one.

### Security Perspective

Security Review Trigger Assessment:

- **Authentication/authorization:** No — Operator introduces no new auth surface; it invokes existing commands under their existing auth model.
- **PII/sensitive data:** No new collection — Operator reads/writes the same artifacts existing commands already produce. See the third-party/untrusted-content item below for the one caveat.
- **Secrets/credential management:** No — no new credential handling; Operator introduces no new external connector.
- **Third-party integrations / untrusted or third-party text (`stack.llm_agent_untrusted_input`):** **Yes.** Operator is the first command surface in this repo whose explicit purpose is to chain many existing commands across an extended loop with materially reduced per-step human checkpointing (per `[LD-9]` above), including commands that read externally-sourced ticket content framed as "data, not instructions" under ARCH.md's standing security convention. The SPEC never restates that convention for Operator's own Human Decision Packages (§17), which may quote artifact content — including fetched or pasted external content — back to a human. That quoted content must remain reference material, never something Operator (or any future automated consumer of a decision package) treats as live instruction. This alone is sufficient to trigger security review, independent of the other criteria.
- **Data retention/compliance:** Conditional. Operator run records (`OPRUN-NNNN`, §14) will presumably be committed as repo artifacts like everything else in ForgeKit. If a decision package embeds content originating from a project whose `compliance.frameworks` is non-`none`, the same compliance-gate confirmation that already governs `SPEC-NNNN.source.md` capture (per `templates/reference/external-work-intake.md`) should extend to whatever Operator persists. Not addressed in the SPEC — flag for DESIGN; not treated as its own blocking SPEC finding since it's a reuse of an existing safeguard rather than genuinely new surface.
- **Regulatory/compliance framework scope:** n/a for ForgeKit's own `project.yml` (`compliance.frameworks: [none]`), but Operator is explicitly meant to be stack/project-agnostic (§3, §41) and will run against other projects whose compliance scope varies. Consistent with the SPEC's own framing elsewhere; no separate finding beyond the retention item above.

`security_review_required: yes` — driven by the third-party/untrusted-content trigger above, compounded by `[LD-9]`'s unresolved ambiguity about which existing human-confirmation points survive under autonomous policy.

### Readiness Verdict

**`needs_revision`.** Findings `[LD-1]` through `[LD-7]` require adding or restructuring required SPEC sections (Goals, User Stories, NFRs, Constraints & Assumptions, Security Considerations, Observability Considerations, Open Questions) — exactly the class of finding `/review-spec` treats as spec-level, not design-level. `[LD-8]` and `[LD-9]` are foundational ambiguities about the execution and autonomy model that constrain what those new sections must say, and should be resolved before DESIGN begins rather than discovered during it. `[QE-1]` (fail-closed policy default) is safety-adjacent and should be resolved at the SPEC/AC level alongside AC-13/AC-20. `[SC-1]` (scope split) is a non-blocking product decision for the TPO. The Acceptance Criteria (§35), Non-Goals (§5), and the overall architectural reasoning (§6–§33) are unusually strong and mostly need to be *organized under*, and lightly extended by, the missing sections — not rewritten.

## Analysis Round 2

### Resolution Check

Read SPEC-0001 v2's `## Revision Summary — Round 1`. Every Round 1 finding (`[SC-1]`, `[LD-1]`–`[LD-11]`, `[QE-1]`–`[QE-4]`) is listed as Addressed, none left as deferred-without-resolution. Checked each against the actual v2 text, not just the summary's claim:

- `[SC-1]`, `[LD-1]`–`[LD-7]`: confirmed — §45–§51 added, each in the required shape; Goal↔AC traceability (§45) checked directly against all 27 base ACs plus AC-15a–e and holds in both directions.
- `[LD-8]`, `[LD-9]`: confirmed resolved at the level §48 states them — see new finding `[LD-12]` below, which is a distinct, adjacent gap exposed *by* `[LD-9]`'s resolution, not a sign `[LD-9]` itself is unresolved.
- `[LD-10]`, `[LD-11]`: confirmed — §47 and §48/§42 respectively.
- `[QE-1]`: confirmed — AC-25, §49, and Safety Invariant #18 are mutually consistent.
- `[QE-2]`, `[QE-3]`, `[QE-4]`: confirmed — AC-15a–e, AC-26, AC-27 all present and independently stated.

### Lead Developer Perspective

`[LD-12]` **blocking** — §48's resolution of `[LD-9]` states that "routine draft-commit confirmations... may be satisfied autonomously by Operator... for actions an explicit Autonomy Policy `allow:` entry permits." This is the right call, but it has a mechanical precondition the SPEC never states: every existing ForgeKit command's Handoff is literally written as "ask the user to confirm, then wait" — there is currently no way for a command to know it is being driven by Operator under explicit, policy-permitted autonomy rather than by a human directly. Without *some* such recognition mechanism, `[LD-9]`'s resolution is not achievable against the commands as they exist today; Operator would have no way to legitimately be the one answering its own embedded prompt. This is the same "fix introduces an adjacent gap" pattern this project's own history has hit repeatedly — resolving *which* confirmations are skippable immediately exposes that *how a command would know to skip one* was never stated. §41's "Operator Readiness Prerequisites" list is presented as the complete control-surface prerequisite set and does not include this. Per the same pattern already used for every other mechanism in this SPEC (state the requirement now, defer the exact representation to DESIGN via §51), this needs: (a) one line added to §41's prerequisite list naming this capability, and (b) one entry added to §51 Open Questions deferring its exact mechanism (a flag, an environment marker, a convention — DESIGN's call) to DESIGN. The requirement itself — not its representation — is what's currently missing.

### Quality Engineer Perspective

`[QE-5]` **non-blocking** — §18's Operator Outcomes enumeration doesn't yet account for two cases introduced by this revision: AC-26's not-found/already-complete case, and reaching the new autonomy-containment cap (§47). Both are now real, named requirements elsewhere in the SPEC. Recommend one clarifying sentence in §18 noting its list is extended by these two cases (exact naming is already, consistently, deferred to DESIGN via §51) — cheap to fix, not a scope gap.

### Security Perspective

No change to the Round 1 trigger assessment. `security_review_required: yes` stands, for the same reason (untrusted-content handling under reduced checkpointing). The new §49 content resolves `[LD-5]` adequately; `[LD-12]` above is an orchestration-mechanism gap, not a new security trigger in its own right — noted for completeness, not re-scored here.

### Scope Coherence Re-check

`[SC-1]`'s resolution (keep as one SPEC; sequence prerequisites as earliest PLAN milestones) is sound and specific enough to act on at PLAN time. No further action needed.

### Readiness Verdict

**`needs_revision`** — narrowly. Only `[LD-12]` requires a SPEC change (two short additions: §41 and §51), consistent with how every other "state the requirement, defer the mechanism" item in this SPEC has already been handled. `[QE-5]` is non-blocking and can be folded in at the same time for convenience but would not by itself justify another round on its own. Everything else from Round 1 is confirmed resolved with no new issues.

## Analysis Round 3

### Resolution Check

Read SPEC-0001 v3's `## Revision Summary — Round 2` and verified directly against the text (not just the summary's claim):

- `[LD-12]`: confirmed — §41 now names the Operator-invocation-recognition capability as a required prerequisite; §51 defers its exact mechanism to DESIGN. Consistent with §48's formal-gates-vs-routine-confirmations distinction; no contradiction introduced.
- `[QE-5]`: confirmed — §18 now cross-references AC-26 and the §47 autonomy-containment cap as outcome cases its list is extended by, with naming deferred to DESIGN via §51 like the rest of the list.

### Lead Developer Perspective

`[LD-13]` **non-blocking** — Every other item in §41's prerequisite list maps to at least one existing AC that would exercise it (e.g., "Structured current review outcome" → AC-06; "`/status` resolver" → AC-03–05; "finding recurrence lineage" → AC-08; "implementation execution completeness" → AC-10; "canonical BUG lifecycle states" / "test-failure classification" → AC-12; "release approval binding" → AC-14; "release execution progress" → AC-21). The new ninth item (Operator-invocation recognition, added in v3) does not yet have a corresponding AC. Not blocking: the capability is already exercised implicitly by every AC that assumes an autonomous draft/revise action occurs at all (AC-01, AC-07), and this will necessarily be covered when TEST-PLAN is built regardless of whether a dedicated AC names it. Worth adding a short AC at the next convenient revision (would naturally map to Goal G6) purely for pattern-consistency with the rest of §41 — not required before DESIGN.

### Quality Engineer / Security Perspective

No new findings. No change to the `security_review_required: yes` trigger assessment — nothing in this round touched security-relevant surface.

### Scope Coherence Re-check

No change since Round 2; `[SC-1]`'s resolution stands.

### Readiness Verdict

**`ready`.** `[LD-13]` is the only new item this round, and it is explicitly non-blocking — it names a style/completeness improvement, not a gap that constrains DESIGN. Every finding from Rounds 1 and 2 is confirmed resolved with no new blocking issues. SPEC-0001 v3 is ready for `/approve-spec`.
