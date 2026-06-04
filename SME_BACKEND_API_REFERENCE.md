# SME App — Backend API Reference

**Status:** Living document. Updated every time a new endpoint ships.
**Audience:** Frontend / mobile engineers building the SME app.
**Base URL (staging):** `https://api.riverly.ng`
**Last refreshed:** 2026-06-02 — BE-01 (in-app KYB) shipped end to end. Three phases live on staging: transaction guard + status emails + reviewer tracking (Phase 1), owner-personal data + new doc slots (Phase 2), private S3 + multipart upload + presigned URLs (Phase 3).

> **What changed since 2026-05-29 (read this section if you've been building against the old doc):**
> 1. **KYB Phase 2 (post-signup) is now fully spec-aligned.** New endpoint `POST /sme/kyb/owner-personal` captures the business owner's title, DOB, and personal address. Required document set is now `CacCertificate` + `CacForm7` + `ProofOfPersonalAddress` + `ProofOfBusinessAddress`. The old slots (CacStatusReport, ProprietorId, ProprietorBvn, Selfie) still upload but no longer gate submission. See [§5](#5-sme-enrollment--kyb).
> 2. **Status enum gained `ActionRequired`.** Five states now: `NotStarted, PendingReview, Approved, Rejected, ActionRequired`. Ops can flag a submission for follow-up instead of outright rejecting it. The user re-uploads, resubmits, status flows back to PendingReview. See [§5.2](#52-kyb-status--dashboard-banner).
> 3. **Document upload now has two paths.** Legacy URL-handoff (`POST /sme/kyb/documents`) still works. **Prefer the new multipart path** (`POST /sme/kyb/documents/upload`) — it streams the file through the BE, server-validates mime + size (PDF / JPG / PNG ≤ 5 MB), and lands the file in a private S3 bucket. See [§5.4](#54-kyb-document-upload).
> 4. **Documents are now private.** New uploads store an opaque S3 key, not a public URL. To view a doc, call `GET /sme/kyb/documents/{slot}/url` — returns a short-lived (15-min) presigned URL. The bulk `/sme/profile` response no longer carries inline URLs for private rows. See [§5.5](#55-get-document-url-presigned).
> 5. **Transaction guard.** `POST /sme/transfers/external` now rejects with a status-aware message when `kybStatus != Approved`. Gate transfer UI client-side too. See [§7.transfer-guard](#post-apiv1smetransfersexternal).
> 6. **Status emails on every transition.** Submit / Approve / Reject / ActionRequired all trigger a Resend email *in addition to* the push notification. Nothing for the FE to do — just heads-up.
> 7. **New single-shot SME enrollment endpoint.** `POST /api/v1/identity/products/sme/enroll` mirrors the Personal + Corporate enroll endpoints. Collapses the 7-step BE-106→BE-109 onboarding into one call for scripted setups. UI flow unchanged. See [§5.1](#51-single-shot-enrollment-new).
> 8. **Staging-only `debugOtpCode` in OTP responses.** Email-OTP-firing endpoints (`/identity/signup`, `/email/resend`, `/email/add`, `/phone/skip`, `/phone/verify`-success) return `data.debugOtpCode` on non-prod hosts. Lets you test against fake email addresses. Always `null` in production.
>
> **What changed since 2026-05-26 (older — still applies):**
> 9. **Phone SMS via Sendchamp.** Phone OTP is now 5 digits (was logged-only); delivered by Sendchamp from sender `SC-OTP` the moment the wallet is funded.
> 10. **OTPs are SEQUENTIAL.** Phone fires at signup; email fires automatically when phone is verified or skipped. Avoids the 10-min TTL expiry race.
> 11. **Transfer flow has stricter validation, a PIN attempt counter, and a 10-min pending expiry.** See [§7](#7-banks-name-enquiry-external-transfer) and [§8](#8-transaction-pin).
> 12. **Beneficiaries: name-lookup is required before save**, saved name is bank-verified. Server-side recipient search via `?search=`. See [§9](#9-beneficiaries).
> 13. **Rate limits on payment-flow endpoints.** Hit `429` and back off.
> 14. **Free-text sanitization** (narration, nickname, search). HTML/script content stripped.
> 15. **Append-only payment audit log** runs transparently.

> **READ ME FIRST — the onboarding flow has changed.** The single-step `/api/v1/sme/enroll` payload (and the old `/api/v1/identity/signup` that took `fullName + password`) are gone. Onboarding is now a 7-step flow. See **Section 0 — Onboarding flow (BE-102 → BE-109)** below. The legacy endpoints documented further down still exist for everything *post-onboarding* (dashboard, transfers, settings, etc.).

> **Provider status:**
> - **Email (Resend) — LIVE.** Real OTP + welcome emails ship from `noreply@mail.riverly.ng`. No code workaround needed any more.
> - **Push (Firebase Cloud Messaging) — LIVE.** Register a device token via `POST /api/v1/sme/push/devices` and you'll start receiving pushes on `deposit`, `transfer`, and `kyb` events.
> - **SMS (Sendchamp) — INTEGRATED, AWAITING WALLET.** Phone OTP code generation, send, and verify are all wired through Sendchamp's `/verification/create` + `/verification/confirm`. Real SMS will deliver as soon as the Sendchamp wallet is funded; until then the API call fails-soft (signup still completes, email OTP still works, but no SMS reaches the phone). Once funded: no FE change required.

---

## Table of contents

- [0. Onboarding flow (BE-102 → BE-109)](#0-onboarding-flow-be-102--be-109)
  - [Sign-up payload (BE-102)](#sign-up-payload-be-102)
  - [OTP verify payloads (BE-103 / BE-104)](#otp-verify-payloads-be-103--be-104)
  - [Business profile (BE-106)](#business-profile-be-106)
  - [Logo upload (BE-106 comment)](#logo-upload-be-106-comment)
  - [Owner role (BE-107)](#owner-role-be-107)
  - [Address (BE-108)](#address-be-108)
  - [Password (BE-109)](#password-be-109)
- [0.1 Public reference endpoints](#01-public-reference-endpoints)
- [1. Auth model (read this first)](#1-auth-model-read-this-first)
- [2. Response envelope](#2-response-envelope)
- [3. HTTP status codes](#3-http-status-codes)
- [4. Identity endpoints](#4-identity-endpoints)
  - [Password reset (added 2026-06-04 — covers SME + Corporate + Personal)](#password-reset-added-2026-06-04--covers-sme--corporate--personal)
- [**5. SME enrollment + KYB (BE-01 — UPDATED)**](#5-sme-enrollment--kyb)
  - [5.1 Single-shot enrollment (NEW)](#51-single-shot-enrollment-new)
  - [5.2 KYB status + dashboard banner](#52-kyb-status--dashboard-banner)
  - [5.3 Business owner personal info (NEW)](#53-business-owner-personal-info-new)
  - [5.4 KYB document upload](#54-kyb-document-upload)
  - [5.5 Get document URL (presigned, NEW)](#55-get-document-url-presigned-new)
  - [5.6 Submit KYB for review](#56-submit-kyb-for-review)
  - [5.7 What ops can do (status transitions)](#57-what-ops-can-do-status-transitions)
- [6. Account, balance, transactions](#6-account-balance-transactions)
- [7. Banks, name enquiry, external transfer](#7-banks-name-enquiry-external-transfer)
- [8. Transaction PIN](#8-transaction-pin)
- [9. Beneficiaries](#9-beneficiaries)
- [9.5 Settings, notifications, support, push devices](#95-settings-notifications-support-push-devices-slice-5)
- [10. Webhooks (read-only context for FE)](#10-webhooks-read-only-context-for-fe)
- [11. Known constraints + business rules](#11-known-constraints--business-rules)
- [12. End-to-end happy path (cheat sheet)](#12-end-to-end-happy-path-cheat-sheet)
- [13. Verification status](#13-verification-status-smoke-tested-on-staging)
- [14. Changelog](#14-changelog)

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
{ "code": "12345" }     // phone — 5 digits (Sendchamp)
{ "code": "123456" }    // email — 6 digits (Riverly)
```

Validation rules:
- **Email OTP** (Riverly's `OtpService`): 6 digits, 10-min expiry, 5 attempts per OTP, 3 resends per 10 minutes. Delivered for real via Resend.
- **Phone OTP** (Sendchamp's `/verification/*` endpoints): 5 digits, 10-min expiry, attempt + resend limits enforced by Sendchamp. Sender ID `SC-OTP`. Delivered via SMS the moment the Sendchamp wallet has balance; if the wallet is empty the API call fails-soft and the user can still complete signup on the email channel.

Resend cooldown returns `60` (seconds) so the FE can show a countdown.

**Sequential dispatch (important):**
- Only the phone OTP fires at `/identity/signup`. The email OTP is dispatched **automatically** when phone is verified (`/phone/verify` success) or skipped (`/phone/skip`).
- This means a slow user can never have a stale email code waiting for them at the email screen — it's freshly issued the moment they finish the phone step.
- For Personal users who don't even have an email at signup, the email OTP dispatch happens on `/identity/email/add` instead (see §0.2 below).

**During the funding gap:** phone OTP codes still log to CloudWatch on the server side (Sendchamp's `Low balance` response is logged with the request) — QA can read them there if needed. Real users should use email verification.

**Staging-only debug echo (`data.debugOtpCode`):** when the host is not Production, every email-OTP-firing response (`/identity/signup`, `/email/resend`, `/email/add`, `/phone/skip`, `/phone/verify`-success when email is on file) carries a `debugOtpCode` field with the plaintext 6-digit code. Always `null` in production. Use it to test signup/recovery flows against fake email addresses without checking the inbox. Phone OTPs (Sendchamp) cannot be echoed this way — the code is generated on Sendchamp's side, not ours.

---

### 0.2. Cross-product addendum: Personal email-add flow

The SME app submits name + phone + email together at signup, so this section is informational for SME. It matters for Personal (and anywhere else that wants to defer email until a later screen).

`POST /api/v1/identity/email/add` — body `{ "email": "user@example.com" }`. Requires identity bearer token.

- Sets the email on the identity row.
- Dispatches the email OTP immediately. Success response message: `"We've sent a code to your email."`
- Re-callable if the user wants to change the email before it's verified (each call clears any previous `EmailVerifiedAt` and re-sends).
- Rejected if another identity already owns this email (`"An account with this email already exists. Please log in instead."`).
- Rejected if this identity already has a *verified* email and is trying to switch (`"Your email is already verified. Use settings to change it."`).
- Rate-limited via the shared `"otp"` bucket (5 / minute / user).

After this returns, the FE flow is the same as SME: `POST /identity/email/verify { code }`.

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

### Password reset (added 2026-06-04 — covers SME + Corporate + Personal)

Three-step flow that lives under `/identity/*` so SME and Corporate users (whose credential is `Identity.PasswordHash`, not the legacy Personal `users.Passcode`) can recover access. **OTP is always delivered to the email on file** — SMS is not used here because users travelling abroad lose carrier signal.

These endpoints are anonymous (no JWT required).

#### `POST /api/v1/identity/passcode/forgot`
Start the reset.

**Request:** `{ "identifier": "jane@example.com" }`  // email OR phone

**Response:** always 200 OK with a generic message, regardless of whether the identifier matched — prevents user enumeration.
```json
{
  "status": true,
  "message": "If an account exists with this identifier, a reset code has been sent.",
  "data": { "sent": true, "resendCooldownSeconds": 60, "debugOtpCode": null }
}
```
`debugOtpCode` is populated only on Staging/Development so QA can complete the flow without an inbox round-trip. Always `null` in Production.

#### `POST /api/v1/identity/passcode/verify-otp`
Exchange the OTP for a short-lived (10-minute) reset token.

**Request:**
```json
{
  "identifier": "jane@example.com",
  "otpCode":    "123456"
}
```

**Response:**
```json
{
  "status": true,
  "message": "OTP verified. Please set your new passcode within 10 minutes.",
  "data": {
    "resetToken": "<opaque base64>",
    "productType": "Sme"               // tells FE which credential format to collect next
  }
}
```

**Use `productType` to render the right input on the next screen:**
- `Personal` → 6-digit numeric keypad
- `Sme` / `Corporate` → password field with 8+ char complexity rules

#### `POST /api/v1/identity/passcode/reset`
Finalise the reset.

**Request:**
```json
{
  "identifier":      "jane@example.com",
  "resetToken":      "<from verify-otp>",
  "newPasscode":     "Str0ng!Pass",
  "confirmPasscode": "Str0ng!Pass"
}
```

Server validates `newPasscode` against the identity's product:
- **Personal:** exactly 6 numeric digits
- **SME / Corporate:** 8+ chars with at least one uppercase, lowercase, digit, and special character

**Response:** `{ "status": true, "message": "Your passcode has been reset...", "data": null }`. After this the user can log in via `/identity/login` with the new credential.

#### `POST /api/v1/identity/passcode/change`
**Authenticated.** For Settings → Change password: the user supplies their current credential, no OTP. Works for any product.

**Request:**
```json
{
  "currentPasscode": "Str0ng!Pass",
  "newPasscode":     "Even5tr0nger!",
  "confirmPasscode": "Even5tr0nger!"
}
```

Server validates:
- Current passcode matches the identity's stored hash (otherwise: "Your current passcode is incorrect.")
- New and confirm match
- New passcode satisfies the product's format rule (Personal: 6 digits; SME/Corporate: 8+ chars with upper/lower/digit/special)
- New passcode is **different** from the current one ("New passcode must be different from your current passcode.")

**Response:** `{ "status": true, "message": "Your passcode has been updated.", "data": null }`.

---

## 5. SME enrollment + KYB

Two ways to enroll a new SME identity into the Riverly SME product:

- **The 7-step UI flow** documented in [§0](#0-onboarding-flow-be-102--be-109): signup → phone verify → email verify → business profile → owner role → address → password. This is what the SME app walks the user through; persists state across steps. Use this for any FE-driven enrollment.
- **The single-shot API endpoint** ([§5.1](#51-single-shot-enrollment-new)): one POST, all the fields. Use this for scripted setups, QA, integration tests.

KYB Phase 2 (post-signup; this is BE-01) happens AFTER the user has an SME profile and is on the dashboard. It requires owner-personal info ([§5.3](#53-business-owner-personal-info-new)) + four documents ([§5.4](#54-kyb-document-upload)), then a submit-for-review call. Ops takes over from there ([§5.7](#57-what-ops-can-do-status-transitions)).

### 5.1 Single-shot enrollment (NEW)

`POST /api/v1/identity/products/sme/enroll`

Auth: identity token (post-signup, post-OTP-verify).

Collapses BE-106 (business info) + BE-109 (password) into one call. Owner role + address can be done afterwards via the existing onboarding endpoints if needed.

**Request:**
```json
{
  "businessName": "Acme Ventures",
  "businessRegistrationNumber": "BN12345",
  "industry": "fashion-apparel",
  "description": "We sell hand-poured candles…",  // optional, 20-500 chars
  "password": "AcmeRocks!9",
  "websiteUrl": "https://acme.ng",                 // optional
  "logoUrl": null                                  // optional
}
```

**Response (`data`):**
```json
{
  "identityId":   "<guid>",
  "smeProfileId": "<guid>",
  "membershipId": "<guid>",
  "role":         "Owner"
}
```

**Side effects:** sets `identity.PasswordHash`, stamps `identity.OnboardingCompletedAt`, creates `SmeProfile` (KYB state `NotStarted`) + `ProductMembership(ProductType=Sme, Role=Owner)`.

**Common failures:** `Please verify your email or phone before continuing.`, `Business registration number is required.`, `Password must be at least 8 characters.`, `You already have an SME profile.`

---

### 5.2 KYB status + dashboard banner

`GET /api/v1/sme/profile` is the single source of truth.

**Response shape (`data`):**
```json
{
  "id": "<guid>",
  "businessName": "...",
  "businessRegistrationNumber": "...",
  "industry": "...",

  // Operating + registered business address (from BE-108)
  "operatingAddressLine": "...",
  "operatingCity": "...",
  "operatingState": "...",
  "operatingPostalCode": "...",
  "operatingCountry": "Nigeria",
  "isRegisteredAddressSameAsOperating": true,
  // registeredAddressLine / registeredCity / etc. — populated only when above = false

  // BE-01 KYB status
  "kybStatus": "NotStarted" | "PendingReview" | "Approved" | "Rejected" | "ActionRequired",
  "kybRejectionReason": null | "string",
  "kybSubmittedAt": null | "ISO",
  "kybReviewedAt":  null | "ISO",
  "accountNumber":  null | "string",         // populated AFTER KYB approved

  // BE-01 Section A — business owner personal info (see §5.3)
  "ownerTitle":           null | "Mr" | "Mrs" | "Miss",
  "ownerDateOfBirth":     null | "YYYY-MM-DD",
  "ownerPersonalStreet":  null | "string",
  "ownerPersonalState":   null | "string",
  "ownerPersonalCountry": null | "Nigeria",
  "ownerPersonalCompleted": true | false,    // computed — all 5 owner fields present

  // BE-01 Section B — documents
  "documents": [
    {
      "slot": "CacCertificate",
      "fileUrl": "",                          // empty for private/new rows
      "fileName": "cac-cert.pdf",
      "uploadedAt": "ISO",
      "isPrivate": true,                      // true = call §5.5 to get a URL; false = use fileUrl directly
      "fileSizeBytes": 184320
    }
  ],
  "missingRequiredSlots": ["CacForm7", "ProofOfPersonalAddress"],   // empty when all four uploaded

  // ── Identity fields (mirrors /identity/me — added 2026-06-04) ──
  // Lets the FE render profile/settings without a second call. Identity
  // is eager-loaded server-side so these come for free.
  "firstName":     "Jane",
  "surname":       "Doe",
  "fullName":      "Jane Doe",
  "email":         "jane@example.com",
  "phoneNumber":   "+2348101234567",
  "emailVerified": true,
  "phoneVerified": true,

  // True when the identity has a transaction PIN on file (added 2026-06-04).
  // Use it to decide between routing to /sme/transaction-pin/set vs /verify
  // on the first money-movement attempt.
  "transactionPinSet": false
}
```

**Drive the dashboard banner off `kybStatus`:**

| `kybStatus` | Banner copy | What the user does next |
|---|---|---|
| `NotStarted` | "Verify your business to start using your account." | Tap → start KYB flow (owner-personal + docs) |
| `PendingReview` | "Documents under review — we'll notify you." | Wait (1–2 business days). Push + email come on transition. |
| `Approved` | *(no banner — account is live)* | Transact freely. |
| `Rejected` | "Your verification wasn't approved." + show `kybRejectionReason` | Open KYB screen, re-upload docs, resubmit. Status flips to `NotStarted` on re-upload. |
| `ActionRequired` | "We need a couple more details on your verification." + show `kybRejectionReason` | Same flow as Rejected — re-upload flagged docs, resubmit. |

**FE submit-button gate:** enable only when `ownerPersonalCompleted === true` AND `missingRequiredSlots.length === 0`. The BE enforces this server-side too.

---

### 5.3 Business owner personal info (NEW)

`POST /api/v1/sme/kyb/owner-personal`

BE-01 Section A. Captures the four fields the ticket requires. Idempotent — call as many times as you like before submit; each call replaces the prior values.

**Request:**
```json
{
  "title": "Mr",                       // "Mr" | "Mrs" | "Miss" (string enum)
  "dateOfBirth": "1990-04-15",         // ISO date (YYYY-MM-DD)
  "personalStreet": "12 Adeola Odeku St",
  "personalState": "Lagos",
  "personalCountry": "Nigeria"         // optional, defaults to Nigeria; rejected if anything else
}
```

**Response (`data`):** the full `SmeProfileResponse` (same shape as `GET /sme/profile`), so the FE can re-render the completion checklist without an extra fetch.

**Server-side validation:**
- `dateOfBirth` must be in the past
- User must be 18+ (`dateOfBirth ≤ today − 18 years`)
- `personalCountry`, if supplied, must be `"Nigeria"` (case-insensitive)
- `title` is a hard enum — bind fails on anything else

**Common failures:**
- `You must be at least 18 years old.`
- `Date of birth must be in the past.`
- `Personal country must be Nigeria.`
- `KYB is already approved. Owner details can't be changed from here.`
- `KYB is under review. Wait for the result before editing owner details.`

---

### 5.4 KYB document upload

**Two paths.** Both write to the same DB. The multipart path is preferred for new FE work; the legacy URL-handoff path remains for backward compat.

#### 5.4.a Multipart upload (PREFERRED)

`POST /api/v1/sme/kyb/documents/upload`

Content-Type: `multipart/form-data`. Fields:
- `file` — the binary. PDF / JPG / PNG only. Hard cap 5 MB (server-side enforced).
- `slot` — the document slot name (see slot enum below).

Server uploads to a **private** S3 bucket and stores the opaque object key. The bulk `/sme/profile` response will report this row with `isPrivate: true` and an empty `fileUrl`. To view the file, call [§5.5](#55-get-document-url-presigned-new) to mint a fresh short-lived presigned URL.

Idempotent per slot (re-upload replaces; old S3 object is best-effort deleted).

**Common failures:**
- `Please attach a file.`
- `File is too large. Max 5 MB.`
- `File type not allowed. PDF, JPG, or PNG only.`
- `KYB is already approved. Documents cannot be changed.`
- `KYB is under review. Wait for the result before re-uploading.`

#### 5.4.b Legacy URL-handoff (still works)

`POST /api/v1/sme/kyb/documents`

For when the FE has already uploaded to S3 elsewhere and just wants to register the URL.

**Request:**
```json
{
  "slot": "CacCertificate",
  "fileUrl": "https://...",
  "fileName": "cac-cert.pdf",
  "contentType": "application/pdf"
}
```

Rows from this path land with `isPrivate: false` — the URL is returned verbatim in `/sme/profile.documents[].fileUrl`. No presign needed.

#### Document slot enum

| Slot | Required for submit? | Purpose |
|---|:---:|---|
| `CacCertificate` | ✅ | CAC business-name registration certificate (CAC 2) |
| `CacForm7` | ✅ | CAC Form 7 — Particulars of Directors / Owners |
| `ProofOfPersonalAddress` | ✅ | Utility bill / bank statement matching `ownerPersonalStreet` |
| `ProofOfBusinessAddress` | ✅ | Utility bill matching the operating address |
| `CacStatusReport` |  | CAC status report — accepted but not required |
| `ProprietorId` |  | Driver's licence / NIN slip / passport — accepted but not required |
| `ProprietorBvn` |  | BVN value (11 digits) — accepted but not required |
| `Selfie` |  | In-app selfie URL — accepted but not required |

---

### 5.5 Get document URL (presigned, NEW)

`GET /api/v1/sme/kyb/documents/{slot}/url`

Mints a fresh **15-minute** presigned GET URL for one of the caller's own KYB documents. Use this when the user clicks "View document" — do not cache the URL, just open it.

**Response (`data`):**
```json
{
  "url": "https://riverly-staging-assets.s3.eu-north-1.amazonaws.com/...?X-Amz-Signature=…",
  "expiresAtUtc": "2026-06-02T07:15:00Z",
  "slot": "CacCertificate",
  "fileName": "cac-cert.pdf",
  "contentType": "application/pdf",
  "fileSizeBytes": 184320
}
```

**Common failures:** `No document found in that slot.`, `Document storage reference is missing.`

For legacy (`isPrivate: false`) rows, the `url` field is the stored public URL unchanged.

---

### 5.6 Submit KYB for review

`POST /api/v1/sme/kyb/submit`

Moves `kybStatus` from `NotStarted` → `PendingReview`. Two server-side gates — the call refuses with a specific message if either is unmet:

1. **Owner personal info complete.** All five `owner*` fields populated. Error: `Please complete the business owner personal details before submitting.`
2. **All four required documents uploaded.** Error: `Please provide the following before submitting: CacForm7, ProofOfPersonalAddress.`

On success:
- Status flips to `PendingReview`, `kybSubmittedAt` stamped
- User gets a push notification: *"We received your business verification…"*
- User gets a Resend email: *"Your KYB submission is in review"* (sets expectation of 1–2 business days)

---

### 5.7 What ops can do (status transitions)

These are admin endpoints — gated by `X-Admin-Key` header. Not part of the FE flow, but worth knowing because they drive the notifications the user sees.

| Endpoint | What it does | Notification to user |
|---|---|---|
| `POST /admin/sme/kyb/{profileId}/approve` | `PendingReview` → `Approved`. Triggers Anchor customer creation + deposit account opening (asynchronously). | Push + email: *"Your business is verified."* |
| `POST /admin/sme/kyb/{profileId}/reject` body `{ reason, reviewerEmail? }` | `PendingReview` → `Rejected`. | Push + email: *"We couldn't verify {businessName}"* + reason |
| `POST /admin/sme/kyb/{profileId}/action-required` body `{ reason, reviewerEmail? }` | `PendingReview` or `Approved` → `ActionRequired`. User can re-upload the flagged docs. | Push + email: *"Action needed on your business verification"* + reason |
| `GET /admin/sme/kyb/{profileId}/documents/{slot}/url` | Mints a 15-min presigned URL for ops to view any KYB document. | (none — read-only) |

**Reviewer identity** is stamped on `SmeProfile.KybReviewerEmail` for every transition. Ops can pass it in the body or via an `X-Admin-Reviewer` header (logging only — not the auth gate).

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

**Side-effect (RIV-PAY-002):** a successful lookup is cached server-side for 10 minutes against this identity. Saving the recipient (`POST /sme/beneficiaries`) within that window is allowed; outside it, the save endpoint rejects with `"Please look up the account name before saving this recipient."` — re-call this endpoint to refresh.

**Rate limit:** `10 requests / minute / user`. Exceeded → `429`. Anchor lookup itself has a server-side 10-second timeout; on timeout the response is `isValid: false` and the lookup is audited as `errorCode="lookup_failed"`.

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

**Amount rules (RIV-PAY-003):** server enforces, regardless of any client-side check.
- Min: ₦100
- Max single transfer: ₦5,000,000
- Validation errors: `"Amount must be a positive number."`, `"Minimum transfer is ₦100.00."`, `"Maximum single transfer is ₦5,000,000.00."`

**Narration (PAY-SYS-03):** sanitized on the server — HTML tags, script/style blocks, dangerous URL schemes (`javascript:`, `data:`), and event handlers are stripped before the value is stored or sent to Anchor. Max 300 chars after sanitization.

**KYB guard (BE-01):** The server now blocks transfers when `kybStatus != Approved`. Gate this client-side too — if the dashboard banner is showing, the transfer button should be disabled. Status-aware error copy:
- `Your business verification is still being reviewed (usually 1–2 business days). You'll be able to send money once it's approved.` — when `PendingReview`
- `Your business verification wasn't approved. Please log in to see what went wrong and resubmit.` — when `Rejected`
- `We need a couple more details on your business verification. Please log in to upload the flagged items and resubmit.` — when `ActionRequired`
- `Please complete your business verification before you can send money. Open the dashboard to start.` — when `NotStarted` or no profile

**Common failures (all friendly):**
- `Please enter your 4-digit transaction PIN.` (validation)
- `Transaction PIN is not set. Set your PIN before making transfers.`
- `Incorrect transaction PIN. N attempts remaining.` (RIV-PAY-004 — counter is user-level, 10-min window)
- `Transfer locked after 3 incorrect PIN attempts. Please reset your PIN to continue.` (this transfer reference is now locked for 1 hour — FE should route the user to `/transaction-pin/forgot`)
- `This transfer is locked because of too many incorrect PIN attempts. Please reset your PIN to continue.` (same reference re-attempted after lockout)
- `Insufficient balance.`
- `The transfer couldn't be completed. Please try again in a moment.` (Anchor-side failure)
- `Transfer already submitted.` (you reused an idempotencyKey)

**Idempotency + concurrency (PAY-SYS-01 / PAY-SYS-02):**
- ALWAYS generate a fresh `idempotencyKey` (UUID) when the user lands on the review screen. Reuse it for retries of the same logical submission.
- Server returns the existing transaction if the key matches an earlier call — never double-sends.
- The whole balance-check + Anchor-call + deduct sequence runs in a serializable PostgreSQL transaction with a row lock on the account; concurrent transfers from the same SME account serialize. There's nothing for the FE to do about this — just be aware that a parallel transfer attempt may briefly block instead of failing.
- A transfer in Pending state for **more than 10 minutes** is auto-cancelled by a background sweeper (Status flips to `Failed`, error code `pending_expired`). If your retry lands after this, treat it as a fresh transfer with a new idempotency key.

**Rate limit:** `5 requests / minute / user`. Exceeded → `429`.

**Audit (PAY-SYS-05):** every PIN attempt (success or fail), transfer creation, and transfer completion is written to `paymentAuditLogs`. Account numbers are masked, PIN values are NEVER stored. Transparent to the FE.

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

**Rate limit:** `5 requests / minute / user`. Exceeded → `429`.

### `POST /api/v1/sme/transaction-pin/change`
**Request:** `{ "currentPin": "7392", "newPin": "5837", "confirmNewPin": "5837" }`

**Common failures:** `New PINs do not match.`, `New PIN must be different from the current PIN.`, `Current PIN is incorrect.`, weak-PIN rule.

### `POST /api/v1/sme/transaction-pin/forgot`
**Request:** `{ "password": "...", "newPin": "4628", "confirmNewPin": "4628" }`

Identity proof is the account password (we don't have email/SMS OTP wired yet). Returns success on match.

---

## 9. Beneficiaries

### `GET /api/v1/sme/beneficiaries?search=<text>` (RIV-PAY-001)
Returns saved recipients ordered by most-recently-used. Empty list when the user has none.

**Query params (all optional):**
- `search` — case-insensitive substring match on `accountName` OR `nickname`. Server-side, sanitized before the `LIKE` filter so a malicious string can't be re-rendered from a stored beneficiary. Whitespace-only is treated as no filter.

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

### `POST /api/v1/sme/beneficiaries` (RIV-PAY-002)
**Request:**
```json
{ "accountName": "Alika Mohammed",   // IGNORED — see below
  "accountNumber": "0123456789",
  "bankCode": "000013",
  "bankName": "GTBANK PLC",
  "nickname": "Alika" }
```

**Behavior changed in this batch:**
- **A successful `POST /sme/name-enquiry` for the same `(accountNumber, bankCode)` must have happened within the last 10 minutes for this identity.** If not, returns `"Please look up the account name before saving this recipient."` FE flow: name-enquiry → confirm with user → save. Both in the same screen flow.
- **The saved `accountName` is the bank-verified one** from the most recent lookup — whatever the FE sends in `accountName` is ignored. This prevents a user from saving "John Doe" against an account whose lookup returned a different name.
- **Duplicate save is now an error**, not a silent refresh: `"This recipient is already saved on your account."` Look it up in the list and reuse it. (Pre-existing rows still get their `lastUsedAt` bumped automatically when the recipient is used inside a transfer's `saveBeneficiary: true` path.)
- `nickname` is sanitized (HTML/script stripped) before storage.

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
- **Transfer amount range:** ₦100 – ₦5,000,000 per transaction. Server validates regardless of any DTO-level check.
- **PIN rules:** exactly 4 digits, not sequential, not repeated. Max 3 wrong attempts (user-level, 10-min window) → that transfer reference is locked for 1 hour → reset via `/transaction-pin/forgot`.
- **Pending transfer TTL:** 10 minutes. Pending transfers older than this are auto-cancelled by a background sweeper (every 60s).
- **Idempotency keys:** required for transfers; recommended UUIDs.
- **Free-text sanitization:** narration, nickname, and search are stripped of HTML/script content before storage. Don't try to send rich text or markdown — it'll be flattened.
- **Rate limits (429 on breach):** signup `5/min`, login `10/min`, name-enquiry `10/min`, transfers + PIN endpoints `5/min`.
- **Same-session lookup:** saving a recipient (`POST /sme/beneficiaries`) requires a successful `/sme/name-enquiry` for the same (bank, account) within the previous 10 minutes — the saved name is bank-verified, not FE-supplied.

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
| 2026-06-04 | Password reset trio added under `/identity/passcode/{forgot,verify-otp,reset}` — works for SME + Corporate + Personal; OTP always delivered to email. See [§4 Password reset](#password-reset-added-2026-06-04--covers-sme--corporate--personal). | — |
| 2026-06-04 | `GET /sme/profile` now returns identity fields (`firstName`, `surname`, `fullName`, `email`, `phoneNumber`, `emailVerified`, `phoneVerified`) plus `transactionPinSet`. No more double-call to `/identity/me` for profile/settings screens. See [§5.2](#52-kyb-status--dashboard-banner). | — |
| 2026-06-04 | `POST /identity/passcode/change` added — authenticated Settings → Change password. Same product-aware validation as set/reset. See [§4](#post-apiv1identitypasscodechange). | — |
