# sdlc/postmortems/

Postmortem reports authored by the SRE role after resolved production incidents.

## Contents

```
PM-NNNN-short-title.md    # one postmortem per incident
```

## When postmortems are required

Postmortem requirements are determined at incident resolution time by `/resolve-incident`, based on the incident's severity and the `pipeline.postmortem` threshold configured in `project.yml`:

| `pipeline.postmortem` value | Requirement |
|---|---|
| `always` | Required for every incident |
| `SEV-2` *(default)* | Required for SEV-1 and SEV-2 incidents |
| `SEV-3` | Required for SEV-1, SEV-2, and SEV-3 incidents |
| `SEV-4` | Required for all incidents |
| `never` | Always optional |

For incidents below the configured threshold, postmortems are optional but encouraged when the incident reveals a systemic gap.

## Format

Each postmortem uses 5 Whys root cause analysis and records:
- **5 Whys chain** — symptom → root cause, with each "why" answered explicitly
- **Root cause** — the systemic gap in process, system, or structure
- **Contributing factors** — conditions that allowed the root cause to have impact
- **Timeline** — events from detection to resolution
- **What went well / poorly** — response retrospective
- **Action items** — typed (code-fix, feature, observability, infrastructure, process), owned, linked to downstream artifacts

## Action item tracking

- `code-fix` items → `/create-bug` inline at postmortem creation time
- `feature` items → deferred to `/create-spec`
- `observability`, `infrastructure`, `process` items → tracked in PM-NNNN, no pipeline artifact

Open code-fix and feature action items surface in `/sync-registry` and as named risks in `/create-release-plan`.

## Blameless

Postmortems are blameless. Name systems, decisions, and processes — not individuals.
