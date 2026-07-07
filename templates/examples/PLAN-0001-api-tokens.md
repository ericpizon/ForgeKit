---
id: PLAN-0001
type: plan
title: Personal API Token Management
status: approved
created: 2026-05-10
updated: 2026-05-14
version: 1
owner: Raj Mehta
target_design_version: 1
milestone_count: 2
current_review_round: 1
includes_observability_tasks: true
includes_security_tasks: true
generation:
  command: /create-plan
  model: claude-sonnet-4-6
  generated_at: 2026-05-10
pipeline_status:
  spec: SPEC-0001 approved
  analysis: ANALYSIS-0001 completed
  design: DESIGN-0001 approved
  obs: OBS-0001 approved
  sec: SEC-0001 completed
  plan: approved
  test: TEST-0001 approved
---

> **Example artifact.** This file shows the expected output quality for a PLAN in the ForgeKit pipeline. The feature and project (Fieldstone) are fictional.

# PLAN-0001 — Personal API Token Management

## Sequencing & Dependencies

Tasks follow this phase order per ARCH.md:

1. **Schema** — migrations must exist before any code that reads or writes the table
2. **Core logic** — pure functions (hashing, token generation) before the DAL that calls them
3. **Data access layer** — DAL functions before the API handlers that call them
4. **Middleware** — bearer auth middleware before the endpoints it protects
5. **API layer** — endpoints after all dependencies are in place
6. **UI layer** — pages and components after API endpoints are stable
7. **Observability** — instrumentation goes in the milestone that introduces the code being instrumented
8. **Security mitigations** — go in the same milestone as the code they protect

All M1 tasks must reach `pr_merged` before M2 begins — M2's DAL and API tasks depend on the `api_tokens` table from M1.

## Task Breakdown

---

**T1** | functional | `drizzle/migrations/0012_add_api_tokens.sql`
Create the `api_tokens` table migration: `id uuid primary key`, `user_id uuid not null references users(id) on delete cascade`, `label varchar(64) not null`, `token_hash varchar(64) not null`, `token_suffix char(4) not null`, `last_used_at timestamptz`, `revoked_at timestamptz`, `created_at timestamptz not null default now()`. Index on `token_hash` (unique) and `user_id`.
**ACs:** AC1, AC2, AC6
**Milestone:** M1

---

**T2** | functional | `src/db/schema/api-tokens.ts`
Define the Drizzle schema for `api_tokens` matching the migration. Export `ApiToken` and `NewApiToken` TypeScript types.
**ACs:** AC1, AC2, AC6
**Milestone:** M1

---

**T3** | functional | `src/lib/tokens/generate.ts`
Implement `generateToken(): { raw: string; hash: string; suffix: string }`. Use `crypto.randomBytes(32)` for entropy (satisfies NFR2). Compute HMAC-SHA256 of raw bytes using `TOKEN_HMAC_SECRET` env var. Return raw (hex string, transmitted once), hash (stored), and suffix (last 4 chars of hex string). Pure function — no I/O, unit-testable.
**ACs:** AC1 (NFR2, NFR3)
**Milestone:** M1

---

**T4** | functional | `src/db/tokens.ts`
Implement DAL functions:
- `createToken(userId: string, label: string): Promise<{ token: ApiToken; raw: string }>` — calls `generateToken()`, inserts row, returns row + raw token (raw is passed through to the handler for one-time display, never persisted beyond this call).
- `revokeToken(tokenId: string, userId: string): Promise<void>` — sets `revoked_at = now()` where `id = tokenId AND user_id = userId` (ownership check in the query, not just in the handler). Throws `NotFoundError` if no matching row — handler converts this to 404.
- `listTokens(userId: string): Promise<ApiToken[]>` — returns all non-revoked tokens for user, ordered by `created_at desc`.
- `findTokenByHash(hash: string): Promise<ApiToken | null>` — used by bearer auth middleware; returns null for revoked tokens (`revoked_at IS NULL` in query).
**ACs:** AC1, AC2, AC4, AC5, AC6
**Milestone:** M1

---

**T5** | functional | `src/middleware/token-auth.ts`
Implement bearer auth middleware `withBearerAuth(handler)`. Extract `Authorization: Bearer <token>` header. Hash the presented token with `generateToken`'s hash function (same HMAC secret). Call `findTokenByHash(hash)`. If found: attach `{ userId, tokenId }` to request context and update `last_used_at` asynchronously (fire-and-forget — do not block the request). If not found or header absent: return `{ status: 401, body: { error: "invalid_token" } }`. No distinction between "never existed" and "revoked" in the response (AC5).
**ACs:** AC5, AC6
**Milestone:** M1

---

**T6** | observability | `src/middleware/token-auth.ts`
**Source:** OBS-0001 IT-1 — see OBS-0001 for full instrumentation spec.
**Milestone:** M1

---

**T7** | security | `src/middleware/token-auth.ts`
**Source:** SEC-0001 F-1 — see SEC-0001 for full mitigation spec (constant-time hash comparison replacing `===`).
**Milestone:** M1

---

**T8** | functional | `src/app/api/v1/tokens/route.ts`
Implement `POST /api/v1/tokens`. Requires session auth (existing `withAuth` middleware). Validate body: `label` required, max 64 chars (AC3). Call `createToken(userId, label)`. Return `{ token: { id, label, suffix, created_at }, raw }` — `raw` is present only in this response and never returned again. Status 201.
**ACs:** AC1, AC3
**Milestone:** M2

---

**T9** | functional | `src/app/api/v1/tokens/route.ts`
Implement `GET /api/v1/tokens`. Requires session auth. Call `listTokens(userId)`. Return array of `{ id, label, suffix, last_used_at, created_at }` — no `token_hash`, no raw token. Status 200.
**ACs:** AC2
**Milestone:** M2

---

**T10** | functional | `src/app/api/v1/tokens/[id]/route.ts`
Implement `DELETE /api/v1/tokens/[id]`. Requires session auth. Call `revokeToken(tokenId, userId)` — ownership check is in the DAL query (T4), not duplicated here. On `NotFoundError`: return 404. On success: append audit log event (`token_revoked`) and return 204.
**ACs:** AC4, AC5
**Milestone:** M2

---

**T11** | functional | `src/lib/audit.ts` (new function), `src/app/api/v1/tokens/route.ts`, `src/app/api/v1/tokens/[id]/route.ts`
Implement `writeAuditEvent(event: AuditEvent): Promise<void>`. Called in T8 (token_created) and T10 (token_revoked). AuditEvent shape: `{ user_id, event_type, token_label, token_suffix, occurred_at }`. Writes to `audit_events` table. Full token value is never passed to this function — callers pass only `token_suffix` from the DAL result.
**ACs:** AC7
**Milestone:** M2

---

**T12** | functional | `src/app/account/tokens/page.tsx`
Implement `/account/tokens` page. Fetch tokens via `GET /api/v1/tokens`. Render token list table: label, last-used date, last 4 chars, Revoke button. Include "Generate new token" button that opens the generation form (inline or modal — per DESIGN-0001). Requires session — Next.js middleware redirects unauthenticated users.
**ACs:** AC2, AC4
**Milestone:** M2

---

**T13** | functional | `src/app/account/tokens/page.tsx`
Implement the one-time token display state shown immediately after successful generation. Renders the raw token in a copy-to-clipboard field with a prominent warning: "Copy this token now. It will not be shown again." After the user dismisses or navigates, this state is cleared from memory. Subsequent renders show only the token list (no raw value).
**ACs:** AC1
**Milestone:** M2

---

**T14** | functional | `src/components/tokens/RevokeModal.tsx`
Implement revoke confirmation modal. Props: `{ tokenId, tokenLabel, onConfirm, onCancel }`. Renders token label in the confirmation message ("Revoke [label]? This cannot be undone."). Confirm button calls `DELETE /api/v1/tokens/[id]` then refreshes token list. Cancel dismisses without changes.
**ACs:** AC4
**Milestone:** M2

---

## Milestones

### M1 — Foundation: Schema, Token Logic & Auth Middleware

**Tasks:** T1, T2, T3, T4, T5, T6, T7
**Implementation status:** pr_merged
**AC Coverage:** AC1 (partial — token is created and hashed; UI display is M2), AC2 (partial — suffix stored; list UI is M2), AC5, AC6
**Depends on:** —
**Rollback:** Safe — no UI surface; migration is additive (new table only, no existing table modifications). Rolling back means dropping the new table.

### M2 — API Endpoints, UI & Audit

**Tasks:** T8, T9, T10, T11, T12, T13, T14
**Implementation status:** approved
**AC Coverage:** AC1 (complete), AC2 (complete), AC3 (complete), AC4 (complete), AC5 (complete — end-to-end revocation), AC7 (complete)
**Depends on:** M1 (`pr_merged`)
**Rollback:** Revert the three API route files and the three UI files. The `api_tokens` table from M1 remains; no data migration needed on rollback.

---

## Happy Path Trace

### US1 — Generate a token

| Flow step | Task(s) |
|---|---|
| User navigates to `/account/tokens` | T12 — page route + token list fetch via T9 |
| User clicks "Generate new token" | T12 — opens generation form |
| User enters label and submits | T8 — POST /api/v1/tokens → T4 (createToken) → T3 (generateToken) |
| Raw token displayed in copy field | T13 — one-time display component |
| User copies token and dismisses | T13 — clears raw value from state |
| User navigates back; sees token in list | T12 + T9 — list shows label, suffix, dates; no raw value |
| Audit log written | T11 — `token_created` event |

Gap check: all steps covered.

### US3 — Revoke a token

| Flow step | Task(s) |
|---|---|
| User clicks Revoke next to a token | T12 — Revoke button visible per row |
| Confirmation modal appears | T14 — modal with token label |
| User confirms | T14 → T10 — DELETE /api/v1/tokens/[id] → T4 (revokeToken, ownership checked in query) |
| Audit log written | T11 — `token_revoked` event |
| Token disappears from list | T12 — page refreshes after delete |
| Subsequent API call with revoked token → 401 | T5 — findTokenByHash returns null (revoked_at IS NOT NULL excluded in T4) |

Gap check: all steps covered.

---

## Out of Scope for This Plan

| Item | Justification |
|---|---|
| Token scopes / permissions | Non-Goal in SPEC-0001 |
| Team or service account tokens | Non-Goal in SPEC-0001 |
| Token expiry / automatic rotation | Non-Goal in SPEC-0001 |
| IP allowlisting per token | Non-Goal in SPEC-0001 |
| Maximum token count enforcement | Non-Goal in SPEC-0001 (known gap, deferred) |
| HMAC secret rotation procedure | Documented in SEC-0001 — operational, not a code task |

---

## Definition of Done

| AC | Description (verbatim from SPEC-0001) | Covered by | Status |
|---|---|---|---|
| AC1 | An authenticated user can generate a new API token at `/account/tokens`. The full token value is displayed exactly once in the UI immediately after creation, inside a copy-to-clipboard field. | T3, T4, T8, T13 | ✅ M1 (generation) + M2 (display) |
| AC2 | After navigating away from the creation confirmation view, the full token value is not shown anywhere in the application. The token list displays only: label, creation date, last-used date (or "Never"), and the last 4 characters of the token. | T4 (suffix stored), T9, T12 | ✅ M2 |
| AC3 | The token generation form requires a non-empty label of at most 64 characters. Submitting without a label, with an empty label, or with a label exceeding 64 characters returns a field-level validation error and does not create a token. | T8 (server validation), T12 (form UI) | ✅ M2 |
| AC4 | An authenticated user can revoke any of their own tokens from `/account/tokens`. Revocation requires a confirmation step: click Revoke → confirm in a modal → token is deleted. Cancelling the modal makes no change. | T4 (revokeToken), T10, T12, T14 | ✅ M2 |
| AC5 | An API call presenting a revoked token returns HTTP 401 with `{"error": "invalid_token"}`. This response is indistinguishable from the response for a token that never existed. | T4 (findTokenByHash excludes revoked), T5, T7 (constant-time) | ✅ M1 (middleware) + M2 (revocation flow) |
| AC6 | API endpoints under `/api/v1/` that require authentication accept a bearer token. A valid token grants the same response the user would receive via session auth. | T4 (findTokenByHash), T5, T6 (OBS) | ✅ M1 |
| AC7 | Each token creation and revocation event is appended to the audit log with: `user_id`, `event_type`, `token_label`, `token_suffix`, and `occurred_at`. The raw token value never appears in any audit record. | T11 | ✅ M2 |
