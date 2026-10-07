# Getting Started with ForgeKit

This guide is for your first week. It covers the two adoption paths — solo and team — and gives you concrete "what do I do on day one?" guidance rather than repeating what the README already says.

---

## Before you start

**ForgeKit is a discipline tool, not a shortcut.** It makes AI-assisted delivery trustworthy by putting structure around the decisions. That structure takes more time upfront than firing off a chat prompt. It pays off when something goes wrong mid-implementation and you can trace every decision back to its source — or when someone joins the thread three weeks later and can orient in minutes instead of hours.

If you want to use AI to go fast without thinking about process, ForgeKit is not what you want. If you want to go fast *and* be able to explain what you built and why, read on.

**Three things that determine whether this works:**

1. **You invest in ARCH.md.** Every role persona reads it before doing any work. A shallow ARCH.md produces shallow designs and plans. If your ARCH.md is incomplete, fix it first — everything downstream inherits the gap. For brownfield projects, `/forge-init` scans your codebase and drafts ARCH.md; your job is to read the draft carefully and approve it only when it accurately reflects how your system actually works.

2. **You treat quality gates as checkpoints, not ceremonies.** The approve-* commands ask specific questions that cannot be answered reflexively. If you find yourself approving without actually answering them, you're using ForgeKit as a rubber stamp and the audit trail it produces is worthless. The questions exist because they catch the things that cause expensive problems later.

3. **You stay in the pipeline.** It is tempting to skip from spec to code when a feature feels small. Don't. The planning artifacts are where implementation surprises get caught before they become blockers. A feature that "feels small" often isn't — and the only way to know is to plan it.

---

## Solo path

You play every role yourself, switching hats as you move through the pipeline. This sounds inefficient. It isn't — the role-switching is the work.

**Why roles matter when you're alone:**

The TPO and Lead Developer see the same spec differently. The TPO asks "does this describe the right outcome?" The Lead Developer asks "can this be implemented without making decisions the spec should have made?" When you wear both hats on your own spec, you catch things you wouldn't catch staying in one mode. This dual-perspective review is where solo developers catch ambiguous acceptance criteria, missing non-goals, and tasks that would block each other mid-implementation.

**Day one:**

```
1. Run /forge-init in your project directory.
2. Answer the setup questions. For the stack, be specific — "unknown" is fine for 
   fields you'll add later, but the more you fill in now, the better the role 
   context will be.
3. For brownfield projects: read the ARCH.md draft carefully before approving. 
   Add anything the scanner missed — hidden constraints, off-limits modules, 
   known tech debt. This is your one chance to ground everything.
4. Run /create-spec with the first feature you want to build.
```

**If you ran `/forge-init` from ForgeKit's own directory** (passing your project's path as an argument, rather than opening Claude Code directly inside your project) — stop before step 4 and open a new session rooted in your project directory instead. `CLAUDE.md` auto-loads based on the session's working directory, and it won't find your project's generated context if the session is still rooted in ForgeKit. This also matters across context compaction: a session rooted in your project gets its `CLAUDE.md` reloaded automatically when context resets; a session rooted elsewhere doesn't, and silently loses that ambient context for the rest of the thread.

After `/create-spec` completes, you have a draft spec. Before calling it done, switch hats:

> *Put on the Lead Developer hat.* Read the spec as someone who has to implement it. Find the AC that can be interpreted two ways. Find the non-goal that's too vague to use as a decision rule. That's the revision you need before review.

Then run `/review-spec` — even though you're alone, the analysis artifact it produces is what drives the plan. Don't skip it. Don't skip `/validate-spec` either — the product validation questions are where you confirm you're building the right thing, not just building the thing right.

**Solo commit workflow:**

Work locally. Commit at each approve-* step. Do not push after every commit — push when you want a remote backup or when you're sharing work with someone. The pipeline doesn't require a remote; the git history is what matters.

Branch naming still applies even solo: `thread/NNNN-slug` for the SDLC thread, `feature/PLAN-NNNN-M1` for milestone implementation. It keeps your git history readable when you look back in three months.

**Habits that help:**

- Use `/status PLAN-NNNN` at the start of every session. Cold starts happen constantly when you work alone — this orients you in under 30 seconds without rereading everything.
- Don't try to hold the whole pipeline in one session. Each command is a checkpoint. End the session at a command boundary and resume with `/status`.
- When something discovered during implementation invalidates the spec or design, use the back-cycle commands — `/reopen-spec`, `/reopen-design`, `/reopen-plan`. Don't silently diverge. The discovery record is how you remember what you learned and why you changed course.

---

## Team path

ForgeKit's pipeline is an async coordination tool. The artifacts are the handoffs. When the TPO finishes a spec, the Lead Developer reads the ANALYSIS artifact — not a Slack message explaining the spec. When implementation review completes, the address summary in the IMPL-REVIEW is what the reviewer reads in the next round — not a verbal update.

**Day one for the team:**

```
1. One person runs /forge-init (usually the Lead Developer or whoever owns setup).
2. Commit the generated files and push to the main branch:
   git add .claude/ sdlc/ CLAUDE.md PIPELINE.md
   git commit -m "chore: initialize ForgeKit SDLC pipeline"
   git push origin main
3. Everyone on the team reads PIPELINE.md. It is the canonical reference for 
   what command runs when and who runs it.
4. Assign roles before the first thread starts (see below).
```

**Role assignment:**

You don't need one person per role. Two people can cover all eight roles between them on a small team. What matters is that everyone knows who plays which role for a given thread, and that the person playing a role actually adopts its mental model — not just their own perspective with a different label.

Common small-team patterns:

| Team size | TPO | Architect | Lead Dev | Developer | QE | Security | SRE | DBA |
|---|---|---|---|---|---|---|---|---|
| 2 people | Person A | Person B | Person B | Person A/B | Person A | Person B | Person B | Person B |
| 3 people | Person A | Person B | Person B | Person C | Person A | Person B | Person B | Person B |
| 4–5 people | Product-side person | Most senior engineer | Senior engineer | Any engineer | QA-inclined person | Security-aware person | DevOps-inclined person | Data-model owner |

For enterprise teams where these roles are already staffed: the pipeline maps directly to your existing org structure. The Architect who does design review is the same Architect who uses `/review-design`. No new roles, just structured handoffs.

**How async handoff works:**

The thread branch (`thread/NNNN-slug`) carries all SDLC artifacts. When the TPO pushes a new spec draft, the Lead Developer pulls the branch and runs `/review-spec`. The ANALYSIS artifact the command produces is committed to the branch. The TPO pulls, reads the analysis, makes revisions, pushes. No synchronous meeting required unless the feedback surface is large enough to warrant one.

When a review has findings that need discussion before the author can address them — especially blocking findings from `/review-design` or `/review-implementation` — handle the discussion in whatever channel your team uses, then record the resolution in the address artifact before moving on. The artifact is the record; the conversation is how you got there.

**Git workflow for teams:**

- `main` — SDLC artifacts accumulate here via approve-* commits until implementation starts; milestone PRs target main
- `thread/NNNN-slug` — created at `/create-spec`, carries all planning artifacts through approve-plan; everyone working on the thread shares this branch
- `feature/PLAN-NNNN-M1`, `feature/PLAN-NNNN-M2` — one branch per milestone, branched from main after previous milestone merges

The single-commit-per-phase rule matters most for teams: the full spec phase (all review rounds, address notes, approval) is one commit. That commit is the complete record of how the spec was produced. Reviewing `git log` shows every phase decision in sequence without noise.

---

## Your first thread

Regardless of solo or team, the first thread follows this sequence. The first one takes longer than subsequent threads because you're building ARCH.md literacy alongside the artifacts.

**Phase 1 — Specification (Days 1–2)**

1. `/create-spec` — TPO authors the spec. Read ARCH.md for stack constraints before writing a single AC.
2. `/review-spec SPEC-NNNN` — Lead Dev reviews; produces ANALYSIS-NNNN. Expect one revision cycle on your first spec — the Lead Dev will find ACs that are ambiguous at implementation time.
3. `/revise-spec SPEC-NNNN` — TPO addresses the analysis findings.
4. `/validate-spec SPEC-NNNN` — TPO runs product validation. The two-tier questions are where you confirm the spec represents real product intent. Don't rush this.
5. `/approve-spec SPEC-NNNN` — TPO approves. Answer the three confirmation questions honestly. If a question exposes a gap, go back to revise-spec; don't approve past it.

**Phase 2 — Design (Days 2–3)**

6. `/create-design SPEC-NNNN` — Architect produces options analysis, gets confirmation, then writes the full design. Read ARCH.md first — it tells you which patterns are settled and which need a new ADR.
7. `/review-design DESIGN-NNNN` — multi-perspective review. Produces `revision_requested` or clean. One revision cycle is normal.
8. `/approve-design DESIGN-NNNN` — Architect approves; updates ARCH.md with any new cross-cutting patterns.

Stages 4a (OBS) and 4b (SEC) run in parallel after design approval.

**Phase 3 — Planning and testing (Days 3–4)**

9. `/create-plan` — Lead Dev produces the ordered task breakdown. Read DESIGN-NNNN and OBS-NNNN in full before writing tasks. The Happy Path Trace will catch gaps.
10. `/create-test-plan PLAN-NNNN` — QE produces test cases against the AC coverage map.

**Phase 4 — Implementation (Days 4–N)**

Each milestone is independent: implement → execute-tests → address-test-failures (if needed) → review-implementation → address-implementation-review (if needed) → approve-implementation → create-pr → close-milestone.

Use `/status PLAN-NNNN` at the start of each session during implementation. It reads frontmatter and tells you exactly where you are and what to run next.

---

## What slows people down

**Skipping validate-spec.** It's new and feels like a ceremony. It isn't. The two-tier questions surface misalignment between what was written and what was intended. One question in validate-spec can save two rounds of implementation review.

**Approving past blockers.** The approve-* confirmation questions have a "go back" path. Use it. An approval that papers over a gap doesn't make the gap go away — it makes it more expensive to find later when it shows up in code review or in production.

**Letting ARCH.md go stale.** Every new ARCH entry must be date-stamped. Every superseded pattern must be moved to the Superseded section. If ARCH.md drifts from what the system actually looks like, role personas operating from it will give wrong advice. The Architect's update obligation after `/approve-design` exists for this reason.

**Treating back-cycles as failures.** `/reopen-spec`, `/reopen-design`, `/reopen-plan` exist because real engineering work discovers invalid assumptions. Using them is correct behavior. Silently diverging from a design because implementation revealed a problem — without going through a back-cycle — is the failure state. The discovery record the back-cycle commands produce is part of the audit trail.

**Working across session boundaries without `/status`.** AI sessions don't have memory across conversations. Running `/status PLAN-NNNN` is the cold-start protocol. It reads only frontmatter — not full artifacts — and emits the current pipeline state and the exact next command to run. It takes under 30 seconds and prevents you from running the wrong command because you forgot where you were.

**Implementing before the test plan is approved.** The test plan drives what test files get written during `/implement-plan`. If TEST-NNNN isn't approved before implementation starts, the developer is guessing at test coverage. `/implement-plan` blocks on this explicitly — if you're hitting that block, finish the test plan, don't work around the check.

---

## Hybrid mode: when your SPEC or BUG lives in Jira

If your team manages requirements in Jira, Linear, or another tracker, use **hybrid mode** (set during `/forge-init`). In hybrid mode:

- SPEC, BUG, and INC live in your external tool as the source of truth
- All technical artifacts (DESIGN, PLAN, TEST, SEC, OBS, IMPL-REVIEW, RELEASE-PLAN) remain as `.md` files in the repo
- Commands that read a SPEC will prompt you to paste the content in; commands that produce review feedback will output content for you to copy back to the ticket

The pipeline behavior is identical. The difference is in where the artifacts live, not in what gets produced or reviewed.

The CLAUDE.md block installed by `/forge-init` carries the hybrid mode context — individual commands don't need to be reconfigured.

---

## When your feature spans more than one repo

Everything above assumes one repo. You don't need to know upfront whether a feature will stay that way — most don't. If partway through ANALYSIS or DESIGN it turns out the feature actually needs a second repo (a shared event contract, a cross-service API), run `/promote-to-thread SPEC-NNNN`. It converts your repo-local SPEC into a `THREAD-NNNN` in a dedicated coordination repo (a repo with no application code, just shared cross-repo artifacts), re-running the cross-repo groundwork your SPEC would have gotten upfront had you known. The original SPEC stays in place, marked `promoted`, pointing at the thread — nothing is deleted.

This applies the same way whether you're solo (you'll set up the coordination repo yourself, the first time any of your repos needs one — `/promote-to-thread` asks and remembers the answer) or on a team (the coordination repo is usually already set up by whoever first needed cross-repo coordination; your repo's `project.yml` just needs to point at it, which `/promote-to-thread` does reactively the first time it's needed).

Once a thread exists, a coordination repo can also host:
- **Routine environment promotions** (`/promote-to-environment`) — a lightweight, append-only record for a `governance: promotion` environment (a shared QA, say), proportional to how often it actually happens — not the full RELEASE-PLAN ceremony that's still exactly right for `governance: release`.
- **Investigation records** (`/create-investigation`, `/create-adr`) — spike or research work that may conclude without any implementation, not forced through a SPEC template that presupposes a build is coming.

See `PIPELINE.md`'s "Multi-repo coordination," "Promotion," "Environment governance," and "Investigation" sections for the full command reference — none of it changes how you work day to day in a normal, single-repo thread.

---

## Quick reference — first week

| Day | What you do | Command(s) |
|---|---|---|
| 1 | Initialize pipeline, approve ARCH.md, draft first spec | `/forge-init`, `/create-spec` |
| 1–2 | Review cycle, analysis, revisions | `/review-spec`, `/revise-spec`, `/validate-spec`, `/approve-spec` |
| 2–3 | Architecture design, options analysis, review | `/create-design`, `/review-design`, `/address-design-review`, `/approve-design` |
| 3 | OBS + SEC in parallel | `/design-observability`, `/security-review` |
| 3–4 | Implementation plan + test plan | `/create-plan`, `/review-plan`, `/approve-plan`, `/create-test-plan`, `/approve-test-plan` |
| 4+ | Implementation, one milestone at a time | `/implement-plan`, `/execute-tests`, `/review-implementation`, `/approve-implementation`, `/create-pr` |

At any point: `/status PLAN-NNNN` — where am I and what do I run next.
