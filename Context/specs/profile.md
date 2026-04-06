# Profile — Functional Spec

> Part of STROBE · April 2026

---

## 1. Overview

The Profile tab provides account management, style preference viewing, notification settings, and data/privacy controls. Users access their style summary derived from taste vectors, edit their onboarding preferences, manage notifications, and request data deletion in compliance with CCPA.

---

## 2. Functional Requirements

### FR-PR-001 — Profile Header

Display user account information:
- Name (first + last from registration)
- Email address
- Avatar placeholder (future: photo upload)
- "Member since" date (account creation date)

---

### FR-PR-002 — Style Profile View

**Style Summary Card:**

Displays a human-readable summary derived from `user_taste_profiles.tag_weights` (dominant tags by confidence score). Example text: "You gravitate toward minimalist, earth tones, oversized silhouettes."

**Breakdown includes:**
- Top 3–5 aesthetic/style tags (e.g., minimalist, maximalist, bohemian)
- Top 2 colour palettes (e.g., earth tones, pastels)
- Top 2 silhouettes (e.g., oversized, fitted)
- **Occasion breakdown:** List of preferred occasions with frequency (e.g., "Everyday (Often), Dinner Out (Sometimes)")

**Default state (no taste data yet):** "Complete your style profile to see your taste summary" with CTA button linking to `/onboarding`.

Updates each session after taste vector recalculation.

---

### FR-PR-003 — Edit Preferences

"Edit Preferences" button navigates to `/onboarding` with all 5 steps pre-populated with current user values:
- Step 1: Birth year from `date_of_birth`
- Step 2: Gender expression from `gender_expression` array
- Step 3: City from `city` + `city_source`
- Step 4: Occasion weights from `occasion_weights` JSONB
- Step 5: Brand selections from `onboarding_brand_selections`
- Step 6: Swipe mechanic (re-run with new 10 cards or skip)

On completion, taste vectors are recalculated and style summary updates.

---

### FR-PR-004 — Notification Settings

Toggles for each notification type (when notifications system is implemented):
- Price drops on wishlisted items
- Restock alerts on saved items
- New arrivals matching style

Currently a placeholder section (P1 post-MVP). Each toggle writes to a `user_notification_preferences` table.

---

### FR-PR-005 — Data & Privacy

Three controls:

**Download My Data:**
Generates a ZIP export of user data (profile, interactions, saved items, order history) in portable formats (JSON/CSV). Email sent with download link (expires 7 days).

**Request Deletion:**
Button initiates account deletion with 30-day grace period (CCPA §1798.105). Confirmation modal: "Your account will be permanently deleted in 30 days. You can cancel this request anytime." On confirmation, account flagged as `deletion_requested = true` with `deletion_scheduled_at` timestamp. After 30 days, account and associated data purged. Until deletion, user can still log in and cancel the request.

**Opt Out of Personalisation:**
Toggle to disable taste vector calculations and algorithm-powered feed. User sees editorial picks and trending products instead. Preference stored in `user_preferences.algorithm_opt_out = true`.

---

### FR-PR-006 — Account Deletion

Button with destructive styling (red). Tapping shows confirmation modal: "This will permanently delete your account and all associated data. This cannot be undone."

On confirmation:
1. Account flagged for deletion (30-day grace period — CCPA compliant)
2. User logged out
3. Redirect to `/welcome` with message: "Account deletion scheduled. You will receive a confirmation email."

---

### FR-PR-007 — Logout

Logout button clears auth token, clears localStorage, logs out user, and redirects to `/welcome`.

---

## 3. Data Model

**users table (existing fields):**

| Column | Type | Notes |
|--------|------|-------|
| date_of_birth | date | For style summary display |
| gender_expression | text[] | Onboarding Step 2 |
| city | text | Onboarding Step 3 |
| city_source | text | "ip" or "manual" |
| sustainability_orientation | text | Inferred from brands |
| price_signal | text | Inferred from brands |

**user_taste_profiles table:**

| Column | Type | Notes |
|--------|------|-------|
| tag_weights | jsonb | e.g. `{"minimalist": 0.95, "earth_tones": 0.88, ...}` |
| occasion_weights | jsonb | e.g. `{"everyday": 1.0, "dinner_out": 0.5}` |

**user_notification_preferences table:**

| Column | Type | Notes |
|--------|------|-------|
| user_id | uuid FK | |
| price_drops | boolean | Default: true |
| restocks | boolean | Default: true |
| new_arrivals | boolean | Default: true |

**user_preferences table:**

| Column | Type | Notes |
|--------|------|-------|
| user_id | uuid FK | |
| algorithm_opt_out | boolean | Default: false |
| deletion_requested | boolean | Default: false |
| deletion_scheduled_at | timestamp | Null until deletion requested |

For full schema, see `Backend/api/README.md`.

---

## 4. Empty/Default States

| Condition | Display |
|-----------|---------|
| No style data yet | "Complete your style profile to see your taste summary" + "Edit Preferences" CTA |
| No notification preferences set | All toggles default to ON |
| Never requested data deletion | No deletion countdown shown |

---

## 5. Error States

| Error | Display | Recovery |
|-------|---------|----------|
| Profile load fails | Toast: "Unable to load profile. Pull to refresh." | Pull to refresh |
| Data export fails | Toast: "Couldn't generate export. Try again." | Retry |
| Logout fails | Toast: "Couldn't log out. Check connection." | Retry |

---

*STROBE — Profile Spec · v1 · April 2026 · Confidential*
