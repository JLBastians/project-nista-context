---
name: frontend-engineer
description: |
  STROBE frontend engineering skill for building the Next.js 16 web app. Use this skill whenever working on: pages, components, hooks, layouts, styling, forms, navigation, animations, or any TypeScript/React code under Frontend/. Also trigger when the user mentions UI, component, page, screen, layout, responsive, Tailwind, Next.js, React, TypeScript frontend, mobile-first, or references Frontend/ files. Trigger for interaction tracking (likes, dismisses, dwell time), Shopify cart API client-side integration, occasion filters, infinite scroll, skeleton loaders, or any user-facing feature work.
---

# STROBE Frontend Engineer

You are building the frontend for STROBE, a fashion discovery app. This document contains the conventions, architecture, and patterns you need to write consistent, production-quality frontend code.

## Project Context

STROBE's frontend is a Next.js 16 web app (App Router, React 19, TypeScript) deployed on Vercel. It serves a personalised fashion feed, cross-brand search, multi-brand bag, and checkout handoff to Shopify. The app is mobile-first (375px viewport baseline) and currently on the `dev` branch — the `main` branch hosts only the landing page at strobelab.co.

**Two-phase approach — know which phase you're building for:**
- **Current (PoC):** Mock data in `src/lib/mock-data.ts`, not yet connected to API. Used for UI development and layout testing.
- **Target (v0):** Connected to FastAPI backend (Railway / Neon), real product data, live interaction tracking.

This skill covers all screens, components, interactions, animations, and client-side API integration.

## Documentation Map

Read these before writing code as necessary:

| If you need to... | Read |
|------------|------|
| Understand the tech stack | `Frontend/README.md` Section 2 |
| Understand the frontend directory structure | `Frontend/README.md` Section 3 |
| Understand routing and route groups | `Frontend/README.md` Section 4 |
| See the functional requirements for a feature | `Context/specs/` (relevant feature spec) |
| Trace the full user journey | `Context/APP_FLOW.md` |
| See screen inventory and navigation rules | `Context/specs/navigation.md` |
| See how the backend serves data | `Backend/api/README.md` Section 4 |
| Understand the overall system architecture | `Context/ARCHITECTURE.md` |
| Check shared types | `Frontend/src/lib/types.ts` |
| Check global state shape | `Frontend/src/lib/store.tsx` |

## Coding Standards

### TypeScript

- Strict mode is on. Never use `any` — use `unknown` + type guards if needed.
- Define all props with explicit interfaces — not inline types.
- Export shared types from `src/lib/types.ts`.
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
- Keep components focused — one responsibility per file. If a component exceeds ~150 lines, split it.
- Every async operation needs a loading state (skeleton, spinner, or shimmer).
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
1. Add the new state shape to the `AppState` type.
2. Add action types to the `AppAction` discriminated union.
3. Handle the new actions in the reducer.
4. Expose via the context provider.

### Forms

All forms use react-hook-form with Zod validation schemas. Field-level validation rules live in the specs (e.g., `Context/specs/auth.md` Section 2.2) — match those exactly.

### Interaction Signal Tracking

STROBE's recommendation engine depends on frontend interaction signals. When building any component that involves user interaction with products, log signals according to the weight table in `Context/ARCHITECTURE.md` (Interaction Signals section). Always include `product_id`, `timestamp`, and `occasion` context (if the user has an occasion filter active).

### Error Handling

- **Optimistic UI** for quick actions (like, save, dismiss) — update UI immediately, revert on failure.
- **Toast notifications** via Sonner for action feedback.
- **Undo snackbars** for destructive actions (dismiss) — 4-second window.
- **Retry buttons** for network failures on page loads.
- **Skeleton loaders** during initial data fetches.

See each feature spec's "Error States" section for exact error messages and recovery flows.

### Accessibility

- All interactive elements: `aria-label` when the visual label isn't sufficient.
- Product cards announced as: "Product: [Title] by [Brand], $[Price]. Buttons: Like, Save to Wishlist, View Details."
- Keyboard navigation: Tab through cards, Enter to select.
- Minimum contrast ratio: WCAG AA (4.5:1 for normal text).

## Before You Start Coding

1. Read the relevant documentation laid out in the Documentation Map.
2. If the feature has a backend endpoint, read `Backend/api/README.md` to understand the request/response contract.
3. Check `src/lib/types.ts` for existing type definitions you can reuse.
4. Check `src/lib/store.tsx` for existing state your feature might need.
5. Run the dev server locally to test: `npm run dev`
