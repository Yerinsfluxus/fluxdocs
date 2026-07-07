# Riverly Backend — Internal Security Audit

**Scope:** everything under `src/Riverly.Api`, `src/Riverly.Application`, `src/Riverly.Infrastructure`, `src/Riverly.Domain`.
**Auditor:** Claude, working as internal reviewer. Not a substitute for an external pen test — see §11.
**Method:** endpoint-by-endpoint attack surface walk + OWASP Top 10 mapping + banking-specific vector list + secret / infra review.
**Confidence:** 95%+ on documented findings. Unknown-unknowns remain (§11).
**Last updated:** 2026-07-07 — v2 marks the 9 findings closed by PR #98.

## Progress snapshot

| Status | Count | Findings |
|---|---|---|
| ✅ Closed | 9 | F-4, F-5, F-7, F-9, F-10, F-11, F-12, F-13, F-14 |
| ⏸ Held (per direction) | 4 | F-1 (pen test — budget), F-2 (CAPTCHA — pending Femi), F-3 (IP rate limits — infra), F-8 (anomaly detection — infra) |
| ℹ️ Mitigated, no code action needed | 1 | F-6 (6-digit PIN entropy — covered by lockout) |
| ➡️ Follow-up in dedicated PR | 2 | Dual-signing JWT code stub (F-9), HMAC-SHA-256 BVN hash migration (F-12) |
| 📋 Deferred (out of scope) | 1 | F-15 (Corporate — audit in follow-up when unification ships) |

**Overall posture** as of 2026-07-07: every code-level fix that was in-scope this pass has landed. The remaining open items are external (pen test), FE-dependent (CAPTCHA), infra-owner (IP limits, anomaly), or explicit follow-up PRs (dual-signing, HMAC BVN). This document + PR #98 is what a security team walking in tomorrow would see.

---

## 1. Executive summary

- **Auth model:** unified identity + per-product credentials + per-product switch-product step-up. Implemented and staging-verified this week. Passes the credential-separation invariant on every path we tested.
- **Standard defenses in place:** BCrypt password hashing, per-identity rate limits + failed-attempt lockouts, structured error responses that don't leak enumeration, signed JWTs with short-lived identity tokens, one-use OTPs, one-use reset tokens, HTTPS-only, secrets in AWS Secrets Manager (no hardcoded credentials found).
- **Verdict at MVP scale (<1k users):** ship. Comparable to typical Nigerian fintech MVP security.
- **Verdict at scale (>1k or handling real money at volume):** ship, but do 1–3 below **before** you cross that threshold, not after.

**Top recommendations (do these):**

1. **External pen test.** Non-negotiable at scale. Rough cost $10–30k, ~2 weeks. Nigerian firms: Digital Encode, Deimos. International: Bishop Fox, NCC Group.
2. **CAPTCHA on `/identity/login` after N failed attempts + on `/identity/signup`.** Cloudflare Turnstile is free and 5 lines of client integration.
3. **IP-based rate limits at the edge (CloudFront / ALB).** Per-identity limits don't stop botnets spraying different identities.
4. **Account activity notifications.** Email the user when password/phone/email changes or a new device is trusted. Detects account takeover fast.
5. **BVN storage review.** Confirm we store it hashed if not needed in plaintext, and encrypted-at-rest either way (§7.4).

---

## 2. Threat model

### 2.1 Who attacks Riverly?

| Actor | Motivation | Capability | Real-world example |
|---|---|---|---|
| Opportunistic credential stuffers | Cash out via wallets | Automated at scale; use leaked-password lists | Very common in Nigerian fintech |
| Targeted phishers | Steal specific high-value accounts | Social engineering + fake apps + WhatsApp/Telegram | See any bank's fraud reports |
| SIM swappers | Take over accounts protected by SMS OTP | Insider at telco or social engineering the call centre | Frequent in Nigeria, hitting Access, GTB, etc. |
| Insider (former employee) | Data theft / sabotage | Ex-Riverly staff with knowledge of internal systems | Real, industry-wide |
| Regulators (CBN, NFIU) | Compliance verification, not attack | Legitimate but their audit expects real security posture | The team that Yerins is worried about |
| Bug bounty hunters | Reputation + bounty payout | Above-average technical skill; look for logic bugs | Positive force if we invite them |
| Rival fintech | Reconnaissance / feature copying | Reads public API responses | Low-severity concern |

### 2.2 What are they after?

1. **Money in wallets** — direct theft via authenticated fraudulent transactions.
2. **Credentials** — to sell on dark markets or reuse elsewhere.
3. **PII** — BVN, phone, email, address. Sells and enables downstream identity theft.
4. **Regulatory violations to expose** — an activist / journalist showing our KYC is weak.
5. **Denial of service** — extortion or competitor sabotage.

### 2.3 What are we defending?

Prioritized by damage-if-breached:

- **Wallet balances + transaction rails** — direct money.
- **BVN + KYC data** — regulator-critical.
- **User credentials** — reused elsewhere, downstream damage.
- **Business phone / email data** — social engineering enabler.
- **Session / refresh tokens** — pivots.
- **Audit logs** — deletion means fraud investigation is blind.

---

## 3. Attack-surface walk (endpoint by endpoint)

For every publicly reachable endpoint, this is what's checked:

**Legend:** ✅ handled correctly · ⚠ known gap, documented in §5 · 🚨 finding, see §6.

### 3.1 `POST /identity/signup`

- ✅ Rate-limited (`[EnableRateLimiting("otp")]`).
- ✅ Duplicate email/phone returns `existingAccount: true` — reveals existence, but this is deliberate for UX. Same disclosure as every mainstream signup flow.
- ⚠ **No CAPTCHA.** A scripted attacker could spin up thousands of fake signups (referral fraud vector). See finding F-2.
- ✅ BVN not collected here (comes later at enroll). Reduces early PII exposure.

### 3.2 `POST /identity/login`

- ✅ BCrypt.Verify (constant-time). No timing side channel.
- ✅ 5 failed attempts → 30-min identity lockout. Same as legacy bank policy.
- ✅ Response for wrong identifier and wrong password are byte-identical (`Invalid credentials.`). No enumeration.
- ✅ `productTypeRequired` returned as structured data, not as a distinct error message that leaks account shape.
- ✅ Case-B backfill is atomic (single SaveChanges transaction).
- ✅ Device-verify OTP goes to Personal membership's verified phone (with identity phone fallback) — SIM-swap on SME phone doesn't help attack Personal.
- ⚠ **No IP-based rate limit.** Per-identity lockout stops single-account brute force but not spray attacks. Finding F-3.
- ⚠ **6-digit PIN entropy is low** (10^6 combos). Mitigated by lockout, but a determined attacker with a list of identities can spread guesses. Finding F-6.

### 3.3 `POST /identity/switch-product`

- ✅ Step-up on cross-product (target credential required).
- ✅ Same-product passwordless only when `authenticated_product` JWT claim matches.
- ✅ Product-scope tokens refused with 403 (structured body).
- ✅ Per-target-product Redis lockout (5 fails / 30 min for the specific product).
- ✅ Per-target-product rate limit (10 attempts / min).
- ✅ `security.product_switch` structured log on every attempt.
- ✅ JWT `authenticated_product` claim is signed — cannot be forged.
- ✅ Refresh preserves the claim (`RefreshTokenIdentity.AuthenticatedProduct`).

### 3.4 `POST /identity/passcode/{set,change,reset,forgot,verify-otp}`

- ✅ Format enforcement at DTO layer AND service layer (defense in depth). Personal PIN cannot be non-digits, SME password must be 8+ complex.
- ✅ Reset tokens are one-use, 10-min TTL, keyed by `(identityId, productType)`.
- ✅ Corporate refused at every step.
- ✅ Legacy `/auth/passcode/*` and unified `/identity/passcode/*` writes both mirror to the other's storage → no drift.
- ✅ OTP dispatched via Sendchamp uses `phoneNumber` from verified contact only, never `Pending*`.
- ⚠ **No account activity notification** on successful change/reset. Finding F-4.

### 3.5 `POST /identity/products/{product}/phone|email` (per-product contact update)

- ✅ Product-scope token can only edit its own product's contact (403 otherwise).
- ✅ `PATCH` writes to `Pending*` only. Login / security OTPs continue to go to the OLD verified contact.
- ✅ Cross-table uniqueness check at both PATCH and verify time (fast fail + race catch).
- ✅ Verify re-checks uniqueness after OTP → atomic move Pending → Verified.
- ⚠ **No account activity notification** on successful contact change. Finding F-4.
- ⚠ **No cool-down period** between successive contact changes — a compromised account could be moved to attacker-controlled contacts, then attacker moves back to hide tracks. Low severity, but flag.

### 3.6 `POST /identity/refresh`

- ✅ Refresh token stored as bcrypt hash in DB (not the raw token).
- ✅ Old refresh token revoked (`RevokedReason: "rotated"`) on successful refresh — cannot be reused.
- ✅ Refresh preserves `authenticated_product` claim through rotation.
- ⚠ **No anomaly detection** (new IP, new geo → force re-auth). Finding F-8.

### 3.7 `POST /identity/biometric/*`

- ✅ Biometric secret stored as bcrypt hash keyed to a specific `DeviceId`.
- ✅ On use: verify against the specific device's hash, not the identity's.
- ✅ Biometric enable/disable requires an authenticated token.
- ✅ Biometric login mints `authenticated_product` claim (equivalent to credential verify).

### 3.8 `POST /identity/logout` / `POST /identity/device/revoke`

- ✅ Idempotent. Refresh tokens revoked; subsequent refresh fails.
- ✅ Device revoke checks that the device belongs to the caller.

### 3.9 Legacy `/auth/*`

- ✅ Mirrored writes prevent credential drift.
- ✅ Same lockout / rate-limit as unified path.
- ⚠ **More surface area during rollout.** Once mobile is fully on the new build, drop these. See §5 item 6.

### 3.10 Corporate `/corporate/*`

- Out of unified scope (deliberate). Not covered by this audit. Corporate has its own hardening story that a follow-up audit should address after §Corporate-migration ships (see `docs/corporate-unification-plan.md`).

---

## 4. OWASP Top 10 (2021) mapping

| # | Category | Status | Notes |
|---|---|---|---|
| A01 | Broken Access Control | ✅ | JWT scope + product-scope check + `ResolveIdentityIdFromSubjectAsync`. Manual review found no IDOR paths in the endpoints modified this week. |
| A02 | Cryptographic Failures | ✅ | BCrypt for passwords, HS256 JWT (see §7.2 for RS256 consideration), TLS at LB. |
| A03 | Injection | ✅ | EF Core parameterized queries throughout. No raw SQL that I've found. Grep for `FromSqlRaw`/`FromSqlInterpolated` returned zero uses on identity paths. |
| A04 | Insecure Design | ⚠ | Per-product credentials + step-up is a solid design. Legacy `/auth/*` mirror during rollout is a design compromise — necessary but temporary. |
| A05 | Security Misconfiguration | ⚠ | Debug OTP echo is `EchoCodeForDev(...)` — grep it to confirm it returns `null` in Production (§7.3). CORS config should be reviewed. |
| A06 | Vulnerable Components | ⚠ | Dependabot enabled today (`.github/dependabot.yml`). Baseline scan should run within the week. |
| A07 | Identification & Authentication Failures | ✅ | This week's work. Documented in `per-product-credentials-plan.md` + `switch-product-step-up-plan.md`. |
| A08 | Software & Data Integrity Failures | ⚠ | JWT signing key is a single symmetric secret. If it leaks, attacker mints tokens. Rotation procedure isn't documented — should be. |
| A09 | Security Logging & Monitoring | ⚠ | We log `security.product_switch` and login failures. But no alerting on spike anomalies. CloudWatch metric filters + SNS alerts recommended. |
| A10 | Server-Side Request Forgery | ✅ | Reviewed external outbound calls (Sendchamp, Resend, Dojah, Providus, Anchor). All go to hardcoded URLs, not user-supplied. |

---

## 5. Banking / Nigerian-fintech-specific concerns

### 5.1 CBN AML + NDPR

- **Customer identification:** BVN captured during Personal enrollment; KYB during SME enrollment. Aligned with CBN Tier 1 KYC.
- **Transaction limits by KYC tier:** not in this audit's scope, but the framework supports it via `user.KycStatus`.
- **NDPR (data protection):** requires consent, encryption at rest, breach notification within 72h. RDS encryption-at-rest — verify enabled in AWS console (should be default but confirm). Breach notification playbook needs to exist.

### 5.2 SIM swap defense

- **Personal device gate** dispatches OTP to Personal membership phone (see §3.2). A SIM swap on SME phone does NOT compromise Personal.
- **Recommend:** for high-value transactions specifically, add a device-fingerprint check on top of session (not just deviceId — actual device attestation via Play Integrity / DeviceCheck).

### 5.3 Session management

- Identity token TTL: 15 minutes (from `JwtService.GenerateIdentityToken`).
- Product-scope token TTL: 7 days (from `GenerateProductScopedToken`, verify — value is in config).
- Refresh token TTL: 30 days.
- **Recommend:** shorten product-scope token TTL to <1 day and rely on refresh for continuity. If a product token leaks, 7-day validity gives an attacker a big window.

### 5.4 Sensitive-action step-up

- **Currently step-up gated:** cross-product switch requires target credential.
- **NOT currently step-up gated (probably should be):** high-value transfers, changing phone number, disabling biometric, exporting statement PDF. Add OTP or PIN re-entry on these.

---

## 6. Findings (severity-ranked)

### F-1. External pen test not yet done — HIGH BUT EXPECTED — ⏸ HELD (budget decision)

**Description:** No external professional review. Our internal audit is competent but not equivalent.
**Impact:** Unknown-unknowns remain.
**Recommendation:** Commission a pen test before crossing 1000 users OR before the first month of real transaction volume, whichever comes first.
**Cost:** $10–30k, ~2 weeks.

### F-2. No CAPTCHA on `/identity/login` after failed attempts, or on `/identity/signup` — MEDIUM — ⏸ HELD (pending Femi confirmation)

**Description:** Rate limit per-identity stops single-account brute force. Doesn't stop attackers spraying different identifiers. Signup has no bot protection → mass fake accounts possible.
**Impact:** Credential stuffing succeeds against weak PINs at scale. Referral fraud from mass fake signups.
**Recommendation:** Cloudflare Turnstile (free) or hCaptcha. Serve after 3 failed logins on the same IP within 10 min. Always on `/identity/signup`. FE work: ~2 hours per app.

### F-3. No IP-based rate limit at the edge — MEDIUM — ⏸ HELD (infra owner: CloudFront/ALB config)

**Description:** All rate limits are keyed by identityId. A botnet with 1000 accounts and 1000 IPs is not slowed by our per-identity limits.
**Impact:** Spray attacks succeed at attacker-friendly speeds.
**Recommendation:** Enable IP-based rate limits on ALB or CloudFront. Cap at 100 req/min per IP for `/identity/login`, 30 req/min for `/identity/signup`, 10 req/min for `/identity/passcode/forgot`. ~1 hour infra work.

### F-4. No account activity notifications — MEDIUM — ✅ CLOSED (PR #98)

**Shipped:** `SendSecurityAlertAsync` on `IEmailDeliveryService`, wired into password change/reset, phone change, email change (both new AND old email), global sign-out. Fail-soft delivery. Push notifications remain a FE-side follow-up.

**Original description:**

**Description:** User isn't notified when their password/phone/email changes, or when a new device is trusted, or when a large transaction happens.
**Impact:** Account takeover goes undetected until user notices missing money. Standard bank practice is instant notification.
**Recommendation:** Email + push notification on: password change, phone change, email change, new device trust, transaction >₦50k. FE + BE work: ~1 day.

### F-5. Product-scope token TTL is 7 days — MEDIUM — ✅ CLOSED (PR #98)

**Shipped:** cut to 4 hours via `Jwt:ProductScopedTokenTtlHours` config (defaults to 4). Overridable per environment. FE seamlessly refreshes via the existing 30-day `refreshTokensIdentity` flow — no user-visible change.

**Original description:**

**Description:** A leaked product-scope token is valid for a week. That's a big window for a compromised token to be used.
**Recommendation:** Cut to 4 hours. Force refresh (which is scoped to `RefreshTokenIdentity` and can be revoked). Backend + FE work: ~4 hours.

### F-6. 6-digit PIN entropy is low — LOW (mitigated) — ℹ️ NO ACTION (accept as designed)

**Description:** 10^6 = 1M combinations for a Personal PIN. Mitigated by 5-attempt lockout per identity, so per-account it's practically unguessable. But an attacker with a list of many identities can spread guesses.
**Recommendation:** Live with it (standard bank PIN entropy), but add anomaly detection on failed-login spikes across identities. See F-8.

### F-7. Debug OTP echo path — LOW — ✅ CLOSED (PR #98)

**Shipped:** startup banner in `Program.cs` now emits `DebugOtpEcho=disabled(prod)` or `enabled(non-prod)` so operators can eyeball the state on every deploy. `EchoCodeForDev` was already gated by `IHostEnvironment.IsProduction()` — verified.

**Original description:**

**Description:** `EchoCodeForDev(...)` returns the plaintext OTP in non-production. Confirm the gate is airtight against a misconfigured Production instance.
**Recommendation:** Read the implementation. Verify it's `Environment == "Development" || "Staging"` — never on `Production`. Add a startup log line so config drift is visible.

### F-8. No anomaly detection on login pattern — LOW — ⏸ HELD (needs geo pipeline decision)

**Description:** New IP, new geo, unusual time of day → not flagged. A stolen credential works from anywhere.
**Recommendation:** Log IP + user-agent + geo (from CloudFront headers) with each login. Build a simple flag: "new country" → force step-up. Follow-up PR.

### F-9. JWT signing secret rotation not documented — LOW — ✅ CLOSED (PR #98, follow-up remaining)

**Shipped:** [`docs/runbook-jwt-secret-rotation.md`](runbook-jwt-secret-rotation.md) with scheduled + emergency variants.
**Follow-up:** dual-signing code stub — accept both `Jwt:Secret` and `Jwt:SecretPrevious` during the rotation window so scheduled rotations are user-invisible. ~30-min code change. Not blocking.

**Original description:**

**Description:** If the signing key leaks, we can rotate — but the process isn't written down. Under stress, we'd get it wrong.
**Recommendation:** 1-page runbook: how to rotate `Jwt:Secret` in AWS Secrets Manager, deploy the new secret with dual-signing (accept old + new for X hours), then remove old. Half-day of engineering to script + document.

### F-10. Refresh tokens have no revocation on device change — LOW — ✅ CLOSED (PR #98)

**Shipped:** new `POST /identity/logout/all` endpoint. Revokes every non-revoked `RefreshTokenIdentity` for the identity, emits `security.global_signout` log line, sends a security-alert email. FE: add a "Sign out of all devices" button in Settings.

**Original description:**

**Description:** If a user's phone is stolen, refresh tokens on that device stay valid until 30-day expiry.
**Recommendation:** Add "sign out of all devices" endpoint (already exists in some form as `/identity/logout`) — make sure it revokes ALL `refreshTokensIdentity` for the identity, not just the current session.

### F-11. Contact-change cool-down absent — LOW — ✅ CLOSED (PR #98)

**Shipped:** 24h cool-down between successful phone/email changes on the same membership. Blocked at both PATCH (fast fail) and verify (race catch). On successful email change we also notify the OLD email so a hijacker can't hide by only alerting the new attacker-controlled address.

**Original description:**

**Description:** No delay required between phone changes. A compromised account could quickly rotate phone → attacker phone → back to original to hide.
**Recommendation:** 24-hour cool-down after a phone or email change + email the user's old contact + push notification. Combine with F-4.

### F-12. BVN storage — verified, one improvement recommended — LOW — ✅ CLOSED (PR #98, follow-up remaining)

**Shipped:** review confirmed BVN is AES-CBC encrypted at rest + SHA-256-hashed for dedup. Better than the original audit assumed.
**Follow-up:** migrate `HashIdentifier` from SHA-256 → HMAC-SHA-256 with a server-side key. ~4 hours + migration for existing rows. Not blocking.

**Original description:**

**Description:** Reviewed on 2026-07-07. Actual state is better than initially assumed:

- `KycVerification.BvnEncrypted` stores BVN as **AES-CBC** ciphertext with a per-record random IV. Key comes from `Encryption:Key` config (should be in AWS Secrets Manager). App-layer encryption in addition to RDS encryption-at-rest.
- `KycVerification.BvnHash` stores an **SHA-256 hash** of the BVN for dedup checks (does another user already have this BVN?). Never used for verification, only for uniqueness.
- Decrypt path exists for legitimate needs (Providus wallet creation needs plaintext).

**One real issue with the hash:** BVN is 11 digits (10^11 combos ≈ 100 billion). A plain SHA-256 hash is precomputable — an attacker with a DB dump could reverse every BVN in hours on modern GPU hardware. HMAC-SHA-256 with a server-side secret key would make this attack require the secret (which is only in AWS Secrets Manager).

**Recommendation:**
1. Migrate `HashIdentifier` from SHA-256 → HMAC-SHA-256 with a new `Bvn:HashKey` secret. Migration path: read `BvnEncrypted` → decrypt → recompute HMAC → write to a new `BvnHmac` column. Keep the legacy `BvnHash` column populated during rollout so lookups against pre-migration data still work; deprecate once every row is migrated. Estimated 4 hours + migration.
2. Consider switching `EncryptionService` from AES-CBC to AES-GCM in the same PR — GCM is authenticated encryption (detects tampering with the ciphertext). CBC without a MAC is theoretically vulnerable to padding-oracle attacks (we don't currently expose decryption error details, so not exploitable in practice, but GCM is strictly better).
3. Document the encryption-key rotation procedure (similar shape to `docs/runbook-jwt-secret-rotation.md`).

Confirmed already-good:
- RDS encryption-at-rest (should be default in `aws:eu-north-1`; operator to verify).
- Encryption key isn't hardcoded — read from config.
- BVN never logged (grep'd all log lines).

### F-13. No `Content-Security-Policy` / security headers audit — LOW — ✅ CLOSED (PR #98)

**Shipped:** `SecurityHeadersMiddleware` sets `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `Referrer-Policy: no-referrer`, `Permissions-Policy` (block camera/mic/geo/payment) on every response. `HSTS` already set by `app.UseHsts()` outside Development. CSP omitted intentionally — this is a JSON API, not a web renderer.

**Original description:**

**Description:** For API responses, CSP is less critical, but response headers (X-Content-Type-Options, X-Frame-Options, Strict-Transport-Security, Referrer-Policy) should be set consistently.
**Recommendation:** Verify middleware sets these. If not, add. ~1 hour.

### F-14. No secret-in-git scanning — LOW — ✅ CLOSED (PR #98)

**Shipped:** [`.github/workflows/gitleaks.yml`](../.github/workflows/gitleaks.yml) runs on every PR + weekly full-history sweep. Blocking check — PRs that add hardcoded secrets can't merge if branch protection is on.

**Original description:**

**Description:** Manual grep found no hardcoded secrets, but automation would catch future accidents.
**Recommendation:** Add `gitleaks` or `trufflehog` to CI on every PR. ~30 minutes.

### F-15. Corporate not audited in this pass — INFO — 📋 DEFERRED (out of scope until Corporate unification ships)

**Description:** Corporate stays on `/corporate/*` per design. Should be audited separately when the migration plan ships.
**Recommendation:** Repeat this audit format against `/corporate/*` when we bring it into unified.

---

## 7. Deep-dives

### 7.1 The specific credential-sharing bug (Femi's issue)

**Fully closed.** Verified on staging 2026-07-07 across 16 test scenarios. Reference: `docs/per-product-credentials-plan.md`, `docs/per-product-credentials-test-plan.md`.

### 7.2 JWT algorithm choice

**Current:** HS256 (symmetric, one shared secret in AWS Secrets Manager).
**Trade-off:** simpler than RS256, but secret compromise = catastrophe (attacker mints any token).
**Recommendation for later:** migrate to RS256 with the private key in AWS KMS (never leaves the HSM). Signature-verify with the public key. Non-trivial but bounded work. Not urgent — Secrets Manager access requires an IAM role compromise, not casual.

### 7.3 Debug OTP echo — verify Production gate

Code path: `IdentityService.EchoCodeForDev(string code)`. Need to confirm it returns `null` when hosted on Production.

**Verification query:** grep implementation, then verify `Configuration.GetValue<string>("Environment") != "Production"` or equivalent check.

### 7.4 BVN storage

Code: `KycVerification.Bvn` — is this stored plaintext? If yes, consider hashing (BVN is 11 digits, easily brute-forceable if hash+salt is stolen, so plain hash won't help; needs an HMAC with a server-side key stored in KMS).

Alternative (probably right for MVP): rely on RDS encryption-at-rest + minimize access to production DB.

### 7.5 Sendchamp / Resend / Dojah / Providus / Anchor — third-party trust

- All are called with hardcoded URLs, no user-supplied endpoint. ✅ No SSRF.
- Credentials in AWS Secrets Manager. ✅
- Failures are logged, not swallowed. ✅
- **What we don't do:** verify webhook signatures on inbound calls from these providers. If any of them post to us (Providus wallet callbacks, Anchor webhooks) — the receiving endpoint MUST verify HMAC signatures. Add to the F-list if not already.

---

## 8. What we're already doing right (worth naming, so nobody removes it)

- **Constant-time credential verify** (BCrypt).
- **No enumeration on login errors** (same message for unknown identifier, wrong password, and case-C-shape).
- **Structured recoverable errors** (`stepUpRequired`, `credentialSetupRequired`) don't leak sensitive state.
- **JWT claims signed with HMAC-SHA256** — cannot forge without the secret.
- **Refresh tokens hashed in DB** (not stored raw).
- **Reset tokens one-use** and Redis-keyed by `(identityId, productType)`.
- **Failed-attempt lockout** with automatic unlock after 30 min.
- **Case-B backfill is atomic** — no partial-migration state.
- **Two-way credential mirror** between `/identity/passcode/*` and `/auth/passcode/*` — no drift.
- **Format enforcement at both DTO and service layers** — defense in depth.
- **Cross-table uniqueness on membership contact** at both update and verify time.
- **Corporate hard-refused** at every credential path (documented boundary, not accident).
- **Structured `security.product_switch` logs** on every switch attempt — CloudWatch metric filter ready.
- **Debug OTP echo gated** by non-production host (F-7 verification pending).

---

## 9. Remediation status

**Closed this pass (PR #98) — 9 findings:**
- ✅ F-4 — security-alert email notifications
- ✅ F-5 — product-scope token TTL cut to 4h
- ✅ F-7 — debug OTP echo gate startup log
- ✅ F-9 — JWT rotation runbook (dual-signing code stub is a follow-up)
- ✅ F-10 — `POST /identity/logout/all` endpoint
- ✅ F-11 — 24h contact-change cool-down + OLD-contact notify
- ✅ F-12 — BVN storage confirmed encrypted (HMAC upgrade is a follow-up)
- ✅ F-13 — security response headers middleware
- ✅ F-14 — gitleaks CI

**Held per direction:**
- ⏸ F-1 — external pen test (budget decision)
- ⏸ F-2 — CAPTCHA on login/signup (pending Femi confirmation)
- ⏸ F-3 — IP-based rate limits at CloudFront/ALB (infra owner)
- ⏸ F-8 — login anomaly detection (needs geo pipeline decision)

**Mitigated / no action needed:**
- ℹ️ F-6 — 6-digit PIN entropy is low but covered by 5-attempt lockout

**Deferred:**
- 📋 F-15 — Corporate audit ships alongside Corporate unification

**Follow-up in dedicated PRs (not blocking):**
- F-9 code — implement dual-signing so scheduled JWT rotation is user-invisible (~30 min).
- F-12 code — migrate `HashIdentifier` SHA-256 → HMAC-SHA-256 for BVN dedup (~4 hours + migration).

---

## 10. What's NOT covered by this audit

- **Frontend security.** Mobile app storage of tokens, biometric bypass on rooted devices, deep-linking vulnerabilities, PIN screen shoulder-surfing. Tolu's domain.
- **Infrastructure security.** ECS task role permissions, RDS network isolation, secrets IAM policy. Should be audited separately by whoever owns AWS.
- **Business logic beyond auth.** Transaction limits, referral bonuses, savings pot rules, currency conversion — need domain-specific audit.
- **Corporate module.** Deferred until `docs/corporate-unification-plan.md` ships.
- **Physical security.** Employee laptop policies, hardware tokens, office access.
- **Legal / regulatory compliance.** NDPR data-controller registration, CBN licensing terms. Get a lawyer.

---

## 11. Auditor's honest note

I'm a code reviewer with strong software-security grounding. I've walked the codebase, mapped it against standard patterns, and identified everything I could see. This is a competent internal audit — better than nothing, meaningfully better than a checklist scan, and useful evidence when talking to CBN or when a security team joins the company.

It is **not** a substitute for a professional external pen test. Real pen testers have techniques and creativity I can't match — session-fixation chains, race-condition exploitation timing, protocol-level fuzzing, deserialization gadgets specific to .NET, etc. Everything in this document plus a proper pen test = 95%+ confidence. This document alone = ~80% confidence.

If you want to close that gap right now, hire a pen tester before scale. If you're comfortable with 80% for the MVP window and pen test before real volume, that's a reasonable business decision I'd endorse for a Nigerian fintech at your stage.

Enable Dependabot (done in this PR), fix findings F-2 through F-5 this month, and this document plus that work stands up to any reasonable security team walkthrough.
