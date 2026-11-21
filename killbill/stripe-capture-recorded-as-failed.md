# Kill Bill: Capture made at Stripe but recorded as failed (Stripe plugin 8.0.4)

I submitted these to the mailing list as recommended in your documentation, but they haven’t shown up after a few days so there may be something wrong. Also note these are not security vulnerabilities (I would have communicated those in private).

As part of Cosegan’s community service initiative, I examined some of your payment logic and noticed a few bugs I will share here. We formally verify critical software (using mathematical proofs). If the proofs pass, then we certify certain functionality, and you don’t have to worry that there are lurking bugs that AI will find later. However, if they don’t pass (like happened here), we know there is likely a bug in the code, even if we don’t yet know what that bug is. In your case, I drilled down to find out why the proofs didn’t pass and came up with this. If you have any questions please contact me at Eric@cosegan.com.

Formal verification is not new, but it’s normally very expensive so it’s only done on critical code like safety of flight or cryptographic libraries. I’m attempting to do it here for you as community service, since your project is open source and free to use.

---

**Environment**

- Kill Bill 0.24.22 (Docker image `killbill/killbill:0.24.22`, database `killbill/mariadb:0.24`), default settings
- killbill-stripe-plugin 8.0.4
- Stripe test mode

**We verified this on our sandbox only**: a test server in Docker, Stripe's test mode, and a slow answer made by a proxy. We have not seen it in production. The lines named below are the same on the plugin's `master` (9.0.1-SNAPSHOT) and `maint-for-0-24-x` branches; we read those and did not run them.

**What we are seeing**

When Stripe's answer to `POST /v1/payment_intents/{id}/capture` takes longer than the plugin's read timeout (60 s by default; we held it back 75 s):

1. Stripe has captured the payment.
2. Kill Bill answers its caller HTTP 504 after 30 s (`org.killbill.payment.plugin.timeout`) and records the capture as `UNKNOWN`. At 60 s the plugin stores its row as `UNDEFINED`.
3. At the Janitor's first check, 5 minutes later, `getPaymentInfo` searches charges for `metadata['kbTransactionId']` with the **capture's** transaction id (`StripePaymentPluginApi.java`, lines 199 to 202). That id is written to Stripe only when the PaymentIntent is created (lines 892 to 895), so the search is always empty. The row is rewritten as `CANCELED`, "Payment didn't happen - Cancelled by Janitor" (lines 212 to 225).
4. Kill Bill records the capture as `PLUGIN_FAILURE`.

Result: Stripe holds the money (`succeeded`, the whole amount received). Kill Bill reports a captured amount of 0, last success state `AUTH_SUCCESS`. After that:

- The same capture asked again fails: Stripe answers "already been captured", and the plugin stores that as "Payment didn't happen" too.
- A refund is refused by Kill Bill: "Invalid payment transition REFUND from state AUTH_SUCCESS".

Same outcome in 4 runs of 4. It ends the same way at once, without the 5-minute wait, when the connection breaks while Stripe is working (3 runs of 3) or the reconnection is refused (3 runs of 3).

From the run below (times are UTC; the proxy's log lines are shortened):

```
$ kb -i -X POST -d '{"amount":303,"currency":"USD"}' "$KB/1.0/kb/payments/$PAYMENT"
HTTP/1.1 504        (30 s after it was asked: "Plugin timeout ... Call TIMEOUT ... plugin='killbill-stripe'")

Six and a half minutes later, Kill Bill (GET /1.0/kb/payments/{paymentId}): "authAmount":303.00, "capturedAmount":0
    AUTHORIZE  303.00  SUCCESS
    CAPTURE    303.00  PLUGIN_FAILURE  gatewayErrorMsg "Payment didn't happen - Cancelled by Janitor"

Stripe now (GET /payment_intents on the proxy):
    pi_…lpmt  amount 30300  status succeeded  received 30300  capturable 0

The proxy's log:
    16:09:18.569  #5 POST /v1/payment_intents/pi_…lpmt/capture  came in
    16:09:19.457  #5 -> HTTP 200 (payment_intent pi_…lpmt, status succeeded, amount 30300, received 30300, capturable 0) after 0.888 s; FAULT (slow): the answer is held back until 75 s after the request came in
    16:10:33.572  #5 -> the held-back answer could not be given, 75.004 s after the request came in: the plugin had closed the connection
    16:14:50.494  #6 GET /v1/charges/search?…  came in
    16:14:50.808  #6 -> HTTP 200 (search_result with 0 item(s)) after 0.315 s

The same capture again:
HTTP/1.1 502        "Failed to submit payment transaction" (Stripe answered 400, payment_intent_unexpected_state)
A refund:
HTTP/1.1 400        "Invalid payment transition REFUND from state AUTH_SUCCESS"
```

**What we are expecting**

The capture is recorded as `SUCCESS`, with the captured amount, once Stripe has captured.

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

3. Authorize, arm the fault and capture:

```bash
# An account whose default payment method is a Stripe test card, and an authorization of 303 USD.
ACCOUNT=$(kb -D - -o /dev/null -X POST -d '{"name":"repro","currency":"USD"}' $KB/1.0/kb/accounts | new_id)
kb -X POST -d '{"pluginName":"killbill-stripe","pluginInfo":{"externalPaymentMethodId":"pm_card_visa"}}' "$KB/1.0/kb/accounts/$ACCOUNT/paymentMethods?isDefault=true"
PAYMENT=$(kb -D - -o /dev/null -X POST -d '{"transactionType":"AUTHORIZE","amount":303,"currency":"USD"}' "$KB/1.0/kb/accounts/$ACCOUNT/payments" | new_id)

# One fault: Stripe's answer to the capture is held back until 75 s after the request.
curl -s -X POST -d '{"method":"POST","path":"/v1/payment_intents/*/capture","fault":"slow","seconds":75}' http://127.0.0.1:8081/arm

# The capture. Kill Bill answers HTTP 504 after 30 s.
kb -i -X POST -d '{"amount":303,"currency":"USD"}' "$KB/1.0/kb/payments/$PAYMENT"
```

4. Wait until six and a half minutes after the capture was asked (the Janitor's first check comes 5 minutes after Kill Bill records it as unknown), then look, and ask for the capture again and for a refund:

```bash
kb "$KB/1.0/kb/payments/$PAYMENT"                # Kill Bill: "authAmount":303, "capturedAmount":0; the CAPTURE is PLUGIN_FAILURE
curl -s http://127.0.0.1:8081/payment_intents     # what Stripe holds now: succeeded, received 30300
curl -s http://127.0.0.1:8081/log                 # every request to Stripe, with times

# Afterwards, with no fault: the same capture again, then a refund.
kb -i -X POST -d '{"amount":303,"currency":"USD"}' "$KB/1.0/kb/payments/$PAYMENT"
kb -i -X POST -d '{"amount":303,"currency":"USD"}' "$KB/1.0/kb/payments/$PAYMENT/refunds"
```

**KPM diagnostic output**

Not attached here. We have the dump of this run, made with `kpm diagnostic --account-export=...` on that server right after the last command, and can send it.

**Possible fix**

The charge search cannot find a capture, a void or a refund, so it should not be applied to their rows. The PaymentIntent's id is in the authorization's row: read the PaymentIntent and decide from its `status` and `amount_received`. We built that on 8.0.4: the capture was `UNKNOWN` at first, then `SUCCESS` with the right amount at the Janitor's check (1 run). Two guards were needed. `succeeded` does not show which capture did it, so a second capture that Stripe refuses must not be taken for done. And the PaymentIntent has to be read even when the stored error is an HTTP 4xx. A refused reconnection is not covered by this change: it is stored as `CANCELED` at once (our report "Double charge after a refused reconnection"). We can send the patch we tested.

---

Posted to the project: https://github.com/killbill/killbill/issues/2342
