# Feed API — Scoping Document

> April 2026 · Project Nista / STROBE

---

## 1. Purpose

This document scopes the backend work needed to connect the frontend feed (on the `dev` branch) to a real API, replacing the current mock data. It covers the required endpoints, new database tables, data shape alignment, and a recommended build sequence. The scope is limited to the For You feed — onboarding, auth, search, cart/checkout, and profile are excluded.

**Source documents:** `Context/specs/feed.md`, `Context/APP_FLOW.md`, `Context/ARCHITECTURE.md` (Sections: Layer 4, Interaction Signals, Personalisation Layer)

---

## 2. Current State

### 2.1 Frontend (dev branch)

The feed page (`src/app/(app)/feed/page.tsx`) is fully built with gesture interactions, infinite scroll, occasion/category filtering, a gesture tutorial, and scroll restoration. All product data comes from a hardcoded mock array of 20 products in `src/lib/mock-data.ts`. There are **zero API calls** — interactions (likes, dismisses, wishlist saves) are stored in React Context + localStorage only and are never sent to the backend.

The client-side scoring function (`src/lib/feed-utils.ts`) currently ranks products by category match (+30), aesthetic tag overlap (+10 each, max +40), price range (+20), and existing matchScore (+10). This will be replaced by server-side style vector similarity once the taste vector system exists.

**Key frontend data types** (from `src/lib/types.ts`):

| Field | Frontend type | Backend equivalent | Gap? |
|-------|--------------|-------------------|------|
| id | string ("p1") | product_id (integer) | **Resolved** — frontend changing to integer IDs (decision #2) |
| name | string | shopify_title | **Resolved** — map shopify_title → name in API response |
| brand | string | brand | OK |
| retailer | string | retailer | OK |
| price | number | price | OK |
| originalPrice | number | shopify_compare_at_price | **Resolved** — map to originalPrice in API response |
| images | string[] | image_urls | OK |
| category | string | category | OK |
| subcategory | string | subcategory | **Resolved** — column now exists on products table |
| sizes | {label, available}[] | `sizes` JSONB column | **✅ Resolved** — denormalised at ingestion. Serialiser maps to `[{label, available}]` (v0: product-level availability) |
| colors | string[] | `colors` TEXT[] column | **✅ Resolved** — denormalised at ingestion. Direct mapping to `string[]` |
| matchScore | number (0–100) | — | Computed at query time from vector distance |
| tags | string[] | structured_tags (JSONB) | Partial — JSONB keys differ from flat tag array |
| occasion | Occasion | — | Not in products table (derivable from style_attributes or structured_tags) |
| isNew | boolean | — | Derivable from created_at |
| isSale | boolean | — | Derivable from shopify_compare_at_price > price |

### 2.2 Backend (current)

Existing endpoints in `api/main.py`:

| Endpoint | Relevant to feed? | Notes |
|----------|--------------------|-------|
| POST /products/similar | **Yes — core building block** | Vector similarity search; accepts any embedding column + optional category filter. Could be the foundation for the feed query. |
| GET /products/{id} | Yes | Single product lookup. |
| GET /products/category/{category} | Partially | Category filter, but no vector ranking. |
| ~~POST /swipes~~ | ~~Partially~~ | **Being removed.** Replaced by `POST /interactions` (see Section 3.2 and TODO 14). |
| ~~GET /users/{id}/swipes~~ | ~~Partially~~ | **Being removed.** Replaced by `GET /users/{id}/interactions` (see TODO 14e). |

**Tables and columns added since initial scope (Shopify migration, April 2026):**

- `brands` table — Shopify store registry (brand_id, name, myshopify_domain, storefront_api_version, is_active, fetch_interval_hours). Products now have a `brand_id` FK to this table.
- `products` table now includes Shopify-sourced columns: `shopify_gid` (unique), `shopify_handle`, `shopify_title`, `shopify_product_type`, `shopify_description`, `shopify_currency`, `shopify_compare_at_price`, `shopify_variants` (JSONB with full size/colour/price per variant), `shopify_created_at`, `shopify_updated_at`.
- `products` table also has new STROBE-internal columns: `subcategory`, `fetched_at`, `expires_at`, `enrichment_status` (pending/processing/complete/failed).
- Data ingestion now uses Shopify tokenless Storefront API via `upsert_shopify_products()` with freshness-based dedup.

**Tables still needed for feed:**

- `swipe_history` — being dropped entirely (decision #6). Replaced by `user_interactions`.
- `style_preferences` — stores a single 768-dim preference vector per profile per product type. The feed uses a different approach (per-occasion 67-dim taste vectors). Can coexist for now.
- `user_interactions` — not yet created. Needed for full signal set (like, dismiss, save, tap, dwell). See TODO 14a.
- `user_saved_items` — not yet created. Needed for wishlist / price drop detection. See TODO 14b.
- `user_taste_vectors` — not yet created. Needed for personalised feed ranking.
- Trending product logic — not yet built. Sole cold-start fallback (editorial picks deferred).

---

## 3. Required New Endpoints

### 3.1 GET /feed

**Purpose:** Return a paginated, personalised product feed for an authenticated user.

**Query parameters:**
- `user_id` (required) — will become implicit once auth is implemented
- `occasion` (optional) — "everyday" | "going_out" | "work" | "special" | "all" (default: "all")
- `category` (optional) — filter by product category
- `limit` (optional, default: 20) — batch size
- `cursor` (optional) — pagination cursor (offset or last-seen product_id)

**Server-side logic:**
1. Look up user's taste vector for the selected occasion (or primary taste vector if "all")
2. If taste vector exists → query products by style_vector similarity (67-dim cosine distance), exclude already-interacted products
3. If no taste vector (cold start) → return trending products (most-saved/interacted in last 7 days)
4. Compute `matchScore` (0–100) from vector distance for each result
5. Return products in ranked order with pagination cursor

**Response shape** (must satisfy frontend `Product` type):
```json
{
  "products": [
    {
      "id": 42,
      "name": "Oversized Wool Blazer",
      "brand": "COS",
      "retailer": "COS",
      "retailerUrl": "https://cos.com/...",
      "price": 250.00,
      "originalPrice": 350.00,
      "currency": "USD",
      "images": ["https://..."],
      "category": "Outerwear",
      "subcategory": "Blazers",
      "sizes": [{"label": "XS", "available": true}, ...],
      "colors": ["Black", "Charcoal"],
      "matchScore": 93,
      "tags": ["minimal", "tailoring"],
      "occasion": "work",
      "isNew": true,
      "isSale": true
    }
  ],
  "cursor": "next_page_token",
  "isPersonalised": true
}
```

**Open questions:**
- ~~Where do `name`, `subcategory`, `sizes`, `colors`, `originalPrice`, and `retailerUrl` come from?~~ **Largely resolved by Shopify migration (April 2026).** `name` → `shopify_title`, `originalPrice` → `shopify_compare_at_price`, `subcategory` → dedicated column, `sizes`/`colors` → extractable from `shopify_variants` JSONB, `retailerUrl` → `url`, `currency` → `shopify_currency`. Remaining work: API serialisation layer to transform DB columns into the frontend response shape (especially variant → sizes/colors transformation). See updated Section 5.
- Should matchScore be computed in the API layer or in a pre-computed column? For PoC, compute at query time. For scale, pre-compute nightly.

### 3.2 POST /interactions

**Purpose:** Log user interactions with products (signals for the recommendation engine).

**Request body:**
```json
{
  "user_id": 1,
  "interactions": [
    {
      "product_id": 42,
      "type": "like",
      "occasion": "work",
      "timestamp": "2026-04-11T10:30:00Z"
    },
    {
      "product_id": 43,
      "type": "dwell",
      "occasion": "work",
      "dwell_seconds": 6.2,
      "timestamp": "2026-04-11T10:30:05Z"
    }
  ]
}
```

**Supported interaction types and weights** (from ARCHITECTURE.md):

| Type | Weight | Notes |
|------|--------|-------|
| like | +0.8 | Heart icon tap |
| dismiss | −0.3 | Swipe left |
| save | +1.0 | Bookmark icon tap (also writes to wishlist) |
| tap | +0.6 | Product card tap (navigated to PDP) |
| dwell | +0.15 or +0.4 | 2–5s = 0.15, 5+s = 0.4 (determined by dwell_seconds) |

**Batch support:** The frontend should batch interactions and send periodically (e.g. every 10 interactions or every 30 seconds) rather than per-action, to reduce API calls.

**Undo support:** For dismiss interactions, the frontend sends a follow-up interaction with `type: "undo_dismiss"` within the 4-second undo window. The backend should soft-delete the original dismiss.

### 3.3 POST /interactions/undo

**Purpose:** Undo a dismiss within the 4-second window. The backend finds the most recent dismiss for that user + product and sets `is_undone = True`. The 4-second window is enforced client-side.

### 3.4 POST /wishlist

**Purpose:** Add a product to the user's wishlist (also logged as a "save" interaction).

**Request body:**
```json
{
  "user_id": 1,
  "product_id": 42,
  "price_at_add": 250.00
}
```

### 3.5 DELETE /wishlist/{product_id}

**Purpose:** Remove a product from the user's wishlist.

### 3.6 GET /wishlist

**Purpose:** Return the user's saved products with current prices (for price drop detection).

**Query parameters:**
- `user_id` (required)

**Response includes `price_at_add` so the frontend can detect price drops.**

### 3.7 GET /feed/trending (internal / optional)

**Purpose:** Return trending products (most-saved in last 7 days) as a cold-start fallback.

Could be a standalone endpoint or an internal function called by GET /feed when no taste vector exists.

---

## 4. Required New Database Tables

### 4.1 user_interactions

Captures all feed signals. Replaces `swipe_history` (being dropped — see Section 7).

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| interaction_id | SERIAL | PRIMARY KEY | |
| user_id | INTEGER | FK → users, NOT NULL | |
| product_id | INTEGER | FK → products, NOT NULL | |
| type | VARCHAR(20) | NOT NULL, CHECK | "like", "dismiss", "save", "tap", "dwell" |
| weight | FLOAT | NOT NULL | Signal weight (from table above) |
| occasion | VARCHAR(20) | | Active occasion filter at time of interaction |
| dwell_seconds | FLOAT | | Only for type="dwell" |
| is_undone | BOOLEAN | DEFAULT false | True if undo_dismiss was called |
| created_at | TIMESTAMP | DEFAULT now() | |

**Indexes:**
- B-tree on (user_id, product_id) — for exclusion queries ("already interacted")
- B-tree on (user_id, type) — for "get all likes" queries
- B-tree DESC on created_at — for recent interaction queries

### 4.2 user_saved_items (wishlist)

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| saved_item_id | SERIAL | PRIMARY KEY | |
| user_id | INTEGER | FK → users, NOT NULL | |
| product_id | INTEGER | FK → products, NOT NULL | |
| price_at_add | DECIMAL(10,2) | NOT NULL | Price when saved (for price drop detection) |
| saved_at | TIMESTAMP | DEFAULT now() | |

**Constraints:**
- Unique on (user_id, product_id)
- FK cascade delete on user and product

### 4.3 user_taste_vectors

Stores computed taste vectors per user per occasion.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| taste_vector_id | SERIAL | PRIMARY KEY | |
| user_id | INTEGER | FK → users, NOT NULL | |
| occasion | VARCHAR(20) | NOT NULL | "everyday", "going_out", "work", "special", "primary" |
| taste_vector | vector(67) | | 67-dim style taste vector |
| interaction_count | INTEGER | DEFAULT 0 | Interactions informing this vector |
| seeded_from | VARCHAR(50) | | "onboarding", "interactions", "brand_prior" |
| updated_at | TIMESTAMP | DEFAULT now() | |

**Constraints:**
- Unique on (user_id, occasion)

**Taste vector recalculation:** Computed inline after each interaction (decision #3). Formula: `0.7 × existing + 0.3 × interaction_signal`. The `POST /interactions` handler triggers this before returning. Initial seeding comes from onboarding brand selections (average of selected brands' style vectors).

### ~~4.4 editorial_picks~~ (deferred)

Not needed for PoC. Cold-start feed uses trending products as the sole fallback. May be revisited later if a curated showcase for new users is needed.

---

## 5. Products Table — Field Mapping (Updated April 2026)

The Shopify migration added most of the previously missing columns. The table below shows the current status of each frontend field.

| Frontend field | Backend source | Status | Remaining work |
|----------------|---------------|--------|----------------|
| name (product title) | `shopify_title` | **✅ Available** | Map in API serialiser |
| subcategory | `subcategory` column | **✅ Available** | Populated from `shopify_product_type` at ingestion |
| sizes | `sizes` JSONB column | **✅ Available** | Denormalised at ingestion from shopify_variants. API serialiser maps to `[{label, available}]` (v0: product-level availability). |
| colors | `colors` TEXT[] column | **✅ Available** | Denormalised at ingestion from shopify_variants. Direct mapping to `string[]`. |
| originalPrice | `shopify_compare_at_price` | **✅ Available** | Map in API serialiser; NULL when not on sale |
| retailerUrl | `url` | **✅ Available** | Direct mapping |
| currency | `shopify_currency` | **✅ Available** | Direct mapping (no longer needs hardcoding) |
| description | `shopify_description` | **✅ Available** | Plain text, no HTML |
| details | — | **❌ Still missing** | Not in Shopify Storefront API (tokenless). Options: (a) parse from description, (b) extract via enrichment pipeline, (c) drop from v0 |
| occasion | style_attributes JSONB | **⚠️ Derivable** | Map from style_attributes at query time (requires enrichment pipeline to have run) |
| isNew | `created_at` or `shopify_created_at` | **✅ Derivable** | Compute in API: created < 7 days ago |
| isSale | `shopify_compare_at_price` > `price` | **✅ Derivable** | Compute in API serialiser |

**All backend work is now complete.** The API serialisation layer (`api/serialisers.py`) transforms DB columns into the frontend response shape. Sizes and colors are denormalised at ingestion time into dedicated columns (`sizes` JSONB, `colors` TEXT[]) — no per-request extraction from `shopify_variants` needed. The only truly missing field is `details` (product detail bullets), which maps to `shopify_description` (plain text) for v0.

---

## 6. Data Shape Alignment

The frontend uses string IDs ("p1", "p2") while the backend uses integer IDs. Two options:

**Decision (April 2026): Frontend adapts to integer IDs.** Change `id` type from `string` to `number` in `src/lib/types.ts`, update mock data IDs in `src/lib/mock-data.ts`, and find-and-replace any string ID comparisons across feed components, context/store, and feed-utils. These changes happen naturally when wiring the feed to the real API.

---

## 7. Relationship to Existing Tables

### swipe_history

The existing `swipe_history` table captures like/dislike per profile. The new `user_interactions` table is broader (more signal types, occasion context, dwell time, undo support). Options:

**Option A:** Keep both tables. `swipe_history` is the source of truth for the swipe mechanic (P1 enhancement); `user_interactions` captures all feed signals. Writes to both on like/dislike.

**Option B:** Deprecate `swipe_history` in favour of `user_interactions`. Migrate existing swipe data. Simpler long-term, but breaks the existing swipe endpoints.

**Decision (April 2026): Option B.** `swipe_history` contains no data and has never been used in production. Drop the table, model, CRUD functions, and endpoints entirely. The P1 swipe onboarding mechanic can use `user_interactions` with `type='like'`/`type='dismiss'`. See `Backend/TODO.md` Section 14f for the cleanup checklist.

### style_preferences

The existing `style_preferences` table stores a 768-dim preference vector per profile per product type. The feed uses a different approach — 67-dim taste vectors per occasion (not per product type). These serve different purposes and can coexist. `style_preferences` may still be useful for the future matching algorithm.

---

## 8. Recommended Build Sequence

Each phase is independently testable. Earlier phases don't depend on later ones.

### Phase 1: Schema + Products Table Updates
1. ~~Add missing columns to products table~~ **Done (April 2026).** Shopify migration added shopify_title, shopify_description, subcategory, shopify_compare_at_price, shopify_variants, shopify_currency + brands table. See `scripts/migrate_shopify_schema.sql`.
2. Create `user_interactions` table
3. Create `user_saved_items` table
4. Create `user_taste_vectors` table
5. ~~Update `models.py` with new tables and columns~~ **Done.** Brand model + all Shopify/STROBE columns added.
6. ~~Update scraper/ingestion to populate new product fields~~ **Done.** Shopify tokenless Storefront API ingestion via `upsert_shopify_products()` populates all new fields.
7. **New:** Build API serialisation layer to transform DB columns → frontend Product shape (especially shopify_variants → sizes/colors)

### Phase 2: Interaction Logging
1. Build POST /interactions endpoint (batch)
2. Build undo_dismiss logic
3. Build POST /wishlist and DELETE /wishlist/{product_id} and GET /wishlist endpoints
4. Wire frontend to send interactions to API (replace localStorage-only flow)

### Phase 3: Feed Endpoint (Cold Start)
1. Build GET /feed with cold-start path only (trending fallback)
2. Build trending query (most-saved/interacted in last 7 days from user_interactions + user_saved_items)
4. Wire frontend to call GET /feed instead of loading mock data
5. Align frontend Product type with API response shape (integer IDs, field mapping)

### Phase 4: Feed Endpoint (Personalised)
1. Implement taste vector seeding (from onboarding brand selections → average style vectors)
2. Build personalised feed path in GET /feed (style_vector similarity query + exclusions + matchScore computation)
3. Implement per-interaction taste vector recalculation (0.7 × existing + 0.3 × interaction_signal, inline in POST /interactions)
4. Cold-to-personalised graduation (5+ meaningful interactions triggers switch)

### Phase 5: Polish
1. Occasion-specific sub-vectors (interactions update the right occasion vector)
2. Price drop detection on wishlisted items
3. Performance tuning (query optimisation, index tuning, response caching)
4. Error handling alignment (frontend error states ↔ API error responses)

---

## 9. Open Decisions

| # | Decision | Options | Impact | Owner |
|---|----------|---------|--------|-------|
| 1 | ~~Where do product name, sizes, description come from?~~ | **Resolved (April 2026).** Shopify migration added `shopify_title`, `shopify_description`, `shopify_compare_at_price`, `shopify_variants`, `shopify_currency`, `subcategory`. Sizes/colors derivable from `shopify_variants` JSONB. Only `details` (bullet points) remains unresolved. | No longer blocks Phase 1 | Josh |
| 2 | ~~Frontend ID type — string or integer?~~ | **Resolved (April 2026). Option A — frontend adapts to integer IDs.** Backend uses integer PKs everywhere; no reason to add string wrapping. Frontend changes needed: update `id` type from `string` to `number` in `src/lib/types.ts`, update mock data IDs in `src/lib/mock-data.ts`, and find-and-replace any string ID comparisons across feed components, context/store, and feed-utils. These changes happen naturally when wiring the feed to the real API. | No longer blocking | Josh |
| 3 | ~~Taste vector computation — real-time or batch?~~ | **Resolved (April 2026). Option A — real-time, after each interaction.** The calculation is trivially cheap (weighted average of 67-dim vectors, no GPU/API calls). Recomputing after every interaction rather than per-session means the feed improves within seconds of a like/dismiss, which is important for demonstrating the product's value prop early. Implementation: `POST /interactions` handler triggers taste vector recalculation inline before returning. Formula: `0.7 × existing + 0.3 × session_signal`. | No longer blocking | Josh |
| 4 | ~~Auth strategy — how does the API know who the user is?~~ | **Resolved for PoC (April 2026). Option C — simple user_id param.** All endpoints accept `user_id` as a parameter while building and testing the feed/interaction system. Real auth (NextAuth.js vs Clerk — TBD) to be implemented before v0 launch. When auth is added: replace `user_id` params with session-derived user identity, add auth middleware, and protect all user-scoped endpoints. | No longer blocks PoC; revisit before v0 | Josh |
| 5 | ~~Editorial picks — table or config?~~ | **Deferred (April 2026).** Not needed for PoC. Cold-start feed will rely on trending (most-saved/interacted products) as the sole fallback. Editorial picks may be revisited later if a curated showcase is needed for new users. | Not blocking | Josh |
| 6 | ~~Keep or deprecate swipe_history?~~ | **Resolved (April 2026). Option B — drop swipe_history.** Table has no data. `user_interactions` replaces it entirely. See TODO 14f for cleanup list. | No longer blocking | Josh |
| 7 | ~~Interaction batching strategy~~ | **Resolved (April 2026). Option A — frontend batches + periodic flush.** Flush every 10 interactions or 30 seconds, whichever comes first. Reduces API load while keeping taste vector updates near-real-time (max 30s lag). Backend `POST /interactions` already accepts a batch. Scheme to be reconsidered once real usage patterns are observed. | No longer blocking | Josh |

---

*Feed API Scope · v3 · April 2026 · Confidential*
*v2 changes: Updated sections 2.1, 2.2, 3.1, 5, 8, and 9 to reflect Shopify schema migration (brands table, Shopify product columns, ingestion pipeline).*
*v3 changes: All 7 open decisions resolved/deferred. Tidied sections 2.2, 3.1, 3.3, 4.3, 4.4, 6, 7, 8 to match decisions. Dropped swipe_history, deferred editorial picks, settled on POST /interactions/undo, integer IDs, per-interaction taste vector recalc, user_id param auth for PoC, batch flush every 10/30s.*
