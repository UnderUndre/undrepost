# tasks.md — undrepost (specs/001-init-from-payload)

Branch: `specs/001-init-from-payload`

## Summary

This document breaks the plan into discrete tasks, assigns agent responsibilities, and identifies parallel lanes and the critical path. Target MVP: posts, comments, member mirror, reactions counts via SSE, and basic admin flows.

## Tasks (by phase)

Phase 0 — Research & scaffolding

- T0.1 [SETUP] Setup fork worktree and branch — Backend-specialist — 0.5d
- T0.2 [BE] Validate Payload v4 beta dependencies and dev scripts — Backend-specialist — 0.5d
- T0.3 [BE] Pin Payload v4-beta build, document rebase/patch points (escape-hatch ladder), track stable tag — Backend-specialist — 0.5d

Phase 1 — Core collections & identity

- T1.1 [DB] Implement `Members` mirror collection schema (with UNIQUE(sub) + syncedAt) — Backend-specialist — 1d
- T1.2 [BE] Implement `Posts` collection schema & basic CRUD API — Backend-specialist — 1d
- T1.3 [BE] JIT-provision endpoint + login hook (upsert Member, ON CONFLICT, re-sync claims, token validation per contract) — Backend-specialist — 2d
- T1.4 [BE] Admin bootstrap by `sub` allowlist only (no email matching) — Backend-specialist — 0.5d

Phase 2 — Social features & SSE

- T2.1 [BE] Implement `Comments` collection (with `deletedAt` soft-delete) and relations — Backend-specialist — 1d
- T2.1b [BE] Comment gating policy: pluggable can-comment hook + rate-limit (FR-011) — Backend-specialist — 1d
- T2.2 [BE] Implement `Reactions` (emojiRef shortcode, UNIQUE(post,author,emojiRef), counters + events) — Backend-specialist — 1d
- T2.3 [OPS] SSE service: authenticated endpoint, heartbeat, replay token, per-user connection limit — Backend-specialist / DevOps (review) — 2d
- T2.4 [DB] Materialized async counters (reaction counts, poll-option counts) with index — Database-architect — 1d
- T2.4b [BE] Counter reconciliation job (idempotent, periodic drift correction from source entities) — Backend-specialist — 0.5d
- T2.5 [BE] Implement `Polls` + `PollVote` collections (UNIQUE(poll,member) constraint, results derived from votes) — Backend-specialist — 1d
- T2.6 [BE] Newsletter backend: `NewsletterSubscriber` (32-byte tokens, 24h confirm expiry, single-use unsub) + confirm/send flow — Backend-specialist — 1.5d
- T2.7 [OPS] Provision SSE infra (Redis/pubsub) for staging/dev + integration — DevOps-engineer — 1d
- T2.8 [OPS] Configure storage adapter (S3 emulation test fixture + CI) — DevOps-engineer — 0.5d
- T2.9 [BE] Input validation & typed error handling (Zod/Payload) — Backend-specialist — 1d
- T2.10 [BE] Structured logging instrumentation (pino-style) — Backend-specialist — 0.5d
- T2.11 [BE] Implement `CustomEmoji` collection + upload handling (SVG sanitization) — Backend-specialist — 1d
- T2.12a [BE] Giveaway entry + content-only winner selection (random from unique entrants) — Backend-specialist — 0.5d
- T2.12b [BE] Giveaway product/paid winner delegation via undrlla contract — Backend-specialist — 0.5d
- T2.13b [BE] Output sanitization (DOMPurify) for rendered rich text + comments + SVG-safe emoji display — Backend-specialist — 0.5d

# Auction integration (commerce seam - hard dependency on undrlla)

- T2.13 [BE] Finalize `AuctionRef` contract adapter interface and reference schema — Backend-specialist — 1d
- T2.14 [BE] Implement `AuctionRef` adapter, bid-submit adapter, and server-side contract shim (test double for CI) — Backend-specialist — 1.5d
- T2.15 [E2E] Contract integration e2e tests (CI: mock contract shim; release gate: undrlla staging endpoint) — Test-engineer — 1.5d
- T2.16 [OPS] Cross-repo staging integration & handshake (undrlla staging endpoint, auth, env) — DevOps-engineer — 1d

Phase 3 — UI, admin, tests

- T3.1 [FE] Payload Admin customizations: posts list, member view — Frontend-specialist — 1.5d
- T3.2 [FE] Post editor and public post page component (SSE client) — Frontend-specialist — 2d
- T3.2b [FE/BE] Post list query optimization (batched author lookup, cached counts, N+1 elimination) — Frontend-specialist / Backend-specialist — 0.5d
- T3.3 [E2E] Integration tests: auth flows, JIT provisioning (incl. concurrent), SSE subscribe — Test-engineer — 2d
- T3.4 [FE] Polls UI & widget integration — Frontend-specialist — 1.5d
- T3.5 [FE] Newsletter admin UI & subscription widget — Frontend-specialist — 0.5d

Phase 4 — Hardening & release

- T4.1 [PERF] Load/scale test for SSE under realistic connections — Performance-optimizer — 1d
- T4.2 [SEC] Security review for token validation (sig/aud/iss/exp), tenant claim, admin bootstrap — Security-auditor — 1d
- T4.3 [DOC] Release notes and migration docs — Documentation-writer — 0.5d
- T4.4 [SEC] Schema audit: verify no commerce money-state persisted (pre-merge gate) — Security-auditor — 0.5d
- T4.5 [OPS] License & copyleft compliance check (runtime path) — DevOps-engineer — 0.5d

## Dependency Graph

- T0.1 → T1.1
- T0.2 → T1.2
- T0.3 → T1.2
- T1.1 → T1.3
- T1.2 → T2.1 + T2.2 + T3.1
- T1.3 → T2.2 + T2.3 + T3.3
- T2.1 → T2.1b
- T2.1b → T2.2
- T2.2 → T2.3 + T2.4
- T2.4 → T2.4b
- T2.4 → T2.3
- T2.7 → T2.3
- T2.8 → T1.2 + T2.1 + T2.11
- T2.5 → T3.4
- T2.6 → T3.5
- T3.1 → T3.2
- T3.2 → T3.2b + T3.3
- T3.4 → T3.3
- T3.5 → T3.3
- T2.3 → T3.2 + T3.3
- T2.1 → T2.13b
- T2.13 → T2.14
- T1.3 → T2.13
- T2.14 → T3.2 + T3.3
- T2.16 → T2.15
- T2.14 → T2.15
- T2.15 → T4.3
- T4.4 → T4.3
- T4.5 → T4.3
- T4.1 → T2.3 + T2.7
- T2.9 → T1.2
- T2.10 → T1.2
- T2.11 → T2.2 + T2.8
- T2.12a → T2.1
- T2.12b → T2.13 + T2.14

## Agent Summary

| Tag     | Role           | Primary tasks                                                                                                   |
| ------- | -------------- | --------------------------------------------------------------------------------------------------------------- |
| [SETUP] | Setup/Worktree | T0.1                                                                                                            |
| [BE]    | Backend        | T0.2, T0.3, T1.2, T1.3, T2.1, T2.1b, T2.2, T2.5, T2.6, T2.9, T2.10, T2.11, T2.12a, T2.12b, T2.13, T2.13b, T2.14 |
| [DB]    | Database       | T1.1, T2.4                                                                                                      |
| [OPS]   | DevOps         | T2.3, T2.7, T2.8, T2.16, T4.5                                                                                   |
| [FE]    | Frontend       | T3.1, T3.2, T3.2b, T3.4, T3.5                                                                                   |
| [E2E]   | Testing        | T3.3, T2.15                                                                                                     |
| [SEC]   | Security       | T4.2, T4.4                                                                                                      |
| [PERF]  | Performance    | T4.1                                                                                                            |
| [DOC]   | Documentation  | T4.3                                                                                                            |

## Parallel lanes

- Lane A (Backend): T0.1 → T1.1 → T1.2 → T1.3 → T2.1 → T2.1b → T2.2 → T2.3
- Lane B (Frontend): T3.1 & T3.2 start once T1.2/T2.2 provide APIs; Polls/Newsletter UI follow their backend tasks
- Lane C (Ops/Infra): T2.7 & T2.8 provision infra in parallel with backend tasks
- Lane D (Tests): T3.3 runs after T1.3 & T2.3

## Critical path

JIT provisioning (T1.3) remains on the critical path because authorship relations and admin flows depend on the `Members` mirror. With the decision to integrate auctions now, the auction integration tasks (T2.13..T2.16) and the contract integration tests (T2.15) are also on the critical path and gate the final release. T2.16 (staging handshake) must complete before T2.15 (contract e2e) can run against staging.

## Total tasks & estimates

- Tasks listed: 37 primary tasks
- Rough calendar estimate (MVP): 2–3 sprints (3–4 weeks) depending on resourcing and review cycles
