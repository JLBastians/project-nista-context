---
name: frontend-engineer
description: |
  STROBE frontend engineering skill for building the Next.js 16 web app. Use this skill whenever building, modifying, or reviewing any frontend component, page, hook, or style for STROBE. This includes: creating new pages or components, implementing UI from specs, connecting frontend to API endpoints, building forms (onboarding, auth, search), implementing interaction tracking (likes, dismisses, dwell time), working with the feed or product cards, styling with Tailwind, debugging frontend issues, or reviewing frontend code. Trigger whenever the user mentions UI, component, page, screen, layout, responsive, Tailwind, Next.js, React, TypeScript frontend, mobile-first, or references any file under Frontend/src/. Also trigger for tasks involving the Shopify cart API integration on the client side, occasion filters, infinite scroll, skeleton loaders, or animation work.
---

# STROBE Frontend Engineer

You are building the frontend for STROBE, a fashion discovery app. This document contains the conventions, architecture, and patterns you need to write consistent, high-quality frontend code.

## Project Context

STROBE is a Next.js 16 web app (App Router, React 19, TypeScript) deployed on Vercel. It serves a personalised fashion feed, cross-brand search, multi-brand bag, and checkout handoff to Shopify. The app is mobile-first (375px viewport baseline) and currently on the `dev` branch — the `main` branch hosts only the landing page at strobelab.co.

Before writing any code, read the relevant spec for the feature you're working on:

| Feature | Spec location |
|---------|--------------|
| Onboarding flow | `Context/specs/onboarding.md` |
| Auth (register, login, reset) | `Context/specs/auth.md` |
| For You feed | `Context/specs/feed.md` |
| Search | `Context/specs/search.md` |
| Product detail page | `Context/specs/product-detail.md` |
| Bag & checkout | `Context/specs/bag-checkout.md` |
| Wishlist | `Context/specs/wishlist.md` |
| Orders | `Context/specs/orders.md` |
| Profile | `Context/specs/profile.md` |
| Navigation & routing | `Context/specs/navigation.md` |
| Full user journey | `Context/APP_FLOW.md` |

These specs are the source of truth for functional requirements, error states, edge cases, and acceptance criteria. If something in the code contradicts a spec, the spec wins unless the user says otherwise.

## Tech Stack & Libraries

| Layer | Tool | Notes |
|-------|------|-------|
| Framework | Next.js 16 (App Router) | React 19, Server Components by default |
| Language | TypeScript (strict mode) | No `any` types — use `unknown` + type guards if needed |
| Styling | Tailwind CSS v4 | Mobile-first; no custom CSS unless absolutely necessary |
| UI primitives | shadcn/ui (Radix UI) | Button, Input, Select, Dialog, etc. — already installed |
| Forms | react-hook-form + Zod | All form validation through Zod schemas |
| State | React Context + useReducer | Global state in `src/lib/store.tsx` |
| Icons | Lucide React | Consistent icon set across the app |
| Toasts | Sonner | All user feedback via toast notifications |
| Font | Clash Grotesk (Fontshare CDN) | Brand typography |
| Analytics | Vercel Analytics | Extended analytics TBD |

## Project Structure

```
Frontend/src/
├── app/
│   ├── (app)/          # Authenticated app routes (feed, search, wishlist, etc.)
│   ├── (auth)/         # Public auth routes (welcome, login, register, reset)
│   ├── (onboarding)/   # Onboarding flow (post-registration)
│   ├── layout.tsx      # Root layout
│   └── page.tsx        # Landing page
├── components/
│   ├── ui/             # shadcn/ui primitives (do not edit directly)
│   ├── product-card.tsx
│   ├── bottom-nav.tsx
│   ├── auth-guard.tsx
│   ├── guest-guard.tsx
│   └── ...
├── hooks/
│   ├── use-mobile.ts
│   └── use-toast.ts
├── lib/
│   ├── store.tsx       # Global state (React Context + useReducer)
│   ├── types.ts        # Shared TypeScript types
│   ├── utils.ts        # Utility functions
│   ├── feed-utils.ts   # Feed-specific helpers
│   └── mock-data.ts    # Mock data (will be replaced by API calls)
└── styles/
```

### Routing Convention

The app uses Next.js route groups to separate concerns:

- `(auth)/` — Public routes: `/welcome`, `/login`, `/register`, `/reset-password`. No bottom nav.
- `(onboarding)/` — Authenticated, full-screen: `/onboarding`. No bottom nav.
- `(app)/` — Authenticated routes: `/feed`, `/search`, `/wishlist`, `/orders`, `/profile`, `/product/[id]`, `/bag`, `/checkout`, `/notifications`. Bottom nav visible on tab routes.

## Coding Standards

### TypeScript

- Strict mode is on. Never use `any`.
- Define all props with explicit interfaces — not inline types.
- Export types from `src/lib/types.ts` when shared across files.
- Use discriminated unions for component state (loading / error / success), not boolean flags.

```typescript
// Good: discriminated union
type FeedState =
  | { status: 'loading' }
  | { status: 'error'; message: string }
  | { status: 'success'; products: Product[] }

// Bad: boolean soup
type FeedState = {
  isLoading: boolean
  isError: boolean
  errorMessage?: string
  products?: Product[]
}
```

### Component Patterns

- Prefer Server Components unless the component needs interactivity (state, effects, event handlers).
- Client Components get the `'use client'` directive at the top.
- Keep components focused — one responsibility per file. If a component exceeds ~150 lines, it probably needs splitting.
- Every async operation needs a loading state (skeleton, spinner, or shimmer). Users should never see a blank screen while data loads.
- All interactive elements need minimum 44×44px touch targets (WCAG).

### Styling with Tailwind

- Mobile-first: start with the 375px layout, then add `sm:`, `md:`, `lg:` breakpoints.
- Use Tailwind utilities directly — avoid `@apply` and custom CSS.
- Consistent spacing: use the Tailwind scale (`p-4`, `gap-3`, etc.), don't use arbitrary values unless matching a specific design pixel.
- The product grid is 2-column on mobile (`grid-cols-2`), scaling up on larger screens.
- Bottom nav is 64px height with safe area inset — account for this in page padding.

### State Management

All global state lives in `src/lib/store.tsx` using React Context + useReducer. This includes auth state, user preferences, bag contents, and active occasion filter.

Pattern for adding new state:
1. Add the new state shape to the `AppState` type
2. Add action types to the `AppAction` discriminated union
3. Handle the new actions in the reducer
4. Expose via the context provider

Don't create separate context providers unless state is truly independent (e.g., a theme context). Keep it in one store for simplicity at this stage.

### Forms

All forms use react-hook-form with Zod validation schemas:

```typescript
const schema = z.object({
  email: z.string().email('Invalid email address'),
  password: z.string().min(8, 'Password must be at least 8 characters'),
})

type FormData = z.infer<typeof schema>
```

Field-level validation rules live in the specs (e.g., `Context/specs/auth.md` Section 2.2). Match those exactly.

### Interaction Signal Tracking

STROBE's recommendation engine depends on frontend interaction signals. When building any component that involves user interaction with products, log signals according to the weight table in `Context/ARCHITECTURE.md` (Interaction Signals section).

Key signals to capture:
- **Like** (heart icon tap): weight 0.8, log to `user_interactions`
- **Dismiss** (long-press or swipe): weight −0.3, show undo snackbar for 4 seconds
- **Save/wishlist** (bookmark tap): weight 1.0, log to both `user_interactions` and `user_saved_items`
- **Product detail tap**: weight 0.6, log source ("feed" or "search")
- **Dwell time**: IntersectionObserver, thresholds at 2s (0.15) and 5s (0.4)
- **Add to bag**: weight 0.9, log to `cart_events`
- **Checkout initiated**: weight 1.0, log to both `cart_events` and `user_interactions`

Always include `product_id`, `timestamp`, and `occasion` context (if the user has an occasion filter active).

### Error Handling Pattern

Every API call and user action should handle failure gracefully:

1. **Optimistic UI** for quick actions (like, save, dismiss) — update UI immediately, revert on failure.
2. **Toast notifications** via Sonner for action feedback ("Added to bag", "Action failed. Please try again.").
3. **Undo snackbars** for destructive actions (dismiss) — 4-second window.
4. **Retry buttons** for network failures on page loads.
5. **Skeleton loaders** during initial data fetches — never show empty white screens.

See each feature spec's "Error States" section for the exact error messages and recovery flows.

### Accessibility

- All interactive elements: `aria-label` when the visual label isn't sufficient.
- Product cards announced as: "Product: [Title] by [Brand], $[Price]. Buttons: Like, Save to Wishlist, View Details."
- Keyboard navigation: Tab through cards, Enter to select, arrow keys to scroll.
- Minimum contrast ratio: WCAG AA (4.5:1 for normal text).
- Screen reader support for all toasts and snackbars.

## Before You Start Coding

1. Read the relevant spec in `Context/specs/`.
2. Check `Context/APP_FLOW.md` to understand where the feature fits in the user journey.
3. Check `src/lib/types.ts` for existing type definitions.
4. Check `src/lib/store.tsx` for existing state that your feature might need.
5. If the feature involves API calls, check `Backend/api/README.md` for endpoint signatures.

## Key Decisions Still Open

- **Auth provider** (NextAuth.js vs Clerk): Not yet decided. Build auth UI to be provider-agnostic.
- **Analytics platform**: TBD. Instrument events now with a simple logging interface that can be swapped later.
- **Native apps**: Web-first for v0. Don't build anything that would block a future React Native migration, but don't over-engineer for it either.
