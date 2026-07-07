# ForgeKit Pipeline

Command reference for the AI-SDLC pipeline installed in this project. Each command guides you through the step and produces the artifact. Stages build on each other — work the chain.

## Feature pipeline

| Stage | Step | Command | Primary role | Produces |
|---|---|---|---|---|
| **1 — Specification** | Write | `/create-spec` | TPO | SPEC-NNNN |
| | Review | `/review-spec SPEC-NNNN` | Lead Dev | feedback |
| | Revise | `/revise-spec SPEC-NNNN` | TPO | — |
| | Validate | `/validate-spec SPEC-NNNN` | TPO | product validation sign-off |
| | Approve | `/approve-spec SPEC-NNNN` | TPO | SPEC → approved |
| **2 — Spec review / Analysis** | *(runs during Stage 1 review)* | | Lead Dev | ANALYSIS-NNNN |
| **3 — Architecture & Design** | Author | `/create-design SPEC-NNNN` | Architect | DESIGN-NNNN |
| | Review | `/review-design DESIGN-NNNN` | Architect (multi-perspective) | feedback |
| | Address feedback | `/address-design-review DESIGN-NNNN` | Architect | — |
| | Approve | `/approve-design DESIGN-NNNN` | Architect | DESIGN → approved |
| **4a — Observability design** | Author | `/design-observability DESIGN-NNNN` | SRE | OBS-NNNN |
| **4b — Security review** | Author | `/security-review SPEC-NNNN DESIGN-NNNN ANALYSIS-NNNN` | Security | SEC-NNNN |
| **5a — Implementation planning** | Author plan | `/create-plan DESIGN-NNNN SPEC-NNNN OBS-NNNN SEC-NNNN` | Lead Dev | PLAN-NNNN |
| | Review plan | `/review-plan PLAN-NNNN SPEC-NNNN DESIGN-NNNN SEC-NNNN` | Lead Dev | feedback |
| | Address plan feedback | `/address-plan-review PLAN-NNNN` | Lead Dev | — |
| | Approve plan | `/approve-plan PLAN-NNNN` | Lead Dev | PLAN → approved |
| **5b — Test planning** | Author test plan | `/create-test-plan PLAN-NNNN` | Quality Engineer | TEST-NNNN |
| | Review test plan | `/review-test-plan TEST-NNNN` | Quality Engineer | feedback |
| | Address test plan feedback | `/address-test-plan-review TEST-NNNN` | Quality Engineer | — |
| | Approve test plan | `/approve-test-plan TEST-NNNN` | Quality Engineer | TEST → approved |
| **6–7 — Implementation** | Implement milestone | `/implement-plan PLAN-NNNN M1` | Developer | code |
| | Run tests | `/execute-tests PLAN-NNNN M1 TEST-NNNN` | Developer | test output |
| | Fix failures | `/address-test-failures PLAN-NNNN M1` | Developer | — |
| | Review code | `/review-implementation PLAN-NNNN M1` | Lead Dev | IMPL-REVIEW |
| | Address review | `/address-implementation-review PLAN-NNNN M1` | Developer | — |
| | Approve code | `/approve-implementation PLAN-NNNN M1` | Lead Dev | milestone → approved |
| | Open PR | `/create-pr PLAN-NNNN M1` | Developer | PR |
| | Close milestone | `/close-milestone PLAN-NNNN M1` | Developer | milestone → pr_merged |
| **8 — *(reserved)*** | | | | |
| **9 — Release planning** | Create plan | `/create-release-plan vX.Y.Z` | SRE | RELEASE-PLAN |
| | Review plan | `/review-release-plan vX.Y.Z` | Lead Dev | feedback |
| | Address feedback | `/address-release-plan-review vX.Y.Z` | SRE | — |
| | Approve plan | `/approve-release-plan vX.Y.Z` | SRE | plan → approved |
| **10 — Release** | Validate | `/validate-release RELEASE-PLAN-vX.Y.Z TEST-NNNN OBS-NNNN` | SRE | validation report |
| | Execute | `/execute-release RELEASE-PLAN-vX.Y.Z` | SRE | deployed |
| | Publish notes | `/generate-release-notes RELEASE-PLAN-vX.Y.Z` | TPO | RELEASE-NOTES |

Repeat stages 6–7 for each milestone (M1, M2, M3 …) until all milestones reach `pr_merged`, then proceed to Stage 9.

Stages 4a and 4b run in parallel after DESIGN is approved. Both must complete before Stage 5a begins.

## Back-cycle

When a downstream discovery invalidates an upstream artifact, use a back-cycle command to reopen it. These capture a structured discovery record, surface blast radius before touching any file, require explicit confirmation, and propagate the correct status signals so no downstream command proceeds on stale foundations.

| When | Command | Primary role |
|---|---|---|
| Spec invalidated by downstream discovery | `/reopen-spec SPEC-NNNN` | TPO |
| Design invalidated by downstream discovery | `/reopen-design DESIGN-NNNN` | Architect |
| Plan invalidated by downstream discovery | `/reopen-plan PLAN-NNNN` | Lead Dev |

A back-cycle is not a failure — it is the correct response when a finding at one stage reveals that an upstream decision no longer holds.

## Bugs

| When | Command | Primary role |
|---|---|---|
| Defect found | `/create-bug` | Anyone |
| Prioritize and assign | `/triage-bug BUG-NNNN` | Lead Dev |
| Verify fix is complete | `/verify-bug BUG-NNNN` | Quality Engineer |
| Deploy a hotfix | `/release-hotfix BUG-NNNN` | Lead Dev |
| Bug causing active production impact | `/escalate-to-incident BUG-NNNN` | SRE |

## Incidents

| When | Command |
|---|---|
| Active production issue, no existing bug | `/create-incident` |
| Active production issue, bug already filed | `/escalate-to-incident BUG-NNNN` |
| New development to log during an incident | `/update-incident INC-NNNN` |
| Incident is contained | `/resolve-incident INC-NNNN` |
| Postmortem required or warranted | `/create-postmortem INC-NNNN` |

See `sdlc/incidents/README.md` for incident severity criteria (SEV-1 through SEV-4).

## Maintenance

| When | Command |
|---|---|
| Orient after a session break — see full pipeline state and next action | `/status [PLAN-NNNN]` |
| Refresh the feature registry index | `/sync-registry` |
| Update stack values (test runner, deploy command, etc.) | `/update-stack` |
| Update architecture reference | `/update-arch` *(placeholder)* |
| Re-configure or refresh pipeline templates | `/forge-init` |

## Roles

Each command lists a primary role — that role's mental model is most relevant for that step. Invoke a role by asking Claude to approach the task from that perspective.

| Role | Identity | Owns |
|---|---|---|
| **TPO** | Technical Product Owner | Spec authoring, spec approval, release notes |
| **Architect** | System designer | Design, design approval, ADRs, technical decisions |
| **Lead Developer** | Implementation lead | Plans, plan approval, code review, milestone approvals |
| **Developer** | Implementer | Code, tests, PRs |
| **Quality Engineer** | Test owner | Test plans, test review, test approval, bug verification |
| **Security** | Security reviewer | Threat modeling, security review |
| **DBA** | Database specialist | Schema design, migration strategy |
| **SRE** | Reliability engineer | Observability design, release planning, release approval, incident response, postmortems |

Role files live in `.claude/roles/`. Reference one with `@.claude/roles/[role].md` in a prompt, or ask Claude to take on that role by name.

## Artifacts

In **markdown mode** (default), all artifacts are `.md` files in `sdlc/`. Commands read and write directly. In **hybrid mode**, SPEC, BUG, and INC live in an external tool — see CLAUDE.md for paste-in/copy-out instructions. Each directory has a `README.md` explaining its artifact format and lifecycle.

| Directory | Artifact | ID format |
|---|---|---|
| `sdlc/specs/` | Feature specifications | SPEC-NNNN |
| `sdlc/analysis/` | Spec analysis | ANALYSIS-NNNN |
| `sdlc/design/` | Architecture designs | DESIGN-NNNN |
| `sdlc/adr/` | Architectural decision records | ADR-NNNN |
| `sdlc/plans/` | Implementation plans | PLAN-NNNN |
| `sdlc/test-plans/` | Test plans | TEST-NNNN |
| `sdlc/security/` | Security reviews | SEC-NNNN |
| `sdlc/observability/` | Observability plans | OBS-NNNN |
| `sdlc/reviews/` | Implementation reviews | IMPL-REVIEW-NNNN-MN |
| `sdlc/release-plans/` | Release plans | RELEASE-PLAN-vX.Y.Z |
| `sdlc/releases/` | Release notes | RELEASE-NOTES-vX.Y.Z |
| `sdlc/bugs/` | Bug reports | BUG-NNNN |
| `sdlc/incidents/` | Incident records | INC-NNNN |
| `sdlc/postmortems/` | Postmortems | PM-NNNN |
| `sdlc/features/` | Feature dashboards *(generated by /sync-registry)* | FEATURE-NNNN |
| `sdlc/registry/` | Global artifact index *(generated by /sync-registry)* | index.md |
