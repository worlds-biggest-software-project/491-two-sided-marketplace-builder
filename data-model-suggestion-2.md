# Data Model Suggestion 2: Event-Sourced / CQRS Model

## Approach

An event-sourcing architecture where every state change is captured as an immutable domain event in an append-only event store. The Command Query Responsibility Segregation (CQRS) pattern separates write operations (commands that produce events) from read operations (projections materialized from the event stream). The event store is the single source of truth; all read models are derived views that can be rebuilt at any time.

## Why This Suits a Two-Sided Marketplace

Marketplace transactions have complex, multi-step lifecycles: inquiry, negotiation, payment hold, fulfilment, release, dispute, refund. Each transition carries business significance and must be auditable. Event sourcing captures the full history of every transaction natively -- there is no need for a separate audit log because the event stream *is* the audit log. This is particularly valuable for:

- **Dispute resolution:** operators can replay the exact sequence of events that led to a dispute.
- **Fraud detection:** behavioural patterns emerge from event streams (velocity of purchases, unusual state transitions).
- **Commission reconciliation:** the precise moment a payment was held, released, or refunded is permanently recorded.
- **Regulatory compliance:** PSD2 and financial regulations require transaction audit trails; event sourcing provides them by construction.

The downside is increased complexity: the team must understand eventual consistency, event versioning, and projection management. Simple CRUD queries (e.g., "list all active listings") require a separate read model rather than a direct table scan.

---

## Event Store Schema

The event store uses PostgreSQL as the backing store. Events are stored in a single append-only table, partitioned by aggregate type for performance.

```sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- ============================================================
-- EVENT STORE (append-only, single source of truth)
-- ============================================================
CREATE TABLE event_store (
    event_id        UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    aggregate_type  VARCHAR(50) NOT NULL,   -- 'User', 'Listing', 'Transaction', etc.
    aggregate_id    UUID NOT NULL,
    event_type      VARCHAR(100) NOT NULL,  -- 'ListingCreated', 'PaymentHeld', etc.
    event_version   INTEGER NOT NULL,       -- per-aggregate sequence number
    payload         JSONB NOT NULL,         -- event data
    metadata        JSONB NOT NULL DEFAULT '{}',  -- correlation_id, causation_id, actor_id
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (aggregate_id, event_version)
);

CREATE INDEX idx_event_aggregate ON event_store (aggregate_id, event_version);
CREATE INDEX idx_event_type ON event_store (event_type);
CREATE INDEX idx_event_created ON event_store (created_at);

-- Partition by aggregate type for query isolation
-- (In production, use declarative partitioning or a dedicated event store like EventStoreDB)

-- ============================================================
-- SNAPSHOTS (optional, for aggregates with long event histories)
-- ============================================================
CREATE TABLE aggregate_snapshot (
    aggregate_id    UUID NOT NULL,
    aggregate_type  VARCHAR(50) NOT NULL,
    snapshot_version INTEGER NOT NULL,
    state           JSONB NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (aggregate_id, snapshot_version)
);

-- ============================================================
-- PROJECTION CHECKPOINTS (track which events each projection has consumed)
-- ============================================================
CREATE TABLE projection_checkpoint (
    projection_name VARCHAR(100) PRIMARY KEY,
    last_event_id   UUID NOT NULL REFERENCES event_store(event_id),
    last_processed  TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Domain Events

Below are the core domain events organized by aggregate. Each event is stored as a JSONB payload in the `event_store.payload` column.

### User Aggregate Events

```json
// UserRegistered
{
  "user_id": "uuid",
  "marketplace_id": "uuid",
  "email": "buyer@example.com",
  "display_name": "Jane Doe",
  "role": "buyer",
  "registered_at": "2026-05-29T10:00:00Z"
}

// SellerProfileCreated
{
  "user_id": "uuid",
  "business_name": "Acme Supplies",
  "business_type": "company",
  "tax_id": "GB123456789"
}

// SellerVerificationSubmitted
{
  "seller_id": "uuid",
  "document_type": "business_license",
  "document_url": "https://storage/docs/abc.pdf"
}

// SellerVerified
{
  "seller_id": "uuid",
  "verified_by": "ai_review",
  "confidence_score": 0.94,
  "verified_at": "2026-05-29T12:00:00Z"
}

// SellerRejected
{
  "seller_id": "uuid",
  "reason": "Document expired",
  "rejected_by": "operator_uuid"
}
```

### Listing Aggregate Events

```json
// ListingCreated
{
  "listing_id": "uuid",
  "seller_id": "uuid",
  "marketplace_id": "uuid",
  "listing_type": "product",
  "title": "Vintage Camera",
  "description": "...",
  "price": 299.00,
  "currency": "USD",
  "category_id": "uuid"
}

// ListingPublished
{
  "listing_id": "uuid",
  "published_at": "2026-05-29T14:00:00Z"
}

// ListingPriceChanged
{
  "listing_id": "uuid",
  "old_price": 299.00,
  "new_price": 249.00
}

// ListingQualityScored
{
  "listing_id": "uuid",
  "ai_quality_score": 4.2,
  "suggestions": ["Improve title clarity", "Add 2 more photos"]
}

// ListingPaused / ListingExpired / ListingSold
{
  "listing_id": "uuid",
  "reason": "seller_request"
}
```

### Transaction Aggregate Events

```json
// TransactionInitiated
{
  "transaction_id": "uuid",
  "listing_id": "uuid",
  "buyer_id": "uuid",
  "seller_id": "uuid",
  "quantity": 1,
  "unit_price": 249.00,
  "commission_rate": 0.10,
  "currency": "USD"
}

// PaymentAuthorized
{
  "transaction_id": "uuid",
  "payment_intent_id": "pi_stripe_abc",
  "amount": 274.90,
  "held_at": "2026-05-29T15:00:00Z"
}

// PaymentHeldInEscrow
{
  "transaction_id": "uuid",
  "escrow_amount": 274.90,
  "expected_release": "2026-06-05T15:00:00Z"
}

// OrderFulfilled
{
  "transaction_id": "uuid",
  "tracking_number": "1Z999AA10123456784",
  "carrier": "UPS",
  "fulfilled_at": "2026-05-30T09:00:00Z"
}

// DeliveryConfirmed
{
  "transaction_id": "uuid",
  "confirmed_by": "buyer",
  "confirmed_at": "2026-06-02T11:00:00Z"
}

// EscrowReleased
{
  "transaction_id": "uuid",
  "seller_payout": 224.91,
  "commission_collected": 24.99,
  "tax_amount": 25.00,
  "released_at": "2026-06-02T11:05:00Z"
}

// TransactionCompleted
{
  "transaction_id": "uuid",
  "completed_at": "2026-06-02T11:05:00Z"
}

// DisputeOpened
{
  "transaction_id": "uuid",
  "opened_by": "buyer_uuid",
  "reason": "Item not as described",
  "evidence_urls": ["https://..."]
}

// DisputeResolved
{
  "transaction_id": "uuid",
  "resolution": "refund_buyer",
  "refund_amount": 274.90,
  "resolved_by": "operator_uuid"
}

// FraudScoreCalculated
{
  "transaction_id": "uuid",
  "score": 0.12,
  "signals": ["normal_velocity", "known_device", "verified_buyer"]
}
```

### Messaging Events

```json
// ConversationStarted
{
  "conversation_id": "uuid",
  "participants": ["buyer_uuid", "seller_uuid"],
  "listing_id": "uuid"
}

// MessageSent
{
  "conversation_id": "uuid",
  "message_id": "uuid",
  "sender_id": "uuid",
  "body": "Is this still available?",
  "sent_at": "2026-05-29T13:00:00Z"
}
```

### Review Events

```json
// ReviewSubmitted
{
  "review_id": "uuid",
  "transaction_id": "uuid",
  "reviewer_id": "uuid",
  "reviewee_id": "uuid",
  "rating": 5,
  "title": "Excellent seller",
  "body": "Fast shipping, item as described."
}

// ReviewModerated
{
  "review_id": "uuid",
  "action": "hidden",
  "reason": "Violates community guidelines",
  "moderated_by": "operator_uuid"
}
```

---

## Command Handlers

Commands are the write side of CQRS. Each handler loads the aggregate from events, validates business rules, and emits new events.

```
Command: CreateListing
  -> Load SellerAggregate(seller_id) from events
  -> Validate: seller is verified, listing fields are valid
  -> Emit: ListingCreated

Command: InitiateTransaction
  -> Load ListingAggregate(listing_id): must be active, quantity available
  -> Load BuyerAggregate(buyer_id): must not be suspended
  -> Run fraud scoring pipeline
  -> Emit: TransactionInitiated, FraudScoreCalculated

Command: ConfirmDelivery
  -> Load TransactionAggregate(transaction_id): must be in 'delivered' state
  -> Validate: confirming user is the buyer
  -> Emit: DeliveryConfirmed, EscrowReleased, TransactionCompleted

Command: OpenDispute
  -> Load TransactionAggregate(transaction_id): must be in fulfillable state
  -> Validate: dispute window has not expired
  -> Emit: DisputeOpened
  -> Side effect: pause escrow release timer

Command: ResolveDispute
  -> Load TransactionAggregate(transaction_id): must have open dispute
  -> Validate: resolver is operator/admin
  -> Emit: DisputeResolved
  -> Conditional: RefundProcessed or EscrowReleased based on resolution
```

---

## Read Projections (Materialized Views)

Projections consume events and build query-optimized read models. Each projection is stored in its own PostgreSQL table.

```sql
-- ============================================================
-- PROJECTION: Active Listings (search & browse)
-- ============================================================
CREATE TABLE proj_active_listings (
    listing_id      UUID PRIMARY KEY,
    marketplace_id  UUID NOT NULL,
    seller_id       UUID NOT NULL,
    seller_name     VARCHAR(255),
    seller_rating   NUMERIC(3,2),
    title           VARCHAR(255) NOT NULL,
    description     TEXT,
    listing_type    VARCHAR(20),
    price           NUMERIC(12,2),
    currency        CHAR(3),
    category_id     UUID,
    category_name   VARCHAR(150),
    image_urls      TEXT[],
    location_lat    NUMERIC(9,6),
    location_lng    NUMERIC(9,6),
    quality_score   NUMERIC(3,2),
    published_at    TIMESTAMPTZ,
    updated_at      TIMESTAMPTZ
);

CREATE INDEX idx_proj_listings_marketplace ON proj_active_listings (marketplace_id);
CREATE INDEX idx_proj_listings_category ON proj_active_listings (category_id);
CREATE INDEX idx_proj_listings_price ON proj_active_listings (price);

-- ============================================================
-- PROJECTION: Seller Dashboard
-- ============================================================
CREATE TABLE proj_seller_dashboard (
    seller_id           UUID PRIMARY KEY,
    user_id             UUID NOT NULL,
    business_name       VARCHAR(255),
    verification_status VARCHAR(20),
    active_listings     INTEGER DEFAULT 0,
    total_sales         INTEGER DEFAULT 0,
    total_revenue       NUMERIC(14,2) DEFAULT 0.00,
    pending_payouts     NUMERIC(14,2) DEFAULT 0.00,
    average_rating      NUMERIC(3,2) DEFAULT 0.00,
    review_count        INTEGER DEFAULT 0,
    quality_score       NUMERIC(3,2) DEFAULT 0.00,
    on_time_rate        NUMERIC(5,4) DEFAULT 1.0000,
    updated_at          TIMESTAMPTZ
);

-- ============================================================
-- PROJECTION: Buyer Order History
-- ============================================================
CREATE TABLE proj_buyer_orders (
    transaction_id  UUID PRIMARY KEY,
    buyer_id        UUID NOT NULL,
    listing_title   VARCHAR(255),
    seller_name     VARCHAR(255),
    status          VARCHAR(30),
    total_amount    NUMERIC(12,2),
    currency        CHAR(3),
    ordered_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    review_id       UUID
);

CREATE INDEX idx_proj_buyer_orders_buyer ON proj_buyer_orders (buyer_id, ordered_at DESC);

-- ============================================================
-- PROJECTION: Operator Admin View
-- ============================================================
CREATE TABLE proj_operator_transactions (
    transaction_id      UUID PRIMARY KEY,
    marketplace_id      UUID NOT NULL,
    buyer_email         VARCHAR(320),
    seller_business     VARCHAR(255),
    listing_title       VARCHAR(255),
    status              VARCHAR(30),
    total_amount        NUMERIC(12,2),
    commission_amount   NUMERIC(12,2),
    fraud_score         NUMERIC(5,4),
    has_dispute         BOOLEAN DEFAULT FALSE,
    created_at          TIMESTAMPTZ
);

CREATE INDEX idx_proj_op_txn_marketplace ON proj_operator_transactions (marketplace_id, created_at DESC);
CREATE INDEX idx_proj_op_txn_disputes ON proj_operator_transactions (marketplace_id) WHERE has_dispute = TRUE;

-- ============================================================
-- PROJECTION: Messaging Inbox
-- ============================================================
CREATE TABLE proj_conversation_inbox (
    conversation_id UUID NOT NULL,
    user_id         UUID NOT NULL,
    other_party     VARCHAR(150),
    listing_title   VARCHAR(255),
    last_message    TEXT,
    last_message_at TIMESTAMPTZ,
    unread_count    INTEGER DEFAULT 0,
    PRIMARY KEY (conversation_id, user_id)
);

CREATE INDEX idx_proj_inbox_user ON proj_conversation_inbox (user_id, last_message_at DESC);
```

---

## Trade-offs

**Strengths:**
- Complete audit trail is built in -- every state change is permanently recorded. Critical for financial transactions and dispute resolution.
- Temporal queries are trivial: "what was the state of this transaction at 3pm on Tuesday?" is answered by replaying events up to that timestamp.
- Read models are independently optimized: the search projection can be denormalized for speed while the admin projection can include fraud scores.
- New features can be added by creating new projections from existing events without modifying the write side.
- Supports event-driven integrations: payment webhooks, notification triggers, and analytics pipelines can all subscribe to the event stream.

**Weaknesses:**
- Significant complexity overhead: eventual consistency between write and read sides, event versioning and upcasting, projection rebuild times.
- Simple queries like "show me listing X" require a pre-built projection; ad hoc queries against the event store are expensive.
- The team must be disciplined about event schema evolution -- renaming or restructuring events requires backward-compatible upcasters.
- Snapshot management adds operational burden for aggregates with thousands of events (e.g., a seller with 50,000 transactions).

## Scalability Considerations

- The event store is append-only, so write contention is minimal -- ideal for high-throughput transaction processing.
- Projections can be rebuilt in parallel and distributed across read replicas or specialized stores (Elasticsearch for search, Redis for dashboards).
- Event streams can be partitioned by marketplace_id for multi-tenant isolation.
- At extreme scale, replace the PostgreSQL event store with a purpose-built event store (EventStoreDB, Apache Kafka with compaction) while keeping the same event schemas.

## Migration Path

An event-sourced system can be introduced incrementally: start with the Transaction aggregate (where auditability has the highest value) while keeping User and Listing as traditional CRUD entities. The `transaction_status_history` table in Suggestion 1 is already a partial event log -- promoting it to a full event store is a natural evolution. Projections can initially be kept in sync via database triggers before moving to an async event processor.
