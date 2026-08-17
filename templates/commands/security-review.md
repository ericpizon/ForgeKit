---
id: security-review
type: command
name: /security-review
description: Security role command to produce SEC-NNNN — a full threat model and findings report for a DESIGN; only invoked when ANALYSIS sets security_review_required yes
status: active
version: 3
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

1. **Scope definition** — Identify which surfaces are in scope: authentication mechanisms, authorization model, credential generation/storage/comparison, trust boundary placement, input validation on security-sensitive fields, any data retention concern, and — if `project.yml`'s `stack.llm_agent_untrusted_input` is `yes` — every prompt built from external or user-controlled text.
2. **Threat enumeration** — For each surface, identify plausible attacker capabilities (unauthenticated, authenticated-but-unauthorized, insider) and what they could achieve. Enumerate threats across all of the following categories — do not skip a category without a stated reason:
   - **Authentication bypass** — can an unauthenticated caller reach protected resources?
   - **Authorization bypass / privilege escalation** — can an authenticated user access or modify resources they don't own?
   - **Injection** — SQL, command, path traversal, template injection at any input boundary
   - **Prompt injection** *(if `stack.llm_agent_untrusted_input` is `yes`)* — for every prompt built from external or user-controlled text, can that text redirect the model's behavior (e.g. override instructions, influence a value the model shouldn't author, exfiltrate other prompt content)? Assess against DESIGN-NNNN's stated mitigation (explicit data-vs-instruction framing, or structural containment per its data-authorship boundary trace) — a DESIGN with no stated mitigation for a known untrusted-input prompt is itself a blocking finding, not just an unmitigated one
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
8. **Compliance Considerations** *(only if `project.yml`'s `compliance.frameworks` is not `[none]`)* — Read ARCH.md's "Compliance Obligations" section for the frameworks in scope and their stated engineering implications. For each framework this DESIGN's feature actually touches: name the specific control(s) implicated (cite the control ID where the framework has one — e.g. PCI-DSS Req 3.4, HIPAA §164.312(a)(1) — a framework without numbered controls like SOC 2's Trust Services Criteria or GDPR gets the specific principle instead), state whether the DESIGN satisfies it, and treat any gap as a regular finding rated by the same Critical/High/Medium/Low/Informational scale in Step 5 — a compliance gap is not a separate severity track. For PCI-DSS specifically: check `compliance.pci_scope` — if `delegated`, confirm the DESIGN actually keeps card data out of application code/logs/DB (a leak into this project's surface silently changes the project's real PCI scope, regardless of what was declared at `forge-init`). **This step identifies control gaps for a human compliance owner to act on. It does not constitute a compliance attestation, and SEC-NNNN must not claim the project "is compliant" with any framework** — say what was checked and what remains open, not a certification verdict.

## Output Contract

SEC-NNNN must contain all Required Sections per the SECURITY-REVIEW template: Scope of Review, Findings (with severity, description, attack scenario, mitigation per finding), Residual Risk Assessment (accepted-risk findings with approver and revisit condition), Compliance Considerations (per-framework control coverage if `compliance.frameworks` is not `[none]`; otherwise state explicitly "no compliance framework applies per project.yml" rather than leaving the section empty).

Frontmatter must include: `findings_status` (map of finding ID to `open` | `mitigated` | `accepted_risk`); `trigger_source` (ANALYSIS-NNNN ID); `target_design_version`; `schema_version: 1`; `generation` block (`command: /security-review`, `command_version: [use version from this command's frontmatter]`, `model` [self-identify from system context], `generated_at: [today's date]`, `session_id: null`).

## Lead Developer Review

After SEC-NNNN is drafted, confirm with the Lead Developer that each open finding's mitigation task is feasible to implement within the current plan scope. A finding that requires a design change (not just an implementation task) must be escalated to the Architect before the PLAN is authored.

## Quality Gate

Before SEC-NNNN is marked `completed`:
- [ ] Every ANALYSIS Risk Flag is addressed (mitigated, accepted, or escalated)
- [ ] Every Critical/High finding either has a concrete mitigation task or an explicitly accepted-risk entry with a named approver
- [ ] No finding mitigation says "consider" — all are specific and actionable
- [ ] Accepted-risk entries have a revisit condition, not just "v2 will fix this"
- [ ] If `compliance.frameworks` is not `[none]`, every applicable framework has named control(s) checked, with citations where the framework has numbered controls; if `[none]`, the section states that explicitly rather than being empty
- [ ] Compliance Considerations makes no attestation/certification claim — findings and open items only
- [ ] `findings_status` frontmatter accurately reflects the current status of each finding
- [ ] `schema_version: 1` present in SEC frontmatter
- [ ] `generation` block present: command, command_version, model self-identified, generated_at set, session_id: null

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
