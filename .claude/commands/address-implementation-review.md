---
id: address-implementation-review
type: command
name: /address-implementation-review
description: Developer command to address findings from an implementation review round — fixes code/tests, appends address summary to IMPL-REVIEW file, re-enters review cycle
status: active
version: 1
primary_role: developer
links:
  - relation: follows
    target: review-implementation
  - relation: precedes
    target: review-implementation
---

# /address-implementation-review

## Purpose

Address all findings from the latest round of `/review-implementation` for a given milestone. Fix code and/or tests, append an address summary to the `IMPL-REVIEW-NNNN-MN.md` file, then re-enter the review cycle.

## Usage

```
/address-implementation-review PLAN-0001 M1
```

Both the plan ID and the milestone are required — the milestone identifies which review file to read and which code to fix.

## Required Inputs

- **`sdlc/reviews/IMPL-REVIEW-NNNN-MN.md`** — read the latest round's findings (the last `## Round N` section in the file)
- **PLAN-NNNN** — milestone MN task list and AC coverage (for context when fixing)
- **TEST-NNNN** — AC coverage map for MN (for QE findings that require test corrections)
- Source files for MN — read the affected files identified in each finding before making changes

## Process

### Step 1 — Read findings

Open `sdlc/reviews/IMPL-REVIEW-NNNN-MN.md`. Read `current_review_round` from frontmatter and navigate directly to `## Round {current_review_round}` — do not scan for it. Collect all findings, separated into:
- **Blocking** — must be fixed before approval
- **Non-blocking** — fix or explicitly defer with a documented reason

### Step 2 — Address each finding

For each blocking finding:
1. Identify the file(s) to change from the finding's description
2. Read the current file content
3. Make the fix — code or test as appropriate
4. Do not change scope beyond what the finding requires

For each non-blocking finding:
- Fix it if the fix is straightforward and scoped
- If deferring: document the reason in the address summary (e.g., "deferred to M4 — not in MN's scope", "accepted risk per ADR-NNNN")

### Step 3 — Verify fixes

After addressing all findings, run the project's verification suite (from `project.yml` stack configuration):

```bash
none
none
```

If `stack.lint` is `none`, skip it. All checks must pass before appending the address summary. If a fix introduces a new failure, resolve it before proceeding.

### Step 4 — Append address summary

Append to `sdlc/reviews/IMPL-REVIEW-NNNN-MN.md` immediately after the last round section:

```markdown
### Address Summary — Round N
Date: [today]

| Finding | Resolution |
|---|---|
| [ROLE-N] | Fixed: [one-line description of what changed] |
| [ROLE-N] | Deferred: [reason] |

**Not addressed:** [any finding explicitly deferred, with reason — or "None."]
```

### Step 5 — Handoff

```
/review-implementation PLAN-NNNN MN
```

State this next step explicitly so the user knows the cycle continues.

## Notes

- Do not open a new file for each address round — all rounds live in the same `IMPL-REVIEW-NNNN-MN.md`
- Do not remove or overwrite prior round content — only append
- If a blocking finding is genuinely unreproducible or the recommended resolution is wrong, surface that in the address summary rather than silently skipping it — the next review round will adjudicate
- **Do not commit code fixes here.** All source changes accumulate in the working tree until `/approve-implementation` commits them as a single milestone commit. The address summary appended to `IMPL-REVIEW-NNNN-MN.md` is also not committed at this stage — it commits at approval alongside everything else.
