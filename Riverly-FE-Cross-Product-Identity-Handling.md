# Riverly Frontend — Cross-Product Identity Handling

## Why this matters

Riverly uses a unified identity model: one user identity can have one or
many products (Personal, SME, Corporate). A user who has Corporate
should be able to add SME (or Personal) to the same account without
creating a separate one — same email, same phone, same password.

This document tells the FE how to handle the cases where a user lands
in one of these cross-product situations:

- They try to sign up on SME with an email they already used for Corporate.
- They log in successfully but don't have the product they're trying to use.
- They have multiple products and need to switch between them.

The backend now returns structured routing hints for all of these.
Old FE that ignores the new fields still gets a clear error message,
but the recommended FE approach is to read the structured fields and
route the user automatically — that's what removes the dead-ends.

---

## What changed on the BE

Two endpoints now return structured failure responses that the FE can
route on:

- `POST /api/v1/identity/signup` — when the identifier matches an
  existing account that already has product memberships.
- `POST /api/v1/identity/switch-product` — when the authenticated
  identity has no membership for the requested product.

Both still return HTTP 400 + the existing `message` field so old FE
keeps showing the message. The new piece is `data` populated with a
`nextStep` value plus useful context.

---

## Case 1 — Signup with an existing email

**Scenario.** User opens the SME app and signs up with
`yerinsabram@gmail.com`. That email already has a Riverly account
(they used it for Corporate or Personal previously, possibly
forgotten).

**Request:**

```
POST /api/v1/identity/signup
{
  "firstName": "Yerins",
  "surname":   "Abraham",
  "email":     "yerinsabram@gmail.com",
  "phoneNumber": "+2349018604316",
  "productType": "Sme"
}
```

**New response (HTTP 400):**

```json
{
  "status": false,
  "message": "You already have a Riverly account with this email. Please log in to add Sme to your account.",
  "data": {
    "accessToken": "",
    "refreshToken": null,
    "identityId": "40453a2f-...",
    "firstName": "Yerins",
    "surname": "Abraham",
    "fullName": "Yerins Abraham",
    "email": "yerinsabram@gmail.com",
    "phoneNumber": null,
    "emailVerified": true,
    "phoneVerified": false,
    "onboardingCompleted": false,
    "availableProducts": [],
    "defaultProduct": null,
    "onboardingRequired": false,
    "debugOtpCode": null,
    "passcodeSet": true,
    "nextStep": "Login",
    "existingAccount": true
  }
}
```

**What the FE should do.**

1. Detect `data.existingAccount === true` (or read `data.nextStep === "Login"`).
2. Route the user to the **Login screen**.
3. Pre-fill the email field with `data.email` (or `data.phoneNumber`) so the user doesn't have to retype.
4. Show a short banner above the login form: *"It looks like you already have a Riverly account. Sign in to add SME to your account."*
5. After successful login, proceed to **Case 2** (no SME product yet).

---

## Case 2 — Login succeeds, but no SME product

**Scenario.** User logs in with `/identity/login`. They have a
Corporate membership but no SME. They try to switch into SME and the
BE rejects.

**Recommended flow.**

After `/identity/login` succeeds, the FE has an identity-scoped JWT.
Before calling `/switch-product`, inspect `data.availableProducts`. If
the user's intended product isn't there, route them straight to
enrollment for that product:

```
POST /api/v1/identity/products/sme/enroll
```

This creates the SmeProfile + ProductMembership using minimum required
business info (collected over the next few onboarding screens).

After enrollment completes, then call `/switch-product` and the user is in.

**Alternative — fail-then-route:**

If the FE prefers to keep its current "always call switch-product
after login" flow, the new BE response tells it what to do:

```
POST /api/v1/identity/switch-product
{ "productType": "Sme" }
```

```json
{
  "status": false,
  "message": "You don't have a Sme account yet. Please complete enrollment first.",
  "data": {
    "accessToken": null,
    "refreshToken": null,
    "identityId": "40453a2f-...",
    "productType": "Sme",
    "profileId": null,
    "role": null,
    "nextStep": "EnrollProduct"
  }
}
```

**What the FE should do.**

1. Detect `data.nextStep === "EnrollProduct"`.
2. Route the user to the SME enrollment screen.
3. After enrollment, call `/switch-product` again — this time it will
   succeed and return the product-scoped JWT.

---

## Case 3 — User has multiple products

**Scenario.** A user has both SME and Personal. They open the SME app —
the FE always wants to land them in SME.

**Recommended.** After login, just call:

```
POST /api/v1/identity/switch-product
{ "productType": "Sme" }
```

`data.availableProducts` from the login response tells the FE everything
the user has. If the SME app sees both Sme and Personal in the list, it
silently picks Sme and switches — no UI prompt needed.

---

## Quick reference — `nextStep` values

The BE uses one consistent vocabulary across all responses that carry
`nextStep`. Today's values:

| Value           | When                                                | What FE does                                     |
|-----------------|-----------------------------------------------------|--------------------------------------------------|
| `VerifyPhone`   | Onboarding — phone OTP not yet verified             | Route to phone OTP screen                        |
| `AddEmail`      | Personal flow — no email on file                    | Route to "add email" screen                      |
| `VerifyEmail`   | Email present but not yet verified                  | Route to email OTP screen                        |
| `SetPasscode`   | Contacts done, no passcode yet                      | Route to "create passcode" screen                |
| `EnrollProduct` | Identity exists, no membership for requested product| Route to product enrollment screen               |
| `Login`         | Signup matched existing identity                    | Route to login screen, pre-fill identifier       |
| `Dashboard`     | Fully onboarded                                     | Route to product dashboard                       |

---

## Future-proofing recommendation

When the FE reads any auth-style response (signup, login, verify, switch-product), the safest pattern is:

1. Check `status` for success/failure.
2. If success: use `data.accessToken` (or product-scoped variant).
3. If failure: check `data.nextStep` first. If present, route there.
4. Fall back to showing `message` only if no `nextStep` is present.

Doing this means future cross-product flows we add on the BE will route
correctly without further mobile rebuilds — the BE just emits a new
`nextStep` and the FE handles it.

---

## Open question for the team

Today Personal uses **phone + 6-digit passcode** (legacy `/auth/login`)
while SME/Corporate use **email + password** (`/identity/login`). A
user who installs both apps will see different login screens for the
same Riverly account.

We're keeping it this way for now (the shipped Personal APK depends on
the legacy flow), but worth discussing whether the next Personal build
should migrate to `/identity/login` so the credential is genuinely
unified across all products.
