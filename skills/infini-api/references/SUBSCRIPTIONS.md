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
    "subscription_end_at": 0,
    "canceled_url": "https://merchant.example/subscription/canceled"
  }
}
```

Outer order fields:

- `request_id`, `amount`, and `subscription` are required.
- `currency` defaults to `USD`.
- `pay_methods` follows hosted checkout payment method values.

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
| `subscription_end_at` | No | Unix timestamp, `0` means no fixed end. |
| `canceled_url` | No | Redirect after cancellation. |

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

## Query Subscription

`GET /v1/acquiring/subscription?merchant_sub_id={merchant_sub_id}`

Status values:

| Status | Meaning |
| --- | --- |
| `pending` | Awaiting first payment. |
| `active` | Active subscription. |
| `canceled` | Canceled. |

Response includes plan, billing interval, current period, next invoice time, payer email, and cancellation fields when applicable.

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
