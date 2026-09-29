# Automatic Refunds

Automatic refunds are a merchant-console feature for eligible underpayments, overpayments, and late crypto payments. They are disabled by default and do not add a merchant API endpoint. Merchants should subscribe to `auto_refund.completed` and `auto_refund.unclaimed`, and handle both events idempotently using `X-Webhook-Event-Id`.

Refunds use the original paid token and require the payer to submit a valid same-network refund address through the email claim flow before its confirmation deadline. If no valid address is submitted in time, the refund becomes unclaimed and the amount is settled to the merchant.

## Gas and Minimum Payout

- For non-BTC refunds, Infini deducts the applicable fixed network gas fee from the original-token refund amount. The fee is configured in USD/USDC and converted with the payment's arrival-rate snapshot.
- BTC refunds deduct a fixed `0.00002 BTC`, independent of the BTC/USD price.
- A BTC automatic refund is created only if the payout remaining after that deduction is at least `0.00001 BTC`; the aggregated BTC refund must therefore be at least `0.00003 BTC`.
- When the applicable gas or minimum-payout threshold is not met, Infini does not create an automatic-refund record or send an `auto_refund.*` webhook. The amount remains available for merchant settlement.

Do not infer an automatic-refund outcome from an order's ordinary payment status alone; consume the final automatic-refund webhook for merchant automation.
