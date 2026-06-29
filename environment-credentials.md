# Environment & Credentials — what's sandbox vs live where

**Last updated:** 2026-06-29
**Owner:** Yerins

This is the single source of truth for which credentials each environment uses for each provider. Read it before running tests, sending money, or assuming what staging will or won't accept.

If you change any of this, update this doc in the same PR.

---

## Environments at a glance

| | Staging | Production |
|---|---|---|
| API URL | `https://api-staging.riverly.ng` | `https://api.riverly.ng` |
| Mobile build flavor | `staging` (app id `com.riverly.personal.staging`, name "Riverly Staging") | `production` (app id `com.riverly.personal`, name "Riverly") |
| AWS RDS | `riverly-staging-db` | `riverly-prod-db` (separate, empty at launch) |
| ECS cluster | `riverly-cluster-0` | `riverly-cluster-prod` |
| Redis (ElastiCache) | `riverly-staging-redis` | `riverly-prod-redis` |
| Secrets Manager entry | `riverly/staging/config` | `riverly/production/config` |
| Touches real money? | **No.** All test rails. | **Yes.** Real customers, real money. |

Both phones can have both apps installed at the same time (different bundle IDs).

---

## Provider-by-provider

Each row = one external dependency, and what credential tier it runs against in each env.

| Provider | Staging | Production | Notes |
|---|---|---|---|
| **Providus / XpressWallet** | Sandbox (`sk_sandbox_...`) | Live (`sk_live_...`) | Properly separated. Staging signups create sandbox NUBANs that can't move real money. Prod uses live NUBANs with real settlement. |
| **Dojah backend** (server-to-server BVN/NIN) | Live (`prod_sk_...`) | Live (`prod_sk_...`) | Same live keys in both envs. Every BVN/NIN looked up on staging hits the real Dojah API. You can't spam fake BVNs against staging — Dojah will reject them. |
| **Dojah mobile widget** | Staging app id `6a218fadf7001f0532c2a569` | Prod app id `6a2991a5776f4e68e2ed79c0` | Different app IDs per env. Each app has its own Sandbox/Live toggle on the Dojah dashboard. Whether you can use test BVNs in the widget depends on what that specific app is set to. |
| **Anchor** (used for bills + SME accounts) | Sandbox (`https://api.sandbox.getanchor.co`) | Live (`https://api.getanchor.co`) | Staging is wired but bills/SME flows in prod still pending Anchor's live activation for our org. |
| **Liberty Core Banking / Vantage** | Stub key (`1234567890`) | Stub key (`1234567890`) | Integration not yet live in either env. Calls to Liberty will fail at runtime in both staging and prod. Pending Fatai's confirmation + real live key. |
| **Sendchamp** (phone OTP via WhatsApp) | Same live key (`sendchamp_live_...`) | Same live key | One live key for the whole company. Sender is `SC-OTP` (Sendchamp's shared sender, no per-app approval needed). Test OTPs to your own phone work in both envs. |
| **Resend** (email) | API key `riverly-sme` (live, scoped to staging) | API key `riverly-prod` (live, scoped to prod) | Two separate API keys under the same Resend account. Sending domain `mail.riverly.ng` is verified for both. |
| **Firebase Cloud Messaging** | Same project `riverly-sme` | Same project `riverly-sme` | Shared Firebase project. Both envs send push from the same FCM sender. Cheap separation if needed later by creating a `riverly-prod` Firebase project. |
| **AWS S3** (KYC docs, statements) | `riverly-staging-assets` (own IAM user) | `riverly-prod-assets` (own IAM user, scoped to this bucket only) | Separate buckets, separate IAM users, separate access keys. |
| **Twilio** | Refactored away. Class still exists but now calls Sendchamp under the hood. | Same. | No Twilio credentials needed anywhere. |
| **SendGrid** | Refactored away. Same pattern as Twilio. | Same. | No SendGrid credentials needed anywhere. |

---

## Test data — what works on staging

### What you CAN do freely on staging (no real money / no real consequences)
- Sign up unlimited test users with any phone number (use `+234999...` prefix to bypass the Sendchamp OTP send and get the code echoed back in the response — staging only).
- Create unlimited sandbox NUBANs via Providus. They can't receive or send real money.
- Fund test accounts via `POST /api/v1/sme/account/test-fund` (SME) or the equivalent on Personal. Refused in prod.
- Send WhatsApp OTPs to your real phone via Sendchamp — they're real WhatsApp messages but cost a small amount of credit (negligible).
- Send real emails via Resend — same sending domain as prod, so don't email random external addresses; stick to team accounts.

### What you CANNOT spam on staging
- **BVN lookups** — Dojah backend is live, every lookup hits the real Dojah and is rate-limited per BVN. Don't run a script that loops through fake BVNs.
- **Anchor sandbox transfers** — Anchor has its own rate limits even on sandbox.
- **Real customer signups** — staging is for the team, not for soft-launching to users.

---

## Common test scenarios — what to use

| I want to test... | Use which env? | How |
|---|---|---|
| New mobile UI changes | Staging | Build with `--flavor staging --dart-define-from-file=dart_defines/staging.json`, install side-by-side with prod app. |
| Real customer signup flow end-to-end (Dojah, NUBAN issuance, etc.) | Production | Production build, real phone number, real BVN. Tiny amount (e.g. ₦100) to verify settlement. |
| A new BE feature before merging to main | Staging via `api-staging.riverly.ng` | Push to a branch → PR → merge to main triggers staging auto-deploy. |
| Pushing a prod deploy | Production | Manual `workflow_dispatch` on `deploy-production.yml` with the verified SHA. See `production-rollout-plan.md` §6.2. |
| A receipt template change | Staging | Trigger a transfer on staging, download the receipt PDF. |
| Confirming a webhook handler works | Staging | Trigger the upstream provider to send a webhook to `api-staging.riverly.ng/api/v1/webhooks/<provider>`. |

---

## Mobile build flavors

The Personal app has two build flavors, set up in `dart_defines/`:

| | Staging | Production |
|---|---|---|
| Android applicationId / iOS bundle id | `com.riverly.personal.staging` | `com.riverly.personal` |
| Home screen name | "Riverly Staging" | "Riverly" |
| `DOJAH_APP_ID` | `6a218fadf7001f0532c2a569` | `6a2991a5776f4e68e2ed79c0` |
| `ENV` | `staging` | `production` |
| Base URL | `https://api-staging.riverly.ng` | `https://api.riverly.ng` |

Both can be installed on the same phone. The build command must always pass both `--flavor` and the matching `--dart-define-from-file=dart_defines/<flavor>.json`.

---

## "I think a credential is wrong" — how to check

1. Open `riverly/staging/config` or `riverly/production/config` in AWS Secrets Manager (eu-north-1).
2. Look up the key (e.g. `Providus__ApiKey`).
3. Cross-check against this doc. If they don't match, this doc is wrong OR the secret is wrong — flag it in the dev channel before changing anything.

For credentials that are hardcoded in the ECS task definition env (rather than in Secrets Manager — staging does this for some Providus values), check:
- Staging: ECS task definition `riverly-task` → container `riverly_staging_container` → environment
- Production: ECS task definition `riverly-task-prod` → container `riverly_api_prod` → environment

---

## What's still TODO (as of 2026-06-29)

These would close out the remaining "?" in the credentials picture:

- **Anchor live activation** — Femi waiting on Anchor enabling our production access.
- **Liberty Core Banking live key** — Fatai to confirm whether Liberty integration is actively used and provide the real key (currently stubbed in both envs).
- **Providus webhook URL** — needs to be set to `https://api.riverly.ng/api/v1/webhooks/providus` on Providus's dashboard so balance updates work in prod.
- **Dojah mobile staging app status** — Tolu to confirm whether `6a218fadf7001f0532c2a569` is currently Sandbox or Live on the Dojah dashboard, depending on testing needs.
