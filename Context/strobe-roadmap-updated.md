# STROBE
## Product Roadmap & Detailed Work Breakdown
### v5 · April 2026 · Confidential

> **v5 update notes:** Aligned with Master Directory v1. Responsibility split between cofounders clarified per task. Onboarding tasks updated to match Onboarding Design Document v1.0. Tech stack decisions documented. Interaction signals taxonomy added. No changes to phase structure or sequencing.

---

# 1. Roadmap Overview

| Pre-build (Wks 1–2) | Phase 1 (Wks 3–5) | Phase 2 (Wks 6–8) | Phase 3 (Wks 9–11) | Phase 4 (Wks 12–15) | Phase 5 (Wks 16–20) | Track |
|---|---|---|---|---|---|---|
| Neon + Vercel + GitHub | Trigger.dev setup | Env vars + secrets | Marqo local running | Cloud infra decisions | Marqo Cloud migration | **Infrastructure** |
| Brand list + taxonomy | Ingestion job v1 | Freshness + upsert | Direct brand list | Brand style vectors | Interaction logging | **Data** |
| Qwen local test | Qwen tag pipeline | Marqo index products | Enrich backlog clear | Onboarding embeddings | Taste vector engine | **AI / Enrichment** |
| Schema migrations | Browse UI + filters | Product card + detail | Search mode (Marqo) | Bag + checkout handoff | For You feed UI | **Product** |
| — | — | — | — | Steps 1–6 build | Swipe mechanic + seeding | **Onboarding** |
| — | — | — | — | Profile schema | Vector calc + session update | **Personalisation** |

| Phase | Goal | Founder Focus | Cofounder Focus |
|-------|------|--------------|-----------------|
| **Pre-build** | Everything needed to start building | Neon, Vercel, Trigger.dev, schema, brand list | Qwen local test, Marqo setup, tag taxonomy draft |
| **Phase 1** | Real products in the database | Trigger.dev Shopify ingestion jobs | Monitor enrichment_status queue |
| **Phase 2** | Something real to show; conversion data flowing | Browse UI, product detail, bag, Shopify cart API, checkout handoff, cart_events instrumentation | — |
| **Phase 3** | AI layer proven and working | Tag-based browse filters, search results UI | Qwen enrichment pipeline, Marqo indexing, backlog clearance |
| **Phase 4** | Real users with real profiles | Auth setup, 6-step onboarding UI, interaction logging | Swipe card selection algorithm, brand style vectors, onboarding embeddings |
| **Phase 5** | Core differentiator working | For You feed UI, occasion filters, style summary card | Session recalculation, taste vector engine, feed ranking API |

---

# 2. Pre-build (Weeks 1–2)

**Goal:** Environment, toolchain, schema, brand data — everything needed before building starts.

## Decisions to Make Before Writing Code

| Decision | Notes | Status |
|----------|-------|--------|
| Marqo hosting | Local unreachable from Vercel. Option (b) recommended: deploy to small cloud VM or Marqo Cloud from start. | 🔲 Pending |
| Trigger.dev vs alternatives | Trigger.dev recommended. Alternatives: Inngest, Vercel Cron. | ✅ Trigger.dev confirmed |
| Neon project structure | Separate dev and prod Neon projects (recommended). | 🔲 Pending |
| GitHub repo structure | Monorepo recommended for small team. | 🔲 Pending |
| Auth provider | NextAuth.js vs Clerk. Decide before Phase 4, but worth evaluating early. | 🔲 Pending |
| Analytics platform | PostHog vs Mixpanel vs Amplitude vs custom. Need instrumentation from Phase 2. | 🔲 Pending |

## Task Breakdown

| Task | Detail | Effort | Owner |
|------|--------|--------|-------|
| **Neon DB setup** | Create prod and dev projects. Connection strings. Confirm pgvector enabled. | S | Founder |
| **Vercel project setup** | Connect GitHub. Configure env vars for Neon, Trigger.dev, Marqo. Preview deployments working. | S | Founder |
| **Trigger.dev setup** | Create account/project. Install SDK. Deploy hello-world test job. | S | Founder |
| **Marqo cloud/VM setup** | Stand up Marqo instance. Confirm API reachable from local and Vercel. Create initial product index. | M | Cofounder |
| **Local Qwen confirm** | Confirm Qwen running locally with expected JSON tag output. Document prompt template. | S | Cofounder |
| **Initial brand list** | Compile 15–20 Shopify brands. Confirm .myshopify.com URLs work. Insert into brands table. | M | Founder |
| **Brand sustainability taxonomy** | Lookup table mapping brands to sustainability tiers. Start with 50 most common US fashion brands. | L | Founder |
| **Brand style vectors** | For each initial brand, prepare representative image. Run through Marqo for brand-level vector. | M | Cofounder |
| **Schema migrations** | Run all migrations: brands, products, product_tags, product_embeddings, users, user_taste_profiles, user_interactions, user_saved_items, onboarding tables, cart tables. Verify dev and prod. | M | Founder |
| **Tag taxonomy definition** | Define canonical tag types and allowed values for Qwen output. | M | Cofounder |

---

# 3. Phase 1 (Weeks 3–5) — Data Pipeline

**Goal:** Real products from real brands reliably in the database, refreshing on schedule.

| Task | Detail | Effort | Owner |
|------|--------|--------|-------|
| **Ingestion job v1** | Trigger.dev job: loop active brands, query Shopify GraphQL, upsert products to Neon. | L | Founder |
| **Freshness logic** | fetched_at / expires_at. Skip fresh products. Configurable per brand. | M | Founder |
| **Enrichment status flag** | Set enrichment_status = 'pending' on new/updated products. Don't re-queue unchanged products. | S | Founder |
| **Data quality checks** | Verify counts per brand, missing images, price in USD, stuck pending items. | M | Founder |
| **Error handling + logging** | Handle Shopify 429s, timeouts, malformed responses. Log to job_errors table. | M | Founder |
| **Job monitoring** | Trigger.dev alerts for failures. Health check: no updates in 12 hours = alert. | S | Founder |
| **Pagination handling** | Cursor-based pagination for large catalogues. | M | Founder |
| **Direct brand outreach list** | Prioritised list of brands to approach for direct relationships post-conversion-data. Business dev task. | S | Founder |

**Exit criteria:** ≥10 brands ingested with 50+ products each. Products refreshing on schedule. enrichment_status populating. No job failures in 48 hours.

---

# 4. Phase 2 (Weeks 6–8) — Browse App + Cart

**Goal:** Working product on Vercel. Browse, product detail, native bag, checkout handoff. Conversion data flowing.

## Browse UI

| Task | Detail | Effort | Owner |
|------|--------|--------|-------|
| **Product grid page** | Responsive grid of product cards. Loads from Neon via API route. Default sort: most recently fetched. | M | Founder |
| **Product card component** | STROBE design system (Space Grotesk, #FAFAF8 canvas, #0057FF accent). Hover state. Click → detail. | M | Founder |
| **Brand filter** | Filter by brand. Multi-select. Fetches active brand list from Neon. | M | Founder |
| **Price range filter** | Dual slider or min/max input. $0–$1000+. Client-side filtering. | S | Founder |
| **Basic text search** | ILIKE on title/description. Placeholder until Marqo search in Phase 3. | M | Founder |
| **Empty states** | No results, no products for brand, search returns nothing. Each suggests an action. | S | Founder |
| **Loading states** | Skeleton loaders for grid and detail. No layout shift. | S | Founder |
| **Mobile responsiveness** | 3 cols (desktop) → 2 (tablet) → 1 (mobile). Scroll-friendly product detail. | M | Founder |
| **404 and error pages** | Custom 404. Global error boundary. Offline detection. | S | Founder |

## Product Detail + Cart

| Task | Detail | Effort | Owner |
|------|--------|--------|-------|
| **Product detail page** | All images (carousel), description, price, size variants, brand name. | M | Founder |
| **Cart schema** | Create carts, cart_line_items, cart_events, confirmed_orders tables. | M | Founder |
| **Add to Bag** | Size/variant selector → Shopify cart API → create/update cart → store checkoutUrl. Toast confirmation. | L | Founder |
| **STROBE bag UI** | Bag drawer/page. Items grouped by brand. Brand name, line items, subtotal, "Checkout with [Brand]" button. Remove item. | L | Founder |
| **Checkout handoff** | "Checkout with [Brand]" opens checkoutUrl. Log checkout_initiated to cart_events. Update cart status. | M | Founder |
| **Cart persistence** | localStorage for guests, carts table for authenticated. Survives refresh. 7-day expiry. | M | Founder |
| **cart_events instrumentation** | Log add, remove, update_qty, checkout_initiated. Include user_id, product_id, brand_id, session_id. | M | Founder |
| **Multi-brand bag display** | Multiple brands = distinct sections. Note: "Each brand checks out separately." | S | Founder |
| **Variant availability** | Check availableForSale before add. Grey out OOS sizes. Block OOS add. | M | Founder |

**Exit criteria:** Browse app live on Vercel. Product detail with size selector. Add to Bag calling Shopify cart API. Bag displaying per-brand groupings. Checkout handoff opening Shopify checkout. cart_events logging to Neon. Mobile layout confirmed.

---

# 5. Phase 3 (Weeks 9–11) — AI Enrichment + Search

**Goal:** Qwen tagging pipeline live. Marqo indexing complete. Natural language search working.

| Task | Detail | Effort | Owner |
|------|--------|--------|-------|
| **Qwen enrichment job** | Trigger.dev job: fetch pending products, send to Qwen, parse JSON, insert product_tags. | L | Cofounder |
| **Qwen prompt engineering** | Finalise prompt template. Structured JSON output with tag_type, tag_value, confidence_score. Test on 20 products. | M | Cofounder |
| **Marqo indexing job** | Enriched products → Marqo embedding. Store marqo_document_id in product_embeddings. | L | Cofounder |
| **Backlog clearance** | Run enrichment against full product backlog. Target 95%+ enriched. | M | Cofounder |
| **Tag-based browse filters** | Replace/extend basic filters with aesthetic, occasion, colour palette from product_tags. confidence > 0.5. | L | Founder |
| **Natural language search** | Replace ILIKE with Marqo vector search. Query → Marqo → ranked IDs → Neon → results grid. | L | Founder (UI), Cofounder (Marqo query) |
| **Search results page** | Dedicated page. Query displayed, result count, product grid. Back navigation preserves query. | M | Founder |
| **Enrichment status dashboard (internal)** | Simple view: total products, enriched, pending, failed counts. | S | Founder |
| **Qwen cloud migration planning** | Identify hosted Qwen API candidates. Document endpoint swap. Planning only. | S | Cofounder |

**Exit criteria:** 95%+ products enriched. Tag filters working. Natural language search returning relevant results. Marqo latency under 500ms.

---

# 6. Phase 4 (Weeks 12–15) — Auth + Onboarding

**Goal:** Real users with real profiles and initial taste vectors.

| Task | Detail | Effort | Owner |
|------|--------|--------|-------|
| **Auth setup** | NextAuth.js or Clerk. Email + password, Google OAuth minimum. Protect personalised routes. Browse/search remain public. | L | Founder |
| **Session management** | Generate session_id per visit. Groups interactions. 30-min inactivity expiry. | M | Founder |
| **Onboarding flow shell** | Multi-step container: step indicator, back/next, skip option, progress persistence. STROBE design system. | L | Founder |
| **Step 1 — Age** | Birth year picker. Validate 13–100. Store date_of_birth. Skippable. | S | Founder |
| **Step 2 — Gender expression** | Multi-select: Womenswear / Menswear / Androgynous+gender-neutral / All. Store gender_expression. Skippable. | S | Founder |
| **Step 3 — City** | IP pre-fill (ipapi.co). Confirm/override with autocomplete. Store city + city_source. Skippable. | M | Founder |
| **Step 4 — Lifestyle activities** | Multi-select grid. Frequency selector per activity (Often/Sometimes/Rarely). Store occasion_weights. Min 1 if not skipped. | L | Founder |
| **Step 5 — Brand selections** | Searchable multi-select + ~30 brand quick-select grid. 3–10 selections + secondhand option. Sustainability inference. Brand-prior vector computation. | L | Founder (UI), Cofounder (vector computation) |
| **Step 6 — Swipe mechanic** | Card stack UI. Swipe right/left. 10 product cards. On complete: compute vector, blend with brand prior, write taste_vector. | XL | Founder (UI), Cofounder (card selection algorithm + vector math) |
| **Swipe card selection algorithm** | Select 10 cards: filter by occasions + gender, distribute across style space using brand-prior, ensure visual diversity, allocate by frequency. | L | Cofounder |
| **Cold start fallback** | Users who skip entirely: editorial picks (~20 curated products). strobe_editorial_picks table. | M | Founder |
| **Saved items (wishlist)** | Heart/bookmark on product cards. user_saved_items table. Also logs user_interactions (type = save). | M | Founder |
| **Auth guards** | Gate For You feed and saved items. Contextual sign-up prompt for unauthenticated users. | S | Founder |
| **Interaction logging** | Client-side capture: tap events (product card clicks), dwell tracking (IntersectionObserver — ≥2s threshold), save events, like events, dismiss events. POST to /api/interactions → user_interactions. | L | Founder |
| **Onboarding embeddings** | Pre-compute Marqo embeddings for onboarding style items (if used in swipe card selection). | M | Cofounder |

**Exit criteria:** Users can sign up, complete all 6 onboarding steps. taste_vector written. Interactions logging. Saved items working.

---

# 7. Phase 5 (Weeks 16–20) — Personalisation

**Goal:** For You feed powered by taste vectors by occasion. Feed adapts over time.

| Task | Detail | Effort | Owner |
|------|--------|--------|-------|
| **Session recalculation job** | On session start: fetch previous session interactions, retrieve Marqo vectors, apply weights, compute session signal, blend 70/30 with existing vector. Update user_taste_profiles. | XL | Cofounder |
| **For You feed API route** | Fetch taste_vector → Marqo nearest-neighbour → filter seen products → Neon detail fetch → ranked list. Target: <800ms. | L | Cofounder (query), Founder (API route) |
| **For You feed UI** | Dedicated tab. Product grid. No filters sidebar. Infinite scroll. "Curated for you" framing. Empty state for new users → onboarding. | L | Founder |
| **Seen products filter** | Rolling 7-day window of seen product IDs per user. Exclude from For You. user_seen_products table with TTL. | M | Cofounder |
| **tag_weights update** | Average tag distributions of positively-engaged products → update tag_weights JSONB. | M | Cofounder |
| **Style summary card** | Profile card: human-readable style summary from tag_weights. E.g. "You gravitate toward minimalist, earth tones, oversized silhouettes." Updates each session. | M | Founder |
| **Occasion filter** | Toggle above For You: Everyday / Going Out / Work / Special / All. Filter results by occasion. Interactions feed occasion-specific sub-vector. | L | Founder (UI), Cofounder (sub-vector logic) |
| **Occasion sub-vectors** | Per-occasion vectors: everyday_vector, going_out_vector, work_vector, special_vector. Updated from filtered sessions only. | L | Cofounder |
| **Cold start graduation** | After 5+ meaningful interactions, switch from editorial/trending to personalised feed. "Your feed is personalising" indicator. | M | Cofounder |
| **Trending feed** | Cold start fallback: top 50 products by save count in last 7 days. | S | Founder |
| **Qwen + Marqo cloud migration** | Migrate Qwen to hosted API. Confirm Marqo Cloud handling production load. Remove local dependencies. | L | Cofounder |
| **Performance optimisation** | Audit feed load time. Cache Marqo results (15 min per user). Next.js Image optimisation. | M | Founder |
| **Cart-to-feed signal** | Add add_to_bag (weight 0.9) and checkout_initiated (weight 1.0) as interaction signals feeding taste vector. | M | Cofounder |
| **Conversion reporting** | Internal funnel report: impression → detail → add to bag → checkout initiated, per brand and occasion. CSV export. | M | Founder |
| **Abandoned cart prompt** | Carts active >24h without checkout: gentle in-app prompt on next session. No email without opt-in. | M | Founder |

**Exit criteria:** For You feed live with personalised results. Session recalculation running with all signal types (explicit + passive + cart). Occasion filter working. tag_weights updating. Conversion funnel report showing per-brand data. Feed quality improving after 3+ sessions.

---

# 8. Post-Launch Priorities

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
| In-app checkout (Stripe) | Full payment processing. Refer to Backend Map (archived) for schema. Only after proving conversion data. |
| SaaS subscription model | Tiered brand subscriptions for insights data. Refer to PRD Appendix A. |
| Native iOS/Android apps | Approach TBD (native vs cross-platform). After web app proven. |

---

# 9. Risks & Mitigations

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

*STROBE · Roadmap · v5 · April 2026 · Confidential*
