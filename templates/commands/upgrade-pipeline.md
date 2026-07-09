---
id: upgrade-pipeline
type: command
name: /upgrade-pipeline
description: Lead Developer command to pull ForgeKit template updates into this project — diffs installed commands and roles against a newer ForgeKit installation and applies changes selectively, preserving project customizations
status: active
version: 1
primary_role: lead-developer
---

# /upgrade-pipeline

## Purpose

Pull ForgeKit template improvements into this project without re-running `/forge-init`. Diffs every installed command and role file against the source templates, presents a **human-readable description of what changed** (not a raw line diff), and applies changes you accept.

ARCH.md, `sdlc/` artifacts, and `project.yml` are never touched — only `.claude/commands/`, `.claude/roles/`, and `PIPELINE.md`.

## Usage

```
/upgrade-pipeline /path/to/ForgeKit
```

`/path/to/ForgeKit` is the directory containing the ForgeKit `templates/` folder. This is wherever you cloned or installed ForgeKit — not the target project directory.

---

## Step 1 — Locate ForgeKit and read project configuration

1. Verify the path argument was provided. If not: "Usage: `/upgrade-pipeline /path/to/ForgeKit` — provide the path to your ForgeKit installation directory."

2. Verify the path exists and contains a `templates/` directory with `commands/` and `roles/` subdirectories. If not found: "ForgeKit templates not found at `[path]`. Check the path and try again."

3. Read `.claude/project.yml` from the current project. Extract:
   - `forgekit_version` — the version recorded at init time
   - Stack values for token substitution: `stack.lint`, `stack.test.unit`, `stack.test.e2e`, `stack.deploy.command`, `stack.deploy.migration_command`, `stack.deploy.env_pull`

4. Read `[ForgeKit path]/.claude/commands/forge-init.md` frontmatter. Extract the `version:` field as the available ForgeKit version.

Show the comparison header before proceeding:

```
Comparing installed pipeline against ForgeKit at [path].

Installed version: [forgekit_version from project.yml]
Available version: [version from forge-init.md frontmatter]
```

---

## Step 2 — Inventory all files

Scan three scopes in sequence. For each scope, read and compare every file.

### Commands

For each file in `[ForgeKit]/templates/commands/`:

1. Apply stack token substitution to the template content (same tokens as `/forge-init` Phase 5 Step 5):
   - `{{stack.lint}}` → value from project.yml
   - `{{stack.test.unit}}` → value from project.yml
   - `{{stack.test.e2e}}` → value from project.yml
   - `{{stack.deploy.command}}` → value from project.yml
   - `{{stack.deploy.migration_command}}` → value from project.yml
   - `{{stack.deploy.env_pull}}` → value from project.yml

2. Check whether `.claude/commands/[filename]` exists in this project.

3. Compare the substituted template content against the installed file content.

4. Categorize:
   - **new** — template file has no counterpart in `.claude/commands/`
   - **updated** — both exist; content differs after substitution
   - **unchanged** — content is identical after substitution

Additionally, note any files in `.claude/commands/` that have no counterpart in the ForgeKit templates — these are **local-only** (project-specific commands added by the team). List them in the summary but never touch them.

### Roles

For each file in `[ForgeKit]/templates/roles/`:

1. Check whether `.claude/roles/[filename]` exists in this project.
2. Compare template against installed file (no token substitution needed for roles).
3. Categorize as new / updated / unchanged.

### PIPELINE.md

Compare `[ForgeKit]/templates/PIPELINE.md` against the project root `PIPELINE.md`. PIPELINE.md is a static reference document — categorize as updated or unchanged.

---

## Step 3 — Present inventory summary

```
Upgrade inventory — ForgeKit v[available] vs installed v[installed]

Commands
  [N] updated      — template changed since install
  [N] new          — not yet installed in this project
  [N] unchanged    — no changes (will be skipped)
  [N] local-only   — project-specific; not touched

Roles
  [N] updated
  [N] new
  [N] unchanged

PIPELINE.md: [updated | unchanged]
```

If everything is unchanged: "Your pipeline is already up to date with ForgeKit v[version]. No changes to apply." Exit.

If there are changes: "Proceed to review each change? (yes / cancel)"

---

## Step 4 — Review and accept changes

Work through changed files in order: new commands → updated commands → new roles → updated roles → PIPELINE.md.

### New files

For each new file, present:

```
NEW COMMAND: [filename]
[description: field from the command's frontmatter]

Accept? (yes / skip)
```

If there are multiple new files, offer a bulk option first: "Accept all [N] new commands? (yes / review individually / skip all)"

### Updated files

For each updated file, present a **semantic summary of what changed** — not a raw line diff. Read both the installed version and the new template version and describe the changes in plain language organized by type:

```
UPDATED: [filename]
[description: field from the command's frontmatter]

What changed in this update:
  Added:   [e.g., "schema_version: 1 check in the Quality Gate"]
  Changed: [e.g., "Step 2 now writes a structured approval: block with approved_role,
            decision_summary, and policy_evaluation: fields — replaces flat approved_by / approved_date"]
  Removed: [e.g., nothing removed"]

Accept / Skip / Show full content?
```

"Show full content" displays both the installed and new versions in full so the user can read them before deciding.

**Customization warning:** Before presenting each updated file, check whether the installed content could be a project customization — the installed content differs from what a clean `/forge-init` would have generated at the installed version. If it appears customized, add:

```
⚠  This file appears to have been locally modified.
   Accepting will overwrite those changes. Use "Show full content" first.
```

Determine this by checking whether the installed file matches the pattern of a ForgeKit-generated command (frontmatter structure, section headers). If it differs significantly in structure or has additions not present in any ForgeKit template, treat it as potentially customized.

### PIPELINE.md

```
UPDATED: PIPELINE.md (static reference document — no customization expected)

What changed:
  [Semantic summary of what's new in the pipeline reference]

Accept? (yes / skip)
```

---

## Step 5 — Apply accepted changes

For each accepted file:

1. For command files: apply stack token substitution to the template before writing. Verify no `{{placeholder}}` tokens remain.
2. Write to the correct location:
   - Commands → `.claude/commands/[filename]`
   - Roles → `.claude/roles/[filename]`
   - PIPELINE.md → project root `PIPELINE.md`
3. Log each write: "Applied: [filename]"

After all files are written, update **only** the `forgekit_version` field in `.claude/project.yml` to the new version. Do not modify any other field.

---

## Quality Gate

Before reporting complete:
- [ ] All accepted command files written with no unreplaced `{{placeholder}}` tokens remaining
- [ ] No files outside `.claude/commands/`, `.claude/roles/`, and `PIPELINE.md` were modified
- [ ] ARCH.md not touched
- [ ] `sdlc/` not touched
- [ ] `project.yml` updated only for `forgekit_version` — no other field changed
- [ ] Local-only commands listed in the summary and not touched

---

## Handoff

Report a complete summary:

```
Upgrade complete — ForgeKit v[version] applied.

Applied:
  [N] commands updated
  [N] new commands added
  [N] roles updated
  [N] new roles added
  PIPELINE.md: [updated | unchanged]

Skipped:
  [N] files (skipped by choice)

Local-only commands (not touched):
  [list, or "none"]
```

If any updated files were skipped, note them:

```
Skipped files — review manually if you want to adopt these updates:
  - .claude/commands/[filename] — [one-line description of what changed]
```

**Do not commit automatically.** The user reviews the applied changes, then commits:

```bash
git add .claude/commands/ .claude/roles/ PIPELINE.md .claude/project.yml
git commit -m "chore: upgrade ForgeKit pipeline to v[version]"
```
