---
id: forge-init
type: command
name: /forge-init
description: ForgeKit initialization command — on first run detects greenfield vs brownfield, configures artifact mode, builds ARCH.md, and generates all pipeline files; on re-run detects existing configuration and routes to targeted update options without touching ARCH.md or sdlc/ artifacts
status: active
version: 1
primary_role: lead-developer
created: 2026-06-21
updated: 2026-06-22
---

# /forge-init

## Purpose

Initialize the ForgeKit SDLC pipeline on a project. Produces a self-contained `.claude/` setup and `sdlc/` artifact structure that does not depend on ForgeKit at runtime.

**First run:** detects greenfield vs brownfield, walks through configuration, performs automated codebase discovery (brownfield), requires ARCH.md sign-off, and generates all pipeline files.

**Re-run** (already-initialized project): detects the existing configuration and presents targeted update options — it never regenerates ARCH.md or touches `sdlc/` artifacts.

## Usage

```
/forge-init                          ← initializes the current working directory
/forge-init /path/to/target-project  ← initializes a specified directory
```

**Run from ForgeKit's directory** with the target project path as argument, or open Claude Code in the target project and run `/forge-init` directly if forge-init.md is accessible.

`/forge-init` is safe to re-run on an already-initialized project — it detects the existing configuration and routes to targeted update options rather than repeating the full setup flow.

---

## Pre-flight

Check whether the target project has already been initialized:

- If `.claude/project.yml` exists, or `.claude/CLAUDE.md` / `CLAUDE.md` contains a line starting with `<!-- ForgeKit Pipeline` → this project has already been initialized. Enter **Re-run mode** (next section) instead of proceeding to Phase 1.

Identify the ForgeKit templates directory (relative to this command file: `../../templates/`). Confirm it exists and contains `commands/`, `roles/`, and `sdlc/` subdirectories. If not found, stop: "ForgeKit templates not found. Ensure you are running /forge-init from a valid ForgeKit installation."

---

## Re-run mode

*Entered when pre-flight detects an existing `.claude/project.yml` or ForgeKit block in CLAUDE.md.*

### Step R1 — Show current configuration

Read `.claude/project.yml` and display a summary so the user knows what they are working with:

```
This project has already been initialized with ForgeKit.

Current configuration:
  Project:       [project.name] ([project.type], [team.size] team)
  Artifact mode: [artifact_mode][if copy-paste: — [external_tool.name]]
  Stack:         [stack.language] / [stack.framework] / [stack.database]
  Test runner:   [stack.test.unit]
  Lint:          [stack.lint]
  E2E runner:    [stack.test.e2e]
  Deploy:        [stack.deploy.command]
  ForgeKit:      v[forgekit_version] (initialized [initialized_at])
```

### Step R2 — What needs to change?

Present the menu:

```
What would you like to update?

  a) Stack values (test runner, deploy command, lint, etc.)
  b) Artifact mode  (currently: [artifact_mode])
  c) Project identity (name, description, team size)
  d) Refresh command and role templates (new ForgeKit version)
  e) Full redo — re-answer everything; ARCH.md and sdlc/ preserved
  f) Cancel
```

---

### Option A — Stack values

Stack value changes are handled by `/update-stack`, which updates `project.yml` and regenerates only the affected command files.

> Run: `/update-stack`

Exit re-run mode without making any changes.

---

### Option B — Artifact mode change

1. Show the current mode and ask:

   > **New artifact mode?**
   > - Markdown — all artifacts are `.md` files in this repo
   > - Hybrid — SPEC, BUG, and INC live in an external tool; technical artifacts remain as `.md` files
   > - Copy-paste — all artifacts live in an external tool

2. If the same mode is selected: "Artifact mode is already `[mode]` — no change needed." Exit.

3. If switching **to hybrid or copy-paste**: ask the external tool questions from Phase 3 (tool name and SPEC/BUG/INC artifact type mapping).

4. If switching **to markdown**: confirm the `external_tool` block will be removed from `project.yml`.

5. Update `project.yml`:
   - Change `artifact_mode`
   - Add `external_tool:` block (with substituted values) if switching to hybrid or copy-paste
   - Remove `external_tool:` block entirely if switching to markdown

6. Regenerate the CLAUDE.md ForgeKit block using the same substitution and conditional process as Phase 5 Step 2. Preserve all content outside the block.

7. Show a diff of all changes and confirm before writing.

8. After writing, note:

   > Artifact mode updated. No command files were regenerated — artifact mode is conveyed through the CLAUDE.md ambient context block, not baked into individual command files.

---

### Option C — Project identity

1. Re-ask each field with the current value shown as the default:

   > **Project name?** (current: [project.name])
   >
   > **One-sentence description?** (current: [project.description])
   >
   > **Team size?** (current: [team.size])

2. If all values are unchanged: "No changes." Exit.

3. Update `project.yml` project section with any changed values.

4. Regenerate the CLAUDE.md ForgeKit block (the `## Pipeline:` header and metadata lines). Preserve all content outside the block.

5. Show a diff and confirm before writing.

6. If the project name changed, note:

   > ARCH.md references the project name in its heading and frontmatter. Update `sdlc/architecture/ARCH.md` manually if needed.

---

### Option D — Template refresh

Command and role template refresh is handled by `/upgrade-pipeline`, which diffs the current `.claude/commands/` and `.claude/roles/` files against the ForgeKit templates and lets you selectively apply updates.

> Run: `/upgrade-pipeline`
>
> Note: `/upgrade-pipeline` is currently a placeholder command — see its documentation for implementation status.

Exit re-run mode without making any changes.

---

### Option E — Full redo

**Step E1 — Warn before asking for confirmation:**

```
Full redo will regenerate:
  ✓ .claude/project.yml
  ✓ CLAUDE.md ForgeKit block
  ✓ PIPELINE.md
  ✓ All role files in .claude/roles/
  ✓ All command files in .claude/commands/

Full redo will NOT touch:
  ✗ sdlc/architecture/ARCH.md
  ✗ sdlc/ artifacts (plans, specs, designs, reviews, etc.)
  ✗ Any other files outside .claude/ or the project root
```

**Step E2 — Check for locally modified files:**

Compare the content of each file in `.claude/commands/`, `.claude/roles/`, and the project root `PIPELINE.md` against the corresponding template in `templates/commands/`, `templates/roles/`, and `templates/PIPELINE.md`. If any file differs from its template source, list it before asking for confirmation:

```
The following files appear to have been locally modified since initialization.
Overwriting them will lose those customizations:
  - .claude/commands/[filename]
  - .claude/roles/[filename]
```

If no modifications are found, skip this notice.

**Step E3 — Require typed confirmation:**

> Type `REDO` to confirm and proceed, or anything else to cancel:

If the user does not type `REDO` exactly: "Full redo cancelled. No changes made." Exit.

**Step E4 — Run the redo sequence:**

1. **Phase 1** — Re-ask project identity. Show current values as defaults; user presses enter to keep them.
2. **Skip Phase 2** — project type (greenfield/brownfield) does not change on redo; preserve the value from `project.yml`.
3. **Phase 3** — Re-ask artifact mode. Show current value as default.
4. **Phase 4A (stack configuration)** — Re-ask all stack questions with current `project.yml` values as defaults. For brownfield projects, skip the Phase 4B automated scan — codebase discovery has already been done and captured in ARCH.md. Ask only the stack questions.
5. **Skip ARCH.md generation** — do not generate, regenerate, or modify `sdlc/architecture/ARCH.md` under any circumstances.
6. **Phase 5** — Regenerate all outputs (Steps 1–5b). For Step 6 (sdlc/ directories): create any missing directories and add missing READMEs; do not overwrite existing directory contents.

**Step E5 — Post-redo note:**

> **Full redo complete.**
>
> `sdlc/architecture/ARCH.md` was not regenerated. If your stack changed significantly, update it manually to reflect the current configuration.

---

### Option F — Cancel

"No changes made." Exit re-run mode.

---

## Phase 1 — Project identity

Ask:

> **Project name?** (default: current directory name)
>
> **One-sentence description?** (what does this project do — this appears in ARCH.md and CLAUDE.md)
>
> **Team size?** (solo — one person working alone / small — 2–5 engineers / enterprise — 6+ or cross-team)

---

## Phase 2 — Project type

Ask:

> **Is this a greenfield project (starting fresh) or brownfield (existing codebase)?**
>
> - Greenfield — new project, no existing code
> - Brownfield — existing codebase, adding the ForgeKit pipeline

---

## Phase 3 — Artifact mode

Ask:

> **How will SDLC artifacts be stored and provided?**
>
> - **Markdown** *(default)* — all artifacts live as `.md` files in this repo. Commands read and write files directly.
> - **Hybrid** — SPEC, BUG, and INC live in an external tool (Jira, Linear, etc.) as the source of truth; all technical artifacts (DESIGN, PLAN, TEST-PLAN, SEC, OBS, IMPL-REVIEW, RELEASE-PLAN, RELEASE-NOTES, PM) remain as `.md` files in the repo. Best for teams where PMs/SREs manage requirements and incidents in a project tracker and engineers own technical docs in the repo.
> - **Copy-paste** — all artifacts live in an external tool. Commands prompt you to paste content in and output content for you to copy back for every artifact type.

If **hybrid** or **copy-paste**:

> **What tool will you be using?** (e.g., Jira, Linear, GitHub Issues, Notion, Confluence)
>
> **Map your tool's artifact types to the pipeline:**
>
> | Pipeline artifact | Your tool's equivalent |
> |---|---|
> | SPEC (feature requirement) | ? |
> | BUG (defect report) | ? |
> | INC (production incident) | ? |
> | *(leave blank to use generic prompts)* | |

---

## Phase 4A — Greenfield stack configuration

*Skip to Phase 4B if brownfield.*

### Step 1 — Stack preset

Start with a preset to pre-fill all fields, then customize any that don't fit.

> **Choose a stack preset, or answer each question individually (option G):**
>
> | | Preset | Language | Framework | Database | Deploy target |
> |---|---|---|---|---|---|
> | **A** | JavaScript / Next.js | TypeScript | Next.js (full-stack) | PostgreSQL | Vercel |
> | **B** | JavaScript / Express | TypeScript | Express + React | PostgreSQL | Railway |
> | **C** | Python / FastAPI | Python | FastAPI + React | PostgreSQL | Railway |
> | **D** | Python / Django | Python | Django (full-stack) | PostgreSQL | Heroku |
> | **E** | Java / Spring Boot | Java | Spring Boot + React | PostgreSQL | AWS ECS |
> | **F** | .NET / ASP.NET Core | C# | ASP.NET Core + Angular | PostgreSQL | Azure App Service |
> | **G** | Ruby / Rails | Ruby | Rails (full-stack) | PostgreSQL | Heroku |
> | **H** | Go / Gin | Go | Gin + React | PostgreSQL | Fly.io |
> | **I** | Custom | — answer each question below — | | | |

Each preset pre-fills: language, framework, database, migration tool, unit test runner, E2E runner, lint command, deploy command, and CI. After selecting, only ask the fields the preset cannot know: auth approach, observability stack, env pull command, migration command, and any overrides the user wants to make.

Accept "unknown" or "TBD" for any field — these can be updated later with `/update-stack`.

### Step 2 — Confirm or customize

After applying the preset (or if Custom was selected, after asking each question), present a full summary and ask the user to confirm or correct any field:

> **Stack configuration — please confirm or correct:**
>
> | Field | Value | What it's used for |
> |---|---|---|
> | Language | [value] | ARCH.md, role context |
> | Framework | [value] | ARCH.md, implementation guidance |
> | Database | [value] | ARCH.md, DBA review context |
> | Migration tool | [value] | Release checklist migration step |
> | Unit test runner | [value] | `{{stack.test.unit}}` in all test/review commands |
> | E2E test runner | [value] | `{{stack.test.e2e}}` in execute-tests, validate-release |
> | Lint / type check | [value] | `{{stack.lint}}` in review and address-review commands — **must be an exact runnable command**, not a description (e.g., `npx tsc --noEmit && npx eslint .`) |
> | Deploy command | [value] | `{{stack.deploy.command}}` in execute-release |
> | Env pull command | [value] | `{{stack.deploy.env_pull}}` in release checklist |
> | Migration command | [value] | `{{stack.deploy.migration_command}}` in release checklist |
> | CI/CD | [value] | ARCH.md, quality gate context |
> | Observability | [value] | ARCH.md, OBS-NNNN instrumentation guidance |
> | Auth approach | [value] | ARCH.md, security review context |
>
> **Confirm, or correct any field?**

### Reference — full option lists per field

Use these when the user selects Custom (option I) or wants to override a preset field.

**Database:**
- *Relational (open source):* PostgreSQL, MySQL, MariaDB, SQLite
- *Relational (enterprise):* SQL Server, Oracle Database, IBM Db2, Aurora (AWS)
- *Document:* MongoDB, CouchDB, Firestore
- *Cloud-native:* DynamoDB (AWS), CosmosDB (Azure), Bigtable (GCP)
- *Other:* Redis (cache/primary), Cassandra, ClickHouse

**Database migration tool:**
- *JavaScript:* drizzle-kit, Prisma Migrate, Knex migrations, Sequelize CLI
- *Java:* Flyway, Liquibase
- *Python:* Alembic, Django migrations
- *.NET:* EF Core (`dotnet ef migrations`)
- *Ruby:* Rails Active Record migrations
- *Go:* golang-migrate, Atlas
- *DB-managed:* Supabase migrations, PlanetScale branching

**Auth approach:**
- *JavaScript libraries:* Auth.js (Next.js), Passport.js, Lucia
- *Managed services (SMB):* Clerk, Auth0, Firebase Auth, Supabase Auth
- *Enterprise identity:* Okta, Azure AD / Microsoft Entra ID, AWS Cognito, PingIdentity, ForgeRock
- *Protocols (self-hosted):* Keycloak (OIDC/SAML), Authentik, Dex
- *Custom:* custom JWT, custom session tokens, API key only
- *None:* internal tool with no auth requirement

If the user selects any session-based auth approach (any library or framework that maintains server-side sessions — e.g., Auth.js, Lucia, Passport for JavaScript; Flask-Login, Django auth for Python; Spring Security sessions for Java; Devise for Ruby), ask one follow-up question before moving on:
> **Session strategy: JWT sessions or database sessions?**
> - JWT — stateless; no DB lookup per request; tokens cannot be revoked without expiry
> - Database sessions — session stored in DB; revocable instantly; recommended for most apps
> *(Recommendation: database sessions unless you have a specific stateless/edge requirement)*

**Observability stack:**
- *Open standard:* OpenTelemetry (OTel) — works with any backend
- *SaaS (SMB):* Datadog, New Relic, Honeycomb, Grafana Cloud, Sentry
- *Enterprise:* Splunk, Dynatrace, AppDynamics, Elastic (ELK), IBM Instana
- *Cloud-native:* AWS CloudWatch, Azure Monitor, Google Cloud Operations
- *Self-hosted:* Prometheus + Grafana, Jaeger, Zipkin
- *None:* no observability configured yet

**Deploy command:**
- *PaaS:* `vercel --prod`, `railway up`, `heroku container:push web`, `fly deploy`
- *Cloud containers:* `aws ecs deploy`, `az webapp deploy`, `gcloud run deploy`
- *Kubernetes:* `kubectl apply -f k8s/`, `helm upgrade --install [release] [chart]`
- *Scripts:* `./deploy.sh`, `make deploy`
- *CI-only:* "handled by CI pipeline — no manual deploy command"

**CI/CD pipeline:**
- *Git-hosted:* GitHub Actions, GitLab CI, Bitbucket Pipelines
- *Enterprise:* Jenkins, Azure DevOps, TeamCity, Bamboo, CircleCI Enterprise
- *Cloud-native:* AWS CodePipeline, Azure Pipelines, Google Cloud Build
- *None:* no CI configured yet

Wait for confirmation, then proceed to generate ARCH.md from the confirmed answers (see Phase 4 → ARCH.md Generation below).

---

## Phase 4B — Brownfield discovery

*Skip to Phase 5 if greenfield.*

### Step 1 — Automated scan

Read the target project directory. Do not modify any files during this phase. Scan the following:

**Tech stack detection:**
- `package.json` / `package-lock.json` / `yarn.lock` / `pnpm-lock.yaml` → language (TypeScript/JavaScript), framework, dependencies, scripts
- `requirements.txt` / `pyproject.toml` / `Pipfile` → Python stack
- `pom.xml` / `build.gradle` / `build.gradle.kts` → Java/Kotlin stack
- `go.mod` → Go stack
- `Gemfile` → Ruby stack
- `*.csproj` / `*.sln` → .NET stack
- `Cargo.toml` → Rust stack

**Directory structure:**
- Top-level directory layout → module organization
- `src/` / `lib/` / `app/` structure → module boundaries, scanned recursively — list all route files, handlers, and nested sub-routes (e.g., `api/flags/[id]/route.ts` is a distinct endpoint from `api/flags/route.ts` and must appear separately in the module layout)
- `tests/` / `__tests__/` / `spec/` → test organization

**Test framework:**
- Existing test files → identify test runner, test patterns, naming conventions
- `vitest.config.*` / `jest.config.*` / `pytest.ini` / `phpunit.xml` → test configuration
- Look for integration test vs unit test separation

**Deployment configuration:**
- `vercel.json` / `netlify.toml` / `Dockerfile` / `docker-compose.yml` → deploy target
- `.github/workflows/` / `Jenkinsfile` / `.gitlab-ci.yml` / `.circleci/config.yml` → CI pipeline
- `k8s/` / `terraform/` / `pulumi/` → infrastructure approach

**Database and migrations:**
- `drizzle.config.*` / `alembic.ini` / `flyway.conf` → migration tool
- `migrations/` / `db/migrate/` / `alembic/versions/` → migration history and patterns
- ORM config files → database type and connection model
- Read all schema definition files (`schema.ts`, `schema.py`, `models.py`, `models/`, `entities/`, `*.entity.ts`, migration files, etc.) and list **every table or model defined** — do not filter, summarize, or omit any. Include application tables, framework adapter tables, audit tables, junction tables, and system tables alike. For each table, list **all columns** — do not summarize to just the notable ones. A missing table or column means ARCH.md will have an incomplete data layer, causing gaps in downstream security reviews and design decisions.

**Authentication:**
- Auth middleware files, auth config, session handling → auth approach and boundaries
- Protected route patterns → how auth is enforced

**Observability:**
- `instrumentation.*` / `opentelemetry.*` / `datadog.*` → observability stack
- Logging patterns in existing source files → log format and fields

**Security vulnerability scan:**
- Hardcoded secrets or credentials in source files (API keys, passwords, tokens in code — not in `.env` files)
- Dependency versions with known CVEs — flag packages that are significantly out of date or appear on common vulnerability lists
- Route handlers or API endpoints with no apparent auth check
- Direct SQL string construction (potential injection risk)
- Use of dangerous functions (`eval`, `exec`, `shell_exec`, `os.system`, `subprocess` with `shell=True`)
- Missing security headers in middleware or server config
- Sensitive data (PII, keys, tokens) appearing in log statements
- Dependencies fetched from untrusted or non-standard sources

### Step 2 — Present discovery findings

Present the full discovery output to the user, organized by section. For each section, state what was found and ask the user to confirm or correct:

```
## Discovery findings — please confirm or correct each section

### Tech stack
Detected: [language], [framework], [database]
Dependencies of note: [list significant ones]
Correct? (yes / corrections: ...)

### Module layout
[directory tree summary]
Notable boundaries: [identified module boundaries]
Correct? (yes / corrections: ...)

### Test framework
Detected: [unit runner], [e2e runner if found]
Test patterns observed: [TDD / integration-heavy / e2e-only / minimal]
Correct? (yes / corrections: ...)

### Deployment
Detected: [deploy target], [CI pipeline]
Deploy command: [detected or unknown]
Correct? (yes / corrections: ...)

### Database and migrations
Detected: [migration tool if any], [N migrations found]
Schema: [every table/model with every column — no omissions]
Correct? (yes / corrections: ...)

### Observability
Detected: [stack if any / none found]
Correct? (yes / corrections: ...)

### Security findings
[List each finding with severity estimate: Critical / High / Medium / Low]
  - [finding 1]
  - [finding 2]
  - (none found)

These will be carried into the pipeline as pre-existing findings for the Security Review phase.
```

### Step 3 — Targeted Q&A for gaps

After the user confirms the automated findings, ask the following questions that automation cannot answer:

> **If a session-based auth library or framework was detected: what is the session strategy?**
> - JWT — stateless; sessions cannot be revoked without token expiry
> - Database sessions — session stored in DB; revocable instantly
>
> *(If no session tables were found in the schema scan, JWT is the likely default — confirm before proceeding. Do not leave session strategy as implicit.)*
>
> **Are there architectural decisions in this codebase that aren't obvious from the code?**
> (e.g., "we use eventual consistency here intentionally," "this service is read-only by design")
>
> **Are there areas of the codebase that are off-limits or should not be modified by the pipeline?**
> (e.g., "don't touch the legacy payment module," "the auth layer is owned by another team")
>
> **What is the known tech debt the pipeline should be aware of?**
> (things that would affect implementation decisions or introduce constraints)
>
> **Are there external systems or APIs this codebase depends on?**
> (upstream services, third-party integrations, shared databases)
>
> **What team conventions exist that are not enforced by linting or tests?**
> (naming patterns, PR process, code review expectations)

### Step 4 — Generate draft ARCH.md

From the confirmed discovery findings and Q&A answers, generate `sdlc/architecture/ARCH.md`. Use the same structure as the ForgeKit ARCH.md template (`templates/sdlc/architecture/ARCH.md.template`), filling in all sections from the discovered information.

Security findings from Step 1 appear in a `## Pre-existing Security Findings` section — these will be picked up by the Security Review phase when a new feature thread is started.

### Step 5 — Human sign-off on ARCH.md

Present the full generated ARCH.md to the user:

> **Please review the generated ARCH.md carefully.**
>
> This document will guide every pipeline decision — roles will read it for tech-specific context, implementation commands will derive verification steps from it, and the Security Review phase will reference its findings.
>
> If anything is incorrect or missing, correct it now before the pipeline unlocks.
>
> **Approve ARCH.md?** (approve / request changes)

If changes requested: apply them and re-present. Repeat until the user approves.

**On approval:** update ARCH.md frontmatter — set `status: approved` and `approved_by: human`.

**The pipeline does not unlock until ARCH.md is explicitly approved.** Do not proceed to Phase 5 without approval.

---

## ARCH.md generation (greenfield)

For greenfield projects, generate `sdlc/architecture/ARCH.md` from the confirmed stack answers. The document should include:

- **Project overview** — name, description, purpose
- **Tech stack** — all confirmed values from Phase 4A, including a `Deploy` row (`{{stack.deploy.command}}`)
- **Module layout** — proposed structure based on framework conventions
- **Data layer** — database, migration approach, connection model
- **Authentication model** — confirmed auth approach and boundaries; if using any session-based auth approach (any library or framework that stores session state server-side), ask the user to decide now: **JWT sessions or database sessions?** Do not leave this as TBD — it is an architectural decision with security and scalability consequences. Recommendation: database sessions for auditability and revocability unless there is a specific stateless requirement
- **Testing strategy** — unit/integration/e2e split, when each type is written, no-mock-DB rule if applicable
- **Deployment** — deploy target, environment variable management approach
- **Observability** — stack, what should be instrumented, what must never be logged
- **Security conventions** — auth enforcement pattern, secrets handling, API key patterns if applicable
- **Architectural decisions** — any framework-specific patterns decided at generation time; for Next.js App Router projects, include: App Router only (no Pages Router), Server Components by default, Server Actions as the preferred mutation pattern (not just for forms — for all data writes unless a JSON API endpoint is explicitly needed for external consumers)
  <!-- FUTURE: Add equivalent framework-specific architectural guidance for other stacks (Django, Rails, Spring Boot, FastAPI, etc.). Currently only Next.js App Router gets a framework-specific callout. -->

**Frontmatter on generation:** set `status: draft` and `approved_by: pending` — do NOT set `status: approved` until the user explicitly approves.

Present for user confirmation before proceeding to Phase 5. Apply corrections if needed. Proceed only on explicit approval.

**On approval:** update the frontmatter — set `status: approved` and `approved_by: human`.

---

## Phase 5 — Generate outputs

After ARCH.md is approved, generate all outputs in this order:

### 1. Write project.yml

Create `.claude/project.yml` from the confirmed configuration. Process `# [INCLUDE ONLY IF artifact_mode: copy-paste OR hybrid]` ... `# [END INCLUDE]` markers: include the `external_tool:` block if `artifact_mode` is `copy-paste` or `hybrid`; omit it entirely for `markdown` mode.

```yaml
project:
  name: "{{project.name}}"
  description: "{{project.description}}"
  type: {{project.type}}           # greenfield or brownfield
  team_size: {{team.size}}         # solo | small | enterprise
  initialized_at: {{today}}
  forgekit_version: "1.0"

artifact_mode: {{artifact_mode}}   # markdown or copy-paste

# [INCLUDE ONLY IF artifact_mode: copy-paste OR hybrid]
external_tool:
  name: "{{external_tool.name}}"
  artifact_map:
    spec: "{{artifact_map.spec}}"
    bug: "{{artifact_map.bug}}"
    incident: "{{artifact_map.incident}}"
# [END INCLUDE]

stack:
  language: {{stack.language}}
  framework: {{stack.framework}}
  database: {{stack.database}}
  migration_tool: {{stack.migration_tool}}
  lint: "{{stack.lint}}"           # lint + type check command, or "none"
  test:
    unit: "{{stack.test.unit}}"
    e2e: "{{stack.test.e2e}}"      # or "none" if no e2e runner
  deploy:
    command: "{{stack.deploy.command}}"
    env_pull: "{{stack.deploy.env_pull}}"
    migration_command: "{{stack.deploy.migration_command}}"
  ci: {{stack.ci}}
  observability: {{stack.observability}}
  auth: {{stack.auth}}

pipeline:
  security_review: conditional
  observability: true
  postmortem: SEV-2              # SEV-1 | SEV-2 | SEV-3 | SEV-4 | always | never
```

### 2. Write or inject CLAUDE.md

**If CLAUDE.md does not exist** in the target project root: create it with the ForgeKit block only.

**If CLAUDE.md exists**: read it fully. Check for an existing ForgeKit block (a line containing `<!-- ForgeKit Pipeline`).
- If no block exists: check whether the existing content is trivial (only framework directives like `@AGENTS.md`, blank lines, or a single comment). If trivial: **replace** the file with the ForgeKit block — the block will include `@AGENTS.md` via the conditional, so no content is lost. If non-trivial (real custom instructions): **append** the ForgeKit block at the end of the existing file.
- If a block exists: this is a re-run — update the block in place, preserve all content outside the block

Generate the ForgeKit block by reading `templates/CLAUDE.md.template` and applying these substitutions and conditionals:

1. Substitute `{{project.name}}`, `{{project.type}}`, `{{artifact_mode}}`, `{{team.size}}` with the confirmed values
2. Process `<!-- IF hybrid -->...<!-- END IF -->` block: if `artifact_mode` is `hybrid`, include the block content (substituting `{{external_tool.name}}`, `{{artifact_map.spec}}`, `{{artifact_map.bug}}`, `{{artifact_map.incident}}`); otherwise remove the block and its markers entirely
3. Process `<!-- IF copy-paste -->...<!-- END IF -->` block: if `artifact_mode` is `copy-paste`, include the block content (substituting `{{external_tool.name}}`, `{{artifact_map.spec}}`, `{{artifact_map.bug}}`, `{{artifact_map.incident}}`); otherwise remove the block and its markers entirely
4. Process `<!-- IF AGENTS.md exists -->...<!-- END IF -->` block: if `AGENTS.md` exists in the target project root, include `@AGENTS.md`; otherwise remove the block and its comment markers entirely. **Exception:** if appending to an existing file that already contains `@AGENTS.md`, omit it from the ForgeKit block to avoid duplication.

Show the user a before/after diff of CLAUDE.md and confirm before writing.

Warn the user if existing CLAUDE.md content appears to contradict pipeline conventions (e.g., instructions that directly conflict with role definitions or the artifact model).

### 3. Copy PIPELINE.md

Copy `templates/PIPELINE.md` to the target project root as `PIPELINE.md`. This is a static reference document — no token substitution required. If `PIPELINE.md` already exists (re-run), overwrite it; it is always regenerated from the template.

### 4. Copy role files

Copy all files from `templates/roles/` → `.claude/roles/` in the target project. Do not overwrite role files that already exist unless the user explicitly confirms.

### 5. Generate command files

For each file in `templates/commands/`:

1. Read the template file
2. Substitute all stack values:
   - `{{stack.lint}}` → value from project.yml
   - `{{stack.test.unit}}` → value from project.yml
   - `{{stack.test.e2e}}` → value from project.yml
   - `{{stack.deploy.command}}` → value from project.yml
   - `{{stack.deploy.migration_command}}` → value from project.yml
   - `{{stack.deploy.env_pull}}` → value from project.yml
   - `{{stack.framework}}` → value from project.yml

   Artifact mode is conveyed to commands through the CLAUDE.md ambient context block, not baked into individual command files.

3. Write the substituted file to `.claude/commands/` in the target project

### 5b. Copy placeholder commands

Copy all files from `placeholder-commands/` → `.claude/commands/` in the target project. These are ForgeKit utility commands (like `/update-stack`) provided as documented stubs rather than generated from templates.

Do not overwrite `.claude/commands/forge-init.md` if it already exists in the target — `/forge-init` is a ForgeKit command, not a pipeline command.

### 6. Create sdlc/ directory structure

Create the following directories in the target project, copying the README.md from each `templates/sdlc/` counterpart:

- `sdlc/architecture/` — ARCH.md and architecture documentation
- `sdlc/analysis/` — ANALYSIS artifacts
- `sdlc/design/` — DESIGN artifacts
- `sdlc/adr/` — Architectural Decision Records
- `sdlc/plans/` — PLAN artifacts
- `sdlc/test-plans/` — TEST-PLAN artifacts
- `sdlc/security/` — SECURITY-REVIEW artifacts
- `sdlc/observability/` — OBSERVABILITY-PLAN artifacts
- `sdlc/releases/` — RELEASE artifacts
- `sdlc/release-plans/` — RELEASE-PLAN artifacts
- `sdlc/reviews/` — implementation review artifacts
- `sdlc/features/` — feature registry entries
- `sdlc/bugs/` — BUG artifacts
- `sdlc/incidents/` — INCIDENT artifacts
- `sdlc/postmortems/` — postmortem reports
- `sdlc/registry/` — master artifact index

If a directory already exists in the target project (brownfield), do not overwrite existing files — only add missing READMEs.

Create `sdlc/registry/index.md` with an empty artifact index.

### 7. Confirm and summarize

Show the user a full summary of what was created:

> **ForgeKit initialization complete.**
>
> Created:
> - `.claude/project.yml` — project configuration
> - `CLAUDE.md` — pipeline context injected [or: ForgeKit block appended to existing CLAUDE.md]
> - `PIPELINE.md` — command reference and role guide
> - `.claude/roles/` — N role files
> - `.claude/commands/` — N pipeline commands + N placeholder commands
> - `sdlc/architecture/ARCH.md` — [approved by user]
> - `sdlc/` — directory structure with READMEs
>
> [If brownfield and security findings were detected:]
> **Security findings:** N pre-existing findings recorded in ARCH.md under `## Pre-existing Security Findings`. These will surface during the Security Review phase of your first feature thread.
>
> **Next step:**
> Read `PIPELINE.md` for the complete command reference and role guide.
> [Greenfield:] Run `/create-spec` to start your first feature thread.
> [Brownfield:] Run `/create-spec` to start a feature thread, or `/create-bug` to file a defect against the existing codebase.

---

## Quality gate

**First-run:**
- [ ] project.yml written with no TBD values for fields the user answered (TBD only for fields explicitly marked unknown)
- [ ] project.yml `external_tool:` block present if `artifact_mode` is `copy-paste` or `hybrid`; absent for markdown mode
- [ ] CLAUDE.md ForgeKit block written or injected without losing existing content; no `<!-- IF ... -->` markers remain in the final file
- [ ] PIPELINE.md copied to project root
- [ ] ARCH.md approved by user — not just generated, explicitly approved
- [ ] All role files copied
- [ ] All command files generated with stack values substituted (no unreplaced `{{placeholders}}` remaining)
- [ ] All 16 `sdlc/` directories created with READMEs
- [ ] Security findings (brownfield) recorded in ARCH.md
- [ ] No existing files overwritten without user confirmation

**Re-run mode — Option B (artifact mode):**
- [ ] project.yml `artifact_mode` updated; `external_tool:` block added (hybrid/copy-paste) or removed (markdown) correctly
- [ ] CLAUDE.md ForgeKit block regenerated; no `<!-- IF ... -->` markers remain; correct mode block (hybrid or copy-paste) included, the other removed
- [ ] No files outside project.yml and CLAUDE.md were modified

**Re-run mode — Option C (project identity):**
- [ ] project.yml project section updated with changed values only
- [ ] CLAUDE.md ForgeKit block regenerated with new values; no `<!-- IF ... -->` markers remain
- [ ] No files outside project.yml and CLAUDE.md were modified

**Re-run mode — Option E (full redo):**
- [ ] User typed `REDO` to confirm — not just "yes"
- [ ] All locally modified command/role files were listed before confirmation
- [ ] ARCH.md was not touched
- [ ] sdlc/ artifacts were not touched
- [ ] PIPELINE.md overwritten from template
- [ ] All Phase 5 checks from first-run apply (except ARCH.md approval and sdlc/ directory creation for directories that already exist)

---

## Do not commit

Bootstrap does not commit. The user reviews the generated files, then commits with:

```bash
git add .claude/ sdlc/ CLAUDE.md PIPELINE.md
git commit -m "chore: initialize ForgeKit SDLC pipeline"
```
