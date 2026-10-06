---
id: approve-spec
type: command
name: /approve-spec
description: TPO command to close the spec phase — confirm review is ready, mark SPEC approved, commit all spec-phase artifacts
status: active
version: 4
primary_role: tpo
output_artifact_type: spec (approved)
changelog:
  - v4 — added write-back to the external work system named in `source_work` (if present): a confirmation-gated attempt to update the originating ADO/Confluence ticket with the approval decision, firing on every approval including re-approvals after `/reopen-spec`. Capability-check-then-attempt, any-non-confirmed-success-is-failure, sanitized failure categories — see [`templates/reference/external-work-intake.md`](../reference/external-work-intake.md). Built for SPEC-0001 (AC4).
  - v3 — coordination-repo commit now includes thread.yml, created by /create-spec but previously never added to any commit until caught by eye as an untracked file.
  - v2 — added the coordination-repo commit path: SPEC-NNNN and ANALYSIS-NNNN live at sdlc/THREAD-NNNN/SPEC.md and sdlc/THREAD-NNNN/ANALYSIS.md for a thread, not sdlc/specs/ and sdlc/analysis/.
---

# /approve-spec

## Purpose

Close the spec phase. Confirm the latest review shows `readiness_verdict: ready`, mark the SPEC `approved`, and commit all spec-phase artifacts in a single commit. Nothing in Stage 3 (Design) begins before `/approve-spec` runs.

## When to Invoke

- `/review-spec` outputs `readiness_verdict: ready`
- Do not invoke if any open question is unresolved or `readiness_verdict: needs_revision`

Pass SPEC and ANALYSIS IDs: `/approve-spec SPEC-NNNN ANALYSIS-NNNN`.

## Required Inputs

- **SPEC-NNNN** — latest version; all revision rounds complete
- **ANALYSIS-NNNN** — must show `readiness_verdict: ready`

## Steps

1. **Final readiness gate.** Read ANALYSIS-NNNN and SPEC-NNNN frontmatter. Confirm:
   - `readiness_verdict: ready`
   - All open questions in SPEC-NNNN are resolved or explicitly deferred with documented rationale
   - SPEC version is current
   - `product_validated: true` in SPEC-NNNN frontmatter

   If `readiness_verdict: needs_revision`: **do not proceed.** Return to `/revise-spec SPEC-NNNN ANALYSIS-NNNN`.
   If `product_validated: false` or missing: **do not proceed.** "Product validation not complete. Run `/validate-spec SPEC-NNNN` first."
   If `product_validated: pending`: **do not proceed.** "Validation is pending sign-off. Update SPEC frontmatter to `product_validated: true` once confirmed, then re-run `/approve-spec`."

2. **Mark SPEC approved.** In SPEC-NNNN frontmatter:
   - Set `status: approved`
   - Add `approval:` block:
     ```yaml
     approval:
       mode: repo_only
       approved_by: "[TPO name]"
       approved_role: tpo
       approved_at: "[today's date]"
       decision_summary: "[one sentence confirming SPEC-NNNN approved for design phase]"
       external_event_id: null
       external_event_uri: null
     policy_evaluation:
       evaluated: null
       policy_pack: null
       result: null
     ```
   - Add changelog entry: `vN — approved after [N] review rounds`
   - Do not modify `product_validated`, `validated_by`, or `validated_date` — these were set by `/validate-spec` and are not overwritten here.

3. **Mark ANALYSIS completed.** In ANALYSIS-NNNN frontmatter:
   - Set `status: completed`
   - Confirm `readiness_verdict: ready` and `security_review_required` reflect the final round's verdict

4. **Update registry.** In `sdlc/registry/index.md`:
   - Add or update SPEC-NNNN entry: `status: approved`
   - Add or update ANALYSIS-NNNN entry: `status: completed`

## Quality Gate

Before committing:
- [ ] ANALYSIS shows `readiness_verdict: ready`
- [ ] SPEC `status: approved`; `approval:` block set (`approved_by`, `approved_role`, `approved_at`, `decision_summary` populated)
- [ ] ANALYSIS `status: completed`
- [ ] Registry updated for both artifacts
- [ ] `product_validated: true` confirmed in SPEC frontmatter — not overwritten
- [ ] No open question remains unresolved without documented rationale

**Self-check before proceeding:** Re-read each item above. Confirm each is complete, or state what was skipped and why before continuing to Handoff.

## Handoff

**Before committing — pause for human review:**

Show the user the approved SPEC and ANALYSIS. Then ask:

> **Before we commit — three questions:**
> 1. State the problem this SPEC solves in one sentence, without using any implementation terms. Does that sentence accurately describe the user need?
> 2. Is there a user scenario or edge case not covered by any AC that a developer might reasonably assume is in scope?
> 3. Read the Non-Goals. Is anything listed as out of scope that would make the feature incomplete or unusable without it?
>
> If any answer reveals a gap: return to `/revise-spec SPEC-NNNN ANALYSIS-NNNN`.
>
> If all three check out: "Ready to commit and move to design. Confirm: (1) proceed with the commit? (2) solo project (no push) or team project (push too)?"

**Write back to the source system (only if `source_work` is present in SPEC-NNNN frontmatter) — after the user confirms "ready to commit," before the commit itself:**

This fires on *every* approval of this SPEC — including a re-approval after `/reopen-spec` — not only the first; if no `source_work` block is present, skip this entirely (AC4). It runs here, after the three-question gate has already passed and the user has already confirmed they're ready to commit — not earlier alongside Steps 1–4's local frontmatter bookkeeping — because a write here reaches a real external system of record other people rely on, and should never fire against a SPEC that the gate above could still send back to `/revise-spec`. Read [`templates/reference/external-work-intake.md`](../reference/external-work-intake.md) for the full rules; applied here with two distinct confirmations, since this write (unlike `/create-spec`'s read-only intake) must never happen without the human having seen and approved the exact content first:

- **Ask first:** "This SPEC originated from `[source_work.system]:[source_work.id]`. Update that ticket with the approval decision?"
  - **Declined:** generate a plain-text copy-paste summary for the human to apply manually. Record `write_back: {attempted: false, result: copy_paste_fallback}`. Skip the rest of this step.
- **If yes:** check for working write access to `[source_work.system]`, the same capability-check-then-attempt way AC1 checks read access.
  - **No working access:** fall back to the copy-paste summary. Record `write_back: {attempted: false, result: copy_paste_fallback}`.
  - **Access exists:** show the human exactly what will be written and require their explicit confirmation of that exact content before writing anything.
    - **Declined at this second confirmation:** fall back to the copy-paste summary — never write without this confirmation. Record `write_back: {attempted: false, result: copy_paste_fallback}`.
    - **Confirmed:** attempt the write. **Any outcome short of a confirmed-complete write is treated as failure** — per D9, there is no partially-updated state; a write that updates some fields but not others is a failure, never a partial success:
      - Confirmed-complete success → record `write_back: {attempted: true, result: updated, written_at: [today's date]}`.
      - Failure (any reason, including partial, or the ticket having changed since it was last read): reduce it to a sanitized category (`authentication_failed` / `not_found` / `network_error`) before showing it to the human or recording it — **never raw connector output**, which can echo credential fragments (D11, extended here from its original fetch-failure scope to cover write-back failures too). Report clearly rather than failing silently. Fall back to the copy-paste summary as well. Record `write_back: {attempted: true, result: write_failed, failure_category: [category]}`.
- A declined, inaccessible, or failed write-back **never blocks SPEC approval** — approval proceeds either way.
- Emit `source_work.write_back_result` (`updated` / `copy_paste_fallback` / `write_failed`, per OBS-0001 IT-2) as a structured log field at the point this step resolves. Not emitted at all when no `source_work` exists.
- `write_back` is recorded into SPEC-NNNN's own frontmatter, so it lands in the same commit below — it must be resolved before the `git add`/`git commit` lines run, not after.

**Commit (after user confirms):**

`repo_role: code`:
```
git add sdlc/specs/SPEC-NNNN-slug.md sdlc/analysis/ANALYSIS-NNNN-slug.md sdlc/registry/index.md
git commit -m "SPEC-NNNN: approved — [N review rounds]"
```

`repo_role: coordination` (SPEC and ANALYSIS live thread-scoped, not in the flat `sdlc/specs/`/`sdlc/analysis/` a `repo_role: code` project uses):
```
git add sdlc/THREAD-NNNN/SPEC.md sdlc/THREAD-NNNN/ANALYSIS.md sdlc/THREAD-NNNN/thread.yml sdlc/registry/index.md
git commit -m "THREAD-NNNN: SPEC approved — [N review rounds]"
```
`thread.yml` was created by `/create-spec` but never committed until now — always include it (an already-tracked, unchanged `thread.yml` on a later approval round of the same thread is a harmless no-op in `git add`).

**Push (only if user confirms team context):**
```
git push -u origin thread/NNNN-slug
```

**Note `security_review_required` for the Architect:**
> State the value from ANALYSIS frontmatter. If `yes`: `/security-review` will be triggered after DESIGN is approved — the Architect should be aware before starting `/create-design`.

→ **Architect** → `/create-design SPEC-NNNN ANALYSIS-NNNN`
