---
id: promote-to-thread
type: command
name: /promote-to-thread
description: Converts a repo-local SPEC into a cross-repo THREAD-NNNN when cross-repo scope is discovered during ANALYSIS or DESIGN — idempotent by construction, safely resumable after an interruption
status: active
version: 1
primary_role: any
output_artifact_type: thread
changelog:
  - v1 — initial authoring. Implements AC8-AC11 (D4/ADR-0001's idempotent-by-construction promotion sequence), T16 (OBS-0001 IT-4), T19b (SEC-0001 SEC-3's reactive `coordination_repo` validation), and T20 (D11's sanitization scope, extended to this command's own coordination-repo errors).
---

# /promote-to-thread

## Purpose

Convert a repo-local SPEC into a `THREAD-NNNN` in a coordination repo, once cross-repo scope is discovered — during ANALYSIS (before the SPEC is ever approved) or during DESIGN (after it is). Every step checks real committed state before acting, so the command is safe to interrupt and re-invoke at any point: it never mints a duplicate `THREAD-NNNN`, and it never leaves the original SPEC ambiguous between promoted and not promoted (D4/ADR-0001).

## When to Invoke

- During `/review-spec`/`/revise-spec` (ANALYSIS stage) or `/create-design`/`/review-design` (DESIGN stage), a reviewer or the TPO/Architect realizes this SPEC's scope actually spans more than one repo
- A prior `/promote-to-thread` run was interrupted — re-invoking resumes it; see Step 2's resume branch

## Usage

```
/promote-to-thread SPEC-NNNN
```

## Required Inputs

- **SPEC-NNNN** — the repo-local SPEC being promoted, in `status: draft` or `status: approved` (AC9 — both are real, reachable trigger points; any other status is handled in Step 0)
- **`project.yml`** — this repo's own `coordination_repo` field, if set (AC8)

## Step 0 — Already-promoted check (AC9's no-op clause)

Read SPEC-NNNN frontmatter first, before anything else. If `status: promoted` already:

> "SPEC-NNNN is already promoted to **[superseded_by]**. No action taken — re-run `/status SPEC-NNNN` if you need the current THREAD state, or work directly in [superseded_by]."

Stop here. Do not mint a second `THREAD-NNNN`, do not re-run any step below. This is a deliberate no-op, not an error — invoking promotion twice on an already-promoted SPEC is a normal thing to accidentally do, not a misuse to reject.

If `status` is anything other than `draft`, `approved`, or `promoted` (e.g. mid-`/reopen-spec`): stop and error clearly — "SPEC-NNNN is in an unexpected state (`[status]`) for promotion. Resolve its current revision cycle before promoting." (TC-EDGE-2 — this is the one state this command refuses outright, since proceeding could corrupt an in-flight single-repo revision.)

Otherwise, continue to Step 1.

## Step 1 — Resolve `coordination_repo` (AC8)

Read this repo's own `project.yml`.

**If `coordination_repo` is already set:** reuse it without re-asking. Nothing else to do in this step.

**If unset:** ask the identical question `/forge-init`'s Phase 2b asks (same full description of what a coordination repo is and does — do not restate a shorter version here that could drift out of sync with that one):

> **Does this project participate — or is it expected to participate — in cross-repo coordination with other repos on your team?** [full Phase 2b description]
>
> Name the existing coordination repo now (`org/repo` shape) — this promotion needs one to actually proceed.

**Validate the answer** against the same safe `org/repo`-shaped pattern Phase 2b uses (`^[\w.-]+/[\w.-]+$`), per SEC-0001 SEC-3/T19b. On a non-match, explain why and re-prompt — do not write an unvalidated value. On a match, persist it as `coordination_repo` in this repo's own `project.yml`, **identically to how AC7's proactive answer is persisted** — this is what makes "asked exactly once" hold across the repo's lifetime (AC8's own stated reason): without this persistence, the *next* SPEC promoted from this same repo would find `coordination_repo` still unset and ask again.

Unlike AC7's `/forge-init`-time question, there is no "not yet" option here — this command cannot proceed without a coordination repo to promote into. If the user has no coordination repo at all, direct them to `/forge-init`'s own coordination-repo setup (on the coordination repo itself) before retrying this command.

## Step 2 — The `pending_promotion` anchor (D4/ADR-0001's core invariant)

**This is the step that makes the entire command idempotent. Read it before touching anything else.**

**If SPEC-NNNN already has `pending_promotion: THREAD-NNNN` set** (a prior run was interrupted after this step but before Step 7): this is a resume.

1. Emit `promote_to_thread.outcome: interrupted` **right now**, for the *prior* attempt — before proceeding with anything else (OBS-0001 IT-4b). The prior run, by construction, never reached its own final step to log this itself; the resuming invocation logs it on the prior run's behalf, the moment it discovers the dangling `pending_promotion`.
2. Do not mint a new `THREAD-NNNN`. The anchored value is the one this resume continues with.
3. Proceed to Step 3, which will detect exactly how much of the sequence already completed and skip accordingly.

**If `pending_promotion` is not set** (a fresh promotion): mint the candidate `THREAD-NNNN` number — scan the coordination repo's `sdlc/` for existing `THREAD-NNNN/` directories and take the next sequential one (the same convenience-allocation-not-collision-guarantee discipline `create-spec.md`'s own coordination path already uses; a CI check on the coordination repo validates uniqueness against HEAD at merge time). **Reading** the coordination repo to determine this number is fine here; **writing** to it is not — nothing is written to the coordination repo until this step's own write completes first:

Write `pending_promotion: THREAD-NNNN` to **SPEC-NNNN's own frontmatter**, and **commit it, in this repo, right now** — before any write to the coordination repo happens. This is the anchor every later step (including a resumed one) checks against. **No write to the coordination repo happens before this commit exists.** This is D4/ADR-0001's one non-negotiable invariant in this entire command: get this order wrong, and the resumability guarantee this command exists to provide is gone.

Any error here (local commit failure — not expected, but not sanitized specially; this never touches the coordination repo) is reported plainly; this step has no cross-repo failure mode.

## Step 3 — Mint the `THREAD-NNNN` directory and copy promoted content

**Check existence first, then — only if it already exists — distinguish "mine" from "collision" before deciding anything else.** Collapsing these into one existence check would treat a genuine collision as if it were this run's own completed work and silently skip the re-mint it actually needs:

1. Does `sdlc/THREAD-NNNN/` already exist in the coordination repo (`THREAD-NNNN` = the value anchored in `pending_promotion`)?
   - **No** → create it now (per [`templates/reference/cross-repo-resolution.md`](../reference/cross-repo-resolution.md)'s resolution algorithm for reaching the coordination repo), write `thread.yml` with an `originated_from: {repo: [this repo], spec: SPEC-NNNN}` field recording this promotion's own identity (the field the collision check below depends on) but an empty `participants: []` — Step 4 populates that list, not this step — then proceed to "Copy the promoted content" below.
   - **Yes** → read `thread.yml`'s `originated_from` field and compare it to this promotion (this repo + `SPEC-NNNN`):
     - **Matches** → this step already completed on a prior, interrupted attempt. Skip straight to Step 4.
     - **Does not match** (claimed by a *different* SPEC's concurrent promotion — a genuine collision) → re-mint a new candidate `THREAD-NNNN` number, update `pending_promotion` to the new value and commit that update in the promoting repo (same mechanics as Step 2), then retry this entire step with the new number. Never error opaquely (DESIGN-0001 Round 2 `[LD-2]`) — this is a rare race, resolved today at the existing CI collision-check layer; this command's own re-mint-and-retry is the cheap, explicit handling of it, not a replacement for that layer.

**Any failure reaching the coordination repo at any point in this step** — unreachable, auth failure, network error — **is reduced to a sanitized failure category before being shown or recorded anywhere** (`authentication_failed` / `not_found` / `network_error`), never raw connector/git output (D11, extended per its own 2026-10-05 amendment to cover this command's coordination-repo errors, not only `/create-spec`'s connector path — T20). Report the sanitized category and stop; `pending_promotion` is already safely committed, so a retry later resumes cleanly from here — nothing is lost by stopping now.

**Copy the promoted content** (only if `sdlc/THREAD-NNNN/SPEC.md` doesn't already exist — same idempotency shape as the directory creation above): write `sdlc/THREAD-NNNN/SPEC.md` from this repo's SPEC-NNNN content. If `ANALYSIS-NNNN` exists for this SPEC (it always does by the time promotion is possible), also write `sdlc/THREAD-NNNN/ANALYSIS.md`. If `DESIGN-NNNN` exists (only when promoted during DESIGN, not ANALYSIS), also write `sdlc/THREAD-NNNN/DESIGN.md`. **This copy is not explicitly itemized as its own step anywhere upstream** — DESIGN-0001's own six-step description and AC9's text both describe "minting the `THREAD-NNNN` directory" without separately calling out that the directory needs this content inside it to be useful at all (Option A's rejected alternative *did* list "copy SPEC content" as its own checkpoint step, for comparison). Folded into this step rather than treated as a separate numbered one, since it shares this step's idempotency shape and failure mode. The original files are never deleted or moved — only copied (AC11).

## Step 4 — `thread.yml` participants (idempotent)

Read `sdlc/THREAD-NNNN/thread.yml` if it exists (it will, from Step 3, unless this is a resume that got further before). Add this repo as a participant **only if it isn't already listed** — write `thread.yml` once per new participant discovered, never duplicate an existing entry. Same coordination-repo failure sanitization as Step 3.

## Step 5 — Re-run the coordination Pre-flight for newly-discovered participants

This is the identical check `/create-spec`'s own `repo_role: coordination` Pre-flight path runs — reused here, not reimplemented, because a promoted SPEC needs exactly the same cross-repo groundwork a SPEC *authored* as a thread from the start would have gotten upfront:

1. Determine the full participant set for this promotion (this repo, plus any other repos the promoted SPEC's own content names).
2. For every participant **not already recorded** in `sdlc/THREAD-NNNN/SPEC.md`'s `inputs.architecture` list: read that participant's `ARCH.md` and add an `inputs.architecture` entry (`repo`/`ref`/`path`, per [`templates/reference/cross-repo-resolution.md`](../reference/cross-repo-resolution.md)).
3. Idempotency: a participant already recorded is skipped — this is what makes a resumed run cheap rather than re-reading every participant's `ARCH.md` from scratch each time it's re-invoked.

Same D11/T20 sanitization as Step 3 applies here too — a participant repo being unreachable is the same risk class (raw git/connector output potentially echoing a credential) as the coordination repo being unreachable; never shown or recorded raw regardless of which repo the error came from.

## Step 6 — Re-examine prior review rounds (AC10)

**Also not itemized in DESIGN-0001's own six-step mechanical sequence** — AC10 is a required acceptance criterion with no corresponding numbered step in that description. Placed here, between Step 5 (cross-repo scope now fully known) and Step 7 (finalization), since this is the first point where "is each prior finding still valid under the newly-discovered scope" is actually answerable, and it must happen before Step 7 commits the SPEC to `promoted` status.

For every completed `## Analysis Round N` section in `ANALYSIS-NNNN`, and every `## Design Review — Round N` section in `DESIGN-NNNN` if it exists: re-examine each finding on substance, not by blanket restart or silent carry-forward (AC10's explicit requirement). For each finding, record one of:

- **Still valid** — the finding and its resolution hold unchanged under the cross-repo scope.
- **Invalidated by the newly-discovered scope** — the finding assumed single-repo scope that no longer holds; state why.
- **Needs re-review** — the newly-discovered scope changes the finding's premise enough that a fresh review pass is warranted; name which round should re-examine it.

Record this disposition directly in the **copied** `sdlc/THREAD-NNNN/ANALYSIS.md`/`DESIGN.md` (Step 3's copies), as a new `## Promotion Disposition — [date]` section appended after the existing review history — never edited into the original review-round sections themselves, which stay as an accurate historical record of what was actually reviewed and when.

## Step 7 — Finalize (AC11, last step per D4)

**Only after Steps 1 through 6 are all confirmed complete** (idempotency checks on each, not re-verification from scratch): update SPEC-NNNN's own frontmatter in this repo —

```yaml
status: promoted
superseded_by: THREAD-NNNN
```

replacing `pending_promotion` (which is removed — its job is done once this field exists). Commit this change in the promoting repo. **This must be the last step, performed only once every other step is confirmed complete** — this is the specific guarantee that an interrupted run never shows `promoted` with an incomplete thread behind it (D4/ADR-0001; TC-ADR-0001-01 tests the specific condition this exists to prevent).

Emit `promote_to_thread.outcome`: `completed` (a fresh run that never hit Step 2's resume branch) or `resumed_and_completed` (a run that did) — at this exact point, its own final step (OBS-0001 IT-4a).

**Leave this SPEC's own git branch untouched** (D8) — promotion is a content-authority transition, not a git-hygiene operation. Do not delete, merge, or otherwise touch the promoting branch.

## Quality Gate

Before considering a promotion run complete:
- [ ] `pending_promotion` was committed in the promoting repo before any write reached the coordination repo (Step 2)
- [ ] `sdlc/THREAD-NNNN/{SPEC,ANALYSIS,DESIGN}.md` (whichever exist) carry the promoted content, not just an empty directory
- [ ] Every participant's `ARCH.md` is recorded in `inputs.architecture` — not just the originating repo's
- [ ] Every prior ANALYSIS/DESIGN review-round finding has an explicit disposition recorded (Step 6) — none silently dropped
- [ ] `status: promoted`/`superseded_by` was written last, only after every other step confirmed complete
- [ ] Any coordination-repo error encountered anywhere in this run was shown/recorded only as a sanitized category, never raw connector/git output
- [ ] The promoting repo's git branch was not touched

**Self-check before proceeding:** Re-read each item above. Confirm each is complete, or state what was skipped and why before continuing to Handoff.

## Handoff

**Show the user a summary before committing anything:**

> **Promoting SPEC-NNNN to THREAD-NNNN.**
>
> [If this is a resume:] Resuming an interrupted promotion — picking up from [first incomplete step].
> Participants discovered: [list]. Coordination repo: [coordination_repo].
> [N] prior review findings re-examined: [still valid] / [invalidated] / [needs re-review].
>
> Confirm: (1) proceed? (2) solo project (no push) or team project (push too)?

**Commit (after user confirms) — this command commits to *two* repos, at different points, never as one combined commit:**

In the promoting repo, immediately at Step 2 (before anything else, per D4) — include `.claude/project.yml` only if Step 1 wrote `coordination_repo` reactively this run:
```bash
git add sdlc/specs/SPEC-NNNN-slug.md
git add .claude/project.yml   # only if coordination_repo was just set reactively in Step 1
git commit -m "SPEC-NNNN: promotion to THREAD-NNNN started (pending_promotion)"
```

In the coordination repo, after Steps 3-6 complete:
```bash
git add sdlc/THREAD-NNNN/
git commit -m "THREAD-NNNN: promoted from [promoting repo]/SPEC-NNNN"
```

In the promoting repo again, at Step 7 (last, per D4):
```bash
git add sdlc/specs/SPEC-NNNN-slug.md
git commit -m "SPEC-NNNN: promoted - superseded_by THREAD-NNNN"
```

**Push (only if user confirmed team context) — push each repo's own commits to its own remote**, in the same order as above.

**Note `coordination_repo` persistence (only if Step 1 asked reactively):**
> `coordination_repo` is now set in this repo's own `project.yml` — committed as part of the first commit above. The next SPEC promoted from this repo will reuse it without asking again.

→ Work continues in **THREAD-NNNN**, in the coordination repo, from here on. The original SPEC-NNNN stays in place, `status: promoted`, as a historical record — never deleted.
