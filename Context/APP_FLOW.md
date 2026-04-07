# STROBE — Application Flow

> v2 · April 2026 · Confidential

---

## How to Read This Document

This document traces the complete user journey through STROBE from first visit to repeat engagement. It shows HOW all the features connect and flow together, not the detailed functional requirements. For detailed functional requirements, error handling, edge cases, and field-level validations for any feature, see the relevant spec file in `Context/specs/`.

---

## Core User Flows

### Flow 1: New User — Registration → Onboarding → First Feed

**Route:** `/welcome` → `/register` → `/onboarding` (7 steps) → `/feed`

**User arrives at `/welcome`:** Sees STROBE logo, value prop, and two CTAs: Create Account and Sign In.

**User taps Create Account:** Lands on `/register` form (first/last name, email, password with strength indicator, Apple/Google OAuth options).

**User submits registration:** System creates user account, sets `isAuthenticated: true`, navigates to `/onboarding`.

**Onboarding (7 steps, ~2–3 minutes total):**
0. Welcome intro: STROBE logo + short personalisation value prop. CTA: "Get started".
1. Age: Birth year picker. Frames taste discovery by age demographic.
2. Gender expression: Multi-select (Womenswear, Menswear, Androgynous, All of the above). ≥1 selection required.
3. Sizing: Gender-adaptive size selectors (tops, bottoms, shoes). Women's US numeric / Men's US chest & waist. Optional — all fields skippable.
4. City: IP geolocation pre-fill, manual override available. Powers location-aware recommendations.
5. Activities: Multi-select lifestyle grid (11 occasions: everyday, office, creative workplace, dinner out, cocktails, formal events, travel, gym, outdoor, beach, festivals). User rates each selected activity as Often/Sometimes/Rarely. Frequency maps directly to occasion taste vector weighting.
6. Brands: Searchable brand list, 3–10 selections required. "I shop vintage/secondhand primarily" toggle option. Brands' style vectors are averaged to seed a brand-prior vector.

System shows loading state ("We're building your feed...") for up to 3 seconds, then navigates to `/feed`.

> **P1 Enhancement:** A swipe mechanic (10 curated product cards, swipe right/left) will be added as a 6th onboarding step post-MVP to strengthen initial taste vector seeding with real interaction data. See `Context/specs/onboarding.md` Section 3.6.

For detailed onboarding spec, see `Context/specs/onboarding.md`.

---

### Flow 2: Returning User — Login → Feed

**Route:** `/welcome` → `/login` → `/feed`

**User taps Sign In on `/welcome`:** Navigates to `/login`.

**User enters email/password (or uses Apple/Google OAuth):** System authenticates, sets `isAuthenticated: true`, redirects to `/feed`.

If email or password is incorrect, show inline error. After 5 failed attempts, form locks with a cooldown message.

For detailed auth spec, see `Context/specs/auth.md`.

---

### Flow 3: Password Reset

**Route:** `/login` → `/reset-password` → email received → user clicks link → new password set → `/login`

**User taps "Forgot Password" on `/login`:** Navigates to `/reset-password`.

**User enters email:** System sends reset link (same success message shown regardless of email validity, to prevent account enumeration).

**User clicks link in email:** Taken to password reset form.

**User sets new password:** Returns to `/login`.

For detailed auth spec, see `Context/specs/auth.md`.

---

### Flow 4: Product Discovery — Feed Browsing

**Route:** `/feed` → scroll/interact → product detail or wishlist

**User lands on `/feed`:** Sees sticky header (logo, notification bell), occasion filter (Everyday / Going Out / Work / Special / All), and a 2-column product grid with skeleton cards. Cards load with stagger animation.

**User interacts with products:**
- **Like (heart icon):** Signals positive preference. Logged as explicit signal (weight 0.8).
- **Dismiss (long-press or swipe):** Removes card, shows undo snackbar (4 seconds). Logged as negative signal (weight −0.3).
- **Save to wishlist (bookmark icon):** Adds to wishlist, logged as explicit signal (weight 1.0).
- **Tap product card:** Navigates to `/product/[id]`. Logged as passive signal (weight 0.6).
- **Dwell time:** Invisible tracking via IntersectionObserver. 2–5 seconds logged (weight 0.15); 5+ seconds logged (weight 0.4).

**Occasion filter:** Selecting an occasion (e.g., Going Out) filters results to products tagged for that occasion and feeds signals into the occasion-specific taste vector.

**Infinite scroll:** New products pre-fetch when user is within 5 cards of list end. Empty state shows when all products dismissed.

For detailed feed spec, see `Context/specs/feed.md`.

---

### Flow 5: Product Search

**Route:** `/search` → type query → results → optional filter/sort → product tap

**User taps Search tab (bottom nav):** Lands on `/search`.

**User types query:** Search box accepts natural language (e.g., "cozy sweater for autumn"). System sends query to Marqo vector search, returns ranked product IDs, fetches product details from database.

**Results display:** 2-column grid of matched products. User can:
- Filter by brand, colour, size, price, or other product attributes
- Sort by relevance, newest, price (low to high, high to low)
- Tap a product → `/product/[id]`

**Search signal:** Query text, result count, and filters applied are logged as metadata (no taste vector weight, but useful for search quality analysis).

For detailed search spec, see `Context/specs/search.md`.

---

### Flow 6: Product Detail → Add to Bag

**Route:** `/product/[id]` → select size → "Add to Bag" → stay on page or navigate to `/bag`

**User taps a product from feed or search:** Lands on `/product/[id]`.

**Product detail shows:**
- Image carousel (full-width, swipeable)
- Product title, brand, price, description
- Size selector (if out of stock, sizes greyed and Add to Bag disabled)
- Wishlist toggle (bookmark icon)
- "Add to Bag" button (prominent)
- Related products carousel (other items from same brand or category)

**User selects size:** Button becomes active.

**User taps "Add to Bag":** System calls Shopify tokenless cart API to create or update cart for that brand. Brief spinner on button. Toast confirms: "Added to bag". User remains on product page; bag icon in bottom nav updates (badge shows count).

**User can then:**
- Continue shopping (back to feed/search)
- Tap bag icon → `/bag` to review cart

For detailed product detail spec, see `Context/specs/product-detail.md`.

---

### Flow 7: Bag → Checkout Handoff

**Route:** `/bag` → review items → "Checkout with [Brand]" → Shopify checkout

**User navigates to `/bag`:** Sees all items grouped by brand (multi-brand support).

**Items are displayed:**
- Product image, title, size, price per item
- Per-item remove button
- Brand section with subtotal, "Checkout with [Brand]" CTA

**User reviews and removes items as needed.**

**User taps "Checkout with [Brand]":** System opens Shopify's hosted checkout URL in a browser tab. Payment, shipping, and order confirmation all happen on Shopify's secure domain.

**On return to STROBE:** Checkout-initiated event is logged as the strongest purchase-intent signal (weight 1.0).

For detailed bag and checkout spec, see `Context/specs/bag-checkout.md`.

---

### Flow 8: Wishlist Management

**Route:** `/wishlist` → browse saved items → optional navigate to product detail

**User taps Wishlist tab (bottom nav):** Lands on `/wishlist`.

**Wishlist displays:**
- All saved products in 2-column grid
- Product image, title, brand, current price
- Price drop badge (e.g., "Was $120, now $95") if price has fallen since save
- Wishlist toggle (remove item)
- Tap product → `/product/[id]`

**Price tracking:** System runs nightly job comparing current prices to saved prices. If price drops, badge displays and a notification is created.

For detailed wishlist spec, see `Context/specs/wishlist.md`.

---

### Flow 9: Orders / Checkout History

**Route:** `/orders` → view past checkouts → optional navigate to product detail

**User taps Orders tab (bottom nav):** Lands on `/orders`.

**Orders display:**
- List of past checkout-initiated events (completed transactions via Shopify)
- Per order: date, brand, items, total, status (where Shopify provides it)
- Tap order → shows details and related products

Note: STROBE does not store full order/payment data (PCI scope avoided). Order status reflects what Shopify's webhooks report back.

For detailed orders spec, see `Context/specs/orders.md`.

---

### Flow 10: Profile & Preferences

**Route:** `/profile` → view/edit preferences → optional navigate to edit onboarding

**User taps Profile tab (bottom nav):** Lands on `/profile`.

**Profile shows:**
- User name, email
- Style summary: "You love minimalist, everyday wear with earthy tones" (generated from taste vectors and interaction history)
- Occasion breakdown: Pie chart of interactions by occasion
- Top brands: Your most-liked brands
- Edit Preferences button → re-launch `/onboarding` with prior answers pre-filled
- Notification settings: Price drop alerts, restock alerts, new arrivals (toggles)
- Account management: Logout, delete account (if supported)

For detailed profile spec, see `Context/specs/profile.md`.

---

## Navigation Overview

STROBE uses a **5-tab bottom navigation bar** (fixed, 64px height, safe area inset) visible on authenticated routes except during onboarding, product detail, bag, and notifications views.

**Bottom nav tabs:**
1. Feed → `/feed`
2. Search → `/search`
3. Wishlist → `/wishlist`
4. Orders → `/orders`
5. Profile → `/profile`

**Non-nav screens:**
- `/welcome`, `/register`, `/login`, `/reset-password` (public, no auth)
- `/onboarding` (authenticated only, full-screen flow)
- `/product/[id]` (authenticated, detail view)
- `/bag` (authenticated, checkout review)
- `/notifications` (authenticated, accessed via bell icon in sticky header)

**Global sticky header** appears above nav tabs: Logo, notification bell with unread badge.

For complete screen inventory, route map, responsive behaviour, and entry points, see `Context/specs/navigation.md`.

---

## Interaction Signal Summary

Every user action feeds the recommendation engine. Signals are categorized as explicit (intentional, high signal-to-noise) or passive (inferred preference, lower weight).

**Explicit signals:**
- Checkout initiated (weight 1.0) — strongest commercial intent
- Save to wishlist (weight 1.0)
- Add to bag (weight 0.9)
- Like (weight 0.8)
- Dismiss (weight −0.3) — negative signal

**Passive signals:**
- Product detail view (tap) (weight 0.6)
- Dwell 5+ seconds (weight 0.4)
- Dwell 2–5 seconds (weight 0.15)
- Under 2 seconds — not logged

All signals are captured with product ID, timestamp, and occasion context (if applicable). Signals flow to the recommendation engine, which updates per-occasion taste vectors and recalculates the For You feed.

For complete signal definitions, weights, and consumption by the engine, see `ARCHITECTURE.md` Section 7 (Interaction Signals).

---

## Decision Points

Key branching logic across the user journey:

**New user vs. returning user** at `/welcome`: Check auth token. If missing, show welcome. If valid, redirect to `/feed`.

**Onboarding completion check:** On first login post-registration, route to `/onboarding`. If user navigates back later via Profile → Edit Preferences, pre-fill prior answers and resume from first incomplete step.

**Occasion filter context:** When user selects an occasion on `/feed`, subsequent interactions feed the occasion-specific taste vector, not just the general vector.

**Guest cart vs. authenticated cart:** In v0, cart state is authenticated (requires login before checkout). Guest cart support (unauthenticated session cart) not implemented.

**Bag grouping by brand:** Multi-brand carts are grouped by brand in the UI, with separate "Checkout with [Brand]" CTAs per group. User must complete checkout separately for each brand.

**Checkout handoff:** Tapping "Checkout with [Brand]" opens Shopify's hosted checkout URL. STROBE does not handle payment processing (PCI scope avoided). On return to STROBE, checkout-initiated event is logged.

---

*STROBE · Application Flow · v2 · April 2026 · Confidential*
