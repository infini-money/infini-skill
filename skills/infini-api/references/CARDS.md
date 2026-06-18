# Organization Cards

Card endpoints are merchant APIs under the independent `/v2/cards` prefix. They require HMAC authentication and card-specific permissions.

## Identifier Rule

The merchant-facing card identifier is always `id`:

- `data.id` from `POST /v2/cards/apply`
- `cards[].id` from `GET /v2/cards/list`
- `id` passed to `GET /v2/cards/status`
- `id` passed to `POST /v2/cards/reveal`
- `id` passed to `POST /v2/cards/top-up`
- `id` passed to `POST /v2/cards/redeem`
- `id` passed to `POST /v2/cards/freeze`
- `id` passed to `POST /v2/cards/unfreeze`

Do not invent or expose other internal card identifiers.

## Permissions

| Permission | Endpoints |
| --- | --- |
| `card.create` | Apply, list, status, top up, redeem, freeze, and unfreeze. |
| `card.reveal` | Reveal PAN/CVV/expiry. |

API keys with card permissions should have a non-empty IP allowlist.

## Apply Card

`POST /v2/cards/apply`

Request:

```json
{
  "product_id": 1,
  "top_up_amount": "100.00",
  "token_type": "USDT",
  "user_email": "jane@example.com",
  "holder_name": "Jane Doe",
  "card_alias": "Travel card"
}
```

Required fields: `product_id`, `top_up_amount`, `token_type`, `user_email`, `holder_name`.

Product IDs:

| product_id | Product |
| --- | --- |
| `1` | Infini Lite |
| `2` | Infini Pro |
| `101` | Infini AI |

Response data:

```json
{
  "id": "a441831c-a5c7-4bed-8f61-793738afd5bc",
  "status": "init",
  "total_top_up_amount": "100.00",
  "total_fee": "1.00",
  "total_pay_amount": "101.00",
  "message": ""
}
```

## List Cards

`GET /v2/cards/list?status=active,pending&card_alias=Travel&page=1&page_size=20`

Response data includes `cards`, `total`, `page`, `page_size`, and `total_pages`.

Card item fields include:

- `id`
- `mask`
- `holder_name`
- `card_alias`
- `status`
- `currency`
- `available_balance`
- `user_id`
- `created_at`
- `updated_at`

## Get Card Status

`GET /v2/cards/status?id={id}`

Poll after apply until `status` becomes `active`.

Status values:

- `init`
- `pending`
- `active`
- `suspend`
- `deleted`

## Reveal Card

`POST /v2/cards/reveal`

Request:

```json
{
  "id": "a441831c-a5c7-4bed-8f61-793738afd5bc"
}
```

Response data:

```json
{
  "card_number": "4111111111111111",
  "cvv": "123",
  "expiration_mmyy": "1228",
  "card_currency": "USD"
}
```

Sensitive data rules:

- Never log full PAN, CVV, or expiry.
- Do not persist CVV.
- Return reveal data only to authorized, audited server-side flows.
- Use `card.reveal` only when the business case requires it.

## Top Up Card

`POST /v2/cards/top-up`

Request:

```json
{
  "id": "a441831c-a5c7-4bed-8f61-793738afd5bc",
  "amount": "50.00",
  "token_type": "USDT",
  "note": "Top up for campaign budget"
}
```

Required fields: `id`, `amount`, `token_type`.

Response data:

```json
{
  "tx_id": "tx_123",
  "card_balance": "150.25"
}
```

## Redeem Card

`POST /v2/cards/redeem`

Request:

```json
{
  "id": "a441831c-a5c7-4bed-8f61-793738afd5bc",
  "amount": "20.00",
  "token_type": "USDT",
  "note": "Redeem unused balance"
}
```

Required fields: `id`, `amount`, `token_type`.

Response data:

```json
{
  "tx_id": "tx_123",
  "card_balance": "80.25"
}
```

## Freeze Card

`POST /v2/cards/freeze`

Request:

```json
{
  "id": "a441831c-a5c7-4bed-8f61-793738afd5bc"
}
```

Response data:

```json
{
  "success": true,
  "message": "Card frozen"
}
```

## Unfreeze Card

`POST /v2/cards/unfreeze`

Request:

```json
{
  "id": "a441831c-a5c7-4bed-8f61-793738afd5bc"
}
```

Response data:

```json
{
  "success": true,
  "message": "Card unfrozen"
}
```
