# Data Model Suggestion 4: Graph Database Model (Neo4j)

## Approach

A property graph model using Neo4j, where marketplace entities are nodes and their relationships (buys_from, sells_on, listed_in, reviewed_by, messages_with) are first-class edges with their own properties. This model treats the marketplace as a network of interconnected actors and resources rather than a collection of flat tables.

## Why a Graph Database Suits a Two-Sided Marketplace

A two-sided marketplace is, at its core, a network. Buyers connect to sellers through transactions. Sellers connect to categories through listings. Trust flows through review chains. Fraud patterns emerge from connection topology (ring trades, review manipulation, collusion). A graph database makes these relationships explicit and queryable in ways that are prohibitively expensive in relational systems.

Specific advantages for this domain:

- **Recommendation engine:** "Buyers who bought from this seller also bought from these sellers" is a 2-hop traversal in a graph, but requires multiple self-joins or a separate recommendation engine in SQL.
- **Trust and fraud detection:** Detecting review rings (A reviews B, B reviews C, C reviews A) is a cycle-detection query that Neo4j handles natively. In SQL, this requires recursive CTEs that become impractical beyond 3-4 hops.
- **Semantic search and matching:** Neo4j's vector index support (since version 5.11) enables embedding-based similarity search alongside graph traversals -- find listings similar to what the buyer viewed AND from sellers with high trust scores, in a single query.
- **Multi-hop queries:** "Show me all sellers who have been verified, have quality scores above 4.0, sell in the Electronics category, and have successfully completed transactions with buyers in my network" is a natural graph traversal.

The trade-off is that graph databases are not ideal for high-volume analytical aggregations (total revenue by month, average commission rate) or for strict ACID transaction guarantees on financial data. The recommended production architecture pairs Neo4j for discovery, matching, and trust analysis with a relational database (PostgreSQL) for financial transactions and payment processing.

---

## Node Definitions

```cypher
// ============================================================
// MARKETPLACE
// ============================================================
CREATE CONSTRAINT marketplace_id IF NOT EXISTS
FOR (m:Marketplace) REQUIRE m.id IS UNIQUE;

// Marketplace node
(:Marketplace {
    id: "uuid",
    name: "Vintage Marketplace",
    slug: "vintage-marketplace",
    custom_domain: "vintage.example.com",
    default_currency: "USD",
    commission_rate: 0.10,
    branding: {
        logo_url: "https://...",
        primary_color: "#3B82F6"
    },
    settings: {
        escrow_enabled: true,
        escrow_hold_days: 7,
        auto_approve_sellers: false
    },
    created_at: datetime("2026-01-15T10:00:00Z")
})

// ============================================================
// USER NODES
// ============================================================
CREATE CONSTRAINT user_id IF NOT EXISTS
FOR (u:User) REQUIRE u.id IS UNIQUE;

CREATE CONSTRAINT user_email IF NOT EXISTS
FOR (u:User) REQUIRE u.email IS UNIQUE;

// User node (can have :Buyer and/or :Seller labels)
(:User:Buyer {
    id: "uuid",
    email: "jane@example.com",
    display_name: "Jane Doe",
    avatar_url: "https://...",
    phone: "+1234567890",
    locale: "en",
    is_active: true,
    email_verified_at: datetime("2026-01-20T09:00:00Z"),
    trust_score: 4.8,
    total_purchases: 12,
    created_at: datetime("2026-01-15T10:00:00Z")
})

(:User:Seller {
    id: "uuid",
    email: "seller@acme.com",
    display_name: "Acme Supplies",
    business_name: "Acme Supplies LLC",
    business_type: "company",
    tax_id: "US123456789",
    stripe_account_id: "acct_abc123",
    verification_status: "verified",
    verified_at: datetime("2026-02-01T14:00:00Z"),
    quality_score: 4.5,
    on_time_rate: 0.97,
    return_rate: 0.02,
    total_sales: 340,
    total_revenue: 89500.00,
    created_at: datetime("2026-01-15T10:00:00Z")
})

// ============================================================
// CATEGORY NODES (hierarchical tree via relationships)
// ============================================================
CREATE CONSTRAINT category_id IF NOT EXISTS
FOR (c:Category) REQUIRE c.id IS UNIQUE;

(:Category {
    id: "uuid",
    name: "Electronics",
    slug: "electronics",
    sort_order: 1,
    is_active: true,
    attribute_schema: [
        { key: "brand", label: "Brand", type: "text", required: true },
        { key: "condition", label: "Condition", type: "select",
          options: ["new", "like_new", "good", "fair"], required: true }
    ]
})

// ============================================================
// LISTING NODES
// ============================================================
CREATE CONSTRAINT listing_id IF NOT EXISTS
FOR (l:Listing) REQUIRE l.id IS UNIQUE;

(:Listing {
    id: "uuid",
    listing_type: "product",
    title: "Vintage Canon AE-1 Camera",
    description: "Classic 35mm SLR camera in excellent condition...",
    status: "active",
    price: 249.00,
    currency: "USD",
    quantity: 1,
    location: point({ latitude: 40.7128, longitude: -74.0060 }),
    location_text: "New York, NY",
    attributes: {
        brand: "Canon",
        condition: "good",
        year: 1976,
        includes_lens: true
    },
    media: [
        { url: "https://...", alt: "Front view", type: "image" },
        { url: "https://...", alt: "Back view", type: "image" }
    ],
    ai_quality_score: 4.2,
    auto_tags: ["vintage", "camera", "film", "canon"],
    view_count: 87,
    is_featured: false,
    published_at: datetime("2026-05-20T14:00:00Z"),
    created_at: datetime("2026-05-20T13:00:00Z")
})

// ============================================================
// TRANSACTION NODES
// ============================================================
CREATE CONSTRAINT transaction_id IF NOT EXISTS
FOR (t:Transaction) REQUIRE t.id IS UNIQUE;

(:Transaction {
    id: "uuid",
    status: "completed",
    quantity: 1,
    unit_price: 249.00,
    subtotal: 249.00,
    commission_amount: 24.90,
    tax_amount: 22.41,
    total_amount: 271.41,
    currency: "USD",
    payment_intent_id: "pi_stripe_abc123",
    fraud_score: 0.08,
    escrow_held_at: datetime("2026-05-21T10:00:00Z"),
    escrow_released_at: datetime("2026-05-28T10:00:00Z"),
    completed_at: datetime("2026-05-28T10:05:00Z"),
    created_at: datetime("2026-05-21T09:30:00Z")
})

// ============================================================
// REVIEW NODES
// ============================================================
CREATE CONSTRAINT review_id IF NOT EXISTS
FOR (r:Review) REQUIRE r.id IS UNIQUE;

(:Review {
    id: "uuid",
    rating: 5,
    title: "Excellent camera, fast shipping",
    body: "Camera arrived in better condition than described...",
    dimension_ratings: {
        communication: 5,
        shipping_speed: 5,
        item_accuracy: 5,
        value: 4
    },
    is_visible: true,
    created_at: datetime("2026-05-29T11:00:00Z")
})

// ============================================================
// CONVERSATION & MESSAGE NODES
// ============================================================
CREATE CONSTRAINT conversation_id IF NOT EXISTS
FOR (c:Conversation) REQUIRE c.id IS UNIQUE;

(:Conversation {
    id: "uuid",
    created_at: datetime("2026-05-20T15:00:00Z"),
    last_message_at: datetime("2026-05-21T09:00:00Z")
})

(:Message {
    id: "uuid",
    message_type: "text",
    body: "Is this still available?",
    created_at: datetime("2026-05-20T15:00:00Z")
})

// ============================================================
// DISPUTE NODES
// ============================================================
(:Dispute {
    id: "uuid",
    status: "resolved_buyer",
    reason: "Item not as described",
    evidence: [
        { type: "image", url: "https://...", description: "Visible damage" }
    ],
    resolution: {
        outcome: "partial_refund",
        refund_amount: 50.00,
        note: "Cosmetic damage not disclosed in listing"
    },
    resolved_at: datetime("2026-06-05T14:00:00Z"),
    created_at: datetime("2026-06-01T10:00:00Z")
})

// ============================================================
// FRAUD SIGNAL NODES
// ============================================================
(:FraudSignal {
    id: "uuid",
    signal_type: "velocity_anomaly",
    score: 0.72,
    details: {
        transactions_in_window: 15,
        window: "1h",
        threshold: 10,
        ip_address: "203.0.113.42"
    },
    created_at: datetime("2026-05-29T08:00:00Z")
})

// ============================================================
// ADDRESS NODES
// ============================================================
(:Address {
    id: "uuid",
    label: "home",
    line1: "123 Main St",
    city: "New York",
    state: "NY",
    postal_code: "10001",
    country_code: "US",
    location: point({ latitude: 40.7128, longitude: -74.0060 }),
    is_default: true
})

// ============================================================
// PAYOUT NODES
// ============================================================
(:Payout {
    id: "uuid",
    amount: 1250.00,
    currency: "USD",
    status: "completed",
    stripe_transfer_id: "tr_abc123",
    scheduled_for: datetime("2026-05-26T00:00:00Z"),
    completed_at: datetime("2026-05-26T06:00:00Z"),
    created_at: datetime("2026-05-25T00:00:00Z")
})
```

---

## Relationship Definitions

```cypher
// ============================================================
// USER <-> MARKETPLACE
// ============================================================
(:User)-[:MEMBER_OF { joined_at: datetime(), role: "buyer" }]->(:Marketplace)

// ============================================================
// SELLER <-> LISTING
// ============================================================
(:User:Seller)-[:SELLS]->(:Listing)
(:Listing)-[:LISTED_ON]->(:Marketplace)

// ============================================================
// CATEGORY HIERARCHY
// ============================================================
(:Category)-[:SUBCATEGORY_OF]->(:Category)
(:Category)-[:BELONGS_TO]->(:Marketplace)
(:Listing)-[:IN_CATEGORY]->(:Category)

// ============================================================
// TRANSACTIONS (the core marketplace graph)
// ============================================================
(:User:Buyer)-[:PURCHASED { at: datetime() }]->(:Transaction)
(:Transaction)-[:FOR_LISTING]->(:Listing)
(:Transaction)-[:SOLD_BY]->(:User:Seller)
(:Transaction)-[:ON_MARKETPLACE]->(:Marketplace)

// ============================================================
// REVIEWS (trust graph)
// ============================================================
(:User)-[:WROTE_REVIEW]->(:Review)
(:Review)-[:ABOUT]->(:User)
(:Review)-[:FOR_TRANSACTION]->(:Transaction)

// ============================================================
// MESSAGING
// ============================================================
(:User)-[:PARTICIPATES_IN]->(:Conversation)
(:Conversation)-[:REGARDING]->(:Listing)
(:Conversation)-[:LINKED_TO]->(:Transaction)
(:Message)-[:IN_CONVERSATION]->(:Conversation)
(:User)-[:SENT]->(:Message)

// ============================================================
// DISPUTES
// ============================================================
(:Dispute)-[:ABOUT_TRANSACTION]->(:Transaction)
(:User)-[:OPENED_DISPUTE]->(:Dispute)

// ============================================================
// PAYOUTS
// ============================================================
(:Payout)-[:TO_SELLER]->(:User:Seller)
(:Payout)-[:INCLUDES]->(:Transaction)
(:Payout)-[:ON_MARKETPLACE]->(:Marketplace)

// ============================================================
// FRAUD SIGNALS
// ============================================================
(:FraudSignal)-[:FLAGGED]->(:User)
(:FraudSignal)-[:FLAGGED]->(:Transaction)

// ============================================================
// ADDRESSES
// ============================================================
(:User)-[:HAS_ADDRESS]->(:Address)

// ============================================================
// BEHAVIOURAL GRAPH (recommendations & fraud)
// ============================================================
(:User)-[:VIEWED { at: datetime(), source: "search" }]->(:Listing)
(:User)-[:WISHLISTED { at: datetime() }]->(:Listing)
(:User)-[:SEARCHED { query: "vintage camera", at: datetime() }]->(:Marketplace)
```

---

## Graph-Native Queries

These queries demonstrate capabilities that are natural in a graph but expensive or impractical in SQL.

```cypher
// ============================================================
// RECOMMENDATION: "Buyers who bought from this seller also bought from..."
// ============================================================
MATCH (seller:Seller {id: $sellerId})<-[:SOLD_BY]-(t1:Transaction)<-[:PURCHASED]-(buyer:Buyer)
      -[:PURCHASED]->(t2:Transaction)-[:SOLD_BY]->(other:Seller)
WHERE other.id <> $sellerId
  AND other.verification_status = 'verified'
RETURN other.id, other.business_name, other.quality_score,
       COUNT(DISTINCT buyer) AS shared_buyers
ORDER BY shared_buyers DESC
LIMIT 10

// ============================================================
// FRAUD: Detect review rings (A reviews B, B reviews C, C reviews A)
// ============================================================
MATCH cycle = (a:User)-[:WROTE_REVIEW]->(:Review)-[:ABOUT]->(b:User)
              -[:WROTE_REVIEW]->(:Review)-[:ABOUT]->(c:User)
              -[:WROTE_REVIEW]->(:Review)-[:ABOUT]->(a)
WHERE a <> b AND b <> c AND a <> c
RETURN DISTINCT a.id, b.id, c.id, a.email, b.email, c.email

// ============================================================
// TRUST: Transitive trust score (seller trusted by buyers I trust)
// ============================================================
MATCH (me:Buyer {id: $myId})-[:PURCHASED]->(t1:Transaction)-[:SOLD_BY]->(seller:Seller),
      (me)-[:PURCHASED]->(t2:Transaction)-[:SOLD_BY]->(trusted:Seller)
      <-[:SOLD_BY]-(t3:Transaction)<-[:PURCHASED]-(peer:Buyer)
      -[:PURCHASED]->(t4:Transaction)-[:SOLD_BY]->(recommended:Seller)
WHERE recommended.verification_status = 'verified'
  AND NOT (me)-[:PURCHASED]->(:Transaction)-[:SOLD_BY]->(recommended)
WITH recommended, COUNT(DISTINCT peer) AS trust_paths,
     AVG(recommended.quality_score) AS avg_quality
RETURN recommended.id, recommended.business_name, trust_paths, avg_quality
ORDER BY trust_paths DESC, avg_quality DESC
LIMIT 20

// ============================================================
// DISCOVERY: Listings similar to what I viewed, from high-quality sellers
// ============================================================
MATCH (me:User {id: $userId})-[:VIEWED]->(viewed:Listing)-[:IN_CATEGORY]->(cat:Category)
      <-[:IN_CATEGORY]-(similar:Listing)<-[:SELLS]-(seller:Seller)
WHERE similar.status = 'active'
  AND similar.id <> viewed.id
  AND seller.quality_score >= 4.0
  AND seller.verification_status = 'verified'
WITH similar, seller, COUNT(DISTINCT viewed) AS relevance
RETURN similar.id, similar.title, similar.price, similar.ai_quality_score,
       seller.business_name, seller.quality_score, relevance
ORDER BY relevance DESC, similar.ai_quality_score DESC
LIMIT 20

// ============================================================
// ANALYTICS: Marketplace network health
// ============================================================
MATCH (m:Marketplace {id: $marketplaceId})
OPTIONAL MATCH (m)<-[:MEMBER_OF]-(buyer:Buyer)
OPTIONAL MATCH (m)<-[:MEMBER_OF]-(seller:Seller)
OPTIONAL MATCH (m)<-[:ON_MARKETPLACE]-(t:Transaction {status: 'completed'})
RETURN COUNT(DISTINCT buyer) AS total_buyers,
       COUNT(DISTINCT seller) AS total_sellers,
       COUNT(DISTINCT t) AS completed_transactions

// ============================================================
// LOCATION: Listings near me from trusted sellers
// ============================================================
MATCH (listing:Listing)
WHERE listing.status = 'active'
  AND point.distance(listing.location, point({latitude: $lat, longitude: $lng})) < 50000
MATCH (listing)<-[:SELLS]-(seller:Seller)
WHERE seller.verification_status = 'verified'
  AND seller.quality_score >= 3.5
RETURN listing.id, listing.title, listing.price,
       point.distance(listing.location, point({latitude: $lat, longitude: $lng})) AS distance_m,
       seller.business_name, seller.quality_score
ORDER BY distance_m
LIMIT 30
```

---

## Indexes

```cypher
// Full-text indexes for search
CREATE FULLTEXT INDEX listing_search IF NOT EXISTS
FOR (l:Listing) ON EACH [l.title, l.description];

// Point indexes for spatial queries
CREATE POINT INDEX listing_location IF NOT EXISTS
FOR (l:Listing) ON (l.location);

CREATE POINT INDEX address_location IF NOT EXISTS
FOR (a:Address) ON (a.location);

// Range indexes for filtering
CREATE INDEX listing_status IF NOT EXISTS FOR (l:Listing) ON (l.status);
CREATE INDEX listing_price IF NOT EXISTS FOR (l:Listing) ON (l.price);
CREATE INDEX listing_type IF NOT EXISTS FOR (l:Listing) ON (l.listing_type);
CREATE INDEX transaction_status IF NOT EXISTS FOR (t:Transaction) ON (t.status);
CREATE INDEX user_email_idx IF NOT EXISTS FOR (u:User) ON (u.email);
CREATE INDEX seller_verification IF NOT EXISTS FOR (s:Seller) ON (s.verification_status);
CREATE INDEX seller_quality IF NOT EXISTS FOR (s:Seller) ON (s.quality_score);

// Vector index for semantic search (Neo4j 5.11+)
// CREATE VECTOR INDEX listing_embedding IF NOT EXISTS
// FOR (l:Listing) ON (l.embedding)
// OPTIONS { indexConfig: {
//     `vector.dimensions`: 1536,
//     `vector.similarity_function`: 'cosine'
// }};
```

---

## Recommended Production Architecture: Graph + Relational Hybrid

A pure graph database is not recommended for the full marketplace stack. The optimal architecture uses both:

| Concern | Technology | Rationale |
|---------|-----------|-----------|
| Financial transactions, payments, escrow | PostgreSQL | ACID guarantees, NUMERIC precision, Stripe webhook idempotency |
| Listing discovery, recommendations | Neo4j | Multi-hop traversals, collaborative filtering, trust propagation |
| Fraud detection, trust scoring | Neo4j | Cycle detection, network analysis, behavioural pattern matching |
| Search | Neo4j full-text + vector indexes (or Elasticsearch) | Combined semantic + graph-aware ranking |
| Analytics & reporting | PostgreSQL (or data warehouse) | Aggregations, time-series, SQL compatibility with BI tools |

Data flows from PostgreSQL (source of truth for transactions) to Neo4j via Change Data Capture (CDC) or event streaming. Neo4j serves as a read-optimized graph projection that powers discovery, recommendations, and trust features.

---

## Trade-offs

**Strengths:**
- Natural representation of marketplace relationships; queries that require 5+ joins in SQL are simple traversals.
- Recommendation and fraud detection algorithms are first-class operations, not bolted-on services.
- Schema flexibility: new node properties and relationship types can be added without migrations.
- Spatial, full-text, and vector search all supported natively in recent Neo4j versions.

**Weaknesses:**
- Not suitable as a sole database for financial transactions; ACID guarantees are per-transaction but lack the multi-statement transaction isolation of PostgreSQL.
- Aggregation queries (SUM, AVG, GROUP BY) are slower than in relational databases and lack window functions.
- Smaller talent pool: fewer engineers have production Neo4j experience compared to PostgreSQL.
- Operational complexity of running two databases (PostgreSQL + Neo4j) in production, including data synchronization.
- Licensing: Neo4j Enterprise (required for clustering and advanced indexes) requires a commercial licence.

## Scalability Considerations

- Neo4j Fabric allows sharding the graph across multiple databases -- marketplace_id can serve as the partition key.
- For read-heavy recommendation workloads, Neo4j read replicas provide horizontal scaling.
- The behavioural graph (VIEWED, WISHLISTED, SEARCHED) grows fastest; consider TTL-based pruning of events older than 90 days, archiving to a data warehouse for historical analysis.
- Hot nodes (popular sellers with thousands of relationships) can cause traversal bottlenecks; use relationship properties and WHERE clauses to limit traversal depth.

## Migration Path

Start with PostgreSQL as the primary database (Suggestion 1 or 3). Add Neo4j as a secondary read store once the marketplace reaches a scale where recommendation quality and fraud detection become differentiators (typically 10,000+ users, 50,000+ listings). Use CDC (Debezium) to stream changes from PostgreSQL to Neo4j. The graph model described here can be populated entirely from the relational schema without requiring application changes to the write path.
