# Shopizer: Shopper charged 100 times the price in a store priced in Japanese yen: ¥9,900 for a ¥99 order

Found via [Cosegan](https://github.com/cosegan) formal/mathematical proof based verification and analysis.

**Effects:** in a store priced in Japanese yen the shopper is charged 100 times the order's total, while Shopizer records the order at its price.

**Environment**

- Shopizer 3.2.7: the Docker image `shopizerecomm/shopizer:3.2.7` (the newest tag, pushed 2025-03-22) with its demo data (store `DEFAULT`, its currency set to JPY through the admin API); the code cited below is the default branch `3.2.7` (head `6a4a0a6`)
- Payment module `stripe3` (Stripe PaymentIntents, stripe-java 19.5.0), Stripe test mode
- Docker on macOS (the image runs under amd64 emulation)

**We verified this on our sandbox only**: Shopizer's own image and demo data on one machine, and Stripe's test mode. We have not seen it in production. The image does not start as published (a separate issue), so step 1 starts the same jar with the jar's own `profiles/docker/database.properties`; nothing else is changed. A shop page cannot confirm a `stripe3` PaymentIntent through the 3.2.7 API either (a separate issue: init returns no client secret), so the steps confirm it with Stripe's test card, as Stripe.js would.

**What we are seeing**

In a store whose currency has no minor unit (JPY; by the same code KRW, VND, CLP and Stripe's other zero-decimal currencies), the shopper is charged 100 times the order's total, while Shopizer records the order at its price.

1. Shopizer writes every Stripe amount with two decimals and then removes the dot, whatever the store's currency. Stripe reads JPY in whole yen, so 99 becomes 9900, which is ¥9,900.
2. At `payment/init`, `ProductPriceUtils.getAmount` parses the amount the page sends with Apache Commons Validator's `CurrencyValidator.getInstance().validate(amount, Locale.US)` (line 454), which gives two decimals ("99" becomes 99.00). `Stripe3Payment.initTransaction` then does `String.valueOf(amount).replace(".","")` and creates the PaymentIntent with the store's currency (lines 59, 60, 79 to 83): 9900 JPY.
3. At checkout the capture amount is the order total formatted by `getAdminFormatedAmount` (always two decimals, `ProductPriceUtils.java` lines 254 to 266) with the dot removed (`Stripe3Payment.java`, lines 313 to 340): 9900 again. The two agree, so Stripe captures.

Shopizer keeps the order at 99 JPY, and the checkout's answer says 99 too; when we read the database after an earlier run, the payment's transaction row also said 99. Nothing in Shopizer shows the ¥9,900.

From the run below (ids shortened):

```
store: HTTP 200
"currency":"JPY"
"total":"JP¥99.00"
# init with "99": the PaymentIntent Shopizer created, confirmed with the test card
  "id": "pi_…18HTxHbo",
  "amount": 9900,
  "currency": "jpy",
  "status": "requires_capture",
checkout: HTTP 200   {"id":1,"billing":{"postalCode":"H2X1Y4","countryCode":null,"firstName
# afterwards at Stripe: ¥9,900 received
  "amount_received": 9900,
  "currency": "jpy",
  "status": "succeeded",
# the order as Shopizer keeps it
"module":"total","value":99.0000
"currency":"JPY"
```

**What we are expecting**

¥99 charged for a ¥99 order: amounts sent to Stripe in the currency's own minor units (none for JPY).

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

3. The admin sets the store's currency to Japanese yen (the store's other fields as they are in the demo data):

```bash
json -X PUT -H "Authorization: Bearer $ADMIN" -w 'store: HTTP %{http_code}\n' "$S/api/v1/private/store/DEFAULT?$Q" \
  -d '{"id":1,"code":"DEFAULT","name":"Default store","defaultLanguage":"en","inBusinessSince":"2021-10-02","email":"admin@shopizer.com","phone":"888-888-8888","template":null,"useCache":false,"currencyFormatNational":true,"retailer":true,"dimension":"IN","weight":"LB","supportedLanguages":["en"],"address":{"address":"1234 Street address","city":"My city","postalCode":"H2H-2H2","stateProvince":"QC","country":"CA","active":true},"currency":"JPY"}'
curl -s "$S/api/v1/store/DEFAULT?lang=en" | grep -o '"currency":"[A-Z]*"' | head -1
CUR=JPY
```

4. One chair, ¥99, paid as a shop page would pay it:

```bash
CART=$(newcart chair1 1)
curl -s "$S/api/v1/cart/$CART/total?$Q" | grep -o '"total":"[^"]*"' | head -1
pay $CART 99
```

5. What Stripe took, and what Shopizer recorded:

```bash
stripe_api "https://api.stripe.com/v1/payment_intents/$PI"
ORDER=$(grep -o '^{"id":[0-9]*' checkout.json | cut -d: -f2)
curl -s -H "Authorization: Bearer $ADMIN" "$S/api/v1/private/orders/$ORDER?$Q" | grep -o -E '"module":"total","value":[0-9.]+|"currency":"[A-Z]+"' | head -2
```

**Possible fix**

Compute Stripe's amount from the `BigDecimal` in the currency's minor units (`amount.movePointRight(currency.getDefaultFractionDigits())`, checked against Stripe's list of zero-decimal currencies), and parse the shopper's amount in the store's currency rather than with a US-dollar validator.
