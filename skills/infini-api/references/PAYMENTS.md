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
  "payment_method": 1
}
```

Fields:

| Field | Required | Notes |
| --- | --- | --- |
| `order_id` | Yes | Existing order ID. |
| `chain` | Yes | Blockchain network name. |
| `token_id` | Yes | Token identifier. |
| `payment_method` | No | Defaults to `1` for crypto. |

Response data:

```json
{
  "payment_id": "pay-123",
  "amount": "100.00",
  "amount_confirmed": "0",
  "amount_confirming": "0",
  "address": "0x1234567890abcdef1234567890abcdef12345678",
  "expires_at": 1763512195
}
```

### Choose a supported chain and token

Call `GET /v1/acquiring/currency` before displaying or creating a direct crypto payment. The response is the authoritative list of chain/token pairs enabled for the authenticated merchant; do not hardcode a platform-wide list.

BTC is available as a native `BTC`/`BTC` pair when it is enabled for the merchant. Currency availability can vary with merchant configuration.

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
- Treat all amounts as decimal strings.
- Treat the returned payment `expires_at` as authoritative. The default payment-address window is 1 hour for stablecoins, 30 minutes for ETH/SOL/TRX/BNB, and 2 hours for BTC; every payment is capped by its order expiration.
- Use `payment_id` or `prepay_id` for polling and reconciliation.
- Still rely on order webhooks for fulfillment.
