# Hyperswitch: Capture made at Stripe but payment recorded as failed, then charged again on retry

I submitted these to the discussion section in your repository as recommended in your documentation. Also note these are not security vulnerabilities (I would have communicated those in private).

As part of Cosegan’s community service initiative, I examined some of your payment logic and noticed a few bugs I will share here. We formally verify critical software (using mathematical proofs). If the proofs pass, then we certify certain functionality, and you don’t have to worry that there are lurking bugs that AI will find later. However, if they don’t pass (like happened here), we know there is likely a bug in the code, even if we don’t yet know what that bug is. In your case, I drilled down to find out why the proofs didn’t pass and came up with this. If you have any questions please contact me at Eric@cosegan.com.

Formal verification is not new, but it’s normally very expensive so it’s only done on critical code like safety of flight or cryptographic libraries. I’m attempting to do it here for you as community service, since your project is open source and free to use.

---

### Bug Description

**We verified this on our sandbox only**: Hyperswitch v1.129.0's released Docker images on one machine, Stripe's test mode, and a connection to Stripe dropped on purpose by a proxy after Stripe had answered. We have not seen it in production. Line numbers below are v1.129.0's; the same code is on `main` (commit `88c32f8`, 2026-11-20), at other line numbers in some files. We read it there and did not run it.

When the connection closes before the router has Stripe's answer to a capture, the router sends the capture again by itself. Stripe refuses the copy (already captured), and Hyperswitch stores the payment as `failed`, although Stripe has taken the money. A sync does not correct it, Stripe's webhooks do not correct it, a refund is refused, and with manual retry turned on, the customer trying again is charged a second time.

### Expected Behavior

The payment is recorded as succeeded, with the captured amount, because Stripe captured it.

### Actual Behavior

1. A payment is authorized with `capture_method: manual`. The capture is sent: `POST /v1/payment_intents/{id}/capture`; Stripe captures and answers 200.
2. The connection closes before the answer reaches the router; the router sends the same body again.
3. Stripe answers the copy 400 `payment_intent_unexpected_state`, "This PaymentIntent could not be captured because it has already been captured."
4. For a capture, any error other than 5xx and 429 is stored as `Failure` (`crates/router/src/core/payments/operations/payment_response.rs`, lines 2765 to 2771): the payment is `failed`.
5. `GET /payments/{id}?force_sync=true` sends nothing to Stripe. Stripe's `payment_intent.succeeded` and `charge.captured` events, delivered to the router with a webhook secret configured, are answered 200 and change nothing (the status operation acts only on payments in progress, `crates/router/src/core/payments.rs`, lines 11384 to 11415). A refund is refused: "This Payment could not be refund because it has a status of failed" (`IR_14`).
6. With manual retry on, the customer confirms the same payment again: a second attempt, a second PaymentIntent, and after the merchant's capture a second charge.

Result: Stripe holds two captured charges for one payment; Hyperswitch shows one succeeded payment of the amount (attempt 1 `failure`, attempt 2 `charged`).

Steps 1 to 5 the same in 3 runs of 3 (twice with plain HTTP to the proxy, once over TLS through a TLS-inspecting egress proxy with Stripe's real address); step 6 in 1 run of 1.

From the run with manual retry (times UTC; the proxy's log lines, shortened):

```
23:03:33.032  POST /v1/payment_intents/pi_…JDNU/capture  body=f47b5b799f  idem=-
23:03:33.798  <- 200 payment_intent pi_…JDNU succeeded received=7006
23:03:33.800  FAULT: Stripe answered; the router's connection closed (FIN) without the answer
23:03:33.804  POST /v1/payment_intents/pi_…JDNU/capture  body=f47b5b799f  idem=-   <- the same body
23:03:34.084  <- 400 payment_intent_unexpected_state "…could not be captured because it has already been captured."
              Hyperswitch: payment pay_QYBRfD0NUaJiW9mReCPX  status failed
23:03:34.163  POST /v1/payment_intents  (the customer's retry: POST /payments/{id}/confirm)
23:03:35.021  <- 200 payment_intent pi_…b1nL requires_capture amount=7006 order_id=pay_QYBRfD0NUaJiW9mReCPX_2
23:03:35.116  POST /v1/payment_intents/pi_…b1nL/capture
23:03:35.948  <- 200 payment_intent pi_…b1nL succeeded received=7006
              Hyperswitch: status succeeded  amount_received 7006  attempt_count 2
```

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

3. Turn on manual retry, authorize, arm the fault and capture:

```bash
# Manual retry on, in the merchant's profile (a customer may try a failed payment again)
PROFILE=$(hs $HS/account/$MID/business_profile | jq -r '.[0].profile_id')
hs -X POST $HS/account/$MID/business_profile/$PROFILE -d '{"is_manual_retry_enabled":true}' | jq -c '{profile_id,is_manual_retry_enabled}'
PAY=repro04_$(date +%s)
hs -X POST $HS/payments -d "{\"payment_id\":\"$PAY\",\"amount\":7006,\"capture_method\":\"manual\",$PAYMENT,$CARD}" | jq -c '{payment_id,status,amount_capturable}'
# One fault: the next capture is sent to Stripe, then the router's connection is closed without the answer.
curl -s "http://127.0.0.1:9081/arm?mode=lost&method=POST&path=/v1/payment_intents/pi_"; echo
hs -X POST $HS/payments/$PAY/capture -d '{}' | jq -c '{status,error_code,error_message}'
```

4. Look at both sides, sync, try a refund, then retry as the customer and capture:

```bash
curl -s http://127.0.0.1:9081/log | grep -E 'capture|FAULT'
PI1=$(hs $HS/payments/$PAY | jq -r .connector_transaction_id)
stripe "/v1/payment_intents/$PI1" | jq -c '{id,status,amount_received}'   # Stripe: captured
hs "$HS/payments/$PAY?force_sync=true" | jq -c '{status,amount_received}'   # Hyperswitch: failed; the sync sends nothing to Stripe
hs -X POST $HS/refunds -d "{\"payment_id\":\"$PAY\",\"amount\":7006}" | jq -c '.error'   # refused: IR_14
# The customer tries again on the same payment, and the merchant captures it.
hs -X POST $HS/payments/$PAY/confirm -d "{$CARD,\"return_url\":\"https://example.com\"}" | jq -c '{status,attempt_count,connector_transaction_id}'
hs -X POST $HS/payments/$PAY/capture -d '{}' | jq -c '{status,amount_received,attempt_count}'
stripe "/v1/payment_intents?limit=20" | jq -c --arg p "$PAY" '.data[] | select((.metadata.order_id // "") | startswith($p)) | {id,status,amount_received,order_id:.metadata.order_id}'   # Stripe: two captured
```

### Context For The Bug

**Why it matters**: the customer has paid and the merchant's records say the payment failed; the money cannot be refunded through Hyperswitch, and the customer trying again is charged again.

**Where**: `crates/external_services/src/http_client.rs`, lines 221 to 260: when the connection closes before the answer is complete (`hyper` "connection closed before message completed"), the request is sent again, once, on a new connection. The comment above it says why that is unsafe: "Since hyper already wrote some of the request, it can't really retry it automatically on a new connection, since the server may have acted already" (lines 230 to 238). It came in with #3426 (merged 2024-02-06) as a temporary fix until reqwest used hyper 1.0; the router now uses hyper 1.6 and reqwest 0.13. The Stripe connector (`crates/hyperswitch_connectors/src/connectors/stripe.rs`) sends no `Idempotency-Key` on any request, so Stripe treats the copy as a new request. Then `payment_response.rs`, lines 2765 to 2771 (a capture's 4xx is `Failure`), and `crates/router/src/core/payments/helpers.rs`, lines 5740 to 5800 and 6154 to 6200 (a `failed` payment whose attempt is `failure` may be confirmed again when manual retry is on). The same cause as our report "Double charge after a dropped connection to Stripe".

**Possible fix**: Do not re-send a request that may have reached the processor (the re-send is only safe when nothing was written), or send an idempotency key that the processor honours, the same on the copy: Stripe's `Idempotency-Key` header (the attempt id for a payment, the refund id for a refund) makes Stripe answer the copy with the first result instead of acting again. Beside it, a capture refused because the payment is already captured is not a failed payment; reading the PaymentIntent (a sync) would show it captured.

GitHub issues are switched off for `juspay/hyperswitch`, so there were no earlier issues to look through. We searched its pull requests for "already been captured", "capture failed", "payment_intent_unexpected_state" and "manual retry" and found no report of this.

### Environment

Are you using hyperswitch hosted version? No.

Run locally, from the released Docker images:
1. Operating System: macOS (Darwin 25.6.0), Docker Desktop 4.47.0; the containers are Linux arm64
2. Rust version: not built by us. The image `juspaydotin/hyperswitch-router:v1.129.0-standalone` (`sha256:dbbc78845ac1c6dd3cdcc6bd00de11646351b37e2173cca5f32dd723a8021dea`) names Rust 1.99.1 in its build string
3. App version: v1.129.0 (the router logs `v1.129.0-dirty-cf913c5-2026-10-31T11:10:00`)

Connector: Stripe, test mode.

---

Posted to the project: https://github.com/juspay/hyperswitch/discussions/15341
