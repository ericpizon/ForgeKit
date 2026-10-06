---
id: OBS-0001
type: observability-plan
schema_version: 1
validation_status: not_yet_validated
target_design_version: 3
generation:
  command: /design-observability
  command_version: "4"
  model: "claude-sonnet-5"
  generated_at: "2026-10-05"
  session_id: null
---

# OBS-0001 — External Work Intake and Durable Coordination Context

## Operationally Critical Paths

No running service exists here — every capability in DESIGN-0001 is an authoring-time or governance-time command, human-invoked, not production traffic. "Operationally critical" is reframed accordingly: the paths whose *silent* failure would be hardest for a team to notice, because nothing forces a human to look.

1. **G1's live-access check (D5)** — if a team's ADO/Confluence connector is subtly misconfigured, every intake attempt could silently fall back to copy-paste (or every write-back silently fall back to a manual summary) with no one ever noticing the connector never actually worked.
2. **`/promote-to-thread`'s resumability (ADR-0001)** — if an interrupted promotion is resumed, nothing currently signals *that it was a resume* rather than a fresh run; a team debugging a confusing coordination-repo state would have no record of which runs were interrupted.
3. **The compliance confirmation gate (D6)** — if "declined" happens far more often than expected, a team would want to notice a pattern, not just experience it one SPEC at a time.

## Confirm the Instrumentation Mechanism

ForgeKit's own repo has no `ARCH.md` and no `stack.observability` field — consistent with this whole SPEC/DESIGN's structural deviation. No metrics library exists or is implied anywhere in this feature. All signals below are **structured log-line emissions**, not counter/histogram/gauge API calls — every later reference to "a signal" in this document means a named log field, queried from wherever a team's logs land, not a metrics backend.

## Key Signals

| Signal (log field) | Values | Instrumentation point | Failure mode detected |
|---|---|---|---|
| `source_work.intake_result` | `fetched` \| `pasted` \| `fetch_failed` | `/create-spec`'s intake step (AC1) | A connector that's configured but never actually succeeds — every intake silently degrading to paste when it shouldn't be |
| `source_work.write_back_result` | `updated` \| `copy_paste_fallback` \| `write_failed` | `/approve-spec`'s write-back step (AC4) | Same class of silent degradation on the write side; `write_failed` specifically surfaces D9's failure-mode rule in a way a team can actually query later |
| `compliance_gate.outcome` | `captured` \| `declined` \| `not_applicable` | `/create-spec`'s capture step (AC2/D6) | An unexpectedly high `declined` rate — either real compliance exposure being correctly caught, or a gate that's miscalibrated and blocking normal work |
| `promote_to_thread.outcome` | `completed` \| `resumed_and_completed` \| `interrupted` | `/promote-to-thread` — `completed`/`resumed_and_completed` at its final step (after `pending_promotion` is replaced by `status: promoted`); `interrupted` logged *by the resuming invocation itself*, upon discovering `pending_promotion` already set from a prior incomplete run, before that resuming invocation proceeds — never by the original interrupted run, which by construction never reaches its own last step to log anything **(amended 2026-10-05, per `/create-test-plan`'s Round 1 `[SRE-1]` finding — the original wording described `interrupted` as emitted at the same final step as the other two values, which is unreachable for a run that was actually interrupted)** | Makes resumption visible after the fact — without this, a resumed run is indistinguishable from a clean one in the historical record |

## SLOs / Thresholds

No empirical usage data exists yet (this is a new capability, not yet implemented). Per the standard process: collect each signal's actual distribution for a representative period before committing to any threshold. Informal baseline only, not alert-worthy: `source_work.intake_result = fetch_failed` and `source_work.write_back_result = write_failed` are expected to be rare — any sustained non-trivial rate is worth a human looking, but nothing here pages anyone.

## Dashboards

No panel-based dashboard — there's no metrics backend to build one in. "Dashboard" here means: a team greps or queries these four structured log fields from wherever their own log aggregation already lands (the same convention ForgeKit's own structured console logging already uses elsewhere). Documented as a query pattern, not a UI:
- `source_work.intake_result` and `source_work.write_back_result`, grouped by value, over time — surfaces silent connector degradation.
- `compliance_gate.outcome`, grouped by value — surfaces gate calibration.
- `promote_to_thread.outcome`, filtered to `interrupted` — surfaces how often promotion actually gets interrupted in practice (informs whether ADR-0001's resumability design is pulling its weight).

## Alerts

No alert here is a 3am pager condition — these are all human-invoked, non-production-traffic paths. One informational flag worth naming explicitly, since it's the one case where silence itself is the risk:

| Alert | Condition | Severity | Likely cause | Runbook summary |
|---|---|---|---|---|
| Connector silently never succeeding | `source_work.intake_result = fetch_failed` (or `write_back_result = write_failed`) on every occurrence over some observation window, with zero `fetched`/`updated` outcomes in between | Informational | A configured MCP server or CLI tool that reports as "available" but never actually completes a request — e.g. expired credentials | Check the connector's own configuration and credentials directly; ForgeKit itself has no diagnostic for this beyond the log field, per D11/D12's explicit non-ownership of the connector |

## Instrumentation Tasks

Tagged `observability` for `/create-plan`.

1. Add `source_work.intake_result` structured log emission in `/create-spec`'s intake step, at the point AC1's fetch-or-paste branch resolves.
2. Add `source_work.write_back_result` structured log emission in `/approve-spec`'s write-back step, at the point AC4's write-or-fallback branch resolves.
3. Add `compliance_gate.outcome` structured log emission in `/create-spec`'s capture step, at the point AC2/D6's confirmation branch resolves (including `not_applicable` when fetched directly or when `compliance.frameworks` is `[none]`).
4. Add `promote_to_thread.outcome` structured log emission at two distinct points in `/promote-to-thread`: (a) its own final step, logging `completed` or `resumed_and_completed`; (b) immediately upon discovering an existing `pending_promotion` from a prior incomplete run, logging `interrupted` for that prior attempt before proceeding with the resume.

## Validation Checklist

| Signal | Validation step |
|---|---|
| `source_work.intake_result` | Trigger all three values manually (fetch with access, fetch without access, a forced fetch failure); confirm each distinct value appears and no other value fires for the wrong trigger. |
| `source_work.write_back_result` | Same approach on the write side — confirm `updated`, `copy_paste_fallback`, and `write_failed` each appear only for their actual trigger. |
| `compliance_gate.outcome` | Confirm `captured` appears for a `compliance.frameworks: [none]` paste, `declined` appears when the user declines confirmation under a non-`[none]` framework, and `not_applicable` appears for a direct fetch. |
| `promote_to_thread.outcome` | Run `/promote-to-thread` to completion once cleanly (confirm `completed`); interrupt a run, then re-invoke it (confirm `interrupted` is logged by the *resuming* invocation upon discovering `pending_promotion`, and `resumed_and_completed` is logged separately at that same invocation's own completion). |
