# Hyperswitch: Void with a free-text cancellation_reason fails at Stripe and leaves the hold on the card

I submitted these to the discussion section in your repository as recommended in your documentation. Also note these are not security vulnerabilities (I would have communicated those in private).

As part of Cosegan’s community service initiative, I examined some of your payment logic and noticed a few bugs I will share here. We formally verify critical software (using mathematical proofs). If the proofs pass, then we certify certain functionality, and you don’t have to worry that there are lurking bugs that AI will find later. However, if they don’t pass (like happened here), we know there is likely a bug in the code, even if we don’t yet know what that bug is. In your case, I drilled down to find out why the proofs didn’t pass and came up with this. If you have any questions please contact me at Eric@cosegan.com.

Formal verification is not new, but it’s normally very expensive so it’s only done on critical code like safety of flight or cryptographic libraries. I’m attempting to do it here for you as community service, since your project is open source and free to use.

---

### Bug Description

**We verified this on our sandbox only**: Hyperswitch v1.129.0's released Docker images on one machine, Stripe's test mode, and no fault: the proxy only passed requests on. We have not seen it in production. Line numbers below are v1.129.0's; the same code is on `main` (commit `88c32f8`, 2026-11-20), at other line numbers in some files. We read it there and did not run it.

`POST /payments/{id}/cancel` takes `cancellation_reason` as free text ("The reason for the payment cancel"). The Stripe connector sends it to Stripe unchanged, but Stripe accepts only `duplicate`, `fraudulent`, `requested_by_customer` or `abandoned`, and refuses the void. Hyperswitch then marks the authorized payment `failed`, and afterwards refuses both a capture and a void, so the authorization stays open at Stripe, holding the customer's funds, until Stripe lets it lapse.

### Expected Behavior

The void succeeds (the reason mapped to one of Stripe's values, or not sent), or, if the processor refuses it, the payment stays `requires_capture` so that it can be voided or captured again.

### Actual Behavior

1. A card payment is authorized with `capture_method: manual` (`requires_capture` on both sides).
2. `POST /payments/{id}/cancel` with `{"cancellation_reason": "customer changed their mind"}`.
3. The Stripe connector copies the field into `POST /v1/payment_intents/{id}/cancel` (`crates/hyperswitch_connectors/src/connectors/stripe/transformers.rs`, lines 4599 to 4610). Stripe answers 400: "Invalid cancellation_reason: must be one of duplicate, fraudulent, requested_by_customer, or abandoned".
4. Hyperswitch answers HTTP 200 with `status: failed` (a void's error other than 5xx is `Failure`, `crates/router/src/core/payments/operations/payment_response.rs`, lines 2775 to 2778).
5. A capture is then refused by Hyperswitch: 400 `IR_14`, "This Payment could not be captured because it has a payment.status of failed". A void with `requested_by_customer` is refused too: 400 `IR_16`, "You cannot cancel this payment because it has status failed". Neither request reaches Stripe.

Result: Stripe holds the PaymentIntent `requires_capture`, the full amount capturable; Hyperswitch shows it `failed` and will neither capture nor release it. Same in 2 runs of 2.

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

3. Authorize, void with a reason in words, then try a capture and a void again:

```bash
PAY=repro06_$(date +%s)
hs -X POST $HS/payments -d "{\"payment_id\":\"$PAY\",\"amount\":7005,\"capture_method\":\"manual\",$PAYMENT,$CARD}" | jq -c '{payment_id,status}'
hs -X POST $HS/payments/$PAY/cancel -d '{"cancellation_reason":"customer changed their mind"}' | jq -c '{status,error_message}'
hs -X POST $HS/payments/$PAY/capture -d '{}' | jq -c '.error'
hs -X POST $HS/payments/$PAY/cancel -d '{"cancellation_reason":"requested_by_customer"}' | jq -c '.error'
```

4. Look at Stripe:

```bash
stripe "/v1/payment_intents/$(hs $HS/payments/$PAY | jq -r .connector_transaction_id)" | jq -c '{id,status,amount_capturable}'   # Stripe: still requires_capture
curl -s http://127.0.0.1:9081/log | grep -c '/cancel'   # one cancel reached Stripe; the capture and the second void did not
```

### Context For The Bug

**Why it matters**: a merchant who writes a reason in words, as the field invites, cannot release a customer's authorization; their records show a failed payment while the customer sees money held on the card. The merchant cannot capture it either. No fault is needed.

**Where**: `crates/api_models/src/payments.rs`, lines 12325 to 12331 (the field, a free string); `crates/hyperswitch_connectors/src/connectors/stripe/transformers.rs`, lines 4599 to 4610 (passed through); Stripe's accepted values are in its API reference for cancelling a PaymentIntent.

**Possible fix**: map the reason onto Stripe's four values (or send none; Stripe's field is optional) and keep the merchant's text in Hyperswitch. Separately, a void the processor refuses leaves an authorization that still exists, so the payment could stay `requires_capture` instead of becoming `failed`.

GitHub issues are switched off for `juspay/hyperswitch`, so there were no earlier issues to look through. We searched its pull requests for "cancellation_reason", "Invalid cancellation_reason" and "void failed stripe" and found no report of this.

### Environment

Are you using hyperswitch hosted version? No.

Run locally, from the released Docker images:
1. Operating System: macOS (Darwin 25.6.0), Docker Desktop 4.47.0; the containers are Linux arm64
2. Rust version: not built by us. The image `juspaydotin/hyperswitch-router:v1.129.0-standalone` (`sha256:dbbc78845ac1c6dd3cdcc6bd00de11646351b37e2173cca5f32dd723a8021dea`) names Rust 1.99.1 in its build string
3. App version: v1.129.0 (the router logs `v1.129.0-dirty-cf913c5-2026-10-31T11:10:00`)

Connector: Stripe, test mode.

---

Posted to the project: https://github.com/juspay/hyperswitch/discussions/15342
