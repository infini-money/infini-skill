# Merchant Customer Deposits

Use this API when a merchant needs permanent deposit addresses for users in its own system. It is separate from hosted checkout: create a Customer, bind a supported asset/network address, then credit the merchant's user ledger only after a completed deposit webhook.

Base paths:

- Production: `https://openapi.infini.money/v2/merchant-customer`
- Sandbox: `https://openapi-sandbox.infini.money/v2/merchant-customer`

## Authentication and permissions

This API uses **Ed25519 v2**. Do not apply the legacy HMAC signing instructions in `AUTH.md` to these requests. Send the credential ID, timestamp, nonce, `Digest`, and Ed25519 `Authorization` signature required by the public API. Sign the exact request path, including its query string. Generate a fresh timestamp, nonce, digest, and signature on every retry; retain the same business fields.

Required permissions:

| Permission | Operations |
| --- | --- |
| `merchant_customer.read` | List currencies; get a customer, quota, or address binding |
| `merchant_customer.write` | Create/update a customer; create/reuse an address binding |

## Integration sequence

1. `GET /currencies` and use its current asset/network list. Do not hard-code supported networks or thresholds.
2. `POST /customers` with a stable, case-sensitive `external_user_id`; persist the returned integer `customer_id`.
3. `POST /addresses` with `customer_id`, `asset`, and `network`; persist the returned binding and address.
4. Subscribe to customer-deposit webhook events, verify and deduplicate them, and credit the merchant's user ledger only from `merchant_customer_deposit.completed`.

## Customer and address semantics

- `external_user_id` is unique per merchant and environment. Creating a Customer returns HTTP 201; reusing/updating it returns HTTP 200. Omitted optional profile fields remain unchanged, while `null` or an empty string clears a supplied field.
- A customer quota is consumed by new Customers, not address bindings. Existing Customers can still be updated at the quota limit.
- An address binding is idempotent by merchant, environment, `customer_id`, `asset`, and `network`. A new ready binding returns HTTP 201; reuse returns HTTP 200.
- `GET /addresses` is read-only. It never allocates a wallet or creates a binding.
- Treat `min_deposit_amount` and `fixed_fee` as decimal strings. They are current display values and do not lock the price or fee policy for a later deposit.
- An address-creation timeout is ambiguous. Retry with the same customer, asset, and network rather than creating another Customer.

## Webhooks and crediting

Events are `merchant_customer_deposit.confirming`, `confirmed`, `completed`, and `failed`. Verify HMAC-SHA256 over the raw payload using:

```text
{timestamp}.{event_id}.{payload}
```

Use `X-Webhook-Event-Id` for delivery deduplication and `data.deposit_id` for final credit reconciliation. Events may be delayed, duplicated, or out of order; never regress a completed local deposit from an older event.

Only `completed` authorizes a user-ledger credit. Use `data.credited_amount` and `data.credited_currency`, not the received `data.amount`. `credited_amount` can be the string `"0"`; then `transaction_id` is null and no merchant ledger settlement occurred, so record the terminal result but do not credit the user.

## Error handling

Handle the HTTP status, response `code`, and `data.error`. Important cases:

| HTTP | Code | `data.error` | Action |
| --- | --- | --- | --- |
| 404 | 47001 | `customer_not_found` | Check the Customer ID and environment. |
| 404 | 47002 | `deposit_address_not_found` | Create the binding first. |
| 409 | 47003 | `customer_limit_exceeded` | Check quota; do not create a duplicate Customer. |
| 409 | 47007 | `address_pool_exhausted` | Contact Infini to replenish the wallet pool, then retry. |
| 503 | 47005 | `price_unavailable` | Retry unchanged business fields after backoff. |
| 503 | 47006 | `dependency_unavailable` | Retry unchanged business fields after backoff. |

Do not branch on response `message` text. Gateway 429/5xx errors can have a different envelope.
