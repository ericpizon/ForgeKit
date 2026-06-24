# sdlc/features/

Feature registry entries produced by `/sync-registry`. This directory is auto-generated — do not edit files here manually.

## Contents

```
FEATURE-NNNN.md    # one dashboard per feature thread — always regenerated
```

The master artifact index (`index.md`) lives in `sdlc/registry/`, not this directory. `/sync-registry` regenerates both.

## Lifecycle

`/sync-registry` scans all `sdlc/` artifact directories, groups artifacts into threads, and regenerates every FEATURE-NNNN.md and `index.md` on each run. Any manual edits to these files will be lost on the next `/sync-registry` call.

## Purpose

The feature registry gives a project-level view of what has been built, which artifacts belong to each thread, and current stage and status. It is particularly useful when starting a new feature thread that builds on or modifies an existing one.

Run `/sync-registry` after any major status transition to keep the registry current:
- After `/close-milestone`
- After `/generate-release-notes`
- After `/create-spec` (new thread)
- Before starting a new feature (to see what is currently in flight)
