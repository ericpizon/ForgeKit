# sdlc/promotions/

One append-only log file per environment whose `governance` is `promotion`, written to by `/promote-to-environment`. Coordination-repo only.

## Contents

```
{environment}.md    # e.g. qa.md — one file per environment, one row per promotion event
```

Not a numbered per-event artifact (`PROMOTION-NNNN`) — this environment is deliberately lightweight, specifically to avoid recreating the ceremony-for-routine-events problem a full RELEASE-PLAN exists to solve for `governance: release`. See each file's own header for the exact row format: a per-participant candidate map, who promoted it, when, and a smoke-test result — explicitly nothing more (no Gating-criteria table, no Rollback plan, no Validation Plan).

## Governance levels

An `environments` entry (declared in this repo's own `project.yml`) names one of three levels — commands reason about the level, never about the environment's literal name:

| Governance | What happens | Where |
|---|---|---|
| `ci` | Nothing — outside ForgeKit's scope entirely, same as today. | — |
| `promotion` | A lightweight append-only record. | here, `sdlc/promotions/{environment}.md` |
| `release` | The existing, unchanged RELEASE-PLAN ceremony. | `sdlc/release-plans/` |

## How `environments` and `known_participants` get set

Both are optional, additive fields on this repo's own `project.yml` — manually edited, not populated by a dedicated command (unlike `coordination_repo`, which `/forge-init`/`/promote-to-thread` *do* write). A coordination repo missing either field is unaffected; nothing in ForgeKit requires them to exist.

```yaml
known_participants:
  - repo: org/api
    purpose: public-api
environments:
  - name: qa
    purpose: acceptance
    governance: promotion
```

## Naming convention

One file per environment, named for the environment itself: `{environment}.md` (e.g. `qa.md`, `staging.md`) — not a sequence number.
