# Fixed Offer Investments — Mobile Integration Guide (Personal + Business apps)

**Audience:** Tolu — `riverly_mobile` (Personal) and `riverly_business` (SME)
**Backend status:** fully live on staging and production, tested. Nothing is integrated in the apps yet — this document is the complete, from-scratch integration spec.
**Corporate web** has its own guide: `docs/fixed-investments-corporate-frontend.md`.

---

## Table of contents

1. [What the feature is](#1-what-the-feature-is)
2. [Base URLs, routes and tokens](#2-base-urls-routes-and-tokens)
3. [Screens to build](#3-screens-to-build)
4. [Endpoint reference with shapes](#4-endpoint-reference-with-shapes)
   - 4.1 List offers
   - 4.2 Offer detail
   - 4.3 Preview (live calculation)
   - 4.4 Invest (place the investment)
   - 4.5 My investments
   - 4.6 Investment detail (with its transactions)
5. [Error handling — every message and what to do](#5-error-handling)
6. [Investment statuses](#6-investment-statuses)
7. [Things that happen automatically (no app work)](#7-automatic-behaviors)
8. [QA checklist for staging](#8-qa-checklist)

---

## 1. What the feature is

A fixed-rate investment. Admin creates an offer (name, annual rate, one fixed maturity date, min/max amounts). The customer picks an offer, enters only an **amount**, sees exactly what they'll earn and when, confirms with their **transaction PIN**, and their wallet is debited. On the maturity date the backend automatically pays principal + interest back to the wallet — the app does nothing for maturity.

Key mental model: the maturity **date** is fixed per offer and shared by every investor. Interest = amount × rate × days-until-maturity / 365 (or /360, per offer). The preview endpoint returns all computed numbers — **never calculate interest in the app**; always display what preview returns.

## 2. Base URLs, routes and tokens

| Environment | URL |
|---|---|
| Staging | `https://api-staging.riverly.ng` |
| Production | `https://api.riverly.ng` |

| App | Route prefix | Token required |
|---|---|---|
| Personal (`riverly_mobile`) | `/api/v1/personal/investments` | the normal Personal session token |
| Business (`riverly_business`) | `/api/v1/sme/investments` | the **product-scoped SME token** — the same one transfers use (from switch-product step-up). An identity token gets **403**. |

All responses use the standard envelope:
```json
{ "status": true|false, "message": "…", "data": … }
```

## 3. Screens to build

Same six screens in both apps:

1. **Offers list** — cards: offer name, rate ("12.5% p.a."), maturity date, tenor ("in 90 days"). Empty state: "No offers available right now."
2. **Offer detail + amount entry** — one input (amount). Shows min/max. As the user types (debounced), call **preview** and live-render: tenor days, expected interest, maturity value, maturity date.
3. **Confirm summary** — amount, rate, tenor, expected interest, maturity value, maturity date — then **PIN entry** (same PIN pad component as transfers).
4. **Success** — confirmation with maturity date and value.
5. **My investments** — list with status badge (Active / Matured / Failed), amount, rate, maturity date. Consider tabs or a filter (`?status=`).
6. **Investment detail** — full figures + its own transaction entries (funding debit; payout credit once matured).

## 4. Endpoint reference with shapes

### 4.1 List offers
```
GET {prefix}/fixed-offers
```
`data`: array of:
```json
{
  "id": "GUID",
  "productName": "Riverly Q3 Fixed Income",
  "interestRatePercent": 12.5,
  "maturityDate": "2026-10-12",
  "tenorDays": 90,
  "minimumAmount": 1000.00,
  "maximumAmount": 5000000.00,
  "issueSize": 100000000.00,
  "subscribedAmount": 25000000.00,
  "remainingAmount": 75000000.00,
  "status": "Active",
  "subscriptionCloseDays": 7
}
```
Only offers that are open for investment are returned — inactive, fully subscribed, and near-maturity (closed-window) offers are already filtered out. `tenorDays` is "if you invest today". Render `maturityDate` as the raw date string (business dates are Africa/Lagos — don't timezone-shift them).

### 4.2 Offer detail
```
GET {prefix}/fixed-offers/{offerId}
```
Same shape as 4.1 plus `interestCalculationMethod`, `createdAt`, `updatedAt`. Can fail with "no longer available/accepting" (see §5) if opened from a stale list.

### 4.3 Preview (live calculation)
```
POST {prefix}/fixed-offers/{offerId}/preview
{ "amount": 100000 }
```
`data`:
```json
{
  "offerId": "GUID",
  "productName": "Riverly Q3 Fixed Income",
  "amount": 100000.00,
  "interestRatePercent": 12.5,
  "tenorDays": 90,
  "startDate": "2026-07-14",
  "maturityDate": "2026-10-12",
  "expectedInterest": 3082.19,
  "maturityValue": 103082.19,
  "interestCalculationMethod": "SimpleInterestActual365"
}
```
Preview also validates (amount range, balance, offer open) — so its error messages (§5) can be shown inline under the amount field before the user ever hits confirm.

### 4.4 Invest (place the investment)
```
POST {prefix}/fixed-offers/{offerId}/invest
{
  "amount": 100000,
  "idempotencyKey": "<fresh v4 UUID>",
  "transactionPin": "1234"
}
```
- `idempotencyKey` — **generate a fresh UUID the moment the user taps Invest**, and reuse THAT SAME value for any retry of that attempt (timeout, network drop). This is what makes retrying safe: same key can never double-debit. A new tap = a new key.
- `transactionPin` — **required.** The same transaction PIN the user already has for transfers; min 4, max 12 chars. No new PIN setup flow needed.

Success (`201`): `data` is the investment object (see 4.5's shape) with `status: "Active"`. Show the success screen.

**Retry rule:** on timeout/unknown result, re-POST with the same body (same idempotencyKey). If the first attempt actually succeeded you'll get the original investment back, not a duplicate.

### 4.5 My investments
```
GET {prefix}/fixed                    (optional ?status=Active or ?status=Matured)
```
`data`: array of:
```json
{
  "id": "GUID",
  "offerId": "GUID",
  "productType": "Sme",
  "reference": "<the idempotency key>",
  "productName": "Riverly Q3 Fixed Income",
  "principalAmount": 100000.00,
  "interestRatePercent": 12.5,
  "tenorDays": 90,
  "expectedInterest": 3082.19,
  "maturityValue": 103082.19,
  "startDate": "2026-07-14",
  "maturityDate": "2026-10-12",
  "status": "Active",
  "debitTransactionReference": "FXI-a1b2c3d4e5f60718-DR",
  "payoutTransactionReference": null,
  "createdAt": "2026-07-14T10:22:00Z",
  "maturedAt": null
}
```
(Any `payoutAttempts` / `lastPayoutError` / `lastPayoutAttemptAt` fields are always `null` on customer endpoints — ignore them.)

### 4.6 Investment detail (with its transactions)
```
GET {prefix}/fixed/{investmentId}
```
`data`:
```json
{
  "investment": { …same shape as 4.5… },
  "transactions": [
    {
      "reference": "FXI-a1b2c3d4e5f60718-DR",
      "direction": "Debit",
      "amount": 100000.00,
      "currency": "NGN",
      "narration": "Investment: Riverly Q3 Fixed Income",
      "occurredAt": "2026-07-14T10:22:00Z"
    }
  ]
}
```
Only THIS investment's ledger entries — the funding debit and, once matured, a `"Credit"` entry with narration `Investment matured: <name>`. No filtering needed app-side.

## 5. Error handling

All errors: `status: false` + a plain-language `message`. Match on these exact strings where UX differs:

| `message` | When | UX |
|---|---|---|
| `Incorrect transaction PIN.` | wrong PIN | shake + re-enter |
| `Transaction PIN has not been set. Please set your PIN first.` | user never set a PIN | route to the existing set-PIN flow, then return |
| PIN lockout message (Personal only, after repeated failures) | brute-force protection | same handling as transfer lockout; disable retry |
| `You don't have enough balance to make this investment.` | insufficient funds | show on amount screen |
| `The minimum investment for this offer is NGN…` / `The maximum investment for this offer is NGN…` | out of range | inline under amount field |
| `This offer is fully subscribed and is no longer accepting investments.` | capacity gone | info state → back to list |
| `This offer only has NGN… remaining.` | amount > remaining capacity | inline; suggest the remaining amount |
| `This offer is no longer accepting new investments.` | offer entered its close window (auto-closes ~7 days before maturity) | friendly "offer closed" state → back to list |
| `This offer is no longer available.` / `This offer has matured and is no longer available.` | deactivated / matured | same as above |
| `Your account isn't active. Please contact support.` | account issue | blocking message |

Treat any of the offer-closed family as *expected states*, not technical errors — no red screens.

## 6. Investment statuses

```
"Active"    money invested, accruing — show maturity countdown
"Matured"   paid out (principal + interest credited) — terminal, success styling
"Failed"    rare: our payout hit an internal problem; ops resolves it.
            Show a neutral/amber badge e.g. "Processing issue — contact support".
            NO customer action exists — do not show any retry button.
"Cancelled" reserved, never returned today
```
Render unknown status strings gracefully (plain text badge) — never crash on a new value.

## 7. Automatic behaviors (no app work)

- **Maturity is automatic.** Backend pays out on the maturity date; status flips to `Matured`. The app just re-fetches.
- **Alerts are sent by the backend**: push + email on placement ("Debit Alert…") and on maturity ("Credit Alert…"). Nothing to implement.
- Wallet balance reflects the debit/credit immediately — refresh balance after a successful invest.
- Ledger narrations in the general transaction history read `Investment: <name>` / `Investment matured: <name>`.

## 8. QA checklist

- [ ] Offers list shows open offers; a near-maturity offer (≤7 days) is absent
- [ ] Amount below min / above max → inline message with the exact limit
- [ ] Preview numbers update as amount changes and match the confirm screen
- [ ] Wrong PIN → "Incorrect transaction PIN.", nothing debited
- [ ] Correct PIN → success screen, wallet reduced, push + email received
- [ ] Same idempotencyKey re-POSTed → same investment returned, no double debit
- [ ] My investments shows the new Active investment; detail shows the debit entry
- [ ] (Staging) after maturity is processed: status Matured, wallet credited, credit entry in detail, credit alert received
- [ ] Unknown status string → screens render without crashing
- [ ] SME app: calling with a non-SME token → 403 (expected)
