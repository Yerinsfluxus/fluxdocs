# Frontend & Design Requirements — SME Product + Unified Identity

**Status:** Draft for design + FE review
**Companion to:** [UNIFIED_IDENTITY_AND_SME_PRODUCT_DESIGN.md](./UNIFIED_IDENTITY_AND_SME_PRODUCT_DESIGN.md) (backend design)
**Audience:** Design team, mobile FE, corporate web FE
**Updated:** 2026-05-26

---

## TL;DR for everyone

We're adding a third product (**SME** — for sole-proprietors with a business-name CAC registration) and at the same time refactoring how people sign in so that **one email/phone can hold all three products** (Personal, SME, Corporate) under one identity — like switching between personal and business accounts on Instagram.

To make this real, design needs to produce a handful of new flows, and both frontend apps (mobile + corporate web) need to adopt a slightly different login pattern, plus build out SME screens.

This document is the punch-list of everything design and FE need to do, organised by team.

---

## 1. The mental model designers/FE need to internalise first

Today: **login = open the product directly**. Phone+passcode opens the Personal app, email+password opens the Corporate web. The product is the same thing as the account.

After this change: **login → identity → pick a product**.

1. User logs in with their email or phone + password.
2. Backend returns the identity *and* the list of products this person already belongs to (Personal? SME? Corporate? Some combination?).
3. User picks one and lands in that product's UI (or auto-lands if they only have one).
4. Inside any product, there's an always-available "switch product" affordance — same identity, different context.
5. Anywhere in the UI, "Add Personal/SME/Corporate" is offered — the user fills in only the *extra* KYC that product needs (because the identity-level details are already on file).

This is the only structural change. Every existing screen inside Personal or Corporate works the same way once the user is in that product context — just with one extra chrome element (the switcher) at the top.

---

## 2. For DESIGNERS

### 2.1 New flows to design end-to-end

| # | Flow | Why | Notes |
|---|---|---|---|
| D1 | **Unified login** | Single entrypoint for all three products. Replaces today's separate Personal/Corporate login screens. | One input that accepts email *or* phone, plus a password field. Mobile may keep the 6-digit passcode for ergonomics — see §5 open Qs. |
| D2 | **Product picker (post-login)** | After login, user sees the products they belong to and picks one. Skipped if they only have one. | Compact card list. Each card shows product name, role (for Corporate), and status (verified vs pending KYB). |
| D3 | **In-app product switcher** | Always-visible affordance inside any product to switch to another. Models Instagram's account switcher. | Probably a top-bar element (avatar + dropdown) or a settings-level "switch" item. Needs a design call on chrome placement for both mobile and web. |
| D4 | **"Add Personal / SME / Corporate to your account"** | An existing user adds a new product without re-registering. | Lives in account settings. Shows only the products the user doesn't yet have. Each kicks off the corresponding enrollment flow (E1, E2, E3 below). |
| D5 | **SME onboarding (E1)** | Brand-new user signs up *as* an SME. | 3 steps: identity (name/email/phone/password/OTP) → business basics (name, RC#, industry, address) → KYB docs (CAC cert upload + status report upload). |
| D6 | **SME enrollment for existing user (E2)** | A Personal user adds SME alongside. | Skips the identity step entirely (already done). Starts at business basics. |
| D7 | **Corporate enrollment for existing user (E3)** | A Personal/SME user opens a Corporate workspace. | Heavier than SME — board resolution, multiple UBOs, directors' BVNs (existing corporate KYB flow). |
| D8 | **KYB status & resubmission** | Show "Pending review" / "Verified" / "Rejected — please resubmit X". | One screen with current status, what's missing, and a re-upload affordance. |
| D9 | **SME dashboard** | Landing screen after picking SME. | Reuse the corporate dashboard pattern (balance card, recent transactions, quick actions) but single-account flavour. |
| D10 | **SME transfer flows** | Inter-Riverly + outbound bank transfer. | Visually similar to Personal app's transfer flow. Reuse components where possible. |
| D11 | **Identity / "your account" settings** | Manage the *identity-level* things: email, phone, password, MFA, transaction PIN, devices. | Surfaced inside any product context. Changes here affect all three products. |
| D12 | **Empty / first-run states for SME** | "You haven't enrolled in SME yet — get started" CTA inside settings or on the product picker. | |

### 2.2 Visual / branding decisions design needs to make

These are calls only design can make. List them so they can be tracked:

1. **One brand or three?** Do Personal/SME/Corporate look like one product with different modes, or three sibling products under the Riverly umbrella? Lean toward "one product, three modes" — easier for users, less work to design, matches the Instagram model.
2. **Visual signal for "which product am I in?"** A coloured top-bar accent? A label next to the logo? An icon? Users must never be confused which context they're transacting in — this matters enormously for trust and for preventing wrong-account transfers.
3. **Switcher chrome.** Where it lives on mobile vs web. Most intuitive pattern is top-right account avatar → dropdown listing products → click to switch.
4. **Empty/pending states.** What does an SME with no balance, no transactions, no KYB done yet look like? What's the most encouraging next step?
5. **Cross-product notifications.** If a user is in Personal and a transaction comes in on their SME account, how is that surfaced? Badge on the switcher? Toast that says "View in SME"?
6. **Logout vs "switch user".** With unified identity, "logout" logs out the whole human. There's no separate per-product logout. Design needs to make sure copy reflects this and nobody thinks they're still logged into Corporate after logging out of Personal.

### 2.3 Deliverables we need from design (in priority order)

To unblock frontend, design should ship in this order:

1. Unified login + product picker (D1, D2) — blocks all FE work
2. SME onboarding screens (D5, D6, D7) — blocks SME launch
3. KYB status screen (D8) — blocks SME launch
4. SME dashboard + transfer flows (D9, D10) — blocks SME launch
5. In-app switcher chrome (D3) — needed for first SME user who also has Personal
6. Add-a-product settings (D4) — needed for first existing user who upgrades
7. Identity settings (D11) — can come later; existing per-product settings keep working in the interim

---

## 3. For MOBILE FRONTEND (Personal app + future SME on mobile)

### 3.1 What changes for existing Personal app users

Bare minimum to keep working with the new backend:

- **Update login call.** Today: `POST /api/v1/auth/login` returns a product JWT directly. After backend phase 2: that endpoint will keep working (no break), but we want the app to migrate to the new `POST /api/v1/identity/login` which returns `{ identityToken, availableProducts, defaultProduct }`. If only one product is available (typical for current users), the client immediately calls `POST /api/v1/identity/switch-product` to get the product JWT and proceeds as today. The user sees no difference.
- **Token storage.** Today stores one JWT. After: store both an identity JWT (short-lived, only for `/identity/*` calls) and a product JWT (used for everything else). A small abstraction in the API client makes this transparent.
- **Logout.** Calls a new `/identity/logout` instead of the current `/auth/logout`. Clears both tokens.

That's the entire ask for existing app behaviour. No UI change required to keep working.

### 3.2 New screens / features mobile needs to build (for the SME product on mobile, if applicable)

If SME is going to be reachable from the mobile app (open question in §5), the app needs:

- Product picker screen (post-login when ≥ 2 products) — design D2
- Switcher in the top chrome — design D3
- "Add SME to your account" path in settings → SME enrollment flow (D6)
- SME dashboard (D9)
- SME transfer flow (D10)
- SME transaction PIN setup / verification (reuses existing PIN components, new endpoints — see §6)

If SME is web-only for MVP, mobile only needs §3.1.

### 3.3 New error states mobile needs to handle

- "This identity already has a product membership but it's still pending KYB" — show pending state instead of dashboard
- "Identity locked" (too many failed attempts) — show lockout countdown
- "Product no longer available" (suspended) — graceful fallback, allow switch to another product

---

## 4. For CORPORATE WEB FRONTEND (current `riverly-cooperate-banking` Next.js app)

### 4.1 What needs to change in the existing app

- **Login screen.** Currently calls `POST /api/v1/corporate/login`. Update to call `POST /api/v1/identity/login` and handle the multi-product response. If the user only has Corporate, auto-switch and continue. If multiple, show the product picker.
- **Header / chrome.** Add the switcher (avatar dropdown listing products) — design D3.
- **Settings page.** Add an "Add SME / Add Personal" section — design D4. Each kicks off the relevant enrollment flow.
- **Existing corporate JWT shape changes.** Today's corporate JWT has `org_id`, `role`, `channel: "web"`. After: the new product JWT for Corporate context has the same fields plus `product: "Corporate"` and `profile_id`. Frontend's `getOrganizationId()` etc. keep working.

### 4.2 New screens to build for SME in corporate web

The web app is the natural home for SME because business owners are usually on a laptop. So most of the SME UI lives here:

- SME onboarding (E1 — full flow) and enrollment (E2 — from existing identity)
- KYB document upload (multipart form, CAC cert + status report)
- KYB status screen
- SME dashboard
- SME transfer flows (inter + external) — clone of corporate equivalents, simpler
- SME transactions list + receipt
- SME transaction PIN management
- SME profile / business info editor

### 4.3 Component reuse from corporate

Don't rebuild from scratch. The corporate web already has:

- Auth shell / card components
- File upload widgets (used in corporate KYB)
- Bank selector + name-enquiry components
- Transaction PIN input
- Transfer review/confirmation modal
- Transactions list

All of these should be lifted into shared components and reused for SME. The SME flow is ~70% reuse, 30% new.

---

## 5. Open questions for the team

These need decisions from product / design / engineering before FE starts building:

1. **Is SME accessible from the mobile app, or web-only for MVP?** Decides whether mobile FE has new screens to build or just an SDK update.
2. **Personal login: keep phone+6-digit passcode, or unify to email+password?** Most users on a mobile app are accustomed to passcode/biometric. Most business users on a desktop expect email+password. Easiest: identity supports both — phone+passcode auths a Personal-only identity; email+password auths an identity with any product. But this means the identity model has two credential paths to maintain.
3. **One SME profile per identity, or many?** A solo trader might own multiple side businesses. MVP is 1:1; future could allow many. Affects how the "Add SME" flow is designed (single-shot vs "add another").
4. **Switcher chrome placement.** Top-bar avatar dropdown vs in-page picker vs settings-only? Affects whether users can hop products in one click or have to go through a menu.
5. **Cross-product notifications.** When user is in Personal and SME gets credited, how do we surface it? Influences notification design and the switcher's "unread badge" treatment.
6. **Do we want to design a "first-time identity" flow** separate from "first-time product" flow? Or does signup always create both at once (identity + first product membership)? Backend supports both; design call.
7. **Brand:** Does SME get its own visual identity (different colour, logo lockup), or does it look identical to Personal/Corporate with just a label change?

---

## 6. API contract designers + FE can rely on

These are the endpoints the backend will provide (per the backend design doc). Treat as the source of truth for what FE can call:

### Identity layer

| Method | Path | Returns | Used by |
|---|---|---|---|
| POST | `/api/v1/identity/signup` | `{ identityToken }` | First-time signup, before any product enrollment |
| POST | `/api/v1/identity/login` | `{ identityToken, availableProducts[], defaultProduct }` | Unified login screen |
| POST | `/api/v1/identity/email/verify` | OTP verify | Email verification step |
| POST | `/api/v1/identity/phone/verify` | OTP verify | Phone verification step |
| POST | `/api/v1/identity/switch-product` | `{ accessToken, refreshToken }` | After product picker, or via switcher |
| GET  | `/api/v1/identity/me` | identity + all memberships | "Your account" / settings |
| PATCH| `/api/v1/identity/me` | updated profile | Editing email/phone/name/etc. |
| POST | `/api/v1/identity/logout` | — | Logout |
| POST | `/api/v1/identity/products/sme/enroll` | new SME membership | "Add SME to my account" |
| POST | `/api/v1/identity/products/corporate/enroll` | new Corporate membership | "Add Corporate to my account" |
| POST | `/api/v1/identity/products/personal/enroll` | new Personal membership | "Add Personal" (for users that started as Corporate) |

### SME product (`/api/v1/sme/*`)

| Method | Path |
|---|---|
| POST | `/sme/kyb/documents` (multipart upload) |
| GET  | `/sme/profile` |
| PATCH| `/sme/profile` |
| GET  | `/sme/account`, `/sme/account/balance` |
| POST | `/sme/transfers/internal` |
| POST | `/sme/transfers/external` |
| GET  | `/sme/transactions`, `/sme/transactions/{ref}/receipt` |
| POST | `/sme/transaction-pin/{set,verify,change,forgot}` |

### Backward-compatible (existing endpoints, will keep working)

- `/api/v1/auth/login` (mobile/personal — wraps identity login internally)
- `/api/v1/corporate/login` (corporate web — wraps identity login internally)
- All existing `/api/v1/accounts/*`, `/api/v1/transfers/*`, `/api/v1/corporate/*` endpoints — unchanged

Frontend can migrate at its own pace; nothing breaks if it doesn't migrate immediately.

---

## 7. What we need from each party to unblock backend implementation

| Team | Needs to deliver | Why |
|---|---|---|
| Design | Decisions on §2.2 visual questions, then D1+D2 wireframes | Determines whether identity JWT is short-lived (forced switcher) or long-lived (passive) |
| Mobile FE | Confirm whether SME ships on mobile in MVP | Affects backend webhook routing and mobile-specific endpoints |
| Corporate web FE | Confirm willingness to host SME UI in the existing app | Decides whether SME is a new Next.js app or screens inside the corporate web |
| Product | Final scope for SME MVP (the doc assumes what was specified: account, KYB, transfers, minimal mgmt) | Lock so we don't expand scope mid-build |
| Compliance | OK on cross-product data sharing (a single identity row sees all three products' KYC) | This is the core of the model — needs an explicit OK |

---

