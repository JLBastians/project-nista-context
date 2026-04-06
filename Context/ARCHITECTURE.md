# STROBE — Technical Architecture

> v1 · April 2026 · Confidential

---

## System Overview

STROBE is a fashion discovery app built on five connected layers. Data flows from Shopify stores through a scheduled ingestion pipeline into a PostgreSQL database, where an AI enrichment layer (Qwen + Marqo) adds visual tags and search embeddings. The Next.js app queries this enriched database to serve Browse mode (tag-filtered SQL), Search mode (vector-powered Marqo queries), and a personalised For You feed (taste vectors by occasion) to end users.

| Layer | Responsibility | Owner |
|-------|---------------|-------|
| Data sources | Shopify stores via tokenless Storefront API | Avanti |
| Ingestion | Trigger.dev scheduled jobs (4–6 hour refresh cycle) | Avanti |
| Database | Neon serverless PostgreSQL with pgvector | Avanti |
| Enrichment | Qwen (visual tagging) + Marqo (vector embeddings) | Josh |
| App | Next.js on Vercel — Browse, Search, For You, Cart | Avanti (UI), Josh (engine) |

**Two-phase approach:**
- **Current (PoC):** Myer and Revolve data via custom scrapers; local PostgreSQL; FastAPI on Railway. Used for enrichment testing and early feature development.
- **Target (v0):** Shopify tokenless Storefront API; Trigger.dev ingestion; Neon serverless PostgreSQL; enrichment layer unchanged. Cart/checkout via Shopify tokenless cart API handoff.

---

## Data Flow

### Current (PoC Phase)

```
Retailers (Myer, Revolve)
    │
    ▼
Custom Scrapers (BeautifulSoup, curl_cffi)
    │
    ▼
Local PostgreSQL (strobe_dev)
    │
    ├──► Marqo Embedding Pipeline (local GPU)
    ├──► Qwen Tagging Pipeline (local GPU)
    │
    ▼
FastAPI on Railway (dev API)
    │
    ▼
Next.js on Vercel (dev branch, mock data)
```

### Target (v0 Architecture)

```
Shopify Stores (tokenless API)
    │
    ▼
Trigger.dev Ingestion Jobs (every 4-6 hours)
    │
    ▼
Neon PostgreSQL
    │
    ├──► Qwen Enrichment (product_tags)
    ├──► Marqo Indexing (product_embeddings)
    ├──► Style Attributes (zero-shot FashionSigLIP)
    │
    ▼
Next.js App (Vercel)
    │
    ├── Browse: SQL queries (products + tags)
    ├── Search: Marqo vector search
    ├── For You: Taste vectors by occasion
    ├── Cart: Shopify tokenless API
    │   └── Checkout: Open checkoutUrl (handoff)
    │
    └──► Instrumentation (user_interactions + cart_events)
         │
         ▼
    Recommendation Engine (Josh scope)
```

---

## Tech Stack

| Layer | Tool | Status | Notes |
|-------|------|--------|-------|
| **Framework** | Next.js 16 (App Router, React 19, TypeScript) | ✅ In use | Mobile-first, Tailwind CSS v4 |
| **Database** | Neon serverless PostgreSQL + pgvector | ✅ In use | Local `strobe_dev` for dev; Neon for production |
| **Hosting (Frontend)** | Vercel | ✅ In use | Main branch = landing page; dev branch = web app |
| **Hosting (Backend)** | Railway.app (current); Trigger.dev (target) | ✅ Confirmed | Current API on Railway; ingestion jobs via Trigger.dev |
| **Data Source** | Shopify tokenless Storefront API | ✅ Confirmed | No auth required; short-lived snapshots; 4-6hr refresh |
| **Enrichment (Tagging)** | Qwen3-VL-Embedding-2B | ✅ Confirmed | Multimodal visual tagging → product_tags; currently local |
| **Enrichment (Embeddings)** | Marqo-FashionSigLIP | ✅ Confirmed | 768-dim fashion vectors → product_embeddings; currently local |
| **Style Attributes** | Zero-shot FashionSigLIP classification | ✅ Operational | → style_attributes JSONB + style_vector |
| **Cart & Checkout** | Shopify tokenless cart API | ✅ Confirmed | Multi-brand bag; checkoutUrl handoff to Shopify |
| **State Management** | React Context + useReducer | ✅ In use | Frontend state via `src/lib/store.tsx` |
| **Forms** | react-hook-form + Zod | ✅ In use | Validation and sign-up/onboarding flows |
| **UI Primitives** | shadcn/ui (Radix UI) | ✅ In use | Button, input, select, dialog, etc. |
| **Notifications** | Sonner | ✅ In use | Toast notifications for feedback |
| **Icons** | Lucide React | ✅ In use | Consistent icon library |
| **Font** | Clash Grotesk (Fontshare CDN) | ✅ In use | Brand typography |
| **Analytics** | Vercel Analytics | ✅ In use | Core metrics; extended analytics TBD |
| **Auth** | TBD | 🔲 Before Phase 4 | NextAuth.js or Clerk (not yet implemented) |
| **iOS app** | TBD | 🔲 Future | Native (Swift/SwiftUI) vs cross-platform (React Native) |
| **Android app** | TBD | 🔲 Future | Native (Kotlin) vs cross-platform |
| **Push notifications** | TBD | 🔲 Deferred | OneSignal is candidate when needed |
| **Extended analytics** | TBD | 🔲 Before Phase 2 | PostHog, Mixpanel, Amplitude, or custom |
| **Image CDN** | TBD | 🔲 Shopify CDN sufficient for v0 | Cloudflare R2 or AWS S3 for user-generated content later |
| **Error monitoring** | TBD | 🔲 Deferred | Sentry or LogRocket candidate |

**Not in v0 stack:** Stripe (future in-app checkout), Elasticsearch (Marqo handles search), AWS Personalize (Josh building custom engine), dedicated image hosting (Shopify CDN used instead).

---

## Database Schema Overview

All data lives in Neon serverless PostgreSQL with pgvector 0.8.2 extension. Schema is organised by domain:

| Domain | Tables | Purpose |
|--------|--------|---------|
| **Product** | `brands`, `products`, `product_tags`, `product_embeddings`, `style_attributes` | Product catalog, enrichment outputs |
| **User** | `users`, `user_taste_profiles`, `user_interactions`, `user_saved_items` | User accounts, preferences, interaction history |
| **Cart** | `carts`, `cart_line_items`, `cart_events`, `confirmed_orders` | Shopping bag state and commerce instrumentation |
| **Onboarding** | `onboarding_style_items`, `onboarding_brand_selections`, `onboarding_swipe_responses` | Onboarding flow data and style seeding |

**For full column-level schema details, see `Backend/api/README.md`.**

---

## Layer 1 — Data Sources

### Shopify Tokenless Storefront API

Shopify exposes a public GraphQL endpoint without authentication:

```
https://{store-name}.myshopify.com/api/2024-01/graphql.json
```

**What the API returns per product:** Title, description, price, currency, all image URLs, product handle (slug), size/colour variants, collection membership.

**What it does not return:** Inventory levels (not always exposed), payment processing (requires hosted checkout), data from non-Shopify brands.

**Query complexity limit:** 1,000 units per request. Practical throughput: 20–30 products per request with a moderate field set.

**Terms consideration:** Shopify's API terms prohibit permanently warehousing product data. STROBE treats product data as short-lived snapshots with `expires_at` timestamps. Products are re-fetched on a 4–6 hour schedule and expired records are purged. This maintains API compliance.

**Current PoC phase:** Myer and Revolve data ingested via custom web scrapers (BeautifulSoup + curl_cffi) to Neon and local PostgreSQL, used for enrichment testing. Custom scrapers will be replaced by Shopify API jobs during Phase 1.

---

## Layer 2 — Ingestion

### Trigger.dev Scheduled Jobs (Target v0)

TypeScript background jobs on Trigger.dev, running on a 4–6 hour schedule. The job loop queries each active brand's Shopify store for products.

**Freshness logic:** Each product row stores `fetched_at` and `expires_at` timestamps. The ingestion job skips products within the freshness window and only re-fetches stale or expired records. This minimises API load while keeping data current.

**Enrichment trigger:** When a product is newly inserted or meaningfully updated (price or image change), `enrichment_status` is set to `'pending'`. A separate enrichment job (Josh scope) picks up pending products and writes `product_tags` and `product_embeddings` rows.

**Dynamic on-view refresh (P1 enhancement):** Certain product details (price, availability) will refresh dynamically when a user views the product, if the data hasn't been refreshed within the past ~1 hour. This supplements the 4–6 hour scheduled refresh for time-sensitive data.

**v0 scope:** Target: 200+ Shopify brands at launch (see `Context/PRODUCT_BRIEF.md` Goal 3). Initial development uses 15–20 brands; brand onboarding will scale before launch. Ingestion is brand-agnostic; the job iterates over active brands and pulls their product catalogs.

**Current PoC phase:** Direct scraper-to-DB inserts (no background job layer). This will be replaced by Trigger.dev before launch.

---

## Layer 3 — Enrichment

The enrichment layer transforms raw product data into an AI-enriched catalog. Two independent enrichment systems operate in parallel:

### Qwen3-VL-Embedding-2B — Visual Tagging

Multimodal vision-language model. Accepts product image + title + description; outputs structured tags across multiple categories:
- **Silhouette:** oversized, fitted, relaxed, A-line, etc.
- **Colour palette:** earth tones, pastels, jewel tones, neutrals, etc.
- **Aesthetic:** minimalist, bohemian, maximalist, vintage, futuristic, etc.
- **Occasion:** everyday, going out, work, special events, sports, etc.
- **Material cues:** cotton, linen, silk, knit, denim, etc.

Each tag includes a confidence score (0–1). Tags below 0.5 confidence are filtered out for display.

**Output:** Writes to `product_tags` table (product_id, tag_type, tag_value, confidence_score).

### Marqo-FashionSigLIP — Vector Embeddings

Fashion-specific image-text embedding model. Accepts product image + text description; outputs a 768-dimensional vector in Marqo's vector space. These vectors power:
- **Nearest-neighbour search** in Browse and Search modes
- **Taste vector comparison** in the For You feed (user vector vs product vector)
- **Onboarding swipe card seeding** (occasion weight-guided selection)

**Output:** Writes to `product_embeddings` table (product_id, marqo_document_id, model_version). Vectors stored in Marqo index.

**Two distinct vector types:**

1. **Product vector (Marqo-FashionSigLIP, 768-dim):** Encodes visual and textual similarity *within* a product category. Used primarily for **search** and **similarity matching** (e.g., "find similar blazers", nearest-neighbour browsing). Optimised for intra-category consistency.

2. **Style vector (zero-shot FashionSigLIP, 67-dim):** Encodes structured style attributes (waist fit, sleeve length, neckline, etc.) and broader aesthetic cross-category similarity. Used for **discovery** and **personalisation** in the For You feed (e.g., "this user likes minimalist aesthetic across dresses, bags, and shoes"). Stored in `style_attributes` JSONB and enables occasion-specific taste vectors that remain coherent across different product categories.

**Agents should understand:** Search queries and similarity operations use the 768-dim product vector. Personalisation (taste vectors, For You feed) relies on the 67-dim style vector to capture cross-category aesthetic preference.

### Style Attribute Extraction

Zero-shot classification using FashionSigLIP embeddings. Extracts structured attributes (e.g., waist fit, sleeve length, neckline) from product images and text, stores in `style_attributes` JSONB column along with a style_vector for precise filtering.

**Ownership:** Josh owns all enrichment implementation. Avanti interface: products with `enrichment_status = 'pending'` are picked up; on completion, `product_tags`, `product_embeddings`, and `style_attributes` rows are written. App reads these tables directly.

**For implementation details, see `Backend/model/README.md` and `Backend/attribute-extraction/README.md`.**

---

## Layer 4 — The App

Next.js on Vercel. Four discovery and commerce modes:

### Browse Mode

Tag-based SQL filtering. Users filter by aesthetic, occasion, colour, silhouette, or other tag categories. The app constructs SQL queries joining `products` → `product_tags` and applies filters. No vector search needed. Fast, deterministic results.

### Search Mode

Natural language search. User query is sent to Marqo, which returns ranked product IDs. Those IDs are fetched from Neon for detail rendering (images, price, etc.). Powered by the 768-dim Marqo-FashionSigLIP embeddings.

### For You Feed

Personalised discovery powered by **taste vectors by occasion**. Each user has four occasion-specific vectors (everyday, going_out, work, special). The feed queries Marqo with the relevant occasion vector, returns ranked product IDs, and renders them in priority order. Cold-start users (no interactions) see editorial picks, then trending (most-saved last 7 days).

**Taste vector calculation:** Owned by Josh. Vectors are seeded during onboarding (Step 4: occasion weights; Step 5: brand prior; Step 6: swipe responses). After each session, vectors are recalculated using new interactions blended with prior vectors (70/30 weighting).

### Cart & Checkout Handoff

**Multi-brand bag:** Each brand in the cart is a separate Shopify cart object. Items are grouped by brand in the UI.

**Checkout flow:**
1. User discovers product → product detail page
2. Selects size/colour variant
3. Taps "Add to Bag" → Shopify cart API creates/updates cart
4. Views bag (multi-brand grouping)
5. Taps "Checkout with [Brand]" → opens `checkoutUrl` in browser
6. User completes payment on Shopify's hosted checkout
7. On return, checkout-initiated event is logged to `cart_events` (strongest purchase-intent signal)

**Instrumentation:** Every step is logged. See "Interaction Signals" section below.

---

## Interaction Signals

These signals feed the recommendation engine. The frontend captures and logs them; the Josh's engine consumes them.

| Signal | Type | Capture Method | Weight (indicative) |
|--------|------|----------------|---------------------|
| Checkout initiated | Explicit | "Checkout with [Brand]" tap → `cart_events` | 1.0 (highest) |
| Save / wishlist add | Explicit | Bookmark icon → `user_interactions` | 1.0 |
| Add to bag | Explicit | "Add to Bag" button → `cart_events` | 0.9 |
| Like | Explicit | Heart icon → `user_interactions` | 0.8 |
| Product detail view (tap) | Passive | Navigate to `/product/[id]` → `user_interactions` | 0.6 |
| Dwell 5+ seconds | Passive | IntersectionObserver (feed card visible) → `user_interactions` | 0.4 |
| Dwell 2–5 seconds | Passive | IntersectionObserver → `user_interactions` | 0.15 |
| Dwell under 2 seconds | Passive | Not logged | 0.0 (ignored) |
| Dismiss / hide | Explicit | Swipe/hide button → `user_interactions` | −0.3 (negative push) |
| Scroll past quickly | Passive | Not captured in v0 | — (future) |

*Weights are indicative. The Josh's engine will tune these based on observed user behaviour.*

---

## Personalisation Layer

**Core concept:** Every user has a taste vector (a point in Marqo's 768-dimensional vector space). The closer a product vector is to the user's taste vector, the more aligned it is with their style. Per-occasion vectors (everyday, going_out, work, special) enable context-specific feed curation.

**Onboarding seeding:**
- **v0:** Brand selections (Step 5) produce the primary taste vector seed by averaging selected brands' embeddings. Occasion weights from Step 4 determine per-occasion vector allocation.
- **P1 enhancement:** A swipe mechanic (10 product cards) will provide direct product-level signals. When implemented, the initial taste vector will be calculated as (0.6 × brand prior) + (0.4 × swipe signal).

**Session recalculation:** At session start, interactions from the previous session are used to update taste vectors. Blend formula: `new_vector = (0.7 × existing) + (0.3 × session_signal)`. This keeps the feed fresh while maintaining long-term preference stability.

**Cold start handling:** Without swipes in v0, cold start is more likely. Brand-prior seeding is weaker than brand+swipe seeding. Editorial picks (curated ~20 products) and trending (most-saved products in last 7 days) are the primary fallbacks until 5+ interactions enable a personalised feed.

**Ownership:** Josh scope. Avanti provides UI for For You feed, style summary display, and occasion filter controls.

---

## Responsibility Split

| Domain | Owner | Notes |
|--------|-------|-------|
| Front-end UX/UI | Avanti | All screens, components, interactions, animations. Built with Claude + Cursor. |
| Data ingestion pipeline | Avanti & Josh | Trigger.dev jobs querying Shopify, data validation, writing to Neon. |
| Database schema & migrations | Avanti & Josh | Neon PostgreSQL setup, schema management, backups. |
| Cart & checkout handoff | Avanti | Shopify tokenless cart API integration, bag UI, checkoutUrl handoff. |
| Onboarding flow | Avanti | 5-step onboarding UI, form validation, data capture. Swipe mechanic (P1 enhancement). |
| AI enrichment (Qwen) | Josh | Prompt engineering, tag taxonomy design, enrichment pipeline. |
| Vector search (Marqo) | Josh | Indexing, nearest-neighbour queries, model selection. |
| Personalisation engine | Josh | Taste vector calculation, session recalculation, feed ranking. |
| Brand style vectors | Josh | Pre-computing brand-level embeddings for onboarding seeding. |
| Swipe card selection | Josh | (P1) Algorithm for selecting 10 onboarding cards based on occasion weights. |

---

## Commerce Model

**v0 priority:** STROBE's primary focus is **proprietary conversion data**. The app owns the full discovery-to-checkout-initiation journey and instruments every step: impressions → detail views → bag adds → checkouts. This funnel data enables direct brand relationships, placement partnerships, and co-marketing arrangements. Revenue is not the v0 goal — proving the user experience and demonstrating conversion signal to brands is.

**Alternative models (documented for future reference):** SaaS subscriptions (retailer dashboards), affiliate commissions, placement fees. See `Context/PRODUCT_BRIEF.md` appendix for details. Not being built in v0, but revisitable once conversion data demonstrates value to brands.

---

## Future Reference

Items explicitly out of scope for v0:

| Feature | Reference | Notes |
|---------|-----------|-------|
| Full in-app checkout | `Backend/BACKEND_MAP.md` (archived) | Stripe PCI DSS Level 1, tokenised cards, marketplace payments. Revisit when conversion data justifies investment. |
| Native mobile apps | `Context/PRODUCT_BRIEF.md` | iOS (Swift/SwiftUI) and Android (Kotlin) or React Native/Expo. Defer until web app is proven. |
| Push notifications | `Backend/BACKEND_MAP.md` (archived) | APNs + FCM delivery, notification centre, frequency capping. OneSignal candidate. |
| Retailer analytics dashboard | `Context/PRODUCT_BRIEF.md` | Data portal for retail partners. Requires sufficient user base and interaction data. |
| Wardrobe tool & outfit pairing | `Context/PRODUCT_BRIEF.md` | Digital wardrobe, cost-per-wear analytics. Phase 2 feature. |

---

*STROBE · Architecture · v1 · April 2026 · Confidential*
                                                                        