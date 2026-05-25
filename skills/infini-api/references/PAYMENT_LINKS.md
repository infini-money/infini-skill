# Payment Links

Payment link public endpoints are anonymous endpoints used to render and submit hosted payment links.

## Get Payment Link Info

`GET /v1/acquiring/paymentlink/info?url_key={url_key}`

Returns data needed to render a payment link:

- `link_id`
- `title`
- `description`
- `currency`
- `amount`
- `pricing_mode`
- `collect_name`
- `collect_email`
- `image_url`
- `merchant_name`
- `merchant_avatar`
- `status`

`pricing_mode` values:

- `1`: fixed amount.
- `2`: open pricing; payer supplies amount.

## Checkout

`POST /v1/acquiring/paymentlink/checkout`

Request:

```json
{
  "url_key": "abc123",
  "amount": "25.00",
  "customer_name": "Jane Doe",
  "customer_email": "jane@example.com"
}
```

Rules:

- `url_key` is required.
- `amount` is required for open pricing mode.
- `customer_name` is required when the link has `collect_name=true`.
- `customer_email` is required when the link has `collect_email=true`.

Response data:

```json
{
  "order_id": "ord-123",
  "checkout_url": "https://checkout.infini.money/pay/xxxx",
  "expires_at": 1763512195
}
```

## Guidance

- Do not require merchant API signing for these public endpoints unless the user's deployment adds its own gateway.
- Validate user-entered amount, name, and email before submitting.
- Use the returned `checkout_url` for buyer redirect.
- Fulfillment still depends on order webhooks.
