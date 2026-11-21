# Kill Bill: Expired 3-D Secure payments left open at Stripe (Stripe plugin 8.0.4)

I submitted these to the mailing list as recommended in your documentation, but they haven’t shown up after a few days so there may be something wrong. Also note these are not security vulnerabilities (I would have communicated those in private).

As part of Cosegan’s community service initiative, I examined some of your payment logic and noticed a few bugs I will share here. We formally verify critical software (using mathematical proofs). If the proofs pass, then we certify certain functionality, and you don’t have to worry that there are lurking bugs that AI will find later. However, if they don’t pass (like happened here), we know there is likely a bug in the code, even if we don’t yet know what that bug is. In your case, I drilled down to find out why the proofs didn’t pass and came up with this. If you have any questions please contact me at Eric@cosegan.com.

Formal verification is not new, but it’s normally very expensive so it’s only done on critical code like safety of flight or cryptographic libraries. I’m attempting to do it here for you as community service, since your project is open source and free to use.

---

**Environment**

- Kill Bill 0.24.22 (Docker image `killbill/killbill:0.24.22`, database `killbill/mariadb:0.24`), default settings and `org.killbill.server.test.mode=true` so that its clock can be moved
- killbill-stripe-plugin 8.0.4
- Stripe test mode, the test card `pm_card_threeDSecure2Required`

**We verified this on our sandbox only**: a test server in Docker and Stripe's test mode. Kill Bill's clock was moved; Stripe's was not. No 3-D Secure challenge was completed, so no money was taken in these runs. We have not seen it in production. The lines named below are the same on the plugin's `master` (9.0.1-SNAPSHOT) and `maint-for-0-24-x` branches; we read those and did not run them.

**What we are seeing**

`getPaymentInfo` checks expiry before it asks Stripe anything (`StripePaymentPluginApi.java`, lines 146 to 163). A pending payment older than the expiry period (3 hours by default for one waiting for 3-D Secure) gets `overriddenTransactionStatus=CANCELED`, "Payment Expired - Cancelled by Janitor". No request goes to Stripe: the PaymentIntent is neither read nor cancelled.

For an invoice that Kill Bill pays by itself with a card that needs 3-D Secure:

1. The PaymentIntent is `requires_action`, and Kill Bill records `PENDING`.
2. Janitor check after 1 hour: the plugin reads the PaymentIntent; still pending.
3. Janitor check after 25 hours: the plugin marks the row `CANCELED` without contacting Stripe. Kill Bill records `PLUGIN_FAILURE` and queues an invoice retry.
4. The retry creates a new PaymentIntent, which also waits. Steps 2 to 4 repeat until Kill Bill's retry limit.

Result, run to the end: 9 PaymentIntents at Stripe for one invoice, all still `requires_action`, and 0 cancel requests. Kill Bill shows one payment with 9 failed purchase transactions, and the invoice unpaid. Same outcome in 4 runs of 4.

Not run: a customer who completes one of those PaymentIntents later would be charged for a purchase Kill Bill has recorded as failed.

From the run below:

```
Kill Bill (GET /1.0/kb/accounts/{accountId}/payments, the transactions' statuses counted):
       9 "status":"PLUGIN_FAILURE"
Kill Bill (the account): "accountBalance":308.00

Stripe now (GET /payment_intents on the proxy), nine lines:
    pi_…5tJX  amount 30800  status requires_action  received 0  created 16:31:04Z
    pi_…aVeB  amount 30800  status requires_action  received 0  created 16:31:41Z
    ... seven more, all requires_action, the last created 16:36:42Z

Cancel requests in the proxy's log: 0

The first of the nine, in the proxy's log and in Kill Bill's log (real times, UTC; Kill Bill's clock is moved between the lines):
    16:31:04.328  #3 POST /v1/payment_intents  came in
    16:31:04.868  #3 -> HTTP 200 (payment_intent pi_…5tJX, status requires_action, amount 30800)
    16:31:04.928  #4 GET /v1/payment_intents/pi_…5tJX  came in            (the plugin reads it back)
    16:31:05.194  #4 -> HTTP 200 (payment_intent pi_…5tJX, status requires_action, amount 30800)
    16:31:13.586  #5 GET /v1/payment_intents/pi_…5tJX  came in            (Kill Bill's clock 1 hour on: the Janitor's first check)
    16:31:13.790  #5 -> HTTP 200 (payment_intent pi_…5tJX, status requires_action, amount 30800)
    16:31:25.952  Kill Bill's log: "Canceling expired Stripe transaction pi_…5tJX (created 2026-11-18T16:31:04)"
                  (Kill Bill's clock 25 hours on; no request reaches the proxy)
    16:31:41.429  #6 POST /v1/payment_intents  came in                     (Kill Bill's retry: a new PaymentIntent)
```

**What we are expecting**

When the plugin gives up on a pending payment, the PaymentIntent is cancelled at Stripe, so that it cannot be completed later, and `CANCELED` is recorded only when Stripe confirms.

**How to reproduce**

`fault-proxy.py`, a small proxy of our own (Python 3, standard library only; not included here, we can send it), stands between the plugin and Stripe. No fault is made here; it is used for its log of every request and to read what Stripe holds. The plugin's `apiBase` points at it, and it puts the real test key on each request, so Kill Bill's settings hold only a placeholder.

1. Start the database, the proxy and Kill Bill (in test mode):

```bash
# Needs Docker, curl, fault-proxy.py (our proxy script; we can send it) in this directory, and a Stripe TEST secret key:
#   export STRIPE_SECRET_KEY=sk_test_...
[ -f stripe-plugin-8.0.4.jar ] || curl -sSfO https://repo1.maven.org/maven2/org/kill-bill/billing/plugin/java/stripe-plugin/8.0.4/stripe-plugin-8.0.4.jar
docker network create kb-net
docker run -d --name kb-db --network kb-net -e MYSQL_ROOT_PASSWORD=killbill -e MARIADB_ROOT_PASSWORD=killbill killbill/mariadb:0.24
until docker exec kb-db mariadb -h127.0.0.1 -uroot -pkillbill killbill -e 'select count(*) from stripe_responses' >/dev/null 2>&1; do sleep 2; done
docker run -d --name kb-proxy --network kb-net -p 127.0.0.1:8081:8081 -e STRIPE_SECRET_KEY \
  -v "$PWD/fault-proxy.py:/fault-proxy.py:ro" python:3.12-slim python3 -u /fault-proxy.py
docker run -d --name kb-server --network kb-net -p 127.0.0.1:8080:8080 -e KB_org_killbill_server_test_mode=true \
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

3. Create an invoice on a card that needs 3-D Secure, then move the clock through every queued Janitor check and retry:

```bash
# An account whose default payment method is Stripe's test card that always needs 3-D Secure.
ACCOUNT=$(kb -D - -o /dev/null -X POST -d '{"name":"repro","currency":"USD"}' $KB/1.0/kb/accounts | new_id)
kb -X POST -d '{"pluginName":"killbill-stripe","pluginInfo":{"externalPaymentMethodId":"pm_card_threeDSecure2Required"}}' "$KB/1.0/kb/accounts/$ACCOUNT/paymentMethods?isDefault=true"

# An invoice of 308 USD, which Kill Bill tries to pay by itself. No fault is made.
kb -X POST -d '[{"accountId":"'$ACCOUNT'","amount":308,"currency":"USD","description":"repro"}]' "$KB/1.0/kb/invoices/charges/$ACCOUNT?autoCommit=true"
sleep 10
kb "$KB/1.0/kb/accounts/$ACCOUNT/payments"     # one payment, its transaction PENDING

# Move Kill Bill's clock to 30 s past each queued repair check or retry in turn, until nothing is queued.
next_due() { docker exec kb-db mariadb -uroot -pkillbill killbill -N -B -e "select date_format(min(effective_date) + interval 30 second, '%Y-%m-%dT%H:%i:%s.000Z') from notifications where queue_name like '%janitor%' or queue_name like '%retry%'"; }
for i in $(seq 1 60); do
  T=$(next_due); [ "$T" = NULL ] && break
  kb -X POST "$KB/1.0/kb/test/clock?requestedDate=$T&timeoutSec=20"; echo
  sleep 10
done
```

4. Look:

```bash
kb "$KB/1.0/kb/accounts/$ACCOUNT/payments" | grep -o '"status":"[A-Z_]*"' | sort | uniq -c   # Kill Bill: 9 x PLUGIN_FAILURE
kb "$KB/1.0/kb/accounts/$ACCOUNT?accountWithBalance=true"                                   # Kill Bill: "accountBalance":308
curl -s http://127.0.0.1:8081/payment_intents                                                # what Stripe holds now: 9 x requires_action
curl -s http://127.0.0.1:8081/log | grep -c '/cancel'                                        # cancel requests sent to Stripe: 0
```

**KPM diagnostic output**

Not attached here. We have the dump of this run, made with `kpm diagnostic --account-export=...` on that server right after the last command, and can send it.

**Possible fix**

When a pending payment expires, cancel the PaymentIntent at Stripe, and store `CANCELED` only when Stripe confirms. If Stripe reports that it succeeded in the meantime, record that instead. We built that on 8.0.4 and ran the same scenario to the end: 9 PaymentIntents, all 9 cancelled at Stripe by the plugin as each expired, none left open. Kill Bill's side was as before (9 failed purchase transactions, the invoice unpaid). We can send the patch we tested.
