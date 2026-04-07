# Product Detail — Functional Spec

> Part of STROBE · April 2026

---

## 1. Overview

The Product Detail Page (PDP) displays the full product information for a single item, including image gallery, pricing, size/variant selector, and call-to-action buttons. The page logs interaction signals (page tap, dwell time on exit) that feed the recommendation engine. Users can add items to their bag, save to wishlist, or view related recommendations without leaving the page.

---

## 2. Functional Requirements

### FR-PD-001 — Image Gallery & Carousel
Product detail page displays all product images in a horizontal carousel with dot indicators. Dots are tappable to navigate directly to a specific image. Images load directly from retailer CDNs (Shopify in v0). Placeholder (STROBE logo on grey background) shown if image fails to load.

| UI State | Description |
|----------|-------------|
| Default | First image displayed with dot indicators |
| Swiping | Carousel animates to next/previous image |
| Placeholder | Grey background with STROBE logo on image load failure |

### FR-PD-002 — Product Information Display
Page displays: brand name (tappable link to brand filter on search results), full product title (no truncation), current price, original price (if sale), sale percentage badge, description text, and product details (accordion expand/collapse).

| Element | Requirement |
|---------|-------------|
| Brand Name | Tappable; navigates to `/search` with brand filter pre-applied |
| Product Title | Full title displayed, no truncation |
| Price Display | Sale price in danger red, original crossed out, % off badge shown |
| Description | Full text, scrollable |
| Details Accordion | Expandable section for material, care, dimensions, fit |

### FR-PD-003 — Badge Display
Product detail page displays contextual badges above the title:
- **New:** Product ingested within 48 hours
- **Sale:** Product has a sale_price lower than standard price

> **Internal metric:** Taste vector match scores (how well a product aligns with the user's style) are used internally for personalization ranking and analytics. They are not displayed to users.

### FR-PD-004 — Size/Variant Selector
Size/variant selector displays all available sizes as buttons. Out-of-stock sizes are greyed with strikethrough and cannot be selected. Selected size is highlighted with a distinct visual state (outline or filled). Selecting the same size twice deselects it. Size selection is required before "Add to Bag" is enabled; attempting to add without a size triggers a shake animation and error toast.

| State | Appearance | Behaviour |
|-------|-----------|-----------|
| Available | Clickable button, default styling | Tap to select |
| Out of Stock | Greyed, strikethrough, disabled cursor | Unclickable; no selection possible |
| Selected | Highlighted outline/fill | Tap to deselect (toggle) |
| Attempting Add without Size | Button shakes; error toast appears | User must select size |

### FR-PD-005 — Wishlist & Share Actions
Wishlist action (bookmark icon) is a toggle button displayed in the sticky bottom bar. Tapping adds the product to wishlist without requiring a size. If product is already wishlisted, tapping removes it. Toast confirmation appears: "Saved to wishlist" or "Removed from wishlist". Share action (share icon) copies product URL to clipboard with confirmation: "Link copied to clipboard".

| Action | Effect | Confirmation |
|--------|--------|--------------|
| Add to Wishlist | Toggles wishlisted state | Toast: "Saved to wishlist" |
| Remove from Wishlist | Toggles wishlisted state | Toast: "Removed from wishlist" |
| Share | Copies URL to clipboard | Toast: "Link copied to clipboard" |

### FR-PD-006 — Add to Bag
Tapping "Add to Bag" requires a size to be selected first. If size is selected, the system calls the Shopify tokenless cart API for the product's brand store, creating or updating a cart object. The returned checkoutUrl is stored. Confirmation toast appears: "Added to bag". The user remains on the product detail page. The bag icon in the header updates to show current item count.

| Precondition | Action | Result |
|--------------|--------|--------|
| Size not selected | Tap Add to Bag | Button shakes, error toast: "Please select a size" |
| Size selected | Tap Add to Bag | Shopify cart API called; item added; toast: "Added to bag"; bag icon count increments; user stays on page |
| Shopify API fails | Tap Add to Bag | Toast: "Couldn't add to bag. Please try again." User can retry. |
| Variant out of stock | Size unavailable | Size greyed out, cannot be selected; cannot add |

### FR-PD-007 — Out of Stock Handling
If a product is entirely out of stock (all sizes unavailable), the "Add to Bag" button is replaced with "Notify Me When Back In Stock". Tapping this button opens a restock notification signup (or toggles subscription state if user is already subscribed). Confirmation toast: "We'll notify you when this comes back."

| Scenario | Display | Action |
|----------|---------|--------|
| All sizes in stock | "Add to Bag - $[price]" button active | User can select size and add |
| Some sizes in stock | Add to Bag active; OOS sizes greyed | User selects available size and adds |
| All sizes out of stock | "Notify Me When Back In Stock" button | Tap to subscribe to restock alert |

### FR-PD-008 — Interaction Logging
Navigating to a product detail page logs a `tap` interaction to `user_interactions` table with type `tap`, source (feed/search/wishlist/recommendation), and product_id. Time spent on the page is tracked via a timer set on page load. On exit (back navigation, navigation to another page, or tab close), the elapsed time is logged as `dwell_seconds` in a second `user_interactions` record with type `dwell`.

| Signal | Trigger | Logged To | Fields |
|--------|---------|-----------|--------|
| Product Tap | Navigate to `/product/[id]` | user_interactions | product_id, source, type: tap |
| Dwell Time | Exit product detail page | user_interactions | product_id, dwell_seconds, type: dwell |

### FR-PD-009 — Related Recommendations Section
Product detail page displays two recommendation sections:
1. **"You Might Also Like"** — 4 related products from Marqo nearest-neighbour search (same vector space)
2. **"From [Brand]"** — up to 8 other products from the same brand

Each recommendation is clickable and navigates to the product detail page for that product.

### FR-PD-010 — Tags Display
Below the product details accordion, a row of product tags is displayed. Tags are sourced from the `product_tags` table (type, tag_value, confidence_score). Only tags with confidence_score > 0.5 are shown. Tags are read-only; user cannot interact with them on this page.

---

## 3. User Flow

**Happy Path:**
```
User browses /feed or /search
  → Taps product card
  → Navigates to /product/[id]
  → Scrolls to view details, images, recommendations
  → Selects size from selector
  → Taps "Add to Bag"
  → Stays on product detail page
  → Bag icon updates with item count
  → (Optional) Taps wishlist to save
  → (Optional) Taps back or navigates to /bag
```

**Alternative Flow (Out of Stock):**
```
User navigates to /product/[id]
  → Product is entirely out of stock
  → "Notify Me When Back In Stock" displayed instead of Add to Bag
  → Taps to subscribe to restock notification
  → Receives notification when product is back in stock
```

---

## 4. Error States & Recovery

| Error | Display | Recovery |
|-------|---------|----------|
| Product not found | Centered message: "Product not found" | Back navigation returns to previous page |
| Add to Bag fails (Shopify API error) | Toast: "Couldn't add to bag. Please try again." | User can retry |
| Variant no longer available (after page load) | Size greyed out; toast: "This size is no longer available." | User selects alternative size |
| Product images fail to load | Grey placeholder with STROBE logo in carousel slot | Placeholder persists; page remains functional |
| Network error (dwell time logging) | Interaction logging fails silently | No user-facing error; subsequent interactions logged normally |

---

## 5. Data Model

For complete schema details, see **Backend/api/README.md**. Key fields:

| Table | Key Columns | Usage |
|-------|-------------|-------|
| `products` | id, brand_id, title, description, price, sale_price, primary_image_url, image_urls (array), created_at, updated_at | Product core data displayed on PDP |
| `product_tags` | id, product_id, tag_type, tag_value, confidence_score | Tags displayed on PDP (confidence > 0.5) |
| `product_embeddings` | product_id, marqo_document_id, model_version | Used for "You Might Also Like" nearest-neighbour queries |
| `brands` | id, name, logo_url | Brand name and logo display on PDP |
| `user_interactions` | id, user_id, product_id, interaction_type, source, dwell_seconds, created_at | Logs tap and dwell signals for personalisation engine |
| `carts` | id, user_id, brand_id, checkout_url, status, created_at, expires_at | Stores Shopify cart objects and checkout URLs for multi-brand bag |

---

## 6. Acceptance Criteria

| AC ID | Criterion | Test Type | Pass Condition |
|-------|-----------|-----------|----------------|
| AC-PD-001 | Product detail page loads for valid product ID | Functional | Page renders within 2 seconds with all required elements (images, title, price, size selector, Add to Bag) |
| AC-PD-002 | Size selection is required before Add to Bag | Functional | Tapping Add to Bag without size selection triggers shake animation and error toast |
| AC-PD-003 | Add to Bag calls Shopify cart API and updates bag icon | Integration | Shopify cart API returns valid cart; bag icon count increments by 1 |
| AC-PD-004 | Dwell time is logged on page exit | Integration | user_interactions table contains dwell record with correct dwell_seconds and product_id |
| AC-PD-005 | Out-of-stock variants cannot be selected | Functional | Out-of-stock sizes are greyed and unclickable; "Notify Me When Back In Stock" shown if all sizes OOS |
| AC-PD-006 | Product not found returns 404 state | Functional | Invalid product ID shows centered "Product not found" message |
| AC-PD-007 | Related recommendations render correctly | Functional | "You Might Also Like" and "From [Brand]" sections display 4 and 8 items respectively (or fewer if insufficient products) |
| AC-PD-008 | Wishlist toggle works without size selection | Functional | User can save product to wishlist without selecting a size; product appears in `/wishlist` without size |

---

*Product Detail — STROBE v2 · April 2026 · Functional Specification*
