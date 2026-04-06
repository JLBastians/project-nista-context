# Wishlist — Functional Spec

> Part of STROBE · April 2026

---

## 1. Overview

The Wishlist is a persistent collection of saved products accessible via the Wishlist tab in bottom navigation. Users bookmark products from feed, search, or product detail pages. The Wishlist displays items in a 2-column grid with sorting and filtering options. Price change detection alerts users when wishlisted items drop ≥10% from the price at time of save.

---

## 2. Functional Requirements

### FR-WL-001 — Save/Unsave Toggle

Bookmark icon on product cards (feed, search) and product detail page. Tap toggles save state.

- **Unsaved → Saved:** Icon fills with electric blue; toast "Saved to wishlist"
- **Saved → Unsaved:** Icon returns to outline; toast "Removed from wishlist"
- **Interaction signal:** Logged to `user_interactions` (type: save, weight 1.0) and `user_saved_items` table

---

### FR-WL-002 — Wishlist Page

**Header:** "Wishlist" title + item count

**Sorting:** Dropdown with options:
- Date Added (default — reverse chronological)
- Price: Low to High
- Price: High to Low

**Filtering:** Chip-based filters by brand and category

**Layout:** 2-column product grid. Each card displays:
- Product image
- Brand name
- Product name
- Current price
- Original price (if changed since save)
- Selected size (if saved with size)
- Remove (×) button

**Size selection persistence:** If user selected a size before saving, size is shown as a badge on the card. If no size was selected, a "Select Size" chip appears; tapping opens an inline size picker.

**Empty state:** Heart icon + "Your Wishlist is Empty" + "Browse Feed" CTA

---

### FR-WL-003 — Price Change Indicator

On render, compare current `product.price` against `price_at_add` stored in `user_saved_items` table.

- **Price drop ≥10%:** Display green "Price Dropped" badge overlaid on product image with old price crossed out and new price highlighted
- **Price unchanged or increased:** No badge displayed

This is a non-intrusive visual signal; no notification is sent (price drop notifications are P1, post-MVP).

---

### FR-WL-004 — Share Wishlist (P2)

Top-right "Share" button generates a shareable link: `strobe.com/wishlist/[user-token]`. Tapping the link opens a read-only public view of the wishlist. Token is regenerable and invalidates the previous link. Confirmation toast: "Link copied to clipboard".

---

### FR-WL-005 — Remove from Wishlist

Tapping × removes item from `user_saved_items` table and from display. Optimistic UI with undo snackbar (4 seconds). Logged to `user_interactions` (type: remove).

---

## 3. Interaction Signal

| Signal | Type | Weight | Logged To |
|--------|------|--------|-----------|
| Save | Explicit | 1.0 | `user_interactions` + `user_saved_items` |
| Remove | Explicit | — | `user_interactions` |

---

## 4. Data Model

**user_saved_items table:**

| Column | Type | Notes |
|--------|------|-------|
| id | uuid PK | |
| user_id | uuid FK | |
| product_id | uuid FK | |
| price_at_add | decimal | Price when product was saved |
| size_at_add | text (nullable) | Size if selected before saving |
| saved_at | timestamp | When product was added to wishlist |

**product_embeddings / product_tags:** Joined for display details and filtering.

For full schema, see `Backend/api/README.md`.

---

## 5. Error States

| Error | Display | Recovery |
|-------|---------|----------|
| Product no longer in catalogue | Product silently omitted from wishlist render | Remove from list |
| Save fails (optimistic UI revert) | Item reverts to unsaved state; toast: "Couldn't save. Try again." | Retry |
| Product out of stock | Item still visible (no removal) but size unavailable on PDP | Navigate to PDP; size shown as out-of-stock |
| Remove fails | Item re-appears; toast: "Couldn't remove item. Try again." | Retry |
| Wishlist load fails | Toast: "Unable to load wishlist. Pull to refresh." | Pull to refresh |

---

## 6. Empty State

"Your Wishlist is Empty" with heart icon and "Browse Feed" CTA linking to `/feed`.

---

*STROBE — Wishlist Spec · v1 · April 2026 · Confidential*
