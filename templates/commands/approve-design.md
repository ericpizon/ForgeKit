---
id: approve-design
type: command
name: /approve-design
description: Architect command to close the design review loop — confirm all findings resolved, mark DESIGN approved, update ARCH, trigger Stage 4
status: active
version: 4
primary_role: architect
output_artifact_type: design (approved)
changelog:
  - v4 — Interface Freeze now also regenerates the coordination repo's SYSTEM-ARCH.md Dependency Map wholesale from the current active-XDEP set (D1/ADR-0002), creating the file lazily if needed; a regeneration failure logs a warning and never blocks the Interface Freeze approval itself. Built for SPEC-0001 (AC6).
  - v3 — Interface Freeze Step 2 now points at cross-repo-resolution.md's self-referential-ref pattern for approved.ref instead of the unresolvable "advance approved.ref to match working" instruction (working carries no ref to copy).
  - v2 — added the coordination-repo Interface Freeze path (`/approve-design DESIGN-NNNN interface-freeze`): approves the Cross-Repo Contract sub-section independently of the rest of DESIGN, advancing its working/approved revision; full approval now updates each affected participant's own ARCH.md rather than a coordination repo's (which carries no cross-cutting patterns). Moved "Sync Your Branch" ahead of the Interface Freeze path so a team context pulls before reading review findings on that path too, not just the full-approval path.
---

# /approve-design

## Purpose

Close the design review loop. Confirm all review sections are present and all blocking findings are resolved or accepted as documented risk. Mark the DESIGN `approved`, update ARCH with any cross-cutting patterns introduced, then trigger Stage 4 (Observability Design + conditional Security Review).

This is the exit gate for Stage 3. Nothing in Stage 4 begins before `/approve-design` runs.

## When to Invoke

- `/review-design` outputs a clean `## Design Review — Round N` section (zero blocking findings)
- Do not invoke if any blocking finding remains unresolved

Pass the DESIGN ID: `/approve-design DESIGN-NNNN`.

## Before Starting — Sync Your Branch

**Team context:** Pull before reading — all reviewer sections must be on the branch.
```
git pull origin thread/NNNN-slug
```
**Solo context:** Skip — already local.

This applies to both paths below, including the Interface Freeze path — a coordination repo exists specifically to be shared across repos and people, so reading a stale local copy of the Cross-Repo Contract Review findings is exactly the kind of mistake this step exists to prevent.

## Interface Freeze path (coordination-repo threads only)

`/approve-design DESIGN-NNNN interface-freeze` approves only the Cross-Repo Contract sub-section, ahead of the rest of DESIGN — used when `/review-design`'s Cross-Repo Contract Review is clean but other findings on the DESIGN remain open. This exists because "wait for the whole DESIGN to be approved" is a liability with several participants: one unresolved design question in one participant's section would otherwise block every other participant's `create-plan`, even when nothing in their own work depends on it.

1. Confirm the latest `## Design Review — Round N`'s Cross-Repo Contract Findings are clean (zero blocking). If not, do not proceed — direct the user to `/address-design-review DESIGN-NNNN` first.
2. Set `cross_repo_contract.working.status: approved` in DESIGN-NNNN frontmatter (see [`templates/reference/cross-repo-resolution.md`](../reference/cross-repo-resolution.md#the-cross-repo-contract-working-and-approved-are-tracked-separately)), then advance `approved.revision` to match `working.revision`. `approved.ref` has no equivalent in `working` to copy — it must point at this same approval commit, which doesn't have a SHA yet. Use the self-referential-ref pattern: write `approved.ref: PENDING_SELF_COMMIT`, make the approval commit (step 7 below), then make an immediate one-line follow-up commit replacing the sentinel with the real SHA (see [`templates/reference/cross-repo-resolution.md`](../reference/cross-repo-resolution.md#the-self-referential-ref-pattern)).
3. Update the `status` field on every XDEP this revision touched to `active` (superseding any prior revision's XDEP file per the append-only lifecycle in `templates/sdlc/dependencies/README.md`).
4. **Regenerate `SYSTEM-ARCH.md`'s Dependency Map** in the coordination repo (D1/ADR-0002) — this is the exact moment an XDEP transitions to `active`, which is what triggers regeneration; it is never incrementally patched and never requires a separate sync command. Read every `active`-status XDEP currently on record across every thread in this coordination repo (not just this thread's own), and rebuild the `## Dependency Map` section wholesale from that complete set. If `SYSTEM-ARCH.md` does not exist yet in this coordination repo, create it lazily now (frontmatter `type: system-architecture`, `schema_version: 1`, `dependency_map_generated_at: [today]`, `changelog: []`; body starts with an empty `## External Subsystems` section — hand-maintained, never touched by this step — followed by the freshly generated `## Dependency Map`). Update `dependency_map_generated_at` to today regardless of whether the file was just created or already existed. **If regeneration fails** (e.g., a malformed XDEP file encountered while reading the active set): log a warning naming the specific XDEP and the parse problem, and proceed — `SYSTEM-ARCH.md` is a derived convenience view, not a gate; it must never block the Interface Freeze approval itself, which is what actually matters (the Cross-Repo Contract's own approval, step 2 above).
5. Do **not** update `status: approved` on DESIGN-NNNN itself, and do not touch ARCH in any participant repo — the rest of the DESIGN remains `draft`/under review independently.
6. Show the user the approved Cross-Repo Contract section and confirm before committing: "Cross-Repo Contract approved — revision [N] is now the approved contract for this thread. Every participant whose PLAN depends only on XDEPs may now run `create-plan`. [If SYSTEM-ARCH.md was regenerated:] SYSTEM-ARCH.md's Dependency Map has been regenerated to reflect the current active XDEP set. [If regeneration failed:] SYSTEM-ARCH.md regeneration failed ([warning detail]) — this does not block approval, but the Dependency Map may be stale until the next successful regeneration. Confirm: (1) proceed with the commit? (2) solo project (no push) or team project (push too)?"
7. Commit and push exactly as in the full-approval Handoff below, but with a commit message of the form `DESIGN-NNNN: Cross-Repo Contract approved (Interface Freeze, rev [N])`, also including `SYSTEM-ARCH.md` if it was created or updated, and without touching ARCH.

Once the remaining DESIGN findings are also resolved, run `/approve-design DESIGN-NNNN` (the plain, full-approval form) to close Stage 3 normally — it will detect the Cross-Repo Contract is already approved and skip re-approving it (step 2 below).

## Required Inputs

- **DESIGN-NNNN** — latest `## Design Review — Round N` section must show zero blocking findings
- **ANALYSIS-NNNN** — read `security_review_required` to determine whether Stage 4b is triggered
- **`sdlc/architecture/ARCH.md`** — will be updated if this DESIGN introduces cross-cutting patterns. For a `repo_role: coordination` thread, there is no local ARCH.md to update — see Step 3's coordination-repo branch instead.

## Steps

1. **Final review gate.** Read `current_review_round` from DESIGN-NNNN frontmatter. Navigate directly to `## Design Review — Round {current_review_round}` — do not scan for it. Confirm:
   - Blocking finding count is zero
   - DESIGN version is current

   If any blocking finding remains: **do not proceed**. Direct the user to run `/address-design-review DESIGN-NNNN` to resolve them first.

   Non-blocking findings do not block approval. Accept them as developer notes — do **not** address them inline here. If the user wants to address non-blocking findings, they run `/address-design-review DESIGN-NNNN` separately, before or after approval.

2. **Update DESIGN status to `approved`.** In DESIGN frontmatter:
   - Set `status: approved`
   - Add `approval:` block:
     ```yaml
     approval:
       mode: repo_only
       approved_by: "[Architect name]"
       approved_role: architect
       approved_at: "[today's date]"
       decision_summary: "[one sentence confirming DESIGN-NNNN approved for Stage 4]"
       external_event_id: null
       external_event_uri: null
     policy_evaluation:
       evaluated: null
       policy_pack: null
       result: null
     ```
   - Add changelog entry: `vN — approved after [N] review rounds`

   **If `project.yml`'s `repo_role` is `coordination` and `cross_repo_contract.working.status` is not already `approved`:** run the Interface Freeze steps above (setting `working.status: approved` and advancing `approved`) as part of this same approval, rather than requiring a separate `interface-freeze` invocation. If it's already `approved` (a prior `interface-freeze` run), leave it untouched.

3. **Update ARCH.**

   **`repo_role: code` (default):** apply any cross-cutting patterns introduced by this DESIGN that belong in the system-wide baseline:
   - New data access patterns (new table shapes, new query conventions)
   - New API conventions (new route patterns, new response shapes)
   - New auth patterns (new trust boundary decisions)
   - New module layout conventions

   In ARCH: increment version, add changelog entry referencing this DESIGN, update the relevant section.

   **Date-stamp new entries.** Every new pattern added to ARCH must include `(added: YYYY-MM-DD, DESIGN-NNNN)` so future readers know when it was introduced and which design decision drove it.

   **Handle superseded patterns.** If this DESIGN replaces or significantly revises an existing ARCH pattern: mark the old entry `(superseded by [new pattern name], YYYY-MM-DD)` and move it to a `## Superseded Patterns` section at the bottom of ARCH.md (create the section if it does not exist) — do not delete it (audit trail) but remove it from the active sections so it does not mislead future readers.

   If a foundational section of ARCH changed — trigger ARCH review before proceeding. If only a new section or entry was appended, no ARCH review needed.

   **`repo_role: coordination`:** the coordination repo carries no cross-cutting patterns of its own (per `forge-init`'s coordination-repo ARCH.md, which deliberately omits Tech Stack/Module Layout/Data Layer sections) — there is nothing to update here. Instead, for each participant this DESIGN introduces a new pattern for, name the specific participant repo and the pattern in the Handoff summary below, so that repo's own ARCH.md gets updated. If that participant repo is reachable as one of this session's own working directories, apply the update directly (same date-stamping and superseded-pattern rules as the `repo_role: code` case, just written into that repo's `sdlc/architecture/ARCH.md`); otherwise, hand the update off explicitly rather than silently skipping it — a pattern this DESIGN establishes but no one's ARCH.md ever records will be re-litigated the next time someone designs against that participant.

## Quality Gate

Before committing:
- [ ] Latest Design Review section shows zero blocking findings
- [ ] Non-blocking findings accepted as-is (no inline resolution required)
- [ ] DESIGN frontmatter: `status: approved`; `approval:` block set (`approved_by`, `approved_role: architect`, `approved_at`, `decision_summary` populated)
- [ ] *(repo_role: code)* ARCH updated to reflect this DESIGN's cross-cutting contributions
- [ ] *(repo_role: code)* New ARCH entries date-stamped with `(added: YYYY-MM-DD, DESIGN-NNNN)`
- [ ] *(repo_role: code)* Superseded patterns moved to `## Superseded Patterns` section and marked `(superseded by ..., YYYY-MM-DD)`
- [ ] *(repo_role: code)* ARCH version incremented, changelog entry added
- [ ] *(repo_role: coordination)* `cross_repo_contract.working.status: approved` and `approved` advanced to match, unless already set by a prior `interface-freeze` run
- [ ] *(repo_role: coordination, Interface Freeze reached this run)* `SYSTEM-ARCH.md`'s Dependency Map regenerated wholesale from the current full set of `active` XDEPs (not incrementally patched); `dependency_map_generated_at` updated; file created lazily if it didn't exist. A regeneration failure was logged as a warning and did not block this approval.
- [ ] *(repo_role: coordination)* Every participant repo whose ARCH.md needs a new pattern is named explicitly, with the update applied directly or handed off — never silently skipped

**Self-check before proceeding:** Re-read each item above. Confirm each is complete, or state what was skipped and why before continuing to Handoff.

## Handoff

**Before committing — pause for human review:**

Show the user the approved DESIGN and the ARCH diff. Then ask:

> **Before we commit — three questions:**
> 1. Name one trade-off accepted by not choosing the alternative design option. Is that trade-off still the right call given current constraints?
> 2. What happens when the primary external dependency (database, auth provider, or primary external service) is unavailable? Is the fail mode explicitly documented?
> 3. Is there any Key Decision that would require a developer to make an architectural judgment call to implement? If yes, the decision needs more specificity.
>
> If any answer reveals a gap: return to `/address-design-review DESIGN-NNNN`.
>
> If all three check out: "Ready to mark DESIGN approved and trigger Stage 4. Confirm: (1) proceed with the commit? (2) solo project (no push) or team project (push too)?"

**Commit (after user confirms):**

`repo_role: code`:
```
git add sdlc/design/DESIGN-NNNN-slug.md sdlc/adr/ADR-NNNN-slug.md sdlc/architecture/ARCH.md sdlc/registry/index.md
git commit -m "DESIGN-NNNN: approved — [N review rounds]"
```

`repo_role: coordination` (this repo has no local, flat `sdlc/adr/` — any ADR this DESIGN authored is thread-scoped at `sdlc/THREAD-NNNN/adr/`; the top-level `sdlc/dependencies/` directory holds this thread's XDEP files, named `THREAD-NNNN-XDEP-NN.yml`):
```
git add sdlc/THREAD-NNNN/DESIGN.md sdlc/THREAD-NNNN/adr/ADR-NNNN-slug.md sdlc/dependencies/THREAD-NNNN-XDEP-*.yml sdlc/architecture/SYSTEM-ARCH.md sdlc/registry/index.md
git commit -m "THREAD-NNNN: DESIGN approved — [N review rounds]"
```
`sdlc/architecture/SYSTEM-ARCH.md` is only ever touched when an Interface Freeze happened this run (full approval's Step 2 coordination branch, or a standalone `interface-freeze` invocation) — omit it from `git add` on a run that never reached Interface Freeze (e.g. re-approving after a non-Cross-Repo-Contract finding was addressed).
Then, for each participant repo whose ARCH.md was updated directly (per Step 3 above), commit that change separately in that repo — it is not part of this coordination-repo commit.

**Push (only if user confirms team context):**
```
git push origin thread/NNNN-slug
```

**Trigger Stage 4 (both run in parallel):**
- **SRE** → `/design-observability DESIGN-NNNN` (Stage 4a — always)
- Check ANALYSIS-NNNN `security_review_required`: if `yes` → **Security** → `/security-review SPEC-NNNN DESIGN-NNNN ANALYSIS-NNNN` (Stage 4b — conditional)

Both must complete before Lead Developer begins `/create-plan`.
