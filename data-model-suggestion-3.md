# Data Model Suggestion 3: Hybrid Relational + JSONB Model (PostgreSQL)

## Approach

A PostgreSQL schema that keeps core transactional entities (users, transactions, payouts) in normalized relational tables while using JSONB columns for flexible, schema-variable data (listing attributes, marketplace configuration, verification metadata, AI scoring details). This is a pragmatic middle ground: relational where integrity matters, document-oriented where flexibility matters.

## Why This Suits a Two-Sided Marketplace

The central challenge for a marketplace builder is supporting multiple verticals from a single schema. A product marketplace needs SKU, weight, and shipping dimensions. A services marketplace needs availability slots, skill tags, and hourly rates. A rental marketplace needs calendar blocks, deposit rules, and condition checklists. In a pure relational model, this variability forces either wide tables with many nullable columns or an EAV (Entity-Attribute-Value) pattern that is cumbersome to query.

JSONB columns solve this elegantly: the `listing.attributes` column stores whatever vertical-specific fields the marketplace operator has configured, indexed with GIN for efficient querying. The operator can add a "shoe_size" attribute to a fashion marketplace or a "certification_level" attribute to a professional services marketplace without any schema migration. Meanwhile, financial columns (price, commission, payout amounts) remain as proper NUMERIC types with CHECK constraints -- you never want money stored in a loosely-typed JSON field.

PostgreSQL's JSONB support is mature: it offers containment operators (`@>`), path queries (`->>`, `#>>`), GIN indexing, and full integration with the query planner.

---

## Schema Definition

```sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pgcrypto";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";

-- ============================================================
-- MARKETPLACE INSTANCE
-- ============================================================
CREATE TABLE marketplace (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    custom_domain   VARCHAR(255),

    -- Branding and UI config stored as JSONB for maximum flexibility
    branding        JSONB NOT NULL DEFAULT '{
        "logo_url": null,
        "favicon_url": null,
        "primary_color": "#3B82F6",
        "secondary_color": "#1E40AF",
        "font_family": "Inter",
        "custom_css": null
    }',

    -- Marketplace-level settings: commission, currencies, features toggles
    settings        JSONB NOT NULL DEFAULT '{
        "default_currency": "USD",
        "supported_currencies": ["USD"],
        "commission_rate": 0.10,
        "commission_type": "percentage",
        "escrow_enabled": true,
        "escrow_hold_days": 7,
        "auto_approve_sellers": false,
        "listing_requires_approval": true,
        "supported_listing_types": ["product"],
        "max_images_per_listing": 10,
        "review_moderation": "post",
        "messaging_enabled": true
    }',

    -- Payment provider config (Stripe Connect, etc.)
    payment_config  JSONB NOT NULL DEFAULT '{
        "provider": "stripe",
        "stripe_account_id": null,
        "payout_schedule": "weekly",
        "payout_day": "monday",
        "minimum_payout": 10.00
    }',

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
    role                VARCHAR(20) NOT NULL DEFAULT 'buyer'
                        CHECK (role IN ('buyer', 'seller', 'both', 'admin', 'operator')),
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    email_verified_at   TIMESTAMPTZ,

    -- Extensible profile data: avatar, phone, locale, social links, preferences
    profile             JSONB NOT NULL DEFAULT '{
        "avatar_url": null,
        "phone": null,
        "locale": "en",
        "timezone": "UTC",
        "notification_prefs": {
            "email_marketing": true,
            "email_transactional": true,
            "push_enabled": false
        }
    }',

    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (marketplace_id, email)
);

CREATE INDEX idx_user_marketplace ON "user" (marketplace_id);
CREATE INDEX idx_user_email ON "user" (email);

-- ============================================================
-- SELLER PROFILE
-- ============================================================
CREATE TABLE seller_profile (
    id                  UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id             UUID NOT NULL UNIQUE REFERENCES "user"(id) ON DELETE CASCADE,
    business_name       VARCHAR(255),
    verification_status VARCHAR(20) NOT NULL DEFAULT 'pending'
                        CHECK (verification_status IN ('pending','in_review','verified','rejected')),
    stripe_account_id   VARCHAR(255),
    verified_at         TIMESTAMPTZ,

    -- Business details vary by region and business type
    business_details    JSONB NOT NULL DEFAULT '{
        "business_type": null,
        "tax_id": null,
        "registration_number": null,
        "registered_address": null,
        "website": null,
        "description": null,
        "social_links": {}
    }',

    -- Verification documents and AI review results
    verification_data   JSONB NOT NULL DEFAULT '{
        "documents": [],
        "ai_review": null,
        "manual_review": null,
        "risk_flags": []
    }',
    -- Example verification_data.documents entry:
    -- { "type": "business_license", "url": "https://...", "uploaded_at": "...",
    --   "ai_result": { "confidence": 0.94, "extracted_fields": {...} },
    --   "status": "verified" }

    -- Quality metrics (relational for efficient aggregation)
    quality_score       NUMERIC(3,2) DEFAULT 0.00,
    on_time_rate        NUMERIC(5,4) DEFAULT 1.0000,
    return_rate         NUMERIC(5,4) DEFAULT 0.0000,
    total_sales         INTEGER NOT NULL DEFAULT 0,
    total_revenue       NUMERIC(14,2) NOT NULL DEFAULT 0.00,

    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_seller_verification ON seller_profile (verification_status);
CREATE INDEX idx_seller_business_details ON seller_profile USING gin (business_details);

-- ============================================================
-- CATEGORIES (relational tree with JSONB attribute definitions)
-- ============================================================
CREATE TABLE category (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    marketplace_id  UUID NOT NULL REFERENCES marketplace(id) ON DELETE CASCADE,
    parent_id       UUID REFERENCES category(id) ON DELETE CASCADE,
    name            VARCHAR(150) NOT NULL,
    slug            VARCHAR(150) NOT NULL,
    sort_order      INTEGER NOT NULL DEFAULT 0,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,

    -- Attribute schema defined per category as JSONB
    -- This replaces the separate category_attribute table from the normalized model
    attribute_schema JSONB NOT NULL DEFAULT '[]',
    -- Example:
    -- [
    --   { "key": "brand", "label": "Brand", "type": "text", "required": true },
    --   { "key": "size", "label": "Size", "type": "select", "options": ["S","M","L","XL"], "required": false },
    --   { "key": "weight_kg", "label": "Weight (kg)", "type": "number", "required": false },
    --   { "key": "is_organic", "label": "Organic", "type": "boolean", "required": false }
    -- ]

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (marketplace_id, slug)
);

CREATE INDEX idx_category_parent ON category (parent_id);

-- ============================================================
-- LISTINGS (core columns relational, vertical-specific data in JSONB)
-- ============================================================
CREATE TABLE listing (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    marketplace_id  UUID NOT NULL REFERENCES marketplace(id) ON DELETE CASCADE,
    seller_id       UUID NOT NULL REFERENCES seller_profile(id) ON DELETE CASCADE,
    category_id     UUID REFERENCES category(id) ON DELETE SET NULL,

    -- Core fields stay relational for indexing, sorting, filtering
    listing_type    VARCHAR(20) NOT NULL DEFAULT 'product'
                    CHECK (listing_type IN ('product', 'service', 'rental')),
    title           VARCHAR(255) NOT NULL,
    description     TEXT,
    status          VARCHAR(20) NOT NULL DEFAULT 'draft'
                    CHECK (status IN ('draft','pending_review','active','paused','sold','expired','rejected')),
    price           NUMERIC(12,2) NOT NULL,
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    quantity        INTEGER DEFAULT 1,

    -- Location as relational columns for spatial queries
    location_lat    NUMERIC(9,6),
    location_lng    NUMERIC(9,6),
    location_text   VARCHAR(255),

    -- Vertical-specific attributes stored as JSONB
    -- Validated against category.attribute_schema at the application layer
    attributes      JSONB NOT NULL DEFAULT '{}',
    -- Product example:  { "brand": "Canon", "weight_kg": 0.5, "condition": "used" }
    -- Service example:  { "hourly_rate": 150, "certifications": ["AWS","GCP"], "remote": true }
    -- Rental example:   { "daily_rate": 45, "deposit": 200, "min_days": 2 }

    -- Media stored as JSONB array (avoids join for common read pattern)
    media           JSONB NOT NULL DEFAULT '[]',
    -- [ { "url": "https://...", "alt": "Front view", "type": "image", "order": 0 },
    --   { "url": "https://...", "alt": "Demo video", "type": "video", "order": 1 } ]

    -- Availability rules for services/rentals
    availability    JSONB,
    -- { "type": "calendar",
    --   "timezone": "America/New_York",
    --   "blocked_dates": ["2026-06-15", "2026-06-16"],
    --   "recurring_availability": [
    --     { "day": "monday", "start": "09:00", "end": "17:00" },
    --     { "day": "tuesday", "start": "09:00", "end": "17:00" }
    --   ]
    -- }

    -- AI-generated metadata
    ai_metadata     JSONB DEFAULT '{}',
    -- { "quality_score": 4.2,
    --   "suggestions": ["Add more photos", "Lower price by 10%"],
    --   "auto_tags": ["vintage", "electronics", "camera"],
    --   "embedding_model": "text-embedding-3-small",
    --   "similar_listings": ["uuid1", "uuid2"] }

    -- Semantic search vector (requires pgvector)
    embedding       VECTOR(1536),

    is_featured     BOOLEAN NOT NULL DEFAULT FALSE,
    view_count      INTEGER NOT NULL DEFAULT 0,
    published_at    TIMESTAMPTZ,
    expires_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Relational indexes for common queries
CREATE INDEX idx_listing_marketplace ON listing (marketplace_id);
CREATE INDEX idx_listing_seller ON listing (seller_id);
CREATE INDEX idx_listing_category ON listing (category_id);
CREATE INDEX idx_listing_status ON listing (status) WHERE status = 'active';
CREATE INDEX idx_listing_price ON listing (price);
CREATE INDEX idx_listing_type ON listing (listing_type);
CREATE INDEX idx_listing_title_trgm ON listing USING gin (title gin_trgm_ops);

-- GIN index on JSONB attributes for flexible filtering
CREATE INDEX idx_listing_attributes ON listing USING gin (attributes jsonb_path_ops);

-- Spatial index for location-based search
CREATE INDEX idx_listing_location ON listing USING gist (
    point(location_lng, location_lat)
) WHERE location_lat IS NOT NULL;

-- ============================================================
-- TRANSACTIONS
-- ============================================================
CREATE TABLE transaction (
    id                  UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    marketplace_id      UUID NOT NULL REFERENCES marketplace(id),
    listing_id          UUID NOT NULL REFERENCES listing(id),
    buyer_id            UUID NOT NULL REFERENCES "user"(id),
    seller_id           UUID NOT NULL REFERENCES seller_profile(id),

    -- Financial columns stay strictly relational
    status              VARCHAR(30) NOT NULL DEFAULT 'inquiry'
                        CHECK (status IN (
                            'inquiry','booked','awaiting_payment','payment_held',
                            'in_progress','delivered','completed','disputed',
                            'refunded','cancelled'
                        )),
    quantity            INTEGER NOT NULL DEFAULT 1,
    unit_price          NUMERIC(12,2) NOT NULL,
    subtotal            NUMERIC(12,2) NOT NULL,
    commission_amount   NUMERIC(12,2) NOT NULL,
    tax_amount          NUMERIC(12,2) NOT NULL DEFAULT 0.00,
    total_amount        NUMERIC(12,2) NOT NULL,
    currency            CHAR(3) NOT NULL DEFAULT 'USD',

    -- Payment and fulfillment details as JSONB (varies by payment provider)
    payment_details     JSONB NOT NULL DEFAULT '{}',
    -- { "provider": "stripe",
    --   "payment_intent_id": "pi_abc123",
    --   "charge_id": "ch_xyz789",
    --   "payment_method": "card",
    --   "card_last4": "4242",
    --   "receipt_url": "https://..." }

    -- Fulfillment details vary by listing type
    fulfillment         JSONB NOT NULL DEFAULT '{}',
    -- Product: { "tracking_number": "1Z...", "carrier": "UPS", "shipped_at": "..." }
    -- Service: { "scheduled_at": "...", "duration_hours": 2, "location": "remote" }
    -- Rental: { "check_in": "2026-06-01", "check_out": "2026-06-05", "condition_report": {...} }

    -- Escrow state
    escrow_details      JSONB DEFAULT '{}',
    -- { "held_at": "...", "hold_amount": 274.90, "release_date": "...",
    --   "released_at": null, "release_triggered_by": null }

    -- Fraud assessment
    fraud_assessment    JSONB DEFAULT '{}',
    -- { "score": 0.12, "signals": [...], "model_version": "v3", "assessed_at": "..." }

    -- Status change history embedded for fast access
    status_history      JSONB NOT NULL DEFAULT '[]',
    -- [ { "from": null, "to": "inquiry", "at": "...", "by": "buyer_uuid" },
    --   { "from": "inquiry", "to": "awaiting_payment", "at": "...", "by": "system" } ]

    completed_at        TIMESTAMPTZ,
    cancelled_at        TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_transaction_marketplace ON transaction (marketplace_id);
CREATE INDEX idx_transaction_buyer ON transaction (buyer_id);
CREATE INDEX idx_transaction_seller ON transaction (seller_id);
CREATE INDEX idx_transaction_status ON transaction (status);
CREATE INDEX idx_transaction_payment ON transaction USING gin (payment_details jsonb_path_ops);

-- ============================================================
-- PAYOUTS (strictly relational -- money columns must not be JSONB)
-- ============================================================
CREATE TABLE payout (
    id                  UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    seller_id           UUID NOT NULL REFERENCES seller_profile(id),
    marketplace_id      UUID NOT NULL REFERENCES marketplace(id),
    amount              NUMERIC(12,2) NOT NULL,
    currency            CHAR(3) NOT NULL DEFAULT 'USD',
    status              VARCHAR(20) NOT NULL DEFAULT 'pending'
                        CHECK (status IN ('pending','scheduled','processing','completed','failed')),
    transaction_ids     UUID[] NOT NULL,  -- array of included transaction IDs
    scheduled_for       TIMESTAMPTZ,
    completed_at        TIMESTAMPTZ,

    -- Provider-specific payout details
    provider_details    JSONB DEFAULT '{}',
    -- { "stripe_transfer_id": "tr_abc", "stripe_payout_id": "po_xyz", "failure_code": null }

    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_payout_seller ON payout (seller_id);
CREATE INDEX idx_payout_status ON payout (status);

-- ============================================================
-- CONVERSATIONS & MESSAGES
-- ============================================================
CREATE TABLE conversation (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    marketplace_id  UUID NOT NULL REFERENCES marketplace(id),
    listing_id      UUID REFERENCES listing(id),
    transaction_id  UUID REFERENCES transaction(id),
    participants    UUID[] NOT NULL,  -- array of user IDs
    metadata        JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_conversation_participants ON conversation USING gin (participants);

CREATE TABLE message (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    conversation_id UUID NOT NULL REFERENCES conversation(id) ON DELETE CASCADE,
    sender_id       UUID NOT NULL REFERENCES "user"(id),
    message_type    VARCHAR(20) NOT NULL DEFAULT 'text'
                    CHECK (message_type IN ('text','image','system','transaction_update')),
    body            TEXT NOT NULL,
    attachments     JSONB DEFAULT '[]',
    -- [ { "url": "https://...", "filename": "receipt.pdf", "size_bytes": 42000 } ]
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_message_conversation ON message (conversation_id, created_at);

-- ============================================================
-- REVIEWS
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

    -- Structured rating dimensions (vary by marketplace vertical)
    dimension_ratings JSONB DEFAULT '{}',
    -- { "communication": 5, "shipping_speed": 4, "item_accuracy": 5, "value": 4 }

    moderation       JSONB DEFAULT '{}',
    -- { "flagged": false, "reason": null, "moderated_by": null, "moderated_at": null }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_review_reviewee ON review (reviewee_id);

-- ============================================================
-- DISPUTES
-- ============================================================
CREATE TABLE dispute (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    transaction_id  UUID NOT NULL REFERENCES transaction(id),
    opened_by       UUID NOT NULL REFERENCES "user"(id),
    status          VARCHAR(20) NOT NULL DEFAULT 'open'
                    CHECK (status IN ('open','under_review','resolved_buyer','resolved_seller','escalated','closed')),
    reason          TEXT NOT NULL,
    evidence        JSONB DEFAULT '[]',
    -- [ { "type": "image", "url": "https://...", "description": "Damaged packaging", "submitted_by": "uuid" } ]

    resolution      JSONB DEFAULT '{}',
    -- { "outcome": "refund_buyer", "refund_amount": 274.90, "note": "...", "resolved_by": "uuid" }

    resolved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_dispute_transaction ON dispute (transaction_id);
CREATE INDEX idx_dispute_status ON dispute (status);

-- ============================================================
-- BROWSE & RECOMMENDATION EVENTS
-- ============================================================
CREATE TABLE browse_event (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id         UUID NOT NULL REFERENCES "user"(id),
    listing_id      UUID NOT NULL REFERENCES listing(id),
    event_type      VARCHAR(30) NOT NULL,
    context         JSONB DEFAULT '{}',
    -- { "source": "search", "query": "vintage camera", "position": 3, "session_id": "..." }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_browse_user ON browse_event (user_id, created_at DESC);
CREATE INDEX idx_browse_listing ON browse_event (listing_id);

-- ============================================================
-- FRAUD SIGNALS
-- ============================================================
CREATE TABLE fraud_signal (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    entity_type     VARCHAR(20) NOT NULL,  -- 'user', 'transaction', 'listing'
    entity_id       UUID NOT NULL,
    signal_type     VARCHAR(50) NOT NULL,
    score           NUMERIC(5,4) NOT NULL,
    details         JSONB NOT NULL DEFAULT '{}',
    -- { "ip_address": "...", "device_fingerprint": "...", "velocity_window": "1h",
    --   "transactions_in_window": 15, "threshold": 10 }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_fraud_entity ON fraud_signal (entity_type, entity_id);
```

---

## Example JSONB Queries

```sql
-- Find all listings in a fashion marketplace with brand "Nike" and size "L"
SELECT id, title, price
FROM listing
WHERE marketplace_id = '...'
  AND status = 'active'
  AND attributes @> '{"brand": "Nike", "size": "L"}';

-- Find all service listings with hourly rate under $100
SELECT id, title, (attributes->>'hourly_rate')::numeric AS rate
FROM listing
WHERE listing_type = 'service'
  AND status = 'active'
  AND (attributes->>'hourly_rate')::numeric < 100;

-- Get marketplace settings
SELECT settings->>'default_currency' AS currency,
       (settings->>'commission_rate')::numeric AS commission,
       settings->'supported_listing_types' AS types
FROM marketplace
WHERE slug = 'vintage-cameras';

-- Find transactions with Stripe failures
SELECT id, status, payment_details->>'failure_code' AS failure
FROM transaction
WHERE payment_details ? 'failure_code'
  AND payment_details->>'failure_code' IS NOT NULL;

-- Multi-dimensional review average
SELECT reviewee_id,
       AVG(rating) AS overall,
       AVG((dimension_ratings->>'communication')::numeric) AS communication,
       AVG((dimension_ratings->>'shipping_speed')::numeric) AS shipping
FROM review
WHERE is_visible = TRUE
GROUP BY reviewee_id;
```

---

## Trade-offs

**Strengths:**
- Vertical flexibility without schema migrations: new listing attributes, review dimensions, and marketplace settings are added at runtime through JSONB.
- Financial data retains full relational integrity with CHECK constraints and proper NUMERIC types.
- Single database technology (PostgreSQL) -- no need to operate a separate document store.
- GIN indexes on JSONB columns provide efficient containment queries for attribute filtering.
- Media arrays embedded in listings eliminate the most common N+1 query (listing + images).

**Weaknesses:**
- JSONB columns bypass database-level constraint enforcement: type checking and required-field validation must happen in the application layer.
- Complex JSONB queries (deep nesting, array element filtering) can be slower than equivalent relational joins and harder to optimize.
- Schema documentation requires discipline -- without a formal schema, JSONB fields can drift into inconsistency across records.
- ORM support for JSONB varies: some ORMs treat JSONB as opaque blobs, losing type safety.

## Scalability Considerations

- GIN indexes on JSONB can become large; partial indexes (e.g., only on active listings) help control index size.
- The `browse_event` table should be partitioned by time range to prevent unbounded growth.
- JSONB columns increase row width, which can impact sequential scan performance. TOAST compression (automatic in PostgreSQL) mitigates this for large JSONB values.
- For very high-volume attribute filtering, consider materializing frequently-queried JSONB fields into generated columns (PostgreSQL 12+).

## Migration Path

This model is the most natural evolution from the normalized schema (Suggestion 1): individual tables can adopt JSONB columns one at a time. Start by converting `category_attribute` + `listing_attribute_value` into the `category.attribute_schema` + `listing.attributes` pattern. Other tables can follow as the need for flexibility becomes clear. If strict auditability is needed later, the `transaction.status_history` JSONB array can be promoted to a full event store (Suggestion 2). The hybrid approach provides a smooth on-ramp in both directions.
