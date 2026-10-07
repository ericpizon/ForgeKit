# sdlc/analysis/

Specification analysis artifacts. Each ANALYSIS-NNNN maps to a SPEC-NNNN and breaks down requirements into technical components, identifies risk flags, and determines whether a security review is required. It is the primary input to `/security-review` and provides supplementary context for `/create-design`.

## When created

ANALYSIS-NNNN is produced during the spec review cycle by the Lead Developer — typically as part of the `/review-spec` discussion before design begins. It is required input for `/security-review` when `security_review_required: yes`.

## Status lifecycle

```
draft → completed
```

## Key fields

- `security_review_required: yes | no` — determines whether `/security-review` is triggered
- Risk Flags section — specific concerns that security review must address

## Naming convention

`ANALYSIS-NNNN-[kebab-case-title].md`
