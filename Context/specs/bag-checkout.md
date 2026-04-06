# Bag & Checkout — Functional Spec

> Part of STROBE · April 2026

---

> **Priority: P1 (Post-MVP).** Cart and checkout handoff is not required for v0 launch. Core browse, search, and personalised feed (P0) are built first. Shopify tokenless cart API is available and architecture is resolved — this spec defines what to build once P0 is complete. See `Context/DECISIONS.md` for sequencing rationale.

---

## 1. Overview

The Bag is STROBE's native multi-brand shopping cart experience. It groups items by retailer brand (each brand maps to a separate Shopify cart object) and provides per-brand checkout handoff to Shopify's hosted checkout. STROBE owns the full discovery-to-checkout-initiation journey (feed → detail page → size selection → bag → checkout initiation) and instruments every step for conversion data and personalisation signals. Payment processing is handled entirely by Shopify; STROBE has no visibility into payment completion unless the brand provides an order webhook. Each brand in the user's bag generates its own independent Shopify checkout session.

---

## 2. Bag Flow

The standard checkout flow consists of four steps:

| Step | User Action | System Response |
|------|-------------|-----------------|
| 1 | Taps "Add to Bag" on product detail page (after size selection) | Shopify tokenless cart API called for that product's brand store. Cart object created or updated. checkoutUrl returned and stored in `carts` table. Product appears in user's bag. |
| 2 | Taps bag icon or navigates to `/bag` | Bag page renders with all items grouped by brand. Each brand section shows: brand name/logo, line items (product image, title, size, price at time of add), brand subtotal, and "Checkout with [Brand]" primary button. |
| 3 | Taps "Checkout with [Brand]" button for a specific brand | checkout_initiated event logged to `cart_events` and `user_interactions` (strongest purchase-intent signal). Cart status updated to checkout_initiated. Stored checkoutUrl opened in browser as external navigation. |
| 4 | Completes payment on Shopify checkout | Payment handled entirely by Shopify (PCI scope outside STROBE). STROBE has no visibility into payment success unless brand provides order webhook. User returns to STROBE app (or closes checkout). |

---

## 3. Functional Requirements

### FR-BG-001 — Bag Display
The Bag page (`/bag`) displays all active cart items grouped by brand. Each brand section contains: brand name/logo, line items (product thumbnail image, product title, selected size, price at time of add — to prevent display inconsistency if price changes elsewhere), and a remove button (×) per line item. Brand subtotal shown below line items. "Checkout with [Brand]" primary CTA button per brand section.

| UI State | Content | Actions |
|----------|---------|---------|
| Populated (single brand) | Items from one brand, brand section, line items with remove buttons, checkout CTA | Tap line item to view product; tap ×; tap Checkout |
| Populated (multi-brand) | Items grouped into 2+ brand sections, each with separate checkout button | Same as above; note displayed: "Each brand checks out separately." |
| Empty | Centered empty state with heart icon, heading "Your bag is empty. Find something you love.", Browse Feed CTA | Tap Browse Feed → `/feed` |
| Loading | Skeleton loader (shimmer) for bag structure | Wait for cart data to load |

### FR-BG-002 — Multi-Brand Grouping
When the bag contains items from multiple brands, each brand is rendered as a distinct, visually separated section. Each section has its own "Checkout with [Brand]" button. A note is displayed prominently near the top of the bag: "Each brand checks out separately." This ensures transparency — users understand they will complete multiple checkout sessions (one per brand).

| Scenario | Display | Explanation |
|----------|---------|-------------|
| Single brand (1–N items) | One brand section | Straightforward; one checkout button |
| Two brands (items from both) | Two distinct brand sections | "Each brand checks out separately." note shown |
| Three+ brands | Three+ distinct sections | Note repeated or shown once at top |

### FR-BG-003 — Add to Bag Mechanism
On the product detail page, user selects a size/variant from the size selector (connected to Shopify variant IDs). Tapping "Add to Bag" calls the Shopify tokenless cart API for the product's brand store. The API creates a new cart object (if none exists for that brand) or updates an existing cart. The response includes a checkoutUrl, which is stored in the `carts` table. A confirmation toast appears: "Added to bag". The bag icon in the header increments its count. The user remains on the product detail page.

**Validation & Error Handling:**
- If no size is selected when Add to Bag is tapped, the button shakes and error toast appears: "Please select a size"
- If the Shopify cart API fails, error toast: "Couldn't add to bag. Please try again." User can retry.
- If a selected variant is out of stock (Shopify API field `availableForSale` = false), the size is shown as greyed/unavailable and cannot be added. Toast: "This size is no longer available."

| Precondition | Action | Result |
|--------------|--------|--------|
| Size selected, variant in stock | Tap Add to Bag | Shopify API call succeeds; checkoutUrl stored; item appears in bag; toast "Added to bag"; bag icon increments |
| No size selected | Tap Add to Bag | Button shakes; toast "Please select a size" |
| Shopify API failure | Tap Add to Bag | Toast "Couldn't add to bag. Please try again." |
| Variant out of stock | Try to select size | Size greyed out; cannot select |

### FR-BG-004 — Remove from Bag
Tapping the × button on a line item removes that product from the Shopify cart via cart API mutation and removes it from the bag display. If the last item from a brand is removed, the entire brand section is removed from the bag. The UI uses optimistic updates: the item is removed from the display immediately. An undo snackbar (with "Undo" button) appears for 4 seconds. If user taps Undo, the item is re-added to the Shopify cart and display. If the snackbar expires, the removal is confirmed. Removal event is logged to `cart_events` (type: remove).

| State | Display | Action |
|-------|---------|--------|
| Item visible in bag | × button on line item | Tap to remove |
| Item removed (optimistic) | Item disappears from display | Undo snackbar appears for 4 seconds |
| Undo within 4s | Item reappears | Snackbar dismissed; item back in bag |
| Snackbar expires | Item removed from Shopify cart | Removal confirmed |

### FR-BG-005 — Checkout Handoff
Tapping "Checkout with [Brand]" button executes the following sequence:
1. Log `checkout_initiated` event to `cart_events` table with fields: user_id (nullable for guests), product_id, brand_id, cart_id, session_id, timestamp
2. Also log to `user_interactions` with type: checkout_initiated, weight: 1.0 (strongest signal for taste vector)
3. Update cart status in `carts` table to `checkout_initiated`
4. Open the stored checkoutUrl in a new browser tab/window (external navigation)
5. Shopify's hosted checkout loads with cart items pre-populated
6. User completes payment on Shopify (STROBE has no visibility unless brand provides webhook)
7. On successful payment, user either returns to STROBE (if checkout URL includes return URL) or closes tab

| Step | Action | Result |
|------|--------|--------|
| User taps Checkout | Events logged | cart_events and user_interactions updated |
| checkoutUrl opened | Browser navigation | Shopify checkout loads with items pre-populated |
| User enters payment info | External to STROBE | STROBE no longer visible |
| Payment completes | Shopify confirms | Order confirmation on Shopify side; optional webhook to STROBE |

### FR-BG-006 — Cart Persistence
Cart state is persisted to enable recovery across page refreshes and app closures.

**For unauthenticated guests:** Cart data stored in `localStorage` under a deterministic key (e.g., `strobe_guest_cart`). Survives page refresh, app close, and even browser restart (until localStorage is cleared).

**For authenticated users:** Cart data stored in `carts` and `cart_line_items` tables in the database. Multiple carts can exist per user (one per brand). Survives indefinitely until explicitly removed.

**Expiry:** Cart items expire after 7 days of inactivity. On cart page load or bag access, expired items are automatically removed. Expired carts are marked with `status: 'expired'` and are not displayed.

| User Type | Storage | Persistence | Expiry |
|-----------|---------|------------|--------|
| Guest | localStorage | Page refresh, app restart | Not applicable; cleared on browser cache clear |
| Authenticated | carts + cart_line_items tables | Indefinite until removal | 7 days of inactivity |

### FR-BG-007 — Cart Events Instrumentation
Every cart action is logged to the `cart_events` table for conversion analysis and personalisation. Events are the primary commercial data layer.

**Event types:**
- **add** — Item added to bag (also logged to user_interactions)
- **remove** — Item removed from bag
- **update_qty** — Quantity updated (if applicable; v0 treats size+product as distinct line items)
- **checkout_initiated** — User taps "Checkout with [Brand]" (also logged to user_interactions with weight 1.0)

**Required fields per event:** user_id (nullable for guests), product_id, brand_id, cart_id, session_id, event_type, timestamp.

| Event | Trigger | Required Fields |
|-------|---------|-----------------|
| add | "Add to Bag" successful | product_id, brand_id, cart_id, session_id, size |
| remove | × button on line item tapped | product_id, brand_id, cart_id, session_id |
| update_qty | Quantity changed (future feature) | product_id, brand_id, quantity |
| checkout_initiated | "Checkout with [Brand]" tapped | product_id, brand_id, cart_id, session_id |

### FR-BG-008 — Variant/Size Availability Check
Before adding a product to the bag, the system checks the Shopify API to confirm that the selected variant's `availableForSale` field is true. If false (out of stock), the size is displayed as greyed/unavailable on the product detail page and cannot be selected. If a variant goes out of stock after being added to the bag (between page load and checkout), the size is refreshed on page reload or when the user accesses the bag.

| Scenario | Display | Behaviour |
|----------|---------|-----------|
| Variant in stock | Size button, normal styling | User can select and add |
| Variant out of stock (at page load) | Size button, greyed, strikethrough | User cannot select |
| Variant goes OOS (after add) | Item in bag shows size; may be unavailable at checkout | Shopify handles; checkout may warn user |

---

## 4. User Flow

**Happy Path:**
```
User browses /feed or /search
  → Taps product card
  → Navigates to /product/[id]
  → Selects size
  → Taps "Add to Bag"
  → Toast confirms addition
  → Bag icon updates
  → (Optional) User adds more items from different brands
  → Taps bag icon or navigates to /bag
  → Bag page displays all items grouped by brand
  → Reads "Each brand checks out separately" note
  → Taps "Checkout with [Brand]" for Brand A
  → checkout_initiated logged
  → Shopify checkout opens in new tab
  → User enters payment info on Shopify
  → (Optional) Taps "Checkout with [Brand]" for Brand B
  → Repeat checkout for second brand
```

**Edge Case — Out of Stock During Checkout:**
```
User adds item to bag
  → Later returns to bag
  → Item is still visible
  → Taps "Checkout with [Brand]"
  → Shopify checkout loads with item
  → During checkout, user discovers item is OOS
  → Shopify warns; user cannot complete with that item
```

---

## 5. Error States & Recovery

| Error | Display | Recovery |
|-------|---------|----------|
| Bag empty | Centered empty state with Browse Feed CTA | Tap Browse Feed → `/feed` |
| Remove from bag fails (API error) | Item re-appears in bag. Toast: "Couldn't remove item. Try again." | Tap × again to retry |
| checkoutUrl expired or invalid (between add and checkout) | Toast: "Checkout link expired. We're refreshing it." System re-fetches checkoutUrl via Shopify cart API. | User taps "Checkout with [Brand]" again; refreshed URL opens. |
| Network error during checkout handoff | Toast: "Couldn't open checkout. Check your connection." | Retry; check network and tap "Checkout with [Brand]" again |
| Shopify cart mutation after URL generated | checkoutUrl becomes stale; clicking it shows Shopify error | System refreshes checkoutUrl on every cart mutation (add/remove) |
| Guest user attempts checkout | Prompt to sign up appears: "Create an account to check out." Cart preserved in localStorage. | User signs up or logs in; cart data transferred to `carts` table; checkout continues. |
| User navigates back from Shopify checkout | Returns to STROBE `/bag` page | Bag state unchanged; items still visible; user can retry checkout or modify bag |

---

## 6. Data Model

For complete schema details, see **Backend/api/README.md**. Key tables:

| Table | Key Columns | Usage |
|-------|-------------|-------|
| `carts` | id, user_id (nullable), brand_id, checkout_url, status (active/checkout_initiated/expired), created_at, expires_at | Stores Shopify cart objects per brand; one row per brand per user |
| `cart_line_items` | id, cart_id, product_id, variant_id, size, price_at_add, quantity, created_at | Line items within a cart; one row per product per cart |
| `cart_events` | id, user_id (nullable), product_id, brand_id, cart_id, session_id, event_type (add/remove/update_qty/checkout_initiated), created_at | Every cart action logged; primary commercial data |
| `products` | id, brand_id, title, price, sale_price, image_urls | Product core data; referenced by cart_line_items |
| `brands` | id, name, logo_url | Brand info; one row per retailer |
| `confirmed_orders` | id, user_id, brand_id, order_id (from brand webhook), amount, status, created_at | Populated only when brand provides order confirmation webhook |
| `user_interactions` | id, user_id, product_id, interaction_type (checkout_initiated), weight, created_at | Taste vector signals; checkout_initiated is weight 1.0 (highest) |

---

## 7. Acceptance Criteria

| AC ID | Criterion | Test Type | Pass Condition |
|-------|-----------|-----------|----------------|
| AC-BG-001 | Adding to bag creates/updates a Shopify cart object | Integration | Shopify cart API returns valid cart with correct line items; checkoutUrl stored in `carts` table |
| AC-BG-002 | Multi-brand bag displays brands separately with individual checkout buttons | Functional | Adding items from 3 different brands results in 3 distinct brand sections; "Each brand checks out separately." note visible |
| AC-BG-003 | Checkout handoff opens correct Shopify checkout with pre-populated items | Integration | checkoutUrl opened in browser; Shopify checkout displays correct product, size, and price |
| AC-BG-004 | checkout_initiated event logged on every handoff | Integration | `cart_events` table contains event with correct product_id, brand_id, session_id, and timestamp for every handoff |
| AC-BG-005 | Cart persists across page refresh for both guests and authenticated users | Functional | Refresh page after adding item; bag contents unchanged; item count in bag icon matches before refresh |
| AC-BG-006 | Out-of-stock variants cannot be added | Functional | Selecting an OOS size is prevented (greyed out); "Add to Bag" cannot add OOS variants; attempt shows error |
| AC-BG-007 | Remove from bag uses optimistic UI with undo | Functional | Tap × on item; item disappears immediately; undo snackbar appears; tapping undo restores item within 4 seconds |
| AC-BG-008 | Empty bag displays empty state with Browse Feed CTA | Functional | Remove all items from bag; empty state renders; tapping Browse Feed navigates to `/feed` |

---

## 8. Appendix — Future In-App Checkout

The original FRD v1.0 Section 8 (Checkout) specified a full in-app purchase flow with Stripe payment processing. This specification included:
- Shipping address management (collection and validation)
- Credit/debit card tokenisation via Stripe Elements
- Apple Pay and Google Pay integration
- Afterpay / Buy Now Pay Later option
- Order review screen with tax calculation
- Payment processing with idempotency
- Order confirmation screen with receipt

This specification is retained in **Backend/BACKEND_MAP.md** (archived) for future reference if STROBE pursues in-app checkout. Key dependencies for in-app checkout include:
- **PCI DSS Level 1 compliance** (payment card data handling) — significant operational burden
- **Stripe integration** (payment processing, token management, webhooks)
- **Brand payment agreements** — each retailer must authorize STROBE to collect payment on their behalf (legal/commercial complexity)
- **Tax calculation engine** (sales tax, VAT, shipping)
- **Refund handling** (return policy coordination with brands)

See **Context/DECISIONS.md** for the strategic timing rationale for deferring in-app checkout to a later phase. The current Shopify handoff model (FR-BG-005) is sufficient to validate the STROBE discovery and conversion funnel with minimal operational overhead.

---

*Bag & Checkout — STROBE v2 · April 2026 · Functional Specification*
