# sdlc/plans/

Implementation plan artifacts. Each PLAN-NNNN maps to a DESIGN-NNNN and breaks the feature into milestones with tasks, acceptance criteria, and implementation sequencing.

## Commands

| Command | Role | What it does |
|---|---|---|
| `/create-plan DESIGN-NNNN` | Lead Dev | Author implementation plan from approved design |
| `/review-plan PLAN-NNNN` | Lead Dev | Multi-perspective review: Developer, QE, and SRE viewpoints |
| `/address-plan-review PLAN-NNNN` | Lead Dev | Address plan review findings |
| `/approve-plan PLAN-NNNN` | Lead Dev | Formal approval; unlocks `/implement-plan` |
| `/implement-plan PLAN-NNNN M1` | Developer | Execute a milestone: branch, implement, verify, commit |
| `/execute-tests PLAN-NNNN M1` | Developer | Run unit and E2E tests against the milestone implementation |
| `/address-test-failures PLAN-NNNN M1` | Developer | Fix failing tests, re-verify |
| `/review-implementation PLAN-NNNN M1` | Lead Dev | Code review against plan, design, and quality bar |
| `/address-implementation-review PLAN-NNNN M1` | Developer | Address implementation review findings |
| `/approve-implementation PLAN-NNNN M1` | Lead Dev | Approve milestone; commit implementation + IMPL-REVIEW |
| `/create-pr PLAN-NNNN M1` | Developer | Open PR for the milestone branch |
| `/close-milestone PLAN-NNNN M1` | Developer | Record PR merged; advance to next milestone or Release Planning |

## Milestone status lifecycle

```
pending → in_progress → approved → tests_passed → pr_open → pr_merged
                              ↘ test_failed → (fix) → tests_passed
```

## Naming convention

`PLAN-NNNN-[kebab-case-title].md`

## Milestone branch convention

`feature/PLAN-NNNN-M[N]`
