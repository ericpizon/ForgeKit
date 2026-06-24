# sdlc/security/

Security review artifacts. Each SEC-NNNN maps to a SPEC-NNNN and is produced when a feature touches authentication, secrets, user data, or API boundaries.

## Commands

| Command | Role | What it does |
|---|---|---|
| `/security-review SPEC-NNNN DESIGN-NNNN ANALYSIS-NNNN` | Security | Assess attack surface, auth coverage, data exposure; produce findings |

## When security review is triggered

The project.yml `pipeline.security_review` setting defines the default policy:
- `conditional` — run when `security_review_required: yes` is set in ANALYSIS-NNNN
- `always` — run for every feature thread (ANALYSIS still sets `security_review_required: yes` automatically)
- `never` — never run (not recommended for production projects)

Per feature, the actual trigger is the `security_review_required` verdict in ANALYSIS-NNNN, set by `/review-spec`. `/approve-design` checks this field and triggers `/security-review` if `yes`.

## Status lifecycle

```
draft → completed
```

## Naming convention

`SEC-NNNN-[kebab-case-title].md`
