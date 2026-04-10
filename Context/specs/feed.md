# Feed — Functional Spec

> Part of STROBE · April 2026

---

## 1. Overview

The For You feed is the primary product discovery surface and default landing page for authenticated users. Powered by occasion-specific taste vectors, it delivers personalised recommendations that improve with every interaction. The feed is the core engagement loop: users discover products, express preferences through likes/dismisses/saves, and the system refines its understanding of their style in real-time.

**Vector type used:** The feed uses **style vectors** (67-dim, zero-shot FashionSigLIP) for cross-category aesthetic matching — meaning a user who likes a minimalist linen shirt will also see minimalist trousers, bags, and outerwear. This is distinct from search, which uses **product vectors** (768-dim Marqo-FashionSigLIP) for intra-category similarity. See `ARCHITECTURE.md` Layer 3 for full vector distinction.

---

## 2. Feed Composition

The feed is populated via a query to Marqo vector search, which ranks products by similarity to the user's taste vector for the selected occasion. The process:

1. **Vector selection:** If an occasion filter is active (Everyday / Going Out / Work / Special), use the occasion-specific style sub-vector. Otherwise use the primary taste_vector (style-based).
2. **Vector query:** Send the selected style vector to Marqo. Returns ranked product IDs by cross-category aesthetic similarity.
3. **Filter exclusions:** Remove products the user has already interacted with (liked, dismissed, viewed in detail).
4. **Detail fetch:** Batch-fetch product data (images, price, brand, tags) from Neon PostgreSQL.
5. **Render:** Display in a 2-column grid with skeleton cards during load.

For cold-start users (no taste vector or fewer than 5 meaningful interactions), serve **editorial picks** (~20 curated products) followed by **trending** (most-saved products in the last 7 days). After 5+ meaningful interactions, graduate to personalised feed using the seeded taste vector.

See **ARCHITECTURE.md Section 8 (Personalisation)** for taste vector calculation, session recalculation (0.7 × existing + 0.3 × session_signal), and brand-prior blending.

---

## 3. Functional Requirements

### FR-FD-001 — Feed Display

The feed displays products in a 2-column responsive grid. Each product card shows:
- Product image (square, 1:1 aspect ratio)
- Brand name (small, subdued)
- Product title (2–3 lines, truncated with ellipsis)
- Price (formatted with currency)
- Occasion tags (e.g. "Everyday", "Going Out" — max 2, badge-style)

Skeleton cards render while products load. On successful load, products stagger in with a subtle slide-up animation (50ms delay between rows).

### FR-FD-002 — Sticky Header

The feed header is sticky at the top of the viewport:
- Left: STROBE logo (text or icon)
- Right: Notification bell icon with unread badge (red dot if unread notifications exist)

Header background semi-transparent with a subtle blur effect on scroll.

### FR-FD-009 — Occasion Filter

A lightweight toggle row sits directly below the sticky header, never scrolls off:
- Five toggle buttons: Everyday / Going Out / Work / Special / All
- Default state: All (shows full personalised feed using main taste_vector)
- Selected state: Highlighted with brand color + underline
- Behaviour: Selecting an occasion filters feed results to products tagged for that occasion
- Vector impact: Interactions while a filtered occasion is active update the occasion-specific sub-vector (e.g. liking products while "Going Out" is selected strengthens the going_out_vector)
- Persistence: Occasion selection persists during the session; resets to "All" on app cold start

### FR-FD-003 — Like Interaction

User taps the heart icon on a product card.

- **UI feedback:** Heart scale-bounces 1→1.35→1 (300ms) + red glow pulse (400ms) + haptic vibration. Un-liking produces no animation (heart reverts to outline silently).
- **Data capture:** Logged to `user_interactions` (type: "like", product_id, weight: 0.8, timestamp)
- **Feed impact:** Product remains on feed; heart stays filled for remainder of session
- **Taste vector impact:** Positive signal — product vector weighted +0.8 blended into taste vector at session end
- **Error state:** If action fails, heart reverts to outline and toast appears: "Action failed. Please try again."
- **Reduced motion:** Animation disabled when `prefers-reduced-motion: reduce` is set

### FR-FD-004 — Dismiss Interaction

User swipes left on product card.

> **Note:** Long-press (300ms) no longer triggers dismiss — it opens the Quick-View Drawer (see FR-FD-013). Dismiss is swipe-left only.

- **UI feedback:** Product collapses with fade-out animation (200ms). Undo snackbar appears at bottom: "Not for me — Undo (4s countdown)"
- **Data capture:** Logged to `user_interactions` (type: "dismiss", product_id, weight: −0.3, timestamp)
- **Feed impact:** Product removed from current view. Next product in queue renders below.
- **Taste vector impact:** Negative signal — product vector weighted −0.3 blended into taste vector at session end
- **Undo:** User can tap undo snackbar within 4 seconds to restore product and cancel the dismiss signal
- **Error state:** If undo fails, product remains hidden and toast shows: "Unable to undo — this item stays hidden for now."

### FR-FD-005 — Wishlist Interaction

User taps the bookmark icon on a product card.

- **UI feedback:** Bookmark scale-bounces 1→1.35→1 + blue glow pulse (400ms) + haptic vibration when saving. Removing produces no animation (bookmark reverts to outline silently).
- **Data capture:** Logged to `user_interactions` (type: "save", product_id, weight: 1.0, timestamp) AND to `user_saved_items` (product_id, saved_at)
- **Feed impact:** Product remains on feed; bookmark stays filled for remainder of session
- **Wishlist tab:** Product appears in the user's Wishlist tab
- **Taste vector impact:** Strong positive signal — product vector weighted +1.0 blended into taste vector at session end
- **Toggle:** Tapping bookmark again removes the save (bookmark outline, removed from wishlist)
- **Error state:** If action fails, bookmark reverts and toast shows: "Action failed. Please try again."
- **Reduced motion:** Animation disabled when `prefers-reduced-motion: reduce` is set

### FR-FD-006 — Product Detail Navigation

User taps the product card (any area except heart/bookmark icons).

- **Navigation:** Opens `/product/[id]` page in place (or modal on mobile, TBD by design)
- **Data capture:** Logged to `user_interactions` (type: "tap", product_id, weight: 0.6, timestamp, source: "feed")
- **Return behaviour:** User returns to feed at same scroll position
- **Time tracking:** Dwell time on product detail page logged on exit as separate dwell interaction

### FR-FD-010 — Dwell Time Tracking

IntersectionObserver monitors how long each product card is visible in the viewport (product card occupies at least 50% of viewport height).

- **Threshold 1 (0–2 seconds):** Not logged. Assumes user is scrolling past.
- **Threshold 2 (2–5 seconds):** Logged as weak signal (type: "dwell", product_id, dwell_seconds, weight: 0.15, timestamp)
- **Threshold 3 (5+ seconds):** Logged as meaningful signal (type: "dwell", product_id, dwell_seconds, weight: 0.4, timestamp)
- **Data capture:** Entries written to `user_interactions` table with dwell_seconds field populated
- **Taste vector impact:** Weak signal (0.15) and meaningful signal (0.4) both blended into taste vector at session end
- **Accuracy:** Dwell time pauses if user scrolls product off-screen, resumes if scrolled back in

### FR-FD-007 — Infinite Scroll

Feed loads products in batches of 20.

- **Pre-fetch trigger:** When user scrolls within 5 cards of list end, next batch of 20 is fetched in background
- **Loading state:** Skeleton cards appear at bottom of list while fetching
- **Append:** New products automatically render at end of list with stagger animation
- **Empty state:** If all products dismissed or feed exhausted, show: "No more items. Pull down to refresh for new recommendations."

### FR-FD-008 — Pull-to-Refresh

User pulls down on the feed to manually refresh product recommendations.

- **Gesture:** Pull-down gesture on iOS/Android (or keyboard shortcut on web — TBD)
- **Data capture:** Refresh logged as analytics event (type: "feed_refresh", timestamp)
- **Behaviour:** Re-queries taste vector, clears dismissed products from hiding, fetches fresh batch of 20
- **Visual feedback:** Spinner animation during refresh; "Refreshing..." toast
- **Duration:** Completes within 1–2 seconds typical

### FR-FD-012 — Match Score Pill

Product cards display a match score pill when the product's score is ≥88% and the user has completed onboarding.

- **Display:** `✦ XX% match` pill positioned bottom-left of card image (`absolute bottom-2 left-2`), styled with semi-transparent canvas background + backdrop blur
- **Threshold:** Only renders when `matchScore >= 88`. No pill shown for lower scores.
- **Personalisation gate:** Pill only appears when `isPersonalised === true` (onboarding complete + taste vector seeded). Non-personalised users see no pills anywhere on feed.
- **Data source:** `matchScore` field on product object (PoC: from mock data; v0: returned by Marqo vector search)

---

### FR-FD-013 — Quick-View Drawer

Users can add a product to their bag without navigating away from the feed.

- **Open triggers:** Long-press on card (≥500ms) OR tap the shopping bag button in the card's top-right corner
- **Short tap:** Still navigates to `/product/[id]` — quick-view does not intercept normal tap
- **Swipe-left:** Still dismisses the product — quick-view does not intercept swipe
- **Drawer contents:** Product thumbnail, brand, name, price; size selector buttons; Add to Bag CTA; Wishlist toggle; "View Details" link to PDP
- **Add to Bag (size selected):** Adds to bag state, shows toast "Added to bag" with "View Bag" action → `/bag`, closes drawer, clears selected size
- **Add to Bag (no size):** Shake animation on CTA + error toast "Please select a size"; drawer stays open
- **Dismiss drawer:** Drag down or tap X; no action taken
- **State tracking:** `gestureTutorialComplete` in store tracks whether the first-use interactive gesture tutorial has been completed (persists via localStorage). See FR-FD-015.

---

### FR-FD-015 — Interactive Gesture Tutorial

On a user's first visit to the feed after completing onboarding, a 4-step interactive overlay teaches all feed gestures using a real product card.

- **Trigger:** `gestureTutorialComplete === false` in store AND feed has loaded products. Runs once per account (state persisted in localStorage).
- **Target:** The first product card in the feed grid is the tutorial target — gestures are real actions, not a sandbox demo.
- **Step sequence:** Swipe right → Swipe left → Double-tap → Long-press
- **Per-step behaviour:**
  - Instruction phase: Coach mark pill above the card shows step title + subtitle + progress dots (e.g. "Swipe right to save / Save items to your wishlist")
  - Hint phase: After 3 seconds idle, an animated hand SVG appears demonstrating the gesture motion
  - Confirm phase: On correct gesture, confirmation chip shows ("Saved!", "Dismissed!", "Liked!", "Nice!") for 700ms then advances
  - Step 2 (swipe-left): Card is NOT dismissed — the product stays in the grid for the remainder of the tutorial
- **Touch blocking:** Four transparent `pointer-events: auto` panels surround the spotlight card, preventing accidental interaction with the rest of the feed during the tutorial
- **Skip:** "Skip" button (top-right) dismisses the tutorial at any step and sets `gestureTutorialComplete: true`
- **Scroll lock:** `document.body.style.overflow = "hidden"` while tutorial is active
- **Reduced motion:** All animations disabled; hint appears immediately at instruction phase
- **Completion:** After step 4 confirm, overlay fades out (200ms) and normal feed interaction begins
- **Migration:** Users who saw the old gesture tooltip (`gestureTooltipShown: true` in localStorage) are treated as tutorial-complete — they will not see the tutorial

---

### FR-FD-014 — Price Drop Toast (Feed)

On feed mount, if any wishlisted items have a current price below `price_at_add`, a toast fires once per session.

- **Timing:** Fires 1200ms after feed mounts (allows feed content to load first)
- **Content:** "{N} wishlist item(s) dropped in price" with "View" action → `/wishlist`
- **Duration:** Toast persists for 5000ms
- **De-duplication:** Keyed in sessionStorage to prevent repeat toasts within the same browser session. Resets on new session.
- **Threshold:** Any price reduction qualifies (no minimum percentage for the toast; the wishlist page badge/banner uses the same signal)

---

### FR-FD-011 — Cold Start & Graduation

**New user (no taste vector or ≤4 meaningful interactions):**
- Feed displays editorial picks (curated ~20 products hand-selected by founder)
- If editor picks unavailable, fall back to trending (most-saved products in last 7 days)
- Persistent banner: "Complete your profile to personalise your feed." with CTA → `/onboarding`

**Graduation trigger:** After 5+ meaningful interactions (likes + saves + product detail taps, weighted; dismisses do not count toward graduation), system computes initial taste vector and switches to personalised feed

**Banner dismissal:** User can close the profile banner; it re-appears on feed reload

---

## 4. User Flow (Happy Path)

1. User logs in or navigates to `/feed`
2. Feed page loads; skeleton cards render
3. Product cards render in 2-column grid with stagger animation (50ms per row)
4. User encounters first product card
5. User may:
   - **Like:** Tap heart icon → heart fills → interaction logged
   - **Save:** Tap bookmark icon → bookmark fills → interaction logged + product added to wishlist
   - **Dismiss:** Swipe left on card → product collapses with undo snackbar → interaction logged
   - **Quick-view:** Long-press card ≥300ms or tap `+` → quick-view drawer opens (no navigation, no dismiss)
   - **View detail:** Tap card → navigate to `/product/[id]` → interaction logged + dwell time tracked
   - **Filter by occasion:** Tap occasion toggle (e.g. "Going Out") → feed re-queries with occasion sub-vector → interactions update occasion-specific vector
6. User scrolls down; IntersectionObserver tracks dwell time per card (≥2s logged)
7. User scrolls within 5 cards of list end → next batch of 20 pre-fetches in background
8. User continues interacting; feed personalises over session and across sessions
9. After 5+ meaningful interactions, feed graduates from editorial/trending to personalised

---

## 5. Error States

| Error | Scenario | Display | Recovery |
|-------|----------|---------|----------|
| Feed fails to load | Network timeout on initial page load | Skeleton cards persist; retry button appears after 5s: "Failed to load feed. Tap to retry." | User taps retry button; page re-queries |
| Like action fails | API error on like submission | Heart reverts to outline. Toast: "Action failed. Please try again." | User can retry like |
| Wishlist action fails | API error on save submission | Bookmark reverts to outline. Toast: "Action failed. Please try again." | User can retry save |
| Dismiss action fails | API error on dismiss submission | Product remains visible. Toast: "Couldn't remove this item. Try again." | User can retry dismiss |
| Undo fails | API error during undo window | Product remains hidden. Toast: "Unable to undo — this item stays hidden for now." | No recovery in v0 |
| Infinite scroll fails | Pre-fetch of next batch times out | Skeleton cards at bottom disappear; retry option offered | User scrolls up and down to trigger retry, or pulls to refresh |
| Occasion filter fails | API error when switching occasion | Filter toggle reverts to previous selection. Toast: "Couldn't apply filter. Try again." | User can retry filter toggle |
| Personalisation fails | Engine error on feed generation | Trending feed displayed with banner: "We had trouble personalising your feed. We'll keep learning as you browse!" | Banner dismissible; feed continues with fallback algorithm |

---

## 6. Edge Cases

| Case | Behaviour |
|------|-----------|
| All products dismissed | Feed shows empty state: "No more items. Pull down to refresh for new recommendations." User can pull-to-refresh or navigate elsewhere. |
| New user with no taste vector | Feed displays editorial picks; if unavailable, falls back to trending. Profile banner shown. |
| User selects occasion with 0 matching products | Empty state shown with occasion name: "No [Occasion] items at the moment. Try a different occasion or pull to refresh." |
| User scrolls back to top after scrolling | Previous skeleton cards are cached; no re-fetch. Smooth scroll experience. |
| User switches occasions mid-session | Feed re-queries with new occasion sub-vector. Previous scroll position reset to top. |
| Interaction recorded while offline | Logged to local queue. Synced when connection restored. |
| Dwell time tracked across page visibility changes | Pauses when tab loses focus; resumes when regains focus. |
| Product updated in Neon while displayed on feed | Next feed refresh shows updated product data (price, images, etc.). In-session updates not applied. |
| User receives notification while on feed | Notification bell badge updates in real-time. Tapping bell opens notification centre. |
| Feed renders on very small viewport (≤375px width) | Cards stack in 1 column instead of 2. Card height adjusted to avoid oversizing. |

---

## 7. Metrics & Instrumentation

| Metric | Capture Method | Purpose |
|--------|----------------|---------|
| Feed impression | Feed page rendered | Usage baseline |
| Product card render | Skeleton → product load | Performance tracking |
| Like rate | Like interactions / total impressions | Engagement + taste signal |
| Wishlist rate | Save interactions / total impressions | High-intent signal |
| Dismiss rate | Dismiss interactions / total impressions | Negative taste signal quality |
| Dwell time | IntersectionObserver | Engagement + attention tracking |
| Scroll depth | Max scroll position per session | Session engagement |
| Occasion filter adoption | Filter toggle taps / sessions | Feature usage |
| Feed refresh rate | Pull-to-refresh count | User satisfaction proxy |
| Cold-to-warm conversion | Sessions until graduation (5+ interactions) | Onboarding quality |
| Search-to-feed return | Navigation pattern | Feed stickiness |

---

## 8. Accessibility & Responsive Design

- **Touch targets:** All interactive elements (heart, bookmark, card, occasion toggle) ≥44x44px
- **Keyboard navigation:** Tab through product cards; Enter to select; arrow keys to scroll (web)
- **Screen reader:** Card announced as "Product: [Title] by [Brand], $[Price]. Buttons: Like, Save to Wishlist, View Details."
- **High contrast:** Icon fills use brand color (WCAG AA minimum)
- **Mobile-first:** Designed for 375px viewport. Cards and text scale fluidly to desktop.
- **Dark mode:** Follow system preference (TBD by design)

---

*STROBE Feed Spec · v1 · April 2026 · Confidential*
