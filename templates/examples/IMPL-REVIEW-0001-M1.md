---
id: IMPL-REVIEW-0001-M1
type: impl-review
schema_version: 1
plan: PLAN-0001
milestone: M1
current_review_round: 2
approval:
  mode: repo_only
  approved_by: Raj Mehta
  approved_role: lead-developer
  approved_at: 2026-05-16
  decision_summary: IMPL-REVIEW-0001-M1 Round 2 — zero blocking findings, all Round 1 findings resolved; M1 approved for merge
  external_event_id: null
  external_event_uri: null
policy_evaluation:
  evaluated: null
  policy_pack: null
  result: null
generation:
  command: /review-implementation
  command_version: 1
  model: claude-sonnet-4-6
  generated_at: 2026-05-14
  session_id: null
---

> **Example artifact.** This file shows the expected output quality for an IMPL-REVIEW in the ForgeKit pipeline. It covers two rounds: Round 1 with blocking and non-blocking findings, an address summary, and Round 2 clearing all findings. The feature and project (Fieldstone) are fictional.

# Implementation Review — PLAN-0001 M1

## Task Reference — M1

*Derived from PLAN-0001 at review creation time.*

| Task | File(s) | Description |
|---|---|---|
| T1 | `drizzle/migrations/0012_add_api_tokens.sql` | Create `api_tokens` table migration |
| T2 | `src/db/schema/api-tokens.ts` | Drizzle schema for `api_tokens`; export `ApiToken` and `NewApiToken` types |
| T3 | `src/lib/tokens/generate.ts` | `generateToken()` — crypto.randomBytes(32) + HMAC-SHA256; pure function |
| T4 | `src/db/tokens.ts` | DAL: `createToken`, `revokeToken`, `listTokens`, `findTokenByHash` |
| T5 | `src/middleware/token-auth.ts` | Bearer auth middleware `withBearerAuth(handler)` |
| T6 | `src/middleware/token-auth.ts` | OBS-0001 IT-1 — observability instrumentation |
| T7 | `src/middleware/token-auth.ts` | SEC-0001 F-1 — constant-time hash comparison mitigation |

---

## Round 1 — 2026-05-14

### Lead Developer Findings

**[LD-1] blocking — `revokeToken` ownership check is in the handler, not the query**

`src/db/tokens.ts:47` — `revokeToken` fetches the token by ID first, then checks `token.userId === userId` in application code before calling `delete`. This is correct in sequence, but a TOCTOU race exists between the fetch and the delete: two concurrent revocations could both pass the ownership check on the same token.

More importantly, PLAN-0001 T4 specifies that the ownership check must be in the query (`WHERE id = $tokenId AND user_id = $userId`). This is both safer and simpler. The current implementation deviates from the design without surfacing a blocker.

Resolution: change `revokeToken` to a single `DELETE FROM api_tokens WHERE id = $tokenId AND user_id = $userId RETURNING id`. If no rows returned, throw `NotFoundError`. Remove the fetch-then-check pattern.

**[LD-2] non-blocking — `updateLastUsed` is not truly fire-and-forget**

`src/middleware/token-auth.ts:61` — `updateLastUsed(tokenId)` is called without `await` but without `void`. In the current Next.js version, unhandled Promise rejections in route handlers can surface as unhandled exceptions in the runtime. Pattern should be `void updateLastUsed(tokenId)` to explicitly signal this is intentional.

Resolution: add `void` keyword. Minor.

### Security Findings

**[SEC-1] blocking — hash comparison uses `===`, not constant-time**

`src/middleware/token-auth.ts:44` — `storedHash === computedHash` is a standard string equality comparison. String `===` in V8 short-circuits on the first differing character, which leaks timing information proportional to the length of the shared prefix. This directly violates SEC-0001 F-1 (T7) and enables a timing oracle against the token hash.

Resolution: replace with `crypto.timingSafeEqual(Buffer.from(storedHash, 'hex'), Buffer.from(computedHash, 'hex'))`. Wrap in a try/catch because `timingSafeEqual` throws if buffers differ in length; a malformed hash should return false, not throw.

This is a security requirement from SEC-0001, not an enhancement request. It was assigned as T7 and must be in M1.

**Clean perspectives:** SRE, QE

**Summary:** 2 blocking (LD-1, SEC-1), 1 non-blocking (LD-2).

**Next step:** `/address-implementation-review PLAN-0001 M1`

---

### Address Summary — Round 1

*Recorded by Raj Mehta (Lead Developer), 2026-05-15.*

**LD-1** — Fixed. `revokeToken` rewritten as a single parameterized DELETE with `WHERE id = $1 AND user_id = $2 RETURNING id`. Fetch-then-check pattern removed. `NotFoundError` thrown when `returning` is empty. `src/db/tokens.ts:44–52`.

**SEC-1** — Fixed. String comparison replaced with `crypto.timingSafeEqual`. Added length guard: if hashes differ in byte length (malformed input), returns false without calling `timingSafeEqual`. `src/middleware/token-auth.ts:41–50`.

**LD-2** — Fixed. Added `void` to the `updateLastUsed` call. `src/middleware/token-auth.ts:63`.

---

## Round 2 — 2026-05-16

*Scope: LD-1, SEC-1, LD-2 only. No full re-review of unchanged code.*

**LD-1 verification:** Read `src/db/tokens.ts:44–52`. Single `DELETE ... RETURNING id` with both `id` and `user_id` in the WHERE clause. `NotFoundError` thrown when returning array is empty. Matches PLAN-0001 T4 spec and resolves TOCTOU concern. ✅

**SEC-1 verification:** Read `src/middleware/token-auth.ts:41–50`. `timingSafeEqual` with `Buffer.from(..., 'hex')` on both operands. Length check before the call returns false on mismatch without throwing. Matches SEC-0001 F-1 mitigation spec exactly. ✅

**LD-2 verification:** `void updateLastUsed(tokenId)` at line 63. ✅

**Clean perspectives:** Lead Developer, Security, SRE, QE

**Summary:** 0 blocking, 0 non-blocking findings. All Round 1 findings resolved.

**Next step:** `/approve-implementation PLAN-0001 M1`

---

*Targeted confirmation questions (from `/approve-implementation` Step 7):*

- *Is every function with meaningful complexity covered by a test?* Yes — `generateToken` (pure function) has unit tests; `createToken`, `findTokenByHash`, `revokeToken` have integration tests against a real test database per ARCH.md rules.
- *Would you be comfortable being paged on this code at 3am?* Yes — the middleware is a single read path with a clear failure mode (return 401). The only non-obvious behavior is the async `updateLastUsed`, now correctly marked `void`.
- *Does the implementation match the DESIGN-0001 contract?* Yes — API shapes, DAL signatures, and middleware interface all match. No silent decisions during implementation.

**Approval decision:** APPROVED — Raj Mehta, 2026-05-16.
