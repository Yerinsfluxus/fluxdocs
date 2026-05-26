# SME App — Backend API Reference

**Status:** Living document. Updated every time a new endpoint ships.
**Audience:** Frontend / mobile engineers building the SME app.
**Base URL (staging):** `https://api.riverly.ng`
**Last refreshed:** 2026-05-26 — **Onboarding rebuilt to match Femi's BE-102 → BE-109 tickets.**

> **READ ME FIRST — the onboarding flow has changed.** The single-step `/api/v1/sme/enroll` payload (and the old `/api/v1/identity/signup` that took `fullName + password`) are gone. Onboarding is now a 7-step flow. See **Section 0 — Onboarding flow (BE-102 → BE-109)** below. The legacy endpoints documented further down still exist for everything *post-onboarding* (dashboard, transfers, settings, etc.).

> **Firebase / FCM status:** push device registration (`/api/v1/sme/push/devices`) still works — it stores tokens. **Actual push delivery is pending** while ops finalises the Firebase project + service-account credentials. The same applies to SendGrid (welcome email) and Twilio (welcome SMS). OTPs and welcome messages are stubbed to `_logger.LogInformation` — they appear in the staging logs so QA can pull codes during testing.

---

## 0. Onboarding flow (BE-102 → BE-109)

The SME app walks a new user through 7 screens. Each step has a dedicated endpoint; the user receives an **identity token** at sign-up (BE-102) and uses it for every subsequent onboarding call. After the password step (BE-109) the same token is reused — there is no separate "temp session" exchange.

| Screen | Ticket | Endpoint | Auth | Notes |
|---|---|---|---|---|
| 1. Sign-up | BE-102 | `POST /api/v1/identity/signup` | none | Returns identity token + triggers phone & email OTPs in the background |
| 2. Phone OTP | BE-103 | `POST /api/v1/identity/phone/verify` | identity | Skip allowed → `POST /api/v1/identity/phone/skip`. Resend → `POST /api/v1/identity/phone/resend` |
| 3. Email OTP | BE-104 | `POST /api/v1/identity/email/verify` | identity | **No skip.** Resend → `POST /api/v1/identity/email/resend`. Error copy masks the email |
| 4. Business profile | BE-106 | `POST /api/v1/sme/onboarding/business-profile` | identity | Picks industry from `/api/v1/reference/industries` (BE-105). Logo upload is a separate call: `POST /api/v1/sme/onboarding/logo` (multipart) |
| 5. Owner role | BE-107 | `POST /api/v1/sme/onboarding/owner-role` | identity | role must be `"Business Owner"`; ownership 1–100 |
| 6. Address | BE-108 | `POST /api/v1/sme/onboarding/address` | identity | When `isRegisteredAddressSameAsOperating=false` the registered_* fields become required. State picker uses `/api/v1/reference/states` |
| 7. Password | BE-109 | `POST /api/v1/sme/onboarding/password` | identity | Server enforces 8+ chars / upper / lower / digit / special. Sets `onboardingCompletedAt`. **Now eligible to call `/api/v1/identity/switch-product` and land on the dashboard.** |

KYB documents (CAC certificate, status report, ID, BVN, selfie) are uploaded **after** onboarding from the dashboard, gated by a banner — not in the onboarding flow itself.

### Sign-up payload (BE-102)

```json
{
  "firstName": "Jane",
  "surname":   "Doe",
  "phoneNumber": "+2348012345678",   // accepts +234XXXXXXXXXX or 0XXXXXXXXXX
  "email":     "jane@example.com"
}
```

Response `data` includes `accessToken` (identity, 7-day), plus `emailVerified`, `phoneVerified`, `onboardingCompleted` booleans the FE can use to route to the right screen.

### OTP verify payloads (BE-103 / BE-104)

```json
{ "code": "123456" }
```

Validation rules (in `OtpService`): 6 digits, 10-minute expiry, 5 attempts per OTP, 3 resends per 10 minutes. These follow the existing OTP infrastructure — adding stricter constraints isn't necessary right now per product decision.

### Business profile (BE-106)

```json
{
  "businessName": "Acme Ventures",
  "businessRegistrationNumber": "BN1234567",
  "registrationType": "Business Name Only",
  "description": "We sell hand-poured candles and home fragrances to retail and gift markets across Nigeria.",
  "industry": "fashion-apparel",     // slug from /reference/industries
  "websiteUrl": "https://acme.ng",   // optional, must start with https://
  "logoUrl": null                    // optional, populate with URL from logo upload
}
```

### Logo upload (BE-106 comment)

`POST /api/v1/sme/onboarding/logo` — `multipart/form-data` with one field `file`. Server validates ≤ 2 MB, PNG/JPG/SVG only, magic-byte safety check. Returns `{ logoUrl: "https://..." }`. FE then sends `logoUrl` on the next business-profile PATCH (or on subsequent enrollment).

### Owner role (BE-107)

```json
{ "role": "Business Owner", "ownershipPercentage": 100 }
```

### Address (BE-108)

```json
{
  "operatingAddressLine": "12 Adeola Odeku Street",
  "operatingCity": "Lagos",
  "operatingState": "Lagos",
  "operatingPostalCode": "100001",
  "isRegisteredAddressSameAsOperating": true,
  "registeredAddressLine": null,
  "registeredCity": null,
  "registeredState": null,
  "registeredPostalCode": null
}
```

If `isRegisteredAddressSameAsOperating` is `false`, the four `registered*` fields become required.

### Password (BE-109)

```json
{ "password": "AcmeRocks!9", "confirmPassword": "AcmeRocks!9" }
```

Success message: `"Onboarding complete. Welcome to Riverly."` — FE can now route the user through `switch-product` to the dashboard.

---

## 0.1. Public reference endpoints

These power onboarding pickers. No auth, response cached server-side for 24h plus an explicit `Cache-Control: public, max-age=86400` header so the app can cache client-side too.

| Endpoint | Purpose | Response shape |
|---|---|---|
| `GET /api/v1/reference/industries` | BE-105 industry picker | `{ "data": [ { "slug": "fashion-apparel", "name": "Fashion & Apparel" }, ... ] }` |
| `GET /api/v1/reference/states` | BE-108 state picker | `{ "data": [ { "code": "LA", "name": "Lagos" }, ... ] }` |

Server stores both lists in DB so ops can edit without a code deploy. FE should send `industry` as the **slug** and `state` as the **name** (e.g. `"Lagos"`).

---

## ↓ Legacy reference (post-onboarding endpoints still apply unchanged below) ↓

---

## 1. Auth model (read this first)

The SME app uses a **two-token flow**. The user enters their email/phone + password once; the app does the rest automatically.

1. `POST /api/v1/identity/signup` or `/login` → returns an **identity token** (short-lived, ~15 min). This token can only call `/api/v1/identity/*` endpoints.
2. The app immediately follows up with `POST /api/v1/identity/switch-product` → returns a **product token** (7 days). This is what every other endpoint takes. Save it; refresh on 401.

There is **no product picker UI** in the SME app — it always asks for `productType: "Sme"` (numeric `1`). If the user has no SME membership yet, the switch fails and the app routes them to enrollment (`POST /api/v1/sme/enroll`), then retries the switch.

**Headers:**
- All authenticated calls: `Authorization: Bearer <token>`
- All JSON requests: `Content-Type: application/json`

---

## 2. Response envelope

Every response has the same shape:

```json
{
  "status": true | false,
  "message": "<human-readable summary>",
  "data": <object or null>
}
```

For validation failures we add a per-field map:

```json
{
  "status": false,
  "message": "Please check the highlighted fields and try again.",
  "data": null,
  "errors": {
    "transactionPin": "Please enter your 4-digit transaction PIN.",
    "bankCode":       "Please pick a bank."
  }
}
```

**FE pattern:**
- `status: true` → use `data`
- `status: false` → show `message` as a toast/banner
- If `errors` is present → overlay each entry on its form field inline (and also show `message` as the summary toast)

All `message` and `errors` values are **safe to display verbatim** to end users. Backend never leaks Anchor / Provider error text in user-facing strings; those go to logs.

---

## 3. HTTP status codes

| Code | Meaning |
|---|---|
| 200 | Request reached the handler. **Check `status` in the body** — service-level failures use 200 with `status: false`. |
| 400 | Model validation failed (missing fields, wrong types). Body has the friendly per-field `errors` map. |
| 401 | Missing or invalid `Authorization` header. Token expired? Refresh the product token. |
| 403 | Authenticated but role doesn't allow this action. |
| 500 | Server error. Show generic "Something went wrong" and retry. |

---

## 4. Identity endpoints

### `POST /api/v1/identity/signup`
Create a brand-new Riverly identity.

**Request:**
```json
{ "fullName": "Jane Doe",
  "email": "jane@example.com",
  "phoneNumber": "+2348100000000",
  "password": "MyStrongPass1!" }
```

**Success (`status: true`):** returns `data.accessToken` (identity token).

**Common failures:**
- `An account with this email already exists. Please log in instead.`
- `An account with this phone number already exists. Please log in instead.`
- Validation: missing fullName/email/phone/password, password too short, bad email format.

### `POST /api/v1/identity/login`
Sign in.

**Request:** `{ "identifier": "<email or phone>", "password": "..." }`

**Success:** returns identity token + `availableProducts` array (used by Settings → "Your other Riverly products"). `defaultProduct` set if user has only one.

**Common failures:** `Invalid credentials.`, `Your account is suspended.`, `Your account is temporarily locked.`

### `POST /api/v1/identity/switch-product`
Exchange identity token for a product-scoped token. **The SME app always sends `productType: 1` (SME)** — no user input.

**Request:** `{ "productType": 1, "profileId": null }`

**Success:** returns `data.accessToken` (product token, 7-day). Store this; it replaces the identity token for all subsequent calls.

**Common failures:** `You don't have a Sme product on this account.` → route the user to enrollment.

### `GET /api/v1/identity/me`
Used by Settings → "Your other Riverly products". Returns the identity profile + all memberships across products. Use the membership list to render which other Riverly apps the user has access to.

### `POST /api/v1/identity/logout`
**Request:** `{ "refreshToken": "<the refresh token stored when switch-product returned>" }`

**Success:** revokes the refresh token. Client should clear both tokens locally and route to sign-in.

---

## 5. SME enrollment + KYB

### `POST /api/v1/sme/enroll`
Auth: identity token (this happens before `switch-product` succeeds).

**Request:**
```json
{ "businessName": "Acme Ventures",
  "rcNumber": "BN12345",
  "industry": "Technology",
  "businessAddress": "12 Adeola Odeku",
  "city": "Lagos",
  "state": "Lagos",
  "postalCode": "100001",
  "country": "Nigeria" }
```

**Success:** SME profile created, KYB state = `NotStarted`. User can now `switch-product` and land on the dashboard.

**Common failures:** `This RC number is already registered.`, plus validation errors.

### `GET /api/v1/sme/profile`
Returns the SME profile **plus** the KYB document state. This is the single source of truth for the dashboard banner.

**Response shape (data):**
```json
{ "id": "<guid>",
  "businessName": "...",
  "rcNumber": "...",
  "industry": "...",
  "businessAddress": "...",
  "city": "...", "state": "...", "postalCode": "...", "country": "Nigeria",
  "kybStatus": "NotStarted" | "PendingReview" | "Approved" | "Rejected",
  "kybRejectionReason": null | "string",
  "kybSubmittedAt": null | "ISO",
  "kybReviewedAt":  null | "ISO",
  "accountNumber":  null | "string",       // populated after KYB approved
  "documents": [
    { "slot": "CacCertificate", "fileUrl": "...", "fileName": "...", "uploadedAt": "..." }
  ],
  "missingRequiredSlots": ["ProprietorBvn", "Selfie"]  // empty when all uploaded
}
```

**FE: drive the dashboard banner off `kybStatus`:**
- `NotStarted` → "Verify your business to start using your account."
- `PendingReview` → "Documents under review — we'll notify you."
- `Rejected` → "Some documents need your attention" + show `kybRejectionReason`.
- `Approved` → no banner, account is live.

### `POST /api/v1/sme/kyb/documents`
Upload one slot. Idempotent per slot (re-upload replaces).

**Request:**
```json
{ "slot": 0,                              // see KYB slot enum below
  "fileUrl": "https://...",               // URL to the file you've already put in S3/storage
  "fileName": "cac-cert.pdf",
  "contentType": "application/pdf" }
```

**KYB slot enum:**
| Value | Slot | Purpose |
|---|---|---|
| 0 | `CacCertificate` | CAC business-name registration certificate |
| 1 | `CacStatusReport` | CAC status report |
| 2 | `ProprietorId` | Driver's licence / NIN slip / passport |
| 3 | `ProprietorBvn` | **Put the BVN value (11 digits) in `fileUrl`** — we treat this slot as text for now |
| 4 | `Selfie` | In-app selfie URL (after liveness pass) |

**Common failures:**
- `KYB is already approved. Documents cannot be changed.`
- `KYB is under review. Wait for the result before re-uploading.`

### `POST /api/v1/sme/kyb/submit`
Move status from `NotStarted` → `PendingReview`. Refuses if any required slot is missing.

**Common failures:** `Please provide the following before submitting: ProprietorBvn, Selfie.`

---

## 6. Account, balance, transactions

### `GET /api/v1/sme/account`
Returns the local account record (one per profile). Available only after KYB is approved.

**Response (`data`):**
```json
{ "id": "<guid>",
  "accountNumber": "1234567890",
  "bankName": "Anchor MFB",
  "currency": "NGN",
  "availableBalance": 12500.00,
  "ledgerBalance":    12500.00,
  "status": "Active",
  "lastBalanceRefreshAt": "2026-05-26T10:00:00Z" }
```

**Common failures:** `No SME account found. Complete KYB verification first.`

### `GET /api/v1/sme/account/balance`
Triggers a live refresh from Anchor, updates the local cache, and returns the same shape as `/sme/account`.

### `GET /api/v1/sme/transactions?limit=50`
Latest first. `limit` clamped to 1–200.

**Response (`data`):** array of:
```json
{ "id": "<guid>",
  "reference": "ref-abc",
  "direction": "Credit" | "Debit",
  "type": "Deposit" | "BankTransfer",
  "amount": 1000.00, "fee": 0.00, "currency": "NGN",
  "balanceAfter": 12500.00,
  "status": "Pending" | "Completed" | "Failed" | "Reversed",
  "counterpartyName": "...",
  "counterpartyAccountNumber": "...",
  "counterpartyBankCode": "000013",
  "narration": "...",
  "createdAt": "...", "completedAt": "..." }
```

### `GET /api/v1/sme/transactions/{reference}/receipt`
Single-transaction detail for receipt rendering. Adds `sourceAccountNumber`, `sourceAccountName`, `anchorTransactionId`. Returns 404-style fail (`Transaction not found.`) if the reference doesn't belong to this user's account.

### `GET /api/v1/sme/transactions/{reference}/receipt.pdf`
**Returns a PDF binary** (not JSON envelope). Content-Type: `application/pdf`. Filename: `riverly-receipt-{reference}.pdf`. Render or download client-side. If the transaction can't be found, falls back to the JSON envelope shape.

---

## 7. Banks, name enquiry, external transfer

### `GET /api/v1/sme/banks`
Returns the full Nigerian bank list, sourced from Anchor.

**Response (`data`):** array of `{ "code": "000013", "name": "GTBANK PLC" }`.

**⚠️ Important:** these codes are **6-digit NIBSS codes**, not the legacy 3-digit codes. Use them as-is for name enquiry and transfer — don't translate.

### `POST /api/v1/sme/name-enquiry`
Looks up the account name for a (bank, accountNumber) pair.

**Request:** `{ "accountNumber": "0123456789", "bankCode": "000013" }`

**Response (`data`):**
```json
{ "isValid": true,
  "accountName": "Alika Mohammed",
  "accountNumber": "0123456789",
  "bankCode": "000013" }
```

`isValid: false` with `accountName: null` means Anchor couldn't resolve. FE should disable the "Continue" button until `isValid: true`.

### `POST /api/v1/sme/transfers/external`
Initiate an outbound bank transfer.

**Request:**
```json
{ "accountNumber":  "0123456789",
  "accountName":    "Alika Mohammed",
  "bankCode":       "000013",
  "amount":         1000.00,
  "narration":      "Payment for invoice 123",
  "transactionPin": "7392",
  "idempotencyKey": "<unique GUID per submit attempt>",
  "saveBeneficiary": true                            // optional
}
```

**Success (`data`):** the transaction summary (same shape as transactions list, `status: Pending` typically — settled later via webhook).

**Common failures (all friendly):**
- `Please enter your 4-digit transaction PIN.` (validation)
- `Transaction PIN is not set. Set your PIN before making transfers.`
- `Incorrect transaction PIN.`
- `Insufficient balance.`
- `The transfer couldn't be completed. Please try again in a moment.` (Anchor-side failure)
- `Transfer already submitted.` (you reused an idempotencyKey)

**Idempotency:** ALWAYS generate a fresh `idempotencyKey` (UUID) when the user lands on the review screen. Reuse it for retries of the same logical submission. Server returns the existing transaction if the key matches an earlier call — never double-sends.

---

## 8. Transaction PIN

### `POST /api/v1/sme/transaction-pin/set`
**Request:** `{ "pin": "7392", "confirmPin": "7392" }`

**Rules:** 4 digits. Cannot be sequential (`1234`, `2345`) or repeated (`1111`).

**Common failures (all friendly):**
- `PINs do not match.`
- `PIN cannot be sequential (e.g. 1234) or repeated digits (e.g. 1111).`
- `Transaction PIN already set. Use change PIN instead.`

### `POST /api/v1/sme/transaction-pin/verify`
**Request:** `{ "pin": "7392" }` → returns `status: true` if correct.

Use this if you want to gate a sensitive action with PIN re-entry without actually submitting a transfer.

### `POST /api/v1/sme/transaction-pin/change`
**Request:** `{ "currentPin": "7392", "newPin": "5837", "confirmNewPin": "5837" }`

**Common failures:** `New PINs do not match.`, `New PIN must be different from the current PIN.`, `Current PIN is incorrect.`, weak-PIN rule.

### `POST /api/v1/sme/transaction-pin/forgot`
**Request:** `{ "password": "...", "newPin": "4628", "confirmNewPin": "4628" }`

Identity proof is the account password (we don't have email/SMS OTP wired yet). Returns success on match.

---

## 9. Beneficiaries

### `GET /api/v1/sme/beneficiaries`
Returns all saved recipients ordered by most-recently-used.

**Response (`data`):** array of:
```json
{ "id": "<guid>",
  "accountName": "...",
  "accountNumber": "...",
  "bankCode": "...",
  "bankName": "...",
  "nickname": "...",
  "createdAt": "...", "lastUsedAt": "..." }
```

### `POST /api/v1/sme/beneficiaries`
**Request:**
```json
{ "accountName": "Alika Mohammed",
  "accountNumber": "0123456789",
  "bankCode": "000013",
  "bankName": "GTBANK PLC",
  "nickname": "Alika" }
```

**Idempotent:** posting the same `(accountNumber, bankCode)` twice doesn't duplicate — refreshes nickname + `lastUsedAt` and returns the existing row.

### `DELETE /api/v1/sme/beneficiaries/{id}`
Hard delete. Returns `status: true` on success.

**Failure:** `Beneficiary not found.` (someone else's beneficiary, or wrong ID).

---

## 9.5 Settings, notifications, support, push devices (slice 5)

### Profile editing
- `PATCH /api/v1/sme/profile/identity` body `{ fullName?, phoneNumber? }` — updates the identity row. Changing phone clears verified-at, so a future re-verification flow should re-prompt the OTP.
- `PATCH /api/v1/sme/profile/business` body `{ industry?, businessAddress?, city?, state?, postalCode?, country? }` — updates the SME business info. Business name and RC number are immutable (locked to KYB).

### Notification preferences
- `GET /api/v1/sme/notifications/preferences` — returns the toggle state. If the user has never set them, defaults are returned (everything on except marketing).
- `PATCH /api/v1/sme/notifications/preferences` — partial update; only fields you include are changed. Toggles: `transfersPush/Email`, `depositsPush/Email`, `kybPush/Email`, `securityPush/Email`, `marketingPush/Email`.

### Push device registration
- `POST /api/v1/sme/push/devices` body `{ deviceId, pushToken, platform }` — registers (or refreshes) an FCM/APNs device token for the current identity. Re-registering the same `deviceId` updates the token; doesn't duplicate.
- `DELETE /api/v1/sme/push/devices/{deviceId}` — unregister. Always returns success even if the device wasn't found (idempotent).

> Push send itself is wired but waits on Firebase credentials. Once those land, deposit/transfer/KYB events fan out to every registered device automatically based on `notifications/preferences`.

### Support / report-issue
- `GET /api/v1/sme/support/tickets` — list this user's tickets.
- `POST /api/v1/sme/support/tickets` body `{ category, subject, body, relatedTransactionReference? }` — open a new ticket. Categories: `TransferIssue`, `Account`, `KybVerification`, `Technical`, `Other`.
- `GET /api/v1/sme/support/tickets/{id}` — full ticket + ordered comments.
- `POST /api/v1/sme/support/tickets/{id}/comments` body `{ body }` — add a comment to an open ticket. Fails on resolved/closed tickets with copy `"This ticket has been closed. Open a new one to continue."`

Statuses: `Open` → `InProgress` → `Resolved` / `Closed`. Admin transitions happen outside the user-facing API.

---

## 10. Webhooks (read-only context for FE)

These are server-to-server events from Anchor — the FE doesn't call them. Listed here so you understand the timing of dashboard updates.

| Event | What we do | What the FE sees |
|---|---|---|
| `customer.identification.success` | Mark KYB approved + open Anchor deposit account (if not already done by admin endpoint) | `GET /sme/profile` will reflect `kybStatus: "Approved"` + a populated `accountNumber` |
| `customer.identification.failed` | Mark KYB rejected with reason | `kybStatus: "Rejected"` with `kybRejectionReason` |
| `deposit.completed` | Re-verify the transaction by calling Anchor, then add a `SmeTransaction` row + bump the account balance | A new row appears in `GET /sme/transactions`; `GET /sme/account/balance` shows the new value |

**Polling guidance:** when on the dashboard, refresh `/sme/profile` every ~10s while `kybStatus == "PendingReview"`, and after a transfer is submitted refresh `/sme/account/balance` + `/sme/transactions` every ~5s for ~1 min (or wire push notifications when those land in slice 5).

---

## 11. Known constraints + business rules

- **Phone number format:** anything you pass is normalised server-side (`0xx…` → `+234xx…`). Either works.
- **Email is case-insensitive.**
- **One SME profile per identity** for now. A user can't open a second SME under the same email.
- **Account is single-account-per-profile** (MVP). One business account.
- **NGN only**, kobo conversion handled server-side. FE sends amounts in NGN (e.g., `1000.00`).
- **Transfer amount range:** ₦1 – ₦5,000,000 per transaction.
- **PIN rules:** exactly 4 digits, not sequential, not repeated.
- **Idempotency keys:** required for transfers; recommended UUIDs.

---

## 12. End-to-end happy path (cheat sheet)

```
1. POST /identity/signup       { fullName, email, phone, password }   → identity token
2. POST /sme/enroll            { businessName, rcNumber, … }          → SME profile (KYB: NotStarted)
3. POST /identity/switch-product { productType: 1 }                   → product token
4. Land on dashboard, show KYB banner from /sme/profile
5. POST /sme/kyb/documents x5  (CAC cert, status, ID, BVN, selfie)
6. POST /sme/kyb/submit         → KYB: PendingReview
7. [admin reviews + approves → KYB: Approved, Anchor account created]
8. GET /sme/account/balance     → show real balance
9. POST /sme/transaction-pin/set                                       → first-time PIN
10. User wants to send money:
   a. GET /sme/banks
   b. POST /sme/name-enquiry          → resolved name
   c. POST /sme/transfers/external   { with transactionPin + idempotencyKey }
   d. Show success + receipt from /sme/transactions/{ref}/receipt
```

---

## 13. Verification status (smoke-tested on staging)

| Endpoint group | Status | Notes |
|---|---|---|
| Identity (signup, login, switch-product, me, logout) | ✅ verified | Slice 1 |
| SME enrollment + KYB document upload + submit | ✅ verified | Slice 1 |
| Admin KYB approve / reject | ✅ verified (returns "not found" for fake IDs as expected) | Slice 2 |
| Anchor webhook signature + verify-before-credit | ✅ verified (forged sig → 401; verified sig + bogus ref → "Refusing") | Slice 2 |
| `GET /sme/banks` | ✅ verified (289 real banks) | Slice 3 |
| `POST /sme/name-enquiry` | ✅ verified — "Alika Mohammed" for the GTBank sandbox test | Slice 3 |
| `GET /sme/account` (pre-KYB) | ✅ verified — clean "No account" | Slice 3 |
| `GET /sme/transactions` (pre-KYB) | ✅ verified — empty list | Slice 3 |
| Transaction PIN (set/verify/change/forgot + weak rejection) | ✅ verified | Slice 4 |
| Beneficiaries (CRUD + idempotency) | ✅ verified | Slice 4 |
| Transfer PIN gating (missing → 400; wrong → fail) | ✅ verified | Slice 4 |
| Friendly validation messages (per-field + summary) | ✅ verified | Slice 4.5 |

**Not yet verified end-to-end (need full chain or external dependency):**
- External transfer happy path — needs an admin-approved test user + funded sandbox account (next slice)
- Anchor `deposit.completed` webhook → SmeTransaction persisted — needs a real sandbox deposit triggered from Anchor (next slice)

---

## 14. Changelog

| Date | Change | Slice |
|---|---|---|
| 2026-05-26 | Initial document covering identity, SME enroll, KYB, account, transfers, PIN, beneficiaries, receipts, webhooks | 1–4 |
| 2026-05-26 | Slice 5 added: PDF receipt download, identity/business profile edits, notification preferences, push device registration, support/report-issue tickets | 5 |
