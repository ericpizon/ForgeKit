---
id: update-stack
type: command
name: /update-stack
description: Update one or more values in project.yml's stack section and regenerate all pipeline commands that bake in those values — use when a stack decision changes after /forge-init (e.g., switching test runner, updating deploy command)
status: placeholder
version: 0
primary_role: lead-developer
created: 2026-06-21
---

# /update-stack

## Intent

This command is a **placeholder**. It is not yet implemented.

The intent is: after /forge-init, if any stack value changes (test runner, deploy command, migration command, etc.), this command updates `.claude/project.yml` and regenerates the subset of pipeline commands whose text was baked from those values.

---

## Why this is needed

At /forge-init, execution values (deploy command, test runner, migration command) are baked directly into generated command files rather than read from project.yml at runtime. This avoids YAML parsing risk during critical operations. The tradeoff is that when a value changes, affected commands need to be regenerated.

`/update-stack` is the managed path for doing this — so the user doesn't have to manually find and edit every affected command file.

---

## What the implemented command should do

1. Read `.claude/project.yml` and identify current stack values
2. Prompt the user for which field(s) to update and the new value(s)
3. Validate the new value (e.g., deploy command should be a non-empty string; test runner should be a recognized tool name or custom)
4. Update `.claude/project.yml` with the new value(s)
5. Identify all `.claude/commands/*.md` files that contain the old value baked in (search for the literal string)
6. Show the user the list of affected command files and the substitution that will be made
7. Confirm before making changes
8. Apply the substitution in each affected file
9. Report what was changed

---

## Fields that trigger command regeneration

When any of these fields change, associated commands must be regenerated:

| project.yml field | Affected commands (partial list) |
|---|---|
| `stack.lint` | `implement-plan`, `approve-implementation` |
| `stack.test.unit` | `implement-plan`, `approve-implementation`, `execute-tests`, `address-test-failures`, `validate-release` |
| `stack.test.e2e` | `execute-tests` |
| `stack.deploy.command` | `execute-release`, `validate-release` |
| `stack.deploy.migration_command` | `execute-release` |
| `stack.deploy.env_pull` | `execute-release` |
| `stack.framework` | `review-implementation`, `validate-release` |

---

## Fields that do NOT trigger command regeneration

Behavioral fields in project.yml (`artifact_mode`, `team_size`, `project.type`) flow through CLAUDE.md ambient context — no command regeneration needed for these. Artifact mode in particular is handled entirely by the CLAUDE.md block, not baked into individual command files.

---

## Scope note

This command only updates **this project's** `.claude/` directory. It does not pull new command versions from ForgeKit. For that, see `/upgrade-pipeline`.

---

## How to implement later

To implement: read `templates/commands/` from the ForgeKit installation, re-run the substitution step from /forge-init Phase 5 Step 4, but only for the fields that changed and only for the affected files. The "affected files" list can be determined by scanning for the old value string in existing command files.
