---
id: update-stack
type: command
name: /update-stack
description: Lead Developer command to update one or more stack values in project.yml and propagate changes into all command files that have those values baked in — use when a stack decision changes after /forge-init
status: active
version: 1
primary_role: lead-developer
---

# /update-stack

## Purpose

Update stack configuration after `/forge-init`. When a stack value changes — switching test runner, updating a deploy command, adding a migration command — this command updates `.claude/project.yml` and re-substitutes every command file that has the old value baked in.

Only `.claude/project.yml` and `.claude/commands/` are modified. ARCH.md, `sdlc/` artifacts, roles, and PIPELINE.md are never touched.

## Usage

```
/update-stack
```

No arguments — the command reads current values from `.claude/project.yml` and walks you through what to change.

---

## Step 1 — Read and display current stack configuration

Read `.claude/project.yml`. Display current values in two groups:

**Fields baked into command files** (changing these updates command files):

| Field | Current value | Commands affected |
|---|---|---|
| `stack.lint` | [value] | address-implementation-review, address-test-failures, approve-implementation, implement-plan |
| `stack.test.unit` | [value] | address-implementation-review, address-test-failures, approve-implementation, execute-tests, implement-plan |
| `stack.test.e2e` | [value] | execute-tests |
| `stack.deploy.command` | [value] | create-release-plan, execute-release |
| `stack.deploy.migration_command` | [value] | create-release-plan, execute-release |
| `stack.deploy.env_pull` | [value] | create-release-plan, execute-release |

**Fields stored in project.yml only** (changing these updates project.yml; no command files affected):

| Field | Current value |
|---|---|
| `stack.language` | [value] |
| `stack.framework` | [value] |
| `stack.database` | [value] |
| `stack.migration_tool` | [value] |
| `stack.ci` | [value] |
| `stack.observability` | [value] |
| `stack.auth` | [value] |

---

## Step 2 — Select fields to update

Ask:

> **Which field(s) do you want to update?**
> List the field names and new values (e.g., `stack.lint: npx tsc --noEmit && npx eslint .`), or say "cancel" to exit.

Accept multiple fields in one pass. For each field provided:
- Confirm it exists in the list above
- Record the current (old) value and the new value
- If the field is not recognized: "Unknown field: [name]. Check the field names in the table above."

---

## Step 3 — Validate new values

For each field being updated:

- **`stack.lint`** — must be a runnable shell command (non-empty string, not just a description). Remind the user: this command runs verbatim in review steps — it must succeed in the project's CI environment.
- **`stack.test.unit`** and **`stack.test.e2e`** — must be a runnable shell command, or the literal string `none` if no runner is configured.
- **`stack.deploy.command`**, **`stack.deploy.migration_command`**, **`stack.deploy.env_pull`** — must be a runnable shell command or the literal string `none`.
- **`project.yml`-only fields** — any non-empty string is accepted.

If any value looks like a description rather than a command (e.g., contains "handled by" or "see docs"), warn:
> "This looks like a description rather than a runnable command. Stack values are substituted verbatim into command files. Confirm this is correct? (yes / correct it)"

---

## Step 4 — Show impact for baked-in fields

For each field being updated that is in the **baked-in** group:

1. Search `.claude/commands/` for files containing the old value as a literal string.
2. Present the impact:

```
Updating stack.lint:
  Old value: [old]
  New value: [new]

  Command files that will be updated:
    - .claude/commands/implement-plan.md
    - .claude/commands/approve-implementation.md
    - .claude/commands/address-implementation-review.md
    - .claude/commands/address-test-failures.md

  (N files found containing the old value)
```

**If the old value is a generic string** (`none`, `unknown`, `TBD`, or any value shorter than 4 characters): warn before scanning:
> "The current value '[value]' is generic and may match unrelated content in command files. Review the list of affected files carefully before confirming."

**If no command files are found containing the old value:** note this:
> "No command files were found containing '[old value]'. The value will be updated in project.yml only. If your command files were manually edited, update them manually as well."

For **project.yml-only fields**: skip the scan — just confirm the value change.

---

## Step 5 — Confirm and apply

After showing impact for all fields being updated, ask for a single confirmation:

> **Summary of changes:**
> [list each field: old → new, N command files]
>
> **Confirm? (yes / cancel)**

If cancelled: "No changes made." Exit.

On confirmation, apply changes in this order:

1. **Update command files** — for each baked-in field with affected files: replace all occurrences of the old value with the new value in each affected `.claude/commands/*.md` file.

2. **Update project.yml** — replace the old value with the new value for each updated field in `.claude/project.yml`.

Log each file as it is written: "Updated: .claude/commands/[filename]"

---

## Quality Gate

Before reporting complete:
- [ ] All affected command files updated — no occurrence of old value remains in any file that was listed as affected
- [ ] `project.yml` updated for all changed fields
- [ ] No files outside `.claude/commands/` and `.claude/project.yml` were modified
- [ ] ARCH.md not touched
- [ ] `sdlc/` not touched

---

## Handoff

Report a complete summary:

```
Stack update complete.

Updated fields:
  stack.[field]: [old] → [new]  ([N] command files updated)
  stack.[field]: [old] → [new]  (project.yml only)

Updated command files:
  - .claude/commands/[filename]
  - ...
```

**Do not commit automatically.** Review the changes, then commit:

```bash
git add .claude/commands/ .claude/project.yml
git commit -m "chore: update stack — [what changed]"
```

If ARCH.md references the changed stack value (e.g., in the Tech stack table), note:
> "ARCH.md may reference [field] in its Tech stack table. Update it manually if needed — `/update-arch` will help when implemented."
