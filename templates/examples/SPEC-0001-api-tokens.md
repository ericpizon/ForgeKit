---
id: SPEC-0001
type: spec
schema_version: 1
title: Personal API Token Management
status: approved
approval:
  mode: repo_only
  approved_by: Jordan Kim
  approved_role: tpo
  approved_at: 2026-05-06
  decision_summary: SPEC-0001 approved for design phase after two review rounds
  external_event_id: null
  external_event_uri: null
policy_evaluation:
  evaluated: null
  policy_pack: null
  result: null
created: 2026-05-01
updated: 2026-05-06
version: 2
owner: Jordan Kim
priority: P1
security_review_required: true
product_validated: true
validated_by: Jordan Kim
validated_date: 2026-05-05
stakeholders:
  - Jordan Kim (TPO)
  - Raj Mehta (Lead Developer)
  - Platform team
links: []
generation:
  command: /create-spec
  command_version: 1
  model: claude-sonnet-4-6
  generated_at: 2026-05-01
  session_id: null
---

> **Example artifact.** This file shows the expected output quality for a SPEC in the ForgeKit pipeline. The feature and project (Fieldstone) are fictional.

# SPEC-0001 — Personal API Token Management

## Problem Statement

Developers building integrations against the Fieldstone API have no way to authenticate programmatically without embedding their account password. The current auth model requires a browser-based session, which cannot be used in CI pipelines, automated scripts, or third-party integration tools. When a script's credentials are compromised, the only remediation is changing the account password — which simultaneously invalidates every other active session and integration. There is no surgical revocation path.

## Goals

1. Developers can generate a named API token from their account settings that authenticates API requests without requiring a session.
2. Developers can list their active tokens to know which integrations hold standing access.
3. Developers can revoke any of their tokens immediately, with effect taking place on the next request that presents the revoked token.

## Non-Goals

- **Token scopes or permissions** — tokens grant the same access level as the creating user's account. Fine-grained permission controls are a separate spec. This spec must not imply or assume a permission model that downstream work will need to unwind.
- **Team or service account tokens** — tokens are personal and owned by the creating user. A token created by User A cannot be used to authenticate as a team or org entity. Service accounts are a separate capability.
- **Token expiry or automatic rotation** — tokens are permanent until manually revoked. If an auditor asks "do tokens expire?", the answer for this token type is no. This is a known trade-off: accepted in exchange for implementation simplicity; the security review must assess the residual risk.
- **IP allowlisting per token** — not in this slice. A developer cannot restrict a token to specific source IPs.
- **Usage analytics beyond last-used date** — the UI shows the last-used timestamp per token only. Per-request history, endpoint breakdown, and usage volume are not in scope.
- **Maximum token count per user** — no limit is enforced in this slice. This is a known gap; if volume becomes a concern before a follow-on spec ships, a server-side limit can be added as a constraint without requiring a spec revision.

## User Stories

**US1 — Generate a token**
As a developer, I want to generate a named API token from my account settings so that I can authenticate automated scripts and CI pipelines without embedding my account password.

**US2 — Review standing access**
As a developer, I want to see a list of my active tokens with their labels, creation dates, and last-used dates so that I can identify integrations I no longer use and have forgotten to revoke.

**US3 — Revoke a token**
As a developer, I want to revoke any of my tokens immediately so that I can respond to a credential exposure without resetting my account password or disrupting other integrations.

## Acceptance Criteria

**AC1** — An authenticated user can generate a new API token at `/account/tokens`. The full token value is displayed exactly once in the UI immediately after creation, inside a copy-to-clipboard field.

**AC2** — After navigating away from the creation confirmation view, the full token value is not shown anywhere in the application. The token list displays only: label, creation date, last-used date (or "Never"), and the last 4 characters of the token.

**AC3** — The token generation form requires a non-empty label of at most 64 characters. Submitting without a label, with an empty label, or with a label exceeding 64 characters returns a field-level validation error and does not create a token.

**AC4** — An authenticated user can revoke any of their own tokens from `/account/tokens`. Revocation requires a confirmation step: click Revoke → confirm in a modal → token is deleted. Cancelling the modal makes no change.

**AC5** — An API call presenting a revoked token in `Authorization: Bearer <token>` returns HTTP 401 with body `{"error": "invalid_token"}`. This response is indistinguishable from the response for a token that never existed — no enumeration signal.

**AC6** — API endpoints under `/api/v1/` that require authentication accept a bearer token in the `Authorization` header. A valid bearer token from the authenticated user grants the same response the user would receive via session auth.

**AC7** — Each token creation and revocation event is appended to the audit log with: `user_id`, `event_type` (`token_created` or `token_revoked`), `token_label`, `token_suffix` (last 4 characters), and `occurred_at`. The raw token value never appears in any audit record.

## Non-Functional Requirements

**NFR1 — Token validation latency:** Bearer token validation must not add more than 5ms P99 latency measured at the middleware layer. Instrumentation strategy is defined in OBS-0001. If the initial implementation exceeds this threshold, the performance investigation is a launch blocker — not a post-launch action item.

**NFR2 — Token entropy:** Generated tokens must have at least 128 bits of entropy. `crypto.randomBytes(32)` (or equivalent) satisfies this. Any lower-entropy generation approach is a security finding, not a code style note.

**NFR3 — No plaintext token persistence:** The raw token is generated server-side, transmitted to the client exactly once over TLS, and never written to the database, logs, error responses, or any other storage in plaintext form. The only persisted form is the HMAC-SHA256 hash. This constraint is absolute — not a "should."

**NFR4 — Hash key management:** The HMAC secret is read from an environment variable at startup. It is not hardcoded, not committed to source control, and not logged. The key rotation procedure must be documented in the security review before launch.

## Constraints & Assumptions

- **Stack:** TypeScript / Next.js App Router, PostgreSQL via Drizzle ORM, Auth.js database sessions — per ARCH.md. No new infrastructure dependencies are introduced by this spec.
- **Scope of bearer auth:** bearer auth is applied only to endpoints under `/api/v1/`. Existing session-protected routes under `/app` and `/account` are unaffected.
- **Existing data:** no existing API token infrastructure exists. The `api_tokens` table is entirely new. There is no migration risk to existing records.
- **User trust level:** authenticated users only. The `/account/tokens` page and `POST /api/v1/tokens` endpoint require an active session. Unauthenticated requests redirect to sign-in.
- **Single environment:** no staging-specific token namespacing required. Tokens generated in production cannot be used in development (different HMAC secret per environment).

## Security Considerations

`security_review_required: true`

**Sensitive surfaces:**

- **Token generation response:** the raw token value is in-memory server-side and present in the HTTP response body exactly once. Any logging middleware that captures response bodies is a high-risk path.
- **Token storage:** only the HMAC-SHA256 hash is persisted. The server-side secret used for hashing must not be logged, must not be rotatable without a migration plan (changing the secret invalidates all tokens), and must be documented in the security review.
- **Bearer auth middleware:** every `/api/v1/` call performs a token lookup and hash comparison. Hash comparison must be constant-time to prevent timing attacks. A naive `===` comparison leaks token validity via timing.
- **Revocation authorization:** the revocation endpoint must verify the authenticated session user owns the token being revoked. A missing ownership check enables horizontal privilege escalation (User A revokes User B's token).
- **Enumeration resistance:** HTTP 401 responses for an invalid token, a revoked token, and a never-issued token must be identical in body, headers, and timing.

## Observability Considerations

**Metrics:**
- `api_tokens.created` — counter, label: `user_id` — one increment per successful token creation
- `api_tokens.revoked` — counter, label: `user_id` — one increment per successful revocation
- `api_tokens.auth_failure` — counter, label: `reason` (`invalid_hash` | `token_revoked` | `malformed_header`) — one increment per failed bearer auth attempt
- `api_tokens.validate_duration_ms` — histogram — bearer token validation duration from middleware entry to auth decision

**Traces:** Span named `api_token.validate` on the bearer auth middleware, wrapping token lookup and hash comparison. Attribute: `auth_result` (`success` | `failure`).

**Logs:** Structured event per creation and revocation:
```json
{"event": "token_created", "user_id": "...", "token_label": "...", "token_suffix": "....", "occurred_at": "..."}
{"event": "token_revoked", "user_id": "...", "token_label": "...", "token_suffix": "....", "occurred_at": "..."}
```
**Must never log:** full token value, HMAC hash, HMAC secret, session tokens, passwords, or any value that could be used to impersonate a user.

## Open Questions

All open items were resolved during review. None remain.

---

## Product Validation

*Recorded by `/validate-spec SPEC-0001` on 2026-05-05 (claude-sonnet-4-6). Sign-off: Jordan Kim (TPO).*

**AC translations (plain language, one sentence each):**

| AC | Translation |
|---|---|
| AC1 | Logged-in user goes to account settings, creates an API key with a name, and sees the full key once to copy. |
| AC2 | After leaving that screen, only the name, dates, and last 4 digits of the key are visible — never the full key again. |
| AC3 | The form won't create a key if the name is missing or over 64 characters; it shows an error on the field instead. |
| AC4 | User clicks Revoke next to a key, confirms in a popup, and the key is immediately deleted. Cancelling the popup does nothing. |
| AC5 | Any API call using a revoked key gets a "no" that looks exactly the same as if the key never existed. |
| AC6 | Scripts can authenticate API calls using `Authorization: Bearer <token>` and get back the same data the user would see when logged in. |
| AC7 | Every time a key is created or revoked, a record is saved — who did it, when, which key (by name and last 4 digits only, never the full value). |

**Tier 1 — Structural sign-off:**

*Problem in one sentence without implementation terms:* "Developers have no safe way to authenticate automated scripts against the Fieldstone API without exposing their account password."

*Edge case not covered by any AC:* What happens when a user tries to generate a token after hitting a large number of active tokens? Resolved: added to Non-Goals ("maximum token count per user is not enforced in this slice") with a note that a server-side limit can be added without a spec revision. AC7 is unchanged.

*Non-Goals specific enough as decision rules:* Yes — reviewed each. "No permission scoping," "no team tokens," "no automatic expiry," and "no IP allowlisting" are all specific enough that a developer encountering a scope extension request during implementation can apply them without asking.

**Tier 2 — Per-AC sign-off:**

- *AC5:* Confirmed: a revoked token and a never-issued token return identical status, body, and timing. No oracle.
- *AC7:* Confirmed: "audit log" means the application's own `audit_events` table, not an external service. No new infrastructure dependency.
