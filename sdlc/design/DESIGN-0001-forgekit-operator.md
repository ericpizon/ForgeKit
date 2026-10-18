---
id: DESIGN-0001
type: design
schema_version: 1
title: "ForgeKit Operator — Technical Design"
status: approved
version: 6
updated: 2026-10-07
current_review_round: 4
target_spec_version: 3
approval:
  mode: repo_only
  approved_by: "Eric Pizon"
  approved_role: architect
  approved_at: "2026-10-07"
  decision_summary: "DESIGN-0001 (ForgeKit Operator) re-approved after a Stage-4-discovered back-cycle (SEC-0001/OBS-0001 findings resolved); Option A (Interpretive Autonomy Overlay) confirmed; 3 ADRs accompany it."
  external_event_id: null
  external_event_uri: null
policy_evaluation:
  evaluated: null
  policy_pack: null
  result: null
changelog:
  - "v2 — Added an OPRUN-NNNN lookup-before-create convention (DBA-1); a pre-existing-artifact fallback rule for /status (DBA-2); a live-vs-persisted split for the compliance-gate decline case (SEC-1); and a full gate-to-outcome resolution procedure (§3.6, LD-2). Addresses Design Review Round 1's 4 blocking findings; non-blocking findings accepted as-is."
  - "v3 — Corrected Key Decision 5's outcome count from 11 to the schema's actual 12, explicitly separating the one active/non-stopping `in_progress` value from the 11 true stopping outcomes. Addresses Design Review Round 2's 1 blocking finding (DBA-3); DBA-4 accepted as-is."
  - "v4 — Fixed §3.6's opening sentence, which still described `OPRUN-NNNN.status` as having 11 total values after v3 corrected that count to 12. Wording-only; no functional change. Addresses Design Review Round 3's 1 non-blocking finding (DBA-5), fixed directly as a clear factual correction per address-design-review's own rule for such findings."
  - "v4 — approved after 3 review rounds"
  - "v5 — revision_requested 2026-10-07 — discovery: SEC-0001 found security-review.md's Handoff lacks a mechanically-distinguishable sub-step for an accepted_risk disposition, contradicting §3.3/ADR-0003's per-step classification claim, and that the resumption read-path needs an explicit containment restatement; OBS-0001 found OPRUN-NNNN's schema is missing fields SPEC §34's metrics require."
  - "v6 — Resolved the Discovery: §3.2's Autonomy Overlay Rule gained a findings_status content-check special case for security-review.md (any accepted_risk entry, old or new, forces a hard stop — deliberately not trying to diff 'new this round' given findings_status has no per-entry round data); §3.3's table updated to match; §3.4 gained the SEC-3 resumption-containment rule and the oprun_lookup_result/recovery_event logging points; §3.5 gained the ambiguous_cause distinction; §4 formalized the history: entry schema (actor/oprun_lookup_result/recovery_event) and changed escalation.reason from free text to an enum; §8 moved security-review.md into 'requires modification' with the rationale for choosing the lighter-weight content-check fix over SEC-0001's suggested Handoff restructuring. ADR-0002 updated with the ambiguous_cause field. No Goal, Non-Goal, or AC changed — all additive refinements within the already-approved architecture."
  - "v6 — approved after 4 review rounds"
generation:
  command: /create-design
  command_version: 3
  model: claude-sonnet-5
  generated_at: 2026-10-07
  session_id: null
---

# DESIGN-0001 — ForgeKit Operator

## 1. Design Options

### Option A: Interpretive Autonomy Overlay (one new command, zero edits to existing commands)

**Approach:** `/operate` resolves state via the extended `/status` resolver, then has the agent read and follow the next sub-command's own instructions directly — exactly as if a human had invoked it — except that wherever a sub-command's Handoff asks for a routine, non-gate confirmation, the agent checks the Autonomy Policy's `allow.*` categories (a small, fixed table stated once inside `operate.md`) and either self-answers or stops. Formal `/approve-*`/`/reopen-*` gates, and any Handoff step matching a Safety Invariant or Human Decision Boundary, are never self-answered — this classification happens per Handoff *step*, not per command, since a single command (e.g. `security-review.md`) can contain both kinds.

**Existing code integration:** zero changes to any of the ~50 existing command templates' engineering text. New: `operate.md`, `project.yml`'s `operator:` block, the `OPRUN-NNNN` artifact type, and the §41 structured-field additions (needed under any option, since they make gate state parseable regardless of who's reading it).

**Pros:**
- Lowest blast radius — no edits to proven existing command text.
- Matches the Core Principle literally: all new orchestration logic lives in exactly one new file.
- Joins the established `primary_role: any` precedent (`/status`, `/promote-to-thread`, `/promote-to-environment`, `/create-investigation`, `/create-adr`) ARCH.md already documents.

**Cons:**
- `operate.md` must generically recognize every sub-command's Handoff shape; an unusually-worded future Handoff could go unrecognized (fails safe — defaults to asking — but silently under-delivers autonomy rather than erroring loudly).
- The command-to-policy-category mapping is knowledge living in one file, not declared by each command itself; a new command added to the pipeline later needs a human to remember to add it.

**Risk:** Low.

### Option B: Operator-aware Handoff mode in every command

**Approach:** add an explicit `--operator-run <id>` convention to every command with a confirmation Handoff; each command checks the Autonomy Policy itself and declares its own autonomy behavior.

**Existing code integration:** touches every one of the ~50 command templates that has a human-confirmation Handoff step (`create-spec`, `create-design`, `create-plan`, `create-test-plan`, `design-observability`, every `address-*-review`, `implement-plan`, `execute-tests`, `create-pr`, `promote-to-environment`, `create-bug`, etc.).

**Pros:**
- Each command explicitly owns its own autonomy behavior — fully reviewable and versioned like the rest of its text.
- A new command added later is forced to make an explicit, visible choice rather than relying on someone remembering to update a separate table.

**Cons:**
- A ~50-file edit surface for one feature, directly against the "orchestrates, doesn't replace" framing.
- A much larger-scale version of the exact cross-file drift ARCH.md's own Known Tech Debt already documents as a real, observed problem (the solo/team confirmation-phrasing drift across a handful of files) — this option multiplies that risk class across the entire command surface.

**Risk:** Medium.

### Option C: External Operator runtime outside the markdown-instruction model

**Approach:** a script or process that manipulates ForgeKit artifacts and git directly via its own coded logic, rather than an AI agent reading and following each command's own instruction text.

**Existing code integration:** re-implements, in code, the engineering reasoning ForgeKit's command templates currently express in prose for an AI agent to interpret — replaces rather than orchestrates.

**Pros:**
- Potentially more deterministic per-step execution than an AI agent re-reading a markdown file each time.

**Cons:**
- Directly contradicts the Core Principle ("must not become a second implementation of ForgeKit's engineering intelligence") and two explicit Non-Goals ("does not introduce a generic workflow-definition language," "does not introduce an Operator-owned engineering-state database") — this option effectively requires both.
- ARCH.md explicitly characterizes this repo as having no runtime, framework, or deploy target of its own; this introduces exactly that, as a second execution paradigm alongside the existing AI-interpreted-markdown one.

**Risk:** High — a scope/governance risk, not an implementation one.

## 2. Recommended Option

**Recommended: Option A.**

Reasoning: smallest possible footprint while satisfying the Core Principle (SPEC-0001 §3) and multiple Non-Goals (§5: "does not replace existing ForgeKit commands," "does not introduce a generic workflow-definition language," "does not introduce an Operator-owned engineering-state database") exactly. ANALYSIS-0001's `[LD-12]` (Round 2) asked only for "a way... to recognize" Operator-driven invocation — not a mandate that every command declare this itself — so Option A satisfies that finding at its narrowest reasonable scope. It also matches ARCH.md's established "role-driven, not tool-driven" pattern directly: `operate.md` simply joins the existing `primary_role: any` precedent.

Trade-off accepted by not choosing Option B: future command authors don't get a forced, structural prompt to declare Operator-mode behavior when a new command is added. Mitigation (see §6 Key Decisions, ADR-0001): Tier 1 structural review (`/review-forgekit-pr`) is extended to check that every command with a confirmation Handoff step appears in `operate.md`'s categorization table, catching drift at review time rather than preventing it structurally.

Option C is rejected outright, not a live trade-off — it builds the thing this SPEC explicitly rules out.

## 3. Architecture Overview

### 3.1 System boundaries

`/operate` is invoked directly by a human, in a single continuously-running agent session (per SPEC-0001 §48 — no daemon, no unattended re-trigger). It shares exactly the access the invoking human already has to git, the filesystem, and (optionally) `gh` — it introduces no new authentication mechanism and crosses no new security boundary (confirmed by SPEC §49: "No new credential or auth surface").

It reads: `project.yml` (the new `operator:` policy block), the work unit's own artifacts (SPEC through RELEASE-PLAN, BUG/INCIDENT, THREAD/XDEP for multi-repo), and the extended `/status` resolver's `resolved_state` output.

It writes: `OPRUN-NNNN` run records (new artifact type, `sdlc/operator-runs/`), plus whatever artifacts the invoked sub-command naturally writes — unchanged, since Option A makes no change to any sub-command's own write behavior.

### 3.2 The Command Execution Cycle, concretely

SPEC-0001 §15 states the cycle abstractly (resolve → determine → verify policy → invoke → verify outcome → re-resolve). Under Option A, "invoke the existing ForgeKit command" means: the same agent, in the same session, reads the target command's `.md` file and follows its instructions exactly as if a human had typed that command — with one interpretive rule layered on top, applied while following *any* sub-command's text:

> **The Autonomy Overlay Rule:** when a sub-command's Handoff reaches a point that asks the user to confirm an action, first classify that specific pause:
> - If it is a formal `/approve-*` or `/reopen-*` gate, or matches anything named in SPEC-0001 §16 (Human Decision Boundaries) or §32 (Safety Invariants) — **always stop** and record a `HUMAN_DECISION_REQUIRED` escalation in the current `OPRUN-NNNN`, regardless of policy.
> - **Special case — a draft that itself carries a formal-gate disposition:** before self-answering `security-review.md`'s routine pause specifically, first scan the draft's own `findings_status` map for *any* entry whose value is `accepted_risk` — not only ones changed this round; `findings_status` has no reliable per-entry round/timestamp to diff against within an uncommitted review cycle, so deliberately erring toward the simpler, stricter rule. If any such entry exists, this pause is *not* routine — treat it as a formal gate (per the bullet above), regardless of the `create_artifacts` policy key. In practice this means any SEC-NNNN artifact carrying a standing accepted-risk entry (common — see SEC-0001's own `RET-1`/`RET-2`) always requires human confirmation to commit, for the life of that artifact; this is an acceptable cost, since SEC-NNNN commits are infrequent, high-stakes events, not a routine autonomous action. This is the one place a command's own draft *content*, not just its Handoff wording, determines the classification — because, unlike `execute-release.md` or `promote-to-environment.md`, `security-review.md`'s Handoff text has only one combined pause with no separate step to key off of (§3.3's table note; added during a Design Review Round 2 downstream discovery, SEC-0001 `[SEC-2]`).
> - Otherwise (a routine "please review this draft before I commit" pause) — look up the invoking command in the categorization table below. If the corresponding `operator.allow.*` key in `project.yml` is `true`, answer the pause as "yes, proceed" on the human's behalf, and record that this confirmation was given autonomously (which command, which pause, under which policy key) in the current run's history. If the key is `false` or absent, **stop** with `HUMAN_DECISION_REQUIRED`.

### 3.3 Command → Autonomy Policy category table

| Command | Policy category | Notes |
|---|---|---|
| `create-spec`, `revise-spec` | `create_artifacts` / `revise_artifacts` | `validate-spec` is excluded below — see note |
| `create-design`, `address-design-review` | `create_artifacts` / `revise_artifacts` | |
| `security-review` | `create_artifacts` — **with a special-case content check** (§3.2): any `accepted_risk` entry in `findings_status`, old or new, forces a hard stop | its `accepted_risk` disposition is a formal gate (AC-27) regardless of policy. Resolved via a `findings_status` content scan rather than a separate Handoff step — `security-review.md`'s own text has only one combined pause, unlike `execute-release.md`/`promote-to-environment.md`/`execute-tests.md`, which do separate their formal-gate case into a distinct step (corrected during a Design Review Round 2 downstream discovery, SEC-0001 `[SEC-2]`) |
| `design-observability` | `create_artifacts` | |
| `create-plan`, `address-plan-review` | `create_artifacts` / `revise_artifacts` | |
| `create-test-plan`, `address-test-plan-review` | `create_artifacts` / `revise_artifacts` | |
| `implement-plan` | `implement_code` | |
| `execute-tests`, `address-test-failures` | `run_tests` | QE's `production_bug` vs. `test_bug` classification (AC-12) is always a formal gate |
| `create-pr`, `close-milestone` | `create_pr` | |
| `create-release-plan`, `address-release-plan-review` | `create_artifacts` / `revise_artifacts` | |
| `execute-release` | `nonproduction_promotion` (non-prod only) | production execution is always a formal gate (AC-20, `require_human.production_release`) regardless of policy |
| `promote-to-environment` | `nonproduction_promotion` (non-prod only) | `release`-governance promotions are always a formal gate |
| `promote-to-thread` | *(not policy-gated)* | invoked directly per SPEC §25/AC-18 when ANALYSIS/DESIGN discovers a second participant; idempotent by construction, no routine confirmation to classify |
| `create-bug` | `create_artifacts` | triggered by a `production_bug` classification, per §20 |
| `approve-*`, `reopen-*` | *(excluded — always a formal gate)* | never appear in this table as autonomous-eligible |
| `validate-spec` | *(excluded — always human-run directly)* | product validation is itself the human-in-the-loop step; Operator stops and tells the human to run it, never answers it |

This table is the one artifact Option A asks a future command author to remember to update (the accepted trade-off from §2). `/review-forgekit-pr`'s Tier 1 checks are extended (see §8) to catch a command added without a corresponding row.

### 3.4 Operator run record (`OPRUN-NNNN`)

One file per run, `sdlc/operator-runs/OPRUN-NNNN.md` — a plain ForgeKit artifact like any other, so it's committed, diffable, and durable across sessions without inventing a new storage mechanism. See §4 for the full field list. Per SPEC §6, when this record's own understanding of state ever conflicts with the authoritative artifacts it was derived from, the artifacts win — `OPRUN-NNNN` is a convenience/history record, never itself a second source of truth. A fresh `/operate` invocation on an existing `OPRUN-NNNN` with a non-terminal `status` resumes it after re-resolving state from scratch; it never trusts its own prior `last_checkpoint` without re-verifying it.

**Lookup-before-create (resolves Design Review Round 1 `[DBA-1]`):** before creating a new `OPRUN-NNNN`, `/operate` scans `sdlc/operator-runs/` frontmatter for an existing record whose `work_unit` matches the one just invoked and whose `status` is in the **non-terminal** set (§4 defines terminal vs. non-terminal explicitly) — if found, that run is resumed, never duplicated. Only when no non-terminal record exists for this `work_unit` does `/operate` create a new `OPRUN-NNNN`. This is the mechanism that makes AC-11's idempotency guarantee apply to `OPRUN-NNNN` itself, not only to the artifacts the sub-commands it invokes produce. The outcome of this lookup (`resumed_existing` or `created_new`) is logged into the first `history:` entry of the invocation (§4's `history:` schema) — this is what makes the idempotency guarantee itself auditable rather than merely asserted (OBS-0001 `S7`).

**Resumption containment (resolves Design Review Round 2 downstream discovery, SEC-0001 `[SEC-3]`):** `OPRUN-NNNN` is a first-party ForgeKit artifact, the same structural trust tier as a SPEC or DESIGN file — but its body may contain previously-quoted external content (from an earlier Human Decision Package). Every re-read of an `OPRUN-NNNN`'s `history:` list or decision-package body on resumption — including content Operator itself previously wrote — is subject to the exact same data-not-instruction containment as a first-time external read (SPEC §49, ARCH.md's standing convention). No content earns elevated trust by virtue of being self-authored or previously processed; a resumed run must never let anything in its own prior body alter its policy or advance it past an escalation it was already on.

**Interrupted-implementation recovery logging (OBS-0001 `S9`):** on the resume path, immediately upon re-resolving state and discovering `implementation_execution.state: started` with no `completed_at` for the milestone about to be acted on (SPEC §12/§37), `/operate` appends a `recovery_event: interrupted_implementation_detected` entry to `history:` *before* invoking the implementation recovery path, never after — this is what makes Safety Invariant #16 (never send partial work to review) independently auditable rather than only behaviorally true.

### 3.5 `/status` extension

`/status` gains the general pre-PLAN form (reading SPEC/ANALYSIS/DESIGN/OBS/SEC frontmatter directly, the same way its existing PLAN-and-later form already reads PLAN/TEST/milestone frontmatter), a corrected milestone branch in its Next Action Inference (explicit handling for `test_failed`/`tests_passed`/`pr_open`, which it previously had no branch for at all), and a new `resolved_state` structured block appended to its existing human-readable output (schema in §4, decision in ADR-0002). None of this changes `/status`'s existing output for a human reading it directly — it's additive.

**Pre-existing-artifact fallback (resolves Design Review Round 1 `[DBA-2]`):** when `/status` needs a structured field this feature adds (`latest_round`, `implementation_execution`, etc.) to compute `resolved_state`, and the artifact it's reading predates this feature and lacks that field entirely, `/status` must **not** fall back to parsing the artifact's prose — doing so would defeat AC-06's "never parse prose" guarantee for exactly the artifacts where it matters most (the ones Operator is most likely to encounter first, from before this feature shipped). Instead, the absence of a required structured field forces `resolved_state.gate: ambiguous`, the same outcome as any other case `/status` cannot resolve cleanly. A human can then either run the normal `/review-*` command for that stage once (which backfills the field as a side effect, per §4) or manually add the field, after which a re-run resolves normally. This is a one-time cost per pre-existing artifact, not a standing limitation.

`resolved_state` distinguishes *why* `gate` came back `ambiguous` via a new `status_resolver.ambiguous_cause` field (added per a Design Review Round 2 downstream discovery, OBS-0001 `S8` / ADR-0002): `missing_structured_field` for exactly the one-time, self-healing case above, versus `unrecognized_state_combination` for every other case `/status` cannot resolve. This split matters operationally — the first is an expected migration cost, the second is a genuine resolver gap worth filing a BUG against (ADR-0002 updated accordingly).

### 3.6 Gate-to-outcome resolution procedure

*(New — resolves Design Review Round 1 `[LD-2]`.)* ADR-0002 states that `resolved_state.gate`'s four values are deliberately coarser than the 11 stopping outcomes among `OPRUN-NNNN.status`'s 12 total values (§4 Key Decision 5), and that `/operate` computes the mapping — but did not specify that computation. It is a two-stage procedure, because gate classification (content-level) and execution outcome (infrastructure-level) are genuinely different axes: `gate: ready` can still fail mechanically (git unreachable), and a content-level block can resolve several different ways depending on policy and context.

**Stage A — pre-check**, before `resolved_state.gate` is even computed:
- `work_unit` does not resolve to any known artifact → `status: not_found`, stop.
- `work_unit` resolves but has no further eligible lifecycle action (a terminal SPEC/THREAD state, e.g. already `promoted`/`shipped`) → `status: already_complete`, stop.

**Stage B — classify `resolved_state.gate`:**
- `ready` → proceed to Stage C.
- `ambiguous` → `status: ambiguous_state`, stop.
- `human_decision_required` → `status: human_decision_required`, stop. This single status covers every content-level case SPEC names as a hard human boundary regardless of its specific cause — non-convergence (AC-08), test classification (AC-12), a security/policy exception (AC-27), or a genuine cross-repo dependency *cycle* (AC-17/§24) — `/operate` records the specific cause in `escalation.reason`, but the `status` value itself stays the one generic value; SPEC never asked for a dedicated status per cause, only that each one reliably reaches a human.
- `blocked` → one of four outcomes, in this order:
  1. If resolving it is an autonomous correction loop (AC-07) and the invoking command's `operator.allow.*` category permits it → proceed to Stage C (attempt the correction).
  2. If it's an autonomous correction loop the policy does *not* permit → `status: human_decision_required`, stop.
  3. If it's specifically waiting on another participant's not-yet-done, non-cyclic, eventually-resolvable work (ordinary multi-repo sequencing, not a deadlock) → `status: blocked_dependency`, stop — resumable later without a human, by contrast with the cyclic case above.
  4. If it's specifically an unavailable or misconfigured target environment → `status: blocked_environment`, stop.

**Stage C — attempt the action, then classify the attempt itself:**
- Taking this action would exceed `operator.max_actions_per_run` → `status: action_limit_reached`, stop (checked *before* the action runs, never after).
- Action succeeds → re-resolve state (back to Stage A); if nothing remains eligible, `status: complete`, stop; otherwise continue the cycle.
- Action fails due to a transient infrastructure issue (network) → `status: retryable_error`, stop.
- Action fails due to a setup/auth issue requiring a human to fix outside this run (e.g. `gh` unauthenticated) → `status: external_action_required`, stop.
- Action fails in a way no retry or human action through this path can resolve (e.g. a structurally malformed artifact) → `status: terminal_error`, stop.

## 4. Component / Data Model Changes

**New artifact type — `OPRUN-NNNN`** (`sdlc/operator-runs/OPRUN-NNNN.md`):
```yaml
id: OPRUN-NNNN
type: operator-run
schema_version: 1
work_unit: SPEC-NNNN | THREAD-NNNN
status: in_progress | complete | human_decision_required | external_action_required |
        blocked_environment | blocked_dependency | retryable_error | terminal_error |
        ambiguous_state | not_found | already_complete | action_limit_reached
started_at: <timestamp>
updated_at: <timestamp>
attempt_count: <int>
actions_taken: <int>              # counted against operator.max_actions_per_run
last_attempted_action: <string>   # exact command invocation
last_checkpoint:
  stage: <string>
  gate: <string>
  subject: <ARTIFACT-NNNN or THREAD-NNNN>
review_loop_counts:
  <ARTIFACT-NNNN>: {round: <int>, count: <int>}
escalation:
  reason: non_convergence | test_classification | security_policy_exception |
          dependency_cycle | other | null   # enum, not free text — resolves a
                                              # Design Review Round 2 downstream
                                              # discovery, OBS-0001 S3
  decision_package:
    capture_status: captured | redacted_on_decline   # see compliance-gate note below
generation:
  command: /operate
  command_version: <int>
  model: <string>
  generated_at: <date>
  session_id: null
```

**Terminal vs. non-terminal `status` (resolves `[DBA-1]`'s lookup convention above):**
- **Terminal** (a finished run; a fresh `/operate` invocation on the same `work_unit` starts a *new* `OPRUN-NNNN`, never resumes this one): `complete`, `terminal_error`, `not_found`, `already_complete`.
- **Non-terminal** (a paused run; a fresh `/operate` invocation on the same `work_unit` resumes this record): `in_progress`, `human_decision_required`, `external_action_required`, `blocked_environment`, `blocked_dependency`, `retryable_error`, `ambiguous_state`, `action_limit_reached`.

**`history:` entry schema** *(formalized during a Design Review Round 2 downstream discovery — OBS-0001 needed a concrete shape to instrument against; previously stated only loosely as "one entry per Command Execution Cycle step"):*
```yaml
history:
  - at: <timestamp>
    action: <string>                       # exact command invocation
    actor: operator_autonomous | human_confirmed | null   # set only when this
                                            # action passed through a Handoff pause
                                            # (§3.2) — null for a step with no pause
                                            # to classify (OBS-0001 S2)
    oprun_lookup_result: resumed_existing | created_new   # present only on the
                                            # first entry of an invocation (§3.4,
                                            # resolves [DBA-1]; OBS-0001 S7)
    recovery_event: interrupted_implementation_detected    # present only when
                                            # applicable (§3.4; OBS-0001 S9)
    resolved_state: <resolved_state snapshot, per ADR-0002>
```
Fields marked "present only when applicable" are omitted entirely otherwise — they are not written as `null` on every entry, to keep the common-case entry small. Body also holds, when stopped, a rendered Human Decision Package (SPEC §17 fields). This is where SPEC §49's compliance-gate extension applies (§6, Key Decision 7) — any quoted artifact content in a decision package is scanned the same way `external-work-intake.md` already scans pasted ticket content.

**Compliance-gate decline handling (resolves Design Review Round 1 `[SEC-1]`):** unlike `external-work-intake.md`'s original use (where declining just skips an optional companion-file capture while the main artifact proceeds unaffected), here the gated content *is* the reason Operator stopped — AC-27/§17 require the human deciding right now to actually see it. So on a compliance-gate trigger: the full, unredacted content is still shown live, in-session, to the human making the decision; only the **persisted** copy written into `OPRUN-NNNN`'s body is affected — if the human declines the capture confirmation, the persisted decision package replaces the sensitive portion with a placeholder (`[redacted — shown live in session; not persisted per compliance-gate decline]`) and `escalation.decision_package.capture_status` is set to `redacted_on_decline` rather than `captured`. The live decision is never blocked by this; only what survives into git history is affected.

**New `project.yml` block:**
```yaml
operator:
  max_review_rounds: 3
  max_actions_per_run: 25
  allow:
    create_artifacts: false
    revise_artifacts: false
    implement_code: false
    run_tests: false
    create_pr: false
    nonproduction_promotion: false
  require_human:
    risk_acceptance: true        # always enforced true regardless of this value
    production_release: true     # always enforced true regardless of this value
```
Every `allow.*` key defaults to `false` — absence of the whole block, or absence of a specific key, both mean "false." This is the literal mechanism implementing AC-25 and Safety Invariant #18. `require_human.*` keys are shown for legibility but Operator treats them as permanently `true`; a project cannot configure around Safety Invariants #11/#20.

**New frontmatter field — `latest_round`** (added to DESIGN, PLAN, TEST-PLAN, IMPL-REVIEW, RELEASE-PLAN):
```yaml
latest_round:
  round: <int>
  blocking: <int>
  non_blocking: <int>
  verdict: ready | needs_revision
```
Written by the corresponding `review-*` command alongside its existing prose `**Status:**`/`**Summary:**` line — the prose stays for humans; this is additive.

**New convention — finding lineage.** A finding's bracket ID (`[LD-1]`) stays round-scoped as today; when an `address-*-review` command's disposition references a prior round's finding, it adds `recurs_from: {round: <int>, finding: <id>}` to that disposition entry. No new file — a documentation addition to each `address-*-review.md`'s "On Re-Review" section.

**New field — `implementation_execution`** (added to each milestone entry inside PLAN-NNNN, alongside the existing `**Implementation status:**` line):
```yaml
implementation_execution:
  state: started | completed
  attempt: <int>
  started_at: <timestamp>
  completed_at: <timestamp> | null
```
Written by `implement-plan.md` at invocation start (`state: started`, `attempt` incremented) and at successful completion (`state: completed`).

**Extended `RELEASE-PLAN` `approval:` block** — two new fields on the existing, already-universal schema:
```yaml
approval:
  # ...existing fields unchanged...
  candidate_ref: <ref/SHA>
  release_plan_revision: <int>   # == RELEASE-PLAN's own `version` at approval time
```
**New `RELEASE-PLAN` field — `deployment_progress`:**
```yaml
deployment_progress:
  migrations_applied: false
  deployed: false
  tagged: false
  updated_at: <timestamp>
```
Written incrementally by `execute-release.md` after each of its existing numbered steps, not only at the end.

**New field — `classification`** (column added to `execute-tests.md`'s test-execution table): `production_bug | test_bug`, recorded at the point QE answers — structured, not Notes-only prose.

**Canonical BUG status vocabulary** (documentation, not a new field — the existing `status` field's allowed values, declared once): `reported → accepted | rejected | needs_info → pending_verification → verified | reopened → incident_active → fix_deployed → closed`. Every command that reads or writes BUG `status` is audited against this single list (§8).

## 5. Interface / API Changes

No external or network-facing API exists in this repo (per ARCH.md's "no stack" framing) — Step 8's version-prefix requirement for external API surfaces does not apply, the same way it's N/A for every other ForgeKit command.

**New command surface:** `/operate <SPEC-NNNN|THREAD-NNNN>` — `primary_role: any`, `templates/commands/operate.md`.

**Extended command surface:** `/status <ID>` gains the `resolved_state` structured block and the pre-PLAN general form (§3.5) — its existing human-readable invocation and output shape are unchanged.

## 6. Key Decisions

1. **Orchestration mechanism: Interpretive Autonomy Overlay (Option A).** → **ADR-0001**.
2. **`resolved_state` schema and 4-value `gate` enum** (`ready | blocked | human_decision_required | ambiguous`), resolving SPEC §51 Open Question 1 (§7's `resolved_state` schema). → **ADR-0002**.
3. **Fail-closed Autonomy Policy defaults; per-Handoff-step (not per-command) classification of formal gates vs. routine confirmations**, resolving SPEC §51 Open Question 6 and operationalizing §48's distinction. → **ADR-0003**.
4. **`implementation_execution` lives on the milestone entry itself**, not a separate artifact — resolves SPEC §51 Open Question 2. Rationale: colocated with the state it qualifies; avoids a new artifact type for a field that's meaningless outside its milestone's context.
5. **`OPRUN-NNNN.status` is 12 values total: one active/non-stopping liveness state (`in_progress`) plus 11 stopping outcomes** — the SPEC's own 8 (`COMPLETE`, `HUMAN_DECISION_REQUIRED`, `EXTERNAL_ACTION_REQUIRED`, `BLOCKED_ENVIRONMENT`, `BLOCKED_DEPENDENCY`, `RETRYABLE_ERROR`, `TERMINAL_ERROR`, `AMBIGUOUS_STATE`) plus 3 gap-filling ones (`NOT_FOUND`, `ALREADY_COMPLETE`, `ACTION_LIMIT_REACHED`) identified during ANALYSIS-0001 Round 2/3 (`[QE-5]`) — resolves SPEC §51 Open Question 3. `in_progress` is categorically distinct from the other 11 and is not itself an "outcome": SPEC §18 only asked Operator to distinguish materially different *stopping* conditions, and `in_progress` is, definitionally, the not-yet-stopped state (corrected during Design Review Round 2 — `[DBA-3]` — which found this miscounted as 11 against the schema's actual 12).
6. **Release-approval binding reuses the existing universal `approval:` block**, adding only `candidate_ref`/`release_plan_revision` rather than inventing a parallel structure — resolves SPEC §51 Open Question 4.
7. **`deployment_progress` is written incrementally by `execute-release.md`'s existing steps**, not only at the end — resolves SPEC §51 Open Question 5, directly adopting the original gap analysis's G5 recommendation.
8. **Compliance-gate extension to Operator-persisted content reuses `external-work-intake.md`'s exact existing mechanism** (regex credential scan + `compliance.frameworks` check, same confirm/decline shape) — resolves SPEC §51 Open Question 7. No new security surface.

## 7. Alternatives Considered

Covered in full in §1 (Design Options) and carried forward into **ADR-0001**: Option B (per-command Operator-mode branches) and Option C (external runtime) were both rejected. See ADR-0002 and ADR-0003 for the alternatives considered on the `resolved_state` schema and the Autonomy Policy defaults specifically.

## 8. Impact on Existing Systems

**Commands requiring modification:**
- `status.md` — pre-PLAN general form, corrected milestone branch, `resolved_state` output block, now including `status_resolver.ambiguous_cause` (§3.5, ADR-0002).
- `security-review.md` — **moved here from "no modification" after a Design Review Round 2 downstream discovery (SEC-0001 `[SEC-2]`)**: this DESIGN resolves the finding via a `findings_status` content-check in `operate.md`'s own overlay (§3.2) rather than SEC-0001's suggested alternative of also adding a separately-labeled Handoff sub-step to `security-review.md` itself — the content-check achieves the same safety property with no change to this command's actual text, consistent with Option A's minimal-footprint principle. Listed here anyway (not "no modification") because this command's `findings_status` map must remain reliably readable by that check — a future edit to this command's output shape would silently break the special case.
- `review-design.md`, `review-plan.md`, `review-test-plan.md`, `review-implementation.md`, `review-release-plan.md` — `latest_round` field.
- `address-design-review.md`, `address-plan-review.md`, `address-test-plan-review.md`, `address-implementation-review.md`, `address-release-plan-review.md` — `recurs_from` finding-lineage convention.
- `implement-plan.md` — writes `implementation_execution`.
- `execute-tests.md` — adds `classification` column.
- `create-release-plan.md` / `approve-release-plan.md` — `candidate_ref`/`release_plan_revision` fields.
- `execute-release.md` — writes `deployment_progress` incrementally; checks approval binding before each step.
- `create-bug.md` and every command reading/writing BUG `status` (`triage-bug.md`, `resolve-incident.md`, `escalate-to-incident.md`, and siblings) — reconciled against the one canonical vocabulary (§4).
- `project.yml.template` — adds the `operator:` block with its fail-closed defaults.
- `review-forgekit-pr.md` — Tier 1 extended to check every command with a confirmation Handoff appears in `operate.md`'s categorization table (§3.3).
- `PIPELINE.md` — documents `/operate` in the pipeline overview.
- **New:** `operate.md`, `sdlc/operator-runs/README.md`.

**Commands requiring no modification:** everything in the milestone-implementation cycle beyond the two listed above (`address-test-failures.md`, `create-pr.md`, `close-milestone.md`), every `approve-*.md` (the existing `approval:`/`policy_evaluation:` schema is already sufficient), `security-review.md`'s `findings_status` map (already the cleanest gate in the corpus), `promote-to-thread.md`/`promote-to-environment.md` (already idempotent by construction), `thread.yml`'s schema (stays status-field-free per SPEC §23 — thread completion stays derived, never persisted), the XDEP schema (unchanged — SPEC §22 keeps XDEP satisfaction live-derived in v1), `merge-pr.md`/`review-pr.md`/`address-pr-review.md` (deprecated stubs, out of scope).

**ARCH.md updates required at `/approve-design`** (not applied now — per existing convention, `approve-design.md` applies these): extend the existing `primary_role: any` bullet to name `/operate`; add `sdlc/operator-runs/` to the Module layout section; extend the Testing Strategy's Tier 2 description to name Operator-specific scenario classes (interrupted/resumed sessions, fail-closed policy defaults, dependency deadlocks). All additive — no existing ARCH.md section is superseded.

## 9. Failure Modes

- **`OPRUN-NNNN` write/commit fails mid-run:** Operator must not report `COMPLETE`. Per SPEC §6, the authoritative artifacts remain the source of truth regardless of whether the run record itself persisted — a lost or corrupted `OPRUN-NNNN` never grants permission a fresh policy check wouldn't independently grant (AC-25 still applies).
- **`/status` resolver cannot parse an artifact or hits an unrecognized state combination:** Operator emits `AMBIGUOUS_STATE` and stops — the existing `/status` escape hatch, now formalized as a first-class outcome rather than a free-text fallback line.
- **`git`/`gh` unavailable when Operator needs them** (e.g., no network, `gh` unauthenticated): classified via the same sanitized-failure-category convention as `external-work-intake.md` — a transient network issue is `RETRYABLE_ERROR`; an auth/setup problem is `EXTERNAL_ACTION_REQUIRED`.
- **`max_actions_per_run` reached mid-sequence:** Operator stops with `ACTION_LIMIT_REACHED`, persists `last_checkpoint`, and never silently truncates the run or reports it as complete.

## 10. Forward Compatibility

`resolved_state`'s 4-value `gate` enum (ADR-0002) is deliberately small and stable — a new lifecycle stage is expected to map onto one of the four existing values, not require a new one; extending the enum itself needs a new ADR. `operate.md`'s categorization table (§3.3) is the one place that must be touched when a new pipeline command is added — an accepted, documented cost of Option A, not a hidden one. Operator's own entry points (`SPEC-NNNN`/`THREAD-NNNN`) and output (`OPRUN-NNNN`) are both plain ForgeKit artifacts, so a future Product Operator (SPEC §43) can invoke `/operate` the same way a human does today, with no special integration required.

## 11. Observability Implementation

Maps SPEC-0001 §50 to concrete fields: the `history:` list inside each `OPRUN-NNNN` body is the trace record (§50's "Traces"); the frontmatter fields (`status`, `actions_taken`, `review_loop_counts`, `escalation`) are the source for every metric in SPEC §34 (§50's "Metrics") — computed by reading `OPRUN-NNNN` files, never duplicated elsewhere. §50's logging redaction rule (never persist raw connector output, credential-shaped strings, or unredacted compliance-sensitive content) applies directly to what gets written into a decision package inside `OPRUN-NNNN`'s body — enforced via the same mechanism as Key Decision 8.

## 12. Performance Considerations

No new latency-sensitive path is introduced — `/operate` is driven by the same agent-reads-a-markdown-file mechanism as every other command, just repeated across a sequence. The only new performance-adjacent constraint is `max_actions_per_run` (§4), which bounds worst-case *side-effecting operations* per run (a correctness/safety control per SPEC §47's autonomy-containment NFR), not a latency target.

## Design Review — Round 1

**Date:** 2026-10-07
**Status:** Changes requested — 4 blocking, 6 non-blocking

### Options Analysis Findings

- [OPT-1] **non-blocking** — A hybrid wasn't considered: the generic overlay (Option A) for the ~35 commands with only routine pauses, plus explicit per-command branches (Option B's mechanism) only for the handful of genuinely mixed-pause commands (`security-review.md`, `execute-release.md`, `promote-to-environment.md`). Unlikely to change the recommendation — it still requires overlay logic for the majority of commands, and only trades 3 commands' worth of ambiguity for 3 explicit edits — but worth noting as a considered-and-rejected alternative rather than an unconsidered gap.

### DBA Findings

- [DBA-1] **blocking** — No lookup/indexing convention is stated for finding an existing in-progress `OPRUN-NNNN` for a given `work_unit` before creating a new one. This is the "hot path" query every `/operate` invocation needs, and without it, `OPRUN-NNNN` creation itself could violate AC-11's idempotency guarantee — N duplicate run records for N interrupted-and-resumed sessions against the same work unit. Recommend either (a) an explicit stated scan convention ("before creating `OPRUN-NNNN`, scan `sdlc/operator-runs/` frontmatter for `work_unit` == this ID and `status` in the non-terminal set — resume the most recent match instead of creating new"), or (b) a naming convention keyed by work-unit ID directly rather than sequential `NNNN`, making lookup trivial by filename alone.
- [DBA-2] **blocking** — No stated behavior for `/status`'s `resolved_state` computation when a pre-existing artifact (created before this feature's schema additions ship) lacks `latest_round`/`implementation_execution`/etc. Falling back to prose-parsing would defeat AC-06's "never parse prose" guarantee; silently treating it as `ready` would be unsafe. Needs an explicit decision — most consistent with AC-06 and ADR-0002 would be: absence of a required structured field forces `resolved_state.gate: ambiguous`, never a prose fallback — but the DESIGN should say so explicitly rather than leaving it implicit, since this directly determines whether any in-flight work predating this feature (including this very thread) can be Operator-driven without a manual backfill.

### Security Findings

- [SEC-1] **blocking** — Key Decision 8's reuse of `external-work-intake.md`'s compliance-gate confirm/decline shape doesn't fully transfer to `OPRUN-NNNN`'s decision packages. In the original use case, declining just skips an optional companion-file capture while the main SPEC proceeds unaffected; here, the gated content *is* the reason Operator stopped — if a human declines the compliance confirmation, Operator cannot simply omit the decision package, since AC-27/§17 require the human to actually see what's blocking before they can decide. DESIGN needs to state what happens on decline specifically for this case — most likely: the content is still shown live, in-session, to the human now deciding, but the *persisted* copy in `OPRUN-NNNN` is redacted/summarized rather than carrying the raw sensitive content forward in git history indefinitely. This needs to be designed for this specific situation, not inherited wholesale from a mechanism built for a different one.
- [SEC-2] **non-blocking** — SPEC §49's "artifact prose is data, never instruction" framing should be explicitly restated at the point of `OPRUN-NNNN` *resumption* (re-reading a past run's own decision-package content, which may embed quoted external content), not just at the point of initially quoting content into it — resumption is a genuinely new read-path this DESIGN introduces, and restating the existing principle there removes any ambiguity for an implementer.

### SRE Findings

- [SRE-1] **non-blocking** — §9's Failure Modes gives concrete triggering criteria for `AMBIGUOUS_STATE`, `RETRYABLE_ERROR`, `EXTERNAL_ACTION_REQUIRED`, and `ACTION_LIMIT_REACHED`, but not for `TERMINAL_ERROR` or `BLOCKED_ENVIRONMENT` — both real values in the outcome enum (Key Decision 5). Recommend adding examples (e.g., `BLOCKED_ENVIRONMENT`: the target environment itself is down per an existing environment-governance check; `TERMINAL_ERROR`: a sub-command's own Pre-flight check fails in a way no retry or human action can resolve) so an implementer has equally clear criteria for all eleven values.
- [SRE-2] **non-blocking** — No heartbeat/staleness signal is mentioned for a *stuck* (not stopped, not erroring) run. Lower severity than it would be in an unattended model, since SPEC §48 establishes a human-attended, single-session execution model — the human is presumably watching — but worth a one-line acknowledgment that `updated_at` staleness is the human's own signal to notice, rather than leaving it unstated.
- [SRE-3] **non-blocking** — The concurrent-writer check applies directly here: two `/operate` sessions against the same work unit is exactly "a resource with more than one writer path." SPEC §48 already scopes active prevention out ("assumed not to happen... not actively prevented by new tooling"), so this isn't a DESIGN gap to fix — but DESIGN should state explicitly *why* that's an acceptable risk: git's own natural conflict surface plus each sub-command's existing Pre-flight re-checks (e.g. `implement-plan.md` re-verifying milestone status before writing) provide real, if imperfect, protection even without new tooling. Stating this prevents a future reader from mistaking an accepted SPEC-level assumption for an unconsidered DESIGN hole.

### Lead Developer Findings

- [LD-1] **non-blocking** — The Autonomy Overlay Rule's "matches anything named in §16/§32" branch relies on the agent recognizing abstract categories (material scope ambiguity, conflicting requirements, etc.) that aren't mechanically checkable the way "is this file a `/approve-*`/`/reopen-*` command" is. Recommend an explicit tiebreak for genuinely ambiguous future cases: default to treating an unclear Handoff pause as a hard stop, never as autonomy-eligible by default — generalizing AC-25/Safety Invariant #18's fail-closed spirit to this judgment call, not a new principle.
- [LD-2] **blocking** — `resolved_state.gate` (ADR-0002) has exactly 4 values; `OPRUN-NNNN.status` (Key Decision 5) has 11. ADR-0002 states "finer distinctions... are computed by `/operate` itself," but no section of this DESIGN (§3, §4, or §9) actually specifies that computation — there is no stated mapping or decision procedure from a given `gate` value to the specific `status` value Operator records. This is a missing decision, not just missing detail: without it, two implementers would reasonably build incompatible logic, and AC-18/22/26 each name specific outcome values that something must decide to fire. Needs a concrete resolution table or procedure before this is implementable as a PLAN task.

**Clean perspectives:** none — every perspective raised at least one finding.

### Contradictions Requiring Arbitration

None. One cascade worth noting: `[DBA-1]`'s fix (explicit OPRUN lookup-before-create) partially mitigates `[SRE-3]`'s concurrent-run concern by reducing (not eliminating) the chance of two concurrent runs both independently starting fresh against the same work unit — worth cross-referencing when both are addressed, though `[SRE-3]`'s git-level argument stands independently for genuinely simultaneous invocations `[DBA-1]` can't catch.

## Address Summary — Round 1

**Date:** 2026-10-07

### Addressed
- `[DBA-1]` — Added an explicit lookup-before-create convention to §3.4: `/operate` scans `sdlc/operator-runs/` for an existing non-terminal `OPRUN-NNNN` matching the target `work_unit` before creating a new one. §4 now defines the terminal/non-terminal status split this depends on.
- `[DBA-2]` — Added an explicit fallback rule to §3.5: a pre-existing artifact missing a required structured field forces `resolved_state.gate: ambiguous` rather than falling back to prose-parsing, with a stated one-time recovery path (run the normal `/review-*` command once, or backfill manually).
- `[SEC-1]` — Added a live-vs-persisted split to §4's `OPRUN-NNNN` section: on a compliance-gate decline, the content is still shown live to the human deciding, but the persisted copy is redacted with a placeholder and a new `escalation.decision_package.capture_status: redacted_on_decline` field records this distinctly from a normal `captured` state.
- `[LD-2]` — Added §3.6, a concrete three-stage gate-to-outcome resolution procedure (pre-check → gate classification → action-attempt classification) mapping `resolved_state.gate`'s 4 values to `OPRUN-NNNN.status`'s 11 values, including the specific rule distinguishing a resolvable multi-repo wait (`blocked_dependency`) from a genuine dependency cycle (`human_decision_required`, per AC-17). ADR-0002 updated to point to this procedure rather than asserting the computation without specifying it.
- `[SRE-1]` (non-blocking, resolved as a side effect) — §3.6's Stage B/C, added to resolve `[LD-2]`, independently gives `TERMINAL_ERROR` and `BLOCKED_ENVIRONMENT` the concrete triggering criteria this finding asked for.

### Not Addressed
- `[OPT-1]` — **accepted** — the hybrid (overlay + explicit branches only for mixed-pause commands) is a reasonable alternative but, as the finding itself notes, unlikely to change the recommendation; not worth the added complexity of two mechanisms for a marginal reduction in ambiguity on 3 commands.
- `[SEC-2]` — **accepted** — restating the data-not-instructions framing at the resumption read-path is good hygiene but not a design change; will be reflected in `operate.md`'s own text at implementation time rather than requiring a DESIGN revision now.
- `[SRE-2]` — **accepted** — a one-line acknowledgment that `updated_at` staleness is the human's own signal in this human-attended execution model; not substantial enough to warrant a DESIGN section of its own.
- `[SRE-3]` — **accepted** — SPEC §48 already scopes active prevention of concurrent runs out; §3.4's new lookup-before-create convention (from `[DBA-1]`) additionally narrows the window this concerns, which is noted in the Round 1 Cross-Perspective cascade.
- `[LD-1]` — **accepted** — the fail-closed tiebreak for ambiguous future Handoff classification is a reasonable implementation-time default; will be stated directly in `operate.md`'s own text rather than requiring a DESIGN revision now.

## Design Review — Round 2

**Date:** 2026-10-07
**Status:** Changes requested — 1 blocking, 1 non-blocking

**Options Analysis:** no new findings — domain unaffected by Round 1's address (`[OPT-1]` was accepted as-is; no DESIGN section Options Analysis would re-examine was touched).

### DBA Findings

- [DBA-3] **blocking** — `[DBA-1]`/`[DBA-2]` are confirmed resolved, but re-examining the `OPRUN-NNNN.status` enum itself (§4) while checking those fixes surfaced a direct contradiction: Key Decision 5 states the outcome set is "the SPEC's own 8 values plus 3 gap-filling ones" — 11 total — but §4's actual schema lists **12** values, because `in_progress` (the run's active/not-yet-stopped state) is in the field but was never counted. This isn't cosmetic: `in_progress` is categorically different from the other 11 — it's a liveness state, not a stopping condition (SPEC §18 only ever asked Operator to "distinguish materially different *stopping* conditions") — and §3.6's own procedure uses `status: complete` and implicitly "stays `in_progress`" during Stage C's "otherwise continue the cycle" branch without ever reconciling this with Key Decision 5's "11 total" claim. Needs: Key Decision 5's text corrected to state 12 values explicitly (the 11 stopping outcomes plus the one active value), with `in_progress` called out as categorically distinct — not folded into "the outcome set" language as if it were a twelfth stopping condition.
- [DBA-4] **non-blocking** — The new `escalation.decision_package.capture_status: captured | redacted_on_decline` field (added for `[SEC-1]`) doesn't state what it reads — or whether the whole `decision_package` key is even present — for the common case where a run never triggers the compliance gate at all (no sensitive content ever detected). Recommend stating explicitly that `escalation` (and everything nested under it) is absent entirely when `status` is not one of the stopping outcomes that involve a Human Decision Package in the first place, rather than leaving the field's behavior in the non-triggered case implicit.

**Clean perspectives:** Security (`[SEC-1]` confirmed resolved — the live-vs-persisted split correctly addresses it; no new findings), SRE (`[SRE-1]` confirmed resolved as the side effect already credited in Round 1; no new findings), Lead Developer (`[LD-2]` confirmed resolved — §3.6's procedure is concrete and directly implementable; no new findings beyond `[DBA-3]` above, which is filed under DBA since it's fundamentally an enum-completeness contradiction).

### Contradictions Requiring Arbitration

None.

## Address Summary — Round 2

**Date:** 2026-10-07

### Addressed
- `[DBA-3]` — Corrected Key Decision 5 to state 12 total values (11 stopping outcomes + the one active `in_progress` state), explicit about `in_progress` being categorically distinct from an "outcome" per SPEC §18's own framing.

### Not Addressed
- `[DBA-4]` — **accepted** — a real completeness nit (what `capture_status`/`decision_package` presence means in the non-triggered case), but it requires a judgment call rather than being a clear factual error, so per the default rule it's accepted as-is rather than addressed this round; will be stated explicitly in `operate.md`'s own text at implementation time (absent entirely when no Human Decision Package was ever produced).

## Design Review — Round 4

**Date:** 2026-10-07
**Status:** Changes requested — 0 blocking, 2 non-blocking

This round re-examines only what the discovery-driven revision touched: §3.2/§3.3/§8 (the `security-review.md` fix) and §3.4/§3.5/§4 plus ADR-0002 (the schema additions). Confirmed resolved on substance: `[SEC-2]` (the `findings_status` content-check correctly fires regardless of whether the risk-acceptance entry is new or carried forward — a deliberately conservative choice given the data has no per-entry round tracking) and `[SEC-3]` (the resumption-containment rule is now stated directly in §3.4, not deferred to prose).

**Options Analysis, SRE:** no new findings — domain unaffected by this revision (neither the Design Options/Recommended Option nor any SRE-domain section from prior rounds was touched).

### DBA Findings

- [DBA-6] **non-blocking** — `escalation.reason`'s 5-value enum has no value for §3.6 Stage B.2's "autonomous correction loop the policy does not permit" case — likely one of the *more* common stopping reasons in practice (any project with conservative `operator.allow.*` defaults hits this constantly), and it currently falls under `other`, which undermines the Human Escalation Reason metric (SPEC §34) this enum exists to support if `other` ends up the dominant bucket. Separately, the new §3.2 special case (an `accepted_risk` entry forcing a stop) never states which `escalation.reason` value it sets — `security_policy_exception` is the obvious fit, but it's not cross-referenced. Recommend: add a sixth value (e.g. `policy_denied_autonomy`) for Stage B.2, and state explicitly, next to each Stage B/C branch in §3.6, which `escalation.reason` value it produces — the same traceability §3.6 already gives for `status` values.

### Lead Developer Findings

- [LD-3] **non-blocking** — SEC-0001 verified the single-combined-pause pattern against exactly four commands (`security-review`, `execute-release`, `promote-to-environment`, `execute-tests`) and found `security-review.md` to be the outlier. That's a real, verified fix for the instance found — but it leaves open whether any of the other ~46 commands with a Handoff confirmation step share the same shape (a draft that can carry a formal-gate-worthy disposition behind one undifferentiated pause). Recommend a PLAN/TEST-PLAN task auditing the remaining commands for this pattern rather than assuming `security-review.md` is the only instance; not blocking, since nothing claims otherwise today.

**Clean perspectives:** Security (`[SEC-2]`/`[SEC-3]` confirmed resolved on substance; no new findings).

### Contradictions Requiring Arbitration

None.

## Discovery — 2026-10-07

**Discovered by:** Eric Pizon — Architect (via Security and SRE review findings on SEC-0001/OBS-0001)
**Stage triggered:** design
**Generated with:** claude-sonnet-5

**What was discovered:**

`SEC-0001`'s security review (Stage 4b) found that `security-review.md`'s single, undifferentiated Handoff pause gives `operate.md` no mechanically-distinguishable sub-step to recognize an `accepted_risk` disposition, contradicting this DESIGN's own §3.3/ADR-0003 claim that formal-gate/routine-confirmation classification happens "per Handoff step" for every command — verified directly against `security-review.md`'s actual text, which has exactly one combined confirmation. Separately, `SEC-0001` found the `OPRUN-NNNN` resumption read-path lacks an explicit restatement of the data-not-instruction containment rule (`[SEC-3]`, following up on Design Review Round 1's `[SEC-2]`, which had been deferred to informal implementation-time prose). `OBS-0001` (Stage 4a) independently found that computing SPEC §34's required Operator metrics needs four fields never specified in `OPRUN-NNNN`'s schema (`history[].actor`, `oprun_lookup.result`, `status_resolver.ambiguous_cause`, `recovery_event`), plus formalizing `escalation.reason` as a literal enum instead of free text.

**Why this invalidates the DESIGN:**

§3.2 (Autonomy Overlay Rule) needs an explicit check against a draft's `findings_status` map before self-answering `security-review.md`'s commit pause. §3.3 (categorization table) needs a note on `security-review.md`'s new sub-step. §3.4 (`OPRUN-NNNN` record) needs the resumption containment rule stated directly, plus the `oprun_lookup.result`/`recovery_event` logging points. §4 (`OPRUN-NNNN` schema) needs the four new fields and the `escalation.reason` enum. §8 (Impact on Existing Systems) incorrectly lists `security-review.md` under "Commands requiring no modification" and needs `status.md` noted for the `status_resolver.ambiguous_cause` field.

**Blast radius:**
- OBS-0001: affected — drafted, uncommitted; its instrumentation tasks reference fields that must now actually exist in this DESIGN for the OBS plan to be accurate against it. Will be re-confirmed once this revision lands, not redrafted from scratch.
- SEC-0001: affected — drafted, uncommitted; `findings_status` for `SEC-2`/`SEC-3` should move toward `mitigated` once this revision lands — a follow-up re-assessment, not performed by this command.
- PLAN-0001: not affected — does not exist yet.
- TEST-0001: not affected — does not exist yet.
- M1 in_progress: not affected — no milestones exist yet; this is the earliest possible point to catch this.
- Open PRs: none found.

**Status change:** DESIGN-0001 `approved` → `revision_requested`

## Design Review — Round 3

**Date:** 2026-10-07
**Status:** Changes requested — 0 blocking, 1 non-blocking

**Options Analysis, Security, SRE, Lead Developer:** no new findings — domain unaffected by Round 2's address (only Key Decision 5's text changed; none of these perspectives' own sections were touched).

### DBA Findings

- [DBA-5] **non-blocking** — `[DBA-3]`'s fix to Key Decision 5 is correct, but §3.6's opening sentence wasn't updated to match: it still reads "`OPRUN-NNNN.status`'s eleven," describing 11 as the field's total value count — now stale against Key Decision 5's corrected "12 values total (11 stopping + 1 active)." The underlying procedure in §3.6 is unaffected and still correct (it was already specifically about mapping to the 11 *stopping* outcomes, never claiming to cover `in_progress`), so this is a wording-only residual, not a functional gap — but worth tightening so a reader of §3.6 alone doesn't get the same miscount `[DBA-3]` just corrected elsewhere.

**Clean perspectives:** Options Analysis, Security, SRE, Lead Developer.

### Contradictions Requiring Arbitration

None.

## Address Summary — Round 3

**Date:** 2026-10-07

### Addressed
- `[DBA-5]` — Fixed directly, per the rule allowing a clear factual correction with no judgment involved even though non-blocking: §3.6's opening sentence now says "the 11 stopping outcomes among `OPRUN-NNNN.status`'s 12 total values" instead of the stale "eleven," matching Key Decision 5's v3 correction.

### Not Addressed
*(none — the one finding from Round 3 is listed above.)*
