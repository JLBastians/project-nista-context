# STROBE
## Fashion Search & Discovery Engine
### Product Requirements Document · v2 · Confidential
### April 2026 · United States Launch

> **v2 update notes:** Revised to align with Architecture v5 and Master Directory v1. Commerce model updated to reflect Shopify checkout handoff and proprietary conversion data focus. Onboarding updated to match Onboarding Design Document v1. Alternative commercial models moved to Appendix A for future reference. Features section revised.

---

# 1. Problem Statement

Fashion consumers today face a deeply fragmented and inefficient discovery experience. Despite the explosive growth of online retail in the United States — a market worth over $100 billion in apparel e-commerce annually — there is no single destination where fashion-forward Americans can browse, discover, and purchase across the full spectrum of brands and retailers in a way that reflects their personal taste.

The current landscape forces users to jump between dozens of websites, social media platforms, and apps — none of which are meaningfully personalized, and none of which communicate with each other. The result is a time-intensive, frustrating process where finding items aligned with one's aesthetic requires significant manual effort, and relevant products are routinely missed.

**Core Problem:** Fashion-interested US consumers aged 18–44 with household incomes of $100k+ spend an average of 3–5 hours per week browsing multiple disconnected platforms to discover fashion items. Existing solutions lack personalization, cross-retailer aggregation, and a mechanism for users to express and refine their tastes over time. Simultaneously, brands have no unified channel to reach and understand this high-value consumer segment at scale.

STROBE solves this by acting as the definitive fashion discovery layer — a single, intelligent platform that aggregates product inventory from across the US market and learns each user's style preferences through every interaction, creating a virtuous feedback loop between consumers and brands.

**One sentence version:** STROBE is an intelligent platform that curates products from across the market and learns each user's style preferences through every interaction, creating a virtuous feedback loop between consumers and brands.

---

# 2. Goals & Objectives

**Goal 1 — Prove the User Experience & Build Proprietary Conversion Data**

Target: Demonstrate a measurable conversion funnel from impression → detail view → add to bag → checkout initiation across a meaningful product catalogue. Accumulate sufficient funnel data to approach brands with a direct relationship proposal within 90 days of consumer launch.

Measurability: Track full funnel metrics per brand. Target: 500+ checkout_initiated events across 5+ brands within the first 90 days.

**Goal 2 — Deliver Meaningful Personalization from First Use**

Target: Achieve a personalization satisfaction score of ≥4.0/5.0 in in-app surveys within 90 days of a user's first session, driven by the taste vector recommendation engine.

Measurability: Monthly in-app surveys, plus feed engagement rates (clicks, likes, wishlist adds, add-to-bag) as implicit signals.

**Goal 3 — Build the Fashion Aggregation Catalogue**

Target: Ingest product catalogues from a minimum of 50 Shopify-hosted brands within 3 months of launch, covering menswear and womenswear across luxury, premium, and contemporary tiers.

Measurability: Track total active brands, total indexed products, and category coverage monthly.

**Goal 4 — Establish Strong User Engagement & Retention**

Target: Achieve a 30-day user retention rate of ≥40% and a WAU/MAU ratio of ≥0.35 within 6 months of consumer launch.

Measurability: Retention cohort analysis tracked weekly; WAU/MAU ratio reported monthly.

**Goal 5 — Establish Direct Brand Relationships**

Target: Use proprietary conversion funnel data to negotiate direct brand partnerships (placement fees, co-marketing arrangements) with at least 5 brands within 12 months of launch.

Measurability: Number of signed brand partnerships; revenue from direct relationships.

---

# 3. Success Metrics

| Metric | Definition | 6-Month Target | 12-Month Target |
|--------|-----------|----------------|-----------------|
| Monthly Active Users (MAU) | Unique users who open the app ≥1x per month | 50,000 | 250,000 |
| 30-Day Retention Rate | % of new users still active 30 days after first use | 35% | 45% |
| Feed CTR (Click-Through Rate) | % of feed items clicked relative to impressions | 8% | 15% |
| Brand Count | Active brands with ingested product catalogues | 50 | 150 |
| Checkout Initiations | Total "Checkout with [Brand]" taps per month | 2,000 | 15,000 |
| Conversion Funnel Rate | Impression → checkout initiated | 2% | 5% |

---

# 4. Target Personas

## Persona 1 — The Aspirational Professional

**Demographics:** Maya Chen, 29, New York, NY. $110,000/year (finance sector). MS Finance, NYU Stern. Single, renting in Brooklyn.

**Technical Proficiency:** High. Daily user of Instagram, Pinterest, and TikTok. Comfortable with app-based commerce, uses Apple Pay and Afterpay regularly. Early adopter.

**Pain Points:** Spends 4+ hours per week across ASOS, Nordstrom, Revolve, and Instagram trying to curate her wardrobe. Regularly misses drops from brands she likes. Finds algorithmic social feeds too noisy. Feels her taste is not reflected by mass-market recommendations.

**Goals:** Build a cohesive, elevated wardrobe without spending excessive time browsing. Discover new brands aligned with her aesthetic. Find specific items without visiting ten websites. Track items and be alerted on sale.

## Persona 2 — The Style-Conscious Male Professional

**Demographics:** James Okafor, 36, Chicago, IL. $145,000/year (tech sector). BS Computer Science, University of Michigan. Married, homeowner.

**Technical Proficiency:** Very high. Product manager. Expects seamless UX and low-friction commerce. Expects Spotify/Netflix-level personalization from fashion.

**Pain Points:** Menswear discovery is fragmented — fewer dedicated platforms. Hard to identify when brands have new stock in his size (XL / 34 waist). Poor UX on retailer websites. No single place to build a cross-brand wishlist.

**Goals:** Efficiently update wardrobe each season. Discover niche menswear brands. Trust that his feed reflects his taste (minimal, quality-focused, earth tones). Browse and purchase with minimal friction.

---

# 5. Features & Requirements

## P0 — Must-Have (MVP)

**F-01: Unified Product Catalogue & Search**

A single, searchable index of fashion products aggregated from Shopify-hosted brands via the tokenless Storefront API, refreshed every 4–6 hours. Products are enriched with tags and vector embeddings (Qwen & Marqo) enabling both filtered browsing and natural language search.

*User Story:* "As a fashion consumer, I want to search for a specific item type, color, and price range across all brands simultaneously, so that I can find the best option without visiting multiple websites."

Acceptance Criteria: Users can execute a free-text search and receive results within 2 seconds. Search results can be filtered by gender, category, color, size, price range, and brand. Each product listing displays: image (≥1), name, brand, price (including sale price if applicable), and available sizes. Out-of-stock items are labelled as unavailable. Natural language search (powered by Marqo) returns semantically relevant results.

Success Metric: Search-to-click rate of ≥25% within 3 months of launch.

**F-02: Personalized Discovery Feed (For You)**

An AI-powered feed of fashion products tailored to each user's taste vector, built through the 6-step onboarding flow and reinforced through explicit signals (likes, dismisses, wishlist adds, add-to-bag) and passive signals (dwell time, product detail views). Taste vectors are maintained per occasion (everyday, going out, work, special).

*User Story:* "As a returning user, I want my home feed to surface products that match my aesthetic taste, so that I spend less time searching and more time discovering items I genuinely want."

Acceptance Criteria: New users complete the onboarding flow which seeds their initial taste vector. The feed updates to reflect new interactions within the next session. No two consecutive feed sessions return identical item ordering. Users can filter by occasion (everyday, going out, work, special). The feed achieves a CTR of ≥12% within 90 days per user cohort.

Success Metric: Average feed CTR ≥12% across all users by Month 3.

**F-03: Like, Dismiss & Wishlist Interactions**

Users can signal their preferences on any product by liking, dismissing, or adding to a wishlist. These interactions are primary inputs to the recommendation engine.

*User Story:* "As a user browsing my feed, I want to quickly signal which products I love or dislike, so that my future recommendations improve and I can revisit items I'm considering purchasing."

Acceptance Criteria: Like, dismiss, and wishlist actions are registered within 500ms via optimistic UI. Dismissed items are removed from the current session and suppressed in future sessions. Wishlist is persistent across sessions and devices. If a wishlisted item drops in price by ≥10%, the user is notified.

Success Metric: ≥60% of active users perform ≥5 interaction actions per session within 60 days.

**F-04: User Account & Style Profile**

Users create an account that stores their identity, preferences, interaction history, and size information. The style profile is built through onboarding and refined through ongoing interactions.

*User Story:* "As a user, I want to create an account that remembers my preferences and past interactions, so that my experience improves over time."

Acceptance Criteria: Account creation via email or OAuth (Google / Apple). Six-step onboarding flow (age, gender expression, city, lifestyle activities, brand selections, swipe mechanic) seeds the initial taste vector. Users can view a summary of their inferred style profile. All account data is deletable within 30 days per CCPA.

Success Metric: ≥75% of new visitors complete account creation.

**F-05: Product Catalogue Ingestion Pipeline**

Automated ingestion of product data from Shopify-hosted brands via the tokenless Storefront API, running on a 4–6 hour refresh schedule via Trigger.dev.

Acceptance Criteria: Products are ingested and available in the catalogue within one refresh cycle of being added to a brand's Shopify store. Stock availability and pricing sync automatically every 4–6 hours. New products are flagged for AI enrichment (Qwen tagging + Marqo indexing).

Success Metric: ≥95% of ingested products successfully enriched within 24 hours of ingestion.

**F-05B: Shopify Cart & Checkout Handoff**

STROBE operates a native bag experience that owns the full pre-checkout journey — product discovery, detail, size selection, and bag review — before handing off to each brand's Shopify-hosted checkout. The bag groups items by brand, each with its own "Checkout with [Brand]" button that opens the Shopify checkoutUrl.

*User Story:* "As a user who has found products I want to buy, I want to add them to a bag and check out with each brand seamlessly, without leaving the STROBE experience until payment."

Acceptance Criteria: Users can add products to a bag from the product detail page after selecting a size. Bag displays items grouped by brand. Each brand section has a "Checkout with [Brand]" CTA. Tapping checkout opens the Shopify-hosted checkout with the cart pre-populated. All cart events (add, remove, checkout initiated) are logged to the instrumentation layer.

Success Metric: ≥40% of add-to-bag events result in a checkout initiation within the same session.

## P1 — Should-Have (Post-MVP, within 6 months)

**F-06: Push Notifications & Alerts**

Personalised, opt-in notifications for price drops on wishlisted items, new arrivals matching style, and restocks.

Acceptance Criteria: Notifications delivered within 2 hours of triggering event. Max 3 push notifications per day. Tapping deep-links to the relevant product.

**F-07: Brand Analytics Dashboard**

A data portal for brand partners showing aggregated consumer behaviour insights: impressions, clicks, wishlist adds, and conversion funnel data. This is the commercial instrument — the data that justifies direct brand relationships.

Acceptance Criteria: Dashboard displays product-level funnel metrics with ≤4 hour data lag. All data is anonymised and aggregated.

## P2 — Nice-to-Have (V2 and beyond)

- F-08: Social & Sharing Features
- F-09: Outfit Builder
- F-10: Virtual Try-On (AR)
- F-11: Trend Intelligence for Brands (premium data product)
- F-12: Style Concierge (AI Chat)
- F-13: Wardrobe Tool & Auto-Capture
- F-14: Wardrobe Analytics & Insights
- F-15: Native iOS & Android Apps
	- F-16: Full in-app checkout with unified multi-brand cart and payment processing

---

# 6. Explicitly Out of Scope (v0)

1. Own-label or white-label fashion products
2. Secondhand, resale, or peer-to-peer marketplace
3. Influencer content, editorial, or social feed
4. B2B wholesale marketplace
5. Physical retail integrations
6. Subscription box or personal styling service
7. International markets outside the US
8. Full returns management (users directed to brand's returns process)
9. In-app payment processing (Shopify checkout handoff only in v0)
10. Loyalty program or points system
11. Sponsored product placements in organic feed (v0 feed is purely algorithmic)

---

# 7. User Scenarios

## Scenario 1 — First-Time User Onboarding & Feed Seeding

Maya downloads STROBE. She signs up via Apple ID (2 taps). The 6-step onboarding flow asks her age, how she likes to dress (womenswear + androgynous), confirms her city (New York, pre-filled), asks about her lifestyle activities (dinner out: often, office: often, weekends away: sometimes), asks which brands she's shopped recently (The Row, Toteme, COS, Aritzia), then presents 10 curated product cards for swipe-based feedback.

After onboarding, her For You feed loads within 3 seconds with products matched to her taste vector, weighted toward her most frequent occasions (dinner and office). She likes 4 items, dismisses 2, and adds 1 to her wishlist. She closes the app after 12 minutes.

Next session, the feed incorporates her interaction signals. The "dinner out" occasion vector has shifted toward the products she liked.

## Scenario 2 — Cross-Brand Search & Purchase

James needs a slim-fit navy blazer. He searches "slim fit navy blazer" — results load in 1.4 seconds via Marqo semantic search: 87 items across 14 brands. He filters by Size L and Price ≤$400. He taps a blazer from a brand on Shopify, selects Size L, and taps "Add to Bag."

In the bag, the blazer appears under the brand name with a "Checkout with [Brand]" button. He taps it. Shopify's hosted checkout opens with his blazer pre-populated. He completes payment on Shopify.

STROBE has logged: product impression, detail view, add to bag, and checkout initiated — the full conversion funnel for this brand.

## Scenario 3 — Brand Uses Conversion Data

A buyer at Brand X reviews STROBE's conversion funnel data (shared via the analytics dashboard or a CSV export). She sees that Brand X's camel trench coat generated 200 impressions, 45 detail views, 12 add-to-bag events, and 8 checkout initiations on STROBE in the last 30 days — a 4% impression-to-checkout rate. She also sees that "camel outerwear" is trending +42% week-on-week across STROBE's user base. This data supports a reorder decision and opens a conversation about a direct placement arrangement with STROBE.

---

# 8. Non-Functional Requirements

## Performance
- Search results within 2 seconds on up to 500,000 indexed products
- Discovery feed loads within 3 seconds on standard 4G
- ≥99.5% uptime on a rolling 30-day basis
- Product data syncs every 4–6 hours

## Security
- All data in transit encrypted via TLS 1.2+; data at rest encrypted via AES-256
- STROBE does not store or process payment data in v0 (Shopify handles checkout)
- Account deletion within 30 days per CCPA
- All third-party integrations via OAuth 2.0 or secure token exchange

## Accessibility
- WCAG 2.1 AA at launch: screen reader support, ≥4.5:1 contrast, ≥44×44px touch targets
- All product images include descriptive alt text from product metadata

## Scalability
- Support up to 500 brands and 2,000,000 indexed products by Month 18
- Personalisation engine maintains ≤3 second feed load as user base grows to 500,000 MAU
- Architecture supports future international expansion without core redesign

## Platform Support
- v0: Responsive web application (Next.js on Vercel), mobile-first design
- Future: Native iOS and Android apps (approach TBD — native vs cross-platform)

---

# 9. Dependencies & Constraints

## External Dependencies
- Shopify tokenless Storefront API availability and terms stability
- Brand catalogue availability on Shopify (non-Shopify brands cannot be ingested in v0)
- Auth provider decision (NextAuth.js or Clerk) before Phase 4
- Marqo and Qwen availability (local for development, hosted for production)

## Technical Constraints
- Shopify tokenless API query complexity cap of 1,000 units limits products per request to ~20–30
- Personalisation quality proportional to interaction volume — cold start mitigated by 6-step onboarding
- Cart objects are scoped to a single Shopify store — multi-brand bag requires per-brand grouping

## Business Constraints
- MVP must be delivered within budget
- US-only at launch
- Conversion funnel data (500+ checkout_initiated events across 5+ brands) required before approaching brands for direct relationships

---

# 10. Timeline

| Milestone | Timeframe | Included |
|-----------|-----------|----------|
| Pre-build | Weeks 1–2 | Environment, schema, brand list, toolchain |
| Phase 1 | Weeks 3–5 | Shopify ingestion pipeline live, real products in DB |
| Phase 2 | Weeks 6–8 | Browse app + native cart + checkout handoff live on Vercel |
| Phase 3 | Weeks 9–11 | AI enrichment (Qwen + Marqo) + natural language search |
| Phase 4 | Weeks 12–15 | Auth + 6-step onboarding flow |
| Phase 5 | Weeks 16–20 | Personalisation engine + For You feed live |
| Brand outreach | Post Phase 5 | Use conversion data to approach brands |

---

# Appendix A — Alternative Commercial Models (Future Reference)

These models are not being pursued in v0 but are documented for future consideration.

**Model 1: SaaS Subscription Revenue**
Tiered subscriptions for brand partners providing access to consumer behaviour insights, trend data, and product performance analytics. Tiers could include: Basic (product-level metrics), Pro (market trend data), Enterprise (custom reports + API access). Target: $750,000 in combined subscription revenue within 12 months of commercial launch.

**Model 2: Affiliate Commission Revenue**
Commission on sales generated through STROBE. Requires either affiliate network partnerships (e.g., CJ Affiliate, ShareASale, Rakuten) or direct brand agreements. Typical fashion affiliate commission: 5–15% of sale value. This model is simpler to implement than direct brand relationships but yields lower margins and less strategic value.

**Model 3: Hybrid (Direct Relationships + Affiliate)**
Use proprietary conversion data to negotiate direct placement fees and co-marketing arrangements with top-performing brands, while using affiliate commission as the default revenue model for the long tail of brands where direct relationships are not yet established.

# Appendix B — In-App Checkout Spec (Future Reference)

The full in-app checkout specification from the original FRD (Section 8) is retained here for reference when STROBE pursues in-app payment processing. Key components: Stripe integration, PCI DSS compliance, tokenised card storage, Apple Pay / Google Pay / Afterpay support, multi-step checkout flow (shipping → payment → review → confirmation), order management, and returns handling. See archived FRD v1.0 and Backend Map for detailed schemas.

---

*STROBE — PRD · v2 · April 2026 · US Market · Confidential*
