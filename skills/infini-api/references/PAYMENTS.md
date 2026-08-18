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
| `force_create` | No | Defaults to `false`. When omitted, Infini may return an existing reusable payment instead of allocating a fresh address. |

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

If `GET /v1/acquiring/order` reports `kyc_required=true`, unauthenticated `POST /v1/acquiring/payment` calls fail with business code `46001`. Redirect the buyer to hosted checkout or another Infini-authenticated flow instead of trying to allocate the payment directly.

## Quote Payment

`POST /v1/acquiring/payment/quote`

Previews the payable token amount without creating a payment or allocating an address.

```json
{
  "order_id": "ord-123",
  "chain": "ETHEREUM",
  "token_id": "USDT",
  "payment_method": 1
}
```

Response fields include:

- `amount`
- `due_amount_fiat`
- `amount_confirmed`
- `amount_confirming`
- `quoted_token_amount`
- `quoted_token_currency`
- `quoted_token_price_usdc`

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
- Query `GET /v1/acquiring/order` first when the buyer flow may require KYC; if `kyc_required=true`, use hosted checkout or another Infini-authenticated flow for actual payment creation.
- Treat all amounts as decimal strings.
- Use `payment_id` or `prepay_id` for polling and reconciliation.
- Still rely on order webhooks for fulfillment.
