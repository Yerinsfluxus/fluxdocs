# SME Mobile App — Design & Frontend Requirements

**Scope:** The new SME mobile app (separate APK / store listing) for sole-proprietor businesses with a CAC business-name registration. Account creation, KYB, transfers (inter and intra), minimal account management. Core banking via Anchor.

This document is in two sections — what the design team produces in Figma, and what the frontend team builds against those designs.

---

## For the design team (Figma)

The SME app is its own product surface. It shares an identity (email + password) with Riverly's Personal app and Corporate web — so a single user can sign in to any of the three with the same credentials — but each app stays in its own product context at runtime. There is no in-app product switching. Inside the SME app, the user is always in SME. The only cross-product touchpoint is an informational list in Settings telling the user "you also have access to Personal / Corporate — here's a link to that app/site".

### Authentication & onboarding flow

Onboarding ends at the dashboard, not at KYB. The user signs up, gives the business basics, and lands on the home screen — KYB upload is something they do from there, on their own pace.

**A1 — Sign up (new identity)**
Single screen capturing full name, email, phone number, password. Sends an OTP to the phone (and email if provided) for verification. After OTP verify, lands the user in the business-info step (A2). This creates a fresh Riverly identity AND immediately enrolls them in the SME product — there's no "pick a product" gate, because they're already in the SME app.

**A2 — Business basics**
Collects: business name (as registered with CAC), RC number (the CAC number for business names), industry (dropdown), business address. One screen, four fields. Save → goes to the dashboard (A8). No KYB upload prompt here.

**A7 — Sign in (existing identity)**
For users who already have a Riverly identity (e.g., they signed up on the Personal app first and are now opening the SME app for the first time). One screen: email/phone + password, then OTP if device is unrecognised.

The SME app handles product context automatically — there is no product picker. After the user authenticates:
- The app calls `/identity/switch-product` with `productType: SME`.
- If the identity has an SME membership → the call succeeds and the user lands on the dashboard (A8). KYB state is reflected via the banner.
- If the identity has no SME membership yet → the call fails with a "no SME on this account" response; the app routes the user through A2 (business basics) → enrollment → dashboard. Personal / Corporate memberships the user may already have are ignored for this client; the SME app only cares about SME.

### KYB upload (post-onboarding, reachable from dashboard or settings)

KYB lives outside the onboarding chain. The screens below are reached either by tapping the dashboard banner or from Settings → "Business verification". The user can leave and come back any time without losing progress.

**A3 — KYB document upload**
The screen for uploading the documents. Required items:
- CAC certificate (file upload — PDF or image)
- CAC status report (file upload)
- Proprietor's government ID — NIN slip / driver's license / passport (file upload)
- Proprietor's BVN (text input — 11 digits)
- Selfie / liveness capture (in-app camera)

Each item is its own step or stacked on one scrollable screen — designer's call. Each accepts retry / re-upload. The user can save and resume — partial uploads persist. The screen has a clearly disabled "Submit for review" button until everything is supplied; once everything is in, the button becomes the way to flip the status to Pending Review.

**A4 — KYB pending review**
Reached after the user taps "Submit for review" on A3 — or by tapping the dashboard banner while review is in progress. Shows: "We're reviewing your business — usually within X hours." Lists what was submitted. Has a "Contact support" affordance.

**A6 — KYB rejected / needs more info**
Reached by tapping the dashboard banner when status is Rejected. Shows which document was rejected and why (free-text from the admin reviewer). Re-upload affordance for that specific item, plus a Submit button that returns the status to Pending Review.

(There is no separate "Approved" screen — when backend flips KYB to Approved, the dashboard banner disappears and the user gains the ability to transact. A push notification announces it.)

### Home & navigation

**A8 — SME dashboard (home)**
Single business account view. Designer needs to handle four KYB states, because the dashboard is where users live during the entire KYB lifecycle:

1. **KYB Not Started** (just signed up, no documents uploaded yet) — show a prominent banner near the top: *"Verify your business to start using your account"* with a "Get started" CTA that opens A3. Below the banner, the balance card shows ₦0.00 with a subtle "Pending verification" label. Quick-action buttons (Send Money, etc.) are visible but disabled / tappable-but-blocked with a tooltip "Complete verification to enable transfers".
2. **KYB Pending Review** (user has submitted documents, awaiting admin) — banner copy changes to *"Documents under review — we'll notify you when it's approved"* with a "View status" CTA that opens A4. Same balance + disabled-actions treatment as above.
3. **KYB Rejected** (admin rejected at least one document) — banner becomes prominent / amber: *"Some documents need your attention"* with a CTA that opens A6. Same disabled-actions treatment.
4. **KYB Approved** — no banner. Full dashboard: balance card with real available + ledger balance, account number with copy button, recent-transactions list, fully-enabled quick-action buttons (Send Money, Receive, View Transactions, Manage Account). Top-right: profile / switcher entry (see SW1).

In all four states the bottom navigation (A9) and top chrome are identical — the variance is only in the banner area and the enabled/disabled state of transfer actions.

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
View business name, RC number, industry, address, proprietor name. Edit address + contact details (other fields are locked because they're tied to KYB). Also contains a "Business verification" entry that opens whichever KYB screen matches the current status — A3 if Not Started or Rejected, A4 if Pending Review. (Approved state shows a checkmark and the submission summary.)

**S2 — Personal info (identity-level)**
View/edit name, email, phone. Change password. Two-factor / device management. Logout. These fields are *identity-level* — changing them changes them across the user's other Riverly products too. The screen needs a small note to that effect.

**S3 — Transaction PIN management**
Change PIN, reset PIN (uses account password as proof). Both are existing backend endpoints.

**S4 — Notifications preferences**
Toggle email / push / SMS per notification category (transfers, security, marketing).

**S5 — Help / support**
FAQ links, contact support (live chat or email), report a problem with a transaction.

### "Your other Riverly products" — informational only, not a switcher

Each Riverly product is its own client surface — Personal app, SME app, Corporate web. Users do not switch products inside the SME app; the SME app is the SME app. The cross-product affordance exists only so the user can be **aware** that the same Riverly identity gives them access to the other products, and so they can **jump out** to those products if they want.

**SW1 — "Your other Riverly products" (inside Settings)**
A settings-level list (not a global top-bar element). Each row represents one of the three Riverly products and shows one of three states:

- **Active here** — the SME row in the SME app. Just labelled "SME (current)".
- **Active elsewhere** — e.g. the user has a Personal membership. Row shows "Personal — Open Personal app" with a deep-link button that tries to launch the installed Personal app and falls back to the Play Store / App Store listing.
- **Not activated** — e.g. the user has no Corporate workspace yet. Row shows "Corporate banking — Visit our web app" with a button that opens the corporate web URL. Copy makes it clear they can sign in there with the **same email and password** they used for SME.

For corporate specifically, when the user does have a corporate membership, the link can include a short-lived identity token so the corporate web app drops them straight into a signed-in session.

This screen does not change the SME app's behaviour or the active token. It is pure navigation + awareness.

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

### Authentication & onboarding

The SME app always operates in SME context. No product picker is ever shown to the user — the app automatically asks the backend to scope the session to SME and routes the user through enrollment if no SME membership exists yet.

- On launch: check for a stored product-scoped JWT (`product: "SME"`). If valid → home (A8). If absent or expired → sign-in screen.
- On sign-in: call `POST /api/v1/identity/login` with `{ identifier, password }`. Then immediately follow up with `POST /api/v1/identity/switch-product` body `{ productType: "SME" }`. Handle:
  - `switch-product` returns 200 with a product JWT → store it, go to A8 (dashboard); the banner reflects current KYB status via `GET /api/v1/sme/profile`.
  - `switch-product` returns "no SME membership" → identity exists but isn't enrolled in SME yet. Keep the identity JWT, route the user through A2 (business basics) which calls `POST /api/v1/sme/enroll`. After enrollment, repeat `switch-product` to get the SME JWT, then go to A8.
  - Login itself fails (bad credentials, locked, suspended) → show the relevant message.
- On sign-up: call `POST /api/v1/identity/signup` with `{ fullname, email, phone, password }`. Then A2 (business basics) → enrollment → `switch-product` → A8 (dashboard).
- Logout: `POST /api/v1/identity/logout`, clear stored tokens, return to sign-in.

The `availableProducts` array on the login response is used **only** by Settings → "Your other Riverly products" (SW1) to surface which other Riverly products this email has access to. It is not used to route the user to a different in-app context.

### KYB lifecycle on the dashboard

The dashboard (A8) drives the entire KYB experience. The frontend should:

- On every dashboard load, call `GET /api/v1/sme/profile` and inspect the `kybStatus` field. Map to one of: `NotStarted` / `PendingReview` / `Rejected` / `Approved`.
- Render the banner + balance card + action-button enablement based on that status (per the four states described under A8 in the design section).
- Tapping the banner opens A3 (if Not Started or Rejected) or A4 (if Pending Review).
- When the user finishes uploading on A3 and taps "Submit for review", call `POST /api/v1/sme/kyb/documents` once per file as they're uploaded, then `POST /api/v1/sme/kyb/submit` to flip status to PendingReview. Update local cache, return user to dashboard.
- Listen for the `kyb.status_changed` push notification — when received, refresh `GET /api/v1/sme/profile` and re-render. This is how the dashboard transitions from PendingReview to Approved or Rejected without the user having to manually refresh.

### Token handling

Two tokens to manage:
- A short-lived **identity JWT** — used only to call `/api/v1/identity/*` endpoints (notably `switch-product`).
- A longer-lived **product JWT** (SME-scoped) — sent on every other request as `Authorization: Bearer <token>`.

Refresh: backend provides a refresh-token flow. On a 401 from any product endpoint, attempt refresh once before forcing re-login.

### KYB endpoints

- `POST /api/v1/identity/products/sme/enroll` accepts the business basics as JSON. Called at the end of A2. Creates the SME profile + membership, sets `kybStatus = NotStarted`. The user lands on the dashboard right after.
- `POST /api/v1/sme/kyb/documents` accepts file uploads as multipart/form-data, one call per file (or one call with all files — backend supports both). Idempotent: re-uploading the same document slot replaces the previous file. Status stays at `NotStarted` while uploads accumulate.
- `POST /api/v1/sme/kyb/submit` flips status from `NotStarted` to `PendingReview`. Fails if any required document is missing.
- `GET /api/v1/sme/profile` returns the current `kybStatus` plus a `kybDocuments` array (which slots are filled, which aren't, any rejection reasons from admin).

### SME endpoints to integrate

| Method | Path | Purpose |
|---|---|---|
| GET | `/api/v1/sme/profile` | Profile + KYB status + document slots |
| PATCH | `/api/v1/sme/profile` | Edit address / contact |
| POST | `/api/v1/sme/kyb/documents` | Upload one KYB document (multipart) — idempotent per slot |
| POST | `/api/v1/sme/kyb/submit` | Submit completed KYB for admin review (NotStarted → PendingReview) |
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
