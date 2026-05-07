# Two-Sided Marketplace Builder

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> A configurable, AI-native marketplace platform that lets founders and product teams launch two-sided transaction platforms with built-in payments, trust scoring, and intelligent matching -- without enterprise pricing or vendor lock-in.

Two-Sided Marketplace Builder is an open-source platform for creating and operating two-sided marketplaces -- connecting buyers with sellers, renters with owners, or service providers with clients. It targets startup founders launching niche verticals, enterprise product teams adding marketplace channels, and agencies building white-label platforms for clients. The core problem it solves: existing tools force a choice between fast-but-limited SaaS (Sharetribe at $119/mo with constrained logic) and powerful-but-inaccessible enterprise platforms (Mirakl at six-figure contracts with months-long implementation).

---

## Why Two-Sided Marketplace Builder?

- **The mid-market gap is real.** Sharetribe gets you to launch fast but limits custom business logic and lacks B2B capabilities. Mirakl handles enterprise scale but requires a certified system integrator and enterprise-tier budgets. Nothing serves the mid-market operator who needs both flexibility and speed.
- **B2B marketplace tooling is underdeveloped.** B2B marketplaces are projected at USD 36 trillion globally by 2026, yet purpose-built B2B tooling is limited to Yo!Kart (dated UI, high integration effort) and Mirakl (enterprise-only). Purchase orders, credit terms, and organisation accounts remain niche add-ons rather than first-class features.
- **Vertical-specific workflows are missing.** Current platforms are built for generic product sales. Services marketplaces, rental platforms, and professional services exchanges all need specialised transaction workflows (escrow, multi-step approval, availability calendars) that require extensive custom development today.
- **No-code and self-hosted options both have ceilings.** Bubble offers flexibility but hits performance walls at scale. CS-Cart gives source code ownership but demands ongoing DevOps overhead and has weak C2C support. Neither provides AI-native capabilities out of the box.
- **Seller onboarding is still manual.** Across all incumbents, vendor verification and category classification require human review queues -- a bottleneck that scales linearly with marketplace growth.

---

## Key Features

### Seller and Buyer Management
- Separate buyer and seller onboarding flows with profile management
- Identity verification integration for seller onboarding (document check or business verification)
- Operator admin console with vendor approval, listing moderation, and dispute management
- White-label branding with custom domain, logo, and colour scheme

### Listing and Catalogue Management
- Product or service listings with photos, pricing, availability, and category organisation
- Configurable listing types for different marketplace verticals (products, services, rentals)
- Operator-controlled category structure with configurable attributes per category
- AI listing quality optimiser providing automated suggestions for titles, descriptions, and pricing

### Transaction Workflows
- Two-sided transaction lifecycle: inquiry or booking, approval, payment, and completion
- Stripe Connect split payment with configurable commission rates and scheduled seller payouts
- Multi-currency support with automatic conversion for cross-border transactions
- Buyer-to-seller messaging before and during transactions

### Search and Discovery
- Text search, category filtering, and location-based listing search
- Embedding-based semantic search surfacing relevant results for natural language queries
- Personalised buyer feed with recommendation engine based on purchase history and browse behaviour

### Trust and Quality
- Buyer and seller rating and review system with moderation controls
- AI-driven fraud and trust scoring for transaction risk assessment
- Vendor quality scorecard tracking on-time fulfilment, return rates, and satisfaction metrics

---

## AI-Native Advantage

The AI capabilities in this project address specific operational bottlenecks that incumbents solve with manual processes. Automated seller onboarding uses AI to review business documents, verify identity, and classify seller categories -- eliminating the human review queue that becomes a scaling bottleneck. Dynamic trust and fraud scoring assesses transaction risk in real time using behavioural signals, reducing chargebacks and policy abuse. Embedding-based semantic search matches buyer intent to listings rather than relying on keyword filters, and AI-generated listing optimisation helps sellers improve conversion rates based on marketplace-wide performance data.

---

## Tech Stack & Deployment

- **Payments:** Stripe Connect for split payments, commission handling, and seller payouts. Stripe's hosted model offloads PCI DSS scope from the marketplace operator.
- **Standards:** OAuth 2.0 / OpenID Connect for identity federation; ISO 20022 for cross-border payment messaging; PSD2-compatible Open Banking APIs for European payout rails.
- **Architecture:** Headless API-first design enabling fully custom frontends, with a default responsive storefront included.
- **Deployment:** Self-hosted and cloud options. Self-hosted deployments provide full code ownership; managed cloud option reduces DevOps overhead.

---

## Market Context

Global ecommerce platform infrastructure is projected at USD 13.92 billion in 2026, growing to USD 61.83 billion by 2034 at a CAGR of approximately 20.5% (Fortune Business Insights, 2026). The broader digital marketplace market was valued at USD 580 billion in 2024 and is forecast to exceed USD 1 trillion by 2030 (NextMSC, 2025). SaaS marketplace tools range from USD 30-120/month for indie founders up to enterprise contracts exceeding USD 100k/year, with revenue-share models of 1-2% of GMV common at the managed-service end. Primary buyers are startup founders launching niche verticals, enterprise product teams adding marketplace channels, and agencies building white-label marketplaces.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
