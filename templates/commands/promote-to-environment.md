---
id: promote-to-environment
type: command
name: /promote-to-environment
description: Records a governance:promotion environment transition as one append-only row in a per-environment log — lightweight, proportional to routine-promotion frequency, never the full RELEASE-PLAN ceremony
status: active
version: 1
primary_role: any
output_artifact_type: promotion-record
changelog:
  - v1 — initial authoring. Implements AC13 (governance-value reasoning), AC14 (`governance: ci` has no command path), AC15 (append-only per-environment record, reusing `approve-release-plan.md`'s `candidate_ref` shape). See ADR-0004.
---

# /promote-to-environment

## Purpose

Record a promotion of the current candidate(s) to a named environment, when that environment's `governance` is `promotion` — a lightweight, append-only log entry, never the full RELEASE-PLAN ceremony `governance: release` still uses. Coordination-repo only: `environments` (AC5) is declared once, in the coordination repo's own `project.yml`, covering every participant collectively.

## When to Invoke

- A thread's current candidate(s) are ready to move into a routine, non-release environment (e.g. a shared Dev or QA environment) — the gap SPEC-0001's G4 exists to close: today, a team either uses the full RELEASE-PLAN ceremony for *every* environment transition, or tracks routine ones nowhere at all.

## Usage

```
/promote-to-environment <environment>
```

`<environment>` is the environment's **name** as declared in `project.yml`'s `environments` list (e.g. `qa`) — not a governance value. This command resolves the name to a governance value itself (Step 1); it is never invoked with a governance value directly.

## Required Inputs

- **`project.yml`'s `environments` list** (AC5) — this repo must be `repo_role: coordination` and must declare the named environment
- **Candidate ref(s)** — the commit/ref being promoted, per participant (same shape AC9/`approve-release-plan.md` already use)

## Step 1 — Resolve governance, never the environment's name (AC13)

Read `project.yml`'s `environments` list. Find the entry matching `<environment>`. **Reason only about its `governance` value from here on — never branch on the literal environment name** (`dev`, `qa`, `staging`, `production`, or anything else). This is AC13's own explicit requirement: every command that reasons about environment progression must work identically for an environment named `qa` and one named `pre-prod-integration-ring-3`, as long as their `governance` values match (TC-FUNC-30).

If `<environment>` isn't found in the list at all: stop — "`[environment]` is not declared in this repo's `environments` list. Add it to `project.yml` first (see `templates/sdlc/promotions/README.md`)."

**Validate the matched entry's `name` before using it anywhere in a file path** (SEC-0001 SEC-3's residual-risk note, resolved at M3 by naming this command as the point-of-use validator for `environments[].name`, since no command writes that field — it's direct-edited, per `templates/sdlc/promotions/README.md`). Reject anything outside `[a-z0-9-]` — the same character class `create-spec.md`'s slug derivation already enforces for the same reason (this value becomes `{environment}` in `sdlc/promotions/{environment}.md` below; an unsanitized value containing `../` could otherwise escape that directory). On a non-match: stop — "`environments[].name: [value]` in `project.yml` doesn't match the expected `[a-z0-9-]` shape. Fix it directly in `project.yml` before promoting to it." Do not attempt to sanitize and proceed anyway — unlike a derived slug, this is a human-edited config value; silently mangling it could promote to a different file than the human intended.

Branch on the resolved `governance` value:

- **`governance: ci`** (AC14) — **stop immediately.** "`[environment]`'s governance is `ci` — this stays outside ForgeKit's scope entirely, same as today. There is no command path here, by design; whatever already handles CI deploys for this environment is unaffected." This command has no further behavior for this governance value — not a degraded or partial path, no path at all.
- **`governance: release`** — **stop immediately.** "`[environment]`'s governance is `release` — use the existing RELEASE-PLAN ceremony (`/create-release-plan`, `/approve-release-plan`), unchanged. `/promote-to-environment` is scoped to `governance: promotion` only."
- **`governance: promotion`** — continue to Step 2.
- **Missing, empty, or any other value** (not one of the three above — e.g. the key absent entirely, or a typo like `promotoin`) — **stop immediately, do not guess.** "`[environment]`'s `governance` value (`[value, or "missing" if the key is absent]`) is not one of `ci`/`promotion`/`release`. Fix it directly in `project.yml` before promoting to it." Silently defaulting to any of the three real values here — treating a typo as `promotion` because that's this command's own scope, say — would be exactly the "governance value, never a hardcoded environment name" principle's quiet inverse: inferring governance from context instead of reading what's actually declared. (TC-EDGE-3 — flagged in TEST-0001 as an open question for whether this is a genuine spec gap, never explicitly resolved before TEST-0001's own approval; resolved here, during M5's test execution, as refuse-and-direct-to-fix-config, consistent with AC13's own "reason about the governance value" framing: a value outside the three defined ones isn't a fourth governance level to silently handle, it's invalid input.)

## Step 2 — Gather the candidate map and smoke-test result

Ask for (or read, if already known from context — e.g. a just-completed `/execute-tests` run):

> **Candidate ref(s) for each participant being promoted to `[environment]`:**
> [repo]: [ref]
> [repo]: [ref]
> ...
>
> **Smoke-test result?** (pass / fail / skipped — be honest; "skipped" is a legitimate answer for a low-stakes environment, not a gap to hide)

The candidate map's shape is **identical** to `approve-release-plan.md`'s own `candidate_ref` field — the same per-participant `{repo: ref}` mapping, not a newly-invented structure (AC15's explicit requirement: "the exact shape already used in `approve-release-plan.md`'s coordination approval model — not a newly-named field"). Unlike that command's full approval dance, there is no `PENDING_SELF_COMMIT` self-referential-ref step here and no per-participant `approved`/`coordination`/`overall` gate — this record is a log entry, not an approval artifact.

## Step 3 — Append the row (ADR-0004: one append-only log, never a numbered artifact)

**File:** `sdlc/promotions/{environment}.md` (e.g. `sdlc/promotions/qa.md`) — one file per environment, created lazily on this environment's first promotion if it doesn't exist yet (same lazy-creation pattern `approve-design.md`'s `SYSTEM-ARCH.md` regeneration already uses for its own first-time case). **Never** a new numbered artifact (`PROMOTION-NNNN`) — that would recreate exactly the ceremony-for-routine-events problem G4 exists to solve (ADR-0004).

**If the file doesn't exist yet, create it with this header, then the first row:**

```markdown
# Promotions — [environment]

One append-only row per `/promote-to-environment` invocation against this environment. Deliberately excludes a Gating-criteria table, Rollback plan, and Validation Plan — those stay exclusive to `governance: release`'s RELEASE-PLAN ceremony. See `templates/sdlc/promotions/README.md`.

| Candidate ref(s) | Promoted by | Promoted at | Smoke test |
|---|---|---|---|
| [repo]: [ref], [repo]: [ref] | [name] | [date] | [pass/fail/skipped] |
```

**If the file already exists:** append one new row to the existing table — never overwrite, never remove a prior row, never start a new table (AC15/ADR-0004). Two separate promotions of the same environment produce two rows in this one file, never two files or two artifacts (TC-ADR-0004-01).

**The row contains exactly these four fields and nothing else** (AC15's explicit exclusion): candidate ref(s), promoted by, promoted at, smoke-test result. No Gating-criteria table. No Rollback plan. No Validation Plan. Those three sections are exclusive to `governance: release`'s existing RELEASE-PLAN ceremony — adding any of them here would blur the exact distinction G4 exists to draw.

## Quality Gate

Before committing:
- [ ] Branched on `governance`, never on the environment's literal name
- [ ] `governance: ci` or `governance: release` both stopped with no row written — never a degraded path
- [ ] The candidate map reuses `approve-release-plan.md`'s exact `{repo: ref}` shape
- [ ] The appended row has exactly four fields — no Gating-criteria, Rollback, or Validation Plan content anywhere in it
- [ ] A prior row (if any) is unchanged — this is an append, never a rewrite

**Self-check before proceeding:** Re-read each item above. Confirm each is complete, or state what was skipped and why before continuing to Handoff.

## Handoff

**Show the user the row before committing:**

> **Appending to `sdlc/promotions/[environment].md`:**
> `[row content]`
>
> Confirm: (1) proceed with the commit? (2) solo project (no push) or team project (push too)?

**Commit (after user confirms):**
```bash
git add sdlc/promotions/{environment}.md
git commit -m "promotion: {environment} - [N] participant(s), smoke test [pass/fail/skipped]"
```

**Push (only if user confirms team context):**
```bash
git push origin main
```

No further handoff — this command's effect is the log row itself. It does not trigger any downstream command the way `/approve-spec` triggers `/create-design`; a routine promotion is a record, not a pipeline gate.
