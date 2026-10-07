# sdlc/release-plans/

Release plan artifacts. Each RELEASE-PLAN-vX.Y.Z captures gating criteria, migration sequencing, rollback plan, and open risk acknowledgment for a production release. An approved RELEASE-PLAN is required before `/execute-release`.

## Commands

| Command | Role | What it does |
|---|---|---|
| `/create-release-plan vX.Y.Z` | SRE | Author release plan: gate assessment, rollback plan, postmortem action item risks |
| `/review-release-plan vX.Y.Z` | Lead Dev | Review readiness, rollback specificity, migration sequencing |
| `/address-release-plan-review vX.Y.Z` | SRE | Address release plan review findings |
| `/approve-release-plan vX.Y.Z` | SRE | Formal approval; unlocks `/execute-release` |
| `/execute-release vX.Y.Z` | SRE | Deploy to production per the approved plan |

## Status lifecycle

```
draft → approved → (deployment record populated by /execute-release)
```

## Naming convention

`RELEASE-PLAN-vX.Y.Z.md`
