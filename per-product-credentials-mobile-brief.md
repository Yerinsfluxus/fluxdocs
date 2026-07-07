# Mobile Brief — Per-Product Credentials + Switch-Product Step-Up

**For:** Tolu (Personal + SME apps)
**Backend:** shipped to staging. Prod pending your build.
**Full specs (background only):** `docs/per-product-credentials-plan.md`, `docs/switch-product-step-up-plan.md`.

---

## What changed on the BE and why it matters for you

Before now, one identity had one PIN/password and one phone number shared across all its products. That caused two real bugs:

1. Users with both Personal and SME on the same identity (e.g. Femi) — resetting the Personal 6-digit PIN silently overwrote the SME 8+ char password. SME wallet ended up protected by a PIN.
2. No way for a user to have a Personal phone AND a separate SME business phone on the same account.

The fix: each product membership now has its own credential and its own phone+email. Same identity, separate credentials per product.

**Rollout compat:** the current app you have in the store keeps working for single-product users — no forced update. Multi-product users need the new build.

---

## Contract changes

### 1. `POST /identity/login` — add `productType` to the body

```json
{
  "identifier": "email or phone",
  "password": "the PIN or password",
  "productType": "Personal"   // NEW — always send this
}
```

Always send `productType` matching the app the user is signed into (`"Personal"` for the Personal app, `"Sme"` for SME).

### 2. Login response — three new fields to handle

```json
{
  "accessToken": "...",
  ...
  "credentialSetupRequired": false,
  "credentialFormat": "Pin",         // or "Password"
  "productTypeRequired": false
}
```

Three failure states you must handle (all return `accessToken: ""` or null):

- `productTypeRequired: true` — old flow bug (you should never see this once you're sending `productType`, but handle gracefully with "Please update your app.")
- `credentialSetupRequired: true` — the user has a membership for this product but no credential set for it yet (cross-product enrollment or a multi-product identity that hasn't reset). Response also has `nextStep: "ForgotPasscode"` — route them to the forgot-passcode flow.
- Everything else same as before.

`credentialFormat` tells you which input to render: `"Pin"` = 6-digit numeric keypad, `"Password"` = full keyboard.

### 3. `POST /identity/passcode/set` — add `productType`

```json
{
  "passcode": "123456",
  "confirmPasscode": "123456",
  "productType": "Personal"   // NEW
}
```

Same for `/change` and `/reset` and `/forgot` and `/verify-otp` — always thread `productType` through. Server format-validates strictly by product now (`Personal` → exactly 6 digits, no letters).

### 4. NEW endpoints — per-product phone/email

These are for a signed-in user managing their own product contact **inside settings**, NOT during signup. Signup keeps its existing single-phone flow. Each product manages its own contact independently.

#### Where this lives in the UI

Settings → Profile → **"Manage this account's phone/email"**. Two flavours:

- On the **Personal app**, tapping "Change phone" hits `/products/personal/phone` — Personal's own phone. SME's phone is not touched.
- On the **SME app**, tapping "Change business phone" hits `/products/sme/phone` — SME's business phone. Personal's phone is not touched.

Same idea for email.

#### UI flow

```
Settings → [Change phone number]
        ↓
[Enter new phone number]  ← user types +234...
        ↓
PATCH /identity/products/personal/phone { phoneNumber }
        ↓
[Verify — 6-digit code sent to +234... shown masked]
   OTP dispatched to the NEW phone (pending, not verified yet)
        ↓
[User enters OTP]
        ↓
POST /identity/products/personal/phone/verify { otpCode }
        ↓
[Success — your Personal phone is now +234...]
   Server moves Pending → Verified atomically, uniqueness re-checked.
```

Server-side safety:
- The **verified** phone on the membership is not touched until the OTP is confirmed. Until then, the number lives in a `PendingPhoneNumber` slot; login OTPs and security OTPs continue to go to the OLD verified number.
- If the new number is already claimed by another Riverly account (or by another membership on the same identity), the PATCH returns a plain "This phone number is already in use" error — no OTP dispatched.

#### Why this doesn't affect the identity link

**The two memberships stay linked because they share the same identity row**, not because they share a phone number.

Concretely, when Alice's Personal has phone `+A` and her SME has phone `+B`, both rows on the DB look like:

```
productMemberships:
  { identityId: 0f9a…, productType: Personal, phoneNumber: +A }
  { identityId: 0f9a…, productType: Sme,      phoneNumber: +B }

identities:
  { id: 0f9a…, email: alice@x.com, phoneNumber: +A }  ← identity-level phone
```

Login by any of `alice@x.com`, `+A`, or `+B` finds Alice's identity via the dual-table lookup. The membership rows never disconnect — they always point at the same `identityId`.

#### Signup flow is unchanged

Signup still asks for **one** phone number per product signup. If the user signs up on Personal first with `+A`, and later signs up on SME with `+B`, the server tries to find an existing identity via any of the identifiers they provide:
- If the SME signup uses the same email as Personal → server finds Alice's existing identity, attaches an SME membership, sets SME's phone to `+B`.
- If the SME signup uses a different email AND `+B` isn't already on file anywhere → server creates a **new** identity. Alice ends up with two separate accounts and would need to reach out for a manual merge (rare).
- If `+B` matches an existing identity's or membership's verified phone → server refuses the signup ("This phone number is already in use").

So: for a user who's already Alice-on-Personal and wants to add SME with a different phone, the safe path is **stay logged in on Personal → tap "Add SME" → the existing enrollment endpoint attaches SME to the same identity → user then goes to SME settings and changes the SME phone to `+B`.** This is the flow Femi described.

**Update phone (dispatches OTP to the new number):**
```
PATCH /identity/products/personal/phone
{ "phoneNumber": "+2348xxxxxxxx" }
```
Response: `MaskedPhoneNumber` + (non-prod) `debugOtpCode`.

**Verify phone (moves pending → verified):**
```
POST /identity/products/personal/phone/verify
{ "otpCode": "123456" }
```

**Same shape for email:**
```
PATCH /identity/products/personal/email    { "email": "..." }
POST  /identity/products/personal/email/verify { "otpCode": "..." }
```

Route product is `personal` or `sme`. Corporate is refused (400) — corporate app owns its own contact management.

Uniqueness is enforced: the same phone/email can't already exist on another Riverly account. Server returns a clean error if it does.

### 5. Token scope note

A Personal product-scoped token can only:
- update its own `.../products/personal/*` contact
- set/change its own Personal credential

Same for SME. If your app somehow ends up submitting `productType: Sme` while holding a Personal token, the request is 403 Forbid. You shouldn't hit this in normal flows — it's a defense-in-depth check.

### 6. `POST /identity/switch-product` — step-up when moving between products

**Same-product reissue (Personal → Personal, or SME → SME, from a freshly-logged-in identity token):** unchanged, no password needed. This is the call you make right after login to swap the identity token for a product-scope token. Server checks the JWT `authenticated_product` claim (set only by `/identity/login`) matches the target product and passes through.

**Cross-product switch (Personal → SME, or SME → Personal):** now requires the target product's password in the body:

```json
POST /identity/switch-product
Authorization: Bearer {identity-scope token}

{
  "productType": "Sme",
  "password": "the target product's password"   // NEW — required for cross-product
}
```

Server verifies the password against the target membership's hash. This is the security fix that means a leaked Personal PIN can't be used to reach SME.

**Response states you must handle:**

- `200` with `stepUpRequired: true` → server refused because the body didn't include `password`. Route to a "Enter your {product} password" screen using `credentialFormat` from the response (`"Pin"` = 6-digit numeric keypad, `"Password"` = full keyboard). Include the password on the retry.
- `200` with `credentialSetupRequired: true` + `nextStep: "ForgotPasscode"` → target membership has no password set yet (existing multi-product users on first switch). Route to forgot-passcode for that product; same handler you already use on `/identity/login`.
- `400 Invalid password.` → inline error, allow retry.
- `400 Switching to {product} is temporarily locked. Please try again in 30 minutes.` → 5 wrong passwords in 30 min. Show lockout screen.
- `429 Too many attempts. Please slow down.` → 10 attempts against the same identity in a minute. Toast + slow down.
- `403 Please sign in again to switch products.` → you sent a product-scope token to `/switch-product`. Route back to login. Should never happen in normal flow (you'd only use identity tokens for switch).

---

## What you should build (in priority order)

1. **Send `productType` on every existing call** — this is the single most important change. Login, set, change, forgot, verify-otp, reset. Fixes multi-product users immediately.
2. **Handle `credentialSetupRequired`** — new failure state on login response. Route to forgot-passcode when true.
3. **Handle `productTypeRequired`** — the "please update your app" screen. You shouldn't trigger it on the new build, but it's a safety net.
4. **Read `credentialFormat` from login response** — render the right keypad (PIN vs Password) instead of hardcoding.
5. **Per-product phone management screen** — new screen where an SME user can set a business phone distinct from their Personal phone. Two-step: PATCH → OTP → POST verify.
6. **Per-product email management screen** — same shape.

Points 1–4 fix the bugs. Points 5–6 unlock the new UX Femi asked for.

---

## Test flow to run once your build is ready

Multi-product user (has both Personal and SME):

1. Log into SME with SME password + `productType: "Sme"` → succeeds.
2. Log into Personal with the SME password + `productType: "Personal"` → should fail with generic "Invalid credentials" (Personal credential is separate now).
3. From SME session, enroll Personal (existing flow).
4. `POST /identity/passcode/set` with new Personal PIN + `productType: "Personal"`.
5. Log into Personal with the new PIN + `productType: "Personal"` → succeeds.
6. Reset the Personal PIN → check the SME password still works. This is the exact bug we were fixing.

Then per-product phone:
7. `PATCH /identity/products/sme/phone` with a new number → OTP arrives on that number, not the Personal one.
8. Verify OTP → SME's PhoneNumber is now the business one; Personal's is unchanged.

---

## Questions

Ping me on WhatsApp for anything unclear. Happy to walk through the auth response fields on a call if helpful.
