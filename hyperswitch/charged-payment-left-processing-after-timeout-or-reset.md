# Hyperswitch: Charged payment left processing for good after a Stripe timeout or reset

I submitted these to the discussion section in your repository as recommended in your documentation. Also note these are not security vulnerabilities (I would have communicated those in private).

As part of Cosegan’s community service initiative, I examined some of your payment logic and noticed a few bugs I will share here. We formally verify critical software (using mathematical proofs). If the proofs pass, then we certify certain functionality, and you don’t have to worry that there are lurking bugs that AI will find later. However, if they don’t pass (like happened here), we know there is likely a bug in the code, even if we don’t yet know what that bug is. In your case, I drilled down to find out why the proofs didn’t pass and came up with this. If you have any questions please contact me at Eric@cosegan.com.

Formal verification is not new, but it’s normally very expensive so it’s only done on critical code like safety of flight or cryptographic libraries. I’m attempting to do it here for you as community service, since your project is open source and free to use.

---

### Bug Description

**We verified this on our sandbox only**: Hyperswitch v1.129.0's released Docker images on one machine, Stripe's test mode, and Stripe's answer held back, or the connection reset, by a proxy after Stripe had answered. We have not seen it in production. Line numbers below are v1.129.0's; the same code is on `main` (commit `88c32f8`, 2026-11-20), at other line numbers in some files. We read it there and did not run it.

If Stripe takes longer than the router's 30-second limit, or the connection is reset before the answer arrives, Hyperswitch stores the payment as `processing` with no Stripe id, while Stripe has charged the card. Hyperswitch's sync needs the Stripe id, so it never asks Stripe; the charge cannot be found by Hyperswitch's own means. After a reset the merchant is answered HTTP 500 "Something went wrong".

### Expected Behavior

The payment's outcome can be learned: a sync, the scheduler or Stripe's webhook brings it to `succeeded` (for example by an idempotency key on the payment request, or by finding the PaymentIntent from the `metadata[order_id]` Hyperswitch already sends).

### Actual Behavior

1. A payment is confirmed; the router sends `POST /v1/payment_intents`; Stripe charges the card.
2a. Time-out: the answer is held back past 30 s (`REQUEST_TIME_OUT = 30`, `crates/common_utils/src/consts.rs`, line 277). The router stores error code `TIMEOUT` with status 504; the attempt becomes `pending` (`crates/hyperswitch_interfaces/src/api_client.rs`, lines 450 to 469). The merchant gets HTTP 200, `status: processing`, "Connector did not respond in specified time".
2b. Reset: the call ends with `ProcessingStepFailed`; the merchant gets HTTP 500 "Something went wrong" (`HE_00`); the attempt stays `pending`. No re-send is made after a reset.
3. In both cases: `processing`, no `connector_transaction_id`.
4. `GET /payments/{id}?force_sync=true` sends nothing to Stripe. The router logs "validate_psync_reference_id failed, hence skipping call to connector" (`crates/router/src/core/payments/flows/psync_flow.rs`, lines 200 to 215; the default check needs the processor's id, `crates/hyperswitch_interfaces/src/api.rs`, lines 974 to 986).

Result: Stripe holds a succeeded charge; Hyperswitch shows a payment that never finishes. Same in 4 runs of 4 (two time-outs and one reset with plain HTTP to the proxy, one reset over TLS through a TLS-inspecting egress proxy). What happens next is in our reports "Stripe webhook answered with HTTP 500 for a payment whose answer was lost" and "Scheduler marks a payment Stripe charged as failed; a retry charges the customer again".

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

3. A payment whose answer comes after 35 s, then a forced sync:

```bash
# One fault: Stripe's answer to the next POST /v1/payment_intents is held back 35 s (the router gives up at 30 s).
curl -s "http://127.0.0.1:9081/arm?mode=delay&method=POST&path=/v1/payment_intents&seconds=35"; echo
PAY1=repro07a_$(date +%s)
hs -X POST $HS/payments -d "{\"payment_id\":\"$PAY1\",\"amount\":6103,\"capture_method\":\"automatic\",$PAYMENT,$CARD}" | jq -c '{status,error_code,connector_transaction_id}'
hs "$HS/payments/$PAY1?force_sync=true" | jq -c '{status,connector_transaction_id}'
docker logs hs-router 2>&1 | grep -c 'validate_psync_reference_id failed, hence skipping call to connector'
```

4. A payment whose connection is reset after Stripe answered:

```bash
# One fault: the next POST /v1/payment_intents is sent to Stripe, then the router's connection is reset (RST).
curl -s "http://127.0.0.1:9081/arm?mode=reset&method=POST&path=/v1/payment_intents"; echo
PAY2=repro07b_$(date +%s)
hs -o /dev/null -w 'HTTP %{http_code}\n' -X POST $HS/payments -d "{\"payment_id\":\"$PAY2\",\"amount\":6104,\"capture_method\":\"automatic\",$PAYMENT,$CARD}"
hs $HS/payments/$PAY2 | jq -c '{status,connector_transaction_id}'
```

5. Look at Stripe:

```bash
stripe "/v1/payment_intents?limit=20" | jq -c --arg a "${PAY1}_1" --arg b "${PAY2}_1" '.data[] | select(.metadata.order_id==$a or .metadata.order_id==$b) | {id,status,amount_received,order_id:.metadata.order_id}'   # both charged
curl -s http://127.0.0.1:9081/log | grep -c 'GET /v1/payment_intents/'   # 0: no sync request ever reached Stripe
```

### Context For The Bug

**Why it matters**: the customer has paid and the merchant sees a payment that never finishes; after a reset the merchant was also told "Something went wrong", which invites a second try and a second charge.

**Where**: as above. The Stripe connector sends `metadata[order_id]` (the attempt id) with every payment (`crates/hyperswitch_connectors/src/connectors/stripe/transformers.rs`, lines 300 to 305), but nothing uses it to find a payment whose id is missing, and no idempotency key is sent, so a repeated request cannot recover the first result either.

**Possible fix**: send an idempotency key with the payment request so that a retry or a sync gets the first result; or, when the Stripe id is missing, find the PaymentIntent by `metadata[order_id]` (Stripe's search API).

GitHub issues are switched off for `juspay/hyperswitch`, so there were no earlier issues to look through. We searched its pull requests for "validate_psync_reference_id", "Connector did not respond in specified time", "payment stuck processing" and "timeout stripe" and found one thing: pull request #14488, open since 2026-09-29 and not merged, which lets the Stripe connector's sync find a payment by `metadata[order_id]` when the Stripe id is missing. It is the second direction above. It is not in v1.129.0 or on `main` at `88c32f8`, and we have not run it.

### Environment

Are you using hyperswitch hosted version? No.

Run locally, from the released Docker images:
1. Operating System: macOS (Darwin 25.6.0), Docker Desktop 4.47.0; the containers are Linux arm64
2. Rust version: not built by us. The image `juspaydotin/hyperswitch-router:v1.129.0-standalone` (`sha256:dbbc78845ac1c6dd3cdcc6bd00de11646351b37e2173cca5f32dd723a8021dea`) names Rust 1.99.1 in its build string
3. App version: v1.129.0 (the router logs `v1.129.0-dirty-cf913c5-2026-10-31T11:10:00`)

Connector: Stripe, test mode.
