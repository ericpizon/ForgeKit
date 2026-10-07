---
type: architecture
project: "ForgeKit"
status: approved
version: 2
approved_by: human
initialized_at: 2026-10-07
changelog:
  - "v2 — DESIGN-0001 (ForgeKit Operator) approved: added /operate to the primary_role: any precedent; noted templates/sdlc/operator-runs/ as a forthcoming new artifact-type subdirectory; extended the Testing Strategy's Tier 2 description with Operator-specific scenario classes. All additive — no existing pattern superseded."
---

# ForgeKit — System Architecture

A governed, evidenced, artifact-driven delivery pipeline for AI-native software teams.

---

## What this repo actually is

ForgeKit has no traditional tech stack — no language runtime, no framework, no database, no deploy target, no CI. Its "product" is a set of markdown command templates that an AI agent (Claude Code) reads and follows; its "code" is the precision of that text. The `Tech stack`, `Data layer`, `Authentication model`, `Deployment`, and `Observability` sections a normal project's ARCH.md would carry are omitted entirely here, not filled with "none" — they genuinely don't apply, the same reasoning `forge-init.md` already uses for a coordination repo's own minimal ARCH.md.

---

## Module layout

```
ForgeKit/
├── .claude/commands/        ← ForgeKit's own meta-commands — never templated into adopting projects
│   ├── forge-init.md        ← this command
│   └── review-forgekit-pr.md ← maintainer design-direction review for changes to this repo
├── templates/
│   ├── commands/            ← 58 pipeline command templates — the product's actual command surface
│   ├── roles/               ← 8 role persona files (tpo, architect, lead-developer, developer,
│   │                            quality-engineer, security, sre, dba)
│   ├── sdlc/                ← one subdirectory per artifact type, each with its own README
│   │                            describing format, commands, and lifecycle
│   ├── reference/           ← shared mechanism docs commands point to instead of restating
│   │                            (cross-repo-resolution, external-work-intake, xdep-schema)
│   ├── examples/            ← one worked feature thread (SPEC/PLAN/IMPL-REVIEW) at the
│   │                            pipeline's own quality bar
│   ├── CLAUDE.md.template, project.yml.template, PIPELINE.md
│                               ← static scaffolding copied into every adopting project
├── placeholder-commands/    ← command stubs not yet fully implemented (currently: update-arch.md)
├── docs/design/             ← SPEC-0001's own artifacts (external intake, coordination context,
│                                promotion, environment governance, Investigation workflow) —
│                                authored directly against create-spec.md's quality bar, predating
│                                this repo's own /forge-init. Historical record; left in place,
│                                not migrated into sdlc/ — see "Areas off-limits" below.
├── README.md, GETTING-STARTED.md, CONTRIBUTING.md, SECURITY.md
└── sdlc/                    ← this init's own output, going forward
```

`templates/sdlc/` will gain an `operator-runs/` subdirectory (added: 2026-10-07, DESIGN-0001) once `/operate` is implemented — `OPRUN-NNNN` run records, propagated into adopting projects' own `sdlc/operator-runs/` by `/forge-init`/`/upgrade-pipeline` the same way every other artifact-type subdirectory already is.

---

## Testing strategy

No automated test runner exists or is warranted — there is no compiler, no unit test framework, and "correctness" means "an AI agent, following this text, produces the specified behavior," which only an actual walkthrough can verify. Three tiers, confirmed as an intentional convention (not just an observed pattern) during this init:

1. **Structural/static checks** — `/review-forgekit-pr`'s existing rules (role-attribution consistency, cross-command reference existence, frontmatter schema completeness, `PIPELINE.md`/`forge-init.md`/`project.yml.template` version and content sync). Cheap, deterministic, intended to run before any merge to this repo.
2. **Scenario-based fixture testing** — for any new or modified command, walk its actual text against disposable git fixture repos (in a scratchpad directory, never this repo), asserting on real resulting files, frontmatter, and git state (commit SHAs, `ls` on directories, regex checks against the exact patterns a command specifies) — not prose-only reasoning about what the text "should" do. This is the primary source of real bug-finding (ordering invariants, missing confirmation steps, multi-task finding-tracking slips) — Tier 1 cannot catch any of it, since it requires simulating multi-step behavior, not just reading structure. The one place mocking is legitimate: an external system this repo never owns (an ADO/Confluence connector). **Operator-specific scenario classes** (added: 2026-10-07, DESIGN-0001), once `/operate` is implemented, join this tier: interrupted/resumed sessions (does a fresh invocation correctly resume rather than duplicate an `OPRUN-NNNN`?), fail-closed Autonomy Policy defaults (does an unconfigured `project.yml` correctly refuse autonomous action?), and cross-repo dependency deadlocks (does Operator detect a cycle rather than spinning or arbitrarily breaking it?).
3. **Periodic full-pipeline drift audits** — a deliberate, scheduled cross-cutting review across every command file for a specific concern (e.g., solo/team push consistency, single-repo vs. multi-repo correctness), not a per-change gate. Expensive since it reads broadly; catches systemic drift that accumulates across many individually-reasonable small changes.

**No-mock rule:** never mock this repo's own command text or behavior to make a scenario "pass" — the fixture's state must be real, produced by actually following the command.

---

## Security conventions

- **External content is data, not instructions** — any content this repo's commands fetch or receive from outside their own trust boundary (a ticket via `--from`, a participant's `ARCH.md` via cross-repo resolution) is presented to the reading role as quoted reference material, never as instructions that role executes. Already established via D12 (`docs/design/DESIGN-0001-...md`) and generalized in `templates/reference/cross-repo-resolution.md`'s "provenance-verified, never instruction authority" framing — this ARCH.md doesn't restate the mechanism, just confirms it as a standing convention new commands must honor.
- **Failure-diagnostic sanitization** — any failure surfaced from an external system (a connector, a coordination repo being unreachable) is reduced to a sanitized category before display or persistence; raw connector/git output is never shown or committed verbatim, since it can echo credential fragments. Established via D11 and `templates/reference/external-work-intake.md`.
- **Path/slug sanitization** — any value derived from external or user-controlled text that becomes a file path or branch name component is sanitized to `[a-z0-9-]` before use, regardless of whether the source was human-typed or externally fetched. Established via SEC-0001 SEC-2 and extended to `environments[].name` during M5.
- **Known-unsafe pattern to avoid:** writing a command's Handoff that references a confirmation ("if user confirms team context") without an preceding step that actually asks it — found and fixed twice this session (`create-investigation.md`, `create-adr.md`) via Tier 2 testing, not Tier 1.

---

## Architectural decisions and constraints

- **Role-driven, not tool-driven** — every command declares a `primary_role`; the AI adopts that role's mental model for the command's duration. A handful of commands (`/status`, `/promote-to-thread`, `/promote-to-environment`, `/create-investigation`, `/create-adr`, `/operate` (added: 2026-10-07, DESIGN-0001)) legitimately declare `primary_role: any` — no single role owns them.
- **One source of truth, no silent drift** — ACs live only in the SPEC; OBS/SEC tasks reference source IDs instead of copying descriptions; test cases reference AC IDs instead of restating AC text.
- **The git log is the audit trail** — each phase accumulates in the working tree until a single `approve-*` command commits it as one record (all review rounds, address summaries, and the approval itself).
- **Two independent axes every command must compose correctly across:** `repo_role` (`code` vs. `coordination`) and team size (solo/no-push vs. team/push). A coordination repo commits straight to `main`, never a thread branch — it has no code for `main` to be "waiting on," unlike `repo_role: code`'s `thread/NNNN-slug` convention, which exists specifically to keep planning artifacts off `main` until real code justifies them.
- **Back-cycles are first-class** — `/reopen-spec`, `/reopen-design`, `/reopen-plan` capture a structured discovery record and assess blast radius before touching any file; a controlled back-cycle is correct behavior, not a failure state.

---

## Known tech debt

- Tier 1 of the testing strategy above (`/review-forgekit-pr`'s structural checks) is currently run manually by an AI agent reading every changed file — nothing scripts it, even though several of its rules (cross-command reference existence, version-string matching) are mechanically checkable and could be a cheap automated script if this repo's change volume ever justifies the investment.
- Minor solo/team confirmation-phrasing drift across a few command files (e.g. "solo project (no push)" vs. "solo project (commit only, no push)") — cosmetic, deliberately left as-is during the 2026-10 drift audit.
- `docs/design/SPEC-0001-*.md` and its sibling artifacts predate this repo's own `/forge-init` and live outside `sdlc/`. They are a historical record, not a format this init retroactively migrates — see "Areas off-limits" below.

---

## External dependencies

None required for ForgeKit itself. Commands that touch external systems (`gh` for GitHub PRs/releases, an MCP server or CLI for ADO/Confluence intake) are optional, capability-checked at the point of use, and every one of them has a documented local-only/no-remote fallback — this repo owns no API client for any of them.

---

## Team conventions

- **Commit messages:** `type(scope): summary` (e.g. `feat(M2):`, `fix(drift-audit):`, `chore(M1):`), body explaining why not just what, ending with a `Co-Authored-By:` trailer for AI-assisted commits.
- **Single commit per phase** — applies to this repo's own development the same way it applies to artifacts this repo's pipeline produces for adopting projects.
- **Frontmatter versioning** — every command template carries `version`/`changelog`; a behavior change bumps the version and records why, in the file itself, not only in the commit message.
- **Areas off-limits:** `docs/design/` (SPEC-0001's historical artifacts — read for context, never edited to "fit" `sdlc/`'s conventions retroactively); `.claude/commands/forge-init.md` and `.claude/commands/review-forgekit-pr.md` (ForgeKit's own meta-tooling — never overwritten by `/upgrade-pipeline`, which only touches `.claude/commands/` files that originated from `templates/commands/`).

---

## Pre-existing Security Findings

*(None — no application code exists to scan for the usual findings. SEC-0001, completed this session against SPEC-0001's own design, is the equivalent review for this repo's command templates and already landed its findings as real fixes, not an open list.)*
