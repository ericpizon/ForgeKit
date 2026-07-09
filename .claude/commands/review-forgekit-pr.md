---
id: review-forgekit-pr
type: command
name: /review-forgekit-pr
description: ForgeKit maintainer command to review changes to the ForgeKit repo itself — checks that new or modified commands, templates, and docs stay within the established design direction before merging
status: active
version: "1.0"
primary_role: lead-developer
---

# /review-forgekit-pr

## Purpose

Review changes to the ForgeKit codebase against the established design invariants and design direction before merging. Catches schema drift, role attribution errors, broken cross-command references, quality gate weakening, and framing inconsistencies before they reach the public repo.

This is a ForgeKit meta-command — it reviews changes to ForgeKit itself, not artifacts produced by the pipeline.

## Usage

```
/review-forgekit-pr                      ← reviews staged + unstaged changes vs main
/review-forgekit-pr --files file1 file2  ← reviews specific files only
```

---

## Step 1 — Identify changed files

If no `--files` argument is given: run `git diff main...HEAD --name-only` to get the list of all files changed relative to the main branch. If on main with uncommitted changes, use `git diff --name-only`.

Display the list of changed files grouped by location:

```
Changed files:
  ForgeKit commands:      .claude/commands/[file]
  Pipeline templates:     templates/commands/[file]
  Role templates:         templates/roles/[file]
  Artifact READMEs:       templates/sdlc/[dir]/README.md
  Root templates:         templates/[file]
  Examples:               templates/examples/[file]
  Root docs:              README.md, GETTING-STARTED.md, CONTRIBUTING.md, SECURITY.md, etc.
  GitHub files:           .github/[file]
  Other:                  [file]
```

Read the full content of every changed file before beginning any checks.

---

## Step 2 — Run invariant checks by file category

Work through each changed file. Apply the checks for its category. Record every finding with:
- **Severity:** blocking or non-blocking
- **File and location:** specific line or section
- **What the invariant requires**
- **What the file currently does**
- **What to fix**

---

### Category A — Pipeline command templates (`templates/commands/*.md`)

For each new or significantly modified command file:

**A1. Frontmatter completeness** *(blocking if any field missing)*
Required fields: `id`, `type: command`, `name`, `description`, `status`, `version`, `primary_role`.
- `status` must be `active` or `placeholder` — no other values
- `version` must be `"1.0"` or higher (quoted string format matching forge-init version convention)
- `primary_role` must be exactly one of: `tpo`, `lead-developer`, `developer`, `quality-engineer`, `architect`, `security`, `sre`, `dba`

**A2. Required sections** *(blocking if any section missing)*
Every active command must have:
- `## Purpose`
- `## Quality Gate`
- `## Handoff`

The Handoff must include either a `git commit` instruction or an explicit "Do not commit" statement with a reason.

**A3. Approve-* command pattern** *(blocking if any field missing or wrong)*
Commands whose `id` starts with `approve-` must include in their Steps section an `approval:` block with all 7 fields and a sibling `policy_evaluation:` block with all 3 fields:

```yaml
approval:
  mode: repo_only
  approved_by: "[name]"
  approved_role: [role]
  approved_at: "[date]"
  decision_summary: "[one sentence]"
  external_event_id: null
  external_event_uri: null
policy_evaluation:
  evaluated: null
  policy_pack: null
  result: null
```

The `approved_role:` value must match the command's `primary_role:`. The Quality Gate must verify the `approval:` block is populated (not just present).

**A4. Artifact schema — new artifact types** *(blocking if fields missing)*
If the command introduces a new artifact type (new `output_artifact_type:` or a new inline YAML template that represents a new kind of artifact), the artifact frontmatter must include:
- `schema_version: 1`
- `generation:` block with: `command`, `command_version: "[use version from this command's frontmatter]"`, `model: "[self-identify from system context]"`, `generated_at: "[today's date]"`, `session_id: null`

**A5. Stack tokens** *(blocking if unapproved token used)*
The only permitted `{{stack.*}}` tokens in command templates are:
- `{{stack.lint}}`
- `{{stack.test.unit}}`
- `{{stack.test.e2e}}`
- `{{stack.deploy.command}}`
- `{{stack.deploy.migration_command}}`
- `{{stack.deploy.env_pull}}`

Any other `{{stack.*}}` token is a blocking finding. Other `{{...}}` tokens (like `{{project.name}}`) are not permitted in command templates — those belong in `templates/CLAUDE.md.template`, `templates/project.yml.template`, or `templates/sdlc/architecture/ARCH.md.template` only.

**A6. Cross-command references** *(blocking if command does not exist)*
For every `/command-name` reference in the command body: verify the referenced command exists as a file in `templates/commands/` or `.claude/commands/`. A reference to a non-existent command is a blocking finding.

**A7. PIPELINE.md consistency** *(non-blocking)*
If the command appears in the pipeline flow (it is called from a stage or produces an artifact used by a stage), check whether PIPELINE.md has been updated to include it. If not, flag as non-blocking: "PIPELINE.md not updated to include [command]."

**A8. Quality gate rigor** *(blocking if gate is weakened)*
If the command is modifying an existing command: check whether any existing Quality Gate checks have been removed or softened. A Quality Gate check may be added but not removed. If a check was removed, it is a blocking finding unless the corresponding command step was also removed.

**A9. Back-cycle completeness** *(non-blocking)*
If the command can produce a `revision_requested` or `needs_revision` status on an artifact, verify that a corresponding `address-*` command exists. If not, flag as non-blocking.

---

### Category B — Role templates (`templates/roles/*.md`)

**B1. Role identity** *(blocking)*
Role files must not introduce new role identities beyond the established 8: `tpo`, `lead-developer`, `developer`, `quality-engineer`, `architect`, `security`, `sre`, `dba`. Any new role requires explicit design justification.

**B2. Scope consistency** *(non-blocking)*
If a role file change expands a role's scope in a way that conflicts with another role's established domain, flag it for discussion.

---

### Category C — Artifact READMEs (`templates/sdlc/*/README.md`)

**C1. Role attribution** *(blocking)*
For every command listed in a README's command table: verify the "Role" column matches the command's `primary_role:` in `templates/commands/`. A mismatch is a blocking finding.

**C2. Command existence** *(blocking)*
Every command listed in a README's command table must exist as a file in `templates/commands/`. A reference to a non-existent command is a blocking finding.

---

### Category D — `templates/PIPELINE.md`

**D1. Role attribution** *(blocking)*
For every row in the pipeline table: verify the role column matches the `primary_role:` of the corresponding command file.

**D2. Command existence** *(blocking)*
Every command listed must exist in `templates/commands/`.

**D3. Stage numbering** *(non-blocking)*
Stages should be sequential without gaps. A gap or renumbering without a corresponding comment is a non-blocking flag.

---

### Category E — `forge-init.md` and `project.yml.template`

**E1. Version alignment** *(blocking)*
`forge-init.md` frontmatter `version:` must match `forgekit_version:` in the inline project.yml block and in `templates/project.yml.template`. All three must use the same string format.

**E2. sdlc/ directory count** *(blocking)*
If the sdlc/ directory list in Phase 5 Step 6 changed, the Quality Gate item "All [N] `sdlc/` directories created" must reflect the new count.

**E3. Placeholder reference accuracy** *(non-blocking)*
Phase 5b should accurately describe what is in `placeholder-commands/` and use a current example. If a command moved from `placeholder-commands/` to `templates/commands/`, the example must be updated.

---

### Category F — Example artifacts (`templates/examples/*.md`)

**F1. Schema compliance** *(blocking)*
All example artifact frontmatter must include `schema_version: 1`, a `generation:` block with `command_version` and `session_id: null`, and (where approved) a structured `approval:` block matching the current schema.

**F2. Fictional project boundary** *(blocking)*
Examples must use the Fieldstone fictional project. No real company names, real personal names, real infrastructure references, or content that could be traced to a specific employer.

---

### Category G — Root docs and `.github/` files

**G1. IP cleanliness** *(blocking)*
No employer names, internal system references, personal emails, real credentials, or content that could be traced to prior employer IP.

**G2. Framing consistency** *(non-blocking)*
Documentation should consistently frame ForgeKit as:
- An experimental AI-SDLC protocol
- Open protocol (Apache 2.0), not compliance software
- Repo-first with hybrid-ready approvals
Check for language that overstates certification, guarantees, or production readiness.

**G3. Placeholder command labeling** *(non-blocking)*
References to placeholder commands (currently only `/update-arch`) should be honest: clearly marked as stubs. References to formerly-placeholder commands that are now active (`/update-stack`, `/upgrade-pipeline`) should not call them placeholders.

---

## Step 3 — Holistic design direction assessment

After completing per-file checks, assess the changeset as a whole against ForgeKit's four core design principles:

**"The AI does the work. The human makes the decisions."**
Do any changes shift decision-making away from humans — removing approval gates, weakening confirmation requirements, or making human sign-off implicit?

**"One source of truth. No silent drift."**
Do any changes duplicate information that has an authoritative home elsewhere, or create a new artifact type that overlaps with an existing one without superseding it?

**"Process continuity across AI sessions."**
Do any new commands leave the pipeline in an ambiguous state? Does every command's Handoff clearly indicate what to run next so a cold-start agent can orient?

**"The git log is the audit trail."**
Do any changes push commits earlier (before review) or later (after the window closes) than the pipeline expects?

If any principle is violated by the changeset as a whole, record it as a holistic finding separate from the per-file findings.

---

## Quality Gate

Before presenting the verdict:
- [ ] Every changed file has been read in full — no skipped files
- [ ] All Category A–G checks applied to every applicable file
- [ ] Holistic assessment completed
- [ ] Every finding includes: severity, specific location, what was expected, what was found, what to fix

---

## Verdict

Present findings organized by severity:

```
## /review-forgekit-pr — [date]

### Blocking findings ([N])

[finding-id] blocking — [one-line title]
[file, location]
[what the invariant requires]
[what the file does]
[what to fix]

---

### Non-blocking findings ([N])

[finding-id] non-blocking — [one-line title]
[same structure]

---

### Holistic assessment

[One paragraph: does the changeset as a whole stay within the design direction? Any systemic drift or tension with the four principles?]

### Verdict

[CLEAN — no findings, change is within design direction and ready to merge]
  OR
[FINDINGS PRESENT — [N] blocking, [N] non-blocking. Blocking findings must be resolved before merge.]
```

Blocking findings must be resolved before the change is committed or merged. Non-blocking findings are advisory — the maintainer decides whether to address them now or track them as follow-up.
