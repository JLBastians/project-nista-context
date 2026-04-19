# STROBE — Decision Log

> v1 · April 2026 · Confidential

---

## How to Use This File

This document is the authoritative record of all project decisions — both resolved and open. Before making assumptions about undecided topics, check this file. When a decision is made, move it from Open Decisions to Resolved with the date and rationale. When something remains under consideration, it stays in Open with options, deadline, and notes.

---

## Open Decisions

| Decision | Context | Options | Deadline | Notes |
|----------|---------|---------|----------|-------|
| **Auth provider** | Application needs user authentication for onboarding and personalisation | (a) NextAuth.js (open-source, self-managed sessions); (b) Clerk (managed service, free tier supports 10k MAU, built-in UI) | Before Phase 4 (Wk 12) | Both support OAuth (Google/Apple). Clerk has faster setup; NextAuth gives more control. |
| **Guest cart sessions** | Should unauthenticated users be able to browse and bag products? | (a) Allow guest carts; prompt sign-up at checkout. (b) Require auth before browsing. (c) Allow guest browse; require auth to bag. | Before Phase 2 (Wk 6) | Current direction: (a). Maximises reach, captures intent signal before ask. |
| **iOS/Android approach** | Build native or cross-platform mobile apps. Not needed until web app is proven. | (a) Native iOS (Swift/SwiftUI) + native Android (Kotlin) — best UX, two codebases. (b) Cross-platform (React Native or Expo) — leverages React/TS, one codebase. (c) Defer decision; focus on web. | Post-web launch (Phase 6+) | Current approach: (c). Web app built mobile-first (375px). Re-evaluate user demand after 4–6 weeks. |
| **Analytics platform** | Track user behaviour (DAU/MAU, feed CTR, conversion funnel, interaction distribution). Currently using Vercel Analytics only. | (a) PostHog (open-source, free tier, event capture); (b) Mixpanel (closed-source, event analytics); (c) Amplitude (product intelligence); (d) Custom events to Neon. | Before Phase 2 (Wk 6) | Need instrumentation from Phase 2 start. Vercel Analytics sufficient for basic metrics until decision made. PostHog or custom recommended for early-stage. |
| **Marqo hosting for v0** | Local Marqo instance unreachable from Vercel. Must move to cloud before Phase 3. | (a) Marqo Cloud (free tier available); (b) Self-hosted on small VM (e.g. Linode, DigitalOcean); (c) Alternative search (Typesense, Elasticsearch). | Before Phase 3 (Wk 9) | Marqo Cloud drop-in replacement; only endpoint URL changes. Recommended: (a). |
| **70/30 taste vector blend ratio** | Session recalculation uses: new_vector = (0.7 × existing) + (0.3 × session_signal). This tuning parameter affects feed freshness vs stability. | Test multiple ratios (60/40, 70/30, 80/20) and measure feed quality, user satisfaction, and vector drift. | Post-Phase 5 (Wk 20+) | Instrument in Phase 5. Start with 70/30 as default. A/B test after launch. |
| **Negative scroll signals** | Should fast scroll-past (user sees card <1s, scrolls immediately) be tracked as negative? | (a) Not in v0 — positive signals only + explicit dismiss. (b) Track and weight at -0.2. (c) A/B test both approaches. | Post-Phase 5 | Current approach: (a). Simplifies initial system. Revisit with real behaviour data. |
| **Non-Shopify brand ingestion** | How to ingest brands not on Shopify (e.g. via custom APIs, CSV imports, custom scrapers)? | (a) Defer until Shopify catalogue is proven. (b) Build generic ingestion framework in Phase 1. (c) Support CSV upload workflow. | Post-Phase 1 (Wk 5+) | Current approach: (a). Focus on Shopify first. Future: generic framework for other platforms. |
| **Neon project structure** | Should dev and prod use separate Neon projects or the same project with schemas? | (a) Separate projects (isolation, cost control, easier schema evolution). (b) Same project, separate schemas (simpler billing, shared migrations). | Before Phase 1 (Wk 3) | Recommended: (a). Isolates risk, allows independent scaling. Use pg_dump/pg_restore for data sync. |
| **GitHub repo structure** | Monorepo (single repo, Frontend + Backend folders) or separate repos? | (a) Monorepo (simpler for small team, shared tooling, easier refactoring). (b) Separate repos (cleaner separation, independent deployments). | Before Phase 1 (Wk 3) | Current approach: (a) monorepo. Recommended for PoC. Can split later if team grows. |
| **Style tiles periodic recalibration** | Resurrect style tiles / brand aesthetics as a periodic recalibration step (e.g. every 6 months)? | (a) Not in v0; defer. (b) Include in onboarding; optional recalibration in profile. | Post-Phase 5 (Phase 6+) | Current approach: (a). Initial seeding via onboarding is sufficient for v0. Revisit if users report stale taste. |
| **In-app checkout timing** | When (if ever) should Stripe in-app checkout be pursued? Requires PCI DSS, brand payment agreements. | (a) Only after proving conversion data value to brands (threshold: 500+ checkout_initiated across 5+ brands). (b) Build in parallel to Phase 5 if capacity exists. (c) Forever defer (rely on Shopify handoff). | Post-Phase 5 | Current approach: (a). Shopify handoff is proof-of-concept sufficient. Reference Backend Map (archived) for full schema when ready. |
| **Cart & checkout handoff timing** | Cart/checkout moved from P0 to P1 in updated PRODUCT_BRIEF.md. Core browse, search, and feed are P0 priorities. | (a) Build cart/checkout immediately after core browse/search completion. (b) Defer until feed personalisation is proven (Phase 4+). (c) Build minimal cart in parallel with feed work in Phase 2. | Before Phase 2 completion (Wk 6) | Per updated PRODUCT_BRIEF.md, Shopify cart handoff is P1. Browse + search + feed are P0. Shopify tokenless cart API available; handoff via checkoutUrl already resolved. This decides sequencing, not architecture. |

---

## Resolved Decisions

| Decision | Resolution | Date | Rationale |
|----------|-----------|------|-----------|
| **Commerce model (v0)** | Proprietary conversion data. STROBE owns discovery-to-checkout-initiation journey. No affiliate or SaaS for v0. | April 2026 | Maximises brand partnership potential. Alternative models (affiliate, SaaS subscriptions) documented in PRD appendix for future. |
| **Checkout architecture (v0)** | STROBE owns pre-checkout experience (discovery, detail, bag). Hands off to Shopify hosted checkout via checkoutUrl. No in-app payment processing. | April 2026 | Avoids PCI compliance burden. User pays on Shopify. STROBE logs checkout_initiated signal. Handoff is drop-in for future Stripe integration. |
| **Product data source** | Shopify tokenless Storefront API only (no auth required). Data treated as short-lived snapshots with expiry timestamps. Re-fetch every 4–6 hours. | April 2026 | Shopify's API terms prohibit permanent warehousing. Snapshot approach maintains compliance. PoC scrapers (Myer, Revolve) replaced by Trigger.dev in Phase 1. |
| **Embedding models** | Marqo-FashionSigLIP (768-dim) for primary search/feed. Qwen3-VL-Embedding-2B (2048-dim) for tagging. Both multi-modal vision-language. | April 2026 | Marqo optimised for fashion. Qwen for rich attribute tagging. Two models enable comparison; redundancy. Local setup in PoC; cloud migration in Phase 3. |
| **Background job platform** | Trigger.dev for scheduled ingestion (Shopify queries) and enrichment jobs (Qwen + Marqo). Replaces custom scrapers. | April 2026 | Managed service, type-safe TypeScript, easy error recovery, monitoring. Integrates with existing Next.js stack. Cost-effective for v0 load. |
| **Database** | Neon serverless PostgreSQL with pgvector 0.8.2 extension. v0 uses Neon production; local strobe_dev for development. | April 2026 | Serverless = no ops overhead. pgvector native = no separate embedding store. Supports 768-dim + 2048-dim vectors. Neon free tier sufficient for PoC. |
| **Frontend hosting** | Vercel. Main branch = landing page (strobelab.co); dev branch = web app. Feed page now connected to local FastAPI backend (`NEXT_PUBLIC_API_URL=http://localhost:8000`). Other pages (wishlist, bag, orders, search) still use mock data — to be connected in later tasks. | April 2026 | Deep Next.js integration. Environment variables for API endpoints. Preview deployments. Vercel Analytics included. Zero-config deployment. |
| **Backend hosting (current)** | Railway.app hosts FastAPI API for PoC (Myer/Revolve scraper data). Will be replaced by Trigger.dev ingestion + Neon in v0. | April 2026 | Temporary setup for PoC enrichment testing. Trigger.dev replaces this in Phase 1. |
| **Framework (frontend)** | Next.js 16 (App Router) with React 19 and TypeScript. Tailwind CSS v4 for styling. shadcn/ui for primitives. | April 2026 | App Router required for dynamic routes and API integration. TypeScript enforces type safety. Tailwind for design system consistency. Mobile-first. |
| **Enrichment approach** | Qwen for visual tagging (product_tags). Marqo for vector embeddings (product_embeddings). Style attribute extraction via zero-shot FashionSigLIP. | April 2026 | Qwen outputs structured JSON tags. Marqo enables nearest-neighbour search. FashionSigLIP for precise attribute extraction. All models run locally or via Trigger.dev. |
| **Style attribute extraction** | Zero-shot FashionSigLIP (fashion-specific embeddings) for structured attribute extraction. Outputs to style_attributes JSONB + style_vector. | April 2026 | Avoids need for hand-labelled training data. Fashion-specific model more accurate than generic. Operational in PoC; integrated in Phase 3. |
| **Development approach (data)** | PoC uses scraped data (Myer, Revolve) for testing. v0 production uses Shopify tokenless API. Scrapers will be archived after Phase 1. | April 2026 | Scraper-based PoC allows rapid enrichment iteration without Shopify setup. Shopify API production-grade, compliant, sustainable long-term. |
| **Shopify DB column naming** | All Shopify-sourced product columns use a `shopify_` prefix in the DB (e.g. `shopify_gid`, `shopify_title`, `shopify_variants`). STROBE-internal columns and existing columns have no prefix. The Pydantic API layer maps `shopify_*` names to clean frontend-friendly names (`shopify_title` → `name`, `shopify_description` → `description`, etc.). | April 2026 | `shopify_` prefix makes data provenance explicit — agents and engineers can immediately see which columns come from Shopify vs other sources. Clean API names keep the frontend contract readable. |
| **Product primary key strategy** | Keep integer SERIAL PKs. API returns `id` as integer; frontend `Product.id` is `number`. No UUID migration. | April 2026 | Integer PKs are simpler and already in use across scrapers and enrichment pipeline. Frontend types align with backend — no string casting needed. UUID migration would require updating all existing rows, scrapers, and tooling. |
| **Onboarding flow design** | 5-step flow for v0: (1) age, (2) gender expression, (3) city, (4) lifestyle activities, (5) brand selections. Swipe mechanic (10-card product swiping) planned as P1 enhancement for stronger taste vector seeding. | April 2026 | Steps 1–5 provide sufficient cold-start personalisation. Brand-prior vector provides initial taste seeding without swipe mechanic. Swipe mechanic deferred to P1 as higher-friction enhanced signal. Reduces onboarding drop-off risk for v0 launch. |
| **Onboarding step count** | 5 steps for v0 (age, gender expression, city, lifestyle activities, brand selections). Swipe mechanic deferred to P1 as enhanced onboarding. | April 2026 | Reduces onboarding friction for v0. Brand-prior vector provides sufficient initial taste seeding. Swipe mechanic adds stronger signal but is not required for launch. |

---

## Future Reference Archive

The following items are explicitly out of v0 scope but documented for future planning:

| Item | Reference | Notes |
|------|-----------|-------|
| Full in-app checkout (Stripe) | `Backend/BACKEND_MAP.md` (archived) | PCI DSS Level 1, tokenised cards, marketplace payments, Apple/Google Pay. Revisit when conversion data demonstrates value. |
| Native mobile apps (iOS + Android) | `Context/PRODUCT_BRIEF.md` | Native (Swift/SwiftUI + Kotlin) or cross-platform (React Native/Expo). Defer until web app proves user demand. Mobile-first web app is primary experience for v0. |
| Push notification infrastructure | `Backend/BACKEND_MAP.md` (archived) | APNs + FCM delivery, frequency capping, notification centre. OneSignal candidate. Build when feature justified by engagement metrics. |
| Retailer analytics dashboard | `Context/PRODUCT_BRIEF.md` | Data portal for partners. Shows funnel, conversion metrics, user analytics. Requires sufficient user base and brand relationships. Phase 2+ feature. |
| SaaS subscription model | `Context/PRODUCT_BRIEF.md` appendix | Tiered brand subscriptions for insights data. Fast revenue path if direct brand relationships take longer. Revisit after proving conversion data. |
| Affiliate commission model | `Context/PRODUCT_BRIEF.md` appendix | Commission on sales via affiliate tracking. May be fastest path if brand relationships are slow. Requires affiliate network setup. |
| Wardrobe tool & outfit pairing | `Context/PRODUCT_BRIEF.md` | Digital closet, outfit recommendations, cost-per-wear analytics. P2 feature, requires user upload flows. |

---

*STROBE · Decision Log · v1 · April 2026 · Confidential*
