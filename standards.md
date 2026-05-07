# Standards & API Reference

> Project: Two-Sided Marketplace Builder · Generated: 2026-05-07

## Industry Standards & Specifications

### ISO Standards

- **ISO 20022** — Global financial messaging standard for cross-border payments. Since November 2025, ISO 20022 has become the exclusive standard for cross-border interbank payments, replacing legacy MT formats. Relevant to marketplace payout rails where sellers receive funds across borders; the standard uses structured XML/MX messages and supports JSON syntax. Official resource: https://www.swift.com/standards/iso-20022

### W3C & IETF Standards

- **W3C Payment Request API** — A W3C web standard that replaces traditional checkout forms with a browser-mediated payment dialog. The related Web-based Payment Handler API (updated January 2026) defines how browser-registered payment handlers respond to payment requests. Relevant to marketplace checkout UX; major payment processors including Adyen implement it. Spec: https://www.w3.org/TR/web-based-payment-handler/

- **RFC 9110 — HTTP Semantics** (IETF, June 2022) — The definitive specification for HTTP request methods, status codes, and header semantics across HTTP/1.1, HTTP/2, and HTTP/3. Applies to all REST API design in the marketplace platform. Reference: https://datatracker.ietf.org/doc/html/rfc9110

- **RFC 9205 — Building Protocols with HTTP** (IETF) — Guidance for designing application-level protocols and APIs over HTTP, covering extensibility, versioning, and interoperability. Relevant to the design of marketplace REST API endpoints and webhook delivery. Reference: https://httpwg.org/specs/rfc9205.html

- **OpenID Federation 1.0** — Achieved Final status in early 2026; defines how multilateral identity federations are built using OpenID Connect and OAuth 2.0. Relevant when a marketplace operator wants to federate seller or buyer identity across partner platforms or B2B ecosystems. Spec: https://openid.net/specs/openid-federation-1_0.html

### Data Model & API Specifications

- **OpenAPI Specification 3.1** — The industry-standard format for describing REST APIs with full JSON Schema 2020-12 compatibility. The marketplace platform's public API surface (operator API, seller API, buyer API) should be described in OAS 3.1 to support auto-generated SDKs, documentation, and testing tools. In 2026, OAS 3.2.0 has also been published. Spec: https://spec.openapis.org/oas/v3.1.0.html · https://spec.openapis.org/oas/v3.2.0.html

- **JSON Schema (2020-12)** — The companion specification for validating and documenting JSON data structures. Directly relevant to defining marketplace listing schemas, transaction event payloads, and webhook notification bodies. Reference: https://json-schema.org/

- **GraphQL (June 2018 Specification)** — Query language and runtime for APIs. Used by Marketplacer and increasingly by headless commerce platforms for flexible, client-driven data access. Relevant where the marketplace needs to expose a flexible data API to multiple frontends (web, mobile, third-party integrations). Reference: https://spec.graphql.org/

### Security & Authentication Standards

- **OAuth 2.0 (RFC 6749)** — The foundational authorization framework used for granting third-party applications delegated access to marketplace resources. Required for any integration API allowing external tools or operators to access marketplace data on behalf of sellers or buyers. Reference: https://datatracker.ietf.org/doc/html/rfc6749

- **OpenID Connect 1.0** — Identity layer on top of OAuth 2.0; provides standardised buyer and seller authentication, SSO, and session management. Billions of users and millions of applications rely on it. Reference: https://openid.net/developers/how-connect-works/

- **OpenID for Verifiable Credentials (OpenID4VC)** — Emerging standard suite including OpenID4VP 1.0 and OpenID4VCI 1.0, with self-certification launched February 2026. Relevant to decentralised seller identity verification and portable KYC credentials. Reference: https://openid.net/developers/specs/

- **PCI DSS v4.0.1** — The Payment Card Industry Data Security Standard, now fully in effect as of March 2025 with all 64 new or updated requirements mandatory. Requires marketplace operators handling card data to implement script management for payment pages (Requirement 6.4.3), continuous compliance monitoring (Requirement 11.6.1), and MFA across all cardholder data environment access. Stripe Connect and Adyen's hosted payment flows significantly reduce the PCI scope for marketplace operators. Reference: https://www.pcisecuritystandards.org/

- **GDPR (EU 2016/679)** — Marketplace operators are considered data controllers for personal data in seller listings and buyer accounts. Requires lawful basis for processing, data minimisation, right to erasure, and Article 32 security controls (encryption, resilience, regular testing). As of 2026, enforcement around automated decision-making (Article 22) is intensifying, relevant to AI-powered fraud scoring and recommendation features. Reference: https://gdpr-info.eu/

- **PSD2 / Open Banking (EU PSD2 Directive)** — Enables alternative payout rails for European marketplace deployments through bank-to-bank transfers via TPP APIs. The Berlin Group NextGenPSD2 API is the most widely implemented PSD2-compliant API standard. PSD3/PSR is expected to enter formal adoption in mid-2026, beginning an 18-month transition. Relevant to European seller payouts and buyer payment methods. Reference: https://www.openbankingtracker.com/guides/open-banking-api-standards

### MCP Server Specifications

- **Model Context Protocol (MCP)** — Anthropic's open protocol for connecting AI models to external data sources and tools. Relevant if the marketplace builder exposes its operator console, listing database, or analytics to AI agents for automated marketplace management, seller onboarding, or inventory optimisation. Building MCP server endpoints for core marketplace resources (listings, transactions, users) would enable AI-native automation. Reference: https://modelcontextprotocol.io/

---

## Similar Products — Developer Documentation & APIs

### Stripe Connect

- **Description:** The de facto standard for split payments and marketplace payouts. Processes split transactions between a platform and connected seller accounts, with automatic commission deduction and configurable payout schedules. Used by thousands of marketplaces including Lyft, Shopify, and Squarespace.
- **API Documentation:** https://docs.stripe.com/connect
- **SDKs/Libraries:** JavaScript, Python, Ruby, PHP, Java, Go, .NET — https://docs.stripe.com/libraries
- **Developer Guide:** https://docs.stripe.com/connect/marketplace/tasks/accept-payment/separate-charges-and-transfers
- **Standards:** REST/JSON, OpenAPI-described endpoints, webhook events for payment lifecycle
- **Authentication:** Stripe API keys (restricted keys recommended); OAuth for connected account authorisation

---

### Adyen for Marketplaces

- **Description:** Enterprise-grade marketplace payment processing supporting split payments, seller onboarding, KYC/AML verification, payout management, and card issuing. Used by eBay, Delivery Hero, and other large-scale platforms.
- **API Documentation:** https://docs.adyen.com/marketplaces/
- **SDKs/Libraries:** Java, .NET, PHP, Python, Node.js, Ruby, Go — https://github.com/Adyen
- **Developer Guide:** https://docs.adyen.com/platforms/
- **Standards:** REST/JSON, OpenAPI 3.0 descriptions, webhook notifications
- **Authentication:** API key + HMAC signature verification for webhooks; mTLS for high-security flows

---

### Braintree Marketplace (PayPal)

- **Description:** PayPal's Braintree gateway supports marketplace-style split payments where a master merchant splits transactions with sub-merchants. Note: limited to US-domiciled participants only; not compatible with PayPal payments or recurring billing.
- **API Documentation:** https://developer.paypal.com/braintree/docs/guides/braintree-marketplace/overview
- **SDKs/Libraries:** JavaScript, iOS, Android client SDKs; server SDKs for Ruby, Python, PHP, Java, .NET, Node.js
- **Developer Guide:** https://developer.paypal.com/braintree/docs/
- **Standards:** REST/JSON; GraphQL API also available
- **Authentication:** API keys (public key, private key, merchant ID)

---

### Sharetribe Marketplace & Integration APIs

- **Description:** Two-tier API architecture: the Marketplace API for user-facing client applications (listing management, transactions, messaging), and the Integration API for trusted backend systems with full data access. JavaScript SDKs available for both.
- **API Documentation:** https://www.sharetribe.com/api-reference/marketplace.html
- **SDKs/Libraries:** JavaScript SDK (Marketplace API and Integration API) — https://www.sharetribe.com/docs/concepts/api-sdk/js-sdk/
- **Developer Guide:** https://www.sharetribe.com/docs/
- **Standards:** REST/JSON; Authentication API for session management
- **Authentication:** OAuth 2.0 PKCE for Marketplace API (client-side); API token for Integration API (server-side)

---

### Mirakl Marketplace Platform API

- **Description:** Enterprise marketplace operator and seller APIs. The Operator API manages vendor onboarding, catalogue moderation, commission configuration, and order routing. The Seller API (Shop API) provides seller-facing product listing, inventory sync, and order management. Fully described in OpenAPI 3.0.
- **API Documentation:** https://developer.mirakl.com/
- **SDKs/Libraries:** No official SDKs; OpenAPI definitions support code generation
- **Developer Guide:** https://developer.mirakl.com/content/product/mmp/rest/seller/openapi3
- **Standards:** REST/JSON, OpenAPI 3.0
- **Authentication:** Operator API key (server-to-server); shop-specific API keys for seller integrations

---

### Marketplacer (Operator & Seller APIs)

- **Description:** GraphQL-first marketplace platform with separate Operator API (for marketplace operators managing sellers, products, and orders) and Seller API (for sellers automating product listings and order processing). A legacy REST-based Seller API also exists.
- **API Documentation:** https://api.marketplacer.com/docs/
- **SDKs/Libraries:** No official SDKs; GraphQL introspection schema available
- **Developer Guide:** https://api.marketplacer.com/docs/start-here
- **Standards:** GraphQL (primary); REST (legacy Seller API); webhook events for order and product updates
- **Authentication:** API key passed as request header

---

### Commercetools Composable Commerce API

- **Description:** Headless commerce platform with a full REST API for product catalogues, orders, customers, cart, and pricing. Supports multi-vendor and marketplace patterns via its flexible product and channel model. SDKs in four languages; API described in both RAML and OAS.
- **API Documentation:** https://docs.commercetools.com/api
- **SDKs/Libraries:** Java, TypeScript, PHP, C# — https://docs.commercetools.com/sdk
- **Developer Guide:** https://docs.commercetools.com/docs
- **Standards:** REST/JSON, OpenAPI 3.0 and RAML descriptions; MCP Developer server available
- **Authentication:** OAuth 2.0 client credentials flow

---

### Amazon Selling Partner API (SP-API)

- **Description:** Amazon's REST API for marketplace sellers, providing programmatic access to listings, catalogue items, inventory, orders, and pricing. The Listings Items API and Catalog Items API define JSON Schema-based product type definitions that can inform canonical listing data models.
- **API Documentation:** https://developer-docs.amazon.com/sp-api/
- **SDKs/Libraries:** Auto-generated SDKs in multiple languages from OpenAPI definitions; C# and Java officially maintained
- **Developer Guide:** https://developer-docs.amazon.com/sp-api/docs/what-is-the-selling-partner-api
- **Standards:** REST/JSON, OpenAPI definitions, Login with Amazon (LWA) for OAuth token exchange
- **Authentication:** OAuth 2.0 (Login with Amazon); AWS Signature Version 4 for API request signing

---

### Algolia Search & Discovery API

- **Description:** Hosted search API widely used for marketplace listing search and discovery. Provides faceted search, typo tolerance, geo-filtering, AI Browse for category pages, and personalisation. Strong marketplace-specific documentation including federation search across listing types.
- **API Documentation:** https://www.algolia.com/doc/rest-api/search
- **SDKs/Libraries:** JavaScript, Python, PHP, Ruby, Go, Java, .NET, iOS, Android — https://www.algolia.com/developers/integrations
- **Developer Guide:** https://www.algolia.com/developers/search-api
- **Standards:** REST/JSON; OpenAPI description available
- **Authentication:** Application ID + API key (search-only vs. admin keys with ACL)

---

### Persona (Identity Verification API)

- **Description:** Modular identity verification platform favoured by marketplaces for its configurable KYC workflows per user segment. Supports document verification, selfie biometrics, database checks, and custom decision flows. Better suited for non-regulated marketplace KYC than Onfido/Entrust for most startup and mid-market cases.
- **API Documentation:** https://docs.withpersona.com/reference/introduction
- **SDKs/Libraries:** JavaScript (hosted flow embed); REST API for server-side integrations
- **Developer Guide:** https://docs.withpersona.com/docs
- **Standards:** REST/JSON; webhook events for verification completion and status changes
- **Authentication:** API key (Bearer token); webhook HMAC signature verification

---

### Twilio / SendGrid (Notifications & Messaging)

- **Description:** Programmable messaging and email API for transactional notifications across the marketplace lifecycle: new order, payment confirmed, payout processed, review request. SendGrid's Event Webhook delivers real-time email engagement data (opens, clicks, bounces) via HTTP POST to a configured endpoint.
- **API Documentation:** https://www.twilio.com/docs/sendgrid/for-developers · https://www.twilio.com/en-us/messaging/apis/programmable-messaging-api
- **SDKs/Libraries:** Node.js, Python, PHP, Ruby, Java, C# — https://www.twilio.com/docs/sendgrid/for-developers/sending-email/libraries
- **Developer Guide:** https://www.twilio.com/docs/sendgrid/for-developers/tracking-events/getting-started-event-webhook
- **Standards:** REST/JSON; HTTPS POST webhooks for event delivery; HMAC-SHA256 signature verification
- **Authentication:** API key (Bearer token for Twilio); SendGrid API key header

---

## Notes

**PSD3 transition**: PSD2 Open Banking APIs in Europe are entering a transition period in mid-2026 as PSD3/PSR proceeds toward formal adoption. Marketplace payout integrations targeting EU bank-to-bank transfers should plan for updated API consent and data-access requirements by late 2027.

**ISO 20022 adoption**: Since November 2025, ISO 20022 is the exclusive standard for cross-border interbank payments. Marketplace payout providers (Adyen, Stripe, Wise) are absorbing this internally; marketplace builders generally do not need to implement ISO 20022 directly, but should ensure their chosen payout provider is compliant.

**OpenAPI 3.2**: OAS 3.2.0 has been published (2026). Where greenfield API design is starting now, consider 3.2 for full JSON Schema Dialect support and improved webhook description.

**MCP for marketplace AI agents**: The Model Context Protocol is gaining adoption as the integration layer for AI agents in enterprise workflows. Exposing marketplace resources (listings, transactions, analytics) via MCP servers would enable AI-native operator tooling without requiring bespoke integrations.
