# Shopizer 3.2.7

Six findings in the checkout and Stripe payment code of [Shopizer](https://github.com/shopizer-ecommerce/shopizer), release 3.2.7. Found by [Cosegan](https://cosegan.com) on 2026-11-20.

One more high-severity bug that affects merchants has been reported privately and responsibly to the Shopizer maintainer. Its details will be added here once it is fixed.

| # | Finding | Report | Posted to the project |
|---|---|---|---|
| 1 | Card charged but no order kept, and charged again when the shopper pays again | [report](charged-no-order.md) | — |
| 2 | Orders of 1,000.00 or more cannot be paid with stripe3, and the amount stays held on the shopper's card | [report](orders-from-1000-held-never-paid.md) | — |
| 3 | Shopper charged 100 times the price in a store priced in Japanese yen: ¥9,900 for a ¥99 order | [report](yen-charged-100-times.md) | — |
| 4 | Docker image 3.2.7 stops at start-up: database.properties not found | [report](image-does-not-start.md) | — |
| 5 | The stripe payment module, which the React shop uses, cannot take payments: Stripe refuses card tokens on the Charges API | [report](stripe-module-cannot-take-payments.md) | — |
| 6 | stripe3 payments cannot be completed from a shop page: payment init returns no PaymentIntent id or client secret | [report](init-gives-no-client-secret.md) | — |
