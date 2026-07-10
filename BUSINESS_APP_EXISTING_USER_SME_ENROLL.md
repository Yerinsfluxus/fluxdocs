# Business (SME) App — Fix the "existing user → SME" dead-end

**For:** Tolu (business app / `riverly_business`)
**From:** Backend
**Type:** Frontend change (business app only). **No backend change needed — the backend already returns everything you need.** Base URL: prod `https://api.riverly.ng`, staging `https://api-staging.riverly.ng`. All routes below are under `/api/v1`.

---

## 1. What's wrong today

When a person who **already has a Riverly account** (they signed up on Personal, or they're a legacy XpressWallet user — same phone/email) opens the **business app** and tries to get an SME account, both entry points dead-end:

- **They tap "Sign up":** the backend replies *"You already have a Riverly account… please log in to add SME."* The app shows that as a plain error toast and **stops** — no way forward.
- **They tap "Log in":** login **succeeds**, then the app immediately calls `switch-product` (productType 1). Because they don't have an SME product yet, that call fails with *"you don't have an Sme product"*. The app shows the message as a toast and **dead-ends on the login screen.**

The endpoint that fixes this — `identity/products/sme/enroll` — exists in your code (`OnboardingService.enroll`) but is **never called**.

**Net:** an existing user is completely locked out of creating an SME account from the business app. A brand-new SME user (new phone/email) is fine — the happy path already works. This fix is only about the "already have an account" case.

---

## 2. The idea (one sentence)

Instead of dead-ending, the app should recognise the two "you already exist / you need to enrol" signals the backend already sends, and route the user into the **SME enrolment flow** (which reuses your existing business-profile / owner / address screens) — then `switch-product` to the dashboard. **The user reuses their existing identity; they just set a separate SME password and do the SME onboarding + KYB.**

---

## 3. The two changes to make

### Change A — On "Sign up", detect the existing account and send them to Login

When you call `POST /identity/signup` and the person already exists, the response comes back with **`status: false`** and a **`data`** object that tells you what to do:

```jsonc
// POST /api/v1/identity/signup   (existing user)
{
  "status": false,
  "message": "You already have a Riverly account with this email. Please log in to add Sme…",
  "data": {
    "existingAccount": true,      // ← read this
    "nextStep": "Login",          // ← and this
    "passcodeSet": true
  }
}
```

**Do:** in your signup handler, **before** showing the error toast, check `data.existingAccount == true` (or `data.nextStep == "Login"`). If so, **don't toast — navigate to the Login screen** with the phone/email pre-filled. (Right now you only read this inside the error/catch path and just print the message.)

### Change B — On Login, detect "needs SME enrolment" and route into the enrol flow

When an existing user logs in on the business app, login **succeeds** but they have no SME membership yet. The login response carries the signal:

```jsonc
// POST /api/v1/identity/login   (existing user, no SME yet)
{
  "status": true,
  "data": {
    "accessToken": "…",           // ← identity token — KEEP IT for the enrol call
    "identityId": "…",
    "nextStep": "EnrollProduct",  // ← read this
    "defaultProduct": 1,          // 1 = Sme
    "phoneVerified": true,
    "emailVerified": true
    // …
  }
}
```

**Do:** after login, **check `data.nextStep`.**
- If `nextStep == "EnrollProduct"` (or `switch-product` returns `nextStep: "EnrollProduct"` — see note) → **route into the SME enrolment flow (Change C)** using the `accessToken` from this login response. **Do NOT** call `switch-product` yet, and do **not** dead-end.
- If `nextStep == "Dashboard"` (they already have SME) → your current flow: `switch-product` → dashboard.

> Note: `POST /identity/switch-product {productType:1}` for a user without SME also returns `status:false` with `data.nextStep == "EnrollProduct"`. So you can trigger the enrol flow from **either** the login response's `nextStep` **or** the `switch-product` failure — reading `nextStep` in both places is the robust option.

### Change C — The SME enrolment flow (reuses your existing screens)

The user is now authenticated with the **identity token** from login. Collect the SME details on your existing onboarding screens, then:

**Step 1 — Create the SME profile + membership (sets a NEW, SME-specific password):**

```jsonc
// POST /api/v1/identity/products/sme/enroll
// Header: Authorization: Bearer <identity token from login>
{
  "businessName": "Acme Ventures",
  "businessRegistrationNumber": "BN1234567",
  "industry": "fashion-apparel",        // slug from GET /reference/industries
  "description": "We sell …",           // optional, 20–500 chars
  "password": "AcmeSme!9",              // NEW SME password (8+ chars) — SEPARATE from their Personal password
  "websiteUrl": null,                    // optional
  "logoUrl": null                        // optional
}
// → 200 { status:true, data:{ identityId, smeProfileId, membershipId, role } }
```

This creates the SME profile + membership with **its own password**. The user's existing (Personal/legacy) password is untouched — this is the per-product-password design.

**Step 2 — Owner role + address** (same endpoints you already use in onboarding, still with the identity token):

```
POST /api/v1/sme/onboarding/owner-role   { role:"Business Owner", ownershipPercentage:100, bvn:"…" }
POST /api/v1/sme/onboarding/address      { operatingAddressLine, operatingCity, operatingState, … }
```

**Step 3 — Switch to the SME product** (now the membership exists, this succeeds):

```jsonc
// POST /api/v1/identity/switch-product
{ "productType": 1 }
// → 200 { status:true, data:{ accessToken, refreshToken, productType:1, profileId, role } }
```

**Store `data.accessToken` (this is the SME product token) and use it for all dashboard calls** — exactly like your existing new-user flow does. Route to the dashboard.

**Step 4 — KYB** happens later from the dashboard banner, exactly as it does for a new SME user (`/sme/kyb/*`). No change there.

---

## 4. Summary of the target flow

```
Existing user opens business app
  │
  ├─ taps "Sign up"  → POST /identity/signup
  │                     └─ data.existingAccount==true / nextStep=="Login"  →  go to Login  (Change A)
  │
  └─ taps "Log in"   → POST /identity/login  (their existing phone + existing passcode)
                        └─ data.nextStep=="EnrollProduct"                  →  enrol flow    (Change B)
                              │  (keep the identity accessToken)
                              ├─ POST /identity/products/sme/enroll  { business info + NEW SME password }
                              ├─ POST /sme/onboarding/owner-role
                              ├─ POST /sme/onboarding/address
                              └─ POST /identity/switch-product { productType:1 }  → SME token → Dashboard
                                    └─ KYB later from dashboard (unchanged)
```

---

## 5. Things to get right (please read)

- **The SME password is NEW and separate.** On the enrol screen, ask the user to create an SME password. Do **not** reuse/assume their Personal password. (Each product has its own password by design.)
- **Keep the token straight:** use the **identity token** from login for steps 1–2 (enrol, owner-role, address), then **swap to the SME product token** from `switch-product` for the dashboard. This is the same pattern as your new-user flow — just entered via login instead of signup.
- **Read `nextStep` / `existingAccount` on `status:false` responses too.** The signup "already exists" reply is `status:false`; your current code only shows its message. You must inspect `data.existingAccount` / `data.nextStep` and route on them.
- **Login is passcode-only for a known device.** The existing user's device may already be trusted (no OTP). If it's a new device, your existing device-verification screen handles it — no new work.
- **Don't call `switch-product` before the membership exists.** For an existing user without SME, call the enrol flow first; `switch-product` is the **last** step.

## 6. Please DON'T change
- The brand-new SME signup happy path (it works).
- The onboarding screens themselves — you're **reusing** them, just reaching them via login for existing users.
- Any KYB flow.

---

## 7. Acceptance criteria (how we'll know it's fixed)

1. **New SME user** (fresh phone/email): signup → OTP → onboarding → dashboard. *(unchanged — must still work)*
2. **Existing user, taps Sign up:** app detects existing account and routes to **Login** (no dead-end toast).
3. **Existing user, logs in:** app detects `EnrollProduct`, walks them through SME enrol (business info + new SME password → owner → address → switch-product) and lands on the **SME dashboard**.
4. **Existing user who already has SME:** login → dashboard directly (unchanged).
5. The existing user's **Personal/legacy login still works** with their original password afterwards (SME password is separate).

Ping backend if any response field doesn't match what's above and we'll confirm against the code. Everything here is already live on staging and prod.
