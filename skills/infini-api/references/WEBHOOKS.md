# Webhooks

Infini sends webhooks for order, subscription, and organization card state changes.

## Event Types

| Subscribe to | Events received |
| --- | --- |
| `order.create` | `order.create` |
| `order.update` | `order.processing`, `order.completed`, `order.expired`, `order.late_payment` |
| `subscription.update` | `subscription.update` |
| `subscription.cancel` | `subscription.cancel` |
| `card.status_change` | `card.status_change` |
| `card.transaction` | `card.transaction` |
| `card.challenge` | `card.challenge` |

Order event meanings:

- `order.create`: order created and pending.
- `order.processing`: partial payment received or funds confirming.
- `order.completed`: full amount received and confirmed.
- `order.expired`: order expired.
- `order.late_payment`: payment received within 24 hours after expiration.

Card event meanings:

- `card.status_change`: organization card lifecycle snapshot, for example activation, suspension, or final deletion.
- `card.transaction`: organization card transaction update, including top-up, consume, refund, reversal, clearing, and the latest status.
- `card.challenge`: one-time authorization code challenge for an organization card transaction.

Use exact event names when creating new webhook subscriptions. Wildcard subscriptions are not supported by the current webhook management API, and `card.challenge` must be subscribed explicitly if you need those events.

## Headers

Every webhook should include:

- `Content-Type: application/json`
- `X-Webhook-Timestamp`
- `X-Webhook-Event-Id`
- `X-Webhook-Signature-Version` with the current value `v1`
- `X-Webhook-Signature`

Use `X-Webhook-Event-Id` for idempotency.

## Signature Verification

Read the raw request body, not a parsed/re-serialized JSON object.

Signed content:

```text
{timestamp}.{event_id}.{payload}
```

Expected signature:

```text
hex(HMAC-SHA256(WEBHOOK_SECRET, signed_content))
```

Python Flask example:

```python
import hashlib
import hmac
import time

from flask import Flask, jsonify, request

app = Flask(__name__)
WEBHOOK_SECRET = "load-from-secret-manager"


def valid_webhook() -> tuple[bool, str]:
    signature = request.headers.get("X-Webhook-Signature")
    signature_version = request.headers.get("X-Webhook-Signature-Version")
    timestamp = request.headers.get("X-Webhook-Timestamp")
    event_id = request.headers.get("X-Webhook-Event-Id")
    if not signature or not timestamp or not event_id:
        return False, "missing required webhook headers"
    if signature_version and signature_version != "v1":
        return False, "unsupported webhook signature version"

    now = int(time.time())
    if abs(now - int(timestamp)) > 300:
        return False, "webhook timestamp outside tolerance"

    payload = request.get_data(as_text=True)
    signed_content = f"{timestamp}.{event_id}.{payload}"
    expected = hmac.new(
        WEBHOOK_SECRET.encode("utf-8"),
        signed_content.encode("utf-8"),
        hashlib.sha256,
    ).hexdigest()
    if not hmac.compare_digest(expected, signature):
        return False, "invalid webhook signature"
    return True, event_id


@app.post("/infini/webhook")
def infini_webhook():
    ok, result = valid_webhook()
    if not ok:
        return jsonify({"error": result}), 401

    event_id = result
    event = request.get_json()
    # Deduplicate event_id, enqueue work, and return quickly.
    return jsonify({"status": "ok", "event_id": event_id})
```

## Order Payload

Common fields:

- `event`
- `order_id`
- `client_reference`
- `amount`
- `currency`
- `status`
- `amount_confirmed`
- `amount_confirming`
- `created_at`
- `updated_at`
- `exception_tags`

Example:

```json
{
  "event": "order.completed",
  "order_id": "20290d05-8f5c-4ecb-84f0-f78d6f30557f",
  "client_reference": "ORDER-10001",
  "amount": "1",
  "currency": "USD",
  "status": "paid",
  "amount_confirmed": "1",
  "amount_confirming": "0",
  "created_at": 1763512349,
  "updated_at": 1763512573
}
```

## Subscription Payload

Common fields:

- `event`
- `subscription_id`
- `merchant_sub_id`
- `plan_name`
- `trigger_method`
- `status`
- `currency`
- `amount`
- `interval_unit`
- `interval_count`
- `payer_email`
- `current_period_start`
- `current_period_end`
- `next_invoice_at`
- `cancel_reason`
- `canceled_at`
- `created_at`
- `updated_at`

Cancellation fields appear only when relevant.

## Card Webhook Envelope

Organization card webhooks use a versioned envelope:

- `id`
- `event`
- `version`
- `occurred_at`
- `data`

Order and subscription webhooks keep the legacy flat JSON payload shape.

## Card Status Change Payload

`card.status_change` sends:

- `data.card.card_id`
- `data.card.alias`
- `data.card.last_four`
- `data.card.status`
- `data.card.currency`

`pending_delete` is not emitted as a standalone status-change webhook. Wait for `deleted` or reconcile with `GET /v2/cards/status`.

Example:

```json
{
  "id": "b7ef2c62-6177-4ea8-84ec-3080f2db58f0",
  "event": "card.status_change",
  "version": 1,
  "occurred_at": 1763513200,
  "data": {
    "card": {
      "card_id": "a441831c-a5c7-4bed-8f61-793738afd5bc",
      "alias": "Travel card",
      "last_four": "1234",
      "status": "active",
      "currency": "USD"
    }
  }
}
```

## Card Transaction Payload

`card.transaction` sends:

- `data.card`
- `data.transaction_id`
- `data.related_transaction_id` when relevant
- `data.type`: `apply_card`, `top_up`, `redeem`, `consume`, `refund`, `reversal`, or `clearing`
- `data.status`: `pending`, `authorized`, `completed`, or `failed`
- `data.amount`
- `data.fee`
- `data.currency`
- `data.direction`: `debit`, `credit`, or `none`
- optional `data.transaction_amount` and `data.transaction_currency`
- optional `data.failure`
- optional `data.auth_settle_adjustment`

Failed transactions use `direction = none`. Refunds and reversals use `direction = credit`. Completed `consume` events can include `auth_settle_adjustment` when authorization and settlement differ.

## Card Challenge Payload

`card.challenge` sends:

- `data.card`
- `data.challenge_id`
- `data.challenge_type` (currently `authorization_code`)
- `data.challenge`
- `data.expires_at`

Handle `card.challenge` immediately, because it is delivered only until `expires_at`. Provider-only IDs, merchant routing data, ciphertext, PAN, and CVV are not exposed in the public payload, and `card.challenge` cannot be manually resent from the developer console after terminal delivery.

## Retry Strategy

Infini retries when the endpoint does not return HTTP 200:

- Maximum retries: 8 sends total.
- Early retries are around 30 seconds apart.
- Later retries use incremental backoff.

Return 200 only after validating the signature and recording/deduplicating the event.

## State Handling

- Use webhooks as the primary fulfillment signal.
- Make processing idempotent by event ID and by business object state.
- Never fulfill from frontend redirects alone.
- For `order.late_payment`, order status may remain `expired`; decide ship/refund policy in merchant logic.
- Do not roll terminal local state backward on late duplicate events.
- Process `card.challenge` immediately and do not assume it can be replayed later.
