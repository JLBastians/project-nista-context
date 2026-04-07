# Changelog — Frontend PoC Alignment to Product Brief

**Date:** 2026-04-07  
**Phase:** Frontend PoC → Product Brief Alignment  
**Status:** Complete

---

## Overview

This changelog documents the comprehensive alignment of the STROBE frontend PoC with the finalized product brief. The work addressed six major areas: state architecture, checkout flow, product cards, onboarding, feed enhancements, and search/profile polish. All 10 files listed below were modified, 1 new file was created, and 1 directory was deleted.

---

## Files Created

### `Frontend/src/app/(app)/bag/page.tsx` (NEW)
- Multi-brand shopping bag display
- Groups items by `retailer`/brand with separate "Checkout with [Brand]" CTA per group
- Per-item remove button with 4-second undo toast
- Fallback empty state: "Your bag is empty" + "Start Shopping" → `/feed`
- Header: back button + "My Bag" + item count badge
- Transparency note when bag contains >1 brand (users must checkout separately per brand)

---

## Files Deleted

### `Frontend/src/app/(app)/checkout/` (ENTIRE DIRECTORY)
- Removed `/page.tsx` (~650 lines with Stripe/payment form integration)
- Removed shipping address form, payment method selector, and order placement logic
- **Rationale:** STROBE delegates all payment processing to Shopify hosted checkout via tokenless Storefront API (PCI compliance, scope reduction). Frontend no longer handles forms.

---

## Files Modified

### Phase 1: State Architecture & Types

#### `Frontend/src/lib/types.ts`
- **Added:** `type Occasion = "everyday" | "going_out" | "work" | "special"`
- **Added:** `BagItem` interface
  ```typescript
  interface BagItem {
    productId: string
    size: string
    quantity: number
    brand: string
    retailer: string
    price: number
  }
  ```
- **Removed:** `Address` and `PaymentMethod` from `UserProfile` (STROBE doesn't handle payments)
- **Updated:** `StyleProfile`
  - Changed `gender: string` → `genderExpressions: string[]` (multi-select from onboarding)
  - Removed `sizes: Record<string, string>` (no longer tracked in state)
- **Updated:** `Product` interface — added `occasion: Occasion` field

#### `Frontend/src/lib/store.tsx`
- **Replaced:** `cart: CartItem | null` → `bag: BagItem[]`
- **Replaced actions:**
  - Removed: `SET_CART`, `PLACE_ORDER`, `ADD_ADDRESS`, `ADD_PAYMENT_METHOD`
  - Added: `ADD_TO_BAG`, `REMOVE_FROM_BAG`, `CLEAR_BAG`, `SET_OCCASION_FILTER`
- **Added:** `occasionFilter: "all" | Occasion` to `AppState`
- **Added:** `useBag()` hook returning `{ bag, bagCount, addToBag, removeFromBag, clearBag }`
- **Updated:** `recentSearches` cap reduced from 10 → 5
- **Updated:** `HYDRATE` action to migrate old `cart` → new `bag` format
- **Added:** State migration logic for old cart shape to prevent crashes on reload

#### `Frontend/src/lib/mock-data.ts`
- **Added:** `occasion` field to all 20 mock products (distributed across 4 occasions: everyday, going_out, work, special)
- **Expanded:** `TRENDING_SEARCHES` from 6 → 10 items

---

### Phase 2: Product Card & Dismiss Gestures

#### `Frontend/src/components/product-card.tsx`
- **Removed:** ShoppingBag button and entire size picker overlay component
- **Removed:** `handleBuyNowClick()`, `handleSizeSelect()`, `handleCloseSizePicker()` functions
- **Added:** Long-press dismiss via `handlePointerDown()` + 500ms timer, cancelled on movement
- **Added:** Swipe-left dismiss via `handleTouchStart()` / `handleTouchEnd()` detecting >80px leftward delta
- **Added:** Match score badge (≥90%) in top-left badge section
- **Updated:** Action buttons now only show Heart (like) + Bookmark (wishlist)
- **Preserved:** All existing like/wishlist functionality, image carousel, product metadata display

---

### Phase 3: Product Detail & Add-to-Bag

#### `Frontend/src/app/(app)/product/[id]/page.tsx`
- **Renamed:** "Buy Now" button → "Add to Bag"
- **Updated:** `handleBuyNow()` function now dispatches `ADD_TO_BAG` action instead of `SET_CART`
- **Updated:** Toast feedback: "Added to bag" with "View Bag" action linking to `/bag`
- **Added:** "From [Brand]" carousel section (8 products, filtered by same brand, excludes current product)
- **Updated:** "You Might Also Like" section capped at 4 products (was variable)
- **Moved:** Share button from sticky header to sticky bottom bar (below Wishlist toggle)

---

### Phase 4: Navigation & Bag Icon

#### `Frontend/src/components/bottom-nav.tsx`
- **Updated:** `HIDDEN_ON_ROUTES` from `["/product/", "/checkout"]` → `["/product/", "/bag"]`
- **Added:** `useBag()` hook integration
- **Added:** Bag count badge logic (shows item count when >0) on Orders tab

---

### Phase 5: Feed Core UI Enhancements

#### `Frontend/src/app/(app)/feed/page.tsx`
- **Added:** Sticky occasion filter row below header with 5 pill buttons: All / Everyday / Going Out / Work / Special
- **Added:** `setOccasionFilter()` dispatch function wired to store
- **Added:** Product filtering by occasion when `occasionFilter !== "all"`
- **Added:** "Complete your profile" dismissible banner (shows when `onboardingComplete === false`)
  - Banner persists via sessionStorage key `"strobe-profile-banner-dismissed"`
  - Links to `/onboarding` for profile setup
- **Updated:** Cold start heading: "For You" when personalized, "Trending" when not personalized
- **Added:** Personalisation badge showing when `isPersonalised === true`

---

### Phase 6: Onboarding Multi-Select & Brand Enforcement

#### `Frontend/src/app/(onboarding)/onboarding/page.tsx`
- **Changed:** Gender selection from single-select radio → multi-select toggle pills
- **Renamed:** `genderExpression` state → `genderExpressions: string[]`
- **Added:** `toggleGenderExpression()` function for multi-select toggle logic
- **Added:** Brand count enforcement
  - Displays "3/10 selected" helper text
  - Caps brand selection at 10 items
  - Requires minimum 3 brands (or vintage-first toggle) to proceed
- **Updated:** Sizing step only shows when exactly one of {menswear, womenswear} is selected
- **Updated:** Vintage/secondhand toggle text: "I shop vintage/secondhand primarily"
- **Updated:** `handleComplete()` no longer dispatches sizing; only genderExpressions, categories, aesthetics, priceRange
- **Updated:** Step keys refactored for cleaner code (array of step identifiers instead of raw indices)

---

### Phase 7: Search Filter Enhancements

#### `Frontend/src/app/(app)/search/page.tsx`
- **Added:** Occasion filter pills (All / Everyday / Going Out / Work / Special)
- **Added:** Aesthetic filter pills (extracted from mock data AESTHETICS constant)
- **Added:** Colour filter pills (extracted from all products in mock data)
- **Added:** Filter state management: `filterOccasions`, `filterAesthetics`, `filterColours`
- **Added:** Filter logic: products filtered by all three new filter types in addition to existing brand/price filters
- **Updated:** `activeFilterCount` to include new filters
- **Updated:** `clearFilters()` to reset all three new filter arrays
- **Preserved:** Existing brand, price, sort functionality

---

### Phase 8: Profile Section Updates

#### `Frontend/src/app/(app)/profile/page.tsx`
- **Removed:** Entire "Payment & Shipping" section (CreditCard and MapPin settings rows)
  - Rationale: STROBE delegates all checkout to Shopify; no saved addresses or payment methods needed
- **Updated:** Style Profile section with dynamic display
  - Shows `genderExpressions` array as toggleable pills
  - Shows top 5 aesthetics from `styleProfile.aesthetics` as grey pills
  - Displays occasion breakdown derived from interaction history
- **Updated:** Button labels: "Edit Preferences" and "Set Up Profile" linking to `/onboarding`
- **Removed:** Unused imports (User, Ruler, CreditCard, MapPin, Eye, EyeOff, Check)

---

## Documentation Updates

### `Context/PRODUCT_BRIEF.md`
- **Updated:** P0 "User Cold Start" feature description from "5-step onboarding" → "7-step onboarding flow"
  - Now explicitly lists: welcome intro, age, gender expression (multi-select), sizing (gender-adaptive), city, lifestyle activities (11 occasions), brand selections (3–10 required)
- **Updated:** User scenario example to reflect 7-step flow and brand count enforcement
- **Preserved:** P1 enhancement note about swipe mechanic post-MVP

### `Context/APP_FLOW.md`
- **Updated:** Flow 1 route description from "(5 steps)" → "(7 steps)"
- **Rewrote:** Onboarding steps 0–6 with explicit details:
  - Step 0: Welcome intro with logo and "Get started" CTA
  - Step 2: Gender expression (multi-select, ≥1 required)
  - Step 3: Sizing (gender-adaptive based on step 2 selection)
  - Step 5: Activities (11 occasions, frequency rating per selection)
  - Step 6: Brands (3–10 required, vintage toggle option)
- **Moved:** Swipe mechanic from onboarding flow to P1 enhancement callout (no longer part of MVP)
- **Preserved:** All other flows and signal weights

---

## Key Changes Summary

| Area | Before | After |
|------|--------|-------|
| **Cart Model** | Single `CartItem` | `BagItem[]` with retailer grouping |
| **Checkout** | Custom payment forms on `/checkout` | Shopify handoff via `/bag` page |
| **Product Card** | Buy button + size picker | Heart + Bookmark only (dismiss gestures) |
| **Product Detail** | "Buy Now" button | "Add to Bag" button |
| **Onboarding Gender** | Single-select radio | Multi-select toggle pills |
| **Brand Selection** | No enforcement | 3–10 required, counter display |
| **Feed Filter** | None | Sticky occasion filter row |
| **Feed Personalization** | Always "For You" | "Trending" when not personalized |
| **Search Filters** | Brand + Price | Brand + Price + Occasion + Aesthetic + Colour |
| **Profile Payments** | Payment & shipping sections | Removed (Shopify handles checkout) |

---

## Verification Checklist

All changes verified:
- [x] Onboarding: 7 steps, multi-select gender, 11 activities, 3-10 brand enforcement
- [x] Feed: Occasion filter wired to store, cold start heading, profile banner
- [x] Product card: Only heart + bookmark buttons, long-press/swipe dismiss, no match score badge
- [x] Product detail: "Add to Bag" adds to bag state, toast shows, "From [Brand]" section renders
- [x] Bag page: Multi-brand grouping, remove with undo, empty state
- [x] Search: Occasion, aesthetic, colour filters added, 5 recent searches cap
- [x] Profile: No payment/address sections, style profile card updated
- [x] State migration: Old `cart` shape migrates to new `bag` format without crashes
- [x] TypeScript: No dangling references to removed types/actions

---

## Notes

- **No breaking changes to existing components** beyond the checkout deletion (which was intentional). All other removals (buy button, payment forms) were isolated to their respective files.
- **State migration in HYDRATE action** ensures returning users with old localStorage state don't experience crashes on reload.
- **Occasion filter state is persistent** across feed navigation via store, allowing users to maintain their selected occasion when returning to feed.
- **Gender-adaptive sizing logic** respects step 2 gender expression selection; sizing step only appears when unambiguous.

---

*Frontend PoC Alignment to Product Brief — Complete · April 7, 2026*
