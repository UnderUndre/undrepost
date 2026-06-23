# data-model.md — undrepost collections (overview)

This document captures the primary collections and key fields for the `undrepost` implementation on Payload.

Collections

- Members (mirror)

  - `id` (UUID) — primary key
  - `sub` (string) — undrlla subject (unique, DB-level constraint)
  - `email` (string)
  - `displayName` (string)
  - `roles` (enum[]) — `admin|member`
  - `profile` (object) — avatarUrl, bio, metadata
  - `syncedAt` (datetime) — last IdP re-sync timestamp (for TTL-based staleness check)
  - `createdAt`, `updatedAt`
  - Indexes: `sub` (unique), `email`

- Posts

  - `id` (UUID)
  - `title` (string)
  - `slug` (string) — unique per instance
  - `content` (rich text / blocks)
  - `author` (relation -> Members)
  - `status` (enum) — draft|published|archived
  - `publishedAt` (datetime)
  - `meta` (object) — tags, excerpts
  - Indexes: `slug`, `publishedAt`

- Comments

  - `id` (UUID)
  - `post` (relation -> Posts)
  - `author` (relation -> Members) — NOT NULL; member-only (gating per FR-011)
  - `body` (text)
  - `createdAt` (datetime)
  - `deletedAt` (datetime, nullable) — soft-delete (FR-011)
  - Indexes: `post`, `createdAt`

- Reactions

  - `id` (UUID)
  - `post` (relation -> Posts)
  - `author` (relation -> Members)
  - `emojiRef` (string) — shortcode (e.g. `:pepe:`, `❤️`); resolved against CustomEmoji collection if server-managed, else treated as unicode literal
  - `createdAt` (datetime)
  - **Unique constraint**: `UNIQUE(post, author, emojiRef)` — one reaction per emoji per member (FR-020)
  - Indexes: `post`, `(post, emojiRef)`

- CustomEmoji

  - `id` (UUID)
  - `shortcode` (string) — unique (e.g. `pepe`)
  - `image` (upload -> S3)
  - `uploadedBy` (relation -> Members)
  - `createdAt` (datetime)
  - Indexes: `shortcode` (unique)

- Polls

  - `id` (UUID)
  - `post` (relation -> Posts)
  - `question` (string)
  - `options` (array of { id, label })
  - `results` (materialized counts — derived from PollVote aggregation)
  - `expiresAt` (datetime, optional)
  - `status` (enum) — active|closed
  - Indexes: `post`, `status`

- PollVote

  - `id` (UUID)
  - `poll` (relation -> Polls)
  - `member` (relation -> Members)
  - `optionId` (string) — references Poll.options[].id
  - `createdAt` (datetime)
  - **Unique constraint**: `UNIQUE(poll, member)` — one vote per member per poll (FR-022)
  - Indexes: `poll`, `(poll, optionId)`

- NewsletterSubscriber

  - `id` (UUID)
  - `email` (string) — unique
  - `status` (enum) — pending|confirmed|unsubscribed
  - `confirmToken` (string) — 32-byte URL-safe base64, expires 24h
  - `confirmTokenExpiresAt` (datetime)
  - `unsubToken` (string) — 32-byte URL-safe base64, single-use
  - `createdAt`, `confirmedAt`, `unsubscribedAt`
  - Indexes: `email` (unique), `confirmToken`, `unsubToken`

- GiveawayRef

  - `id` (UUID)
  - `post` (relation -> Posts)
  - `prizeType` (enum) — content|product (determines winner selection: undrepost-local vs undrlla-delegated)
  - `undrllaRef` (string, nullable) — pointer to undrlla giveaway entity (for product/paid prizes)
  - `status` (enum) — active|closed
  - `winnerId` (relation -> Members, nullable) — set on close (content-only prizes)
  - `closesAt` (datetime)
  - Indexes: `post`, `status`

- AuctionRef
  - `id` (UUID)
  - `post` (relation -> Posts)
  - `undrllaRef` (string) — pointer to undrlla auction entity
  - `presentationConfig` (object) — widget display options
  - Indexes: `post`

Counters strategy

- Store reaction events (Reactions collection) and poll votes (PollVote) as source-of-truth entities.
- Maintain per-post reaction counts and per-poll-option counts as **async-updated materialized counters** (chosen over sync-in-transaction to avoid write bottleneck under SSE load).
- An **idempotent reconciliation job** runs periodically (every 5 min) to correct counter drift by re-aggregating from source entities. Tolerance: ±1 drift acceptable; job corrects to exact.
- Counters are NEVER authoritative — source entities (Reactions, PollVote) are. On dispute, recount from source.

Relations & FK rules

- All authorship and participation FKs reference `Members.id` (mirror). If a Member does not exist, JIT-provision from IdP before completing write operations.
- Comments are member-only — no anonymous author (FR-011 gating requires membership).
- If a Member is deleted upstream (undrlla-side), local mirror record is retained as a tombstone (displayName → "[deleted]") to preserve authored-content integrity. FKs remain valid.
