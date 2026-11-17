# Customer deposits

Use the Customer Deposit API when a merchant needs a permanent address for each of its end users to receive repeated, variable-amount digital-asset deposits. It is separate from hosted checkout and uses the `/v2/merchant-customer` prefix.

## Authentication and permissions

Requests use Ed25519 v2 authentication, not the HMAC request signer used by `/v1/acquiring`. Send the required `X-API-Key-ID`, `X-API-Timestamp`, `X-API-Nonce`, `Digest`, and `Authorization: Ed25519 signature="..."` headers. Generate a fresh timestamp, nonce, digest, and signature for every retry.

- Read routes require `merchant_customer.read`.
- Create/update routes require `merchant_customer.write`.
- Verify both HTTP status and the standard `{code,message,data}` envelope.

## Customer and address lifecycle

- `POST /customers` creates or updates the Customer identified by `external_user_id`; first creation returns HTTP 201 and an existing Customer returns HTTP 200. Omitted profile fields are preserved on updates, while explicit `null` or an empty string clears them.
- Customer IDs are 14-digit int64 values. Preserve them as strings or safe integer types in languages where JSON numbers lose integer precision.
- `GET /customers` requires exactly one of `customer_id` or `external_user_id`.
- `GET /customers/quota` returns `limit` and `used`; calculate remaining capacity as `max(limit-used, 0)`. A quota read does not reserve a slot.
- `POST /addresses` is idempotent for merchant, environment, Customer, asset, and network. It returns HTTP 201 for a new ready binding and HTTP 200 for an existing binding. Retry a timeout with the same fields.
- `GET /addresses` is read-only. It neither allocates a wallet nor creates a binding.
- Query `GET /currencies` before choosing an asset/network. `min_deposit_amount` is a display threshold in asset units calculated from the current price; it is not a price lock.

The same Customer may have separate asset/network binding IDs that share a physical address. Do not infer a balance or transfer eligibility from address reuse; use the exact received asset/network and only the completed webhook for final credit.

## Webhook processing

Customer deposit webhooks use HMAC-SHA256 with the merchant Webhook Secret, not Ed25519. Verify the raw payload using `timestamp.event_id.raw_payload`, require `X-Webhook-Signature-Version: v1`, and retain the versioned envelope fields `id`, `event`, `version`, `occurred_at`, and `data`.

Process only `merchant_customer_deposit.completed` as a final credit signal. Its `credited_amount`, `credited_currency`, and `completed_at` are non-null. `transaction_id` is non-null only for a positive credit; a zero-net completed deposit has `credited_amount="0"`, a null `transaction_id`, and no merchant ledger settlement. Deduplicate final business processing by `data.deposit_id`, in addition to delivery deduplication by event ID.

`confirming` and `confirmed` are progress notifications only. A `failed` event includes a chain failure code and must not credit the user. Deliveries may be duplicated, delayed, or out of order.

## Error handling

- `47001` Customer not found and `47002` deposit address not found: verify merchant/environment and identifiers.
- `47003` Customer limit reached: refresh quota; existing Customers can still be updated.
- `47007` address pool exhausted: contact Infini to replenish the merchant wallet pool, then retry.
- `47005` price unavailable and `47006` dependency unavailable: back off and retry unchanged business fields.
- `40007` unsupported chain/token and `40915` disabled currency: refresh `GET /currencies` and correct the requested pair.
