# Error Handling and Troubleshooting

Infini responses use an envelope:

```json
{
  "code": 0,
  "message": "",
  "data": {}
}
```

Business success is `code === 0`. Non-zero `code` means the request was received but failed business validation or authorization.

Error example:

```json
{
  "code": 40001,
  "message": "Invalid request",
  "detail": "expires_at must be greater than current timestamp"
}
```

## Common Error Codes

| HTTP | Code | Meaning |
| --- | --- | --- |
| 400 | `40003` | Amount must be positive. |
| 400 | `40006` | Amount must be greater than `0.01`. |
| 401 | `401` | Invalid HMAC signature. |
| 404 | `40401` | Order does not exist. |
| 409 | `40902` | `client_reference` duplicate. |
| 409 | `40906` | Order expired. |
| 404 | `43000` | Subscription not found. |
| 400 | `43002` | Subscription already canceled. |

## Troubleshooting Invalid HMAC Signature

Check:

- `Date` is GMT and within +/- 300 seconds.
- Signing string uses exact method and path including query string.
- Signing string format is exactly:

```text
{keyId}
{METHOD} {path}
date: {GMT_time}
```

- There is a trailing newline after the `date` line.
- `Digest` is base64 SHA-256 of the exact request body bytes.
- Body digest is not included in the signing string.
- `Authorization` uses the same `keyId` that belongs to the secret.
- Request comes from an IP allowed by the API key.

## Troubleshooting Order Issues

- Duplicate local retry: reuse the original `request_id`; do not create a second order accidentally.
- Buyer says paid but order is processing: inspect `amount_confirming`, `amount_confirmed`, and payment transactions.
- Expired order with payment: handle `order.late_payment` or `partial_paid` according to merchant policy.
- Frontend redirect succeeded but webhook missing: query order and inspect webhook delivery logs/retry settings.

## Troubleshooting Webhooks

- Use raw request body for signature verification.
- Compare signatures with constant-time comparison.
- Deduplicate by `X-Webhook-Event-Id`.
- Return HTTP 200 after recording the event, then process asynchronously.
- If business processing fails after 200, run local retry/reconciliation; Infini will consider the event delivered.

## Troubleshooting Cards

- Use `id`, not masked card number, for status and reveal.
- Ensure API key has `card.create` or `card.reveal`.
- Ensure API key has an IP allowlist.
- Do not log full reveal responses.

## Troubleshooting Withdrawals

- Chain and token are uppercase.
- Amount is a decimal string.
- Poll by `request_id`.
- Treat `pending`, `processing`, `completed`, `failed`, and `cancelled` as distinct lifecycle states.
