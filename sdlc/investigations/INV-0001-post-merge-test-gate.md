---
id: INV-0001
type: investigation
schema_version: 1
status: concluded
decision: recommend_implementation
linked_adr: null
created: 2026-10-09
concluded_at: 2026-10-09
generation:
  command: /create-investigation
  command_version: "2"
  model: "claude-sonnet-5"
  generated_at: "2026-10-09"
  session_id: null
---

# INV-0001 — Should ForgeKit support a configurable post-merge test gate?

## Question

Should ForgeKit support a second milestone workflow mode where `/approve-implementation` offers to merge the milestone branch to `main` immediately, with `/execute-tests` validating the merged/deployed state afterward (QE testing in a shared QA environment) — rather than today's only mode, where `/execute-tests` must pass against the still-unmerged feature branch before any merge happens at all?

Raised directly: "Once implementation is approved, it should ask to merge the branch. Then, test will not be picking it up from the branch. This will be the default behavior in enterprise. QE will test it in QA environment. We can set this up as a workflow question during init."

## Approach

Read the actual text of every command in today's milestone-closing sequence (`implement-plan.md`, `approve-implementation.md`, `execute-tests.md`, `create-pr.md`) to establish precisely what today's model actually guarantees and where it would break under the proposed alternative. Checked whether ForgeKit already has any directionally-similar mechanism (the `environments`/governance model, `stack.ci_trigger_command`'s CI Path) that a real implementation should reuse rather than duplicate. Considered the one question the proposal itself doesn't yet answer: what happens when a post-merge test fails, given `main` would by then already carry the regression.

## Findings

**Today's model is test-gates-merge, confirmed directly against the command text, not assumed:** `approve-implementation.md` commits milestone code to `feature/PLAN-NNNN-MN` — never `main`. `execute-tests.md` validates that same feature branch (its "CI Path" explicitly triggers CI "for the current branch," not `main`). Only once `tests_passed` does `create-pr.md` either open a real PR or (solo/local-only) fast-forward-merge into `main`. There is currently no point in this sequence where unverified milestone code ever reaches `main`.

**The proposed model (merge-gates-test) is a real, common, legitimate alternative — not a workaround.** Trunk-based development with CI running post-merge against `main`, validated in a shared QA/staging environment that continuously deploys from it, is standard in many enterprises. The actual gate in that model is code review (ForgeKit's `/approve-implementation` already *is* a human-confirmed review gate), not test-passing — tests run after, and a failure triggers a fix-forward or revert, not a blocked merge.

**This is not a small plumbing change — it relocates what `/execute-tests` validates and introduces a new failure-response question today's model never has to answer.** Under the proposed mode, `/execute-tests` would need to validate the merged/deployed state (`main`, or whatever QA environment deploys from it) rather than a feature branch — a genuinely different target, not just a different flag. And because the code is already on `main` by the time a test runs, a P0 failure discovered this way means `main` is, for some window, carrying a real regression. Today's model never faces this question because it structurally can't happen.

**ForgeKit already has partial, directionally-similar scaffolding worth reusing rather than duplicating:**
- The `environments`/`governance` model (`ci | promotion | release` in `project.yml`) already distinguishes "stays outside ForgeKit's scope entirely" (`ci`) from governed promotion tracking — but today only the coordination-repo, multi-participant case (`promotion`) and full production release (`release`) are modeled; a single repo's own milestone-to-QA handoff has no governed shape yet.
- `execute-tests.md`'s CI Path already triggers an external CI system via `stack.ci_trigger_command` — but explicitly scoped "for the current branch," confirming today's CI integration point is pre-merge by design, not an accident that happens to match the rest of the model.
- The existing BUG/incident machinery (`/create-bug`, `/escalate-to-incident`) is already the right shape for "something broke that's already shipped somewhere" — the post-merge-failure question this proposal raises doesn't need new machinery invented from scratch, just a defined path into the machinery that already exists.

**`/create-pr`/`/close-milestone` become largely redundant under a post-merge-test mode** — there is no PR left to open once the merge already happened before testing. The milestone's actual closure point moves to immediately after the merge, and `/execute-tests`'s role shifts from "gate before merge" to "post-merge verification" — structurally closer to what `/validate-release` already does at the release stage than to what `/execute-tests` does today.

## Decision

**`recommend_implementation`**

## Rationale

The Findings support building this, not deferring it further — the pattern is real, explicitly requested, and ForgeKit already has most of the structural pieces (human-confirmed merge gate, BUG/incident response machinery, a precedent for environment-governance configuration) a real implementation should reuse rather than reinvent. It is not `no_action`: dismissing this would leave ForgeKit without a mode a large class of real adopting teams actually need. It is not `investigate_further`: the open sub-questions (how to configure it, what a post-merge failure should trigger) have reasonably clear answers already visible in ForgeKit's own existing conventions, not blockers requiring more research before a SPEC could be written.

Concretely, for whoever picks this up as a SPEC: a new `project.yml` field (e.g. `pipeline.test_gate: pre_merge | post_merge`, default `pre_merge` — preserving today's behavior for every existing project), asked as a new `/forge-init` workflow question, the same way `artifact_mode`/`repo_role` are configured today. Under `post_merge`: `approve-implementation.md`'s Handoff offers the merge directly (instead of "run `/execute-tests`"); `execute-tests.md` validates the merged/deployed state; a P0 failure discovered post-merge routes into `/create-bug` (or `/escalate-to-incident` if the QA environment is shared and blocking others) rather than `test_failed`, since the code is no longer sitting in a reviewable, revertible-without-consequence feature branch once this point is reached. `create-pr.md`/`close-milestone.md` are skipped entirely under this mode, consistent with there being no PR left to open.

This is advisory only — nothing was created or written externally, and no `project.yml`/command file has been changed by this investigation itself.

## Next step

`/create-spec` (optionally scoped narrowly to this one workflow-mode addition) when ready to act on this — a reasonably self-contained feature, structurally similar in size to the release-approval-binding work PLAN-0001 M3 just finished, not a large undertaking.
