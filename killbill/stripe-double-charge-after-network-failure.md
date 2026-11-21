# Kill Bill: Double charge after a network failure (Stripe plugin 8.0.4)

I submitted these to the mailing list as recommended in your documentation, but they haven’t shown up after a few days so there may be something wrong. Also note these are not security vulnerabilities (I would have communicated those in private).

As part of Cosegan’s community service initiative, I examined some of your payment logic and noticed a few bugs I will share here. We formally verify critical software (using mathematical proofs). If the proofs pass, then we certify certain functionality, and you don’t have to worry that there are lurking bugs that AI will find later. However, if they don’t pass (like happened here), we know there is likely a bug in the code, even if we don’t yet know what that bug is. In your case, I drilled down to find out why the proofs didn’t pass and came up with this. If you have any questions please contact me at Eric@cosegan.com.

Formal verification is not new, but it’s normally very expensive so it’s only done on critical code like safety of flight or cryptographic libraries. I’m attempting to do it here for you as community service, since your project is open source and free to use.

---

**Environment**

- Kill Bill 0.24.22 (Docker image `killbill/killbill:0.24.22`, database `killbill/mariadb:0.24`), default settings
- killbill-stripe-plugin 8.0.4
- Stripe test mode

**We verified this on our sandbox only**: a test server in Docker, Stripe's test mode, and a network failure made by a proxy. We have not seen it in production. The lines named below are the same on the plugin's `master` (9.0.1-SNAPSHOT) and `maint-for-0-24-x` branches; we read those and did not run them.

**What we are seeing**

An invoice is charged twice at Stripe when the connection to Stripe breaks while Stripe is still working on `POST /v1/payment_intents`:

1. The JDK's `HttpURLConnection` sends the POST again by itself, with the same `Idempotency-Key`.
2. Stripe answers the copy with 409 `idempotency_key_in_use`, because the first request is still running. The first request then succeeds.
3. The plugin stores the transaction as `UNDEFINED` (`StripePluginProperties.mapExceptionToCallResult`, line 240).
4. Kill Bill calls `getPaymentInfo` inside the same payment call. `Charge.search` on `metadata['kbTransactionId']` returns nothing: in our runs the search was sent before Stripe had answered the first request. The plugin rewrites the row as `CANCELED`, "Payment didn't happen - Cancelled by Janitor" (`StripePaymentPluginApi.java`, lines 199 to 225).
5. Kill Bill records `PLUGIN_FAILURE`. Its invoice retry pays the invoice again 300 seconds later under a new transaction id, and Stripe charges again.

Result: two succeeded PaymentIntents for one invoice, while Kill Bill shows the invoice paid once. It happened in 12 runs of 12, with the break 0.1 to 0.6 seconds after the request.

From the run below (times are UTC; the proxy's log lines are shortened):

```
Kill Bill (GET /1.0/kb/accounts/{accountId}/payments): one payment, two transactions
    PURCHASE  301.00  PLUGIN_FAILURE  16:03:15  gatewayErrorCode idempotency_key_in_use
    PURCHASE  301.00  SUCCESS         16:08:16
Kill Bill (the account): "accountBalance":0.00

Stripe now (GET /payment_intents on the proxy):
    pi_…SJHE  amount 30100  status succeeded  received 30100  created 16:03:15Z
    pi_…QjUp  amount 30100  status succeeded  received 30100  created 16:08:16Z

The proxy's log:
    16:03:15.347  #3 POST /v1/payment_intents  came in
    16:03:15.650  #3 -> FAULT (drop-after): the plugin's connection is closed 0.303 s after the request came in; Stripe has not answered yet
    16:03:15.652  #4 POST /v1/payment_intents  came in  (the same request again, same key and body: the JDK's own resend)
    16:03:15.809  #4 -> HTTP 409 (error type invalid_request_error, code idempotency_key_in_use) after 0.157 s
    16:03:15.862  #5 GET /v1/charges/search?…  came in
    16:03:16.010  #5 -> HTTP 200 (search_result with 0 item(s)) after 0.147 s
    16:03:16.281  #3 -> Stripe's answer to this FIRST copy (the plugin never gets it): HTTP 200 (payment_intent pi_…SJHE, status succeeded, amount 30100), 0.935 s after it came in
    16:08:16.119  #6 POST /v1/payment_intents  came in
    16:08:17.103  #6 -> HTTP 200 (payment_intent pi_…QjUp, status succeeded, amount 30100) after 0.984 s
```

**What we are expecting**

One charge for one invoice. When the plugin does not know whether Stripe made a payment, the transaction should stay `UNKNOWN` until Stripe can say. An empty search made in the same second should not be recorded as "payment didn't happen".

**How to reproduce**

`fault-proxy.py`, a small proxy of our own (Python 3, standard library only; not included here, we can send it), stands between the plugin and Stripe. The plugin's `apiBase` points at it, it sends every request on to Stripe's test mode, and it can make one fault on one chosen request. It puts the real test key on each request, so Kill Bill's settings hold only a placeholder.

1. Start the database, the proxy and Kill Bill:

```bash
# Needs Docker, curl, fault-proxy.py (our proxy script; we can send it) in this directory, and a Stripe TEST secret key:
#   export STRIPE_SECRET_KEY=sk_test_...
[ -f stripe-plugin-8.0.4.jar ] || curl -sSfO https://repo1.maven.org/maven2/org/kill-bill/billing/plugin/java/stripe-plugin/8.0.4/stripe-plugin-8.0.4.jar
docker network create kb-net
docker run -d --name kb-db --network kb-net -e MYSQL_ROOT_PASSWORD=killbill -e MARIADB_ROOT_PASSWORD=killbill killbill/mariadb:0.24
until docker exec kb-db mariadb -h127.0.0.1 -uroot -pkillbill killbill -e 'select count(*) from stripe_responses' >/dev/null 2>&1; do sleep 2; done
docker run -d --name kb-proxy --network kb-net -p 127.0.0.1:8081:8081 -e STRIPE_SECRET_KEY \
  -v "$PWD/fault-proxy.py:/fault-proxy.py:ro" python:3.12-slim python3 -u /fault-proxy.py
docker run -d --name kb-server --network kb-net -p 127.0.0.1:8080:8080 \
  -e KILLBILL_DAO_URL=jdbc:mysql://kb-db:3306/killbill -e KILLBILL_DAO_USER=root -e KILLBILL_DAO_PASSWORD=killbill \
  -v "$PWD/stripe-plugin-8.0.4.jar:/var/lib/killbill/bundles/plugins/java/killbill-stripe/8.0.4/stripe-plugin-8.0.4.jar:ro" \
  killbill/killbill:0.24.22
until curl -sf http://127.0.0.1:8080/1.0/healthcheck >/dev/null; do sleep 3; done; sleep 5
```

2. Create the tenant and point the plugin at the proxy:

```bash
KB=http://127.0.0.1:8080
kb() { curl -s -u admin:password -H "X-Killbill-ApiKey: bob" -H "X-Killbill-ApiSecret: lazar" -H "X-Killbill-CreatedBy: demo" -H "Content-Type: application/json" -H "Accept: application/json" "$@"; }
new_id() { grep -i '^location:' | grep -Eo '[0-9a-f-]{36}' | tail -1; }   # the new object's id, from the Location header

# The tenant, and the plugin's settings: Stripe's address is the proxy, and the key is a placeholder (the proxy puts the real one on).
curl -s -u admin:password -H "X-Killbill-CreatedBy: demo" -H "Content-Type: application/json" \
  -X POST -d '{"apiKey":"bob","apiSecret":"lazar"}' $KB/1.0/kb/tenants
curl -s -u admin:password -H "X-Killbill-ApiKey: bob" -H "X-Killbill-ApiSecret: lazar" -H "X-Killbill-CreatedBy: demo" \
  -H "Content-Type: text/plain" -X POST $KB/1.0/kb/tenants/uploadPluginConfig/killbill-stripe \
  -d 'org.killbill.billing.plugin.stripe.apiKey=sk_test_placeholder
org.killbill.billing.plugin.stripe.publicKey=pk_test_placeholder
org.killbill.billing.plugin.stripe.apiBase=http://kb-proxy:8080'
sleep 5
```

3. Arm the fault and create an invoice:

```bash
# An account whose default payment method is a Stripe test card.
ACCOUNT=$(kb -D - -o /dev/null -X POST -d '{"name":"repro","currency":"USD"}' $KB/1.0/kb/accounts | new_id)
kb -X POST -d '{"pluginName":"killbill-stripe","pluginInfo":{"externalPaymentMethodId":"pm_card_visa"}}' "$KB/1.0/kb/accounts/$ACCOUNT/paymentMethods?isDefault=true"

# One fault: the next POST /v1/payment_intents is sent on to Stripe, and the plugin's connection is closed 0.3 s later.
curl -s -X POST -d '{"method":"POST","path":"/v1/payment_intents","fault":"drop-after","seconds":0.3}' http://127.0.0.1:8081/arm

# An invoice of 301 USD, which Kill Bill pays by itself.
kb -X POST -d '[{"accountId":"'$ACCOUNT'","amount":301,"currency":"USD","description":"repro"}]' "$KB/1.0/kb/invoices/charges/$ACCOUNT?autoCommit=true"
```

4. Wait five and a half minutes (Kill Bill retries the invoice 300 seconds after the failure), then look:

```bash
kb "$KB/1.0/kb/accounts/$ACCOUNT/payments"                  # Kill Bill: one payment, two PURCHASE transactions (PLUGIN_FAILURE, then SUCCESS)
kb "$KB/1.0/kb/accounts/$ACCOUNT?accountWithBalance=true"   # Kill Bill: "accountBalance":0
curl -s http://127.0.0.1:8081/payment_intents                # what Stripe holds now
curl -s http://127.0.0.1:8081/log                            # every request to Stripe, with times
```

**KPM diagnostic output**

Not attached here. We have the dump of this run, made with `kpm diagnostic --account-export=...` on that server right after the last command, and can send it.

**Possible fix**

Letting stripe-java repeat the request was enough in our test. With `setMaxNetworkRetries(2)` on the request options (`StripeConfigProperties.toRequestOptions`), stripe-java sends the request again after the 409 with the same `Idempotency-Key`, and Stripe returns the first request's answer: one PaymentIntent, the invoice paid at once (2 runs of 2, built on 8.0.4). Apart from that, an empty charge search made right after the request should not be taken to mean that the payment did not happen; Stripe's search also lags behind writes. We can send the patch we tested.

---

Posted to the project: https://github.com/killbill/killbill/issues/2340
