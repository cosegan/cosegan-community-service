# Shopizer: Orders of 1,000.00 or more cannot be paid with stripe3, and the amount stays held on the shopper's card

Found via [Cosegan](https://github.com/cosegan) formal/mathematical proof based verification and analysis.

**Effects:** an order of 1,000.00 or more cannot be paid with stripe3, and its amount stays held on the shopper's card until Stripe releases it.

**Environment**

- Shopizer 3.2.7: the Docker image `shopizerecomm/shopizer:3.2.7` (the newest tag, pushed 2025-03-22) with its demo data (store `DEFAULT`, CAD); the code cited below is the default branch `3.2.7` (head `6a4a0a6`)
- Payment module `stripe3` (Stripe PaymentIntents, stripe-java 19.5.0), Stripe test mode
- Docker on macOS (the image runs under amd64 emulation)

**We verified this on our sandbox only**: Shopizer's own image and demo data on one machine, and Stripe's test mode. We have not seen it in production. The image does not start as published (a separate issue), so step 1 starts the same jar with the jar's own `profiles/docker/database.properties`; nothing else is changed. A shop page cannot confirm a `stripe3` PaymentIntent through the 3.2.7 API either (a separate issue: init returns no client secret), so the steps confirm it with Stripe's test card, as Stripe.js would.

**What we are seeing**

Any order of 1,000.00 or more fails at checkout after the shopper's card has been authorized. Shopizer answers 503 and keeps no order, and the PaymentIntent stays `requires_capture`: the amount is held on the card until Stripe lets the authorization lapse.

1. At checkout `Stripe3Payment.authorizeAndCapture` makes Stripe's amount from the order total formatted for display, with the dot removed (lines 313 to 340):

   ```
   String amnt = productPriceUtils.getAdminFormatedAmount(store, amount);
   …
   strAmount = strAmount.replace(".","");
   …
   .setAmountToCapture(Long.parseLong(strAmount))
   ```

2. `getAdminFormatedAmount` uses `NumberFormat.getInstance(Locale.US)` with two decimals (`ProductPriceUtils.java`, lines 254 to 266), which groups thousands: 1194.00 becomes "1,194.00", then "1,19400", and `Long.parseLong` throws `NumberFormatException` before the capture is sent.
3. The PaymentIntent was created at `payment/init` with the right amount (119400), because that path uses `String.valueOf(BigDecimal)`, which has no comma (lines 59 and 60), and the shopper has already confirmed it. Nothing cancels it.

The same formatting is in `Stripe3Payment.capture` and `Stripe3Payment.refund`, and in the older `StripePayment` module, which sends that string to Stripe as the charge's amount (we read these and did not run them).

From the run below (ids shortened):

```
"total":"$1,194.00"
  "id": "pi_…1ajJJnKX",
  "amount": 119400,
  "amount_capturable": 119400,
  "status": "requires_capture",
checkout: HTTP 503   {"timestamp":"2026-11-21T01:25:03.212+00:00","status":503,"error":"Ser
# afterwards: the PaymentIntent, Shopizer's orders, Shopizer's log
  "amount_capturable": 119400,
  "amount_received": 0,
  "status": "requires_capture",
"recordsTotal":0
NumberFormatException: For input string: "1,19400"
# the last command, which releases the hold
  "status": "canceled",
```

**What we are expecting**

An order of 1,000.00 is captured like any other. When checkout fails after the shopper has confirmed the payment, the PaymentIntent is cancelled, so nothing stays held on the card.

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

2. Turn on `stripe3` with your test key, sign a shopper in, and define three helpers:

```bash
S=http://127.0.0.1:8080; Q='store=DEFAULT&lang=en'; CUR=CAD
json() { curl -s -H 'Content-Type: application/json' "$@"; }
token() { grep -o '"token":"[^"]*"' | cut -d'"' -f4; }
ADMIN=$(json -X POST -d '{"username":"admin@shopizer.com","password":"password"}' "$S/api/v1/private/login" | token)
# The module's form also requires a publishable key; these steps do not use it.
json -X POST -H "Authorization: Bearer $ADMIN" -w 'module stripe3: HTTP %{http_code}\n' "$S/api/v1/private/modules/payment?$Q" \
  -d '{"code":"stripe3","active":true,"defaultSelected":true,"integrationKeys":{"publishableKey":"pk_test_not_used_by_these_steps","secretKey":"'"$STRIPE_SECRET_KEY"'","transaction":"AUTHORIZECAPTURE"}}'
json -X POST -o /dev/null -w 'register: HTTP %{http_code}\n' "$S/api/v1/customer/register?$Q" \
  -d '{"emailAddress":"shopper@example.com","userName":"shopper@example.com","password":"Password1!","repeatPassword":"Password1!","firstName":"Test","lastName":"Shopper","gender":"M","language":"en","billing":{"firstName":"Test","lastName":"Shopper","address":"1 Test Street","city":"Montreal","postalCode":"H2X1Y4","country":"CA","zone":"QC","stateProvince":"QC","phone":"5145550101"}}'
SHOPPER=$(json -X POST -d '{"username":"shopper@example.com","password":"Password1!"}' "$S/api/v1/customer/login?$Q" | token)

# newcart <sku> <quantity>: a new cart with one product; prints the cart's code.
newcart() { json -X POST -d '{"product":"'$1'","quantity":'$2'}' "$S/api/v1/cart?$Q" | grep -o '"code":"[^"]*"' | head -1 | cut -d'"' -f4; }
# stripe_api <curl arguments>: a Stripe call with your key; prints the PaymentIntent's main fields.
stripe_api() { curl -s -u "$STRIPE_SECRET_KEY:" -H 'Stripe-Version: 2020-03-02' "$@" | grep -E '^  "(id|amount|amount_capturable|amount_received|currency|status)"'; }
# pay <cart> <amount>: a stripe3 payment as a shop page makes it. Init makes Shopizer create the PaymentIntent; its answer
# has no PaymentIntent id (a separate issue), so the id is read from Stripe: your account's newest PaymentIntent.
# Stripe's test card confirms it, as Stripe.js would. Then the shopper's checkout, where Shopizer captures.
pay() {
  json -X POST -o /dev/null -w 'init: HTTP %{http_code}\n' "$S/api/v1/cart/$1/payment/init?$Q" \
    -d '{"paymentModule":"stripe3","paymentType":"CREDITCARD","transactionType":"INIT","amount":"'$2'"}'
  PI=$(curl -s -G -u "$STRIPE_SECRET_KEY:" https://api.stripe.com/v1/payment_intents -d limit=1 | grep -o '"id": "pi_[^"]*"' | head -1 | cut -d'"' -f4)
  stripe_api -X POST "https://api.stripe.com/v1/payment_intents/$PI/confirm" -d payment_method=pm_card_visa
  json -X POST -H "Authorization: Bearer $SHOPPER" -o checkout.json -w 'checkout: HTTP %{http_code}   ' "$S/api/v1/auth/cart/$1/checkout?$Q" \
    -d '{"currency":"'$CUR'","payment":{"paymentType":"CREDITCARD","transactionType":"AUTHORIZECAPTURE","paymentModule":"stripe3","paymentToken":"'$PI'","amount":"'$2'"}}'
  cut -c1-70 checkout.json; echo
}
```

3. Six tables (199.00 each, 1,194.00), paid as a shop page would pay them:

```bash
CART=$(newcart table1 6)
curl -s "$S/api/v1/cart/$CART/total?$Q" | grep -o '"total":"[^"]*"' | head -1
pay $CART 1194.00
```

4. Afterwards: the PaymentIntent is still `requires_capture` with 119400 capturable, Shopizer has no order, and its log has the cause. The last command cancels the PaymentIntent, which releases the hold on the test card:

```bash
stripe_api "https://api.stripe.com/v1/payment_intents/$PI"
curl -s -H "Authorization: Bearer $ADMIN" "$S/api/v1/private/orders?$Q" | grep -o '"recordsTotal":[0-9]*'
docker logs shopizer 2>&1 | grep -o 'NumberFormatException: For input string: "[^"]*"' | sort -u
stripe_api -X POST "https://api.stripe.com/v1/payment_intents/$PI/cancel"
```

**Possible fix**

Compute Stripe's amount from the `BigDecimal` in the currency's minor units, for example `amount.movePointRight(currency.getDefaultFractionDigits()).longValueExact()`, never from a display string; and cancel the PaymentIntent when checkout fails after it was confirmed.
