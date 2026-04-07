# Search — Functional Spec

> Part of STROBE · April 2026

---

## 1. Overview

Search is the secondary product discovery surface, enabling users to find apparel through natural language queries. Powered by Marqo vector embeddings (semantic search) in Phase 3 onwards, it replaces traditional text-based search. Users can search by style intent ("oversized blazers for the office"), colour, brand, occasion, or aesthetic, and refine results with multi-dimensional filters.

**Personalised search (Phase 3+):** Search results are not purely relevance-ranked — they reflect the user's fingerprint. Certain filters are pre-applied based on the user's profile (gender expression, preferred sizes, price range from onboarding data), so results are immediately relevant without manual filtering. Beyond pre-filtering, results are ordered by likelihood the user will like the product, blending Marqo relevance score with taste vector similarity. This means two users searching the same query see different result orderings.

---

## 2. Search Modes

### Phase 2 (Temporary PoC)

**Search algorithm:** ILIKE text search on product title and description columns in Neon.

- User types query ≥2 characters
- SQL ILIKE query: `WHERE title ILIKE '%query%' OR description ILIKE '%query%'`
- Results ranked by title match priority, then description match
- Deterministic, fast; does not leverage visual embeddings

**Status:** Temporary placeholder. Used during early PoC before Marqo integration.

### Phase 3+ (Target Architecture)

**Search algorithm:** Natural language vector search via Marqo.

- User types query ≥2 characters
- Query is converted to vector embedding (same model as product embeddings: Marqo-FashionSigLIP, 768-dim)
- Marqo returns ranked product IDs by cosine similarity to query vector
- Top results (up to 100) are fetched from Neon for detail rendering
- Results ranked by relevance score (0–1.0)

**Advantages:** Semantic understanding of style intent, colour, aesthetic, occasion — not just keyword matching. Examples: "something trendy for going out" or "sustainable earth tones" both work naturally.

---

## 3. Functional Requirements

### SR-001 — Pre-Search State

Before the user types, the search page displays:

- **Recent searches:** List of user's past 5 searches (stored in localStorage + server, persistable across sessions)
  - Each entry: query text + clear icon (removes that query)
  - "Clear All" link at bottom (clears entire recent list)
- **Trending on STROBE:** Pill-grid of ~10 trending search terms (most-searched queries in last 7 days across all users, refreshed daily)
  - Tapping a trending item populates query field and triggers search

### SR-002 — Search Entry

Search bar is fixed at top of page. User can:
- Type query (≥2 characters to trigger search)
- Clear query with X button
- Submit search with return key (or auto-submit after 500ms of no typing — debounce)

Placeholder text: "Find your style..."

### SR-003 — Live Results

Once query reaches 2 characters:
- Results render in real-time (debounced 500ms after last keystroke)
- Skeleton cards appear while results load (max 1.5s wait)
- Result count shown: "[n] results found" (or "0 results" if empty)
- 2-column product grid with stagger animation on render

**Phase 2:** Text search results ranked by title match.
**Phase 3+:** Vector search results ranked by Marqo relevance score.

### SR-004 — Filters

Filter panel accessible via filter icon (button with funnel icon, badge showing active filter count).

**Available filters:**

| Filter | Type | Options | UI |
|--------|------|---------|-----|
| Category | Multi-select | Dresses, Tops, Bottoms, Outerwear, Activewear, Accessories | Horizontal chips |
| Brand | Multi-select | ~50 brands (searchable dropdown) | Searchable chips |
| Occasion | Multi-select | Everyday, Going Out, Work, Special Events | Chips |
| Aesthetic | Multi-select | (from product_tags: minimalist, bohemian, maximalist, vintage, futuristic, etc.) | Chips |
| Price Range | Slider | $0–$2000 (dual slider, min–max) | Range input |
| Sale Only | Toggle | Yes / No | Toggle switch |
| Colour | Multi-select | (from product_tags: earth tones, pastels, jewel tones, neutrals, monochrome, bright) | Colour swatches + label |

**Behaviour:**
- Multiple filters apply with AND logic (e.g. "Category=Dresses AND Price=$50–$200 AND Aesthetic=Minimalist")
- Active filter chips displayed below search bar; each dismissible individually
- "Clear All Filters" link resets all filters
- Applying filters re-runs search with same query; filters persist until cleared

### SR-005 — Sort Options

Sort dropdown (default: Best Match). Options:

| Sort | Phase 2 | Phase 3+ |
|------|---------|---------|
| Best Match (default) | Title relevance + newest first | Marqo relevance × taste vector similarity (personalised ranking per user fingerprint) |
| New Arrivals | Ingestion date DESC | Ingestion date DESC |
| Price: Low to High | Price ASC | Price ASC |
| Price: High to Low | Price DESC | Price DESC |
| Most Saved | user_saved_items count DESC | user_saved_items count DESC |

### SR-006 — Results Display

**Result card (2-column grid):**
- Product image (square, 1:1)
- Brand name
- Product title (2 lines, truncated)
- Price

> **Internal metric:** Marqo relevance scores and taste vector match scores are used for backend ranking and analytics only. They are not displayed to users.

**Empty state:**
- Search icon + "No Results" heading + copy: "Try different keywords or adjust your filters."
- "Clear All" link to reset filters
- Trending pill-grid below to suggest alternative searches

### SR-007 — Interaction Signals

All interactions on search results are captured and attributed to "search" source:

| Interaction | Capture | Impact |
|-------------|---------|--------|
| Search query submitted | Logged to analytics: (query_text, result_count, filters_applied, timestamp) | Feeds search analytics; contributes to trending queries |
| Product like | Logged to user_interactions (source: "search") | Feeds taste vector |
| Product save/wishlist | Logged to user_interactions (source: "search") | Feeds taste vector + wishlist |
| Product detail tap | Logged to user_interactions (source: "search", product_id) | Feeds taste vector + product popularity |
| Product dwell | IntersectionObserver; logged same as feed | Feeds taste vector |

### SR-008 — Navigation

**User flow:**
1. User enters query ≥2 characters
2. Results render with sort + filter options
3. User may filter or sort results
4. User taps product card → `/product/[id]`
5. On return to search, query/filters/sort preserved; user continues browsing

**Clear search:**
- User taps X button in search bar
- Results cleared; pre-search state (recent + trending) restored

---

## 4. User Flow (Happy Path)

```
/search → [pre-search state] → type query → [live results] → filter/sort → [refined results] → tap product → /product/[id]
```

1. User taps Search tab (bottom nav) or searches from elsewhere
2. Search page loads showing recent searches + trending pills
3. User types "sustainable denim" (or taps trending item)
4. Live results appear (Phase 3+: semantic search for sustainable denim products)
5. 47 results shown with sort options + filter panel
6. User taps filter icon, selects "Price: $50–$150" + "Aesthetic: Minimalist"
7. Results re-filter to 12 matching products
8. User taps product card → product detail page
9. User returns to search; query, filters, sort preserved

---

## 5. Error States

| Error | Scenario | Display | Recovery |
|-------|----------|---------|----------|
| No results | Query returns 0 products | Empty state: "No Results" + "Try different keywords or adjust your filters." | User adjusts query or clears filters |
| Search timeout | Marqo query ≥3 seconds | Skeleton cards persist; retry message after 3s: "Search took too long. Tap to retry." | Retry button re-submits query |
| Filter error | API error applying filters | Toast: "Couldn't apply filters. Try again." Filters revert to previous state. | User retries filter |
| Network error | Connection lost during search | Toast: "Something went wrong. Check your connection." Results remain cached on screen. | User retries or clears query |

---

## 6. Edge Cases

| Case | Behaviour |
|------|-----------|
| User clears query | Results cleared; pre-search state returns (recent + trending) |
| User filters with zero results | Empty state shown; "Clear All" link removes filters |
| User switches sort during filtered results | Sort applied to current filtered set; pagination reset to page 1 |
| User searches for brand name | Phase 2: text search finds brand. Phase 3+: vector search understands intent, returns products from that brand |
| User searches with special characters | Sanitized by frontend; sent to backend for safe processing |
| Recent searches exceed 5 items | Oldest search removed; new search added (FIFO queue) |
| Trending queries updated daily | Daily job refreshes trending based on 7-day search volume. Existing results stay; next page load shows new trending. |
| Search bar loses focus (mobile) | Keyboard dismisses; search bar remains sticky at top. Results visible below. |

---

## 7. Metrics & Instrumentation

| Metric | Capture | Purpose |
|--------|---------|---------|
| Search volume | Count of search_query events | Feature engagement |
| Avg result count | Mean results per query | Catalogue coverage |
| Search-to-click rate | (product taps / searches) | Search quality target: ≥25% |
| Filter adoption | Filter_applied events / searches | Filter utility validation |
| Recent search click-through | Clicks on recent searches / displayed | User convenience |
| Trending search engagement | Clicks on trending pills / searches | Trending relevance |
| Bounce rate | Sessions with search but no product tap | Search quality proxy |
| Query similarity | Detect duplicate/similar queries | Autocomplete training |
| Phase 3 vector quality | Click-through rate post-vector search | Vector model performance |

---

## 8. Acceptance Criteria

| AC ID | Criterion | Pass Condition |
|-------|-----------|----------------|
| AC-SR-001 | Search returns results within 1.5s on 4G | 95th percentile latency ≤1500ms |
| AC-SR-002 | Pre-search state loads instantly | Recent searches + trending render within 500ms |
| AC-SR-003 | Filters apply correctly (AND logic) | Multi-filter results are intersection of all selected filters |
| AC-SR-004 | Search-to-click rate ≥25% | (Product detail taps / total searches) ≥0.25 |
| AC-SR-005 | Recent searches persist across sessions | Reopening /search shows previous 5 queries |
| AC-SR-006 | Trending queries refresh daily | Trending pills updated 24h apart; stale queries removed |
| AC-SR-007 | Phase 3: Vector search quality | Manual audit of 50 queries confirms semantic relevance (TBD by cofounder) |
| AC-SR-008 | Filter state preserved on product return | User returns from /product/[id] to search; filters/sort unchanged |

---

## 9. Phase 2 → Phase 3 Migration

**Transition plan (Timeline: TBD by cofounder)**

1. **Phase 2 (now):** Text search (ILIKE) on title/description. Fast, deterministic fallback.
2. **Phase 3 (TBD):** Marqo vector integration. User query → embedding → Marqo search → results. Product vectors (768-dim Marqo-FashionSigLIP) are used for intra-category similarity matching. Results personalised via user fingerprint pre-filtering and taste vector re-ranking.
3. **Cutover:** Toggle feature flag `USE_VECTOR_SEARCH` in backend config.
4. **Rollback:** If vector quality poor, revert to text search (flag false); text search remains in codebase as fallback.

**Monitoring:** Track search-to-click rate (target ≥25%) for 2 weeks post-Phase 3 launch. If drops below 20%, rollback to text search and investigate vector model quality.

---

*STROBE Search Spec · v1 · April 2026 · Confidential*
