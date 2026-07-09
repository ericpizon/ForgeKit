---
name: Feature request
about: Propose a new command, pipeline stage, artifact type, or schema change
title: "[FEATURE] "
labels: enhancement
assignees: ''
---

## What problem does this solve?

What gap in the pipeline does this address? Be specific about which stage, role, or workflow is affected.

## Proposed solution

Describe the command, artifact, or schema change you have in mind. If proposing a new command:
- What is the primary role?
- What artifact does it produce or modify?
- Where does it sit in the pipeline (which stage, after which command, before which command)?
- What quality gate checks would it include?

## Alternatives considered

Are there ways to achieve this with the existing pipeline? Why is a new command or change needed?

## Design constraints

Does this proposal respect the existing constraints?
- [ ] Does not weaken any quality gate
- [ ] Has a clear primary role
- [ ] Fits within the existing artifact schema (or proposes a schema change with a rationale)
- [ ] Routes back correctly through address-* / reopen-* commands if it produces findings

## Additional context

Related commands, artifacts, or prior discussion.
