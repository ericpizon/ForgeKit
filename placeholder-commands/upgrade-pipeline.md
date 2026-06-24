---
id: upgrade-pipeline
type: command
name: /upgrade-pipeline
description: Pull the latest command and role templates from a ForgeKit installation and show a diff against this project's current pipeline files — allows opt-in adoption of ForgeKit improvements without re-running /forge-init
status: placeholder
version: 0
primary_role: lead-developer
created: 2026-06-21
---

# /upgrade-pipeline

## Intent

This command is a **placeholder**. It is not yet implemented.

The intent is: when a team has a newer version of ForgeKit locally and wants to pull pipeline improvements into a project that was /forge-initped with an older version, this command shows what changed and lets the user adopt changes selectively.

---

## Why this is needed

Bootstrap generates command and role files at a point in time. ForgeKit templates will evolve — new quality checks, improved role definitions, bug fixes in pipeline logic. Projects need a safe, diff-first path to adopt those improvements without re-running full /forge-init (which would overwrite customizations).

---

## Design constraints

- **Never silently overwrite.** Every change must be shown as a diff and explicitly accepted.
- **Project customizations are preserved.** If a project has edited a generated command file, those edits are shown in the diff context and the user decides what to keep.
- **Stack values are not re-substituted on top of customizations.** The diff is between the project's current file and what a fresh generation from the new template + current project.yml would produce. The user sees the full picture.
- **Roles and commands are upgraded independently.** The user can accept changes to roles without accepting changes to commands, or vice versa.

---

## What the implemented command should do

1. Identify the ForgeKit installation being used (path to ForgeKit directory, required as argument or from `.claude/project.yml` if recorded)
2. Read `forgekit_version` from `.claude/project.yml`; compare to the detected ForgeKit version
3. If versions match: "No upgrade available — your pipeline is already up to date."
4. If newer version detected: for each file in `templates/commands/` and `templates/roles/`:
   a. Generate what the new file would look like (template + project.yml substitution)
   b. Diff against current `.claude/commands/[file]` or `.claude/roles/[file]`
   c. If no diff: skip (unchanged)
   d. If diff: present it to the user with context
5. For each changed file: ask "Accept / Skip / Edit manually"
6. Apply accepted changes
7. Update `forgekit_version` in `.claude/project.yml` to the new version
8. Report: N files updated, N files skipped, N files left for manual review

---

## Usage (intended)

```
/upgrade-pipeline /path/to/ForgeKit     ← upgrade from a local ForgeKit installation
```

---

## What this command does NOT do

- It does not fetch ForgeKit from the internet or a package registry — ForgeKit is always a local installation
- It does not re-run /forge-init discovery — stack values, ARCH.md, and project identity are not touched
- It does not push changes — the user commits after reviewing

---

## How to implement later

This command is lower priority than `/update-stack` and `/update-arch`. Implement after ForgeKit has been applied to at least one additional project and template evolution has been observed in practice. The key engineering challenge is diffing against potentially-customized files without losing project-specific edits.
