---
id: SEC-0001
type: security-review
schema_version: 1
findings_status:
  SEC-1: open
  SEC-2: open
  SEC-3: open
trigger_source: ANALYSIS-0001
target_design_version: 3
generation:
  command: /security-review
  command_version: "5"
  model: "claude-sonnet-5"
  generated_at: "2026-10-05"
  session_id: null
---

# SEC-0001 — External Work Intake and Durable Coordination Context — Security Review

## Scope of Review

In scope: G1's intake and write-back paths (trust boundary against externally-sourced ADO/Confluence content, input handling of that content, `.source.md` capture and retention); `coordination_repo`/`known_participants`/`environments` values (trust boundary against the local operator's own input); `/promote-to-thread`'s state transitions (business-logic-abuse angle). Out of scope: authentication/authorization (ForgeKit implements none anywhere — every command here requires either local filesystem/git write access, the same trust level as running any ForgeKit command at all, or an already-configured external connector a human set up themselves; not a gap introduced by this DESIGN). `stack.llm_agent_untrusted_input` cannot be checked formally — ForgeKit's own repo has no `project.yml` — but the substance of that check (text from outside the local trust boundary feeding an AI-driven drafting process) applies directly to G1 and is assessed on its merits below.

## Threat Enumeration

- **Authentication bypass:** N/A — no auth model exists to bypass.
- **Authorization bypass / privilege escalation:** N/A — no per-command authorization layer exists anywhere in ForgeKit; this DESIGN doesn't change that baseline.
- **Injection:** Assessed in depth — see **SEC-2** (slug derivation from externally-sourced titles) and **SEC-3** (unvalidated repo/environment-name values).
- **Prompt injection:** DESIGN-0001's D12 states an explicit data-vs-instruction framing mitigation (fetched/pasted content is always quoted reference material to the TPO role, never executed instructions) — this is the correct level of mitigation for a DESIGN document, matching the bar `cross-repo-resolution.md` already sets for other cross-repo inputs. **Confirmed mitigated**, not an open finding.
- **Credential and sensitive data exposure:** Two angles. (1) Failure diagnostics from a failed fetch/write (AC1/AC4) — D11 requires sanitizing to a failure category before display or commit. **Confirmed mitigated.** (2) The `.source.md` capture itself (AC2) — D6's confirmation gate is keyed on `compliance.frameworks`, which catches *regulated* data but not an accidentally-pasted *credential* in a project with no compliance obligations at all. **Open — see SEC-1.**
- **Enumeration:** N/A — sequential IDs (`THREAD-NNNN`, `SPEC-NNNN`, `INV-NNNN`) are existing ForgeKit convention, not new to this DESIGN, and none are exposed via any query interface — there's nothing to probe.
- **Timing attacks:** N/A — no secret comparison anywhere in this DESIGN.
- **Business logic abuse:** `/promote-to-thread`'s steps being invoked out of order, concurrently, or repeated — already addressed by ADR-0001's idempotent-by-construction design and the Round 2 design-review confirmation that ordinary git concurrent-commit handling covers the remaining race. **Confirmed mitigated**, not an open finding.
- **Security header and middleware gaps:** N/A — no web server or middleware anywhere in this DESIGN.

## Findings

### SEC-1 — Medium — Credential-pattern exposure in `.source.md` capture is not checked independently of compliance-framework status

**Description:** AC2/D6's confirmation gate before writing a `.source.md` companion file is keyed entirely on `compliance.frameworks` being non-`[none]`. A project with no compliance obligations gets the file auto-captured with no gate at all — but a credential accidentally included in pasted PRD text (a stray API key visible in a copied screenshot, a code snippet with a hardcoded secret) is a real exposure regardless of whether the project has any regulatory obligations. This is the same irreversibility problem SPEC-0001's own original SEC-1 identified for PII (once committed, effectively impossible to reliably un-commit), but triggered by a condition D6's gate was never designed to catch.

**Attack scenario:** Not an active external attacker — an accidental self-inflicted exposure. A developer pastes a Confluence PRD into `/create-spec` without `compliance.frameworks` set; the PRD happens to include a visible API key from a screenshot or an embedded code sample. The companion file is captured automatically, committed, and the credential is now in git history.

**Mitigation:** Before auto-capturing (i.e., whenever D6's own compliance-based gate would *not* fire), run a lightweight pattern check against common credential shapes (`sk-[A-Za-z0-9]{20,}`, `ghp_[A-Za-z0-9]{36}`, `AKIA[0-9A-Z]{16}`, `-----BEGIN PRIVATE KEY-----` blocks, and similar) in `/create-spec`'s capture step. On a match, apply the same confirmation-gate behavior D6 already defined for the compliance case — warn the user, require explicit confirmation before capture — even when `compliance.frameworks` is `[none]`. This decouples the credential safety net from the regulatory-compliance gate, since they fire on different conditions.

**Recommended disposition:** Open — concrete, implementable mitigation task, no design change required.

### SEC-2 — Medium — Slug derivation from an externally-sourced title has no stated sanitization

**Description:** `/create-spec`'s existing slug-derivation instruction ("lowercase, hyphens, drop filler words") has never been a problem before, because the source title was always authored fresh by the TPO role from a human-stated problem. G1 is the first time a slug's source title can come from genuinely external, lower-trust content — an ADO ticket's own title, fetched or pasted. The instruction is natural-language guidance for an AI to follow, not an explicit, stated sanitization control; a title containing path-traversal-shaped text (`../../`) or other filesystem-unsafe characters has no stated guarantee of being stripped before the resulting slug is used in a file path or branch name.

**Attack scenario:** A malicious or compromised ADO ticket's title is crafted to contain path-traversal sequences; if slug derivation is ever implemented more mechanically (not by an AI naturally avoiding unsafe output, but by a literal lowercase-and-hyphenate transform with no explicit character allow-list), the resulting file write could land outside the intended `sdlc/specs/` (or `docs/design/`) directory.

**Mitigation:** `/create-spec`'s slug-derivation step (and the equivalent in the new `/create-investigation` and `/create-adr`) must state an explicit sanitization rule: strip or reject every character outside `[a-z0-9-]` before using the result in any file path or git branch name — regardless of whether the source title was human-authored or externally fetched. This closes the gap uniformly, not just for G1's new path.

**Recommended disposition:** Open — concrete, implementable mitigation task, no design change required.

### SEC-3 — Low — `coordination_repo`/`known_participants[].repo`/`environments[].name` values are not validated against their expected shape

**Description:** These values are set by a trusted local operator through `/forge-init`'s own interactive Q&A (AC7) or `/promote-to-thread`'s reactive equivalent (AC8) — not an external attacker surface. But `cross-repo-resolution.md`'s own resolution algorithm interpolates a `repo` value into a `gh api repos/{repo}/contents/{path}` URL; an unvalidated, malformed value (accidental typo, copy-paste error) could produce unexpected API resolution behavior.

**Attack scenario:** Low-likelihood, not attacker-driven — a careless or malformed input from the operator setting up `coordination_repo` produces a value that doesn't cleanly resolve to `org/repo`, with an unclear failure mode at resolution time.

**Mitigation:** `/forge-init`'s coordination-repo question (AC7) and `/promote-to-thread`'s reactive equivalent (AC8) validate the supplied value against a safe `org/repo`-shaped pattern (e.g. `^[\w.-]+/[\w.-]+$`) before writing it to `project.yml`, re-prompting on a non-match. Same validation applies to `environments[].name`.

**Recommended disposition:** Open — concrete, implementable mitigation task, no design change required; defense-in-depth, not a live exploit path.

## Assessment of ANALYSIS Risk Flags

| Risk Flag | Assessment |
|---|---|
| The live write-back path (AC4), including its PII/credential-irreversibility angle (ANALYSIS-0001's own Lead Developer Perspective, carried from SPEC-0001's original SEC-1) | Mitigated for the *regulated-data* case via D6's confirmation gate, and for *failure-diagnostic* leakage via D11. **Not fully mitigated for the credential-in-pasted-content case** — see SEC-1 above, which sharpens this risk flag into a specific, still-open finding rather than treating D6 as a complete closure. |

## Residual Risk Assessment

No findings are accepted as risk in this review — all three (SEC-1, SEC-2, SEC-3) have concrete, implementable mitigation tasks with no design change required, so none need Architect escalation or a named risk-acceptance approver. Each is tracked as `open` in frontmatter, to be resolved during `/create-plan`/implementation, not deferred.

## Compliance Considerations

N/A — ForgeKit's own repo has no `project.yml`/`compliance.frameworks` field, consistent with this whole SPEC/DESIGN's structural deviation (never forge-init'd on itself). For any adopting team using this feature, Compliance Considerations must be assessed against *their own* `project.yml`'s `compliance.frameworks` value at their own adoption time — this review cannot and does not attest to any adopting team's own compliance posture. (SEC-1's mitigation, once implemented, is itself a relevant engineering control for a future team's own SOC 2/GDPR-adjacent obligations, but naming it here is not a compliance attestation.)
