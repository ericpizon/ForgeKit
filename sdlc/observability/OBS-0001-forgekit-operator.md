---
id: OBS-0001
type: observability-plan
schema_version: 1
validation_status: not_yet_validated
target_design_version: 6
generation:
  command: /design-observability
  command_version: "5"
  model: "claude-sonnet-5"
  generated_at: "2026-10-07"
  session_id: null
---

# OBS-0001 — ForgeKit Operator

## Operationally Critical Paths

No running service exists here — `/operate` is a human-attended, single-session, AI-interpreted markdown command (SPEC-0001 §48), not production traffic. "Operationally critical" is reframed the same way `docs/design/OBS-0001` reframed it for external-work-intake: the paths whose *silent* failure would be hardest for a human to notice, because nothing forces anyone to look — except here the blast radius is larger, since Operator's whole premise is reducing exactly the per-step human checkpointing that would normally catch these failures.

1. **The gate-to-outcome resolution procedure (DESIGN-0001 §3.6)** — if this classification logic is subtly wrong, Operator could misreport a stopping condition (e.g. record `complete` when work is actually `blocked`), and nothing else in the system would catch it — the OPRUN-NNNN record itself is the only account of what happened.
2. **OPRUN-NNNN lookup-before-create (§3.4, resolves `[DBA-1]`)** — if this scan silently fails to find an existing non-terminal run, duplicate run records accumulate for the same `work_unit`, defeating AC-11's idempotency guarantee for the one artifact type Operator itself owns.
3. **The Autonomy Overlay Rule's self-answer vs. human-confirmed distinction (§3.2, ADR-0001/ADR-0003)** — if Operator doesn't durably record *which* confirmations it self-answered versus which a human actually gave, there is no way to later audit how much autonomy was actually exercised — undermining the governance premise the whole feature depends on (SPEC §34 Automation Coverage, Gate Compliance).
4. **Compliance-gate decline handling (§4, resolves `[SEC-1]`)** — if the persisted copy isn't correctly redacted on decline, sensitive content leaks into git history permanently. This is the one path where an observability gap and a security incident are the same event.
5. **Cross-repo dependency deadlock detection (§24/AC-17)** — if Operator fails to recognize a cycle and instead spins, nothing signals this except `actions_taken` quietly climbing toward `max_actions_per_run` — by the time the cap fires, the run has already burned its whole budget on a condition a human could have named immediately.

## Confirm the Instrumentation Mechanism

Per ARCH.md ("What this repo actually is") and this project's own `stack.observability: none`, ForgeKit has no metrics library of any kind. All signals below are **structured log-line emissions** — concretely, fields written into `OPRUN-NNNN` frontmatter/body by `operate.md`, or into `/status`'s `resolved_state` output by `status.md` — never a counter/histogram/gauge API call. "Dashboard" below means a query pattern over committed `sdlc/operator-runs/*.md` files, not a UI. This mirrors `docs/design/OBS-0001`'s precedent exactly, and is consistent with DESIGN-0001 §11's own framing: "the frontmatter fields... are the source for every metric in SPEC §34 — computed by reading OPRUN-NNNN files, never duplicated elsewhere."

DESIGN-0001 §4/§11 already names most of the fields these signals read (`status`, `actions_taken`, `review_loop_counts`, `escalation`). Four signals below (`S2`, `S7`, `S8`, `S9`) are new fields this plan adds because DESIGN specified *that* a mechanism exists (the lookup scan, the ambiguous fallback, the recovery detection) without specifying that its *outcome* gets logged anywhere durable — without a log field, the mechanism's own correctness is unverifiable after the fact. These are flagged explicitly in the Instrumentation Tasks below and in this report's closing note, since they extend OPRUN-NNNN's schema beyond DESIGN-0001 §4's literal text.

## Key Signals

| Signal (log field) | Values | Instrumentation point | Failure mode detected |
|---|---|---|---|
| **S1** `OPRUN-NNNN.status` | 12-value enum (§4 Key Decision 5) | `operate.md`, at each Stage A/B/C stopping point of the gate-to-outcome procedure (§3.6) | The one field that distinguishes genuine completion from every blocked/ambiguous/erroring condition — a wrong or missing write here is indistinguishable from success until a human manually re-derives state |
| **S2** `history[].actor` *(new)* | `operator_autonomous` \| `human_confirmed` | `operate.md`, appended to the `history:` entry for every Command Execution Cycle step that passed through a Handoff pause (§3.2 Autonomy Overlay Rule) | Without this, Automation Coverage and Gate Compliance (SPEC §34) cannot be computed — there is no record of whether policy actually reduced human involvement or Operator merely mirrored default confirmations |
| **S3** `escalation.reason` | `non_convergence` \| `test_classification` \| `security_policy_exception` \| `dependency_cycle` \| `other` | `operate.md`, Stage B `human_decision_required` branch (§3.6), set before the Human Decision Package is rendered | SPEC §34's Human Escalation Reason metric needs a cause, not just the fact of a stop; without it every escalation looks identical regardless of why |
| **S4** `review_loop_counts.<ARTIFACT-NNNN>.{round,count}` | existing field (§4) | `operate.md`, Stage B.1 autonomous correction loop (§3.6), incremented each time Operator invokes an `address-*-review` command on its own behalf | Drives Adversarial Convergence and Non-Convergence Rate (§34); without per-round counts, oscillation and genuine linear progress look the same |
| **S5** `actions_taken` vs. `operator.max_actions_per_run` | existing field (§4) | `operate.md`, Stage C pre-check (§3.6) — incremented *before* each attempted action, never after | The autonomy-containment NFR (SPEC §47) is only real if this is checked pre-action; logging it post-action would let one over-the-cap action slip through unrecorded |
| **S6** `escalation.decision_package.capture_status` | `captured` \| `redacted_on_decline` (§4, resolves `[SEC-1]`) | `operate.md`'s Human Decision Package rendering step, at the compliance-gate scan point (reusing `external-work-intake.md`'s mechanism per Key Decision 8) | Directly verifies the live-vs-persisted split works — a miscoded redaction would otherwise ship sensitive content into git history with no signal until someone reads the raw diff |
| **S7** `oprun_lookup_result` *(new)* | `resumed_existing` \| `created_new` | `operate.md`, the lookup-before-create step (§3.4, resolves `[DBA-1]`), logged into the first `history:` entry of that invocation | Makes AC-11's idempotency guarantee for `OPRUN-NNNN` itself auditable — without it, a duplicate-creation bug is invisible until someone manually diffs `sdlc/operator-runs/` |
| **S8** `resolved_state.ambiguous_cause` *(new)* | `missing_structured_field` \| `unrecognized_state_combination` | `status.md`'s `resolved_state` computation, written whenever `gate` is set to `ambiguous` (resolves `[DBA-2]`) | Distinguishes the expected, self-healing cost of a pre-existing artifact lacking a new field from a genuine resolver gap — without this split, Operator Failure Rate (§34) can't separate "working as designed" from "resolver bug" |
| **S9** `recovery_event` *(new)* | `interrupted_implementation_detected` | `operate.md`'s resume path, logged (by the resuming invocation) immediately upon discovering `implementation_execution.state: started` with no `completed_at` for the targeted milestone, before invoking the recovery path (SPEC §12/§37) | Drives Execution Recovery (§34) and verifies Safety Invariant #16 (never send partial work to review) is actually enforced in practice, not just in theory |
| **S10** `updated_at` | existing field (§4), no new write needed | Already refreshed at every Command Execution Cycle step | The human-attended model's own staleness signal (Round 1 `[SRE-2]`, accepted as-is) — this plan's job is only to make it queryable, not to add a new write |

**Mapping to SPEC-0001 §34 Metrics:** Automation Coverage ← S2 ratio; Human Intervention Count ← count of S1 `human_decision_required` runs; Human Escalation Reason ← S3; Adversarial Convergence / Non-Convergence Rate ← S4; Gate Compliance ← cross-check of S1/S2 (no `resolved_state.gate: blocked` run shows `actor: operator_autonomous` proceeding past it); Rework ← read directly off the `history:` trace's sequence of `resolved_state.stage` values, no new field needed; Execution Recovery ← S7 + S9; Operator Failure Rate ← S1 (`terminal_error`/`retryable_error`/`action_limit_reached`) combined with S8 (`unrecognized_state_combination` specifically, not the expected `missing_structured_field` case).

## Latency / Timing Considerations

No request/response path exists, so there is no latency distribution to protect from cold starts, connection pooling, or external-call skew (Step 4 is structurally N/A here, the same way it was for `docs/design/OBS-0001`). The one timing-adjacent risk is a *stuck* run — not slow, but silently not progressing. That risk is already covered by **S10** (`updated_at` staleness) and Round 1's `[SRE-2]` finding, which accepted this as the human's own signal to notice under SPEC §48's human-attended execution model rather than requiring a new mechanism.

## SLOs / Thresholds

No empirical data exists yet — Operator v1 is unimplemented. Per the standard pre-baseline process: once `/operate` ships, collect, for a representative initial period (recommend: the first full milestone sequence of this feature's own PLAN-0001, run against itself per `dogfooding`, plus any early adopting-project runs):
- the distribution of `OPRUN-NNNN.status` values across completed runs,
- the `history[].actor` autonomous-vs-human ratio,
- `review_loop_counts` distributions per artifact type,
- `escalation.reason` frequency,
- how often `actions_taken` approaches `max_actions_per_run` without hitting it.

Only after that baseline period should specific numeric thresholds (e.g., "more than N% `redacted_on_decline` is worth investigating," or "more than N `retryable_error` runs per week") be committed. Until then, the Alerts below use qualitative conditions ("recurring," "elevated") rather than fixed numbers — informational references, not alert-worthy on their own, per Step 5.

## Dashboards

No panel-based dashboard — no metrics backend exists to build one in. Each "dashboard" is a documented query pattern over `sdlc/operator-runs/*.md` frontmatter (and, for S8, `/status` invocation output where it's surfaced to a human):

- **Run outcomes** — aggregate `status` (S1) across all `OPRUN-NNNN`, grouped by value, over time. Surfaces the overall health/failure-category mix (Operator Failure Rate).
- **Autonomy coverage** — ratio of `history[].actor = operator_autonomous` to total Handoff-bearing steps (S2), across all runs. Surfaces Automation Coverage directly.
- **Escalation causes** — aggregate `escalation.reason` (S3) for every `status: human_decision_required` run. Surfaces Human Escalation Reason.
- **Convergence health** — `review_loop_counts` (S4) per artifact, flagged whenever `count` approaches the project's `operator.max_review_rounds`. Surfaces Adversarial Convergence / Non-Convergence Rate before a run actually stops.
- **Resolver gap tracker** — `resolved_state.ambiguous_cause` (S8), filtered to `unrecognized_state_combination` only (excluding the expected `missing_structured_field` case). Any non-zero count here is itself worth a human looking, independent of volume.
- **Compliance-gate pattern** — `escalation.decision_package.capture_status` (S6) distribution across all escalations carrying a decision package. Mirrors `docs/design/OBS-0001`'s existing `compliance_gate.outcome` dashboard for the analogous mechanism in external-work-intake.

## Alerts

None of these are 3am pager conditions — Operator is human-attended by construction (SPEC §48), so "alert" means an informational or warning-level flag a human notices on their own schedule, not a page.

| Alert | Condition | Severity | Likely cause | Runbook summary |
|---|---|---|---|---|
| Compliance-gate redaction elevated | `capture_status: redacted_on_decline` (S6) recurring across multiple escalations for the same project | Informational | Either real compliance-sensitive content correctly caught repeatedly, or the gate's `compliance.frameworks` match is miscalibrated and over-triggering | Review the flagged `OPRUN-NNNN` decision packages directly; check against the project's `compliance.frameworks` configuration; if miscalibrated, this is an `external-work-intake.md` gate-tuning issue (shared mechanism), not an Operator-specific bug |
| Recurring Operator-side failure | `status` (S1) in `{terminal_error, retryable_error, action_limit_reached}` recurring across multiple runs against the same `work_unit` | Warning | A structurally malformed artifact, a persistent `git`/`gh` problem, or a state-resolution defect producing more eligible actions than expected | Inspect `last_attempted_action`/`last_checkpoint` on the most recent `OPRUN-NNNN`; for `action_limit_reached` specifically, distinguish a miscalibrated `max_actions_per_run` from a genuine runaway — do not raise the cap reflexively |
| Resolver gap (not a legacy-field cost) | `resolved_state.ambiguous_cause: unrecognized_state_combination` (S8) fires even once | Warning | `/status`'s lifecycle coverage has a genuine gap — a real state combination it was never taught to resolve | File a BUG against `status.md`; the triggering artifact/state combination should be visible in the `/status` invocation output or the OPRUN escalation context |
| Dependency deadlock detected | `escalation.reason: dependency_cycle` (S3) fires | Warning | Two or more THREAD participants are mutually blocked (AC-17/§24) | Review the decision package's named cycle participants and dependencies; a human must break the cycle manually — Operator will never do this itself (Safety Invariant #17) |
| Non-convergence reached | `review_loop_counts.<ARTIFACT>.count` (S4) reaches `operator.max_review_rounds` with `latest_round.verdict` still `needs_revision` | Warning | A recurring/oscillating blocking finding, or genuinely incompatible reviewer requirements that can't converge autonomously | Review `recurs_from` lineage across rounds in the artifact itself; decide to override policy, revise scope, or resolve the incompatibility directly — Operator stops and cannot resolve this on its own (AC-08) |

## Instrumentation Tasks

Tagged `observability` for `/create-plan`. Each names the file, the step it modifies (by DESIGN-0001 section reference, since these are markdown instruction steps rather than compiled functions), and the exact field.

1. **`operate.md`** — at the Command Execution Cycle step that appends to `OPRUN-NNNN`'s `history:` list after invoking any sub-command (§3.2 Autonomy Overlay Rule), add an `actor: operator_autonomous | human_confirmed` field to that entry, set according to whether the preceding Handoff pause (if any) was self-answered under an `operator.allow.*` policy key or required/received an explicit human confirmation. *(New field — closes S2.)*
2. **`operate.md`** — at Stage B's `human_decision_required` branch (§3.6), before rendering the Human Decision Package, set `escalation.reason` to exactly one of `non_convergence | test_classification | security_policy_exception | dependency_cycle | other`, matching the specific cause already required to be recorded in prose. *(Formalizes an enum for S3.)*
3. **`operate.md`** — at the lookup-before-create step (§3.4, resolves `[DBA-1]`), log `oprun_lookup_result: resumed_existing | created_new` into the first `history:` entry written for that invocation. *(New field — closes S7.)*
4. **`status.md`** — in the `resolved_state` computation (resolves `[DBA-2]`), whenever a required structured field is missing on a pre-existing artifact (forcing `gate: ambiguous`), set `resolved_state.ambiguous_cause: missing_structured_field`; for every other case that reaches `gate: ambiguous`, set `unrecognized_state_combination`. *(New field — closes S8.)*
5. **`operate.md`** — at the Human Decision Package rendering step, immediately before persisting any quoted artifact content into `OPRUN-NNNN`'s body, invoke the same compliance-gate scan `external-work-intake.md` already uses (Key Decision 8) and set `escalation.decision_package.capture_status: captured | redacted_on_decline` accordingly, replacing the sensitive portion with the stated placeholder on decline. *(Makes DESIGN-0001 §4's already-specified field concrete at an exact step.)*
6. **`operate.md`** — on the resume path, immediately upon re-resolving state and discovering `implementation_execution.state: started` with no `completed_at` for the milestone about to be acted on (SPEC §12/§37), append `recovery_event: interrupted_implementation_detected` to `history:` before invoking the implementation recovery path, never after. *(New field — closes S9.)*
7. **`operate.md`** — ensure every Command Execution Cycle step (§15: resolve → determine → verify policy → invoke → verify outcome → re-resolve) refreshes `OPRUN-NNNN.updated_at`, not only at run start and stop, so staleness (S10) reflects true last-activity time.

## Validation Checklist

| Signal | Validation step |
|---|---|
| S1 `status` | Fixture-drive each of the 12 enum values (per §3.6's own trigger criteria) and confirm the committed `OPRUN-NNNN.status` matches exactly, with no other value appearing for a different trigger. |
| S2 `history[].actor` | Run one scenario with the relevant `operator.allow.*` key `true` and one with it `false` (or absent) against the same routine draft-commit pause; confirm `actor: operator_autonomous` only in the first case and `human_confirmed` in the second. |
| S3 `escalation.reason` | Trigger each of the four named causes (a recurring DESIGN finding, a QE test-classification wait, a SEC `accepted_risk` scenario, a two-repo dependency-cycle fixture) and confirm each produces its own distinct value. |
| S4 `review_loop_counts` | Run a fixture review that recurs the same finding up to `max_review_rounds`; confirm the count increments once per round and that reaching the limit produces `status: human_decision_required` with `escalation.reason: non_convergence`. |
| S5 `actions_taken` | Set `max_actions_per_run` to a small test value in a fixture `project.yml`; run a scenario with more eligible actions than the cap; confirm the stop occurs at exactly the configured count and the pre-check fires before, not after, the over-the-cap action. |
| S6 `capture_status` | Trigger a compliance-gate scenario under a fixture project with `compliance.frameworks` set non-`none`; confirm the live, in-session decision package shows raw content regardless of outcome; confirm declining persists the placeholder with `redacted_on_decline`, and accepting persists the real content with `captured`. |
| S7 `oprun_lookup_result` | Interrupt a fixture run mid-sequence; re-invoke `/operate` against the same `work_unit`; confirm `resumed_existing` and no duplicate `OPRUN-NNNN` file; separately confirm a fresh `work_unit` produces `created_new`. |
| S8 `resolved_state.ambiguous_cause` | Run `/status` against a fixture artifact missing a required structured field; confirm `missing_structured_field`; separately construct a fixture with a genuinely unrecognized state combination and confirm `unrecognized_state_combination`. |
| S9 `recovery_event` | Interrupt a fixture `/implement-plan` run after `implementation_execution.state: started` but before completion; resume via `/operate`; confirm the event is logged before the recovery path runs and that implementation review is never invoked directly on the interrupted work. |
| S10 `updated_at` | Run a multi-step fixture sequence; confirm `updated_at` changes at every Command Execution Cycle step, not only at run start/stop. |
