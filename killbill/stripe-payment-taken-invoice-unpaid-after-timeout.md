# Kill Bill: Payment taken but invoice stays unpaid after a timeout (Stripe plugin 8.0.4)

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

When Stripe's answer to `POST /v1/payment_intents` takes longer than the plugin's read timeout (60 s by default; we held it back 75 s):

1. Stripe has made the payment.
2. The plugin stores the row with `overriddenTransactionStatus=UNDEFINED`, and Kill Bill records the payment as `UNKNOWN`.
3. `getPaymentInfo` then searches charges, finds the charge, logs "Fixing Stripe transaction", and updates the row from the PaymentIntent (`StripePaymentPluginApi.java`, lines 203 to 211).
4. `StripeDao.updateResponse` merges the new data into the old (`originalData.putAll(...)`, `StripeDao.java` line 268) and never removes `overriddenTransactionStatus`. The row now holds `status=succeeded` **and** `overriddenTransactionStatus=UNDEFINED`, and the override wins.
5. Kill Bill's Janitor logs "Unable to repair ... currentTransactionStatus='UNKNOWN', newTransactionStatus='UNKNOWN'".

Result: Stripe holds the money. In Kill Bill the payment stays `UNKNOWN` and the invoice stays unpaid. The Janitor's first check changed nothing in 5 runs of 5. In the 2 runs taken further (Kill Bill in test mode, its clock moved), the charge was found again at each of the seven default checks (5 minutes, 1 hour, then daily) and the payment was still `UNKNOWN` after the last.

From the run below (times are UTC; the proxy's log lines are shortened):

```
Kill Bill (GET /1.0/kb/accounts/{accountId}/payments): one payment, one transaction
    PURCHASE  305.00  UNKNOWN  gatewayErrorMsg "IOException during API request to Stripe (http://kb-proxy:8080): Read timed out ..."
Kill Bill (the account): "accountBalance":305.00

Stripe now (GET /payment_intents on the proxy):
    pi_…tVDs  amount 30500  status succeeded  received 30500

The proxy's log:
    16:17:54.063  #3 POST /v1/payment_intents  came in
    16:17:54.864  #3 -> HTTP 200 (payment_intent pi_…tVDs, status succeeded, amount 30500, received 30500) after 0.801 s; FAULT (slow): the answer is held back until 75 s after the request came in
    16:18:54.169  #4 GET /v1/charges/search?…  came in
    16:18:54.386  #4 -> HTTP 200 (search_result with 1 item(s): ch_…kpih succeeded) after 0.218 s
    16:18:54.428  #5 GET /v1/payment_intents/pi_…tVDs  came in
    16:18:54.727  #5 -> HTTP 200 (payment_intent pi_…tVDs, status succeeded, amount 30500, received 30500) after 0.300 s
    16:19:09.069  #3 -> the held-back answer could not be given, 75.006 s after the request came in: the plugin had closed the connection
    16:23:57.953  #6 GET /v1/charges/search?…  came in
    16:23:58.254  #6 -> HTTP 200 (search_result with 1 item(s): ch_…kpih succeeded) after 0.302 s
    16:23:58.258  #7 GET /v1/payment_intents/pi_…tVDs  came in
    16:23:58.488  #7 -> HTTP 200 (payment_intent pi_…tVDs, status succeeded, amount 30500, received 30500) after 0.230 s

Kill Bill's log:
    "Fixing Stripe transaction": 2 times (at 16:18:54 and at the Janitor's check, 16:23:58)
    Unable to repair paymentId='60300f7a…', paymentTransactionId='452664fd…', currentTransactionStatus='UNKNOWN', newTransactionStatus='UNKNOWN'
```

**What we are expecting**

Once the plugin has found the charge and read the PaymentIntent as `succeeded`, the payment becomes `SUCCESS` and the invoice is paid.

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

# One fault: Stripe's answer to the next POST /v1/payment_intents is held back until 75 s after the request.
curl -s -X POST -d '{"method":"POST","path":"/v1/payment_intents","fault":"slow","seconds":75}' http://127.0.0.1:8081/arm

# An invoice of 305 USD, which Kill Bill pays by itself.
kb -X POST -d '[{"accountId":"'$ACCOUNT'","amount":305,"currency":"USD","description":"repro"}]' "$KB/1.0/kb/invoices/charges/$ACCOUNT?autoCommit=true"
```

4. Wait six and a half minutes (the Janitor's first check comes 5 minutes after the payment is recorded as unknown), then look:

```bash
kb "$KB/1.0/kb/accounts/$ACCOUNT/payments"                  # Kill Bill: one payment, its transaction UNKNOWN
kb "$KB/1.0/kb/accounts/$ACCOUNT?accountWithBalance=true"   # Kill Bill: "accountBalance":305
curl -s http://127.0.0.1:8081/payment_intents                # what Stripe holds now: succeeded, received 30500
curl -s http://127.0.0.1:8081/log                            # every request to Stripe, with times
docker logs kb-server 2>&1 | grep -c 'Fixing Stripe transaction'
docker logs kb-server 2>&1 | grep -o 'Unable to repair[^"]*' | tail -1
```

**KPM diagnostic output**

Not attached here. We have the dump of this run, made with `kpm diagnostic --account-export=...` on that server right after the last command, and can send it.

**Possible fix**

When a charge is found for an `UNDEFINED` row, remove `overriddenTransactionStatus` (or write the status the PaymentIntent gives). That is three lines in `getPaymentInfo` in our test build on 8.0.4: the payment became `SUCCESS` and the invoice paid about a minute after the request, as soon as the plugin's own call had timed out and Kill Bill asked for the state (1 run). We can send the patch we tested.

---

Posted to the project: https://github.com/killbill/killbill/issues/2344
