# Shopizer: Card charged but no order kept, and charged again when the shopper pays again

Found via [Cosegan](https://github.com/cosegan) formal/mathematical proof based verification and analysis.

**Effects:** the shopper's card is charged and Shopizer keeps no order; paying for the same cart again charges the card again.

**Environment**

- Shopizer 3.2.7: the Docker image `shopizerecomm/shopizer:3.2.7` (the newest tag, pushed 2025-03-22) with its demo data (store `DEFAULT`, CAD); the code cited below is the default branch `3.2.7` (head `6a4a0a6`)
- Payment module `stripe3` (Stripe PaymentIntents, stripe-java 19.5.0), Stripe test mode
- Docker on macOS (the image runs under amd64 emulation)

**We verified this on our sandbox only**: Shopizer's own image and demo data on one machine, and Stripe's test mode. We have not seen it in production. The image does not start as published (a separate issue), so step 1 starts the same jar with the jar's own `profiles/docker/database.properties`; nothing else is changed. A shop page cannot confirm a `stripe3` PaymentIntent through the 3.2.7 API either (a separate issue: init returns no client secret), so the steps confirm it with Stripe's test card, as Stripe.js would.

**What we are seeing**

Checkout fails after Stripe has captured the payment, and Shopizer keeps no order. The shopper gets HTTP 503. If they pay for the same cart again, Stripe captures again, and again there is no order.

1. `OrderServiceImpl.process` takes the payment first (`paymentService.processPayment`, line 147), then saves the order (line 170), then lowers stock, all in one database transaction.
2. The stock step looks the product up by the order line's own id: `productService.getById(orderProduct.getId())` (line 197). An order line's id comes from its own sequence and has nothing to do with the product bought.
3. `getById` is `repository.getOne(id)` (`SalesManagerEntityServiceImpl.java`, lines 38 to 40), a lazy reference, so the `p == null` check on the next line never fires. When no product has that id, `p.getAvailabilities()` throws `EntityNotFoundException` and the transaction rolls back: the order and its transaction row are gone.
4. Stripe's capture is not undone. Nothing left in Shopizer's database names the PaymentIntent (the row saved at `payment/init` has no details). `OrderApi` answers 503 (lines 384 to 391).
5. The cart is only marked as ordered after success (`OrderFacadeImpl.java`, lines 1331 to 1336), so the shopper can pay for it again, and each new payment is captured and rolled back the same way.

With the demo data (its products have ids 1, 50, 51 and 52), every checkout after the first on a freshly started server fails this way. The first order line gets id 1, `table1`'s id, so the table's stock is lowered although a chair was bought; the next ids, 2 to 49, are no product's. When an order line's id is another product's id, the order goes through and that product's stock is lowered instead (after a restart, order lines start at 50, `chair1`'s id). Pull request #1116 describes the lookup, the stock drift and the `EntityNotFoundException` on a second order; it does not say that the card has been charged by then.

The same `process` method runs for every payment module; we ran it with `stripe3` only.

Result: one cart, two captured payments of 99.00, no order.

From the run below (ids shortened):

```
# order 1, one chair
checkout: HTTP 200   {"id":1,"billing":{"postalCode":"H2X1Y4","countryCode":null,"firstName
table1 "quantity":999
chair1 "quantity":1000
# order 2, the same chair: the PaymentIntent confirmed, then the checkout
  "status": "requires_capture",
checkout: HTTP 503   {"timestamp":"2026-11-21T01:21:36.033+00:00","status":503,"error":"Ser
# the same cart, paid again
  "status": "requires_capture",
checkout: HTTP 503   {"timestamp":"2026-11-21T01:21:38.053+00:00","status":503,"error":"Ser
# afterwards at Stripe: the two PaymentIntents of the second cart
  "id": "pi_…OypjN2H",
  "amount_received": 9900,
  "status": "succeeded",
  "id": "pi_…1BepXMUe",
  "amount_received": 9900,
  "status": "succeeded",
# afterwards in Shopizer: one order, and the log
"recordsTotal":1
"id":1,"totals"
EntityNotFoundException: Unable to find com.salesmanager.core.model.catalog.product.Product with id 2
EntityNotFoundException: Unable to find com.salesmanager.core.model.catalog.product.Product with id 3
```

**What we are expecting**

No payment captured without an order. A failure after the payment should undo it (cancel or refund the PaymentIntent) or the capture should come after the order is saved. The stock step should look up the product each line was bought as.

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

3. A first order, one chair (99.00): Shopizer answers 200 and keeps order 1. Then the stock of the table and of the chair:

```bash
CART1=$(newcart chair1 1); pay $CART1 99.00
for sku in table1 chair1; do echo "$sku $(curl -s "$S/api/v2/product/$sku?$Q" | grep -o '"quantity":[0-9-]*' | head -1)"; done
```

4. A second order, the same chair: Stripe captures 99.00, then Shopizer answers 503. The shopper pays for the same cart again: captured again, 503 again:

```bash
CART2=$(newcart chair1 1); pay $CART2 99.00; PI2=$PI
pay $CART2 99.00; PI3=$PI
```

5. Afterwards: both PaymentIntents of the second cart are captured, and Shopizer has one order:

```bash
stripe_api "https://api.stripe.com/v1/payment_intents/$PI2"
stripe_api "https://api.stripe.com/v1/payment_intents/$PI3"
curl -s -H "Authorization: Bearer $ADMIN" "$S/api/v1/private/orders?$Q" | grep -o -E '"recordsTotal":[0-9]+|"id":[0-9]+,"totals"'
docker logs shopizer 2>&1 | grep -o 'EntityNotFoundException: Unable to find .* with id [0-9]*' | sort -u
```

**Possible fix**

Look the product up by the line's SKU, as #1116 does. Apart from that, take the money last or undo it: authorize at checkout, save the order, then capture; or cancel or refund the PaymentIntent when anything after the payment fails, and keep its id on a row that survives the rollback.
