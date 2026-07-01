# Riverly API — Frontend Reference
> Staging: `https://api-staging.riverly.ng` — Production: `https://api.riverly.ng`
> Interactive docs: `/swagger` on either host
> Last updated: 2026-07-01

---

## Table of Contents

### Basics
- [Global Conventions](#global-conventions) — response format, auth header, rate limits, device headers

### Endpoints (what to send, what you get back)
- [0. Unified Identity (Preferred)](#0-unified-identity--apiv1identity-preferred) — signup, phone/email OTP, BVN, liveness, enroll, login, refresh, biometric, device revoke, profile — **use this for new Personal work**
- [1. Auth](#1-auth--apiv1auth-legacy--personal-only) — legacy Personal login, logout, refresh, passcode reset, biometric, profile
- [2. Onboarding](#2-onboarding--apiv1onboarding) — legacy signup steps: phone → email → BVN/NIN → profile → passcode → selfie
- [3. Accounts](#3-accounts--apiv1accounts) — create account, get balances, set/change PIN
- [4. Transactions](#4-transactions--apiv1accountsaccountidtransactions) — transaction history, filters, detail view
- [5. Transfers](#5-transfers--apiv1transfers) — send money to another Riverly account
- [6. Beneficiaries](#6-beneficiaries--apiv1beneficiaries) — save, edit, delete saved recipients
- [7. Bill Payments](#7-bill-payments--apiv1payments) — airtime, data, electricity, cable TV
- [8. KYC](#8-kyc--apiv1kyc) — document upload, submit for review
- [9. Dashboard](#9-dashboard--apiv1dashboard) — home screen data
- [10. Statements](#10-statements--apiv1statements) — request bank statements
- [11. Notifications](#11-notifications--apiv1notifications) — push tokens, preferences
- [12. Chat](#12-chat--apiv1chat) — support chat
- [13. Support](#13-support--apiv1support) — contact info

### Step-by-Step Flows (what to call and when)
- [Onboarding Flow](#flow-onboarding-new-user-signup) — full signup from phone number to selfie
- [Login Flow](#flow-login-existing-user) — passcode login + new device verification
- [Biometric Login Flow](#flow-biometric-login-app-launch) — Face ID / fingerprint login on app open
- [Buy Airtime Flow](#flow-buy-airtime) — select network → enter amount → PIN → receipt
- [Buy Data Flow](#flow-buy-data) — select network → pick plan → PIN → receipt
- [Pay Bill Flow](#flow-pay-bill-electricity-cable-tv-etc) — categories → biller → validate → PIN → receipt
- [Transfer Flow](#flow-transfer-to-riverly-account-internal) — enter account → confirm → PIN → receipt
- [Transaction History Flow](#flow-view-transaction-history) — list, filter, detail
- [Forgot Passcode Flow](#flow-forgot-passcode) — OTP → reset token → new passcode
- [KYC Upload Flow](#flow-kyc-document-upload-post-onboarding-tier-upgrade) — upload docs → submit
- [Dashboard Flow](#flow-dashboard-home-screen) — what to load on home screen
- [Bank Statement Flow](#flow-request-bank-statement) — request → poll → download
- [Beneficiaries Flow](#flow-manage-beneficiaries) — add, edit, delete saved recipients

### Quick Reference
- [Field Names to Watch](#quick-reference--field-names-to-watch) — common mistakes and correct field names

---

## Global Conventions

### Every response has this wrapper:
```json
{
  "status": true,
  "message": "Request completed successfully.",
  "data": { }
}
```
- `status: true` = success, `status: false` = failure
- `data` is always inside the wrapper, not at the root level

### Auth header (for all protected endpoints):
```
Authorization: Bearer <accessToken>
```

### Rate limits:
| Policy | Applies to | Limit |
|--------|-----------|-------|
| `auth` | Login, new-device login | 10 req/min |
| `otp` | Initiate phone OTP | 5 req/min |
| `ip_policy` | All endpoints | 20 req/min per IP |

Returns **HTTP 429** when exceeded.

### Device Headers (required where marked ✅):
```
X-Device-Id: <unique UUID, generated once on install>   ← REQUIRED
X-Device-Name: "iPhone 14 Pro"                          ← optional
X-Device-Type: "ios" | "android" | "web"                ← optional
X-Device-Os: "iOS 17.1"                                 ← optional
```

---

## 0. Unified Identity — `/api/v1/identity` (Preferred)
> The single onboarding + auth surface across Personal, SME, and Corporate.
> **This is the path the Personal mobile app should use going forward.** The
> legacy `/auth/*` and `/personal/onboarding/*` sections below still work
> during the migration window but won't get new features.

**Token shapes on this path:**

- **Identity token** — short-lived (15 min). Returned by signup / OTP verify
  / login. Only usable against `/identity/*` endpoints. Callers use it while
  finishing onboarding.
- **Product-scoped token** — 7-day. Returned by `POST /identity/switch-product`.
  This is the token for `/personal/*`, `/sme/*`, `/corporate/*` endpoints.
- **Refresh token** — 30-day. Returned alongside the product token; rotate
  via `POST /identity/refresh`.

**Envelope + Auth header + Device headers** — same as Global Conventions above.

### Onboarding

#### POST `/api/v1/identity/signup`
Create a new identity. First screen of Personal, SME, or Corporate signup.
Device headers ✅ (via `[RequiresDeviceHeaders]` on Login only — signup itself
doesn't enforce them but including them lets us register the device on first
enrollment).

Request:
```json
{
  "firstName": "Aishatu",         // optional for Personal, required for SME/Corporate
  "surname": "Sadiq",             // optional for Personal, required for SME/Corporate
  "phoneNumber": "+2349990000001",// required (test-range +234999XXXXXXX for staging OTP echo)
  "email": "user@example.com",    // optional for Personal, required for SME/Corporate
  "productType": 0,               // 0 = Personal, 1 = Sme, 2 = Corporate
  "referralCode": "RIVERLY01"     // optional Personal only; validated against existing users.ReferralCode
}
```

Response `data`:
```json
{
  "accessToken": "…identity JWT…",
  "identityId": "…",
  "phoneNumber": "+234...",
  "emailVerified": false,
  "phoneVerified": false,
  "onboardingCompleted": false,
  "passcodeSet": false,
  "availableProducts": [],
  "defaultProduct": null,
  "nextStep": "VerifyPhone",         // high-level cross-product hint
  "personalNextStep": "PhoneVerification",  // Personal-only, matches legacy PersonalOnboardingStep enum
  "debugOtpCode": "123456"           // staging only, +234999 numbers only
}
```

**Errors:** referral code invalid → `"Invalid referral code."`; email already in use on another identity → structured `FailWithData` with `nextStep: "Login"` and `existingAccount: true`.

#### POST `/api/v1/identity/phone/verify`
Auth: identity token.
Request: `{ "code": "123456" }` (5 for SME apk, 6 for Personal — see the OTP-length notes)
Response: full auth-response shape (same as signup) — `phoneVerified: true`, `personalNextStep` advances.

#### POST `/api/v1/identity/phone/resend`
Auth: identity token. Empty body. Returns `{ sent: true, debugOtpCode: "…"? }`.

#### POST `/api/v1/identity/phone/skip`
Auth: identity token. Personal only — skip phone step (rarely used).

#### POST `/api/v1/identity/email/add`
Auth: identity token. Personal attaches email after phone verify.
Request: `{ "email": "user@example.com" }`. Dispatches email OTP.

#### POST `/api/v1/identity/email/verify`
Auth: identity token. Request: `{ "code": "123456" }`. Personal + SME + Corporate.

#### POST `/api/v1/identity/email/resend`
Auth: identity token. Empty body.

#### POST `/api/v1/identity/passcode/set`
Auth: identity token. Sets the login credential once ANY contact is verified.
Personal: `{ "passcode": "123456", "confirmPasscode": "123456" }` (6 digits).
SME/Corporate: 8+ char password with upper/lower/digit/special. Server validates against the identity's `signupProductType`.

#### POST `/api/v1/identity/kyc/bvn/verify` ← NEW
Auth: identity token. Personal BVN verification via Dojah.
Request: `{ "bvn": "12345678901" }` (11 digits).
Response `data`:
```json
{
  "verified": true,
  "firstName": "Aishatu",
  "lastName": "Sadiq",
  "middleName": null,
  "dateOfBirth": "1990-01-15"
}
```
Prod-only: rejects with `"This BVN is already linked to another Riverly account."` if a different user already has it. Staging keeps the check off so QA can reuse test BVNs.

**Sandbox caveat:** on staging, Providus/XpressWallet's own sandbox refuses to provision two wallets with the same BVN. If two QA sessions use the same test BVN, the second wallet creation will fail with `"Wallet creation failed."` — use a different test BVN per user.

#### POST `/api/v1/identity/kyc/liveness/check` ← NEW
Auth: identity token. Personal liveness via Dojah.
Request: `{ "image": "…base64…" }` (max 5MB).
Response: `{ "passed": true }`.
Requires BVN already verified.

#### POST `/api/v1/identity/products/personal/enroll` ← NEW (fields extended)
Auth: identity token. Device headers ✅. Final onboarding step for Personal — creates the users row (or reuses one seeded by BVN/liveness) and attaches a Personal ProductMembership.
Request (all fields optional; server writes what's supplied through to the users row):
```json
{
  "passcode": "123456",       // optional if already set via /identity/passcode/set
  "firstName": "Aishatu",     // optional; falls back to BVN result / identity
  "surname": "Sadiq",
  "state": "Lagos",
  "lga": "Ikeja",
  "area": "GRA",
  "address": "10 Some Street",
  "zipCode": "100211",        // NEW
  "profession": "Software Engineer",
  "annualIncomeRange": "500000-1000000",  // NEW
  "referralCode": "RIVERLY01" // ignored if signup already captured one
}
```
Response `data`: `{ identityId, personalProfileId, membershipId, role }`.
After this + PIN setup, `personalNextStep` → `Completed`.

#### POST `/api/v1/identity/products/sme/enroll`
Auth: identity token. Body: `{ businessName, businessRegistrationNumber, industry, description, password, websiteUrl?, logoUrl? }`.

#### POST `/api/v1/identity/products/corporate/enroll`
Auth: identity token. Body: `{ organizationName, registrationNumber, password, industry?, country? }`.

### Login + session

#### POST `/api/v1/identity/login`
Device headers ✅. Anonymous (no bearer).
Request: `{ "identifier": "email or phone", "password": "passcode or password" }`.
Response `data`: full auth response.

**New-device gate (Personal only):** if the identity has a Personal membership and the `X-Device-Id` isn't in `UserDevices`, response comes back with `deviceVerificationRequired: true`, null tokens, and dispatches a phone OTP. The FE then calls:

#### POST `/api/v1/identity/device/verify` ← NEW
Device headers ✅. Anonymous.
Request: `{ "identityId": "…", "otpCode": "123456" }`.
Trusts the device and returns full auth response.

#### POST `/api/v1/identity/refresh` ← NEW
Anonymous. Rotates a refresh token.
Request: `{ "refreshToken": "…" }`.
Response `data`: `{ accessToken, refreshToken, identityId }`.

#### POST `/api/v1/identity/switch-product`
Auth: identity token. Trades the identity token for a product-scoped JWT.
Request: `{ "productType": 0, "profileId": null }` (profileId only needed if the identity has multiple memberships of the same product).
Response `data`: `{ accessToken, refreshToken, identityId, productType, profileId, role }`.
This is the token to send on `/personal/*`, `/sme/*`, `/corporate/*` from now on.

#### GET `/api/v1/identity/me`
Auth: identity token. Thin auth-response shape — same fields as login response, includes `personalNextStep`.

#### GET `/api/v1/identity/profile` ← NEW
Auth: identity token. Rich profile: address, KYC tier, transaction limits, account details, biometric flag, product memberships. Use this for profile screens. Response fields:
```
identityId, personalUserId,
firstName, lastName, email, phoneNumber,
isEmailVerified, isPhoneVerified,
state, lga, area, address, zipCode, profession, incomeRange, referralCode,
kycStatus, tier, lastLoginAt, createdAt,
accountNumber, accountName, bankName, currency, accountStatus, isPinSet,
dailyTransactionLimit, singleTransactionLimit,
isBiometricEnabled,
availableProducts[], defaultProduct
```

#### POST `/api/v1/identity/logout`
Auth: identity or product token. Body: `{ "refreshToken": "…" }?`. Idempotent.

### Biometric (Personal only, mirrors legacy `/auth/biometric/*`)

#### POST `/api/v1/identity/biometric/enable` ← NEW
Auth: product-scoped token (Personal). Request: `{ "deviceId": "device-fingerprint" }`. Device must already be trusted.
Response `data`: `{ "deviceId": "…", "biometricToken": "…" }` — store the token in Keychain / Keystore.

#### POST `/api/v1/identity/biometric/disable` ← NEW
Auth: product token. Empty body. Wipes tokens for all devices.

#### GET `/api/v1/identity/biometric/status` ← NEW
Auth: product token. Response `data`: `{ isEnabled, deviceId, enabledAt }`.

#### POST `/api/v1/identity/biometric/login` ← NEW
Device headers ✅. Anonymous.
Request: `{ "deviceId": "device-fingerprint", "biometricToken": "the token from enable" }`.
Response `data`: full identity auth response.

### Device management

#### POST `/api/v1/identity/device/revoke` ← NEW
Auth: identity or product token. Body: `{ "deviceId": "…UserDevices.Id GUID…" }`. Revokes device + all refresh tokens.

### Password reset

Same three-step flow across products; passcode format is server-validated against the identity's product type.

#### POST `/api/v1/identity/passcode/forgot`
Anonymous. `{ "identifier": "email or phone" }`. Always returns 200 (no enumeration).

#### POST `/api/v1/identity/passcode/verify-otp`
Anonymous. `{ "identifier": "…", "otpCode": "123456" }` → `{ resetToken, productType }`.

#### POST `/api/v1/identity/passcode/reset`
Anonymous. `{ "identifier": "…", "resetToken": "…", "newPasscode": "…", "confirmPasscode": "…" }`.

#### POST `/api/v1/identity/passcode/change`
Auth: identity or product token. `{ "currentPasscode": "…", "newPasscode": "…", "confirmPasscode": "…" }`.

### `personalNextStep` values (Personal only)

Matches the legacy `PersonalOnboardingStep` enum so the mobile app can route to the exact screen. Server computes from current identity/user/KYC/account state:

| Value | Meaning | Route to |
|---|---|---|
| `PhoneVerification` | Phone OTP not yet verified | Phone OTP screen |
| `PasscodeCreation` | Phone verified, no passcode set | Set-passcode screen |
| `EmailVerification` | Passcode set, email not verified | Email OTP screen |
| `IdentityVerification` | Email verified, BVN or liveness not passed | BVN entry / liveness screen |
| `ProfileSetup` | BVN + liveness done, no Personal membership yet | Complete profile screen |
| `PinSetup` | Enrolled, no transaction PIN set on account | PIN setup screen |
| `Completed` | Fully onboarded | Dashboard |

Non-Personal identities get `null` here — use `nextStep` (`VerifyPhone`, `SetPasscode`, `EnrollProduct`, `Dashboard`, etc.) for those.

---

## 1. Auth — `/api/v1/auth` (Legacy — Personal only)
> **Deprecated for new Personal features.** Use `/api/v1/identity/*` above.
> Kept live during the mobile-app migration window.
> Login, logout, password reset, biometric login, and user profile.

### POST `/api/v1/auth/login`
Login with passcode. Device headers required.

**Request:**
```json
{
  "identifier": "+2348012345678",   // phone or email — field is "identifier" NOT "phoneNumber"
  "passcode": "123456"
}
```

**Response `data`:**
```json
{
  "accessToken": "eyJ...",
  "refreshToken": "...",
  "userId": "guid",
  "name": "Fatai Sanni",
  "phoneNumber": "+2348012345678",
  "onboardingStep": "completed",
  "deviceVerificationRequired": false,
  "deviceVerificationMessage": null
}
```
> If `deviceVerificationRequired: true`, redirect to verify-device flow before proceeding.

---

### POST `/api/v1/auth/login/new-device`
Login attempt from an unrecognised device — triggers OTP to phone. Device headers required.

**Request:**
```json
{
  "identifier": "+2348012345678"   // phone only
}
```

**Response `data`:**
```json
{
  "userId": "guid",
  "deviceVerificationRequired": true,
  "deviceVerificationMessage": "OTP sent to your phone"
}
```

---

### POST `/api/v1/auth/login/verify-device`
Verify the OTP sent during new-device login. Device headers required.

**Request:**
```json
{
  "userId": "guid",
  "otpCode": "123456"
}
```

---

### POST `/api/v1/auth/refresh`
Get a new access token using a refresh token.

**Request:**
```json
{
  "refreshToken": "..."
}
```

**Response `data`:**
```json
{
  "accessToken": "eyJ...",
  "refreshToken": "..."
}
```

---

### POST `/api/v1/auth/logout`
🔒 **Requires auth.** Revokes refresh token.

**Request:**
```json
{
  "refreshToken": "..."
}
```

---

### POST `/api/v1/auth/passcode/forgot`
Initiate passcode reset — sends OTP to phone/email.

**Request:**
```json
{
  "identifier": "+2348012345678"   // phone or email
}
```

---

### POST `/api/v1/auth/passcode/verify-otp`
Verify OTP from forgot-passcode flow — returns a reset token.

**Request:**
```json
{
  "identifier": "+2348012345678",
  "otpCode": "123456"
}
```

**Response `data`:**
```json
{
  "resetToken": "..."
}
```

---

### POST `/api/v1/auth/passcode/reset`
Reset passcode using the token from verify-otp.

**Request:**
```json
{
  "identifier": "+2348012345678",
  "resetToken": "...",
  "newPasscode": "654321",         // exactly 6 digits
  "confirmNewPasscode": "654321"
}
```

---

### POST `/api/v1/auth/passcode/change`
🔒 **Requires auth.** Change passcode when already logged in.

**Request:**
```json
{
  "currentPasscode": "123456",
  "newPasscode": "654321",         // exactly 6 digits
  "confirmNewPasscode": "654321"
}
```

---

### GET `/api/v1/auth/user/profile`
Get user profile. No auth required on the route but needs a valid JWT to return user data.

**Response `data`:**
```json
{
  "userId": "guid",
  "firstName": "Fatai",
  "lastName": "Sanni",
  "email": "fatai@example.com",
  "phoneNumber": "+2348012345678",
  "isEmailVerified": true,
  "isPhoneVerified": true,
  "state": "Lagos",
  "lga": "Ikeja",
  "address": "...",
  "profession": "...",
  "incomeRange": "...",
  "kycStatus": "...",
  "status": "...",
  "lastLoginAt": "2026-04-09T...",
  "createdAt": "2026-04-01T...",
  "accountNumber": "...",
  "accountName": "...",
  "bankName": "...",
  "currency": "NGN",
  "tier": "...",
  "dailyTransactionLimit": 100000,
  "singleTransactionLimit": 50000
}
```

---

### POST `/api/v1/auth/biometric/enable`
🔒 **Requires auth.** Register device for biometric login.

> The backend never sees the user's face or fingerprint. What happens: the user does Face ID / fingerprint on their phone → if it passes, you call this endpoint → backend gives you a `biometricToken` → save it securely (Keychain on iOS, Keystore on Android). That token is what you send later to log in.

**Request:**
```json
{
  "deviceId": "uuid-of-device"        // use the SAME X-Device-Id you send in headers
}
```

**Response `data`:**
```json
{
  "deviceId": "uuid-of-device",       // same value you sent — confirmation only
  "biometricToken": "..."             // store this securely in Keychain / Keystore
}
```

---

### POST `/api/v1/auth/biometric/login`
Login using stored biometric token. Device headers required.

**Request:**
```json
{
  "deviceId": "uuid-of-device",       // same X-Device-Id used during enable
  "biometricToken": "..."             // the token stored from enable response
}
```

**Response `data`:**
```json
{
  "accessToken": "eyJ...",
  "refreshToken": "...",
  "userId": "guid",
  "name": "Fatai Sanni",
  "phoneNumber": "+2348012345678",
  "onboardingStep": "completed"
}
```

---

### GET `/api/v1/auth/biometric/status`
🔒 **Requires auth.** Check if biometric is enabled for the current user.

### POST `/api/v1/auth/biometric/disable`
🔒 **Requires auth.** Revoke biometric for current user.

---

### Biometric Flow Guide (for frontend)

**Step A — Enable biometric (Settings screen, after user is logged in):**
```
1. User taps "Enable Face ID" or "Enable Fingerprint" in app settings
2. Trigger native biometric prompt (FaceID / TouchID / Android BiometricPrompt)
3. If native prompt succeeds:
     POST /api/v1/auth/biometric/enable
     Headers: Authorization: Bearer <accessToken>
     Body:    { "deviceId": "<your X-Device-Id>" }
4. Store the response biometricToken securely:
     - iOS: Keychain
     - Android: EncryptedSharedPreferences or Keystore
5. Also store a flag locally: biometricEnabled = true
```

**Step B — Biometric login (app launch, when biometric is enabled):**
```
1. Check local storage: is biometricEnabled == true and biometricToken exists?
2. If yes, trigger native biometric prompt (FaceID / TouchID / fingerprint)
3. If native prompt succeeds:
     POST /api/v1/auth/biometric/login
     Headers: X-Device-Id: <your device UUID>
     Body:    { "deviceId": "<same X-Device-Id>", "biometricToken": "<stored token>" }
4. Response contains accessToken + refreshToken — user is logged in
```

**Edge cases to handle:**
| Scenario | What happens | Frontend action |
|----------|-------------|-----------------|
| Token expired (90 days) | `"Biometric session expired"` | Clear stored token, show passcode login |
| App reinstalled (new X-Device-Id) | Old token is orphaned | User must log in with passcode, re-enable biometric |
| User disabled biometric | `/biometric/login` returns error | Clear stored token, show passcode login |
| Native prompt fails/cancelled | No API call needed | Show passcode login as fallback |

**Important:** The `deviceId` in the request body must be the **exact same string** as your `X-Device-Id` header. Generate it once on first install and persist it.

### POST `/api/v1/auth/device/revoke`
🔒 **Requires auth.** Revoke a specific device session.

**Request:**
```json
{
  "deviceId": "guid"
}
```

---

## 2. Onboarding — `/api/v1/onboarding`

> New user signup — 8 steps in order. The user must complete each step before moving to the next.
> **Step order:** Phone → Verify Phone → Email → Verify Email → BVN/NIN → Profile → Passcode → Selfie

### POST `/api/v1/onboarding/phone/initiate`
Step 1 — Start signup with phone number. Sends OTP.

**Request:**
```json
{
  "phoneNumber": "+2348012345678",
  "referralCode": "ABC123"          // optional
}
```

---

### POST `/api/v1/onboarding/phone/verify`
Step 2 — Verify phone OTP. Device headers required. Returns a JWT for subsequent onboarding steps.

**Request:**
```json
{
  "target": "+2348012345678",       // field is "target", NOT "phoneNumber"
  "code": "123456"                  // field is "code", NOT "otp"
}
```

**Response `data`:**
```json
{
  "accessToken": "eyJ...",
  "userId": "guid",
  "onboardingStep": "email"
}
```

---

### POST `/api/v1/onboarding/email/submit`
Step 3 — Submit email address. 🔒 Requires onboarding JWT.

**Request:**
```json
{
  "email": "user@example.com"
}
```

---

### POST `/api/v1/onboarding/email/verify`
Step 4 — Verify email OTP. 🔒 Requires auth. Device headers required.

**Request:**
```json
{
  "target": "user@example.com",     // field is "target"
  "code": "123456"                  // field is "code"
}
```

---

### POST `/api/v1/onboarding/resend-otp`
Resend OTP for phone or email.

**Request:**
```json
{
  "identifier": "+2348012345678"    // phone or email
}
```

---

### POST `/api/v1/onboarding/identity/verify`
Step 5 — BVN or NIN verification. 🔒 Requires auth.

> Send at least one (BVN or NIN). You can send both. The backend verifies them automatically.

**Request:**
```json
{
  "bvn": "12345678901",            // provide one or both
  "nin": "12345678901"
}
```

---

### POST `/api/v1/onboarding/profile/update`
Step 6 — Submit residential and professional details. 🔒 Requires auth.

**Request:**
```json
{
  "state": "Lagos",
  "lga": "Ikeja",
  "area": "GRA",
  "address": "14 Street Name",
  "zipCode": "100001",
  "profession": "Software Engineer",
  "annualIncomeRange": "1000000-5000000"   // field is "annualIncomeRange"
}
```

---

### POST `/api/v1/onboarding/passcode/set`
Step 7 — Set login passcode. 🔒 Requires auth.

**Request:**
```json
{
  "passcode": "123456",            // exactly 6 digits
  "confirmPasscode": "123456"
}
```

---

### POST `/api/v1/onboarding/liveness/check`
Step 8 — Selfie verification. 🔒 Requires auth.

> Open the camera, take a selfie, convert it to a base64 string, and send it. The backend checks if it's a real person (not a photo of a photo). This only happens once during signup — it's **not** the same as Face ID login.

**Request:**
```json
{
  "image": "base64encodedstring..."    // base64-encoded selfie from camera (not a stored biometric)
}
```

---

## 3. Accounts — `/api/v1/accounts`
> Create accounts, check balances, manage transaction PIN.

🔒 All endpoints require auth.

### POST `/api/v1/accounts`
Create a bank account for the user.

**Request:**
```json
{
  "accountProductCode": "12",
  "branchCode": "101",
  "accountOfficerCode": "10",
  "currency": "NGN",
  "oldAccountNo": null             // optional
}
```

---

### GET `/api/v1/accounts`
Get all user accounts with balances.

**Response `data`:** Array of account objects with balances.

---

### GET `/api/v1/accounts/{accountId}/balance`
Get balance for a single account.

---

### POST `/api/v1/accounts/set-pin`
Set a 4-digit transaction PIN.

**Request:**
```json
{
  "pin": "1234",                   // exactly 4 digits
  "confirmPin": "1234"
}
```

---

### POST `/api/v1/accounts/change-pin`
Change transaction PIN.

**Request:**
```json
{
  "currentPin": "1234",
  "newPin": "5678",
  "confirmNewPin": "5678"
}
```

---

### POST `/api/v1/accounts/forgot-pin`
Reset PIN using passcode.

**Request:**
```json
{
  "passcode": "123456",
  "newPin": "5678",
  "confirmNewPin": "5678"
}
```

---

### POST `/api/v1/accounts/{accountId}/verify`
Verify transaction PIN before a transaction.

**Request:**
```json
{
  "pin": "1234"
}
```

---

## 4. Transactions — `/api/v1/accounts/{accountId}/transactions`
> View transaction history with filters, pagination, and detail view.

🔒 All endpoints require auth.

### GET `/api/v1/accounts/{accountId}/transactions`
Get paginated transaction history. All filters are query params.

**Query params (all optional):**
```
?page=1
&pageSize=20
&dateFrom=2026-01-01
&dateTo=2026-04-01
&type=debit                        // TransactionType enum
&status=successful
&direction=outbound
&category=transfer
&amountMin=1000
&amountMax=50000
&reference=TXN12345
&search=john                       // searches narration + counterparty name
```

**Response `data`:**
```json
{
  "items": [],
  "page": 1,
  "pageSize": 20,
  "totalCount": 0,
  "totalPages": 0,
  "hasNextPage": false,
  "hasPreviousPage": false
}
```

---

### GET `/api/v1/accounts/{accountId}/transactions/{transactionId}`
Get single transaction detail / receipt.

### GET `/api/v1/transactions/{reference}`
Look up any transaction by reference string.

---

## 5. Transfers — `/api/v1/transfers`
> Send money to another Riverly account.

🔒 All endpoints require auth.

### GET `/api/v1/transfers/internal-lookup/{accountNumber}`
Look up a Riverly account before transferring.

**Response `data`:** Account name and number if found.

---

### POST `/api/v1/transfers/internal`
Transfer funds between Riverly accounts.

**Request:**
```json
{
  "sourceAccountId": "guid",
  "destinationAccountNumber": "0123456789",
  "amount": 5000.00,
  "narration": "Rent payment",         // optional, max 100 chars
  "transactionPin": "1234",
  "idempotencyKey": "uuid-v4"          // generate a new UUID per transfer attempt
}
```

---

### GET `/api/v1/transfers/receipt/{reference}`
Get transfer receipt by reference.

---

## 6. Beneficiaries — `/api/v1/beneficiaries`
> Saved recipients — so users don't re-enter account details every time.

🔒 All endpoints require auth.

### GET `/api/v1/beneficiaries/banks`
Get list of all supported banks (name + bank code).

### POST `/api/v1/beneficiaries/validate`
Validate an external bank account before saving/transferring.

**Request:**
```json
{
  "accountNumber": "0123456789",
  "bankCode": "058"                // from /banks list
}
```

### GET `/api/v1/beneficiaries`
List all saved beneficiaries.

**Query params (all optional):**
```
?search=john
&type=internal                     // BeneficiaryType enum: internal | external
&isFavorite=true
&page=1
&pageSize=20
```

### GET `/api/v1/beneficiaries/{beneficiaryId}`
Get single beneficiary.

### POST `/api/v1/beneficiaries`
Save a new beneficiary.

**Request:**
```json
{
  "accountNumber": "0123456789",
  "bankCode": "058",
  "nickname": "Ahmed GTB",         // optional
  "isFavorite": false
}
```

### PATCH `/api/v1/beneficiaries/{beneficiaryId}`
Update nickname or favourite status.

**Request:**
```json
{
  "nickname": "Ahmed",             // optional
  "isFavorite": true               // optional
}
```

### DELETE `/api/v1/beneficiaries/{beneficiaryId}`
Remove a beneficiary (soft delete).

---

## 7. Bill Payments — `/api/v1/payments`
> Buy airtime, data bundles, pay electricity, cable TV, and other bills.

🔒 All endpoints require auth.

### GET `/api/v1/payments/data/plans/{networkProvider}`
Get data plans for a network. `networkProvider` = `mtn` | `glo` | `airtel` | `9mobile`

### GET `/api/v1/payments/bills/categories`
Get all biller categories (electricity, cable TV, etc.).

### GET `/api/v1/payments/bills/{category}`
Get all billers within a category.

### POST `/api/v1/payments/bills/validate`
Validate a customer reference (meter number, smartcard, etc.) before payment.

**Request:**
```json
{
  "billerId": "IKEDC",
  "customerReference": "45062000001"
}
```

### POST `/api/v1/payments/airtime`
Purchase airtime.

**Request:**
```json
{
  "accountId": "guid",
  "phoneNumber": "+2348012345678",
  "networkProvider": "mtn",         // mtn | glo | airtel | 9mobile
  "amount": 1000,                   // min 50, max 500000
  "transactionPin": "1234",
  "idempotencyKey": "uuid-v4"
}
```

### POST `/api/v1/payments/data`
Purchase data bundle.

**Request:**
```json
{
  "accountId": "guid",
  "phoneNumber": "+2348012345678",
  "networkProvider": "mtn",
  "planId": "provider-specific-id",  // from data plans endpoint
  "amount": 2000,
  "transactionPin": "1234",
  "idempotencyKey": "uuid-v4"
}
```

### POST `/api/v1/payments/bills/pay`
Pay a bill.

**Request:**
```json
{
  "accountId": "guid",
  "billerId": "IKEDC",
  "customerReference": "45062000001",
  "amount": 5000,
  "transactionPin": "1234",
  "idempotencyKey": "uuid-v4",
  "billerPlanId": null              // optional, for cable TV bouquets
}
```

### GET `/api/v1/payments/airtime/{reference}/receipt`
### GET `/api/v1/payments/data/{reference}/receipt`
### GET `/api/v1/payments/bills/{reference}/receipt`
Get receipts by transaction reference.

---

## 8. KYC — `/api/v1/kyc`
> Account upgrade — user uploads identity documents for verification.

🔒 All endpoints require auth.

### POST `/api/v1/kyc/documents/upload`
Upload a KYC document. **Must be `multipart/form-data`**, max 5 MB.

**Form fields:**
```
File           → the document file
DocumentType   → enum value (NIN slip, passport, driver's license, etc.)
Side           → enum value (front/back)
```

### POST `/api/v1/kyc/submit`
Submit KYC for review after documents are uploaded.

### GET `/api/v1/kyc/status`
Get current KYC verification status and tier.

---

## 9. Dashboard — `/api/v1/dashboard`
🔒 Requires auth.

### GET `/api/v1/dashboard`
Main dashboard data — accounts, balances, recent transactions, KYC status. Response is cached for **30 seconds per user**.

---

## 10. Statements — `/api/v1/statements`
🔒 All endpoints require auth.

### POST `/api/v1/statements/request`
Request a bank statement (async — check status after).

**Request:**
```json
{
  "accountId": "guid",
  "dateFrom": "2026-01-01T00:00:00Z",
  "dateTo": "2026-04-01T00:00:00Z",
  "delivery": "Download"            // StatementDelivery enum: Download | Email
}
```

### GET `/api/v1/statements/{requestId}/status`
Check statement status and get download URL when ready.

---

## 11. Notifications — `/api/v1/notifications`
🔒 All endpoints require auth.

### POST `/api/v1/notifications/push-token`
Register device push token (call after login on mobile).

**Request:**
```json
{
  "deviceId": "uuid",
  "token": "firebase-or-apns-token",
  "platform": "ios"                 // ios | android | web
}
```

### DELETE `/api/v1/notifications/push-token/{deviceId}`
Deregister push token on logout.

### GET `/api/v1/notifications/preferences`
Get notification preferences.

### PATCH `/api/v1/notifications/preferences`
Update notification preferences (all fields optional — partial update).

**Request:**
```json
{
  "pushEnabled": true,
  "emailEnabled": true,
  "smsEnabled": false,
  "transferAlerts": true,
  "billPaymentAlerts": true,
  "cardAlerts": true,
  "loanAlerts": false,
  "securityAlerts": true,
  "promotionalAlerts": false
}
```

---

## 12. Chat — `/api/v1/chat`
🔒 All endpoints require auth.

### POST `/api/v1/chat/sessions`
Start a new support chat session. Returns **HTTP 201**.

**Request:**
```json
{
  "subject": "Issue with transfer"   // optional
}
```

### POST `/api/v1/chat/messages`
Send a message in an existing session.

**Request:**
```json
{
  "sessionId": "guid",
  "content": "Hello, I need help..."   // max 2000 chars
}
```

### GET `/api/v1/chat/sessions`
Get all chat sessions for the user.

### GET `/api/v1/chat/sessions/{sessionId}/messages`
Get all messages in a session.

### POST `/api/v1/chat/sessions/{sessionId}/close`
Close a chat session.

---

## 13. Support — `/api/v1/support`
🔒 Requires auth.

### GET `/api/v1/support/contact`
Get customer support contact options (phone, email, etc.).

---

## 14. Frontend Flow Guides — Screen-by-Screen

> **Start here if you want to know "what do I call and when?"**
> Each flow shows the exact screens and API calls in order, from when the user taps something to when the action is complete.

---

### Flow: Onboarding (new user signup)
```
Screen: Welcome → "Sign Up"
  │
  ├─ 1. Phone number input screen
  │     User enters phone → POST /api/v1/onboarding/phone/initiate
  │     Body: { "phoneNumber": "+234...", "referralCode": "ABC123" }  ← referralCode optional
  │
  ├─ 2. OTP verification screen
  │     User enters 6-digit code → POST /api/v1/onboarding/phone/verify
  │     Body: { "target": "+234...", "code": "123456" }
  │     ✅ Response gives you an accessToken — use it for ALL remaining steps
  │     ✅ Response gives onboardingStep — tells you which screen to show next
  │     ⚠️  If OTP expired: user taps "Resend" → POST /api/v1/onboarding/resend-otp
  │
  ├─ 3. Email input screen
  │     User enters email → POST /api/v1/onboarding/email/submit
  │     Body: { "email": "user@example.com" }
  │
  ├─ 4. Email OTP verification screen
  │     User enters code → POST /api/v1/onboarding/email/verify
  │     Body: { "target": "user@example.com", "code": "123456" }
  │
  ├─ 5. Identity verification screen (BVN/NIN)
  │     User enters BVN and/or NIN → POST /api/v1/onboarding/identity/verify
  │     Body: { "bvn": "12345678901", "nin": "12345678901" }
  │     ⚠️  At least one is required. Backend verifies via Dojah.
  │
  ├─ 6. Profile details screen
  │     User fills form → POST /api/v1/onboarding/profile/update
  │     Body: { "state", "lga", "area", "address", "zipCode", "profession", "annualIncomeRange" }
  │
  ├─ 7. Set passcode screen
  │     User enters 6-digit passcode twice → POST /api/v1/onboarding/passcode/set
  │     Body: { "passcode": "123456", "confirmPasscode": "123456" }
  │
  └─ 8. Liveness check screen (selfie)
        Open camera → capture selfie → convert to base64
        POST /api/v1/onboarding/liveness/check
        Body: { "image": "base64string..." }
        ✅ Onboarding complete → navigate to Dashboard
```

---

### Flow: Login (existing user)
```
Screen: Welcome → "Log In"
  │
  ├─ 1. Login screen
  │     User enters phone/email + passcode
  │     POST /api/v1/auth/login
  │     Headers: X-Device-Id required
  │     Body: { "identifier": "+234...", "passcode": "123456" }
  │
  ├─ Response A: deviceVerificationRequired == false
  │     ✅ Login success → save accessToken + refreshToken → go to Dashboard
  │
  └─ Response B: deviceVerificationRequired == true (new/unrecognised device)
        │
        ├─ 2. Backend already sent OTP to user's phone
        │     Show OTP input screen
        │     POST /api/v1/auth/login/verify-device
        │     Body: { "userId": "<from response>", "otpCode": "123456" }
        │
        └─ ✅ Device verified → save tokens → go to Dashboard
```

---

### Flow: Biometric Login (app launch)
```
App opens → check local storage
  │
  ├─ biometricEnabled == true AND biometricToken exists?
  │     YES → show biometric prompt (Face ID / fingerprint)
  │       │
  │       ├─ Native prompt succeeds:
  │       │     POST /api/v1/auth/biometric/login
  │       │     Headers: X-Device-Id
  │       │     Body: { "deviceId": "<X-Device-Id>", "biometricToken": "<stored>" }
  │       │     ✅ Save new tokens → go to Dashboard
  │       │
  │       └─ Native prompt fails/cancelled:
  │             Show passcode login screen (fallback)
  │
  └─ NO → show normal passcode login screen
```

---

### Flow: Buy Airtime
```
Dashboard → "Pay Bills" or "Airtime" category
  │
  ├─ 1. Airtime form screen
  │     Show form: phone number, network provider dropdown, amount input
  │     Network options: mtn | glo | airtel | 9mobile  ← hardcoded in app
  │     ⚠️  No API call needed yet — this is just a form
  │
  ├─ 2. User fills form → taps "Continue"
  │     Show confirmation screen with summary:
  │       Phone: +234...
  │       Network: MTN
  │       Amount: ₦1,000
  │       Source account: from GET /api/v1/accounts (pick default or let user choose)
  │
  ├─ 3. User confirms → show PIN input
  │     User enters 4-digit transaction PIN
  │
  ├─ 4. Submit purchase
  │     POST /api/v1/payments/airtime
  │     Body: {
  │       "accountId": "<selected account guid>",
  │       "phoneNumber": "+234...",
  │       "networkProvider": "mtn",
  │       "amount": 1000,
  │       "transactionPin": "1234",
  │       "idempotencyKey": "<generate new UUID>"
  │     }
  │
  └─ 5. Receipt screen
        ✅ On success → show receipt from response
        OR fetch: GET /api/v1/payments/airtime/{reference}/receipt
```

---

### Flow: Buy Data
```
Dashboard → "Pay Bills" or "Data" category
  │
  ├─ 1. Select network screen
  │     User picks: mtn | glo | airtel | 9mobile
  │
  ├─ 2. Fetch available plans
  │     GET /api/v1/payments/data/plans/{networkProvider}
  │     Example: GET /api/v1/payments/data/plans/mtn
  │     ✅ Response: list of plans with planId, name, amount
  │     Show plans as selectable cards/list
  │
  ├─ 3. User picks a plan + enters phone number → taps "Continue"
  │     Show confirmation screen:
  │       Phone: +234...
  │       Plan: MTN 1GB - 30 days
  │       Amount: ₦2,000
  │       Source account: from GET /api/v1/accounts
  │
  ├─ 4. User confirms → show PIN input
  │     User enters 4-digit transaction PIN
  │
  ├─ 5. Submit purchase
  │     POST /api/v1/payments/data
  │     Body: {
  │       "accountId": "<guid>",
  │       "phoneNumber": "+234...",
  │       "networkProvider": "mtn",
  │       "planId": "<from plans list>",
  │       "amount": 2000,
  │       "transactionPin": "1234",
  │       "idempotencyKey": "<new UUID>"
  │     }
  │
  └─ 6. Receipt screen
        GET /api/v1/payments/data/{reference}/receipt
```

---

### Flow: Pay Bill (Electricity, Cable TV, etc.)
```
Dashboard → "Pay Bills"
  │
  ├─ 1. Show bill categories
  │     GET /api/v1/payments/bills/categories
  │     ✅ Response: list of categories (electricity, cable TV, internet, etc.)
  │     Show as grid or list
  │
  ├─ 2. User taps a category (e.g. "Electricity")
  │     GET /api/v1/payments/bills/electricity   ← the category slug
  │     ✅ Response: list of billers (IKEDC, EKEDC, AEDC, etc.)
  │     Show as selectable list
  │
  ├─ 3. User picks a biller → enters customer reference
  │     (meter number, smartcard number, etc.)
  │     Tap "Validate" →
  │     POST /api/v1/payments/bills/validate
  │     Body: { "billerId": "IKEDC", "customerReference": "45062000001" }
  │     ✅ Response: customer name + validated details
  │     Show: "Customer: JOHN DOE — Meter: 45062000001"
  │     ⚠️  If validation fails, show error — don't proceed
  │
  ├─ 4. User enters amount (+ selects bouquet for cable TV if applicable)
  │     Show confirmation:
  │       Biller: IKEDC
  │       Customer: JOHN DOE
  │       Amount: ₦5,000
  │       Source account: from GET /api/v1/accounts
  │
  ├─ 5. User confirms → PIN input → submit
  │     POST /api/v1/payments/bills/pay
  │     Body: {
  │       "accountId": "<guid>",
  │       "billerId": "IKEDC",
  │       "customerReference": "45062000001",
  │       "amount": 5000,
  │       "transactionPin": "1234",
  │       "idempotencyKey": "<new UUID>",
  │       "billerPlanId": null           ← only for cable TV bouquets
  │     }
  │
  └─ 6. Receipt screen
        GET /api/v1/payments/bills/{reference}/receipt
```

---

### Flow: Transfer to Riverly Account (Internal)
```
Dashboard → "Transfer" → "To Riverly Account"
  │
  ├─ 1. Enter recipient account number
  │     User types account number →
  │     GET /api/v1/transfers/internal-lookup/{accountNumber}
  │     ✅ Response: account name + number
  │     Show: "FATAI SANNI — 0123456789"
  │     ⚠️  If not found, show "Account not found"
  │
  ├─ 2. User enters amount + narration → taps "Continue"
  │     Show confirmation:
  │       To: FATAI SANNI — 0123456789
  │       Amount: ₦5,000
  │       Narration: Rent payment
  │       From: your account (from GET /api/v1/accounts)
  │
  ├─ 3. User confirms → PIN input → submit
  │     POST /api/v1/transfers/internal
  │     Body: {
  │       "sourceAccountId": "<guid>",
  │       "destinationAccountNumber": "0123456789",
  │       "amount": 5000.00,
  │       "narration": "Rent payment",
  │       "transactionPin": "1234",
  │       "idempotencyKey": "<new UUID>"
  │     }
  │
  ├─ 4. Receipt screen
  │     GET /api/v1/transfers/receipt/{reference}
  │
  └─ 5. Optional: "Save as beneficiary?" prompt
        POST /api/v1/beneficiaries
        Body: { "accountNumber": "0123456789", "bankCode": "riverly", "nickname": "Fatai" }
```

---

### Flow: View Transaction History
```
Dashboard → tap account card or "Transactions"
  │
  ├─ 1. Load transactions
  │     GET /api/v1/accounts/{accountId}/transactions?page=1&pageSize=20
  │     ✅ Show list of transactions (paginated)
  │
  ├─ 2. User scrolls down → load more
  │     GET /api/v1/accounts/{accountId}/transactions?page=2&pageSize=20
  │     Append to list (infinite scroll)
  │
  ├─ 3. User taps filter icon → filter options
  │     Add query params: ?type=debit&dateFrom=2026-01-01&dateTo=2026-04-01
  │
  └─ 4. User taps a transaction → detail screen
        GET /api/v1/accounts/{accountId}/transactions/{transactionId}
        Show full receipt / detail view
```

---

### Flow: Forgot Passcode
```
Login screen → "Forgot Passcode?"
  │
  ├─ 1. Enter phone/email
  │     POST /api/v1/auth/passcode/forgot
  │     Body: { "identifier": "+234..." }
  │
  ├─ 2. OTP input screen
  │     POST /api/v1/auth/passcode/verify-otp
  │     Body: { "identifier": "+234...", "otpCode": "123456" }
  │     ✅ Response: { "resetToken": "..." }
  │
  └─ 3. New passcode screen
        User enters new 6-digit passcode twice
        POST /api/v1/auth/passcode/reset
        Body: {
          "identifier": "+234...",
          "resetToken": "<from step 2>",
          "newPasscode": "654321",
          "confirmNewPasscode": "654321"
        }
        ✅ Success → navigate to login screen
```

---

### Flow: KYC Document Upload (post-onboarding tier upgrade)
```
Settings → "Upgrade Account" or Dashboard KYC banner
  │
  ├─ 1. Check current KYC status
  │     GET /api/v1/kyc/status
  │     Shows current tier + what's needed for upgrade
  │
  ├─ 2. Document upload screen (can upload multiple)
  │     For each document:
  │       POST /api/v1/kyc/documents/upload
  │       Content-Type: multipart/form-data
  │       Fields: File, DocumentType, Side (front/back)
  │       ⚠️  Max 5 MB per file
  │
  └─ 3. Submit for review
        POST /api/v1/kyc/submit
        ✅ KYC is now "pending review" — check status later with GET /api/v1/kyc/status
```

---

### Flow: Dashboard (home screen)
```
After login / app launch (authenticated)
  │
  ├─ 1. Load dashboard
  │     GET /api/v1/dashboard
  │     ✅ Returns: accounts, balances, recent transactions, KYC status
  │     ⚠️  Cached 30 seconds per user — safe to call on every app foreground
  │
  ├─ 2. Quick actions from dashboard:
  │     "Transfer" → Transfer flow
  │     "Airtime"  → Airtime flow
  │     "Data"     → Data flow
  │     "Pay Bill" → Bill payment flow
  │
  └─ 3. Pull to refresh → call GET /api/v1/dashboard again
```

---

### Flow: Request Bank Statement
```
Settings → "Statements" or Account → "Statement"
  │
  ├─ 1. Statement request form
  │     User picks: account, date range, delivery method (Download or Email)
  │     POST /api/v1/statements/request
  │     Body: {
  │       "accountId": "<guid>",
  │       "dateFrom": "2026-01-01T00:00:00Z",
  │       "dateTo": "2026-04-01T00:00:00Z",
  │       "delivery": "Download"
  │     }
  │     ✅ Response: { "requestId": "..." }
  │
  └─ 2. Check status / download
        GET /api/v1/statements/{requestId}/status
        ⚠️  Statement is generated async — poll every few seconds or show "Processing..."
        When ready: response includes download URL
```

---

### Flow: Manage Beneficiaries
```
Transfer screen → "Beneficiaries" or Settings → "Saved Beneficiaries"
  │
  ├─ 1. List beneficiaries
  │     GET /api/v1/beneficiaries?page=1&pageSize=20
  │     Optional filters: ?search=ahmed&type=internal&isFavorite=true
  │
  ├─ 2. Add new beneficiary
  │     a. Get bank list → GET /api/v1/beneficiaries/banks
  │     b. User enters account number + picks bank
  │     c. Validate → POST /api/v1/beneficiaries/validate
  │        Body: { "accountNumber": "0123456789", "bankCode": "058" }
  │        ✅ Shows account name
  │     d. Save → POST /api/v1/beneficiaries
  │        Body: { "accountNumber": "...", "bankCode": "058", "nickname": "Ahmed GTB" }
  │
  ├─ 3. Edit beneficiary
  │     PATCH /api/v1/beneficiaries/{beneficiaryId}
  │     Body: { "nickname": "New name", "isFavorite": true }
  │
  └─ 4. Delete beneficiary
        DELETE /api/v1/beneficiaries/{beneficiaryId}
```

---

## Quick Reference — Field Names to Watch

| Common mistake | Correct field name |
|---------------|-------------------|
| `phoneNumber` on login | `identifier` |
| `otp` / `otpCode` | `code` (on /onboarding/phone/verify and /email/verify) |
| `phoneNumber` on verify | `target` |
| `token` on login response | `accessToken` |
| `pin` as 6 digits | PIN is `4 digits`, passcode is `6 digits` |
| `amount` as string | `amount` must be a **number** (decimal) |
| Missing `idempotencyKey` | Required on transfers, airtime, data, and bill payments |
| Missing `X-Device-Id` header | Required on login, new-device login, phone verify, email verify, biometric login |
