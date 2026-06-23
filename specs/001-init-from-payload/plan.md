# Implementation Plan — undrepost: Initialize from Payload

Feature branch: `specs/001-init-from-payload`

## Summary

This plan implements the `undrepost` migration: fork `payload` (payload-monorepo v4.0.0-beta.0), re-platform legacy `underblog` features onto Payload collections, and add the social features and contracts described in the spec.

Primary goals

- Keep divergence small from upstream Payload (Sync fork strategy).
- Provide a local `Members` mirror (JIT-provisioned from undrlla IdP) as the canonical author/member reference for undrepost collections.
- Implement SSE-based social counts (reactions/polls) with heartbeat/replay and connection limits.
- Deliver a minimal MVP (posts, comments, reactions counts, admin UI, Member mirror).

## Technical Context

- Platform: PayloadCMS forked monorepo (app/collection layer customizations).
- Identity: undrlla IdP is authoritative; undrepost maintains a local `Members` mirror created on first login using IdP `sub` claim and caching role/profile. Re-synced on every login.
- Tenancy: Single-tenant per deploy (validate IdP claim `tenant_id` against instance config).
- Real-time: SSE endpoint for social counts. Backend holds counts and publishes deltas; clients subscribe to updates for optimistic UI.

## Key Design Decisions

- Members mirror (local collection) instead of remote lookups to simplify FK relations and queries.
- JIT provisioning endpoint: on successful OIDC login + full token validation (signature/aud/iss/exp), upsert mirrored Member record. ON CONFLICT semantics for concurrency safety.
- SSE design: authenticated endpoint, heartbeat pings, reconnection with replay token (per-feed last-seq), and connection limits per user + per IP.
- Audit & roles: Bootstrap admin by `sub` allowlist only (no email matching — security risk per review). Validate against IdP claim.

## Deliverables / Artifacts

- `plan.md` (this file)
- `tasks.md` (task breakdown & agent dispatch)
- `data-model.md` (collection schemas)
- `contracts/members.contract.md` (auth/Member contract & JIT API)
- `quickstart.md` (dev startup steps)

## Implementation Phases

Phase 0 — Research & scaffolding (2.5–3.5 days)

- Validate upstream Payload version and fork strategy.
- Pin beta build, document rebase/patch points, track stable tag (T0.3).
- Create repo worktree, branch `specs/001-init-from-payload`.
- Generate skeleton collections and minimal app/collection layer.

Phase 1 — Core collections & identity (3.5–5.5 days)

- Implement `Members` mirror collection (with UNIQUE(sub) + syncedAt) and `Posts` collection schema.
- Add JIT-provisioning flow: on IdP login, upsert Member (ON CONFLICT, re-sync claims).
- Add admin bootstrap by `sub` allowlist.

Phase 2 — Social features & SSE (5–7 days)

- Implement `Reactions` (emojiRef shortcode, UNIQUE triple), `Comments` (soft-delete), `Polls` + `PollVote` collections and relations.
- Implement comment gating policy (can-comment hook + rate-limit).
- Implement SSE feed service: authenticated endpoint, counts, playback/replay, heartbeat.
- Async materialized counters with periodic reconciliation job.
- Output sanitization (DOMPurify) for rendered content.

Commerce integration & release gate

- Per project decision, auctions are a MUST for this init (FR-030) and therefore the release is gated on undrlla readiness. Cross-repo integration tasks are added in `tasks.md` (T2.13..T2.16) to finalize the auction contract, implement the `AuctionRef` adapter, provision staging handshakes, and run contract e2e tests.
- The final GA release requires an undrlla-ready staging endpoint and passing contract integration tests; early content work may proceed, but the release pipeline will verify the undrlla signal before merging release artifacts.

SSE reliability defaults (to be tuned in plan):

- Heartbeat interval: 15s
- Replay window: support `Last-Event-ID` replay for last 100 events per feed
- Replay miss behavior: if `Last-Event-ID` is older than the 100-event window, send a `snapshot` event (current state) followed by live deltas. Client catches up via snapshot, no silent gap.
- Connection caps: 5,000 concurrent connections per instance with per-user limit (5 sessions) and per-IP soft-limit at 50 connections and a backpressure policy.
- Authentication: SSE endpoint requires valid token (membership). Unauthenticated SSE is rejected at the transport layer — no anonymous subscription.

Storage & infra tasks added:

- Configure S3-compatible storage adapter for media uploads; include S3 emulator in CI.
- Provision Redis/pubsub for SSE in staging; fallback to in-process pubsub in single-node dev.

Phase 3 — UI, admin, tests (3.5–5.5 days)

- Build minimal Post editor + Admin dashboards (Payload admin customizations + React where needed).
- Add client SSE integration and optimistic updates.
- Post list query optimization (batched author lookup, cached counts, N+1 elimination).
- Add unit/integration tests and Playwright e2e for core flows.

Phase 4 — Hardening & release (2–4 days)

- Performance tuning, indexes, rate-limits, security review, docs.
- Create rollout & rollback plan for production fork.

## Acceptance Criteria (MVP)

- Can create/edit/delete Posts with author FK to local Member mirror.
- JIT provisioning successfully creates Member on IdP login (idempotent under concurrent requests).
- Comments persist with soft-delete; gating policy enforces can-comment + rate-limit.
- Reactions persist with per-emoji uniqueness; counts update via SSE within ~1s under normal load.
- Polls enforce one-vote-per-member; results derived from PollVote.
- Admin can manage posts and members via Payload admin.
- SSE requires authentication; unauthenticated connections rejected.

## Risks & Mitigations

- Upstream drift: Keep customization confined to `app/collection` layer; document patch points and rebase strategy. Pin beta build + track stable tag (T0.3).
- Identity mismatch: Validate claims (sig/aud/iss/exp) and include an audit log for JIT provisioning; fail-fast on tenant mismatch. Admin bootstrap by `sub` only.
- SSE scale: Start with in-process SSE for Wave 1; add redis/pubsub if load increases. Per-user connection limit prevents single-user fan-out abuse.
- XSS: DOMPurify sanitization on rendered rich text/comments; SVG-safe emoji handling (T2.13b).

## Constitution Check

This feature is built within the `underhelpers/undrepost` repo, which uses the `clai-helpers` constitution (`.specify/memory/constitution.md` v1.5.0). The applicable product-level principles are:

- **Principle VI** (Cross-AI Review Gate): implementation blocked until `analyze.md` PASS + ≥2 external reviewer PASS.
- **Principle VII** (Artifact Versioning): every speckit stage tagged via snapshot scripts.
- **Principle IX** (Two-Phase Review Flow): `specs/<slug>` planning branch; implementation branch created after planning merge.

Product-specific concerns (payment boundary, PII handling, auth posture) are governed by `spec.md` FR-\* requirements + this plan's design decisions, NOT by a separate product constitution. If product-level governance is needed, a product constitution should be authored as a separate decision (flagged by review F11-claude).

## Next steps

1. Create the collections and JIT endpoint skeleton (`Members`, `Posts`).
2. Implement identity flow & admin bootstrap (`sub` only).
3. Build Reaction counters (async + reconciler) and SSE proof-of-concept.
4. Coordinate with undrlla to finalize auction contract and staging integration (T2.13..T2.16), then run contract integration e2e (T2.15).
5. Run integration tests and basic load test on SSE.

---

Generated artifacts: `data-model.md`, `contracts/members.contract.md`, `quickstart.md`, `tasks.md`.
