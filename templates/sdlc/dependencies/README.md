# sdlc/dependencies/

XDEP records — cross-repo dependencies for feature threads that span more than one code repo. Only present in **coordination repos** (`project.yml`'s `repo_role: coordination`); a normal code repo never has this directory.

## Contents

```
THREAD-NNNN-XDEP-NN.yml    # one file per cross-repo dependency
```

## What an XDEP is — and isn't

An XDEP is a durable, directed relationship between two participant repos (one provider, one consumer) — not a contract version, and not an acceptance criterion. See [`xdep-schema.md`](../../.claude/reference/xdep-schema.md) for the full schema, the four `dependency_kind` values, and why acceptance criteria are traceability only, never an XDEP's identity or the path that resolves a dependency to a milestone.

## When XDEPs are created

The Architect creates XDEPs during `/create-design` for a coordination-repo thread, as part of the Cross-Repo Contract sub-section of DESIGN — never authored per-participant, and never resolved through either side's own local review alone.

## IDs are thread-qualified, and collisions are checked at merge time

`THREAD-NNNN-XDEP-NN` avoids collisions between threads. Two branches concurrently proposing an XDEP for the *same* thread can still land on the same number before either merges — this repo's CI validates ID uniqueness against the current HEAD on every PR touching this directory; a collision requires the later branch to reallocate before merging, the same way a migration-numbering conflict gets resolved.

## Lifecycle is append-only

A dependency that stops applying is never deleted — its `status` moves to `superseded` (with a `superseded_by` pointer) or `retired` (with a `reason`). This is the same convention already used for ADRs: the record stays as an audit trail even after it stops being active.
