# Withdrawals

Withdraw APIs move merchant funds from Infini to an external wallet address.

New integrations should use the independent `/v2/funds` prefix. The legacy `/v1/acquiring/fund/withdraw` routes remain available for backward compatibility only.

## Organization Fund APIs

All `/v2/funds` endpoints require HMAC authentication plus the `fund.withdraw` permission.

### Apply Withdraw

`POST /v2/funds/withdraw`

Request:

```json
{
  "request_id": "e94b4e88-36c2-4550-907e-839742cf5fae",
  "chain": "ETHEREUM",
  "token_type": "USDT",
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
| `amount` | Yes | Positive decimal string. Digits after the sixth decimal place are truncated without rounding. |
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
- New integrations should query `GET /v2/funds/withdraw/fees` for the enabled chain/token pairs and global default fees instead of hard-coding a fee table. This is not an organization-specific fee quote.
- API keys that include `fund.withdraw` should always be protected by a strict IP allowlist.

### Query Withdraw Status

`GET /v2/funds/withdraw/status?request_id={request_id}`

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
    "token_type": "USDT"
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

### List Withdrawal Fees

`GET /v2/funds/withdraw/fees`

Returns global default fees for enabled chain/token pairs. Actual withdrawals use the organization's configured fee, including an explicit zero fee. A business child account without its own fee inherits its root account's configured fee; if neither has a configured fee, the global default applies. The fee list does not resolve these organization overrides, so it may differ from the charged fee. Use `gas_fee` and `gas_fee_currency` from withdrawal status for reconciliation.

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
    "available_balance_usdc": "75.25"
  }
}
```

### Supported Chains and Tokens

Use `GET /v2/funds/withdraw/fees` for enabled pairs and global default fees. Organization-specific fees can differ as described above. In current public docs:

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
