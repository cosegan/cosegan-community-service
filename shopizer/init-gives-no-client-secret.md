# Shopizer: stripe3 payments cannot be completed from a shop page: payment init returns no PaymentIntent id or client secret

Found via [Cosegan](https://github.com/cosegan) formal/mathematical proof based verification and analysis.

**Effects:** a stripe3 payment cannot be completed from a shop page, because payment init returns no PaymentIntent id or client secret.

**Environment**

- Shopizer 3.2.7: the Docker image `shopizerecomm/shopizer:3.2.7` (the newest tag, pushed 2025-03-22) with its demo data (store `DEFAULT`, CAD); the code cited below is the default branch `3.2.7` (head `6a4a0a6`)
- Payment module `stripe3` (Stripe PaymentIntents), Stripe test mode
- Docker on macOS (the image runs under amd64 emulation)

**We verified this on our sandbox only**: Shopizer's own image and demo data on one machine, and Stripe's test mode. We have not seen it in production. The image does not start as published (a separate issue), so step 1 starts the same jar with the jar's own `profiles/docker/database.properties`; nothing else is changed.

**What we are seeing**

With `stripe3`, a shop page starts a payment with `POST /api/v1/cart/{code}/payment/init`. Shopizer creates a PaymentIntent at Stripe, but its answer carries neither the PaymentIntent's id nor its client secret. Without the client secret the page cannot confirm the payment with Stripe.js, and without the id it cannot name the payment at checkout (`paymentToken`), so a `stripe3` payment cannot be completed through the API.

1. `Stripe3Payment.initTransaction` creates the PaymentIntent and puts its id and client secret into the transaction's details map (`TRANSACTIONID`, line 95; `INTENTSECRET`, line 98).
2. `PaymentServiceImpl.initTransaction` stores the transaction with `transactionService.save` (line 775). Only `TransactionServiceImpl.create` turns the details map into the stored `details` (lines 38 to 48), so `details` stays null.
3. `OrderPaymentApi.init` (line 88) answers with that transaction through `ReadableTransactionPopulator`, which copies `details` (line 40): `"details":null`.

The official React shop (`shopizer-shop-reactjs`, `src/pages/other/Checkout.js`) does not call `payment/init`; it sends a card token to the older `stripe` module.

From the run below:

```
module stripe3: HTTP 200
{"id":1,"orderId":null,"details":null,"transactionDate":"2026-11-21","amount":"$99.00","paymentType":"CREDITCARD","transactionType":"AUTHORIZE"}
HTTP 200
# the newest PaymentIntent in the test account: the one that init created
      "id": "pi_…Vc89zK8",
      "amount": 9900,
      "capture_method": "manual",
      "currency": "cad",
      "status": "requires_payment_method",
client secrets in Stripe's answer: 1
```

**What we are expecting**

The init answer carries what a shop page needs to finish the payment: the PaymentIntent's client secret, to confirm it with Stripe.js, and its id, for the checkout.

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

2. Turn on `stripe3` with your test key:

```bash
S=http://127.0.0.1:8080; Q='store=DEFAULT&lang=en'
json() { curl -s -H 'Content-Type: application/json' "$@"; }
ADMIN=$(json -X POST -d '{"username":"admin@shopizer.com","password":"password"}' "$S/api/v1/private/login" | grep -o '"token":"[^"]*"' | cut -d'"' -f4)
# The module's form also requires a publishable key; these steps do not use it.
json -X POST -H "Authorization: Bearer $ADMIN" -w 'module stripe3: HTTP %{http_code}\n' "$S/api/v1/private/modules/payment?$Q" \
  -d '{"code":"stripe3","active":true,"defaultSelected":true,"integrationKeys":{"publishableKey":"pk_test_not_used_by_these_steps","secretKey":"'"$STRIPE_SECRET_KEY"'","transaction":"AUTHORIZECAPTURE"}}'
```

3. A cart with one chair (99.00), and the payment's init:

```bash
CART=$(json -X POST -d '{"product":"chair1","quantity":1}' "$S/api/v1/cart?$Q" | grep -o '"code":"[^"]*"' | head -1 | cut -d'"' -f4)
json -X POST -w '\nHTTP %{http_code}\n' "$S/api/v1/cart/$CART/payment/init?$Q" \
  -d '{"paymentModule":"stripe3","paymentType":"CREDITCARD","transactionType":"INIT","amount":"99.00"}'
```

4. The PaymentIntent that init created (the newest in your test account) has an id and a client secret; the answer above has neither:

```bash
curl -s -G -u "$STRIPE_SECRET_KEY:" https://api.stripe.com/v1/payment_intents -d limit=1 | grep -E '^      "(id|amount|currency|capture_method|status)"'
echo "client secrets in Stripe's answer: $(curl -s -G -u "$STRIPE_SECRET_KEY:" https://api.stripe.com/v1/payment_intents -d limit=1 | grep -c '"client_secret": "pi_')"
```

**Possible fix**

Store the init transaction with `transactionService.create`, which writes `details`, and return the PaymentIntent's id and client secret in the init answer.
