# sdlc/incidents/

Incident records. Each INC-NNNN captures a production incident — severity, timeline, impact, resolution, and links to the postmortem. Incidents are declared when active production impact warrants a faster response than the standard development pipeline.

## Incident criteria

Declare an incident — not a bug — when at least one of the following is true **and** the normal pipeline is too slow to respond:

- Service unavailable or error-rate elevated for users
- Data integrity issue affecting live data
- Active or suspected security breach
- SLO violation in progress
- A bug that appeared benign in staging is critical at production scale

## Commands

| Command | Role | What it does |
|---|---|---|
| `/create-incident` | SRE | Declare a new incident — 3 questions, then INC-NNNN is live |
| `/escalate-to-incident BUG-NNNN` | SRE | Convert an existing bug to an active incident |
| `/update-incident INC-NNNN` | SRE | Append a timestamped timeline event; optionally update severity or status |
| `/resolve-incident INC-NNNN` | SRE | Record mitigation, calculate duration, determine postmortem requirement |
| `/create-postmortem INC-NNNN` | SRE | Author PM-NNNN from a resolved incident (5 Whys root cause) |

## Severity tiers

| Severity | Meaning |
|---|---|
| SEV-1 | Full outage, data loss, or active security breach |
| SEV-2 | Major feature outage or significant degradation, no workaround |
| SEV-3 | Partial degradation, workaround available |
| SEV-4 | Minor impact, most users unaffected |

## Status lifecycle

```
active → monitoring → resolved
```

`monitoring` = contained but not fully resolved (e.g., mitigation applied, watching for recurrence).

## Relationship to bugs

- Bugs can be escalated to incidents via `/escalate-to-incident BUG-NNNN`
- The linked bug's status becomes `incident_active` during the incident
- On resolution: bug returns to `open` (fix still needed) or advances to `fix_deployed` (hotfix permanent)

## Naming convention

`INC-NNNN-[kebab-case-title].md`
