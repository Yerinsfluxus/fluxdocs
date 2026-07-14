# Fixed Investments — Mobile App Update (July 2026)

**Audience:** Tolu (mobile — Personal, SME/Business, Corporate apps)
**Status:** All backend changes are live on staging and production. Nothing here blocks current builds — but items 1 and 2 need an app release soon.

This is everything the apps need to pick up from the July fixed-investment backend work, with exact endpoints and values. The investment endpoints themselves have NOT moved; the changes are one new request field, one new status value, one new error state, and one endpoint you may not have integrated yet.

---

## Base URLs & routes

| Environment | URL |
|---|---|
| Staging | `https://api-staging.riverly.ng` |
| Production | `https://api.riverly.ng` |

Routes per product (same shape, different prefix):

| App | Prefix |
|---|---|
| Personal | `/api/v1/personal/investments` |
| SME / Business | `/api/v1/sme/investments` |
| Corporate | `/api/v1/corporate/investments` |

Endpoints under each prefix:
```
GET  /fixed-offers                      list offers open for investment
GET  /fixed-offers/{offerId}            offer detail
POST /fixed-offers/{offerId}/preview    interest/maturity preview
POST /fixed-offers/{offerId}/invest     place investment
GET  /fixed                             my investments (?status=Active|Matured)
GET  /fixed/{investmentId}              ONE investment + its own transactions
```

**Token note (SME app):** the SME investment endpoints now require the **product-scoped SME token** (the one you get after switch-product step-up — same token transfers use). An identity token gets 403. If the SME app already reuses the transfer token for investments, nothing to change.

---

## Change 1 — Transaction PIN on invest (ACTION REQUIRED)

Investing debits the wallet, so it now takes the same transaction PIN as transfers.

**What to build:** add the PIN entry step to the invest confirmation screen (all three apps), and send it in the invest body:

```json
POST /fixed-offers/{offerId}/invest
{
  "amount": 100000,
  "idempotencyKey": "<fresh v4 UUID per attempt>",
  "transactionPin": "1234",
  "corporateAccountId": null        // Corporate only, when org has >1 account
}
```

- `transactionPin`: string, max 12 chars. The SAME PIN the user already uses for transfers — no new PIN setup flow needed.
- **Rollout:** the field is OPTIONAL on the backend today so current builds keep working. Once your builds that send it are live, we flip it to REQUIRED. So: always send it from the next release.

**Error responses you must handle** (standard envelope, `status: false`):

| `message` | Show / do |
|---|---|
| `Incorrect transaction PIN.` | Let the user re-enter. |
| `Transaction PIN has not been set. Please set your PIN first.` | Route to the existing set-PIN flow. |
| (Personal only) lockout message after repeated wrong PINs | Same handling as the transfer PIN lockout — show the message, disable retry. |

---

## Change 2 — New investment status: `Failed` (ACTION REQUIRED)

`status` on an investment can now be:

```
"Active" | "Matured" | "Failed"      ("Cancelled" still unused)
```

`Failed` is rare — it means our payout to the customer hit a problem and operations is on it. The customer's money is safe and support/ops resolves it.

**What to build:** make sure the investments list and detail screens render an unknown/new status string without crashing, and show `Failed` plainly (e.g., a neutral/amber badge "Processing issue — contact support" is fine). No customer action exists for it — do NOT show a retry button in the app.

---

## Change 3 — Offers close before maturity (handle one error string)

Offers now automatically stop accepting investments a few days before their maturity date (default 7, set per offer by admin).

- The offers **list simply won't include** closed offers — no app change needed there.
- But a user sitting on a stale offer screen can still tap through. Detail, preview, and invest can now return:

```json
{ "status": false, "message": "This offer is no longer accepting new investments.", "data": null }
```

**What to build:** show that message as a friendly info state (offer closed) and pop back to the offers list — don't treat it as a technical error.

---

## Change 4 — Investment detail endpoint (integrate if you haven't)

```
GET /fixed/{investmentId}
```

Returns the investment plus ONLY its own ledger entries — the funding debit and, once matured, the payout credit. Built for the investment detail screen so you don't have to filter the general transaction history.

```json
{
  "status": true,
  "data": {
    "investment": {
      "id": "…", "productName": "…", "principalAmount": 100000,
      "interestRatePercent": 12.5, "tenorDays": 90,
      "expectedInterest": 3082.19, "maturityValue": 103082.19,
      "startDate": "2026-07-14", "maturityDate": "2026-10-12",
      "status": "Active", "createdAt": "…", "maturedAt": null
    },
    "transactions": [
      {
        "reference": "FXI-a1b2c3d4e5f60718-DR",
        "direction": "Debit",
        "amount": 100000,
        "currency": "NGN",
        "narration": "Investment: Riverly Q3 Fixed Income",
        "occurredAt": "2026-07-14T10:22:00Z"
      }
    ]
  }
}
```

After maturity a second entry appears: `direction: "Credit"`, narration `Investment matured: <name>`.

---

## FYI — no app work needed

- Customers now get a **push notification and email** on investment placement ("Debit Alert…") and on maturity ("Credit Alert…"). Backend sends these; nothing to implement — just expect users to see them.
- Ledger narrations in transaction history now read `Investment: <offer name>` and `Investment matured: <offer name>`.

## Suggested QA checklist (staging)

- [ ] Invest with wrong PIN → "Incorrect transaction PIN.", nothing debited
- [ ] Invest with correct PIN → success, push + email arrive, investment in list
- [ ] Open `GET /fixed/{id}` → debit entry shows; after maturity, credit entry too
- [ ] Offer with < 7 days to maturity → absent from list; invest by stale screen → friendly "no longer accepting" state
- [ ] Force an unknown status string → screens don't crash
