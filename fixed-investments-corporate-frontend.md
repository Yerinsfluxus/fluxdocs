# Fixed Offer Investments — Corporate Web Integration Guide

**Audience:** whoever builds `riverly-cooperate-banking` (corporate internet banking web)
**Backend status:** fully live on staging and production, tested. Nothing is integrated in the web app yet — this is the complete, from-scratch spec.
**Mobile apps** (Personal/Business) have their own guide: `docs/fixed-investments-mobile-frontend.md`. The flow is the same; this doc covers what is DIFFERENT for corporate: roles, organization scoping, and multi-account selection.

---

## Table of contents

1. [What the feature is](#1-what-the-feature-is)
2. [Base URL, routes, auth and roles](#2-base-url-routes-auth-and-roles)
3. [Corporate-specific rules](#3-corporate-specific-rules)
4. [Screens to build](#4-screens-to-build)
5. [Endpoint reference](#5-endpoint-reference)
6. [Error handling](#6-error-handling)
7. [Statuses and automatic behaviors](#7-statuses-and-automatic-behaviors)
8. [QA checklist](#8-qa-checklist)

---

## 1. What the feature is

A fixed-rate investment for the organization's funds. Admin (Riverly staff) creates offers with a rate and one fixed maturity date; a corporate user picks an offer, enters an amount, previews the exact interest and maturity value, confirms with their transaction PIN, and the chosen corporate account is debited. On the maturity date the backend automatically credits principal + interest back. Never compute interest client-side — always render what the preview endpoint returns.

## 2. Base URL, routes, auth and roles

| Environment | URL |
|---|---|
| Staging | `https://api-staging.riverly.ng` |
| Production | `https://api.riverly.ng` |

Route prefix: **`/api/v1/corporate/investments`**, called with the corporate product-scoped JWT (the normal corporate session token; it carries the organization).

**Role gating (mirrors transfers):**

| Action | Who |
|---|---|
| Browse offers, view investments/detail | any signed-in corporate user (incl. Viewer, Checker) |
| Preview and Invest (money movement) | **Owner, Admin, Maker only** — others get **403** |

UI rule: hide/disable the Invest button for Viewer/Checker roles rather than letting them hit the 403.

## 3. Corporate-specific rules

1. **Organization scoping.** All investment lists and details are scoped to the organization of the current session. A user who belongs to two organizations only ever sees the current org's investments.
2. **Account selection.** If the org has **one** corporate account, omit `corporateAccountId` — the backend picks it. If the org has **more than one**, you MUST send `corporateAccountId` on preview and invest, or you'll get:
   `"Your organization has more than one account. Please pass corporateAccountId in the request to pick which one to debit."`
   → Build an account picker into the amount screen whenever the org has multiple accounts.
3. **Missing org context** returns: `"Your session isn't scoped to a corporate organization. Please switch products first via /identity/switch-product."` — treat as a session problem, re-login flow.
4. **Attribution.** The invest is recorded against the logged-in user (shows in the org's transaction audit as initiated by them). Maturity payouts are recorded as system-initiated.

## 4. Screens to build

1. **Offers list** — name, rate ("12.5% p.a."), maturity date, tenor. Closed/inactive/full offers never appear (backend filters).
2. **Offer detail + amount entry** — amount input (+ account picker when org has >1 account); debounced live preview showing tenor, expected interest, maturity value.
3. **Confirm summary + transaction PIN** — same PIN the user has for corporate money actions (min 4, max 12 chars).
4. **Success** confirmation.
5. **Org investments list** — status badge (Active / Matured / Failed), amount, rate, maturity date; `?status=` filter.
6. **Investment detail** — figures + the investment's own two ledger entries (funding debit, payout credit once matured).

## 5. Endpoint reference

```
GET  /api/v1/corporate/investments/fixed-offers
GET  /api/v1/corporate/investments/fixed-offers/{offerId}
POST /api/v1/corporate/investments/fixed-offers/{offerId}/preview     (MakerOnly)
POST /api/v1/corporate/investments/fixed-offers/{offerId}/invest      (MakerOnly)
GET  /api/v1/corporate/investments/fixed                              (?status=Active|Matured)
GET  /api/v1/corporate/investments/fixed/{investmentId}
```

All shapes are identical to the mobile guide (§4 there) except the two POST bodies, which take the account selector:

**Preview**
```json
POST /fixed-offers/{offerId}/preview
{ "amount": 500000, "corporateAccountId": "GUID-or-null" }
```
Returns `tenorDays`, `startDate`, `maturityDate`, `expectedInterest`, `maturityValue`, `interestRatePercent`, `interestCalculationMethod`.

**Invest**
```json
POST /fixed-offers/{offerId}/invest
{
  "amount": 500000,
  "idempotencyKey": "<fresh v4 UUID per attempt>",
  "transactionPin": "1234",
  "corporateAccountId": "GUID-or-null"
}
```
- `idempotencyKey`: generate a fresh UUID when the user clicks Invest; reuse the SAME value for retries of that attempt (safe against double-debit). New click = new key.
- `transactionPin`: **required** — verified against the user's unified transaction PIN.

Success `201` → the investment object (`status: "Active"`).

**Investment detail** returns `{ investment, transactions[] }` where `transactions` are only this investment's ledger entries (`direction: "Debit"` funding; later `"Credit"` payout, narrations `Investment: <name>` / `Investment matured: <name>`).

## 6. Error handling

Standard envelope (`status: false`, plain-language `message`). Everything in the mobile guide §5 applies (PIN wrong/not set, insufficient balance, min/max, fully subscribed, remaining capacity, offer closed family, account inactive), plus corporate-specific:

| `message` | UX |
|---|---|
| `Your organization has more than one account. Please pass corporateAccountId…` | show the account picker |
| `The chosen account doesn't belong to your organization.` | shouldn't happen with a proper picker; treat as error |
| `You don't belong to this corporate organization.` | session/role problem — re-login |
| `Your session isn't scoped to a corporate organization…` | product-switch/re-login flow |
| HTTP **403** on preview/invest | user's role can't move money — hide the button for Viewer/Checker so this is never seen |

Offer-closed messages (`"This offer is no longer accepting new investments."` etc.) are expected states — friendly info panel, back to list, no red error styling.

## 7. Statuses and automatic behaviors

Statuses: `Active` (accruing) → `Matured` (paid out, terminal). `Failed` is rare (internal payout issue; Riverly ops resolves — show neutral "Processing issue — contact support", **no retry control in the web app**). Render unknown status strings without breaking.

Automatic (no FE work): maturity payout, push/email alerts on placement and maturity, ledger entries in the account's transaction history. Refresh the account balance after a successful invest.

## 8. QA checklist

- [ ] Viewer/Checker: offers browsable, Invest hidden; direct POST → 403
- [ ] Org with 2 accounts: invest without `corporateAccountId` → picker message; with it → success
- [ ] Wrong PIN → "Incorrect transaction PIN.", nothing debited
- [ ] Correct PIN → success; org account balance reduced; investment listed
- [ ] User in two orgs sees only the current org's investments
- [ ] Same idempotencyKey re-POST → same investment, no double debit
- [ ] Near-maturity offer absent from list; stale invest attempt → friendly closed state
- [ ] (Staging) after maturity: status Matured, account credited, credit entry in detail
