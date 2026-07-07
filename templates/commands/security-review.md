---
id: security-review
type: command
name: /security-review
description: Security role command to produce SEC-NNNN — a full threat model and findings report for a DESIGN; only invoked when ANALYSIS sets security_review_required yes
status: active
version: 1
primary_role: security
reviewing_roles:
  - lead-developer
  - architect
output_artifact_type: security-review
---

# /security-review

## Purpose

Produce a full security review (SEC-NNNN) covering the auth surface, secrets handling, trust boundaries, and attack vectors introduced by a DESIGN. This command is **conditional** — invoked only when ANALYSIS-NNNN sets `security_review_required: yes`. The output gates PLAN-NNNN authorship: open Critical/High findings block the plan until mitigated or explicitly accepted as risk with a named approver.

## When to Invoke

- ANALYSIS-NNNN `security_review_required: yes` AND DESIGN-NNNN is `approved`

Do not invoke when `security_review_required: no`. Record the negative trigger decision in the ANALYSIS as sufficient documentation.

Pass artifact IDs as arguments: `/security-review SPEC-NNNN DESIGN-NNNN ANALYSIS-NNNN`.

## Required Inputs

- **SPEC-NNNN** — approved
- **DESIGN-NNNN** — approved
- **ANALYSIS-NNNN** — completed; read the Risk Flags section — these are the primary inputs to the security review scope
- **ARCH.md** — read for the project's auth model, secrets handling patterns, and known security conventions

## Steps

1. **Scope definition** — Identify which surfaces are in scope: authentication mechanisms, authorization model, credential generation/storage/comparison, trust boundary placement, input validation on security-sensitive fields, and any data retention concern.
2. **Threat enumeration** — For each surface, identify plausible attacker capabilities (unauthenticated, authenticated-but-unauthorized, insider) and what they could achieve. Enumerate threats across all of the following categories — do not skip a category without a stated reason:
   - **Authentication bypass** — can an unauthenticated caller reach protected resources?
   - **Authorization bypass / privilege escalation** — can an authenticated user access or modify resources they don't own?
   - **Injection** — SQL, command, path traversal, template injection at any input boundary
   - **Credential and sensitive data exposure** — in logs, error responses, API responses, or persisted storage
   - **Enumeration** — timing differences, response differentiation, or predictable IDs that reveal existence of private resources
   - **Timing attacks** — secret comparison or token validation that leaks information through execution time
   - **Business logic abuse** — workflow steps that can be skipped, replayed, or invoked out of order
   - **Security header and middleware gaps** — missing CORS policy, CSP, rate limiting, or auth middleware scope
3. **Assess each ANALYSIS Risk Flag** — Evaluate each RF item from ANALYSIS-NNNN. State whether the DESIGN mitigates it, accepts it (with rationale), or leaves it open.
4. **Identify additional findings** — Apply the Security role quality bar to surfaces the ANALYSIS did not flag: timing attacks, log exposure, header security, middleware scope.
5. **Rate each finding** — Critical (exploitable unauthenticated), High (authenticated but high impact), Medium (specific conditions required), Low (defense-in-depth gap), Informational.
6. **Draft mitigations** — For each open finding, provide a concrete recommended action specific to the project's tech stack (from ARCH.md) and codebase: specific function, specific file, specific code pattern. Not "consider using X" — name the exact mechanism and where to apply it.
7. **Draft accepted-risk findings** — For each finding that cannot be mitigated within this slice's scope, document: the risk, why it is accepted, who approves it, and the specific condition under which it must be revisited.

## Output Contract

SEC-NNNN must contain all Required Sections per the SECURITY-REVIEW template: Scope of Review, Findings (with severity, description, attack scenario, mitigation per finding), Residual Risk Assessment (accepted-risk findings with approver and revisit condition), Compliance Considerations.

Frontmatter must include: `findings_status` (map of finding ID to `open` | `mitigated` | `accepted_risk`); `trigger_source` (ANALYSIS-NNNN ID); `target_design_version`; `generation` block (`command: /security-review`, `model` [self-identify from system context], `generated_at: [today's date]`).

## Lead Developer Review

After SEC-NNNN is drafted, confirm with the Lead Developer that each open finding's mitigation task is feasible to implement within the current plan scope. A finding that requires a design change (not just an implementation task) must be escalated to the Architect before the PLAN is authored.

## Quality Gate

Before SEC-NNNN is marked `completed`:
- [ ] Every ANALYSIS Risk Flag is addressed (mitigated, accepted, or escalated)
- [ ] Every Critical/High finding either has a concrete mitigation task or an explicitly accepted-risk entry with a named approver
- [ ] No finding mitigation says "consider" — all are specific and actionable
- [ ] Accepted-risk entries have a revisit condition, not just "v2 will fix this"
- [ ] `findings_status` frontmatter accurately reflects the current status of each finding
- [ ] `generation` block present: command, model self-identified, generated_at set

## Handoff

**Before committing — pause for human review:**
> Show the user the SEC file and ask: "Please review the security findings before I commit. Confirm: (1) proceed with the commit? (2) solo project (no push) or team project (push too)?"

**Commit (after user confirms):**
```
git add sdlc/security/SEC-NNNN-slug.md sdlc/registry/index.md
git commit -m "SEC-NNNN: Security review complete — [N Critical/High open, N accepted risk]"
```

**Push (only if user confirms team context):**
```
git push origin thread/NNNN-slug
```

Open Critical/High findings that are not accepted-risk block PLAN-NNNN authorship.
