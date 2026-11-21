# Kill Bill 0.24.22

Nine findings in the payment code of [Kill Bill](https://github.com/killbill/killbill) and two of its payment plugins: the Stripe plugin 8.0.4, the Braintree plugin 1.0.3 and Kill Bill 0.24.22 itself. Found by [Cosegan](https://cosegan.com) between 2026-11-12 and 2026-11-16.

**How they were found.** Cosegan formally verifies software: what the payment code must guarantee is written down as exact statements (one request charges the customer at most once; what Kill Bill records is what the processor did) and then proved or refuted. For the Stripe plugin several statements did not hold. Some of the findings here came out of that work, the others from reading and testing the code around it. Each was then reproduced on a test server (Kill Bill in Docker, the processor's test mode, one network fault made on purpose where a fault is needed), and each report's steps were run as written on a fresh server on 2026-11-18. None of this was seen in production.

| # | Finding | In | Report | Posted to the project |
|---|---|---|---|---|
| 1 | Double charge after a network failure | Stripe plugin 8.0.4 | [report](stripe-double-charge-after-network-failure.md) | https://github.com/killbill/killbill/issues/2340 |
| 2 | Customer charged but payment recorded as unknown | Braintree plugin 1.0.3 | [report](braintree-charged-but-payment-unknown.md) | https://github.com/killbill/killbill/issues/2341 |
| 3 | Capture made at Stripe but recorded as failed | Stripe plugin 8.0.4 | [report](stripe-capture-recorded-as-failed.md) | https://github.com/killbill/killbill/issues/2342 |
| 4 | Double charge after a refused reconnection | Stripe plugin 8.0.4 | [report](stripe-double-charge-after-refused-reconnection.md) | https://github.com/killbill/killbill/issues/2343 |
| 5 | Payment taken but invoice stays unpaid after a timeout | Stripe plugin 8.0.4 | [report](stripe-payment-taken-invoice-unpaid-after-timeout.md) | https://github.com/killbill/killbill/issues/2344 |
| 6 | Refund made but recorded as failed | Stripe plugin 8.0.4 | [report](stripe-refund-recorded-as-failed.md) | https://github.com/killbill/killbill/issues/2345 |
| 7 | Void made at Stripe but recorded as failed | Stripe plugin 8.0.4 | [report](stripe-void-recorded-as-failed.md) | https://github.com/killbill/killbill/issues/2346 |
| 8 | Expired 3-D Secure payments left open at Stripe | Stripe plugin 8.0.4 | [report](stripe-expired-3ds-payments-left-open.md) | https://github.com/killbill/killbill/issues/2347 |
| 9 | Failed void answered with 204 | Kill Bill 0.24.22 | [report](killbill-failed-void-answered-204.md) | https://github.com/killbill/killbill/issues/2348 |
