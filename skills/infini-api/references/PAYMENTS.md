# Payments

Use payment APIs when the merchant needs programmatic payment creation instead of hosted checkout.

## Create Crypto Payment

`POST /v1/acquiring/payment`

Request:

```json
{
  "order_id": "ord-123",
  "chain": "ETHEREUM",
  "token_id": "USDT",
  "payment_method": 1,
  "force_create": false
}
```

Fields:

| Field | Required | Notes |
| --- | --- | --- |
| `order_id` | Yes | Existing order ID. |
| `chain` | Yes | Blockchain network name. |
| `token_id` | Yes | Token identifier. |
| `payment_method` | No | Defaults to `1` for crypto. |
| `force_create` | No | When `true`, always allocate a new payment instead of reusing a compatible unexpired payment. |

Response data:

```json
{
  "payment_id": "pay-123",
  "amount": "100.00",
  "due_amount_fiat": "100.00",
  "amount_confirmed": "0",
  "amount_confirming": "0",
  "address": "0x1234567890abcdef1234567890abcdef12345678",
  "expires_at": 1763512195,
  "quoted_token_amount": "100.00",
  "quoted_token_currency": "USDT",
  "quoted_token_price_usdc": "0"
}
```

If `GET /v1/acquiring/order` returns `kyc_required: true`, Infini-hosted login and approved KYC are required before a payment can be created. In that case `POST /v1/acquiring/payment` may return business code `46001`.

## Quote Payment

`POST /v1/acquiring/payment/quote`

Returns the current payable amount without creating a payment address.

```json
{
  "amount": "100.00",
  "due_amount_fiat": "100.00",
  "amount_confirmed": "0",
  "amount_confirming": "0",
  "quoted_token_amount": "100.00",
  "quoted_token_currency": "USDT",
  "quoted_token_price_usdc": "0"
}
```

Use quote first when the merchant needs a price preview or when `kyc_required: true` blocks direct creation until the buyer logs in and passes KYC in hosted checkout.

## Query Payment

`GET /v1/acquiring/payment?payment_id={payment_id}`

Returns payment address, network, due amount, confirmed/confirming amounts, transactions, status, amount status, exception tags, crypto amount, channel fee, and order amount.

## List Payments

`GET /v1/acquiring/payment/list?order_id={order_id}`

Returns all payment records for an order.

## Binance Pay

Create:

`POST /v1/acquiring/payment/binance`

```json
{
  "order_id": "ord-123"
}
```

Response includes:

- `amount`
- `qrcode_link`
- `qr_content`
- `checkout_url`
- `deeplink`
- `universal_url`
- `prepay_id`
- `expire_time` as Unix milliseconds

Query:

`GET /v1/acquiring/payment/binance?prepay_id={prepay_id}`

Status values include `created`, `success`, and `closed`.

## Guidance

- Prefer hosted checkout unless the user explicitly needs direct payment address or Binance Pay objects.
- Query the order first; treat `kyc_required: true` as a signal that the buyer must complete Infini-hosted login and KYC before payment creation.
- Treat all amounts as decimal strings.
- Use `payment_id` or `prepay_id` for polling and reconciliation.
- Still rely on order webhooks for fulfillment.
