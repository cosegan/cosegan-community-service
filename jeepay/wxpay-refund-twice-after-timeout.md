# Jeepay: Double refund after a WeChat Pay refund times out

Found via [Cosegan](https://github.com/cosegan) formal/mathematical proof based verification and analysis.

**Effects:** the customer is refunded twice, and the merchant is told the first refund failed.

中文在前，English below.

**环境 / Environment**

- Jeepay `master` `e1ac9086d679826bab593f20431b7ecd292eb685` (2026-10-01; tag V3.2.9 plus 16 commits), built from source
- `jeepay-payment` on Java 17 (`eclipse-temurin:17-jre`) with Jeepay's own `conf/payment/application.yml`; MySQL 8.0 with `docs/sql/init.sql`; Redis 7; ActiveMQ classic 6.1.4 (`isys.mq.vender=activeMQ`)
- WeChat Pay: `weixin-java-pay` 4.7.2.B as Jeepay bundles it; API V2 (`refundV2`, which calls `/secapi/pay/refundv2`); normal merchant

**我们只在自己的测试环境中验证过**：Jeepay 网关未做任何修改，运行在 Docker 中；微信支付由我们自己写的一个模拟程序扮演（不是微信支付）。我们没有在生产环境中见到这个问题，也不知道真实的微信支付在这种情况下会怎样应答。这次运行说明的是：微信支付已经受理退款、而它的应答在 Jeepay 停止等待之后才到达时，Jeepay 怎样处理这笔退款。

**现象**

WxJava 把 HTTP 调用中的任何异常都包装成不带错误码的 `WxPayException`。`WxpayRefundService.refund` 把所有 `WxPayException` 都当作明确失败（第 128–132 行：`ChannelRetMsg.confirmFail()`）。所以微信支付的应答丢失时，这笔退款：

1. 被保存为失败（状态 3），接口应答 `state` 3、"Read timed out"，并向商户发送状态为 3 的通知（`RefundOrderController.java` 第 230–233 行）；
2. 再也不会被查询：`RefundOrderReissueTask` 只查状态 1（第 52 行）；
3. 支付订单的 `refund_amount` 没有变化，所以同样金额的第二次退款可以通过 Jeepay 的检查（第 94–114 行）。原商户退款单号已被占用（唯一键 `Uni_MchNo_MchRefundNo`），重试只能用新单号；Jeepay 生成新的 `refundOrderId`，也就是微信支付收到的 `out_refund_no`，微信支付再退一次。

结果：顾客收到两笔退款，Jeepay 只记录了一笔。见下面的运行记录。同样的请求在没有故障时：状态 1，下一分钟查询后为状态 2，只有一笔退款。

微信支付的接口文档（申请退款，更新时间 2025.03.21，https://pay.weixin.qq.com/doc/v2/merchant/4011935217 ）对 `SYSTEMERROR`（"接口返回错误"，原因"系统超时等"）写道："请不要更换商户退款单号，请使用相同参数再次调用API。" 并且："申请退款接口的返回仅代表业务的受理情况，具体退款是否成功，需要通过退款查询接口获取结果。"

**预期结果**

应答丢失的退款应当按结果未知处理：进入状态 1（退款中），由 `RefundOrderReissueTask` 调用 `/pay/refundquery` 查询；在结果明确之前，不告诉商户退款失败。退款在状态 1 时，同一支付订单的第二次退款会被已有的检查拒绝（"支付订单具有在途退款申请，请稍后再试"，第 102–104 行）。

作为对比：支付宝退款在 Gitee issue ICWR2G 之后会捕获所有异常并返回 `waiting()`（"等待补单"，`AlipayRefundService.java` 第 87–90 行）。我们对支付宝退款注入同样的故障，结果是状态 1、下一分钟被查询、然后状态 2。

**复现步骤**

我们的微信支付模拟程序是一个小型 HTTPS 服务（Java；此处不附，需要的话我们可以提供）。网关容器把 `api.mch.weixin.qq.com` 解析到它，网关的 JVM 信任为它签发证书的测试 CA。它按微信支付 V2 的格式应答（用商户密钥 MD5 签名），记录自己受理的每一笔付款和退款，并且可以让指定的某一个请求先执行、再延迟应答。

1. 按上面的环境启动网关，并像运营人员在后台那样配置商户、应用和微信支付参数（V2，商户 API 证书）。
2. 用 `WX_BAR` 支付 1.70 元并成功。
3. 让模拟程序受理下一笔 0.70 元的 `/secapi/pay/refundv2`，并在 30 秒后才应答。
4. 以商户身份调用 `POST /api/refund/refundOrder`：`payOrderId`、`refundAmount=70`，以及 `mchRefundNo`、`refundReason`、`notifyUrl`、`currency=cny`、`mchNo`、`appId`、`reqTime`、`version=1.0`、`signType=MD5`、`sign`。
5. 收到失败后，用新的 `mchRefundNo` 再退款 0.70 元。
6. 查看两次接口应答、商户收到的通知、`t_refund_order` 和 `t_pay_order`，以及模拟程序收到的、带第一笔 `out_refund_no` 的请求和它的退款记录。

对 Jeepay 来说，关键只在于微信支付的应答在 WxJava 停止等待之后才到达（我们的运行中是 20 秒）。这次运行的网关日志、数据库记录以及模拟程序的账本和请求日志都在我们手里，需要的话可以提供。

**范围**

- 我们运行的是 V2（`refundV2`）。同一方法的 V3 分支由同一个 `catch` 处理；WxJava 的 V3 请求（`postV3`）同样把 I/O 异常包装成不带错误码的 `WxPayException`（字节码）。V3 我们没有运行。
- V3.2.9 发布版、`master`（`e1ac908`）和 `dev`（`3395690`）上的相关代码完全相同；目前所有开着的分支都没有改动这些文件（2026-11-20 核对）。
- 同一原因也影响微信支付的付款和转账，以及支付宝的条码支付和转账，我们分别另开 issue。

**验收标准**

- 调用 `refundV2`/`refundV3` 时读取超时或连接被重置，退款单为状态 1，接口应答不是状态 3，也不向商户发送失败通知。
- 一分钟内 `RefundOrderReissueTask` 调用 `/pay/refundquery` 查询它；微信支付已经退款的，最终为状态 2，支付订单的 `refund_amount` 相应增加。
- 退款处于状态 1 时，同一支付订单的第二次退款被拒绝。
- 微信支付确实返回了错误码的应答，照现在的方式处理。
- 有回归测试：`WxPayService` 抛出由 `SocketTimeoutException` 引起、不带错误码的 `WxPayException` 时，退款单为状态 1。

**可能的修复**（我们没有实现或测试）

在 `WxpayRefundService.refund` 的 `catch (WxPayException e)` 中，没有错误码的异常返回 `ChannelRetMsg.waiting()`，与 `AlipayRefundService.refund` 相同。

---

**English**

**We verified this on our test setup only**: Jeepay's gateway, unchanged, in Docker, with WeChat Pay played by a stand-in program of our own (not WeChat Pay). We have not seen this in production, and we do not know what the real WeChat Pay answers in this situation. What the run shows is how Jeepay handles a refund that WeChat Pay has accepted when WeChat Pay's answer arrives after Jeepay has stopped waiting.

**What we are seeing**

WxJava wraps any exception of its HTTP call in a `WxPayException` that carries no error code. `WxpayRefundService.refund` treats every `WxPayException` as a definite failure (lines 128 to 132: `ChannelRetMsg.confirmFail()`). So when WeChat Pay's answer is lost, the refund:

1. is stored as failed (state 3); the API answers `state` 3, "Read timed out", and the merchant is sent a notice with state 3 (`RefundOrderController.java`, lines 230 to 233);
2. is never asked about: `RefundOrderReissueTask` selects state 1 only (line 52);
3. leaves the payment's `refund_amount` unchanged, so a second refund of the same amount passes Jeepay's checks (lines 94 to 114). The merchant's refund number is taken (unique key `Uni_MchNo_MchRefundNo`), so the retry carries a new one; Jeepay makes a new `refundOrderId`, which is the `out_refund_no` WeChat Pay receives, and WeChat Pay refunds again.

Result: the customer is refunded twice, and Jeepay records one refund. See the run below. The same request without the fault: state 1, state 2 after the next minute's query, one refund.

WeChat Pay's documentation of this API (申请退款, updated 2025.03.21, https://pay.weixin.qq.com/doc/v2/merchant/4011935217 ) says, for `SYSTEMERROR` ("接口返回错误", cause "系统超时等", a timeout and the like): "请不要更换商户退款单号，请使用相同参数再次调用API。" (Do not change the merchant refund number; call the API again with the same parameters.) And: "申请退款接口的返回仅代表业务的受理情况，具体退款是否成功，需要通过退款查询接口获取结果。" (The refund API's answer only says whether the refund was accepted; whether it succeeded must be read from the refund query.)

**What we are expecting**

A refund whose answer was lost is treated as unknown: it goes to state 1 (退款中), `RefundOrderReissueTask` asks `/pay/refundquery`, and the merchant is not told it failed before the outcome is known. While the refund is in state 1, a second refund of the same payment is refused by the check that already exists ("支付订单具有在途退款申请，请稍后再试", lines 102 to 104).

For comparison: since Gitee issue ICWR2G, Alipay refunds catch every exception and return `waiting()` ("等待补单", `AlipayRefundService.java`, lines 87 to 90). Our run of an Alipay refund with the same fault: state 1, queried at the next minute, then state 2.

**How to reproduce**

Our stand-in for WeChat Pay is a small HTTPS server of our own (Java; not included here, we can send it). The gateway's container resolves `api.mch.weixin.qq.com` to it, and the gateway's JVM trusts the test certificate authority that signed its certificate. It answers in WeChat Pay's V2 format (MD5-signed with the merchant key), keeps a ledger of the payments and refunds it has accepted, and can be told to carry out one chosen request and answer it late.

1. Start the gateway as above, and configure a merchant, an app and the WeChat Pay interface (V2, with the merchant's API certificate), as an operator does in the console.
2. Pay 1.70 CNY with `WX_BAR`; it succeeds.
3. Tell the stand-in to accept the next `/secapi/pay/refundv2` of 0.70 CNY and to answer it only after 30 seconds.
4. As the merchant, call `POST /api/refund/refundOrder` with `payOrderId`, `refundAmount=70`, and `mchRefundNo`, `refundReason`, `notifyUrl`, `currency=cny`, `mchNo`, `appId`, `reqTime`, `version=1.0`, `signType=MD5`, `sign`.
5. Answered "failed", refund 0.70 CNY again with a new `mchRefundNo`.
6. Read both API answers, the notices the merchant received, `t_refund_order` and `t_pay_order`, and the stand-in's requests for the first `out_refund_no` and its refunds.

For Jeepay, all that matters is that WeChat Pay's answer arrives after WxJava stops waiting (20 seconds in our runs). We have the gateway's log, the database rows, and the stand-in's ledger and request log of this run, and can send them.

**Scope**

- We ran V2 (`refundV2`). The V3 branch of the same method is handled by the same `catch`, and WxJava's V3 request (`postV3`) wraps an I/O exception in a `WxPayException` without an error code in the same way (bytecode). We did not run V3.
- The code is the same in the V3.2.9 release, on `master` (`e1ac908`) and on `dev` (`3395690`); none of the open branches changes these files (checked 2026-11-20).
- The same cause reaches WeChat Pay payments and payouts, and Alipay barcode payments and payouts; we are reporting those separately.

**Acceptance criteria**

- On a read timeout or a reset connection during `refundV2` or `refundV3`, the refund is in state 1, the API answer is not state 3, and no failure notice goes to the merchant.
- Within a minute `RefundOrderReissueTask` asks `/pay/refundquery` about it; a refund WeChat Pay has made ends in state 2, and the payment's `refund_amount` is increased.
- While the refund is in state 1, a second refund of the same payment is refused.
- An answer that does arrive from WeChat Pay with an error code is handled as today.
- A regression test: when `WxPayService` throws a `WxPayException` with no error code, caused by a `SocketTimeoutException`, the refund is in state 1.

**Possible fix** (not built or tested by us)

In `catch (WxPayException e)` of `WxpayRefundService.refund`, return `ChannelRetMsg.waiting()` for an exception without an error code, as `AlipayRefundService.refund` does.

---

**运行记录 / From our run** (times are China Standard Time, as Jeepay logged them)

```text
A WX_BAR payment of 1.70, paid:   pay_order_id P2123924181081436162

POST /api/refund/refundOrder   payOrderId=P2123924181081436162  refundAmount=70
Jeepay's answer, after 20.1 s:
    {"code": 0, "data": {"errMsg": "Read timed out", "payAmount": 170, "refundAmount": 70,
     "refundOrderId": "R2123924181463117825", "state": 3, ...}, "msg": "SUCCESS", ...}
Notice to the merchant's notifyUrl, 07:39:40:   state 3, amount 70
t_pay_order after it:   refund_amount 0, refund_state 0

The merchant refunds 70 again with a new mchRefundNo:
    {"code": 0, "data": {"channelOrderNo": "5030000202611205010", "refundOrderId": "R2123924265575690242",
     "state": 1, ...}}   then state 2 after the minute task of 07:40:00

The stand-in's requests for out_refund_no R2123924181463117825:
    07:39:20  /secapi/pay/refundv2   (answered after 30 s; no /pay/refundquery afterwards)
(the second refund was queried at 07:40:00, as every refund in state 1 is)

The stand-in's refunds on this payment:
    out_refund_no R2123924181463117825   refund_id 5030000202611205008   0.70   SUCCESS
    out_refund_no R2123924265575690242   refund_id 5030000202611205010   0.70   SUCCESS
t_pay_order at the end:   refund_amount 70, refund_times 1

Without the fault (a refund of 0.60 on another payment, just before): state 1, /pay/refundquery at the
next minute, state 2, one refund.
```

**代码 / Code** (`master` `e1ac908`; the same in V3.2.9 and on `dev`)

https://github.com/jeequan/jeepay/blob/e1ac9086d679826bab593f20431b7ecd292eb685/jeepay-payment/src/main/java/com/jeequan/jeepay/pay/channel/wxpay/WxpayRefundService.java#L128-L137

https://github.com/jeequan/jeepay/blob/e1ac9086d679826bab593f20431b7ecd292eb685/jeepay-payment/src/main/java/com/jeequan/jeepay/pay/ctrl/refund/RefundOrderController.java#L229-L233

https://github.com/jeequan/jeepay/blob/e1ac9086d679826bab593f20431b7ecd292eb685/jeepay-payment/src/main/java/com/jeequan/jeepay/pay/task/RefundOrderReissueTask.java#L51-L52

https://github.com/jeequan/jeepay/blob/e1ac9086d679826bab593f20431b7ecd292eb685/jeepay-payment/src/main/java/com/jeequan/jeepay/pay/channel/alipay/AlipayRefundService.java#L87-L90
