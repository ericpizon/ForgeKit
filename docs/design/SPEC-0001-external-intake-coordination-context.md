---
id: SPEC-0001
type: spec
schema_version: 1
title: "External Work Intake and Durable Coordination Context"
status: approved
created: 2026-10-05
updated: 2026-10-05
version: 3
owner: "Eric Pizon"
priority: TBD   # TPO call — not yet set; flagged in Open Questions
product_validated: true
validated_by: "Eric Pizon — Architect"
validated_date: 2026-10-05
changelog:
  - v3 — approved after 3 review rounds
approval:
  mode: repo_only
  approved_by: "Eric Pizon"
  approved_role: tpo
  approved_at: "2026-10-05"
  decision_summary: "SPEC-0001 approved for design phase — external work intake, durable coordination context, mid-flow promotion, environment governance, and Investigation workflow."
  external_event_id: null
  external_event_uri: null
policy_evaluation:
  evaluated: null
  policy_pack: null
  result: null
stakeholders:
  - Engineering
  - Architecture
links: []
generation:
  command: /create-spec
  command_version: "2"
  model: "claude-sonnet-5"
  generated_at: "2026-10-05"
  session_id: null
---

# SPEC-0001 — External Work Intake and Durable Coordination Context

## Problem Statement

Teams running more than one repo through an already-established external work-management process — a Product Manager drafts a spec in Confluence, refines it with engineering, the result becomes an ADO Epic broken into Features/Enablers/Investigations, work is assigned to developers, and completed changes move through a Dev (integration) → QA promotion flow before release — have no clean way to adopt ForgeKit's engineering-execution governance underneath that process.

Five concrete frictions prevent it today:

1. **No intake path.** ForgeKit assumes a SPEC originates inside ForgeKit itself. A team that has already captured a requirement in Confluence and tracked it as an ADO Feature/Enabler — whether fully refined there or only partially, with refinement expected to continue — would have to re-litigate it from scratch inside ForgeKit, or bypass ForgeKit's review rigor entirely to avoid doing so.
2. **No durable system-wide context.** A coordination repo today has no `ARCH.md` and no standing description of how a team's repos relate to each other or to shared external subsystems — only a scattered set of per-feature XDEPs, each scoped to one thread, with nothing rolling them up into a current view of the system.
3. **No mid-flow promotion path.** Whether a feature needs one repo or several is decided once, upfront, at `/forge-init`/`/create-spec` time. There is no path to discover mid-ANALYSIS or mid-DESIGN that a repo-local SPEC actually needs to become a cross-repo thread, without starting over.
4. **No proportional environment governance.** Release governance assumes a single, implicit production target. A routine promotion to a Dev or QA environment has no lightweight path — only the full RELEASE-PLAN ceremony built for production, or nothing at all.
5. **No home for spike/investigation work.** Work that may legitimately conclude without any implementation (an ADO Investigation) has nowhere to go in the pipeline — every existing entry point assumes a build is coming.

Each of these is a real gap in ForgeKit's own architecture, independent of any one team's specific tools — they would block adoption by any organization that already runs a mature, externally-governed, multi-repo delivery process before encountering ForgeKit.

## Background

This SPEC follows directly from the multi-repo coordination model already built and stress-tested (see [`docs/design/multi-repo-coordination.md`](./multi-repo-coordination.md)). It does not revise that model's core artifact chain (SPEC → ANALYSIS → DESIGN → ADR/OBS/SEC → PLAN → TEST → IMPLEMENT → RELEASE) — it extends the boundary around that chain: how work enters it, what durable context a coordination repo can hold, how a repo-local SPEC can become cross-repo after the fact, how non-production environments are governed, and how spike work that may never reach implementation is represented.

The scope was shaped through an explicit design review that deliberately rejected several larger alternatives — a new "Workspace" domain model, a parallel representation of an external tool's Epic/Feature/Enabler hierarchy, a migration to Claude Code's plugin distribution model — in favor of extending existing ForgeKit mechanisms (`repo_role: coordination`, XDEPs, `candidate_ref`, `/status`) wherever they already cover the need. Where a prior design iteration proposed a mechanism later found to duplicate something ForgeKit already had, the exclusion is recorded explicitly in Non-Goals rather than silently dropped.

`/review-spec`'s Scope Coherence Check (Round 1, `[SC-1]`) found that G1, {G2,G3}, G4, and G5 are four largely independent slices that could each ship separately, and flagged that G1 now carries a meaningfully different risk profile (`security_review_required: yes`) than the other three. The TPO decision is to **keep them together in this one SPEC**: all five goals were deliberately scoped as a single coherent milestone through the full design-review conversation that produced this SPEC, and G1's differing risk is already isolated via its own security self-assessment rather than diffused across the others — splitting now would add four separate SPEC→DESIGN→PLAN cycles without a corresponding benefit. This is a conscious decision, not an oversight; a future revision may still split this SPEC if the goals' delivery timelines diverge enough in practice to warrant it.

## Goals

- **G1 — External work intake with provenance.** Accept an external work item (e.g., an ADO Feature/Enabler) — at whatever refinement level it has actually reached, fully or partially — as a SPEC's origin, preserving source provenance. Both intake and the optional post-approval update check, at the point of use, whether the running environment already has working access to the external tool (e.g., a connected MCP server): fetch/write directly when it does, fall back to a copy-paste exchange with the human when it doesn't — without ForgeKit building or owning that access itself. ForgeKit's own SPEC review and validation (`/review-spec`, `/validate-spec`) still apply in full regardless of how refined the input was or how it was captured — intake supplies a starting point, not a shortcut past ForgeKit's existing quality bar. On approval, optionally close the loop back to the originating ticket, with the user's explicit confirmation gating any write either way.
- **G2 — Durable coordination-repo context.** Give a `repo_role: coordination` repo a standing, team-wide context — a roster of known participant repos and environments, and a system-wide architecture/dependency map — without introducing a new repo-role or a parallel "workspace" concept.
- **G3 — Mid-flow single-repo → cross-repo promotion.** Support discovering, during ANALYSIS or DESIGN, that a SPEC authored as repo-local actually needs to become a cross-repo thread, and promote it without losing identity, provenance, or already-completed review substance — while still re-examining anything the newly-discovered scope actually invalidates.
- **G4 — Proportional environment governance.** Generalize environment-based release governance beyond a single implicit production target, so a team's own environment names map to one of a small number of governance levels, each with ceremony proportional to its actual risk.
- **G5 — Investigation/decision workflow.** Support spike or research work that may conclude without any implementation, as a lightweight decision record distinct from SPEC/DESIGN, with an explicit, recommendation-only path back into intake (G1) when a build is actually warranted.

## Non-Goals

- ForgeKit will **not** build, maintain, or assume any ADO/Confluence API client, credential flow, or connector of its own in this SPEC. Where the running environment already provides working access to ADO/Confluence (e.g., an already-configured MCP server or CLI tool available to the session), both intake and the optional post-approval update (AC4) opportunistically use it. Where no such access exists, both gracefully fall back to a copy-paste flow with a human performing the actual read or write themselves. ForgeKit owns the capability-check-and-branch logic; it never owns the integration itself.
- ForgeKit will **not** automatically write back to ADO or Confluence — every write (AC4) requires the user's explicit, per-approval confirmation (AC4 fires on every approval of a given SPEC, not once per SPEC's lifetime — see AC4); nothing is updated silently or on a schedule. This includes when an Investigation's decision recommends building a Feature/Enabler: that recommendation is advisory text only regardless of access availability; a human creates the resulting ticket, which then re-enters ForgeKit via G1's intake path as ordinary new work.
- ForgeKit will **not** introduce `EPIC-NNNN`, `FEATURE-NNNN`, or `USER-STORY-NNNN` artifact types, or any other parallel representation of an external tool's work-item hierarchy. That external tool remains the sole source of truth for its own hierarchy; ForgeKit records only a pointer back to it.
- ForgeKit will **not** introduce a new "Workspace" domain model, repo-role, or `workspace.yml` schema. A team's existing shared/context repo, where one exists, is expected to be designated `repo_role: coordination` using the mechanism that already exists — not a second, parallel one.
- ForgeKit will **not** migrate command or skill distribution to Claude Code's plugin mechanism in this SPEC. The existing local-install-plus-`/upgrade-pipeline` model is unchanged. (Plugin-based distribution was evaluated and set aside specifically because plugin skills are namespaced — `/forgekit:create-spec`, not `/create-spec` — which is not a transparent drop-in for the current model; it may be revisited as a separate, later initiative, not here.)
- ForgeKit will **not** build a new test-execution framework for QA validation. A QA-level promotion record captures a result; it does not run or replace tests.
- ForgeKit will **not** introduce a `revision_set` field or concept. Multi-repo candidate tracking at any governance level reuses the per-participant `candidate_ref` shape already established in `approve-release-plan.md`'s coordination approval model.
- **"ForgeKit Operator"** — a separately planned future milestone — is explicitly out of scope and untouched by this SPEC.

## User Stories

- **US1 (Developer / intake).** As a developer picking up an assigned ADO Feature/Enabler, I want to start a ForgeKit SPEC directly from that ticket, so that ForgeKit's review rigor applies to real work without re-typing it — regardless of how much refinement has already happened upstream, and without that upstream refinement, however far it got, being thrown away.
- **US2 (Architect).** As an Architect working in a coordination repo, I want a durable, current view of which repos exist, what they're responsible for, and how they depend on each other and on shared external subsystems, so that I don't have to reconstruct the system's topology from a scattered history of past threads' dependency contracts every time I need it.
- **US3 (Lead Developer).** As a Lead Developer who discovers mid-DESIGN that a feature actually needs a second repo, I want to promote the existing SPEC into a cross-repo thread without restarting it, so that the team's existing review investment isn't thrown away — while anything the newly-discovered scope genuinely invalidates still gets re-examined, not silently carried forward.
- **US4 (SRE / Lead Developer).** As the person promoting a change to a QA environment, I want a lightweight, traceable promotion record scoped to my team's own environment names and risk levels, so that routine non-production promotion doesn't require full RELEASE-PLAN ceremony, while a genuine production release still does.
- **US5 (Developer / Investigation).** As a developer assigned an ADO Investigation, I want a place to record findings and a recommendation without being forced through a SPEC/DESIGN cycle that assumes a build is happening, so that spike work gets the same traceability as everything else, proportional to the fact that it may conclude with nothing to implement at all.

## Acceptance Criteria

**Intake (G1)**
- **AC1.** `/create-spec` accepts an optional external work reference (e.g., an ADO ticket ID). It first checks whether the running environment already has working read access to that system (e.g., a connected MCP server or CLI tool) — if so, it fetches the ticket directly; if not, or if the fetch attempt itself fails (invalid ticket ID, connector error), it prompts the user to paste the content and supply the ticket's identifying fields manually rather than blocking on the failure. Either way, the produced SPEC-NNNN's frontmatter records a `source_work: {system, id, type, parent_epic}` block.
- **AC2.** When content is fetched directly (access available), the SPEC's frontmatter records a `product_spec: {system, page_id, revision, fetched_at}` block; no companion file is required, since the connector can independently re-verify that revision later. When content is supplied by copy/paste (no access available), **and the initializing project's own `compliance.frameworks` is `[none]`**, the SPEC's directory additionally contains an immutable companion file (e.g. `SPEC-NNNN.source.md`), written once at intake and never edited afterward, holding exactly what was supplied; the frontmatter's `product_spec` block adds `captured_at` and `content_digest` fields referencing it, since there is no connector to re-verify against later. When copy/paste is the only path **and `compliance.frameworks` is non-`[none]`**, ForgeKit warns the user before writing the companion file and requires an explicit "this does not contain regulated data" confirmation before it is created — the companion file is never written automatically in that case, per Security Considerations.
- **AC3.** The `product_spec` block's core fields (`system`, `id`/`page_id`, `revision`) are identical regardless of access path — only the presence of `fetched_at` vs. `captured_at`/`content_digest`/a companion file differs, keyed on whether access existed at the moment of intake and, when it didn't, on the compliance confirmation outcome (AC2) — never hardcoded to "no connector exists yet," and never skipping the compliance check when one applies.
- **AC4.** When `/approve-spec` approves a SPEC that carries a `source_work` block (AC1), its Handoff step prompts the user: do they want the originating ADO ticket updated to reflect the approved SPEC? This prompt fires on every approval of that SPEC, including a later re-approval after a `/reopen-spec` → `/revise-spec` cycle — not only the first. If yes, ForgeKit checks for working write access the same way AC1 checks read access — if available, it updates the ticket directly (after showing the user what will be written, and only on their explicit confirmation); if not, it generates a plain-text summary for the user to paste in manually. If the write itself fails after confirmation (permission denied, network error, or the ticket having changed since it was last read), ForgeKit reports the failure clearly to the user and falls back to generating the copy-paste summary — it never fails silently, since a user who believes an update succeeded when it didn't would be relying on a false signal. If no `source_work` block is present, this prompt is skipped entirely.

**Coordination context (G2)**
- **AC5.** A `repo_role: coordination` repo's `project.yml` may declare `known_participants: [{repo, purpose}]` and `environments: [{name, purpose, governance}]`. Both fields are optional and additive — a coordination repo lacking them is unaffected.
- **AC6.** A `repo_role: coordination` repo may hold a durable system-architecture artifact (name TBD — see Open Questions) containing: (a) external subsystems the whole system depends on that no single participant's own `ARCH.md` owns, and (b) a Dependency Map section whose content is derived from the current set of `active`-status XDEPs across all threads — never a hand-maintained, independently-drifting duplicate of XDEP content.
- **AC7.** `/forge-init`, when initializing a `repo_role: code` project, asks whether the project participates (or is expected to participate) in cross-repo coordination with other repos on the team and, if so, whether to point at an existing coordination repo or defer the decision. The question's text explains what a coordination repo is for (per the description already established in `forge-init.md`'s own coordination-repo path, extended to cover the new durable-context capability from AC5/AC6) and explicitly prompts the user to consider whether an existing shared/context repo on their team already plays this role. An answered value is recorded as `coordination_repo` in the initializing repo's own `project.yml`; "not yet" leaves it unset.
- **AC8.** `/promote-to-thread` (AC9) reads `coordination_repo` from the promoting repo's `project.yml` first. If already set, it is reused without re-asking. If unset, the same question from AC7 (full description included) is asked at that point instead, and the answer is persisted back into `coordination_repo` in the promoting repo's own `project.yml` — identically to how AC7's proactive answer is persisted. This persistence is what makes "asked exactly once" hold across the repo's lifetime, not just within a single promotion: without it, a second, later promotion of a different SPEC in the same repo would find `coordination_repo` still unset and ask again.

**Promotion (G3)**
- **AC9.** A new command, `/promote-to-thread SPEC-NNNN`, converts a repo-local SPEC into a THREAD-NNNN in the designated coordination repo (per AC7/AC8): it mints a collision-checked `THREAD-NNNN` (same uniqueness discipline as existing THREAD/XDEP IDs), writes `thread.yml` with the discovered participants, and re-runs the coordination Pre-flight checks `/create-spec`'s own coordination path would have run upfront — including reading every newly-discovered participant's `ARCH.md`, not just the originating repo's. `/promote-to-thread` is valid against a SPEC in either `status: draft` (promotion discovered during ANALYSIS, before the SPEC is ever approved) or `status: approved` (promotion discovered during DESIGN, per G3's own "during ANALYSIS or DESIGN" scope) — both are real, reachable trigger points, not just one. Invoking it against a SPEC already `status: promoted` is a no-op that reports the existing `THREAD-NNNN` rather than minting a second one or erroring ambiguously.
- **AC10.** Promotion re-examines, on substance, any review round (`ANALYSIS`, `DESIGN` review) that already completed under the single-repo assumption — it is neither a blanket restart nor a silent carry-forward. The disposition of each prior finding (still valid, invalidated by the newly-discovered scope, needs re-review) is recorded explicitly.
- **AC11.** After promotion, the original repo-local SPEC-NNNN is set to `status: promoted`, `superseded_by: THREAD-NNNN` — never deleted, and never silently left in `draft`/`approved` status as if nothing happened.
- **AC12.** `/status SPEC-NNNN`, run against a promoted SPEC, reports the promotion explicitly and names the authoritative `THREAD-NNNN` — it does not show stale single-repo pipeline status as if the SPEC were still the authoritative artifact.

**Environment governance (G4)**
- **AC13.** Each entry in a coordination repo's `environments` list (AC5) declares a `governance` value of `ci`, `promotion`, or `release`. Every command that reasons about environment progression reasons about the `governance` value, never about a hardcoded environment name (`dev`, `qa`, `production`, or any other).
- **AC14.** A `governance: ci` environment transition requires no new ForgeKit artifact — it remains outside ForgeKit's governance scope, unchanged from today.
- **AC15.** A `governance: promotion` environment transition produces a lightweight record containing: a per-participant `candidate_ref` map (the exact shape already used in `approve-release-plan.md`'s coordination approval model — not a newly-named field), who promoted it, when, and a smoke-test result. It explicitly does **not** include a Gating-criteria table, Rollback plan, or Validation Plan section — those remain exclusive to `governance: release`, where today's existing RELEASE-PLAN ceremony is unchanged.

**Investigation / decision workflow (G5)**
- **AC16.** A command authors a standalone artifact type (not nested inside SPEC or DESIGN — see Open Questions for its ID sequence) that records an Investigation's question, approach, findings, and exactly one of four decision outcomes: no action, a standalone ADR, "recommend implementation" (advisory text only, per Non-Goals), or "investigate further." Whether this command is wholly new or an extension of an existing one, and its exact name, are deferred to DESIGN (see Open Questions) — but that a command exists for this is not deferred.
- **AC17.** A standalone ADR-authoring path exists that does not require a parent `DESIGN-NNNN` — for the case where an Investigation's own decision is itself the cross-cutting architectural decision worth preserving, independent of whether any implementation follows.

## Non-Functional Requirements

- **Performance:** No new latency-sensitive runtime path is introduced — every capability in this SPEC is an authoring-time or governance-time command, not production code. No threshold needed.
- **Reliability:** `/promote-to-thread` (AC9) must be safely resumable if interrupted after minting a `THREAD-NNNN` but before all steps complete. A partial promotion must never leave the original SPEC ambiguous between "promoted" and "not promoted." Exact mechanism (e.g., idempotency keyed on whether `thread.yml` already exists for the target `THREAD-NNNN`) is deferred to DESIGN — see Open Questions.
- **Security:** External content, fetched or pasted, is reviewed engineering material, never instruction authority — same framing `cross-repo-resolution.md` already applies to other cross-repo inputs. The write-back path (AC4) must never execute without the user's explicit, per-SPEC confirmation of the exact content being written. See Security Considerations below for the full reasoning, including why this SPEC now self-assesses `security_review_required: yes`.
- **Forward compatibility:** The `product_spec`/`source_work` provenance schema (AC1–AC3) must not assume the permanent absence of a connector. Field names and shapes must be identical whether content is captured by copy-paste today or fetched automatically later — only the presence of a `.source.md` companion file should differ.
- **Operability:** No new infrastructure dependency is introduced. The mechanism that derives a coordination repo's Dependency Map (AC5) from active XDEPs is deferred to DESIGN — it must not require a service, database, or process beyond what ForgeKit's command-driven model already uses.

## Constraints & Assumptions

- Builds directly on, and does not revise, the existing multi-repo coordination model's core artifact chain (SPEC → ANALYSIS → DESIGN → ADR/OBS/SEC → PLAN → TEST → IMPLEMENT → RELEASE).
- Assumes a team's pre-existing shared/context repo, where one exists, is designated `repo_role: coordination` rather than requiring a second, parallel concept. This SPEC does not require that pairing — a team without such a repo simply designates a plain coordination repo as usual — but is written assuming the paired case is common.
- Does not assume live ADO or Confluence API access exists, and does not assume it doesn't — both intake and the optional post-approval update check for working access in the current environment at the point of use and branch accordingly (direct fetch/write when available, copy-paste when not). ForgeKit itself never ships or manages the credentials, client, or connector that access depends on.
- Assumes XDEPs remain the authoritative record for any specific cross-repo dependency's details (verification layers, rollout ordering). The system-architecture artifact in AC5 is a derived rollup view, never an independent second source of truth for a dependency's specifics.

## Security Considerations

- External content — whether fetched directly or pasted — must never be treated as instruction authority by any role adopting a command that reads it, consistent with the existing framing in `cross-repo-resolution.md` for other cross-repo inputs. This applies identically regardless of access path; a fetched ticket is not more trusted than a pasted one just because it arrived automatically.
- `content_digest` values are non-sensitive (a hash, not the content itself) and may be freely committed. The `.source.md` companion file's sensitivity matches whatever the underlying PRD actually contains — if a Confluence PRD could ever contain customer PII or credentials, committing the companion file inherits that risk. **This risk is categorically worse than ordinary data exposure, because it is effectively irreversible**: once a `.source.md` file is committed, it cannot be reliably un-committed in practice — rewriting git history after the fact doesn't retract what's already been cloned, mirrored, or cached elsewhere. This directly conflicts with any "right to erasure" a compliance framework (e.g. GDPR) might require if regulated data ends up captured this way. Mitigation direction for DESIGN: when the adopting project's own `compliance.frameworks` is non-`[none]`, warn the user before writing a `.source.md` file and require an explicit "this does not contain regulated data" confirmation before capture — never capture automatically in that case.
- **Write access is a materially different risk than read access.** AC4's direct-update path, when available, writes to a system of record other people (PMs, stakeholders) rely on without reviewing ForgeKit's own process. The explicit-confirmation-before-write requirement in AC4 (show exactly what will be written, require per-SPEC confirmation, never write on a schedule or silently) is the control this SPEC relies on — DESIGN must not weaken it.
- ForgeKit does not manage, store, or provision the credentials any live access depends on — it only checks whether access already works in the current environment. Whatever scopes that access has (read-only vs. write, which tickets/projects it can reach) are entirely the adopting team's own configuration, outside this SPEC's control. That boundary itself should be stated plainly to any team adopting this, not left implicit.
- **`security_review_required`: yes.** The write-back path in AC4, even confirmation-gated, is a third-party integration touching an external system of record — `/review-spec`'s own Security Perspective checklist treats "third-party integrations" as sufficient on its own to trigger a review, independent of whether any other criterion also fires. (This differs from an earlier draft of this SPEC, written before AC4's direct-write path existed, which self-assessed "no" — that assessment no longer holds once a real write path is in scope, even one ForgeKit doesn't itself implement the transport for.)

## Observability Considerations

- **Metrics:** None required. Every capability in this SPEC is an authoring-time or governance-time pipeline command, not a running service.
- **Traces:** None required, for the same reason.
- **Logs / audit trail:** No separate logging mechanism is needed. Every new artifact this SPEC introduces (a promotion record, an Investigation decision record, a `.source.md` capture) is itself the audit record — consistent with ForgeKit's existing "the git log is the audit trail" principle. What must never be logged or committed in plain form: raw credential content, if it were ever present in captured external content (see Security Considerations) — the audit trail is the artifact's own git history, not a separate log stream that could diverge from it.

## Open Questions

- **OQ1.** What is the exact mechanism that derives the coordination repo's Dependency Map (AC5) from the current set of active XDEPs — a manual sync step run by a command, an automatic aggregation on read, or something else? Deferred to DESIGN.
- **OQ2.** What is the exact idempotency/resumability mechanism for `/promote-to-thread` if interrupted mid-flight (see Reliability NFR)? Deferred to DESIGN.
- **OQ3.** What should the new system-wide architecture artifact actually be named? `SYSTEM-ARCH.md` was used as a working placeholder throughout this SPEC's drafting; not finalized. Deferred to DESIGN, with the TPO/Architect's input.
- **OQ4.** Two related questions about the Investigation/decision artifact (AC16), both deferred to DESIGN: (a) does it need its own ID sequence (e.g., `INV-NNNN`), or should it reuse/extend an existing one; (b) is the command that authors it wholly new, or an extension of an existing command, and what is it called? AC16 already settles that *a* command exists — only its identity and the artifact's ID sequence remain open here.
- **OQ5.** What is this SPEC's priority relative to other ForgeKit work currently planned, including the separately-scoped "ForgeKit Operator" milestone this SPEC explicitly excludes? Frontmatter `priority` is currently a placeholder (`TBD`) pending the TPO's answer.
- **OQ6.** What exactly does "checking whether the environment already has working access" (AC1, AC4) mean mechanically — probing for a specific class of MCP server, attempting a CLI tool invocation and checking its exit code, a user-declared config flag, or some combination? Deferred to DESIGN; AC1/AC3/AC4 are intentionally written at the level of "access exists or it doesn't," not tied to one specific detection mechanism, so DESIGN has room to choose without a SPEC revision.
- **OQ7.** What happens to a promoted SPEC's own git branch (`[LD-2]`, Round 1) — merged, abandoned, or left alone? Deferred to DESIGN.
- **OQ8.** Is a `/forge-init`-time "not yet" answer to the coordination-repo question (AC7) permanently sticky, or re-askable on a later full redo (`[LD-4]`, Round 1)? Deferred to DESIGN.
- **OQ9.** AC2's compliance-aware confirmation gate (added Round 2, per `[SEC-1]`) has no stated outcome if the user declines the "this does not contain regulated data" confirmation — with no connector and no companion file, there is nothing left to capture the content with. Does `/create-spec` proceed with `source_work` recorded but no `product_spec` content at all, prompt for a different capture method, or block entirely? Deferred to DESIGN.

## Revision Summary — Round 1

**Date:** 2026-10-05

### Addressed

- **[SC-1]** — Scope coherence (G1 independently deliverable from {G2,G3}, G4, G5; differing risk profile) — TPO decision recorded explicitly in Background: keep together as one deliberately-scoped milestone, rationale stated, not silently absorbed.
- **[LD-1]** — AC8 didn't persist a reactively-obtained `coordination_repo` answer, undermining its own "asked exactly once" claim — AC8 revised to require persistence identical to AC7's proactive path.
- **[LD-3]** — AC4 didn't state whether the ADO-update prompt re-fires on re-approval after `/reopen-spec` — AC4 revised to state it fires on every approval of the SPEC, not just the first.
- **[QE-1]** — AC16 named no command, making it untestable as written — AC16 revised to require a command exists (new or extended; exact name deferred to DESIGN per OQ4, which remains open for that narrower question only).
- **[QE-2]** — AC4 had no write-failure path — AC4 revised to require clear failure reporting and fallback to the copy-paste summary, never a silent failure.
- **[QE-3]** — AC1 had no intake-failure fallback — AC1 revised to state a failed fetch attempt falls back to the paste prompt rather than blocking.
- **[QE-4]** — Status × operation gap: `/promote-to-thread`'s validity against `draft` vs. `approved` SPECs was unaddressed — AC9 revised to state explicitly that both are valid trigger points, matching G3's own "during ANALYSIS or DESIGN" scope.
- **[QE-5]** — Double-promotion behavior was undefined — resolved in the same AC9 revision: a no-op reporting the existing `THREAD-NNNN`, not a second mint or an ambiguous error.
- **[SEC-1]** — Security Considerations named the PII-inheritance risk but not its irreversibility, and proposed no mitigation — revised to name the git-immutability/right-to-erasure conflict explicitly and require a compliance-aware confirmation gate before any `.source.md` capture, as a concrete DESIGN-time requirement rather than a vague caveat.

### Not Addressed

- **[LD-2]** — **deferred to design** — What happens to a promoted SPEC's own git branch is an implementation detail with no scope impact either way; tracked as new Open Question OQ7.
- **[LD-4]** — **deferred to design** — Whether a `/forge-init`-time "not yet" answer is permanently sticky or re-askable on a later full redo doesn't change this SPEC's Goals or ACs; tracked as new Open Question OQ8.

## Revision Summary — Round 2

**Date:** 2026-10-05

### Addressed

- **[LD-1]** — AC16 opened with "A new command authors..." while the next sentence deferred to DESIGN whether it's new or an extension — self-contradictory. Reworded to "A command authors..." so the opening no longer presupposes an answer the rest of the AC says is still open.
- **[SEC-1]** — Security Considerations' Round 1 mitigation (a compliance-aware confirmation gate before `.source.md` capture) was stated only in prose, with no corresponding AC — unenforceable as written. AC2 revised to incorporate the gate directly: the companion file is written automatically only when `compliance.frameworks` is `[none]`; otherwise, an explicit confirmation is required first. New Open Question OQ9 added for the previously-unstated case where the user declines that confirmation.
- **[LD-2]** — Non-Goals' "per-SPEC confirmation" phrasing could be misread as "once ever," given AC4 explicitly fires on every re-approval. Reworded to "per-approval confirmation" with an inline pointer to AC4.

### Not Addressed

*(None — all three Round 2 findings were addressed; no new deferrals.)*

### Self-caught during this revision (not a Round 2 finding)

- AC3's claim that the capture-method differs "keyed strictly on whether access existed" was no longer accurate once AC2 gained a second keying factor (compliance confirmation outcome) during this same revision. Fixed inline — AC3 now names both factors.

## Product Validation

**Validated by:** Eric Pizon — Architect
**Date:** 2026-10-05
**Model:** claude-sonnet-5
**Status:** validated

### Acceptance Criteria — Plain Language Review

**AC1 (original):** `/create-spec` accepts an optional external work reference (e.g., an ADO ticket ID). It first checks whether the running environment already has working read access to that system (e.g., a connected MCP server or CLI tool) — if so, it fetches the ticket directly; if not, or if the fetch attempt itself fails (invalid ticket ID, connector error), it prompts the user to paste the content and supply the ticket's identifying fields manually rather than blocking on the failure. Either way, the produced SPEC-NNNN's frontmatter records a `source_work: {system, id, type, parent_epic}` block.
**In plain terms:** When a developer starts a new SPEC from an existing ADO ticket, ForgeKit either pulls the ticket's content in automatically (if it already has access) or asks the developer to paste it in — either way, the new SPEC remembers which ticket it came from.

**AC2 (original):** When content is fetched directly (access available), the SPEC's frontmatter records a `product_spec: {system, page_id, revision, fetched_at}` block; no companion file is required. When content is supplied by copy/paste (no access available), and the initializing project's own `compliance.frameworks` is `[none]`, the SPEC's directory additionally contains an immutable companion file holding exactly what was supplied. When copy/paste is the only path and `compliance.frameworks` is non-`[none]`, ForgeKit warns the user and requires an explicit "this does not contain regulated data" confirmation before the companion file is created.
**In plain terms:** When ForgeKit pulls a ticket in automatically, it just notes which version it read. When a developer pastes content in instead, ForgeKit keeps a permanent copy of exactly what was pasted — unless the project has compliance rules in place, in which case it asks the developer to confirm the content is safe to keep before saving that copy.

**AC3 (original):** The `product_spec` block's core fields are identical regardless of access path — only the presence of `fetched_at` vs. `captured_at`/`content_digest`/a companion file differs, keyed on whether access existed and, when it didn't, on the compliance confirmation outcome.
**In plain terms:** No matter how the ticket's info got into the SPEC, the SPEC always remembers the same basic facts about where it came from — only whether there's also a saved copy of the raw text changes, based on how it was captured.

**AC4 (original):** When `/approve-spec` approves a SPEC that carries a `source_work` block, its Handoff step prompts the user to update the originating ADO ticket, on every approval including after a reopen/revise cycle. It checks for write access, updates directly on confirmation after showing what will be written, or falls back to a copy-paste summary — and reports clearly rather than failing silently if the write itself fails.
**In plain terms:** When a SPEC that came from an ADO ticket gets approved, ForgeKit asks the developer if they'd like the original ticket updated to match. If ForgeKit can update it directly, it shows exactly what it's about to write and only does so once the developer says yes. If that update fails, or if ForgeKit can't update tickets directly, it gives the developer the text to paste in themselves instead — it never pretends an update happened when it didn't. This happens every time the SPEC gets approved, not just the first time.

**AC5 (original):** A `repo_role: coordination` repo's `project.yml` may declare `known_participants` and `environments`. Both fields are optional and additive.
**In plain terms:** A team's shared coordination repo can keep a running list of which repos belong to the team and what environments exist — this is optional, and nothing breaks if a team doesn't set it up.

**AC6 (original):** A `repo_role: coordination` repo may hold a durable system-architecture artifact containing external subsystems and a Dependency Map derived from active XDEPs.
**In plain terms:** A team's coordination repo can also keep an up-to-date picture of the whole system — what outside services everything depends on, and how the team's repos actually connect to each other — built automatically from existing records rather than typed in by hand and left to go stale.

**AC7 (original):** `/forge-init`, when initializing a `repo_role: code` project, asks whether the project participates in cross-repo coordination and, if so, whether to point at an existing coordination repo, explaining what one is for.
**In plain terms:** When setting up a new repo, ForgeKit asks whether it's part of a team that coordinates work across multiple repos, and if so, whether there's already a shared repo for that coordination — explaining what that shared repo is for so the developer can recognize if one already exists on their team.

**AC8 (original):** `/promote-to-thread` reads `coordination_repo` first; if unset, asks and persists the answer, identically to AC7's proactive path.
**In plain terms:** If a repo already knows which shared coordination repo it uses, ForgeKit won't ask again later. If it doesn't know yet, ForgeKit asks at the moment it's actually needed — and remembers the answer afterward, so it's never asked twice.

**AC9 (original):** `/promote-to-thread SPEC-NNNN` converts a repo-local SPEC into a THREAD-NNNN, valid from either `draft` or `approved` status; a no-op if already promoted.
**In plain terms:** If a developer realizes partway through — while still reviewing or while designing — that a SPEC they started for one repo actually needs more than one repo, there's a command that upgrades it into a proper multi-repo effort, no matter which of those two points they're at. Running it again on something already upgraded just tells them it's already done, instead of causing a mess.

**AC10 (original):** Promotion re-examines, on substance, any review round that already completed under the single-repo assumption.
**In plain terms:** When a SPEC gets upgraded to a multi-repo effort, any review work already done on it doesn't get thrown away or blindly kept — each earlier finding gets a fresh look to see if the new, bigger scope actually changes anything about it.

**AC11 (original):** After promotion, the original SPEC is set to `status: promoted`, `superseded_by: THREAD-NNNN` — never deleted.
**In plain terms:** The original single-repo version of the SPEC sticks around after being upgraded — it's clearly marked as superseded and points to the new multi-repo version, rather than disappearing or looking like nothing happened.

**AC12 (original):** `/status SPEC-NNNN`, run against a promoted SPEC, reports the promotion and names the authoritative THREAD-NNNN.
**In plain terms:** If someone checks the status of that original, now-superseded SPEC, ForgeKit tells them it's been upgraded and points them straight to the real, current version — instead of showing old information as if it still mattered.

**AC13 (original):** Each environment declares a `governance` value of `ci`, `promotion`, or `release`; commands reason about governance, never hardcoded environment names.
**In plain terms:** A team can name their environments whatever they want — the important thing ForgeKit cares about is how much oversight each one needs, not what it's literally called.

**AC14 (original):** A `governance: ci` transition requires no new ForgeKit artifact.
**In plain terms:** Deploying to a plain CI/build environment doesn't trigger any extra ForgeKit paperwork at all — that stays exactly as lightweight as it is today.

**AC15 (original):** A `governance: promotion` transition produces a lightweight record (candidate map, who, when, smoke-test result) — explicitly excluding Gating-criteria, Rollback plan, or Validation Plan sections.
**In plain terms:** Promoting a change to an environment like QA creates a small record of who promoted what, when, and whether a quick smoke test passed — but it skips all the heavier paperwork that a real production release still requires.

**AC16 (original):** A command authors a standalone artifact recording an Investigation's question, approach, findings, and one of four decision outcomes.
**In plain terms:** There's a dedicated place to record a spike or research task's question, what was tried, what was found, and what should happen next — even when the answer is "we decided not to build anything."

**AC17 (original):** A standalone ADR-authoring path exists that does not require a parent `DESIGN-NNNN`.
**In plain terms:** A team can write down and preserve an important architectural decision on its own, even when it didn't come out of a full design process for a specific feature.

### Structural Questions

1. **Problem statement describes pain, not solution?**
   Confirmed by TPO/Architect sign-off — no correction requested.

2. **Non-Goals specific enough to use as decision rules?**
   Confirmed by TPO/Architect sign-off — no correction requested.

3. **Uncovered user scenario?**
   None identified by TPO/Architect sign-off.

### AC-Specific Questions

For AC1 through AC17, the TPO/Architect's recorded answer to "Is the plain-language translation the complete definition of done, or is there an implied condition not stated?" was a single blanket response covering all seventeen: **"No gaps. Proceed."** No implied condition was called out for any individual AC.

### Gaps Found

None.

### Sign-off

> Eric Pizon — Architect confirms: the plain-language translations are accurate and the questions above are answered. This SPEC is ready for design.
