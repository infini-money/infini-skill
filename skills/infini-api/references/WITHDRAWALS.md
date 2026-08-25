# Withdrawals

Withdraw APIs move organization funds from Infini to an external wallet address.

New integrations should use the independent `/v2/funds` prefix. The legacy `/v1/acquiring/fund/withdraw` routes remain available for backward compatibility only.

## Organization Fund APIs

All `/v2/funds` endpoints require HMAC authentication plus the `fund.withdraw` permission.

| Method | Endpoint | Purpose |
| --- | --- | --- |
| `POST` | `/v2/funds/withdraw` | Create one withdrawal. |
| `GET` | `/v2/funds/withdraw/status?request_id={request_id}` | Query one withdrawal status by request ID. |
| `POST` | `/v2/funds/withdraw/status/batch` | List withdrawal statuses by creation-time window. |
| `GET` | `/v2/funds/withdraw/fees` | List currently enabled chain/token withdrawal fees. |
| `GET` | `/v2/funds/balances` | Read available USDT, USDC, and USD Cash balances. |

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
| `request_id` | No | Optional UUID idempotency key. Reuse it on retry. |
| `chain` | Yes | Uppercase public chain name such as `ETHEREUM`, `ARBITRUM`, `SOLANA`, `BSC`, `BASE`, or `TRON`. |
| `token_type` | Yes | For example `USDT`, `USDC`, `ETH`, `SOL`, `BNB`, or `TRX` depending on chain. |
| `source_currency` | No | Defaults to `token_type`. Set it to `USD` to deduct from the organization's USD Cash Account when withdrawing `USDT` or `USDC`; other cross-currency pairs are rejected. |
| `amount` | Yes | Positive decimal string. Digits after the sixth decimal place are truncated without rounding. When `source_currency` is `USD`, at most two decimal places are allowed. |
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
    "is_duplicate": false
  }
}
```

Notes:

- When the same merchant retries with the same `request_id`, the API returns the existing withdrawal with `data.is_duplicate = true`.
- New integrations should query `GET /v2/funds/withdraw/fees` for the enabled chain/token pairs and their current fees instead of hard-coding a fee table.
- API keys that include `fund.withdraw` should always be protected by a strict IP allowlist.

### Query Withdraw Status

`GET /v2/funds/withdraw/status?request_id={request_id}`

`request_id` may be either the UUID supplied on create or the business ID returned by Infini, such as `wdw_...`.

Response envelope:

```json
{
  "code": 0,
  "message": "",
  "data": {
    "request_id": "e94b4e88-36c2-4550-907e-839742cf5fae",
    "status": "completed",
    "amount": "11",
    "fee": "0.1",
    "actual_amount": "10.9",
    "transaction_hash": "0xabc123...",
    "chain": "ETHEREUM",
    "token_type": "USDT",
    "source_currency": "USD",
    "gas_fee": "0.1",
    "gas_fee_currency": "USDT",
    "fx_fee": "0",
    "fx_fee_currency": "USD",
    "fee_paid_by": "BENEFICIARY"
  }
}
```

Status values:

| Status | Meaning |
| --- | --- |
| `pending` | Created, waiting to be submitted or under review. |
| `processing` | Submitted on-chain and waiting for confirmations. |
| `completed` | Confirmed on-chain. |
| `failed` | Failed on-chain. |

### List Withdraw Statuses

`POST /v2/funds/withdraw/status/batch`

Request:

```json
{
  "start_time": 1787011200,
  "end_time": 1787097599,
  "page": 1,
  "page_size": 20
}
```

Fields:

| Field | Required | Notes |
| --- | --- | --- |
| `start_time` | Yes | Inclusive Unix timestamp in seconds. |
| `end_time` | Yes | Inclusive Unix timestamp in seconds. |
| `page` | No | Page number, default `1`. |
| `page_size` | No | Items per page, default `20`, max `100`. |

Response envelope:

```json
{
  "code": 0,
  "message": "",
  "data": {
    "withdrawals": [
      {
        "request_id": "wdw_01K3H8M8ANM5A7YV7Q4QEX5B64",
        "created_at": 1787088800,
        "status": "completed",
        "amount": "11",
        "fee": "0.1",
        "actual_amount": "10.9",
        "transaction_hash": "0xabc123...",
        "chain": "ETHEREUM",
        "token_type": "USDT",
        "source_currency": "USD",
        "gas_fee": "0.1",
        "gas_fee_currency": "USDT",
        "fx_fee": "0",
        "fx_fee_currency": "USD",
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

Notes:

- Results are ordered by withdrawal creation time descending.
- This is a POST request with a JSON body, so include `Digest` and sign the exact path `/v2/funds/withdraw/status/batch`.

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

`available_balance_usd` is `"0"` when the organization does not have a USD Cash Account open.

### Supported Chains and Tokens

Use `GET /v2/funds/withdraw/fees` as the source of truth. In current public docs:

| Chain | Tokens | Fee |
| --- | --- | --- |
| `ETHEREUM` | `USDT`, `USDC`, `ETH` | dynamic |
| `ARBITRUM` | `USDT`, `USDC` | dynamic |
| `SOLANA` | `USDT`, `USDC`, `SOL` | dynamic |
| `BSC` | `USDT`, `USDC`, `BNB` | dynamic |
| `BASE` | `USDT`, `USDC` | dynamic |
| `TRON` | `USDT`, `TRX` | dynamic |

`TTRON` may appear in non-production environments only.

## Legacy Compatibility Routes

These older endpoints still exist for backward compatibility:

- `POST /v1/acquiring/fund/withdraw`
- `GET /v1/acquiring/fund/withdraw?request_id={request_id}`

Legacy notes:

- The legacy create response returns only `request_id` rather than the richer `/v2/funds` envelope data.
- Legacy status responses may still include `cancelled`.

## Safety

- Never initiate withdraws from frontend code.
- Confirm destination addresses in merchant systems before calling Infini.
- Use API keys with the `fund.withdraw` permission and strict IP allowlists.
- Poll status until terminal state and keep local audit logs.
