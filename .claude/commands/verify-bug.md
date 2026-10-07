---
id: verify-bug
type: command
name: /verify-bug
description: Quality Engineer command to confirm a bug fix holds in production — executes the original reproduction steps against the live environment and closes or reopens BUG-NNNN
status: active
version: 2
primary_role: quality-engineer
changelog:
  - v2 — Handoff now asks solo-vs-team before pushing, like every other commit-producing command in the pipeline. Found during a full-pipeline drift audit: the entire Bug/Incident/Postmortem family committed locally but never pushed regardless of team context.
---

# /verify-bug

## Purpose

Confirm that the fix for BUG-NNNN works in the production environment. Executes the original reproduction steps from the bug report against the live system. If the defect no longer occurs, BUG-NNNN is closed as `verified`. If it still occurs or a regression is introduced, BUG-NNNN is `reopened` and returned to the fix pipeline.

## Usage

```
/verify-bug BUG-NNNN
```

Run after the fix has been deployed to production. BUG-NNNN must be in `pending_verification` status.

---

## Step 1 — Read BUG-NNNN

Read `sdlc/bugs/BUG-NNNN.md`. Confirm `status: pending_verification` — if not, stop: "BUG-NNNN is not in `pending_verification` status. Confirm the fix has been deployed to production before running `/verify-bug`."

Note the original reproduction steps, actual behavior, expected behavior, and the fix description from the `## Fix` section.

---

## Step 2 — Execute reproduction steps in production

Present the original reproduction steps to the user for execution against the production environment:

> **Verification check for BUG-NNNN — [title]**
>
> Run the original reproduction steps against the production URL:
>
> **Environment:** production
> **Version:** [fix version — from linked release]
>
> [Numbered reproduction steps from BUG-NNNN, verbatim]
>
> **Previously:** [actual behavior from the bug report]
> **Expected now:** [expected behavior from the bug report]
>
> Result? (fixed — defect no longer occurs / still present / different behavior observed)

Wait for the user's response before proceeding.

---

## Step 3 — Verify no regression on related paths

For Critical and High severity bugs, verify adjacent behavior was not broken by the fix. Derive 2–3 related checks from the linked SPEC AC or the surrounding feature flow:

> **Regression check — confirm the fix did not break adjacent behavior:**
>
> [2–3 checks derived from SPEC-NNNN ACs adjacent to the fixed behavior]
>
> Result? (all pass / regression found — describe)

Skip regression checks for Medium and Low severity bugs unless the fix touched shared code paths noted in the `## Fix` section.

---

## Step 4 — Record verification result

### If verified (defect no longer occurs, no regression)

Update BUG-NNNN frontmatter:
```yaml
status: verified
verified_at: [date]
```

Append to the `## Verification` section:
```markdown
## Verification

**Result:** Verified — fixed in production
**Date:** [date]
**Verified by:** [QE name]
**Version:** [production version]

Reproduction steps executed against production. Defect no longer occurs. [Adjacent regression checks passed | Regression checks skipped — Medium/Low severity.]
```

### If still present or regression found

Update BUG-NNNN frontmatter:
```yaml
status: reopened
```

Append to the `## Verification` section:
```markdown
## Verification

**Result:** Reopened — [defect still present | regression introduced]
**Date:** [date]
**Verified by:** [QE name]
**Version:** [production version]

[Description of what was observed — what still fails or what new behavior was introduced.]

**Next step:** Return to triage — `/triage-bug BUG-NNNN` to reassign the fix.
```

---

## Quality Gate

- [ ] BUG-NNNN was in `pending_verification` status before starting
- [ ] Original reproduction steps executed verbatim in production (not staging or local)
- [ ] Regression checks run for Critical and High severity bugs
- [ ] Verification result documented with date and version
- [ ] BUG-NNNN frontmatter `status` updated

---

## Handoff

Commit the verification result:

```bash
git add sdlc/bugs/BUG-NNNN.md
git commit -m "bug(BUG-NNNN): [verified — fixed in production | reopened — fix did not hold]"
```

**Ask before pushing:**
> "Solo project (commit only, no push) or team project (push too)?"

```bash
# team only:
git push origin main
```

- **Verified:** "BUG-NNNN is closed. Fix confirmed in production."
- **Reopened:** "BUG-NNNN reopened. Run `/triage-bug BUG-NNNN` to reassign the fix."
