# Kill Bill: Customer charged but payment recorded as unknown (Braintree plugin 1.0.3)

I submitted these to the mailing list as recommended in your documentation, but they haven’t shown up after a few days so there may be something wrong. Also note these are not security vulnerabilities (I would have communicated those in private).

As part of Cosegan’s community service initiative, I examined some of your payment logic and noticed a few bugs I will share here. We formally verify critical software (using mathematical proofs). If the proofs pass, then we certify certain functionality, and you don’t have to worry that there are lurking bugs that AI will find later. However, if they don’t pass (like happened here), we know there is likely a bug in the code, even if we don’t yet know what that bug is. In your case, I drilled down to find out why the proofs didn’t pass and came up with this. If you have any questions please contact me at Eric@cosegan.com.

Formal verification is not new, but it’s normally very expensive so it’s only done on critical code like safety of flight or cryptographic libraries. I’m attempting to do it here for you as community service, since your project is open source and free to use.

---

**Environment**

- Kill Bill 0.24.22 (Docker image `killbill/killbill:0.24.22`, database `killbill/mariadb:0.24`), default settings
- killbill-braintree (braintree-plugin) 1.0.3
- Braintree sandbox

**We verified this on our sandbox only**: a test server in Docker and Braintree's sandbox. The dropped connection was made by a TCP proxy, with Java held to TLS 1.2 so that the proxy could tell where an answer starts. We have not seen it in production. The lines named below are the same on the plugin's `master` (2.0.1-SNAPSHOT) and `maint-for-0-24-x` branches; we read those and did not run them.

**What we are seeing**

`BraintreePluginProperties.toAdditionalDataMap` uses the result's transaction without a null check (lines 66 to 67). A Braintree validation error carries no transaction, so storing the answer throws a `NullPointerException`. `executeInitialTransaction` catches only `BraintreeException` and `SQLException` (`BraintreePaymentPluginApi.java`, lines 524 to 536), so the exception reaches Kill Bill:

- the API call gets HTTP 500, "Internal payment error : java.lang.NullPointerException";
- the payment is recorded `PURCHASE_ERRORED`, its transaction `UNKNOWN`;
- the plugin stores no row, so the Janitor cannot resolve it: `getPaymentInfo` returns at once for a payment it has no row for, and Kill Bill logs "Unable to repair ... UNKNOWN".

A purchase with an amount Braintree refuses as invalid is enough to see it, and no money moves (step 3 below).

The customer is charged when the same thing happens on a sale that Braintree has made. If the connection drops just as Braintree starts answering a sale, the JDK's `HttpURLConnection` sends the POST again by itself. Braintree has made the sale from the first copy and refuses the second, because the one-time nonce is already used. That refusal has no transaction, so the plugin crashes as above.

Result: Braintree holds a sale submitted for settlement, with the Kill Bill transaction id as its order id, while Kill Bill shows that transaction `UNKNOWN` with no Braintree reference. Same outcome in 9 runs of 9. When the payment is an invoice's (3 of those runs), Kill Bill does not try again: the attempt ends `ABORTED`, no retry is queued, and the invoice stays unpaid with its whole amount owed while Braintree holds the sale.

From the run below (times are UTC):

```
$ kb -i -X POST -d '{"transactionType":"PURCHASE","amount":13.123,"currency":"USD"}' "$KB/1.0/kb/accounts/$ACCOUNT/payments"
HTTP/1.1 500
{"className":"org.killbill.billing.payment.api.PaymentApiException","code":7019,"message":"Internal payment error : java.lang.NullPointerException","causeClassName":"java.lang.NullPointerException", ...}

The purchase of 32.00 with the cut: the same HTTP 500. Kill Bill's log, for each of the two:
    Caused by: java.lang.NullPointerException: null
        at org.killbill.billing.plugin.braintree.core.BraintreePluginProperties.toAdditionalDataMap(BraintreePluginProperties.java:67)
        at org.killbill.billing.plugin.braintree.dao.BraintreeDao.addResponse(BraintreeDao.java:136)
        at org.killbill.billing.plugin.braintree.api.BraintreePaymentPluginApi.executeInitialTransaction(BraintreePaymentPluginApi.java:532)

Five and a half minutes later, Kill Bill (GET /1.0/kb/accounts/{accountId}/payments): two payments
    PURCHASE  13.12  UNKNOWN
    PURCHASE  32.00  UNKNOWN
Kill Bill's log at the Janitor's first check of each:
    Unable to repair paymentId='8404a546…', paymentTransactionId='faa42717…', currentTransactionStatus='UNKNOWN', newTransactionStatus='UNKNOWN'
    Unable to repair paymentId='d1189545…', paymentTransactionId='c7a3d5f6…', currentTransactionStatus='UNKNOWN', newTransactionStatus='UNKNOWN'

Braintree's sandbox, searched by order id (the two Kill Bill transaction ids):
    faa42717…  (the 13.123 purchase)  no transaction
    c7a3d5f6…  (the 32.00 purchase)   one sale, 32.00 USD, submitted_for_settlement, created 16:38:02Z

The proxy's log for the 32.00 purchase (TLS records by type and length; it cannot read them):
    16:38:02.555  conn 3  plugin->Braintree  application-data record, 389 bytes  <- a request starts (request 2 on this connection)
    16:38:02.555  conn 3  plugin->Braintree  application-data record, 366 bytes
    16:38:03.522  conn 3  Braintree->plugin  application-data record, 1393 bytes  <- an answer starts (answer 2 on this connection; answer 2 since the cut was armed)
    16:38:03.522  conn 3  CUT: this first record of the answer is NOT passed on; the plugin's connection is closed now
    16:38:03.538  conn 4  opened by 172.21.0.4; connected on to 159.242.242.128
    16:38:03.617  conn 4  plugin->Braintree  application-data record, 389 bytes  <- a request starts (request 1 on this connection)
    16:38:03.618  conn 4  plugin->Braintree  application-data record, 366 bytes
    16:38:03.865  conn 4  Braintree->plugin  application-data record, 1393 bytes  <- an answer starts (answer 1 on this connection)
    16:38:03.865  conn 4  Braintree->plugin  application-data record, 773 bytes
```

**What we are expecting**

A result without a transaction is stored as a failed payment with Braintree's message, with no exception. When Braintree holds the sale, Kill Bill ends with the payment as `SUCCESS`, or at least with a row that the Janitor can settle.

**How to reproduce**

The first part needs no proxy. For the second, `tls-cut-proxy.py`, a small proxy of our own (Python 3, standard library only; not included here, we can send it), stands where Braintree's sandbox is for Kill Bill. It cannot read the traffic; it forwards bytes and can close the plugin's connection at the moment a chosen answer starts.

1. Start the database, the proxy and Kill Bill:

```bash
# Needs Docker, curl, tls-cut-proxy.py (our proxy script; we can send it) in this directory, and Braintree SANDBOX keys:
#   export BRAINTREE_MERCHANT_ID=... BRAINTREE_PUBLIC_KEY=... BRAINTREE_PRIVATE_KEY=...
[ -f braintree-plugin-1.0.3.jar ] || curl -sSfO https://repo1.maven.org/maven2/org/kill-bill/billing/plugin/java/braintree-plugin/1.0.3/braintree-plugin-1.0.3.jar
docker network create kb-net
docker run -d --name kb-db --network kb-net -e MYSQL_ROOT_PASSWORD=killbill -e MARIADB_ROOT_PASSWORD=killbill killbill/mariadb:0.24
until docker exec kb-db mariadb -h127.0.0.1 -uroot -pkillbill killbill -e 'select count(*) from payments' >/dev/null 2>&1; do sleep 2; done
# A TCP proxy that stands where Braintree's sandbox is for Kill Bill. It cannot read the traffic (TLS runs end to end).
docker run -d --name kb-proxy --network kb-net -p 127.0.0.1:8081:8081 \
  -v "$PWD/tls-cut-proxy.py:/tls-cut-proxy.py:ro" python:3.12-slim python3 -u /tls-cut-proxy.py
sleep 2
PROXY_IP=$(docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' kb-proxy)
# Java is held to TLS 1.2 so that the proxy can tell where an answer starts. The plugin takes its keys from the environment.
docker run -d --name kb-server --network kb-net -p 127.0.0.1:8080:8080 \
  --add-host "api.sandbox.braintreegateway.com:$PROXY_IP" -e JAVA_OPTS="-Djdk.tls.client.protocols=TLSv1.2" \
  -e BRAINTREE_ENVIRONMENT=sandbox -e BRAINTREE_MERCHANT_ID -e BRAINTREE_PUBLIC_KEY -e BRAINTREE_PRIVATE_KEY \
  -e KILLBILL_DAO_URL=jdbc:mysql://kb-db:3306/killbill -e KILLBILL_DAO_USER=root -e KILLBILL_DAO_PASSWORD=killbill \
  -v "$PWD/braintree-plugin-1.0.3.jar:/var/lib/killbill/bundles/plugins/java/killbill-braintree/1.0.3/braintree-plugin-1.0.3.jar:ro" \
  killbill/killbill:0.24.22
until curl -sf http://127.0.0.1:8080/1.0/healthcheck >/dev/null; do sleep 3; done; sleep 5
```

2. Put the id of a customer in your Braintree sandbox into `BT_CUSTOMER`, then create the tenant, an account and a payment method:

```bash
KB=http://127.0.0.1:8080
kb() { curl -s -u admin:password -H "X-Killbill-ApiKey: bob" -H "X-Killbill-ApiSecret: lazar" -H "X-Killbill-CreatedBy: demo" -H "Content-Type: application/json" -H "Accept: application/json" "$@"; }
new_id() { grep -i '^location:' | grep -Eo '[0-9a-f-]{36}' | tail -1; }   # the new object's id, from the Location header

# The tenant, an account, and a payment method made from Braintree's test nonce for the sandbox customer $BT_CUSTOMER.
curl -s -u admin:password -H "X-Killbill-CreatedBy: demo" -H "Content-Type: application/json" \
  -X POST -d '{"apiKey":"bob","apiSecret":"lazar"}' $KB/1.0/kb/tenants
ACCOUNT=$(kb -D - -o /dev/null -X POST -d '{"name":"repro","currency":"USD"}' $KB/1.0/kb/accounts | new_id)
kb -X POST -d '{"pluginName":"killbill-braintree","pluginInfo":{}}' "$KB/1.0/kb/accounts/$ACCOUNT/paymentMethods?isDefault=true&pluginProperty=bt_nonce=fake-valid-nonce&pluginProperty=bt_customer_id=$BT_CUSTOMER"
```

3. The crash alone, with no fault and no money moved:

```bash
# A purchase with an amount Braintree refuses as invalid (three decimals). No fault is made and no money moves.
kb -i -X POST -d '{"transactionType":"PURCHASE","amount":13.123,"currency":"USD"}' "$KB/1.0/kb/accounts/$ACCOUNT/payments"
sleep 3
kb "$KB/1.0/kb/accounts/$ACCOUNT/payments"     # Kill Bill: the PURCHASE is UNKNOWN
docker logs kb-server 2>&1 | grep -c 'BraintreePluginProperties.java:67'
```

4. The case where the customer is charged:

```bash
# One cut: the plugin's connection is closed when Braintree's 2nd answer from now starts.
# (The 1st answer is to the plugin's request for a nonce; the 2nd is to the sale.)
curl -s -X POST -d '{"answer":2}' http://127.0.0.1:8081/arm

# A purchase of 32.00 USD.
kb -i -X POST -d '{"transactionType":"PURCHASE","amount":32.00,"currency":"USD"}' "$KB/1.0/kb/accounts/$ACCOUNT/payments"
```

5. Wait five and a half minutes (the Janitor's first check), then look:

```bash
kb "$KB/1.0/kb/accounts/$ACCOUNT/payments"     # Kill Bill: both PURCHASE transactions are UNKNOWN
docker logs kb-server 2>&1 | grep -o 'Unable to repair[^"]*' | tail -2
curl -s http://127.0.0.1:8081/log | grep -v handshake | tail -25   # the cut, and Java's second copy of the sale on a new connection
```

In the Braintree sandbox, the transaction search by order id (the Kill Bill transaction id of the 32.00 purchase) shows one sale of 32.00 USD, submitted for settlement.

**KPM diagnostic output**

Not attached here. We have the dump of this run, made with `kpm diagnostic --account-export=...` on that server right after the last command, and can send it.

**Possible fix**

Two changes, and never the first alone. We built them on 1.0.3 and ran the scenarios again.

- Handle a result without a transaction in `toAdditionalDataMap` and in `BraintreeDao.addResponse` (line 158), so that a validation error becomes a stored failure. The `braintree_id` column does not take null. With this the invalid amount gives HTTP 502 and a row with Braintree's message, "Amount is an invalid format."
- For "nonce already used" (validation error 91564), look the sale up at Braintree by order id before recording a failure: the first copy of the request may have gone through. In 2 runs of 2 the search found the sale, the payment was `SUCCESS`, and there was one sale.

The first change without the second makes it worse: the refusal becomes an ordinary failure, Kill Bill retries the invoice 5 minutes later, and Braintree holds two sales for one invoice (1 run). The crash at least stops that retry. We can send the patch we tested.
