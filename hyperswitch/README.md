# Hyperswitch v1.129.0

Eleven findings in the payment code of [Hyperswitch](https://github.com/juspay/hyperswitch), release v1.129.0, with Stripe as the processor. Found by [Cosegan](https://cosegan.com) on 2026-11-20.

| # | Finding | Report | Posted to the project |
|---|---|---|---|
| 1 | Double charge after a dropped connection to Stripe | [report](double-charge-after-dropped-connection.md) | https://github.com/juspay/hyperswitch/discussions/15354 |
| 2 | Refund paid out twice after a dropped connection to Stripe | [report](refund-paid-twice-after-dropped-connection.md) | https://github.com/juspay/hyperswitch/discussions/15354 |
| 3 | Refund made at Stripe but recorded as failed after a dropped connection | [report](refund-recorded-failed-after-dropped-connection.md) | https://github.com/juspay/hyperswitch/discussions/15354 |
| 4 | Capture made at Stripe but payment recorded as failed, then charged again on retry | [report](capture-recorded-failed-then-charged-again.md) | https://github.com/juspay/hyperswitch/discussions/15354 |
| 5 | Void made at Stripe but payment recorded as failed | [report](void-recorded-failed-after-dropped-connection.md) | https://github.com/juspay/hyperswitch/discussions/15354 |
| 6 | Void with a free-text cancellation_reason fails at Stripe and leaves the hold on the card | [report](void-with-free-text-reason-strands-hold.md) | — |
| 7 | Charged payment left processing for good after a Stripe timeout or reset | [report](charged-payment-left-processing-after-timeout-or-reset.md) | https://github.com/juspay/hyperswitch/discussions/15355 |
| 8 | Scheduler marks a payment Stripe charged as failed; a retry charges the customer again | [report](scheduler-marks-charged-payment-failed.md) | https://github.com/juspay/hyperswitch/discussions/15355 |
| 9 | Stripe webhook answered with HTTP 500 for a payment whose answer was lost | [report](webhook-500-for-payment-with-lost-answer.md) | https://github.com/juspay/hyperswitch/discussions/15355 |
| 10 | Payout sent twice after a dropped connection to Stripe | [report](payout-sent-twice-after-dropped-connection.md) | https://github.com/juspay/hyperswitch/discussions/15354 |
| 11 | Payout made at Stripe but recorded as failed after a dropped connection | [report](payout-made-recorded-failed.md) | https://github.com/juspay/hyperswitch/discussions/15354 |
