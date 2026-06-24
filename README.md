# ForgeKit

AI-powered SDLC pipeline for Claude Code. One `/forge-init` command gives any project a complete development workflow: spec → design → plan → implement → test → release — with role-based AI assistance at every stage and a built-in incident response track.

## What you get

After running `/forge-init` in any project:

- **60+ slash commands** covering the full SDLC — spec authoring, design review, implementation planning, test execution, release management, incident response, and postmortems
- **8 AI role personas** — TPO, Architect, Lead Developer, Developer, Quality Engineer, Security, DBA, SRE — each with a specific mental model and quality bar baked in
- **Structured artifact chain** — every decision is traceable from spec to ship; artifacts link forward and backward through the pipeline
- **Project configuration** — `project.yml` captures your stack, and commands use it for tool-specific invocations (lint, test runner, deploy command, migrations)
- **Incident and postmortem workflow** — production incidents are first-class: SEV-1 through SEV-4 severity tiers, 5 Whys root cause analysis, blameless postmortems with typed action items
- **Feature registry** — `/sync-registry` maintains a live index of all threads from spec to release
- **`PIPELINE.md`** installed in your project root — the complete command reference and role guide, always at hand

## Quick start

1. Open Claude Code in your project
2. Run `/forge-init` (if `forge-init.md` is not already in your `.claude/commands/`, copy it from ForgeKit's `.claude/commands/` directory)
3. Answer the setup questions — project type, stack, artifact mode, team size
4. For brownfield projects: review the automated codebase scan and approve `ARCH.md` before the pipeline unlocks
5. Read `PIPELINE.md` in your project root — all commands, roles, and artifacts are documented there

## What `/forge-init` installs

```
CLAUDE.md              ← ambient pipeline context; ForgeKit block injected or appended
PIPELINE.md            ← command reference and role guide
sdlc/architecture/
  ARCH.md              ← architecture reference (filled and approved during /forge-init)
.claude/
  project.yml          ← project config: stack values, artifact mode, pipeline settings
  commands/            ← 60+ pipeline slash commands
  roles/               ← 8 role persona files
sdlc/
  specs/               ← SPEC-NNNN
  analysis/            ← ANALYSIS-NNNN
  design/              ← DESIGN-NNNN
  adr/                 ← ADR-NNNN
  plans/               ← PLAN-NNNN
  test-plans/          ← TEST-NNNN
  security/            ← SEC-NNNN
  observability/       ← OBS-NNNN
  reviews/             ← IMPL-REVIEW-NNNN-MN
  release-plans/       ← RELEASE-PLAN-vX.Y.Z
  releases/            ← RELEASE-NOTES-vX.Y.Z
  bugs/                ← BUG-NNNN
  incidents/           ← INC-NNNN
  postmortems/         ← PM-NNNN
  features/            ← FEATURE-NNNN dashboards (generated)
  registry/            ← index.md global registry (generated)
```

## Artifact modes

**Markdown** *(default)* — all artifacts are `.md` files in `sdlc/`. Commands read and write directly. Works entirely within the repo.

**Hybrid** — SPEC, BUG, and INC live in an external tool (Jira, Linear, etc.) as the source of truth for business and operational artifacts. Commands prompt you to paste those artifact contents in when needed, and output them for you to copy back. All technical artifacts (DESIGN, PLAN, TEST-PLAN, SEC, OBS, IMPL-REVIEW, RELEASE-PLAN, RELEASE-NOTES, PM) remain as `.md` files in the repo. Best for teams where PMs and SREs own requirements and incidents in Jira, and engineers own technical docs in the repo.

**Copy-paste** — all artifacts live in an external tool. Commands prompt you to paste content in and output content to copy back for every artifact type. Set at initialization time; change later with `/forge-init` → option b.

## Pipeline overview

```
SPEC → DESIGN → PLAN → IMPLEMENT → TEST → RELEASE-PLAN → RELEASE
         ↓                  ↑ (repeat per milestone)
     SEC + OBS
      reviews

BUG → [triage] → [fix through pipeline] → VERIFY → close
  ↓
INCIDENT → [contain] → RESOLVE → POSTMORTEM → action items → BUG / SPEC
```

Full command reference: `PIPELINE.md` in your initialized project.

## Re-running `/forge-init`

`/forge-init` is safe to re-run on an already-initialized project. When it detects an existing `project.yml` or ForgeKit block in `CLAUDE.md`, it enters re-run mode:

```
a) Update stack values      → redirects to /update-stack
b) Change artifact mode     → updates project.yml + CLAUDE.md block
c) Change project identity  → updates project.yml + CLAUDE.md block
d) Refresh templates        → redirects to /upgrade-pipeline
e) Full redo                → re-answers everything; ARCH.md and sdlc/ preserved
f) Cancel
```

`ARCH.md` and all `sdlc/` artifacts are never touched by any re-run option.

## Repository layout

```
ForgeKit/
  README.md                    ← you are here
  .claude/commands/
    forge-init.md               ← the ForgeKit init command
  templates/
    CLAUDE.md.template         ← ambient context template
    project.yml.template       ← project config template
    PIPELINE.md                ← installed to project root
    commands/                  ← all pipeline command templates
    roles/                     ← role persona templates
    sdlc/                      ← per-directory README templates + artifact templates
      architecture/
        ARCH.md.template
      specs/ analysis/ design/ adr/ plans/ test-plans/
      security/ observability/ reviews/ release-plans/ releases/
      bugs/ incidents/ postmortems/ features/ registry/
  placeholder-commands/        ← utility stubs (update-stack, update-arch, upgrade-pipeline)
```

## Requirements

- [Claude Code](https://claude.ai/code) (CLI, desktop app, or VS Code / JetBrains extension)
- A project directory (git repo recommended but not required)
