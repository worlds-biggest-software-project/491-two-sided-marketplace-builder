# Two-Sided Marketplace Builder — Feature & Functionality Survey

> Candidate #491 · Researched: 2026-05-02

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| Sharetribe | SaaS | Commercial (from ~$119/mo) | https://www.sharetribe.com |
| Arcadier | SaaS | Commercial (custom pricing) | https://www.arcadier.com |
| CS-Cart Multi-Vendor | Self-hosted | Commercial (from $1,499 one-time) | https://www.cs-cart.com/multivendor |
| Yo!Kart B2B | Self-hosted | Commercial (from $999 one-time) | https://www.yo-kart.com |
| Mirakl | Enterprise SaaS | Commercial (enterprise pricing) | https://www.mirakl.com |

## Feature Analysis by Solution

### Sharetribe

**Core features**
- Purpose-built SaaS platform for two-sided transaction marketplaces
- Stripe Connect integration built in: handles split payments between buyers and sellers, with configurable commission rates
- Listing management: seller-facing listing creation with photos, descriptions, pricing, and availability calendar
- Transaction workflow: booking or purchase request, approval, payment, and review lifecycle managed in the platform
- Custom domain with white-label branding: own domain, logo, and colour scheme without Sharetribe branding
- User management: separate buyer and seller onboarding flows, identity verification hook points, and profile management

**Differentiating features**
- Fastest path to a working marketplace from zero: MVP deployable in days without engineering resources
- Hosted API: Sharetribe Marketplace API allows developers to build fully custom frontends on top of Sharetribe's backend
- Bootstrapped and profitable: unlike VC-backed competitors, no acquisition or shutdown risk

**UX patterns**
- Search and browse marketplace frontend with customisable filters and map view
- Seller dashboard: listing management, order history, and payout tracking
- Review and rating system for both buyers and sellers after transaction completion

**Integration points**
- Stripe Connect for split payment processing and seller payouts
- Google Maps for location-based listing search and display
- Zapier for CRM and marketing automation connections

**Known gaps**
- Limited custom business logic without engineering work on the Marketplace API
- B2B marketplace features (bulk ordering, invoice payment terms, organisation accounts) are not native
- Performance at extreme scale (millions of listings) is less proven than enterprise platforms

**Licence / IP notes**
- Proprietary SaaS. Core marketplace logic and Stripe Connect integration are proprietary.

---

### Mirakl

**Core features**
- Enterprise marketplace platform used by large retailers and B2B operators to add marketplace channels to existing commerce infrastructure
- Operator console: vendor onboarding workflow, catalogue management, order routing, and commission configuration
- Vendor portal: self-service product listing, inventory synchronisation, order management, and dispute resolution
- Marketplace Connect: pre-built connectors to major e-commerce platforms (Salesforce Commerce, SAP Hybris, Magento) for integrating Mirakl with existing storefronts
- Payout management: automated seller commission calculation and payment disbursement

**Differentiating features**
- Only marketplace platform proven at top-200 retailer scale (Carrefour, Best Buy, Toyota, Airbus)
- B2B marketplace capabilities: purchase order payment, credit terms, and organisation-level account management
- AI-powered catalogue management: automated attribute extraction from vendor-submitted product data

**UX patterns**
- Operator dashboard with real-time GMV, active vendor count, and dispute backlog
- Vendor onboarding flow with document upload, contract signing, and category approval workflow
- Quality scorecard per vendor showing on-time shipping, return rates, and customer satisfaction metrics

**Integration points**
- Salesforce Commerce Cloud, SAP Hybris, Magento, and Commercetools via pre-built connectors
- Stripe, Adyen, and PayPal for payment processing with enterprise-grade options
- ERP systems (SAP, Oracle) for purchase order and invoice management

**Known gaps**
- Enterprise-only pricing and implementation timeline (months); inaccessible for startups or mid-market operators
- Requires an existing e-commerce storefront; Mirakl is the marketplace layer, not the full commerce platform
- High implementation cost from Mirakl-certified system integrator requirement

**Licence / IP notes**
- Proprietary SaaS. No open-source components. Has raised over USD 1 billion in funding.

---

### CS-Cart Multi-Vendor

**Core features**
- Self-hosted multi-vendor marketplace platform with full source code ownership
- Product catalogue: unlimited vendors, categories, and product variants with configurable attributes per category
- Commission engine: percentage, flat-fee, and category-specific commission structures
- Vendor panel: independent vendor storefronts with own branding, product management, and order handling
- Order management: automatic order splitting by vendor with separate fulfilment workflows per vendor
- Payment gateways: Stripe, PayPal, Authorize.net, and 80+ others via add-on modules

**Differentiating features**
- Full source code ownership: the only significant self-hosted option in this comparison with no ongoing platform fees after purchase
- White-label vendor storefronts: each vendor can have their own branded sub-store within the marketplace
- CS-Cart Marketplace for add-ons: large ecosystem of paid and free extensions for extending functionality

**UX patterns**
- Operator admin panel with vendor approval, product moderation, and commission reporting
- Vendor-specific dashboards showing orders, revenue, and product performance
- Customer-facing responsive storefront with multi-vendor cart and vendor filter

**Integration points**
- QuickBooks and accounting export via add-on modules
- Shipping providers (FedEx, UPS, DHL) via built-in and add-on integrations
- REST API for headless frontend development

**Known gaps**
- C2C marketplace features (peer-to-peer rentals, services) are limited; primarily designed for B2C product sales
- Requires server hosting, maintenance, and security patching — ongoing DevOps overhead
- Less suitable for complex marketplace transaction workflows (escrow, multi-step approval) without custom development

**Licence / IP notes**
- Commercial licence for source code; one-time purchase with optional annual support. Modifications to source code are permitted for own use under the licence terms.

---

## Cross-Cutting Feature Themes

### Table-Stakes Features
- Seller onboarding and profile management with identity verification hook points
- Product or service listing management with photos, pricing, availability, and category organisation
- Two-sided transaction flow: request, approval, payment, and post-transaction review
- Split payment processing: marketplace commission deducted automatically at checkout with seller payout
- Search and discovery with filtering, sorting, and location-based listing search
- Buyer and seller rating and review system

### Differentiating Features
- B2B marketplace features: purchase order payment, credit terms, and organisation-level account management (Mirakl, Yo!Kart B2B)
- AI-powered catalogue management: automated attribute extraction from vendor product submissions
- AI-driven fraud and trust scoring for transaction risk assessment and seller quality monitoring
- Headless API-first architecture enabling fully custom frontends on Sharetribe or Mirakl backends
- No-code marketplace builder with visual configuration of listing types, transaction workflows, and commission structures

### Underserved Areas / Opportunities
- Mid-market marketplace builder combining Sharetribe's ease of launch with Mirakl's B2B capabilities at a price point between the two
- Vertical-specific marketplace templates (services, rentals, professional services, B2B parts) with pre-configured transaction workflows and trust models
- AI-native seller onboarding: automated identity verification, document review, and category classification without manual review queues
- Dynamic trust and fraud scoring updating in real time as new transaction signals arrive

### AI-Augmentation Candidates
- Automated seller onboarding: AI reviewing business documents, verifying identity, and classifying seller categories without human review queues
- Intelligent search and matching: embedding-based semantic search surfacing the most relevant listings for each buyer's intent rather than keyword filters
- AI-generated listing optimisation: automated suggestions to sellers to improve titles, descriptions, and pricing based on conversion data
- Personalised buyer feeds: recommendation engine surfacing listings based on individual purchase history and browse behaviour

## Legal & IP Summary

Stripe Connect's split payment capabilities are available under Stripe's standard commercial API terms with no licensing restrictions for marketplace use cases. PCI DSS compliance is required for any marketplace handling card data; Stripe Connect's hosted model offloads PCI scope from the marketplace operator for most transaction flows. The ISO 20022 and PSD2 standards are open and freely implementable. Sharetribe's Marketplace API terms permit building custom frontends commercially. CS-Cart's licence permits source modification for own use but not redistribution of modified source. Mirakl's platform is entirely proprietary. The core marketplace transaction lifecycle (listing, booking, payment, review) involves no patent-protected software algorithms. The primary legal obligations for any new entrant are payment processing regulatory compliance (money transmission licences in applicable jurisdictions) and consumer protection law compliance for marketplace operators.

## Recommended Feature Scope

**Must-have (MVP)**:
- Seller and buyer onboarding with separate registration flows and profile management
- Listing management: product or service listings with photos, pricing, availability, and category
- Two-sided transaction workflow: inquiry or booking, approval, payment, and completion
- Stripe Connect split payment: commission deducted automatically at checkout with scheduled seller payouts
- Search and discovery with text search, category filter, and location filter
- Buyer and seller rating and review system with moderation controls

**Should-have (v1.1)**:
- Operator admin console: vendor approval, listing moderation, commission configuration, and dispute management
- Identity verification integration for seller onboarding (ID document check or business verification)
- White-label branding: custom domain, logo, and colour scheme without platform branding
- Messaging system: buyer-to-seller communication before and during transactions
- Multi-currency support with automatic conversion for cross-border transactions

**Nice-to-have (backlog)**:
- AI-powered seller onboarding: automated document verification and category classification
- Semantic search: embedding-based listing discovery surfacing relevant results for natural language queries
- AI listing quality optimiser: automated suggestions for improving listing titles, descriptions, and pricing
- B2B marketplace features: purchase order payment, credit terms, and organisation-level accounts
- Personalised buyer feed with recommendation engine based on purchase history and browse behaviour
