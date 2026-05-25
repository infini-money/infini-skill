# Hosted Checkout

Hosted Checkout is the recommended merchant integration.

## Create Order

`POST /v1/acquiring/order`

Creates an order and returns a hosted checkout URL.

Request fields:

| Field | Required | Notes |
| --- | --- | --- |
| `request_id` | Yes | Merchant idempotency key. Generate and persist before calling Infini. |
| `amount` | Yes | Fiat amount. Use a decimal string. |
| `currency` | No | Defaults to `USD`. Supported order currencies: `USD`, `EUR`, `KRW`, `GBP`, `SGD`, `JPY`, `AUD`, `HKD`. |
| `client_reference` | No | Merchant order number. Recommended unique. |
| `order_desc` | No | Description shown in merchant records. |
| `expires_in` | No | TTL in seconds. Backend default is used when omitted. |
| `merchant_alias` | No | Overrides configured display name for this order. |
| `success_url` | No | Buyer redirect after success. |
| `failure_url` | No | Buyer redirect after failure. |
| `pay_methods` | No | Payment modes: `[1]` crypto, `[2]` card, `[3]` Binance Pay, `[5]` Apple Pay, `[6]` Google Pay, or a combined list such as `[1,2,3,5,6]`. Defaults to merchant config. |

Currency notes:

- Use uppercase currency codes.
- Use whole-number amounts for zero-decimal currencies `JPY` and `KRW`.
- Use decimal strings for amounts in generated code to avoid floating-point rounding.

Response data:

```json
{
  "order_id": "10290d05-xxxx",
  "request_id": "a759b99a-9d22-433d-bced-ab1d2e1bea1d",
  "checkout_url": "https://checkout.infini.money/pay/xxxx",
  "client_reference": "ORDER-10001",
  "token": "..."
}
```

## Query Order

`GET /v1/acquiring/order?order_id={order_id}`

Returns real-time order state, amounts, expiration, merchant info, available payment methods, and the last paid payment when available.

Important fields:

- `status`
- `order_amount`
- `order_currency`
- `amount_confirming`
- `amount_confirmed`
- `expires_at`
- `exception_tag`
- `last_paid_payment`

Status values:

| Status | Meaning |
| --- | --- |
| `pending` | Awaiting payment. |
| `processing` | Payment is partially received or confirming. |
| `paid` | Order amount satisfied. |
| `partial_paid` | Expired after partial payment. |
| `expired` | Expired without full payment. |

## Reissue Checkout Token

`POST /v1/acquiring/order/token/reissue`

Use when the checkout page was closed or the token expired.

```json
{
  "order_id": "ord-123"
}
```

Response:

```json
{
  "order_id": "ord-123",
  "checkout_url": "https://checkout.infini.money/pay/xxxx",
  "token": "..."
}
```

## Implementation Notes

- Create orders on the backend only.
- Return only `checkout_url` and merchant-facing state to the frontend.
- Use webhooks as the primary source for fulfillment state.
- Poll query order as a fallback or reconciliation job.
- Do not fulfill solely on frontend redirect; redirects can be skipped or forged.
