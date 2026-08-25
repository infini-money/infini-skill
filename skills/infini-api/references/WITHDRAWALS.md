# Withdrawals

Withdraw APIs move organization funds from Infini to an external wallet address.

New integrations should use the independent `/v2/funds` prefix. The legacy `/v1/acquiring/fund/withdraw` routes remain available for backward compatibility only.

## Organization Fund APIs

All `/v2/funds` endpoints require HMAC authentication plus the `fund.withdraw` permission.

- API keys with `fund.withdraw` should always use a strict IP allowlist.
- For GET requests with query parameters, the HMAC signing path must include the full query string.
- For POST requests with a body, include `Digest` and sign the exact `/v2/funds/...` path.

### Apply Withdraw

`POST /v2/funds/withdraw`

Request:

```json
{
  "request_id": "e94b4e88-36c2-4550-907e-839742cf5fae",
  "chain": "ETHEREUM",
  "token_type": "USDT",
  "source_currency": "USD",
  "amount": "10.00",
  "wallet_address": "0x5f716e5775b18409917e2a2f0762d29d6c385cb0",
  "note": "Treasury withdrawal"
}
```

Fields:

| Field | Required | Notes |
| --- | --- | --- |
| `request_id` | No | Optional UUID idempotency key. Reuse it on retry for the same logical withdraw. |
| `chain` | Yes | Uppercase public chain name such as `ETHEREUM`, `ARBITRUM`, `SOLANA`, `BSC`, `BASE`, or `TRON`. |
| `token_type` | Yes | Target on-chain token type such as `USDT` or `USDC`. |
| `source_currency` | No | Defaults to `token_type`. Set it to `USD` to deduct from the organization's USD Cash Account when withdrawing `USDT` or `USDC`; other cross-currency pairs are rejected. |
| `amount` | Yes | Maximum amount deducted from the source account. Digits after the sixth decimal place are truncated without rounding; when `source_currency=USD`, at most two decimal places are allowed. |
| `wallet_address` | Yes | Destination address valid for the selected chain. |
| `note` | No | Merchant note. |

Response envelope:

```json
{
  "code": 0,
  "message": "",
  "data": {
    "request_id": "e94b4e88-36c2-4550-907e-839742cf5fae",
    "status": "pending",
    "is_duplicate": false,
    "fee_paid_by": "BENEFICIARY"
  }
}
```

Notes:

- When the same merchant retries with the same `request_id`, the API returns the existing withdrawal with `data.is_duplicate = true`.
- `data.fee_paid_by` is currently fixed to `BENEFICIARY` for this V1 withdrawal surface.
- New integrations should query `GET /v2/funds/withdraw/fees` for enabled chain/token pairs and current fees instead of hard-coding a fee table.

### Query Withdraw Status

`GET /v2/funds/withdraw/status?request_id={request_id}`

`request_id` accepts either the original UUID idempotency key or the canonical `wdw_<32-hex>` withdrawal identifier.

Response envelope:

```json
{
  "code": 0,
  "message": "",
  "data": {
    "request_id": "wdw_8876f07b288845b38d16534ebb4f66e0",
    "status": "completed",
    "amount": "11",
    "fee": "0.1",
    "actual_amount": "10.9",
    "transaction_hash": "0xabc123",
    "chain": "ETHEREUM",
    "token_type": "USDT",
    "source_currency": "USD",
    "gas_fee": "0.08",
    "gas_fee_currency": "USDT",
    "fx_fee": "0.02",
    "fx_fee_currency": "USD",
    "fee_paid_by": "BENEFICIARY"
  }
}
```

Response fields to pay attention to:

- `fee` is a legacy gas-fee alias. New integrations should use `gas_fee` and `gas_fee_currency`.
- `actual_amount` is the amount actually sent on-chain in `token_type`.
- `source_currency` is the debited account currency.
- `fx_fee` is `0` for same-currency withdrawals.

Status values:

| Status | Meaning |
| --- | --- |
| `pending` | Created, waiting to be submitted or reviewed. |
| `processing` | Submitted on-chain and waiting for confirmations. |
| `completed` | Confirmed and completed. |
| `failed` | Failed. |

### List Withdrawal Statuses

`POST /v2/funds/withdraw/status/batch`

Request:

```json
{
  "start_time": 1787500000,
  "end_time": 1787599999,
  "page": 1,
  "page_size": 20
}
```

Rules:

- `start_time` and `end_time` are inclusive Unix timestamps in seconds.
- Results are ordered by `created_at` descending.
- `page` defaults to `1`.
- `page_size` defaults to `20` and cannot exceed `100`.
- An empty matching range still returns HTTP 200 with an empty `withdrawals` array.

Response envelope:

```json
{
  "code": 0,
  "message": "",
  "data": {
    "withdrawals": [
      {
        "request_id": "wdw_8876f07b288845b38d16534ebb4f66e0",
        "created_at": 1787564636,
        "status": "completed",
        "amount": "11",
        "fee": "0.1",
        "actual_amount": "10.9",
        "transaction_hash": "0xabc123",
        "chain": "ETHEREUM",
        "token_type": "USDT",
        "source_currency": "USDT",
        "gas_fee": "0.1",
        "gas_fee_currency": "USDT",
        "fx_fee": "0",
        "fx_fee_currency": "USDT",
        "fee_paid_by": "BENEFICIARY"
      }
    ],
    "total": 1,
    "page": 1,
    "page_size": 20,
    "total_pages": 1
  }
}
```

### List Withdrawal Fees

`GET /v2/funds/withdraw/fees`

Response envelope:

```json
{
  "code": 0,
  "message": "",
  "data": {
    "fees": [
      {
        "chain": "ARBITRUM",
        "token_type": "USDC",
        "withdraw_fee": "0.5"
      },
      {
        "chain": "ETHEREUM",
        "token_type": "USDT",
        "withdraw_fee": "2"
      }
    ]
  }
}
```

Each `withdraw_fee` is denominated in the corresponding `token_type`.

### Get Available Balances

`GET /v2/funds/balances`

Response envelope:

```json
{
  "code": 0,
  "message": "",
  "data": {
    "available_balance_usdt": "120.50",
    "available_balance_usdc": "75.25",
    "available_balance_usd": "1000.00"
  }
}
```

`available_balance_usd` returns `"0"` when the organization does not have a USD Cash Account.

## Legacy Compatibility Routes

These older endpoints still exist for backward compatibility:

- `POST /v1/acquiring/fund/withdraw`
- `GET /v1/acquiring/fund/withdraw?request_id={request_id}`

Legacy notes:

- The legacy create response returns only `request_id`.
- Legacy status responses may still include `cancelled`.

## Safety

- Never initiate withdraws from frontend code.
- Confirm destination addresses in merchant systems before calling Infini.
- Use API keys with `fund.withdraw` and strict IP allowlists.
- Poll or reconcile status until terminal state and keep local audit logs.
