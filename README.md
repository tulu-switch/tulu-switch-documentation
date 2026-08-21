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

## When should you pin a provider?

Pass an explicit `provider` only when you need exact control:

- Your bank-code set came from a specific provider (`GET /v2/wallet/banks?provider=…`)
- Reconciliation requires knowing the rail up front
- You want identical behavior every time, regardless of balances

Otherwise omit it — auto-selection handles multi-provider customers
gracefully, including the case that used to fail:
*"specify provider to disambiguate."*
