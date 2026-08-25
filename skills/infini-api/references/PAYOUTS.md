# Fiat Payouts

Fiat Payout APIs send USD from the authenticated organization's Infini USD balance to a third-party bank account.

All payout endpoints use the independent `/v2/payouts` prefix.

## Permissions

| Permission | Endpoints | Notes |
| --- | --- | --- |
| `payout.write` | `POST /v2/payouts/quotes`, `POST /v2/payouts` | Required to quote and create payouts. Protect these API keys with a strict IP allowlist. |
| `payout.read` | `GET /v2/payouts/{payout_id}`, `POST /v2/payouts/status/batch` | Required to read payout status. |

## Minimum API Surface

| Method | Endpoint | Purpose |
| --- | --- | --- |
| `POST` | `/v2/payouts/quotes` | Validate one bank destination and obtain a time-limited payout quote. |
| `POST` | `/v2/payouts` | Create and immediately start a payout using a valid quote. |
| `POST` | `/v2/payouts/status/batch` | Query up to 100 payout statuses in one request. |
| `GET` | `/v2/payouts/{payout_id}` | Query one payout's latest status. |

There is no reusable recipient resource in this release. Bank destination details are submitted with the quote request and are immutably bound to `quote_id`.

## Quote Modes

Only these combinations are supported:

| `amount_mode` | `fee_paid_by` | Meaning |
| --- | --- | --- |
| `receive` | `PAYER` | The recipient receives the exact requested amount. Fees are added on top of the sender's debit. |
| `send` | `BENEFICIARY` | The sender's debit stays exact. Fees are deducted from the recipient amount. |

Change `fee_paid_by` or `amount_mode` by requesting a new quote. `quote_id` freezes the destination snapshot, fee responsibility, rate, and calculated amounts used by `POST /v2/payouts`.

## Execution Model

- API payouts do not enter the dashboard approval workflow.
- `POST /v2/payouts` performs the authoritative pre-execution checks and starts payout execution immediately.
- A successful create response usually returns `status=processing`; that means execution started, not that the receiving bank has credited the account.
- `client_reference_id` is merchant-scoped idempotency for one logical payout. Reuse it only for retries of the same business request.
- Only one payout submission per organization can enter execution at a time. A concurrent request fails with `108015 payout_submission_in_progress`; retry later with the same `client_reference_id` for the same logical payout.

## Statuses

| Status | Meaning |
| --- | --- |
| `processing` | Execution started and final settlement is pending. |
| `completed` | The banking partner reported completion at the latest known state. |
| `failed` | The payout failed before successful bank completion. |

A receiving bank can still reject and return a previously completed transfer. In that case the payout stays `completed`, and the compensating refund is exposed through the optional `return` object.

## Batch Status Query

`POST /v2/payouts/status/batch`

Use this route to reconcile between 1 and 100 known payout IDs in a single request.

- Results preserve request order.
- A found payout item returns `success=true`.
- An invalid, unknown, or inaccessible payout ID returns `success=false` plus an item-level `error_code` and `error_message`.
- Item-level failures do not fail the whole batch response.

## Security Notes

- Treat bank-account details as sensitive data. Do not put raw bank details in URLs, logs, `client_reference_id`, or other free-text metadata.
- Keep payout credentials server-side only.
- Use decimal strings for all money fields.
