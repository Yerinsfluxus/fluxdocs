# Riverly — Information Security Controls Inventory (ISO/IEC 27001:2022)

**System:** Riverly Backend API (`riverly-api`) — .NET 8, ASP.NET Core, PostgreSQL, AWS (eu-north-1)
**Prepared for:** External cybersecurity / ISO 27001 assessment team
**Prepared by:** Internal engineering security review (full-codebase audit)
**Date:** 14 July 2026
**Classification:** Confidential — Internal / Assessor use only
**Scope of this document:** Application backend, its CI/CD pipeline, cloud deployment configuration, and third-party integration surface. It maps implemented controls to ISO/IEC 27001:2022 Annex A themes, states each control's location in the codebase, and maintains an honest gaps register.

> **How to read this.** Every section lists **controls in place** (with the exact file and line so an assessor can verify) and, where relevant, **weaknesses or gaps**. Statuses use the legend in [§2](#2-how-to-read-this-document). Nothing here is aspirational — if a control is only partial or absent, it is marked as such. Known gaps are consolidated and risk-ranked in [§12](#12-consolidated-gaps-register-risk-ranked).

---

## Table of Contents

1. [Executive Summary and Security Posture](#1-executive-summary-and-security-posture)
2. [How to Read This Document](#2-how-to-read-this-document)
3. [ISO 27001:2022 Annex A Coverage Map](#3-iso-270012022-annex-a-coverage-map)
4. [Scope, Architecture and Environment Separation](#4-scope-architecture-and-environment-separation)
5. [Identity and Authentication (A.5.16, A.5.17, A.8.5)](#5-identity-and-authentication-a516-a517-a85)
6. [Authorization and Access Control (A.5.15, A.5.18, A.8.2, A.8.3)](#6-authorization-and-access-control-a515-a518-a82-a83)
7. [Cryptography and Secrets Management (A.8.24, A.5.17)](#7-cryptography-and-secrets-management-a824-a517)
8. [API and Transport Security (A.8.20, A.8.21, A.8.26, A.8.28)](#8-api-and-transport-security-a820-a821-a826-a828)
9. [Data Protection and Privacy (A.8.10, A.8.11, A.8.12, A.5.34)](#9-data-protection-and-privacy-a810-a811-a812-a534)
10. [Third-Party Integrations and Webhooks (A.5.19, A.5.20, A.5.22, A.8.21)](#10-third-party-integrations-and-webhooks-a519-a520-a522-a821)
11. [Infrastructure, CI/CD and Secure Development (A.8.9, A.8.25, A.8.29, A.8.31)](#11-infrastructure-cicd-and-secure-development-a89-a825-a829-a831)
12. [Logging, Monitoring and Audit Trail (A.8.15, A.8.16, A.5.28)](#12-logging-monitoring-and-audit-trail-a815-a816-a528)
13. [Consolidated Gaps Register (Risk-Ranked)](#13-consolidated-gaps-register-risk-ranked)
14. [Regulatory Alignment (CBN, NDPR, NIBSS/NPS)](#14-regulatory-alignment-cbn-ndpr-nibssnps)
15. [Vulnerability Management and Dependencies (A.8.8)](#15-vulnerability-management-and-dependencies-a88)
16. [Evidence Requested and Out-of-Scope Items](#16-evidence-requested-and-out-of-scope-items)
17. [Appendix A — Key File Reference Index](#17-appendix-a--key-file-reference-index)
18. [Appendix B — Operational Runbooks and Plans](#18-appendix-b--operational-runbooks-and-plans)

---

## 1. Executive Summary and Security Posture

Riverly is a Nigerian multi-product fintech (Personal, SME, Corporate wallets) built on a .NET 8 clean-architecture backend. The platform holds customer funds, moves money over Nigerian bank rails (Providus/XpressWallet, Anchor, a Liberty core-banking system), and stores regulated KYC data (BVN, NIN, identity documents, liveness selfies).

**Overall posture:** The application implements a competent, defense-in-depth control set that is consistent with — and in several areas ahead of — a typical Nigerian fintech MVP. Authentication, cryptographic password handling, webhook authenticity, object-level access control on the Personal/SME products, and money-movement idempotency are all genuinely implemented, not merely claimed.

**What is strong:**

- Unified identity with per-product credentials and step-up on cross-product switching.
- BCrypt for all passwords, PINs, refresh tokens, biometric secrets and OTP codes; cryptographic RNG for OTPs, refresh and biometric tokens.
- Per-identity failed-attempt lockout and per-endpoint rate limiting.
- HMAC signature verification, fail-closed, on **both** inbound money webhooks (Providus, Anchor), with constant-time comparison and verify-before-credit on deposit paths.
- Secrets injected at runtime from AWS Secrets Manager; CI/CD deploys via GitHub OIDC (no long-lived AWS keys in CI); `gitleaks` secret scanning in the pipeline.
- Idempotency records on Personal, SME, and bill-payment money movement; server-side name-enquiry gates that override client-supplied beneficiary names; row-level `FOR UPDATE` locks on SME transfers.
- Field-level AES encryption of BVN at rest, S3 documents private with presigned-URL access, and a masked, sanitized payment audit log.

**What needs attention (top priorities):**

1. **Two confirmed cross-organization IDOR flaws** in the Corporate module leak UBO/beneficial-owner PII and pending-invite lists across tenants ([§6](#6-authorization-and-access-control-a515-a518-a82-a83)).
2. **NIN stored unencrypted** — the encryption column exists but is never populated; BVN is protected, NIN is not ([§9](#9-data-protection-and-privacy-a810-a811-a812-a534)).
3. **Full KYC PII payloads logged to CloudWatch** at Information level (owner BVN, ID, address); BVN/NIN also appear in outbound request URLs ([§12](#12-logging-monitoring-and-audit-trail-a815-a816-a528)).
4. **Destructive admin operations** (identity/user "wipe", bulk funding, KYB approval) gated only by a **shared static `X-Admin-Key`** with no per-actor identity, expiry, or rotation ([§6](#6-authorization-and-access-control-a515-a518-a82-a83)).
5. **Swagger/OpenAPI exposed unauthenticated** in staging/production ([§8](#8-api-and-transport-security-a820-a821-a826-a828)).
6. **No MFA for privileged admin accounts** ([§5](#5-identity-and-authentication-a516-a517-a85)).
7. **Money-movement concurrency/reconciliation edges** — Personal-transfer and bill-payment idempotency check happens before the row lock; core-banking debits can silently degrade to local-only posting ([§10](#10-third-party-integrations-and-webhooks-a519-a520-a522-a821)).
8. **No cloud monitoring/alerting yet** — zero CloudWatch alarms, no GuardDuty, no CloudTrail as of 14 July 2026 ([§12](#12-logging-monitoring-and-audit-trail-a815-a816-a528)).

**Verdict:** Suitable to operate at controlled MVP volume with the fixes in [§13](#13-consolidated-gaps-register-risk-ranked) prioritized. An **external penetration test** and the monitoring build-out in [Appendix B](#18-appendix-b--operational-runbooks-and-plans) are the two items that most move the needle for an ISO 27001 certification path.

---

## 2. How to Read This Document

Each control is tagged with an implementation status:

| Status | Meaning |
|---|---|
| ✅ **Implemented** | Control exists and works as intended; verifiable at the cited location. |
| ⚠️ **Partial** | Control exists but has a documented weakness, narrow coverage, or configuration risk. |
| ❌ **Gap** | Control is absent or effectively non-functional. |
| ℹ️ **Evidence needed** | Control is expected to exist in cloud infrastructure (outside the repo); assessor should request AWS-side evidence. |

File references use the form `path/to/File.cs:line`. Line numbers reflect the codebase at the audit date and may drift as code changes; the surrounding symbol name is given where practical.

ISO control identifiers (e.g. `A.8.24`) refer to **ISO/IEC 27001:2022 Annex A**. This is a controls inventory, not a formal Statement of Applicability (SoA); it is intended to feed one.

---

## 3. ISO 27001:2022 Annex A Coverage Map

This table is the one-page overview. Each row links to the detailed section.

| Annex A theme / control | Coverage | Detail |
|---|---|---|
| A.5.15 Access control | ⚠️ Partial | [§6](#6-authorization-and-access-control-a515-a518-a82-a83) |
| A.5.16 Identity management | ✅ Implemented | [§5](#5-identity-and-authentication-a516-a517-a85) |
| A.5.17 Authentication information | ✅ / ⚠️ | [§5](#5-identity-and-authentication-a516-a517-a85), [§7](#7-cryptography-and-secrets-management-a824-a517) |
| A.5.18 Access rights | ⚠️ Partial | [§6](#6-authorization-and-access-control-a515-a518-a82-a83) |
| A.5.19–5.22 Supplier / third-party security | ⚠️ Partial | [§10](#10-third-party-integrations-and-webhooks-a519-a520-a522-a821) |
| A.5.28 Collection of evidence (audit) | ⚠️ Partial | [§12](#12-logging-monitoring-and-audit-trail-a815-a816-a528) |
| A.5.34 Privacy and PII protection | ⚠️ Partial | [§9](#9-data-protection-and-privacy-a810-a811-a812-a534) |
| A.8.2 Privileged access rights | ⚠️ Partial | [§6](#6-authorization-and-access-control-a515-a518-a82-a83) |
| A.8.3 Information access restriction | ⚠️ Partial | [§6](#6-authorization-and-access-control-a515-a518-a82-a83) |
| A.8.5 Secure authentication | ✅ / ⚠️ | [§5](#5-identity-and-authentication-a516-a517-a85) |
| A.8.8 Technical vulnerability management | ⚠️ Partial | [§15](#15-vulnerability-management-and-dependencies-a88) |
| A.8.9 Configuration management | ⚠️ Partial | [§11](#11-infrastructure-cicd-and-secure-development-a89-a825-a829-a831) |
| A.8.10 Information deletion | ❌ Gap | [§9](#9-data-protection-and-privacy-a810-a811-a812-a534) |
| A.8.11 Data masking | ⚠️ Partial | [§9](#9-data-protection-and-privacy-a810-a811-a812-a534) |
| A.8.12 Data leakage prevention | ⚠️ Partial | [§9](#9-data-protection-and-privacy-a810-a811-a812-a534) |
| A.8.15 Logging | ⚠️ Partial | [§12](#12-logging-monitoring-and-audit-trail-a815-a816-a528) |
| A.8.16 Monitoring activities | ❌ Gap | [§12](#12-logging-monitoring-and-audit-trail-a815-a816-a528) |
| A.8.20–8.21 Network / transport security | ✅ / ⚠️ | [§8](#8-api-and-transport-security-a820-a821-a826-a828) |
| A.8.23 Web filtering / edge controls | ⚠️ Partial | [§8](#8-api-and-transport-security-a820-a821-a826-a828) |
| A.8.24 Use of cryptography | ✅ / ⚠️ | [§7](#7-cryptography-and-secrets-management-a824-a517) |
| A.8.25 Secure development lifecycle | ⚠️ Partial | [§11](#11-infrastructure-cicd-and-secure-development-a89-a825-a829-a831) |
| A.8.26 Application security requirements | ✅ / ⚠️ | [§8](#8-api-and-transport-security-a820-a821-a826-a828) |
| A.8.28 Secure coding | ✅ Implemented | [§8](#8-api-and-transport-security-a820-a821-a826-a828) |
| A.8.29 Security testing in dev/acceptance | ❌ Gap | [§11](#11-infrastructure-cicd-and-secure-development-a89-a825-a829-a831) |
| A.8.31 Separation of dev/test/prod | ✅ / ⚠️ | [§11](#11-infrastructure-cicd-and-secure-development-a89-a825-a829-a831) |
| A.8.13 Information backup | ℹ️ Evidence needed | [§16](#16-evidence-requested-and-out-of-scope-items) |

---

## 4. Scope, Architecture and Environment Separation

**Architecture.** Clean architecture across four projects — `Riverly.Api` (host, middleware, controllers), `Riverly.Application` (DTOs, validation, helpers), `Riverly.Domain` (entities), `Riverly.Infrastructure` (EF Core, external service clients). Target framework `net8.0` across all projects. PostgreSQL via Npgsql/EF Core; Redis (ElastiCache) for OTP state, rate-limit and lockout counters; AWS S3 for documents; ECS Fargate runtime behind an Application Load Balancer.

**Environment separation (A.8.31).** ✅ Documented and enforced across distinct AWS resources:

| | Staging | Production |
|---|---|---|
| API URL | `api-staging.riverly.ng` | `api.riverly.ng` |
| RDS | `riverly-staging-db` | `riverly-prod-db` |
| ECS cluster | `riverly-cluster-0` | `riverly-cluster-prod` |
| Redis | `riverly-staging-redis` | `riverly-prod-redis` |
| Secrets Manager | `riverly/staging/config` | `riverly/production/config` |
| Touches real money | No — test rails | Yes |

Separate Secrets Manager paths, clusters, IAM roles, and S3 buckets per environment. Source: [`docs/environment-credentials.md`](../docs/environment-credentials.md).

⚠️ **Environment blast-radius bleed.** Some third-party credentials are shared across environments: Dojah (KYC) uses the **same live keys** in staging and prod; Sendchamp uses **one live key** company-wide; Firebase uses a shared project; Liberty CBS is stubbed (`1234567890`) in both. A staging incident can therefore touch real Dojah/settlement paths. Source: [`docs/environment-credentials.md:35-41`](../docs/environment-credentials.md).

⚠️ **Production task definition not in the repo.** Only the staging ECS task definition (`task-definition.json`) is committed; the production container security configuration (user, secrets, roles) lives only in AWS and cannot be audited from source. Request as evidence ([§16](#16-evidence-requested-and-out-of-scope-items)).

---

## 5. Identity and Authentication (A.5.16, A.5.17, A.8.5)

### 5.1 Controls in place

**Unified identity, per-product credentials.** ✅ A single `Identity` resolves to per-product `ProductMembership` credentials (Personal / SME / Corporate). Login resolves across identities → memberships → legacy `users`/`corporateUsers` with auto-migration on a verified credential. `src/Riverly.Infrastructure/Services/IdentityService.cs:786-1461`.

**Password / passcode hashing.** ✅ BCrypt (`BCrypt.Net.BCrypt.HashPassword`) is used for **every** secret: Personal passcode, per-product membership password, admin password, refresh tokens, OTP codes, biometric tokens, and transaction PINs. Salt is auto-generated per hash.

**Password / passcode policy.** ✅ Enforced at both DTO and service layers (defense in depth):
- Personal passcode: exactly 6 digits.
- SME/Corporate password: 8–100 chars, upper + lower + digit + special.
- Admin password: ≥12 chars with all four character classes (`AdminAuthService.cs:19-21`).
- Change/reset blocks reuse of the current secret.

**Failed-attempt lockout.** ✅ Per-identity: 5 failures → `Locked`, 30-minute `LockoutEnd`, auto-unlock on expiry, counter reset on success (`IdentityService.cs:1322-1337`; legacy `AuthService.cs:137-144`). Admin: threshold 5 → lock 15 min **and** `TokenVersion` bump that invalidates live admin JWTs (`AdminAuthService.cs:116-126`).

**Login enumeration hardening.** ✅ A `TimingDummyHash` BCrypt burn runs for unknown identifiers, and a byte-identical `Invalid credentials.` response is returned for both unknown-identifier and wrong-password cases (`IdentityService.cs:33-34, 1037-1040`).

**OTP.** ✅ In-house OTP uses `RandomNumberGenerator.GetInt32` (cryptographic RNG), 6 digits, stored **BCrypt-hashed in Redis**, 10-minute expiry, max 5 verify attempts, max 3 resends per 10 min (`OtpService.cs:12-13, 25-26, 49-56, 74-84`). Phone OTP is delegated to Sendchamp (`verification/create` + `/confirm`) with the reference held in Redis and deleted on success to prevent reuse.

**JWT issuance and validation.** ✅ HS256, signing key from `Jwt:Secret` (config → Secrets Manager in deployed envs; no hardcoded fallback in code). Validation enables issuer, audience, lifetime, and signing-key checks (`Program.cs:283-293`). Token lifetimes: identity token 15 min; product-scoped token 4 h (config `Jwt:ProductScopedTokenTtlHours`); refresh token 30 days.

**Refresh tokens.** ✅ Generated with `RandomNumberGenerator.GetBytes(64)`, stored **BCrypt-hashed**, rotated on use (old token `RevokedReason="rotated"`), and revoked on logout, passcode change/reset, and device change (`JwtService.cs:43-47`; `AuthService.cs:354-397`).

**Admin instant revocation.** ✅ Admin tokens carry a `token_version` claim re-checked against the `AdminUsers` row on every request; a version bump (on lock, status change, password reset, invite acceptance) invalidates all live admin sessions immediately (`Program.cs:296-322`). This is the strongest session-invalidation control in the system.

**Device binding.** ✅ New-device detection with an OTP gate before token issue; single-device model deletes other devices and revokes their refresh tokens on a new-device login (`AuthService.cs:296-348`). The Personal device-verify OTP is dispatched to the Personal membership's verified phone, so a SIM swap on an SME number cannot compromise Personal.

**Biometric auth.** ✅ Personal only; secret is `RandomNumberGenerator.GetBytes(64)`, stored BCrypt-hashed in Redis keyed to a specific device, returned once to the client (`AuthService.cs:768-805`).

**Step-up on product switch.** ✅ Cross-product switch requires the target product's credential; per-target rate limit (10/min) and step-up lockout (5/30 min) in Redis; same-product passwordless only when the signed `authenticated_product` claim matches (`IdentityService.cs:1660-1863`).

### 5.2 Weaknesses and gaps

- ⚠️ **No MFA/2FA for admin users** — password + BCrypt only; mitigations are `token_version`, lockout, and email invite tokens. (A.8.5)
- ⚠️ **No attempt limit on standalone SME/Corporate transaction-PIN verify** — `SmeService.cs:2014`, `CorporateEntityService.cs:359` allow unlimited PIN guesses on those endpoints. The SME *transfer* path does enforce a lock.
- ⚠️ **SME transfer PIN lock is keyed by the client-supplied idempotency key** (`SmeService.cs:1565`) — a new key resets the 3-attempt hard lock, so it is bypassable.
- ⚠️ **Non-admin access tokens are not revocable before expiry.** Logout and password change revoke *refresh* tokens, but live JWTs (identity 15 min, product-scoped 4 h, legacy Personal/Corporate 7 days) remain valid. No per-user `token_version`.
- ⚠️ **`TestDeviceOtp` static per-account device-bypass** persists with no expiry and uses plain equality; if ever set on a real account it bypasses new-device verification in production (`Identity.cs:80`; `IdentityService.cs:1494-1499`).
- ⚠️ **`EchoCodeForDev` / `234999` test-phone bypass** returns plaintext OTPs in any non-Production environment; the sole guard is the `ASPNETCORE_ENVIRONMENT` value.
- ⚠️ **Reset-token comparison is not constant-time** (plain string compare) in both legacy and identity reset paths (`AuthService.cs:627`, `IdentityService.cs:3259`). The admin invite path correctly uses `CryptographicOperations.FixedTimeEquals`.
- ⚠️ **BCrypt work factor is the library default (~10–11)**, not explicitly tuned; PINs are 4 digits (Personal transaction PIN) / 6 digits (login passcode).
- ⚠️ **JWT `ClockSkew` is the default 5 minutes** and no `ValidAlgorithms` allow-list is pinned.
- ⚠️ **Long-lived legacy access tokens** — legacy Personal and Corporate access tokens are valid for **7 days** (`JwtService.cs:36, 87`).
- ⚠️ **Refresh verification is an O(n) BCrypt table scan** — all non-revoked tokens are loaded and BCrypt-verified per refresh, a scaling and cost-amplification concern (`AuthService.cs:357-363`).
- ⚠️ **Corporate `FailedLoginAttempts`/`LockoutEnd` fields exist but are never incremented** — corporate lockout relies solely on the identity layer.

---

## 6. Authorization and Access Control (A.5.15, A.5.18, A.8.2, A.8.3)

### 6.1 Controls in place

**JWT bearer scheme with policy-based RBAC.** ✅ Policies registered in `Program.cs:326-343`: `AdminUser` (`scope=admin`), `SuperAdmin` (`scope=admin` + role `SuperAdmin`), Corporate roles (`Owner`/`Admin`/`Maker`/`Checker`/`Viewer`), and `SmeProduct` (`product=Sme`).

**Correct middleware ordering.** ✅ Exception handler → security headers → HTTPS redirect → CORS → rate limiter → **authentication → authorization** → corporate membership guard (`Program.cs:461-476`). Authentication precedes authorization.

**Object-level authorization (IDOR protection) on Personal and SME.** ✅ Controllers derive the user id from the `NameIdentifier` JWT claim (`GetUserId()`), never from the request body/route, and service queries enforce ownership:
- Accounts: `a.Id == accountId && a.UserId == userId` (`AccountService.cs:237`).
- Transactions: `t.Id == transactionId && t.Account.UserId == userId` (`TransactionService.cs:120`).
- Beneficiaries: `b.Id == beneficiaryId && b.UserId == userId` (`BeneficiaryService.cs:141`).

**Product-scoped money-movement access.** ✅ SME fixed-investment endpoints require the `SmeProduct` policy, not a plain identity token, so investing from the SME wallet requires the SME product-scoped JWT obtained through switch-product step-up (`src/Riverly.Api/Controllers/Sme/FixedInvestmentController.cs:10-18`). Admin fixed-investment mutations require a `SuperAdmin` JWT; the legacy `X-Admin-Key` no longer satisfies `HasSuperAdminAccess` (`src/Riverly.Api/Common/AdminAccess.cs:26-35`; `src/Riverly.Api/Controllers/Admin/AdminFixedInvestmentController.cs:48-70,105-110`).

**Maker-checker (segregation of duties) on Corporate approvals.** ✅ `ApprovalsController` gates submit with `MakerOnly` and approve/reject/return with `CheckerOnly`. The service layer blocks self-approval (`InitiatedById == checkerId`), blocks duplicate approval by the same checker, and verifies the checker belongs to the organization (`ApprovalService.cs:192-195, 342-346`). Organization id is JWT-derived, not client-supplied, on these paths.

**Corporate live offboarding.** ✅ `CorporateMembershipGuardMiddleware` re-checks the `CorporateUsers` seat (`IsActive` + `Status==Active`) on **every** request bearing a Corporate token and returns 401 immediately on revocation (`middleware/CorporateMembershipGuardMiddleware.cs:37-68`).

**Webhook endpoint authentication.** ✅ Anonymous only where justified and independently authenticated (HMAC — see [§10](#10-third-party-integrations-and-webhooks-a519-a520-a522-a821)).

### 6.2 Weaknesses and gaps (this is the highest-risk section)

- ❌ **Confirmed IDOR — cross-org UBO/beneficial-owner PII disclosure.** `GET /corporate/entity/ubos/{organizationId}` filters only by the route-supplied `organizationId` with no check that the caller belongs to that org (`CorporateEntityController.cs:149`; `CorporateEntityService.cs:753-756`). Any Corporate Owner/Admin can read another organization's UBO PII by changing the route id.
- ❌ **Confirmed IDOR — cross-org invite enumeration.** `GET /corporate/entity/invite/{organizationId}` similarly trusts the route id (`CorporateEntityController.cs:243`; `CorporateEntityService.cs:1037-1053`), exposing another org's pending-invite emails/names.
- ⚠️ **Destructive admin operations behind a shared static key.** `AdminIdentityController` (`/wipe`, deletes an identity + all profiles) and `AdminPersonalUserController` (`wipe-by-phone`) are class-level `[AllowAnonymous]` and gated only by `X-Admin-Key` (a single shared secret, constant-time compared) via `AdminAccess.HasAdminAccess`. No per-actor identity (the "reviewer" is a spoofable header), no expiry, no rotation, no lockout (`AdminAccess.cs:10-24, 48-49`; `AdminIdentityController.cs:19, 102-176`).
- ⚠️ **Dual admin authorization model remains on non-investment admin surfaces.** `HasAdminAccess` returns true for **either** a `scope=admin` JWT **or** the shared static key — inconsistent and hard to audit. Privileged fixed-investment mutations have already been migrated to require a SuperAdmin JWT (`AdminAccess.cs:26-35`), but KYB approval, identity wipe, personal-user wipe and bulk/test funding paths still need to be swept so admin writes consistently require named admin identities.
- ⚠️ **No global fallback authorization policy.** `AddAuthorization` sets no `FallbackPolicy`, so any endpoint without an explicit `[Authorize]` is anonymous by default. `ReferenceController` currently has no auth attribute at all. Recommend `FallbackPolicy = RequireAuthenticatedUser` with explicit `[AllowAnonymous]` opt-outs.
- ⚠️ **Class-level `[AllowAnonymous]` on admin controllers** means safety depends on every action remembering to call `IsAdmin()`; a single omission on a new method makes it fully public.
- ⚠️ **Owner/Admin roles span both Maker and Checker policies**, so segregation of duties rests solely on the record-level `InitiatedById != checkerId` check; there is no amount-threshold N-eyes at the policy layer.
- ⚠️ **Live revocation only for Corporate and Admin tokens** — a suspended Personal or SME user keeps access until token expiry.

---

## 7. Cryptography and Secrets Management (A.8.24, A.5.17)

### 7.1 Controls in place

- ✅ **TLS in transit** terminated at the AWS ALB; production listener uses `ELBSecurityPolicy-TLS13-1-2-2021-06`.
- ✅ **Passwords, PINs, tokens BCrypt-hashed** (see [§5](#5-identity-and-authentication-a516-a517-a85)).
- ✅ **JWTs signed HMAC-SHA256.**
- ✅ **Field-level encryption of BVN at rest** — `KycVerification.BvnEncrypted` stores AES-CBC ciphertext with a per-record random IV via `EncryptionService`; key from `Encryption:Key` config (`EncryptionService.cs:15-44`; written at `IdentityService.cs:3607`, `OnboardingService.cs:345`). A separate SHA-256 `BvnHash` supports dedup only.
- ✅ **Secrets injected from AWS Secrets Manager** at runtime — the ECS task definition references every real secret via `secrets[].valueFrom` ARNs; none are inlined (`task-definition.json:129-186`).
- ✅ **No hardcoded secrets in `.cs` source** — grep for key/token/secret patterns across `src/**/*.cs` returned no matches; `Jwt:Secret` is read from config with no code fallback.
- ✅ **Committed `appsettings.json` contains no populated secrets** (empty `Anchor:ApiKey`), and `appsettings.Development.json` is git-ignored and was never committed (verified against git history).
- ✅ **JWT signing-secret rotation runbook** exists with scheduled and emergency variants ([`gitignore_docs/runbook-jwt-secret-rotation.md`](runbook-jwt-secret-rotation.md)).
- ✅ **Committed PEM files contain no private key** — `api.riverly.ng.pem` and `api.riverly.ng-fullchain.pem` hold only `BEGIN CERTIFICATE` blocks (verified: 1 and 3 certificates respectively, zero private-key blocks). These are public TLS certificates.

### 7.2 Weaknesses and gaps

- ⚠️ **AES-CBC without authentication.** `EncryptionService` uses CBC with no MAC/authenticated mode. AES-GCM is recommended for integrity (detects ciphertext tampering). Not currently exploitable because decryption error details are not exposed, but GCM is strictly better. (A.8.24)
- ⚠️ **Encryption key management for production unverified.** `Encryption__Key` does not appear in the staging `task-definition.json` secrets list; confirm the production key is supplied from Secrets Manager and has a documented rotation procedure. There is no KMS/envelope-encryption or key-rotation mechanism in code. (A.8.24)
- ⚠️ **BVN dedup hash is plain SHA-256** (precomputable for an 11-digit space if the DB is dumped). Migrate to HMAC-SHA-256 with a server-side key (tracked follow-up).
- ⚠️ **Single symmetric JWT secret** signs *all* token types including SuperAdmin. A leak allows minting any identity. Consider a separate signing key for admin tokens, or asymmetric RS256 with the private key in KMS.
- ⚠️ **Real sandbox credential in a local dev file.** `appsettings.Development.json` (git-ignored, never committed) holds a real-looking Anchor **sandbox** API key and a base64 dev encryption key on developer disks — should be replaced with placeholders / .NET user-secrets. (A.5.17)
- ⚠️ **Dev DB password committed** in `src/Riverly.Api/docker-compose.yml:7` (`riverly_dev$01`, local-only, no env interpolation).
- ⚠️ **`*.pem` is not in `.gitignore`** — the committed certs are harmless, but the pattern risks a future private-key commit. Add `*.pem` / `*.key` to `.gitignore`.
- ℹ️ **RDS/EBS encryption-at-rest** is expected to use AWS-managed KMS keys; confirm in the AWS console ([§16](#16-evidence-requested-and-out-of-scope-items)).

---

## 8. API and Transport Security (A.8.20, A.8.21, A.8.26, A.8.28)

### 8.1 Controls in place

- ✅ **Security headers middleware** sets `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `Referrer-Policy: no-referrer`, and a restrictive `Permissions-Policy` on every response, including 401/403/500 (via `Response.OnStarting`) (`middleware/SecurityHeadersMiddleware.cs`).
- ✅ **HSTS** enabled outside Development (`app.UseHsts()`); **HTTPS redirect** registered (`Program.cs:457-465`).
- ✅ **CORS allow-list** — explicit origins (app/admin/web front-ends), not `AllowAnyOrigin`, and compatible with credentials (`Program.cs:262-277`).
- ✅ **Rate limiting** — .NET 8 built-in fixed-window limiter partitioned per client IP: `otp` 5/min, `auth` 10/min, `pin` 5/min, `lookup` 10/min, `tin-verification` 5/min. Applied to login, OTP, signup, SME PIN, and lookup endpoints (`Program.cs:346-386`).
- ✅ **SQL injection resistance** — data access is EF Core (parameterized). The only raw SQL is `SELECT … FOR UPDATE` row locks using positional/interpolated parameters (`TransferService.cs:100`, `SmeService.cs:1735`, `FixedInvestmentService.cs`); no string-concatenated SQL, no Dapper, no `ExecuteSqlRaw` with concatenation.
- ✅ **Input validation** — DataAnnotations (`[Required]`, `[EmailAddress]`, `[StringLength]`, `[RegularExpression]`) across ~75 DTOs, enforced by `[ApiController]`, with a custom friendly error envelope (`Program.cs:41-78`; `Validation/FriendlyValidationMessages.cs`).
- ✅ **Input sanitization helper** — `InputSanitizer.CleanFreeText` strips HTML/script/dangerous URL schemes (applied to SME transfer narration, search, nickname).
- ✅ **Idempotency on money movement** — `IdempotencyRecord` keys on Personal wallet/bank transfers, SME transfers (re-checked inside the DB lock), and bill payments (`TransferService.cs:77-84`; `SmeService.cs:1708-1729`; `BillPaymentService.cs`).
- ✅ **Global exception handler** returns a generic JSON envelope and logs the full exception server-side; no stack traces leaked to clients in the general case (`middleware/GlobalExceptionHandler.cs`).
- ✅ **Per-action request size limits** on uploads (KYC 5 MB, SME docs 3–10 MB), with MIME allow-lists and **magic-byte verification** for SME image uploads (`SmeService.cs:355-399`).

### 8.2 Weaknesses and gaps

- ⚠️ **Swagger/OpenAPI exposed unauthenticated in staging/production** — `UseSwagger()` / `UseSwaggerUI()` are registered unconditionally, so the full API schema (including admin/corporate/money-movement routes) is publicly enumerable (`Program.cs:452-456`). Gate behind `IsDevelopment()` or authentication.
- ⚠️ **No global/default rate limit** — endpoints without an explicit `[EnableRateLimiting]` are unthrottled. Notably, **Personal transfer endpoints and bill-payment endpoints have no rate-limit attribute**, and the `ip_policy` (20/min) limiter is defined but never applied.
- ⚠️ **Error-message leak on one path** — `InvalidOperationException => (500, "Server configuration error: " + exception.Message)` echoes the raw exception message to clients in all environments (`GlobalExceptionHandler.cs:42`); EF/config internals can surface. Return a generic message.
- ⚠️ **No global request body size limit** — Kestrel default (30 MB) applies to all non-annotated endpoints; no `MaxRequestBodySize`/`MultipartBodyLengthLimit`.
- ⚠️ **CORS includes plaintext `http://localhost:*` and broad `*.vercel.app` origins with credentials** in the production-built policy; these are hardcoded and not environment-gated.
- ⚠️ **`AllowedHosts = "*"`** — no host filtering (`appsettings.json:8`).
- ⚠️ **Rate-limit partition trusts the first `X-Forwarded-For` hop** with no `UseForwardedHeaders` known-proxy allow-list, so a spoofed header can evade per-IP limits (`Program.cs:353-361`).
- ⚠️ **`InputSanitizer` is applied to only 3 SME fields** — Personal/Corporate free-text (transfer notes, support tickets, beneficiary names) is not routed through it.
- ⚠️ **SVG allowed for SME logo upload** — SVG can carry script; stored-XSS risk if ever served inline rather than as an attachment.
- ⚠️ **No API-versioning framework** — versioning is route-string convention (`api/v1/...`) only; no deprecation strategy.

---

## 9. Data Protection and Privacy (A.8.10, A.8.11, A.8.12, A.5.34)

### 9.1 Controls in place

- ✅ **BVN encrypted at rest** (AES-CBC, per-record IV) — see [§7](#7-cryptography-and-secrets-management-a824-a517).
- ✅ **S3 documents private** — `UploadObjectAsync` sets `S3CannedACL.Private` and SSE-S3 AES256; downloads are via **presigned URLs with a 15-minute TTL over HTTPS**; filenames sanitized with a GUID prefix (`S3StorageService.cs:52-91`).
- ✅ **Payment audit log masks and sanitizes** — `PaymentAuditService` masks account numbers (`012***7890`) and strips digit runs from free text; the entity contract explicitly forbids storing PINs/full account numbers/tokens/passwords (`PaymentAuditLog.cs:14-23`; `PaymentAuditService.cs:103-116`).
- ✅ **OTP codes never logged** — OTP/phone/email are masked in all log lines (`OtpService.cs:37,54`; `SendchampPhoneOtpService.cs:107`; `EmailService.cs:57`).
- ✅ **Webhook raw bodies are not persisted** — handlers parse transiently and re-verify against the provider API rather than storing untrusted payloads.

### 9.2 Weaknesses and gaps

- ❌ **NIN stored unencrypted.** `KycVerification.NinEncrypted` is declared but **never written** by any service. NIN is also placed in a Dojah request URL (`DojahService.cs:94`) and used as an in-memory cache key (`:70`). BVN is protected; NIN is not. (A.5.34, A.8.24)
- ❌ **SME proprietor BVN stored plaintext** — `SmeProfile.OwnerBvn` (`SmeProfile.cs:67`) bypasses `EncryptionService`.
- ⚠️ **ID document number and BVN-derived PII plaintext** — `KycVerification.DocumentNumber` (`:74`), BVN name/DOB/phone fields (`:26-44`).
- ❌ **Full PII payload logged to CloudWatch.** `AnchorBusinessService.cs:195` logs the entire business-customer creation payload (owner names, BVN, ID, address) at Information level; Dojah full response bodies are logged on failure (`DojahService.cs:43,103`). (A.8.12 Data leakage prevention)
- ⚠️ **BVN/NIN in outbound request URIs** — `DojahService.cs:34,94` (`?bvn=...`, `?nin=...`), captured by any HTTP/proxy access logging.
- ⚠️ **TOTP secret and `TestDeviceOtp` stored plaintext** (`User.cs:67,90`).
- ⚠️ **No systematic masking in API responses** — account numbers and KYC fields are returned in full; masking exists only in the audit table and log lines. (A.8.11)
- ❌ **No data retention / deletion / anonymization.** No soft-delete global filter, no PII purge on deactivation, no data-subject export/erasure endpoint; KYC data and S3 documents are retained indefinitely. Only beneficiaries have a soft delete. (A.8.10 — an NDPR obligation)
- ⚠️ **Plaintext transfer payloads at rest** — `CorporateApprovalRequest.PayloadJson` and `IdempotencyRecord.ResponsePayload` store full serialized requests/responses (may include destination account numbers, balances) unmasked.
- ⚠️ **DB SSL not enforced in the connection string** — no `SslMode=Require`; the Npgsql default is opportunistic. Confirm the production connection string enforces TLS ([§16](#16-evidence-requested-and-out-of-scope-items)).

---

## 10. Third-Party Integrations and Webhooks (A.5.19, A.5.20, A.5.22, A.8.21)

### 10.1 Integration inventory

| Integration | Purpose | Auth mechanism | Service file |
|---|---|---|---|
| Providus / XpressWallet | NUBAN wallets, transfers, name enquiry | `Authorization: Bearer` | `ProvidusWalletService.cs` |
| Anchor (BaaS/KYB) | Business/KYB, deposit accounts, NIP transfers | `x-anchor-key` header | `AnchorBusinessService.cs` |
| Anchor Bill Payments | Airtime/data/electricity/TV | `x-anchor-key` header | `AnchorPaymentProvider.cs` |
| Sendchamp | Phone OTP + generic SMS | `Authorization: Bearer` | `SendchampPhoneOtpService.cs`, `TwilioService.cs` |
| Liberty CBS ("Vantage") | Core banking / ledger | `apikey` + `Shortname` headers | `CoreBankingService.cs` |
| Dojah | KYC (BVN/NIN) | `Authorization` + `AppId` headers | `DojahService.cs` |
| Resend | Email delivery | `Resend:ApiKey` | `ResendEmailService.cs` |
| AWS S3 | Document storage | static access key/secret | `S3StorageService.cs` |
| Firebase FCM | Push notifications | service-account JSON | `FirebaseFcmService.cs` |

All base URLs are `https://`; no custom certificate-validation callbacks are registered, so default OS TLS validation applies. All credentials are config-driven (Secrets Manager in deployed envs). No user-supplied URLs reach outbound calls — no SSRF surface.

### 10.2 Inbound webhook controls (money-affecting)

- ✅ **Providus webhook** (`POST api/v1/webhooks/providus`) — HMAC-SHA512 (hex) over the raw body, header `x-xpresswallet-signature`, constant-time compare, **fail-closed** on missing secret/header/bad signature (401). Deposit crediting re-fetches the transaction from the Providus API before crediting (`ProvidusWebhookController.cs:34-136`; `ProvidusWebhookHandler.cs:65-107`).
- ✅ **Anchor webhook** (`POST api/v1/webhooks/anchor`) — HMAC-SHA1 (Base64) over the raw body, header `x-anchor-signature`, constant-time compare, fail-closed. Deposits and settlements re-verify via `VerifyTransactionAsync` and trust the provider's status, not the event name (`AnchorWebhookController.cs:39-110`; `AnchorWebhookHandler.cs:123-142`).

### 10.3 Outbound transfer controls

- ✅ **Server-side name enquiry gate** — bank-returned account name overrides the client-supplied name on both Personal bank transfers (`TransferService.cs:372-399`) and SME transfers, which additionally require a recent (10-min) cached successful name enquiry before allowing the transfer (`SmeService.cs:1618-1639`).
- ✅ **Limit and balance checks** — single-transaction limit, daily/cumulative debit caps (counting in-flight Pending), and available-balance checks before debit on Personal and SME paths.
- ✅ **SME transfer robustness** — serializable transaction + `SELECT … FOR UPDATE`, idempotency re-checked inside the lock, reserve-on-initiation with settle/refund on webhook or sweeper (`SmeService.cs:1705-1848`).
- ✅ **Test-fund money-creation endpoints are production-gated** (`IHostEnvironment.IsProduction()` 403) on both SME and Corporate.

### 10.4 Weaknesses and gaps

- ⚠️ **Non-deposit webhook paths trust unverified payload data.** Providus bank-transfer/batch handlers execute refund/credit-back branches based on the payload alone (`ProvidusWebhookHandler.cs:279-291, 433-443`); the HMAC signature is the sole protection there (deposit paths re-verify, these do not).
- ⚠️ **Idempotency race on Personal transfers and all bill payments.** The idempotency existence check runs *before* the `FOR UPDATE` lock and there is no unique DB constraint on the idempotency key, so two concurrent same-key requests can both call the provider → duplicate money movement (`TransferService.cs:79-87`; `BillPaymentService.cs:48`). The SME path already fixed this by re-checking inside the lock.
- ⚠️ **SME settlement double-apply race** — the Anchor settlement webhook and the stale-pending sweeper both settle Pending debits without a row lock on the transaction/account (`AnchorWebhookHandler.cs:110-226` vs `Jobs/SmeStalePendingTransferSweeper.cs:97-157`).
- ⚠️ **Core-banking debit/reversal can silently degrade to a local-only ledger posting** — `CoreBankingService.DebitAccountAsync`/`ReversalAsync` return synthetic success (`LOCAL-{ref}`) when the settlement account is unset or the endpoint 404s, so a "debit"/"reversal" may never post at the bank (`CoreBankingService.cs:380-387, 480-509`). This is a ledger-integrity and reconciliation risk.
- ⚠️ **No replay/timestamp/event-id protection at the webhook controllers** — dedup is only downstream by transaction reference.
- ⚠️ **No retry / circuit-breaker (no Polly)** — only per-client `HttpClient.Timeout`. Provider-call-before-DB-commit ordering on Personal transfers leaves reconciliation-only recovery on DB failure.
- ⚠️ **Webhook handlers are fire-and-forget** (`Task.Run` after returning 200 OK) — an event is lost if the process crashes before processing; no durable queue.
- ⚠️ **Providus HMAC secret is the same value as its API Bearer key** — compromise of one compromises both. Anchor uses HMAC-**SHA1** (provider-dictated, but weaker).
- ⚠️ **Anchor environment mismatch** — the two Anchor HttpClients have different DI base-URL defaults (one sandbox, one live); if `Anchor:BaseUrl` is unset they hit different environments.
- ⚠️ **Corporate test-fund creditable by the lowest `CorporateViewer` role** on staging.

---

## 11. Infrastructure, CI/CD and Secure Development (A.8.9, A.8.25, A.8.29, A.8.31)

### 11.1 Controls in place

- ✅ **CI/CD via GitHub OIDC — no long-lived AWS keys in CI.** Both `deploy.yml` (staging, push to `main`) and `deploy-production.yml` (manual `workflow_dispatch`) assume dedicated IAM roles via `aws-actions/configure-aws-credentials@v4` with `id-token: write`; separate staging/prod roles.
- ✅ **Production deploy is guarded** — manual trigger requiring a commit SHA, verified to be an ancestor of `origin/main`, image **promoted from ECR** (not rebuilt), a pre-deploy RDS snapshot, a `production` GitHub environment for reviewer approval, and separate cluster/DB/role (`deploy-production.yml`).
- ✅ **Secret scanning in CI** — `gitleaks` runs on every PR to `main` plus a weekly full-history sweep (`.github/workflows/gitleaks.yml`).
- ✅ **Dependency automation** — Dependabot covers NuGet and GitHub Actions weekly (`.github/dependabot.yml`).
- ✅ **Secrets via Secrets Manager, not plaintext env** in the ECS task definition; centralized `awslogs` logging; container health check on `/health`.
- ✅ **No `bin/`/`obj/` build artifacts committed** (verified); `appsettings.Development.json`, `task-definition.json`, and `*.py` test scripts are git-ignored.
- ✅ **Multi-stage Dockerfile** (SDK build → aspnet runtime + Caddy proxy); TLS terminates at the ALB.
- ✅ **Environment separation** — see [§4](#4-scope-architecture-and-environment-separation).

### 11.2 Weaknesses and gaps

- ❌ **No automated test/build/security gate before deploy** (A.8.29). A push to `main` deploys to staging with no `dotnet build`/`dotnet test`/SAST gate; the only check is a *post*-deploy smoke test. `gitleaks` runs on PRs, not on the push that actually deploys.
- ⚠️ **Long-lived IAM access keys injected into the running container.** `AWS__AccessKey`/`AWS__SecretKey` are provided as secrets and used by the app despite the ECS **task role** being available (`task-definition.json:130-137`). Prefer the task role; retire the static keys.
- ⚠️ **No IAM privilege separation** — the task role and execution role are the same `ecsTaskExecutionRole` (`task-definition.json:190-191`).
- ⚠️ **Containers run as root**, no `readonlyRootFilesystem`, base images pinned only to floating major tags (no digest pinning) — non-reproducible builds (`Dockerfile`).
- ⚠️ **Production approval/branch protection is comment-documented only** — actual enforcement depends on GitHub repo settings not verifiable from source; the prod workflow header still says "keep disabled until every TODO resolved."
- ⚠️ **Auto-migrate on startup** — `db.Database.Migrate()` runs on every boot with the app's DB role holding DDL rights (`Program.cs:444-449`).
- ⚠️ **Anonymous `/health`** endpoint uses an EF/DB health check whose default detailed output can disclose dependency status to unauthenticated callers (`Program.cs:477`).
- ⚠️ **Parallel local deploy path** — `scripts/deploy-staging.sh` (using the local `riverly-live` AWS profile) can push to ECS outside CI, bypassing `gitleaks`. A committed `scripts/wipe-user-data.sql` truncates ~45 PII tables with no prod guardrail.
- ⚠️ **No central package management or lock files** — non-deterministic restores; `RestoreLockedMode` not enforced.

---

## 12. Logging, Monitoring and Audit Trail (A.8.15, A.8.16, A.5.28)

### 12.1 Controls in place

- ✅ **Structured logging (Serilog)** to console + rolling file; container stdout ships to **CloudWatch Logs** (`/ecs/riverly-api`, `/ecs/riverly-api-prod`).
- ✅ **Sensitive data masked in logs** — OTP codes never logged; phone/email/account masked (see [§9](#9-data-protection-and-privacy-a810-a811-a812-a534)).
- ✅ **Append-only payment audit log** — `PaymentAuditLog` records NameLookup / TransferCreated / PinAttempt / TransferCompleted with masked account numbers and sanitized free text; write failures are swallowed so they never break the money flow (`PaymentAuditService.cs`).
- ✅ **Structured security events** — `security.product_switch`, login-failure, and `security.global_signout` log lines are emitted for CloudWatch metric filters.
- ✅ **Admin action versioning** — `token_version` changes provide a durable signal of admin lock/reset events.

### 12.2 Weaknesses and gaps

- ❌ **No cloud monitoring or alerting as of 14 July 2026** (A.8.16). Verified against the live account: **zero CloudWatch alarms, no SNS topics, GuardDuty off, no CloudTrail trail, no Security Hub/Config, ALB access logs off, RDS log export off.** If the prod API dies, a payout silently fails, or an AWS key is stolen, nobody is notified. Remediation is planned and scripted — see [Appendix B](#18-appendix-b--operational-runbooks-and-plans). Because this is live AWS state, re-run and attach the evidence commands in `gitignore_docs/cloud-logging-security-alerting-plan.md` before submitting the audit pack.
- ❌ **PII leaked into logs** — full KYC payloads and provider responses at Information level (see [§9](#9-data-protection-and-privacy-a810-a811-a812-a534)). This directly undermines log-handling controls.
- ⚠️ **Payment-audit immutability is code-convention only** — no PostgreSQL trigger or grant restriction enforces append-only; the table is mutable/deletable by the app role, and there is no hash-chaining/tamper evidence (`PaymentAuditLog.cs:20-23`). (A.5.28 collection of evidence)
- ⚠️ **No general audit trail for admin actions or logins** — admin freeze/suspend/KYC approve-reject only `LogWarning`; there is no persisted, queryable admin audit table. `CreatedBy`/`UpdatedBy` columns are last-writer-wins, not an audit log.
- ⚠️ **Log retention is 30 days** on the prod log group — short for a financial product where disputes surface months later (planned bump to 365 days).
- ⚠️ **Staging load balancer TLS/redirect issues** (verified 14 July) — staging HTTPS listener accepts TLS 1.0/1.1 (`ELBSecurityPolicy-2016-08`) and staging port 80 forwards plain HTTP instead of redirecting; staging holds real original user accounts. One-line listener fixes, planned.

---

## 13. Consolidated Gaps Register (Risk-Ranked)

Ranked by exploitability × impact for a fintech holding customer funds and BVN/NIN. "Ref" links to the detailed section.

| # | Severity | Finding | Ref | Recommended action |
|---|---|---|---|---|
| 1 | **High** | Cross-org IDOR: Corporate UBO PII readable by any org admin | [§6](#6-authorization-and-access-control-a515-a518-a82-a83) | Add caller-membership check to `GetUbosAsync` |
| 2 | **High** | Cross-org IDOR: Corporate invite list enumerable | [§6](#6-authorization-and-access-control-a515-a518-a82-a83) | Add caller-membership check to `GetInvitesAsync` |
| 3 | **High** | NIN stored unencrypted (field exists, never written) | [§9](#9-data-protection-and-privacy-a810-a811-a812-a534) | Encrypt NIN with `EncryptionService`; remove from URLs/cache keys |
| 4 | **High** | Full KYC PII (BVN/ID/address) logged to CloudWatch | [§9](#9-data-protection-and-privacy-a810-a811-a812-a534), [§12](#12-logging-monitoring-and-audit-trail-a815-a816-a528) | Redact provider request/response bodies before logging |
| 5 | **High** | Destructive admin ops behind shared static `X-Admin-Key` | [§6](#6-authorization-and-access-control-a515-a518-a82-a83) | Migrate all admin mutations to SuperAdmin JWT; retire the key |
| 6 | **High** | No cloud monitoring/alerting (alarms, GuardDuty, CloudTrail) | [§12](#12-logging-monitoring-and-audit-trail-a815-a816-a528) | Execute the alerting plan (Appendix B) |
| 7 | **Med-High** | Money-movement idempotency race (Personal, bills) | [§10](#10-third-party-integrations-and-webhooks-a519-a520-a522-a821) | Unique DB constraint on idempotency key + re-check inside lock |
| 8 | **Med-High** | CBS debit/reversal silently degrades to local-only posting | [§10](#10-third-party-integrations-and-webhooks-a519-a520-a522-a821) | Fail loudly when no real GL posting occurs; reconcile |
| 9 | **Medium** | Swagger exposed unauthenticated in prod/staging | [§8](#8-api-and-transport-security-a820-a821-a826-a828) | Gate behind `IsDevelopment()` or auth |
| 10 | **Medium** | No MFA for privileged admin accounts | [§5](#5-identity-and-authentication-a516-a517-a85) | Add TOTP MFA to admin login |
| 11 | **Medium** | No global rate limit; Personal transfers unthrottled | [§8](#8-api-and-transport-security-a820-a821-a826-a828) | Add a default limiter + PIN limiter on transfers |
| 12 | **Medium** | SME proprietor BVN + ID document number plaintext | [§9](#9-data-protection-and-privacy-a810-a811-a812-a534) | Route through `EncryptionService` |
| 13 | **Medium** | No test/SAST gate before deploy | [§11](#11-infrastructure-cicd-and-secure-development-a89-a825-a829-a831) | Require build+test (and gitleaks) as deploy predecessors |
| 14 | **Medium** | Long-lived IAM keys in container; role == exec role | [§11](#11-infrastructure-cicd-and-secure-development-a89-a825-a829-a831) | Use the ECS task role; split roles; least privilege |
| 15 | **Medium** | Unlimited-guess PIN verify (SME/Corporate standalone) + resettable SME lock | [§5](#5-identity-and-authentication-a516-a517-a85) | Add lockout; key the lock on identity, not idempotency key |
| 16 | **Medium** | Webhook non-deposit paths trust unverified payload; no replay protection | [§10](#10-third-party-integrations-and-webhooks-a519-a520-a522-a821) | Re-verify via provider API; add event-id dedup |
| 17 | **Low-Med** | No data retention/deletion (NDPR erasure) | [§9](#9-data-protection-and-privacy-a810-a811-a812-a534) | Build purge/anonymization + data-subject export |
| 18 | **Low-Med** | Payment-audit immutability code-only; no admin audit trail | [§12](#12-logging-monitoring-and-audit-trail-a815-a816-a528) | DB trigger for append-only; persist admin actions |
| 19 | **Low-Med** | Non-admin access tokens not revocable before expiry | [§5](#5-identity-and-authentication-a516-a517-a85) | Add per-user `token_version` |
| 20 | **Low-Med** | Staging LB accepts TLS 1.0/1.1; port 80 not redirected | [§12](#12-logging-monitoring-and-audit-trail-a815-a816-a528) | Update listener policy + 301 redirect |
| 21 | **Low** | AES-CBC (no MAC); single JWT secret for all tokens incl. admin | [§7](#7-cryptography-and-secrets-management-a824-a517) | Move to AES-GCM; separate admin signing key / RS256 |
| 22 | **Low** | Error message leak on `InvalidOperationException` | [§8](#8-api-and-transport-security-a820-a821-a826-a828) | Return generic 500 message |
| 23 | **Low** | Reset-token compare not constant-time | [§5](#5-identity-and-authentication-a516-a517-a85) | Use `FixedTimeEquals` |
| 24 | **Low** | `TestDeviceOtp` static bypass persists with no expiry | [§5](#5-identity-and-authentication-a516-a517-a85) | Ensure null in prod; add TTL; restrict to non-prod |
| 25 | **Low** | Dev DB password + sandbox key on disk; `*.pem` not gitignored | [§7](#7-cryptography-and-secrets-management-a824-a517) | Use user-secrets; add `*.pem`/`*.key` to `.gitignore` |
| 26 | **Info** | Held for external decision | [§16](#16-evidence-requested-and-out-of-scope-items) | External pen test; CAPTCHA; edge IP rate limits |

---

## 14. Regulatory Alignment (CBN, NDPR, NIBSS/NPS)

**CBN AML / KYC.** Customer identification via BVN (Personal) and KYB (SME); the framework supports KYC-tier transaction limits via user KYC status. Name-enquiry-verified beneficiary names on transfers.

**NDPR (Nigeria Data Protection Regulation).** Encryption in transit (TLS) and partial encryption at rest (BVN). **Gaps against NDPR:** NIN unencrypted ([§9](#9-data-protection-and-privacy-a810-a811-a812-a534)), PII in logs, and **no data-subject erasure/retention mechanism** (A.8.10). A breach-notification playbook (72-hour obligation) and NDPR data-controller registration are organizational items to confirm.

**NIBSS / NPS (Nigeria Payments System) checklist.** The internal NPS assessment ([`gitignore_docs/nps_security_accessment.txt`](nps_security_accessment.txt)) records the platform as meeting the large majority of compulsory NPS controls (TLS 1.2+, encryption at rest/in transit, RBAC, audit trails, dynamic liveness for account opening, transaction limits, VAPT, MFA for critical systems, NIBSS fraud-management integration, IP whitelisting), with one item marked "NO" (Windows LAPS local-admin management — not applicable to this Linux/container stack). This document's findings should be reconciled against that self-assessment — in particular the MFA-for-critical-systems and comprehensive-audit-trail claims, which this review found are only partially met at the application layer.

> **Note for assessors:** several NPS checklist answers assert controls (e.g. bi-annual VAPT, digital signatures for message integrity, MFA) that are organizational or infrastructure-level. Where this application-layer review found a divergence — MFA for admin, application audit-trail immutability — the code-level finding is the authoritative statement of the *current* implementation.

---

## 15. Vulnerability Management and Dependencies (A.8.8)

- ✅ **Dependabot** monitors NuGet and GitHub Actions weekly.
- ✅ **gitleaks** secret scanning on PRs + weekly sweep.
- ⚠️ **Outdated/mismatched packages flagged:** `Microsoft.AspNetCore.Authentication.JwtBearer 8.0.0` (unpatched base — JWT validation fixes land in 8.0.x patches), a legacy `Microsoft.AspNetCore.Authentication 2.3.9` reference, `Microsoft.AspNetCore.Http.Features 5.0.17` on a net8 target, mixed EF Core 8.0.0/8.0.8, `Microsoft.Extensions.* 10.0.3` on net8, and an **unused `Twilio 7.14.3`** dependency (Twilio was refactored out).
- ❌ **No SAST/dependency-audit gate** in CI (no `dotnet list package --vulnerable`, no CodeQL); no central package management or lock files.
- ❌ **No external penetration test** performed yet (held pending budget) — this is the single most valuable addition for an ISO 27001 posture. (A.8.29)

**Recommended:** pin JwtBearer to the latest 8.0.x, remove the legacy 2.x/5.x/unused packages, add `dotnet list package --vulnerable`/CodeQL to CI as a required gate, adopt `Directory.Packages.props` + committed lock files, and commission a professional pen test before scaling transaction volume.

---

## 16. Evidence Requested and Out-of-Scope Items

**Evidence to request from the AWS/infrastructure owner (not verifiable from source):**

- RDS encryption-at-rest confirmation (KMS) for both databases.
- Production connection string enforces `SslMode=Require`.
- Production ECS task definition (`riverly-task-prod`) container security config — user, secrets, task vs execution role separation.
- S3 bucket-level "Block Public Access" enabled on `riverly-*-assets`.
- Automated RDS backup schedule, encryption, and retention (A.8.13).
- GitHub repo settings: branch protection on `main`, required status checks (gitleaks), and required reviewers on the `production` environment.
- Production `Encryption__Key` supplied from Secrets Manager.

**Out of scope for this application-layer review:**

- **Frontend/mobile security** — token storage on device, root/jailbreak handling, deep-link handling (owned by the mobile team).
- **Physical and people controls** (A.6, A.7) — employee laptop policy, office access, onboarding/offboarding.
- **Cloud infrastructure hardening** beyond what the repo reveals — VPC/security-group design, WAF rules, network segmentation.
- **Legal/regulatory registration** — NDPR data-controller registration, CBN licensing terms.

---

## 17. Appendix A — Key File Reference Index

| Area | Primary files |
|---|---|
| Host, middleware, policy config | `src/Riverly.Api/Program.cs` |
| Security headers | `src/Riverly.Api/middleware/SecurityHeadersMiddleware.cs` |
| Global error handling | `src/Riverly.Api/middleware/GlobalExceptionHandler.cs` |
| Corporate offboarding guard | `src/Riverly.Api/middleware/CorporateMembershipGuardMiddleware.cs` |
| Admin access helper | `src/Riverly.Api/Common/AdminAccess.cs` |
| Identity / login | `src/Riverly.Infrastructure/Services/IdentityService.cs` |
| Legacy auth | `src/Riverly.Infrastructure/Services/AuthService.cs` |
| Admin auth | `src/Riverly.Infrastructure/Services/AdminAuthService.cs` |
| JWT | `src/Riverly.Infrastructure/Services/JwtService.cs` |
| OTP | `src/Riverly.Infrastructure/Services/OtpService.cs`, `SendchampPhoneOtpService.cs` |
| Encryption | `src/Riverly.Infrastructure/Services/EncryptionService.cs` |
| Money movement | `TransferService.cs`, `SmeService.cs`, `BillPaymentService.cs`, `CoreBankingService.cs` |
| Webhooks | `Controllers/Webhooks/ProvidusWebhookController.cs`, `AnchorWebhookController.cs` + handlers |
| Storage | `src/Riverly.Infrastructure/Services/S3StorageService.cs` |
| Audit | `src/Riverly.Domain/Entities/PaymentAuditLog.cs`, `PaymentAuditService.cs` |
| CI/CD | `.github/workflows/deploy.yml`, `deploy-production.yml`, `gitleaks.yml`, `.github/dependabot.yml` |
| Deployment | `task-definition.json`, `src/Riverly.Api/Dockerfile`, `docker-compose.yml` |

## 18. Appendix B — Operational Runbooks and Plans

- **JWT signing-secret rotation** — scheduled and emergency variants, dual-signing window, refresh-token global revoke SQL: [`gitignore_docs/runbook-jwt-secret-rotation.md`](runbook-jwt-secret-rotation.md).
- **Cloud logging and security alerting plan** (verified against the live account 14 July 2026) — SNS alert channel, payout-failure alarms, GuardDuty, uptime/health alarms, CloudTrail, log-retention increase, staging TLS fixes, key-management hardening: [`gitignore_docs/cloud-logging-security-alerting-plan.md`](cloud-logging-security-alerting-plan.md).
- **Environment and credentials matrix** — sandbox vs live per provider per environment: [`docs/environment-credentials.md`](../docs/environment-credentials.md).
- **Prior internal security audit** (OWASP Top 10 mapping, endpoint-by-endpoint walk, F-1…F-15 findings and remediation via PR #98): [`gitignore_docs/security-audit.md`](security-audit.md).
- **NPS/NIBSS security checklist self-assessment:** [`gitignore_docs/nps_security_accessment.txt`](nps_security_accessment.txt).

---

*End of document. This inventory reflects the codebase and live-account state at 14 July 2026. It is an internal engineering review intended to support, not replace, an external penetration test and formal ISO 27001 audit.*
