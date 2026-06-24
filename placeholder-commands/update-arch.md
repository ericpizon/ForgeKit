---
id: update-arch
type: command
name: /update-arch
description: Re-run brownfield discovery on a specified scope (file, directory, or module) and diff the findings against the existing ARCH.md — use when significant code changes have made ARCH.md stale
status: placeholder
version: 0
primary_role: lead-developer
created: 2026-06-21
---

# /update-arch

## Intent

This command is a **placeholder**. It is not yet implemented.

The intent is: after a significant refactor or new module is added, run targeted discovery on a specified scope and surface the delta against `sdlc/architecture/ARCH.md` so the human can decide what to update.

---

## Why this is needed

ARCH.md is approved by a human at /forge-init and serves as the living reference for every pipeline decision. It will drift as the codebase evolves. `/update-arch` provides a managed path to re-sync it without re-running full /forge-init.

---

## Usage (intended)

```
/update-arch                          ← re-scan entire project
/update-arch src/auth/                ← re-scan a specific directory
/update-arch src/auth/ src/payments/  ← re-scan multiple paths
```

---

## What the implemented command should do

1. Read current `sdlc/architecture/ARCH.md`
2. Accept a scope argument (defaults to full project if not specified)
3. Re-run the relevant brownfield discovery steps (tech stack, directory structure, auth, observability, security scan) limited to the specified scope
4. Produce a diff: "what discovery found vs what ARCH.md currently says"
   - New modules not in ARCH.md
   - Removed modules still mentioned in ARCH.md
   - Changed patterns (e.g., new auth middleware, different logging approach)
   - New security findings not in ARCH.md
5. Present the diff to the user, section by section
6. For each section with a delta: ask whether to accept the detected change, keep the current text, or manually edit
7. Apply accepted changes to ARCH.md
8. Require explicit user sign-off on the updated ARCH.md before completing (same gate as /forge-init)

---

## ARCH.md sign-off requirement

The pipeline ARCH.md is a human-reviewed document. `/update-arch` must not silently overwrite sections — every proposed change requires explicit acceptance. The command ends only when the user approves the final ARCH.md state.

---

## Scope note

Security findings discovered in `/update-arch` should be added to the `## Pre-existing Security Findings` section in ARCH.md, not silently dropped. The Security Review phase of the next feature thread will pick them up.

---

## How to implement later

To implement: extract the brownfield discovery scan logic from `/forge-init` Phase 4B Steps 1–4 into a reusable discovery routine. `/update-arch` calls that routine with a scope filter, then computes a structural diff against the existing ARCH.md sections. The diff presentation format can mirror the discovery findings format used at /forge-init.
