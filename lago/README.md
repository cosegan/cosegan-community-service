# Lago v1.57.0

Two findings in the payment code of [Lago](https://github.com/getlago/lago-api), release v1.57.0. Found by [Cosegan](https://cosegan.com) on 2026-11-20.

One more high-severity bug that affects merchants has been reported privately and responsibly to the Lago team. Its details will be added here once it is fixed.

| # | Finding | Report | Posted to the project |
|---|---|---|---|
| 1 | Moneyhash payments drop the cents, and Lago records the full amount as paid | [report](moneyhash-cents-dropped.md) | — |
| 2 | A Stripe error after Stripe has charged leaves the invoice failed, and a retry charges the customer again (v1.57.0; a comment on getlago/lago-api#6903, where it was first reported) | [report](comment-on-6903-double-charge.md) | — |
