# Kill Bill: Void made at Stripe but recorded as failed (Stripe plugin 8.0.4)

I submitted these to the mailing list as recommended in your documentation, but they haven’t shown up after a few days so there may be something wrong. Also note these are not security vulnerabilities (I would have communicated those in private).

As part of Cosegan’s community service initiative, I examined some of your payment logic and noticed a few bugs I will share here. We formally verify critical software (using mathematical proofs). If the proofs pass, then we certify certain functionality, and you don’t have to worry that there are lurking bugs that AI will find later. However, if they don’t pass (like happened here), we know there is likely a bug in the code, even if we don’t yet know what that bug is. In your case, I drilled down to find out why the proofs didn’t pass and came up with this. If you have any questions please contact me at Eric@cosegan.com.

Formal verification is not new, but it’s normally very expensive so it’s only done on critical code like safety of flight or cryptographic libraries. I’m attempting to do it here for you as community service, since your project is open source and free to use.

---

**Environment**

- Kill Bill 0.24.22 (Docker image `killbill/killbill:0.24.22`, database `killbill/mariadb:0.24`), default settings
- killbill-stripe-plugin 8.0.4
- Stripe test mode

**We verified this on our sandbox only**: a test server in Docker, Stripe's test mode, and network faults made by a proxy. We have not seen it in production. The lines named below are the same on the plugin's `master` (9.0.1-SNAPSHOT) and `maint-for-0-24-x` branches; we read those and did not run them.

**What we are seeing**

When the answer to `POST /v1/payment_intents/{id}/cancel` does not reach the plugin, Stripe has canceled the PaymentIntent and Kill Bill ends with the void as `PLUGIN_FAILURE`:

- **Reconnection refused.** The plugin stores `CANCELED` at once (`StripePluginProperties.mapExceptionToCallResult`, line 217).
- **Connection broken while Stripe is working, or an answer slower than the read timeout (60 s).** The plugin stores `UNDEFINED`. `getPaymentInfo` then searches charges for `metadata['kbTransactionId']` with the **void's** transaction id (`StripePaymentPluginApi.java`, lines 199 to 202). No charge carries a void's id, so the search is always empty, and the row is rewritten as `CANCELED`, "Payment didn't happen - Cancelled by Janitor" (lines 212 to 225). With the slow answer this happens at the Janitor's check 5 minutes later.

Result: the hold is released at Stripe, while Kill Bill still reports the authorized amount, last success state `AUTH_SUCCESS`. After that:

- The same void asked again fails: Stripe answers "You cannot cancel this PaymentIntent because it has a status of canceled", and the plugin stores that as "Payment didn't happen" too. Kill Bill goes on reporting the authorization.
- A capture fails: "could not be captured because it has a status of canceled".

Same outcome in 10 runs: 5 with a refused reconnection, 2 with a broken connection, 3 with a slow answer. In one more run with a broken connection the fault had no effect: the JDK's own resend got Stripe's saved answer, and the void was recorded correctly.

From the run below, with the refused reconnection (times are UTC; the proxy's log lines are shortened):

```
$ kb -i -X DELETE -d '{}' "$KB/1.0/kb/payments/$PAYMENT"
HTTP/1.1 204

Kill Bill (GET /1.0/kb/payments/{paymentId}): "authAmount":307.00
    AUTHORIZE  307.00  SUCCESS
    VOID               PLUGIN_FAILURE  gatewayErrorMsg "IOException during API request to Stripe (http://kb-proxy:8080): Connection refused ..."

Stripe now (GET /payment_intents on the proxy):
    pi_…ChYW  amount 30700  status canceled  received 0  capturable 0  refunds: 1 (30700)
    (the released hold shows in Stripe's refund list, with the uncaptured amount)

The proxy's log:
    16:17:03.798  #5 POST /v1/payment_intents/pi_…ChYW/cancel  came in
    16:17:04.459  #5 -> HTTP 200 (payment_intent pi_…ChYW, status canceled, amount 30700) after 0.662 s; FAULT (refuse-after): Stripe's whole answer is read (7293 bytes); now closing the listening socket
    16:17:04.475  #5 -> the listening socket is closed (new connections are refused for 5 s); now closing the plugin's connection without an answer
    16:17:09.480  listening for the plugin on port 8080

The same void again:
HTTP/1.1 204        and a second VOID recorded PLUGIN_FAILURE, "You cannot cancel this PaymentIntent because it has a status of canceled. ..."
A capture:
HTTP/1.1 502        CAPTURE recorded PLUGIN_FAILURE, "This PaymentIntent could not be captured because it has a status of canceled. ..."
Kill Bill still reports "authAmount":307.00.
```

**What we are expecting**

The void is recorded as `SUCCESS` once Stripe has canceled the PaymentIntent.

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

3. Authorize, arm the fault, void, look, then ask for the void again and for a capture:

```bash
# An account whose default payment method is a Stripe test card, and an authorization of 307 USD.
ACCOUNT=$(kb -D - -o /dev/null -X POST -d '{"name":"repro","currency":"USD"}' $KB/1.0/kb/accounts | new_id)
kb -X POST -d '{"pluginName":"killbill-stripe","pluginInfo":{"externalPaymentMethodId":"pm_card_visa"}}' "$KB/1.0/kb/accounts/$ACCOUNT/paymentMethods?isDefault=true"
PAYMENT=$(kb -D - -o /dev/null -X POST -d '{"transactionType":"AUTHORIZE","amount":307,"currency":"USD"}' "$KB/1.0/kb/accounts/$ACCOUNT/payments" | new_id)

# One fault: the cancel request is sent on; when Stripe's whole answer is read the proxy stops listening,
# then closes the plugin's connection without an answer. It listens again 5 s later.
curl -s -X POST -d '{"method":"POST","path":"/v1/payment_intents/*/cancel","fault":"refuse-after","seconds":5}' http://127.0.0.1:8081/arm

# The void.
kb -i -X DELETE -d '{}' "$KB/1.0/kb/payments/$PAYMENT"
sleep 8
kb "$KB/1.0/kb/payments/$PAYMENT"                # Kill Bill: "authAmount":307; the VOID is PLUGIN_FAILURE
curl -s http://127.0.0.1:8081/payment_intents     # what Stripe holds now: canceled
curl -s http://127.0.0.1:8081/log                 # every request to Stripe, with times

# Afterwards, with no fault: the same void again, then a capture.
kb -i -X DELETE -d '{}' "$KB/1.0/kb/payments/$PAYMENT"
kb -i -X POST -d '{"amount":307,"currency":"USD"}' "$KB/1.0/kb/payments/$PAYMENT"
sleep 3
kb "$KB/1.0/kb/payments/$PAYMENT"
```

**KPM diagnostic output**

Not attached here. We have the dump of this run, made with `kpm diagnostic --account-export=...` on that server right after the last command, and can send it.

**Possible fix**

The charge search cannot find a void, so it should not be applied to void rows. The PaymentIntent's id is in the authorization's row: read the PaymentIntent, and take `canceled` to mean the void happened. For the refused reconnection the row also has to be stored as `UNDEFINED` and not `CANCELED`, or the status check never looks at it. In a test build on 8.0.4 with these changes, the void in the scenario above was `UNKNOWN` at first and `SUCCESS` at the Janitor's check 5 minutes later (2 runs of 2). The PaymentIntent has to be read even when the stored error is an HTTP 4xx. We can send the patches we tested.
