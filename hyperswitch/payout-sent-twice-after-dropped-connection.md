# Hyperswitch: Payout sent twice after a dropped connection to Stripe

Found via [Cosegan](https://github.com/cosegan) formal/mathematical proof based verification and analysis.

---

### Bug Description

**We verified this on our sandbox only**: Hyperswitch v1.129.0's released Docker images on one machine, Stripe's test mode, and a connection to Stripe dropped on purpose by a proxy after Stripe had answered. We have not seen it in production. Line numbers below are v1.129.0's; the same code is on `main` (commit `88c32f8`, 2026-11-20), at other line numbers in some files. We read it there and did not run it.

A Stripe payout is a transfer to the payee's connected account (`POST /v1/transfers`), then a payout from that account to its bank (`POST /v1/payouts`). When the connection closes before the router has the transfer's answer, the router sends the same transfer again by itself, with no idempotency key, and Stripe transfers the amount a second time. Hyperswitch records one payout.

### Expected Behavior

One transfer to the payee for one payout.

### Actual Behavior

1. A payout of 10.01 USD to a payee whose Stripe account Hyperswitch holds. The router adds the bank account (`POST /v1/accounts/{id}/external_accounts`), then sends `POST /v1/transfers`. Stripe transfers and answers 200.
2. The connection closes before the answer reaches the router. 5 ms later the router sends the identical body again on a new connection ("Retrying request due to connection closed before message could complete").
3. Stripe makes a second transfer of 10.01 to the same account; both carry `transfer_group` = the payout attempt.
4. The router goes on to the payout from the account (10.01) and records one payout, `pending`.

Result: the merchant's balance paid 20.02 for a payout of 10.01; 10.01 went on to the payee's bank and 10.01 stays in the payee's Stripe balance, which Hyperswitch does not know of.

From the run (times UTC; the proxy's log lines, shortened):

```
02:02:57.371  POST /v1/transfers  body=b4ec0f695b  idem=-
02:02:58.226  <- 200 transfer tr_…wwcW amount=1001
02:02:58.229  FAULT: Stripe answered; the router's connection closed (FIN) without the answer
02:02:58.234  POST /v1/transfers  body=b4ec0f695b  idem=-      <- the same body, new connection
02:02:59.124  <- 200 transfer tr_…WBIU amount=1001
02:02:59.137  POST /v1/payouts  -> 200 po_…mzTo pending amount=1001

Stripe, GET /v1/transfers?destination=acct_…XDVi: two transfers of 1001, transfer_group po_p1_95226576_1
Hyperswitch, GET /payouts/po_p1_95226576: status pending, amount 1001
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

3. The payout connector and the payee:

```bash
# A payout connector (Stripe), beside the payment connector
hs -X POST $HS/account/$MID/connectors -d '{"connector_type":"payout_processor","connector_name":"stripe","connector_label":"stripe_payout","connector_account_details":{"auth_type":"HeaderKey","api_key":"sk_test_placeholder"},"test_mode":true,"payment_methods_enabled":[{"payment_method":"bank_transfer","payment_method_types":[{"payment_method_type":"ach","minimum_amount":1,"maximum_amount":68607706,"recurring_enabled":true,"installment_payment_enabled":false}]}]}' | jq -c '{merchant_connector_id,connector_type}'
# The payee's Stripe account. Hyperswitch would create it with Accounts v1 (POST /v1/accounts); our test account refuses
# that ("Stripe no longer recommends Accounts v1 for new Connect integrations"), so it is made with Accounts v2 here.
ACCT=$(curl -s https://api.stripe.com/v2/core/accounts -H "Authorization: Bearer $STRIPE_SECRET_KEY" -H 'Stripe-Version: 2026-11-18.endive' --json '{"display_name":"repro payee","contact_email":"jane.doe@example.com","dashboard":"none","identity":{"country":"US","entity_type":"individual","individual":{"given_name":"Jane","surname":"Doe","email":"jane.doe@example.com","phone":"+18885551234","date_of_birth":{"day":1,"month":1,"year":1901},"address":{"line1":"address_full_match","city":"San Francisco","state":"CA","postal_code":"94105","country":"US"},"id_numbers":[{"type":"us_ssn","value":"000000000"}]},"attestations":{"terms_of_service":{"account":{"date":"2026-11-20T00:00:00.000Z","ip":"203.0.113.10"}}}},"defaults":{"profile":{"business_url":"https://hyperswitch.io"},"responsibilities":{"fees_collector":"application","losses_collector":"application"}},"configuration":{"recipient":{"capabilities":{"stripe_balance":{"stripe_transfers":{"requested":true}}}}}}' | jq -r .id)
echo "payee account $ACCT"
# A Hyperswitch customer whose recipient for this connector is that account: what Hyperswitch stores itself once it has
# paid a payee (stored in crates/router/src/core/payouts.rs, lines 1777 to 1848; read in crates/router/src/core/utils.rs,
# lines 272 to 288), so the payout starts at the payee's bank account.
hs -X POST $HS/customers -d '{"customer_id":"payee1","name":"Jane Doe","email":"jane.doe@example.com","phone":"8885551234","phone_country_code":"+1"}' | jq -c '{customer_id}'
PROFILE=$(hs $HS/account/$MID/business_profile | jq -r '.[0].profile_id')
docker exec hs-pg psql -q -U db_user -d hyperswitch_db -c "update customers set connector_customer = jsonb_build_object('${PROFILE}_stripe', '$ACCT') where customer_id = 'payee1' and merchant_id = '$MID'"
PAYOUT='"currency":"USD","confirm":true,"auto_fulfill":true,"connector":["stripe"],"payout_type":"bank","entity_type":"Individual","customer_id":"payee1","payout_method_data":{"bank":{"bank_routing_number":"110000000","bank_account_number":"000123456789","bank_name":"Stripe Test Bank","bank_country_code":"US","bank_city":"San Francisco","account_holder_name":"Jane Doe"}},"billing":{"address":{"line1":"address_full_match","city":"San Francisco","state":"CA","zip":"94105","country":"US","first_name":"Jane","last_name":"Doe"}},"metadata":{"vendor_details":{"account_type":"custom","business_type":"individual","business_profile_mcc":5734,"business_profile_url":"https://hyperswitch.io","capabilities_transfers":true},"individual_details":{"external_account_account_holder_type":"individual"}}'
```

4. Arm the fault on the transfer and make the payout:

```bash
curl -s "http://127.0.0.1:9081/arm?mode=lost&method=POST&path=/v1/transfers"; echo
PO=po_r10_$(date +%s)
hs -X POST $HS/payouts/create -d "{\"payout_id\":\"$PO\",\"amount\":1001,$PAYOUT}" | jq -c '{payout_id,status,amount,connector_transaction_id}'
```

5. Look at both sides:

```bash
curl -s http://127.0.0.1:9081/log | grep -E 'POST /v1/(transfers|payouts)|FAULT|<- 200 (transfer|payout)'   # two POST /v1/transfers, the same body
stripe "/v1/transfers?destination=$ACCT&limit=10" | jq -c '.data[] | {id,amount,transfer_group}'   # Stripe: two transfers of 1001
hs $HS/payouts/$PO | jq -c '{status,amount,connector_transaction_id}'   # Hyperswitch: one payout
```

### Context For The Bug

**Why it matters**: money leaves the merchant twice for one payout, and Hyperswitch shows one.

**Where**: `crates/external_services/src/http_client.rs`, lines 221 to 260: when the connection closes before the answer is complete (`hyper` "connection closed before message completed"), the request is sent again, once, on a new connection. The comment above it says why that is unsafe: "Since hyper already wrote some of the request, it can't really retry it automatically on a new connection, since the server may have acted already" (lines 230 to 238). It came in with #3426 (merged 2024-02-06) as a temporary fix until reqwest used hyper 1.0; the router now uses hyper 1.6 and reqwest 0.13. The Stripe connector (`crates/hyperswitch_connectors/src/connectors/stripe.rs`) sends no `Idempotency-Key` on any request, so Stripe treats the copy as a new request. The transfer is `crates/hyperswitch_connectors/src/connectors/stripe.rs`, lines 3174 to 3245. The same cause as our report "Double charge after a dropped connection to Stripe".

**How the payee was set up**: Hyperswitch creates a payout recipient with Stripe's Accounts v1 API (`POST /v1/accounts`), and our test account refuses that ("Stripe no longer recommends Accounts v1 for new Connect integrations. Create connected accounts with POST /v2/core/accounts instead."). So the payee's account is made with Accounts v2 and registered as the customer's existing recipient, which is what Hyperswitch stores itself after paying a payee once. From the bank account on, everything is Hyperswitch's own flow. On a Stripe platform that still allows Accounts v1, Hyperswitch creates the recipient, and the requests below are the same. (That every payout to a new recipient fails on an Accounts-v2 platform is a separate matter, not reported here.)

**Possible fix**: Do not re-send a request that may have reached the processor (the re-send is only safe when nothing was written), or send an idempotency key that the processor honours, the same on the copy: Stripe's `Idempotency-Key` header (the attempt id for a payment, the refund id for a refund) makes Stripe answer the copy with the first result instead of acting again. For a transfer, the payout attempt id would serve as the key.

GitHub issues are switched off for `juspay/hyperswitch`, so there were no earlier issues to look through. We searched its pull requests for "payout twice", "duplicate transfer", "stripe connect payout" and "idempotency" and found no report of this.

### Environment

Are you using hyperswitch hosted version? No.

Run locally, from the released Docker images:
1. Operating System: macOS (Darwin 25.6.0), Docker Desktop 4.47.0; the containers are Linux arm64
2. Rust version: not built by us. The image `juspaydotin/hyperswitch-router:v1.129.0-standalone` (`sha256:dbbc78845ac1c6dd3cdcc6bd00de11646351b37e2173cca5f32dd723a8021dea`) names Rust 1.99.1 in its build string
3. App version: v1.129.0 (the router logs `v1.129.0-dirty-cf913c5-2026-10-31T11:10:00`)

Connector: Stripe, test mode.

---

Posted to the project: https://github.com/juspay/hyperswitch/discussions/15354
