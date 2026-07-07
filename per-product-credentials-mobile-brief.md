# Mobile Brief — Per-Product Credentials

**For:** Tolu (Personal + SME apps)
**Backend:** shipped. Test against staging as soon as you're ready.
**Full spec:** `docs/per-product-credentials-plan.md` (background only — you don't need to read the whole thing).

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

These replace the identity-level phone/email update. Each product manages its own contact independently.

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
