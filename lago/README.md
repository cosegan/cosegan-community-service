# Lago v1.57.0

Three findings in the payment code of [Lago](https://github.com/getlago/lago-api), release v1.57.0. Found by [Cosegan](https://cosegan.com) on 2026-11-20. Findings 1 and 2 were reported privately to the Lago team. Lago fixed both on 2026-11-23 and credited Cosegan for the reports. The details of finding 1 were held back here until the fix was merged.

| # | Finding | Report | Posted to the project | Fixed by the project |
|---|---|---|---|---|
| 1 | Invoices in HUF, ISK and UGX are charged one hundredth of their amount through Stripe and recorded as paid in full | [report](huf-isk-ugx-charged-one-hundredth.md) | reported privately | https://github.com/getlago/lago-api/pull/7319 |
| 2 | Moneyhash payments drop the cents, and Lago records the full amount as paid | [report](moneyhash-cents-dropped.md) | reported privately | https://github.com/getlago/lago-api/pull/7296 |
| 3 | A Stripe error after Stripe has charged leaves the invoice failed, and a retry charges the customer again (v1.57.0; a comment on getlago/lago-api#6903, where it was first reported) | [report](comment-on-6903-double-charge.md) | — | — |
