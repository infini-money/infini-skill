# API Reference

Concise merchant-facing endpoint map. Base path is `/v1/acquiring`.

Use public documentation as the inclusion rule. A local generated `api.json` may contain additional routes for platform plugins, CRM/support forms, or implementation-specific flows; do not add those to the generic public merchant API reference unless they are also documented as public merchant APIs.

## Authentication

Most merchant server-to-server endpoints require:

- `Date`
- `Authorization`
- `Digest` for JSON body requests
- `Content-Type: application/json` for JSON body requests

See `AUTH.md`.

## Public Endpoints

| Method | Path | Purpose | Auth |
| --- | --- | --- | --- |
| `GET` | `/currency` | Supported cryptocurrencies available for creating orders. | HMAC |
| `POST` | `/order` | Create hosted checkout order. | HMAC |
| `GET` | `/order?order_id={order_id}` | Query order. | HMAC |
| `GET` | `/order/list` | List merchant orders. | HMAC |
| `POST` | `/order/token/reissue` | Reissue checkout token. | HMAC |
| `POST` | `/payment` | Create crypto payment. | HMAC |
| `GET` | `/payment?payment_id={payment_id}` | Query payment. | HMAC |
| `GET` | `/payment/list?order_id={order_id}` | List order payments. | HMAC |
| `POST` | `/payment/binance` | Create Binance Pay payment. | HMAC |
| `GET` | `/payment/binance?prepay_id={prepay_id}` | Query Binance Pay status. | HMAC |
| `POST` | `/fund/withdraw` | Apply withdraw. | HMAC |
| `GET` | `/fund/withdraw?request_id={request_id}` | Query withdraw status. | HMAC |
| `POST` | `/subscription` | Create subscription and first checkout order. | HMAC |
| `GET` | `/subscription?merchant_sub_id={merchant_sub_id}` | Query subscription. | HMAC |
| `POST` | `/subscription/cancel` | Cancel subscription. | HMAC |
| `GET` | `/subscription/invoice/checkout` | User invoice checkout link. | Link/JWT flow |
| `POST` | `/subscription/unsubscribe` | User unsubscribe link. | Link/JWT flow |
| `POST` | `/card/apply` | Apply organization card. | HMAC + `card.create` |
| `GET` | `/card/list` | List organization cards. | HMAC + `card.create` |
| `GET` | `/card/status?id={id}` | Query card status. | HMAC + `card.create` |
| `POST` | `/card/reveal` | Reveal PAN/CVV/expiry. | HMAC + `card.reveal` |
| `GET` | `/paymentlink/info?url_key={url_key}` | Get public payment link info. | Public |
| `POST` | `/paymentlink/checkout` | Create order through public payment link. | Public |
| `POST` | merchant webhook URL | Infini order/subscription event push. | Verify webhook HMAC |

## Endpoint Notes

- Hosted checkout uses `/order`, `/order?order_id=...`, and `/order/token/reissue`.
- The public docs may describe token reissue as `/token/reissue`; the current public service route mounts it under the order group as `/order/token/reissue`.
- Webhook receiving URLs are merchant-owned, not Infini paths.
- Exclude generated-but-not-general-public routes such as `/shopify/*`, `/shoplazza/*`, `/support/*`, and internal dashboard routes under `/api/v1/*` unless the user asks about that specific integration and provides public docs for it.

## Payment Method Values

Known values:

- `1`: crypto.
- `2`: card.
- `3`: Binance Pay.

## Status Summary

Order:

- `pending`
- `processing`
- `paid`
- `partial_paid`
- `expired`

Withdraw:

- `pending`
- `processing`
- `completed`
- `failed`
- `cancelled`

Subscription:

- `pending`
- `active`
- `canceled`

Card:

- `init`
- `pending`
- `active`
- `suspend`
- `deleted`
