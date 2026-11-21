# Jeepay: Double charge after an Alipay barcode payment times out

Found via [Cosegan](https://github.com/cosegan) formal/mathematical proof based verification and analysis.

**Effects:** the customer is charged twice, and the first charge cannot be refunded through Jeepay.

中文在前，English below.

**环境 / Environment**

- Jeepay `master` `e1ac9086d679826bab593f20431b7ecd292eb685` (2026-10-01; tag V3.2.9 plus 16 commits), built from source
- `jeepay-payment` on Java 17 (`eclipse-temurin:17-jre`) with Jeepay's own `conf/payment/application.yml`; MySQL 8.0 with `docs/sql/init.sql`; Redis 7; ActiveMQ classic 6.1.4 (`isys.mq.vender=activeMQ`)
- Alipay: `alipay-sdk-java` 4.38.61.ALL as Jeepay bundles it; normal merchant; public-key mode; way code `ALI_BAR`

**我们只在自己的测试环境中验证过**：Jeepay 网关未做任何修改，运行在 Docker 中；支付宝由我们自己写的一个模拟程序扮演（不是支付宝，也不是支付宝沙箱）。我们没有在生产环境中见到这个问题，也不知道真实的支付宝在这种情况下会怎样应答。这次运行说明的是：支付宝已经扣款、而它的应答在 Jeepay 停止等待之后才到达时，Jeepay 怎样处理这笔订单。

**现象**

条码支付（`ALI_BAR`）调用 `alipay.trade.pay` 时，支付宝当场扣款。支付宝的应答没有按时到达时，`AlipayClientWrapper.execute` 把异常（无论是什么异常）都转成 `ChannelException.sysError(...)`（第 65–74 行；`AliBar.java` 第 79 行的注释："如果异常， 将直接跑出 ChannelException"）。之后这笔订单：

1. 停留在状态 0（订单生成）：`SYS_ERROR` 的处理是"订单不再处理。 为： 生成状态"（`AbstractPayOrderController.java` 第 354–356 行），收银台收到的应答是 `code` 9999，"系统：java.net.SocketTimeoutException: Read timed out"（第 215–222 行）；
2. 不会被查询：只有需要查单时才发补单消息（第 362–365 行），`PayOrderReissueTask` 只查状态 1（`PayOrderReissueTask.java` 第 55 行）；
3. 到了过期时间（默认两小时，第 275 行）被关闭（状态 6，订单关闭），`updateOrderExpired` 把状态 0 和 1 的过期订单直接关闭，不询问支付宝（`PayOrderService.java` 第 191–201 行）；
4. 关闭之后不能在 Jeepay 里退款：`RefundOrderController.java` 第 90–92 行拒绝所有不是状态 2 的订单（"订单状态不正确， 无法完成退款"），所以这笔钱只能在 Jeepay 之外退还。

收银员被告知支付失败，自然会用新订单再收一次款：顾客被扣了两次钱，Jeepay 显示一笔支付成功、一笔订单关闭。见下面的运行记录。同样的请求在没有故障时：状态 2，只扣一次款。

**预期结果**

应答丢失的条码支付应当按"结果未知"处理：进入状态 1（支付中），通过补单消息调用 `alipay.trade.query` 查询；多次查询后仍然不明确的，调用 `alipay.trade.cancel` 撤销（`PayOrderReissueMQReceiver.java` 第 76 行已有 `//TODO 调用【撤销订单】接口`）。在结果明确之前，不告诉收银台支付失败。这也是 Jeepay 自己的定义：`ChannelRetMsg.ChannelState.UNKNOWN` 是"接口超时，或网络异常等请求， 或者返回结果的签名失败： 状态不明确"（`ChannelRetMsg.java` 第 67 行），`processChannelMsg` 会把 `UNKNOWN` 置为状态 1（`AbstractPayOrderController.java` 第 346–352 行）；而 `SYS_ERROR` 是"本系统出现不可预知的异常"（第 69 行），支付宝一侧的读取超时并不属于这种情况。

**复现步骤**

我们的支付宝模拟程序是一个小型 HTTPS 服务（Java；此处不附，需要的话我们可以提供）。网关容器把 `openapi.alipay.com` 解析到它，网关的 JVM 信任为它签发证书的测试 CA。它按支付宝的格式应答 Jeepay 的调用，用测试密钥签名（对应的公钥填在商户的支付宝参数里），记录自己完成的每一笔扣款，并且可以让指定的某一个请求先执行、再延迟应答。

1. 按上面的环境启动网关，并像运营人员在后台那样配置商户、应用和支付宝支付参数，开通 `ALI_BAR`。
2. 让模拟程序执行下一笔 1.01 元的 `alipay.trade.pay`（扣款），并在 30 秒后才应答。
3. 以收银台（商户）身份调用 `POST /api/pay/unifiedOrder`：`wayCode=ALI_BAR`、`amount=101`、`channelExtra={"authCode":"287654321098765432"}`、`expiredTime=180`（商户参数，单位秒，只是为了不等默认的两小时），以及 `mchNo`、`appId`、`mchOrderNo`、`currency=cny`、`subject`、`body`、`notifyUrl`、`reqTime`、`version=1.0`、`signType=MD5`、`sign`。
4. 收到失败后，立即用新的 `mchOrderNo` 为同一件商品再收一次款（没有故障）。
5. 每 30 秒查看第一笔订单在 `t_pay_order` 中的状态，直到过期时间之后；查看模拟程序收到的、带这个 `out_trade_no` 的请求，以及它的扣款记录。
6. 以商户身份申请退还第一笔订单（`POST /api/refund/refundOrder`，`refundAmount=101`）。

对 Jeepay 来说，关键只在于支付宝的应答在支付宝客户端停止等待之后才到达（我们的运行中是 15 秒）。这次运行的网关日志、数据库记录以及模拟程序的账本和请求日志都在我们手里，需要的话可以提供。

**范围**

- 所有经过 `AlipayClientWrapper` 的支付宝调用都会把异常变成 `SYS_ERROR`。我们运行的是条码支付，支付宝在调用中当场扣款；其他支付宝支付方式走同一个 wrapper，我们没有运行。
- V3.2.9 发布版、`master`（`e1ac908`）和 `dev`（`3395690`）上的相关代码完全相同；目前所有开着的分支都没有改动这些文件（2026-11-20 核对）。
- 相关：Gitee issue ICWR2G（2025-09-07）报告过支付宝退款的同一问题，维护者回复"有道理， 应该异常做补单处理"，现在 `AlipayRefundService.refund` 捕获异常并返回 `waiting()`（"等待补单"，第 87–90 行）。我们对支付宝退款注入同样的故障，结果是状态 1、下一分钟被查询、然后状态 2，说明那个修复有效。条码支付没有做同样的修改。
- 同一原因也影响支付宝转账，以及微信支付的付款、退款和转账，我们分别另开 issue。

**验收标准**

- `alipay.trade.pay` 在请求可能已经到达支付宝之后抛出异常（读取超时、连接被重置、应答验签失败）时，订单保存为状态 1 而不是 0，收银台收到的应答不表示失败。
- 补单消息调用 `alipay.trade.query` 查询它；支付宝已经扣款的，最终为状态 2。
- 多次查询后仍然不明确的订单，在关闭之前调用 `alipay.trade.cancel` 撤销。
- 支付宝已经扣款的订单，不会在不询问支付宝的情况下被关闭。
- 有回归测试：支付宝客户端抛出由 `SocketTimeoutException` 引起的 `AlipayApiException` 时，订单为状态 1，并发出了补单消息。

**可能的修复**（我们没有实现或测试）

在 `AlipayClientWrapper` 中区分请求发出之前的失败（可以视为"未发出"）和发出之后的失败（`UNKNOWN`，并设置需要查单）；补完 `PayOrderReissueMQReceiver` 中的撤销。

---

**English**

**We verified this on our test setup only**: Jeepay's gateway, unchanged, in Docker, with Alipay played by a stand-in program of our own (not Alipay, and not Alipay's sandbox). We have not seen this in production, and we do not know what the real Alipay answers in this situation. What the run shows is how Jeepay handles an order Alipay has charged when Alipay's answer arrives after Jeepay has stopped waiting.

**What we are seeing**

For a barcode payment (`ALI_BAR`), Alipay takes the money during `alipay.trade.pay`. When Alipay's answer does not arrive in time, `AlipayClientWrapper.execute` turns the exception, whatever it is, into `ChannelException.sysError(...)` (lines 65 to 74; the comment at `AliBar.java`, line 79: "如果异常， 将直接跑出 ChannelException", on an exception a ChannelException is thrown). The order then:

1. stays in state 0 (订单生成): for `SYS_ERROR` the code says "订单不再处理。 为： 生成状态" (the order is no longer processed; it stays "created"; `AbstractPayOrderController.java`, lines 354 to 356), and the cashier is answered `code` 9999, "系统：java.net.SocketTimeoutException: Read timed out" (lines 215 to 222);
2. is not queried: the re-check message is sent only when a query is asked for (lines 362 to 365), and `PayOrderReissueTask` selects state 1 only (`PayOrderReissueTask.java`, line 55);
3. is closed at its expiry (two hours by default, line 275) as state 6 (订单关闭): `updateOrderExpired` closes expired orders in state 0 or 1 without asking Alipay (`PayOrderService.java`, lines 191 to 201);
4. cannot be refunded through Jeepay after that: `RefundOrderController.java`, lines 90 to 92, refuses any order not in state 2 ("订单状态不正确， 无法完成退款"), so the money can only be returned outside Jeepay.

The cashier, told the payment failed, naturally charges again with a new order: the customer is charged twice, and Jeepay shows one order paid and one closed. See the run below. The same request without the fault: state 2, one charge.

**What we are expecting**

A barcode payment whose answer was lost is treated as unknown: it goes to state 1 (支付中), the re-check message asks `alipay.trade.query`, and an order still unclear after the re-checks is cancelled with `alipay.trade.cancel` (`PayOrderReissueMQReceiver.java`, line 76, already has `//TODO 调用【撤销订单】接口`, call the cancel API). The cashier is not told the payment failed before the outcome is known. That is Jeepay's own definition: `ChannelRetMsg.ChannelState.UNKNOWN` is "接口超时，或网络异常等请求， 或者返回结果的签名失败： 状态不明确" (a timeout, a network error, or an answer whose signature fails: the state is unclear; `ChannelRetMsg.java`, line 67), and `processChannelMsg` puts `UNKNOWN` in state 1 (`AbstractPayOrderController.java`, lines 346 to 352). `SYS_ERROR` is "本系统出现不可预知的异常" (an unforeseeable error in this system, line 69), which a timeout on Alipay's side is not.

**How to reproduce**

Our stand-in for Alipay is a small HTTPS server of our own (Java; not included here, we can send it). The gateway's container resolves `openapi.alipay.com` to it, and the gateway's JVM trusts the test certificate authority that signed its certificate. It answers Jeepay's calls in Alipay's format, signed with a test key whose public half is in the merchant's Alipay settings, keeps a ledger of the charges it has made, and can be told to carry out one chosen request and answer it late.

1. Start the gateway as above, and configure a merchant, an app and the Alipay interface with `ALI_BAR`, as an operator does in the console.
2. Tell the stand-in to carry out (charge) the next `alipay.trade.pay` of 1.01 CNY and to answer it only after 30 seconds.
3. As the cashier (the merchant), call `POST /api/pay/unifiedOrder` with `wayCode=ALI_BAR`, `amount=101`, `channelExtra={"authCode":"287654321098765432"}`, `expiredTime=180` (a merchant parameter, in seconds, only so as not to wait the default two hours), and `mchNo`, `appId`, `mchOrderNo`, `currency=cny`, `subject`, `body`, `notifyUrl`, `reqTime`, `version=1.0`, `signType=MD5`, `sign`.
4. Answered "failed", charge the same goods again at once with a new `mchOrderNo` (no fault).
5. Read the first order's state in `t_pay_order` every 30 seconds until after its expiry; read the stand-in's requests for its `out_trade_no` and its charges.
6. As the merchant, ask for a refund of the first order (`POST /api/refund/refundOrder`, `refundAmount=101`).

For Jeepay, all that matters is that Alipay's answer arrives after the Alipay client stops waiting (15 seconds in our runs). We have the gateway's log, the database rows, and the stand-in's ledger and request log of this run, and can send them.

**Scope**

- Every Alipay call through `AlipayClientWrapper` turns an exception into `SYS_ERROR`. We ran barcode payments, where Alipay takes the money during the call; the other Alipay payment methods go through the same wrapper, and we did not run them.
- The code is the same in the V3.2.9 release, on `master` (`e1ac908`) and on `dev` (`3395690`); none of the open branches changes these files (checked 2026-11-20).
- Related: Gitee issue ICWR2G (2025-09-07) reported the same thing for Alipay refunds. The maintainer answered "有道理， 应该异常做补单处理" (makes sense; on an exception, re-check the order), and `AlipayRefundService.refund` now catches the exception and returns `waiting()` ("等待补单", lines 87 to 90). Our run of an Alipay refund with the same fault shows that fix working: state 1, queried at the next minute, then state 2. Barcode payments did not get the change.
- The same cause reaches Alipay payouts and WeChat Pay payments, refunds and payouts; we are reporting those separately.

**Acceptance criteria**

- When `alipay.trade.pay` throws after the request may have reached Alipay (a read timeout, a reset connection, an answer whose signature check fails), the order is stored in state 1, not 0, and the cashier's answer does not say it failed.
- The re-check message asks `alipay.trade.query` about it; an order Alipay has charged ends in state 2.
- An order still unclear after the re-checks is cancelled with `alipay.trade.cancel` before it is closed.
- An order Alipay has charged is never closed without asking Alipay.
- A regression test: when the Alipay client throws an `AlipayApiException` caused by a `SocketTimeoutException`, the order is in state 1 and a re-check message is sent.

**Possible fix** (not built or tested by us)

In `AlipayClientWrapper`, tell a failure before the request is sent (safe to call "not sent") from one after it (`UNKNOWN`, with a query asked for); and write the cancel in `PayOrderReissueMQReceiver`.

---

**运行记录 / From our run** (times are China Standard Time, as Jeepay logged them)

```text
POST /api/pay/unifiedOrder   wayCode=ALI_BAR  amount=101  expiredTime=180  channelExtra={"authCode":"287654321098765432"}
Jeepay's answer, after 15.1 s:
    {"code": 9999, "msg": "系统：java.net.SocketTimeoutException: Read timed out"}
t_pay_order:   pay_order_id P2123921812625076225   state 0   channel_order_no null   expired_time 07:32:55
The stand-in's ledger:   trade_no 202611202210080001   1.01   buyer 208866180802   TRADE_SUCCESS   07:29:55

At once, the cashier charges the same goods again with a new mchOrderNo:
    {"code": 0, "data": {"orderState": 2, "payOrderId": "P2123921875782905857", ...}, "msg": "SUCCESS", ...}
The stand-in's charges for the two orders:
    trade_no 202611202210080001   1.01   buyer 208866180802   07:29:55   (first order)
    trade_no 202611202210110001   1.01   buyer 208866180802   07:30:10   (second order)

First order, read every 30 s:   state 0 until expired_time; state 6 from the minute task of 07:33:00
The stand-in's requests for out_trade_no P2123921812625076225:
    07:29:55  alipay.trade.pay   (answered after 30 s; no alipay.trade.query or other request afterwards)

07:37:09  the merchant asks Jeepay to refund the first order (101):
    {"code": 9999, "msg": "订单状态不正确， 无法完成退款"}   (no refund row; nothing sent to Alipay)

Without the fault (1.00, just before): orderState 2, one charge.
```

**代码 / Code** (`master` `e1ac908`; the same in V3.2.9 and on `dev`)

https://github.com/jeequan/jeepay/blob/e1ac9086d679826bab593f20431b7ecd292eb685/jeepay-payment/src/main/java/com/jeequan/jeepay/pay/model/AlipayClientWrapper.java#L65-L74

https://github.com/jeequan/jeepay/blob/e1ac9086d679826bab593f20431b7ecd292eb685/jeepay-payment/src/main/java/com/jeequan/jeepay/pay/ctrl/payorder/AbstractPayOrderController.java#L346-L365

https://github.com/jeequan/jeepay/blob/e1ac9086d679826bab593f20431b7ecd292eb685/jeepay-service/src/main/java/com/jeequan/jeepay/service/impl/PayOrderService.java#L190-L201

https://github.com/jeequan/jeepay/blob/e1ac9086d679826bab593f20431b7ecd292eb685/jeepay-payment/src/main/java/com/jeequan/jeepay/pay/mq/PayOrderReissueMQReceiver.java#L71-L77
