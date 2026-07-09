# Contributing to ForgeKit

ForgeKit is an early-stage open protocol. Feedback, bug reports, and contributions are welcome — here is how to engage effectively.

---

## What this project is

ForgeKit is an experimental AI-SDLC protocol, not a production compliance platform. It is a set of governed, evidenced markdown command files for Claude Code. The value is in the protocol design — the command structure, quality gates, role model, and artifact schema — not in any compiled software.

This shapes what kinds of contributions are most useful.

---

## What contributions are in scope

**High value:**
- Bug reports — commands that produce incorrect output, missing quality gate checks, broken cross-command references, role attribution errors
- Accuracy improvements — inconsistencies between what a command says it does and what the artifact template requires, mismatches between PIPELINE.md and command frontmatter
- New command proposals — additions to the pipeline that follow the existing role model and artifact schema (file an issue first to align on design before submitting a PR)
- Example artifacts — high-quality fictional examples at the quality bar the pipeline enforces (Fieldstone is the established fictional project; additions should use it)
- Clarity improvements — commands or docs that are ambiguous, incomplete, or harder to follow than they should be

**Lower priority (may be declined):**
- Changes that require a specific external tool, platform, or paid service to function
- Commands that bypass or weaken quality gates
- Changes that add complexity without a clear pipeline problem they solve

---

## Filing an issue

Use the issue templates provided:
- **Bug report** — for commands that produce wrong output, broken references, or inaccurate instructions
- **Feature request** — for new commands, new pipeline stages, or new artifact types

Before filing: search existing issues to avoid duplicates.

For each issue, be specific about which command or file is affected and what the expected behavior is. Vague reports are hard to act on.

---

## Submitting a pull request

1. **File an issue first** for anything beyond a clear one-line fix. This avoids wasted effort if the direction is not aligned.

2. **Keep PRs focused.** One fix or one feature per PR. Mixed changes are harder to review and harder to reason about.

3. **Follow the existing patterns.** Commands use a consistent structure: frontmatter, Purpose, Steps, Quality Gate, Handoff. New commands should follow this structure. Artifact templates use `schema_version: 1`, structured `approval:` blocks, and `generation:` blocks — new artifact types should follow suit.

4. **Run a self-review before opening the PR.** Check:
   - Does the command's `primary_role:` match what PIPELINE.md says?
   - Are there cross-command references that need to be updated?
   - Does the Quality Gate check actually verify what the command produces?
   - Are any `{{placeholder}}` tokens substituted or intentionally left for forge-init to handle?

5. **No generated content.** PRs should contain human-authored changes, not AI-generated output submitted without review. (Using AI to draft and then carefully reviewing is fine.)

---

## Design constraints to respect

- **Never weaken quality gates.** The gates are what make approvals meaningful.
- **Back-cycles are first-class.** Commands that produce findings should route back correctly through address-* and reopen-* commands.
- **Role attribution matters.** Every command has one primary role. The right human runs the right step.
- **Artifact schema is governed.** Changes to frontmatter fields affect every project that uses ForgeKit. Propose schema changes as issues before implementing.
- **The git log is the audit trail.** Commands that change artifacts should commit at the right boundary — not too early (before review) and not too late (after the window closes).

---

## License

By contributing to ForgeKit, you agree that your contributions will be licensed under the [Apache License 2.0](LICENSE).
