# ForgeKit

**A governed, evidenced, artifact-driven delivery pipeline for AI-native software teams.**

---

## The problem this solves

AI can write the code. That stopped being the hard part.

The hard part is trust. When an AI generates a feature in minutes, the bottleneck shifts entirely to the human side:

- Did we describe the right thing before design began?
- Did the AI make architectural decisions we didn't notice?
- Is the process documented well enough that someone joining mid-thread can orient without starting over?
- When something discovered during implementation invalidates an upstream decision, how do we back-cycle without silently breaking everything downstream?
- If an auditor asks "how was this decision made?" — can we answer?

ForgeKit answers those questions. It turns AI-assisted development from a chat-driven activity into a governed delivery pipeline where every decision is traceable, every approval is a named human judgment, and every phase of work ends in a commit that records not just the artifact but the evidence that the right process was followed.

---

## Design principles

**The AI does the work. The human makes the decisions.**

Not "the human reviews the AI's output." That's a rubber stamp at a different point in the process. In ForgeKit, human decisions are baked into approval gates as specific, structured questions that cannot be answered reflexively. You cannot approve a spec by saying "proceed" — you have to state the problem in one sentence without implementation terms, name an edge case not covered by any AC, and confirm the Non-Goals are specific enough to use as decision rules. The gate requires real engagement.

**One source of truth. No silent drift.**

Every piece of information has exactly one authoritative home. Acceptance criteria live only in the SPEC. Observability tasks reference source IDs instead of copying descriptions. Test cases reference AC IDs instead of rewriting AC text. When an artifact changes, the information it owns changes in one place and everything else references it. Drift is structurally prevented, not just warned against.

**Process continuity across AI sessions.**

The pipeline is not dependent on one conversation staying alive. Every artifact carries frontmatter with version, status, review round, and pipeline state. `/status PLAN-NNNN` reads only frontmatter and emits a full pipeline summary with the exact next command to run — from a cold start, in under 30 seconds.

**Back-cycles are first-class, not failure states.**

Real engineering work discovers invalid assumptions. When a finding during implementation invalidates the spec, or a technical constraint invalidates the design, ForgeKit provides controlled back-cycle commands (`/reopen-spec`, `/reopen-design`, `/reopen-plan`) that capture a structured discovery record, surface blast radius before touching any file, require explicit confirmation, and propagate the right status signals so no downstream command proceeds on stale foundations.

**The git log is the audit trail.**

Each phase of work — spec, design, plan, test — accumulates in the working tree until a single approve-* command commits it. That commit captures the complete artifact: all review rounds, address summaries, and the approval itself. A commit is not just code. It is a record of the decisions made and the evidence that the right process produced them.

---

## Pipeline overview

```
Stage 1  SPEC       TPO writes feature spec → multi-perspective review → product validation
Stage 2  ANALYSIS   Embedded in review-spec; produces ANALYSIS-NNNN alongside SPEC
Stage 3  DESIGN     Architect produces options analysis → confirmed design → ADRs
Stage 4  OBS + SEC  SRE designs observability instrumentation; Security reviews threat model (parallel)
Stage 5a PLAN       Lead Developer produces ordered milestone task breakdown
Stage 5b TEST       QE produces test plan against PLAN's AC coverage map (parallel with plan approval)
Stage 6  IMPLEMENT  Developer implements one milestone at a time → multi-role review → milestone commit
Stage 7  PR         PR per milestone → review → merge
Stage 8  RELEASE    Release plan → approval → execution → validation → release notes

Back-cycle commands: /reopen-spec, /reopen-design, /reopen-plan
Operational track:   /create-bug → triage → fix through pipeline → verify
Incident track:      /create-incident → contain → resolve → /create-postmortem → action items
```

**Artifact chain** — every decision is traceable end to end:

```
SPEC-NNNN
  └─ ANALYSIS-NNNN       (spec review output)
  └─ DESIGN-NNNN         (technical spec)
       └─ ADR-NNNN       (key decisions)
       └─ OBS-NNNN       (instrumentation design)
       └─ SEC-NNNN       (security findings)
       └─ PLAN-NNNN      (implementation task breakdown)
            └─ TEST-NNNN (test contract)
            └─ IMPL-REVIEW-NNNN-MN  (per-milestone review)
            └─ RELEASE-PLAN-vX.Y.Z
```

---

## Role model

ForgeKit is role-driven, not tool-driven. Each command declares a primary role. The AI adopts that role's mental model, quality bar, and perspective for the duration of the command.

| Role | Owns |
|---|---|
| TPO (Technical Product Owner) | SPEC, product validation, back-cycle decisions on scope |
| Architect | DESIGN, ADRs, ARCH.md pattern registry |
| Security | SEC-NNNN, security test coverage |
| SRE | OBS-NNNN, instrumentation verification |
| Quality Engineer | TEST-NNNN, test execution, failure triage |
| Lead Developer | PLAN-NNNN, implementation reviews, approval |
| Developer | Source code, milestone implementation |
| DBA | Schema review embedded in design review |

---

## What makes this different from "just using AI"

| Chat-driven AI development | ForgeKit |
|---|---|
| Starts fresh each session | Artifact chain survives session boundaries |
| AI makes decisions silently | Every decision is a named human judgment at a gate |
| No traceability from spec to ship | AC IDs flow through design, plan, test, review, release |
| Back-cycles break everything downstream | Back-cycle commands capture discovery, assess blast radius, update status |
| Rubber-stamp approvals | Targeted questions require real engagement before any commit |
| Hard to audit | Single-commit-per-phase = complete phase history in git |

---

## Quick start

1. Open Claude Code in your project
2. Run `/forge-init` (copy `forge-init.md` from `.claude/commands/` if not already installed)
3. Answer setup questions: project type, stack, artifact mode, team size
4. For brownfield projects: review the automated codebase scan and approve `ARCH.md` before the pipeline unlocks
5. Read `PIPELINE.md` installed in your project root — full command reference and role guide

For a deeper guide covering solo vs. team adoption, the first-week sequence, and common stumbling blocks, see [`GETTING-STARTED.md`](GETTING-STARTED.md).

---

## Example artifacts

Not sure what "good" looks like? `templates/examples/` contains a complete fictional feature thread — a personal API token spec, implementation plan, and a two-round implementation review — written to the quality bar the pipeline enforces. Read the examples before your first real spec.

---

## What `/forge-init` installs

```
CLAUDE.md              ← ambient pipeline context
PIPELINE.md            ← command reference and role guide
.claude/
  project.yml          ← project config: stack values, artifact mode, pipeline settings
  commands/            ← 60+ pipeline commands
  roles/               ← 8 role persona files
sdlc/
  architecture/ARCH.md ← architecture pattern registry
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
  registry/            ← index.md (generated by /sync-registry)
```

## Artifact modes

**Markdown** *(default)* — all artifacts are `.md` files in `sdlc/`. Commands read and write directly. Works entirely within the repo.

**Hybrid** — SPEC, BUG, and INC live in an external tool (Jira, Linear, etc.) as the source of truth for business and operational artifacts. All technical artifacts remain as `.md` files in the repo. Best for teams where PMs own requirements in Jira and engineers own technical docs in git.

**Copy-paste** — all artifacts live in an external tool. Commands prompt you to paste content in and output content to copy back.

## Re-running `/forge-init`

Safe to re-run on an already-initialized project. Detects existing configuration and enters update mode:

```
a) Update stack values      → redirects to /update-stack
b) Change artifact mode     → updates project.yml + CLAUDE.md
c) Change project identity  → updates project.yml + CLAUDE.md
d) Refresh templates        → redirects to /upgrade-pipeline
e) Full redo                → re-answers everything; ARCH.md and sdlc/ preserved
f) Cancel
```

`ARCH.md` and all `sdlc/` artifacts are never touched by any re-run option.

## Repository layout

```
ForgeKit/
  README.md
  .claude/commands/
    forge-init.md               ← the ForgeKit init command
  templates/
    CLAUDE.md.template
    project.yml.template
    PIPELINE.md                 ← installed to project root
    commands/                   ← all pipeline command templates
    roles/                      ← role persona templates
    sdlc/                       ← artifact templates and per-directory READMEs
    examples/                   ← example SPEC, PLAN, IMPL-REVIEW showing quality bar
  placeholder-commands/         ← utility stubs (update-arch)
```

## Requirements

- [Claude Code](https://claude.ai/code) (CLI, desktop app, or VS Code / JetBrains extension)
- A project directory (git repo recommended but not required)

---

*Designed for engineering teams who want to move fast with AI without losing the traceability, accountability, and evidence trail that real software delivery requires.*
