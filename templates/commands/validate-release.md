---
id: validate-release
type: command
name: /validate-release
description: SRE + QE command to validate the deployed release in production — derived smoke tests, manual test cases deferred from testing phase, and OBS signal validation
status: active
version: 2
primary_role: sre
---

# /validate-release

## Purpose

Confirm the release is working correctly in production across three tracks: smoke tests (critical paths live), manual test cases deferred from the testing phase, and observability signal validation. Results are recorded in RELEASE-PLAN and TEST-NNNN. A passing validation gates `/generate-release-notes` and the formal feature sign-off.

## Usage

```
/validate-release RELEASE-PLAN-vX.Y.Z TEST-NNNN OBS-NNNN
```

## Required Inputs

- **RELEASE-PLAN-vX.Y.Z** — deployment record (production URL), scope
- **TEST-NNNN** — test cases marked `deferred-to-release`
- **OBS-NNNN** — signal definitions to validate in production

Read the production URL from `RELEASE-PLAN-vX.Y.Z.md`'s Deployment Record section before starting.

---

## Track 1 — Smoke Tests

Verify the production URL is live and the critical user paths from SPEC-NNNN work end-to-end. Derive smoke test steps directly from SPEC-NNNN rather than using a fixed list.

**Deriving smoke tests from SPEC-NNNN:**

1. Read all User Stories in SPEC-NNNN. Identify the 1–2 primary stories — the ones a user would follow to exercise the feature's core value on first contact.
2. For each primary User Story, extract the AC-level happy path: what does the user do, and what do they expect to see? Each discrete action + observable outcome becomes one smoke step.
3. Include one representative negative path per authentication or authorization control — wrong credential, missing token, unauthorized access.
4. **If RELEASE-PLAN-vX.Y.Z's Gating criteria shows "Deployment protection confirmed: pass"** (i.e., this release relies on platform-level access protection or app-level auth to keep the deployment non-public), include a step that verifies it from the outside: an unauthenticated request to the production URL is actually challenged/blocked, not just configured. A gate that was checked off but isn't actually enforced in production is a **critical** validation failure — treat it the same as any other smoke test failure below, and do not let "the RELEASE-PLAN said pass" substitute for observing it.
5. Aim for 5–8 steps total. If more than 8, collapse related sub-steps; smoke tests are sanity checks, not full regression suites.
6. Each step should be independently confirmable in a browser or via `curl` — no step requires the user to interpret logs or check a database.

Present the derived steps as a numbered checklist. Confirm each step with the user before proceeding to the next.

> **Smoke test — run each step against `[production URL]`:**
>
> [N derived steps based on SPEC-NNNN User Stories and ACs]
>
> Confirm each step: PASS / FAIL

Wait for the user to confirm each step before proceeding to Track 2.

If any step fails — stop. See **If Validation Fails** below.

---

## Track 2 — Manual Test Cases

Read TEST-NNNN. Filter to all test cases with `Status: deferred-to-release`. For each one, present it to the user for execution:

> **TC-[ID] — [description]** *(manual, deferred from testing phase)*
>
> Procedure:
> [steps from the test case's `Procedure:` block in TEST-NNNN]
>
> Expected:
> [expected result from the test case]
>
> Result? PASS / FAIL

Wait for the user to confirm each result. Record in TEST-NNNN's Execution Results section.

If no test cases are marked `deferred-to-release`, note "No manual test cases deferred to release phase" and skip this track.

---

## Track 3 — Observability Signal Validation

Adopt the SRE role (`.claude/roles/sre.md`).

Read OBS-NNNN's Key Signals section. For each signal applicable to the deployed milestones, derive a verification step the user can execute in the production environment:

**Deriving OBS validation steps from OBS-NNNN:**

1. Read each signal entry in OBS-NNNN's Key Signals section. Group by signal type: structured logs, metrics (counters, histograms), distributed traces.
2. For each structured log signal: name the log entry's required fields and any field that must NOT appear.
3. For each metric signal: state the metric name, expected label set, and whether it should be incrementing.
4. For each trace/span signal: name the span and its required attributes.
5. Identify which signals require an external telemetry exporter to be configured and note them separately — absence of exporter config is not a failure.
6. Each verification step must be executable via the project's log viewer, a `curl` request, or the user's configured telemetry backend.

Present the derived steps as a numbered checklist:

> **OBS validation — verify each signal in production:**
>
> [N derived steps based on OBS-NNNN Key Signals]
>
> Result? [signal-specific confirmation options]

After all signals are checked, update OBS-NNNN frontmatter:
```yaml
validation_status: validated_at_release
```

---

## Record Validation Results

Update `RELEASE-PLAN-vX.Y.Z.md` Validation Results section:

```markdown
## Validation Results

- Smoke test: PASS — [date]
- Manual tests: N passed, M failed (see TEST-NNNN Execution Results — Production Round)
- OBS signal validation: validated | partial (structured logs confirmed; telemetry export not configured) — [date]
- Validated at: [timestamp]
```

---

## If Validation Fails

### Smoke test failure or P0 manual test failure

Stop immediately — this is a production incident.

> "Critical validation failure in production. Do not proceed to `/generate-release-notes`.
> Document the incident. Once resolved, re-run `/validate-release` before signing off the release."

### P1+ manual test failure

Record the failure, note it as a known gap, and proceed. Document in `/generate-release-notes`.

### OBS signals not visible — exporter not configured

Not a failure. Record as: "Structured logs confirmed in production logs. Metric/trace export requires telemetry exporter to be configured." Proceed.

---

## Handoff

> **Validation complete.**
>
> Smoke tests: PASS
> Manual tests: [N passed / no deferred cases]
> OBS validation: [result]
>
> Run `/generate-release-notes RELEASE-PLAN-vX.Y.Z` to complete the release.
