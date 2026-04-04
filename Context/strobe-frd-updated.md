# STROBE
## Consumer App — Functional Requirements Document
### v2 · April 2026 · United States Launch · Confidential

> **v2 update notes:** Onboarding rewritten to match Onboarding Design Document v1.0 (6-step flow powering recommendation engine). Checkout rewritten for Shopify checkout handoff model (replaces Stripe in-app checkout). Orders section revised — STROBE does not own fulfilment. Original Stripe checkout spec moved to Appendix. All sections now self-contained — no external references required.

| Document Type | Functional Requirements Document (FRD) |
|---|---|
| Product | STROBE — Consumer Mobile & Web App |
| Platform Scope | Responsive Web (v0); iOS, Android (future, approach TBD) |
| Version | 12 |
| Date | 3 April 2026 |
| Status | ACTIVE |

---

# 1. Introduction & Scope

## 1.1 Purpose
This FRD defines the complete set of functional behaviours for the STROBE consumer-facing application. It covers every screen, interaction, data input, system response, error state, and edge case. It is the primary reference for engineering, QA, and design.

This document does not cover the retailer portal, backend infrastructure architecture, or data pipeline design. Those are addressed in separate specifications.

## 1.2 Platform Scope
v0 is a responsive web application (Next.js on Vercel), designed mobile-first. Native iOS and Android apps are a future consideration (approach TBD — native vs cross-platform). Feature parity across platforms will be required when mobile apps are built.

## 1.3 Navigation Architecture
Five primary navigation tabs via a persistent bottom navigation bar (mobile) or top navigation bar (web):

| Tab | Destination |
|-----|-------------|
| Tab 1 — Feed | Personalised discovery feed (default landing post-login) |
| Tab 2 — Search | Free-text and filtered product search |
| Tab 3 — Wishlist | Saved items |
| Tab 4 — Orders | Checkout history and order tracking |
| Tab 5 — Profile | Account settings, style profile, preferences |

## 1.4 Requirement ID Convention
`FR-[MODULE]-[###]` where MODULE = AU (Auth), ON (Onboarding), FD (Feed), SR (Search), PD (Product Detail), BG (Bag), WL (Wishlist), OR (Orders), PR (Profile), NT (Notifications), GS (Global/Shared).

## 1.5 Status Key
P0 — MVP: Must be implemented before public launch. P1 — Post-MVP: Required within 6 months of launch. P2 — Future: Planned but not committed.

---

# 2. Authentication (FR-AU)

## 2.1 Screen: Registration

### 2.1.1 Overview
The Registration screen is the first screen presented to a new user who selects "Create Account" from the Welcome screen. It collects the minimum information needed to create an account. Registration can be completed via email/password or via social OAuth providers.

### 2.1.2 Functional Requirements

| REQ ID | Screen / Module | Functional Requirement | UI States | Error / Edge States |
|--------|----------------|----------------------|-----------|-------------------|
| FR-AU-001 | Registration | System presents two registration paths: (1) Email & Password, (2) Continue with Apple, (3) Continue with Google. All three options visible without scrolling. | Default / Idle | If OAuth provider returns an error, display: "Sign-up with [Provider] failed. Please try again or use email." |
| FR-AU-002 | Registration — Email Path | User enters: First Name, Last Name, Email Address, Password, Confirm Password. Form validates all fields before submission is enabled. | Empty / Typing / Valid / Invalid / Submitting | See field-level validation in Section 2.1.3 |
| FR-AU-003 | Registration — Submit | On valid submission, system creates account and sends a verification email. User is navigated to Onboarding (Section 3). Account is created in an "unverified" state; browsing is permitted but purchasing is blocked until email is verified. | Submitting (loading spinner on button) / Success / Failure | If email already exists: "An account with this email already exists. Log in instead?" with link to Login. |
| FR-AU-004 | Registration — OAuth | On selecting Apple or Google sign-in, system initiates the native OAuth flow. On success, if account is new, user is directed to Onboarding. If account already exists, user is directed to Feed. | OAuth modal open / Completing / Success / Cancelled | On OAuth cancellation: return to Registration with no error. On OAuth failure: toast "Something went wrong. Please try again." |

### 2.1.3 Field-Level Validation

| Field Name | Type | Required | Validation Rules | Notes |
|-----------|------|----------|-----------------|-------|
| First Name | Text | Yes | 2–50 characters. Letters, hyphens, apostrophes only. No numbers. | Validated on blur and on submit. |
| Last Name | Text | Yes | 2–50 characters. Letters, hyphens, apostrophes only. No numbers. | Validated on blur and on submit. |
| Email Address | Email | Yes | Must match RFC 5322 email format. Must be unique in the system. | Uniqueness checked server-side on submit only. |
| Password | Password | Yes | Minimum 8 characters. Must contain: ≥1 uppercase, ≥1 lowercase, ≥1 number. Maximum 128 characters. | Strength indicator shown inline (Weak / Fair / Strong). Toggle visibility icon. |
| Confirm Password | Password | Yes | Must exactly match Password field value. | Validated on blur. Error: "Passwords do not match." |

### 2.1.4 Acceptance Criteria

| AC ID | Acceptance Criterion | Test Type | Pass Condition |
|-------|---------------------|-----------|----------------|
| AC-AU-001 | Registration form submit button is disabled until all fields pass client-side validation. | Functional | Button is non-interactive (greyed out) with one or more invalid fields. |
| AC-AU-002 | User receives a verification email within 2 minutes of successful registration. | Integration | Email arrives within 120 seconds. |
| AC-AU-003 | Attempting to register with an existing email shows an error without clearing the form. | Functional | Error message appears inline. All other field values preserved. |
| AC-AU-004 | OAuth registration creates a valid account and bypasses the email/password fields. | Integration | User directed to Onboarding after successful OAuth without completing email/password form. |

## 2.2 Screen: Login

### 2.2.1 Functional Requirements

| REQ ID | Screen / Module | Functional Requirement | UI States | Error / Edge States |
|--------|----------------|----------------------|-----------|-------------------|
| FR-AU-010 | Login | Login screen presents: Email field, Password field (with show/hide toggle), "Log In" button, "Forgot Password?" link, "Continue with Apple", "Continue with Google", and "Create Account" link. | Default / Typing / Submitting / Error | On incorrect credentials: "Email or password is incorrect." (intentionally generic — do not specify which field is wrong). |
| FR-AU-011 | Login — Rate Limiting | After 5 consecutive failed login attempts for the same email within a 15-minute window, the account is temporarily locked. A CAPTCHA challenge is presented. | Normal / Pre-lockout warning (3rd attempt) / Locked | Lockout message: "Too many login attempts. Please wait 15 minutes or reset your password." |
| FR-AU-012 | Login — Session | On successful login, a session token is issued with a 30-day expiry. Users remain logged in across app restarts within the expiry window. Biometric authentication (Face ID / Touch ID / Android Biometrics) can be enabled from Profile settings (future — when native apps are built). | Active session / Expired session | On session expiry, user is redirected to Login with message: "Your session expired. Please log in again." |

### 2.2.2 Acceptance Criteria

| AC ID | Acceptance Criterion | Test Type | Pass Condition |
|-------|---------------------|-----------|----------------|
| AC-AU-010 | 5 failed login attempts within 15 minutes trigger account lockout with CAPTCHA. | Security | On 5th failure, lockout message displayed and CAPTCHA rendered. Login button disabled. |
| AC-AU-011 | Successful login with valid credentials navigates user to Feed tab within 2 seconds. | Performance | Feed screen renders within 2,000ms of login API 200 response. |
| AC-AU-012 | Session persists across app termination and restart within the 30-day window. | Functional | Re-opening app within expiry window shows Feed without prompting for login. |

## 2.3 Screen: Password Reset

| REQ ID | Screen / Module | Functional Requirement | UI States | Error / Edge States |
|--------|----------------|----------------------|-----------|-------------------|
| FR-AU-020 | Password Reset — Request | User enters their email address and submits. System sends a reset link regardless of whether the email exists (to prevent account enumeration). Screen shows: "If an account exists for this email, a reset link has been sent." | Default / Submitting / Sent confirmation | Network failure: "Unable to send. Check your connection and try again." |
| FR-AU-021 | Password Reset — Token | Reset link contains a time-limited token (valid for 60 minutes). Opening the link directs user to a New Password screen. Expired tokens show an error with a "Request a new link" CTA. | Valid token / Expired token / Already used | Expired: "This link has expired. Please request a new one." |
| FR-AU-022 | Password Reset — New Password | User sets a new password meeting the same validation rules as Registration. On success, all existing sessions are invalidated and user is redirected to Login. | Typing / Submitting / Success | Same password as previous: "Your new password must be different from your current password." |

---

# 3. Onboarding (FR-ON)

## 3.1 Overview

Onboarding is presented immediately after account creation. It serves a dual purpose: collect data to power the recommendation engine from the first session, and communicate STROBE's identity. The flow consists of **6 steps** designed so that every question directly feeds the taste vector personalisation engine.

Onboarding outputs two things:
- **State attributes:** Fixed profile fields that act as permanent filters (age, gender expression, city, sustainability orientation, price signal)
- **Seed taste vectors:** Initial mathematical fingerprints by occasion, derived from brand selections and the swipe mechanic

Each step is skippable. Skipped steps result in a more generic initial feed. A "complete your profile" prompt surfaces later. Target completion time: under 2 minutes.

**Note:** Onboarding is shown only once per account. Returning users who did not complete onboarding will be prompted via an in-app banner, not forced through the flow.

## 3.2 Screen Flow

| Step | Screen | User Action | System Response | What It Powers |
|------|--------|-------------|-----------------|----------------|
| 1 | Age | Enter birth year via picker/dropdown | Validate age 13–100. Store as date_of_birth. | Age-appropriate curation, cohort analysis |
| 2 | Gender expression | Multi-select: Womenswear, Menswear, Androgynous/gender-neutral, All of the above | Validate ≥1 selection. Store as gender_expression array. | Product gender filtering in feed |
| 3 | City | Confirm IP-detected city or type correction | Pre-fill from IP geolocation. Allow manual override with autocomplete (US cities). Store city + city_source. | Climate-relevant curation, occasion norms |
| 4 | Lifestyle activities | Multi-select activity grid; for each selected, rate frequency: Often / Sometimes / Rarely | Validate ≥1 selection. Store as occasion_weights JSONB. | **Occasion taste vectors** — frequency maps to vector weighting |
| 5 | Recently shopped brands | Searchable multi-select + curated quick-select grid (~30 brands). 3–10 selections, or "I shop vintage/secondhand" option | Cross-reference sustainability taxonomy → sustainability_orientation. Compute brand-derived style vector. | Sustainability inference, initial style vector seed |
| 6 | Swipe mechanic | 10 curated product cards. Swipe right = like, left = pass. | Compute weighted vector from swipe responses. Blend 60/40 with brand prior. Write to user_taste_profiles. | **Direct taste vector seeding** in Marqo's vector space |

## 3.3 Functional Requirements

**FR-ON-001 — Step 1: Age**
Single birth year input (dropdown or scroll picker). Validates age between 13 and 100. Stored as date_of_birth on user record (age calculated dynamically). Framing: "How old are you? We'll use this to tailor your edit." Skippable.

UI States: Default (placeholder) / Selected / Skipped.

**FR-ON-002 — Step 2: Gender Expression**
Multi-select with four options: Womenswear, Menswear, Androgynous/gender-neutral, I dress across all of the above. Framing: "How do you like to dress?" At least 1 required if not skipped. Users who select multiple or "all" receive unfiltered product results. Stored as gender_expression text array.

UI States: Default / 1+ selected / Skipped.

**FR-ON-003 — Step 3: City**
IP geolocation pre-fill using ipapi.co or equivalent. Detected city presented as confirmation ("Is this where you're based?"). Manual override via text input with autocomplete (US cities). Stores city name string and city_source ("ip" or "manual"). No precise geolocation stored — city name only. Skippable.

UI States: Pre-filled (awaiting confirmation) / Manual entry / Confirmed / Skipped.
Error: If geolocation fails, city field is empty with placeholder "Enter your city". No error shown to user.

**FR-ON-004 — Step 4: Lifestyle Activities**
Multi-select activity grid. Presented as tappable cards:

| Activity | Internal Occasion Mapping |
|----------|--------------------------|
| Everyday / errands | Everyday occasion vector |
| Working in an office | Office / corporate vector |
| Creative or casual workplace | Creative workplace vector |
| Working from home | Everyday + WFH vector |
| Dinner out / restaurants | Dinner / elevated casual vector |
| Cocktail events / parties | Cocktail / semi-formal vector |
| Formal events / galas | Formal / black tie vector |
| Weekends away / travel | Travel vector |
| Gym / fitness | Activewear vector |
| Outdoor activities / hiking | Outdoor / active vector |
| Beach / resort | Resort / swimwear vector |
| Festivals / concerts | Festival / creative event vector |
| Date nights | Date night / going out vector |
| Weddings / celebrations | Wedding guest / special occasion vector |

For each selected activity, frequency selector appears: Often (weight 1.0) / Sometimes (weight 0.5) / Rarely (weight 0.15). Frequency maps directly to occasion taste vector weighting and feed real estate allocation. At least 1 selection required if not skipped. Stored as occasion_weights JSONB (e.g. {"everyday": 1.0, "dinner_out": 0.5, "formal": 0.15}).

UI States: Default grid / 1+ selected (frequency selectors visible) / Skipped.
Note: Activity names shown to users are conversational — internal occasion mapping labels are never displayed.

**FR-ON-005 — Step 5: Brand Selections**
Searchable brand list with autocomplete + curated quick-select grid of ~30 common brands. Allow 3–10 selections. "I shop vintage/secondhand primarily" option available (doesn't require brand selection).

On completion, two signals are derived:

*Signal 1 — Sustainability position:* User's brand selections cross-referenced against internal brand sustainability taxonomy.

| Pattern | Stored Value |
|---------|-------------|
| 3+ brands tagged as certified sustainable / ethical / B-Corp | sustainability_orientation = "high" |
| Mix of sustainable and conventional brands | sustainability_orientation = "mixed" |
| Conventional brands only, or fewer than 3 selections | sustainability_orientation = "low" |
| User selects vintage / secondhand option | sustainability_orientation = "secondhand" |

*Signal 2 — Initial style vector seed:* Each brand has a pre-computed Marqo embedding (brand-level style vector). Selected brands' vectors are averaged to produce a first-cut taste vector. This is a weak prior — it will be immediately blended with swipe data in Step 6.

Stored in onboarding_brand_selections table. Skippable.

UI States: Default (search + grid) / Searching / 3–10 selected / Secondhand selected / Skipped.
Error: Brand not in catalogue → "No matching brands" in dropdown. User can continue without that brand.

**FR-ON-006 — Step 6: Swipe Mechanic**
Full-screen card stack. 10 product cards from the STROBE catalogue. Swipe right = like, swipe left = pass. Binary forced choice — no ratings, no skip on individual cards.

*Card selection (cofounder scope):* Cards selected algorithmically:
- Filter catalogue by user's top 2 occasions (from Step 4) and gender expression (from Step 2)
- Use brand-derived style vector (from Step 5) to select products distributed across the likely style space — not clustered (include items the user will reject for informative negative signal)
- Allocate cards proportionally to occasion frequency (e.g. 6 for top occasion, 4 for second)
- Ensure visual diversity — no two cards should look nearly identical

*Vector computation after 10 swipes:*
- Right-swiped product vectors averaged with weight +1.0
- Left-swiped product vectors averaged with weight −0.3 (mild negative — user may have passed for wrong colour or already-owned, not wrong aesthetic)
- Resulting vector blended 60/40 with brand-derived prior from Step 5
- Combined vector written as initial taste_vector in user_taste_profiles

Stored in onboarding_swipe_responses table. The swipe step itself is not skippable (individual cards must all be swiped), but the user can skip the entire onboarding flow before reaching Step 6.

UI States: Card stack visible / Swiping (animation) / All 10 complete / Loading feed.
Error: If card generation fails (insufficient products matching criteria), fall back to trending/editorial cards. Note: "We're still building your catalogue — these are some of our favourites."

**FR-ON-007 — Onboarding Completion**
On completion of Step 6 (or skip from any step), system compiles onboarding data and triggers feed generation. Loading state ("We're building your feed...") shown for up to 3 seconds. If feed generation fails, show trending feed with banner: "We had trouble personalising your feed. We'll keep learning as you browse!"

User lands on For You feed. Subtle message: "Your edit is ready — it'll keep getting better as you explore."

## 3.4 Onboarding Data Model

**Fields on users table:**

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| date_of_birth | date | No | Derived from birth year. Age calculated dynamically. |
| gender_expression | text[] | No | Array e.g. ["womenswear", "androgynous"]. |
| city | text | No | City name string. |
| city_source | text | No | "ip" or "manual". |
| sustainability_orientation | text | No | Inferred: "high" / "mixed" / "low" / "secondhand". |
| price_signal | text | No | Inferred from brands: "investment" / "mid" / "value" / "mixed". |
| onboarding_complete | boolean | Yes | True once Step 6 finished or explicitly skipped. |
| onboarding_completed_at | timestamp | No | When onboarding was finished. |

**Fields on user_taste_profiles:**

| Field | Type | Notes |
|-------|------|-------|
| seed_method | text | "swipe" / "brand_prior_only" / "skipped" |
| occasion_weights | jsonb | e.g. {"everyday": 1.0, "dinner_out": 0.5, "formal": 0.15} |
| brand_prior_vector | vector | Brand-derived style vector from Step 5, stored separately for debugging. |

**Separate tables:**

*onboarding_brand_selections:* id (uuid PK), user_id (uuid FK), brand_name (text), brand_id (uuid FK, nullable if brand not in catalogue), selected_at (timestamp).

*onboarding_swipe_responses:* id (uuid PK), user_id (uuid FK), product_id (uuid FK), response (text: "like" or "pass"), card_position (int: 1–10), responded_at (timestamp).

## 3.5 Acceptance Criteria

| AC ID | Criterion | Test Type | Pass Condition |
|-------|-----------|-----------|----------------|
| AC-ON-001 | Onboarding completable in ≤2 minutes | Usability | 95th percentile ≤2 minutes in user testing (n≥20). |
| AC-ON-002 | Back navigation preserves selections on all steps | Functional | Navigating back shows previously made selections. |
| AC-ON-003 | Skipping any step proceeds to next step | Functional | Skip link visible; tapping advances without storing data for that step. |
| AC-ON-004 | First feed loads within 3 seconds of completion | Performance | Feed renders with ≥20 product cards within 3,000ms on 4G. |
| AC-ON-005 | Swipe mechanic presents 10 visually diverse cards | Functional | No two cards from same brand; visual diversity verified. |
| AC-ON-006 | Taste vector written on completion | Integration

---

# 4. Home / Feed (FR-FD)

*No structural changes from FRD v1 Section 4. Feed composition rules, product card spec, like/dismiss/wishlist interactions, pull-to-refresh, feedback prompt, and new arrivals badge remain as specified.*

Key updates:
- Feed is powered by taste vectors by occasion (not rules-based scoring)
- Occasion filter toggle added above feed: Everyday / Going Out / Work / Special
- Interactions logged include passive signals: dwell time tracked via IntersectionObserver on feed cards (≥2s threshold for logging, ≥5s for meaningful signal)

All FR-FD requirements from v1.0 remain valid. Add:

**FR-FD-009 — Occasion Filter**
A lightweight toggle row above the feed: Everyday / Going Out / Work / Special / All (default). Selecting an occasion filters For You results to products tagged for that occasion. Interactions during a filtered session update the occasion-specific sub-vector.

**FR-FD-010 — Dwell Time Tracking**
IntersectionObserver tracks how long each product card is visible in the viewport. Dwell under 2s: ignored. Dwell 2–5s: logged as weak signal (weight 0.15). Dwell 5s+: logged as meaningful signal (weight 0.4). Logged to user_interactions with dwell_seconds field.

---

# 5. Search (FR-SR)

*No changes from FRD v1.0 Section 5. Search entry, autocomplete, results, filters, sort, result cards, and recent searches remain as specified. Search is powered by Marqo vector search (natural language) rather than traditional text search.*

---

# 6. Product Detail Page (FR-PD)

*Page composition and most functional requirements remain from FRD v1.0 Section 6, with one key change: "Buy Now" is replaced by "Add to Bag".*

Updates:

**FR-PD-006 — Add to Bag (replaces Buy Now)**
Tapping "Add to Bag" requires a size to be selected first. If size is selected, the system calls the Shopify tokenless cart API for the product's brand store — creating or updating a cart object — and adds the item to the STROBE bag. A confirmation toast appears: "Added to bag". The user remains on the product detail page. A bag icon in the header shows the current item count.

If product is out of stock entirely, "Add to Bag" is replaced with "Notify Me When Back In Stock" (unchanged from v1.0 FR-PD-007).

**FR-PD-008 — Interaction Logging**
Navigating to a product detail page logs a `tap` interaction to user_interactions. Time spent on the page is tracked and logged as dwell time on exit.

All other PDP requirements (image gallery, price display, retailer info, size selector, wishlist, recommendations) remain as specified in v1.0.

---

# 7. Wishlist (FR-WL)

*No changes from FRD v1.0 Section 7. List view, sorting/filtering, price change indicator, item removal, size selection, share functionality, and price drop notification logic remain as specified.*

---

# 8. Bag & Checkout Handoff (FR-BG) — REWRITTEN

## 8.1 Overview

The Bag is STROBE's native cart experience. It groups items by brand (each brand = one Shopify cart object) and provides per-brand checkout handoff to Shopify's hosted checkout. STROBE does not process payments.

## 8.2 Bag Flow

| Step | User Action | System Response |
|------|-------------|-----------------|
| 1 | Taps "Add to Bag" on product detail (size selected) | Shopify tokenless cart API called for that brand. Cart object created or updated. checkoutUrl returned and stored. Item appears in bag. |
| 2 | Opens bag (bag icon in header) | Bag displays items grouped by brand. Each brand section shows: brand name, line items (image, title, size, price), subtotal, "Checkout with [Brand]" button. |
| 3 | Taps "Checkout with [Brand]" | checkout_initiated event logged. Cart status updated. Shopify checkoutUrl opened in browser. |
| 4 | Completes payment on Shopify checkout | Payment handled entirely by Shopify. STROBE has no visibility unless brand provides order webhook. |

## 8.3 Functional Requirements

**FR-BG-001 — Bag Display**
Bag shows all active cart items grouped by brand. Each brand section displays: brand name/logo, line items with image (thumbnail), product title, selected size, price (price at time of add to prevent display inconsistency), and a remove button (×). Brand subtotal shown. "Checkout with [Brand]" primary CTA per brand section.

UI States: Loading / Populated / Empty.
Empty state: "Your bag is empty. Find something you love." with "Browse Feed" CTA.

**FR-BG-002 — Multi-Brand Grouping**
When the bag contains items from multiple brands, each brand is a distinct section with its own checkout button. A note is displayed: "Each brand checks out separately." This is transparent — no false promise of a unified checkout.

**FR-BG-003 — Add to Bag**
On product detail page: size/variant selector connected to Shopify variant IDs. "Add to Bag" calls Shopify tokenless cart API for the product's brand store. Creates or updates the cart object. Stores returned checkoutUrl. Confirmation toast: "Added to bag". Bag icon count increments.

Error states: If Shopify cart API fails: toast "Couldn't add to bag. Please try again." If variant is out of stock (availableForSale = false): size shown as greyed/unavailable, cannot be added.

**FR-BG-004 — Remove from Bag**
Tapping × on a line item removes it from the Shopify cart (via cart API mutation) and from the bag display. If the last item from a brand is removed, the brand section is removed. Optimistic UI with undo snackbar (4 seconds).

**FR-BG-005 — Checkout Handoff**
Tapping "Checkout with [Brand]" opens the stored checkoutUrl in the browser. Before opening: log checkout_initiated event to cart_events (user_id, cart_id, product_id, brand_id, session_id). Update cart status to checkout_initiated.

**FR-BG-006 — Cart Persistence**
Cart state persisted in localStorage for unauthenticated users and in the carts table for authenticated users. Cart survives page refresh and app close. Cart items expire after 7 days.

**FR-BG-007 — Cart Events Instrumentation**
Every cart action is logged to cart_events: add, remove, update_qty, checkout_initiated. Each event includes user_id (nullable for guests), product_id, brand_id, session_id, timestamp. This is the commercial data layer.

**FR-BG-008 — Variant/Size Availability**
Before adding to bag, check that the selected variant is in stock via Shopify API (availableForSale field). Show out-of-stock state on size selector. Do not allow out-of-stock variants to be added.

## 8.4 Acceptance Criteria

| AC ID | Criterion | Pass Condition |
|-------|-----------|----------------|
| AC-BG-001 | Adding to bag creates/updates a Shopify cart object | Shopify cart API returns valid cart with correct line items. |
| AC-BG-002 | Multi-brand bag displays brands separately | Items from 3 different brands show 3 distinct sections with separate checkout buttons. |
| AC-BG-003 | Checkout handoff opens correct Shopify checkout | checkoutUrl opens with correct items pre-populated. |
| AC-BG-004 | checkout_initiated event logged on every handoff | cart_events table contains event with correct product_id, brand_id, session_id. |
| AC-BG-005 | Cart persists across page refresh | Refresh page; bag contents unchanged. |
| AC-BG-006 | Out-of-stock variants cannot be added | Tapping an OOS size has no effect; "Add to Bag" is blocked. |

---

# 9. Orders / Checkout History (FR-OR) — REVISED

## 9.1 Overview

The Orders tab displays the user's checkout history — events where STROBE handed off to a brand's Shopify checkout. In v0, STROBE does not own fulfilment or have visibility into order completion unless a brand provides an order webhook. The primary data is checkout_initiated events, supplemented by confirmed_orders where brand webhook relationships exist.

## 9.2 Functional Requirements

**FR-OR-001 — Checkout History List**
Orders tab displays checkout-initiated events in reverse chronological order. Each entry shows: product image (thumbnail), product name, brand, size, price at time of add, date/time of checkout initiation, and status.

Status values:
- "Checkout Started" (default — STROBE handed off to Shopify)
- "Order Confirmed" (only if brand webhook confirms the order)
- "Shipped" / "Delivered" (only if brand provides tracking webhook)

UI States: Loading / Has entries / Empty.
Empty state: "No orders yet. Find something you love and check out in seconds." with "Start Browsing" CTA.

**FR-OR-002 — Limited Visibility Disclaimer**
A subtle note at the top of the Orders tab: "Order status is provided by each brand. For detailed tracking, visit the brand's website." This sets expectations that STROBE's visibility is limited.

**FR-OR-003 — Confirmed Order Detail (where available)**
If a brand has provided a confirmed order via webhook, the entry shows: order value, confirmed timestamp, and a "Track on [Brand]" link opening the brand's order tracking page.

## 9.3 Acceptance Criteria

| AC ID | Criterion | Pass Condition |
|-------|-----------|----------------|
| AC-OR-001 | Every checkout_initiated event appears in Orders tab | All cart_events with type checkout_initiated are listed. |
| AC-OR-002 | Confirmed orders display enriched status | Where confirmed_orders data exists, entry shows "Order Confirmed" status. |

---

# 10. Profile & Account Settings (FR-PR)

*No structural changes from FRD v1.0 Section 10. Profile header, style profile view, notification settings, data & privacy, account deletion, and photo upload remain as specified.*

Key updates:
- Style Profile View (FR-PR-001) shows style summary derived from tag_weights on user_taste_profiles, including occasion breakdown
- "Edit Preferences" navigates to the editable onboarding preferences (6-step flow, pre-populated with current values)
- Payment Methods section is removed (STROBE does not store payment data in v0)
- Addresses section is removed (STROBE does not manage shipping in v0)

---

# 11. Notifications (FR-NT)

*Notification types, triggers, and in-app centre remain as specified in FRD v1.0 Section 11, with the following notes:*

- Push notification delivery (APNs, FCM) is deferred to when native apps are built. In v0, notifications are in-app only (notification centre accessible via bell icon).
- Order Update notifications are limited to checkout_initiated confirmations and confirmed_orders where webhook data exists.

---

# 12. Global & Shared Behaviours (FR-GS)

*No changes from FRD v1.0 Section 12. Offline & connectivity, API error handling, loading states (skeleton screens, button loading), accessibility requirements (screen reader, touch targets, contrast, dynamic type), and analytics event tracking remain as specified.*

Updates to analytics events:

| Event Name | Trigger | Required Properties |
|------------|---------|---------------------|
| product_viewed | User opens Product Detail Page | product_id, brand_id, price, source (feed/search/wishlist/recommendation) |
| product_liked | User likes a product | product_id, brand_id, source, feed_position |
| product_dismissed | User dismisses a product | product_id, brand_id, source, feed_position |
| wishlist_added | User adds product to Wishlist | product_id, brand_id, price, source |
| wishlist_removed | User removes product from Wishlist | product_id |
| bag_added | User adds product to bag | product_id, brand_id, price, size, source |
| bag_removed | User removes product from bag | product_id, brand_id |
| checkout_initiated | User taps "Checkout with [Brand]" | product_id, brand_id, cart_id, session_id |
| search_query | User submits a search | query_text, result_count, filters_applied |
| dwell_time | Product card visible ≥2s in viewport | product_id, dwell_seconds, source, feed_position |
| onboarding_completed | User finishes onboarding | time_to_complete_seconds, steps_completed, seed_method |
| onboarding_step_completed | User completes an onboarding step | step_number, step_name, time_on_step_seconds |
| onboarding_abandoned | User exits onboarding | step_abandoned |
| notification_opened | User taps a notification | notification_type, product_id |

---

# Appendix — Original In-App Checkout Spec (v1.0, archived)

The original FRD v1.0 Section 8 (Checkout) specified a full in-app purchase flow with Stripe payment processing, including: shipping address management, credit/debit card tokenisation, Apple Pay / Google Pay / Afterpay, order review with tax calculation, payment processing with idempotency, and order confirmation. This specification is retained for future reference if/when STROBE pursues in-app checkout. Key dependencies: PCI DSS Level 1 compliant payment processor, Stripe integration, brand-by-brand payment agreements. See also: Backend Map (archived) for the full Orders, Notifications, and Payment schemas.

---

*STROBE Consumer App FRD · v2 · April 2026 · US Market · Confidential*
