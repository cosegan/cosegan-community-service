# Shopizer: The stripe payment module, which the React shop uses, cannot take payments: Stripe refuses card tokens on the Charges API

Found via [Cosegan](https://github.com/cosegan) formal/mathematical proof based verification and analysis.

**Effects:** a shop using the stripe module, as the official React shop does, cannot take a card payment through Stripe.

**Environment**

- Shopizer 3.2.7: the Docker image `shopizerecomm/shopizer:3.2.7` (the newest tag, pushed 2025-03-22) with its demo data (store `DEFAULT`, CAD); the code cited below is the default branch `3.2.7` (head `6a4a0a6`)
- Payment module `stripe` (Stripe's Charges API, stripe-java 19.5.0), Stripe test mode
- Docker on macOS (the image runs under amd64 emulation)

**We verified this on our sandbox only**: Shopizer's own image and demo data on one machine, and one Stripe test account. We have not seen it in production, and we have not tried an older Stripe account or live mode. The image does not start as published (a separate issue), so step 1 starts the same jar with the jar's own `profiles/docker/database.properties`; nothing else is changed.

**What we are seeing**

Every checkout with the `stripe` module fails with HTTP 503, and nothing is charged.

1. `StripePayment.authorizeAndCapture` (line 252) creates a charge with Stripe's Charges API and the card token from Stripe.js as its `source` (lines 296 to 307, `Charge.create`).
2. Stripe refuses that call with HTTP 400: "Card Token usage on the Customers API and Charges API has reached end of life. If you're building a new integration, read about supported options here: https://docs.stripe.com/payments/payment-methods/integration-options. If you have an existing integration, you need to migrate it to the Payment Intents and Payment Methods APIs, as described here: https://docs.stripe.com/payments/payment-methods/migrate-from-legacy-cards-sources"
3. Shopizer's log says only "InvalidRequest error with stripe": line 501 passes Stripe's message as an argument with no `{}` placeholder, so the reason is not logged.

The official React shop pays with this module (`shopizer-shop-reactjs`, `src/pages/other/Checkout.js`: `stripe.createToken(card)` at line 524, `"paymentModule": "stripe"` at lines 548 and 611), so a shop built on it cannot take a card payment through Stripe on a Stripe account like ours. Shopizer's other Stripe module, `stripe3`, uses PaymentIntents, but a shop page cannot complete a payment with it through the API (a separate issue).

From the run below:

```
module stripe: HTTP 200
{"timestamp":"2026-11-21T01:26:55.301+00:00","status":503,"error":"Service Unava
checkout: HTTP 503
InvalidRequest error with stripe
# the same call sent straight to Stripe
    "message": "Card Token usage on the Customers API and Charges API has reached end of life. If you're building a new integration, read about supported options here: https://docs.stripe.com/payments/payment-methods/integration-options. If you have an existing integration, you need to migrate it to the Payment Intents and Payment Methods APIs, as described here: https://docs.stripe.com/payments/payment-methods/migrate-from-legacy-cards-sources",
HTTP 400
```

**What we are expecting**

The Stripe module takes payments through an API Stripe accepts (PaymentIntents and PaymentMethods), and Shopizer's log keeps Stripe's reason when a call is refused.

**How to reproduce**

1. Start Shopizer:

```bash
# Needs Docker, curl, unzip and a Stripe TEST secret key:
#   export STRIPE_SECRET_KEY=sk_test_...
# The image does not start as published (a separate issue): the same jar is started with its own docker database.properties.
docker create --name shopizer-files --platform linux/amd64 shopizerecomm/shopizer:3.2.7 >/dev/null
docker cp shopizer-files:/opt/app/shopizer.jar ./shopizer.jar && docker rm shopizer-files >/dev/null
mkdir -p conf && unzip -p shopizer.jar BOOT-INF/classes/profiles/docker/database.properties > conf/database.properties
docker run -d --name shopizer --platform linux/amd64 -p 127.0.0.1:8080:8080 -v "$PWD/conf:/conf:ro" \
  shopizerecomm/shopizer:3.2.7 java -cp /opt/app/shopizer.jar -Dloader.path=/conf \
  -Dloader.main=com.salesmanager.shop.application.ShopApplication org.springframework.boot.loader.PropertiesLauncher >/dev/null
until curl -sf "http://127.0.0.1:8080/api/v1/store/DEFAULT?lang=en" >/dev/null; do sleep 3; done
```

2. Turn on the `stripe` module with your test key, and sign a shopper in:

```bash
S=http://127.0.0.1:8080; Q='store=DEFAULT&lang=en'
json() { curl -s -H 'Content-Type: application/json' "$@"; }
token() { grep -o '"token":"[^"]*"' | cut -d'"' -f4; }
ADMIN=$(json -X POST -d '{"username":"admin@shopizer.com","password":"password"}' "$S/api/v1/private/login" | token)
# The module's form also requires a publishable key; these steps do not use it.
json -X POST -H "Authorization: Bearer $ADMIN" -w 'module stripe: HTTP %{http_code}\n' "$S/api/v1/private/modules/payment?$Q" \
  -d '{"code":"stripe","active":true,"defaultSelected":true,"integrationKeys":{"publishableKey":"pk_test_not_used_by_these_steps","secretKey":"'"$STRIPE_SECRET_KEY"'","transaction":"AUTHORIZECAPTURE"}}'
json -X POST -o /dev/null -w 'register: HTTP %{http_code}\n' "$S/api/v1/customer/register?$Q" \
  -d '{"emailAddress":"shopper@example.com","userName":"shopper@example.com","password":"Password1!","repeatPassword":"Password1!","firstName":"Test","lastName":"Shopper","gender":"M","language":"en","billing":{"firstName":"Test","lastName":"Shopper","address":"1 Test Street","city":"Montreal","postalCode":"H2X1Y4","country":"CA","zone":"QC","stateProvince":"QC","phone":"5145550101"}}'
SHOPPER=$(json -X POST -d '{"username":"shopper@example.com","password":"Password1!"}' "$S/api/v1/customer/login?$Q" | token)
```

3. A cart with one chair (99.00), paid with a card token as the React shop pays (Stripe's test token `tok_visa` stands for the token Stripe.js makes):

```bash
CART=$(json -X POST -d '{"product":"chair1","quantity":1}' "$S/api/v1/cart?$Q" | grep -o '"code":"[^"]*"' | head -1 | cut -d'"' -f4)
json -X POST -H "Authorization: Bearer $SHOPPER" -w '\ncheckout: HTTP %{http_code}\n' "$S/api/v1/auth/cart/$CART/checkout?$Q" \
  -d '{"currency":"CAD","payment":{"paymentType":"CREDITCARD","transactionType":"AUTHORIZECAPTURE","paymentModule":"stripe","paymentToken":"tok_visa","amount":"99.00"}}' | cut -c1-80
docker logs shopizer 2>&1 | grep -o 'InvalidRequest error with stripe' | sort -u
```

4. The call `StripePayment` makes, sent straight to Stripe:

```bash
curl -s -u "$STRIPE_SECRET_KEY:" -H 'Stripe-Version: 2020-03-02' https://api.stripe.com/v1/charges \
  -d amount=9900 -d currency=cad -d source=tok_visa -d capture=true -w 'HTTP %{http_code}\n' | grep -E '"message"|HTTP'
```

**Possible fix**

Move the `stripe` module and the React shop to PaymentIntents with PaymentMethods, as Stripe's migration guide describes (`stripe3` is most of the way there, once its init returns the client secret), and log the exception's message in `StripePayment` (`LOGGER.error("InvalidRequest error with stripe: {}", ex.getMessage())`).
