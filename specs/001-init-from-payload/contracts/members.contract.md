# contracts/members.contract.md — Member contract & JIT provisioning

## Purpose

Define the contract between undrepost and the authoritative IdP (undrlla) for JIT-provisioning Members and the shape of Member records used by collections.

## Token Validation (Mandatory)

Before ANY claim is trusted for JIT-provisioning or authorization, the incoming JWT MUST be fully validated:

1. **Signature**: verified against undrlla JWKS (`IDP_JWKS_URL`). Unsigned or wrong-key tokens are rejected with 401.
2. **Issuer** (`iss`): MUST match `IDP_ISSUER` config exactly.
3. **Audience** (`aud`): MUST include `undrepost` (token minted for undrepost, NOT for the marketplace). A marketplace-audience token is rejected.
4. **Expiry** (`exp`): MUST not be expired. Clock-skew tolerance: ±60s.
5. **Subject** (`sub`): MUST be present and non-empty.

Only after all 5 pass is the `sub` trusted for identity mapping.

## JIT Provisioning API (behavioral contract)

- **Trigger**: Successful OIDC authentication + full token validation (above).
- **Action**: Upsert a `Members` record where `sub == id_token.sub`.
- **Concurrency**: DB-level `UNIQUE(sub)` constraint + `INSERT ... ON CONFLICT (sub) DO UPDATE` (or equivalent). Handles concurrent provisioning from multi-tab / replay without duplicates.
- **Re-sync**: On every successful login (not just first), refresh `email`, `displayName`, `profile`, `roles` from token claims. Update `syncedAt` timestamp. This prevents mirror staleness.
- **Validation**: Verify `tenant_id` claim matches instance config; reject with 403 + audit event otherwise.

Example endpoint (internal hook) — semantics only

- `POST /api/internal/jit-provision`
  - Payload: `{ "sub": "<sub>", "email": "...", "name": "...", "roles": ["member"] }`
  - Response: `200 { "memberId": "<uuid>", "created": true|false }`

## Member JSON Schema (informal)

- id: string (uuid)
- sub: string (undrlla subject, unique, DB-enforced)
- email: string
- displayName: string
- roles: ["admin"|"member"]
- profile: object
- syncedAt: string (datetime) — last re-sync
- createdAt: string (datetime)

## Contract Rules

- `sub` is authoritative for identity mapping. Do not derive identity from email alone.
- **Admin bootstrap**: matching by `sub` ONLY. Email-based allowlist is a known risk (user changes email → gains admin) and is **not permitted**. The initial bootstrap must use `sub` values from the IdP.
- Upsert must be idempotent and log the source claims for audit.
- If `tenant_id` claim fails validation, return 403 and write an audit event.
- Profile fields (`email`, `displayName`, `profile`) are re-synced on every login — the mirror is eventually consistent with a per-session freshness guarantee.
- If a Member is deleted upstream, the local mirror record is retained as a tombstone (`displayName` → "[deleted]", `profile` cleared) to preserve FK integrity for authored content. No cascade delete of posts/comments.
