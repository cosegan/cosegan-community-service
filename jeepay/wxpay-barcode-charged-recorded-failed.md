# Jeepay: WeChat Pay barcode payment taken but recorded as failed after a timeout

Found via [Cosegan](https://github.com/cosegan) formal/mathematical proof based verification and analysis.

**Effects:** the customer is charged while Jeepay records the payment as failed, and nothing corrects it.

中文在前，English below.

**环境 / Environment**

- Jeepay `master` `e1ac9086d679826bab593f20431b7ecd292eb685` (2026-10-01; tag V3.2.9 plus 16 commits), built from source
- `jeepay-payment` on Java 17 (`eclipse-temurin:17-jre`) with Jeepay's own `conf/payment/application.yml`; MySQL 8.0 with `docs/sql/init.sql`; Redis 7; ActiveMQ classic 6.1.4 (`isys.mq.vender=activeMQ`)
- WeChat Pay: `weixin-java-pay` 4.7.2.B as Jeepay bundles it; API V2; normal merchant; way code `WX_BAR`

**我们只在自己的测试环境中验证过**：Jeepay 网关未做任何修改，运行在 Docker 中；微信支付由我们自己写的一个模拟程序扮演（不是微信支付）。我们没有在生产环境中见到这个问题，也不知道真实的微信支付在这种情况下会怎样应答。这次运行说明的是：微信支付已经扣款、而它的应答在 Jeepay 停止等待之后才到达时，Jeepay 怎样记录这笔订单。

**现象**

付款码支付（`WX_BAR`）调用 `/pay/micropay` 时，微信支付当场扣款。微信支付的应答没有按时到达时（读取超时、连接被重置），WxJava 抛出不带错误码的 `WxPayException`。`WxBar.pay` 只把 `SYSTEMERROR`、`USERPAYING`、`BANKERROR` 当作"结果未知、需要查单"，其余所有 `WxPayException` 都当作明确失败（`WxBar.java` 第 99–110 行）。所以：

1. 订单被保存为支付失败（状态 3，`AbstractPayOrderController.java` 第 342–344 行），收银台收到 `orderState` 3、"Read timed out"；
2. 之后不会被查询：只有需要查单时才发补单消息，`PayOrderReissueTask` 只查状态 1；微信支付的付款码支付接口没有 `notify_url` 参数，也不会有支付通知来纠正它；
3. 第 87–89 行的注释说"如果抛异常，则订单状态为： 生成状态……订单将超时关闭"，这只对 `WxPayException` 以外的异常成立。

结果：顾客已经付款，Jeepay 记为支付失败。收银员被告知失败，自然会请顾客再付一次，顾客就被扣了两次钱（第二次付款我们没有对微信支付运行）。见下面的运行记录。同样的请求在没有故障时：`orderState` 2，只扣一次款。

微信支付的文档（付款码支付，更新时间 2026.06.10，https://pay.weixin.qq.com/doc/v2/merchant/4011937125 ）写道："当返回结果为“系统错误”时，商户系统等待5秒后调用【查询订单API】，查询支付实际交易结果；" 以及："在调用查询接口返回后，如果交易状况不明晰，请调用【撤销订单API】，此时如果交易失败则关闭订单，该单不能再支付成功；如果交易成功，则将扣款退回到用户账户。" 开发指引（更新时间 2026.10.22，https://pay.weixin.qq.com/doc/v2/merchant/4011936672 ）写道："当交易超时或支付交易失败，商户收银系统必须调用【撤销订单API】，撤销此交易。"

**预期结果**

不带错误码的 `WxPayException`（即 I/O 失败）和 `SYSTEMERROR` 一样是结果未知：订单进入状态 1（支付中）并查单（`/pay/orderquery`）；多次查询后仍然不明确的，按微信支付文档的要求调用撤销订单（`/secapi/pay/reverse`）。在结果明确之前，不告诉收银台支付失败。

**复现步骤**

我们的微信支付模拟程序是一个小型 HTTPS 服务（Java；此处不附，需要的话我们可以提供）。网关容器把 `api.mch.weixin.qq.com` 解析到它，网关的 JVM 信任为它签发证书的测试 CA。它按微信支付 V2 的格式应答（用商户密钥 MD5 签名），记录自己完成的每一笔扣款，并且可以让指定的某一个请求先执行、再延迟应答。

1. 按上面的环境启动网关，并像运营人员在后台那样配置商户、应用和微信支付参数（V2），开通 `WX_BAR`。
2. 让模拟程序执行下一笔 1.51 元的 `/pay/micropay`（扣款），并在 30 秒后才应答。
3. 以收银台（商户）身份调用 `POST /api/pay/unifiedOrder`：`wayCode=WX_BAR`、`amount=151`、`channelExtra={"authCode":"134567890123456789"}`，以及 `mchNo`、`appId`、`mchOrderNo`、`currency=cny`、`subject`、`body`、`notifyUrl`、`reqTime`、`version=1.0`、`signType=MD5`、`sign`。
4. 应答之后以及其后三次分钟任务之后，查看 `t_pay_order` 中这笔订单、模拟程序收到的带这个 `out_trade_no` 的请求，以及商户收到的通知。

对 Jeepay 来说，关键只在于微信支付的应答在 WxJava 停止等待之后才到达（我们的运行中是 20 秒）。这次运行的网关日志、数据库记录以及模拟程序的账本和请求日志都在我们手里，需要的话可以提供。

**范围**

- 我们运行的是 V2。V3 的付款码支付类（`paywayV3/WxBar.java` 第 35–48 行）直接调用 V2 的这个类，我们读了代码，没有运行。
- V3.2.9 发布版、`master`（`e1ac908`）和 `dev`（`3395690`）上的相关代码完全相同；目前所有开着的分支都没有改动这些文件（2026-11-20 核对）。
- 同一原因也影响微信支付的退款和转账，以及支付宝的条码支付和转账，我们分别另开 issue。

**验收标准**

- `/pay/micropay` 读取超时或连接被重置时，订单为状态 1 并发出补单消息，收银台收到的不是 `orderState` 3。
- 补单查询调用 `/pay/orderquery`；微信支付已经扣款的，最终为状态 2。
- 多次查询后仍然不明确的订单，在关闭之前调用撤销订单。
- 有回归测试：`WxPayService.micropay` 抛出由 `SocketTimeoutException` 引起、不带错误码的 `WxPayException` 时，订单为状态 1。

**可能的修复**（我们没有实现或测试）

在 `WxBar.pay` 的 `catch` 中，把没有错误码的 `WxPayException` 和 `SYSTEMERROR` 一样处理（`WAITING`，需要查单）；补完 `PayOrderReissueMQReceiver` 第 76 行的撤销。

---

**English**

**We verified this on our test setup only**: Jeepay's gateway, unchanged, in Docker, with WeChat Pay played by a stand-in program of our own (not WeChat Pay). We have not seen this in production, and we do not know what the real WeChat Pay answers in this situation. What the run shows is how Jeepay records an order WeChat Pay has charged when WeChat Pay's answer arrives after Jeepay has stopped waiting.

**What we are seeing**

For a barcode payment (`WX_BAR`), WeChat Pay takes the money during `/pay/micropay`. When WeChat Pay's answer does not arrive in time (a read timeout, a reset connection), WxJava throws a `WxPayException` with no error code. `WxBar.pay` treats only `SYSTEMERROR`, `USERPAYING` and `BANKERROR` as "unknown, query the order", and every other `WxPayException` as a definite failure (`WxBar.java`, lines 99 to 110). So:

1. the order is stored as failed (state 3, `AbstractPayOrderController.java`, lines 342 to 344), and the cashier is answered `orderState` 3, "Read timed out";
2. it is never queried: the re-check message is sent only when a query is asked for, and `PayOrderReissueTask` selects state 1 only; WeChat Pay's barcode API has no `notify_url` parameter, so no payment notice comes to correct it either;
3. the comment at lines 87 to 89 says that on an exception "订单状态为： 生成状态……订单将超时关闭" (the order stays "created" and is closed at its expiry), which holds only for exceptions other than `WxPayException`.

Result: the customer has paid and Jeepay records the payment as failed. A cashier told it failed will naturally ask the customer to pay again, which charges them twice (we did not run that second payment for WeChat Pay). See the run below. The same request without the fault: `orderState` 2, one charge.

WeChat Pay's documentation (付款码支付, updated 2026.06.10, https://pay.weixin.qq.com/doc/v2/merchant/4011937125 ) says: "当返回结果为“系统错误”时，商户系统等待5秒后调用【查询订单API】，查询支付实际交易结果；" (on a "system error", wait 5 seconds and query the order for the actual result) and "在调用查询接口返回后，如果交易状况不明晰，请调用【撤销订单API】，此时如果交易失败则关闭订单，该单不能再支付成功；如果交易成功，则将扣款退回到用户账户。" (if the query leaves the outcome unclear, call the reverse API: a failed trade is closed, a successful one is refunded to the customer). Its developer guide (updated 2026.10.22, https://pay.weixin.qq.com/doc/v2/merchant/4011936672 ) says: "当交易超时或支付交易失败，商户收银系统必须调用【撤销订单API】，撤销此交易。" (On a timeout or a failed payment, the cashier system must call the reverse API.)

**What we are expecting**

A `WxPayException` without an error code (an I/O failure) is an unknown outcome, like `SYSTEMERROR`: the order goes to state 1 (支付中) and is queried (`/pay/orderquery`); if it is still unclear after the queries, it is reversed (`/secapi/pay/reverse`), as WeChat Pay's documentation requires. The cashier is not told the payment failed before the outcome is known.

**How to reproduce**

Our stand-in for WeChat Pay is a small HTTPS server of our own (Java; not included here, we can send it). The gateway's container resolves `api.mch.weixin.qq.com` to it, and the gateway's JVM trusts the test certificate authority that signed its certificate. It answers in WeChat Pay's V2 format (MD5-signed with the merchant key), keeps a ledger of the charges it has made, and can be told to carry out one chosen request and answer it late.

1. Start the gateway as above, and configure a merchant, an app and the WeChat Pay interface (V2) with `WX_BAR`, as an operator does in the console.
2. Tell the stand-in to carry out (charge) the next `/pay/micropay` of 1.51 CNY and to answer it only after 30 seconds.
3. As the cashier (the merchant), call `POST /api/pay/unifiedOrder` with `wayCode=WX_BAR`, `amount=151`, `channelExtra={"authCode":"134567890123456789"}`, and `mchNo`, `appId`, `mchOrderNo`, `currency=cny`, `subject`, `body`, `notifyUrl`, `reqTime`, `version=1.0`, `signType=MD5`, `sign`.
4. Read the order in `t_pay_order` after the answer and after each of the next three minute tasks, the stand-in's requests for its `out_trade_no`, and the notices the merchant received.

For Jeepay, all that matters is that WeChat Pay's answer arrives after WxJava stops waiting (20 seconds in our runs). We have the gateway's log, the database rows, and the stand-in's ledger and request log of this run, and can send them.

**Scope**

- We ran V2. The V3 barcode class (`paywayV3/WxBar.java`, lines 35 to 48) calls this V2 class directly; we read that and did not run it.
- The code is the same in the V3.2.9 release, on `master` (`e1ac908`) and on `dev` (`3395690`); none of the open branches changes these files (checked 2026-11-20).
- The same cause reaches WeChat Pay refunds and payouts, and Alipay barcode payments and payouts; we are reporting those separately.

**Acceptance criteria**

- On a read timeout or a reset connection during `/pay/micropay`, the order is in state 1 with a re-check message sent, and the cashier is not answered `orderState` 3.
- The re-check asks `/pay/orderquery`; an order WeChat Pay has charged ends in state 2.
- An order still unclear after the re-checks is reversed before it is closed.
- A regression test: when `WxPayService.micropay` throws a `WxPayException` with no error code, caused by a `SocketTimeoutException`, the order is in state 1.

**Possible fix** (not built or tested by us)

In the `catch` of `WxBar.pay`, treat a `WxPayException` that has no error code like `SYSTEMERROR` (`WAITING`, with a query asked for); and write the reverse at `PayOrderReissueMQReceiver.java`, line 76.

---

**运行记录 / From our run** (times are China Standard Time, as Jeepay logged them)

```text
POST /api/pay/unifiedOrder   wayCode=WX_BAR  amount=151  channelExtra={"authCode":"134567890123456789"}
Jeepay's answer, after 20.1 s:
    {"code": 0, "data": {"errMsg": "Read timed out", "orderState": 3, "payOrderId": "P2123921812717350913", ...},
     "msg": "SUCCESS", ...}
t_pay_order:   state 3   err_msg "Read timed out"   channel_order_no null
               the same at 07:31:20, 07:32:19 and 07:33:20
The stand-in's ledger:   out_trade_no P2123921812717350913   transaction_id 42000020261125005   total_fee 151
                         trade_state SUCCESS   07:29:55
The stand-in's requests for this out_trade_no:
    07:29:55  /pay/micropay   (answered after 30 s; no other request for it afterwards)
Notices to the merchant for this order:   none

Without the fault (1.50, just before): orderState 2, one charge.
```

**代码 / Code** (`master` `e1ac908`; the same in V3.2.9 and on `dev`)

https://github.com/jeequan/jeepay/blob/e1ac9086d679826bab593f20431b7ecd292eb685/jeepay-payment/src/main/java/com/jeequan/jeepay/pay/channel/wxpay/payway/WxBar.java#L87-L110

https://github.com/jeequan/jeepay/blob/e1ac9086d679826bab593f20431b7ecd292eb685/jeepay-payment/src/main/java/com/jeequan/jeepay/pay/ctrl/payorder/AbstractPayOrderController.java#L341-L365

https://github.com/jeequan/jeepay/blob/e1ac9086d679826bab593f20431b7ecd292eb685/jeepay-payment/src/main/java/com/jeequan/jeepay/pay/task/PayOrderReissueTask.java#L54-L55
