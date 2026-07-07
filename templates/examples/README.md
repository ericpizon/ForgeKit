# ForgeKit Example Artifacts

These files show what high-quality ForgeKit pipeline output looks like. Use them to calibrate your own artifact quality before starting a new thread.

| File | Artifact | Stage |
|---|---|---|
| `SPEC-0001-api-tokens.md` | Feature specification | Stage 1 — approved, product-validated |
| `PLAN-0001-api-tokens.md` | Implementation plan | Stage 5a — approved |
| `IMPL-REVIEW-0001-M1.md` | Implementation review, M1 | Stage 6 — Round 1 findings + address summary + Round 2 approval |

## The fictional project

All examples use **Fieldstone** — a fictional developer-facing API product built on TypeScript / Next.js App Router / PostgreSQL (Drizzle ORM). The feature is **Personal API Token Management**: allowing developers to generate named bearer tokens and authenticate programmatic API calls without session cookies.

Artifacts referenced but not shown: `DESIGN-0001`, `OBS-0001`, `SEC-0001`, `TEST-0001`, `ANALYSIS-0001`. These are omitted to keep examples scoped; their IDs appear in the PLAN and IMPL-REVIEW exactly as they would in a real pipeline.

## What to look for

- **SPEC-0001** — problem framing (no solution terms), specific non-goals usable as decision rules, bidirectional AC traceability, security and observability sections that constrain DESIGN before a designer touches anything, and a complete Product Validation record.
- **PLAN-0001** — `pipeline_status` frontmatter snapshot, task IDs that trace to ACs, observability and security tasks that reference source artifact IDs rather than copying descriptions, the Happy Path Trace catching coverage gaps, and a DoD table with verbatim AC text.
- **IMPL-REVIEW-0001-M1** — multi-perspective findings with specific file and line references, null suppression (perspectives with no findings are listed clean rather than showing empty sections), the address summary as an inline record, and Round 2 approving without re-reviewing unchanged code.
