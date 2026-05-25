# Organization Cards

Card endpoints are merchant APIs under `/v1/acquiring/card`. They require HMAC authentication and card-specific permissions.

## Identifier Rule

The merchant-facing card identifier is always `id`:

- `data.id` from `POST /v1/acquiring/card/apply`
- `cards[].id` from `GET /v1/acquiring/card/list`
- `id` passed to `GET /v1/acquiring/card/status`
- `id` passed to `POST /v1/acquiring/card/reveal`

Do not invent or expose other internal card identifiers.

## Permissions

| Permission | Endpoints |
| --- | --- |
| `card.create` | Apply, list, and status. |
| `card.reveal` | Reveal PAN/CVV/expiry. |

API keys with card permissions should have a non-empty IP allowlist.

## Apply Card

`POST /v1/acquiring/card/apply`

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

`GET /v1/acquiring/card/list?status=active,pending&card_alias=Travel&page=1&page_size=20`

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

`GET /v1/acquiring/card/status?id={id}`

Poll after apply until `status` becomes `active`.

Status values:

- `init`
- `pending`
- `active`
- `suspend`
- `deleted`

## Reveal Card

`POST /v1/acquiring/card/reveal`

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
