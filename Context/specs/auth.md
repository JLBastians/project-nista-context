# Auth — Functional Spec

> Part of STROBE · April 2026

---

## 1. Overview

Authentication covers account creation (registration), user login, and password recovery. Auth provider (NextAuth.js or Clerk) is TBD before Phase 4.

**Priority tiers:**
- **P0 (MVP):** Email/password registration and login, password reset. Required for onboarding and personalisation.
- **P1 (Post-MVP):** OAuth via Apple and Google. Adds convenience but not required for launch.

Session tokens persist for 30 days, enabling seamless return visits across app restarts.

---

## 2. Registration (FR-AU-001 to FR-AU-004)

### 2.1 Functional Requirements

| REQ ID | Requirement | UI States | Error / Edge States |
|--------|-------------|-----------|-------------------|
| FR-AU-001 | **P0:** System presents email & password registration. **P1:** Add "Continue with Apple" and "Continue with Google" OAuth options, all visible without scrolling. | Default / Idle | If OAuth provider returns an error (P1), display: "Sign-up with [Provider] failed. Please try again or use email." |
| FR-AU-002 | User enters: First Name, Last Name, Email Address, Password, Confirm Password. Form validates all fields before submission is enabled. | Empty / Typing / Valid / Invalid / Submitting | See field-level validation in Section 2.2 |
| FR-AU-003 | On valid submission, system creates account and sends a verification email. User is navigated to Onboarding. Account is created in "unverified" state; browsing is permitted but purchasing is blocked until email is verified. | Submitting (loading spinner on button) / Success / Failure | If email already exists: "An account with this email already exists. Log in instead?" with link to Login. |
| FR-AU-004 | **(P1)** On selecting Apple or Google sign-in, system initiates the native OAuth flow. On success, if account is new, user is directed to Onboarding. If account already exists, user is directed to Feed. | OAuth modal open / Completing / Success / Cancelled | On OAuth cancellation: return to Registration with no error. On OAuth failure: toast "Something went wrong. Please try again." |

### 2.2 Field-Level Validation

| Field | Type | Required | Validation Rules | Notes |
|-------|------|----------|-----------------|-------|
| First Name | Text | Yes | 2–50 characters. Letters, hyphens, apostrophes only. No numbers. | Validated on blur and on submit. |
| Last Name | Text | Yes | 2–50 characters. Letters, hyphens, apostrophes only. No numbers. | Validated on blur and on submit. |
| Email Address | Email | Yes | Must match RFC 5322 email format. Must be unique in the system. | Uniqueness checked server-side on submit only. |
| Password | Password | Yes | Minimum 8 characters. Must contain: ≥1 uppercase, ≥1 lowercase, ≥1 number. Maximum 128 characters. | Strength indicator shown inline (Weak / Fair / Strong). Toggle visibility icon. |
| Confirm Password | Password | Yes | Must exactly match Password field value. | Validated on blur. Error: "Passwords do not match." |

### 2.3 Acceptance Criteria

| AC ID | Criterion | Test Type | Pass Condition |
|-------|-----------|-----------|----------------|
| AC-AU-001 | Registration form submit button is disabled until all fields pass client-side validation. | Functional | Button is non-interactive (greyed out) with one or more invalid fields. |
| AC-AU-002 | User receives a verification email within 2 minutes of successful registration. | Integration | Email arrives within 120 seconds. |
| AC-AU-003 | Attempting to register with an existing email shows an error without clearing the form. | Functional | Error message appears inline. All other field values preserved. |
| AC-AU-004 | **(P1)** OAuth registration creates a valid account and bypasses the email/password fields. | Integration | User directed to Onboarding after successful OAuth without completing email/password form. |

### 2.4 User Flow — Happy Path & Error States

**Happy Path:**
```
/welcome → /register (email/OAuth) → /onboarding → /feed
```

**Registration Screen (`/register`):**
- UI: Logo, first name field, last name field, email field, password field with strength indicator + show/hide toggle, confirm password field. **(P1):** Apple sign-in button, Google sign-in button. "Already have an account?" link to `/login`
- Validation: All fields validated on blur; submit button disabled until all pass
- System response: Loading spinner on submit → account created → verification email sent → user navigated to `/onboarding`

**Error States:**

| Error | Trigger | Display | Recovery |
|-------|---------|---------|----------|
| Apple/Google OAuth failure | Provider auth error | Toast: "Sign-up with [Provider] failed. Please try again or use email." | Email form visible as fallback |
| Email already registered | Duplicate email on register | Inline error below email field: "An account with this email already exists. Log in instead?" with link to `/login` | User taps link to go to `/login` |
| Weak password | Submit with invalid rules | Strength bar turns red; rules checklist highlights failing items (missing uppercase, lowercase, number, or length) | User must fix password before resubmitting |
| Network error on register | API timeout or connection failure | Toast: "Something went wrong. Please check your connection and try again." | Retry button available |

**Edge Cases:**
- User abandons registration mid-form: Form state not persisted; refreshing clears fields. No partial account created until submission.
- User navigates back from OAuth modal: Returns to registration form with no error.
- User registers, then tries to register again with same email: Error message shown on second attempt with link to `/login`.

---

## 3. Login (FR-AU-010 to FR-AU-012)

### 3.1 Functional Requirements

| REQ ID | Requirement | UI States | Error / Edge States |
|--------|-------------|-----------|-------------------|
| FR-AU-010 | Login screen presents: Email field, Password field (with show/hide toggle), "Log In" button, "Forgot Password?" link, "Continue with Apple", "Continue with Google", and "Create Account" link. | Default / Typing / Submitting / Error | On incorrect credentials: "Email or password is incorrect." (intentionally generic — do not specify which field is wrong). |
| FR-AU-011 | After 5 consecutive failed login attempts for the same email within a 15-minute window, the account is temporarily locked. A CAPTCHA challenge is presented. | Normal / Pre-lockout warning (3rd attempt) / Locked | Lockout message: "Too many login attempts. Please wait 15 minutes or reset your password." |
| FR-AU-012 | On successful login, a session token is issued with a 30-day expiry. Users remain logged in across app restarts within the expiry window. Biometric authentication (Face ID / Touch ID / Android Biometrics) can be enabled from Profile settings (future — when native apps are built). | Active session / Expired session | On session expiry, user is redirected to Login with message: "Your session expired. Please log in again." |

### 3.2 Acceptance Criteria

| AC ID | Criterion | Test Type | Pass Condition |
|-------|-----------|-----------|----------------|
| AC-AU-010 | 5 failed login attempts within 15 minutes trigger account lockout with CAPTCHA. | Security | On 5th failure, lockout message displayed and CAPTCHA rendered. Login button disabled. |
| AC-AU-011 | Successful login with valid credentials navigates user to Feed tab within 2 seconds. | Performance | Feed screen renders within 2,000ms of login API 200 response. |
| AC-AU-012 | Session persists across app termination and restart within the 30-day window. | Functional | Re-opening app within expiry window shows Feed without prompting for login. |

### 3.3 User Flow — Happy Path & Error States

**Happy Path:**
```
/welcome → /login → /feed
```

**Login Screen (`/login`):**
- UI: Logo, email field with mail icon, password field with lock icon + show/hide toggle, "Log In" button (initially disabled), "Forgot Password?" link → `/reset-password`, "Continue with Apple" button, "Continue with Google" button, "Create Account" link → `/register`
- Validation:
  - Email: must contain `@`
  - Password: ≥6 characters (client-side; server-side validation stricter)
  - Submit button disabled until both fields pass client-side validation
- System response: Loading spinner on button → `login(email, password)` → navigate to `/feed`
- Attempt tracking: Failed attempts tracked per email; counter shown after 3rd failure (pre-warning); account locked after 5th failure within 15 minutes

**Error States:**

| Error | Trigger | Display | Recovery |
|-------|---------|---------|----------|
| Invalid credentials (email or password) | Submitted credentials do not match database | Toast or inline error: "Email or password is incorrect." | Retry; attempt counter increments (visible after 3rd attempt) |
| Too many attempts (≥5) | 5 consecutive failed attempts within 15 minutes | Lockout message: "Too many login attempts. Please wait 15 minutes or reset your password." CAPTCHA rendered. Login button disabled. | User waits 15 minutes OR taps "Forgot Password?" to initiate password reset → `/reset-password` |
| OAuth failure | Provider auth error (user cancels, provider error, network) | Toast: "Sign-in failed. Please try again or use email." | Use email form as fallback |
| Network error | API timeout or connection failure | Toast: "Something went wrong. Please check your connection." | Retry button available |
| Session expired during session | Token expires after 30 days of issue date or invalidated by server | Redirect to `/login` with message: "Your session expired. Please log in again." | User logs in again |

**Edge Cases:**
- Already authenticated user visits `/login`: Redirect to `/feed` automatically.
- User logs in from multiple devices: Each device receives its own session token with independent 30-day expiry.
- Session expires mid-action (e.g., submitting a form): Redirect to `/login`; intended destination not preserved in v1.0.

---

## 4. Password Reset (FR-AU-020 to FR-AU-022)

### 4.1 Functional Requirements

| REQ ID | Requirement | UI States | Error / Edge States |
|--------|-------------|-----------|-------------------|
| FR-AU-020 | User enters their email address and submits. System sends a reset link regardless of whether the email exists (to prevent account enumeration). Screen shows: "If an account exists for this email, a reset link has been sent." | Default / Submitting / Sent confirmation | Network failure: "Unable to send. Check your connection and try again." |
| FR-AU-021 | Reset link contains a time-limited token (valid for 60 minutes). Opening the link directs user to a New Password screen. Expired tokens show an error with a "Request a new link" CTA. | Valid token / Expired token / Already used | Expired: "This link has expired. Please request a new one." Already used: "This reset link has already been used. Request a new one." |
| FR-AU-022 | User sets a new password meeting the same validation rules as Registration. On success, all existing sessions are invalidated and user is redirected to Login. | Typing / Submitting / Success | Same password as previous: "Your new password must be different from your current password." |

### 4.2 Acceptance Criteria

| AC ID | Criterion | Test Type | Pass Condition |
|-------|-----------|-----------|----------------|
| AC-AU-020 | Password reset email sent within 2 minutes for valid email addresses. | Integration | Email arrives within 120 seconds of form submission. |
| AC-AU-021 | Reset token expires after 60 minutes. | Security | Attempting to use token after 60 minutes shows expiry error. |
| AC-AU-022 | All active sessions invalidated after password change. | Security | Existing session tokens no longer authenticate; user must log in again. |

### 4.3 User Flow — Happy Path & Error States

**Happy Path:**
```
/login → "Forgot Password?" → /reset-password (email input) → [email sent confirmation] → [user opens email link] → [new password form] → /login
```

**Reset Request Screen (`/reset-password`):**
- UI: Logo, email input field with mail icon, "Send Reset Link" button (initially disabled), "Back to Sign In" link → `/login`
- Validation: Email must contain `@`; submit button disabled until valid
- System response: Loading spinner on button → success state shown
- Security: Same success message regardless of email existence (prevents account enumeration)

**Success State (after email submission):**
- UI: Green checkmark icon (or similar), heading "Check Your Email", confirmation copy: "If an account exists for [email], a reset link has been sent. The link expires in 60 minutes." "Back to Sign In" button → `/login`
- No countdown timer shown; user clicks link in email

**Reset Token / New Password Screen (user clicks email link):**
- UI: Logo, new password field with strength indicator + show/hide toggle, confirm password field, "Set New Password" button
- Validation: Same rules as Registration (≥8 chars, uppercase, lowercase, number); confirm password must match; new password cannot be same as current (if account existed)
- System response: Loading spinner → password updated → all sessions invalidated → redirect to `/login` with message: "Your password has been reset. Please log in."

**Error States:**

| Error | Trigger | Display | Recovery |
|-------|---------|---------|----------|
| Invalid email format | Submit with malformed email | Button disabled (form validation); no error toast shown | Enter valid email format |
| Network error on email send | API timeout or connection failure | Toast: "Failed to send reset email. Try again." | Retry button available; user can resubmit |
| Expired reset token | User clicks email link after 60 minutes | Error page: "This link has expired. Please request a new one." with "Request a new link" button → back to `/reset-password` | User returns to `/reset-password` and requests new link |
| Already-used reset token | User clicks same reset link twice | Error page: "This reset link has already been used. Request a new one." | User requests new link via `/reset-password` |
| New password same as old | User enters password matching their current password | Inline error: "Your new password must be different from your current password." | User enters different password |
| Weak new password | Submit with invalid rules | Strength bar turns red; rules checklist highlights failures | User fixes password |

**Edge Cases:**
- User requests password reset twice: First link expires; second link sent. User can use second link or request another.
- User clicks reset email on different device: Reset link works (token is device-agnostic); user can set new password on any device.
- User submits password reset form without email: Button disabled by form validation; no submission allowed.

---

## 5. Data Model

### 5.1 Users Table Schema

Key auth-related fields (full schema in `Backend/api/README.md`):

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| id | UUID (PK) | Yes | User account identifier |
| email | TEXT | Yes | Unique email address; used for login and password reset |
| password_hash | TEXT | No | Bcrypt hash of password (null for OAuth-only accounts) |
| first_name | TEXT | Yes | User's first name |
| last_name | TEXT | Yes | User's last name |
| email_verified | BOOLEAN | Yes (default: false) | True after user clicks verification link in email |
| oauth_provider | TEXT | No | "apple" or "google" if account created via OAuth |
| oauth_id | TEXT | No | Provider's user ID (for OAuth accounts) |
| created_at | TIMESTAMP | Yes | Account creation timestamp |
| last_login_at | TIMESTAMP | No | Most recent successful login timestamp |
| onboarding_complete | BOOLEAN | Yes (default: false) | True once 5-step onboarding finished or explicitly skipped |
| onboarding_completed_at | TIMESTAMP | No | When onboarding was finished |

### 5.2 Authentication Session Storage

Sessions are managed by the auth provider (NextAuth.js or Clerk TBD). Session tokens:
- **Token type:** JWT (JSON Web Token) or provider-specific session token
- **Expiry:** 30 days from issuance
- **Storage:** Secure httpOnly cookie (server-side session state) or provider-managed session
- **Validation:** On every authenticated request; expired tokens trigger redirect to `/login`

### 5.3 Password Reset Tokens Table

| Field | Type | Notes |
|-------|------|-------|
| id | UUID (PK) | Reset token identifier |
| user_id | UUID (FK) | Links to users table |
| token_hash | TEXT | Bcrypt hash of reset token (token itself never stored) |
| expires_at | TIMESTAMP | 60 minutes after creation |
| used_at | TIMESTAMP | Null until token is redeemed; set on successful password change |

---

## 6. Integration Notes

- **Email service:** Sends verification emails and password reset emails. Must arrive within 2 minutes (requirement AC-AU-002, AC-AU-020).
- **OAuth providers:** Apple Sign-In and Google Sign-In. Both must return user email and first/last name (or fallback logic if unavailable).
- **Rate limiting:** 5 failed login attempts per email per 15-minute window triggers CAPTCHA + account lock.
- **Session validation:** Auth provider manages token validation; frontend redirects to `/login` on 401 responses.

---

*STROBE Auth · Functional Spec · April 2026 · Confidential*
