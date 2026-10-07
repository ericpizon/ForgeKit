# sdlc/bugs/

Defect reports and fix tracking records. Each BUG-NNNN is filed by anyone, triaged by the Lead Developer, fixed through the standard plan → implement → review → test → release pipeline, and verified post-release by the QE.

## Status lifecycle

```
reported → accepted → in_progress → fix_merged → pending_verification → verified
         ↘ rejected (not-a-bug / duplicate / wont-fix / out-of-scope)
         ↘ needs_info → (accepted or rejected after follow-up)
                                                       ↘ reopened → accepted
```

**Incident-path statuses** (set by the incident workflow, not the standard fix path):

```
accepted → incident_active    ← set by /escalate-to-incident
incident_active → open        ← set by /resolve-incident when fix still needed
incident_active → fix_deployed ← set by /resolve-incident when hotfix is permanent
fix_deployed                  ← terminal; no further pipeline action required
```

## Commands

| Command | Role | What it does |
|---|---|---|
| `/create-bug` | QE / anyone | File a defect report — creates BUG-NNNN in `reported` state |
| `/triage-bug BUG-NNNN` | Lead Developer | Accept or reject; confirm severity, priority, and fix path |
| `/release-hotfix BUG-NNNN` | Lead Developer | Full pipeline for Critical/P0 bugs (minimal PLAN, immediate patch release) |
| `/escalate-to-incident BUG-NNNN` | SRE | Convert an active production bug to an INC-NNNN incident |
| `/verify-bug BUG-NNNN` | QE | Confirm fix holds in production; close or reopen |

## Fix paths by priority

- **P0/Critical** — `/release-hotfix`: minimal PLAN, full pipeline, patch release immediately
- **P1** — new milestone on existing PLAN or new PLAN; patch release
- **P2/P3** — task added to next planned milestone; ships with next feature release
- **Active production impact** — `/escalate-to-incident`: incident response takes over; standard fix path resumes after resolution if needed

## Naming convention

`BUG-NNNN-[kebab-case-title].md`
