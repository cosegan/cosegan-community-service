# Kill Bill: Refund made but recorded as failed (Stripe plugin 8.0.4)

I submitted these to the mailing list as recommended in your documentation, but they haven’t shown up after a few days so there may be something wrong. Also note these are not security vulnerabilities (I would have communicated those in private).

As part of Cosegan’s community service initiative, I examined some of your payment logic and noticed a few bugs I will share here. We formally verify critical software (using mathematical proofs). If the proofs pass, then we certify certain functionality, and you don’t have to worry that there are lurking bugs that AI will find later. However, if they don’t pass (like happened here), we know there is likely a bug in the code, even if we don’t yet know what that bug is. In your case, I drilled down to find out why the proofs didn’t pass and came up with this. If you have any questions please contact me at Eric@cosegan.com.

Formal verification is not new, but it’s normally very expensive so it’s only done on critical code like safety of flight or cryptographic libraries. I’m attempting to do it here for you as community service, since your project is open source and free to use.

---

**Environment**

- Kill Bill 0.24.22 (Docker image `killbill/killbill:0.24.22`, database `killbill/mariadb:0.24`), default settings
- killbill-stripe-plugin 8.0.4
- Stripe test mode

**We verified this on our sandbox only**: a test server in Docker, Stripe's test mode, and a failed read made by a proxy (an HTTP 500). We have not seen it in production. The lines named below are the same on the plugin's `master` (9.0.1-SNAPSHOT) and `maint-for-0-24-x` branches; we read those and did not run them.

**What we are seeing**

`refundPayment` calls `Refund.create` and then `PaymentIntent.retrieve` in one step (`StripePaymentPluginApi.java`, lines 763 to 765). If the retrieve fails after the refund succeeded:

1. Stripe has made the refund.
2. The plugin stores the refund's row as `UNDEFINED`.
3. `getPaymentInfo` searches charges for `metadata['kbTransactionId']` with the **refund's** transaction id (lines 199 to 202). No charge carries a refund's id, so the search is always empty, and the row is rewritten as `CANCELED`, "Payment didn't happen - Cancelled by Janitor" (lines 212 to 225).
4. Kill Bill answers the caller HTTP 502 and records the refund as `PLUGIN_FAILURE`, refunded amount 0.
5. The same refund, asked again, succeeds.

Result: Stripe has refunded twice (2 × 25.00 on a 306.00 charge in the run below), and Kill Bill shows one refund. Same outcome in 4 runs of 4.

From the run below (times are UTC; the proxy's log lines are shortened):

```
$ kb -i -X POST -d '{"amount":25}' "$KB/1.0/kb/payments/$PAYMENT/refunds"
HTTP/1.1 502        "Failed to submit payment transaction"

Kill Bill (GET /1.0/kb/payments/{paymentId}): "refundedAmount":0
    PURCHASE  306.00  SUCCESS
    REFUND     25.00  PLUGIN_FAILURE

The same refund again:
HTTP/1.1 201

Kill Bill: "refundedAmount":25.00
    PURCHASE  306.00  SUCCESS
    REFUND     25.00  PLUGIN_FAILURE
    REFUND     25.00  SUCCESS

Stripe now (GET /payment_intents on the proxy):
    pi_…ZPnq  amount 30600  status succeeded  received 30600  refunds: 2 (2500, 2500)

The proxy's log:
    16:16:22.476  #4 POST /v1/refunds  came in
    16:16:23.455  #4 -> HTTP 200 (refund re_…mXbA, status succeeded, amount 2500) after 0.979 s
    16:16:23.459  #5 GET /v1/payment_intents/pi_…ZPnq  came in
    16:16:23.459  #5 -> FAULT (error500): NOT sent to Stripe; the plugin is answered HTTP 500
    16:16:23.531  #6 GET /v1/charges/search?…  came in
    16:16:23.682  #6 -> HTTP 200 (search_result with 0 item(s)) after 0.151 s
    16:16:26.937  #7 POST /v1/refunds  came in
    16:16:27.747  #7 -> HTTP 200 (refund re_…P0lJ, status succeeded, amount 2500) after 0.810 s
```

**What we are expecting**

The refund is recorded as `SUCCESS` once Stripe has made it, so that Kill Bill's refunded amount matches Stripe's and nobody is led to ask for it again.

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

3. Make a purchase, arm the fault, refund, and refund again:

```bash
# An account whose default payment method is a Stripe test card, and a purchase of 306 USD.
ACCOUNT=$(kb -D - -o /dev/null -X POST -d '{"name":"repro","currency":"USD"}' $KB/1.0/kb/accounts | new_id)
kb -X POST -d '{"pluginName":"killbill-stripe","pluginInfo":{"externalPaymentMethodId":"pm_card_visa"}}' "$KB/1.0/kb/accounts/$ACCOUNT/paymentMethods?isDefault=true"
PAYMENT=$(kb -D - -o /dev/null -X POST -d '{"transactionType":"PURCHASE","amount":306,"currency":"USD"}' "$KB/1.0/kb/accounts/$ACCOUNT/payments" | new_id)

# One fault: the plugin's next read of the PaymentIntent is answered HTTP 500 by the proxy.
curl -s -X POST -d '{"method":"GET","path":"/v1/payment_intents/*","fault":"error500"}' http://127.0.0.1:8081/arm

# A refund of 25 USD. Kill Bill answers HTTP 502.
kb -i -X POST -d '{"amount":25}' "$KB/1.0/kb/payments/$PAYMENT/refunds"
sleep 3
kb "$KB/1.0/kb/payments/$PAYMENT"      # "refundedAmount":0; the REFUND is PLUGIN_FAILURE

# The same refund again, with no fault. Kill Bill answers HTTP 201.
kb -i -X POST -d '{"amount":25}' "$KB/1.0/kb/payments/$PAYMENT/refunds"
sleep 3
kb "$KB/1.0/kb/payments/$PAYMENT"      # "refundedAmount":25
curl -s http://127.0.0.1:8081/payment_intents    # what Stripe holds now: two refunds of 2500
curl -s http://127.0.0.1:8081/log                # every request to Stripe, with times
```

**KPM diagnostic output**

Not attached here. We have the dump of this run, made with `kpm diagnostic --account-export=...` on that server right after the last command, and can send it.

**Possible fix**

Once `Refund.create` has returned, the refund happened: record it from the `Refund` object. In our test build on 8.0.4 we created the refund with `expand[]=payment_intent` and stored the PaymentIntent that comes back inside the `Refund`, so nothing is read afterwards: the refund was `SUCCESS` at once, with one refund at Stripe (2 runs of 2). The charge search should also not be applied to refund rows, since it cannot find them. A refund whose own answer is lost then stays `UNKNOWN`; to settle that one the plugin needs something to find the refund by, such as the transaction id in the refund's metadata, which we did not build. We can send the patch we tested.
