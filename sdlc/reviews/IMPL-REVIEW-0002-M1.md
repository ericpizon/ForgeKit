---
id: IMPL-REVIEW-0002-M1
type: impl-review
schema_version: 1
plan: PLAN-0002
milestone: M1
current_review_round: 2
latest_round:
  round: 2
  blocking: 0
  non_blocking: 0
  verdict: ready
generation:
  command: /review-implementation
  command_version: "5"
  model: "claude-sonnet-5"
  generated_at: "2026-10-18"
  session_id: null
approval:
  mode: repo_only
  approved_by: "Eric Pizon"
  approved_role: lead-developer
  approved_at: "2026-10-18"
  decision_summary: "PLAN-0002 M1 (Configurable Post-Merge Test Gate) approved for merge — 2 review rounds, all findings resolved."
  external_event_id: null
  external_event_uri: null
policy_evaluation:
  evaluated: null
  policy_pack: null
  result: null
---

# Implementation Review — PLAN-0002 M1

## Task Reference — M1

*Derived from PLAN-0002 at review creation time. Reviewers use this to verify coverage without loading the full PLAN.*

| Task | File(s) | Description |
|---|---|---|
| T1 | `templates/project.yml.template` | Add `pipeline.test_gate` (pre_merge \| post_merge, fail-closed default) and `stack.qa_trigger_command` fields |
| T2 | `.claude/commands/forge-init.md` | New `test_gate` workflow question (Phase 4C) — states the `post_merge` trade-off plainly, defaults to `pre_merge` |
| T3 | `templates/commands/approve-implementation.md` | `post_merge` branch in Step 9 — offer merge directly, solo fast-forward / team retry-safe PR-create-then-merge, verify before advancing, sanitized `blocked_reason` on a blocked merge |
| T4 | OBS-0002 IT-1 | `post_merge_merge_attempt` field — see OBS-0002 for full implementation spec |
| T5 | `templates/commands/execute-tests.md` | Post-Merge Validation Path — pre-flight routing, `qa_trigger_command`/path-aware local-sync validation, `post_merge_test` write, P0 routing to `/create-bug`/`/escalate-to-incident` |

## Round 1 — 2026-10-18

### Lead Developer Findings

**[LD-1] blocking — `gh`/git infra failures are silently conflated with real negative results at three separate call sites, risking a duplicate PR, a false blocked-merge classification, or a test run against stale state.**

`DESIGN-0002` §3.6 explicitly names one infra-failure distinction (`stack.qa_trigger_command` failing to run is never conflated with a real test result) and the implementation correctly honors it in `execute-tests.md`'s `qa_trigger_command` branch. But the identical class of gap exists, unaddressed, at three other points this same milestone introduces — none of them named in DESIGN-0002's own Failure Modes list, so this wasn't just missed in implementation, it was never named as a failure mode at all through SPEC/DESIGN/PLAN/TEST either:

1. **`approve-implementation.md`, team-path PR lookup** (`gh pr list --head <branch> --state open`): if this command itself fails — auth expired, network error, rate limit — rather than genuinely returning zero results, the text as written has no way to distinguish that from "no open PR exists" and falls straight through to `gh pr create`. If a PR actually exists but the lookup merely couldn't see it, this creates exactly the duplicate PR `ADR-0004`'s own invariant exists to prevent.
2. **`approve-implementation.md`, merge verification** (`gh pr view feature/PLAN-NNNN-MN --json state`): if this command fails to run at all (same causes as above) rather than returning a real `state` other than `MERGED`, the text's "Anything else" branch treats it identically to a genuine branch-protection block — writing a `post_merge_merge_attempt.blocked_reason` of `required_status_check_pending`/`required_review_missing`/`branch_protection_rule` for a condition that was actually "the check itself couldn't run," not a real blocking requirement. This misinforms whoever reads that record later.
3. **`execute-tests.md`, local-fallback sync** (`git checkout main && git pull` on the team path): no stated behavior if `git checkout` is blocked (uncommitted changes) or `git pull` fails (network, or an actual conflict). As written, the command would either error unhandled or — worse — proceed to run tests against whatever state the checkout silently left behind, which per `DESIGN-0002` §3.4 becomes a **permanently unoverwritable** `post_merge_test.status` the moment it's written.

All three share the same shape: a command's own operational failure is not distinguished from the command successfully returning a real, meaningful result. Needs: each of the three call sites gets an explicit "the command itself didn't complete" branch — surfaced directly to the user as an infrastructure problem, exactly like `qa_trigger_command`'s own treatment — before falling through to the branch that currently assumes the call succeeded.

**Clean perspectives:** QE.

*(Security Perspective not applicable — `includes_security_tasks: false`.)*

### Cross-Perspective Check

**Cascade:** SRE's own "Error handling: Critical paths have appropriate error handling" check independently reaches the same three sites as `[LD-1]` — not duplicated as a separate finding, since it's the identical root cause, but worth noting this isn't a Lead-Developer-only concern.

### Summary
**Summary:** 1 blocking, 0 non-blocking findings.

### Next Step
`/address-implementation-review PLAN-0002 M1`

### Address Summary — Round 1

Date: 2026-10-18

| Finding | Resolution |
|---|---|
| [LD-1] | Fixed: all three call sites now have an explicit "command itself failed to complete" branch, surfaced directly to the user as an infrastructure problem, before the branch that assumes the call succeeded. `gh pr list` (approve-implementation.md): a failed lookup is no longer treated as "no open PR exists" — stops rather than risking a duplicate PR. `gh pr view` merge verification (approve-implementation.md): a failed check no longer writes a `post_merge_merge_attempt.blocked_reason` — that field is now reserved for a real, returned state other than `MERGED`. Local git sync (execute-tests.md): a failed `git checkout`/`git pull` now stops before running any tests, rather than risking a result computed against a half-synced checkout becoming a permanent record. |

**Not addressed:** None — the one Round 1 finding is fully resolved above.

No pre-commit checks to run (`stack.lint`/`stack.test.unit` are both `none` for this repo).

**Next Step:** `/review-implementation PLAN-0002 M1`

## Round 2 — 2026-10-18

Scope: `approve-implementation.md`'s `gh pr list`/`gh pr view` sections and `execute-tests.md`'s local-sync section — the sections Round 1's address pass touched.

**Lead Developer:** `[LD-1]` confirmed resolved at all three call sites — each now has an explicit "the command itself failed to complete" branch, surfaced directly as an infrastructure problem, before the branch that assumes the call succeeded. Checked one adjacent edge case not named in Round 1 (`gh pr create` itself failing, as opposed to `gh pr merge`): not explicitly branched on, but the downstream `gh pr view` verification step naturally catches this too (nothing to verify / PR not found fails the same way an infra error does) and routes to the same "stop, surface, no `blocked_reason` written" handling — safe by construction even though the failure-cause framing would be slightly imprecise for a genuine permissions error. Not worth a separate finding.

**SRE:** no new findings — domain unaffected beyond `[LD-1]`'s own resolution, which SRE's error-handling check already credited in Round 1's Cross-Perspective note.

**QE:** no new findings — domain unaffected; the fix adds stop-conditions only, no change to any already-reviewed test-relevant behavior.

**Clean perspectives:** Lead Developer, SRE, QE.

*(Security Perspective not applicable — `includes_security_tasks: false`.)*

### Contradictions Requiring Arbitration

None.

### Summary
**Summary:** 0 blocking, 0 non-blocking findings.

### Next Step
`/approve-implementation PLAN-0002 M1`
