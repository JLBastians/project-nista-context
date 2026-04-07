# STROBE — Product Brief

> v1 · April 2026 · Confidential

---

## Problem Statement

Fashion consumers today face a deeply fragmented and inefficient discovery experience. Despite the explosive growth of online apparel e-commerce in the United States — a $100+ billion annual market — there is no single destination where fashion-forward Americans can browse, discover, and purchase across brands and retailers in a way that reflects their personal taste.

The current landscape forces users to jump between dozens of websites, social media platforms, and apps — none of which are meaningfully personalized, and none of which communicate with each other. Finding items aligned with one's aesthetic requires 3–5 hours per week of manual browsing, and relevant products are routinely missed.

**Core Problem:** Fashion-interested US consumers aged 18–44 with household incomes of $100k+ lack a single, intelligent platform for cross-retailer discovery that learns and reflects their style preferences over time. Simultaneously, brands have no unified channel to reach this high-value consumer segment or measure engagement at scale.

**STROBE's Solution:** A discovery layer that aggregates product inventory from across the US market and learns each user's style preferences through every interaction, creating a virtuous feedback loop between consumers and brands.

---

## Goals & Objectives

**Goal 1 — Prove the User Experience & Build Proprietary Conversion Data**

Demonstrate a measurable conversion funnel from impression → detail view → add to bag → checkout initiation across a meaningful product catalogue. Accumulate sufficient funnel data to approach brands with a direct relationship proposal within 90 days of consumer launch.

**Goal 2 — Deliver Meaningful Personalization from First Use**

Achieve a personalization satisfaction score of ≥4.0/5.0 in in-app surveys within 90 days of a user's first session, driven by the taste vector recommendation engine reinforced through interactions and onboarding signals.

**Goal 3 — Build the Fashion Aggregation Catalogue**

Ingest product catalogues from a minimum of 200 Shopify-hosted brands before launch, covering menswear and womenswear across luxury, premium, and contemporary tiers.

**Goal 4 — Establish Strong User Engagement & Retention**

Achieve a 30-day user retention rate of ≥40% and a WAU/MAU ratio of ≥0.35 within 6 months of consumer launch, with feed engagement (likes, dismisses, wishlist adds) as primary retention drivers.

**Goal 5 — Establish Direct Brand Relationships**

Use proprietary conversion funnel data to negotiate direct brand partnerships (placement fees, co-marketing arrangements) with at least 5 brands within 12 months of launch.

---

## Success Metrics

| Metric | Definition | 6-Month Target | 12-Month Target |
|--------|-----------|----------------|-----------------|
| Monthly Active Users (MAU) | Unique users who open the app ≥1x per month | 50,000 | 250,000 |
| 30-Day Retention Rate | % of new users still active 30 days after first use | 35% | 45% |
| Feed CTR | % of feed items clicked relative to impressions | 8% | 15% |
| Brand Count | Active brands with ingested product catalogues | 500 | 1000 |
| Checkout Initiations | Total "Checkout with [Brand]" taps per month | 2,000 (4% of MAU) | 15,000 (6% of MAU) |
| Conversion Funnel Rate | Impression → checkout initiated | 2% | 5% |

---

## Target Personas

### Persona 1 — The Aspirational Professional

**Maya Chen, 29, New York, NY** | Finance sector | $110,000/year | Early adopter.

**Pain Points:** Spends 4+ hours per week across ASOS, Nordstrom, Revolve, and Instagram trying to curate her wardrobe. Regularly misses drops from brands she likes. Finds algorithmic social feeds too noisy. Feels her taste is not reflected by mass-market recommendations.

**Goals:** Build a cohesive, elevated wardrobe without excessive browsing. Discover new brands aligned with her aesthetic. Find specific items without visiting ten websites. Be alerted when wishlisted items drop in price.

### Persona 2 — The Style-Conscious Male Professional

**James Okafor, 36, Chicago, IL** | Tech sector (Product Manager) | $145,000/year | Expects Spotify/Netflix-level personalization.

**Pain Points:** Menswear discovery is fragmented — fewer dedicated platforms. Hard to identify when brands have new stock in his size (XL / 34 waist). Poor UX on retailer websites. No single place to build a cross-brand wishlist.

**Goals:** Efficiently update wardrobe each season. Discover niche menswear brands. Trust that his feed reflects his taste (minimal, quality-focused, earth tones). Browse and purchase with minimal friction.

---

## Feature Summary

Features are organized by priority tier. For detailed functional requirements and acceptance criteria, see `Context/specs/` and `ARCHITECTURE.md`.

### P0 — MVP

- **Product Catalogue Ingestion Pipeline:** Automated ingestion from Shopify-hosted brands via tokenless Storefront API, running every 4–6 hours via Trigger.dev. Products available within one refresh cycle. [Stock and pricing sync automatically].

- **Product Enrichment:** Ingested products are enriched to have product vector (to match similarity against products within category - e.g. used for search), style vector (to match style similarity across product categories - e.g. used for discovery), feature tagging (e.g. feminine, minimalist, etc.).

- **User Cold Start:** 7-step onboarding flow (welcome intro, age, gender expression [multi-select], sizing [gender-adaptive], city, lifestyle activities [11 occasions], brand selections [3–10 required]) to seed taste vectors and preferences.

- **User Interaction Feedback Loop:** Track explicit interactions (likes, dismisses, wishlist adds, add-to-bag) and passive signals (dwell time, detail views). Feed interactions into algorithm, accounting for factors such as session preferences vs historical, etc.

- **Personalized Discovery Feed (For You):** Feed tailored to each user's based on user interaction feedback loop. Optionally occasion-filtered (everyday, going out, work, special). Dismissed items suppressed from future sessions.

- **Unified Product Catalogue & Search:** Searchable index aggregated from Shopify-hosted brands via tokenless Storefront API. Filters by color, brand - some features should be pre-filtered for the user (e.g. gender, size and price per user's fingerprint). Beyond pre-filtering search should reflect the user's fingerprint - e.g. products are in order of likelihood that the user will like it. Semantic search via Marqo returns results within 2 seconds. Search-to-click target: ≥25%.

- **User Account & Style Profile:** Account creation via email. Users can view their inferred style profile.


### P1 — Post-MVP Feature Development

- **Shopify Cart & Checkout Handoff:** STROBE owns the pre-checkout journey (discovery, detail, size selection, bag review). Multi-brand bag groups items by brand (we build this), each with its own "Checkout with [Brand]" button that opens Shopify's hosted checkout with pre-populated cart. All cart events logged for funnel tracking. Target: ≥40% of add-to-bag events result in checkout initiation.

- **OAuth:**

- [**Improved Onboarding:** Initial swiping of product cards to seed taste vector with real interaction data.]

- **Dynamic Product Database Updates** Certain product details refresh dynamically (e.g. price + availability when a user sees the product, if it hasn't been refreshed within the past ~hour).

- **Data Capture:** Get clear on all data captured so that we can later build analytics around it. For example, we should be capturing all the events in the conversion funnel (impression, detail view, add to bag, checkout initiated) but also other events that could be useful for improving the recommendation engine (e.g. dwell time, scroll depth on product card, etc).


### P2 Pre-launch

- **Social and Sharing Features:** Wishlist sharing, product sharing via social media.

- **Customer Support:** In-app support ticket submission and handling system. FAQ and help center content covering account management, product inquiries, and troubleshooting.

- **Bug-testing and QA**: Comprehensive testing across devices and browsers

- **Conversion to Native iPhone / Android Apps:** Convert web app to completely native iOS and Android apps for improved performance and access to native features.

- **Legal Compliance:** Trademark / wordmark, privacy policy, data privacy, CCPA.


### P3 - Post-launch

- **Push Notifications & Alerts:** Opt-in notifications for price drops on wishlisted items, new arrivals matching style, and restocks. Delivery within 2 hours. Max 3 per day. Deep-links to relevant products.

- **Brand Analytics Dashboard:** Data portal for brand partners showing aggregated funnel metrics (impressions, clicks, wishlist adds, conversion data). ≤4 hour data lag. All data anonymised and aggregated.

- **Customer Data Analytics** Launch with data collection capabilities but won't have built the analytics side where we can 1) drive improvements to our recommendation engine and 2) provide insights to retailers.

- **Direct Brand Partnerships / Shopify Tokens:** Use conversion data to negotiate direct relationships with top-performing brands for placement fees and co-marketing arrangements.

### P4 — Future

Real-time data updates (e.g. product inventory), full in-app checkout with unified multi-brand cart and payment processing, our own loyalty, outfit builder, virtual try-on (AR), trend intelligence for brands, style concierge (AI chat), wardrobe tool & auto-capture, wardrobe analytics.

---

## Explicitly Out of Scope (v0)

- Own-label or white-label fashion products
- Secondhand, resale, or peer-to-peer marketplace
- Influencer content, editorial, or social feed
- B2B wholesale marketplace
- Physical retail integrations
- Subscription box or personal styling service
- International markets outside the US
- Sponsored product placements in organic feed (feed is purely algorithmic in v0)

---

## User Scenarios

### Scenario 1 — First-Time User Onboarding & Feed Seeding

Maya downloads STROBE and signs up via Apple ID. The 7-step onboarding asks age, gender expression (multi-select), sizing, city (pre-filled via IP), lifestyle activities (dinner out: often, office: often, weekends away: sometimes), and recently shopped brands (The Row, Toteme, COS, Aritzia). [P1: swiping of product cards for additional taste vector seeding is deferred post-MVP.]

Her For You feed loads within 3 seconds with products matched to her taste vector. She likes 4 items, dismisses 2, adds 1 to wishlist. Next session, the feed incorporates her signals, with occasion vectors shifted toward dinner and office.

### Scenario 2 — Cross-Brand Search & Purchase

James searches "slim fit navy blazer" — results load in 1.4 seconds via Marqo semantic search: 87 items across 14 brands. He filters by Size L, Price ≤$400, finds a blazer from a Shopify brand, selects Size L, and taps "Add to Bag."

In the bag, the blazer appears under the brand with a "Checkout with [Brand]" button. He taps it. Shopify's hosted checkout opens with the blazer pre-populated. He completes payment on Shopify.

STROBE has logged: impression, detail view, add to bag, and checkout initiated — the full conversion funnel for this brand.

### Scenario 3 — Brand Uses Conversion Data

A buyer at Brand X reviews STROBE's conversion funnel data (via dashboard or CSV). She sees that Brand X's camel trench coat generated 200 impressions, 45 detail views, 12 add-to-bag events, and 8 checkout initiations in the last 30 days — a 4% impression-to-checkout rate. She also sees that "camel outerwear" is trending +42% week-on-week. This data supports a reorder decision and opens a conversation about a direct placement arrangement with STROBE.

---

## Non-Functional Requirements

**Performance**
- Search results within 2 seconds on up to 500,000 indexed products
- Discovery feed loads within 3 seconds on standard 4G
- ≥99.5% uptime on a rolling 30-day basis
- Product data syncs every 4–6 hours [!!! Is this duplicative with the wording above? Also same point re some things syncing dynamically when a user views the product, if it hasn't been refreshed within the past ~hour]

**Security**
- All data in transit encrypted via TLS 1.2+; data at rest encrypted via AES-256
- STROBE does not store or process payment data in v0 (Shopify handles checkout)
- All third-party integrations via OAuth 2.0 or secure token exchange

**Accessibility**
- WCAG 2.1 AA at launch: screen reader support, ≥4.5:1 contrast, ≥44×44px touch targets
- All product images include descriptive alt text from product metadata

**Scalability**
- Support up to 5,000 brands and 5,000,000 indexed products
- Personalisation engine maintains ≤3 second feed load as user base grows to 500,000 MAU
- Architecture supports future international expansion without core redesign

**Platform Support**
- v0: Responsive web application (Next.js on Vercel), mobile-first design (375px viewport)
- Future: Native iOS and Android apps (approach TBD)

---

## Dependencies & Constraints

**External Dependencies**
- Shopify tokenless Storefront API availability and terms stability
- Brand catalogue availability on Shopify (non-Shopify brands cannot be ingested in v0)
- Auth provider decision (NextAuth.js or Clerk) before Phase 4
- Marqo and Qwen model availability (local for development, hosted for production)

**Technical Constraints**
- Shopify tokenless API query complexity cap of 1,000 units limits products per request to ~20–30. TBD how many requests can be made in parallel without hitting rate limits.
- Cart objects are scoped to a single Shopify store — multi-brand bag requires per-brand grouping

---

## Appendix A — Alternative Commercial Models (Future Reference)

These models are not being pursued in v0 but are documented for future consideration once conversion data demonstrates STROBE's value to brands.

**Model 1: SaaS Subscription Revenue**
Tiered subscriptions for brand partners providing access to consumer behaviour insights, trend data, and product performance analytics. Tiers could include: Basic (product-level metrics), Pro (market trend data), Enterprise (custom reports + API access). Target: $750,000 in combined subscription revenue within 12 months of commercial launch.

**Model 2: Affiliate Commission Revenue**
Commission on sales generated through STROBE, tracked via affiliate links or direct attribution. Requires either affiliate network partnerships (e.g., CJ Affiliate, ShareASale, Rakuten) or direct brand agreements. Typical fashion affiliate commission: 5–15% of sale value.

**Model 3: Hybrid (Direct Relationships + Affiliate)**
Use proprietary conversion data to negotiate direct placement fees and co-marketing arrangements with top-performing brands, while using affiliate commission as the default revenue model for the long tail of brands where direct relationships are not yet established.

---

*STROBE — Product Brief · v1 · April 2026 · Confidential*
