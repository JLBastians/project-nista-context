# STROBE — Product Roadmap

> v1 · April 2026 · Confidential

---

## Roadmap Overview

| Pre-build (Wks 1–2) | Phase 1 (Wks 3–5) | Phase 2 (Wks 6–8) | Phase 3 (Wks 9–11) | Phase 4 (Wks 12–15) | Phase 5 (Wks 16–20) | Track |
|---|---|---|---|---|---|---|
| Neon + Vercel + GitHub | Trigger.dev setup | Env vars + secrets | Marqo local running | Cloud infra decisions | Marqo Cloud migration | **Infrastructure** |
| Brand list + taxonomy | Ingestion job v1 | Freshness + upsert | Direct brand list | Brand style vectors | Interaction logging | **Data** |
| Qwen local test | Qwen tag pipeline | Marqo index products | Enrich backlog clear | Onboarding embeddings | Taste vector engine | **AI / Enrichment** |
| Schema migrations | Browse UI + filters | Product card + detail | Search mode (Marqo) | Bag + checkout handoff | For You feed UI | **Product** |
| — | — | — | — | Steps 1–5 build | Swipe mechanic (P1) | **Onboarding** |
| — | — | — | — | Profile schema | Vector calc + session update | **Personalisation** |

| Phase | Goal | Avanti Focus | Josh Focus |
|-------|------|--------------|-----------------|
| **Pre-build** | Everything needed to start building | Neon, Vercel, Trigger.dev, schema, brand list | Qwen local test, Marqo setup, tag taxonomy draft |
| **Phase 1** | Real products in the database | Trigger.dev Shopify ingestion jobs | Monitor enrichment_status queue |
| **Phase 2** | Something real to show; conversion data flowing | Browse UI, product detail, bag, Shopify cart API, checkout handoff, cart_events instrumentation | — |
| **Phase 3** | AI layer proven and working | Tag-based browse filters, search results UI | Qwen enrichment pipeline, Marqo indexing, backlog clearance |
| **Phase 4** | Real users with real profiles | Auth setup, 5-step onboarding UI, interaction logging | Brand style vectors, onboarding embeddings (swipe card selection deferred to P1) |
| **Phase 5** | Core differentiator working | For You feed UI, occasion filters, style summary card | Session recalculation, taste vector engine, feed ranking API |

---

## Pre-build (Weeks 1–2)

**Goal:** Environment, toolchain, schema, brand data — everything needed before building starts.

### Decisions to Make Before Writing Code

| Decision | Notes | Status |
|----------|-------|--------|
| Marqo hosting | Local unreachable from Vercel. Option (b) recommended: deploy to small cloud VM or Marqo Cloud from start. | 🔲 Pending |
| Trigger.dev vs alternatives | Trigger.dev recommended. Alternatives: Inngest, Vercel Cron. | ✅ Trigger.dev confirmed |
| Neon project structure | Separate dev and prod Neon projects (recommended). | 🔲 Pending |
| GitHub repo structure | Monorepo recommended for small team. | 🔲 Pending |
| Auth provider | NextAuth.js vs Clerk. Decide before Phase 4, but worth evaluating early. | 🔲 Pending |
| Analytics platform | PostHog vs Mixpanel vs Amplitude vs custom. Need instrumentation from Phase 2. | 🔲 Pending |

### Task Breakdown

| Task | Detail | Effort | Owner |
|------|--------|--------|-------|
| **Neon DB setup** | Create prod and dev projects. Connection strings. Confirm pgvector enabled. | S | Josh |
| **Vercel project setup** | Connect GitHub. Configure env vars for Neon, Trigger.dev, Marqo. Preview deployments working. | S | Avanti |
| **Trigger.dev setup** | Create account/project. Install SDK. Deploy hello-world test job. | S | Josh |
| **Marqo cloud/VM setup** | Stand up Marqo instance. Confirm API reachable from local and Vercel. Create initial product index. | M | Josh |
| **Local Qwen confirm** | Confirm Qwen running locally with expected JSON tag output. Document prompt template. | S | Josh |
| **Initial brand list** | Compile initial 15–20 Shopify brands for development. Target: 200+ brands at launch. Confirm .myshopify.com URLs work. Insert into brands table. | M | Avanti |
| **Brand sustainability taxonomy** | Lookup table mapping brands to sustainability tiers. Start with 50 most common US fashion brands. | L | Avanti |
| **Brand style vectors** | For each initial brand, prepare representative image. Run through Marqo for brand-level vector. | M | Josh |
| **Schema migrations** | Run all migrations: brands, products, product_tags, product_embeddings, users, user_taste_profiles, user_interactions, user_saved_items, onboarding tables, cart tables. Verify dev and prod. | M | Josh |
| **Tag taxonomy definition** | Define canonical tag types and allowed values for Qwen output. | M | Josh |

**Exit criteria:** Neon + Vercel + Trigger.dev operational. Brand list seeded. Schema live in dev and prod. Marqo and Qwen running locally and reachable.

---

## Phase 1 (Weeks 3–5) — Data Pipeline

**Goal:** Real products from real brands reliably in the database, refreshing on schedule.

| Task | Detail | Effort | Owner |
|------|--------|--------|-------|
| **Ingestion job v1** | Trigger.dev job: loop active brands, query Shopify GraphQL, upsert products to Neon. | L | Josh |
| **Freshness logic** | fetched_at / expires_at. Skip fresh products. Configurable per brand. | M | Josh |
| **Enrichment status flag** | Set enrichment_status = 'pending' on new/updated products. Don't re-queue unchanged products. | S | Josh |
| **Data quality checks** | Verify counts per brand, missing images, price in USD, stuck pending items. | M | Avanti |
| **Error handling + logging** | Handle Shopify 429s, timeouts, malformed responses. Log to job_errors table. | M | Avanti |
| **Job monitoring** | Trigger.dev alerts for failures. Health check: no updates in 12 hours = alert. | S | Avanti |
| **Pagination handling** | Cursor-based pagination for large catalogues. | M | Josh |
| **Direct brand outreach list** | Prioritised list of brands to approach for direct relationships post-conversion-data. Business dev task. | S | Avanti |

**Exit criteria:** ≥10 brands ingested with 50+ products each. Products refreshing on schedule. enrichment_status populating. No job failures in 48 hours. *Note: Brand count scales to 200+ before launch per PRODUCT_BRIEF.md Goal 3.*

---

## Phase 2 (Weeks 6–8) — Browse App + Cart

**Goal:** Working product on Vercel. Browse, product detail, native bag, checkout handoff. Conversion data flowing.

### Browse UI

| Task | Detail | Effort | Owner |
|------|--------|--------|-------|
| **Product grid page** | Responsive grid of product cards. Loads from Neon via API route. Default sort: most recently fetched. | M | Avanti |
| **Product card component** | STROBE design system (Clash Grotesk, #FAFAF8 canvas, #0057FF accent). Hover state. Click → detail. | M | Avanti |
| **Brand filter** | Filter by brand. Multi-select. Fetches active brand list from Neon. | M | Avanti |
| **Price range filter** | Dual slider or min/max input. $0–$1000+. Client-side filtering. | S | Avanti |
| **Basic text search** | ILIKE on title/description. Placeholder until Marqo search in Phase 3. | M | Avantio |
| **Empty states** | No results, no products for brand, search returns nothing. Each suggests an action. | S | Avanti |
| **Loading states** | Skeleton loaders for grid and detail. No layout shift. | S | Avanti |
| **Mobile responsiveness** | 3 cols (desktop) → 2 (tablet) → 1 (mobile). Scroll-friendly product detail. | M | Avanti |
| **404 and error pages** | Custom 404. Global error boundary. Offline detection. | S | Avanti |

### Product Detail + Cart

| Task | Detail | Effort | Owner |
|------|--------|--------|-------|
| **Product detail page** | All images (carousel), description, price, size variants, brand name. | M | Avanti |
| **Cart schema** | Create carts, cart_line_items, cart_events, confirmed_orders tables. | M | Avanti |
| **Add to Bag** | Size/variant selector → Shopify cart API → create/update cart → store checkoutUrl. Toast confirmation. | L | Avanti |
| **STROBE bag UI** | Bag drawer/page. Items grouped by brand. Brand name, line items, subtotal, "Checkout with [Brand]" button. Remove item. | L | Avanti |
| **Checkout handoff** | "Checkout with [Brand]" opens checkoutUrl. Log checkout_initiated to cart_events. Update cart status. | M | Avanti |
| **Cart persistence** | localStorage for guests, carts table for authenticated. Survives refresh. 7-day expiry. | M | Avanti |
| **cart_events instrumentation** | Log add, remove, update_qty, checkout_initiated. Include user_id, product_id, brand_id, session_id. | M | Avanti |
| **Multi-brand bag display** | Multiple brands = distinct sections. Note: "Each brand checks out separately." | S | Avanti |
| **Variant availability** | Check availableForSale before add. Grey out OOS sizes. Block OOS add. | M | Avanti |

**Exit criteria:** Browse app live on Vercel. Product detail with size selector. Add to Bag calling Shopify cart API. Bag displaying per-brand groupings. Checkout handoff opening Shopify checkout. cart_events logging to Neon. Mobile layout confirmed.

**Note on Cart/Checkout Priority:** Cart and checkout handoff are now P1 priorities per updated PRODUCT_BRIEF.md. Phase 2 builds browse UI and product detail as the foundation. Full cart/checkout implementation should be sequenced after core browse and search are proven to work reliably, ensuring conversion funnel is validated before personalisation.

---

## Phase 3 (Weeks 9–11) — AI Enrichment + Search

**Goal:** Qwen tagging pipeline live. Marqo indexing complete. Natural language search working.

| Task | Detail | Effort | Owner |
|------|--------|--------|-------|
| **Qwen enrichment job** | Trigger.dev job: fetch pending products, send to Qwen, parse JSON, insert product_tags. | L | Josh |
| **Qwen prompt engineering** | Finalise prompt template. Structured JSON output with tag_type, tag_value, confidence_score. Test on 20 products. | M | Josh |
| **Marqo indexing job** | Enriched products → Marqo embedding. Store marqo_document_id in product_embeddings. | L | Josh |
| **Backlog clearance** | Run enrichment against full product backlog. Target 95%+ enriched. | M | Josh |
| **Tag-based browse filters** | Replace/extend basic filters with aesthetic, occasion, colour palette from product_tags. confidence > 0.5. | L | Josh |
| **Natural language search** | Replace ILIKE with Marqo vector search. Query → Marqo → ranked IDs → Neon → results grid. | L | Avanti (UI), Josh (Marqo query) |
| **Search results page** | Dedicated page. Query displayed, result count, product grid. Back navigation preserves query. | M | Avanti |
| **Enrichment status dashboard (internal)** | Simple view: total products, enriched, pending, failed counts. | S | Josh |
| **Qwen cloud migration planning** | Identify hosted Qwen API candidates. Document endpoint swap. Planning only. | S | Josh |

**Exit criteria:** 95%+ products enriched. Tag filters working. Natural language search returning relevant results. Marqo latency under 500ms.

---

## Phase 4 (Weeks 12–15) — Auth + Onboarding

**Goal:** Real users with real profiles and initial taste vectors.

For the full 5-step onboarding flow specification, see `Context/specs/onboarding.md`.

| Task | Detail | Effort | Owner |
|------|--------|--------|-------|
| **Auth setup** | NextAuth.js or Clerk. Email + password, Google OAuth minimum. Protect personalised routes. Browse/search remain public. | L | Avanti |
| **Session management** | Generate session_id per visit. Groups interactions. 30-min inactivity expiry. | M | Avanti |
| **Onboarding flow shell** | Multi-step container (5 steps): step indicator, back/next, skip option, progress persistence. STROBE design system. | L | Avanti |
| **Step 1 — Age** | Birth year picker. Validate 13–100. Store date_of_birth. Skippable. | S | Avanti |
| **Step 2 — Gender expression** | Multi-select: Womenswear / Menswear / Androgynous+gender-neutral / All. Store gender_expression. Skippable. | S | Avanti |
| **Step 3 — City** | IP pre-fill (ipapi.co). Confirm/override with autocomplete. Store city + city_source. Skippable. | M | Avanti |
| **Step 4 — Lifestyle activities** | Multi-select grid. Frequency selector per activity (Often/Sometimes/Rarely). Store occasion_weights. Min 1 if not skipped. | L | Avanti |
| **Step 5 — Brand selections** | Searchable multi-select + ~30 brand quick-select grid. 3–10 selections + secondhand option. Sustainability inference. Brand-prior vector computation. | L | Avanti (UI), Josh (vector computation) |
| **Step 6 — Swipe mechanic (P1 — deferred)** | Card stack UI + vector computation. Moved to P1 post-MVP. Not in v0 onboarding. | XL | Avanti (UI), Josh (algorithm) |
| **Swipe card selection algorithm (P1 — deferred)** | Select 10 cards: filter by occasions + gender, distribute across style space. Deferred to P1. | L | Josh |
| **Cold start fallback** | Users who skip entirely: editorial picks (~20 curated products). strobe_editorial_picks table. | M | Avanti |
| **Saved items (wishlist)** | Heart/bookmark on product cards. user_saved_items table. Also logs user_interactions (type = save). | M | Avanti |
| **Auth guards** | Gate For You feed and saved items. Contextual sign-up prompt for unauthenticated users. | S | Avanti |
| **Interaction logging** | Client-side capture: tap events (product card clicks), dwell tracking (IntersectionObserver — ≥2s threshold), save events, like events, dismiss events. POST to /api/interactions → user_interactions. | L | Avanti |
| **Onboarding embeddings** | Pre-compute Marqo embeddings for onboarding style items (if used in swipe card selection). | M | Josh |

**Exit criteria:** Users can sign up, complete all 5 onboarding steps. taste_vector written. Interactions logging. Saved items working.

---

## Phase 5 (Weeks 16–20) — Personalisation

**Goal:** For You feed powered by taste vectors by occasion. Feed adapts over time.

| Task | Detail | Effort | Owner |
|------|--------|--------|-------|
| **Session recalculation job** | On session start: fetch previous session interactions, retrieve Marqo vectors, apply weights, compute session signal, blend 70/30 with existing vector. Update user_taste_profiles. | XL | Josh |
| **For You feed API route** | Fetch taste_vector → Marqo nearest-neighbour → filter seen products → Neon detail fetch → ranked list. Target: <800ms. | L | Josh (query), Josh (API route) |
| **For You feed UI** | Dedicated tab. Product grid. No filters sidebar. Infinite scroll. "Curated for you" framing. Empty state for new users → onboarding. | L | Avanti |
| **Seen products filter** | Rolling 7-day window of seen product IDs per user. Exclude from For You. user_seen_products table with TTL. | M | Josh |
| **tag_weights update** | Average tag distributions of positively-engaged products → update tag_weights JSONB. | M | Josh |
| **Style summary card** | Profile card: human-readable style summary from tag_weights. E.g. "You gravitate toward minimalist, earth tones, oversized silhouettes." Updates each session. | M | Avanti |
| **Occasion filter** | Toggle above For You: Everyday / Going Out / Work / Special / All. Filter results by occasion. Interactions feed occasion-specific sub-vector. | L | Avanti (UI), Josh (sub-vector logic) |
| **Occasion sub-vectors** | Per-occasion vectors: everyday_vector, going_out_vector, work_vector, special_vector. Updated from filtered sessions only. | L | Avanti |
| **Cold start graduation** | After 5+ meaningful interactions, switch from editorial/trending to personalised feed. "Your feed is personalising" indicator. | M | Avanti |
| **Trending feed** | Cold start fallback: top 50 products by save count in last 7 days. | S | Avanti |
| **Qwen + Marqo cloud migration** | Migrate Qwen to hosted API. Confirm Marqo Cloud handling production load. Remove local dependencies. | L | Josh |
| **Performance optimisation** | Audit feed load time. Cache Marqo results (15 min per user). Next.js Image optimisation. | M | Avanti |
| **Cart-to-feed signal** | Add add_to_bag (weight 0.9) and checkout_initiated (weight 1.0) as interaction signals feeding taste vector. | M | Josh |
| **Conversion reporting** | Internal funnel report: impression → detail → add to bag → checkout initiated, per brand and occasion. CSV export. | M | Josh |
| **Abandoned cart prompt** | Carts active >24h without checkout: gentle in-app prompt on next session. No email without opt-in. | M | Josh |

**Exit criteria:** For You feed live with personalised results. Session recalculation running with all signal types (explicit + passive + cart). Occasion filter working. tag_weights updating. Conversion funnel report showing per-brand data. Feed quality improving after 3+ sessions.

---

## Post-Launch Priorities

| Feature | Notes |
|---------|-------|
| Automatic taste cluster discovery | K-means on interaction history. Requires 50+ interactions/user. Build after 4–6 weeks of user data. |
| Negative scroll signal | Track fast scroll-past. A/B test against positive-only baseline. |
| Non-Shopify brand ingestion | Separate ingestion path. Defer until Shopify catalogue established. |
| Brand self-registration portal | Brands submit .myshopify.com URL for ingestion. Currently manual. |
| Push notifications | Price drops, new arrivals, restocks. Requires OneSignal or equivalent + native app. |
| Style profile sharing | Public profile pages, sharing mechanic. Organic acquisition driver. |
| Analytics dashboard (internal) | DAU/MAU, feed CTR, save rate, add-to-bag rate, checkout rate, onboarding completion by step. |
| Confirmed order webhooks | Per-brand Shopify webhooks. Definitive commercial metric. |
| Direct brand commercial model | Use conversion data to approach brands. Target: 500+ checkout_initiated across 5+ brands. |
| In-app checkout (Stripe) | Full payment processing. For details, refer to ARCHITECTURE.md Section 7 (archived). Only after proving conversion data. |
| SaaS subscription model | Tiered brand subscriptions for insights data. Refer to Master Directory, Section 2.1. |
| Native iOS/Android apps | Approach TBD (native vs cross-platform). After web app proven. |

---

## Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| Shopify API terms tighten | Short-lived snapshots with expiry. Strongest defensible position. |
| Qwen enrichment quality | Manual QA on 50 products before full run. confidence_score filter at 0.5. |
| Marqo search relevance | Monitor search CTR. Re-index with updated embeddings after Phase 5. |
| Cold start feed quality | Editorial picks hand-curated. Swipe mechanic gives immediate signal. |
| Build velocity | XL tasks documented in detail for smaller Cursor sessions. Flag early if slipping. |
| Shopify checkoutUrl expiry | Refresh checkoutUrl on every cart mutation. |
| Onboarding drop-off | 6 steps may be too many. Monitor completion rates by step. Consider combining Steps 1+2. |

---

*STROBE · Roadmap · v1 · April 2026 · Confidential*
                                                                                                                                                                                                     