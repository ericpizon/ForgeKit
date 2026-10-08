# sdlc/operator-runs/

Operator run records produced by `/operate` (DESIGN-0001 §3.4, §4).

## Contents

Each file in this directory is one `OPRUN-NNNN` — a plain ForgeKit artifact tracking a single `/operate` run against one work unit:

```
OPRUN-NNNN.md    # one run record per work_unit (SPEC-NNNN or THREAD-NNNN)
```

## Lifecycle

1. `/operate <SPEC-NNNN|THREAD-NNNN>` scans this directory for an existing record matching the target `work_unit` with a **non-terminal** `status` before creating anything new — an existing non-terminal run is always resumed, never duplicated.
2. If none is found, `/operate` creates a new `OPRUN-NNNN`, `status: in_progress`.
3. Every Command Execution Cycle step (resolve → determine → verify policy → invoke → verify outcome → re-resolve) appends an entry to the record's `history:` list and refreshes `updated_at`.
4. The run ends in exactly one of 11 stopping outcomes (`complete`, `human_decision_required`, `external_action_required`, `blocked_environment`, `blocked_dependency`, `retryable_error`, `terminal_error`, `ambiguous_state`, `not_found`, `already_complete`, `action_limit_reached`) — see DESIGN-0001 §3.6 for the resolution procedure and §4 for the full schema.

**Terminal vs. non-terminal `status`** (this is what step 1 above checks):
- **Terminal** (run finished; a fresh `/operate` invocation on the same `work_unit` starts a *new* record): `complete`, `terminal_error`, `not_found`, `already_complete`.
- **Non-terminal** (run paused; a fresh `/operate` invocation on the same `work_unit` resumes this record): `in_progress`, `human_decision_required`, `external_action_required`, `blocked_environment`, `blocked_dependency`, `retryable_error`, `ambiguous_state`, `action_limit_reached`.

## Source of truth

Per SPEC-0001 §6: when an `OPRUN-NNNN` record's own understanding of state ever conflicts with the authoritative artifacts it was derived from (SPEC, DESIGN, PLAN, etc.), the artifacts win. `OPRUN-NNNN` is a convenience and history record — it is never itself a second source of truth, and a lost, deleted, or corrupted run record never grants permission a fresh policy check wouldn't independently grant.

## Do not edit manually

Run records are written and updated by `/operate` itself. Manual edits can corrupt the `history:`/`last_checkpoint`/`escalation` structure that resumption and the metrics in SPEC-0001 §34 depend on.
