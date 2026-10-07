# XDEP and cross-repo scenario schema

Shared reference for the data model behind cross-repo dependencies and E2E scenarios. Referenced by `create-design`/`review-design` (which author and review XDEPs), `create-plan`/`review-plan` (which resolve them to milestones), and `create-test-plan` (which authors cross-repo scenarios) — not a command itself.

## Why acceptance criteria are never an XDEP's identity

Acceptance criteria describe required outcomes. They get reworded, split, merged, renumbered, and removed as a SPEC revises — completely normal under real review pressure. A cross-repo dependency (a specific contract, at a specific revision, between a specific provider and consumer) is not the same thing as the outcome an AC describes, and an AC's own churn should never invalidate a dependency that hasn't actually changed. This applies twice over: an AC is not an XDEP's *identity*, and it is not on the *resolution path* that finds which milestone implements a dependency either — PLAN tasks reference XDEPs directly (`satisfies_xdeps`), independently of whatever ACs they also satisfy.

```
AC ─────────► PLAN task     (requirements traceability)
XDEP ───────► PLAN task     (dependency traceability)
PLAN task ──► Milestone     (implementation structure)
```

## XDEP identity and IDs

One XDEP is one directed edge — one provider, one consumer. Fan-out (one producer, many consumers — the normal shape in pub/sub) is multiple XDEPs sharing the same `contract.id`, not one XDEP with a list of consumers.

IDs are thread-qualified and sequential: `THREAD-NNNN-XDEP-NN`. This avoids collisions between threads but not between two branches concurrently proposing a dependency for the *same* thread — the coordination repo's CI validates ID uniqueness against HEAD at merge time; a collision requires the later branch to reallocate, the same way a migration-numbering conflict is resolved. IDs stay sequential and human-readable rather than switching to an opaque identifier — consistent with every other ForgeKit ID (`SPEC-NNNN`, `PLAN-NNNN`, `ADR-NNNN`).

**Identity is the durable relationship, not a pinned contract revision.** `contract.revision` evolves in place as the schema revises compatibly (2.1 → 2.2) — that's ordinary git history on the XDEP file itself, an append-only audit trail with no extra changelog field needed. A **new** XDEP is warranted only when the relationship changes in kind: a different provider, consumer, capability, or a breaking change to what's actually being depended on. Treating every compatible version bump as a new XDEP floods the coordination repo with superseded records for changes that were never a new relationship.

**Don't confuse this with the Cross-Repo Contract's own `working`/`approved` revision.** `contract.revision` (above) is the externally-owned interface/schema version for *one* XDEP. `cross_repo_contract.working.revision` / `approved.revision` (see [`templates/reference/cross-repo-resolution.md`](cross-repo-resolution.md#the-cross-repo-contract-working-and-approved-are-tracked-separately)) is a separate, ForgeKit-owned integer counter on the DESIGN as a whole, tracking the Interface Freeze approval state across every XDEP the DESIGN's Cross-Repo Contract section touches. Editing one XDEP's `contract.revision` from 2.1 to 2.2 is what *causes* `cross_repo_contract.working.revision` to bump — the two numbers move together but count different things and are never the same value.

## Schema, discriminated by `dependency_kind`

A `contract`-kind dependency has a `contract` block; a `deployment`-kind dependency does not, because there's no interface to describe, only an ordering constraint.

```yaml
id: THREAD-0142-XDEP-01
status: active                     # active | superseded | retired

dependency_kind: contract          # contract | artifact | deployment | environment
interface_kind: async-event        # required only when dependency_kind: contract —
                                    # async-event | rest | grpc | package

provider:
  repo: org/transformer-service
  capability: DocumentProcessed

consumer:
  repo: org/document-worker

# --- kind-specific payload, present only for the matching dependency_kind ---

contract:                          # dependency_kind: contract
  id: "asyncapi://document-events/DocumentProcessed"
  revision: "2.1"
  compatibility_required: backward

# artifact:                        # dependency_kind: artifact
#   package: org.generated.document-client
#   version: "3.x"

# deployment:                      # dependency_kind: deployment
#   prerequisite: provider          # provider | consumer — which side must exist first

# environment:                     # dependency_kind: environment
#   requires:
#     repo: org/transformer-service
#     health_check: "..."          # reachability alone is not sufficient — see Verification below

verification:                      # a list, not a single mechanism — layered per interface_kind.
                                    # May be empty/TBD while drafting; required before this XDEP
                                    # is part of an approved Interface Freeze.
  - mechanism: schema-registry-compatibility
    gate: publish
  - mechanism: consumer-deserialization
    gate: ci

rollout:
  ordering: independent            # independent | provider-first | consumer-first

related_acceptance_criteria:       # traceability only — a DESIGN-time hint, not kept in
  - AC-3                           # sync with PLAN's own task-level AC association, and never
  - AC-7                           # the resolution path to a milestone (see above)
```

A first XDEP between two repos needs only `id`, `dependency_kind`, `provider`, `consumer`, and the matching kind-specific block — `verification` and `rollout` fill in as the situation demands.

## What each `dependency_kind` means and how it's verified

| `dependency_kind` | What it means | Verified by |
|---|---|---|
| **contract** | Both sides must agree what an interface means before either implements against it | DESIGN review + Interface Freeze |
| **artifact** | The consumer needs a generated type, client, or published package from the provider | Build-time — codegen/package resolution, checked in CI |
| **deployment** | Producer vN must exist in an environment before consumer vN is deployed there | Release sequencing, not contract testing |
| **environment** | A repo's own test needs a *direct* dependency actually running — a lightweight precondition, not a formal scenario | `execute-tests`'s pre-flight. For a genuinely multi-repo, multi-hop scenario, use a cross-repo scenario instead (below) — it doesn't live on a single XDEP or belong to one repo |

Contract verification never requires a deployed environment — that would collapse it into the deployment/environment case and lose the fast-feedback property that makes it worth having on its own.

**Verification is layered per `interface_kind`, not one mechanism:** a schema-compatibility check proves structural compatibility, not semantic/behavioral compatibility — renaming what a field *means* can be schema-compatible and still break every consumer.

| `interface_kind` | Typical verification layers |
|---|---|
| async-event | Schema registry compatibility (prefer *transitive*, not just N vs. N-1), plus a semantic/behavioral check where meaning can drift without a schema change |
| rest | OpenAPI breaking-change check; provider conformance test; consumer-driven contract test where consumer assumptions exceed what the schema states |
| grpc | Protobuf compatibility/breaking-change check; integration test where behavior isn't captured by the schema |
| package | Semantic-version/API-compatibility check |

Whether a verification set is *appropriate* for an interface type is the Architect's call — the same "boundaries and contracts" judgment already central to that role. Whether it's *symmetrically implemented* on both sides is also the Architect's call, since each participant's own local review only checks its own bar, not whether the two sides are commensurately rigorous for a requirement that's fundamentally shared.

## Cross-repo E2E scenarios

For a dependency that genuinely needs two or more services deployed and running together, the specification is centralized (it describes system behavior, not one repo's implementation) — the executable implementation never is, with no exception for simple cases:

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
  repo: org/system-tests            # always an executor-owned repo, never the coordination repo
  ref: e912abc
  path: document-processing/searchable-document.spec.ts

owners:
  - team: document-platform
  - team: search-platform
triage_owner: document-platform

environment:
  profile: cross-repo-preview

execution:
  binding: org/test-platform/document-e2e
  revision: workflow@7fa82c
  triggers:
    release: required
    nightly: required
    boundary_pr: optional

result:
  allowed:
    - PASS
    - FAIL                          # the system behaved incorrectly
    - BLOCKED_ENVIRONMENT           # a required dependency wasn't ready
    - ERROR                         # the execution mechanism itself failed — harness crash,
                                     # runner failure, credential/platform unavailability
```

**One scenario can cover multiple XDEPs across multiple hops** (upload → convert → index → search spans two XDEPs in one flow); **one XDEP can need several scenarios** (happy path, unsupported format, retry, duplicate event); **most XDEPs need none**, because contract verification already covers them. This many-to-many shape is why scenarios get their own thread-level identity (`THREAD-NNNN-E2E-NN`) rather than being identified by the XDEP they relate to.

A participant referencing a scenario (`cross_repo_scenarios: [THREAD-0142-E2E-01]`) means it *participates in* that scenario — never that its own `execute-tests` run decides the scenario's result. Execution is bound to an external system; ForgeKit declares the requirement and consumes the evidence, it does not provision, schedule, or run anything.

**A required scenario going `quarantined` needs the same accepted-risk treatment as a SEC finding** — a named approver and a revisit condition, never a silent exclusion from the gate.

## Evidence

A required run produces a small, fully ref-pinned record — not the full log/trace/screenshot history, which stays with the execution platform:

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

revision_set:                       # which versions were actually tested — belongs to the
  transformer-service: a81f32       # execution request, not the scenario's own identity. The
  document-worker: f92e10           # same scenario runs against a PR cohort, a nightly baseline,
                                     # or a release candidate without its identity ever changing.

target_environment: production
result: PASS
evidence:
  run: "https://..."
```

**Evidence is only eligible to satisfy a release requirement if it matches on every dimension — not merely "a recent PASS exists for this scenario ID."** A 9am nightly `PASS` does not satisfy an 11:30am release gate if the required services deployed at 11:00 and 11:30 — that evidence tested the wrong revision set, regardless of how recently it ran.

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
