# Production Rollout Plan



## TL;DR

| Topic | Recommendation |
|---|---|
| Branch strategy | **Single `main` branch**, no `production` branch |
| Deploy to staging | Auto on every push to `main` (unchanged) |
| Deploy to production | Manual `workflow_dispatch` pinning a `main` commit SHA |
| Domain | `api-staging.riverly.ng` (staging), `api.riverly.ng` (prod) |
| AWS resources | Fully isolated — own RDS, own ECS, own secrets, own log group |
| Same Docker image | Yes — image SHA promotes from staging to prod, only config differs |
| Staging-only admin endpoints | Refused at the controller layer when env=Production |

---

## 1. Why not a `production` branch?

The most common alternative — `develop` → staging, `main` → production — sounds clean and almost always becomes a mess:

- Hotfixes go to one branch but not the other; you discover this two weeks later.
- The "which branch is real" question gets asked monthly.
- Cherry-picks and merges drift, leading to "this commit is on main but not on staging" mysteries.

**One branch (`main`) avoids all of that.** What changes is *how* prod gets deployed, not *what* gets deployed. Every line of code that hits prod has already been baked on staging, because both come from the same commit on the same branch.

If we ever need to hotfix prod without shipping unrelated staging work, we cut a short-lived branch from the prod-deployed SHA, fix, PR back to `main`, then deploy that SHA to prod. Clean and rare.

---

## 2. Two URLs, two environments

**Current reality (important):** `api.riverly.ng` is *already in use* — it points at our existing cluster, which is wired to Anchor sandbox + Providus and is effectively staging. The deploy workflow even smoke-tests against `https://api.riverly.ng/api/v1/reference/industries` (`.github/workflows/deploy.yml:118`). Every Riverly mobile build, Tolu's web work, and Postman collection out in the wild is pointing here.

**Target state:**

| Environment | URL | Purpose |
|---|---|---|
| Staging | `api-staging.riverly.ng` | Anchor sandbox, Providus (XpressWallet) sandbox, test data, throwaway accounts |
| Production | `api.riverly.ng` | Anchor live, Providus (XpressWallet) live, real customers, real money |

**This requires an explicit domain cutover BEFORE prod infrastructure exists** — see §10.0 below. We don't repurpose `api.riverly.ng` for prod until every client is repointed to `api-staging.riverly.ng`. Cutting over without that sequence means real customer mobile builds hitting whatever happens to be on `api.riverly.ng` at the moment — unacceptable.

**Why separate hostnames matter:** if both environments served from one domain and differentiated only by a header or path prefix, one misconfigured mobile build could point its real customers' transfers at staging (or worse, vice versa). Different DNS makes this physically impossible at the network layer.

Mobile apps get build-time variants: staging build hits staging URL, production build hits production URL. Same for Tolu's web work. The user never sees the URL, but a developer or QA can instantly know which environment they're in.

---

## 3. AWS infrastructure split

Every long-lived resource is duplicated. We share only the things that are immutable / append-only (ECR image registry).

| Resource | Staging | Production |
|---|---|---|
| RDS Postgres | existing instance | **new instance** — fresh, empty |
| ECS cluster + service + task definition | existing | **new** — own cluster |
| ECR repository (Docker images) | existing | **shared** — image SHA promoted to prod |
| Secrets Manager | `riverly/staging/config` | **`riverly/production/config`** |
| CloudWatch log group | `/riverly/staging` | **`/riverly/production`** |
| OIDC deploy role | `riverly-github-actions-deployer` | **`riverly-github-actions-deployer-prod`** |
| Route53 record | `api-staging.riverly.ng` | `api.riverly.ng` |
| ACM certificate | existing | **new for `api.riverly.ng`** |
| ALB / target group | existing | **new** |

Sharing ECR is intentional: it guarantees the bytes that run on prod are the exact bytes we tested on staging. We just point the prod task definition at the same image tag.

---

## 4. Secrets that must be different on prod

Anything that could let a staging credential affect prod (or vice versa) gets a fresh value:

- **Anchor** — production API key (and configure the production webhook URL on Anchor's live portal to point at `api.riverly.ng`)
- **XpressWallet** — production API key + webhook URL
- **JWT signing secret** — brand new. A leaked staging access token must not be valid against prod under any circumstance.
- **Admin API key (`X-Admin-Key`)** — brand new. Knowledge of the prod key should be limited to ops; staging key can stay with developers.
- **Refresh-token hashing pepper** (if any) — brand new.

Things that can be shared without risk:
- **Sendchamp / Resend / Firebase** API keys (different sender IDs / templates if we want stricter separation, but not required for security).

---

## 5. Code changes required before first prod deploy

**These are HARD BLOCKERS, not "small patches before prod."** No prod traffic until all three are merged and verified on staging.

### 5.1 Refuse staging-only admin endpoints when env=Production

The following endpoints exist to work around Anchor sandbox flakiness or to give ops a fast lever in test environments. In production, every one of them is a money-creation backdoor and must hard-refuse:

- `POST /admin/sme/kyb/fund-all-approved` — gated only by `X-Admin-Key` today (`AdminSmeKybController.cs:142`).
- `POST /admin/sme/transactions/{ref}/force-complete` — same.
- `POST /admin/sme/transactions/{ref}/mirror-to-peer` — same.
- `POST /api/v1/sme/account/test-fund` — **verified unguarded today.** The endpoint at `SmeController.cs:149` calls `_sme.TestFundAccountAsync` directly with no `IHostEnvironment` check. The XML comment claims "Refused in Prod" but no code enforces it. This must change before any prod deploy.

Implementation: inject `IHostEnvironment` into each controller and short-circuit with `403 Forbidden` on `IsProduction()`. One-line guard per endpoint. Belt-and-braces: also add a second guard inside the corresponding service methods so a future controller that calls them isn't an accidental backdoor.

### 5.2 Anchor + Providus base URLs hardcoded per env

Currently `Anchor:BaseUrl` is configurable. We pin it via env-specific config so prod can never accidentally talk to sandbox and vice versa:

- Staging config: `Anchor:BaseUrl = "https://api.sandbox.getanchor.co"`
- Production config: `Anchor:BaseUrl = "https://api.getanchor.co"`

Same pattern for Providus / XpressWallet. The actual config namespace in this codebase is `Providus:*` (`Program.cs:244` reads `Providus:BaseUrl`; service uses `Providus:ApiKey`, `Providus:AccountPrefix`, `Providus:TransactionLookupTemplate`). "XpressWallet" is the product name; "Providus" is the bank — we picked Providus as the config key. Anyone documenting credential rotation should use that name to avoid confusion.

### 5.3 (Optional, recommended) Startup banner

On boot, log a clear, unmissable line:

```
=== RIVERLY API STARTING — ENV=Production — Anchor=https://api.getanchor.co ===
```

So anyone tailing logs immediately knows which environment they're looking at.

---

## 6. Deploy workflows

### 6.1 Staging (needs minor cleanup)

`.github/workflows/deploy.yml`:
- Trigger: push to `main`, or manual `workflow_dispatch`
- Builds the image, pushes to ECR, updates staging ECS service.
- **Today's resource names are a mess:** the workflow targets ECS service `riverly-task-service-z8o468om`, container `riverly_staging_container`, and smoke-tests `api.riverly.ng` (`.github/workflows/deploy.yml:23-28,118`). The container is named "staging" but serves the domain we want for prod. Before we provision prod we should:
  - Rename the smoke-test URL to `api-staging.riverly.ng` (after §10.0 cutover).
  - Optionally rename the ECS service / container to something like `riverly-api-staging` so prod can use the clean name. ECS doesn't allow renaming services in place — this means creating a new service + cutting traffic over. Lower-effort alternative: leave the staging name ugly, name the new prod resources cleanly. Recommendation: lower-effort.

### 6.2 Production (new)

`.github/workflows/deploy-production.yml`:
- Trigger: `workflow_dispatch` only — manual, no auto-deploy from any branch.
- Input: `commit_sha` (required) — must already exist on `main`.
- Assumes the production OIDC role.
- Pulls the existing image from ECR by SHA (does NOT rebuild — guarantees parity with what was tested on staging).
- Updates production ECS service.
- Smoke test against `api.riverly.ng`.
- Gated by a GitHub "production" environment with required reviewers (e.g. Yerins + Femi must approve before the deploy runs).

Later we can add tag-triggered deploys (`v1.0.0` → auto-deploy) once we have release rhythm.

### 6.3 Rollback

Because every deploy is keyed off an image SHA, rolling back is "deploy the previous SHA". One-click via the same `workflow_dispatch`. No code revert needed.

---

## 7. Data on first prod deploy

- Prod RDS starts empty.
- First boot runs `db.Database.Migrate()` (`Program.cs:381`) which creates every table from scratch in dependency order.
- The `BackfillPersonalIdentities` migration (and any other backfill migrations) become no-ops on an empty database — they only act on existing rows. Safe for the very first prod deploy.

**Risk this introduces for every subsequent prod deploy:** `db.Database.Migrate()` runs on every container boot. A bad migration in a future PR will roll out to prod the instant a new task starts, with no human review gate. We need to harden this before opening signups:

1. **Pre-deploy snapshot.** The prod deploy workflow takes an RDS snapshot *before* updating the ECS service. Restore is a 5-minute fix if a migration corrupts data. (Manual ops alternative until automated: snapshot in console before clicking "deploy".)
2. **Migration review checklist.** Any PR adding a new migration file requires sign-off from a second engineer plus a manual call-out in the PR description ("This adds migration `X` — backfill is `Y`, rollback plan is `Z`").
3. **Optional, later:** decouple migrations from container boot — run them as a one-shot task before the ECS service updates. This lets us see migration logs separately and fail the deploy without a half-running cluster. Skip for now; revisit if we hit a bad-migration incident.

---

## 8. Webhooks

Each upstream provider needs its production webhook URL pointed at `api.riverly.ng`. **Use the actual routes in the codebase:**

- **Anchor:** live portal → webhook URL → `https://api.riverly.ng/api/v1/webhooks/anchor` (controller at `AnchorWebhookController.cs:19`).
- **Providus (XpressWallet):** live portal → webhook URL → `https://api.riverly.ng/api/v1/webhooks/providus` (controller at `ProvidusWebhookController.cs:13`; reads `Providus:ApiKey` for auth).

Staging webhook URLs:
- Anchor sandbox → `https://api-staging.riverly.ng/api/v1/webhooks/anchor`
- Providus sandbox → `https://api-staging.riverly.ng/api/v1/webhooks/providus`

Verify HMAC / signature / shared-secret values are env-specific in our config so a webhook signed with the staging secret can't be replayed against prod, and vice versa.

---

## 9. Observability

- **Logs**: every log line auto-enriched with `Environment=Production` or `Environment=Staging` via Serilog (probably already configured — confirm).
- **CloudWatch dashboards**: separate dashboards per environment so we don't accidentally read staging metrics when triaging a prod incident.
- **Alerts**: prod gets paging alerts (PagerDuty / SNS → SMS / Slack); staging gets info-level only.
- **Error tracking** (if/when we add Sentry-style): separate project per env.

---

## 10. Step-by-step rollout order

A deliberate sequence so we always have a working escape hatch. **§10.0 must complete before §10.1 starts** — otherwise we end up with prod infrastructure colliding with the staging cluster currently serving `api.riverly.ng`.

0. **Domain cutover (no code, no infra, just DNS + coordination)** —
   - Provision DNS record `api-staging.riverly.ng` pointing at the current staging cluster's ALB (same target as `api.riverly.ng` today).
   - Update the smoke-test URL in `deploy.yml` to `api-staging.riverly.ng`.
   - Coordinate with Tolu, Fatai, and Femi: every mobile build, every web client, every Postman collection has to switch from `api.riverly.ng` → `api-staging.riverly.ng`. Cut a Riverly mobile release containing the change BEFORE proceeding.
   - Keep `api.riverly.ng` pointing at the staging cluster for an overlap window (1–2 weeks) so any straggling client doesn't break. Monitor request logs on the old hostname; when traffic drops to zero from real clients, the cutover is complete.
   - **Only after the overlap window expires** is `api.riverly.ng` free to be repointed at the prod cluster.
1. **Infra (no code)** — Provision prod RDS, ECS cluster/service/task def, secrets, OIDC role, ACM cert, ALB, Route53 record (`api.riverly.ng` pointing at the *new* prod ALB). Verify all in AWS console.
2. **Code safety gates** — Land sections 5.1 + 5.2 + 5.3 on `main`. Deploy to staging via existing auto-deploy. Verify staging still works exactly as before (the env guard is a no-op on staging).
3. **Production deploy workflow** — Land `deploy-production.yml`. Don't trigger it yet.
4. **First prod deploy** — Manually dispatch `deploy-production.yml` with the current `main` commit SHA. Smoke-test using a known-good endpoint (`GET /api/v1/reference/industries`).
5. **Provider webhook URLs** — Configure Anchor + XpressWallet to send webhooks to `api.riverly.ng`. Send a test webhook and verify it lands in prod logs.
6. **Internal end-to-end test** — One internal user signs up against prod via the production mobile build, completes onboarding, does a small (₦100) transfer to a known account. Verify funds settle through the live providers.
7. **Limited beta** — Invite a handful of internal users to prod. Watch metrics for 48 hours.
8. **Open signup** — Mobile apps now point at prod by default. Staging remains for ongoing development.

---

## 11. After prod is live: working rhythm

Day-to-day:
1. Feature branch (or Yerins_branch) → PR → `main`.
2. PR merges to `main` → auto-deploys to staging.
3. QA / dev verify on staging.
4. When stable, manually dispatch `deploy-production.yml` with the verified SHA.

Hotfix to prod:
1. Cut hotfix branch from the prod-deployed SHA (NOT from latest `main` if `main` has unreleased work).
2. Fix, PR, merge to `main`.
3. Deploy that specific SHA to prod.
4. The same fix is now also on `main` so the next regular prod deploy carries it forward.

---

## 12. Open questions for review

These need a decision before we start:

1. **AWS account split?** All in one account (current) or a separate AWS account for production (extra isolation, common in regulated fintech)? Recommendation: stay in one account for now; revisit when we have ops bandwidth for cross-account IAM.
2. **Required reviewers for prod deploy** — who clicks "approve" on the GitHub environment? Recommendation: Yerins or Femi.
3. **Production database backup policy** — RDS automated backups (retention period) + manual snapshots before risky migrations. Recommendation: 14-day automated, snapshot before every migration.
4. **Are we ready to retire any staging-only feature flags / endpoints permanently**, or just gate them? Recommendation: gate, don't delete, so we can keep using them on staging.
5. **Customer support / ops dashboard** — do we want a separate read-only admin path for prod, or do we use the same admin endpoints (minus the staging-only ones) with the prod admin key?
6. **Is `api.riverly.ng` currently used by mobile or web staging builds?** (Raised in review.) If yes — and it almost certainly is — then §10.0's overlap window is non-negotiable. We need a list of every client that hits `api.riverly.ng` today so we know when the cutover is safe to finalize.
7. **Provider naming consistency.** (Raised in review.) The config namespace is `Providus:*` but team conversation says "XpressWallet". Decision: use "Providus" in code + ops runbooks, "XpressWallet" only as a product label in user-facing copy. Rename anything that still mixes them?

---

## 13. What I need from you to proceed

1. Sign-off on the recommendations above, or flag what to change.
2. Decision on the open questions in §12.
3. Green light to start §10 step 1 (infra provisioning) — this is the only step that costs real AWS money and is hard to undo.

Once those are in, the code-side work (§5) is ~half a day; the deploy workflow (§6.2) is another half-day; the infra setup depends on whether you want me to drive it via AWS console or do it as Terraform / CDK.
