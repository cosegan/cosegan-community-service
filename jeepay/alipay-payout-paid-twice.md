# Jeepay: Payee paid twice after an Alipay payout times out

Found via [Cosegan](https://github.com/cosegan) formal/mathematical proof based verification and analysis.

**Effects:** the payee is paid twice, and Jeepay leaves the payout "created" for ever and tells the merchant it failed.

中文在前，English below.

**环境 / Environment**

- Jeepay `master` `e1ac9086d679826bab593f20431b7ecd292eb685` (2026-10-01; tag V3.2.9 plus 16 commits), built from source
- `jeepay-payment` on Java 17 (`eclipse-temurin:17-jre`) with Jeepay's own `conf/payment/application.yml`; MySQL 8.0 with `docs/sql/init.sql`; Redis 7; ActiveMQ classic 6.1.4 (`isys.mq.vender=activeMQ`)
- Alipay: `alipay-sdk-java` 4.38.61.ALL as Jeepay bundles it; normal merchant; public-key mode

**我们只在自己的测试环境中验证过**：Jeepay 网关未做任何修改，运行在 Docker 中；支付宝由我们自己写的一个模拟程序扮演（不是支付宝，也不是支付宝沙箱）。我们没有在生产环境中见到这个问题，也不知道真实的支付宝在这种情况下会怎样应答。这次运行说明的是：支付宝已经完成转账、而它的应答在 Jeepay 停止等待之后才到达时，Jeepay 怎样处理这笔转账。

**现象**

支付宝对 `alipay.fund.trans.uni.transfer` 的应答没有按时到达时，`AlipayClientWrapper.execute` 把异常（无论是什么异常）都转成 `ChannelException.sysError(...)`（第 65–74 行）。之后这笔转账：

1. 停留在状态 0（订单生成）：`SYS_ERROR` 的处理是"订单不再处理。 为： 生成状态"（`TransferOrderController.java` 第 221–223 行），商户收到的应答是 `code` 9999，"系统：java.net.SocketTimeoutException: Read timed out"（第 141–148 行）；
2. 再也不会被查询：`TransferOrderReissueTask` 只查状态 1（第 52–54 行），转账单也没有超时关闭任务，所以这一行一直是状态 0；
3. 商户被告知失败、再次转账时，收款人会再收到一笔：原商户订单号已被占用（唯一键 `Uni_MchNo_MchOrderNo`），重试只能用新单号；Jeepay 生成新的 `transferId`，也就是支付宝收到的 `out_biz_no`（`AlipayTransferService.java` 第 78 行），支付宝无从知道这是重复的转账。

结果：收款人收到两笔钱，Jeepay 显示一笔转账成功、一笔"订单生成"。见下面的运行记录。同样的请求在没有故障时只产生一笔转账，状态 2。

**预期结果**

应答丢失的转账应当按"结果未知"处理：进入状态 1（转账中），由 `TransferOrderReissueTask` 调用 `alipay.fund.trans.common.query` 查询；在 Jeepay 不知道结果的时候，不告诉商户转账失败。这也是 Jeepay 自己的定义：`ChannelRetMsg.ChannelState.UNKNOWN` 是"接口超时，或网络异常等请求， 或者返回结果的签名失败： 状态不明确"（`ChannelRetMsg.java` 第 67 行），`processChannelMsg` 会把 `UNKNOWN` 置为状态 1（`TransferOrderController.java` 第 213–219 行）；而 `SYS_ERROR` 是"本系统出现不可预知的异常"（第 69 行），支付宝一侧的读取超时并不属于这种情况。

**复现步骤**

我们的支付宝模拟程序是一个小型 HTTPS 服务（Java；此处不附，需要的话我们可以提供）。网关容器把 `openapi.alipay.com` 解析到它，网关的 JVM 信任为它签发证书的测试 CA。它按支付宝的格式应答 Jeepay 的调用，用测试密钥签名（对应的公钥填在商户的支付宝参数里），记录自己完成的每一笔转账，并且可以让指定的某一个请求先执行、再延迟应答。

1. 按上面的环境启动网关，并像运营人员在后台那样配置商户、应用和支付宝支付参数。
2. 让模拟程序执行下一笔 1.21 元的 `alipay.fund.trans.uni.transfer`，并在 30 秒后才应答。
3. 以商户身份调用 `POST /api/transferOrder`：`ifCode=alipay`、`entryType=ALIPAY_CASH`、`amount=121`、`accountNo=payee@example.com`，以及 `accountName`、`transferDesc`、`notifyUrl`、`mchNo`、`appId`、`mchOrderNo`、`currency=cny`、`reqTime`、`version=1.0`、`signType=MD5`、`sign`。
4. 应答之后以及其后三次分钟任务之后，查看 `t_transfer_order` 中这一行，以及模拟程序收到的、带这个 `out_biz_no` 的请求。
5. 以商户身份用新的 `mchOrderNo` 再向同一收款人转账 1.21 元。

对 Jeepay 来说，关键只在于支付宝的应答在支付宝客户端停止等待之后才到达（我们的运行中是 15 秒）。这次运行的网关日志、数据库记录以及模拟程序的账本和请求日志都在我们手里，需要的话可以提供。

**范围**

- 所有经过 `AlipayClientWrapper` 的支付宝调用都会把异常变成 `SYS_ERROR`。本 issue 只涉及转账（`AlipayTransferService.java` 第 97 行的调用不在 `try` 中）。
- V3.2.9 发布版、`master`（`e1ac908`）和 `dev`（`3395690`）上的相关代码完全相同；目前所有开着的分支都没有改动这些文件（2026-11-20 核对）。
- 相关：Gitee issue ICWR2G（2025-09-07）报告过支付宝退款的同一问题，维护者回复"有道理， 应该异常做补单处理"，现在 `AlipayRefundService.refund` 捕获异常并返回 `waiting()`（"等待补单"，第 87–90 行）。我们对支付宝退款注入同样的故障，结果是状态 1、下一分钟被查询、然后状态 2，说明那个修复有效。转账没有做同样的修改。
- 同一原因也影响支付宝条码支付，以及微信支付的付款、退款和转账，我们分别另开 issue。

**验收标准**

- `alipay.fund.trans.uni.transfer` 在请求可能已经到达支付宝之后抛出异常（读取超时、连接被重置、应答验签失败）时，转账单保存为状态 1 而不是 0，给商户的应答不表示失败。
- 一分钟内 `TransferOrderReissueTask` 调用 `alipay.fund.trans.common.query` 查询它；支付宝已经付款的，最终为状态 2。
- 请求发出之前的失败（例如私钥错误、签名失败）仍可按失败处理，因为请求并没有发出。
- 有回归测试：支付宝客户端抛出由 `SocketTimeoutException` 引起的 `AlipayApiException` 时，转账单停在状态 1。

**可能的修复**（我们没有实现或测试）

在 `AlipayClientWrapper` 中区分请求发出之前的失败（可以视为"未发出"）和发出之后的失败（`UNKNOWN`）；或者像 ICWR2G 之后的 `AlipayRefundService.refund` 那样，在 `AlipayTransferService.transfer` 中捕获异常并返回 `ChannelRetMsg.waiting()`。

---

**English**

**We verified this on our test setup only**: Jeepay's gateway, unchanged, in Docker, with Alipay played by a stand-in program of our own (not Alipay, and not Alipay's sandbox). We have not seen this in production, and we do not know what the real Alipay answers in this situation. What the run shows is how Jeepay handles a payout that Alipay has carried out when Alipay's answer arrives after Jeepay has stopped waiting.

**What we are seeing**

When Alipay's answer to `alipay.fund.trans.uni.transfer` does not arrive in time, `AlipayClientWrapper.execute` turns the exception, whatever it is, into `ChannelException.sysError(...)` (lines 65 to 74). The payout then:

1. stays in state 0 (订单生成): for `SYS_ERROR` the code says "订单不再处理。 为： 生成状态" (the order is no longer processed; it stays "created"; `TransferOrderController.java`, lines 221 to 223), and the merchant is answered `code` 9999, "系统：java.net.SocketTimeoutException: Read timed out" (lines 141 to 148);
2. is never asked about: `TransferOrderReissueTask` selects state 1 only (lines 52 to 54), and payouts have no expiry task, so the row stays in state 0;
3. is paid a second time when the merchant, told it failed, pays again: the merchant's order number is taken (unique key `Uni_MchNo_MchOrderNo`), so the retry carries a new one; Jeepay makes a new `transferId`, which is the `out_biz_no` Alipay receives (`AlipayTransferService.java`, line 78), and Alipay cannot know it is a repeat.

Result: the payee is paid twice, and Jeepay shows one payout succeeded and one still "created". See the run below. The same request without the fault gives one payout, state 2.

**What we are expecting**

A payout whose answer was lost is treated as unknown: it goes to state 1 (转账中), `TransferOrderReissueTask` asks `alipay.fund.trans.common.query`, and the merchant is not told it failed while Jeepay does not know. That is Jeepay's own definition: `ChannelRetMsg.ChannelState.UNKNOWN` is "接口超时，或网络异常等请求， 或者返回结果的签名失败： 状态不明确" (a timeout, a network error, or an answer whose signature fails: the state is unclear; `ChannelRetMsg.java`, line 67), and `processChannelMsg` puts `UNKNOWN` in state 1 (`TransferOrderController.java`, lines 213 to 219). `SYS_ERROR` is "本系统出现不可预知的异常" (an unforeseeable error in this system, line 69), which a timeout on Alipay's side is not.

**How to reproduce**

Our stand-in for Alipay is a small HTTPS server of our own (Java; not included here, we can send it). The gateway's container resolves `openapi.alipay.com` to it, and the gateway's JVM trusts the test certificate authority that signed its certificate. It answers Jeepay's calls in Alipay's format, signed with a test key whose public half is in the merchant's Alipay settings, keeps a ledger of the payouts it has made, and can be told to carry out one chosen request and answer it late.

1. Start the gateway as above, and configure a merchant, an app and the Alipay interface, as an operator does in the console.
2. Tell the stand-in to carry out the next `alipay.fund.trans.uni.transfer` of 1.21 CNY and to answer it only after 30 seconds.
3. As the merchant, call `POST /api/transferOrder` with `ifCode=alipay`, `entryType=ALIPAY_CASH`, `amount=121`, `accountNo=payee@example.com`, and `accountName`, `transferDesc`, `notifyUrl`, `mchNo`, `appId`, `mchOrderNo`, `currency=cny`, `reqTime`, `version=1.0`, `signType=MD5`, `sign`.
4. Read the payout's row in `t_transfer_order` after the answer and after each of the next three minute tasks, and the stand-in's requests for its `out_biz_no`.
5. As the merchant, pay the same payee 1.21 CNY again with a new `mchOrderNo`.

For Jeepay, all that matters is that Alipay's answer arrives after the Alipay client stops waiting (15 seconds in our runs). We have the gateway's log, the database rows, and the stand-in's ledger and request log of this run, and can send them.

**Scope**

- Every Alipay call through `AlipayClientWrapper` turns an exception into `SYS_ERROR`. This issue is about payouts (`AlipayTransferService.java`, line 97: the call is not inside a `try`).
- The code is the same in the V3.2.9 release, on `master` (`e1ac908`) and on `dev` (`3395690`); none of the open branches changes these files (checked 2026-11-20).
- Related: Gitee issue ICWR2G (2025-09-07) reported the same thing for Alipay refunds. The maintainer answered "有道理， 应该异常做补单处理" (makes sense; on an exception, re-check the order), and `AlipayRefundService.refund` now catches the exception and returns `waiting()` ("等待补单", lines 87 to 90). Our run of an Alipay refund with the same fault shows that fix working: state 1, queried at the next minute, then state 2. Payouts did not get the change.
- The same cause reaches Alipay barcode payments and WeChat Pay payments, refunds and payouts; we are reporting those separately.

**Acceptance criteria**

- When `alipay.fund.trans.uni.transfer` throws after the request may have reached Alipay (a read timeout, a reset connection, an answer whose signature check fails), the payout is stored in state 1, not 0, and the merchant's answer does not say it failed.
- Within a minute `TransferOrderReissueTask` asks `alipay.fund.trans.common.query` about it; a payout Alipay has paid ends in state 2.
- A failure before the request leaves Jeepay (a bad private key, a signing error) may still be a failure, since nothing was sent.
- A regression test: when the Alipay client throws an `AlipayApiException` caused by a `SocketTimeoutException`, the payout stays in state 1.

**Possible fix** (not built or tested by us)

In `AlipayClientWrapper`, tell a failure before the request is sent (safe to call "not sent") from one after it (`UNKNOWN`); or catch the exception in `AlipayTransferService.transfer` and return `ChannelRetMsg.waiting()`, as `AlipayRefundService.refund` has done since ICWR2G.

---

**运行记录 / From our run** (times are China Standard Time, as Jeepay logged them)

```text
POST /api/transferOrder   ifCode=alipay  entryType=ALIPAY_CASH  amount=121  accountNo=payee@example.com
Jeepay's answer, after 15.0 s:
    {"code": 9999, "msg": "系统：java.net.SocketTimeoutException: Read timed out"}

t_transfer_order, just after the answer and at 07:31:20, 07:32:19 and 07:33:20:
    transfer_id T2123921779003535361   state 0   channel_order_no null   updated_at 07:29:47.331 (unchanged)

The stand-in's requests for out_biz_no T2123921779003535361:
    07:29:47  alipay.fund.trans.uni.transfer   (answered after 30 s; no alipay.fund.trans.common.query
                                                afterwards, up to 07:52 when we stopped)

The merchant pays the same payee again with a new mchOrderNo:
    {"code": 0, "data": {..., "state": 2, "transferId": "T2123921842102644737"}}

The stand-in's payouts to payee@example.com:
    out_biz_no T2123921779003535361   order_id 20261120110701006   1.21   SUCCESS   07:29:47
    out_biz_no T2123921842102644737   order_id 20261120110701009   1.21   SUCCESS   07:30:02
```

**代码 / Code** (`master` `e1ac908`; the same in V3.2.9 and on `dev`)

https://github.com/jeequan/jeepay/blob/e1ac9086d679826bab593f20431b7ecd292eb685/jeepay-payment/src/main/java/com/jeequan/jeepay/pay/model/AlipayClientWrapper.java#L65-L74

https://github.com/jeequan/jeepay/blob/e1ac9086d679826bab593f20431b7ecd292eb685/jeepay-payment/src/main/java/com/jeequan/jeepay/pay/rqrs/msg/ChannelRetMsg.java#L63-L70

https://github.com/jeequan/jeepay/blob/e1ac9086d679826bab593f20431b7ecd292eb685/jeepay-payment/src/main/java/com/jeequan/jeepay/pay/ctrl/transfer/TransferOrderController.java#L213-L223

https://github.com/jeequan/jeepay/blob/e1ac9086d679826bab593f20431b7ecd292eb685/jeepay-payment/src/main/java/com/jeequan/jeepay/pay/task/TransferOrderReissueTask.java#L51-L54

https://github.com/jeequan/jeepay/blob/e1ac9086d679826bab593f20431b7ecd292eb685/jeepay-payment/src/main/java/com/jeequan/jeepay/pay/channel/alipay/AlipayRefundService.java#L87-L90
