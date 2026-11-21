# Kill Bill: Failed void answered with 204 (Kill Bill 0.24.22)

Found via [Cosegan](https://github.com/cosegan) formal/mathematical proof based verification and analysis.

---

**Environment**

- Kill Bill 0.24.22 (Docker image `killbill/killbill:0.24.22`, database `killbill/mariadb:0.24`), default settings
- Seen with killbill-stripe-plugin 8.0.4 against Stripe test mode; the code in question is in Kill Bill itself and is not specific to a plugin

**We verified this on our sandbox only**: a test server in Docker, one payment plugin, and gateway failures made by a proxy in front of Stripe's test mode. We have not seen it in production. The two methods named below are identical at the tags `killbill-0.24.22` and `killbill-0.25.7` and on `master` (0.25.8-SNAPSHOT), five lines further down in the newer two; we read those and did not run them.

**What we are seeing**

`DELETE /1.0/kb/payments/{paymentId}` answers `204 No Content` even when the void is recorded as failed. `PaymentResource.voidPaymentInternal` calls `createVoidWithPaymentControl` and returns `Response.status(Status.NO_CONTENT)` without looking at the transaction (`PaymentResource.java`, lines 538 to 546 at tag `killbill-0.24.22`). Only an exception gives another answer; we saw 504 when the plugin call passed Kill Bill's 30 s limit.

The endpoint's own `@ApiResponses` (lines 474 to 481) declare 402 "Transaction declined by gateway", 502 "Failed to submit payment transaction" and 503 "Payment in unknown status, failed to receive gateway response". A capture and a refund do return these: they build their answer with `JaxRsResourceBase.createPaymentResponse` (lines 699 to 736), which maps the transaction's status.

With the same failure of the gateway call (an HTTP 500) on each:

- a void: Kill Bill records `PLUGIN_FAILURE` and answers **204**
- a capture: Kill Bill records `PLUGIN_FAILURE` and answers 502

Result: the caller is told the void worked. In the run below the authorization was still open at Stripe, so the hold stayed on the card. Every void that Kill Bill recorded as `PLUGIN_FAILURE` in our runs was answered 204: 11 of 11.

From the run below (times are UTC; the proxy's log lines are shortened):

```
$ kb -i -X DELETE -d '{}' "$KB/1.0/kb/payments/$PAYMENT"
HTTP/1.1 204

Kill Bill (GET /1.0/kb/payments/{paymentId}):
    AUTHORIZE  309.00  SUCCESS
    VOID               PLUGIN_FAILURE  gatewayErrorMsg "made by fault-proxy.py"

Stripe now (GET /payment_intents on the proxy):
    pi_…9bmW  amount 30900  status requires_capture  received 0  capturable 30900

The proxy's log:
    16:01:49.250  #5 POST /v1/payment_intents/pi_…9bmW/cancel  came in
    16:01:49.250  #5 -> FAULT (error500): NOT sent to Stripe; the plugin is answered HTTP 500

The same fault on a capture of the second authorization:
$ kb -i -X POST -d '{"amount":310,"currency":"USD"}' "$KB/1.0/kb/payments/$PAYMENT2"
HTTP/1.1 502
{"className":"org.killbill.billing.payment.api.PaymentApiException","code":null,"message":"Failed to submit payment transaction", ...}
```

**What we are expecting**

204 only when the void worked. For a void that failed, the status codes the endpoint declares (402, 502, 503), as a capture and a refund give.

**How to reproduce**

Any way of making a plugin's void fail will do. We used the Stripe plugin and `fault-proxy.py`, a small proxy of our own (Python 3, standard library only; not included here, we can send it), which stands between the plugin and Stripe. The plugin's `apiBase` points at it, it sends every request on to Stripe's test mode, and it can make one fault on one chosen request. It puts the real test key on each request, so Kill Bill's settings hold only a placeholder.

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

3. Authorize, make the void fail, and compare with a capture that fails the same way:

```bash
# An account whose default payment method is a Stripe test card, and an authorization of 309 USD.
ACCOUNT=$(kb -D - -o /dev/null -X POST -d '{"name":"repro","currency":"USD"}' $KB/1.0/kb/accounts | new_id)
kb -X POST -d '{"pluginName":"killbill-stripe","pluginInfo":{"externalPaymentMethodId":"pm_card_visa"}}' "$KB/1.0/kb/accounts/$ACCOUNT/paymentMethods?isDefault=true"
PAYMENT=$(kb -D - -o /dev/null -X POST -d '{"transactionType":"AUTHORIZE","amount":309,"currency":"USD"}' "$KB/1.0/kb/accounts/$ACCOUNT/payments" | new_id)

# One fault: the cancel request is not sent to Stripe; the proxy answers the plugin HTTP 500.
curl -s -X POST -d '{"method":"POST","path":"/v1/payment_intents/*/cancel","fault":"error500"}' http://127.0.0.1:8081/arm

# The void. Kill Bill answers HTTP 204.
kb -i -X DELETE -d '{}' "$KB/1.0/kb/payments/$PAYMENT"
sleep 3
kb "$KB/1.0/kb/payments/$PAYMENT"                # Kill Bill: the VOID is PLUGIN_FAILURE
curl -s http://127.0.0.1:8081/payment_intents     # what Stripe holds now: still requires_capture
curl -s http://127.0.0.1:8081/log                 # the cancel request never reached Stripe

# For comparison, the same fault on a capture of a second authorization. Kill Bill answers HTTP 502.
PAYMENT2=$(kb -D - -o /dev/null -X POST -d '{"transactionType":"AUTHORIZE","amount":310,"currency":"USD"}' "$KB/1.0/kb/accounts/$ACCOUNT/payments" | new_id)
curl -s -X POST -d '{"method":"POST","path":"/v1/payment_intents/*/capture","fault":"error500"}' http://127.0.0.1:8081/arm
kb -i -X POST -d '{"amount":310,"currency":"USD"}' "$KB/1.0/kb/payments/$PAYMENT2"
```

**KPM diagnostic output**

Not attached here. We have the dump of this run, made with `kpm diagnostic --account-export=...` on that server right after the last command, and can send it.

**Possible fix**

Keep 204 for a void that is `SUCCESS` or `PENDING`, and for the other statuses answer as `createPaymentResponse` does. We ran that change on 0.24.22: the failed void above is then answered 502, "Failed to submit payment transaction", and a void that works is still answered 204. `voidPaymentByExternalKey` uses the same method (line 514); we read that and did not run it. We can send the patch we tested.
