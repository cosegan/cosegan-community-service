# Lago: A Stripe error after Stripe has charged leaves the invoice failed, and a retry charges the customer again (v1.57.0; a comment on getlago/lago-api#6903, where it was first reported)

Found via [Cosegan](https://github.com/cosegan) formal/mathematical proof based verification and analysis.

**Effects:** the customer is charged twice for one invoice, and Lago shows one payment; for an hour in between, every retry of the payment is accepted and silently dropped.

We see the same on v1.57.0, the newest release. On `main` today the Stripe lines cited above are unchanged; `invoices/payments/create_service.rb` has been reworked, but a failed payment is still followed by a new payment, with a new id and so a new idempotency key.

**We verified this on our sandbox only**: Lago v1.57.0 from the released Docker image (`getlago/api:v1.57.0`) with its PostgreSQL, Redis and Sidekiq worker, and Stripe's test mode. A proxy of ours in front of Stripe (not included here; we can send it) let Stripe process `POST /v1/payment_intents` and then answered Lago with an HTTP 500. We delivered Stripe's events to Lago's webhook ourselves, signed as Stripe signs them. We have not seen it in production.

One run, a one-off invoice of 26.00 USD (times UTC, 2026-11-20):

- 22:56:51 Stripe made a PaymentIntent of 2600 `usd`, `succeeded`. Lago: invoice `failed`, payment `failed` with no `provider_payment_id`.
- Stripe's `payment_intent.succeeded` for it, delivered: Lago answered 200 and nothing changed.
- 22:57:19 `POST /api/v1/invoices/:id/retry_payment`: 200, nothing was sent to Stripe, and the API logged `Not unique Invoices::Payments::CreateJob`. At 23:17 the job's lock was still in Redis with 39.6 minutes to live.
- 23:56:58, once the lock had expired, `retry_payment` again: a new payment, and Stripe a second PaymentIntent of 2600, `succeeded`.

At the end Stripe held two succeeded charges of 26.00 for the invoice. Lago showed the invoice paid once (`total_paid_amount_cents` 2600), with the first payment still `failed`.

One point for the fix suggested above (retry with the same `payment-<id>` key): if the retry reaches Stripe while the first request is still running, Stripe answers 409 `idempotency_key_in_use`. stripe-ruby 6.5.0 raises that as `Stripe::APIError` (only a 400 or 404 of type `idempotency_error` becomes `Stripe::IdempotencyError`), so it lands in the same `rescue ::Stripe::StripeError` branch and the payment is recorded as failed. We ran this once: the connection dropped after Stripe had received the request, Lago's job retried with the same key and got a 409, and the invoice ended with two succeeded charges of 27.00 as above. The 409 in that run came from our proxy, not from Stripe.
