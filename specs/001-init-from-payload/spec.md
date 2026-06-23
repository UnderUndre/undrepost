# Feature Specification: undrepost — Initialize from Payload Fork

**Feature Branch**: `specs/001-init-from-payload`  
**Created**: 2026-06-20  
**Status**: Draft — clarify resolved 2026-06-20 (identity, commerce depth, monetization)  
**Input**: User description: "вот форк (undrepost), нужно включить все фичи underblog и те, что мы планировали в 003"

---

## Overview

`undrepost` is the rewritten content/social platform, founded as a **GitHub fork of `payloadcms/payload`** (currently `payload-monorepo` v4.0.0-beta.0). It replaces the legacy `underblog` (Next.js Pages Router + Prisma + NextAuth) by re-platforming every existing feature onto Payload-native primitives, then layering the social/commerce features planned in undrlla spec 003.

**Foundation method** (ratified, corrected from 003): fork the upstream monorepo, track upstream via **Sync fork**, keep all customization in the **app/collection layer** so divergence stays small — identical to how `undrlla` is built as a fork of `medusajs/medusa`. Payload is "Medusa for content."

**Bounded context**: `undrepost` owns CONTENT/SOCIAL. `undrlla` (Medusa) owns COMMERCE. Auction/giveaway _engines_ live in undrlla; undrepost surfaces them by reference via a typed contract.

---

## User Scenarios & Testing _(mandatory)_

### User Story 1 - Blog posts on Payload (Priority: P1)

As an **author**, I create, edit, draft/publish, and illustrate blog posts through Payload's admin, so the core blog works on the new foundation.

**Why this priority**: This is feature parity with the legacy blog's core — without it there is no product.

**Independent Test**: Create a post with rich text + cover image in Payload admin, publish it, view it on the public site.

**Acceptance Scenarios**:

1. **Given** an authenticated author, **When** they create a post (title, slug, rich-text body, cover image) and publish, **Then** it renders on the public blog at its slug.
2. **Given** a draft post, **When** it is unpublished, **Then** it is not visible publicly but editable in admin.

---

### User Story 2 - Accounts, roles & admin (Priority: P1)

As a **platform operator**, I have an admin panel with role-gated access (admin vs member) backed by **undrlla's shared Better-Auth IdP**, so one account works across marketplace + blog and content/users are manageable.

**Why this priority**: Auth + admin underpin every other feature.

**Independent Test**: Sign in as admin → full access; sign in as member → restricted; anonymous → public-only.

**Acceptance Scenarios**:

1. **Given** an admin, **When** they open the admin panel, **Then** they can manage posts, comments, media, users.
2. **Given** a member, **When** they authenticate, **Then** they can comment/react but cannot reach admin CRUD.

---

### User Story 3 - Comments, gated (Priority: P1)

As a **reader (member)**, I comment on posts, subject to a gating policy, preserving the legacy gated-comments behavior.

**Independent Test**: Post a comment as an allowed member; verify a gated/over-limit member is blocked; verify soft-delete hides it.

**Acceptance Scenarios**:

1. **Given** a member within the gating policy, **When** they submit a comment, **Then** it appears on the post.
2. **Given** a comment soft-deleted by an admin, **When** the post renders, **Then** the comment is hidden but not physically removed.

---

### User Story 4 - Media uploads (Priority: P1)

As an **author**, I upload images (S3-compatible) used as cover images and inline media, preserving legacy media behavior.

**Independent Test**: Upload an image in admin; confirm it lands in S3-compatible storage and serves on the public post.

**Acceptance Scenarios**:

1. **Given** an author, **When** they upload an image, **Then** it is stored via the configured storage adapter and referenceable by posts.

---

### User Story 5 - Reactions with custom emoji (Priority: P2)

As a **reader**, I react to posts with arbitrary/custom emoji (Misskey-style), so engagement is richer than a single like.

**Independent Test**: React to a post with a custom emoji; toggle off; see aggregated counts per emoji.

**Acceptance Scenarios**:

1. **Given** a member, **When** they react with `:customemoji:`, **Then** the post shows the aggregated reaction with their participation flagged.

---

### User Story 6 - Custom emoji library (Priority: P2)

As an **admin**, I upload and manage server custom emoji usable in posts and reactions.

**Independent Test**: Upload a custom emoji with a shortcode in admin; use `:shortcode:` in a post and a reaction; both render the image.

**Acceptance Scenarios**:

1. **Given** an admin-uploaded emoji `:pepe:`, **When** an author writes `:pepe:` in a post, **Then** it renders as the image.

---

### User Story 7 - Polls (Priority: P2)

As an **author**, I attach a poll to a post; readers vote once; results tally with an expiry.

**Independent Test**: Create a poll, vote as a member, confirm one-vote enforcement and live tally, confirm closed after expiry.

**Acceptance Scenarios**:

1. **Given** an active poll, **When** a member votes, **Then** their vote is recorded once and the tally updates.
2. **Given** an expired poll, **When** a member tries to vote, **Then** voting is rejected and final results show.

---

### User Story 8 - Newsletter (Priority: P2)

As a **reader**, I subscribe to a newsletter; as an **admin**, I send issues to subscribers.

**Independent Test**: Subscribe an email; send a test issue; confirm delivery and unsubscribe.

**Acceptance Scenarios**:

1. **Given** a confirmed subscriber, **When** an admin sends an issue, **Then** the subscriber receives it and can unsubscribe.

---

### User Story 9 - Auction embedded in a post (Priority: P2)

As an **author**, I embed an auction in a post; readers bid inline; the auction **engine runs in undrlla**, undrepost only references and renders it.

**Why this priority**: Committed to this init (full integration, chosen 2026-06-20). Depends on the undrlla contract + shared IdP, so it sequences after the content P1 core (see [Sequencing consequence](#-sequencing-consequence)).

**Independent Test**: Embed an `auctionRef` block in a post; the widget reads live state and submits a bid through the undrlla contract; no auction money-state is stored in undrepost.

**Acceptance Scenarios**:

1. **Given** a post with an `auctionRef`, **When** a reader views it, **Then** the widget shows live auction state from undrlla.
2. **Given** an authenticated bidder, **When** they bid, **Then** the bid is processed by undrlla (validation, payment, close), not by undrepost.

---

### User Story 10 - Giveaway (Priority: P2)

As an **author**, I run a giveaway on a post; entry is content-side, winner selection runs where the prize lives (content-only → undrepost; product/paid → undrlla).

**Independent Test**: Create a giveaway, enter as a member, trigger close, confirm winner selection and notification.

**Acceptance Scenarios**:

1. **Given** an active content giveaway, **When** it closes, **Then** a winner is randomly selected from unique entrants and notified.

---

### User Story 11 - Sync-fork discipline (Priority: P3)

As the **maintainer**, I pull upstream Payload releases via Sync fork with minimal conflict, because customization lives in the app/collection layer, not in forked core.

**Independent Test**: Sync upstream into the fork; confirm conflicts are confined to owned app/collection files, not core packages.

**Acceptance Scenarios**:

1. **Given** new upstream commits, **When** the fork is synced, **Then** owned features still build and conflicts (if any) are limited to the app layer.

---

### Edge Cases

- **Custom emoji shortcode collision / missing image** → render the literal `:shortcode:` text, never a broken image.
- **Double-vote / double-react race** → unique constraint per (poll,user) / (post,user,emoji); concurrent submits resolve to one.
- **Auction bid when undrlla is unreachable** → widget degrades to read-only with a clear "bidding temporarily unavailable", never accepts a bid undrepost can't honor.
- **Comment gating policy denies** → typed, user-friendly rejection, not a silent drop.
- **Payload v4-beta breaking change on Sync fork** → bounded migration via upstream notes; divergence kept small to limit blast radius.
- **Legacy data** → greenfield rewrite; no production data migration assumed (see Assumptions).

---

## Requirements _(mandatory)_

### Foundation Requirements

- **FR-001**: undrepost MUST be founded as a GitHub fork of `payloadcms/payload`, tracked against upstream via Sync fork. Customizations MUST live in the app/collection layer; core packages MUST NOT be edited except via the escape-hatch ladder (plugin → component → upstream PR → patch-package → scoped fork-commit, recorded).
- **FR-002**: The runtime core path MUST remain MIT-clean; proprietary logic stays closed without copyleft triggers.
- **FR-003**: undrepost MUST NOT model commerce money-state (bids, payments, paid-subscription ledgers) in its own data layer; such state MUST live in undrlla and be reached via a typed contract.

### Ported Features (parity with legacy underblog)

- **FR-010**: Posts MUST support title, unique slug, rich-text body, cover image, and draft/publish — re-expressed as a Payload `Posts` collection (legacy `BlogPost`). Rich text MAY move from TipTap to Payload's native editor.
- **FR-011**: Comments MUST support member authorship, soft-delete, and a **gating policy** (can-comment / rate-limit), preserving the legacy pluggable-gating behavior via Payload access control / hooks.
- **FR-012**: Media uploads MUST use an S3-compatible storage adapter, preserving legacy storage behavior, via Payload's upload + storage adapter.
- **FR-013**: The legacy **pluggable-adapter philosophy** (swappable Auth / Gating / Storage) SHOULD be preserved through Payload plugins/config so undrepost remains a reconfigurable "forkable starter," not a hardwired build.
- **FR-014**: Authentication MUST delegate to **undrlla's Better-Auth as a shared IdP** from day one (OIDC/JWT) — one account spans marketplace + blog. undrepost is a relying party; Payload's admin + access control consume the undrlla-issued identity. (Chosen 2026-06-20; required by full embedded auctions, FR-030.) The Payload-native admin dashboard MUST role-gate on that identity. _Migration note: replaces legacy NextAuth._

### New Features (planned in 003)

- **FR-020**: Reactions MUST support arbitrary/custom-emoji reactions on posts with per-emoji aggregation and per-user participation, enforced unique per (post, user, emoji).
- **FR-021**: Custom Emoji MUST be admin-managed (shortcode + image upload) and usable in both post bodies and reactions.
- **FR-022**: Polls MUST attach to posts with options, single-vote-per-member enforcement (unique per (poll, user)), live tally, and expiry.
- **FR-023**: Newsletter MUST support subscribe/confirm/unsubscribe and admin-issued sends to confirmed subscribers.

### Commerce Integration (undrlla boundary)

- **FR-030**: Auctions MUST be **fully integrated in this init** (chosen 2026-06-20): a live `auctionRef` widget reads auction state and submits bids through the typed undrlla contract in real time. The auction engine (bids, ACID validation, payment, close, winner) runs in undrlla; undrepost stores no money-state. Requires the shared IdP (FR-014) and a working undrlla auction engine + contract (see Dependencies — **not built in undrlla yet**).
- **FR-031**: Giveaways MUST be **fully integrated in this init**: entry on the content side; winner selection runs where the prize lives (content-only prize → undrepost; product/paid prize → undrlla via the contract).
- **FR-032**: Membership / paid subscription is **OUT OF SCOPE for init** (chosen 2026-06-20). Init ships **free accounts + newsletter** only; paid membership/subscription is deferred to a later spec (billing would route through undrlla). This does not affect auction payments, which flow through undrlla per FR-030.

### Functional Requirements (cross-cutting)

- **FR-040**: All inbound mutations MUST validate input (Zod or Payload field validation) and return typed errors, preserving the legacy `withErrorHandler` discipline.
- **FR-041**: Structured logging (pino-style) with context MUST be retained; no secrets in logs.
- **FR-042**: Real-time surfaces (auction bids, live reaction/poll counts) MUST update without full reload (SSE/WebSocket); auction real-time originates in undrlla.

### Key Entities (Payload collections unless noted)

- **User / Member**: account, role (admin/member), profile. (legacy `User`; identity backend per FR-014)
- **Post**: title, slug, rich-text body, cover image, status. (legacy `BlogPost`)
- **Comment**: body, author, post, soft-delete. (legacy `Comment`)
- **Media**: S3-backed upload. (legacy `MediaFile`)
- **CustomEmoji**: shortcode (unique), image.
- **Reaction**: post, user, emoji — unique per triple.
- **Poll / PollVote**: poll on post; vote unique per (poll, user).
- **NewsletterSubscriber**: email, status, confirm/unsub tokens.
- **AuctionRef / GiveawayRef** (reference objects, not money-state): pointer to the undrlla entity + presentation config.

---

## Success Criteria _(mandatory)_

- **SC-001**: 100% of legacy underblog features (posts, gated comments, auth/admin, media) are reachable in undrepost with equivalent behavior.
- **SC-002**: All four new social features (reactions, custom emoji, polls, newsletter) are usable end-to-end by a member/admin.
- **SC-003**: Adding any new collection changes only owned app-layer files — 0 edits in forked Payload core packages.
- **SC-004**: A Sync-fork pull of upstream Payload completes with conflicts confined to the app layer (none in core).
- **SC-005**: No auction/payment money-state exists in the undrepost data layer (verified by schema review); all of it is in undrlla.
- **SC-006**: License check passes with 0 copyleft packages in the runtime core path.

---

## Assumptions

- **Greenfield rewrite**: no production data migration from legacy underblog assumed; legacy schema (Prisma models) is a _mapping reference_ for the new Payload collections, not a data source.
- Foundation is **Payload v4-beta** per the actual fork (`payload-monorepo` 4.0.0-beta.0). Beta churn is accepted and absorbed via Sync-fork discipline.
- Rich text migrates from TipTap to Payload's native editor (Lexical) unless a hard reason to keep TipTap emerges.
- The blog app lives inside the fork (exact location — `apps/`, `templates/`, or root `app/` — is a plan-stage decision).
- undrlla is the commerce + (optionally) identity backbone; the two integration seams (control-plane provisioning, runtime auction contract) are per the pending bounded-context ADR.

## Dependencies

- **undrlla Better-Auth IdP** — **HARD** dependency (FR-014): identity from day one. undrlla must expose an OIDC/JWT IdP endpoint before undrepost auth is functional.
- **undrlla auction engine + typed contract** — **HARD** dependency (FR-030/031): full auction/giveaway integration. ⚠️ The auction engine does **not exist in undrlla yet** — it must be built there first (separate spec). The auction parts of this init BLOCK on it.
- The corrected foundation decision (fork payloadcms/payload), originally drafted as undrlla spec 003.
- Payload v4-beta upstream.

## Out of Scope

- **Membership / paid subscription** (deferred; later spec — billing via undrlla).
- The auction/giveaway **engine** implementation (lives in undrlla; separate spec).
- undrepost **hosting/provisioning** by undrlla as a sold product (control-plane seam; separate spec).
- Multi-tenant density model (later phase).
- Detailed undrepost↔undrlla contract schema (separate contract artifact).

---

## Resolved Decisions (clarify 2026-06-20)

1. **Identity** (FR-014) → **undrlla Better-Auth IdP from day one** (shared account, marketplace + blog).
2. **Commerce depth** (FR-030/031) → **full auction + giveaway integration now** (live widget + undrlla contract).
3. **Monetization** (FR-032) → **out of scope for init** (free accounts + newsletter; paid membership later).

### ⚠️ Sequencing consequence

Choices 1 + 2 turn undrepost init into a **hard downstream of undrlla**: it cannot fully ship until undrlla exposes (a) a Better-Auth IdP endpoint and (b) a working auction engine + typed contract — **and the auction engine is not built in undrlla yet.** Plan accordingly. Recommended **two-wave split**:

- **Wave 1 — Content MVP (P1)**: foundation fork + posts, comments (gated), media, admin, reactions, custom emoji, polls, newsletter. **Zero undrlla dependency** — ships standalone. (Identity: stand up the undrlla IdP early, or run a temporary local auth seam swapped for the IdP before Wave 2.)
- **Wave 2 — Commerce (P2)**: embedded auction + giveaway. **Gated on** undrlla IdP + auction engine + contract being ready.

This keeps the blog from being blocked by undrlla delivery while honoring the "full commerce" decision.
