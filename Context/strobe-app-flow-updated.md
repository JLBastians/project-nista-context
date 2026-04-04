# STROBE — Application Flow Documentation
> v1.1 · April 2026 · US Market · Confidential
> Aligned with Master Directory v1.1, Architecture v4.1, Onboarding Design Doc v1.0
> Based on Next.js App Router, Shopify tokenless cart API, Marqo + Qwen enrichment

> **v1.1 update notes:** Onboarding rewritten to 6-step flow (age → gender expression → city → activities → brands → swipe). Checkout rewritten as Bag + Shopify checkout handoff (replaces Stripe in-app checkout). Orders revised as Checkout History. Product detail "Buy Now" replaced with "Add to Bag". Interaction signals (explicit + passive) documented throughout. All flows aligned with Architecture v4.1.

---

## Table of Contents

1. [Entry Points](#1-entry-points)
2. [Core User Flows](#2-core-user-flows)
3. [Navigation Map](#3-navigation-map)
4. [Screen Inventory](#4-screen-inventory)
5. [Decision Points](#5-decision-points)
6. [Error Handling](#6-error-handling)
7. [Responsive Behavior](#7-responsive-behavior)
8. [Animations & Transitions](#8-animations--transitions)

---

## 1. Entry Points

### 1.1 Direct URL Access

| URL | Destination | Auth Required |
|-----|-------------|---------------|
| `strobelab.co` | Landing page (`/`) — static marketing page, waitlist signup | No |
| `strobelab.co/welcome` | App welcome screen | No |
| `strobelab.co/feed` | Personalised discovery feed | Yes → redirect to `/login` |
| `strobelab.co/search` | Search page | Yes → redirect to `/login` |
| `strobelab.co/product/[id]` | Product detail page | Yes → redirect to `/login` |
| `strobelab.co/bag` | STROBE bag (multi-brand cart) | Yes → redirect to `/login` |
| `strobelab.co/wishlist` | Wishlist | Yes → redirect to `/login` |
| `strobelab.co/orders` | Checkout history | Yes → redirect to `/login` |
| `strobelab.co/profile` | User profile | Yes → redirect to `/login` |

### 1.2 Deep Links

| Trigger | Deep Link Pattern | Landing Destination |
|---------|-------------------|---------------------|
| Price drop notification | `/product/[id]?ref=price_drop` | Product detail page |
| Restock alert | `/product/[id]?ref=restock` | Product detail page |
| Wishlist reminder | `/wishlist?ref=reminder` | Wishlist page |
| Onboarding incomplete reminder | `/onboarding?ref=reminder` | Onboarding flow |
| Marketing email CTA | `/welcome?ref=email_campaign` | Welcome screen |

### 1.3 OAuth / Social Login

| Provider | Entry Flow |
|----------|------------|
| Apple Sign-In | `/login` or `/register` → Apple OAuth → callback → `/feed` (existing) or `/onboarding` (new) |
| Google Sign-In | `/login` or `/register` → Google OAuth → callback → `/feed` (existing) or `/onboarding` (new) |

**Fallback:** If OAuth fails, user is returned to `/login` or `/register` with error message: `"Sign-in failed. Please try again or use email."` Email/password form available as fallback.

### 1.4 Search Engines

Marketing landing page (`strobelab.co`) is publicly indexed. All app routes (`/feed`, `/search`, etc.) are behind auth and return `noindex` meta tags.

### 1.5 Marketing Campaigns

| Campaign Type | UTM Parameters | Landing Page |
|---------------|----------------|--------------|
| Social ad (new user) | `?utm_source=instagram&utm_medium=paid` | `strobelab.co` (landing page) |
| Email campaign (existing user) | `?utm_source=email&utm_medium=newsletter` | `/welcome` or deep link |
| Influencer / referral | `?ref=[handle]` | `strobelab.co` (landing page) |

---

## 2. Core User Flows

---

### 2.1 User Registration & Onboarding

#### HAPPY PATH

```
/welcome → /register → /onboarding (6 steps) → /feed
```

**Step 1 — Welcome Screen (`/welcome`)**
- UI: STROBE logo with glow animation, wordmark, value proposition copy, feature pills (Personalised Feed, 500+ Brands, Price Tracking), `Create Account` CTA (primary), `Sign In` CTA (secondary), legal disclaimer
- User action: Taps **Create Account**
- System response: Navigate to `/register`

**Step 2 — Registration (`/register`)**
- UI: Logo, first name field, last name field, email field, password field with strength indicator + show/hide toggle, Apple sign-in button, Google sign-in button, link to `/login`
- User inputs: First name, last name, email, password
- Validation rules:
  - First name: non-empty string
  - Last name: non-empty string
  - Email: must contain `@`
  - Password: ≥8 chars, ≥1 uppercase, ≥1 number (live strength bar shows progress)
- System response: Loading spinner on submit button → `register()` called → navigate to `/onboarding`
- Success criteria: User object created in store with email, firstName, lastName; `isAuthenticated: true`

**Step 3 — Onboarding (`/onboarding`)**

Six-step flow designed to power the recommendation engine. Every question directly feeds the taste vector personalisation engine.

- **Screen 1 — Age:** Birth year picker/dropdown. Framing: "How old are you? We'll use this to tailor your edit." Validates age 13–100. Stored as date_of_birth. Skippable.
- **Screen 2 — Gender expression:** Multi-select four options: Womenswear / Menswear / Androgynous + gender-neutral / I dress across all of the above. Framing: "How do you like to dress?" At least 1 selection required if not skipped. Stored as gender_expression array.
- **Screen 3 — City:** IP geolocation pre-fill (detected city shown as confirmation). Manual override via text input with autocomplete (US cities). Stored as city + city_source ("ip" or "manual"). Skippable.
- **Screen 4 — Lifestyle activities:** Multi-select activity grid (14 activities: everyday/errands, office work, creative workplace, WFH, dinner out, cocktail events, formal events, travel, gym, outdoor, beach/resort, festivals, date nights, weddings). For each selected activity, frequency selector: Often (weight 1.0) / Sometimes (0.5) / Rarely (0.15). Frequency maps to occasion taste vector weighting. Min 1 selection if not skipped. Stored as occasion_weights JSONB.
- **Screen 5 — Recently shopped brands:** Searchable brand list with autocomplete + curated quick-select grid (~30 brands). 3–10 selections. "I shop vintage/secondhand primarily" option. On completion: cross-reference sustainability taxonomy → sustainability_orientation. Average brand style vectors → brand-derived style vector (weak prior). Stored in onboarding_brand_selections.
- **Screen 6 — Swipe mechanic:** Full-screen card stack. 10 product cards from STROBE catalogue (selected algorithmically by cofounder's engine using occasion weights from Screen 4 + brand-prior from Screen 5 + gender expression from Screen 2; cards distributed across likely style space, not clustered). Swipe right = like, swipe left = pass. Binary forced choice, no skip on individual cards. After 10 swipes: right-swiped vectors averaged with +1.0 weight, left-swiped with −0.3, blended 60/40 with brand prior → initial taste_vector written to user_taste_profiles. Stored in onboarding_swipe_responses.

- System response on completion: Loading state ("We're building your feed...") for up to 3 seconds → navigate to `/feed`
- Subtle message on first feed: "Your edit is ready — it'll keep getting better as you explore."

**Step 4 — Feed (`/feed`)**
- UI: Sticky header with logo + notification bell, "For You" heading, occasion filter toggle (Everyday / Going Out / Work / Special / All), 2-column product grid, skeleton cards on load
- System response: skeleton load → products render with stagger animation
- After 8 interactions: Tooltip appears — `"Your feed is learning your taste — the more you engage, the better it gets."`

#### ERROR STATES

| Error | Trigger | Display | Recovery |
|-------|---------|---------|----------|
| Apple/Google OAuth failure | Provider auth error | Toast: `"Sign-in failed. Please try again or use email."` | Email form visible below |
| Email already registered | Duplicate email on register | Inline error below email field: `"An account with this email already exists."` | Link to `/login` |
| Weak password | Submit with failing rules | Strength bar turns red; rules checklist highlights failing items | User must fix password |
| Network error on register | API timeout | Toast: `"Something went wrong. Please check your connection and try again."` | Retry button |
| Onboarding skipped entirely | User navigates directly to `/feed` | Generic trending feed displayed; persistent banner: `"Complete your profile to personalise your feed."` | Banner CTA → `/onboarding` |
| IP geolocation fails (Step 3) | Geolocation API error | City field is empty with placeholder "Enter your city". No error shown. | User types city manually |
| Brand search returns no match (Step 5) | User types brand not in catalogue | "No matching brands" message in dropdown. User can continue without that brand. | Select other brands or skip |
| Swipe card generation fails (Step 6) | Insufficient products matching criteria | Fall back to trending/editorial product cards for swipe. Note: "We're still building your catalogue — these are some of our favourites." | Swipe proceeds with fallback cards |
| Feed fails to load | Network error | Skeleton screen persists; retry button appears after 5s: `"Failed to load feed. Tap to retry."` | Retry button |
| Feed personalisation fails | Engine error | Show trending feed with banner: `"We had trouble personalising your feed. We'll keep learning as you browse!"` | Banner dismissible |

#### EDGE CASES

- **User abandons registration mid-form:** Form state not persisted; refreshing clears fields. No partial account created until submission.
- **User navigates back during onboarding:** Back navigation returns to previous step. All prior selections preserved.
- **User skips individual onboarding steps:** Skip link on each step. Skipped steps store no data. Feed quality degrades proportionally. Persistent "complete your profile" banner on feed.
- **User skips onboarding entirely:** `IF onboarding skipped THEN show trending feed + persistent profile completion banner`
- **Session expires during onboarding:** User redirected to `/login`; on re-login, onboarding resumes from beginning (no partial state saved in v1.0).
- **User who skipped returns to complete onboarding later:** Profile → "Complete Your Profile" navigates to `/onboarding` with previously completed steps shown as complete, resuming from first incomplete step.

---

### 2.2 Authentication (Login)

#### HAPPY PATH

```
/welcome → /login → /feed
```

**Step 1 — Login (`/login`)**
- UI: Logo, email field with mail icon, password field with lock icon + show/hide toggle, Forgot Password link → `/reset-password`, Apple + Google sign-in buttons, link to `/register`
- Validation:
  - Email: must contain `@`
  - Password: ≥6 characters
  - Submit button disabled until both rules pass
- System response: loading spinner → `login(email)` → navigate to `/feed`
- Attempt limit: 5 failed attempts locks form with message: `"Too many failed attempts. Please try again later."`

#### ERROR STATES

| Error | Message Shown | Recovery |
|-------|--------------|----------|
| Invalid credentials | `"Invalid email or password."` | Retry; attempt counter increments |
| Too many attempts (≥5) | `"Too many failed attempts. Please try again later."` | Wait (no timer shown in v1.0) |
| OAuth failure | `"Sign-in failed. Please try again or use email."` | Use email form |
| Network error | `"Something went wrong. Please check your connection."` | Retry |

#### EDGE CASES

- **Already authenticated user visits `/login`:** Redirect to `/feed`
- **Session expires mid-session:** Any protected route redirects to `/login`; intended destination not preserved in v1.0

---

### 2.3 Password Reset

#### HAPPY PATH

```
/login → /reset-password → [email sent state] → /login
```

**Step 1 — Reset Request (`/reset-password`)**
- UI: Logo, email input with mail icon, `Send Reset Link` button, back link to `/login`
- Validation: Email must contain `@`
- System response: loading → success state shown

**Step 2 — Success State**
- UI: Green checkmark icon, `"Check Your Email"` heading, confirmation copy with email address, `Back to Sign In` button → `/login`
- Security: Same success message shown regardless of whether email exists (prevents account enumeration)

#### ERROR STATES

| Error | Message | Recovery |
|-------|---------|----------|
| Invalid email format | Button disabled (form validation) | Enter valid email |
| Network error | Toast: `"Failed to send reset email. Try again."` | Retry |

---

### 2.4 Product Discovery (Feed)

#### HAPPY PATH

```
/feed → scroll/interact → product card tap → /product/[id]
```

**Feed Page (`/feed`)**
- Sticky header: STROBE logo + notification bell (with unread badge)
- Occasion filter toggle: Everyday / Going Out / Work / Special / All (default)
- Products load with skeleton → stagger animation on render
- User can: like (heart icon), save to wishlist (bookmark icon), tap card → product detail, dismiss product (long-press or swipe), load more (infinite scroll)

**Interaction signals captured on feed:**
- **Like:** Heart icon tap → `user_interactions` (type: like, weight 0.8)
- **Dismiss:** Long-press ≥500ms or swipe left → "Not for me" → `user_interactions` (type: dismiss, weight −0.3). Product removed with collapse animation. Undo snackbar for 4 seconds.
- **Wishlist add/remove:** Bookmark icon toggle → `user_interactions` (type: save, weight 1.0) + `user_saved_items`
- **Dwell time:** IntersectionObserver tracks viewport time per card. Under 2s: ignored. 2–5s: logged (weight 0.15). 5s+: logged (weight 0.4). Written to `user_interactions` (type: dwell, dwell_seconds field).
- **Product detail tap:** Card tap → navigation logged as `user_interactions` (type: tap, weight 0.6)

**Occasion filter behaviour:**
- Selecting an occasion filters For You results to products tagged for that occasion
- Interactions during a filtered session feed the occasion-specific sub-vector (e.g. interactions while "Going Out" is active update going_out_vector)
- "All" shows the full personalised feed using the main taste_vector

**Load more:** Infinite scroll; next batch of 20 cards pre-fetched when user is within 5 cards of list end.

**Empty State:** All products dismissed: `"No more items. Pull down to refresh for new recommendations."`

#### ERROR STATES

| Error | Display | Recovery |
|-------|---------|----------|
| Feed fails to load | Skeleton persists → retry message after 5s | Retry button reloads feed |
| Like/wishlist action fails | Toast: `"Action failed. Please try again."` UI reverts optimistic update. | User can retry |
| Dismiss undo fails | Product remains hidden. Toast: `"Unable to undo — this item stays hidden for now."` | — |

---

### 2.5 Product Search

#### HAPPY PATH

```
/search → type query → results → filter/sort → product tap → /product/[id]
```

**Pre-Search State**
- Shows recent searches (clearable) + "Trending on STROBE" pill grid
- Tapping a trending item or recent search populates query and triggers search

**Active Search**
- Query ≥2 characters triggers live results via Marqo vector search (natural language semantic matching)
- Filters available: Category (multi-select chips), Brand (multi-select chips), Price range (dual sliders, $0–$2000), Aesthetic (from product_tags), Occasion (from product_tags), Sale only (toggle)
- Sort options: Best Match (default — Marqo relevance + personalisation), New Arrivals, Price: Low to High, Price: High to Low
- Active filter chips displayed below search bar; each dismissible individually
- Filter count badge on filter button icon

**Results State**
- `[n] result(s)` count shown
- 2-column product grid with stagger animation
- Zero results: Search icon + `"No Results"` heading + `"Try different keywords or adjust your filters."`

**Interaction signals captured on search:**
- Search query logged (query_text, result_count, filters_applied)
- All product interactions (like, wishlist, tap, dwell) logged as on feed, with source: "search"

#### ERROR STATES

| Error | Display | Recovery |
|-------|---------|----------|
| No results | Empty state with search icon | Adjust query or clear filters |
| Network error | Toast error | Retry search |

#### EDGE CASES

- **Filter with no results:** Empty state shown; `Clear All` link removes all filters
- **User clears query:** Results cleared, pre-search state returns
- **Recent searches persist:** Stored locally; cleared on explicit `Clear` tap

---

### 2.6 Product Detail

#### HAPPY PATH

```
/feed or /search → product card tap → /product/[id] → select size → Add to Bag → stays on page
```

**Product Detail Page (`/product/[id]`)**
- Image carousel with dot indicators (tap to navigate)
- Badges: New (ingested within 48hrs), Sale, Match Score (≥90%)
- Brand name (tappable → brand filter search results)
- Product name (full, no truncation)
- Price — sale price in danger red + original crossed out + % off badge
- Size selector: Available sizes as buttons; user's stored size highlighted (not pre-selected); out-of-stock sizes greyed with strikethrough, cannot be selected
- Description text
- Product details accordion (expand/collapse)
- Tags row (from product_tags, confidence > 0.5)
- "You Might Also Like" — 4 similar products (Marqo nearest-neighbour from same vector space)
- "From [Brand]" — up to 8 other products from same brand
- Sticky bottom bar: Wishlist bookmark button + `Add to Bag - $[price]` CTA

**Interaction signals captured:**
- Page load: `user_interactions` (type: tap, source: feed/search/wishlist/recommendation)
- Time on page: tracked, logged as dwell on exit
- Like: heart icon in header
- Wishlist: bookmark in sticky bar
- Add to Bag: logged to both `cart_events` (type: add) and `user_interactions` (type: add_to_bag, weight 0.9)

**Wishlist action:** Toggle; if not wishlisted → toast `"Saved to wishlist"`; if wishlisted → toast `"Removed from wishlist"`. If added without size selected, wishlisted without size (size settable from wishlist later).

**Share action:** Toast `"Link copied to clipboard"`

**Add to Bag validation:**
- If no size selected → button shakes (`animate-shake`) + toast error: `"Please select a size"`
- If size selected → Shopify tokenless cart API called for that brand → cart created/updated → checkoutUrl stored → toast `"Added to bag"` → bag icon in header updates count → user stays on product detail page
- If variant out of stock (availableForSale = false) → size greyed out, cannot be selected

**Out of stock (all sizes):** "Add to Bag" replaced with "Notify Me When Back In Stock". Tapping subscribes to restock alert. Confirmation: "We'll notify you when this comes back."

#### ERROR STATES

| Error | Display | Recovery |
|-------|---------|----------|
| Product not found | `"Product not found"` centered message | Back navigation |
| Add to Bag fails (Shopify API error) | Toast: `"Couldn't add to bag. Please try again."` | Retry |
| Variant no longer available | Toast: `"This size is no longer available."` Size refreshes. | Select another size |

#### EDGE CASES

- **User taps same size twice:** Deselects size (toggle behaviour)
- **Deep link to invalid product ID:** Not found state shown
- **User taps Add to Bag for same product twice:** Quantity updates in existing cart (or duplicate prevented)
- **Product images fail to load:** Branded placeholder (STROBE logo on grey) in carousel slot

---

### 2.7 Bag & Checkout Handoff

#### HAPPY PATH

```
/product/[id] → Add to Bag → /bag → Checkout with [Brand] → Shopify hosted checkout (external)
```

**Bag Page (`/bag`)**
- Header: "Your Bag" + total item count
- Items grouped by brand. Each brand section displays:
  - Brand name / logo
  - Line items: product image (thumbnail), product name, selected size, price (price_at_add to prevent inconsistency if price changes), remove button (×)
  - Brand subtotal
  - `"Checkout with [Brand]"` primary CTA button
- If multiple brands: note displayed at top: `"Each brand checks out separately."`

**Remove item:** Tapping × removes item from Shopify cart (cart API mutation) and from bag display. If last item from a brand, brand section removed. Optimistic UI with undo snackbar (4 seconds). Logged to cart_events (type: remove).

**Checkout handoff:** Tapping `"Checkout with [Brand]"`:
1. Log checkout_initiated event to cart_events (user_id, cart_id, product_id, brand_id, session_id) — this is the strongest purchase-intent signal and the primary commercial metric
2. Also log to user_interactions (type: checkout_initiated, weight 1.0) for taste vector
3. Update cart status to checkout_initiated
4. Open stored checkoutUrl in browser (Shopify hosted checkout with cart pre-populated)
5. User completes payment on Shopify — STROBE has no further visibility unless brand provides order webhook

**Cart persistence:** localStorage for unauthenticated users, carts table for authenticated users. Survives page refresh and app close. Cart items expire after 7 days.

**Empty State:** `"Your bag is empty. Find something you love."` with `"Browse Feed"` CTA.

#### ERROR STATES

| Error | Display | Recovery |
|-------|---------|----------|
| Bag empty | Empty state with Browse Feed CTA | Navigate to `/feed` |
| Shopify cart API fails on remove | Item re-appears. Toast: `"Couldn't remove item. Try again."` | Retry |
| checkoutUrl expired or invalid | Toast: `"Checkout link expired. We're refreshing it."` System re-fetches checkoutUrl via cart API. | Automatic retry; user taps checkout again |
| Network error during checkout handoff | Toast: `"Couldn't open checkout. Check your connection."` | Retry |

#### EDGE CASES

- **User adds items from 3+ brands:** Bag shows 3 distinct brand sections, each with own checkout button. No limit on brand count.
- **User removes all items from one brand:** Brand section disappears from bag.
- **Shopify cart mutated after checkoutUrl generated:** checkoutUrl refreshed on every cart mutation (add/remove) to prevent stale state.
- **Guest user taps checkout:** Prompt to sign up. `"Create an account to check out."` Cart preserved in localStorage.
- **User navigates back from Shopify checkout:** Returns to STROBE bag. Cart status remains checkout_initiated. Bag still shows items.

---

### 2.8 Wishlist Management

#### HAPPY PATH

```
/product/[id] → bookmark tap → /wishlist
```

**Wishlist (`/wishlist`)**
- Header: `"Wishlist"` + item count
- Sort selector: Date Added (default), Price: Low to High, Price: High to Low
- Filter chips: by category, by brand
- 2-column product grid; price drop badge overlaid if `product.price < priceAtAdd`
- Each card: product image, brand, name, current price, original price if changed, size (if set), remove (×) button
- If no size set: "Select Size" chip on card; tapping opens inline size picker
- Empty state: Heart icon + `"Your Wishlist is Empty"` + `"Browse Feed"` CTA

**Price drop detection:** On render, compare current `product.price` against `priceAtAdd` stored at time of wishlist add. If lower → green `"Price Dropped"` badge with old and new price.

**Share Wishlist:** Top-right button generates shareable link (strobe.com/wishlist/[user-token]). Read-only public view, no login required. Token regenerable (invalidates previous link).

#### ERROR STATES

| Error | Display |
|-------|---------|
| Product no longer in catalogue | Product silently omitted from wishlist render |
| Empty wishlist | Empty state with Browse Feed CTA |
| Wishlist remove fails | Item re-appears. Toast: `"Couldn't remove item. Try again."` |

---

### 2.9 Orders / Checkout History

#### HAPPY PATH

```
/bag → Checkout with [Brand] → /orders
```

**Orders (`/orders`)**
- List of checkout events in reverse chronological order (most recent first)
- Each entry shows: product image (thumbnail), product name, brand, size, price, date/time of checkout initiation, status badge
- Status values:
  - `"Checkout Started"` (grey) — default, STROBE handed off to Shopify
  - `"Order Confirmed"` (green) — only if brand webhook confirms order
  - `"Shipped"` (orange) — only if brand provides tracking webhook
  - `"Delivered"` (green) — only if brand provides tracking webhook
- Subtle note at top: `"Order status is provided by each brand. For detailed tracking, visit the brand's website."`
- Where confirmed order exists (webhook data): shows order value + "Track on [Brand]" link
- Empty state: `"No orders yet. Find something you love and check out in seconds."` with `"Start Browsing"` CTA

#### ERROR STATES

| Error | Display | Recovery |
|-------|---------|----------|
| Orders fail to load | Toast: `"Unable to load order history. Pull to refresh."` | Pull to refresh |
| Empty orders | Empty state with Start Browsing CTA | Navigate to `/feed` |

---

### 2.10 Account Management

#### Profile (`/profile`)
- Display: Name, email, avatar placeholder, member since date
- Style profile summary card: top aesthetic (from tag_weights), preferred occasions, top brands, price range — human-readable. E.g. "You gravitate toward minimalist, earth tones, oversized silhouettes." Updates each session.
- "Edit Preferences" → navigates to editable onboarding preferences (6-step flow, pre-populated with current values)
- "Complete Your Profile" → shown if onboarding incomplete; navigates to `/onboarding`
- Notification settings: toggles per notification type
- Data & Privacy: Download My Data, Delete Account, Opt Out of Personalisation
- Logout CTA → clears auth state → redirect to `/welcome`

*Note: No Payment Methods or Addresses sections in v0 (STROBE does not store payment data — Shopify handles checkout).*

#### Notifications (`/notifications`)
- Accessed via bell icon in feed header (not a bottom nav tab)
- List of in-app notifications: price drops on wishlisted items, restock alerts, new arrivals matching style
- Unread count badge on bell icon
- Mark all as read action
- Each notification tappable → deep links to relevant product detail page
- Empty state: `"No Notifications"`

*Note: Push notification delivery (APNs/FCM) deferred to native app build. v0 is in-app notification centre only.*

---

## 3. Navigation Map

```
strobelab.co (Landing Page — public, JavaScript, main branch, separate codebase)
│
└── /welcome  [PUBLIC]
    ├── /register  [PUBLIC]
    │   └── /onboarding  [AUTHENTICATED — new users only]
    │       └── /feed  ←────────────────────────────────┐
    └── /login  [PUBLIC]                                 │
        └── /reset-password  [PUBLIC]                    │
                                                         │
AUTHENTICATED APP (requires login)                       │
├── /feed  [AUTHENTICATED — default landing] ────────────┘
│   └── /product/[id]  [AUTHENTICATED]
│       ├── Add to Bag → stays on page (bag icon updates)
│       └── View Bag → /bag
│
├── /search  [AUTHENTICATED]
│   └── /product/[id]  [AUTHENTICATED]
│
├── /bag  [AUTHENTICATED]
│   └── Checkout with [Brand] → opens Shopify checkoutUrl (external browser)
│
├── /wishlist  [AUTHENTICATED]
│   └── /product/[id]  [AUTHENTICATED]
│
├── /orders  [AUTHENTICATED]
│
├── /profile  [AUTHENTICATED]
│   └── Edit Preferences → /onboarding (pre-populated)
│
└── /notifications  [AUTHENTICATED — via bell icon]

BOTTOM NAV (visible on all authenticated routes except /bag, /product/[id], /onboarding)
  Feed → /feed
  Search → /search
  Wishlist → /wishlist
  Orders → /orders
  Profile → /profile
```

### Cross-Links Between Sections

| From | Action | To |
|------|--------|----|
| Feed product card | Tap card | `/product/[id]` |
| Feed product card | Bookmark icon | Adds to wishlist (stays on feed) |
| Feed product card | Heart icon | Likes product (stays on feed) |
| Product detail | Add to Bag | Stays on `/product/[id]`; bag icon updates |
| Product detail | Bookmark | Adds to wishlist (stays on detail) |
| Product detail | "You Might Also Like" card | `/product/[id]` (new product) |
| Product detail | Bag icon in header | `/bag` |
| Bag | Checkout with [Brand] | Shopify checkoutUrl (external) |
| Bag empty state | Browse Feed | `/feed` |
| Wishlist product card | Tap card | `/product/[id]` |
| Wishlist empty state | Browse Feed | `/feed` |
| Orders empty state | Start Browsing | `/feed` |
| Notification bell | Tap | `/notifications` |
| Notification item | Tap | `/product/[id]` (deep link) |
| Profile | Edit Preferences | `/onboarding` (pre-populated) |
| Profile | Complete Your Profile | `/onboarding` (resume) |
| Profile | Logout | `/welcome` |

---

## 4. Screen Inventory

### `/welcome`
- **Access:** Public
- **Purpose:** App entry point; convert visitor to registered user
- **Key UI:** Logo with glow, value proposition, feature pills, Create Account CTA, Sign In CTA
- **Actions:** Create Account → `/register`, Sign In → `/login`
- **States:** Single state

---

### `/register`
- **Access:** Public (redirect to `/feed` if already authenticated)
- **Purpose:** New user account creation
- **Key UI:** Name fields, email, password + strength indicator, Apple/Google SSO, link to login
- **Actions:** Submit → `/onboarding`, Apple/Google → OAuth flow, Login link → `/login`
- **States:** Default, loading, error (inline field errors, toast)

---

### `/login`
- **Access:** Public (redirect to `/feed` if already authenticated)
- **Purpose:** Returning user authentication
- **Key UI:** Email, password, forgot password link, Apple/Google SSO, register link
- **Actions:** Submit → `/feed`, Forgot Password → `/reset-password`, Register → `/register`
- **States:** Default, loading, error (inline + attempt counter), locked (≥5 attempts)

---

### `/reset-password`
- **Access:** Public
- **Purpose:** Initiate password reset via email
- **Key UI:** Email input, Send Reset Link button, back to login link
- **Actions:** Submit → success state, Back → `/login`
- **States:** Default, loading, success

---

### `/onboarding`
- **Access:** Authenticated (new users, or returning users completing profile)
- **Purpose:** 6-step style profile seeding to power recommendation engine
- **Key UI:**
  - Step 1: Birth year picker. Skippable.
  - Step 2: Gender expression multi-select (4 options). Skippable.
  - Step 3: City confirmation (IP pre-fill) or manual entry. Skippable.
  - Step 4: Lifestyle activities multi-select grid with frequency (Often/Sometimes/Rarely). Skippable.
  - Step 5: Brand multi-select (searchable + quick-select grid, 3–10 selections). Skippable.
  - Step 6: Swipe mechanic — 10 product card stack, swipe right/left. Not skippable per-card.
  - Progress indicator across all steps. Back/Next navigation. Skip link per step.
- **Actions:** Complete all steps → `/feed` (personalised), Skip → `/feed` (trending + banner)
- **States:** Step 1–6, loading (feed generation), skipped (generic feed)

---

### `/feed`
- **Access:** Authenticated
- **Purpose:** Personalised fashion discovery feed powered by taste vectors
- **Key UI:** Sticky header (logo + bag icon with count + bell icon with unread badge), occasion filter toggle, "For You" heading, 2-col product grid, infinite scroll
- **Actions:** Tap card → `/product/[id]`, like, bookmark (→ wishlist), dismiss, occasion filter, bag icon → `/bag`, bell → `/notifications`
- **Signals captured:** Like, dismiss, wishlist add/remove, dwell time (IntersectionObserver), product detail tap
- **States:** Loading (skeleton), populated, empty (all dismissed), error (retry)

---

### `/product/[id]`
- **Access:** Authenticated
- **Purpose:** Full product detail, size selection, add to bag
- **Key UI:** Image carousel, badges, brand/name/price, size selector, description, details accordion, tags (from product_tags), "You Might Also Like", "From [Brand]", sticky bottom bar (wishlist + Add to Bag)
- **Actions:** Back → previous page, like, share, wishlist toggle, size select, Add to Bag → stays on page (bag updates), bag icon → `/bag`
- **Signals captured:** Product detail view (tap), dwell time on page, like, wishlist, add to bag
- **States:** Loading, populated, not found, size shake (no size selected), out of stock (notify me)

---

### `/bag`
- **Access:** Authenticated
- **Purpose:** Multi-brand cart with per-brand checkout handoff to Shopify
- **Key UI:** Items grouped by brand, each with line items + subtotal + "Checkout with [Brand]" button. Multi-brand note. Remove (×) per item.
- **Actions:** Remove item, Checkout with [Brand] → Shopify checkoutUrl (external), Browse Feed (empty state)
- **Signals captured:** Remove from bag (cart_events), checkout_initiated (cart_events + user_interactions)
- **States:** Loading, populated (single brand), populated (multi-brand), empty

---

### `/search`
- **Access:** Authenticated
- **Purpose:** Cross-catalogue product discovery via Marqo semantic search with filtering
- **Key UI:** Search input, filter panel (category, brand, price, aesthetic, occasion, sale), results grid, sort dropdown, filter chips, trending pills, recent searches
- **Actions:** Search, filter, sort, tap result → `/product/[id]`, tap trending/recent → auto-search
- **Signals captured:** Search query, all product interactions (like, wishlist, tap, dwell)
- **States:** Pre-search (recent + trending), searching (live results), results, no results, error

---

### `/wishlist`
- **Access:** Authenticated
- **Purpose:** Saved products with price drop tracking
- **Key UI:** Header with count, sort/filter controls, 2-col grid, price drop badges, size selector chips, share button
- **Actions:** Tap card → `/product/[id]`, remove from wishlist, select size, share wishlist
- **States:** Populated, empty (with Browse Feed CTA)

---

### `/orders`
- **Access:** Authenticated
- **Purpose:** Checkout history — checkout-initiated events + confirmed orders where available
- **Key UI:** List with product thumbnail, name, brand, size, price, date, status badge. Limited visibility note.
- **Actions:** Tap entry → product detail (future: order detail), Start Browsing (empty state)
- **States:** Populated, empty

---

### `/profile`
- **Access:** Authenticated
- **Purpose:** Account management, style profile, preferences
- **Key UI:** Name, email, avatar, style summary card (from tag_weights), Edit Preferences, Complete Profile (if incomplete), notification toggles, data/privacy, logout
- **Actions:** Edit profile, Edit Preferences → `/onboarding`, logout → `/welcome`
- **States:** Default, editing

---

### `/notifications`
- **Access:** Authenticated (via bell icon, not bottom nav)
- **Purpose:** In-app notification centre — price drops, restocks, new arrivals
- **Key UI:** Notification list with icon, message, relative timestamp, read/unread state. Mark all read.
- **Actions:** Tap notification → deep link to relevant product, mark read
- **States:** Populated (unread/read), empty

---

## 5. Decision Points

### 5.1 Authentication Guard

```
IF user visits protected route (/feed, /search, /wishlist, /orders, /profile, /product/[id], /bag, /notifications)
  IF isAuthenticated === true
    THEN render requested page
  ELSE
    THEN redirect to /login
    THEN after successful login, redirect to /feed (not original destination in v1.0)

ELSE IF user visits public route (/welcome, /login, /register, /reset-password)
  IF isAuthenticated === true
    THEN redirect to /feed
  ELSE
    THEN render requested page
```

### 5.2 Onboarding Gate

```
IF user has just registered (isNewUser === true)
  THEN redirect to /onboarding after registration

ELSE IF user has previously registered and logs in
  IF onboarding_complete === false
    THEN show /feed with persistent "Complete your profile" banner
  ELSE
    THEN show /feed (personalised)

IF user skips onboarding entirely
  THEN show trending feed + persistent profile completion banner

IF user completes onboarding partially (e.g. Steps 1-4 but skips 5-6)
  THEN seed feed with available data (weaker personalisation)
  THEN show persistent banner inviting completion
```

### 5.3 Feed Personalisation

```
IF user has completed all 6 onboarding steps (including swipe mechanic)
  THEN seed feed from taste_vector (blended brand-prior + swipe signal)
  THEN refine with every interaction (like, dismiss, wishlist, add to bag, checkout, dwell)

ELSE IF user completed Steps 1-5 but skipped swipe (Step 6)
  THEN seed feed from brand-prior vector only (weaker signal)

ELSE IF user skipped onboarding but has ≥5 meaningful interactions
  THEN compute taste_vector from interactions (graduation from cold start)

ELSE (cold start, minimal data)
  THEN show editorial picks (curated ~20 products) + trending (most-saved 7 days)
  THEN show "Your feed is personalising" indicator

IF user has completed ≥20 interactions
  THEN recommendation engine uses full behavioural signals
  THEN tooltip after 8 interactions: "Your feed is learning your taste..."
```

### 5.4 Occasion Filter Logic

```
IF user selects an occasion filter (Everyday / Going Out / Work / Special)
  THEN filter For You results to products tagged for that occasion
  THEN interactions in this session feed the occasion-specific sub-vector
  THEN e.g. likes while "Going Out" active → update going_out_vector

IF user selects "All" (default)
  THEN show full personalised feed using main taste_vector
  THEN interactions feed the main taste_vector
```

### 5.5 Bag & Checkout State

```
IF user taps Add to Bag on product detail
  IF size is selected
    THEN call Shopify tokenless cart API for product's brand
    THEN create or update cart object
    THEN store returned checkoutUrl
    THEN show toast "Added to bag"
    THEN update bag icon count
    THEN log cart_events (type: add) + user_interactions (type: add_to_bag)
    THEN remain on product detail page
  ELSE
    THEN shake Add to Bag button (animate-shake)
    THEN show toast: "Please select a size"
    THEN remain on product detail

IF user opens /bag
  IF bag is empty (no active carts)
    THEN show empty state with Browse Feed CTA
  ELSE
    THEN render items grouped by brand with per-brand checkout buttons

IF user taps "Checkout with [Brand]"
  THEN log checkout_initiated to cart_events (strongest commercial signal)
  THEN log to user_interactions (weight 1.0 for taste vector)
  THEN update cart status to checkout_initiated
  THEN open checkoutUrl in browser (Shopify hosted checkout)
  THEN user completes payment on Shopify (outside STROBE visibility)
```

### 5.6 Guest Cart Handling

```
IF unauthenticated user adds to bag
  THEN cart stored in localStorage (user_id = null)
  THEN bag functions normally

IF guest user taps "Checkout with [Brand]"
  THEN prompt: "Create an account to check out."
  THEN CTA → /register
  THEN on registration, localStorage cart merged to authenticated cart
```

### 5.7 Price & Stock Validation

```
IF user taps Add to Bag
  THEN check availableForSale on Shopify variant
  IF variant out of stock
    THEN toast: "This size is no longer available."
    THEN refresh size selector to show updated availability

IF product data has not synced within 6 hours (expires_at exceeded)
  THEN show disclaimer on product page: "Stock availability may have changed."
```

### 5.8 Wishlist Price Drop

```
FOR EACH item in wishlist
  IF product.currentPrice < priceAtAdd
    THEN display "Price Dropped" badge (success green) with old and new price
  IF product.currentPrice >= priceAtAdd
    THEN display no badge (price increases are not surfaced)
```

### 5.9 Notification Badge

```
IF unreadCount > 0
  THEN show badge on bell icon with count
ELSE
  THEN hide badge
```

### 5.10 Feed Dismissal

```
IF user dismisses a product (long-press ≥500ms or swipe left)
  THEN remove from visible feed with collapse animation
  THEN add to dismissedProducts array
  THEN log user_interactions (type: dismiss, weight −0.3)
  THEN show toast: "Removed from feed" with Undo action (4 second window)

  IF user taps Undo within 4 seconds
    THEN remove from dismissedProducts
    THEN product reappears in feed
    THEN dismiss interaction is removed/nullified
  ELSE
    THEN dismissal permanent for session
    THEN product suppressed in future sessions
```

### 5.11 Login Attempt Throttling

```
IF loginAttempts >= 5
  THEN disable submit button
  THEN show message: "Too many failed attempts. Please try again later."
  THEN form remains locked (no timer in v1.0)
ELSE
  IF credentials valid
    THEN authenticate and navigate to /feed
  ELSE
    THEN increment attempt counter
    THEN show error: "Invalid email or password."
```

### 5.12 Password Strength

```
FOR each PASSWORD_RULE (length ≥8, uppercase, number)
  IF rule.test(password) === true
    THEN mark rule as passed (green checkmark)
  ELSE
    THEN mark rule as failed (muted)

IF all 3 rules passed → strength bar = green, submit enabled
ELSE IF 2 rules passed → strength bar = amber
ELSE IF 1 rule passed → strength bar = red
ELSE → submit button disabled
```

---

## 6. Error Handling

### 6.1 404 — Page Not Found
- **Display:** Centred message: `"Page not found"`, subtext, Back to Feed button
- **User actions:** Navigate to `/feed`
- **Implementation:** `app/not-found.tsx`

### 6.2 500 — Server Error
- **Display:** `"Something went wrong"` with retry button
- **User actions:** Retry current action, navigate away
- **Implementation:** `app/error.tsx` global error boundary

### 6.3 Network Offline
- **Display:** Persistent banner: `"You're offline. Check your connection."` — persists until reconnected
- **Feed:** Skeleton persists; retry button after 5 seconds
- **Bag/Checkout:** Block all actions; show: `"Connection lost. Please reconnect to continue."`
- **Search:** Results do not update; last results remain visible
- **Recovery:** Automatic retry on reconnection; no data loss for in-progress forms

### 6.4 Permission Denied / Unauthenticated
- **Display:** Redirect to `/login`
- **Recovery:** Session refresh attempt before redirect
- **Note:** Original destination URL not preserved in v1.0

### 6.5 Form Validation Failures

| Form | Field | Validation Rule | Error Behaviour |
|------|-------|----------------|-----------------|
| Register | First name | Non-empty | Submit disabled |
| Register | Last name | Non-empty | Submit disabled |
| Register | Email | Contains `@` | Submit disabled |
| Register | Password | All 3 rules pass | Strength bar + rule checklist |
| Login | Email | Contains `@` | Submit disabled |
| Login | Password | ≥6 characters | Submit disabled |
| Reset Password | Email | Contains `@` | Submit disabled |
| Product Detail | Size | Must select before Add to Bag | Shake animation + toast |

### 6.6 Product / Data Errors

| Scenario | Display | Recovery |
|----------|---------|----------|
| Product not found (`/product/[id]`) | `"Product not found"` centred | Back navigation |
| Invalid product ID in URL | Same as above | Back navigation |
| Bag empty at `/bag` | `"Your bag is empty"` + Browse Feed CTA | Navigate to `/feed` |
| Shopify cart API error | Toast: `"Couldn't update bag. Please try again."` | Retry |
| checkoutUrl expired | System auto-refreshes via cart API. Toast: `"Refreshing checkout link..."` | Automatic |

---

## 7. Responsive Behaviour

STROBE v1.0 is a **mobile-first responsive web app** (Next.js on Vercel). Native iOS and Android apps are a future consideration.

### 7.1 Mobile (Primary — ≤640px)
- **Layout:** Single column for forms/text; 2-column grid for product cards
- **Navigation:** Fixed bottom nav bar (64px) with safe area inset for iOS notch/home indicator
- **Interaction model:** Touch targets ≥44×44px (WCAG 2.1 AA)
- **Gestures:** Tap (primary), scroll (vertical), swipe left (dismiss on feed), swipe right/left (onboarding Step 6)
- **Headers:** Sticky top bar (blur backdrop) on all authenticated screens
- **Bag:** Full-screen page; sticky bottom area for any global actions
- **Product images:** 3:4 aspect ratio; full-width carousel
- **Max content width:** `max-w-lg` (512px) centred with `mx-auto`

### 7.2 Tablet (641px–1024px)
- **Layout:** Same as mobile in v1.0; `max-w-lg` constraint maintains mobile feel
- **Navigation:** Bottom nav persists
- **Product grid:** 2 columns (3-column consideration for v2)

### 7.3 Desktop (>1024px)
- **Current state:** Same as mobile, constrained to `max-w-lg` centred
- **Planned:** Wider layout with sidebar nav, 3–4 column product grid, side-by-side product detail. Bottom nav replaced with left sidebar.

### 7.4 Touch vs Pointer

| Interaction | Mobile (touch) | Desktop (pointer) |
|-------------|---------------|-------------------|
| Product card hover | No hover state | `group-hover` border glow |
| Button hover | Active scale (`active:scale-[0.97]`) | `hover:` styles |
| Wishlist/like icons | Tap | Click |
| Image carousel | Dot tap | Dot click |
| Feed dismiss | Long-press / swipe left | Long-press / right-click context menu (future) |
| Onboarding swipe | Swipe gesture | Click + drag or button alternatives |

---

## 8. Animations & Transitions

All animation tokens defined in `globals.css` using `--ease-snap` and `--ease-spring` CSS custom properties.

### 8.1 Page Transitions

| Transition | Duration | Easing |
|------------|----------|--------|
| Page load (any route) | 350ms | `--ease-snap` (cubic-bezier 0.25,0,0,1) |
| Onboarding step transition | 350ms | `--ease-snap` (slide left/right) |

### 8.2 Component Animations

| Component | Animation | Duration | Trigger |
|-----------|-----------|----------|---------|
| STROBE logo | `animate-glow` (glow-pulse) | 2s loop | On mount (login/register/welcome) |
| Feed product grid | `stagger-children` | 0–350ms staggered | On feed load |
| Skeleton cards | `animate-shimmer` | 1.5s loop | While loading |
| Filter panel | `animate-in` (fade-in-up) | 350ms | On filter open |
| Details accordion | `animate-in` (fade-in-up) | 350ms | On expand |
| Add to Bag shake | `animate-shake` | 400ms | On Add to Bag without size |
| Onboarding swipe right | Slide right + green glow | 350ms | On right swipe (like) |
| Onboarding swipe left | Slide left + subtle fade | 350ms | On left swipe (pass) |
| Feed dismiss | Card collapse upward | 300ms | On dismiss action |
| Bag item remove | Slide out + collapse | 300ms | On remove |
| Confirmation scale | `animate-scale-in` | 200ms | Modal/overlay entry |
| Loading spinner | `animate-spin-slow` | 1s loop | During async actions |
| Strobe flash | `animate-strobe` | 4s loop | Brand moments (welcome screen) |

### 8.3 Easing Reference

| Token | Value | Use Case |
|-------|-------|----------|
| `--ease-snap` | `cubic-bezier(0.25, 0, 0, 1)` | Page transitions, fade-ins, panels |
| `--ease-spring` | `cubic-bezier(0.34, 1.56, 0.64, 1)` | Toggles, shakes, scale-in |
| `--duration-fast` | `120ms` | Button hover, micro-interactions |
| `--duration-base` | `200ms` | Standard transitions |
| `--duration-slow` | `350ms` | Page-level animations, panel opens |

### 8.4 Loading States

| Context | Loading UI |
|---------|-----------|
| Feed initial load | 6 `SkeletonCard` components with shimmer |
| Button submit (login, register) | Spinner replaces button content |
| Add to Bag | Brief spinner on button (Shopify API call) |
| Onboarding feed generation | "We're building your feed..." text with spinner (up to 3s) |

### 8.5 Success Animations

| Context | Animation |
|---------|-----------|
| Add to Bag | Bag icon bounces + count increments |
| Wishlist add | Bookmark icon transitions to filled electric blue |
| Like | Heart icon transitions to filled danger red |
| Onboarding complete | Feed loads with stagger animation |

### 8.6 Toast Notifications

Using **Sonner** (`position: "top-center"`, `duration: 3000ms`).

| Toast Type | Example Trigger | Duration |
|------------|----------------|----------|
| Default (info) | Wishlist add/remove, share, Add to Bag confirmation | 3000ms |
| With Undo action | Feed product dismiss, bag item remove | 4000ms |
| Error | Size not selected, API failure, network error | 3000ms |

### 8.7 Reduced Motion

All animations respect `@media (prefers-reduced-motion: reduce)`:
- All `.animate-*` classes disabled
- All `stagger-children` delays reset to `0ms`
- Transitions remain (opacity/color) but transforms removed
- Onboarding swipe mechanic uses button tap (Like/Pass) instead of swipe gesture

---

## Appendix: Route → Component Map

| Route | File | Layout |
|-------|------|--------|
| `/` | `app/page.tsx` | Root (redirects to `/welcome`) |
| `/welcome` | `app/welcome/page.tsx` | Root layout (no bottom nav) |
| `/login` | `app/login/page.tsx` | Root layout (no bottom nav) |
| `/register` | `app/register/page.tsx` | Root layout (no bottom nav) |
| `/reset-password` | `app/reset-password/page.tsx` | Root layout (no bottom nav) |
| `/onboarding` | `app/onboarding/page.tsx` | Root layout (no bottom nav) |
| `/feed` | `app/feed/page.tsx` | App layout (with `BottomNav`) |
| `/product/[id]` | `app/product/[id]/page.tsx` | App layout (no bottom nav) |
| `/bag` | `app/bag/page.tsx` | Root layout (no bottom nav) |
| `/search` | `app/search/page.tsx` | App layout (with `BottomNav`) |
| `/wishlist` | `app/wishlist/page.tsx` | App layout (with `BottomNav`) |
| `/orders` | `app/orders/page.tsx` | App layout (with `BottomNav`) |
| `/profile` | `app/profile/page.tsx` | App layout (with `BottomNav`) |
| `/notifications` | `app/notifications/page.tsx` | App layout (no bottom nav) |

---

## Appendix: Planned Future Screens (P2)

### `/wardrobe` (replaces `/orders` in bottom nav)
- **Access:** Authenticated
- **Purpose:** Digital wardrobe inventory, outfit pairing, spending analytics
- **Key UI:** 3-column item grid, "Add Item" FAB, category tabs, wear count badges, Outfits sub-section, Analytics sub-section
- **States:** Empty, populated, analytics unlocked (≥10 items with ≥5 wear logs)

### Navigation Map Update (P2)

```
BOTTOM NAV (updated for V2)
  Feed → /feed
  Search → /search
  Wardrobe → /wardrobe (replaces Orders)
  Wishlist → /wishlist
  Profile → /profile

/wardrobe sub-navigation:
  ├── Items (default) — 3-column grid
  ├── Outfits — saved outfit pairings
  ├── Analytics — insights dashboard (gated by data threshold)
  └── Orders — checkout history (moved from /orders)
```

---

## Appendix: Interaction Signals Reference

All signals the front-end captures and logs. The cofounder's recommendation engine consumes these.

| Signal | Type | Logged To | Weight | Capture Method |
|--------|------|-----------|--------|----------------|
| Like | Explicit | user_interactions | 0.8 | Heart icon tap |
| Dismiss | Explicit | user_interactions | −0.3 | Long-press / swipe left |
| Wishlist add | Explicit | user_interactions + user_saved_items | 1.0 | Bookmark icon tap |
| Wishlist remove | Explicit | user_interactions + user_saved_items | — | Bookmark icon toggle |
| Add to Bag | Explicit | cart_events + user_interactions | 0.9 | Add to Bag button |
| Checkout initiated | Explicit | cart_events + user_interactions | 1.0 | Checkout with [Brand] tap |
| Product detail view | Passive | user_interactions (type: tap) | 0.6 | Navigate to `/product/[id]` |
| Dwell 5+ seconds | Passive | user_interactions (type: dwell) | 0.4 | IntersectionObserver |
| Dwell 2–5 seconds | Passive | user_interactions (type: dwell) | 0.15 | IntersectionObserver |
| Dwell under 2 seconds | Passive | Not logged | 0.0 | — |
| Search query | Metadata | analytics event | — | Search submission |

*Weights are indicative — cofounder's engine will tune based on observed behaviour.*

---

*STROBE — APP_FLOW.md · v1.1 · April 2026 · Confidential*
