# Data Model Suggestion 1: Normalized Relational Model (PostgreSQL)

## Approach

A traditional third-normal-form (3NF+) relational schema in PostgreSQL. Every entity gets its own table with proper foreign keys, check constraints, and indexes. This is the most widely understood pattern for transactional systems and provides strong data integrity guarantees out of the box.

## Why This Suits a Two-Sided Marketplace

Marketplaces are fundamentally transactional systems -- money moves between parties, listings must be consistent, and disputes require an auditable record. A normalized relational model enforces referential integrity at the database level, preventing orphaned transactions or listings that reference deleted sellers. PostgreSQL's Row-Level Security (RLS) can enforce tenant isolation when the platform is operated in multi-tenant mode (multiple marketplace instances on one database). ACID transactions ensure that payment splits, commission calculations, and escrow holds are never left in an inconsistent state.

The downside is schema rigidity: adding new listing attributes or vertical-specific fields requires ALTER TABLE migrations, which can be slow on large tables and require coordination across deployments.

---

## Schema Definition

```sql
-- ============================================================
-- EXTENSIONS
-- ============================================================
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pgcrypto";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";    -- trigram index for text search

-- ============================================================
-- ENUMS
-- ============================================================
CREATE TYPE user_role AS ENUM ('buyer', 'seller', 'both', 'admin', 'operator');
CREATE TYPE verification_status AS ENUM ('pending', 'in_review', 'verified', 'rejected');
CREATE TYPE listing_status AS ENUM ('draft', 'pending_review', 'active', 'paused', 'sold', 'expired', 'rejected');
CREATE TYPE listing_type AS ENUM ('product', 'service', 'rental');
CREATE TYPE transaction_status AS ENUM (
    'inquiry', 'booked', 'awaiting_payment', 'payment_held',
    'in_progress', 'delivered', 'completed', 'disputed',
    'refunded', 'cancelled'
);
CREATE TYPE payout_status AS ENUM ('pending', 'scheduled', 'processing', 'completed', 'failed');
CREATE TYPE dispute_status AS ENUM ('open', 'under_review', 'resolved_buyer', 'resolved_seller', 'escalated', 'closed');
CREATE TYPE message_type AS ENUM ('text', 'image', 'system', 'transaction_update');

-- ============================================================
-- MARKETPLACE INSTANCE (multi-tenant root)
-- ============================================================
CREATE TABLE marketplace (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    custom_domain   VARCHAR(255),
    logo_url        TEXT,
    colour_scheme   JSONB DEFAULT '{}',
    default_currency CHAR(3) NOT NULL DEFAULT 'USD',
    commission_rate NUMERIC(5,4) NOT NULL DEFAULT 0.1000,  -- 10%
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- USERS
-- ============================================================
CREATE TABLE "user" (
    id                  UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    marketplace_id      UUID NOT NULL REFERENCES marketplace(id) ON DELETE CASCADE,
    email               VARCHAR(320) NOT NULL,
    password_hash       TEXT NOT NULL,
    display_name        VARCHAR(150) NOT NULL,
    avatar_url          TEXT,
    role                user_role NOT NULL DEFAULT 'buyer',
    phone               VARCHAR(30),
    locale              VARCHAR(10) DEFAULT 'en',
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    email_verified_at   TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (marketplace_id, email)
);

CREATE INDEX idx_user_marketplace ON "user" (marketplace_id);
CREATE INDEX idx_user_email ON "user" (email);

-- ============================================================
-- SELLER PROFILE & VERIFICATION
-- ============================================================
CREATE TABLE seller_profile (
    id                  UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id             UUID NOT NULL UNIQUE REFERENCES "user"(id) ON DELETE CASCADE,
    business_name       VARCHAR(255),
    business_type       VARCHAR(50),  -- individual, sole_trader, company
    tax_id              VARCHAR(100),
    stripe_account_id   VARCHAR(255),
    verification_status verification_status NOT NULL DEFAULT 'pending',
    verified_at         TIMESTAMPTZ,
    quality_score       NUMERIC(3,2) DEFAULT 0.00,  -- 0.00 to 5.00
    on_time_rate        NUMERIC(5,4) DEFAULT 1.0000,
    return_rate         NUMERIC(5,4) DEFAULT 0.0000,
    total_sales         INTEGER NOT NULL DEFAULT 0,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE seller_verification_document (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    seller_id       UUID NOT NULL REFERENCES seller_profile(id) ON DELETE CASCADE,
    document_type   VARCHAR(50) NOT NULL,  -- id_card, passport, business_license, utility_bill
    file_url        TEXT NOT NULL,
    ai_review_result JSONB,
    human_review_note TEXT,
    status          verification_status NOT NULL DEFAULT 'pending',
    reviewed_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_verification_doc_seller ON seller_verification_document (seller_id);

-- ============================================================
-- BUYER PROFILE
-- ============================================================
CREATE TABLE buyer_profile (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id         UUID NOT NULL UNIQUE REFERENCES "user"(id) ON DELETE CASCADE,
    shipping_address_id UUID,  -- FK added after address table
    trust_score     NUMERIC(3,2) DEFAULT 5.00,
    total_purchases INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- ADDRESSES
-- ============================================================
CREATE TABLE address (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id         UUID NOT NULL REFERENCES "user"(id) ON DELETE CASCADE,
    label           VARCHAR(50) DEFAULT 'home',
    line1           VARCHAR(255) NOT NULL,
    line2           VARCHAR(255),
    city            VARCHAR(100) NOT NULL,
    state_province  VARCHAR(100),
    postal_code     VARCHAR(20),
    country_code    CHAR(2) NOT NULL,
    is_default      BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

ALTER TABLE buyer_profile
    ADD CONSTRAINT fk_buyer_shipping_address
    FOREIGN KEY (shipping_address_id) REFERENCES address(id) ON DELETE SET NULL;

CREATE INDEX idx_address_user ON address (user_id);

-- ============================================================
-- CATEGORIES
-- ============================================================
CREATE TABLE category (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    marketplace_id  UUID NOT NULL REFERENCES marketplace(id) ON DELETE CASCADE,
    parent_id       UUID REFERENCES category(id) ON DELETE CASCADE,
    name            VARCHAR(150) NOT NULL,
    slug            VARCHAR(150) NOT NULL,
    description     TEXT,
    sort_order      INTEGER NOT NULL DEFAULT 0,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (marketplace_id, slug)
);

CREATE INDEX idx_category_parent ON category (parent_id);

-- ============================================================
-- CATEGORY ATTRIBUTES (configurable per category)
-- ============================================================
CREATE TABLE category_attribute (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    category_id     UUID NOT NULL REFERENCES category(id) ON DELETE CASCADE,
    name            VARCHAR(100) NOT NULL,
    attribute_type  VARCHAR(30) NOT NULL,  -- text, number, boolean, select, multi_select, date
    is_required     BOOLEAN NOT NULL DEFAULT FALSE,
    options         TEXT[],  -- for select / multi_select types
    sort_order      INTEGER NOT NULL DEFAULT 0,
    UNIQUE (category_id, name)
);

-- ============================================================
-- LISTINGS
-- ============================================================
CREATE TABLE listing (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    marketplace_id  UUID NOT NULL REFERENCES marketplace(id) ON DELETE CASCADE,
    seller_id       UUID NOT NULL REFERENCES seller_profile(id) ON DELETE CASCADE,
    category_id     UUID REFERENCES category(id) ON DELETE SET NULL,
    listing_type    listing_type NOT NULL DEFAULT 'product',
    title           VARCHAR(255) NOT NULL,
    description     TEXT,
    status          listing_status NOT NULL DEFAULT 'draft',
    price           NUMERIC(12,2) NOT NULL,
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    quantity        INTEGER DEFAULT 1,
    location_lat    NUMERIC(9,6),
    location_lng    NUMERIC(9,6),
    location_text   VARCHAR(255),
    is_featured     BOOLEAN NOT NULL DEFAULT FALSE,
    view_count      INTEGER NOT NULL DEFAULT 0,
    ai_quality_score NUMERIC(3,2),
    embedding       VECTOR(1536),  -- requires pgvector extension for semantic search
    published_at    TIMESTAMPTZ,
    expires_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_listing_marketplace ON listing (marketplace_id);
CREATE INDEX idx_listing_seller ON listing (seller_id);
CREATE INDEX idx_listing_category ON listing (category_id);
CREATE INDEX idx_listing_status ON listing (status) WHERE status = 'active';
CREATE INDEX idx_listing_price ON listing (price);
CREATE INDEX idx_listing_location ON listing USING gist (
    point(location_lng, location_lat)
) WHERE location_lat IS NOT NULL;
CREATE INDEX idx_listing_title_trgm ON listing USING gin (title gin_trgm_ops);

-- ============================================================
-- LISTING IMAGES
-- ============================================================
CREATE TABLE listing_image (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    listing_id      UUID NOT NULL REFERENCES listing(id) ON DELETE CASCADE,
    url             TEXT NOT NULL,
    alt_text        VARCHAR(255),
    sort_order      INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_listing_image_listing ON listing_image (listing_id);

-- ============================================================
-- LISTING ATTRIBUTE VALUES
-- ============================================================
CREATE TABLE listing_attribute_value (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    listing_id      UUID NOT NULL REFERENCES listing(id) ON DELETE CASCADE,
    attribute_id    UUID NOT NULL REFERENCES category_attribute(id) ON DELETE CASCADE,
    value_text      TEXT,
    value_numeric   NUMERIC(15,4),
    value_boolean   BOOLEAN,
    value_date      DATE,
    UNIQUE (listing_id, attribute_id)
);

CREATE INDEX idx_listing_attr_listing ON listing_attribute_value (listing_id);

-- ============================================================
-- AVAILABILITY (for rentals / services)
-- ============================================================
CREATE TABLE listing_availability (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    listing_id      UUID NOT NULL REFERENCES listing(id) ON DELETE CASCADE,
    date_start      DATE NOT NULL,
    date_end        DATE NOT NULL,
    is_available     BOOLEAN NOT NULL DEFAULT TRUE,
    price_override  NUMERIC(12,2),
    CHECK (date_end >= date_start)
);

CREATE INDEX idx_availability_listing ON listing_availability (listing_id, date_start, date_end);

-- ============================================================
-- TRANSACTIONS
-- ============================================================
CREATE TABLE transaction (
    id                  UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    marketplace_id      UUID NOT NULL REFERENCES marketplace(id),
    listing_id          UUID NOT NULL REFERENCES listing(id),
    buyer_id            UUID NOT NULL REFERENCES buyer_profile(id),
    seller_id           UUID NOT NULL REFERENCES seller_profile(id),
    status              transaction_status NOT NULL DEFAULT 'inquiry',
    quantity            INTEGER NOT NULL DEFAULT 1,
    unit_price          NUMERIC(12,2) NOT NULL,
    subtotal            NUMERIC(12,2) NOT NULL,
    commission_amount   NUMERIC(12,2) NOT NULL,
    tax_amount          NUMERIC(12,2) NOT NULL DEFAULT 0.00,
    total_amount        NUMERIC(12,2) NOT NULL,
    currency            CHAR(3) NOT NULL DEFAULT 'USD',
    payment_intent_id   VARCHAR(255),  -- Stripe PaymentIntent
    escrow_released_at  TIMESTAMPTZ,
    completed_at        TIMESTAMPTZ,
    cancelled_at        TIMESTAMPTZ,
    cancellation_reason TEXT,
    fraud_score         NUMERIC(5,4),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_transaction_marketplace ON transaction (marketplace_id);
CREATE INDEX idx_transaction_buyer ON transaction (buyer_id);
CREATE INDEX idx_transaction_seller ON transaction (seller_id);
CREATE INDEX idx_transaction_listing ON transaction (listing_id);
CREATE INDEX idx_transaction_status ON transaction (status);

-- ============================================================
-- TRANSACTION STATUS HISTORY (audit trail)
-- ============================================================
CREATE TABLE transaction_status_history (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    transaction_id  UUID NOT NULL REFERENCES transaction(id) ON DELETE CASCADE,
    old_status      transaction_status,
    new_status      transaction_status NOT NULL,
    changed_by      UUID REFERENCES "user"(id),
    note            TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_txn_history_transaction ON transaction_status_history (transaction_id);

-- ============================================================
-- PAYOUTS
-- ============================================================
CREATE TABLE payout (
    id                  UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    seller_id           UUID NOT NULL REFERENCES seller_profile(id),
    marketplace_id      UUID NOT NULL REFERENCES marketplace(id),
    stripe_transfer_id  VARCHAR(255),
    amount              NUMERIC(12,2) NOT NULL,
    currency            CHAR(3) NOT NULL DEFAULT 'USD',
    status              payout_status NOT NULL DEFAULT 'pending',
    scheduled_for       TIMESTAMPTZ,
    completed_at        TIMESTAMPTZ,
    failure_reason      TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_payout_seller ON payout (seller_id);
CREATE INDEX idx_payout_status ON payout (status);

CREATE TABLE payout_line (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    payout_id       UUID NOT NULL REFERENCES payout(id) ON DELETE CASCADE,
    transaction_id  UUID NOT NULL REFERENCES transaction(id),
    amount          NUMERIC(12,2) NOT NULL
);

-- ============================================================
-- MESSAGING
-- ============================================================
CREATE TABLE conversation (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    marketplace_id  UUID NOT NULL REFERENCES marketplace(id),
    transaction_id  UUID REFERENCES transaction(id),
    listing_id      UUID REFERENCES listing(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE conversation_participant (
    conversation_id UUID NOT NULL REFERENCES conversation(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES "user"(id) ON DELETE CASCADE,
    last_read_at    TIMESTAMPTZ,
    PRIMARY KEY (conversation_id, user_id)
);

CREATE TABLE message (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    conversation_id UUID NOT NULL REFERENCES conversation(id) ON DELETE CASCADE,
    sender_id       UUID NOT NULL REFERENCES "user"(id),
    message_type    message_type NOT NULL DEFAULT 'text',
    body            TEXT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_message_conversation ON message (conversation_id, created_at);

-- ============================================================
-- REVIEWS & RATINGS
-- ============================================================
CREATE TABLE review (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    transaction_id  UUID NOT NULL UNIQUE REFERENCES transaction(id),
    reviewer_id     UUID NOT NULL REFERENCES "user"(id),
    reviewee_id     UUID NOT NULL REFERENCES "user"(id),
    rating          SMALLINT NOT NULL CHECK (rating BETWEEN 1 AND 5),
    title           VARCHAR(255),
    body            TEXT,
    is_visible      BOOLEAN NOT NULL DEFAULT TRUE,
    moderation_note TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_review_reviewee ON review (reviewee_id);

-- ============================================================
-- DISPUTES
-- ============================================================
CREATE TABLE dispute (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    transaction_id  UUID NOT NULL REFERENCES transaction(id),
    opened_by       UUID NOT NULL REFERENCES "user"(id),
    status          dispute_status NOT NULL DEFAULT 'open',
    reason          TEXT NOT NULL,
    resolution_note TEXT,
    resolved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_dispute_transaction ON dispute (transaction_id);

-- ============================================================
-- AI / TRUST SCORING
-- ============================================================
CREATE TABLE fraud_signal (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    transaction_id  UUID REFERENCES transaction(id),
    user_id         UUID REFERENCES "user"(id),
    signal_type     VARCHAR(50) NOT NULL,  -- velocity, geo_mismatch, device_fingerprint, etc.
    score           NUMERIC(5,4) NOT NULL,
    details         TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_fraud_signal_user ON fraud_signal (user_id);
CREATE INDEX idx_fraud_signal_transaction ON fraud_signal (transaction_id);

-- ============================================================
-- RECOMMENDATIONS / BROWSE HISTORY
-- ============================================================
CREATE TABLE browse_event (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id         UUID NOT NULL REFERENCES "user"(id),
    listing_id      UUID NOT NULL REFERENCES listing(id),
    event_type      VARCHAR(30) NOT NULL,  -- view, wishlist, cart_add
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_browse_event_user ON browse_event (user_id, created_at DESC);
CREATE INDEX idx_browse_event_listing ON browse_event (listing_id);
```

---

## Trade-offs

**Strengths:**
- Strong referential integrity prevents data corruption across buyer/seller/transaction relationships.
- Well-understood by most engineering teams; extensive tooling for migrations (Flyway, Alembic, Prisma Migrate).
- PostgreSQL RLS provides row-level tenant isolation without application-level filtering bugs.
- ACID transactions guarantee payment consistency for escrow and split-payment flows.

**Weaknesses:**
- Schema changes to the `listing` or `transaction` tables require ALTER TABLE migrations that can lock large tables.
- Configurable category attributes require the EAV pattern (`listing_attribute_value`), which is harder to query and index than fixed columns.
- Vertical-specific fields (rental duration, service booking slots) must be modeled as nullable columns or separate tables, leading to sparse rows or join-heavy queries.
- Semantic search vectors stored in the same table as listings may cause bloat; a dedicated vector index (pgvector) helps but adds operational complexity.

## Scalability Considerations

- **Read replicas** can offload search, recommendation, and reporting queries.
- **Partitioning** the `transaction` and `browse_event` tables by `created_at` (range partitioning) keeps query performance predictable as data grows.
- **Connection pooling** (PgBouncer) is essential once the marketplace exceeds a few hundred concurrent sessions.
- At very high scale (millions of listings, thousands of transactions per second), consider sharding by `marketplace_id` using Citus or moving to a distributed SQL database.

## Migration Path

This schema serves as a natural starting point. If flexibility requirements grow, individual tables can adopt JSONB columns (see Suggestion 3) without rewriting the entire schema. If auditability becomes paramount, the `transaction_status_history` table can be expanded into a full event store (see Suggestion 2). The normalized model provides a stable foundation that other approaches can evolve from incrementally.
