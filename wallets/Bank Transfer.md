# Bank Transfer

`POST /v2/wallet/transfer` · `POST /v2/wallet/transfer/bulk`

Move money from a customer's wallet to an **external bank account**.

## Flow

```mermaid
sequenceDiagram
    autonumber
    actor B as Builder App
    participant TS as Tulu Switch
    participant PS as Paystack
    participant BK as GTBank

    B->>TS: POST /v2/wallet/name-enquiry (0123456789, 058)
    TS-->>B: accountName: John Doe
    B->>TS: POST /v2/wallet/transfer ₦200 (no provider)
    TS->>TS: rank wallets: Paystack 600 > Flutterwave 300<br/>→ Paystack covers → chosen
    TS->>TS: atomic: debit Ada −200, debit builder holding,<br/>txn PROCESSING (ctfr_…)
    TS->>PS: disburse ₦200 → GTBank
    alt success
        PS->>BK: credit John ₦200
        PS-->>TS: accepted
        TS-->>B: 201 { provider: Paystack, autoSelected: true, status: pending }
        PS-->>TS: webhook confirms
        TS-->>B: customer.transfer.success
    else failure
        PS-->>TS: error
        TS->>TS: compensate: refund Ada + builder holding, txn FAILED
        TS-->>B: 400 + customer.transfer.failed
    end
```

## The one-provider rule

A bank payout always rides **exactly one provider** end to end. When you
omit `provider`, Tulu Switch picks it for you:

1. Rank the customer's active wallets (channel + currency) by balance.
2. Use the **first wallet that covers the full amount** — i.e. the richest
   wallet that can pay it alone.
3. No single wallet covers it → `400`, quoting the richest balance.
   **Bank payouts never split across providers.**

## Scenario

Ada holds ₦600 via Paystack and ₦300 via Flutterwave. She pays her landlord
John ₦200:

```
POST /v2/wallet/transfer
{ "customerId": "cus_ada", "channel": "WALLET", "currency": "NGN",
  "amount": 200, "accountNumber": "0123456789",
  "bankCode": "058", "accountName": "John Doe" }
```

Paystack (600) covers 200 → **Paystack processes the payout**; Flutterwave
is untouched:

```json
{
  "reference": "ctfr_1721375800000_abc123",
  "provider": "Paystack",
  "walletId": "cwlt_001",
  "autoSelected": true,
  "status": "pending"
}
```

Paying ₦700 would fail — no single wallet covers it:

```json
{ "message": "Insufficient balance. Richest wallet (Paystack) holds 600 NGN; requested: 700. Pass `provider` to pick a specific wallet." }
```

## If the provider fails

Automatic compensation: the customer's wallet is refunded in full and the
transaction is marked `FAILED`. Nothing is left half-debited. You get a
`400` with the provider error, and `customer.transfer.failed` fires.

## Bulk payouts

`POST /v2/wallet/transfer/bulk` runs up to 200 payouts in one call:

- **All-or-nothing validation** — any invalid item rejects the whole batch
  with a per-item `errors` list, before anything is debited.
- **Per-item settlement** — each item uses its own selected provider; one
  failure refunds only that item.
- Idempotency via `requestReference` (duplicate → `409`).

Track batches: `GET /v2/wallet/transfers/bulk/:batchReference`.

## Constraints

- Blocked in TEST mode (real money at the provider).
- Fetch bank codes first: `GET /v2/wallet/banks?currency=NGN`; if you pin a
  provider there, reuse it for name-enquiry and transfer.
