# Jeepay: Alipay barcode payment taken but recorded as failed on Alipay's "system busy" and "already paid" answers

Found via [Cosegan](https://github.com/cosegan) formal/mathematical proof based verification and analysis.

**Effects:** the customer is charged while Jeepay records the payment as failed, on Alipay's "system busy" and "already paid" answers.

中文在前，English below.

**环境 / Environment**

- Jeepay `master` `e1ac9086d679826bab593f20431b7ecd292eb685` (2026-10-01; tag V3.2.9 plus 16 commits), built from source
- `jeepay-payment` on Java 17 (`eclipse-temurin:17-jre`) with Jeepay's own `conf/payment/application.yml`; MySQL 8.0 with `docs/sql/init.sql`; Redis 7; ActiveMQ classic 6.1.4 (`isys.mq.vender=activeMQ`)
- Alipay: `alipay-sdk-java` 4.38.61.ALL as Jeepay bundles it; normal merchant; public-key mode; way code `ALI_BAR`

**我们只在自己的测试环境中验证过**：Jeepay 网关未做任何修改，运行在 Docker 中；支付宝由我们自己写的一个模拟程序扮演（不是支付宝，也不是支付宝沙箱）。模拟程序先扣款、再给出下面这些应答。我们没有在生产环境中见到这个问题，也不知道真实的支付宝在扣款之后会多频繁地给出这些应答，以及它对重复发送的同一请求会怎样应答。这次运行说明的是：收到这些应答时，Jeepay 怎样记录这笔订单。

**现象**

`AliBar.pay` 对 `alipay.trade.pay` 应答的判断是：`10000` 成功，`10003` 处理中（"例如等待用户输入密码"），**其余一律为失败**（"其他状态, 表示下单失败"，`AliBar.java` 第 95–109 行）。其中包括表示结果不明确的应答：

- `code` 20000，`sub_code` `isp.unknow-error`（"Service Currently Unavailable"，系统繁忙）；
- `code` 40004，`sub_code` `ACQ.SYSTEM_ERROR`（系统错误）；
- `code` 40004，`sub_code` `ACQ.TRADE_HAS_SUCCESS`（交易已被支付）。

Jeepay 自己的转账代码把支付宝的 `SYSTEM_ERROR` 理解为"若 系统繁忙， 无法确认结果"，并返回 `waiting()`（`AlipayTransferService.java` 第 119–122 行）；条码支付却把它记为失败。订单为状态 3 之后不会再被查询（只有需要查单时才发补单消息，`PayOrderReissueTask` 只查状态 1）。

我们的模拟程序在已经扣款的 `out_trade_no` 再次收到 `alipay.trade.pay` 时应答 `ACQ.TRADE_HAS_SUCCESS`。在我们的运行中，没有任何人重试：模拟程序扣款后断开连接，支付宝 SDK 使用的 Java HTTP 客户端自己把同一个 POST 又发了一次；模拟程序对这个副本应答 `ACQ.TRADE_HAS_SUCCESS`。Jeepay 记为支付失败，错误信息是"Business Failed【交易已被支付】"：一笔失败的订单，错误信息却说已经支付。

结果：顾客已经付款，Jeepay 记为支付失败，之后不再查询。收银员被告知失败，自然会请顾客再付一次（第二次付款我们没有对这三种应答运行）。见下面的运行记录。

**预期结果**

只有明确失败的应答才记为失败。`20000` 和 `ACQ.SYSTEM_ERROR` 是结果未知：订单进入状态 1（支付中），调用 `alipay.trade.query` 查询，仍然不明确的再撤销。`ACQ.TRADE_HAS_SUCCESS` 说明同一个 `out_trade_no` 已经支付：查询 `alipay.trade.query`，`TRADE_SUCCESS` 时记为支付成功。

**复现步骤**

我们的支付宝模拟程序是一个小型 HTTPS 服务（Java；此处不附，需要的话我们可以提供）。网关容器把 `openapi.alipay.com` 解析到它，网关的 JVM 信任为它签发证书的测试 CA。它按支付宝的格式应答 Jeepay 的调用，用测试密钥签名（对应的公钥填在商户的支付宝参数里），记录自己完成的每一笔扣款，对已经扣款的 `out_trade_no` 再次到来的 `alipay.trade.pay` 应答 `ACQ.TRADE_HAS_SUCCESS`，并且可以让指定的某一个请求先扣款、再给出指定的应答或断开连接。

1. 按上面的环境启动网关，并像运营人员在后台那样配置商户、应用和支付宝支付参数，开通 `ALI_BAR`。
2. 让模拟程序对下一笔 1.02 元的 `alipay.trade.pay` 先扣款，再应答 `{"code":"20000","msg":"Service Currently Unavailable","sub_code":"isp.unknow-error","sub_msg":"系统繁忙"}`（带签名）；以收银台（商户）身份调用 `POST /api/pay/unifiedOrder`，`wayCode=ALI_BAR`、`amount=102`、`channelExtra={"authCode":"287654321098765432"}`，以及其他常规参数。
3. 同样，对 1.03 元先扣款，再应答 `{"code":"40004","msg":"Business Failed","sub_code":"ACQ.SYSTEM_ERROR","sub_msg":"系统错误"}`。
4. 对 1.11 元先扣款，再断开连接、不应答。
5. 每一笔都查看接口应答、`t_pay_order`（应答之后以及其后三次分钟任务之后）、模拟程序收到的带这个 `out_trade_no` 的请求，以及它的扣款记录。

这次运行的网关日志、数据库记录以及模拟程序的账本和请求日志都在我们手里，需要的话可以提供。

**范围**

- 我们运行的是条码支付（`AliBar`）。其他支付宝支付方式怎样判断应答，我们没有看。
- V3.2.9 发布版、`master`（`e1ac908`）和 `dev`（`3395690`）上的相关代码完全相同；目前所有开着的分支都没有改动这些文件（2026-11-20 核对）。
- 应答丢失（超时）的情况我们另开了 issue（支付宝条码支付读取超时后重复扣款）。

**验收标准**

- `alipay.trade.pay` 应答 `20000` 或 `ACQ.SYSTEM_ERROR` 时，订单为状态 1 并查单，收银台收到的不是失败。
- 应答 `ACQ.TRADE_HAS_SUCCESS` 时，Jeepay 查询 `alipay.trade.query`，支付宝为 `TRADE_SUCCESS` 时订单记为状态 2。
- 表示支付失败的应答仍然记为失败。
- 对这三种应答各有回归测试。

**可能的修复**（我们没有实现或测试）

在 `AliBar.pay` 中，把 `20000`、`ACQ.SYSTEM_ERROR` 和 `ACQ.TRADE_HAS_SUCCESS` 映射为 `WAITING` 并设置需要查单（`setNeedQuery(true)`），由查单确定订单的最终状态。

---

**English**

**We verified this on our test setup only**: Jeepay's gateway, unchanged, in Docker, with Alipay played by a stand-in program of our own (not Alipay, and not Alipay's sandbox). The stand-in charged first and then gave the answers below. We have not seen this in production, and we do not know how often the real Alipay gives these answers after charging, or what it answers to a second copy of the same request. What the run shows is how Jeepay records the order when it gets these answers.

**What we are seeing**

`AliBar.pay` reads Alipay's answer to `alipay.trade.pay` as: `10000` success, `10003` in progress ("例如等待用户输入密码", e.g. the customer is entering a password), and **everything else a failure** ("其他状态, 表示下单失败", `AliBar.java`, lines 95 to 109). That includes answers that do not say the payment failed:

- `code` 20000, `sub_code` `isp.unknow-error` ("Service Currently Unavailable", 系统繁忙);
- `code` 40004, `sub_code` `ACQ.SYSTEM_ERROR` (系统错误);
- `code` 40004, `sub_code` `ACQ.TRADE_HAS_SUCCESS` (交易已被支付, the trade has been paid).

Jeepay's own payout code reads Alipay's `SYSTEM_ERROR` as "若 系统繁忙， 无法确认结果" (system busy, the result cannot be confirmed) and returns `waiting()` (`AlipayTransferService.java`, lines 119 to 122); the barcode payment records it as a failure. Once in state 3, the order is never queried again (the re-check message is sent only when a query is asked for, and `PayOrderReissueTask` selects state 1 only).

Our stand-in answers `ACQ.TRADE_HAS_SUCCESS` when an `alipay.trade.pay` arrives for an `out_trade_no` it has already charged. In our run nobody retried: the stand-in charged and closed the connection, and Java's HTTP client, which the Alipay SDK uses, sent the same POST again by itself; the stand-in answered the copy with `ACQ.TRADE_HAS_SUCCESS`. Jeepay recorded a failed payment whose error message is "Business Failed【交易已被支付】": a failure whose own message says it has been paid.

Result: the customer has paid, Jeepay records the payment as failed and never asks again. A cashier told it failed will naturally ask the customer to pay again (we did not run that second payment for these three answers). See the run below.

**What we are expecting**

Only an answer that says the payment failed is recorded as failed. `20000` and `ACQ.SYSTEM_ERROR` are unknown outcomes: the order goes to state 1 (支付中), `alipay.trade.query` is asked, and an order still unclear is cancelled. `ACQ.TRADE_HAS_SUCCESS` says the same `out_trade_no` is already paid: `alipay.trade.query` is asked, and the order is recorded as paid on `TRADE_SUCCESS`.

**How to reproduce**

Our stand-in for Alipay is a small HTTPS server of our own (Java; not included here, we can send it). The gateway's container resolves `openapi.alipay.com` to it, and the gateway's JVM trusts the test certificate authority that signed its certificate. It answers Jeepay's calls in Alipay's format, signed with a test key whose public half is in the merchant's Alipay settings, keeps a ledger of the charges it has made, answers `ACQ.TRADE_HAS_SUCCESS` to an `alipay.trade.pay` for an `out_trade_no` it has already charged, and can be told to charge on one chosen request and then give a chosen answer or close the connection.

1. Start the gateway as above, and configure a merchant, an app and the Alipay interface with `ALI_BAR`, as an operator does in the console.
2. Tell the stand-in to charge the next `alipay.trade.pay` of 1.02 CNY and then answer `{"code":"20000","msg":"Service Currently Unavailable","sub_code":"isp.unknow-error","sub_msg":"系统繁忙"}` (signed); as the cashier (the merchant), call `POST /api/pay/unifiedOrder` with `wayCode=ALI_BAR`, `amount=102`, `channelExtra={"authCode":"287654321098765432"}` and the usual parameters.
3. The same for 1.03 CNY with the answer `{"code":"40004","msg":"Business Failed","sub_code":"ACQ.SYSTEM_ERROR","sub_msg":"系统错误"}`.
4. For 1.11 CNY, charge and then close the connection without answering.
5. For each, read the API answer, `t_pay_order` (after the answer and after each of the next three minute tasks), the stand-in's requests for the `out_trade_no`, and its charges.

We have the gateway's log, the database rows, and the stand-in's ledger and request log of this run, and can send them.

**Scope**

- We ran barcode payments (`AliBar`). We did not look at how the other Alipay payment methods read the answer.
- The code is the same in the V3.2.9 release, on `master` (`e1ac908`) and on `dev` (`3395690`); none of the open branches changes these files (checked 2026-11-20).
- The case where the answer is lost (a timeout) we are reporting separately (double charge after an Alipay barcode payment times out).

**Acceptance criteria**

- When `alipay.trade.pay` answers `20000` or `ACQ.SYSTEM_ERROR`, the order is in state 1 and is queried, and the cashier is not answered a failure.
- When it answers `ACQ.TRADE_HAS_SUCCESS`, Jeepay asks `alipay.trade.query`, and the order is in state 2 when Alipay says `TRADE_SUCCESS`.
- Answers that say the payment failed are still recorded as failed.
- A regression test for each of the three answers.

**Possible fix** (not built or tested by us)

In `AliBar.pay`, map `20000`, `ACQ.SYSTEM_ERROR` and `ACQ.TRADE_HAS_SUCCESS` to `WAITING` with a query asked for (`setNeedQuery(true)`), and let the query settle the order.

---

**运行记录 / From our run** (times are China Standard Time, as Jeepay logged them)

```text
1. The stand-in charges 1.02 and answers  {"code":"20000","msg":"Service Currently Unavailable",
                                           "sub_code":"isp.unknow-error","sub_msg":"系统繁忙"}
   Jeepay's answer:  {"code": 0, "data": {"errCode": "isp.unknow-error",
                      "errMsg": "Service Currently Unavailable【系统繁忙】", "orderState": 3,
                      "payOrderId": "P2123923177090887682", ...}, "msg": "SUCCESS", ...}
   t_pay_order: state 3, also at 07:38:00 after three minute tasks
   The stand-in's requests for this out_trade_no: 07:35:20 alipay.trade.pay only (no alipay.trade.query)
   The stand-in's ledger: trade_no 202611202210140001   1.02   TRADE_SUCCESS   07:35:20

2. The stand-in charges 1.03 and answers  {"code":"40004","msg":"Business Failed",
                                           "sub_code":"ACQ.SYSTEM_ERROR","sub_msg":"系统错误"}
   Jeepay's answer:  orderState 3, errCode "ACQ.SYSTEM_ERROR", errMsg "Business Failed【系统错误】"
                     (payOrderId P2123923845545504769)
   t_pay_order: state 3, also at 07:41:00 after three minute tasks; no alipay.trade.query
   The stand-in's ledger: trade_no 202611202210180001   1.03   TRADE_SUCCESS   07:38:00

3. The stand-in charges 1.11 and closes the connection without answering.
   The stand-in's requests for out_trade_no P2123923670500421634:
       07:37:18  alipay.trade.pay   (connection closed after the charge)
       07:37:18  alipay.trade.pay   (the same request again, sent by Java's HTTP client by itself)
   The stand-in's answer to the copy:  {"code":"40004","msg":"Business Failed",
                                        "sub_code":"ACQ.TRADE_HAS_SUCCESS","sub_msg":"交易已被支付"}
   Jeepay's answer:  {"code": 0, "data": {"errCode": "ACQ.TRADE_HAS_SUCCESS",
                      "errMsg": "Business Failed【交易已被支付】", "orderState": 3,
                      "payOrderId": "P2123923670500421634", ...}, "msg": "SUCCESS", ...}
   t_pay_order: state 3 until we stopped watching at 07:42:48; no alipay.trade.query
   The stand-in's ledger: one charge, trade_no 202611202210150001   1.11   TRADE_SUCCESS   07:37:18

No merchant notice for any of the three orders.
```

**代码 / Code** (`master` `e1ac908`; the same in V3.2.9 and on `dev`)

https://github.com/jeequan/jeepay/blob/e1ac9086d679826bab593f20431b7ecd292eb685/jeepay-payment/src/main/java/com/jeequan/jeepay/pay/channel/alipay/payway/AliBar.java#L94-L109

https://github.com/jeequan/jeepay/blob/e1ac9086d679826bab593f20431b7ecd292eb685/jeepay-payment/src/main/java/com/jeequan/jeepay/pay/channel/alipay/AlipayTransferService.java#L117-L122
