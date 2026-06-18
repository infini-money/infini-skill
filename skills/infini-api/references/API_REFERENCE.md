# API Reference

Concise merchant-facing endpoint map. Most endpoints use base path `/v1/acquiring`; card endpoints use the independent `/v2/cards` prefix.

Use public documentation as the inclusion rule. A generated `api.json` may contain routes that are not part of the public merchant API; do not add those routes to this reference unless they are also covered by the public merchant documentation.

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
| `POST` | `/v2/cards/apply` | Apply organization card. | HMAC + `card.create` |
| `GET` | `/v2/cards/list` | List organization cards. | HMAC + `card.create` |
| `GET` | `/v2/cards/status?id={id}` | Query card status. | HMAC + `card.create` |
| `POST` | `/v2/cards/reveal` | Reveal PAN/CVV/expiry. | HMAC + `card.reveal` |
| `POST` | `/v2/cards/top-up` | Top up organization card. | HMAC + `card.create` |
| `POST` | `/v2/cards/redeem` | Redeem funds from organization card. | HMAC + `card.create` |
| `POST` | `/v2/cards/freeze` | Freeze organization card. | HMAC + `card.create` |
| `POST` | `/v2/cards/unfreeze` | Unfreeze organization card. | HMAC + `card.create` |
| `POST` | merchant webhook URL | Infini order/subscription event push. | Verify webhook HMAC |

## Endpoint Notes

- Hosted checkout uses `/order`, `/order?order_id=...`, and `/order/token/reissue`.
- The public docs may describe token reissue as `/token/reissue`; the current public service route mounts it under the order group as `/order/token/reissue`.
- Card APIs are not mounted under `/v1/acquiring`; use the full `/v2/cards/...` path when signing and sending card requests.
- Webhook receiving URLs are merchant-owned, not Infini paths.
- Exclude routes that appear only in generated service specs but are not covered by the public merchant documentation.

## Payment Method Values

Known values:

- `1`: crypto.
- `2`: card.
- `3`: Binance Pay.
- `5`: Apple Pay.
- `6`: Google Pay.

For hosted checkout order creation, `pay_methods` can be a combined list such as `[1,2,3,5,6]` when those methods are enabled for the merchant. Omit `pay_methods` to use the merchant's configured defaults.

## Order Currency Values

Supported order currencies:

- `USD`
- `EUR`
- `KRW`
- `GBP`
- `SGD`
- `JPY`
- `AUD`
- `HKD`

`USD` is the default. Use whole-number amounts for zero-decimal currencies `JPY` and `KRW`.

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
