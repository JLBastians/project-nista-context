# Orders — Functional Spec

> Part of STROBE · April 2026

---

## 1. Overview

The Orders tab displays the user's checkout history — events where STROBE handed off to a brand's Shopify checkout. STROBE does not own fulfilment or order completion visibility. Primary data source is `checkout_initiated` events in `cart_events` table, supplemented by `confirmed_orders` where brand webhooks have been established. Status values are limited to checkout confirmation and shipping webhooks from brand partners.

---

## 2. Functional Requirements

### FR-OR-001 — Checkout History List

Orders tab displays checkout-initiated events in reverse chronological order (most recent first).

**Each entry displays:**
- Product image (thumbnail, 3:4 aspect ratio)
- Product name
- Brand name
- Size (selected at time of checkout)
- Price (price at time of add, from `cart_events`)
- Date/time of checkout initiation (relative format: "2 days ago")
- Status badge

**Status values:**
- "Checkout Started" (grey) — default; STROBE handed off to Shopify
- "Order Confirmed" (green) — only if brand webhook confirms order
- "Shipped" (orange) — only if brand provides tracking webhook
- "Delivered" (green) — only if brand provides tracking webhook

**UI States:**
- Loading: 6 skeleton entries with shimmer
- Populated: List of ≥1 order entries
- Empty: See Empty State (Section 6)

---

### FR-OR-002 — Limited Visibility Disclaimer

Subtle note at the top of the Orders tab: "Order status is provided by each brand. For detailed tracking, visit the brand's website."

This sets expectations that STROBE's visibility is limited to checkout handoff and available webhook data.

---

### FR-OR-003 — Confirmed Order Detail (Where Available)

If a brand has provided a confirmed order via webhook (`confirmed_orders` table exists), the entry displays:
- Order value (total paid, if available)
- Confirmed timestamp
- "Track on [Brand]" link opening the brand's order tracking page (stored in `confirmed_orders.tracking_url`)

This enriches the checkout-initiated entry without requiring a separate detail view in v0.

---

## 3. Acceptance Criteria

| AC ID | Criterion | Test Type | Pass Condition |
|-------|-----------|-----------|----------------|
| AC-OR-001 | Every checkout_initiated event appears in Orders tab | Functional | All `cart_events` with `type: checkout_initiated` are listed in reverse chronological order. |
| AC-OR-002 | Confirmed orders display enriched status | Functional | Where `confirmed_orders` data exists for an order, entry shows "Order Confirmed" status + order value + "Track on [Brand]" link. |
| AC-OR-003 | Orders persists across session | Functional | Returning user sees historical checkout events, not cleared on logout. |

---

## 4. Data Model

**cart_events table:**

| Column | Type | Notes |
|--------|------|-------|
| id | uuid PK | |
| user_id | uuid FK | |
| product_id | uuid FK | |
| brand_id | uuid FK | |
| cart_id | text | Shopify cart ID |
| type | text | "add" / "remove" / "checkout_initiated" |
| price_at_event | decimal | Price at time of event |
| size | text | Selected size |
| session_id | text | Session identifier |
| created_at | timestamp | Event timestamp |

**confirmed_orders table (webhook data):**

| Column | Type | Notes |
|--------|------|-------|
| id | uuid PK | |
| user_id | uuid FK | |
| product_id | uuid FK | |
| brand_id | uuid FK | |
| order_id | text | Brand's order ID |
| order_value | decimal | Total paid |
| order_status | text | "confirmed" / "shipped" / "delivered" |
| tracking_url | text | Brand's tracking page URL |
| confirmed_at | timestamp | When order was confirmed |
| updated_at | timestamp | Last webhook update |

For full schema, see `Backend/api/README.md`.

---

## 5. Error States

| Error | Display | Recovery |
|-------|---------|----------|
| Orders load fails | Toast: "Unable to load order history. Pull to refresh." | Pull to refresh |
| No orders | Empty state with "Start Browsing" CTA | Navigate to `/feed` |
| Webhook data unavailable | Status remains "Checkout Started"; no enrichment shown | No action required |

---

## 6. Empty State

"No orders yet. Find something you love and check out in seconds." with "Start Browsing" CTA linking to `/feed`.

---

*STROBE — Orders Spec · v1 · April 2026 · Confidential*
