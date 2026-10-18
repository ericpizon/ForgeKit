---
id: ADR-0004
type: adr
schema_version: 1
title: "Post-merge test gate's offered merge handles both solo and team paths immediately, never leaving a PR open for async review"
status: accepted
created: 2026-10-17
links:
  - DESIGN-0002
generation:
  command: /create-design
  command_version: "3"
  model: "claude-sonnet-5"
  generated_at: "2026-10-17"
  session_id: null
---

# ADR-0004 — Post-Merge Test Gate's Merge Mechanism

## Context

SPEC-0002's `AC-06` requires that under `pipeline.test_gate: post_merge`, once a milestone merges via `/approve-implementation`'s offered merge, `/create-pr` and `/close-milestone` are never invoked for it, and `**Implementation status:**` reaches `pr_merged` directly from the merge step itself. The SPEC's own Open Question 4 asked whether the offered merge should go directly to a fast-forward merge or still support opening a real PR for async review. The straightforward answer — reuse `create-pr.md`'s existing solo/team choice as-is — breaks on `AC-06` for the team path specifically: a real PR opened for async review normally sits at `pr_open` until a human merges it on GitHub, which would require `/close-milestone` to eventually record that merge, directly contradicting "never invoked."

A team using `post_merge` mode plausibly operates under branch-protection rules requiring a PR to exist before anything lands on `main` — an enterprise persona this feature specifically targets. A design that only ever supports a direct, unreviewed push would be unusable for exactly the adopters this feature exists to serve.

## Decision

`/approve-implementation`'s offered merge, under `post_merge`, handles both sub-paths itself, in the same step, every time, never leaving anything open for later:

- **Solo:** direct fast-forward merge to `main` — identical mechanics to `create-pr.md`'s existing local-only path.
- **Team:** opens a real PR (`gh pr create`) *and* attempts to immediately merge it (`gh pr merge`) in the same step. The PR exists in git history — satisfying branch-protection and audit-trail requirements — but is never left pending a second round of async review, because the review a PR would normally gate already happened inside `/approve-implementation` itself (its own Step 7 three-question gate and full diff review, the same rigor every other milestone approval in this pipeline already requires).

**The merge is verified, never assumed.** `gh pr merge`'s exit code alone is not sufficient — the PR's actual merged state is confirmed before `**Implementation status:**` advances. If a required status check or additional approval blocks the immediate merge (a real possibility under genuine branch protection — the exact scenario this mode exists to accommodate), the PR is left open and the milestone stays at `approved` rather than advancing on a false assumption of success; the human resolves the blocker and re-runs, which finds the existing open PR and retries rather than creating a duplicate.

Either sub-path, once the merge is actually confirmed, reaches `pr_merged` directly within this one step. `create-pr.md`/`close-milestone.md` are genuinely never invoked for this milestone, satisfying `AC-06` for both paths, not just the solo one.

## Alternatives Considered

**Always local-only, no real-PR path under `post_merge`.** Rejected — would make the mode unusable for any team operating under branch-protection rules requiring a PR to exist before a merge to `main`, which plausibly includes a meaningful share of the enterprise adopters this feature targets.

**Reuse `create-pr.md`'s existing solo/team choice unchanged, real-PR path left open at `pr_open` for async review.** Rejected — directly contradicts `AC-06`'s literal "never invoked... `pr_merged` directly" requirement for the team path; would also mean `/close-milestone` is sometimes invoked under `post_merge` and sometimes not, depending on solo vs. team, an inconsistent and harder-to-reason-about shape than either alternative.

## Consequences

`/approve-implementation` must itself contain both merge mechanisms (fast-forward and PR-create-then-merge) for the `post_merge` path, rather than delegating to `create-pr.md` as every other milestone-closing flow in this pipeline does. This is a deliberate, narrow duplication of mechanics already proven in `create-pr.md` — not a reimplementation of its reasoning, since there is no review-and-decide step left to make once `/approve-implementation`'s own gate has already run. A future reader of `approve-implementation.md` needs to understand it now owns a merge mechanism under this one mode, which it does not own under `pre_merge`.

## Status Rationale

Accepted — directly satisfies `AC-06` for both the solo and team paths, and keeps `post_merge` mode genuinely usable for the branch-protected-enterprise persona `SPEC-0002`'s own User Stories describe, rather than forcing a choice between satisfying the AC and serving the target user.

## Revisit Triggers

If a future workflow mode needs the same "merge immediately, no async review window" shape, revisit whether this mechanism should be factored into a shared helper both `approve-implementation.md` and the new mode's own command can reference, rather than each duplicating it independently a second time.
