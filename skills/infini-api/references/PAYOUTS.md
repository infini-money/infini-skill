# Fiat Payouts

The Fiat Payout API lets an organization send USD from its Infini USD balance to a third-party bank account.

All payout endpoints use the independent `/v2/payouts` prefix and require HMAC authentication.

## Minimum API Surface

| Method | Endpoint | Purpose | Permission |
| --- | --- | --- | --- |
| `POST` | `/v2/payouts/quotes` | Validate bank details and obtain a time-limited payout quote. | `payout.write` |
| `POST` | `/v2/payouts` | Create and immediately start a direct payout. | `payout.write` |
| `POST` | `/v2/payouts/status/batch` | Query up to 100 payout statuses in one request. | `payout.read` |
| `GET` | `/v2/payouts/{payout_id}` | Query the latest status for one payout. | `payout.read` |

There is no recipient resource in this release. Bank destination details are submitted with the quote and are immutably bound to `quote_id`.

## Quote First

`POST /v2/payouts/quotes`

The quote request validates one destination and returns a time-limited pricing snapshot.

- The quote does not reserve balance.
- The quote does not create a payout.
- Expired or consumed quotes cannot be reused.
- `POST /v2/payouts` repeats all execution-critical checks before creating the real payout.

Supported amount-control combinations:

| Goal | `amount_mode` | `fee_paid_by` | Result |
| --- | --- | --- | --- |
| Recipient receives an exact amount | `receive` | `PAYER` | Fees are added to the organization's debit. |
| Organization debits an exact amount | `send` | `BENEFICIARY` | Fees are deducted from the recipient side. |

Other `amount_mode` and `fee_paid_by` combinations are not supported in the first release.

## Create Payout

`POST /v2/payouts`

Use a valid `quote_id` plus a merchant-scoped `client_reference_id`.

Key behavior:

- The create call executes directly and does not enter a dashboard approval workflow.
- A successful response usually returns `status=processing`; bank settlement is asynchronous.
- Retrying the same logical payout with the same `client_reference_id` returns the existing payout with `is_duplicate=true`.
- Reusing `client_reference_id` with different business data returns `409 idempotency_conflict`.

Only one payout submission per organization can enter execution at a time. A concurrent submission returns `108015 payout_submission_in_progress`; retry later with the same `client_reference_id` for the same logical payout.

## Status and Reconciliation

Single payout status:

- `processing`: execution started and final settlement is pending.
- `completed`: the banking partner reported payout completion at the latest known state.
- `failed`: payout failed before successful bank completion.

There is intentionally no approval-pending status for API payouts.

`POST /v2/payouts/status/batch` reconciles between 1 and 100 known payout IDs in one request.

- Results preserve request order.
- A found item returns `success=true`.
- An invalid, unknown, or inaccessible payout ID returns `success=false` with item-level `error_code` and `error_message`.
- Item-level failures do not fail the whole batch; the top-level response remains HTTP 200 with `code=0`.

Use `GET /v2/payouts/{payout_id}` for one payout or `POST /v2/payouts/status/batch` for multiple payouts as the authoritative public reconciliation interfaces.

## Bank Return and Refund

A receiving bank can later reject and return a payout that was previously marked `completed`.

- The original payout `status` stays `completed`.
- `failure` remains `null`.
- The optional `return` object describes the bank return and compensating refund.

Important `return` fields:

- `returned_amount`: amount reported as returned by the banking partner.
- `refund_amount`: amount actually credited back to the organization's balance.
- `retained_fee`: original payout fee that was not refunded.
- `returned_at`: Unix-second time when the compensating refund was recorded.

## Error Handling

Use both HTTP status and response-body `code`.

- HTTP status gives the error category.
- `code` and `data.error` give the stable business reason.
- `message` is for troubleshooting and should not drive program logic.

Common payout-specific errors:

- `108001 invalid_fee_mode`: unsupported `amount_mode` and `fee_paid_by` combination.
- `108006 insufficient_balance`: available USD balance is too low for `sending_amount`.
- `108010 payout_source_account_not_found`: the organization does not have a USD payout account.
- `108015 payout_submission_in_progress`: another payout submission is already entering execution.
- `108009 payout_service_unavailable`: temporary provider or execution issue; safe to retry later.

## Safety

- Keep payout requests server-side only; they contain sensitive bank-account data.
- Do not put raw bank details into URLs, logs, `client_reference_id`, `statement_reference`, or free-text metadata.
- Persist `client_reference_id` before calling the create endpoint.
- Treat `quote_id` as short-lived execution input, not a durable business identifier.
