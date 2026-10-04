---
title: Multi-Repo Coordination for ForgeKit
status: draft
created: 2026-09-16
updated: 2026-09-16
owner: Eric Pizon
---

# Multi-Repo Coordination for ForgeKit

## 1. Problem

Every ForgeKit artifact, command, and git convention assumes one repo: one `ARCH.md`, one `project.yml`, one `sdlc/` tree, and a branching model (`thread/NNNN-slug`, `feature/PLAN-NNNN-MN`) scoped entirely to that repo's own history and PRs.

A feature that requires coordinated changes across two or more repos — a producer service and a consumer service exchanging an event is the common shape — has no home in that model today. This is expected to become the *normal* case, not the exceptional one, as the underlying architecture moves toward event-driven microservices, with more than one such thread active concurrently. The design below is scoped accordingly.

**Non-goals:** this does not restructure existing repos into a monorepo, does not build a general-purpose dependency-graph product, and does not make ForgeKit a schema/contract registry. It also does not apply to, and requires no setup for, single-repo or monolithic projects — everything below activates only when a thread declares more than one participant repo (§4). A single-repo project's SPEC/DESIGN/PLAN continue exactly as they work today, with zero new machinery. A monolith's internal module boundaries in particular are not a case this design addresses at all — the coordination problem here (independent deployability, independent review authority, independent CI/release cadence) doesn't exist between modules in one repo; it's ordinary software engineering (interfaces, code review, tests), not something an XDEP should ever describe.

## 2. Vocabulary

| Term | Means |
|---|---|
| **Thread** | A cross-repo feature/workstream's identity — the thing SPEC/ANALYSIS/DESIGN describe, spanning its participant repos |
| **XDEP** | A durable, directed relationship between two participants (one provider, one consumer) — not a contract version, not an AC |
| **Contract** | The externally-owned, machine-readable interface an XDEP references (an event schema, an OpenAPI spec, a package's public surface) — ForgeKit does not own or store this |
| **Contract revision** | The specific version of a contract an XDEP currently expects — evolves within the XDEP over time via ordinary git history, not by creating a new XDEP |
| **AC** | Requirements traceability only, with respect to cross-repo dependencies — never an XDEP's identity, and never on the path that resolves a dependency to a milestone (§5) |
| **PLAN task** | A repo-local unit of work; traces to ACs (what it satisfies) and, independently, to XDEPs (what cross-repo dependency it satisfies) — see §5 |
| **PLAN / Milestone** | Repo-local: one repo's implementation of its own responsibilities, sliced into milestones however that repo's own planning process chooses |
| **Interface Freeze** | Approval of one specific, immutable revision of the Cross-Repo Contract — a sub-approval inside DESIGN, not the whole document. Has a `working` (being drafted) and `approved` (currently in force) revision, tracked separately — see §7.2 |
| **Release Plan** | The thread-wide rollout and completion state across all participants, tracked per-participant and per-environment — lives in the coordination repo, not in any one participant |
| **Cross-repo E2E scenario** | A thread-level, black-box specification of behavior spanning multiple participants — declares the XDEPs/ACs it covers, owned by no single repo. Execution is bound to an external test platform, never implemented by ForgeKit — see §11 |

## 3. Where artifacts live

### 3.1 Coordination repo vs. code repos

A dedicated **coordination repo** (no application code) holds the artifacts that describe *what is being built and why*, for every thread it covers. Each **code repo** keeps its own stack-specific artifacts and the artifacts tied to its own git history.

An org is not assumed to have exactly one coordination repo. A repo whose participants span incompatible trust or access domains (a broadly-accessible repo and a restricted confidential one, say) should use a separate coordination repo scoped to that domain rather than force one global repo to expose the union of everyone's content, or restrict itself down to the intersection. "The coordination repo" below means whichever one covers this thread's participants — not a claim that there is only ever one. A single code repo can be a participant in threads coordinated by different coordination repos at the same time — nothing pins a code repo to one coordination repo globally, only per-thread (§7.3).

```
forge-sdlc/                              <- a coordination repo, no app code
  THREAD-0142/
    thread.yml                           <- identity + participant list only, see §3.3
    SPEC.md
    ANALYSIS.md
    DESIGN.md                            <- includes the Cross-Repo Contract, §9
    SECURITY-REVIEW.md                   <- cross-service + repo-local findings, scoped per finding (§3.1)
    OBSERVABILITY-PLAN.md                <- same
    RELEASE-PLAN-v1.2.3-staging.md       <- one versioned instance per attempt/environment, see §8
    RELEASE-PLAN-v1.2.3-production.md
    SYSTEM-TEST-PLAN.md                  <- only if needed, see §11
  dependencies/
    THREAD-0142-XDEP-01.yml              <- see §5
    THREAD-0142-XDEP-02.yml
  scenarios/
    THREAD-0142-E2E-01.yml               <- only if SYSTEM-TEST-PLAN.md exists, see §11.2

transformer-service/                     <- code repo
  ARCH.md
  project.yml                            <- durable repo config only, see §7.3
  sdlc/
    plans/PLAN-0142-transformer.md       <- carries this thread's inputs, see §7
    test-plans/TEST-0142-transformer.md
    reviews/IMPL-REVIEW-0142-transformer-M1.md

document-worker/                         <- code repo
  ARCH.md
  project.yml
  sdlc/
    plans/PLAN-0142-document-worker.md
    test-plans/TEST-0142-document-worker.md
```

**Rule of thumb for what goes where:** an artifact centralizes if it describes the feature as a whole and would be wrong to fork (SPEC, ANALYSIS, DESIGN, ADRs, SECURITY-REVIEW, OBSERVABILITY-PLAN, RELEASE-PLAN, and — when the thread needs one — `SYSTEM-TEST-PLAN.md`, §11). An artifact stays in its code repo if it's inherently tied to that repo's own files, branches, and PRs (PLAN, TEST-PLAN, IMPL-REVIEW). SEC and OBS centralize specifically because their existing single-repo mechanism — author once, farm tasks/findings out into whichever PLAN implements them — already does the right thing; cross-repo just means the farming-out step also picks *which repo's* PLAN a task belongs to. Each finding/signal carries a `scope` — a list of repos it concerns, or `thread` for genuinely cross-service concerns — rather than one repo or "everything," so a finding relevant to two of six participants doesn't have to overclaim relevance to all six.

**Every repo keeps the full ForgeKit command set.** A repo that's a secondary participant in one thread can be the sole repo for its own unrelated feature next week. There is no stripped-down "participant-only" install.

### 3.2 What does *not* go in the coordination repo

The coordination repo references contracts; it does not own them. The canonical schema/interface definition (AsyncAPI, OpenAPI, Protobuf, or a shared package's public types) lives with the owning service, or in an actual schema/contract registry if the org has one (Confluent Schema Registry, Buf Schema Registry, a Backstage API catalog, etc.). An XDEP (§5) references that artifact by identity and revision — it does not duplicate it.

This mirrors Backstage's own split: Components own and implement APIs; APIs are indexed as a separate, first-class, machine-readable catalog. ForgeKit coordinates the feature-shaped work around a contract. It is not the system of record for the contract itself.

### 3.3 Reading across repos: resolve to an immutable object, not a path

**`thread.yml` does not pin its own commit.** An earlier draft had `thread.yml` record the ref of the coordination repo commit containing that very file — a self-reference that's impossible to write correctly. `thread.yml` records identity and topology only:

```yaml
thread: THREAD-0142
participants:
  - repo: org/transformer-service
  - repo: org/document-worker
```

**Every artifact that consumes a cross-repo input records exactly what it consumed**, as an `inputs` block with an explicit ref — matching ForgeKit's existing habit of tracking `target_design_version` on downstream artifacts:

```yaml
inputs:
  design:
    repo: org/forge-sdlc
    ref: 8f39c2d
    path: THREAD-0142/DESIGN.md
  cross_repo_contract:
    revision: 4              # the APPROVED revision this artifact was built against — see §7.2
    approved_ref: 8f39c2d
  architecture:
    - repo: org/transformer-service
      ref: a73911c
      path: ARCH.md
```

**Resolution reads an immutable git object, not a working tree.** Given an `inputs` entry's `{repo, ref, path}`:
1. If a local checkout of `repo` exists and has `ref` in its object database (not necessarily checked out as HEAD — just present locally), read the content with `git show <ref>:<path>`. This reads the exact committed object, so a dirty working tree or a different current branch can't produce a wrong answer.
2. Otherwise, fetch via `gh api repos/{repo}/contents/{path}?ref={ref}`.

Neither path can disagree with the other, because both resolve the same immutable ref.

**This assumes the coordination repo's history doesn't get rewritten.** A force-push that rewrites the commit an `approved_ref` (§7.2) points to would silently orphan every PLAN already pinned to it, with nothing in this design to detect that specifically — the coordination repo's default branch needs the same force-push protection any shared, depended-upon branch would need. Stated here as a requirement on the coordination repo's own git settings, not something ForgeKit's commands can enforce from the outside.

## 4. Command impact

- `create-spec` (coordination repo): new step — declare `participants` (repos) at thread creation; write `thread.yml`.
- `review-spec`: its Lead Developer perspective reads `ARCH.md` from every participant repo (via §3.3), not one — "implementation feasibility is relative to the project's actual stack" has no single answer for a cross-repo SPEC. Same fix as `create-design`, applied to the command it was missed on.
- `create-design` / `review-design` / `approve-design`: read `ARCH.md` from every participant repo (via §3.3), not one. Gains a new sub-approvable section: the Cross-Repo Contract (§9), tracked with `working`/`approved` revisions (§7.2).
- `design-observability` / `security-review`: the same "read every participant's `ARCH.md`" fix, applied to Stage 4 — an observability or security surface assessment scoped to one implicit stack misses whatever the other participants actually expose.
- `create-plan` / `review-plan` (in each code repo): resolve DESIGN, SEC, OBS from the coordination repo via §3.3; derive only the tasks/findings/signals relevant to this repo; record the `inputs` block (§3.3) on the PLAN itself. Each task that satisfies a cross-repo dependency records `satisfies_xdeps` (§5) — never resolved through the task's ACs.
- `implement-plan`: pre-flight gains a staleness check (§7.1) alongside its existing TEST-NNNN/PLAN-NNNN approval checks — a milestone whose tasks reference a now-stale `inputs.cross_repo_contract` should not start generating code against it silently.
- `create-release-plan`: authors a new versioned `RELEASE-PLAN-vX.Y.Z-{environment}.md` instance in the coordination repo (§8), resolving every participant's PLAN via §3.3, tracking per-participant and system-level state. Matches ForgeKit's existing single-repo convention — one file per version, never a mutated one — just relocated to the coordination repo.
- `review-release-plan`: reads every participant's PLAN/TEST-NNNN and the shared SEC/OBS findings scoped to it (via §3.3) — the same "read every participant, not one" extension already applied to `create-design`/`create-plan`. Also validates or overrides the proposed rollback ordering (§10.2) per participant/XDEP.
- `approve-release-plan`: requires per-participant sign-off bound to the exact candidate/environment, plus a separate thread-level coordination approval for the shared plan itself (§8.2) — not one blanket approver, and not a pure rollup either.
- `execute-release`: unchanged as a per-repo, human-triggered command, scoped to this repo's own readiness. An optional pre-flight check against the prior wave (§10.1) is available for orgs coordinating without existing deployment orchestration — not assumed universally, since an org with real cross-repo orchestration (Argo CD, Spinnaker, etc.) already owns that sequencing and shouldn't get a second, competing gate from ForgeKit.
- `execute-tests`: unchanged in scope — unit, repo-local integration/E2E, and a lightweight pre-flight for any XDEP whose `dependency_kind` is `environment` (§5), producing `PASS` / `FAIL` / `BLOCKED_ENVIRONMENT` / `ERROR` (§6). It does **not** execute cross-repo scenarios (§11.2) — a participant referencing one in `cross_repo_scenarios` means it participates in that scenario, never that its own `execute-tests` run decides the scenario's result.
- `validate-release`: consumes cross-repo scenario evidence (§11.5) from the execution binding as part of its existing observability/manual-check tracks, rather than `execute-tests` producing it locally.
- `generate-release-notes`: authors the combined release notes in the coordination repo (§3.1's centralization rule), aggregating every participant's changes — not one participant's notes standing in for the whole thread.
- `reopen-spec` / `reopen-design` / `reopen-plan`: a back-cycle must flag *every* participant's PLAN as needing revision (via §3.3), not just the repo where the discovery happened. Today's single-repo behavior only touches one PLAN's `pipeline_status`; for a cross-repo thread, silently leaving other participants building against an invalidated SPEC/DESIGN is exactly the failure mode a back-cycle exists to prevent. The discovery record (already required today) names which participants are affected, and each affected PLAN gets the same `revision_requested` treatment as the one where the discovery surfaced.
- `/status`: run in a participant repo, surfaces a cross-repo summary alongside its existing local view — the thread this repo belongs to, the Cross-Repo Contract's current `working`/`approved` state (§7.2), whether this repo's own inputs are `stale` (§7.1), and the resolution status of any XDEPs this repo's current milestone depends on (§5.1). This is the highest-value addition of the six: `/status`'s entire purpose is fast cold-start orientation for someone working alone or async, and "am I blocked on another repo, and is anyone waiting on me" is exactly the fact that's easiest to lose track of without it.
- `/sync-registry`: run in the coordination repo, produces a thread-level portfolio view across every participant (the value this command's own design always pointed at, per an earlier retrospective on this exact tool). Run in a participant repo, its local view gains a one-line pointer to thread membership and coordination-repo location (§7.3) — it does not attempt to reproduce the full cross-repo picture locally, that stays the coordination repo's job.
- `create-bug`: gains an optional `related_xdep` field, alongside the existing `linked_spec`/`linked_plan`, for a bug that's actually a cross-repo contract violation rather than a defect local to one repo.

## 5. Cross-repo dependency identity: XDEP

**Acceptance criteria describe required outcomes.** They get reworded, split, merged, renumbered, and removed as SPEC revises — this session alone demonstrated that happening repeatedly under normal review pressure. An AC is the wrong thing to hang a cross-repo dependency's *identity* on, and — this is the point an earlier draft got wrong — it's also the wrong thing to route *dependency resolution* through, even as a derived step.

### 5.1 IDs, and why AC can't be in the resolution path

A cross-repo dependency gets its own stable identity, thread-qualified: **THREAD-NNNN-XDEP-NN**, one file per dependency in the coordination repo's `dependencies/` directory.

Thread-qualifying avoids collisions *between* threads, but not between two branches concurrently working on the *same* thread (both minting `THREAD-0142-XDEP-03` independently before either merges) — and `THREAD-NNNN` allocation itself has the identical problem. Rather than switch to an opaque identifier (a break from every other sequential, readable ForgeKit ID — `SPEC-NNNN`, `PLAN-NNNN`, `ADR-NNNN`), IDs stay sequential and human-readable, with an explicit rule: **ID uniqueness is validated against the coordination repo's current HEAD at merge/approval time; a collision requires the later branch to reallocate before merging.** This is the same shape of problem as two migration files landing on the same sequence number, and the same fix applies.

Stating that as a rule isn't enough on its own — it needs an actual mechanism, not just a sentence someone might forget to act on. Concretely: a CI check on the coordination repo, run on every PR touching `THREAD-NNNN/` or `dependencies/`, that fails the PR if the proposed ID already exists on the target branch. This is the same enforcement shape a migration-numbering linter already uses in most stacks — small, mechanical, and it belongs in the coordination repo's own CI, not left as an unenforced convention.

An earlier draft had milestone resolution go `XDEP → related ACs → milestone covering those ACs`. That silently put ACs back on the critical path this design exists to remove them from — if AC-7 gets split or reworded, that derivation can break or point somewhere wrong even though neither the XDEP nor the implementing PLAN actually changed. The fix: **PLAN tasks reference XDEPs directly**, alongside and independently of the ACs they satisfy:

```yaml
tasks:
  - id: TASK-17
    description: "Publish DocumentProcessed v2"
    acceptance_criteria:
      - AC-7
    satisfies_xdeps:
      - THREAD-0142-XDEP-01
```

```
AC ─────────► PLAN task     (requirements traceability)
XDEP ───────► PLAN task     (dependency traceability)
PLAN task ──► Milestone     (implementation structure)
```

Milestone resolution for a given XDEP is now: find the tasks (in each participant's own PLAN, read via §3.3) whose `satisfies_xdeps` includes it, then find the milestone containing those tasks. ACs and XDEPs each trace independently into the task; neither routes through the other.

### 5.2 Schema

The schema is discriminated by `dependency_kind` — a `contract`-kind dependency has a `contract` block; a `deployment`-kind dependency does not, because there's no interface to describe, only an ordering constraint.

```yaml
id: THREAD-0142-XDEP-01
status: active                     # active | superseded | retired

dependency_kind: contract          # contract | artifact | deployment | environment — see §6
interface_kind: async-event        # required only when dependency_kind: contract —
                                    # async-event | rest | grpc | package

provider:
  repo: org/transformer-service
  capability: DocumentProcessed

consumer:
  repo: org/document-worker        # one XDEP = one directed edge. Fan-out (one
                                    # producer, many consumers — the normal pub/sub
                                    # shape) is multiple XDEPs sharing one contract:
                                    # THREAD-0142-XDEP-01 (-> document-worker),
                                    # THREAD-0142-XDEP-02 (-> search-indexer), etc.

# --- kind-specific payload, present only for the matching dependency_kind ---

contract:                          # dependency_kind: contract
  id: "asyncapi://document-events/DocumentProcessed"   # the durable interface identity
  revision: "2.1"                                       # the specific revision currently expected
  compatibility_required: backward

# artifact:                        # dependency_kind: artifact
#   package: org.generated.document-client
#   version: "3.x"

# deployment:                      # dependency_kind: deployment
#   prerequisite: provider          # provider | consumer — which side must exist first

# environment:                     # dependency_kind: environment
#   requires:
#     repo: org/transformer-service
#     health_check: "..."          # see §6 — reachability alone is not sufficient

verification:                      # a list, not a single mechanism — see §6.
                                    # May be empty/TBD while drafting; required
                                    # before this XDEP can be part of an approved
                                    # Interface Freeze (§7.2).
  - mechanism: schema-registry-compatibility
    gate: publish
  - mechanism: consumer-deserialization
    gate: ci

rollout:
  ordering: independent            # independent | provider-first | consumer-first
                                    # — the edges checked for cycles/waves, see §10

related_acceptance_criteria:       # traceability only — NOT identity, NOT the
  - AC-3                           # resolution path to a milestone (§5.1).
  - AC-7
```

**`related_acceptance_criteria` is a DESIGN-time hint, not a fact kept in sync with PLAN.** It's authored by the Architect before any PLAN exists, as a rough pointer for whoever picks this XDEP up later. The actual, authoritative AC association happens per-task, at PLAN time, via each task's own `acceptance_criteria` field (§5.1) — which can legitimately diverge from this list (an AC splits, a different AC turns out to be the one that actually applies) without that being an error or requiring reconciliation. If the two disagree, the task-level association wins; this field is not re-validated against it.

**XDEP identity is the durable relationship, not a pinned contract revision.** `contract.revision` evolves in place as the schema revises compatibly (2.1 → 2.2) — that's ordinary git history on the XDEP file itself, an append-only audit trail with no extra changelog field needed. A **new** XDEP is warranted only when the relationship changes in kind: a different provider, a different consumer, a different capability, or a breaking change to what's being depended on — not for routine compatible evolution, which would otherwise flood the coordination repo with superseded records for changes that were never actually a new relationship.

**Lifecycle is append-only**, matching ForgeKit's existing ADR convention:
```yaml
status: superseded
superseded_by: THREAD-0142-XDEP-09
```
```yaml
status: retired
reason: "Consumer no longer requires DocumentProcessed — see DESIGN Round 4."
```

**Field-weight note:** a first XDEP between two repos needs only `id`, `dependency_kind`, `provider`, `consumer`, and the matching kind-specific block to be useful — `verification` and `rollout` fill in as the situation demands, the same way `forge-init` accepts "TBD" for stack fields that get tightened later.

## 6. Dependency kinds and layered verification

| `dependency_kind` | What it means | Verified by |
|---|---|---|
| **contract** | Both sides must agree what an interface means before either implements against it | DESIGN review + Interface Freeze (§9) |
| **artifact** | Repo B needs a generated type, client, or published package from repo A | Build-time — codegen/package resolution, checked in CI |
| **deployment** | Producer vN must exist in an environment before consumer vN is deployed there | Release sequencing (§8, §10), not contract testing |
| **environment** | A repo's own test needs a *direct* dependency actually running — a lightweight precondition, not a formal scenario | `execute-tests`'s pre-flight (§4). For a genuinely multi-repo, multi-hop scenario, see §11 instead — it doesn't live on a single XDEP or belong to one repo |

Contract verification for a `contract`-kind dependency does **not** require a deployed environment — that would collapse it into the `deployment`/`environment` case and lose the fast-feedback property that makes contract verification worth having.

**Verification is a list of layered mechanisms, not one field, chosen per `interface_kind`.** A schema-compatibility check proves structural compatibility, not semantic or behavioral compatibility — renaming what `status: "complete"` *means* can be schema-compatible and still break every consumer.

> Every `active` `contract`-kind XDEP's `verification` list must provide automated compatibility checking appropriate to its `interface_kind` — as many layers as the risk warrants.

| `interface_kind` | Typical verification layers |
|---|---|
| async-event | Schema registry compatibility (prefer *transitive*, not just N vs. N-1), plus a semantic/behavioral check where meaning can drift without a schema change |
| rest | OpenAPI breaking-change check; provider conformance test; consumer-driven contract test where consumer assumptions exceed what the schema states |
| grpc | Protobuf compatibility/breaking-change check; integration test where behavior isn't captured by the schema |
| package | Semantic-version/API-compatibility check |

**Owned by the Architect, not Security.** Whether an `interface_kind` has an *appropriate* verification set is an architecture/contract-quality judgment, inside the Architect role's existing "boundaries and contracts" domain. Security's review is scoped to whether the security-relevant slice is covered (does a PII-bearing event's verification catch a field starting to leak sensitive data) — not general judgment about which mechanism is "right" for an interface type.

**Per-repo review alone doesn't guarantee both sides of a shared verification requirement are commensurately rigorous.** Each participant's TEST-PLAN is reviewed only by that repo's own Lead Developer/Security/QE (§3.1) — a provider team with strong test discipline and a consumer team with weaker discipline can each pass their own local review while producing very different quality implementations of the same declared verification layer: a meaningful `consumer-deserialization` check on one side, something close to a no-op that technically exists on the other. This is the same shape of gap §8.2's thread-level approval was introduced to close for release governance — a shared concern that no single participant's local review actually owns, just applied to test quality instead of release sign-off.

Half of this is already checkable without a human: does a task implementing each declared mechanism exist in the right repo's PLAN (the same farming mechanism SEC/OBS already use, §3.1), and does it reference the same pinned contract (§5.2)? Both are structural, automatable checks. Whether an implementation is actually *rigorous*, not merely present, is a judgment call this design hasn't assigned an owner to yet. The Architect — already the owner of whether an `interface_kind`'s verification set is *appropriate* — is the natural owner of whether it's *symmetrically implemented* too: reviewing both sides' actual verification implementations together for a given `contract`-kind XDEP (reading the counterpart's PLAN/TEST-PLAN via §3.3), not just confirming each side's own local review independently passed its own bar.

**`environment`-kind checks verify more than reachability, and produce a third outcome.** An HTTP 200 from a health check doesn't prove the *required version* is deployed — the old version can be healthy and wrong. The preflight verifies reachable **and** healthy **and** the expected contract/build/version, when that information is exposed. And a test blocked by an unmet environment prerequisite is not the same thing as a test that ran and failed — conflating them (or waving it away as "not a real failure") loses information CI reporting and release readiness both need. `execute-tests` produces one of:

- `PASS`
- `FAIL` — the system behaved incorrectly
- `BLOCKED_ENVIRONMENT` — a required dependency wasn't ready, with a reason (e.g., "Transformer's required revision is not deployed to this environment")
- `ERROR` — the execution mechanism itself couldn't produce a valid result (harness crash, runner failure, credential/platform unavailability) — distinct from both of the above; for gating purposes anything but `PASS` is unsatisfied, but the reason stays visible rather than collapsing into a generic failure

**This is the right mechanism for a repo's own test with a simple deployment precondition — it is not where a genuine cross-repo E2E scenario should live.** An `environment`-kind XDEP describes one repo needing one direct dependency reachable; it does not, by itself, make that repo the rightful owner of a scenario that actually spans several repos. Once a check stops being "my test, gated on your service being up" and becomes "a scenario describing behavior across the whole thread," it belongs in §11's `SYSTEM-TEST-PLAN` instead, not parked here by default.

## 7. Staleness

**Immutable refs solve consistency, not freshness.** A PLAN can be perfectly reproducible against the exact DESIGN it was approved with, and still be quietly obsolete if the Cross-Repo Contract changed the next day.

### 7.1 Stale-input detection

Every artifact with an `inputs.cross_repo_contract` block (§3.3) is checked against the *currently approved* revision at five points: `implement-plan` (before generating code against it — §4), `review-plan`, `approve-plan`, `review-implementation`, and release readiness (§8). A mismatch transitions that artifact's status to `stale`, requiring re-review before proceeding. `stale` is an expected, real state a thread passes through — not a failure, and not silently ignored either. No cross-repo webhooks are required; checking at those five existing gate points is sufficient.

### 7.2 Working and approved are separate, both revision-tracked

Binding approval to a revision number is not enough on its own — if *any* edit to an XDEP bumps the revision and immediately un-approves it, then merely *proposing* a change (before it's itself reviewed) would invalidate every PLAN downstream that's correctly built against the still-valid prior revision. The Cross-Repo Contract tracks two states, not one:

```yaml
cross_repo_contract:
  working:
    revision: 5
    status: draft            # draft | in_review | approved
  approved:
    revision: 4
    ref: 8f39c2d
```

`working` changes freely as the Architect drafts revision 5 — nothing downstream is affected while `approved` still points at revision 4. Only when `working` itself reaches `status: approved` does `approved` advance to match it:

```yaml
cross_repo_contract:
  working:
    revision: 5
    status: approved
  approved:
    revision: 5
    ref: d731ae2
```

*That* transition — not the act of drafting — is what makes every PLAN pinned to `inputs.cross_repo_contract.revision: 4` newly `stale` (§7.1).

**`working.revision` allocation has the same collision shape as XDEP/THREAD IDs (§5.1), and needs the same fix.** Two people independently drafting a competing "revision 5" on separate branches will collide exactly the way two branches minting the same XDEP number would. The same CI check that validates XDEP/THREAD ID uniqueness against the coordination repo's HEAD at merge time also validates `working.revision` — a collision requires the later branch to rebase onto the current `working.revision` before merging, not silently overwrite it.

### 7.3 `project.yml` stays durable config, not a live thread registry

An earlier draft put the expected coordination ref into a block inside each participant's `project.yml`. With many concurrent threads, `project.yml` — a file meant to change rarely — would become a frequently-edited, frequently-conflicting index of ephemeral state. `project.yml` keeps describing only durable repo configuration. The thread association is a lightweight pointer living on the repo-local PLAN itself:

```yaml
thread: THREAD-0142
coordination_repo: org/forge-sdlc
```

The coordination repo remains authoritative for thread membership.

### 7.4 Adding or removing a participant

Scope discovery mid-design is normal ("this turns out to also need a Search Indexer change") — this reuses the same working/approved machinery rather than needing its own mechanism: a participant add/remove is a change to `thread.yml` and to whichever XDEPs it affects, which bumps `working.revision` (§7.2) the same as any other Cross-Repo Contract edit, and requires Interface Freeze reapproval before the affected participants' PLANs can rely on it.

Adding a participant assumes that repo already has ForgeKit initialized (its own `ARCH.md`, the full command set per §3.1) — true for a repo pulled in from elsewhere in the org, not guaranteed for one nobody anticipated needing until mid-design. The add step should check for this and prompt `forge-init` if it's missing, rather than assume every newly-declared participant is already onboarded.

## 8. Release model

**`pr_merged` is a code-complete signal, not a deployment signal.** A cross-repo release has states that don't all compose the same way across participants — real cross-repo rollout is rarely one atomic transition, and a cross-repo feature either worked as a composition or didn't, which isn't a fact any one participant can hold on its own.

### 8.0 One versioned file per attempt, not a mutating one

ForgeKit's own single-repo convention already writes one `RELEASE-PLAN-vX.Y.Z.md` per version — never a file that gets overwritten in place. An earlier draft of this design proposed a single, unversioned `RELEASE-PLAN.md` for the coordination repo and left open whether it should mutate across environments; that wasn't just newly risky for a multi-repo release specifically (staging and production have different candidates, approvals, deployment states, and evidence — and a failed production attempt should remain a real artifact, not disappear when someone retries) — it drifted from a convention ForgeKit already had before this design touched it. The fix is to keep what already exists: `RELEASE-PLAN-vX.Y.Z-{environment}.md`, one file per version-and-environment attempt, in the coordination repo. A retried attempt at the same version gets its own suffixed file rather than overwriting the failed one.

### 8.1 Participant state and system state are different things

`implementation` and `deployment` are properties of one participant — Transformer can independently be implementation-complete and deployed. "Feature verified" is not: a cross-repo scenario either held across the actual composition of services or it didn't, and no single participant can answer that on its own. An earlier draft's own worked example never actually showed a per-participant `feature_verified` field, even though its prose claimed all states applied "per participant/overall" — the example was right, the prose wasn't. The two are split explicitly:

```yaml
release:
  target_environment: production
  state: deploying              # rolled up from participant states below

participants:
  transformer-service:
    implementation: complete      # pr_merged across scoped milestones
    deployment: complete          # live in target_environment
    deployment_validation: passed # optional — this participant's own smoke check, if useful
  document-worker:
    implementation: complete
    deployment: in_progress
  search-indexer:
    implementation: complete
    deployment: pending

system:
  deployment_ready: true          # every deployment-kind XDEP's ordering constraint is satisfiable (§10.1 derives this)
  feature_verification: pending   # see §11.5 — evidence from required cross-repo scenarios, if any
```

Four states in total, but only two of them are per-participant:
1. **Implementation complete** *(per participant)* — scoped milestones are `pr_merged`.
2. **Deployment complete** *(per participant)* — the required version is actually live in `target_environment`.
3. **Deployment ready** *(system)* — every `deployment`-kind XDEP's ordering constraint is satisfiable across all participants (§10.1).
4. **Feature verified** *(system)* — post-deploy/cross-repo integration checks pass: the existing `validate-release` smoke/manual/observability tracks, run against the whole thread, plus evidence from any required `SYSTEM-TEST-PLAN.md` scenarios (§11.5) if the thread has any.

This versioned file lives in the coordination repo (§3.1) — feature-wide, by the same rule that puts SPEC and DESIGN there. This does not block a repo's *unrelated* releases — `create-release-plan` is scoped by SPEC/PLAN ID, so repo A can ship other fixes on its own cadence freely; only a release that specifically claims this thread's feature is gated on these states.

### 8.2 Approval is per-participant *and* thread-level — not a pure rollup

The convoy fix in §9 established that one person shouldn't approve readiness for repos they don't own. It doesn't follow that overall release approval should therefore be nothing but a mathematical AND over participant approvals — deployment-wave ordering, cross-service rollback strategy, `SYSTEM-TEST-PLAN` requirements, and cross-repo E2E evidence are feature-wide concerns no participant repo owns, and a pure rollup leaves nobody explicitly accountable for them.

Approval also has the same "badge outlives what it approved" problem already fixed once for Interface Freeze (§7.2) — a bare `approved: true` says nothing about *which* candidate, *which* release-plan revision, or *which* environment was actually reviewed:

```yaml
approvals:
  participants:
    transformer-service:
      approved: true
      candidate_ref: a81f32
      release_plan_ref: 72ac91    # this RELEASE-PLAN-vX.Y.Z-{env}.md's own commit
      environment: production
      approved_by: ...
    document-worker:
      approved: false
      candidate_ref: f92e10
      release_plan_ref: 72ac91
      environment: production

  coordination:
    approved: false               # the shared plan itself — wave ordering, rollback
    release_plan_ref: 72ac91      # strategy, system verification requirements — approved
    approved_by: ...              # by SRE/Architect/whoever owns coordination for this thread

  overall: false                  # true only once every participant AND coordination are true
```

Changing a participant's candidate invalidates only that participant's approval; changing a system-wide property (rollout ordering, verification requirements) invalidates the coordination approval. The coordination approver isn't approving another team's code — they're approving the coordination plan, the same way `create-release-plan` is already an SRE-authored artifact in the single-repo world today.

## 9. Gates and the convoy problem

1. **Design gate** — the Cross-Repo Contract sub-section of DESIGN.
2. **Implementation-start gate** — falls out of the existing rule that `create-plan` requires an approved input. With two repos, "wait for the whole DESIGN to be approved" is a clean, free gate. **With six repos it becomes a liability** — one unresolved design question in repo F's section blocks repos A–E from planning, even though nothing in their own work depends on it.

**Fix: the Cross-Repo Contract is a separately-approvable sub-section, ahead of the rest of DESIGN**, using the working/approved split from §7.2:

```
Shared DESIGN
  │
  ├── Cross-Repo Contract     approved.revision advances  ──► every participant's
  │     (XDEPs, compatibility     (§7.2)                      create-plan may begin,
  │      policy)                                              recording inputs.cross_repo_contract
  │
  └── Remaining repo-local detail   status: draft   ──► continues independently
```

This is a real change to `create-design`, `review-design`, and `approve-design`, not a footnote — today they track one `status` field for the whole document. `create-plan`'s pre-flight check becomes "the Cross-Repo Contract's `approved` revision covers what this repo needs" rather than "DESIGN is approved," for any participant whose PLAN only depends on XDEPs.

## 10. Rollout ordering: a design-time check and a documented artifact, not an execution gate

Two earlier drafts of this got it wrong in opposite directions — one filed cycle detection under "deferred" entirely, the other had ForgeKit *enforce* the derived order by gating `execute-release` on it. Neither is right, and the second one is inconsistent with a boundary this design holds everywhere else (§3.2, §11.3, §11.6): ForgeKit references and documents; it doesn't own execution. Most orgs coordinating more than a couple of services already have real deployment orchestration — Argo CD's own sync-wave concept, a Spinnaker pipeline, Harness, reusable GitHub Actions workflows — that already sequences dependency-aware rollout. A second, independently-computed gate inside `execute-release` doesn't help there: at best it's redundant with what the orchestrator already enforces, at worst it's a second source of truth that can disagree with the first.

What's actually needed splits into two different things with two different owners:

**Cycle detection is a design-time correctness check**, not a release-time one. Given `deployment`-kind XDEPs `A provider-first B`, `B provider-first C`, `C provider-first A`, there is no valid deploy order at all — that's a real design defect worth catching during DESIGN review (§9), before any implementation starts, not discovered for the first time at release. This is a small, scoped algorithm, not a graph product:

- Build the rollout edges from `deployment`-kind (and `contract`-kind with a non-`independent` `rollout.ordering`) XDEPs.
- Detect cycles among *only* those edges.
- Where acyclic, derive deployment waves (a topological sort) — e.g., Wave 1: transformer-service; Wave 2: document-worker, search-indexer; Wave 3: notification-service.

**This check is deliberately narrow.** A bidirectional API relationship (A calls B's API, B consumes A's event) is legitimate architectural coupling, not a rollout problem, and must not trip this check — only edges that actually constrain deployment *order* are in scope.

**Deriving waves where the graph is acyclic produces a documented artifact, recorded in `RELEASE-PLAN-vX.Y.Z-{environment}.md` (§8) as the `system.deployment_ready` computation — not an enforcement mechanism.** It answers "what order should these deploy in" for whoever or whatever actually runs the deploys.

### 10.1 Consuming the derived order is a per-org choice, not a default gate

The wave order is *always* computed and recorded — that costs nothing and is useful regardless. Whether anything *enforces* it is a choice this design deliberately doesn't make for every org:

- **No existing cross-repo deployment orchestration** (a couple of repos, coordinated manually): the derived order is a genuinely useful manual-coordination aid. `execute-release`'s pre-flight *may* optionally check the prior wave's `deployment: complete` state (§8.1) before proceeding — worth having specifically because nothing else is checking it.
- **Existing cross-repo deployment orchestration** (Argo CD, Spinnaker, Harness, a custom pipeline): the derived wave order is an *input* to that tool's own configuration — translated into Argo CD sync-wave annotations, Spinnaker stage ordering, or equivalent — not something ForgeKit also enforces at `execute-release` time. `execute-release`'s pre-flight stays scoped to this repo's own readiness (tests, security, its own approval per §8.2), exactly as it is today; cross-repo sequencing is the orchestrator's job, not ForgeKit's second opinion on it.

### 10.2 Rollback: the same principle, the same caveat as ordering

Rolling back the wrong repo first can break things exactly the way deploying in the wrong order does — rolling back Transformer while its consumers still expect the new contract is the same class of failure as deploying those consumers before Transformer existed. Reverse wave order is a useful default *proposal*, not a correctness rule: a schema-expand migration deployed in Wave 1 may be deliberately forward-only (rolling it back would be worse than leaving it), while a purely additive, backward-compatible change in a later wave might roll back safely on its own.

ForgeKit's contribution is the same as forward ordering: derive the proposal, record it in `RELEASE-PLAN-vX.Y.Z-{environment}.md`, and let `review-release-plan` (§4) explicitly validate or override it per participant/XDEP — an irreversible or forward-fix-only change is represented explicitly, not silently rolled back because the graph said so. Whether that proposal is executed by a human following a runbook or by a rollback mechanism the org's own progressive-delivery tooling already provides is, like §10.1, not something this design assumes either way.

## 11. Cross-repo integration and E2E testing

Three shapes of "integration testing" once a feature spans repos, and only two are resolved by anything above:

1. **Repo-local integration/E2E** — unaffected, with one prerequisite fix (§11.7) once participants plausibly run different CI systems. Each repo otherwise tests its own stack against its own real dependencies as it always has.
2. **Cross-repo contract-level verification** — already covered by §6's layered `verification` mechanisms on a `contract`-kind XDEP, deliberately designed to run without a live deployed counterpart.
3. **Genuine cross-repo E2E — a scenario that needs two or more services actually deployed and running together** — addressed here.

### 11.1 Centralize the scenario specification, not the executable test

A cross-repo E2E scenario describes system behavior, not one repository's implementation ("given a document is submitted, when Transformer converts it and publishes DocumentProcessed, and Indexer consumes it, then the document becomes searchable") — that belongs with the feature as a whole, by the same centralization rule as SPEC/DESIGN (§3.1).

What centralizes is the *specification*: input, externally observable outcome, participating capabilities, required XDEPs, test-data requirements, environment requirements, timeout/eventual-consistency expectations. It should not need to know that Transformer is .NET, Indexer is Node, or which test runner either one uses — if a "cross-service" scenario needs that level of detail, that's a sign part of it actually belongs lower in the test pyramid as a repo-local integration test instead.

The *executable implementation* is a separate concern, and it belongs outside the coordination repo entirely, with no exception for simple cases. The coordination repo is a governance/control-plane repo; the moment it holds any executable test code, it also holds that code's package dependencies, runtime version, credentials, test-data utilities, and framework upgrades — at that point it's a test-application repo wearing a control-plane repo's name, and there's no clean, forcing moment later to notice that happened and move it out. If a scenario is simple enough to be fully declarative, it stays declarative in the scenario YAML (§11.2) with no implementation reference at all. If it needs actual code, that code lives in an executor-owned repo (`org/system-tests`, or wherever the org's test infrastructure already lives) from day one:

```yaml
implementation:
  repo: org/system-tests
  ref: e912abc
  path: document-processing/searchable-document.spec.ts
```

A new artifact holds the specifications, conditional on the thread actually needing one — a thread whose dependencies are all fully covered by contract verification (§6) doesn't need this artifact at all, the same way OBS/SEC are conditionally triggered rather than mandatory:

```
forge-sdlc/
  THREAD-0142/
    ...
    SYSTEM-TEST-PLAN.md      <- only if the thread has scenarios needing live deployment
```

**`SYSTEM-TEST-PLAN.md` and the scenario YAML files (§11.2) have different jobs — only one of them is the source of truth for any given fact.** `SYSTEM-TEST-PLAN.md` is prose: strategy, scope, rationale, which critical workflows this thread verifies at the system level and why. It names scenarios by ID; it does not restate their field-level details. The scenario YAML is the sole canonical definition of the scenario itself — timeout, entrypoint, expected outcome, coverage. Without this split, the Markdown and the YAML will eventually disagree (one says `timeout: 60s`, the other `120s`), and nothing says which one is real.

### 11.2 Scenario identity: thread-level, not XDEP-level

An earlier draft proposed `TC-XDEP-*`, reusing the identity shape of ForgeKit's existing `TC-ADR-*` invariant tests. The propagation mechanic (a central requirement, referenced from participant TEST-PLANs) is worth reusing — the identity granularity is not. A `TC-ADR-*` test is roughly one ADR to one or a few tests, scoped inside one repo. A real E2E scenario doesn't have that shape: one scenario can cover multiple XDEPs across multiple hops (upload → convert → index → search spans two XDEPs in one flow); one XDEP can need several scenarios (happy path, unsupported format, retry, duplicate event); and most XDEPs need none, because contract verification already covers them.

Scenarios get their own thread-level identity instead. No `scenario_revision` counter is needed — the coordination repo commit that contains the scenario file already gives it immutable identity, the same as any other artifact in this design; a scenario doesn't have the convoy-gate reason Interface Freeze (§7.2) had to need a separate working/approved split, so it doesn't need one either.

```yaml
id: THREAD-0142-E2E-01
status: active                     # active | quarantined | retired

scenario:
  entrypoint: submit-document
  expected_outcome: document-searchable

covers_xdeps:
  - THREAD-0142-XDEP-01
  - THREAD-0142-XDEP-02

covers_acceptance_criteria:
  - AC-4
  - AC-7

implementation:
  repo: org/system-tests
  ref: e912abc
  path: document-processing/searchable-document.spec.ts

owners:
  - team: document-platform
  - team: search-platform
triage_owner: document-platform
```

**A centralized scenario nobody owns rots the way every unowned shared test suite does — flaky, ignored, eventually distrusted by everyone who depends on it.** `owners`/`triage_owner` name who's responsible for flaky-test investigation, scenario updates, implementation maintenance, and failure triage. Avoiding an arbitrary *primary repo* (§11.1) doesn't mean avoiding an owner — those are different problems, and this design solves the first without leaving the second unaddressed.

**`status: quarantined` requires explicit risk acceptance, the same way an accepted-risk SEC finding does — a required scenario doesn't get to silently become optional.** Quarantining a scenario that's required for release (per its execution binding's `triggers`, §11.3) must be recorded as an accepted risk with a named approver and a revisit condition, following the exact convention already established for SEC findings and design-review accepted risks elsewhere in ForgeKit — not a quiet exclusion from the gate.

Participant PLANs/TEST-PLANs reference scenarios rather than duplicate them — this means the repo *participates in* the scenario, never that its own `execute-tests` run is what executes or decides it (§4, §11.3):

```yaml
cross_repo_scenarios:
  - THREAD-0142-E2E-01
```

### 11.3 Execution binding, not an execution platform

ForgeKit needs to know a scenario must run somewhere, when, and what the result means — not how it's provisioned, scheduled, or run. The minimal abstraction is an execution binding: a reference to wherever the actual mechanism lives, plus the trigger policy and result contract:

```yaml
environment:
  profile: cross-repo-preview

execution:
  binding: org/test-platform/document-e2e
  revision: workflow@7fa82c          # which version of the executor produced a given result
  triggers:
    release: required
    nightly: required
    boundary_pr: optional

result:
  allowed:
    - PASS
    - FAIL
    - BLOCKED_ENVIRONMENT
    - ERROR                          # see §6 — the harness itself failed, distinct from FAIL
```

ForgeKit does not need to know whether `cross-repo-preview` means a shared staging environment, an ephemeral Kubernetes namespace, a service-mesh-routed sandbox, or something the org hasn't built yet — that's infrastructure the binding hands off to (GitHub's own cross-repository reusable workflows, pinned to a ref, are one lightweight way this could work without inventing a scheduler). This is the same "reference, don't own" boundary already drawn for the schema/contract registry (§3.2). Cross-repo scenario execution is never routed through a participant's own `execute-tests` (§4) — that would quietly reintroduce the arbitrary-primary-repo problem this whole design exists to avoid, just at the test layer instead of the planning layer. `validate-release` (or a lightweight dedicated check) consumes the evidence this binding produces; it doesn't run the scenario itself.

### 11.4 Revision sets belong to the execution request, not the scenario

**Scenario = what must be proven. Execution = which versions were tested.** These are different things, and conflating them was a mistake in an earlier draft. Knowing a scenario must run isn't enough — running it against the wrong versions produces false results: testing an in-flight Transformer PR against baseline Indexer fails if the coordinated Indexer change hasn't merged yet, for reasons that have nothing to do with whether the Transformer PR is correct. The same scenario (`THREAD-0142-E2E-01`, one stable identity) can execute against a PR cohort, a nightly baseline, a staging release candidate, or a production release — its identity never changes; only the revision set under test does, generated fresh per execution request:

```yaml
revision_set:
  transformer-service:
    ref: PR-183/a81f32
  document-worker:
    ref: PR-491/f92e10
  search-indexer:
    ref: baseline
```

For release validation and nightly runs this is easy (specific released versions, or "current deployed baseline"). For a `boundary_pr` trigger testing coordinated in-flight changes together, composing the right revision set is exactly the problem ephemeral-environment/sandbox-composition tooling is built to solve — ForgeKit names the requirement as part of the execution request; it does not implement the composition.

### 11.5 Evidence is self-contained and revision-bound

A required run produces a small record ForgeKit references for release-gating — not the full log/trace/screenshot history, which stays with the execution platform. It has to answer, on its own, "exactly what test code produced this result" — which means it pins every ref that went into producing it, the same `{repo, ref, path}` discipline §3.3 already established for DESIGN/PLAN inputs, applied here instead of loosely referencing a "revision number" with no defined lifecycle:

```yaml
scenario:
  id: THREAD-0142-E2E-01
  repo: org/forge-sdlc
  ref: c812fea
  path: scenarios/THREAD-0142-E2E-01.yml

implementation:
  repo: org/system-tests
  ref: e912abc
  path: document-processing/searchable-document.spec.ts

executor:
  binding: org/test-platform/document-e2e
  revision: workflow@7fa82c

revision_set:
  transformer-service: a81f32
  document-worker: f92e10

target_environment: production
result: PASS
evidence:
  run: "https://..."
```

**Evidence is only eligible to satisfy a release requirement if it matches, exactly, on every dimension — not merely "a recent PASS exists for this scenario ID."** Without this stated explicitly, an implementation can quietly accept stale evidence: a 9am nightly `PASS` does not satisfy an 11:30am release gate if Transformer v42 and Indexer v19 deployed at 11:00 and 11:30 — that evidence tested the wrong revision set, regardless of how recently it ran. The eligibility check, in order:

```
PASS exists
     │
     ▼
right scenario (ref-pinned)?
     │
right test implementation (ref-pinned)?
     │
right service revisions (revision_set matches the actual candidate/deployed set)?
     │
right target_environment?
     │
occurred after those deployment prerequisites became effective?
     │
     ▼
eligible release evidence
```

`RELEASE-PLAN-vX.Y.Z-{environment}.md`'s system-level `feature_verification` state (§8.1) points at eligible evidence rather than re-deriving it — the run itself isn't committed into git.

### 11.6 What ForgeKit explicitly does not own here

Runner pools, environment provisioning and lifecycle, secrets, test-data isolation, scheduling, parallelization, resource quotas, log retention, and cleanup are infrastructure-platform concerns, not an SDLC planning tool's. ForgeKit's responsibility stops at: declare that a scenario exists and what it must prove, declare which XDEPs/ACs it covers, declare when it's required, bind it to an external executor, and consume the evidence that comes back. Building any of the above inside ForgeKit would be the same mistake as building a schema registry or a dependency-graph platform (§3.2, §12) — solving a real problem by taking on infrastructure this tool has no business owning.

### 11.7 A prerequisite outside this design: heterogeneous CI triggering

Shape 1 (repo-local tests) is unaffected by everything else in this document, with one real exception once participants plausibly use different CI systems — far more likely across repos owned by different teams than within one team's single repo. `execute-tests` today detects CI narrowly: it checks whether `.github/workflows/` exists and triggers it via `gh workflow run`, falling back to running tests locally otherwise. That doesn't generalize to a participant running Jenkins, CircleCI, or anything else — it would silently run that repo's tests locally instead of actually triggering its real pipeline.

This predates multi-repo and isn't something this design introduces — but multi-repo is what turns it from a theoretical gap into something that will actually happen. The fix is the same generalization `project.yml` already applies to `stack.test.unit` and `stack.deploy.command`: a plain runnable command, not an assumption baked in about which tool produced it. Extend `stack.ci` from a descriptive label into a `trigger_command` (`gh workflow run test.yml`, a Jenkins CLI invocation, or `none` meaning always run locally), and have `execute-tests` run that instead of a hardcoded GitHub-Actions-specific path. A thread with heterogeneous-CI participants depends on this being fixed to work correctly, even though the fix itself belongs to ForgeKit's existing single-repo command, not to new multi-repo machinery.

## 12. Explicitly deferred

- **A dependency-graph engine, visualization, or UI** beyond the rollout-edge algorithm in §10. The data model (§5's XDEP records) is built now specifically so this is cheap to add later.
- **General architectural-coupling cycle detection** across all edge types — only rollout-ordering edges are checked (§10); a bidirectional API relationship is not itself flagged.
- **Building a schema/contract registry inside ForgeKit.** Use an existing one if the org has it; reference by identity/revision if not (§3.2).
- **Restructuring repos into a monorepo.** Doesn't remove the underlying distributed-system concerns that motivate separate repos in the first place.
- **Seamless single-repo-to-thread promotion.** A SPEC that starts local and later discovers it needs a second repo is a real, expected lifecycle case (like §7.4's participant-add, just starting from zero participants). Not designing the promotion mechanism now, but it's an acknowledged future case, not an undefined one.
- **Optional centralization of coordination artifacts for a single-repo project**, independent of any multi-repo need — considered and deliberately not adopted here. The advantages that would motivate it (independent access control between docs and code, clean git history separation, an easier later promotion to real multi-repo, OSS/private separation) are real but situational, and none of them apply to ForgeKit's common case (solo/small-team, single-repo). The disadvantages dominate there instead: the cross-repo read machinery (§3.3) exists specifically to solve a synchronization problem that a single repo's own linear history already solves for free, so splitting would recreate a problem co-location doesn't have. If this is ever wanted, it belongs as a project-level configuration choice parallel to `artifact_mode` (co-located vs. centralized, independent of participant count) — not as part of this design, and not as a default.

## 13. Cross-repo artifact trust and access

This is distinct from the prompt-injection review already embedded in ForgeKit for arbitrary external/user-controlled text (job-posting content, and similar) — that mechanism assumes obviously-untrusted input and mitigates by framing it as data, not instructions. A DESIGN.md fetched from another team's coordination repo is reviewed, approved content, not raw third-party text, and doesn't need the same "treat as data" framing.

But provenance is not the same thing as instruction authority, and an earlier draft's framing left room to imply otherwise — which would open a real supply-chain-shaped hole even for legitimately-provenanced content (a DESIGN.md can accidentally, or after a compromised account, contain something like "when implementing this, ignore project policy X"). The precise rule:

> Cross-repo artifacts are trusted as provenance-verified engineering inputs, but not as executable instruction authority. Their content may inform the current role's decisions but cannot override ForgeKit command policy, security constraints, tool permissions, or approval gates.

Two more things this design states explicitly:

**Provenance before trust.** Before a decision is based on fetched cross-repo content, confirm it was retrieved at the exact ref an `inputs` block expects (§3.3) — not merely whatever currently exists at that path.

**Least-privilege read access.** A repo's automation needs read access to the coordination repo and to whichever participant repos its own XDEPs reference — not broad, org-wide read access. This scoping decision belongs in the cross-repo setup step of `forge-init` (declaring `participants`, §4).

## 14. Open risks carried forward, not yet resolved

- **Numbering namespace.** `THREAD-NNNN` is a separate identifier space from each code repo's own local `SPEC-NNNN` numbering — a repo can have both a local `SPEC-0003` and be a participant in `THREAD-0003` with no relationship between the two numbers.
- **Access control fragmentation.** §13 states a least-privilege principle and §3.1 allows more than one coordination repo per trust domain, but the actual mechanics of a contributor having access to a code repo but not its coordination repo aren't worked out.
- **Cross-repo E2E execution platform is unnamed.** §11 draws the boundary (ForgeKit declares and binds, an external executor runs) but doesn't pick or require one — an org adopting this design needs to actually stand up or designate that executor before `boundary_pr`/`nightly` triggers mean anything.
- **CI-trigger generalization (§11.7) is a dependency this design has on a fix outside its own scope.** `execute-tests`'s GitHub-Actions-specific CI detection needs to become a configurable `trigger_command` before a thread with heterogeneous-CI participants can rely on it — until then, a participant on Jenkins/CircleCI/etc. will silently run tests locally instead of triggering its real pipeline.
- **Verification-symmetry review (§6) has an assigned owner (the Architect) but no assigned mechanism or cadence yet** — whether it happens once per XDEP at PLAN-review time, is re-checked on every revision, or something else, isn't decided here.
- **Organizational cost.** Requiring a jointly-approved Cross-Repo Contract before independently-owned teams can start is a real coordination cost. The Interface Freeze (§9) mitigates this for planning but doesn't eliminate the cost of reaching agreement in the first place.
- **Provisioning friction.** A coordination repo is cheap in git terms; standing one up may not be cheap in every org's process terms.

None of these block the design above, but none should be treated as solved either.
