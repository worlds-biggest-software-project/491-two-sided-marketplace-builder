# Two-Sided Marketplace Builder — Phased Development Plan

> Project: 491-two-sided-marketplace-builder · Created: 2026-05-31
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the four `data-model-suggestion-*.md` files into concrete technology decisions, an architecture, and a sequence of implementable phases. The chosen data model is **Suggestion 3 (Hybrid Relational + JSONB)** augmented with `pgvector` for semantic search — relational integrity for money and identity, JSONB for per-vertical listing attributes and marketplace configuration, so a single schema serves product / service / rental verticals without migrations.

---

## Core Requirements (synthesis)

- **What it does**: An open-source, API-first platform for launching and operating two-sided transaction marketplaces (buyers ↔ sellers). It closes the mid-market gap between fast-but-limited SaaS (Sharetribe) and powerful-but-inaccessible enterprise platforms (Mirakl), with AI-native onboarding, trust scoring, search, and listing optimisation built in.
- **Users**: (1) Startup founders launching niche verticals; (2) enterprise product teams adding a marketplace channel; (3) agencies building white-label marketplaces for clients. Plus the runtime personas: **operator** (admin), **seller**, **buyer**.
- **Differentiators**: configurable verticals from one schema; B2B-capable; AI seller onboarding (document review + category classification) replacing manual review queues; real-time fraud/trust scoring; embedding-based semantic search; AI listing optimiser. Self-hostable with full code ownership.
- **Deployment model**: Headless, API-first. Self-hosted (Docker Compose) and managed-cloud. Multi-tenant capable (multiple marketplace instances per database, isolated by `marketplace_id`). A default responsive storefront ships alongside the API.
- **Integration surface**: Stripe Connect (split payments, payouts), an LLM provider via a pluggable gateway (onboarding, optimiser, embeddings), Persona (KYC), SendGrid/Twilio (notifications), object storage (S3-compatible) for images and documents.
- **Standards compliance**: OpenAPI 3.1 for the public API; OAuth 2.0 / OIDC for auth; JSON Schema 2020-12 for listing/webhook payloads; RFC 9110 HTTP semantics; HMAC-SHA256 webhook signature verification; PCI scope offloaded to Stripe; GDPR data-subject controls (erasure, export); MCP server exposing core resources for AI agents.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language | **TypeScript (Node 22 LTS)** | The project is API/integration/frontend-heavy, not ML-training-heavy. One language across API, default storefront, and SDK generation reduces surface area. Strong typing matches the JSON Schema / OpenAPI 3.1 contract-first goal. AI work is API calls to an LLM gateway, which TS handles fine. |
| API framework | **Fastify + `@fastify/swagger`** | Fastify's JSON-Schema-first validation maps directly to OpenAPI 3.1 generation (the standards requirement) and gives per-route request/response validation for free. Higher throughput than Express for the high-listing read path. |
| ORM / DB access | **Drizzle ORM** | Type-safe SQL with first-class JSONB and `pgvector` support, lightweight migrations, no heavy runtime. Keeps the hybrid relational+JSONB model (Suggestion 3) honest at the type level. |
| Database | **PostgreSQL 16 + `pgvector` + `pg_trgm`** | Single store for relational integrity (money, transactions, payouts), JSONB flexibility (vertical attributes), trigram text search, and vector similarity (semantic search). Avoids a second datastore for the MVP. RLS available for tenant isolation. |
| Cache / queue | **Redis 7 + BullMQ** | Async work: webhook fan-out, payout scheduling, embedding generation, AI onboarding reviews, notification sending, fraud scoring. BullMQ gives retries, delayed jobs (scheduled payouts), and dead-letter handling. |
| Object storage | **S3-compatible (MinIO in dev, AWS S3 / R2 in prod)** via `@aws-sdk/client-s3` | Listing images and KYC documents. Presigned upload URLs keep large files off the API process. |
| Payments | **Stripe Connect** (`stripe` Node SDK) | De-facto marketplace standard (research + standards). Separate charges & transfers model for commission + scheduled payouts; hosted flows keep operator out of PCI scope. |
| LLM gateway | **Pluggable `LlmProvider` interface**, default OpenAI-compatible (`openai` SDK), embeddings `text-embedding-3-small` (1536-dim) | Self-hosters may point at any OpenAI-compatible endpoint (incl. local). 1536-dim matches the `VECTOR(1536)` column. Provider behind an interface so fraud/onboarding/optimiser code never hardcodes a vendor. |
| KYC | **Persona** (pluggable `KycProvider`) | Configurable KYC favoured by marketplaces (standards.md). Behind an interface so the AI document-review path can substitute. |
| Notifications | **SendGrid (email) + Twilio (SMS)** behind a `Notifier` interface | Transactional lifecycle messages; HMAC-verified event webhooks. |
| Frontend (default storefront) | **Next.js 16 (App Router) + Tailwind + shadcn/ui** | Optional, headless-friendly. Server Components for SEO on listing pages; consumes the public API only (no direct DB access) to prove the API is complete. |
| Auth | **OAuth 2.0 / OIDC**: password + PKCE for first-party clients, client-credentials for server integrations; JWT access tokens + rotating refresh tokens | Matches standards.md (RFC 6749, OIDC). Sharetribe's two-tier (Marketplace API / Integration API) pattern is mirrored. |
| Validation / contracts | **Zod → JSON Schema → OpenAPI 3.1** | Single source of truth for types, runtime validation, and the published spec. |
| Testing | **Vitest** (unit/integration) + **Supertest** (HTTP) + **Testcontainers** (real Postgres/Redis) + **Playwright** (storefront e2e) | Fast unit loop; real-dependency integration via ephemeral containers; e2e for the storefront happy paths. |
| Quality | **ESLint + Prettier + `tsc --noEmit`** | Standard TS toolchain; enforced in CI and Definition of Done. |
| Package manager / monorepo | **pnpm workspaces + Turborepo** | Multiple packages (api, worker, storefront, sdk, shared schema) with shared types and cached builds. |
| Containerisation | **Docker + docker-compose** | Self-hosted target. Compose brings up Postgres, Redis, MinIO, API, worker, storefront for one-command local/self-host runs. |
| Migrations | **drizzle-kit** | Versioned, reviewable SQL migrations; required in Definition of Done. |

### Project Structure

```
two-sided-marketplace-builder/
├── package.json                      # pnpm workspace root
├── pnpm-workspace.yaml
├── turbo.json
├── docker-compose.yml                # postgres, redis, minio, api, worker, storefront
├── Dockerfile.api
├── Dockerfile.worker
├── .env.example
├── packages/
│   ├── shared/                       # cross-cutting, no runtime deps on api/worker
│   │   ├── src/
│   │   │   ├── schema/               # Zod schemas (listing, transaction, webhook payloads)
│   │   │   ├── enums.ts              # status enums shared with DB
│   │   │   ├── errors.ts             # AppError taxonomy + RFC 9457 problem+json mapping
│   │   │   └── money.ts              # integer-minor-unit money helpers
│   │   └── package.json
│   ├── db/                           # Drizzle schema + migrations + repositories
│   │   ├── src/
│   │   │   ├── schema/               # table definitions (marketplace, user, listing, ...)
│   │   │   ├── migrations/           # drizzle-kit generated SQL
│   │   │   ├── repositories/         # data-access functions per aggregate
│   │   │   └── client.ts             # pool + tenant-scoped query helper
│   │   └── package.json
│   ├── api/                          # Fastify HTTP service
│   │   ├── src/
│   │   │   ├── server.ts
│   │   │   ├── plugins/              # auth, tenant-resolver, error-handler, swagger
│   │   │   ├── routes/
│   │   │   │   ├── auth/  users/  sellers/  buyers/
│   │   │   │   ├── listings/  categories/  search/
│   │   │   │   ├── transactions/  payouts/  reviews/  disputes/
│   │   │   │   ├── messaging/  admin/  webhooks/   # stripe, persona, sendgrid inbound
│   │   │   └── services/             # business logic (TransactionService, PayoutService, ...)
│   │   └── package.json
│   ├── worker/                       # BullMQ processors
│   │   ├── src/
│   │   │   ├── queues.ts
│   │   │   └── processors/           # embeddings, payouts, onboarding-review, fraud, notify
│   │   └── package.json
│   ├── integrations/                 # external-system adapters behind interfaces
│   │   ├── src/
│   │   │   ├── payments/stripe/      # implements PaymentProvider
│   │   │   ├── llm/                  # LlmProvider (chat + embeddings)
│   │   │   ├── kyc/persona/          # KycProvider
│   │   │   ├── notify/               # Notifier (sendgrid, twilio)
│   │   │   └── storage/s3/           # ObjectStore
│   │   └── package.json
│   ├── mcp/                          # MCP server exposing listings/transactions/analytics
│   ├── sdk/                          # generated TS client from OpenAPI (Phase 4 output)
│   └── storefront/                   # Next.js 16 default storefront
└── tests/
    ├── fixtures/                     # sample listings, stripe webhook bodies, KYC payloads
    └── e2e/                          # Playwright specs
```

---

## Phase 1: Foundation — Monorepo, Database, Config, Errors

### Purpose
Stand up the repository, the Postgres schema (hybrid relational + JSONB + pgvector), shared validation, configuration, and the error model. After this phase the API boots, connects to a migrated database, and serves a health endpoint with a generated OpenAPI document. Nothing domain-specific works yet, but every later phase builds on these primitives.

### Tasks

#### 1.1 — Monorepo & tooling bootstrap
**What**: pnpm + Turborepo workspace with all packages scaffolded, ESLint/Prettier/tsc wired, and a one-command Docker Compose dev environment.

**Design**:
- `docker-compose.yml` services: `postgres` (image `pgvector/pgvector:pg16`), `redis:7`, `minio`, plus `api`, `worker`, `storefront` (built from `Dockerfile.*`).
- `.env.example` keys: `DATABASE_URL`, `REDIS_URL`, `JWT_SECRET`, `JWT_ACCESS_TTL=900`, `JWT_REFRESH_TTL=2592000`, `S3_ENDPOINT`, `S3_BUCKET`, `S3_ACCESS_KEY`, `S3_SECRET_KEY`, `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`, `LLM_BASE_URL`, `LLM_API_KEY`, `LLM_CHAT_MODEL`, `LLM_EMBED_MODEL=text-embedding-3-small`, `KYC_PROVIDER=persona`, `PERSONA_API_KEY`, `SENDGRID_API_KEY`, `APP_BASE_URL`.
- Central config loader in `packages/shared`:
```ts
export const Config = z.object({
  databaseUrl: z.string().url(),
  redisUrl: z.string().url(),
  jwtSecret: z.string().min(32),
  jwtAccessTtl: z.coerce.number().default(900),
  jwtRefreshTtl: z.coerce.number().default(2_592_000),
  appBaseUrl: z.string().url(),
  // ... provider keys, all optional unless feature enabled
});
export type AppConfig = z.infer<typeof Config>;
export function loadConfig(env = process.env): AppConfig; // throws on invalid
```

**Testing**:
- Unit: `loadConfig` with full valid env → typed `AppConfig` with defaults applied.
- Unit: `loadConfig` missing `DATABASE_URL` → throws with `databaseUrl` in message.
- Unit: `loadConfig` `JWT_SECRET` shorter than 32 chars → throws.
- Integration (real): `docker compose up` then `pnpm -w turbo run build` → all packages build, exit 0.

#### 1.2 — Database schema & migrations (hybrid relational + JSONB + pgvector)
**What**: Drizzle schema and the initial migration implementing the marketplace data model.

**Design**: Adopt Suggestion 3's hybrid model. Money is always relational `NUMERIC` (never JSON). Vertical-specific listing data lives in `listing.attributes JSONB`. Marketplace config lives in `marketplace.config JSONB`. Add `listing.embedding VECTOR(1536)` from Suggestion 1 for semantic search. Enums implemented as Postgres enums and mirrored in `packages/shared/enums.ts`.

Core tables (full DDL in migration):
```sql
CREATE EXTENSION IF NOT EXISTS "pgcrypto";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";
CREATE EXTENSION IF NOT EXISTS "vector";

CREATE TYPE user_role AS ENUM ('buyer','seller','both','operator','admin');
CREATE TYPE verification_status AS ENUM ('pending','in_review','verified','rejected');
CREATE TYPE listing_status AS ENUM ('draft','pending_review','active','paused','sold','expired','rejected');
CREATE TYPE listing_type AS ENUM ('product','service','rental');
CREATE TYPE transaction_status AS ENUM
  ('inquiry','booked','awaiting_payment','payment_held','in_progress',
   'delivered','completed','disputed','refunded','cancelled');
CREATE TYPE payout_status AS ENUM ('pending','scheduled','processing','completed','failed');
CREATE TYPE dispute_status AS ENUM ('open','under_review','resolved_buyer','resolved_seller','escalated','closed');

CREATE TABLE marketplace (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(255) NOT NULL,
  slug VARCHAR(100) NOT NULL UNIQUE,
  custom_domain VARCHAR(255),
  config JSONB NOT NULL DEFAULT '{}',          -- branding, enabled verticals, feature flags
  default_currency CHAR(3) NOT NULL DEFAULT 'USD',
  commission_rate NUMERIC(5,4) NOT NULL DEFAULT 0.1000,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE "user" (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  marketplace_id UUID NOT NULL REFERENCES marketplace(id) ON DELETE CASCADE,
  email VARCHAR(320) NOT NULL,
  password_hash TEXT,
  display_name VARCHAR(150) NOT NULL,
  role user_role NOT NULL DEFAULT 'buyer',
  locale VARCHAR(10) DEFAULT 'en',
  is_active BOOLEAN NOT NULL DEFAULT TRUE,
  email_verified_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (marketplace_id, email)
);

CREATE TABLE listing (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  marketplace_id UUID NOT NULL REFERENCES marketplace(id) ON DELETE CASCADE,
  seller_id UUID NOT NULL REFERENCES seller_profile(id) ON DELETE CASCADE,
  category_id UUID REFERENCES category(id) ON DELETE SET NULL,
  listing_type listing_type NOT NULL DEFAULT 'product',
  title VARCHAR(255) NOT NULL,
  description TEXT,
  status listing_status NOT NULL DEFAULT 'draft',
  price NUMERIC(12,2) NOT NULL,
  currency CHAR(3) NOT NULL DEFAULT 'USD',
  quantity INTEGER DEFAULT 1,
  attributes JSONB NOT NULL DEFAULT '{}',       -- vertical-specific fields
  location_lat NUMERIC(9,6),
  location_lng NUMERIC(9,6),
  ai_quality_score NUMERIC(3,2),
  embedding VECTOR(1536),
  published_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_listing_tenant_status ON listing (marketplace_id, status);
CREATE INDEX idx_listing_attrs ON listing USING gin (attributes);
CREATE INDEX idx_listing_title_trgm ON listing USING gin (title gin_trgm_ops);
CREATE INDEX idx_listing_embedding ON listing USING hnsw (embedding vector_cosine_ops);
```
Remaining tables — `seller_profile`, `buyer_profile`, `seller_verification_document`, `category`, `category_attribute`, `listing_image`, `listing_availability`, `transaction`, `transaction_status_history`, `payout`, `payout_line`, `conversation`, `conversation_participant`, `message`, `review`, `dispute`, `fraud_signal`, `browse_event` — follow Suggestion 1's columns, with JSONB swapped in for `seller_verification_document.ai_review_result`, `fraud_signal.details`, and `transaction` metadata. Every tenant-scoped table carries `marketplace_id` with an index.

**Testing**:
- Integration (Testcontainers Postgres): run all migrations from empty DB → 0 errors; `\d listing` shows `embedding` as `vector`, `attributes` as `jsonb`.
- Integration: insert listing with `attributes = {"shoe_size":42}` then query `attributes @> '{"shoe_size":42}'` → 1 row.
- Integration: insert two listings with embeddings, ORDER BY `embedding <=> $1` → nearest returned first.
- Integration: delete a `marketplace` row → cascaded delete removes its users and listings.
- Unit: `enums.ts` values exactly match the SQL enum members (snapshot test).

#### 1.3 — Error model & HTTP problem responses
**What**: An `AppError` taxonomy mapped to RFC 9457 `application/problem+json`, wired as the Fastify error handler.

**Design**:
```ts
export class AppError extends Error {
  constructor(public code: ErrorCode, public httpStatus: number,
              message: string, public details?: unknown) { super(message); }
}
export type ErrorCode =
  | 'validation_failed' | 'unauthenticated' | 'forbidden' | 'not_found'
  | 'conflict' | 'payment_failed' | 'rate_limited' | 'tenant_not_resolved'
  | 'invalid_state_transition' | 'internal';
// Handler emits: { type, title, status, code, detail, errors? }
```
Validation errors from Zod/Fastify map to `validation_failed` (422) with a field-keyed `errors` array.

**Testing**:
- Unit: `AppError('not_found',404,...)` serialised by handler → status 404, body `code:"not_found"`, correct `type` URI.
- Integration: POST a malformed body to any route → 422 problem+json listing the offending field.
- Unit: an unexpected thrown `Error` → 500 with `code:"internal"` and no stack leaked in body.

---

## Phase 2: Identity, Auth & Tenancy

### Purpose
Implement the OAuth 2.0 / OIDC-aligned authentication, the buyer/seller/operator role model, and tenant resolution. Every subsequent route depends on knowing *who* the caller is and *which marketplace* they act within. Mirrors Sharetribe's two-tier model: PKCE for first-party clients, client-credentials for server integrations.

### Tasks

#### 2.1 — Registration, login, token issuance
**What**: Email/password registration (Argon2id), login, JWT access + rotating refresh tokens, email verification.

**Design**:
- `POST /v1/auth/register` `{ email, password, displayName, role }` → 201 `{ userId }`; enqueues verification email.
- `POST /v1/auth/login` `{ email, password }` → `{ accessToken, refreshToken, expiresIn }`.
- `POST /v1/auth/refresh` `{ refreshToken }` → new pair; old refresh token revoked (rotation, stored hashed in `refresh_token` table).
- `POST /v1/auth/verify-email` `{ token }` → sets `email_verified_at`.
- Access JWT claims: `sub` (userId), `mkt` (marketplaceId), `role`, `scope`, `exp`. Signed HS256 with `JWT_SECRET`.
- Passwords hashed with Argon2id; never returned.

**Testing**:
- Unit: register hashes with Argon2id; stored hash ≠ plaintext; `verify(hash, pw)` true.
- Integration (real DB): register → login → access token decodes to the user's `sub`/`mkt`/`role`.
- Integration: refresh with a used (rotated) token → 401, and the new token still valid.
- Integration: login with wrong password → 401 `unauthenticated`, no user enumeration in message.
- Integration: register duplicate email within same marketplace → 409 `conflict`.

#### 2.2 — PKCE & client-credentials flows; scopes
**What**: OAuth 2.0 authorization-code + PKCE for first-party SPA/mobile, and client-credentials for server integrations, with scope enforcement.

**Design**:
- `GET /v1/oauth/authorize` (PKCE: `code_challenge`, `S256`) and `POST /v1/oauth/token` (`grant_type=authorization_code|client_credentials|refresh_token`).
- `oauth_client` table: `{ id, marketplace_id, name, type('public'|'confidential'), secret_hash, redirect_uris[], allowed_scopes[] }`.
- Scopes: `listings:read listings:write transactions:read transactions:write payouts:read admin:* mcp:read`.
- Integration tokens (client-credentials) act with `marketplace_id` of the client; cannot use buyer/seller-only scopes.

**Testing**:
- Integration: PKCE round trip with matching verifier → token; mismatched verifier → 400 `invalid_grant`.
- Integration: client-credentials token requesting `admin:*` without grant → 403.
- Unit: scope check helper `requireScope('listings:write')` rejects token lacking scope.

#### 2.3 — Tenant resolution & RBAC guards
**What**: Resolve the active `marketplace_id` per request and enforce role/ownership.

**Design**:
- Fastify plugin resolves tenant in order: JWT `mkt` claim → `X-Marketplace-Slug` header → custom domain `Host` match. Failure → `tenant_not_resolved` (400).
- `request.tenant.marketplaceId` injected; all repository calls require it (see 1.2). A `tenantScoped(db, marketplaceId)` helper auto-adds `WHERE marketplace_id = $`.
- Guards: `requireRole('operator'|'admin')`, `requireSelfOrOperator(resourceOwnerId)`.

**Testing**:
- Integration: request with token for marketplace A querying marketplace B's listing by id → 404 (not 403; no cross-tenant existence leak).
- Unit: tenant resolver prefers JWT claim over header.
- Integration: buyer calling an operator-only route → 403 `forbidden`.

---

## Phase 3: Catalogue — Categories, Listings & Media

### Purpose
The supply side of the marketplace. Sellers create listings across configurable verticals (product/service/rental); operators define categories and per-category attributes. This is the heart of the platform's configurability and ships early. After this phase a marketplace has a browsable catalogue (basic listing fetch) even before search/payments exist.

### Tasks

#### 3.1 — Category tree & configurable attributes
**What**: Operator-managed hierarchical categories with typed, per-category attribute definitions.

**Design**:
- `category` (self-referential `parent_id`) and `category_attribute` (`attribute_type` ∈ text|number|boolean|select|multi_select|date, `is_required`, `options[]`).
- `POST /v1/admin/categories`, `PATCH`, `DELETE`, `GET /v1/categories` (public, tenant-scoped, returns tree).
- `POST /v1/admin/categories/:id/attributes` defines the attribute schema a listing in that category must satisfy.
- Server compiles a category's attributes into a Zod schema used to validate `listing.attributes` on write.

**Testing**:
- Integration: create parent + child category → `GET /v1/categories` returns nested tree.
- Integration: required `select` attribute with `options:['s','m','l']` → listing write with `attributes.size='xl'` rejected 422; `='m'` accepted.
- Integration: deleting a category with listings → `category_id` set null on those listings (no listing loss).

#### 3.2 — Listing CRUD with vertical attributes & lifecycle
**What**: Create/read/update/delete listings with JSONB attributes validated against the category schema, and the `listing_status` lifecycle.

**Design**:
- `POST /v1/listings` (seller scope) `{ categoryId, listingType, title, description, price, currency, quantity, attributes, location? }` → 201 draft.
- `POST /v1/listings/:id/publish` → `draft|paused → pending_review` (if marketplace config requires moderation) else `active`.
- State machine: `draft → pending_review → active → paused → active`, `active → sold|expired`, `pending_review → rejected`. Illegal transitions → `invalid_state_transition` (409).
- Rentals/services: `listing_availability` rows with date ranges; availability checked at booking time (Phase 5).
- On create/update of an `active` or `pending_review` listing, enqueue an embedding job (Phase 7) and an AI-quality job (Phase 8) — fire-and-forget, listing usable without them.

**Testing**:
- Integration: create draft → publish (no moderation) → status `active`, `published_at` set.
- Integration: publish then `active → inquiry`-style illegal transition request → 409.
- Integration: seller A editing seller B's listing → 403.
- Integration: create listing with attributes violating category schema → 422 with field path.
- Unit: state-machine `canTransition(from,to)` truth table.

#### 3.3 — Media uploads (presigned S3)
**What**: Listing images and KYC documents uploaded directly to object storage via presigned URLs.

**Design**:
- `POST /v1/listings/:id/images/presign` `{ contentType, fileName }` → `{ uploadUrl, objectKey }` (presigned PUT, 5-min expiry, content-type pinned, max 10 MB enforced via policy).
- `POST /v1/listings/:id/images` `{ objectKey, altText, sortOrder }` → records `listing_image` after client uploads.
- `ObjectStore` interface in `integrations/storage` with S3 impl; MinIO in dev.

**Testing**:
- Integration (MinIO container): presign → PUT image bytes to URL → 200; record image → appears in listing.
- Integration: presign with `contentType:'application/x-msdownload'` → 422 (only image/* allowed for listing images).
- Integration: presigned URL used after expiry → 403 from storage.

---

## Phase 4: Public API Surface, OpenAPI 3.1 & SDK

### Purpose
Lock the API contract. Generate the OpenAPI 3.1 document from route schemas, publish docs, and generate a TypeScript SDK — proving the platform is genuinely headless and consumable by custom frontends (the README's core promise). This phase is mostly cross-cutting hardening of Phases 2–3 routes plus tooling.

### Tasks

#### 4.1 — OpenAPI 3.1 generation & docs endpoint
**What**: Every route declares Zod request/response schemas surfaced as OAS 3.1 at `/openapi.json` and Swagger UI at `/docs`.

**Design**:
- `@fastify/swagger` configured for OAS 3.1; Zod schemas converted via `zod-to-json-schema` (JSON Schema 2020-12). Security schemes documented: `oauth2` (PKCE + client-credentials), `bearerAuth`.
- CI step fails if the committed `openapi.json` snapshot drifts from generated output.

**Testing**:
- Integration: `GET /openapi.json` → valid OAS 3.1 (validate with `@apidevtools/swagger-parser`).
- Integration: each registered route appears in the spec with documented responses incl. the problem+json error schema.
- Unit: spec `openapi` field === `3.1.0`.

#### 4.2 — Generated TypeScript SDK
**What**: `packages/sdk` generated from `openapi.json`, typed client used by the storefront.

**Design**: `openapi-typescript` + a thin fetch wrapper handling token refresh and tenant header injection. Published build artefact; regenerated in CI from the spec.

**Testing**:
- Integration: SDK `listings.list()` against a running API → typed array of listings.
- Build: SDK type-checks (`tsc --noEmit`) against the current spec.

---

## Phase 5: Transactions, Stripe Connect & Payouts

### Purpose
The money. Implement the two-sided transaction lifecycle, Stripe Connect onboarding for sellers, split payments with automatic commission, escrow-style payment holds, and scheduled payouts. This is the second half of the core value proposition and the most correctness-critical phase.

### Tasks

#### 5.1 — Seller Stripe Connect onboarding
**What**: Connected-account creation and onboarding link for sellers; payout eligibility gating.

**Design**:
- `PaymentProvider` interface (`integrations/payments`) with Stripe impl: `createConnectedAccount`, `createAccountLink`, `createPaymentIntent`, `capture`, `createTransfer`, `refund`.
- `POST /v1/sellers/me/stripe/onboard` → creates Express connected account, stores `seller_profile.stripe_account_id`, returns onboarding URL.
- Stripe `account.updated` webhook flips `seller_profile` payout-eligible once `charges_enabled && payouts_enabled`.

**Testing**:
- Integration (Stripe mock/`stripe-mock`): onboard → connected account id persisted.
- Integration: `account.updated` webhook with `payouts_enabled:true` → seller marked eligible.
- Unit: signature verification rejects a webhook with bad `Stripe-Signature` → 400, no state change.

#### 5.2 — Transaction lifecycle & split payment
**What**: The `transaction` state machine with separate-charges-and-transfers, commission deduction, and audit history.

**Design**:
- State machine (`transaction_status`): `inquiry → booked → awaiting_payment → payment_held → in_progress → delivered → completed`; branches `→ disputed → refunded`, and `→ cancelled`. Every transition writes `transaction_status_history`.
- `POST /v1/transactions` `{ listingId, quantity }` → computes `subtotal`, `commission_amount = round(subtotal * marketplace.commission_rate)`, `total_amount`; creates Stripe PaymentIntent (manual capture) on the platform account; status `awaiting_payment`.
- `POST /v1/transactions/:id/confirm-payment` (after client confirms PI) → capture → status `payment_held` (funds held on platform).
- `POST /v1/transactions/:id/complete` (buyer confirms or auto after delivery window) → schedules transfer to seller; status `completed`.
- Money handled in minor units internally (`money.ts`); rounding rule documented (banker's rounding) to keep commission + payout = total.
- Idempotency: `Idempotency-Key` header on `POST /v1/transactions` and confirm; stored to dedupe Stripe charges.

**Testing**:
- Integration (stripe-mock): create transaction → PI created with `amount` in minor units == total; commission math: $100 @ 10% → commission 1000, transfer target 9000.
- Integration: full happy path inquiry→…→completed writes one history row per transition.
- Integration: replayed `POST /v1/transactions` with same `Idempotency-Key` → same transaction, one PI.
- Integration: `complete` on a `cancelled` transaction → 409 `invalid_state_transition`.
- Unit: rounding — 3 splits of $0.10 commission never lose/gain a cent vs total.

#### 5.3 — Payout scheduling & reconciliation
**What**: Aggregate held funds into scheduled seller payouts via BullMQ delayed jobs; reconcile transfers.

**Design**:
- On transaction `completed`, create `payout_line`; a scheduled job (configurable cadence, default daily) groups unpaid lines per seller into a `payout`, calls `createTransfer`, transitions `pending→processing→completed`.
- Stripe `transfer.*` / `payout.*` webhooks update `payout.status`; failures → `failed` with `failure_reason`, retried with backoff.

**Testing**:
- Integration: two completed transactions for one seller → single payout grouping both lines; `payout.amount == sum(lines)`.
- Integration: transfer webhook `failed` → payout `failed`, line not double-counted on retry.
- Integration: payout attempted for non-eligible seller → skipped, line stays unpaid.

---

## Phase 6: Trust — Reviews, Disputes, Messaging & Notifications

### Purpose
Trust and communication primitives that make a marketplace usable and safe: post-transaction reviews, dispute handling, buyer↔seller messaging, and transactional notifications across the lifecycle. Can be developed in parallel with Phase 7 once Phase 5 is done.

### Tasks

#### 6.1 — Reviews & ratings with moderation
**What**: One review per transaction per direction, 1–5 stars, operator moderation, aggregate scores feeding seller `quality_score`.

**Design**:
- `POST /v1/transactions/:id/reviews` (only on `completed`, only by participants) `{ rating, title, body }`. `UNIQUE (transaction_id, reviewer_id)`.
- Operator `PATCH /v1/admin/reviews/:id` can set `is_visible=false` with `moderation_note`.
- Recompute `seller_profile.quality_score` as rolling average on new visible review.

**Testing**:
- Integration: review before completion → 409; after completion → 201.
- Integration: second review by same reviewer on same txn → 409 conflict.
- Integration: hide review → excluded from public listing and from quality_score recompute.

#### 6.2 — Disputes
**What**: Dispute lifecycle tied to transactions with operator resolution that can trigger refunds.

**Design**:
- `POST /v1/transactions/:id/disputes` → transaction `→ disputed`, dispute `open`.
- Operator resolution `resolved_buyer` → triggers `refund` via PaymentProvider; `resolved_seller` → releases payout. `dispute_status` machine enforced.

**Testing**:
- Integration: open dispute on `payment_held` txn → txn `disputed`; resolve buyer → Stripe refund called, txn `refunded`.
- Integration: resolve a `closed` dispute again → 409.

#### 6.3 — Messaging
**What**: Conversation threads between buyer and seller, scoped to a listing or transaction.

**Design**:
- `conversation`, `conversation_participant`, `message`. `POST /v1/conversations` (participants derived), `POST /v1/conversations/:id/messages`, `GET` with cursor pagination. `last_read_at` for unread counts.
- Authorization: only participants can read/post.

**Testing**:
- Integration: non-participant posting to a conversation → 403.
- Integration: post message → `updated_at` bumped; unread count correct for the other participant.

#### 6.4 — Notifications
**What**: Transactional email/SMS on lifecycle events behind a `Notifier` interface, sent from the worker.

**Design**:
- Events → templates: `order.created`, `payment.confirmed`, `payout.completed`, `review.requested`, `dispute.opened`. Worker subscribes to a `notify` queue; `Notifier` impls SendGrid (email) + Twilio (SMS). Inbound SendGrid event webhook (HMAC-SHA256 verified) records delivery/opens.

**Testing**:
- Integration (mocked Notifier): completing a transaction enqueues `review.requested` once.
- Unit: inbound SendGrid webhook with bad HMAC → rejected, not recorded.

---

## Phase 7: Search & Discovery (incl. Semantic Search)

### Purpose
The demand side. Text + faceted + geo search for parity with incumbents, plus the AI-native embedding-based semantic search and personalised feed. Can parallel Phase 6.

### Tasks

#### 7.1 — Text, facet & geo search
**What**: `GET /v1/search` over active listings with full-text (pg_trgm), category facet, price range, and radius geo filter.

**Design**:
- Query params: `q, categoryId, type, priceMin, priceMax, lat, lng, radiusKm, sort(relevance|price_asc|price_desc|newest), cursor, limit`.
- Trigram similarity on `title`/`description`; `attributes @>` for JSONB facet filters; geo via `earthdistance`/`point` distance. Tenant-scoped, `status='active'` only.

**Testing**:
- Integration: `q=blue bike` returns trigram-matched listings ranked by similarity.
- Integration: `priceMin/priceMax` bounds respected; `radiusKm` excludes out-of-range listings.
- Integration: facet `attributes.brand=Trek` filters correctly via GIN.

#### 7.2 — Embedding pipeline & semantic search
**What**: Generate listing embeddings on write (worker) and a semantic search endpoint using cosine distance.

**Design**:
- Worker `embeddings` processor: on listing create/update, call `LlmProvider.embed(title + description + key attributes)` → write `listing.embedding`. HNSW index for ANN.
- `GET /v1/search/semantic?q=...` → embed query, `ORDER BY embedding <=> $1 LIMIT k`, blended with trust score (`quality_score`) as a tie-break.

**Testing**:
- Integration (mocked LlmProvider, deterministic vectors): publishing a listing populates `embedding`.
- Integration: semantic query closest to a seeded listing returns it first.
- Integration: listing with null embedding (job pending) excluded from semantic results, still in text search.

#### 7.3 — Personalised buyer feed
**What**: Recommendation feed from `browse_event` history + embedding similarity.

**Design**:
- Record `browse_event` (view/wishlist/cart_add) on listing GET. `GET /v1/feed` builds a buyer-intent vector (mean of recently viewed listing embeddings), returns nearest active listings excluding already-seen, blended with recency.

**Testing**:
- Integration: buyer who viewed several bike listings gets bike-similar feed; cold-start buyer gets popularity-ranked fallback.
- Integration: feed excludes the buyer's own seller listings and out-of-stock items.

---

## Phase 8: AI-Native Operations — Onboarding, Fraud Scoring, Listing Optimiser

### Purpose
The differentiating AI layer that removes manual bottlenecks: automated seller document review + category classification, real-time fraud/trust scoring, and the listing quality optimiser. Requires Phases 3 and 5. GDPR Article 22 note: every automated decision is advisory with an operator override path.

### Tasks

#### 8.1 — AI seller onboarding (document review + category classification)
**What**: Worker reviews uploaded KYC documents and proposes seller category, with operator override.

**Design**:
- `KycProvider` (Persona) handles identity; an `onboarding-review` worker job sends document metadata + extracted fields to `LlmProvider` for consistency checks (name match, expiry, document type) → writes `seller_verification_document.ai_review_result` JSONB `{ decision, confidence, reasons[] }` and proposes `seller_profile` category.
- High-confidence pass auto-sets `verification_status='verified'`; low confidence → `in_review` queue for operators. Threshold in marketplace config.
- Prompt template (system): "You are a KYC reviewer. Given extracted document fields and the seller's claimed identity, return strict JSON {decision: 'pass'|'refer'|'fail', confidence: 0..1, reasons: string[]}. Refer if any field is inconsistent or low quality."

**Testing**:
- Integration (mocked LlmProvider): consistent docs + high confidence → auto-verified.
- Integration: name mismatch → `refer` → status `in_review`, surfaced in operator queue.
- Integration: operator override sets `verified` and records `human_review_note`.

#### 8.2 — Real-time fraud & trust scoring
**What**: Score transactions at creation using behavioural signals; gate or flag high-risk transactions.

**Design**:
- `FraudScorer` aggregates signals → `fraud_signal` rows (velocity, geo_mismatch, new_account, price_anomaly) and a composite `transaction.fraud_score` (0–1). On `POST /v1/transactions`, score synchronously (fast heuristics) + enqueue async LLM/behavioural deepening.
- Score ≥ high threshold → transaction held in `inquiry` pending operator review instead of `awaiting_payment`; logged with reasons.

**Testing**:
- Integration: 5 transactions in 60s from one buyer → velocity signal raises score above flag threshold.
- Integration: score above hard threshold → transaction not advanced to payment; operator can release.
- Unit: composite score is monotonic in each signal weight.

#### 8.3 — AI listing optimiser
**What**: Suggest improved title/description/price for a listing based on marketplace conversion data.

**Design**:
- `POST /v1/listings/:id/optimise` (seller) → `LlmProvider` given current listing + category benchmarks (median price, top-converting title patterns) returns `{ suggestedTitle, suggestedDescription, suggestedPrice, rationale }`. Stored as a suggestion; seller applies explicitly (never auto-applied). Also writes `ai_quality_score`.

**Testing**:
- Integration (mocked LlmProvider): returns structured suggestion; applying it updates listing and re-enqueues embedding.
- Integration: optimise on another seller's listing → 403.
- Unit: suggested price clamped to a sane band around category median (no $0 / absurd values).

---

## Phase 9: Operator Console API & White-Label Configuration

### Purpose
The operator-facing control plane: vendor approval, listing moderation, commission configuration, dispute backlog, GMV dashboard data, and white-label branding/domain config. Backs the admin section of the storefront and any custom operator UI.

### Tasks

#### 9.1 — Operator admin endpoints
**What**: Approval queues, moderation actions, and marketplace settings.

**Design**:
- `GET /v1/admin/sellers?status=in_review`, `POST /v1/admin/sellers/:id/approve|reject`.
- `GET /v1/admin/listings?status=pending_review`, moderation actions.
- `PATCH /v1/admin/marketplace` updates `commission_rate`, `config` (branding JSON: `logoUrl`, `colours`, enabled verticals, moderation toggle), `custom_domain`.
- `GET /v1/admin/metrics` → GMV, active sellers, dispute backlog, payout pending totals (aggregations, optionally from a read replica).

**Testing**:
- Integration: approve seller in `in_review` → `verified`; their pending listings become eligible.
- Integration: non-operator hitting any `/v1/admin/*` → 403.
- Integration: metrics GMV equals sum of completed transaction totals for the tenant.

#### 9.2 — White-label branding & custom domain
**What**: Per-marketplace branding served to the storefront and domain-based tenant resolution.

**Design**:
- `config.branding` consumed by storefront theme; custom domain mapped to `marketplace.custom_domain` (resolved in 2.3). Domain verification record stored.

**Testing**:
- Integration: request via mapped custom domain resolves to correct tenant.
- Integration: `GET /v1/marketplace/branding` (public) returns the tenant's branding.

---

## Phase 10: Default Storefront, MCP Server & Release Hardening

### Purpose
Ship the optional default storefront (proving the headless API is complete), expose marketplace resources to AI agents via MCP, and complete deployment hardening (GDPR data-subject endpoints, rate limiting, Docker images, CI). After this phase the product is launchable self-hosted.

### Tasks

#### 10.1 — Default Next.js storefront
**What**: Responsive storefront consuming only the SDK: browse/search, listing detail, seller dashboard, checkout, buyer feed, operator admin.

**Design**: Next.js 16 App Router + shadcn/ui; Server Components for listing/search SEO; Stripe Elements for checkout; reads branding from `/v1/marketplace/branding`. No direct DB access — API only.

**Testing**:
- E2E (Playwright): buyer searches → opens listing → checks out (stripe test mode) → sees confirmation.
- E2E: seller logs in → creates listing → publishes → it appears in search.
- E2E: operator approves a pending seller from the admin UI.

#### 10.2 — MCP server
**What**: MCP server exposing read tools over listings, transactions, and analytics for AI operator agents.

**Design**: Tools `list_listings`, `get_transaction`, `marketplace_metrics`, `search_listings`. Auth via client-credentials token with `mcp:read` scope; tenant-scoped. Read-only in v1.

**Testing**:
- Integration: MCP `search_listings` tool returns the same results as `/v1/search`.
- Integration: MCP call without `mcp:read` scope → rejected.

#### 10.3 — GDPR, rate limiting & deployment
**What**: Data-subject export/erasure, per-client rate limits, Docker images, and CI gates.

**Design**:
- `POST /v1/users/me/export` → packaged JSON of the user's personal data; `DELETE /v1/users/me` → erasure (anonymise PII, retain financial records as required, cascade per FKs).
- `@fastify/rate-limit` keyed by client/token; defaults configurable per marketplace.
- `Dockerfile.api`/`Dockerfile.worker` multistage; `docker compose up` runs the full stack; CI runs lint, typecheck, unit, integration (Testcontainers), OpenAPI drift check, and Docker build.

**Testing**:
- Integration: export returns the requesting user's data only (tenant + ownership scoped).
- Integration: erasure anonymises `user` PII but preserves `transaction` financial rows (referential integrity intact).
- Integration: exceeding rate limit → 429 `rate_limited` with `Retry-After`.
- CI: `docker build` for api and worker succeeds; compose stack health-checks pass.

---

## Phase Summary & Dependencies

```
Phase 1: Foundation (DB, config, errors)        ─── required by everything
    │
Phase 2: Identity, Auth & Tenancy               ─── requires 1
    │
Phase 3: Catalogue (categories, listings, media)─── requires 2
    │
Phase 4: OpenAPI 3.1 & SDK                       ─── requires 2,3 (hardens contract)
    │
Phase 5: Transactions, Stripe Connect, Payouts  ─── requires 3
    ├── Phase 6: Trust (reviews/disputes/msg/notify) ── requires 5 ┐ can run
    └── Phase 7: Search & Discovery                  ── requires 3 ┘ in parallel
         │
Phase 8: AI Ops (onboarding, fraud, optimiser)  ─── requires 3,5 (and 7 for optimiser benchmarks)
    │
Phase 9: Operator Console & White-label         ─── requires 5,6
    │
Phase 10: Storefront, MCP, Release Hardening    ─── requires 4,9 (storefront needs full API + SDK)
```

**Parallelism opportunities**
- Phases **6** and **7** can be built concurrently once Phase 5 (txn) and Phase 3 (catalogue) are done respectively.
- Phase **4** (OpenAPI/SDK) can be progressed incrementally alongside every route-adding phase; treat it as a continuously-maintained contract rather than a one-shot.
- Within Phase 8, the **listing optimiser (8.3)** depends on Phase 7 benchmarks; **onboarding (8.1)** and **fraud (8.2)** do not and can start after Phase 5.

---

## Definition of Done (per phase)

1. All tasks in the phase implemented.
2. All unit and integration tests pass (`pnpm -w turbo run test`); integration tests run against real Postgres/Redis/MinIO via Testcontainers.
3. ESLint and Prettier pass with no errors (`pnpm -w lint`).
4. Type checking passes (`pnpm -w turbo run typecheck`, i.e. `tsc --noEmit` in every package).
5. `docker compose up` brings up the full stack and health checks pass (where the phase touches a service).
6. The phase's feature works end-to-end against the running API (verified by an integration or Playwright e2e test).
7. New configuration keys added to `.env.example` and documented.
8. New/changed API endpoints appear in the generated `openapi.json` (3.1.0) and the committed snapshot is updated; SDK regenerated.
9. Database changes shipped as a reviewed `drizzle-kit` migration that applies cleanly from an empty DB and is reversible or has a documented forward-only note.
10. Money-touching changes (Phases 5, 6.2) include a rounding/idempotency test proving commission + payout = total to the cent.
