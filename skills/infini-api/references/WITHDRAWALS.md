# Withdrawals

Withdraw APIs move merchant funds from Infini to an external wallet address.

## Apply Withdraw

`POST /v1/acquiring/fund/withdraw`

Request:

```json
{
  "chain": "ETHEREUM",
  "token_type": "USDT",
  "amount": "6",
  "wallet_address": "0x5f716e5775b18409917e2a2f0762d29d6c385cb0",
  "note": "Treasury withdrawal"
}
```

Fields:

| Field | Required | Notes |
| --- | --- | --- |
| `chain` | Yes | Uppercase chain name. |
| `token_type` | Yes | For example `USDT` or `USDC`. |
| `amount` | Yes | Decimal string. |
| `wallet_address` | Yes | Destination address. |
| `note` | No | Merchant note. |

Response data:

```json
{
  "request_id": "e94b4e88-36c2-4550-907e-839742cf5fae"
}
```

## Query Withdraw Status

`GET /v1/acquiring/fund/withdraw?request_id={request_id}`

Response data:

```json
{
  "status": "completed",
  "amount": "11",
  "fee": "0.1",
  "actual_amount": "10.9",
  "transaction_hash": "0xabc123...",
  "chain": "ETHEREUM",
  "token_type": "USDT"
}
```

Status values:

| Status | Meaning |
| --- | --- |
| `pending` | Created, waiting to be submitted or under review. |
| `processing` | Submitted on-chain and waiting for confirmations. |
| `completed` | Confirmed on-chain. |
| `failed` | Failed on-chain. |
| `cancelled` | Canceled, for example over-limit cancellation. |

## Supported Chains and Tokens

Sandbox:

| Chain | Tokens |
| --- | --- |
| `TTRON` | `USDT` |

Production:

| Chain | Tokens | Fee |
| --- | --- | --- |
| `ETHEREUM` | `USDT`, `USDC` | `5` |
| `BSC` | `USDT`, `USDC` | `0.5` |
| `SOLANA` | `USDT`, `USDC` | `1` |
| `ARBITRUM` | `USDT`, `USDC` | `0.5` |
| `TRON` | `USDT` | `3` |

Fees are deducted in the same token type.

## Safety

- Never initiate withdraws from frontend code.
- Confirm destination addresses in merchant systems before calling Infini.
- Use API keys with minimum withdrawal permissions and strict IP allowlists.
- Poll status until terminal state and keep local audit logs.
