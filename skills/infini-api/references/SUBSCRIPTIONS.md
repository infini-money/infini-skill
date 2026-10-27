# Subscriptions

Use subscription APIs for recurring invoice-based billing.

## Create Subscription

`POST /v1/acquiring/subscription`

Creates a subscription and the first hosted checkout order.

Request:

```json
{
  "request_id": "sub-create-001",
  "amount": "10.00",
  "currency": "USD",
  "client_reference": "ORDER-10001",
  "expires_in": 3600,
  "success_url": "https://merchant.example/success",
  "failure_url": "https://merchant.example/failure",
  "pay_methods": [1, 2],
  "subscription": {
    "merchant_sub_id": "msub_001",
    "plan_name": "Monthly Plan",
    "amount": "9.99",
    "interval_unit": "MONTH",
    "interval_count": 1,
    "payer_email": "user@example.com",
    "invoice_lead_days": 3,
    "invoice_due_days": 1,
    "first_billing_time": 1760000000,
    "grace_period_days": 3,
    "retry_interval": 1,
    "subscription_end_at": 0,
    "canceled_url": "https://merchant.example/subscription/canceled"
  }
}
```

Outer order fields:

- `request_id`, `amount`, and `subscription` are required.
- `currency` defaults to `USD`. Supported order currencies: `USD`, `EUR`, `KRW`, `GBP`, `SGD`, `JPY`, `AUD`, `HKD`.
- Use whole-number first and recurring amounts for zero-decimal currencies `JPY` and `KRW`.
- Public subscription docs currently describe `pay_methods` as `[1]` crypto, `[2]` card, or `[1,2]`, defaulting to merchant config.

Subscription fields:

| Field | Required | Notes |
| --- | --- | --- |
| `merchant_sub_id` | Yes | Merchant-defined subscription ID, unique per merchant. |
| `plan_name` | Yes | Plan display name. |
| `amount` | Yes | Recurring amount per period. |
| `interval_unit` | Yes | `DAY` or `MONTH`. |
| `interval_count` | Yes | Number of units per cycle, minimum 1. |
| `payer_email` | Yes | Email used for invoice notifications. |
| `invoice_lead_days` | Yes | Days before period end to send renewal invoice. |
| `invoice_due_days` | Yes | Days after invoice creation before expiration, minimum 1. |
| `first_billing_time` | No | Future Unix timestamp used as the first invoice renewal anchor; omitted means resolve at activation. |
| `subscription_end_at` | No | Unix timestamp, `0` means no fixed end. |
| `canceled_url` | No | Redirect after cancellation. |
| `grace_period_days` | No | Days after a failed invoice renewal before cancellation; default `0`. |
| `retry_interval` | No | Reminder retry interval in days during grace; default `1`. |

Response data:

```json
{
  "order_id": "10290d05-xxxx",
  "request_id": "sub-create-001",
  "client_reference": "ORDER-10001",
  "checkout_url": "https://checkout.infini.money/subscription/xxxx",
  "token": "...",
  "subscription": {
    "subscription_id": "sub-xxxx",
    "merchant_sub_id": "msub_001",
    "status": "pending"
  }
}
```

The response additionally includes `direct_debit_eligible`. When true, a paid hosted-checkout subscription meets the interval and amount criteria to upgrade through the cashier to Binance Pay Direct Debit. That upgrade clears any grace period.

## Create Binance Pay Direct Debit Subscription

`POST /v1/acquiring/subscription/binance-pay`

Use this HMAC-authenticated endpoint only when the merchant needs a dedicated Binance Pay Direct Debit contract. Binance Pay must be enabled for the merchant. The payment method is fixed, so do not send `pay_methods`.

```json
{
  "request_id": "binance-dd-001",
  "amount": "10.00",
  "currency": "USD",
  "success_url": "https://merchant.example/success",
  "failure_url": "https://merchant.example/failure",
  "subscription": {
    "merchant_sub_id": "msub_binance_001",
    "plan_name": "Monthly Plan",
    "amount": "9.99",
    "interval_unit": "MONTH",
    "interval_count": 1,
    "payer_email": "user@example.com",
    "first_billing_time": 1760000000
  }
}
```

- The outer `amount` is the first-period amount and may be `"0"` for a free first period. `subscription.amount` is the positive recurring amount.
- `first_billing_time` is required, at least three days ahead, and must fall on UTC day 28 or earlier for a `MONTH` interval.
- `DAY` requires `interval_count > 7`; `MONTH` allows 1 through 24. If supplied, `subscription_end_at` must be later than `first_billing_time`.
- `grace_period_days` is accepted but ignored for Direct Debit (stored as `0`); `retry_interval` defaults to one day.
- Use the returned `subscription_id` and `subscription.update` / `subscription.cancel` webhooks for lifecycle reconciliation. A paid first period returns payment authorization URLs; a free first period does not create an initial order.

## Query Subscription

`GET /v1/acquiring/subscription?merchant_sub_id={merchant_sub_id}`

Status values:

| Status | Meaning |
| --- | --- |
| `pending` | Awaiting first payment. |
| `active` | Active subscription. |
| `canceled` | Canceled. |

Response includes plan, billing interval, current period, next invoice time, payer email, and cancellation fields when applicable. It also includes `auto_debit_method`, `first_billing_time`, `next_billing_time` (`0` means no further charge is planned), `grace_period_days`, and `retry_interval` when applicable.

## Cancel Subscription

`POST /v1/acquiring/subscription/cancel`

Request:

```json
{
  "merchant_sub_id": "msub_001",
  "cancel_reason": "by_merchant_api",
  "note": "Customer requested cancellation"
}
```

Allowed `cancel_reason` values:

- `user_request`
- `by_merchant_api`
- `by_operation`

Canceling preserves service access until the end of the current billing period when applicable.

## Invoice Checkout and Unsubscribe

Invoice checkout and unsubscribe links are user-facing flows driven by emailed tokens:

- `GET /v1/acquiring/subscription/invoice/checkout`
- `POST /v1/acquiring/subscription/unsubscribe`

Do not use these as merchant server-to-server APIs unless the user is explicitly implementing the emailed link flow.

## Webhooks

Subscribe to:

- `subscription.update`
- `subscription.cancel`

Use webhooks to activate service after first payment, renew entitlement periods, and record cancellations.
