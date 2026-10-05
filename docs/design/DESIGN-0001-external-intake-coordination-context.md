---
id: DESIGN-0001
type: design
schema_version: 1
status: approved
version: 4
updated: 2026-10-05
current_review_round: 2
target_spec_version: 3
changelog:
  - v4 — amended post-approval: D11's sanitization scope extended to /promote-to-thread's own coordination-repo errors (Round 2's accepted-as-note [SEC-1], picked up as a blocking finding during /create-plan's own review when no task implemented it); the Round 2 [LD-1] "touching"/"writing to" wording fix and [LD-2] re-mint-on-collision behavior, both also previously only accepted as developer notes, applied directly to the /promote-to-thread description. No scope change, no new ACs — a textual completion of already-approved decisions, not a design reopening.
  - v3 — approved after 2 review rounds
approval:
  mode: repo_only
  approved_by: "Eric Pizon"
  approved_role: architect
  approved_at: "2026-10-05"
  decision_summary: "DESIGN-0001 approved for Stage 4 — Option B (idempotent state-inspection) confirmed for /promote-to-thread; 4 ADRs accompany it."
  external_event_id: null
  external_event_uri: null
policy_evaluation:
  evaluated: null
  policy_pack: null
  result: null
generation:
  command: /create-design
  command_version: "3"
  model: "claude-sonnet-5"
  generated_at: "2026-10-05"
  session_id: null
---

# DESIGN-0001 — External Work Intake and Durable Coordination Context

## Design Options

**Option A: Checkpoint-file-based staged promotion**
- Approach: `/promote-to-thread` writes a local checkpoint file tracking which internal steps completed (mint THREAD-NNNN → write thread.yml → persist coordination_repo → copy SPEC content → re-run pre-flight → set superseded_by). On resume, reads the checkpoint and continues from the first incomplete step.
- Existing code integration: wholly new mechanism — ForgeKit has no other multi-step resumable command; everything else today commits atomically or asks the user to continue manually.
- Pros: precise resumability; never redoes expensive completed steps (e.g. re-reading every participant's ARCH.md); the checkpoint is inspectable for debugging.
- Cons: introduces a new state-tracking mechanism and file type ForgeKit has never needed; the checkpoint itself can go stale or desync from reality (concurrent runs, manual interference).
- Risk: Medium.

**Option B: Idempotent-step design — check real state before acting, no side file**
- Approach: every step checks its own completion condition directly against actual committed state before acting ("does THREAD-NNNN already exist? reuse it. Is thread.yml already updated? skip. Is the SPEC already `status: promoted`? the whole command no-ops," per AC9). Committed git state is the only source of truth.
- Existing code integration: extends a pattern ForgeKit already trusts — THREAD/XDEP ID minting already checks "does this already exist at HEAD" before allocating, and the self-referential-ref pattern already treats committed state as truth over any side mechanism.
- Pros: no new infrastructure; nothing that can desync from reality independent of the real state; naturally handles concurrent or duplicate invocation.
- Cons: slightly more per-step code; less precise "exactly which step failed" diagnostics than an explicit log.
- Risk: Low.

**Option C: Single-transaction, no partial-progress recovery**
- Approach: if interrupted, any partial THREAD-NNNN is simply abandoned; a retry starts over from scratch with a fresh THREAD-NNNN.
- Existing code integration: simplest — no resumability logic at all.
- Pros: minimal code.
- Cons: directly violates SPEC-0001's own Reliability NFR; leaves orphaned THREAD-NNNN allocations behind over time.
- Risk: High.

## Recommended Option

**Option B.** Option C is disqualified outright — it contradicts SPEC-0001's Reliability NFR, not a style tradeoff. Between A and B, B is lower-risk because it extends a pattern ForgeKit already trusts (committed state as the one source of truth) rather than inventing new infrastructure a single command would be the only user of. Trade-off accepted: losing Option A's precise step-by-step failure diagnostics — acceptable for a low-frequency, human-invoked command.

User confirmed Option B.

## Architecture Overview

Of SPEC-0001's five goals, only G3's promotion mechanism (above) had genuinely competing architectures. The rest are direct specifications of what SPEC-0001 already locked in across its own three review rounds — covered below as Key Decisions, each resolving one of SPEC-0001's deferred Open Questions.

**New commands:** `/promote-to-thread` (G3), `/create-investigation` and `/create-adr` (G5), `/promote-to-environment` (G4's `governance: promotion` transitions).

**Modified commands:** `/create-spec` (G1 intake), `/approve-spec` (G1 write-back prompt), `/forge-init` (G2/G3 coordination-repo question), `/status` (G3 promoted-SPEC redirect), `/approve-design`'s Interface Freeze step (G2's SYSTEM-ARCH.md sync).

**New artifact types:** `SYSTEM-ARCH.md` (G2, one per coordination repo), `INV-NNNN` (G5, Investigation records), standalone `ADR-NNNN` (G5, no parent DESIGN required), per-environment promotion logs (G4).

No running service, no database, no new infrastructure dependency anywhere in this DESIGN — every capability is new or modified command text plus new frontmatter fields and markdown artifacts, consistent with SPEC-0001's own Operability NFR.

## Component / Data Model Changes

**SPEC-NNNN frontmatter gains** (G1): `source_work: {system, id, type, parent_epic}`; `product_spec: {system, page_id, revision, fetched_at}` (fetched path) or `{system, page_id, revision, captured_at, content_digest, capture_status}` (paste path — `capture_status: captured | declined`, resolving OQ9: see Key Decision D6); during promotion (G3), `pending_promotion: THREAD-NNNN` — written as `/promote-to-thread`'s *first* step, before the THREAD-NNNN is even minted in the coordination repo, so every later step (including a resumed one) has a concrete, git-committed anchor to check "does a THREAD-NNNN already exist for this SPEC" against; once promotion fully completes, `pending_promotion` is replaced by `status: promoted`, `superseded_by: THREAD-NNNN`.

**`project.yml` (repo_role: coordination) gains** (G2): `known_participants: [{repo, purpose}]`, `environments: [{name, purpose, governance: ci|promotion|release}]` — both optional, additive.

**`project.yml` (repo_role: code) gains** (G2/G3): `coordination_repo: org/repo | null` — written by `/forge-init` (AC7) or `/promote-to-thread` reactively (AC8).

**New artifact — `sdlc/architecture/SYSTEM-ARCH.md`** (G2, coordination repo only): frontmatter `type: system-architecture`, `schema_version: 1`, `dependency_map_generated_at`, `changelog: []`. Body: `## External Subsystems` (hand-maintained — what no single participant's ARCH.md owns), `## Dependency Map` (derived, regenerated wholesale — see ADR-0002, Key Decision D2).

**New artifact type — `sdlc/investigations/INV-NNNN-slug.md`** (G5): frontmatter `id`, `type: investigation`, `schema_version: 1`, `status: open|concluded`, `decision: no_action|adr|recommend_implementation|investigate_further|null`, `linked_adr: ADR-NNNN|null`, `source_work` (reuses G1's exact shape when the Investigation itself originated from an ADO ticket), `created`, `concluded_at`. Body: Question, Approach, Findings, Decision, Rationale.

**New artifact pattern — `sdlc/promotions/{environment}.md`** (G4): one file per environment, append-only table — `| Candidate ref(s) | Promoted by | Promoted at | Smoke test |` — a row per `governance: promotion` transition. Not a numbered per-event artifact (see ADR-0004).

**Standalone ADR path** (G5): `sdlc/adr/ADR-NNNN-slug.md`, authored by the new `/create-adr` command — same file shape and required sections (Context, Decision, Alternatives Considered, Consequences, Status Rationale, Revisit Triggers) as a DESIGN-nested ADR, just without a `target_design` pointer in frontmatter.

## Interface / API Changes

- **`/create-spec [--from <system>:<id>]`** (G1) — new optional argument. When supplied: attempts the real fetch directly (no separate "probe" step — see Key Decision D5); on any error indicating no working connector (not a transient failure), falls back to prompting for paste, surfacing only a sanitized failure category per D11, never raw connector output. Records `source_work` either way. Fetched or pasted content is handed to the TPO role as quoted reference material per D12 — never as instructions.
- **`/approve-spec`** (G1) — Handoff gains a conditional step: if `source_work` is present, prompt to update the originating ticket; attempt the real write directly; on failure (any non-confirmed-success, including partial), report a sanitized failure category per D11 and fall back to a copy-paste summary — never silent, never raw connector output, never treats partial success as success.
- **`/forge-init`** (G2/G3) — `repo_role: code` path gains the coordination-participation question (AC7), writing `coordination_repo`. Answering "not yet" is not sticky (see Key Decision D7) — it is simply "unset," re-askable only through `/forge-init`'s existing Option C (project identity update) or a full redo, never proactively re-prompted outside those.
- **`/promote-to-thread SPEC-NNNN`** (G3, new) — Option B's idempotent design throughout, in this exact order: (1) if the SPEC has no `pending_promotion` field yet, mint the candidate `THREAD-NNNN` number (reading the coordination repo to determine it is fine; *writing* to the coordination repo is not) and write `pending_promotion: THREAD-NNNN` to the SPEC's own frontmatter *before any write to the coordination repo* — this is the anchor every later step, including a resumed one, checks against; (2) reads `coordination_repo` (reusing AC8's persist-on-first-ask logic); (3) mints the `THREAD-NNNN` directory in the coordination repo only if it doesn't already exist (checked against `pending_promotion`'s value, not re-derived) — if the anchored number is found already claimed by a different SPEC's concurrent promotion, re-mint a new number and update `pending_promotion` accordingly rather than erroring opaquely; (4) writes `thread.yml` only if this repo isn't already listed; (5) re-runs the coordination pre-flight only for participants not already recorded; (6) sets the original SPEC's `status: promoted`/`superseded_by` last, after every other step is confirmed complete, replacing `pending_promotion`. An interruption at any point before step 6 leaves `pending_promotion` as the one durable record of exactly which `THREAD-NNNN` this SPEC is committed to — a resumed run reads it first and never mints a second one. Any error surfaced during this sequence — including the coordination repo being unreachable — is sanitized per D11's now-uniform scope, never shown or committed raw. Leaves the original SPEC's git branch untouched (Key Decision D8).
- **`/status SPEC-NNNN`** (G3) — detects `status: promoted` and reports the redirect to `superseded_by` instead of stale pipeline status. Detects `pending_promotion` set without `status: promoted` (an interrupted `/promote-to-thread` run) and reports "promotion in progress — resume via `/promote-to-thread SPEC-NNNN`" rather than stale draft/approved status.
- **`/create-investigation [--from <system>:<id>]`** (G5, new) — authors `INV-NNNN`; reuses G1's exact `source_work`/capture-policy mechanics when started from an external ticket, including D11's sanitized-failure and D12's content-containment rules. Walks Question → Approach → Findings → Decision. On `decision: adr`, hands off to `/create-adr`. On `decision: recommend_implementation`, writes the recommendation as plain text in the artifact — never creates or writes to any external ticket (Non-Goals still hold: advisory only).
- **`/create-adr <slug>`** (G5, new) — authors a standalone `ADR-NNNN`, same required sections as a DESIGN-nested one, independent of any parent DESIGN or SPEC.
- **`/promote-to-environment <environment>`** (G4, new) — valid only for an environment whose `governance` is `promotion` (per `environments` list); `governance: ci` has no command at all (AC14 — stays outside ForgeKit entirely); `governance: release` continues to use the existing RELEASE-PLAN commands unchanged. Appends one row to `sdlc/promotions/{environment}.md` with a per-participant `candidate_ref` map (reusing `approve-release-plan.md`'s exact shape), who, when, and a smoke-test result.
- **`/approve-design`'s Interface Freeze step** (G2) — gains one additional action when a thread's Cross-Repo Contract is approved: regenerate the coordination repo's `SYSTEM-ARCH.md` Dependency Map wholesale from the current set of `active` XDEPs (Key Decision D1/ADR-0002). Failure to regenerate (e.g. a malformed XDEP file) logs a warning and does **not** block the Interface Freeze approval itself — `SYSTEM-ARCH.md` is a derived convenience view, not a gate.

## Key Decisions

- **D1 (resolves OQ1):** `SYSTEM-ARCH.md`'s Dependency Map is regenerated wholesale at `/approve-design`'s Interface Freeze step — the exact moment an XDEP transitions to `active` — not by a separate sync command and not incrementally patched. See ADR-0002.
- **D2 (resolves OQ3):** The system-architecture artifact keeps the name used throughout SPEC-0001's drafting — `SYSTEM-ARCH.md`.
- **D3 (resolves OQ4):** Investigation gets a new standalone command (`/create-investigation`) and artifact type (`INV-NNNN`), not an extension of `/create-bug` or `/create-spec` — their lifecycles are categorically different (a BUG always resolves to a fix; a SPEC always resolves to a build; an Investigation may resolve to nothing at all). See ADR-0003. The ADR-authoring outcome hands off to a new standalone `/create-adr` command, reusable beyond Investigations.
- **D4 (resolves OQ2):** Promotion idempotency is achieved via Option B (state-inspection, not a checkpoint file). See ADR-0001.
- **D5 (resolves OQ6):** "Checking whether access exists" (SPEC AC1/AC4) is not a separate probe — it is the real fetch/write attempt itself. A clean failure signal indicating no working connector (as opposed to a transient error) triggers the copy-paste fallback. This avoids building a second mechanism just to predict what the real attempt would have told us anyway.
- **D6 (resolves OQ9):** If the user declines the compliance confirmation (SPEC AC2), `/create-spec` still proceeds — it records `source_work` and a `product_spec` block with `capture_status: declined`, `captured_at: null`, `content_digest: null`. The SPEC is not blocked; only the raw-content capture is skipped. Provenance is honestly marked as incomplete rather than silently absent.
- **D7 (resolves OQ8):** A `/forge-init`-time "not yet" answer to the coordination-repo question is not permanently sticky, but is also not proactively re-asked — it is simply left `unset`, re-askable only through `/forge-init`'s existing Option C (project identity update) or a full redo. No new re-prompt mechanism is introduced.
- **D8 (resolves OQ7):** A promoted SPEC's own git branch is left untouched — promotion is a content-authority transition, not a git-hygiene operation. ForgeKit doesn't delete or merge branches anywhere else in its existing model either; this stays consistent.
- **D9 (failure-mode rule for AC4):** Any non-confirmed-full-success on the write-back path — including partial success — is treated as failure. There is no "partially updated" state; it's either a confirmed success or it falls back to the copy-paste summary.
- **D10 (record shape for G4's `governance: promotion`):** Promotion records use an append-only per-environment log (`sdlc/promotions/{environment}.md`), not a numbered per-event artifact — the latter would recreate the exact ceremony-for-routine-events problem G4 exists to solve. See ADR-0004.
- **D11 (failure-diagnostic sanitization, added Round 1 per `[SEC-1]`):** Any failure surfaced from AC1's fetch attempt or AC4's write attempt — shown to the user, or recorded in any committed artifact — must be reduced to a sanitized failure category (e.g. `authentication_failed`, `not_found`, `network_error`) before display or persistence. Raw connector error output is never shown or committed verbatim; it could echo credential fragments (a token embedded in a header, a signed URL) from the underlying ADO/Confluence connector. **Addendum, 2026-10-05 (per Round 2's non-blocking `[SEC-1]`, picked up as `/create-plan`'s own `[ARCH-1]` finding):** this rule extends uniformly to `/promote-to-thread`'s own coordination-repo-unreachable errors, not just the AC1/AC4 connector path — a raw git/network error there could analogously leak a credential (e.g. a token embedded in a remote URL), the same risk class this decision exists to prevent.
- **D12 (external-content containment, added Round 1 per `[SEC-2]`):** Content obtained via G1's intake path — whether fetched directly or pasted — is always presented to the TPO role as quoted reference material to read and interpret when drafting Goals/ACs, never as instructions the role executes. This is the architectural location where SPEC-0001's own "external content is data, not instructions" principle is actually enforced, not merely assumed to survive unstated from SPEC into implementation — the same framing `cross-repo-resolution.md` already applies to other cross-repo inputs, now explicitly anchored to the specific commands that read this content (`/create-spec`'s intake step, `/create-investigation`'s equivalent).

## Alternatives Considered

See Design Options above (Option A and Option C, both rejected) for G3. For D3, the rejected alternative — extending `/create-bug` or `/create-spec` with an "investigation mode" flag — is detailed in ADR-0003's own Alternatives Considered section. For D1, the rejected alternative — a separate on-demand sync command — is detailed in ADR-0002.

## Impact on Existing Systems

- `/create-spec`, `/approve-spec`, `/forge-init`, `/status`, `/approve-design` all gain new conditional branches but no change to their existing unconditional behavior — a SPEC with no `source_work`, a repo with no `coordination_repo` question answered, a thread with no XDEP changes, all behave exactly as today.
- No existing artifact's required sections change shape — only new optional frontmatter fields are added, consistent with "additive, not breaking."
- `approve-release-plan.md`'s `candidate_ref` shape is reused, not modified, by `/promote-to-environment` — no change to that command itself.

## Performance Considerations

Not warranted — SPEC-0001's own Performance NFR already established no runtime path exists. Every command here is authoring-time or governance-time, human-invoked, not latency-sensitive.

## Observability Implementation

Not warranted — SPEC-0001's own Observability Considerations already concluded no metrics/traces/logging mechanism is needed beyond the artifacts' own git history. `/design-observability` is expected to find nothing to add here; noted here rather than silently skipped so that expectation is explicit going into Stage 4.

## ARCH Updates Required on Approval

N/A for ForgeKit's own repo — no `ARCH.md` exists to update (see this DESIGN's own Required Inputs substitution). For any adopting project's repo exercising `/promote-to-thread`, `/create-investigation`, `/create-adr`, or `/promote-to-environment` for the first time, that repo's own `ARCH.md` should eventually note these commands exist — deferred to that repo's own `/approve-implementation` Step 6 sync (already built, per the earlier ForgeKit stress-test fix), not something this DESIGN touches directly.

## Design Review — Round 1

**Date:** 2026-10-05
**Status:** Changes requested — 3 blocking, 1 non-blocking

### Options Analysis Findings

Reviewed the Option B recommendation itself (not just its presentation): options present (3) ✓, genuinely distinct ✓, recommendation reasoning cites SPEC's Reliability NFR and real ARCH precedent (ID-collision checks, self-ref pattern), not just "simpler" ✓. No wrong-option or missing-option finding — Option C is correctly disqualified against an approved NFR, and Option A's extra mechanism is correctly weighed against Option B's reuse of an established pattern. ADR-0001's Alternatives Considered correctly references this analysis. Clean — no finding here. (The real gap found this round is in how Option B's own idempotency property is specified, not in which option was chosen — see `[LD-1]` below.)

### Security Findings

- **[SEC-1] blocking** — AC1/AC4's failure paths (a failed fetch, a failed write-back) have no stated requirement that the failure diagnostic shown to the user — or recorded anywhere — be sanitized first. A verbose error from a failed auth attempt against ADO/Confluence could echo credential fragments (a token embedded in a header, a signed URL). SPEC-0001's own Security Considerations treated committed-content risk as serious enough to require an explicit, irreversibility-aware mitigation (the `.source.md` compliance gate) — this DESIGN needs the equivalent for failure diagnostics: never show or record raw connector error output verbatim; surface only a sanitized failure category (e.g., "authentication failed," "ticket not found," "network error") to the user and to any committed artifact.
- **[SEC-2] blocking** — DESIGN never restates or architecturally locates SPEC-0001's own "external content is data, not instructions" principle for the actual mechanism this feature builds. G1's whole design is "fetch or paste third-party ADO/Confluence content, then have the TPO role draft a SPEC from it" — exactly the scenario the Security Perspective's prompt-injection-framing check exists for, independent of whether `project.yml`'s `stack.llm_agent_untrusted_input` can be checked for ForgeKit's own meta-repo. The DESIGN must state explicitly where/how this containment is enforced architecturally — e.g., fetched or pasted content is always presented to the TPO role as quoted reference material to interpret when drafting Goals/ACs, never as instructions the role executes — not simply assume the SPEC-level principle survives into the actual mechanism unstated.

### SRE Findings

- **[SRE-1] non-blocking** — "Coordination repo unreachable" isn't explicitly named among `/promote-to-thread`'s failure modes, though Option B's idempotent design already makes it safe (any failure just means "retry later," nothing is corrupted). Worth stating explicitly for operator clarity; not required, since the underlying safety property already covers it.

### Lead Developer Findings

- **[LD-1] blocking** — Option B's idempotency claim ("mints THREAD-NNNN only if one doesn't already exist for this SPEC") has no stated anchor making that check actually possible after an interruption. If minting the THREAD-NNNN and recording the link "SPEC-NNNN → THREAD-NNNN" happen in different steps (which the current Interface/API Changes description implies — minting, then several more steps, then the final `status: promoted`/`superseded_by` write), an interruption between minting and that final write leaves nothing connecting the two: a resumed run has no way to tell whether a THREAD-NNNN already exists for this SPEC specifically, among potentially many unrelated ones in the coordination repo. It would either mint a duplicate or have to guess. Fix: the very first step of `/promote-to-thread` must record the candidate `THREAD-NNNN` back into the promoting SPEC's own frontmatter (e.g. `pending_promotion: THREAD-NNNN`) *before* minting it in the coordination repo — giving every later step, including a resumed one, a concrete, git-committed anchor to check against. This also needs reflecting in **Component/Data Model Changes** (SPEC-NNNN frontmatter gains `pending_promotion`) and in **Interface/API Changes**' own `/promote-to-thread` description, both currently silent on this field — a direct cascade from this same gap, not a separate issue.

**Clean perspectives:** DBA (no database or schema in this DESIGN — every artifact is a markdown file; nothing to review)

### Contradictions Requiring Arbitration

None.

## Address Summary — Round 1

**Date:** 2026-10-05

### Addressed

- **[LD-1]** — Option B's idempotency claim had no stated anchor for checking "does a THREAD-NNNN already exist for this SPEC" after an interruption. Fixed by adding `pending_promotion: THREAD-NNNN` as `/promote-to-thread`'s literal first step, written before the coordination repo is touched at all — reflected in both Component/Data Model Changes and Interface/API Changes, closing the cascade noted in the finding itself.
- **[SEC-1]** — No requirement existed that failure diagnostics (AC1 fetch failure, AC4 write failure) be sanitized before display or persistence. Added as new Key Decision D11: reduce to a sanitized failure category, never show or commit raw connector output. Wired into `/create-spec`, `/approve-spec`, and `/create-investigation`'s own descriptions.
- **[SEC-2]** — SPEC-0001's "external content is data, not instructions" principle was never architecturally located in this DESIGN for the actual mechanism that reads third-party content. Added as new Key Decision D12: fetched/pasted content is always quoted reference material to the TPO role, never executed instructions. Wired into the same three command descriptions as D11.

### Not Addressed

- **[SRE-1]** — **accepted risk** — "Coordination repo unreachable" isn't individually named among `/promote-to-thread`'s failure modes, but Option B's idempotent design already makes any failure safe to retry — nothing is corrupted regardless of which specific failure occurred. Accepted as-is per the default non-blocking disposition; no judgment-free factual correction applies here, so not addressed without being asked.

### Self-caught during this revision (not a Round 1 finding)

- The `[LD-1]` fix introduced `pending_promotion` as a new intermediate SPEC state, but `/status`'s own description (G3) only accounted for the fully-`promoted` end state — an interrupted `/promote-to-thread` run would have shown stale draft/approved status instead of "promotion in progress." Fixed inline: `/status` now also detects `pending_promotion` set without `status: promoted` and directs the user to resume `/promote-to-thread` rather than minting a false stale-status read.

## Design Review — Round 2

**Date:** 2026-10-05
**Status:** Approved — no blocking findings

`[DBA]`: no new findings — domain unaffected by Round 1 address (still no database or schema in this DESIGN).
`[Options Analysis]`: no new findings — domain unaffected by Round 1 address (no option-analysis-related finding was addressed; Round 1's analysis stands).

### Security Findings

- **[SEC-1] non-blocking** — D11's sanitization requirement (added Round 1) is scoped to AC1's fetch and AC4's write failures — the ADO/Confluence connector path. It doesn't extend to whatever error `/promote-to-thread` surfaces if the coordination repo itself is unreachable. A raw git/network error in that path could analogously leak a credential (e.g. a token embedded in a remote URL), the same risk class D11 exists to prevent on the other path. Should extend D11's rule uniformly rather than scoping it to one connector.

### SRE Findings

Re-checked concurrent-writer safety specifically against the new `pending_promotion` mechanism (two people, or one person twice, invoking `/promote-to-thread` on the same SPEC near-simultaneously): both would race to commit `pending_promotion` to the same SPEC branch — but this is an ordinary concurrent-commit-to-the-same-branch scenario, already handled by git's own push/merge-conflict mechanism, not a new race this DESIGN introduces or needs to solve itself. Confirmed clean — no new finding.

### Lead Developer Findings

- **[LD-1] non-blocking** — Wording imprecision in the `/promote-to-thread` description: "before touching the coordination repo at all" is slightly too strong — determining the candidate `THREAD-NNNN` number necessarily requires *reading* the coordination repo first (to find the next available number), which already happens before `pending_promotion` is committed. The actual guarantee that matters — no *write* to the coordination repo happens before the anchor is committed — still holds; the word "touching" should say "writing to."
- **[LD-2] non-blocking** — No stated behavior for the case where `pending_promotion`'s anchored `THREAD-NNNN` number is later found to already be claimed by a different SPEC's concurrent promotion (a rare race on the same candidate number, resolved today only at the existing CI collision-check layer per `cross-repo-resolution.md`). Worth stating explicitly: on that rejection, `/promote-to-thread` re-mints a new number and updates `pending_promotion` accordingly, rather than erroring opaquely — cheap to state now, not required given the low likelihood and existing infrastructure.

**Clean perspectives:** SRE (re-examined this round; DBA and Options Analysis were not re-examined — see the domain-unaffected lines above)

### Contradictions Requiring Arbitration

None.

