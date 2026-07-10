# Riverly Unified Identity — Architecture & Security Rationale

**Purpose:** Explain *what* the unified identity model is, *why* it was chosen over "each product its own separate login," and *what security controls* keep it tight. Written to be presented to the team, and to answer a security/ISO auditor's questions.

**One-line summary:** Riverly uses **one verified customer identity per human**, under which a person can hold **separate, independently-secured product accounts** (Personal, SME, Corporate). Each product keeps its **own password and its own scoped access** — so we get the isolation of separate logins *plus* the consistency, single-customer-view, and compliance benefits of a unified identity. It is the model modern banks and fintechs use.

---

## Contents

1. [The two models, in plain terms](#1-the-two-models-in-plain-terms)
2. [Why unified is the right call (industry standard)](#2-why-unified-is-the-right-call-and-the-industry-standard)
3. [The architecture (what actually exists)](#3-the-architecture-what-actually-exists)
4. [Objections answered](#4-objections-answered)
    - [4.1 Different legal entities (person vs company)](#41-different-legal-entities-person-vs-company)
    - [4.2 A shared identity, one leaked credential (blast radius)](#42-a-shared-identity-one-leaked-credential-blast-radius)
5. [Security controls (defense in depth)](#5-security-controls-defense-in-depth)
6. [Loopholes we found and closed](#6-loopholes-we-found-and-closed-be-proud-of-this--it-shows-rigor)
7. [Honest trade-offs](#7-honest-trade-offs-and-why-theyre-acceptable)
8. [Q&A — likely questions, with answers](#8-qa--likely-auditor--colleague-questions-with-answers)
9. [Glossary](#9-glossary)

---

## 1. The two models, in plain terms

**Model A — Silos ("each product its own login"):** Personal, SME, and Corporate are three separate systems. The same human signs up three times, verifies their identity (KYC/BVN) three times, and has three unrelated logins. Each silo stores its own copy of the customer.

**Model B — Unified identity (what we built):** There is **one Identity** per human (keyed by verified phone/email). Under that identity, the person can enrol into one or more **products**. Each product has its **own password** and its own profile/account. Logging in authenticates the *identity*; accessing a specific product requires proving *that product's* password.

```
              MODEL A (silos)                         MODEL B (unified — Riverly)
   ┌─────────┐ ┌─────────┐ ┌─────────┐          ┌─────────────── Identity ───────────────┐
   │Personal │ │  SME    │ │Corporate│          │  (one human: verified phone + email)   │
   │  login  │ │ login   │ │ login   │          │                                        │
   │  + KYC  │ │ + KYC   │ │ + KYC   │          │   ┌────────────┬───────────┬─────────┐ │
   │  + data │ │ + data  │ │ + data  │          │   │ Personal   │  SME      │Corporate│ │
   └─────────┘ └─────────┘ └─────────┘          │   │ membership │ membership│ member. │ │
   same person stored 3×,                       │   │ own passwd │ own passwd│ own pwd │ │
   3 logins, 3 KYCs, 3 silos                    │   └────────────┴───────────┴─────────┘ │
                                                └────────────────────────────────────────┘
                                                one person, one KYC, per-product passwords
```

**Crucial point for the skeptic:** unified does **not** mean "one shared password that unlocks everything." Each product still has its **own** credential and its **own** access boundary. We kept everything that makes silos feel safe, and removed what makes silos fragile.

---

## 2. Why unified is the right call (and the industry standard)

Modern banks and fintechs converged on "one customer identity, many products" for concrete reasons:

1. **Single source of truth / single customer view.** One verified record of the human. In silos, the *same person's* name, phone, KYC status can drift out of sync across three databases — a data-integrity and compliance problem. Regulators (AML/KYC, "know your customer") increasingly *expect* a single customer view.
2. **KYC once, reuse safely.** The customer verifies their identity once. Silos force re-verification per product — worse UX and, more importantly, **more copies of sensitive PII (BVN, DOB, address) = a larger breach surface.** Fewer copies of PII is a security win, not a loss.
3. **Consistent security posture.** Auth logic (hashing, lockout, rate-limiting, enumeration protection, token rules) lives in **one** place. Fix a vulnerability once and every product benefits. In silos you maintain three auth stacks; a fix or a policy (e.g., "lock account after 5 tries") has to be re-implemented three times and *will* drift — one silo inevitably lags and becomes the weak link.
4. **Less duplicated code = fewer bugs.** Three copies of login/OTP/reset logic is three times the attack surface and three times the places to get it wrong.
5. **Future products & cross-sell.** Adding a 4th product is enrolment under the existing identity, not a new silo. It's a foundation, not a rebuild.
6. **Cleaner offboarding / fraud response.** Suspend or investigate *the person* once, across products — instead of chasing three disconnected accounts.

**Bottom line to say out loud:** *"Silos give you isolation but duplicate the customer, the PII, and the security code — which is actually more risk and more drift. We kept the isolation (per-product passwords + scoped access) and added a single verified identity, so we're both safer and compliant with single-customer-view expectations."*

---

## 3. The architecture (what actually exists)

- **`Identity`** — one row per human. Holds the verified `Email` / `PhoneNumber` (both enforced unique). This is the anchor.
- **`ProductMembership`** — links an identity to a product (`Personal=0`, `Sme=1`, `Corporate=2`). Each membership carries **its own credential** (`PasswordHash` + `CredentialFormat`), its own `ProfileId`, `Role`, and optionally its own verified contact. **A person can have 1, 2, or 3 memberships — each with a different password.**
- **Two-token model:**
  - **Login → identity token** (short-lived, ~15 min, claim `scope=identity`). Proves *"this is the human."* It is deliberately weak: it cannot directly touch a product's money data (see §4).
  - **Switch-product → product-scoped token** (~4 hr, claims `product`, `profile_id`, `role`). Proves *"this human has proven the password for THIS product."* This is what the product's endpoints require.
- **Step-up on switch-product.** To get a product token you must present **that product's** password (unless the identity token already proved that same product in this session). This is a genuine step-up authentication, rate-limited and lockout-protected per target product.

So a request to move SME money must be backed by a token that proves the **SME** password — not merely that "you logged in somewhere."

---

## 4. Objections answered

### 4.1 Different legal entities (person vs company)

> *"The three products track different legal entities — Personal is an individual, SME/Corporate are companies. The business is legally separate from the individual. So why collect them under one identity?"*

This is the sharpest objection, and the answer is a distinction professional banking systems make explicitly: **the Identity is the login of the human who OPERATES the account — not the account holder.** There are two separate layers:

| Layer | Answers | Personal | SME / Corporate |
|---|---|---|---|
| **Identity** (authentication) | *Who is logging in?* | the individual | the human who operates the account (owner / director / authorised user) |
| **Product / Profile** (account holder of record) | *Whose account & money?* | the individual | **the company** — separate KYB, separate profile, separate money, separate password |

We do **NOT** merge the legal entities. The individual's personal account and the company's business account stay **separate account holders, separately verified (KYC vs KYB), with separate money, data, and passwords.** The only shared thing is the **front door** — authenticating the human who operates them.

Three points that make this airtight:

1. **Regulation *requires* a human behind every business account.** AML/KYC mandates identifying the **beneficial owner** and **controlling persons** of a company account. Our SME KYB **already collects the owner's BVN, date of birth, and personal address** (the owner-personal step). So a verified human always stands behind a business account — by law. Unifying that human's login reflects the compliance reality; it is not an invented link.

2. **This is exactly how real business banking works.** You (a person) log into your business bank with **your own** credentials to operate your **company's** account. The bank KYC'd you *and* KYB'd the company — it did not declare you and the company one legal entity. A director of two companies logs in once and switches between the two companies' accounts. The login is the operator's; the accounts stay legally separate.

3. **The account is not "owned" by one login.** A business account can have **multiple** human operators with roles (Owner / Admin / Maker / Checker / Viewer) — Corporate already runs maker-checker with several users on one organisation. That is only possible *because* we separate the operator (Identity) from the account (Profile / Organisation). Silos, which fuse "the login" with "the account," can't cleanly model "several people operate one company account."

So we are **not** connecting three unrelated businesses under one umbrella. We give one human **one authenticated front door**, behind which sit **separate, isolated, legally-distinct accounts** (own KYC/KYB, money, and password) that this human is authorised to operate. The legal separation the team rightly cares about is fully preserved — it lives in the Product/Profile layer, not in the login.

**One-liner to say in the room:** *"The identity is the login of the person operating the account, not the account holder. The company is still the customer — KYB'd and isolated on its own. We just don't force the same human to log in three times to reach the three accounts they legally operate. Regulation already requires us to know the human behind the business, so that identity always exists anyway."*

### 4.2 A shared identity, one leaked credential (blast radius)

> *"With separate logins, each product is secured independently. A shared identity means if one credential leaks, everything is exposed. Isn't that a bigger blast radius?"*

**No — and here's the precise reason.** There is **no single shared credential.** Each product has its **own password**, and each product's endpoints require a **product-scoped token** that can only be minted by proving **that product's** password (`switch-product` step-up). Concretely:

- Compromising a user's **Personal** password lets an attacker into **Personal only.** It does **not** grant SME or Corporate access, because those require the SME/Corporate password.
- We **enforce this at the resource layer**: SME money/data endpoints (`/sme/account`, `/sme/transactions`, `/sme/transfers`, PIN, beneficiaries, …) reject any token that isn't a genuine `product=Sme` token. A Personal or bare-identity token gets a `403`. *(This is a control we specifically hardened — see §6.)*
- Money movement has an **additional** factor on top of that: a per-product **transaction PIN**.

So the **blast radius is contained per product — exactly like silos** — but with **one** consistently-secured, auditable codebase instead of three that drift. You get silo-level isolation *and* unified-level consistency. That is the whole point, and it is your strongest sentence in the room.

---

## 5. Security controls (defense in depth)

Every control below is implemented in the shared auth layer, so it applies uniformly to all products.

| Control | What it does |
|---|---|
| **Per-product credentials** | Each `ProductMembership` has its own bcrypt password hash + `CredentialFormat`. No cross-product credential reuse. |
| **Password hashing** | **bcrypt** everywhere (login, switch-product, reset, biometric, refresh-token-at-rest). No plaintext, no fast hashes. |
| **Product-scoped tokens + step-up** | Identity token can't touch product money data; a product token requires proving that product's password. |
| **Resource-layer product guard** | SME financial endpoints require a `product=Sme` claim — closes cross-product access with a wrong-product/identity token. |
| **Transaction PIN** | Separate per-product PIN gates actual money movement, independent of login. |
| **Account lockout** | 5 failed attempts → 30-minute lock; status checked *before* the password is even tested (a locked account can't be probed). |
| **Per-IP rate limiting** | Login/OTP/PIN/lookup endpoints are throttled **per source IP** (real client IP via `X-Forwarded-For`) — brute-force and one-client-DoS resistance. |
| **User-enumeration protection** | Unknown user and wrong password return the *same* generic error; password-reset always returns a generic "if an account exists…" and never echoes the code in production. |
| **Short token lifetimes** | Identity token ~15 min, product token ~4 hr (deliberately cut from a longer window during hardening). |
| **Refresh tokens** | Random opaque tokens, **bcrypt-hashed at rest**, single-use **rotation** on refresh, and global **revocation** via logout-all. |
| **Device verification** | Login from a new device triggers a one-time code to the account's verified number before the device is trusted. |
| **Uniqueness enforcement** | DB-level unique indexes on identity + per-membership email/phone, plus app-level "claimed elsewhere" checks at contact-verify time. |
| **Ownership gate before enrolment** | Adding a product to an identity requires proving an existing credential (login) or a verified OTP — a stranger who re-uses your email at signup cannot enrol into your identity. |
| **Admin token revocation** | Admin sessions re-validate against the DB (status + token version) on every request. |
| **Append-only audit trail** | Payment/auth-relevant events are recorded for forensics; account numbers masked, PINs never stored. |
| **Encryption** | TLS in transit; sensitive fields (e.g., BVN) encrypted at rest; secrets in AWS Secrets Manager (KMS), injected at runtime, never in source. |

---

## 6. Loopholes we found and closed (be proud of this — it shows rigor)

A full internal audit of the unified + legacy auth surface was run; findings were fixed:

1. **Multi-product login dead-end** — a user with more than one product could hit a "which product?" wall. **Fixed:** login now auto-resolves the product by matching the entered password to a membership.
2. **Session-renewal bug (hard logout).** The apps' token refresh hit a legacy endpoint that didn't recognise unified tokens, silently logging users out every few hours. **Fixed** backend-side (renewal now accepts unified tokens and reissues the correct product-scoped token). No user action.
3. **Global rate-limiters.** Login/OTP throttles were one shared bucket (one client could exhaust everyone's budget; no per-IP brute-force limit). **Fixed:** partitioned per client IP.
4. **Cross-product access at the resource layer.** SME endpoints authorised on identity id without checking the product claim, so a Personal/identity token could read SME data. **Fixed:** SME money endpoints now require a `product=Sme` token.
5. **Legacy coexistence.** Existing (XpressWallet) users must not be locked out or forced to re-onboard. **Handled:** on login, a legacy user is migrated/linked into an identity with their onboarded state preserved (password still verified first) — so they reach their dashboard, not a re-onboarding loop.

*(Remaining, lower-priority hardening items are tracked and scheduled — being able to name them is itself a sign of a controlled process, which auditors reward.)*

---

## 7. Honest trade-offs (and why they're acceptable)

- **A shared identity requires strong isolation between products.** We answered this with per-product passwords + product-scoped step-up tokens + a resource-layer product guard. Isolation is enforced, not assumed.
- **Single identity store is a higher-value target.** True of any single-customer-view system. Mitigated by: fewer PII copies overall, encryption at rest, least-privilege access, audit logging, and per-product credential separation so the store alone doesn't yield product access.
- **Complexity moves into one place.** We accept concentrated, well-tested complexity over duplicated, drifting complexity across three silos — the former is auditable and fixable once.

---

## 8. Q&A — likely auditor / colleague questions, with answers

**Q: The three products track different legal entities — Personal is a person, SME/Corporate are companies. How does one identity make sense?**
A: The identity is the *login of the human who operates the account*, not the account holder. The company remains the customer of record — separately KYB'd, with its own profile, money, and password. We authenticate the operator once; we don't merge the legal entities. It's the same as logging into your business bank with your own credentials to run your company's account.

**Q: Is the individual even relevant to a business account?**
A: Yes — legally required. AML/KYC obliges us to identify the beneficial owner and controlling persons of a company account; our SME KYB already captures the owner's BVN, DOB, and personal address. A verified human always stands behind a business account, so unifying that human's login is natural, not invented.

**Q: Can one company account have several operators, and can one person operate several accounts?**
A: Yes to both — that's the point of separating operator (Identity) from account (Profile/Organisation). Corporate already supports multiple users on one organisation with maker-checker roles; a person can also operate their Personal account and a company's account with one login, each still isolated.

**Q: If it's one identity, doesn't one leaked password expose all products?**
A: No. Each product has its own password; product endpoints require a token that proves *that* product's password. A Personal breach stays in Personal. Money movement additionally needs the per-product transaction PIN.

**Q: How is this more secure than three independent logins?**
A: It gives the same per-product isolation, but auth is implemented and hardened **once** instead of three times — so security policy is consistent and can't drift, and there are fewer copies of customer PII to breach.

**Q: What stops a Personal-logged-in user from calling SME APIs?**
A: SME financial endpoints require a `product=Sme` product-scoped token. A Personal or bare-identity token is rejected (`403`).

**Q: What is a "product-scoped token" and why two tokens?**
A: Login yields a short-lived *identity* token (proves the human) that can't touch product money. To use a product you `switch-product`, proving that product's password, and receive a *product* token scoped to that product. It's step-up auth.

**Q: How are passwords stored?**
A: bcrypt (salted, adaptive). No plaintext, no reversible storage, no fast hashes. Same for biometric tokens and refresh tokens at rest.

**Q: Brute force / credential stuffing?**
A: Per-account lockout (5 tries → 30 min, checked before testing the password) *and* per-IP rate limiting on auth endpoints. Reset flows don't reveal whether an account exists.

**Q: Account/user enumeration?**
A: Unknown user and wrong password return identical generic errors; password reset always returns a generic message and never leaks the OTP in production.

**Q: Session / token theft?**
A: Short TTLs (identity ~15 min, product ~4 hr); refresh tokens are hashed at rest, single-use with rotation, and revocable via logout-all; admin sessions re-validate every request.

**Q: New device logging in?**
A: New-device login requires a one-time code delivered to the account's verified number before the device is trusted.

**Q: How do you prevent two people claiming the same phone/email?**
A: DB-level unique constraints on identity and per-membership contacts, plus app-level "already claimed" checks at verification time.

**Q: Can someone attach a product to *my* identity by signing up with my email?**
A: No. Enrolling a product into an existing identity requires proving an existing credential (login) or a verified OTP — signup alone against an existing email routes to login, it doesn't enrol.

**Q: What about your existing legacy (XpressWallet) users — did you break them?**
A: No. On login they're migrated/linked into an identity with their onboarded state preserved (password verified first), so they land on their dashboard without re-onboarding.

**Q: Where are secrets and keys?**
A: AWS Secrets Manager (KMS-encrypted), injected into the runtime; never committed to source; not logged.

**Q: How would you detect/investigate an incident?**
A: Append-only audit log of auth/payment events (masked account numbers, PINs never stored) for forensics.

**Q: Did you validate this or just implement it?**
A: A full internal audit of the unified + legacy auth flow was performed; findings were prioritised and fixed (see §6), and hardening continues on a tracked backlog.

---

## 9. Glossary

- **Identity** — the single verified record of a human (phone/email). The anchor of the model.
- **Product** — Personal, SME, or Corporate banking.
- **Product membership** — a person's account within one product, with its own password, profile, and role.
- **Identity token** — short-lived token proving "this is the human"; cannot access product money data.
- **Product-scoped token** — token proving "this human proved THIS product's password"; required by product endpoints.
- **Step-up authentication** — requiring an additional/product-specific credential before granting product access.
- **Single customer view** — one authoritative record of a customer across products; an AML/KYC best practice.

---

*This document reflects the implementation as audited on 2026-07-10. For any specific control an auditor wants to see in code, backend can point to the exact file/line.*
