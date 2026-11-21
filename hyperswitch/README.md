# Hyperswitch v1.129.0

Eleven findings in the payment code of [Hyperswitch](https://github.com/juspay/hyperswitch), release v1.129.0, with Stripe as the processor. Found by [Cosegan](https://cosegan.com) on 2026-11-20.

**How they were found.** The payment code was read against a short list of things it must keep: a card is charged at most once; a refund is made as recorded; a payment the processor took is not recorded as failed; every payment is settled in the end. Each place where the code could break one of them was then run on the released Docker images against Stripe's test mode, with one network fault made on purpose where a fault is needed. Every finding was then reproduced a second time on a setup built separately. None of this was seen in production.

| # | Finding | Report | Posted to the project |
|---|---|---|---|
| 1 | Double charge after a dropped connection to Stripe | [report](double-charge-after-dropped-connection.md) | — |
| 2 | Refund paid out twice after a dropped connection to Stripe | [report](refund-paid-twice-after-dropped-connection.md) | — |
| 3 | Refund made at Stripe but recorded as failed after a dropped connection | [report](refund-recorded-failed-after-dropped-connection.md) | — |
| 4 | Capture made at Stripe but payment recorded as failed, then charged again on retry | [report](capture-recorded-failed-then-charged-again.md) | — |
| 5 | Void made at Stripe but payment recorded as failed | [report](void-recorded-failed-after-dropped-connection.md) | — |
| 6 | Void with a free-text cancellation_reason fails at Stripe and leaves the hold on the card | [report](void-with-free-text-reason-strands-hold.md) | — |
| 7 | Charged payment left processing for good after a Stripe timeout or reset | [report](charged-payment-left-processing-after-timeout-or-reset.md) | — |
| 8 | Scheduler marks a payment Stripe charged as failed; a retry charges the customer again | [report](scheduler-marks-charged-payment-failed.md) | — |
| 9 | Stripe webhook answered with HTTP 500 for a payment whose answer was lost | [report](webhook-500-for-payment-with-lost-answer.md) | — |
| 10 | Payout sent twice after a dropped connection to Stripe | [report](payout-sent-twice-after-dropped-connection.md) | — |
| 11 | Payout made at Stripe but recorded as failed after a dropped connection | [report](payout-made-recorded-failed.md) | — |
