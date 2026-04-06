# STROBE — Master Directory & Consolidated Reference
> v1 · April 2026 · Confidential
> Single source of truth for the STROBE project. All other documents are subordinate to this file.

---

## 1. Document Status Register

| Document | Status | Notes |
|----------|--------|-------|
| **This Master Directory** | ✅ ACTIVE — Governing document | Supersedes all other docs where conflicts exist. |
| **STROBE Architecture v5** | ✅ ACTIVE — Updated 3 April 2026 | Primary technical reference | Defines the five-layer stack, Shopify tokenless ingestion, Marqo + Qwen enrichment, cart/checkout handoff model, and personalisation layer. Updated April 2026 to align with this directory. |
| **STROBE Onboarding v1** | ✅ ACTIVE — Updated 3 April 2026 | Primary onboarding reference | Most detailed thinking on the 6-step onboarding flow. Designed specifically to power the recommendation engine through occasion vectors and taste vector seeding. |
| **STROBE Roadmap v5** | ✅ ACTIVE — Updated 3 April 2026 | Primary build sequence | Defines Pre-build → Phase 5 build order. Updated April 2026 to reflect current tech stack decisions and responsibility split. |
| **STROBE PRD v2** | ✅ ACTIVE — Updated 3 April 2026 | Problem statement, personas, success metrics, and features revised to reflect Shopify handoff commerce model. Alternative commercial models documented in appendix for future reference. |
| **STROBE FRD v2** | ✅ ACTIVE — Updated 3 April 2026 | Checkout section rewritten for Shopify handoff. Onboarding section rewritten to match Onboarding v1.0. Orders section revised. Prior Stripe-based checkout spec retained in appendix. |
| **APP_FLOW.md** | ✅ ACTIVE — Updated 3 April 2026 | Navigation, screen inventory, animations, and responsive behaviour sections remain valid. Auth flows, checkout, and onboarding sections revised in line with latest architecture / roadmap file. |
| **BACKEND_MAP.md** | 📦 ARCHIVED — Retained for future reference | Written for a Stripe-checkout, affiliate-commission architecture. No longer reflects v0 direction. Retained because the schema design for in-app orders, payment methods, and notifications is useful reference material for future versions when full in-app checkout may be implemented. |

---

## 2. Authoritative Positions

### 2.1 Commerce Model

**v0 priority:** STROBE's primary commercial model is built on **proprietary conversion data**. STROBE owns the discovery-to-checkout-initiation journey and instruments every step, accumulating funnel data (impression → detail view → add to bag → checkout initiated) that enables direct brand relationships, placement fees, and co-marketing arrangements. Revenue is not the v0 goal — proving the user experience and demonstrating conversion signal to brands is.

**Alternative models (documented for future reference):**
- **SaaS subscription revenue:** Tiered subscriptions for retail partners providing access to consumer behaviour insights, trend data, and product performance analytics via a retailer dashboard. Target: $750K combined revenue within 12 months of commercial launch.
- **Affiliate commission revenue:** Commission on sales generated through STROBE, tracked via affiliate links or attribution. This model requires affiliate network partnerships or direct brand agreements with commission structures.
- **Placement fees:** Brands pay for premium positioning in the feed or search results. Requires sufficient user base to justify fees.

These alternative models are not being built in v0 but should be revisited once conversion data demonstrates STROBE's value to brands. The SaaS + affiliate model may be the fastest path to revenue if direct brand relationships take longer to establish than expected.

*Source: Architecture v4, Section 6.5; PRD v1.0 Appendix*

### 2.2 Checkout Architecture

**v0 (current):** STROBE owns the **pre-checkout experience** (discovery, product detail, size selection, bag review) and hands off to **Shopify's hosted checkout** via `checkoutUrl`. STROBE does not process payments. Each brand in the bag generates its own checkout session via the Shopify tokenless cart API.

**Future aspiration (not in scope for v0):** Full in-app checkout where STROBE processes payment directly, providing a seamless single-checkout experience across multiple brands. This would require a PCI DSS Level 1 compliant payment processor (likely Stripe), tokenised card storage, and potentially a marketplace payment model (Stripe Connect or equivalent) to split funds to individual brands. The Backend Map's checkout schema and Stripe integration spec should be referenced if/when this is pursued. Key benefits of in-app checkout: unified multi-brand cart, richer post-purchase data (confirmed orders rather than checkout-initiated signals), and a more seamless UX. Key blockers: PCI compliance cost, brand-by-brand payment agreements, and regulatory complexity.

*Source: Architecture v4, Section 6.5; Backend Map (archived, for future reference)*

### 2.3 Product Data Source

All product data is ingested from **Shopify stores via the public tokenless Storefront API**. No API key or access token is required. Data is treated as short-lived snapshots with expiry timestamps. Products are re-fetched on a 4–6 hour schedule via Trigger.dev jobs.

*Source: Architecture v4, Sections 2–3*

### 2.4 Search & Personalisation Engine

Search and product enrichment is powered by **Marqo** and **Qwen** (multimodal vector search and tagging). The personalisation layer uses **taste vectors by occasion** powered by Marqo / Qwen (exploration still underway).

Taste vectors are discovered and reinforced through:
- **Explicit interactions:** like, dismiss, wishlist add/remove, add to bag, checkout initiation
- **Passive signals:** taps/touches (product detail views), time spent viewing an item (dwell time), scroll behaviour

Each interaction type carries a weight that contributes to the user's taste vector. The vector exists per occasion (everyday, going out, work, special), enabling context-specific curation. The cofounder owns the matching engine implementation. This master directory covers the front-end UX, data ingestion, and the interfaces between the app and the matching engine.

*Source: Architecture v4, Sections 5–7; Onboarding v1.0*

### 2.5 Onboarding Flow

The onboarding flow is **6 steps**, designed specifically to power the recommendation engine:

| Step | Content | What it powers |
|------|---------|----------------|
| 1 | Age (birth year input) | Age-appropriate curation, cohort analysis |
| 2 | Gender expression (multi-select: Womenswear, Menswear, Androgynous/gender-neutral, All of the above) | Product gender filtering in feed |
| 3 | City (IP pre-fill with manual override) | Climate-relevant curation, occasion norms |
| 4 | Lifestyle activities (multi-select with frequency: Often/Sometimes/Rarely) | **Occasion taste vectors** — frequency maps to vector weighting |
| 5 | Recently shopped brands (searchable multi-select, 3–10 selections) | Sustainability inference, initial style vector seed (brand-derived) |
| 6 | Swipe mechanic (10 curated product cards, swipe right/left) | **Direct taste vector seeding** — product-level signal to calculate initial taste vectors |

Every question directly feeds the recommendation engine. Steps 4 and 6 are the highest-signal steps — activities define which occasion vectors to seed, and swipes provide the first real product-level preference data.

*Source: Onboarding v1.0*

### 2.6 Database & Schema

**v0 authoritative schema (Architecture v4, Section 4):**

Product side: `brands`, `products`, `product_tags`, `product_embeddings`
User side: `users`, `user_taste_profiles`, `user_interactions`, `user_saved_items`, `onboarding_style_items`
Cart side: `carts`, `cart_line_items`, `cart_events`, `confirmed_orders` (placeholder)
Onboarding side: `onboarding_brand_selections`, `onboarding_swipe_responses`

**Future schema reference (Backend Map, archived):**
The Backend Map's schema for Users (with payment methods, saved addresses), Orders (with Stripe payment data, shipping tracking, return status), Notifications (with push delivery tracking), and Checkout (with payment intents, tax calculation) should be referenced when building future versions that include in-app checkout, order management, and push notifications. Key tables to revisit: Orders, Notifications, and the full User profile with saved payment methods and addresses.

*Source: Architecture v4, Section 4; Onboarding v1.0, Section 3; Backend Map (archived)*

### 2.7 Tech Stack

| Layer | Tool | Status |
|-------|------|--------|
| Framework | Next.js (App Router) | ✅ Confirmed, in use |
| Database | Neon serverless PostgreSQL (with pgvector) | ✅ Confirmed, in use |
| Hosting (web app) | Vercel | ✅ Confirmed, in use |
| Data ingestion source | Shopify tokenless Storefront API | ✅ Confirmed |
| Background jobs | Trigger.dev | ✅ Confirmed |
| Product enrichment (tagging) | Qwen (local → hosted API) | ✅ Confirmed, cofounder scope |
| Vector search & embeddings | Marqo (local → Marqo Cloud) | ✅ Confirmed, cofounder scope |
| Cart & checkout | Shopify tokenless cart API → checkoutUrl handoff | ✅ Confirmed |
| Auth | TBD — NextAuth.js or Clerk | 🔲 Decision pending (before Phase 4) |
| iOS app | TBD | 🔲 Not yet scoped. Native (Swift/SwiftUI) vs cross-platform (React Native/Expo) decision pending. |
| Android app | TBD | 🔲 Not yet scoped. Native (Kotlin) vs cross-platform decision pending. |
| Push notifications | TBD | 🔲 Deferred. OneSignal is a candidate when needed. |
| Analytics / event tracking | TBD | 🔲 Not yet decided. Options: PostHog, Mixpanel, Amplitude, or custom. |
| Image CDN / storage | TBD | 🔲 Product images served from Shopify CDN for v0. User avatars TBD. |
| Error monitoring | TBD | 🔲 Options: Sentry, LogRocket. |

**Mobile app considerations:** The PRD originally scoped iOS and Android native apps at MVP launch with responsive web within 3 months. Current priority is the web app on Vercel. Mobile apps are a future decision. Key question: build native (best UX, two codebases) or cross-platform (React Native/Expo — one codebase, leverages existing React/TypeScript skills)? The Next.js web app should be built mobile-first to serve as the primary mobile experience until native apps are justified by user demand.

**Not in v0 stack (noted for future):** Stripe (future in-app checkout), Elasticsearch/Typesense (Marqo handles search), AWS Personalize (cofounder building custom engine), Cloudflare R2/AWS S3 (Shopify CDN sufficient for v0).

---

## 3. Responsibility Split

| Domain | Owner | Notes |
|--------|-------|-------|
| Front-end UX/UI | Founder (you) | All screens, components, interactions, animations. Built with Claude + Cursor. |
| Data ingestion pipeline | Founder (you) | Trigger.dev jobs querying Shopify, writing to Neon. |
| Database schema & migrations | Founder (you) | Neon PostgreSQL setup and management. |
| Cart & checkout handoff | Founder (you) | Shopify tokenless cart API integration, bag UI, checkoutUrl handoff. |
| Onboarding flow | Founder (you) | 6-step flow UI and data capture. |
| AI enrichment (Qwen) | Cofounder | Prompt engineering, tag taxonomy, enrichment pipeline. |
| Vector search (Marqo) | Cofounder | Indexing, nearest-neighbour queries, model selection. |
| Personalisation engine | Cofounder | Taste vector calculation, session recalculation, feed ranking, interaction weighting. |
| Brand style vectors | Cofounder | Pre-computing brand-level embeddings for onboarding. |
| Onboarding swipe card selection algorithm | Cofounder | Selecting the 10 cards for Step 6 using occasion weights + brand prior. |

---

## 4. Front-End Screen Inventory (Authoritative)

### Public Routes (no auth required)

| Route | Purpose |
|-------|---------|
| `/` | Landing page (strobelab.co) — waitlist signup. Separate codebase (JavaScript, main branch). |
| `/welcome` | App entry point. Logo, value prop, Create Account CTA, Sign In CTA. |
| `/register` | Account creation. Name fields, email, password + strength indicator, Apple/Google SSO. |
| `/login` | Returning user auth. Email, password, forgot password link, Apple/Google SSO. |
| `/reset-password` | Password reset request. Email input, send reset link. |

### Authenticated Routes

| Route | Purpose | Bottom Nav |
|-------|---------|------------|
| `/onboarding` | 6-step style profile seeding (age → gender → city → activities → brands → swipe) | No |
| `/feed` | Personalised discovery feed. Sticky header, "For You" heading, 2-col product grid. | Yes |
| `/search` | Cross-catalogue product search. Search bar, filters, results grid, sort, trending/recent. | Yes |
| `/product/[id]` | Product detail. Image carousel, brand/name/price, size selector, Add to Bag, Wishlist. | No |
| `/bag` | STROBE bag (multi-brand cart). Items grouped by brand, per-brand Checkout button. | No |
| `/wishlist` | Saved products + price tracking. 2-col grid, price drop badges. | Yes |
| `/orders` | Checkout history. Checkout-initiated events; confirmed orders where brand webhooks exist. | Yes |
| `/profile` | Account & preferences. Name, email, style summary, notification prefs, logout. | Yes |
| `/notifications` | In-app notification centre. Via bell icon, not bottom nav. | No |

---

## 5. Navigation Architecture (Authoritative)

```
strobelab.co (Landing Page — public, JavaScript, main branch)
│
└── /welcome  [PUBLIC]
    ├── /register  [PUBLIC]
    │   └── /onboarding  [AUTHENTICATED — new users only]
    │       └── /feed
    └── /login  [PUBLIC]
        └── /reset-password  [PUBLIC]

AUTHENTICATED APP
├── /feed  [DEFAULT LANDING]
│   └── /product/[id]
│       ├── Add to Bag → stays on page (bag updates)
│       └── View in Bag → /bag
│
├── /search
│   └── /product/[id]
│
├── /bag
│   └── Checkout with [Brand] → opens Shopify checkoutUrl (external)
│
├── /wishlist
│   └── /product/[id]
│
├── /orders
│
├── /profile
│
└── /notifications (via bell icon)

BOTTOM NAV (5 tabs)
  Feed → /feed
  Search → /search
  Wishlist → /wishlist
  Orders → /orders
  Profile → /profile
```

---

## 6. Data Flow Summary

```
Shopify Stores (tokenless Storefront API)
    │
    ▼
Trigger.dev Ingestion Jobs (every 4-6 hours)
    │
    ▼
Neon PostgreSQL (products, brands tables)
    │
    ├──► Qwen Enrichment (product_tags)     ← Cofounder
    ├──► Marqo Indexing (product_embeddings) ← Cofounder
    │
    ▼
Next.js App (Vercel)
    │
    ├── Browse Mode: SQL queries on products + product_tags
    ├── Search Mode: User query → Marqo → ranked product IDs → Neon details
    ├── For You Feed: User taste_vector (by occasion) → Marqo → Neon details
    │
    ├── Cart: Shopify tokenless cart API (create/update cart per brand)
    │   └── Checkout: Open Shopify checkoutUrl in browser
    │
    └── Instrumentation: All events → cart_events + user_interactions
            │
            ▼
        Recommendation Engine (cofounder)
            │
            ├── Explicit: like, dismiss, wishlist, add to bag, checkout
            └── Passive: taps, dwell time, scroll behaviour
```

---

## 7. Interaction Signals (Authoritative)

These are the signals that feed the recommendation engine. The front-end is responsible for capturing and logging them; the cofounder's engine consumes them.

| Signal | Type | Capture Method | Weight (indicative) |
|--------|------|---------------|---------------------|
| Save / wishlist add | Explicit | Bookmark icon tap → `user_interactions` | High (1.0) |
| Like | Explicit | Heart icon tap → `user_interactions` | High (0.8) |
| Add to bag | Explicit | Add to Bag button → `cart_events` | Very high (0.9) |
| Checkout initiated | Explicit | Checkout with [Brand] tap → `cart_events` | Highest (1.0) |
| Dismiss / hide | Explicit | Long-press or swipe → `user_interactions` | Negative (-0.3) |
| Product detail view (tap) | Passive | Navigate to `/product/[id]` → `user_interactions` | Medium (0.6) |
| Dwell time 5+ seconds | Passive | IntersectionObserver on feed cards → `user_interactions` | Medium (0.4) |
| Dwell time 2–5 seconds | Passive | IntersectionObserver → `user_interactions` | Weak (0.15) |
| Dwell time under 2 seconds | Passive | Not logged | Ignored (0.0) |
| Scroll past quickly | Passive | Not captured in v0 | Future consideration |

*Weights are indicative — the cofounder's engine will tune these based on observed behaviour.*

---

## 8. Build Phase Summary (from Roadmap v4)

| Phase | Weeks | Goal | Owner Focus |
|-------|-------|------|-------------|
| Pre-build | 1–2 | Environment, schema, brand list, toolchain | You: Neon, Vercel, Trigger.dev, schema. Cofounder: Marqo + Qwen local setup. |
| Phase 1 | 3–5 | Ingestion pipeline live, real products in DB | You: Trigger.dev Shopify ingestion jobs. |
| Phase 2 | 6–8 | Browse app + native cart live on Vercel | You: Product grid, detail page, bag, Shopify cart API, checkout handoff. |
| Phase 3 | 9–11 | AI enrichment + natural language search | Cofounder-led. You: tag-based browse filters, search results UI. |
| Phase 4 | 12–15 | Auth + full 6-step onboarding flow | You: Auth setup, onboarding UI. Cofounder: swipe card selection, brand vectors. |
| Phase 5 | 16–20 | Personalisation engine + For You feed | Cofounder-led. You: For You feed UI, style summary, occasion filters. |

---

## 9. Open Decisions

| Decision | Context | Recommendation |
|----------|---------|----------------|
| Auth provider | NextAuth.js vs Clerk | Decide before Phase 4. |
| Onboarding step count | 6 steps may be too many | Consider combining age + gender onto one screen. Aim for 4–5 screens. |
| Guest cart sessions | Allow unauthenticated bag? | Yes — guest carts, prompt sign-up at checkout initiation. |
| iOS/Android approach | Native vs cross-platform | Defer until web app is proven. Cross-platform (React Native/Expo) likely best for small team. |
| Analytics platform | PostHog vs Mixpanel vs Amplitude vs custom | Decide before Phase 2 (need instrumentation from day one). |
| Style tiles recalibration | Resurrect as periodic recalibration? | Defer to post-launch. |
| Negative scroll signals | Track fast scroll-past as negative? | Not in v0. Positive + explicit negative (dismiss) only. |
| Future in-app checkout | When to pursue Stripe-based checkout? | Only after proving conversion data value to brands. Reference Backend Map schema. |

---

## 10. Future Reference Archive

The following items are explicitly out of scope for v0 but documented here so they are not lost:

| Item | Reference Document | Notes |
|------|-------------------|-------|
| Full in-app checkout (Stripe) | Backend Map, FRD Section 8 (Appendix) | PCI DSS compliant payment processing, tokenised cards, Apple/Google Pay, Afterpay. Revisit when brand relationships justify the investment. |
| STROBE-managed orders & returns | Backend Map (Orders schema), FRD Section 9 | Full order lifecycle management including tracking, returns portal. Requires direct brand webhook integrations. |
| Push notification infrastructure | Backend Map (Notifications schema), FRD Section 11 | APNs + FCM delivery, notification centre, frequency capping. Requires OneSignal or equivalent. |
| Retailer analytics dashboard | PRD F-08 | Data portal for retail partners. Requires sufficient user base and interaction data. |
| SaaS subscription model | PRD Section 2 (Goal 3) | Tiered retailer subscriptions. Revisit after proving conversion data value. |
| Affiliate commission model | PRD Appendix | Commission on sales. May be fastest path to revenue if direct brand relationships are slow. |
| Wardrobe tool & analytics | FRD Section 11.5 | Digital wardrobe, outfit pairing, cost-per-wear. P2 feature. |
| Native mobile apps | PRD Section 8 | iOS (Swift/SwiftUI) and Android (Kotlin) or cross-platform. TBD. |

---

*STROBE — Master Directory · v1.1 · 3 April 2026 · Confidential*
