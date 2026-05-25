# Getting Started

Use this guide when the user is new to Infini or asks for a first working integration.

## Environments

| Environment | Merchant portal | API base URL |
| --- | --- | --- |
| Production | `https://business.infini.money/register` | `https://openapi.infini.money` |
| Sandbox | `https://business-sandbox.infini.money/register` | `https://openapi-sandbox.infini.money` |

Merchant APIs use the `/v1/acquiring` prefix.

Supported order currencies for hosted checkout:

- `USD`
- `EUR`
- `KRW`
- `GBP`
- `SGD`
- `JPY`
- `AUD`
- `HKD`

`USD` is the default when `currency` is omitted. Use uppercase currency codes. `JPY` and `KRW` are zero-decimal currencies, so generated examples should use whole-number amounts for those currencies.

## Onboarding Flow

1. Register an Infini business account.
2. Configure cashier settings under Payments -> Cashier Settings:
   - Merchant display name.
   - Default order expiration time.
3. Create an API key in the Developer page:
   - Save the `keyId`.
   - Save the secret immediately; it is only displayed on creation.
   - Configure IP allowlist for server outbound IPs.
   - Grant only the permissions needed by the integration.
4. Configure webhook endpoint and event subscriptions in the Developer page.
5. Implement HMAC request signing and create the first hosted checkout order.

## Recommended First Integration

Hosted Checkout is the default path:

1. Backend creates an order with `POST /v1/acquiring/order`.
2. Backend returns `checkout_url` to the frontend.
3. Frontend redirects the buyer to `checkout_url`.
4. Backend receives order webhooks and updates local order state.
5. Backend optionally polls `GET /v1/acquiring/order?order_id=...` as reconciliation.

## Minimal Create Order Body

```json
{
  "request_id": "a759b99a-9d22-433d-bced-ab1d2e1bea1d",
  "amount": "10.00",
  "currency": "USD",
  "client_reference": "ORDER-10001",
  "order_desc": "Product purchase",
  "success_url": "https://merchant.example/success",
  "failure_url": "https://merchant.example/failure"
}
```

## Production Checklist

- Store API secrets server-side only.
- Persist `request_id` before calling Infini.
- Use decimal/string amounts.
- Verify webhook signatures before parsing business meaning.
- Deduplicate webhooks by `X-Webhook-Event-Id`.
- Keep API key IP allowlists current.
- Log request IDs, order IDs, and event IDs, but never log secrets or full card data.
