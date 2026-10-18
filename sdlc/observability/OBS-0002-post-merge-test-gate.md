---
id: OBS-0002
type: observability-plan
schema_version: 1
validation_status: not_yet_validated
target_design_version: 2
generation:
  command: /design-observability
  command_version: "5"
  model: "claude-sonnet-5"
  generated_at: "2026-10-17"
  session_id: null
---

# OBS-0002 — Configurable Post-Merge Test Gate

## Instrumentation Implementation Log

| Task | Milestone | Status | Date |
|---|---|---|---|
| IT-1 — `post_merge_merge_attempt` field, written at `approve-implementation.md`'s merge-verification step | M1 | implemented | 2026-10-18 |

## Operationally Critical Paths

No running service exists here (per ARCH.md's "What this repo actually is") — every path below is a markdown-instruction sequence an AI agent executes, not production traffic. "Operationally critical" means: the step whose silent failure would be hardest for a human to notice, because `post_merge` mode specifically trades away the one checkpoint (`pre_merge`'s own gate) that would normally have caught it.

1. **The merge-verification step in `/approve-implementation` (DESIGN-0002 §3.3/ADR-0004, resolves Round 1 `[LD-1]`)** — if this ever advances `**Implementation status:** pr_merged` without actually confirming the PR's merged state, a milestone could be recorded as complete while nothing ever landed on `main`. This is the single highest-consequence silent-failure point in the whole feature.
2. **`post_merge_test.status` write by `/execute-tests` (§3.4), with its P0-only failure trigger (resolves Round 1 `[SRE-1]`)** — a wrong write here collapses "not yet validated," "validated healthy," and "validated and broken" back into indistinguishable states once the milestone's own `**Implementation status:**` has already reached the terminal `pr_merged` — exactly the gap `AC-08` exists to close.
3. **`stack.qa_trigger_command` infrastructure failure vs. a real post-merge test failure (§3.6)** — if these are ever conflated, a flaky QA trigger (auth expiry, network blip) would silently file a spurious `BUG-NNNN` against working code, or worse, an actual regression could be waved off as "just infra."
4. **OQ-5's per-milestone authority rule (§3.2)** — a milestone that already merged under `post_merge` must keep following the post-merge validation path for itself even if the project's current `pipeline.test_gate` has since switched to `pre_merge`. If `/execute-tests` ever reads the live project setting instead of checking for an existing `post_merge_test` block first, a milestone would silently get routed through the wrong validation path after a mid-project mode switch.

## Confirm the Instrumentation Mechanism

Per ARCH.md and this project's own `stack.observability: none`, ForgeKit has no metrics library. Every signal below is a **structured field written into existing markdown artifacts** — `project.yml`'s own `pipeline.test_gate`, or a field on the milestone's own entry in `PLAN-NNNN` — never a counter/histogram/gauge API call. "Dashboard" means a documented query pattern over committed `sdlc/plans/*.md` files, not a UI. This mirrors `OBS-0001`'s own precedent exactly.

DESIGN-0002 §3.4 already names two of the fields below (`post_merge_test.status`/`updated_at`). One signal (`S4`) is new — DESIGN §3.3 specifies *that* a merge-verification mechanism exists and *what* it does behaviorally (leave the PR open, keep the milestone at `approved`, tell the human what blocked it) without specifying that this outcome gets written anywhere durable. Without a field, the mechanism's own correctness — and any recurring pattern of blocked merges — is unverifiable after the fact. This is flagged explicitly in the Instrumentation Tasks below, since it extends the milestone entry's schema beyond DESIGN-0002 §3.4's literal text, the same way `OBS-0001`'s `S2`/`S7`/`S8`/`S9` extended `OPRUN-NNNN` beyond `DESIGN-0001`'s literal text.

## Key Signals

| Signal (field) | Values | Instrumentation point | Failure mode detected |
|---|---|---|---|
| **S1** `pipeline.test_gate` | `pre_merge` \| `post_merge` | `project.yml`; read live at point of use by `/approve-implementation` and `/execute-tests` (§3.2, resolves `OQ-1`/`OQ-5`) | The one field distinguishing which of the two merge/test sequences a milestone is actually running under — a misread silently routes a milestone through the wrong path entirely |
| **S2** `post_merge_test.status` | `pending` \| `passed` \| `failed` | PLAN-NNNN milestone entry; written `pending` by `/approve-implementation`'s merge step (§3.3) the same moment `pr_merged` is set; written `passed`/`failed` by `/execute-tests` (§3.4, P0-only trigger per resolved `[SRE-1]`) | The only durable record distinguishing "merged, awaiting post-merge validation" / "validated healthy" / "validated and broken," once the milestone's own `**Implementation status:**` has already reached the terminal `pr_merged` value — `AC-08`'s entire reason for existing |
| **S3** `post_merge_test.updated_at` | timestamp | Same write points as S2 (resolves Round 1 `[DBA-1]`) | Without it, a `pending` status is indistinguishable from a stalled, forgotten post-merge validation — no way to tell how long a milestone has sat unvalidated on `main` |
| **S4** `post_merge_merge_attempt` *(new)* | `{blocked: true, blocked_reason: <string>, attempted_at: <timestamp>}` — present only when §3.3's merge-verification finds the PR not actually merged | `approve-implementation.md`'s merge step, written at the exact point §3.3/ADR-0004's "Not merged" branch fires | §3.3 already requires the human be told live what blocked the merge, but nothing durable records that it happened — without this field, a milestone stuck at `approved` under repeated branch-protection blocks looks identical to one nobody has tried merging yet, and a recurring pattern of blocked merges across milestones is invisible after the fact |
| **S5** `stack.qa_trigger_command` infra failure | n/a — no new field | `execute-tests.md`'s post-merge validation path (§3.4/§3.6) | Already required by DESIGN to be "surfaced directly to the user" and never written to `post_merge_test.status`; unlike S4, there is no milestone-state ambiguity to resolve with a new field — this plan names the dashboard/alert pattern for it (below) instead of adding a write |

**Mapping to the feature's own AC/Goal intent:** `G1`/`AC-01`/`AC-05` ← S1; `G3`/`AC-02`/`AC-03`/`AC-08` ← S2/S3; the merge-verification half of `AC-06` (resolves `[LD-1]`) ← S4; the infra-failure-vs-test-failure distinction implicit in `AC-02`/`AC-03` ← S5.

## Latency / Timing Considerations

No request/response path exists — structurally N/A, the same way it was for `OBS-0001`. There is no cold-start or connection-pooling skew to protect any distribution from, because there is no distribution; `S2`/`S3` are discrete state transitions, not a latency measurement.

## SLOs / Thresholds

No empirical data exists yet — `post_merge` mode is unimplemented. Per the standard pre-baseline process: once it ships and at least one project adopts it, collect, for a representative initial period:
- the distribution of `post_merge_test.status` values across milestones that merged under `post_merge`,
- how often `S4`'s `blocked` case fires per milestone (zero times, once, or recurring),
- how often `S5`'s infra-failure case fires.

Only after that baseline should specific numeric thresholds (e.g., "more than N blocked-merge attempts for the same milestone is worth investigating") be committed. Until then, the Alerts below use qualitative conditions ("recurring") rather than fixed numbers — informational references, not alert-worthy on their own.

## Dashboards

No panel-based dashboard — no metrics backend exists. Each "dashboard" is a documented query pattern over `sdlc/plans/*.md` frontmatter/milestone entries:

- **Post-merge outcome mix** — aggregate `post_merge_test.status` (S2) across every milestone that merged under `post_merge`, grouped by value. Surfaces how often post-merge validation actually catches something.
- **Merge-block pattern** — presence and frequency of `post_merge_merge_attempt.blocked` (S4) across milestones. A recurring pattern is a signal that this project's branch-protection configuration is harder to satisfy than `post_merge` mode assumed.
- **Mode usage over time** — `pipeline.test_gate` (S1) value read at each milestone's merge time, useful specifically for auditing the `OQ-5` mid-project-switch case (which milestones ran under which mode).

## Alerts

None of these are 3am pager conditions — every command here is human-attended by construction, so "alert" means an informational or warning-level flag a human notices on their own schedule, not a page.

| Alert | Condition | Severity | Likely cause | Runbook summary |
|---|---|---|---|---|
| Recurring blocked merge | `post_merge_merge_attempt.blocked` (S4) fires more than once for the same milestone across successive re-runs | Warning | A required status check or additional approval that's harder to satisfy than this mode assumed | Review the named `blocked_reason`; resolve the actual blocking requirement on GitHub, or reconsider whether `post_merge` is the right mode for this project's specific branch-protection configuration |
| Post-merge test failure | `post_merge_test.status: failed` (S2) fires | Warning | A genuine regression slipped past pre-merge review — the inherent, accepted residual risk `post_merge` mode trades in | Per `AC-03`/DESIGN §3.5, `/create-bug` or `/escalate-to-incident` already fires at the moment this happens; this alert is the aggregate dashboard-level visibility across milestones, not a new routing path |
| Recurring QA-trigger infra failure | `stack.qa_trigger_command` (S5) fails more than once in a short window, for reasons unrelated to test outcomes | Informational | Flaky QA deploy/trigger infrastructure (auth expiry, network), not a code problem | Investigate `stack.qa_trigger_command`'s own reliability before trusting any of its future pass/fail output; do not let a string of infra failures quietly erode confidence in real `failed` results |

## Instrumentation Tasks

Tagged `observability` for `/create-plan`. Each names the file, the step it modifies (by DESIGN-0002 section reference), and the exact field.

1. **`approve-implementation.md`** — at §3.3's merge-verification step (resolves Round 1 `[LD-1]`), when the check (`gh pr view --json state` for the team path, or the fast-forward attempt for the solo path) finds the merge did **not** complete, write `post_merge_merge_attempt: {blocked: true, blocked_reason: "<the specific status check or approval still missing>", attempted_at: <now>}` to the milestone's PLAN-NNNN entry before leaving `**Implementation status:**` at `approved`. *(New field — closes S4.)* On a subsequent successful retry, no separate cleanup write is required — the milestone's own `post_merge_test: {status: pending, updated_at: <now>}` write (already specified in §3.4) supersedes it the moment the merge actually confirms, consistent with "the git log is the audit trail" already governing `post_merge_test` itself.

## Validation Checklist

| Signal | Validation step |
|---|---|
| S1 `pipeline.test_gate` | Set `pipeline.test_gate: post_merge` in a fixture `project.yml`; run `/approve-implementation` on a fixture milestone; confirm the Handoff takes the post-merge branch (never offers `/create-pr`) and that `/execute-tests`'s pre-flight recognizes the mode. |
| S2 `post_merge_test.status` | Fixture-drive all three values: `pending` at merge, `passed` after a clean post-merge test run, `failed` after a fixture P0 failure; separately confirm a fixture P1-only failure does **not** flip status away from `pending`/`passed`. |
| S3 `post_merge_test.updated_at` | Confirm it is set at both the `pending` write and the terminal `passed`/`failed` write, and that the value changes between the two. |
| S4 `post_merge_merge_attempt` | Run a fixture team-path `/approve-implementation` where the merge check simulates `gh pr merge` failing after `gh pr create` succeeds; confirm `post_merge_merge_attempt.blocked: true` is written with a `blocked_reason`, `**Implementation status:**` stays at `approved`; then simulate the block clearing and re-run, confirming the milestone reaches `pr_merged` with `post_merge_test.status: pending`. |
| S5 `stack.qa_trigger_command` infra failure | Point a fixture `stack.qa_trigger_command` at a command that always exits non-zero for a reason unrelated to test outcomes (e.g., simulated auth failure); confirm `post_merge_test.status` is left unchanged — not set to `failed` — and the failure is surfaced directly to the user instead. |
