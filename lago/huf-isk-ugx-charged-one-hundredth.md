# Lago: Invoices in HUF, ISK and UGX are charged one hundredth of their amount through Stripe and recorded as paid in full

Found via [Cosegan](https://github.com/cosegan) formal/mathematical proof based verification and analysis.

**Effects:** an invoice in Hungarian forints, Icelandic krónur or Ugandan shillings is charged at one hundredth of its amount through Stripe, and Lago records it as paid in full; an invoice whose hundredth is under Stripe's minimum is never collected.

**We verified this on our sandbox only**: Lago v1.57.0 from the released Docker image (`getlago/api:v1.57.0`) with its own PostgreSQL, Redis and Sidekiq worker, and Stripe's test mode. We have not seen it in production. The lines named below are the same on `main` today. In our runs Lago reached Stripe through a small pass-through proxy of ours that forwarded the requests unchanged; it plays no part in this bug and is not needed to reproduce it.

**Describe the bug**

An invoice in Hungarian forints (HUF), Icelandic krónur (ISK) or Ugandan shillings (UGX) is charged through Stripe at one hundredth of its amount, and Lago records it as paid in full.

Lago keeps amounts as integers of the currency's smallest unit as the Money gem defines it. In money 7.0.2 that unit is the whole forint, króna and shilling (`"subunit_to_unit": 1` for HUF, ISK and UGX in its `config/currency_iso.json`), so an invoice of 50,000 HUF has `total_amount_cents` 50000. Stripe reads the `amount` of these three currencies in hundredths: for ISK and UGX it asks for two-decimal amounts for backward compatibility (5 ISK is sent as 500), and HUF is charged in two-decimal amounts ([Stripe: currencies, special cases](https://docs.stripe.com/currencies#special-cases)). Lago sends its number unchanged (`app/services/payment_providers/stripe/payments/create_service.rb`, line 160: `amount: payment.amount_cents`), so Stripe charges HUF 500.00. Stripe answers `succeeded` with `amount` 50000, the number Lago sent, and Lago marks the payment and the invoice as paid in full.

From our runs. Stripe's balance transaction converts what Stripe took into the account's currency (USD here, at test mode's rates):

| Invoice in Lago | Lago afterwards | Stripe's charge | Stripe's balance transaction | What Stripe took |
|---|---|---|---|---|
| 50,000 HUF | `payment_status` `succeeded`, `total_paid_amount_cents` 50000 | `amount` 50000, `huf` | 1.47 USD at 0.00293717 | HUF 500.00 |
| 100,000 ISK | `succeeded`, 100000 | 100000, `isk` | 7.99 USD at 0.00799309 | ISK 1,000 |
| 1,000,000 UGX | `succeeded`, 1000000 | 1000000, `ugx` | 2.37 USD at 0.000237034 | UGX 10,000 |
| 500.00 EUR (control) | `succeeded`, 50000 | 50000, `eur` | 548.46 USD at 1.09692 | EUR 500.00 |

When one hundredth of the invoice is below Stripe's minimum charge, the invoice is not collected at all. For an invoice of 10,000 HUF (about 29 USD), Stripe refused the PaymentIntent with `amount_too_small`, in its words: "Amount must convert to at least 50 cents. 100.00 Ft converts to approximately $0.29." Lago keeps such an invoice `pending`, as it does for amounts under Stripe's minimum (the same `create_service.rb`, lines 39 to 42), so it stays unpaid with no failure shown.

We compared Stripe's currency lists with money 7.0.2 for every currency Lago offers: these three are the only ones where they disagree.

**To Reproduce**

You need curl and jq, a Lago v1.57.0 connected to a Stripe test account (Settings → Integrations → Stripe; we used the connection code `stripe_test`), a Lago API key, and the same Stripe test secret key.

```bash
# export LAGO_URL=http://localhost:3000 LAGO_API_KEY=<your Lago API key> STRIPE_SECRET_KEY=sk_test_...
lago()   { curl -s -H "Authorization: Bearer $LAGO_API_KEY" -H "Content-Type: application/json" "$@"; }
stripe() { curl -s -g -u "$STRIPE_SECRET_KEY:" "$@"; }
```

1. A Stripe test customer whose default payment method is Stripe's test card, and the same customer in Lago, in HUF:

```bash
CUS=$(stripe https://api.stripe.com/v1/customers -d name=huf-repro | jq -r .id)
PM=$(stripe https://api.stripe.com/v1/payment_methods/pm_card_visa/attach -d customer=$CUS | jq -r .id)
stripe https://api.stripe.com/v1/customers/$CUS -d "invoice_settings[default_payment_method]=$PM" | jq -c '{id, default_payment_method: .invoice_settings.default_payment_method}'

lago -X POST "$LAGO_URL/api/v1/customers" -d '{"customer": {"external_id": "huf-repro", "name": "HUF repro", "currency": "HUF",
  "billing_configuration": {"payment_provider": "stripe", "payment_provider_code": "stripe_test",
    "provider_customer_id": "'"$CUS"'", "sync_with_provider": false, "provider_payment_methods": ["card"]}}}' \
  | jq -c '.customer | {external_id, currency, provider_customer_id: .billing_configuration.provider_customer_id}'
sleep 5   # Lago reads the card from Stripe
```

2. A one-off invoice of 50,000 HUF, which Lago charges at once:

```bash
lago -X POST "$LAGO_URL/api/v1/add_ons" -d '{"add_on": {"name": "HUF repro", "code": "huf-repro", "amount_cents": 50000, "amount_currency": "HUF"}}' \
  | jq -c '.add_on | {code, amount_cents, amount_currency}'
INVOICE=$(lago -X POST "$LAGO_URL/api/v1/invoices" -d '{"invoice": {"external_customer_id": "huf-repro", "currency": "HUF",
  "fees": [{"add_on_code": "huf-repro", "units": 1, "unit_amount_cents": 50000}]}}' | jq -r .invoice.lago_id)
sleep 20
```

3. The invoice in Lago, and the charge at Stripe:

```bash
lago "$LAGO_URL/api/v1/invoices/$INVOICE" | jq -c '.invoice | {currency, total_amount_cents, total_paid_amount_cents, payment_status}'
stripe -G https://api.stripe.com/v1/charges -d customer=$CUS -d "expand[]=data.balance_transaction" \
  | jq -c '.data[] | {amount, currency, status, settled: .balance_transaction.amount, settled_currency: .balance_transaction.currency, exchange_rate: .balance_transaction.exchange_rate}'
```

What we got:

```
{"currency":"HUF","total_amount_cents":50000,"total_paid_amount_cents":50000,"payment_status":"succeeded"}
{"amount":50000,"currency":"huf","status":"succeeded","settled":147,"settled_currency":"usd","exchange_rate":0.00293717}
```

Stripe settled 1.47 USD, which at that rate is HUF 500. The same happens with ISK or UGX in place of HUF (with another external id and add-on code).

**Expected behavior**

Stripe charges the invoice's amount: 50,000 HUF, which Stripe expects as `amount` 5000000.

**Screenshots**

None. Lago's and Stripe's answers are quoted as text above.

**Additional context**

- The same unchanged number goes to Stripe for checkout links (`app/services/invoices/payments/stripe_service.rb`, line 162: `unit_amount: invoice.total_due_amount_cents`) and for refunds (`app/services/credit_notes/refunds/stripe_service.rb`, line 217: `amount: credit_note.refund_amount_cents`). We read these and did not run them.
- In getlago/lago#325 a maintainer confirmed that Lago treats HUF as having no decimals. That matches Money's table; the conversion for Stripe is what is missing.
- Read in the code, not run (we have no Adyen account): Lago sends Adyen `value: payment.amount_cents` unchanged too (`app/services/payment_providers/adyen/payments/create_service.rb`, line 101). [Adyen's table of minor units](https://docs.adyen.com/development-resources/currency-codes) differs from money 7.0.2 for five currencies Lago offers. For CLP, HUF and ISK, Adyen expects two decimals and Money has none, so Adyen would take one hundredth. For CVE and IDR, Adyen expects none and Money has two, so Adyen would take **a hundred times** the invoice.
- A possible fix: for HUF, ISK and UGX, multiply Lago's number by 100 on the way to Stripe (payments, checkout links, refunds) and divide Stripe's amounts by 100 on the way back.

**Version**

- Lago: v1.57.0 (`getlago/api:v1.57.0`; lago-api `af1b5856`), money 7.0.2, stripe-ruby 6.5.0

---

Reported to the project privately, and published here only after the fix was merged.

Fixed by the project in https://github.com/getlago/lago-api/pull/7319 (merged 2026-11-23), where Lago thanks Cosegan for the report. The Adyen amounts under "Additional context" were fixed in https://github.com/getlago/lago-api/pull/7313 (merged the same day).
