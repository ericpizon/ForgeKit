---
id: create-adr
type: command
name: /create-adr
description: Authors a standalone Architectural Decision Record — no parent DESIGN or SPEC required, reusable anywhere a durable decision needs recording independent of a specific build
status: active
version: 2
primary_role: any
output_artifact_type: adr
changelog:
  - v2 — clarified: this command always writes to the flat `sdlc/adr/`, identically in `repo_role: code` and `repo_role: coordination` (the coordination-repo directory list now includes `sdlc/adr/` for exactly this case). This is a genuinely different location from `create-design.md`'s thread-scoped nested ADRs (`sdlc/THREAD-NNNN/adr/`), which remain unchanged — a standalone ADR has no thread to scope to in the first place. Found during a full-pipeline drift audit: this command had no `repo_role` branch at all despite `/create-investigation` (one of its two callers) being explicitly usable from a coordination repo.
  - v1 — initial authoring. Implements AC17 (standalone ADR authorship, same required sections as a DESIGN-nested ADR, no `target_design` pointer). SEC-0001 SEC-2's slug-sanitization extension (T18b) applied to this command's own slug derivation.
---

# /create-adr

## Purpose

Author an Architectural Decision Record with no parent DESIGN or SPEC — same shape and rigor as the ADRs `create-design.md` authors inline, but usable anywhere a durable decision needs recording independent of any specific build: most commonly as `/create-investigation`'s `decision: adr` hand-off, but not exclusive to it (ADR-0003's own stated reasoning for making this a standalone command rather than bundling it into Investigation).

## When to Invoke

- `/create-investigation` concludes with `decision: adr`
- Any other point where a durable architectural decision needs recording and no DESIGN is in progress to nest it inside

## Usage

```
/create-adr <slug>
```

`<slug>` is a short, human-provided description of the decision (e.g. `event-schema-versioning-strategy`) — used directly in the output filename, not derived from external content. **If this invocation originated from `/create-investigation`'s `--from`-sourced content** (the Investigation itself started from an external ticket), treat the slug the same as any other externally-influenced title: sanitize to `[a-z0-9-]` only before use (SEC-0001 SEC-2/T18b) — do not assume a human-provided argument is automatically safe if its wording was itself suggested by fetched or pasted content.

## Required Inputs

- **The decision itself** — what was decided, and the real alternatives it was weighed against. Does not need to arrive from any particular upstream artifact; a standalone ADR has no required predecessor.

## Steps

Same five required sections `create-design.md` already uses for a DESIGN-nested ADR — reused here verbatim, not a lighter or heavier variant:

1. **Context.** What question or problem made this decision necessary. Enough background that a reader with no memory of the surrounding discussion can follow why this needed deciding at all.
2. **Decision.** The actual decision, stated plainly and specifically enough to act on — not a restatement of the Context, and not hedged across multiple options.
3. **Alternatives Considered.** Every real alternative that was weighed, and why each was rejected. A standalone ADR with only one "alternative" that's a strawman is not meeting this bar — if there was genuinely only one viable approach, state why the obvious alternatives don't apply, don't invent weak ones to fill the section.
4. **Consequences.** What this decision costs or constrains going forward — not just its benefits. Every real decision trades something away; name it.
5. **Status Rationale** and **Revisit Triggers.** Why this decision's current status (`accepted`/`superseded`/etc.) is correct right now, and the specific conditions under which it should be revisited — not "if anything changes," but named conditions a future reader can actually check against.

## Output Contract

Frontmatter must include: `id`, `type: adr`, `schema_version: 1`, `status: accepted` (a standalone ADR is authored as a decision already made, not a draft under review — if the decision itself is still contested, that belongs in a DESIGN/ANALYSIS review cycle first, not here), `created`. **No `target_design` field** — this is the one structural difference from a DESIGN-nested ADR, and it is an omission, not a `null` placeholder; the field does not appear in frontmatter at all (AC17's explicit requirement). If this ADR resulted from an Investigation, note the originating `INV-NNNN` in the body's Context section as prose, not as a new frontmatter field — `create-investigation.md`'s own `linked_adr` field is the authoritative pointer in the other direction; this ADR doesn't need a reciprocal structured field to stay discoverable.

Body: the five required sections above, in order: Context, Decision, Alternatives Considered, Consequences, Status Rationale, Revisit Triggers — identical heading set to a DESIGN-nested ADR, so nothing downstream that reads an ADR (e.g. a future `/review-design`'s Options Analysis Review, which checks for "Alternatives Considered" explicitly) needs to special-case a standalone one.

**Output path — identical regardless of `repo_role`:** always `sdlc/adr/ADR-NNNN-slug.md`, the flat location, never `sdlc/THREAD-NNNN/adr/`. The thread-scoped path is exclusively `/create-design`'s — a standalone ADR has no thread to scope to, by definition, whether authored in a `repo_role: code` project or a `repo_role: coordination` one.

## Quality Gate

Before finalizing:
- [ ] All five required sections present, each with substantive content — not one-liners
- [ ] Alternatives Considered names real alternatives, not strawmen
- [ ] Consequences names a real cost or constraint, not only benefits
- [ ] Revisit Triggers names specific, checkable conditions — not "if anything changes"
- [ ] No `target_design` field anywhere in frontmatter
- [ ] If the slug (or the decision's own wording) was influenced by `--from`-sourced content: sanitized to `[a-z0-9-]` only in the filename

**Self-check before proceeding:** Re-read each item above. Confirm each is complete, or state what was skipped and why before continuing to Handoff.

## Handoff

**Show the user the draft before committing:**
> Display the ADR. Ask: "Review the draft ADR. Confirm when ready. Confirm: (1) proceed with the commit? (2) solo project (no push) or team project (push too)?"

**Commit (after user confirms):**
```bash
git add sdlc/adr/ADR-NNNN-slug.md
git commit -m "ADR-NNNN: [one-line decision summary]"
```

Same sequential `ADR-NNNN` numbering `create-design.md` already uses for DESIGN-nested ADRs — one shared sequence, not a separate counter for standalone ones (scan `sdlc/adr/` for the next available number, same collision discipline as `THREAD-NNNN`/`XDEP-NNNN`).

**Push (only if user confirms team context):**
```bash
git push origin main
```

**If invoked from `/create-investigation`'s hand-off:** return control there — it records this ADR's ID in `INV-NNNN`'s own `linked_adr` field and commits both files together, not as two separate commits.
