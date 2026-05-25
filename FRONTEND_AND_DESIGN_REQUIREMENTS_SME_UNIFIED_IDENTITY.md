# SME Mobile App — Design & Frontend Requirements

**Scope:** The new SME mobile app (separate APK / store listing) for sole-proprietor businesses with a CAC business-name registration. Account creation, KYB, transfers (inter and intra), minimal account management. Core banking via Anchor.

This document is in two sections — what the design team produces in Figma, and what the frontend team builds against those designs.

---

## For the design team (Figma)

The SME app is its own product surface but shares an identity with Riverly's Personal and Corporate products. The screens below are everything the SME app itself needs. A user who also has a Personal account or a Corporate workspace will be able to jump into those other Riverly apps from the SME app via a switcher — the switcher just *links out* to the Personal app or the Corporate web; it doesn't render their UI inside SME.

### Authentication & onboarding flow

**A1 — Sign up (new identity)**
Single screen capturing full name, email, phone number, password. Sends an OTP to the phone (and email if provided) for verification. After OTP verify, lands the user in the business-info step (A2). This creates a fresh Riverly identity AND immediately enrolls them in the SME product — there's no "pick a product" gate, because they're already in the SME app.

**A2 — Business basics**
Collects: business name (as registered with CAC), RC number (the CAC number for business names), industry (dropdown), business address. One screen, four fields. Save → next.

**A3 — KYB document upload**
Two required, two recommended documents (see KYB section below for rationale on which to include):
- CAC certificate (file upload — PDF or image)
- CAC status report (file upload)
- Proprietor's government ID — NIN slip / driver's license / passport (file upload)
- Proprietor's BVN (text input — 11 digits)

Followed by an in-app selfie capture for liveness/face-match against the ID.

Each item is its own step or stacked on one scrollable screen — designer's call. Each accepts retry / re-upload until "Submit for review".

**A4 — KYB pending review**
After submission, user lands on a status screen. Shows: "We're reviewing your business — usually within X hours." Lists what was submitted. Has a "Contact support" affordance.

**A5 — KYB approved / account ready**
When backend flips KYB to approved, the user is bumped into the home dashboard (A8). Push notification at the same time.

**A6 — KYB rejected / needs more info**
Shows which document was rejected and why (free-text from the admin reviewer). Re-upload affordance for that specific item, plus a Submit button.

**A7 — Sign in (existing identity)**
For users who already have a Riverly identity (e.g., they signed up on the Personal app first). One screen: email/phone + password, then OTP if device is unrecognised. If they don't yet have SME enrolled on this identity, route them through A2 → A3 → A4. If they already have SME and it's approved, go to A8. If approved but no transaction PIN set yet, go to T1.

### Home & navigation

**A8 — SME dashboard (home)**
Single business account view. Top: account balance (large), available vs ledger balance, account number with copy button. Below: a recent-transactions list (last 5–10), and quick action buttons — Send Money, Receive (shows account number / share sheet), View Transactions, Manage Account. Top-right: profile / switcher entry (see SW1).

**A9 — Bottom navigation**
Four tabs: Home, Transactions, Profile, More. (Designer can decide on exact labels — "Wallet" might fit better than "Home", etc.)

### Transfer flows

**T1 — Transaction PIN setup (first-time, before any transfer)**
4-digit PIN, then confirm. Same rules as personal app — no sequential (1234), no repeating (1111). Mandatory before first transfer; can be deferred until then.

**T2 — Send money (entry)**
Two options: "To another Riverly account" (intra) or "To another bank" (inter). Designer's call on whether this is a sheet, two cards, or a tab.

**T3 — Intra-bank transfer**
Recipient input by Riverly account number. After entry, screen shows the resolved account holder name. Then amount input, narration (optional), and a "Continue" button to T6 (review).

**T4 — Inter-bank transfer**
Bank picker (searchable list — comes from Xpress Wallet's bank list endpoint), account number input, name-enquiry happens automatically when both bank + 10-digit account number are valid, shows resolved name, then amount + narration, "Continue" to T6.

**T5 — Saved beneficiaries (optional for MVP)**
List of previously transferred-to accounts. Tap one to skip to the amount step. "Save beneficiary" checkbox on T6. Can ship in v1.1 if MVP is tight.

**T6 — Review & confirm**
Summary of all transfer details (source, destination name + bank + account, amount, fee, total, narration). Bottom: a PIN-entry field. Pressing the action button submits with the PIN.

**T7 — Success / failure result**
Success: green check, animated, "Transaction successful" with reference, amount, recipient. Buttons: "Download receipt", "Make another transfer", "Done".
Failure: red, error message from backend (always plain English by the time it reaches the screen — backend already gives clean messages), "Try again" and "Contact support".

**T8 — Receipt screen / download**
PDF preview with all the transfer details. Share / download / done.

### Transactions

**T9 — Transactions list**
Reverse-chronological. Each row: counterparty, amount (with credit/debit color), date, status. Filter button opens T10. Search by reference or counterparty name.

**T10 — Transactions filter**
Date range, type (credit/debit), status (pending/completed/failed), amount range. Apply / clear.

**T11 — Transaction detail**
Full record: amount, fee, total, source, destination, narration, reference, status, timestamps. "Download receipt" + "Report a problem" buttons.

### Profile, settings, security

**S1 — Profile / business info**
View business name, RC number, industry, address, proprietor name. Edit address + contact details (other fields are locked because they're tied to KYB).

**S2 — Personal info (identity-level)**
View/edit name, email, phone. Change password. Two-factor / device management. Logout. These fields are *identity-level* — changing them changes them across the user's other Riverly products too. The screen needs a small note to that effect.

**S3 — Transaction PIN management**
Change PIN, reset PIN (uses account password as proof). Both are existing backend endpoints.

**S4 — Notifications preferences**
Toggle email / push / SMS per notification category (transfers, security, marketing).

**S5 — Help / support**
FAQ links, contact support (live chat or email), report a problem with a transaction.

### Cross-product switcher

**SW1 — Switcher (top-right of every screen)**
Tapping the avatar / icon opens a sheet showing:
- The current identity (name, email)
- A list of *other Riverly products this identity has access to* — for SME users, this might show "Personal — open" and/or "Corporate — open in browser"
- "Sign out of Riverly"

Tapping "Personal — open" tries to deep-link into the installed Personal app (and offers the Play Store link if not installed). Tapping "Corporate" opens the corporate web URL with a one-time auth token so the user lands signed in.

For an SME user who only has the SME product, the sheet just shows their identity and the logout button — no other products listed.

### Empty, loading, error states

Designer should produce explicit states for:
- Empty transactions list (first-time user, no activity)
- KYB pending (account not yet usable for transfers — show a banner on dashboard)
- KYB rejected (banner + CTA to fix)
- Network error / no connection
- Outage / system-wide issue
- Insufficient balance on a transfer
- Daily limit reached

### KYB documents — what to actually require

Riverly is a regulated NG fintech, so the minimum that satisfies CBN (and most Anchor-backed integrations) for a business-name account is:

| Document | Required? | Why |
|---|---|---|
| CAC certificate (business name) | Yes | Proves the business exists |
| CAC status report | Yes | Proves it's currently active |
| Proprietor's BVN | Yes | Regulatory — no Nigerian bank account can be opened without a BVN linked |
| Proprietor's government ID (NIN slip / driver's licence / passport) | Yes | Identity verification |
| Selfie / liveness check | Yes | Fraud / impersonation defence — matches face to ID |
| Utility bill (address proof) | No (for MVP) | Only needed if you want higher transaction limits; skip for v1 |

Other Nigerian SME fintechs (Sparkle, Brass, Prospa) all require at minimum BVN + CAC + ID + selfie. Going below that is unusually permissive and would raise eyebrows during the Anchor compliance review.

---

## For the frontend team

The SME mobile app is a brand-new app. It signs an identity in via Riverly's unified identity endpoints, enrolls them in the SME product, and from then on operates against the `/api/v1/sme/*` endpoints. All transfer / banking work flows through Anchor on the backend; frontend just calls our endpoints.

### Authentication

- On launch: check for a stored product-scoped JWT (`product: "SME"`). If valid → home. If absent or expired → sign-in screen.
- On sign-in: call `POST /api/v1/identity/login` with `{ identifier, password }`. Handle:
  - Success with SME membership → call `POST /api/v1/identity/switch-product` with `{ productType: "SME" }` to get the product JWT. Store it.
  - Success but no SME membership yet → push the user into the SME enrollment flow (business basics → KYB upload), which calls `POST /api/v1/identity/products/sme/enroll`.
  - Device verification required → show OTP screen, post to `POST /api/v1/auth/login/verify-device` (existing endpoint, reused).
  - Locked / suspended → show the relevant message and a "contact support" CTA.
- On sign-up: call `POST /api/v1/identity/signup` with `{ fullname, email, phone, password }`. Email + phone OTP verification using existing OTP endpoints. Then route into business basics → KYB.
- Logout: `POST /api/v1/identity/logout`, clear stored tokens, return to sign-in.

### Token handling

Two tokens to manage:
- A short-lived **identity JWT** — used only to call `/api/v1/identity/*` endpoints (notably `switch-product`).
- A longer-lived **product JWT** (SME-scoped) — sent on every other request as `Authorization: Bearer <token>`.

Refresh: backend provides a refresh-token flow. On a 401 from any product endpoint, attempt refresh once before forcing re-login.

### KYB upload

- `POST /api/v1/identity/products/sme/enroll` accepts the business basics as JSON.
- `POST /api/v1/sme/kyb/documents` accepts the document uploads as multipart/form-data. Frontend uploads each file as it's selected so the user doesn't lose progress mid-flow.
- Poll `GET /api/v1/sme/profile` (or subscribe to push notifications) to detect the moment KYB status flips to `Approved` and bump the user into the dashboard.

### SME endpoints to integrate

| Method | Path | Purpose |
|---|---|---|
| GET | `/api/v1/sme/profile` | Profile + KYB status |
| PATCH | `/api/v1/sme/profile` | Edit address / contact |
| GET | `/api/v1/sme/account` | Bank account details (account number, etc.) |
| GET | `/api/v1/sme/account/balance` | Balance refresh |
| POST | `/api/v1/sme/transaction-pin/set` | First-time PIN |
| POST | `/api/v1/sme/transaction-pin/verify` | Verify PIN before a transfer |
| POST | `/api/v1/sme/transaction-pin/change` | Change PIN |
| POST | `/api/v1/sme/transaction-pin/forgot` | Reset using account password |
| GET | `/api/v1/sme/banks` | Bank list (Xpress Wallet) — for inter-bank transfer recipient |
| GET | `/api/v1/sme/beneficiaries/validate?bankCode=&accountNumber=` | Name enquiry |
| POST | `/api/v1/sme/transfers/internal` | Intra-Riverly transfer |
| POST | `/api/v1/sme/transfers/external` | Outbound bank transfer |
| GET | `/api/v1/sme/transactions` | List with filters |
| GET | `/api/v1/sme/transactions/{reference}` | Single transaction |
| GET | `/api/v1/sme/transactions/{reference}/receipt` | Receipt PDF |

### Transfer flows — sequencing

A typical external transfer from the frontend's perspective:

1. User picks bank + enters 10-digit account → call `GET /sme/beneficiaries/validate` → show resolved name.
2. User enters amount + narration → call `GET /sme/account/balance` to confirm sufficient funds → if not, block and show "Insufficient balance".
3. User taps Continue → show review screen.
4. User enters PIN → call `POST /sme/transfers/external` with `{ bankCode, accountNumber, accountName, amount, narration, transactionPin, idempotencyKey }`.
5. On 200 → success screen with the returned reference; offer to download receipt.
6. On 4xx → show the backend's error message verbatim (backend is already producing clean messages — never show "Something went wrong" to the user).

Intra is the same shape but uses `/sme/transfers/internal` and the recipient lookup uses an internal-lookup endpoint instead of bank+account.

**Idempotency:** every transfer endpoint requires an `idempotencyKey` (UUID). Generate one when the user lands on the review screen and reuse it for any retries (so a network-blip retry doesn't cause a double-send).

### Cross-product switcher implementation

The switcher is *not* in-app product rendering — it deep-links to the other Riverly apps:
- Personal: deep link to `riverly-personal://open` (or whatever the URL scheme is) with the identity JWT; fall back to the Play Store / App Store listing if not installed.
- Corporate: open the corporate web URL with `?identityToken=<jwt>` so the corporate web app can auto-sign-in.

`GET /api/v1/identity/me` returns the list of products this identity has — frontend uses that response to decide which switcher items to show.

### Push notifications

Wire up Firebase Cloud Messaging (or equivalent) to receive backend-triggered events:
- KYB status change (approved / rejected)
- Incoming transfer (account credited)
- Outgoing transfer settled / failed
- Security alert (new device sign-in)

Tapping a notification deep-links into the relevant screen — transaction detail for transaction events, KYB status for KYB events.

### Error handling pattern

Every API call should treat:
- `401` → attempt token refresh once, then force re-login
- `403` → show "you don't have permission" generic error
- `4xx` with a body containing `message` → display that message in a toast / inline error
- `5xx` → "Something went wrong on our side. Please try again." with a retry button
- Network failure → "Check your connection" with retry

Backend now consistently produces clean, user-displayable messages on 4xx responses — frontend can display them directly without translation.

### What the frontend does NOT need to handle

These are backend-owned and don't require any frontend work:
- KYB document review (admin tool, separate from this app)
- BVN verification call to Dojah
- Account creation at Anchor (happens server-side once KYB is approved)
- Webhook from Xpress Wallet when funds land (also server-side; frontend just sees the balance update via polling or push)
- Anti-fraud / limit checks (backend enforces; frontend just shows the error)
