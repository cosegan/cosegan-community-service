# Hyperswitch: Scheduler marks a payment Stripe charged as failed; a retry charges the customer again

Found via [Cosegan](https://github.com/cosegan) formal/mathematical proof based verification and analysis.

---

### Bug Description

**We verified this on our sandbox only**: Hyperswitch v1.129.0's released Docker images on one machine, Stripe's test mode, and Stripe's answer held back past the router's time limit by a proxy, after Stripe had charged. We have not seen it in production. Line numbers below are v1.129.0's; the same code is on `main` (commit `88c32f8`, 2026-11-20), at other line numbers in some files. We read it there and did not run it.

For a payment left `processing` with no Stripe id (after a time-out or a reset; our report "Charged payment left processing for good after a Stripe timeout or reset"), the scheduler's sync task cannot ask Stripe. After its last retry it marks the payment `failed` on its own, although Stripe charged the card. A refund is then refused, Stripe's webhook is ignored, and with manual retry on, the customer trying again is charged a second time.

### Expected Behavior

A payment whose outcome is unknown is not written as `failed`; it is left for review (as the refund side does, below) or found at the processor.

### Actual Behavior

1. A payment is confirmed; Stripe charges it; the answer comes after 30 s, so Hyperswitch stores `processing` with no Stripe id.
2. The scheduler's sync task runs after 60 s and then 5 times, 300 s apart (the seeded `pt_mapping_payment_sync`); each run skips Stripe (no Stripe id).
3. At its last retry (`crates/router/src/workflows/payment_sync.rs`, lines 147 to 195: "If the payment status is still processing and there is no connector transaction_id then change the payment status to failed if all retries exceeded") the attempt becomes `Failure`, `amount_capturable` 0, and the payment `failed`: "This Payment has been moved to failed as there is no response from the connector".
4. A refund is refused (`IR_14`). Stripe's `payment_intent.succeeded` event, delivered to the router after this, is answered 200 and changes nothing. (Before step 3 the same event got HTTP 500: our report "Stripe webhook answered with HTTP 500 for a payment whose answer was lost".)
5. With manual retry on and a session expiry long enough for the retry window, the customer confirms the same payment again, and Stripe charges a second time.

Result: Stripe holds two charges for one payment; Hyperswitch shows one succeeded payment, and its first attempt holds no Stripe id at all.

For a refund in the same state, the same kind of job moves it to `ManualReview` instead (`crates/router/src/core/refunds.rs`, lines 2560 to 2597).

Steps 1 to 4 the same for 4 payments of 4 (plain HTTP and TLS); step 5 in 1 run of 1. From the run with the retry (times UTC):

```
23:16:46  POST /payments  amount 6301, session_expiry 7200   -> after 30 s: processing, TIMEOUT, no Stripe id
          Stripe: pi_…Fzk3 succeeded, amount_received 6301
23:43:11  the scheduler's last retry: payment failed (RETRIES_EXCEEDED)
23:43:46  POST /payments/{id}/confirm (the customer tries again)   -> succeeded, attempt_count 2
          Stripe: pi_…SoJF succeeded, amount_received 6301
```

### Steps To Reproduce

`fault-proxy.py`, a small proxy of our own (Python 3, standard library only; not included here, we can send it), stands between the router and Stripe: Stripe's `base_url` points at it, it sends every request on to Stripe's test mode, and it can make one fault on the next request that matches. It puts the real test key on each request, so Hyperswitch holds only a placeholder. Our test account refuses raw card numbers, so it also swaps the card fields of `POST /v1/payment_intents` for Stripe's test PaymentMethod `pm_card_visa`; on an account that accepts raw card data that is not needed. Any proxy that makes the same fault will do.

1. Start Hyperswitch v1.129.0, PostgreSQL, Redis, the proxy and the scheduler:

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
# The scheduler: the released producer and consumer images, with the same config
for role in consumer producer; do
  docker run -d --name hs-$role --network hs-net -e SCHEDULER_FLOW=$role \
    -v "$PWD/config:/local/config:ro" -v "$PWD/config:/local/bin/config:ro" \
    juspaydotin/hyperswitch-$role:v1.129.0-standalone /local/bin/scheduler -f /local/config/docker_compose.toml
done
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

3. Turn on manual retry, and make a payment whose answer comes after 35 s:

```bash
# Manual retry on, in the merchant's profile (a customer may try a failed payment again)
PROFILE=$(hs $HS/account/$MID/business_profile | jq -r '.[0].profile_id')
hs -X POST $HS/account/$MID/business_profile/$PROFILE -d '{"is_manual_retry_enabled":true}' | jq -c '{profile_id,is_manual_retry_enabled}'
curl -s "http://127.0.0.1:9081/arm?mode=delay&method=POST&path=/v1/payment_intents&seconds=35"; echo
PAY=repro08_$(date +%s)
hs -X POST $HS/payments -d "{\"payment_id\":\"$PAY\",\"amount\":6301,\"capture_method\":\"automatic\",\"session_expiry\":7200,$PAYMENT,$CARD}" | jq -c '{status,error_code,connector_transaction_id}'
```

4. Wait for the scheduler's last retry (about 26 minutes), then look, try a refund, and retry as the customer:

```bash
for i in $(seq 1 80); do [ "$(hs $HS/payments/$PAY | jq -r .status)" = failed ] && break; sleep 30; done
hs $HS/payments/$PAY | jq -c '{status,error_message,connector_transaction_id}'   # failed
stripe "/v1/payment_intents?limit=20" | jq -c --arg o "${PAY}_1" '.data[] | select(.metadata.order_id==$o) | {id,status,amount_received}'   # Stripe: succeeded
hs -X POST $HS/refunds -d "{\"payment_id\":\"$PAY\",\"amount\":6301}" | jq -c '.error'   # refused: IR_14
hs -X POST $HS/payments/$PAY/confirm -d "{$CARD,\"return_url\":\"https://example.com\"}" | jq -c '{status,amount_received,attempt_count}'
stripe "/v1/payment_intents?limit=20" | jq -c --arg p "$PAY" '.data[] | select((.metadata.order_id // "") | startswith($p)) | {id,status,amount_received,order_id:.metadata.order_id}'   # Stripe: two charges
```

### Context For The Bug

**Why it matters**: the customer has paid and the merchant's records say the payment failed; Hyperswitch can neither find nor refund the charge, so the usual next step is to ask the customer to pay again, and with manual retry that second payment goes through the same Hyperswitch payment.

**Where**: `crates/router/src/workflows/payment_sync.rs`, lines 87 to 100 (the sync task uses the same status operation, which skips the connector without a Stripe id) and 147 to 195 (the failure written on the last retry); compare `crates/router/src/core/refunds.rs`, lines 2560 to 2597.

**Possible fix**: do not write `failed` for a payment whose outcome is unknown: mark it for review, as the refund side does, or find the processor's object by the `metadata[order_id]` Hyperswitch already sends. Our report "Charged payment left processing for good after a Stripe timeout or reset" has the directions that avoid the unknown state.

GitHub issues are switched off for `juspay/hyperswitch`, so there were no earlier issues to look through. We searched its pull requests for "moved to failed as there is no response from the connector", "RETRIES_EXCEEDED" and "payment sync failed" and found two things. #2380, merged 2023-10-09, added the move to `failed` on purpose ("make long standing payments failed"): "These payments will not have the connector transaction id. So after the last retry by process tracker, these will be moved to failed." And pull request #14488, open since 2026-09-29 and not merged, which lets the Stripe connector's sync find a payment by `metadata[order_id]` when the Stripe id is missing; it is not in v1.129.0 or on `main` at `88c32f8`, and we have not run it.

### Environment

Are you using hyperswitch hosted version? No.

Run locally, from the released Docker images:
1. Operating System: macOS (Darwin 25.6.0), Docker Desktop 4.47.0; the containers are Linux arm64
2. Rust version: not built by us. The image `juspaydotin/hyperswitch-router:v1.129.0-standalone` (`sha256:dbbc78845ac1c6dd3cdcc6bd00de11646351b37e2173cca5f32dd723a8021dea`) names Rust 1.99.1 in its build string
3. App version: v1.129.0 (the router logs `v1.129.0-dirty-cf913c5-2026-10-31T11:10:00`)

Connector: Stripe, test mode.

---

Posted to the project: https://github.com/juspay/hyperswitch/discussions/15355
