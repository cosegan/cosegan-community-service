# Hyperswitch: Stripe webhook answered with HTTP 500 for a payment whose answer was lost

I submitted these to the discussion section in your repository as recommended in your documentation. Also note these are not security vulnerabilities (I would have communicated those in private).

As part of Cosegan’s community service initiative, I examined some of your payment logic and noticed a few bugs I will share here. We formally verify critical software (using mathematical proofs). If the proofs pass, then we certify certain functionality, and you don’t have to worry that there are lurking bugs that AI will find later. However, if they don’t pass (like happened here), we know there is likely a bug in the code, even if we don’t yet know what that bug is. In your case, I drilled down to find out why the proofs didn’t pass and came up with this. If you have any questions please contact me at Eric@cosegan.com.

Formal verification is not new, but it’s normally very expensive so it’s only done on critical code like safety of flight or cryptographic libraries. I’m attempting to do it here for you as community service, since your project is open source and free to use.

---

### Bug Description

**We verified this on our sandbox only**: Hyperswitch v1.129.0's released Docker images on one machine, Stripe's test mode, and Stripe's answer held back past the router's time limit by a proxy; Stripe's own event delivered by us, signed with the connector's webhook secret. We have not seen it in production. Line numbers below are v1.129.0's; the same code is on `main` (commit `88c32f8`, 2026-11-20), at other line numbers in some files. We read it there and did not run it.

For a payment left `processing` with no Stripe id (after a time-out or a reset), Stripe's `payment_intent.succeeded` webhook is verified and matched to the right payment by `metadata[order_id]`, and then fails with HTTP 500 "Something went wrong". The Stripe connector reads a webhook through its payment-sync answer handler, which decides how to read the body from the payment's **stored** Stripe id (`crates/hyperswitch_connectors/src/connectors/stripe.rs`, lines 907 to 971), which this payment does not have. The webhook can never succeed for it.

### Expected Behavior

The webhook settles the payment to `succeeded`, taking Stripe's id from the event.

### Actual Behavior

1. A payment is confirmed; Stripe charges it; the answer comes after 30 s: `processing`, no Stripe id.
2. Stripe's `payment_intent.succeeded` event for it is delivered to `POST /webhooks/{merchant_id}/{connector_id}`, signed with the connector's webhook secret.
3. The router answers HTTP 500 (`HE_00`) and logs "Incoming webhook flow for payments failed", "Missing connector transaction ID at crates/hyperswitch_connectors/src/connectors/stripe.rs:971", "Expected connector transaction ID not found", with the right `payment_attempt_id`. The signature was accepted: the router took the path it takes only for a verified webhook, handling the event as the answer (`crates/router/src/core/webhooks/incoming.rs`, lines 937 to 952).
4. The payment stays `processing`.

Same in 2 runs of 2 (one time-out, one reset). Stripe retries a webhook that got an error, and every retry fails the same way; after the scheduler marks the payment `failed` the same event is answered 200 and ignored (our report "Scheduler marks a payment Stripe charged as failed; a retry charges the customer again").

### Steps To Reproduce

`fault-proxy.py`, a small proxy of our own (Python 3, standard library only; not included here, we can send it), stands between the router and Stripe: Stripe's `base_url` points at it, it sends every request on to Stripe's test mode, and it can make one fault on the next request that matches. It puts the real test key on each request, so Hyperswitch holds only a placeholder. Our test account refuses raw card numbers, so it also swaps the card fields of `POST /v1/payment_intents` for Stripe's test PaymentMethod `pm_card_visa`; on an account that accepts raw card data that is not needed. Any proxy that makes the same fault will do.

1. Start Hyperswitch v1.129.0, PostgreSQL, Redis and the proxy:

```bash
# Needs Docker, git, curl and jq; fault-proxy.py (our proxy; we can send it) in this directory;
# and a Stripe TEST secret key:   export STRIPE_SECRET_KEY=sk_test_...
[ -d hyperswitch ] || git clone --quiet --depth 1 --branch v1.129.0 https://github.com/juspay/hyperswitch.git
docker network create hs-net
docker run -d --name hs-pg --network hs-net -e POSTGRES_USER=db_user -e POSTGRES_PASSWORD=db_pass -e POSTGRES_DB=hyperswitch_db postgres:16
docker run -d --name hs-redis --network hs-net redis:7
until docker exec hs-pg pg_isready -U db_user -d hyperswitch_db >/dev/null 2>&1; do sleep 1; done
# The release's own migrations, in order (what `just migrate` applies with diesel)
docker run --rm --network hs-net -e PGPASSWORD=db_pass -v "$PWD/hyperswitch/migrations:/m:ro" postgres:16 bash -c \
  'for d in $(ls /m | sort); do [ -f /m/$d/up.sql ] || continue; psql -q -h hs-pg -U db_user -d hyperswitch_db -v ON_ERROR_STOP=1 -f /m/$d/up.sql >/dev/null 2>&1 || exit 1; done'
# Hyperswitch's own config/docker_compose.toml, with this setup's hosts, and Stripe's address pointed at the proxy
mkdir -p config && cp hyperswitch/config/superposition_seed.toml config/
sed -e 's/^host = "pg"$/host = "hs-pg"/' -e 's/^host = "redis-standalone"$/host = "hs-redis"/' \
    -e 's#^stripe.base_url = "https://api.stripe.com/"$#stripe.base_url = "http://hs-proxy:8080/"#' \
    hyperswitch/config/docker_compose.toml > config/docker_compose.toml
docker run -d --name hs-proxy --network hs-net -p 127.0.0.1:9081:8081 -e STRIPE_SECRET_KEY \
  -v "$PWD/fault-proxy.py:/fault-proxy.py:ro" python:3.12-slim python3 -u /fault-proxy.py
docker run -d --name hs-router --network hs-net -p 127.0.0.1:9080:8080 \
  -v "$PWD/config:/local/config:ro" -v "$PWD/config:/local/bin/config:ro" \
  juspaydotin/hyperswitch-router:v1.129.0-standalone /local/bin/router -f /local/config/docker_compose.toml
until curl -sf http://127.0.0.1:9080/health >/dev/null; do sleep 2; done
```

2. Create a merchant, an API key and the Stripe connector:

```bash
HS=http://127.0.0.1:9080
ADMIN='Authorization: admin-api-key=test_admin'   # the admin key of Hyperswitch's config/docker_compose.toml
MID=repro$(date +%s)
curl -s -X POST $HS/accounts -H "$ADMIN" -H 'Content-Type: application/json' \
  -d "{\"merchant_id\":\"$MID\",\"merchant_name\":\"repro\",\"return_url\":\"https://example.com\",\"primary_business_details\":[{\"country\":\"US\",\"business\":\"default\"}]}" | jq -c '{merchant_id}'
KEY=$(curl -s -X POST $HS/api_keys/$MID -H "$ADMIN" -H 'Content-Type: application/json' -d '{"name":"repro","expiration":"never"}' | jq -r .api_key)
hs() { curl -s -H "api-key: $KEY" -H 'Content-Type: application/json' "$@"; }
stripe() { curl -s -G http://127.0.0.1:9081/stripe --data-urlencode "path=$1"; }   # a read-only GET to Stripe through the proxy, which holds the key
# The Stripe connector, with a placeholder key (the proxy puts the real test key on each request)
MCA=$(hs -X POST $HS/account/$MID/connectors -d '{"connector_type":"payment_processor","connector_name":"stripe","connector_account_details":{"auth_type":"HeaderKey","api_key":"sk_test_placeholder"},"test_mode":true,"payment_methods_enabled":[{"payment_method":"card","payment_method_types":[{"payment_method_type":"credit","card_networks":["Visa"],"minimum_amount":1,"maximum_amount":68607706,"recurring_enabled":true,"installment_payment_enabled":false}]}]}' | jq -r .merchant_connector_id)
CARD='"payment_method":"card","payment_method_type":"credit","payment_method_data":{"card":{"card_number":"4242424242424242","card_exp_month":"12","card_exp_year":"2030","card_holder_name":"Repro","card_cvc":"123"}}'
PAYMENT='"currency":"USD","confirm":true,"authentication_type":"no_three_ds","customer_id":"repro","return_url":"https://example.com"'
echo "merchant $MID, connector $MCA"
```

3. Give the connector a webhook secret, and make a payment whose answer comes after 35 s:

```bash
SECRET=whsec_repro_$(openssl rand -hex 12)
hs -X POST $HS/account/$MID/connectors/$MCA -d "{\"connector_type\":\"payment_processor\",\"connector_webhook_details\":{\"merchant_secret\":\"$SECRET\"}}" | jq -c '{merchant_connector_id,status}'
curl -s "http://127.0.0.1:9081/arm?mode=delay&method=POST&path=/v1/payment_intents&seconds=35"; echo
PAY=repro09_$(date +%s)
hs -X POST $HS/payments -d "{\"payment_id\":\"$PAY\",\"amount\":6106,\"capture_method\":\"automatic\",$PAYMENT,$CARD}" | jq -c '{status,error_code,connector_transaction_id}'
```

4. Deliver Stripe's own event for it, signed as Stripe signs:

```bash
EVENT=$(stripe "/v1/events?type=payment_intent.succeeded&limit=20" | jq -c --arg o "${PAY}_1" '[.data[] | select(.data.object.metadata.order_id==$o)][0]')
T=$(date +%s); SIG=$(printf '%s.%s' "$T" "$EVENT" | openssl dgst -sha256 -hmac "$SECRET" | sed 's/^.*= //')
curl -s -w '\nHTTP %{http_code}\n' -X POST $HS/webhooks/$MID/$MCA -H 'Content-Type: application/json' -H "Stripe-Signature: t=$T,v1=$SIG" -d "$EVENT"
docker logs hs-router 2>&1 | grep -c 'Missing connector transaction ID'
hs $HS/payments/$PAY | jq -c '{status,connector_transaction_id}'   # still processing
```

### Context For The Bug

**Why it matters**: the webhook is the usual way a payment system learns the outcome of a charge. For these payments none of Hyperswitch's three ways works: a sync on request skips Stripe, the scheduler's sync skips Stripe, and the webhook gets 500. The scheduler then marks the payment `failed` while the customer has paid.

**Where**: `crates/hyperswitch_connectors/src/connectors/stripe.rs`, lines 907 to 971 (`handle_response` of PSync chooses how to read the answer from the stored id and fails with `MissingConnectorTransactionID` without one).

**Possible fix**: when the stored id is missing, take Stripe's id from the webhook body; the event carries the PaymentIntent, with its id and `metadata[order_id]`.

GitHub issues are switched off for `juspay/hyperswitch`, so there were no earlier issues to look through. We searched its pull requests for "Missing connector transaction ID", "webhook 500" and "webhook processing failed stripe" and found no report of this. One pull request touches the same handler: #14488, open since 2026-09-29 and not merged, changes `handle_response` for a sync without a Stripe id. We have not run it and do not know what it does to this webhook.

### Environment

Are you using hyperswitch hosted version? No.

Run locally, from the released Docker images:
1. Operating System: macOS (Darwin 25.6.0), Docker Desktop 4.47.0; the containers are Linux arm64
2. Rust version: not built by us. The image `juspaydotin/hyperswitch-router:v1.129.0-standalone` (`sha256:dbbc78845ac1c6dd3cdcc6bd00de11646351b37e2173cca5f32dd723a8021dea`) names Rust 1.99.1 in its build string
3. App version: v1.129.0 (the router logs `v1.129.0-dirty-cf913c5-2026-10-31T11:10:00`)

Connector: Stripe, test mode.

---

Posted to the project: https://github.com/juspay/hyperswitch/discussions/15345
