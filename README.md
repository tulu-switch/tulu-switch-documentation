# Tulu Switch — Integration Documentation

Feature-by-feature guides for builders integrating the v2 API. Each guide
explains the feature, walks a real scenario, and shows the flow as a
sequence diagram.

## The one rule to remember

> **If you don't tell us which provider to use, we pick for you.**
>
> We look at the customer's wallets across all providers and use the richest
> one that covers the amount. If no single wallet covers it, we split the
> debit across wallets (internal transfers & conversions only).

Every money-movement response tells you what happened:

| Field | Meaning |
|---|---|
| `provider` | Which provider backed the movement |
| `walletId` | Which customer wallet was debited/credited |
| `autoSelected` | `true` if you didn't pin a provider and we chose |
| `split` / `from.legs[]` | Present when a debit was spread across wallets |

## Picking a provider — both paths

Every money-movement request lets you either **pin a provider** or **omit it**.
The two paths behave very differently:

### Path A — you pass `provider` (pinned)

- The exact wallet backed by that provider is used. Nothing is auto-selected,
  no splitting, no fallback.
- Resolution is **strict**: if the customer has no active wallet with that
  provider → `404`. If that wallet alone can't cover the amount → `400`
  insufficient balance.
- Response reports `autoSelected: false`.
- Bulk items can each pin their own provider (`provider`, `fromProvider`,
  `toProvider`).

### Path B — you omit `provider` (auto)

Tulu Switch resolves wallets for you, per endpoint:

| Endpoint | Source / sender side | Destination side |
|---|---|---|
| **Deposit** | Health-ranked among the customer's wallets — open circuits avoided first, best success rate wins | — (the chosen wallet receives the funds) |
| **Bank Transfer** (+ bulk) | Richest wallet that covers the full amount. **Never splits** — if no single wallet covers it, the request fails with `400` | — |
| **Internal Transfer** (+ bulk) | Richest covering wallet; if none covers but the combined balance does, the debit **splits** richest-first | Defaults to the sender's primary provider; if the recipient holds no wallet there, falls back to their richest active wallet (becomes cross-provider) |
| **Conversion** | Same as internal transfers, in `fromCurrency` | Defaults to the source's provider; falls back to any active wallet in the target currency |

Responses report `autoSelected: true`, plus `split: true` and `from.legs[]`
whenever a debit was spread across wallets.

## Guides

### Identity & setup
- [Customers](customers/Customers.md) — create customers, open per-provider wallets

### Moving money
- [Deposit](wallets/Deposit.md) — inbound via checkout; health-ranked provider pick
- [Bank Transfer](wallets/Bank%20Transfer.md) — wallet → bank account; one provider, highest balance
- [Internal Transfer](wallets/Internal%20Transfer.md) — your customer → your customer; auto-split
- [External Transfer](wallets/External%20Transfer.md) — cross-builder, by wallet id
- [Conversion](wallets/Conversion.md) — currency → currency at platform rates

### Rails & infrastructure
- [Providers](providers/Providers.md) — who supports what; health & circuit breakers
- [Treasury Wallet](treasury/Treasury%20Wallet.md) — how builder holdings mirror customer funds
- [Virtual Account](checkout/Virtual%20Account.md) — one-time accounts for checkout

### Money back & recurring
- [Refund](refunds/Refund.md) — return completed deposits, full or partial
- [Subscription](subscriptions/Subscription.md) — recurring charges on a schedule

### Visibility
- [Webhooks](webhooks/Webhooks.md) — events pushed to you, signature verification
- [Transactions](transactions/Transactions.md) — history, status lifecycle, aggregates
