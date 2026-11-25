# Lago: Moneyhash payments drop the cents, and Lago records the full amount as paid

Found via [Cosegan](https://github.com/cosegan) formal/mathematical proof based verification and analysis.

**Effects:** every Moneyhash payment is collected without its cents, and Lago records the invoice as paid in full.

**We verified this on our sandbox only**: Lago v1.57.0 from the released Docker image (`getlago/api:v1.57.0`) with its own PostgreSQL, Redis and Sidekiq worker. We have no Moneyhash account, so Moneyhash was replaced by a stand-in of ours: a small server that records each request Lago sends and answers the way Lago's own test fixtures show Moneyhash answering (not included here; we can send it). So we saw what Lago sends; what Moneyhash then charges is taken from Moneyhash's documentation. We have not seen it in production. The line named below is the same on `main` today.

**Describe the bug**

Every invoice paid through Moneyhash is charged without its cents, and Lago records it as paid in full.

Lago turns the payment's amount into Moneyhash's decimal `amount` with an integer division (`app/services/payment_providers/moneyhash/payments/create_service.rb`, line 43):

```ruby
amount: payment.amount_cents.div(100).to_f,
```

`Integer#div` drops the remainder, so 2550 becomes `25.0`. Moneyhash's `amount` is in major units with up to two decimals. Its [API reference](https://docs.moneyhash.io/reference/v11_payments_intent_create) says "The amount to charge. The field must have no more than 14 total digits and no more than 2 decimal places", and its examples charge `"amount": 18.75` USD ([recurring payments](https://docs.moneyhash.io/docs/recurring-payments)) and `"amount": "150"` AED ([create a payment](https://docs.moneyhash.io/docs/create-a-payment)). So for an invoice of USD 25.50, Moneyhash is asked for USD 25.00. When Moneyhash reports the payment processed, Lago marks the payment and the invoice as paid with its own amount, 2550.

What Lago sent to our stand-in, and what it recorded afterwards:

| Invoice | Request body Lago sent (excerpt) | Lago afterwards |
|---|---|---|
| USD 25.50 | `"amount": 25.0, "amount_currency": "USD"` | `payment_status` `succeeded`, `total_paid_amount_cents` 2550 |
| USD 30.00 (control) | `"amount": 30.0, "amount_currency": "USD"` | `succeeded`, 3000 |
| KWD 12.345 | `"amount": 123.0, "amount_currency": "KWD"` | `succeeded`, 12345 |

The last row: for a currency with three decimals (Money keeps KWD in fils, 1000 to the dinar), dividing by 100 makes the number ten times too large. We do not know whether Moneyhash accepts KWD or any other three-decimal currency; its documentation allows two decimals and we found no list of currencies.

Lago already reads Moneyhash's amounts correctly in the other direction (`app/services/payment_providers/moneyhash/handle_event_service.rb`, lines 95 to 107: `Money.from_amount(value.to_d, currency).cents`).

**To Reproduce**

You need curl and jq, a Lago v1.57.0 connected to a Moneyhash sandbox account (Settings → Integrations → Moneyhash, with its API key and a flow ID; we used the connection code `moneyhash_test`), and a Lago API key.

```bash
# export LAGO_URL=http://localhost:3000 LAGO_API_KEY=<your Lago API key>
lago() { curl -s -H "Authorization: Bearer $LAGO_API_KEY" -H "Content-Type: application/json" "$@"; }
```

1. A customer in USD who pays through Moneyhash. Lago creates the customer at Moneyhash:

```bash
lago -X POST "$LAGO_URL/api/v1/customers" -d '{"customer": {"external_id": "mh-repro", "name": "Moneyhash repro", "email": "mh-repro@example.com", "currency": "USD",
  "billing_configuration": {"payment_provider": "moneyhash", "payment_provider_code": "moneyhash_test", "sync_with_provider": true}}}' \
  | jq -c '.customer | {external_id, currency, payment_provider: .billing_configuration.payment_provider}'
```

2. Save a card for the customer: open the checkout URL Lago makes for the customer and enter one of Moneyhash's test cards. Moneyhash sends `card_token.created` to Lago, and Lago stores the card.

```bash
lago -X POST "$LAGO_URL/api/v1/customers/mh-repro/checkout_url" | jq -r .customer.checkout_url
# after the card is saved:
lago "$LAGO_URL/api/v1/customers/mh-repro/payment_methods" | jq -c '.payment_methods[] | {provider_method_id, is_default}'
```

3. A one-off invoice of USD 25.50, which Lago charges at once:

```bash
lago -X POST "$LAGO_URL/api/v1/add_ons" -d '{"add_on": {"name": "Moneyhash repro", "code": "mh-repro", "amount_cents": 2550, "amount_currency": "USD"}}' \
  | jq -c '.add_on | {code, amount_cents, amount_currency}'
INVOICE=$(lago -X POST "$LAGO_URL/api/v1/invoices" -d '{"invoice": {"external_customer_id": "mh-repro", "currency": "USD",
  "fees": [{"add_on_code": "mh-repro", "units": 1, "unit_amount_cents": 2550}]}}' | jq -r .invoice.lago_id)
sleep 20
lago "$LAGO_URL/api/v1/invoices/$INVOICE" | jq -c '.invoice | {currency, total_amount_cents, total_paid_amount_cents, payment_status}'
```

What we got from Lago:

```
{"currency":"USD","total_amount_cents":2550,"total_paid_amount_cents":2550,"payment_status":"succeeded"}
```

4. In Moneyhash, the payment intent for this invoice has the amount Lago sent. In our run Lago's request body had `"amount": 25.0`.

**Expected behavior**

Moneyhash is asked for the invoice's amount: USD 25.50 for this invoice.

**Screenshots**

None. The request body and Lago's answers are quoted as text above.

**Additional context**

- The same division is used for Moneyhash payment links, for invoices (`app/services/invoices/payments/moneyhash_service.rb`, line 141: `amount: invoice.total_due_amount_cents.div(100).to_f`) and for payment requests (`app/services/payment_requests/payments/moneyhash_service.rb`, line 150). We read these and did not run them.
- When Moneyhash's event comes back, an existing payment is set to `succeeded` from its status alone (`app/services/invoices/payments/moneyhash_service.rb`, lines 18 to 35); the event's amount is used only when Lago has to create a missing payment. So the difference never shows in Lago.
- A possible fix: convert with the currency's own subunit, for example `Money.new(payment.amount_cents, payment.amount_currency).to_d`, instead of dividing by 100, and without a Float.

**Version**

- Lago: v1.57.0 (`getlago/api:v1.57.0`; lago-api `af1b5856`)

---

Reported to the project privately.

Fixed by the project in https://github.com/getlago/lago-api/pull/7296 (merged 2026-11-23), where Lago thanks Cosegan for the report.
