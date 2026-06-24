# sdlc/releases/

Release notes artifacts. Each RELEASE-NOTES-vX.Y.Z is the permanent record of what shipped, when, validation results, and any known gaps. It formally closes the feature thread.

## Commands

| Command | Role | What it does |
|---|---|---|
| `/validate-release vX.Y.Z` | SRE | Execute smoke tests in production; record results in RELEASE-PLAN |
| `/generate-release-notes vX.Y.Z` | TPO | Author RELEASE-NOTES from the validated release plan; marks SPEC as `shipped` |

`/validate-release` must pass before `/generate-release-notes` can run.

## Status lifecycle

```
(created) → complete
```

RELEASE-NOTES are immutable after creation — they record what happened, not a living document.

## Naming convention

`RELEASE-NOTES-vX.Y.Z.md`
