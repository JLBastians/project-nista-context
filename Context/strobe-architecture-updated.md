# STROBE
## Technical Architecture & Product Design
### Cofounder Reference Document · v5 · April 2026

> **v4.1 update notes:** Aligned with Master Directory v1. Added interaction signals taxonomy. Clarified responsibility split between cofounders. Documented future in-app checkout aspiration. Tech stack decisions updated. No changes to core architecture (five-layer stack, Shopify tokenless ingestion, Marqo + Qwen enrichment, cart/checkout handoff, personalisation layer).

---

# 1. System Overview

STROBE is built on five connected layers. Data flows from Shopify stores through an ingestion pipeline into a PostgreSQL database, where an AI enrichment layer (Qwen + Marqo) adds visual tags and search embeddings. The STROBE Next.js app queries this enriched database to serve two discovery modes — Browse and For You — to end users.

| Layer | Responsibility | Owner |
|-------|---------------|-------|
| Data sources | Shopify stores queried via the public tokenless Storefront API | Founder |
| Ingestion | Trigger.dev scheduled background jobs — refresh every 4–6 hours | Founder |
| Database | Neon serverless PostgreSQL — products, tags, embeddings, users | Founder |
| Enrichment | Qwen (multimodal tagging) + Marqo (vector search indexing) | Cofounder |
| App | Next.js on Vercel — Browse mode and personalised For You feed | Founder (UI), Cofounder (engine) |

> **Local vs cloud:** Qwen and Marqo currently run locally. Before launch, Marqo should move to Marqo Cloud (free tier available) and Qwen to a hosted API. Both are drop-in replacements — only endpoint URLs change.

---

# 2. Layer 1 — Data Sources

## Shopify Tokenless Storefront API

Shopify exposes a subset of store data without authentication. Any store's product catalogue can be queried via GraphQL POST:

`https://{store-name}.myshopify.com/api/2024-01/graphql.json`

No API key or access token required. This also provides cart creation and mutation endpoints, which power STROBE's native bag and checkout handoff.

**What the API returns per product:** Title, description, price, currency, all image URLs, product handle, size/colour variants, collection name.

**What the API does not return:** Payment processing (checkout completion requires Shopify's hosted checkout), inventory levels (not always exposed), data from non-Shopify brands.

**Query complexity limit:** 1,000 units per request. In practice: 20–30 products with a moderate field set per request.

> **Terms consideration:** Shopify's API terms prohibit permanently warehousing product data. STROBE uses short-lived snapshots with expiry timestamps — data is a near-live cache, not a permanent copy. Products are re-fetched on schedule and expired records are purged.

---

# 3. Layer 2 — Ingestion

## Trigger.dev Scheduled Jobs

Background jobs in TypeScript on Trigger.dev, running on a 4–6 hour schedule. The job loops through active brands and queries each Shopify store for products.

**Freshness logic:** Each product row stores fetched_at and expires_at. The job skips products within the freshness window and only re-fetches stale records.

**Enrichment trigger:** When a product is newly inserted or meaningfully updated (price/image change), enrichment_status = 'pending'. A separate enrichment job (cofounder scope) picks up pending products.

**v0 approach:** 15–20 Shopify brands hardcoded in the brands table. The ingestion job is brand-agnostic.

---

# 4. Layer 3 — Database Schema

All data in Neon serverless PostgreSQL. Split into product data, user data, cart data, and onboarding data.

## Product Tables

**brands**

| Column | Type | Purpose |
|--------|------|---------|
| id | uuid PK | Unique identifier |
| name | text | Display name |
| myshopify_url | text | .myshopify.com URL for API queries |
| last_synced_at | timestamp | Last successful query |
| is_active | boolean | Pause without deleting |

**products**

| Column | Type | Purpose |
|--------|------|---------|
| id | uuid PK | Unique identifier |
| brand_id | uuid FK | References brands |
| shopify_product_id | text | Shopify's product ID (deduplication) |
| title | text | Product name |
| description | text | Full description |
| price | decimal | Price in local currency |
| currency | text | Currency code |
| product_url | text | Direct link to product page |
| primary_image_url | text | Main image |
| image_urls | text[] | All images |
| fetched_at | timestamp | Last retrieved from Shopify |
| expires_at | timestamp | When to re-fetch |
| enrichment_status | text | pending / processing / complete / failed |

**product_tags**

| Column | Type | Purpose |
|--------|------|---------|
| id | uuid PK | |
| product_id | uuid FK | References products |
| tag_type | text | silhouette, colour_palette, aesthetic, occasion, etc. |
| tag_value | text | e.g. 'oversized', 'earth tones', 'minimalist' |
| confidence_score | float | LLM confidence (0–1). Filter below 0.5 for display. |

**product_embeddings**

| Column | Type | Purpose |
|--------|------|---------|
| id | uuid PK | |
| product_id | uuid FK | References products |
| marqo_document_id | text | Reference to Marqo vector |
| model_version | text | LLM used |
| created_at | timestamp | |

## User Tables

**users**

| Column | Type | Purpose |
|--------|------|---------|
| id | uuid PK | |
| email | text | User email |
| date_of_birth | date | From onboarding Step 1 |
| gender_expression | text[] | From onboarding Step 2 |
| city | text | From onboarding Step 3 |
| city_source | text | "ip" or "manual" |
| sustainability_orientation | text | Inferred from Step 5 brands |
| price_signal | text | Inferred from brands |
| onboarding_complete | boolean | |
| onboarding_completed_at | timestamp | |
| created_at | timestamp | |

**user_taste_profiles**

| Column | Type | Purpose |
|--------|------|---------|
| id | uuid PK | |
| user_id | uuid FK | One profile per user |
| taste_vector | vector | Current style fingerprint in LLM vector space |
| tag_weights | jsonb | Human-readable style summary |
| occasion_weights | jsonb | From onboarding Step 4 |
| seed_method | text | "swipe" / "brand_prior_only" / "skipped" |
| brand_prior_vector | vector | From Step 5, stored separately |
| everyday_vector | vector | Occasion sub-vector |
| going_out_vector | vector | Occasion sub-vector |
| work_vector | vector | Occasion sub-vector |
| special_vector | vector | Occasion sub-vector |
| last_calculated_at | timestamp | |
| total_interactions | int | Cumulative count |

**user_interactions**

| Column | Type | Purpose |
|--------|------|---------|
| id | uuid PK | |
| user_id | uuid FK | |
| product_id | uuid FK | |
| interaction_type | text | like / dismiss / tap / save / dwell / add_to_bag |
| dwell_seconds | int | Duration for dwell interactions (ignored if under 2s) |
| session_id | text | Groups interactions within a visit |
| source | text | feed / search / wishlist / recommendation |
| feed_position | int | Position in feed (if applicable) |
| created_at | timestamp | |

**user_saved_items**

| Column | Type | Purpose |
|--------|------|---------|
| id | uuid PK | |
| user_id | uuid FK | |
| product_id | uuid FK | |
| price_at_add | decimal | For price drop detection |
| saved_at | timestamp | |

## Cart Tables

**carts**

| Column | Type | Purpose |
|--------|------|---------|
| id | uuid PK | Internal STROBE cart ID |
| user_id | uuid FK | Nullable for guest sessions |
| brand_id | uuid FK | One cart row per brand per session |
| shopify_cart_id | text | Shopify's cart gid |
| checkout_url | text | Refreshed on each cart mutation |
| status | text | active / checkout_initiated / abandoned / expired |
| created_at | timestamp | |
| updated_at | timestamp | |

**cart_line_items**

| Column | Type | Purpose |
|--------|------|---------|
| id | uuid PK | |
| cart_id | uuid FK | |
| product_id | uuid FK | |
| shopify_variant_id | text | Specific size/colour variant |
| quantity | int | |
| price_at_add | decimal | Prevents display inconsistency |
| added_at | timestamp | |

**cart_events**

| Column | Type | Purpose |
|--------|------|---------|
| id | uuid PK | |
| user_id | uuid FK | |
| cart_id | uuid FK | |
| product_id | uuid FK | |
| brand_id | uuid FK | |
| event_type | text | add / remove / update_qty / checkout_initiated |
| session_id | text | |
| created_at | timestamp | |

**confirmed_orders** (placeholder — populated as brand webhooks are established)

| Column | Type | Purpose |
|--------|------|---------|
| id | uuid PK | |
| user_id | uuid FK | Matched by email from webhook |
| brand_id | uuid FK | |
| shopify_order_id | text | From webhook |
| order_value | decimal | For commercial reporting |
| confirmed_at | timestamp | |

## Onboarding Tables

**onboarding_style_items**

| Column | Type | Purpose |
|--------|------|---------|
| id | uuid PK | |
| label | text | Aesthetic label |
| image_url | text | Representative image |
| marqo_document_id | text | Pre-computed embedding |
| representative_tags | text[] | Associated tags |

**onboarding_brand_selections**

| Column | Type | Purpose |
|--------|------|---------|
| id | uuid PK | |
| user_id | uuid FK | |
| brand_name | text | As selected by user |
| brand_id | uuid FK | Nullable if brand not in catalogue |
| selected_at | timestamp | |

**onboarding_swipe_responses**

| Column | Type | Purpose |
|--------|------|---------|
| id | uuid PK | |
| user_id | uuid FK | |
| product_id | uuid FK | |
| response | text | "like" or "pass" |
| card_position | int | 1–10 |
| responded_at | timestamp | |

---

# 5. Layer 4 — Enrichment (Cofounder Scope)

The enrichment layer transforms raw product data into an AI-enriched catalogue: Marqo & Qwen for visual/semantic tags and vector embeddings.

*Implementation details are owned by the cofounder. The founder's interface is: products with enrichment_status = 'pending' are picked up by the enrichment pipeline, and on completion, product_tags rows and product_embeddings rows are written. The app reads these tables.*

---

# 6. Layer 5 — The STROBE App

Next.js on Vercel. Two discovery modes plus cart and bag.

## Browse Mode
Tag-based filtering from Neon. SQL join across products and product_tags. No Marqo query needed.

## Search Mode
User query → Marqo → ranked product IDs → Neon detail fetch → results.

## Cart & Checkout Handoff

STROBE owns the full pre-checkout journey and hands off to Shopify's hosted checkout via checkoutUrl. See Section 4 (Cart Tables) for schema.

**Flow:** User discovers product → product detail → selects size → "Add to Bag" → Shopify cart API creates/updates cart → item in bag → user taps "Checkout with [Brand]" → checkoutUrl opens in browser → user completes payment on Shopify.

**Multi-brand bag:** Items grouped by brand, each group = one Shopify cart object, each with its own checkout button.

> **Future aspiration:** Full in-app checkout where STROBE processes payment directly. Would require Stripe (PCI DSS Level 1), tokenised card storage, and brand-by-brand payment agreements. The Backend Map (archived) contains a detailed schema for this. Not in scope for v0 — revisit when conversion data demonstrates value to brands.

## Instrumentation

Every step of the cart journey is logged to STROBE's database. This is the data that enables direct brand commercial conversations.

| Event | Detail |
|-------|--------|
| Product impression | Product appeared in feed/browse. Logged to user_interactions. |
| Product detail view | User tapped to detail page. Logged as interaction_type = tap. |
| Dwell time | Time product card visible in viewport. Logged as interaction_type = dwell with dwell_seconds. |
| Like | User liked product. Logged as interaction_type = like. |
| Dismiss | User dismissed product. Logged as interaction_type = dismiss. |
| Save / wishlist | User saved product. Logged as interaction_type = save. Also written to user_saved_items. |
| Add to bag | User added to STROBE bag. Logged to cart_events (event_type = add) and user_interactions (interaction_type = add_to_bag). |
| Remove from bag | User removed item. Logged to cart_events (event_type = remove). |
| Checkout initiated | User tapped "Checkout with [Brand]". Logged to cart_events (event_type = checkout_initiated). Strongest purchase-intent signal. |
| Order confirmed (future) | Brand webhook confirms order. Written to confirmed_orders. Requires direct brand relationship. |

---

# 7. Interaction Signals & Weights

These signals feed the recommendation engine. The front-end captures and logs them; the cofounder's engine consumes them.

| Signal | Type | Weight (indicative) | Notes |
|--------|------|---------------------|-------|
| Checkout initiated | Explicit | 1.0 | Strongest signal |
| Save / wishlist add | Explicit | 1.0 | Conscious save |
| Add to bag | Explicit | 0.9 | Strong purchase intent |
| Like | Explicit | 0.8 | Positive signal |
| Product detail view (tap) | Passive | 0.6 | Medium signal |
| Dwell 5+ seconds | Passive | 0.4 | Meaningful attention |
| Dwell 2–5 seconds | Passive | 0.15 | Weak signal |
| Dwell under 2 seconds | Passive | 0.0 | Ignored |
| Dismiss | Explicit | −0.3 | Mild negative push |
| Scroll past quickly | Passive | — | Not captured in v0. Future consideration. |

*Weights are indicative starting points — the cofounder's engine will tune these based on observed behaviour.*

---

# 8. Personalisation Layer (Cofounder Scope)

The personalisation layer powers the For You feed using taste vectors by occasion.

**Core concept:** Every user has a taste vector (a point in Marqo's vector space). The closer a product's vector is to the user's taste vector, the more aligned it is with their style. Per-occasion vectors (everyday, going out, work, special) enable context-specific curation.

**Onboarding seeding:** Brand selections (Step 5) produce a weak brand-prior vector. Swipe mechanic (Step 6) produces direct product-level signal. Combined 60/40 to create the initial taste_vector.

**Session recalculation:** At session start, interactions from the previous session are used to update the taste vector. Blend ratio: new = (0.7 × existing) + (0.3 × session signal). This ratio is a tuning parameter.

**Cold start handling:** Editorial picks (curated ~20 products) for users with no data. Trending (most-saved in last 7 days) as fallback. Graduation to personalised feed after 5+ meaningful interactions.

*Implementation details owned by cofounder.*

---

# 9. Tech Stack

| Layer | Tool | Status |
|-------|------|--------|
| Framework | Next.js (App Router) | ✅ Confirmed |
| Database | Neon serverless PostgreSQL (pgvector) | ✅ Confirmed |
| Hosting (web) | Vercel | ✅ Confirmed |
| Data ingestion | Shopify tokenless Storefront API | ✅ Confirmed |
| Background jobs | Trigger.dev | ✅ Confirmed |
| Enrichment (tagging) | Qwen (local → hosted) | ✅ Confirmed (cofounder) |
| Vector search | Marqo (local → Marqo Cloud) | ✅ Confirmed (cofounder) |
| Cart/checkout | Shopify tokenless cart API | ✅ Confirmed |
| Auth | TBD (NextAuth.js or Clerk) | 🔲 Before Phase 4 |
| iOS/Android | TBD (native vs cross-platform) | 🔲 Future |
| Analytics | TBD (PostHog, Mixpanel, or custom) | 🔲 Before Phase 2 |
| Push notifications | TBD (OneSignal candidate) | 🔲 Deferred |

---

# 10. Open Decisions

| Decision | Context |
|----------|---------|
| Marqo hosting for v0 | Local Marqo unreachable from Vercel. Deploy to Marqo Cloud or small VM before Phase 3. |
| Auth provider | NextAuth.js vs Clerk. Decide before Phase 4. |
| Onboarding step count | 6 may be too many. Consider combining age + gender. |
| 70/30 blend ratio | Taste vector update formula. Instrument and tune based on behaviour. |
| Negative signals | Scroll-past not captured in v0. Revisit with data. |
| Non-Shopify brands | Requires separate ingestion path. Defer until Shopify catalogue established. |
| Guest carts | Allow unauthenticated bag. Prompt sign-up at checkout. |
| In-app checkout timing | Only after proving conversion data value to brands. |

---

# 11. Recommended Build Order

| Phase | Deliverable |
|-------|-------------|
| Pre-build (Wks 1–2) | DB schema + Neon setup. Vercel + Trigger.dev. 15–20 brands inserted. |
| Phase 1 (Wks 3–5) | Ingestion job live. Products flowing from Shopify. Data quality verified. |
| Phase 2 (Wks 6–8) | Browse + cart + checkout handoff live. Users can browse, add to bag, check out via Shopify. |
| Phase 3 (Wks 9–11) | Qwen + Marqo enrichment. Natural language search. Tag-based filters. |
| Phase 4 (Wks 12–15) | Auth + 6-step onboarding. User profiles with taste vectors. |
| Phase 5 (Wks 16–20) | For You feed. Session recalculation. Occasion filters. |

---

# 12. Schema Reference — Future Versions

The Backend Map (archived) contains detailed schemas for features not in v0 but planned for future versions:

- **Full Orders table** with status lifecycle (Processing → Shipped → Delivered → Returned), tracking numbers, shipping addresses, masked payment methods
- **Notifications table** with type-specific triggers, push delivery tracking, frequency capping
- **User profile extensions** including saved payment methods (Stripe Customer ID, tokenised cards), saved addresses, notification preferences per type
- **Checkout tables** including payment intents, tax calculation, promo codes

These schemas should be referenced when building the corresponding features.

---

*STROBE · Architecture · v5 · April 2026 · Confidential*
