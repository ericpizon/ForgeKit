# sdlc/test-plans/

Test plan artifacts. Each TEST-NNNN maps to a PLAN-NNNN and specifies unit, integration, and E2E test cases, coverage expectations, and smoke tests for release validation.

## Commands

| Command | Role | What it does |
|---|---|---|
| `/create-test-plan PLAN-NNNN` | Quality Engineer | Author test plan from implementation plan |
| `/review-test-plan TEST-NNNN` | Quality Engineer | Review test coverage, Happy Path trace, P0 completeness |
| `/address-test-plan-review TEST-NNNN` | Quality Engineer | Address test plan review findings |
| `/approve-test-plan TEST-NNNN` | Quality Engineer | Formal approval; gates `/execute-tests` and `/validate-release` |
| `/validate-release vX.Y.Z` | SRE | Execute smoke tests derived from TEST-NNNN in production |

## Status lifecycle

```
draft → in_review → approved → (execution results recorded post-deploy)
```

## Key fields

- `execution_date` — populated after production validation via `/validate-release`
- `execution_summary` — populated after production validation

## Naming convention

`TEST-NNNN-[kebab-case-title].md`
