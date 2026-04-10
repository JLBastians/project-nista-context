# Navigation & Global Behaviours — Functional Spec

> Part of STROBE · April 2026

---

## 1. Screen Inventory

The authoritative list of all screens in the STROBE v0 application.

### Public Routes (no authentication required)

| Route | Purpose | Bottom Nav |
|-------|---------|-----------|
| `/` | Root redirect (→ `/welcome`) | No |
| `/welcome` | App entry point — logo, value prop, Create Account and Sign In CTAs | No |
| `/register` | Account creation — name, email, password, OAuth options | No |
| `/login` | User authentication — email, password, OAuth options, forgot password link | No |
| `/reset-password` | Password reset request — email input, reset link delivery | No |

### Authenticated Routes (login required)

| Route | Purpose | Bottom Nav |
|-------|---------|-----------|
| `/onboarding` | 7-step style profile seeding (welcome → age → gender expression → sizing → city → activities → brands). Swipe mechanic deferred to P1. | No |
| `/feed` | Personalised discovery feed with occasion filtering (default landing post-login) | Yes |
| `/search` | Cross-catalogue semantic search with filters, sort, trending/recent | Yes |
| `/product/[id]` | Product detail — carousel, size selector, Add to Bag, Wishlist, related items | No |
| `/bag` | Multi-brand cart grouped by retailer; per-brand Shopify checkout handoff | No |
| `/wishlist` | Saved items with price tracking and drop badges | Yes |
| `/orders` | Checkout history and order tracking | Yes |
| `/profile` | Account settings, style summary, preferences, logout | Yes |
| `/notifications` | In-app notification centre (price drops, restocks, new arrivals); accessed via bell icon | No |

---

## 2. Navigation Architecture

Complete route hierarchy showing entry points and navigation flows.

```
strobelab.co (Landing Page — public, JavaScript, main branch)
│
└── /welcome  [PUBLIC]
    ├── /register  [PUBLIC]
    │   └── /onboarding  [AUTHENTICATED — new users only]
    │       └── /feed
    └── /login  [PUBLIC]
        └── /reset-password  [PUBLIC]

AUTHENTICATED APP (requires login)
├── /feed  [AUTHENTICATED — default landing]
│   └── /product/[id]  [AUTHENTICATED]
│       ├── Add to Bag → stays on page (bag icon updates)
│       └── Bag icon → /bag
│
├── /search  [AUTHENTICATED]
│   └── /product/[id]  [AUTHENTICATED]
│
├── /bag  [AUTHENTICATED]
│   └── Checkout with [Brand] → Shopify checkoutUrl (external)
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

BOTTOM NAV (5 tabs, visible on authenticated routes except /onboarding, /product/[id], /bag, /notifications)
├── Feed → /feed
├── Search → /search
├── Wishlist → /wishlist
├── Orders → /orders
└── Profile → /profile
```

---

## 3. Entry Points

### 3.1 Direct URL Access

| URL | Destination | Auth Required | Redirect |
|-----|-------------|---------------|----------|
| `/` | Root (forwarded to `/welcome`) | No | `/welcome` |
| `/welcome` | Welcome screen | No | `/feed` (if authenticated) |
| `/register` | Registration form | No | `/feed` (if authenticated) |
| `/login` | Login form | No | `/feed` (if authenticated) |
| `/reset-password` | Password reset | No | — |
| `/onboarding` | Onboarding flow | Yes | `/login` (if unauthenticated) |
| `/feed` | Personalised feed | Yes | `/login` (if unauthenticated); `/onboarding` (if `onboardingComplete === false`) |
| `/search` | Search | Yes | `/login` (unauthenticated); `/onboarding` (onboarding incomplete) |
| `/product/[id]` | Product detail | Yes | `/login` (unauthenticated); `/onboarding` (onboarding incomplete) |
| `/bag` | Cart | Yes | `/login` (unauthenticated); `/onboarding` (onboarding incomplete) |
| `/wishlist` | Wishlist | Yes | `/login` (unauthenticated); `/onboarding` (onboarding incomplete) |
| `/orders` | Orders | Yes | `/login` (unauthenticated); `/onboarding` (onboarding incomplete) |
| `/profile` | Profile | Yes | `/login` (unauthenticated); `/onboarding` (onboarding incomplete) |
| `/notifications` | Notifications | Yes | `/login` (unauthenticated); `/onboarding` (onboarding incomplete) |

### 3.2 Deep Links

| Trigger | Pattern | Destination |
|---------|---------|-------------|
| Price drop notification | `/product/[id]?ref=price_drop` | Product detail page |
| Restock alert | `/product/[id]?ref=restock` | Product detail page |
| Wishlist reminder | `/wishlist?ref=reminder` | Wishlist page |
| Onboarding incomplete | `/onboarding?ref=reminder` | Onboarding flow |
| Marketing email | `/welcome?ref=email_campaign` | Welcome screen |
| Notification tap | `/product/[id]?ref=notification` | Product detail page |

### 3.3 OAuth / Social Login

- **Apple Sign-In:** `/login` or `/register` → Apple OAuth flow → callback → `/feed` (existing user) or `/onboarding` (new user)
- **Google Sign-In:** `/login` or `/register` → Google OAuth flow → callback → `/feed` (existing user) or `/onboarding` (new user)
- **Fallback on OAuth failure:** User returned to form with message "Sign-in failed. Please try again or use email."

### 3.4 Search Engine Indexing

- Landing page (`strobelab.co`) indexed publicly
- All authenticated app routes (`/feed`, `/search`, etc.) return `noindex` meta tags
- Only public auth pages (`/welcome`, `/login`, `/register`) are crawlable

### 3.5 Marketing Campaign UTM Parameters

| Campaign Type | UTM Parameters | Landing Page |
|---------------|----------------|--------------|
| Social advertising (new users) | `?utm_source=instagram&utm_medium=paid` | `strobelab.co` |
| Email newsletter (existing users) | `?utm_source=email&utm_medium=newsletter` | `/welcome` or deep link |
| Influencer / referral | `?ref=[handle]` | `strobelab.co` |

---

## 4. Responsive Behaviour

STROBE is a **mobile-first responsive web app** (Next.js on Vercel). Native iOS/Android apps are future consideration.

### Breakpoints & Layout Progression

| Device | Viewport Width | Layout |
|--------|----------------|--------|
| Mobile (primary) | ≤640px | Single-column forms, 2-column product grid |
| Tablet | 641px–1024px | Same as mobile (future: 3-column grid) |
| Desktop (planned v2) | >1024px | Sidebar nav, 3–4 column grid, side-by-side product detail |

### Navigation & UI Changes

| Component | Mobile | Desktop (v2 planned) |
|-----------|--------|---------------------|
| Navigation | Fixed bottom bar (64px with safe area) | Left sidebar nav (replaces bottom nav) |
| Content width | `max-w-lg` (512px) centred | Wider layout, full viewport |
| Sticky header | Top bar (blur, sticky) | Top bar remains |
| Product grid | 2 columns | 3–4 columns |

### Touch Interaction Standards

| Metric | Standard | Notes |
|--------|----------|-------|
| Touch targets | ≥44×44px | WCAG 2.1 AA compliance |
| Safe areas | iOS notch/home indicator inset | Padding respect on mobile |
| Gestures | Tap (navigate to PDP), double-tap (like), vertical scroll, swipe left (dismiss), swipe right (save to wishlist), long-press ≥500ms (quick-view drawer) | Desktop: click + button alternatives; gesture tutorial teaches all 4 gestures on first feed visit |

---

## 5. Animations & Transitions

All animation tokens defined in `globals.css` using `--ease-snap` and `--ease-spring` CSS custom properties.

### Page Transitions

| Context | Duration | Easing |
|---------|----------|--------|
| Page load (any route) | 350ms | `--ease-snap` (cubic-bezier 0.25,0,0,1) |
| Onboarding step transition | 350ms | `--ease-snap` (slide left/right) |
| Feed → Product Detail (and reverse) | 250ms | Spring — shared element morph via View Transitions API (`viewTransitionName` on card image + hero image). Chrome/Edge only; graceful fade fallback on unsupported browsers. Disabled with `prefers-reduced-motion`. |

### Component Animations

| Component | Animation | Duration | Trigger |
|-----------|-----------|----------|---------|
| STROBE logo | `animate-glow` | 2s loop | On auth page mount |
| Feed product grid | `stagger-children` | 0–350ms staggered | On feed load |
| Skeleton cards | `animate-shimmer` | 1.5s loop | While loading |
| Filter panel | `animate-in` fade-in-up | 350ms | On filter open |
| Add to Bag shake | `animate-shake` | 400ms | No size selected error |
| Onboarding swipe right | Slide right + green glow | 350ms | Right swipe (like) |
| Onboarding swipe left | Slide left + fade | 350ms | Left swipe (pass) |
| Feed dismiss | Card collapse upward | 300ms | On dismiss action (swipe-left) |
| Like (heart) | Scale bounce 1→1.35→1 + red glow | 300ms | On like (not un-like) |
| Wishlist (bookmark) | Scale bounce 1→1.35→1 + blue glow | 400ms | On save (not remove) |
| Quick-view drawer | Slide up from bottom | vaul default | Long-press (300ms) or tap + button on card |
| Bag item remove | Slide out + collapse | 300ms | On remove |
| Loading spinner | `animate-spin-slow` | 1s loop | During async operations |

### Loading States

| Context | UI |
|---------|-----|
| Feed initial load | 6 skeleton cards with shimmer animation |
| Button submit (login, register) | Spinner replaces button content |
| Add to Bag | Brief spinner on button |
| Onboarding feed generation | "We're building your feed..." text with spinner (≤3s) |

### Reduced Motion

All animations respect `@media (prefers-reduced-motion: reduce)`:
- `.animate-*` classes disabled
- `stagger-children` delays reset to 0ms
- Transitions remain (opacity/color) but transforms removed
- Onboarding swipe uses button tap (Like/Pass) instead of gesture

---

## 6. Global Behaviours (FR-GS)

### Offline & Connectivity

- **Offline detection:** Toast: "Check your connection" displayed when network unavailable
- **API failure retry:** Automatic retry with exponential backoff; manual retry button after 5s
- **Cart persistence:** Cart state saved to localStorage (guests) or carts table (authenticated); survives refresh and app close

### API Error Handling

| Error | User Message | Recovery |
|-------|--------------|----------|
| Network timeout | "Something went wrong. Please check your connection." | Retry button |
| Invalid credentials | "Email or password is incorrect." | Retry on login |
| Account locked (≥5 failed attempts) | "Too many login attempts. Please wait 15 minutes or reset your password." | Password reset |
| Email already registered | "An account with this email already exists." | Link to login |
| Product out of stock | Size selector shows greyed/unavailable state | Cannot add to bag |
| Feed load failure | "Failed to load feed." with retry button | Retry button after 5s |
| Checkout handoff failure | Toast: "Couldn't open checkout. Please try again." | Retry |

### Loading States & Skeletons

- **Button loading:** Spinner replaces text on submit buttons (login, register, Add to Bag)
- **Feed loading:** 6 skeleton cards with shimmer effect on initial load
- **Onboarding completion:** "We're building your feed..." message with spinner (up to 3 seconds)
- **Async operations:** All async operations show spinner or skeleton

### Accessibility Requirements

| Requirement | Standard | Implementation |
|-------------|----------|-----------------|
| Screen reader | ARIA labels on all interactive elements | Alt text on images, aria-labels on buttons |
| Touch targets | ≥44×44px | Minimum size enforced across UI |
| Color contrast | ≥4.5:1 (AA standard) | Text and icon contrast validated |
| Dynamic type | Support system font scaling | Tailwind `text-sm` / `text-base` / `text-lg` used responsively |
| Focus indicators | Visible focus ring on keyboard nav | Focus ring on all buttons, links, inputs |
| Reduced motion | Respect prefers-reduced-motion | Animations disabled when user preference detected |

---

## 7. Analytics Events

The authoritative event taxonomy. All events logged to analytics backend (provider TBD: PostHog, Mixpanel, Amplitude, or custom).

| Event Name | Trigger | Required Properties | Notes |
|------------|---------|---------------------|-------|
| **Discovery Events** |
| product_viewed | User opens Product Detail Page | product_id, brand_id, price, source (feed/search/wishlist/recommendation), feed_position | Passive signal (weight 0.6) |
| product_liked | User taps heart icon | product_id, brand_id, source, feed_position | Explicit signal (weight 0.8) |
| product_dismissed | User dismisses/hides product | product_id, brand_id, source, feed_position | Explicit negative signal (weight −0.3) |
| dwell_time | Card visible ≥2s in viewport | product_id, dwell_seconds, source, feed_position | Passive signal (0.15 for 2–5s, 0.4 for 5+s) |
| **Wishlist Events** |
| wishlist_added | User taps bookmark icon | product_id, brand_id, price, source | Explicit signal (weight 1.0) |
| wishlist_removed | User removes from wishlist | product_id, brand_id | — |
| **Bag & Checkout Events** |
| bag_added | User taps "Add to Bag" | product_id, brand_id, price, size, source | Explicit signal (weight 0.9) |
| bag_removed | User removes from bag | product_id, brand_id | — |
| checkout_initiated | User taps "Checkout with [Brand]" | product_id, brand_id, cart_id, session_id | Strongest commercial signal (weight 1.0) |
| **Search Events** |
| search_query | User submits search | query_text, result_count, filters_applied | Metadata; no taste vector weight |
| **Onboarding Events** |
| onboarding_started | User enters onboarding flow | — | Funnel metric |
| onboarding_step_completed | User completes onboarding step | step_number, step_name, time_on_step_seconds | Funnel metric |
| onboarding_abandoned | User exits onboarding | step_abandoned | Funnel metric |
| onboarding_completed | User finishes all steps or explicitly skips | time_to_complete_seconds, steps_completed, seed_method | Funnel metric |
| **Notification Events** |
| notification_opened | User taps notification | notification_type (price_drop/restock/new_arrival), product_id | Re-engagement metric |
| **Session Events** |
| session_started | User logs in or resumes session | — | Session boundary |
| session_ended | User logs out or session expires | duration_seconds | Session boundary |

All weights are indicative and subject to tuning by the recommendation engine.

---

*STROBE Navigation & Global Behaviours Spec · v1.0 · April 2026 · Confidential*
