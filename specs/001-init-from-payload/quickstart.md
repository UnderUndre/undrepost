# quickstart.md — dev quickstart for undrepost (feature)

Prerequisites

- Node.js 18+ and pnpm (or npm/yarn)
- Docker & Docker Compose (for local DB and optional redis)
- Access to undrlla IdP or a test OIDC provider (for JIT provisioning tests)

Environment

- Create `.env` with the following minimum variables:
  - `DATABASE_URL` — Postgres connection string
  - `IDP_ISSUER` — undrlla OIDC issuer
  - `IDP_JWKS_URL` — JWKS URL to validate tokens
  - `ADMIN_ALLOWLIST` — comma-separated admin sub values (for bootstrap)

Local dev (quick)

1. Clone the repo and switch to the feature branch:

```bash
git checkout -b specs/001-init-from-payload
```

1. Install dependencies and start DB (example using pnpm and docker-compose):

```bash
pnpm install
docker compose up -d postgres
```

1. Run the Payload app (from the payload fork app root):

```bash
pnpm dev
```

1. Test JIT provisioning: perform an OIDC login flow against your test IdP and confirm a `Members` record is created.

Notes

- This quickstart assumes the Payload app is configured at the repo root in the fork. Exact commands may vary depending on workspace layout — consult `README.md` in the forked payload app.
