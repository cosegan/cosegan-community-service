# Jeepay V3.2.9

Seven findings in the payout, refund and barcode-payment code of [Jeepay](https://github.com/jeequan/jeepay), release V3.2.9. Found by [Cosegan](https://cosegan.com) on 2026-11-20. Each report is in Chinese first, then English.

| # | Finding | Report | Posted to the project |
|---|---|---|---|
| 1 | Payee paid twice after an Alipay payout times out | [report](alipay-payout-paid-twice.md) | — |
| 2 | Payee paid twice after a WeChat Pay payout times out | [report](wxpay-payout-paid-twice.md) | — |
| 3 | Double refund after a WeChat Pay refund times out | [report](wxpay-refund-twice-after-timeout.md) | — |
| 4 | Double refund after a slow WeChat Pay refund is closed at two hours | [report](wxpay-refund-twice-after-two-hour-close.md) | — |
| 5 | Double charge after an Alipay barcode payment times out | [report](alipay-barcode-double-charge.md) | — |
| 6 | WeChat Pay barcode payment taken but recorded as failed after a timeout | [report](wxpay-barcode-charged-recorded-failed.md) | — |
| 7 | Alipay barcode payment taken but recorded as failed on Alipay's "system busy" and "already paid" answers | [report](alipay-barcode-unknown-recorded-failed.md) | — |
