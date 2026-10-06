---
id: IMPL-REVIEW-0001-M3
type: impl-review
schema_version: 1
plan: PLAN-0001
milestone: M3
current_review_round: 1
approval:
  mode: repo_only
  approved_by: "Eric Pizon"
  approved_role: lead-developer
  approved_at: "2026-10-06"
  decision_summary: "M3 approved for merge — forge-init coordination question + validation, SYSTEM-ARCH.md Dependency Map regeneration, zero blocking findings."
  external_event_id: null
  external_event_uri: null
policy_evaluation:
  evaluated: null
  policy_pack: null
  result: null
generation:
  command: /review-implementation
  command_version: "4"
  model: "claude-sonnet-5"
  generated_at: "2026-10-06"
  session_id: null
---

# Implementation Review — PLAN-0001 M3

## Task Reference — M3

*Derived from PLAN-0001 at review creation time. Reviewers use this to verify coverage without loading the full PLAN.*

| Task | File(s) | Description |
|---|---|---|
| T6 | `.claude/commands/forge-init.md` | New Phase 2b (`repo_role: code` only): AC7's coordination-participation question, recording `coordination_repo` in `project.yml`; "not yet" leaves it unset entirely. Corrected from PLAN-0001's earlier citation of `templates/commands/forge-init.md`, which doesn't exist — `forge-init.md` has no template counterpart. |
| T7 | `templates/commands/approve-design.md` | Interface Freeze path, new step 4: regenerate `SYSTEM-ARCH.md`'s Dependency Map wholesale from the current active-XDEP set (D1/ADR-0002), lazily creating the file if absent; a regeneration failure logs a warning and never blocks the Interface Freeze approval. |
| T19a | `.claude/commands/forge-init.md` | SEC-0001 SEC-3 — validate `coordination_repo` against `^[\w.-]+/[\w.-]+$` before writing; re-prompt on a non-match. Also extended into Option C (re-run) and Option E (full redo), since both re-ask the same question. |

## Round 1 — 2026-10-06

### Lead Developer Findings

**[LD-1] non-blocking — PLAN-0001's T6/T19a file-path citations corrected during implementation, not caught during `/review-plan`**

Both T6 and T19a's task descriptions, and T19a's Task Reference row, cited `templates/commands/forge-init.md` as the file to modify. No such file exists — `forge-init.md` lives only at `.claude/commands/forge-init.md` (ForgeKit's own bootstrapping command, never templated into adopting projects' `.claude/commands/`, unlike every other command this PLAN touches). This is the same citation-error class T3 already self-caught once (an earlier "Phase 4B" reference), but the file-path version of the same mistake slipped through PLAN review twice more (T6, T19a) without being generalized into a check. Fixed directly in PLAN-0001 before implementing (both task descriptions and the T19a Task Reference row now read `.claude/commands/forge-init.md`, with the correction noted inline).

**Resolution recommendation:** No further action needed here — already fixed. Worth a standing note for future PLAN review rounds on this repo specifically: any task touching `forge-init.md` should be checked against `.claude/commands/`, not `templates/commands/`, since this file is a structural exception to how every other command in this PLAN is organized.

### Security Findings

**[SEC-1] non-blocking — SEC-3's mitigation names three fields; only one (`coordination_repo`) has any task or command path to validate it against**

SEC-0001's SEC-3 finding text: *"`/forge-init`'s coordination-repo question (AC7) and `/promote-to-thread`'s reactive equivalent (AC8) validate the supplied value against a safe `org/repo`-shaped pattern... before writing it to `project.yml`, re-prompting on a non-match. **Same validation applies to `environments[].name`.**"* PLAN-0001's actual task coverage for SEC-3 is T19a (this milestone, `coordination_repo` only) and T19b (M4, `/promote-to-thread`'s reactive equivalent, also `coordination_repo` only). No task anywhere in PLAN-0001 validates `known_participants[].repo` or `environments[].name` — and by design (per T3/M1), neither field has a writing command at all; both are documented as directly-edited `project.yml` fields with no authoring command. SEC-3's own mitigation text calling for the "same validation" on `environments[].name` is not achievable as written for a hand-edited field with no command in the loop — there's nothing to validate *at write time* the way AC7's interactive Q&A allows.

This implementation (T19a, scoped to `.claude/commands/forge-init.md` only) does exactly what T19a's own task description asks, and correctly does not attempt to cover the other two fields — that's not this task's file, and no other task claims them either. The gap is between SEC-0001's finding text and PLAN-0001's task coverage, not in what T19a actually built.

**Resolution recommendation:** Not blocking for M3 — `coordination_repo`'s validation (the one field with an actual command path) is correctly implemented. Before M3 is approved, decide explicitly (rather than leaving it silently unaddressed) whether `known_participants[].repo`/`environments[].name` validation should be: (a) accepted as a documented residual risk in SEC-0001 (these are directly-edited, trusted-operator fields per M1's own design — SEC-3 already rates the overall finding Low, and a malformed `environments[].name` would surface naturally the first time a command reads it), or (b) given a validate-at-read-time task somewhere downstream (e.g., `/promote-to-environment`, M5, the first command that actually reads `environments[].name`). Recommend (a), since `/promote-to-environment` validating its own input at the point of use is already standard defensive practice, not something this specific SEC finding needs a dedicated task to force.

**Clean perspectives:** SRE, QE

### Summary
**Summary:** 0 blocking, 2 non-blocking findings.

### Next Step
`/approve-implementation PLAN-0001 M3`

### Disposition (resolved inline, no address cycle needed — both non-blocking)
| Finding | Resolution |
|---|---|
| [LD-1] | Already fixed (PLAN-0001's T6/T19a path citations corrected before implementation began). No further action. |
| [SEC-1] | Resolved per its own recommendation (a): added a Resolution note directly to SEC-0001's SEC-3 finding, documenting `known_participants[].repo`/`environments[].name` as accepted residual risk. `SEC-3`'s `findings_status` left `open` (not `mitigated`) since T19b (M4) still needs to land before the finding's full stated scope is covered — only `coordination_repo`'s validation is done as of M3. |
