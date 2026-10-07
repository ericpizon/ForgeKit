# sdlc/observability/

Observability plan artifacts. Each OBS-NNNN maps to a DESIGN-NNNN and specifies what must be instrumented, which signals to monitor, dashboard structure, alert definitions, and what must never be logged. Instrumentation tasks from OBS-NNNN become concrete tasks in PLAN-NNNN.

## Commands

| Command | Role | What it does |
|---|---|---|
| `/design-observability DESIGN-NNNN` | SRE | Author observability plan from approved design; Lead Dev confirms task concreteness before committing |

Runs in parallel with `/security-review` after DESIGN-NNNN is approved. Both must complete before `/create-plan`.

## Status lifecycle

OBS-NNNN has no separate `status` field. The lifecycle is:

1. **Drafted** — SRE authors OBS-NNNN; not yet committed
2. **LD-approved** — Lead Developer explicitly confirms instrumentation task specificity; file committed to branch

`validation_status` transitions separately:
- `not_yet_validated` — set at creation
- `validated_at_release` — updated by `/validate-release` after production smoke tests pass

No separate review/approve commands — the inline Lead Dev confirmation is the gate before commit.

## Key fields

- `validation_status: not_yet_validated` — updated to `validated` by `/validate-release`
- `target_design_version` — links to the DESIGN version this plan was authored against

## Naming convention

`OBS-NNNN-[kebab-case-title].md`
