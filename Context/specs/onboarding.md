# Onboarding — Functional Spec

> Part of STROBE · April 2026

---

## 1. Overview

Onboarding is a 5-step flow (v0) presented immediately after account creation. Every question feeds the recommendation engine, collecting state attributes (age, gender expression, city, lifestyle activities, brand selections) and seeding taste vectors from brand prior. The flow is designed for completion in under 2 minutes. Each step is skippable; skipped steps degrade initial feed quality. The flow is shown only once per account; returning incomplete users are prompted via banner, not forced through the flow.

**P1 Enhancement:** A swipe mechanic (10 curated product cards) is planned as a P1 enhancement to provide stronger initial taste vector seeding.

---

## 2. Onboarding Outputs

### 2.1 State Attributes

Fixed profile fields stored on the `users` table, acting as permanent filters:

| Attribute | Type | Source | Notes |
|-----------|------|--------|-------|
| date_of_birth | date | Step 1 (Age) | Age calculated dynamically; used for age-appropriate curation, cohort analysis |
| gender_expression | text[] | Step 2 (Gender Expression) | Array of selections; e.g. ["womenswear", "androgynous"]. Used for product gender filtering. |
| city | text | Step 3 (City) | City name string; used for climate-relevant curation, occasion norms |
| city_source | text | Step 3 (City) | "ip" (geolocation) or "manual" (user input) |
| sustainability_orientation | text | Step 5 (Brand Selections) | Inferred: "high" / "mixed" / "low" / "secondhand" |
| price_signal | text | Step 5 (Brand Selections) | Inferred from brand selections: "investment" / "mid" / "value" / "mixed" |

### 2.2 Seed Taste Vectors

Per-occasion vectors written to `user_taste_profiles` table (v0):

| Vector | Source | Notes |
|--------|--------|-------|
| everyday_vector | Step 4 (Activities) + Step 5 (Brand Selections) | In v0, seeded from Step 4 occasion weight (Often/Sometimes/Rarely = 1.0/0.5/0.15) + Step 5 brand prior only. No blending in v0. |
| going_out_vector | Step 4 + Step 5 | Same weighting structure. Brand prior is the primary seed. |
| work_vector | Step 4 + Step 5 | Same weighting structure. Brand prior is the primary seed. |
| special_vector | Step 4 + Step 5 | Same weighting structure. Brand prior is the primary seed. |
| brand_prior_vector | Step 5 (Brand Selections) | Average of selected brands' pre-computed embeddings; primary taste vector seed in v0. In P1, will be blended 60/40 with swipe signal. |
| occasion_weights | Step 4 (Lifestyle Activities) | JSONB: e.g. {"everyday": 1.0, "dinner_out": 0.5, "formal": 0.15}; maps activities to occasion vectors |

---

## 3. Step-by-Step Specification (FR-ON-001 to FR-ON-007)

### 3.1 Step 1 — Age (FR-ON-001)

**Purpose:** Age-appropriate curation, cohort analysis

**Input:** Birth year via dropdown/scroll picker

**Validation:**
- Age must be 13–100 years old
- Stored as date_of_birth on user record
- Age calculated dynamically from date_of_birth

**Storage:**
- Field: `users.date_of_birth` (date type)
- Constraint: NOT NULL if step completed; NULL if skipped

**Framing:** "How old are you? We'll use this to tailor your edit."

**UI States:**
- Default: Placeholder text "Select your birth year" or equivalent
- Selected: Year/age displayed in picker
- Skipped: Picker cleared; next step shows optional badge

**Acceptance:** User taps Next → validation runs → proceed to Step 2 or taps Skip

**Error Handling:** If user enters invalid year (future or pre-1900s), picker prevents selection or shows validation error

### 3.2 Step 2 — Gender Expression (FR-ON-002)

**Purpose:** Product gender filtering in feed

**Input:** Multi-select from four options:
1. Womenswear
2. Menswear
3. Androgynous / gender-neutral
4. I dress across all of the above

**Validation:**
- ≥1 selection required if step not skipped
- Users who select "all of the above" receive unfiltered product results
- Users who select 1–3 options receive filtered results for those genders only

**Storage:**
- Field: `users.gender_expression` (text array type)
- Example: `["womenswear", "androgynous"]`
- Constraint: NOT NULL and non-empty if step completed; NULL if skipped

**Framing:** "How do you like to dress?"

**UI States:**
- Default: Four buttons/cards, none selected
- 1+ selected: Selected buttons highlighted; visual feedback
- Skipped: Buttons cleared; next step shows optional badge

**Acceptance:** User selects ≥1 option (if not skipping) → taps Next → proceed to Step 3

**Error Handling:** If user tries to proceed without selecting (and step not skipped), show inline error: "Please select at least one option" or disable Next button

### 3.3 Step 3 — City (FR-ON-003)

**Purpose:** Climate-relevant curation, occasion norms

**Input:** IP geolocation pre-fill with manual override via autocomplete

**Validation:**
- Pre-filled city from ipapi.co or equivalent (no precise geolocation stored — city name only)
- Manual override available via text input with US city autocomplete
- No validation error if geolocation fails; city field simply empty

**Storage:**
- Field: `users.city` (text type, city name string only)
- Field: `users.city_source` (text type, "ip" or "manual")
- Constraint: Both NULL if skipped; both populated if completed

**Framing (pre-fill):** "Is this where you're based? [Detected City]" with "Yes" button, "Enter different city" button, or "Skip" button

**Framing (manual entry):** "Where are you based?" with text input + autocomplete dropdown

**UI States:**
- Pre-filled: City displayed with confirmation prompt
- Manual entry: Text input with autocomplete suggestions appearing as user types
- Confirmed: City name shown; ready to proceed
- Skipped: Field cleared; next step shows optional badge

**Error Handling:**
- Geolocation fails: City field is empty with placeholder "Enter your city". No error toast shown to user.
- User types non-existent city: Autocomplete shows no matches. User can proceed anyway (store as-is) or refine search.

**Acceptance:** User confirms pre-filled city OR manually enters city → taps Next → proceed to Step 4. User taps Skip → proceed to Step 4 with no city data.

### 3.4 Step 4 — Lifestyle Activities (FR-ON-004)

**Purpose:** Occasion taste vectors — frequency maps to vector weighting

**Input:** Multi-select activity grid, with frequency selector per activity

**Activity Grid & Internal Occasion Mapping:**

| Activity (shown to user) | Internal Occasion Mapping | Frequency weight: Often | Frequency weight: Sometimes | Frequency weight: Rarely |
|--------------------------|---------------------------|------------------------|---------------------------|-------------------------|
| Everyday / errands | Everyday occasion vector | 1.0 | 0.5 | 0.15 |
| Working in an office | Office / corporate vector | 1.0 | 0.5 | 0.15 |
| Creative or casual workplace | Creative workplace vector | 1.0 | 0.5 | 0.15 |
| Working from home | Everyday + WFH vector | 1.0 | 0.5 | 0.15 |
| Dinner out / restaurants | Dinner / elevated casual vector | 1.0 | 0.5 | 0.15 |
| Cocktail events / parties | Cocktail / semi-formal vector | 1.0 | 0.5 | 0.15 |
| Formal events / galas | Formal / black tie vector | 1.0 | 0.5 | 0.15 |
| Weekends away / travel | Travel vector | 1.0 | 0.5 | 0.15 |
| Gym / fitness | Activewear vector | 1.0 | 0.5 | 0.15 |
| Outdoor activities / hiking | Outdoor / active vector | 1.0 | 0.5 | 0.15 |
| Beach / resort | Resort / swimwear vector | 1.0 | 0.5 | 0.15 |
| Festivals / concerts | Festival / creative event vector | 1.0 | 0.5 | 0.15 |
| Date nights | Date night / going out vector | 1.0 | 0.5 | 0.15 |
| Weddings / celebrations | Wedding guest / special occasion vector | 1.0 | 0.5 | 0.15 |

**Validation:**
- ≥1 activity selection required if step not skipped
- For each selected activity, frequency must be chosen
- No frequency pre-selected; user must explicitly choose Often / Sometimes / Rarely

**Storage:**
- Field: `user_taste_profiles.occasion_weights` (JSONB type)
- Example: `{"everyday": 1.0, "dinner_out": 0.5, "formal": 0.15}`
- Constraint: NOT NULL if step completed; NULL or empty object if skipped
- Frequency weight value used directly in taste vector calculation

**Framing:** "How often do these activities fit your life?"

**UI States:**
- Default: Grid of 14 activity cards, none selected, no frequency selectors visible
- 1+ selected: Selected activities highlighted; frequency selector appears inline (Often / Sometimes / Rarely buttons)
- Completed: All selected activities show frequency choice; Next button enabled
- Skipped: Grid cleared; next step shows optional badge

**Acceptance:** User selects ≥1 activity (if not skipping) → chooses frequency for each → taps Next → proceed to Step 5. User taps Skip → proceed to Step 5 with no occasion weights.

**Error Handling:**
- User selects activity but doesn't choose frequency: Frequency selector shows error or disables Next until frequency chosen
- User tries to proceed with incomplete selections: Inline error or disabled Next button

### 3.5 Step 5 — Brand Selections (FR-ON-005)

**Purpose:** Sustainability inference, initial style vector seed (brand-derived prior)

**Input:** Searchable brand list with autocomplete + curated quick-select grid (~30 brands). Option to select "I shop vintage/secondhand primarily" instead.

**Validation:**
- 3–10 brand selections required if not skipping (and not selecting vintage/secondhand option)
- "I shop vintage/secondhand primarily" option available without requiring brand selection
- Brand search must match brands in internal catalogue; non-existent brands show "No matching brands" in dropdown

**Storage:**
- Table: `onboarding_brand_selections` (id, user_id, brand_name, brand_id FK, selected_at)
- Field: `users.sustainability_orientation` (inferred, stored on users table)
- Field: `user_taste_profiles.brand_prior_vector` (768-dim embedding, averaged from selected brands)

**Signal 1 — Sustainability Orientation:**

User's brand selections cross-referenced against internal brand sustainability taxonomy:

| Pattern | Stored Value | Notes |
|---------|-------------|-------|
| 3+ brands tagged as certified sustainable / ethical / B-Corp | "high" | High sustainability commitment |
| Mix of sustainable and conventional brands | "mixed" | Balanced approach |
| Conventional brands only, or fewer than 3 selections | "low" | Conventional focus |
| User selects vintage / secondhand option | "secondhand" | Vintage/secondhand primary |

**Signal 2 — Initial Style Vector Seed:**

Each brand has a pre-computed Marqo embedding (brand-level style vector, computed by cofounder). Selected brands' vectors are averaged to produce a first-cut taste vector. In v0, this is the primary taste vector seed. When the P1 swipe mechanic is implemented, it will be blended 60/40 (60% brand prior, 40% swipe signal).

**Framing (search/grid):** "Brands you know and love — what do you shop?" or "Select the brands you gravitate towards"

**Framing (vintage/secondhand option):** "I mainly shop vintage, thrift, or secondhand"

**UI States:**
- Default: Search bar + quick-select grid (~30 curated brands), none selected, count showing "0/10 selected"
- Searching: User types in search field; dropdown shows matching brands
- 3–10 selected: Selected brands shown as chips/pills; count updated
- Vintage selected: Vintage option highlighted; brand selection disabled
- Completed: All 3–10 (or vintage selected) shown; Next button enabled
- Skipped: All selections cleared; next step shows optional badge

**Acceptance:**
- Path 1: User selects 3–10 brands → taps Next → proceed to Completion. Sustainability inferred from selections.
- Path 2: User selects "I shop vintage/secondhand primarily" → taps Next → proceed to Completion. sustainability_orientation set to "secondhand".
- Path 3: User taps Skip → proceed to Completion with no brand data.

**Error Handling:**
- Brand search returns no match: "No matching brands" message in dropdown. User can try different spelling or select from grid.
- User selects <3 brands (and not vintage): Inline error or disabled Next: "Select at least 3 brands (or choose vintage/secondhand)"
- User tries to select both brands and vintage: Selecting vintage disables brand selections (vice versa).

### 3.6 P1 Enhancement — Swipe Mechanic

**Priority: P1 (Post-MVP).** This step is not included in the v0 onboarding flow. It is documented here for future implementation.

**Purpose:** Direct taste vector seeding — product-level signal to refine taste vectors (when implemented in P1)

**Input:** Full-screen card stack. 10 curated product cards from STROBE catalogue. Swipe right = like, swipe left = pass. Binary forced choice — no ratings, no skip on individual cards.

**Card Selection Algorithm (Cofounder Scope):**
- Filter catalogue by user's top 2 occasions (from Step 4) and gender expression (from Step 2)
- Use brand-derived style vector (from Step 5) to select products distributed across the likely style space — not clustered (include items user will reject for informative negative signal)
- Allocate cards proportionally to occasion frequency (e.g., 6 for top occasion, 4 for second)
- Ensure visual diversity — no two cards should look nearly identical

**Vector Computation After 10 Swipes (P1):**

1. Right-swiped product vectors averaged with weight +1.0
2. Left-swiped product vectors averaged with weight −0.3 (mild negative — user may have passed for wrong colour or already-owned, not wrong aesthetic)
3. Resulting vector blended 60/40 with brand-derived prior from Step 5:
   - `initial_taste_vector = (0.6 × brand_prior) + (0.4 × swipe_signal)`
4. Combined vector written as initial taste_vector in user_taste_profiles (per occasion, not global)

**Storage (P1):**
- Table: `onboarding_swipe_responses` (id, user_id, product_id FK, response: "like" or "pass", card_position: 1–10, responded_at)
- Field: `user_taste_profiles.taste_vector` (or occasion-specific: everyday_vector, going_out_vector, work_vector, special_vector)
- Field: `user_taste_profiles.seed_method` ("swipe" / "brand_prior_only" / "skipped")

**Framing:** "Show us your style — swipe right on pieces you love"

**UI States:**
- Card stack visible: Current card shown, previous/next cards slightly visible (stacked effect)
- Swiping: Card animates left or right as user swipes
- All 10 complete: Success animation; "Loading feed..." transition
- Loading feed: Spinner + "We're building your feed..." copy for up to 3 seconds

**Acceptance (P1):** User swipes all 10 cards → vectors computed → loading state → navigate to `/feed`

**Error Handling (P1):**
- Card generation fails (insufficient products matching criteria): Fall back to trending/editorial product cards. Note shown: "We're still building your catalogue — these are some of our favourites."
- Network error during swipe: Swipe response not saved; card remains. Toast: "We had trouble saving your choice. Try again."
- User skips Step 6 before starting swipes: Proceed directly to Step 7 (Completion). No swipe responses recorded. seed_method = "brand_prior_only" or "skipped".

**Note:** The swipe step itself is not skippable once entered (all 10 cards must be swiped), but the user can skip the entire onboarding flow before reaching swipes. **In v0, this step does not exist.**

### 3.7 Onboarding Completion (FR-ON-007)

**Purpose:** Finalize taste vectors, trigger feed generation, land user on For You feed

**On Completion (or skip from any step):**
1. System compiles all onboarding data
2. Taste vectors seeded from brand prior only (Step 5 completed) or all steps skipped (no seeding)
3. Feed generation triggered
4. Loading state shown: "We're building your feed..." for up to 3 seconds
5. User navigates to `/feed`

**Feed Landing:**
- First feed renders with skeleton → stagger animation on product cards
- Subtle message: "Your edit is ready — it'll keep getting better as you explore."
- Occasion filter toggle visible (Everyday / Going Out / Work / Special / All)

**UI States:**
- Compiling: Loading spinner + "We're building your feed..." copy
- Success: Feed renders; message displayed
- Failure: Show trending feed with persistent banner: "We had trouble personalising your feed. We'll keep learning as you browse!" with dismiss button

**Error Handling:**
- Feed generation fails: Fallback to trending (most-saved products last 7 days) + banner explaining system error. Personalisation can resume once user generates interactions.
- User skips all steps: Generic trending feed displayed; persistent banner: "Complete your profile to personalise your feed." with CTA → `/onboarding`

**Storage — Completion Marker (v0):**
- Field: `users.onboarding_complete` = true
- Field: `users.onboarding_completed_at` = timestamp
- Constraint: NOT NULL once Step 5 finished or all steps skipped

---

## 4. User Flows

### 4.1 Happy Path

```
/register → /onboarding (5 steps) → /feed
```

**Step-by-step (detailed in Section 3):**
1. Age picker → validation (13–100) → Next → Step 2
2. Gender expression multi-select (≥1) → Next → Step 3
3. City confirmation or manual entry → Next → Step 4
4. Activities grid with frequency selectors (≥1) → Next → Step 5
5. Brand search + selection (3–10 or vintage/secondhand) → vectors computed → loading → /feed

**Total flow time:** Target ≤2 minutes (95th percentile ≤2 minutes per AC-ON-001)

**Note:** The swipe mechanic (10 curated product cards) is planned as a P1 enhancement after v0 launch.

### 4.2 Error States Table

| Error | Trigger | Display | Recovery |
|-------|---------|---------|----------|
| Step validation fails (e.g., age out of range) | Invalid input | Inline error or disabled Next button | User corrects input |
| IP geolocation fails (Step 3) | Geolocation API error or timeout | City field empty with placeholder "Enter your city". No error toast. | User types city manually |
| Brand search no results (Step 5) | User types brand not in catalogue | "No matching brands" in dropdown | Try different spelling; select from grid; skip step |
| Swipe card generation fails (Step 6) | Insufficient products matching occasion/gender/style | Fall back to trending/editorial cards; note: "We're still building our catalogue" | Proceed with fallback cards |
| Network error during swipe save | API timeout on swipe POST | Toast: "We had trouble saving. Try again." Card remains unresolved. | User re-swipes card |
| Feed generation fails (Step 7) | Personalisation engine error | Show trending feed + banner: "We had trouble personalising your feed..." | Dismiss banner; browse trending; personalisation resumes with next interaction |

### 4.3 Edge Cases

| Case | Behavior |
|------|----------|
| User abandons mid-onboarding, refreshes page | Form state not persisted. Reloads from beginning of current step. No partial data saved. |
| User navigates back (browser back button) during onboarding | Returns to previous step. All prior selections preserved. |
| User skips individual steps | Skipped steps store no data. Feed quality degrades proportionally. Persistent "Complete Your Profile" banner appears on feed. |
| User skips entire onboarding flow before entering (never reaches Step 1) | Not possible — onboarding forced after `/register` → user must enter onboarding flow. Skip links within each step allow skipping individual steps. |
| Session expires during onboarding | User redirected to `/login`. On re-login, onboarding resumes from beginning (no partial state saved in v1.0). |
| User who skipped returns to complete later | Profile → "Complete Your Profile" CTA → `/onboarding`. Previously completed steps shown as complete; resume from first incomplete step. |
| User's occasion weights empty but Step 4 skipped | Feed generated without occasion filtering. No occasion vectors seeded. User sees editorial/trending feed. Occasion filter toggle disabled or shows "All" only until data available. |
| Brand prior vector null (Step 5 skipped) | In v0, there is no fallback seeding. User gets editorial/trending feed. seed_method = "skipped". |

---

## 5. Data Model

### 5.1 Fields on users Table

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| id | UUID (PK) | Yes | User account identifier |
| date_of_birth | date | No | Derived from birth year input (Step 1). Age calculated dynamically. |
| gender_expression | text[] | No | Array e.g. ["womenswear", "androgynous"] (Step 2). |
| city | text | No | City name string (Step 3). |
| city_source | text | No | "ip" or "manual" (Step 3). |
| sustainability_orientation | text | No | Inferred from brands: "high" / "mixed" / "low" / "secondhand" (Step 5). |
| price_signal | text | No | Inferred from brands: "investment" / "mid" / "value" / "mixed" (Step 5). |
| onboarding_complete | boolean | Yes (default: false) | True once Step 5 finished or all steps explicitly skipped (v0). |
| onboarding_completed_at | timestamp | No | When onboarding was finished. |

### 5.2 Fields on user_taste_profiles Table

| Field | Type | Notes |
|-------|------|-------|
| user_id | UUID (FK) | Links to users table |
| seed_method | text | In v0: "brand_prior_only" (Step 5 completed) / "skipped" (all steps skipped). In P1: "swipe" will also be available. |
| occasion_weights | jsonb | e.g. {"everyday": 1.0, "dinner_out": 0.5, "formal": 0.15} (Step 4). Frequency weights per occasion. |
| brand_prior_vector | vector(768) | Brand-derived style vector from Step 5 (averaged brand embeddings). Primary seed in v0. Stored separately for debugging. |
| everyday_vector | vector(768) | Per-occasion taste vector. In v0, seeded from Step 4 weight + Step 5 brand prior only. |
| going_out_vector | vector(768) | Per-occasion taste vector. In v0, seeded from Step 4 weight + Step 5 brand prior only. |
| work_vector | vector(768) | Per-occasion taste vector. In v0, seeded from Step 4 weight + Step 5 brand prior only. |
| special_vector | vector(768) | Per-occasion taste vector. In v0, seeded from Step 4 weight + Step 5 brand prior only. |
| created_at | timestamp | When taste profiles were first seeded. |
| updated_at | timestamp | When taste profiles were last recalculated. |

### 5.3 onboarding_brand_selections Table

| Field | Type | Notes |
|-------|------|-------|
| id | UUID (PK) | Record identifier |
| user_id | UUID (FK) | Links to users table |
| brand_name | text | Brand name as entered/selected by user |
| brand_id | UUID (FK, nullable) | Links to brands table; nullable if brand not in catalogue |
| selected_at | timestamp | When user selected this brand |

### 5.4 onboarding_swipe_responses Table

| Field | Type | Notes |
|-------|------|-------|
| id | UUID (PK) | Record identifier |
| user_id | UUID (FK) | Links to users table |
| product_id | UUID (FK) | Links to products table |
| response | text | "like" or "pass" |
| card_position | integer | 1–10 (position in card stack) |
| responded_at | timestamp | When user swiped |

**Note:** This table is used by the P1 swipe mechanic enhancement. Not populated in v0.

---

## 6. Acceptance Criteria

| AC ID | Criterion | Test Type | Pass Condition |
|-------|-----------|-----------|----------------|
| AC-ON-001 | Onboarding completable in ≤2 minutes | Usability | 95th percentile ≤2 minutes in user testing (n≥20) |
| AC-ON-002 | Back navigation preserves selections on all steps | Functional | Navigating back shows previously made selections; no data loss |
| AC-ON-003 | Skipping any step proceeds to next step | Functional | Skip link visible on all steps; tapping advances without storing data for that step |
| AC-ON-004 | First feed loads within 3 seconds of completion | Performance | Feed renders with ≥20 product cards within 3,000ms on 4G network |
| AC-ON-005 | Swipe mechanic presents 10 visually diverse cards | Functional | P1 only. No two cards from same brand; visual diversity verified by QA |
| AC-ON-006 | Taste vector written on completion | Integration | In v0, taste vector(s) seeded from brand prior. Stored in user_taste_profiles with seed_method = "brand_prior_only" or "skipped". |

---

*STROBE Onboarding · Functional Spec · April 2026 · Confidential*
