---
name: infini-api
description: Use when working with the Infini merchant API to integrate hosted checkout, HMAC-SHA256 authentication, order creation/querying, payment creation/querying, Binance Pay, withdrawals, subscriptions, organization cards, webhooks, error handling, or troubleshooting merchant API requests in Python, Node.js, TypeScript, Go, Java, or other backend languages.
---

# Infini API Skill

## Overview

Use this skill to generate, review, or troubleshoot Infini merchant API integrations. Treat the public docs as the source of truth and generate backend-only code that protects secrets and handles payment state safely.

API basics:

- Production base URL: `https://openapi.infini.money`
- Sandbox base URL: `https://openapi-sandbox.infini.money`
- Merchant API prefix: `/v1/acquiring` for acquiring, payments, withdrawals, and subscriptions
- Card API prefix: `/v2/cards`
- Standard response envelope: `code`, `message`, `data`

## Security Requirements

Apply these defaults before writing code:

- Keep API key secrets and webhook secrets on the server only. Never expose them in frontend, mobile, logs, or committed config.
- Load secrets from a secrets manager or environment variables. Do not hardcode example secrets except obvious placeholders.
- Generate a globally unique `request_id` for create calls and persist it before calling Infini. Reuse it on retry.
- Treat money as strings or decimal types. Never use float/double for payment amounts.
- Send `Digest` for requests with a body, but do not include the body digest in the signing string.
- Verify every webhook with `X-Webhook-Signature`, `X-Webhook-Timestamp`, and `X-Webhook-Event-Id`.
- Process webhooks idempotently using `X-Webhook-Event-Id`; return HTTP 200 quickly after validation and enqueue business work when needed.
- Respect terminal order/subscription states. Do not roll a completed/canceled/expired local state backward because of late or duplicate webhooks.
- Configure API key IP allowlists in the merchant dashboard, especially for card and withdrawal permissions.

## Reference Selection

Read only the reference files needed for the user request:

- New integration or onboarding: `references/GETTING_STARTED.md`
- HMAC request signing: `references/AUTH.md`
- Hosted checkout orders: `references/HOSTED_CHECKOUT.md`
- Programmatic crypto payments or Binance Pay: `references/PAYMENTS.md`
- Withdrawals: `references/WITHDRAWALS.md`
- Subscriptions: `references/SUBSCRIPTIONS.md`
- Organization cards: `references/CARDS.md`
- Webhooks: `references/WEBHOOKS.md`
- Error handling and troubleshooting: `references/ERROR_HANDLING.md`
- Endpoint map: `references/API_REFERENCE.md`

## Implementation Guidance

For generated clients:

- Before generating a full project, ask how the user wants to provide credentials and secrets. Default to environment variables with a `.env.example` unless they choose a secret manager:
  - `INFINI_API_KEY`
  - `INFINI_API_SECRET`
  - `INFINI_WEBHOOK_SECRET`
- Build a small reusable request signer around `method`, exact path including query string, and optional JSON body.
- Serialize JSON bodies consistently before calculating `Digest`; send exactly the serialized bytes that were digested.
- Use the exact path in signatures, for example `/v1/acquiring/order?order_id=...`.
- For card APIs, use the exact `/v2/cards/...` path in signatures, not `/v1/acquiring/card/...`.
- Check both HTTP status and response `code`; business success is `code === 0`.
- Keep hosted checkout as the default recommendation. Use payment APIs only when the merchant explicitly needs programmatic payment address or Binance Pay creation.
- Include webhook handling for production payment flows; polling `GET` endpoints may be added as a reconciliation fallback.

## Do Not Document As Public API

Do not expose private implementation details, internal dashboard APIs, generated DAO code, database migrations, gateway internals, provider credentials, or routes that are not merchant-facing public APIs. If a generated `api.json` includes routes that are not covered by the public merchant documentation, do not add them to this skill.
