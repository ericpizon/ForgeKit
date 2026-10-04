# sdlc/scenarios/

Cross-repo E2E scenario definitions — black-box specifications of behavior that spans multiple participant repos, needing two or more services actually deployed and running together. Only present in **coordination repos** (`project.yml`'s `repo_role: coordination`).

## Contents

```
THREAD-NNNN-E2E-NN.yml    # one file per cross-repo scenario
```

## What lives here — and what doesn't

A scenario file is the *specification*: entrypoint, expected externally-observable outcome, which XDEPs and acceptance criteria it covers, and a reference to wherever its executable implementation lives. It does not contain executable test code, and never will — the executable implementation belongs in a dedicated test-infrastructure repo (`org/system-tests`, or wherever the org's test platform lives), even for the simplest scenario. A coordination repo holding any executable test code stops being a governance/control-plane repo and starts being a test-application repo wearing a control-plane repo's name — that's not a case-by-case judgment call, it's a hard boundary.

This directory is conditional: a thread whose cross-repo dependencies are all fully covered by contract-level verification (schema compatibility, consumer-deserialization checks — no live deployment required) doesn't need any scenario files at all. Not every thread needing coordination needs this directory populated.

## Ownership and quarantine

Every scenario names `owners`/`triage_owner` — a centrally-shared scenario with no owner rots the way any unowned shared test suite does. A scenario required for release can be `quarantined`, but only as an explicit accepted risk (named approver, revisit condition) — the same convention already used for accepted-risk SEC findings, never a silent exclusion from the gate.

## Execution is bound, not implemented here

Each scenario references an execution binding — what external system runs it, when it's required (release / nightly / on a boundary PR), and the result contract (`PASS` / `FAIL` / `BLOCKED_ENVIRONMENT` / `ERROR`). ForgeKit declares the requirement and consumes the evidence that comes back; it does not provision environments, schedule runs, or execute the scenario itself. See [`xdep-schema.md`](../../.claude/reference/xdep-schema.md) for the full scenario schema and the evidence eligibility rule.
