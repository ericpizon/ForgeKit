# sdlc/reviews/

Implementation review artifacts produced by `/review-implementation`.

## Contents

Each file in this directory is a structured multi-perspective review for one milestone:

```
IMPL-REVIEW-NNNN-MN.md    # review for PLAN-NNNN milestone MN
```

## Lifecycle

1. `/review-implementation PLAN-NNNN MN` creates `IMPL-REVIEW-NNNN-MN.md`
2. Each address round appends a new section to the same file (Round 1, Round 2, ...)
3. `/approve-implementation PLAN-NNNN MN` reads the latest round's summary to confirm zero blocking findings before proceeding

## Do not edit manually

Review files are written and updated by pipeline commands. Manual edits can corrupt the round-tracking structure that `/approve-implementation` depends on.
