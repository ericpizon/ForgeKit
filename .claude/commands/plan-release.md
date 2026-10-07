---
id: plan-release
type: command
name: /plan-release
description: DEPRECATED — use /create-release-plan vX.Y.Z instead
status: deprecated
version: 1
primary_role: sre
---

# /plan-release — Deprecated

**This command is deprecated.** Use `/create-release-plan vX.Y.Z` instead.

## What to use instead

```
/create-release-plan vX.Y.Z
```

`/create-release-plan` supersedes this command. It provides the same release planning capability plus:
- Full gating criteria assessment (tests, SEC, OBS, migrations, environment)
- Explicit rollback plan with required command-level specificity
- Open postmortem action item risk surfacing
- Cleaner version-based artifact ID (`RELEASE-PLAN-vX.Y.Z` instead of `RELEASE-PLAN-NNNN`)

## Why deprecated

The original `/plan-release` used a sequence-numbered artifact ID that conflicted with the version-based release naming used throughout the pipeline (`vX.Y.Z`). `/create-release-plan` uses the version directly as the artifact identifier, making cross-references unambiguous.
