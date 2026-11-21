# IJPay: Double charge and double refund when a PayPal order or refund is retried

Found via [Cosegan](https://github.com/cosegan) formal/mathematical proof based verification and analysis.

**Effects:** a PayPal card order that a merchant sends again after a lost answer is charged a second time, and a partial refund sent again is paid a second time; IJPay's order, capture and refund calls give the merchant no way to make the retry safe.

### 版本信息 (Versions)

- IJPay 版本 (IJPay version): IJPay-PayPal 3.0.0 (Maven Central), with IJPay-Core 3.0.0 and Hutool 5.8.48
- 开发环境 (Environment): Java 21 (Eclipse Temurin), Linux in Docker
- 支付方式 (Payment method): PayPal REST API (Orders v2, Payments v2), sandbox
- 调用的接口 (Methods called): `PayPalApi.createOrder`, `PayPalApi.refund`. `PayPalApi.captureOrder` builds its headers the same way; we did not run it.

**We verified this on PayPal's sandbox only**: IJPay-PayPal 3.0.0 from Maven Central, Java 21, PayPal's REST sandbox. We have not seen it in production. The lines named below are the same on `master` (`84e08d2`), which differs from the tag `v3.0.0` only in `packages/docs/`.

### 报错信息（注意格式化）(Output)

Nothing is thrown. This is the output of the program under "How to reproduce", from our run on 2026-11-20 (PayPal sandbox ids):

```text
createOrder: HTTP 201, id 8CM25003JR793704W, COMPLETED, capture 7TU111712C423674B of 14.00 USD
createOrder again: HTTP 201, id 08U58705656708628, COMPLETED, capture 5U4997801H3283337 of 14.00 USD
refund: HTTP 201, id 14Y246744N042120C, COMPLETED
refund again: HTTP 201, id 5VH95491HJ7955828, COMPLETED
PayPal holds refund 14Y246744N042120C, COMPLETED, 4.00 USD, of capture 7TU111712C423674B
PayPal holds refund 5VH95491HJ7955828, COMPLETED, 4.00 USD, of capture 7TU111712C423674B
createOrder, fixed id: HTTP 201, id 09084780S1075462P, COMPLETED, capture 7UW31309EV710053N of 14.00 USD
createOrder again, same id: HTTP 200, id 09084780S1075462P, COMPLETED, capture 7UW31309EV710053N of 14.00 USD
refund, fixed id: HTTP 201, id 7503280860616250D, COMPLETED
refund again, same id: HTTP 200, id 7503280860616250D, COMPLETED
```

The first two lines are two orders, each captured: the card was charged 14.00 twice. The next four are two refunds of 4.00 of the first capture.

### 问题描述（包括回显步骤、截图）(Description and steps to reproduce)

**What we are seeing**

When a merchant sends the same PayPal order or refund a second time through IJPay, PayPal carries it out a second time:

- **An order paid by card** (`payment_source.card`, which PayPal captures when the order is created) charges the card again, as a second order. If the answer to the first call was lost, the merchant has no record of the first order.
- **A partial refund** is refunded again.
- **A full refund** sent again is refused with 422 `CAPTURE_FULLY_REFUNDED`, so a merchant whose first answer was lost reads a failure for a refund that was made.

A merchant sends a call again when the first answer did not arrive: a timeout, a dropped connection, or a 5xx after PayPal had already acted. IJPay does not send anything twice by itself; the second call is the merchant's.

PayPal makes that retry safe with the `PayPal-Request-Id` header. Its guide (https://developer.paypal.com/reference/guidelines/idempotency/) says: "You can retry idempotent calls that fail with network timeouts or HTTP 5xx status codes for as long as the server stores the ID", and "when you include a previously specified PayPal-Request-Id header in a request, PayPal returns the latest status of the previous request that used that same header". For a create-order call that carries a card, PayPal's Orders v2 API specification describes the header as "mandatory for all single-step create order calls".

IJPay's `createOrder`, `captureOrder` and `refund` take no request id. Each builds its headers with `getBaseHeaders(accessToken)` (`IJPay-PayPal/src/main/java/com/ijpay/paypal/PayPalApi.java`, lines 201 to 203), which puts a new random id on every call (`PayKit.generateStr()`, a UUID):

```java
public static Map<String, String> getBaseHeaders(AccessToken accessToken) {
    return getBaseHeaders(accessToken, PayKit.generateStr(), null, null);
}
```

So the second call always carries a new id, and PayPal treats it as a new request. With one fixed `PayPal-Request-Id`, the same pairs of calls give one order and one refund: PayPal answers the second call with HTTP 200 and the first result (the last four lines of the output).

**How to reproduce**

Needs Java 11 or later, a PayPal sandbox REST app (client id and secret) whose sandbox business account accepts card payments, and in `lib/` IJPay-PayPal 3.0.0 with its dependencies plus an SLF4J implementation, since IJPay-Core declares `slf4j-api` as `provided` (for example from `mvn dependency:copy-dependencies -DoutputDirectory=lib` in a project that declares `com.github.javen205:IJPay-PayPal:3.0.0` and `org.slf4j:slf4j-simple`).

```bash
export PAYPAL_CLIENT_ID=...   # the sandbox app's client id
export PAYPAL_SECRET=...      # and its secret
java -cp "lib/*" RetryRepro.java
```

`RetryRepro.java`:

```java
import cn.hutool.json.JSONArray;
import cn.hutool.json.JSONObject;
import cn.hutool.json.JSONUtil;
import com.ijpay.core.IJPayHttpResponse;
import com.ijpay.paypal.PayPalApi;
import com.ijpay.paypal.PayPalApiConfig;
import com.ijpay.paypal.PayPalApiConfigKit;
import com.ijpay.paypal.accesstoken.AccessTokenKit;
import com.ijpay.paypal.enums.PayPalApiUrl;
import java.util.Map;
import java.util.UUID;

public class RetryRepro {
    public static void main(String[] args) {
        PayPalApiConfig config = new PayPalApiConfig();
        config.setClientId(System.getenv("PAYPAL_CLIENT_ID"));
        config.setSecret(System.getenv("PAYPAL_SECRET"));
        config.setSandBox(true);
        PayPalApiConfigKit.putApiConfig(config);

        // An order paid by card (PayPal captures it when it is created) and a partial refund.
        String order = "{\"intent\":\"CAPTURE\",\"purchase_units\":[{\"amount\":{\"currency_code\":\"USD\",\"value\":\"14.00\"}}],"
            + "\"payment_source\":{\"card\":{\"number\":\"4111111111111111\",\"expiry\":\"2030-12\",\"security_code\":\"123\",\"name\":\"Test Buyer\"}}}";
        String refund = "{\"amount\":{\"currency_code\":\"USD\",\"value\":\"4.00\"}}";

        // 1. The same order sent twice, as after a lost answer.
        JSONObject first = show("createOrder", PayPalApi.createOrder(config, order));
        show("createOrder again", PayPalApi.createOrder(config, order));

        // 2. The same refund sent twice, then what PayPal holds for the first order.
        String capture = (String) first.getByPath("purchase_units[0].payments.captures[0].id");
        show("refund", PayPalApi.refund(config, capture, refund));
        show("refund again", PayPalApi.refund(config, capture, refund));
        JSONObject held = JSONUtil.parseObj(PayPalApi.queryOrder(config, first.getStr("id")).getBody());
        for (Object o : (JSONArray) held.getByPath("purchase_units[0].payments.refunds")) {
            JSONObject r = (JSONObject) o;
            System.out.println("PayPal holds refund " + r.getStr("id") + ", " + r.getStr("status") + ", "
                + r.getByPath("amount.value") + " USD, of capture " + capture);
        }

        // 3. The same calls, each pair sent with one fixed PayPal-Request-Id.
        String orderRequestId = UUID.randomUUID().toString();
        String ordersUrl = PayPalApi.getReqUrl(PayPalApiUrl.CHECKOUT_ORDERS, true);
        JSONObject third = show("createOrder, fixed id", PayPalApi.post(ordersUrl, order, headers(config, orderRequestId)));
        show("createOrder again, same id", PayPalApi.post(ordersUrl, order, headers(config, orderRequestId)));
        String refundRequestId = UUID.randomUUID().toString();
        String refundUrl = String.format(PayPalApi.getReqUrl(PayPalApiUrl.REFUND, true),
            (String) third.getByPath("purchase_units[0].payments.captures[0].id"));
        show("refund, fixed id", PayPalApi.post(refundUrl, refund, headers(config, refundRequestId)));
        show("refund again, same id", PayPalApi.post(refundUrl, refund, headers(config, refundRequestId)));
    }

    static Map<String, String> headers(PayPalApiConfig config, String payPalRequestId) {
        return PayPalApi.getBaseHeaders(AccessTokenKit.get(config.getClientId()), payPalRequestId, null, null);
    }

    // Prints the HTTP status, the id and status PayPal returned, and an order's capture.
    static JSONObject show(String call, IJPayHttpResponse response) {
        JSONObject o = JSONUtil.parseObj(response.getBody());
        Object capture = o.getByPath("purchase_units[0].payments.captures[0].id");
        System.out.println(call + ": HTTP " + response.getStatus() + ", id " + o.getStr("id") + ", " + o.getStr("status")
            + (capture == null ? "" : ", capture " + capture + " of "
                + o.getByPath("purchase_units[0].payments.captures[0].amount.value") + " USD"));
        return o;
    }
}
```

Each "again" is the same IJPay call with the same arguments, made at once. To see the same with a lost answer, we also ran the order and the refund with a small proxy of our own between IJPay and PayPal's sandbox (not included here; we can send it). It passed the first call on to PayPal and then answered IJPay with HTTP 500. The merchant got 500 and then 201, and PayPal held two captured orders of 13.04 for one purchase, or two refunds of 3.00 on one capture. With one fixed request id, PayPal answered the second call with HTTP 200 and the first order or refund.

### 预期效果描述 (Expected result)

Sending the same order, capture or refund again after a lost answer does not charge or refund again. For that, the caller has to be able to choose the `PayPal-Request-Id` and send the same one on the retry, as PayPal's guide describes.

**Possible fix**

Let the caller pass the request id, for example `createOrder(config, data, payPalRequestId)`, `captureOrder(config, id, data, payPalRequestId)` and `refund(config, captureId, data, payPalRequestId)`, and say in the documentation that a retry must reuse it (the merchant's own order or refund number, for example). Until then a merchant can send its own id through `PayPalApi.post(url, data, PayPalApi.getBaseHeaders(accessToken, payPalRequestId, null, null))`, as the last part of the program does. We have not built or tested a changed IJPay.

---

Posted to the project: https://github.com/Javen205/IJPay/issues/95
