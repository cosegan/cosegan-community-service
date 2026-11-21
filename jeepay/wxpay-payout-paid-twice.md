# Jeepay: Payee paid twice after a WeChat Pay payout times out

Found via [Cosegan](https://github.com/cosegan) formal/mathematical proof based verification and analysis.

**Effects:** the payee is paid twice, and the merchant gets a "failed" notice.

中文在前，English below.

**环境 / Environment**

- Jeepay `master` `e1ac9086d679826bab593f20431b7ecd292eb685` (2026-10-01; tag V3.2.9 plus 16 commits), built from source
- `jeepay-payment` on Java 17 (`eclipse-temurin:17-jre`) with Jeepay's own `conf/payment/application.yml`; MySQL 8.0 with `docs/sql/init.sql`; Redis 7; ActiveMQ classic 6.1.4 (`isys.mq.vender=activeMQ`)
- WeChat Pay: `weixin-java-pay` 4.7.2.B as Jeepay bundles it; API V2; normal merchant; `transferVersion` not set, so Jeepay's default, `entPay` (企业付款到零钱)

**我们只在自己的测试环境中验证过**：Jeepay 网关未做任何修改，运行在 Docker 中；微信支付由我们自己写的一个模拟程序扮演（不是微信支付）。我们没有在生产环境中见到这个问题，也不知道真实的微信支付在这种情况下会怎样应答。这次运行说明的是：微信支付已经完成付款、而它的应答在 Jeepay 停止等待之后才到达时，Jeepay 怎样处理这笔转账。

**现象**

WxJava 把 HTTP 调用中的任何异常都包装成不带错误码的 `WxPayException(message, cause)`（V2 的 `post` 和 V3 的 `postV3` 都是如此，我们读了字节码）。`WxpayTransferService.transfer` 的 `catch`（第 155–168 行）只把 `SYSTEMERROR` 当作"处理中"，其余所有 `WxPayException` 都当作明确失败。所以微信支付的应答丢失时，这笔转账：

1. 被保存为失败（状态 3），接口应答 `state` 3、"Read timed out"，并向商户发送状态为 3 的通知（`TransferOrderController.java` 第 208–211 行）；
2. 再也不会被查询：`TransferOrderReissueTask` 只查状态 1（第 52–54 行）；
3. 商户收到失败后用新的 `mchOrderNo` 重试，Jeepay 生成新的 `transferId`，也就是微信支付收到的 `partner_trade_no`（`WxpayTransferService.java` 第 108 行），微信支付再付一次。

结果：收款人收到两笔钱，Jeepay 显示一笔失败、一笔成功。见下面的运行记录。同样的请求在没有故障时只产生一笔转账：状态 1，下一分钟查询后为状态 2。

这一处代码自己的注释就写着应当怎么做："出现未明确的错误码时（SYSTEMERROR等），请务必用原商户订单号重试，或通过查询接口确认此次付款的结果。"（第 157 行）。微信支付的接口文档（付款，更新时间 2025.11.05，https://pay.weixin.qq.com/doc/v2/merchant/4011989673 ）写道："当返回错误码为“SYSTEMERROR”时，请不要更换商户订单号，一定要使用原商户订单号重试（支持在2个自然日内重试），否则可能造成重复支付等资金风险。" 没有收到应答比 `SYSTEMERROR` 更不明确，而 Jeepay 把它记为失败，引导商户换新单号重试，正是文档所说的重复支付。

**预期结果**

不带错误码的 `WxPayException`（即 I/O 失败）是结果未知：转账单进入状态 1（和 `SYSTEMERROR` 一样返回 `waiting()`），由 `TransferOrderReissueTask` 调用 `gettransferinfo` 查询；在结果明确之前，不向商户发送失败通知。

**复现步骤**

我们的微信支付模拟程序是一个小型 HTTPS 服务（Java；此处不附，需要的话我们可以提供）。网关容器把 `api.mch.weixin.qq.com` 解析到它，网关的 JVM 信任为它签发证书的测试 CA。它按微信支付 V2 的格式应答（用商户密钥 MD5 签名），实现了 `/mmpaymkttransfers/promotion/transfers` 和 `/mmpaymkttransfers/gettransferinfo`，记录自己完成的每一笔付款，并且可以让指定的某一个请求先执行、再延迟应答。

1. 按上面的环境启动网关，并像运营人员在后台那样配置商户、应用和微信支付参数（V2，商户 API 证书）。
2. 让模拟程序执行下一笔 2.21 元的 `/mmpaymkttransfers/promotion/transfers`，并在 30 秒后才应答。
3. 以商户身份调用 `POST /api/transferOrder`：`ifCode=wxpay`、`entryType=WX_CASH`、`amount=221`、`accountNo=<收款人 openid>`，以及 `transferDesc`、`notifyUrl`、`mchNo`、`appId`、`mchOrderNo`、`currency=cny`、`reqTime`、`version=1.0`、`signType=MD5`、`sign`。
4. 查看接口应答、商户 `notifyUrl` 收到的通知、`t_transfer_order` 中这一行（应答之后以及其后两次分钟任务之后），以及模拟程序收到的、带这个 `partner_trade_no` 的请求。
5. 以商户身份用新的 `mchOrderNo` 再向同一收款人转账 2.21 元。

对 Jeepay 来说，关键只在于微信支付的应答在 WxJava 停止等待之后才到达（我们的运行中是 20 秒）。这次运行的网关日志、数据库记录以及模拟程序的账本和请求日志都在我们手里，需要的话可以提供。

**范围**

- 我们运行的是 V2 的 `entPay`。V3 的批量转账（`transferBatches`，第 149 行）走同一个 `catch`（第 155–168 行），2025 年新版商家转账（`version202501Transfer`）有一个完全相同的 `catch`（第 239–252 行）；这两条路径我们读了代码，没有运行。
- V3.2.9 发布版、`master`（`e1ac908`）和 `dev`（`3395690`）上的相关代码完全相同；目前所有开着的分支都没有改动这些文件（2026-11-20 核对）。
- 同一原因也影响微信支付的付款和退款，以及支付宝的条码支付和转账，我们分别另开 issue。

**验收标准**

- 调用 `entPay` 时读取超时或连接被重置，转账单为状态 1，接口应答不是状态 3，也不向商户发送失败通知。
- 一分钟内 `TransferOrderReissueTask` 调用 `gettransferinfo` 查询它；微信支付已经付款的，最终为状态 2。
- V3 批量转账和 2025 年新版商家转账同样如此。
- 微信支付确实返回了错误码的应答，照现在的方式处理。
- 有回归测试：`WxPayService` 抛出由 `SocketTimeoutException` 引起、不带错误码的 `WxPayException` 时，转账单为状态 1。

**可能的修复**（我们没有实现或测试）

在这个 `catch` 里，把没有错误码的 `WxPayException` 和 `SYSTEMERROR` 一样处理，返回 `ChannelRetMsg.waiting()`。

---

**English**

**We verified this on our test setup only**: Jeepay's gateway, unchanged, in Docker, with WeChat Pay played by a stand-in program of our own (not WeChat Pay). We have not seen this in production, and we do not know what the real WeChat Pay answers in this situation. What the run shows is how Jeepay handles a payout that WeChat Pay has made when WeChat Pay's answer arrives after Jeepay has stopped waiting.

**What we are seeing**

WxJava wraps any exception of its HTTP call in a `WxPayException(message, cause)` that carries no error code (both V2's `post` and V3's `postV3`; we read the bytecode). The `catch` in `WxpayTransferService.transfer` (lines 155 to 168) treats `SYSTEMERROR` as "in progress" and every other `WxPayException` as a definite failure. So when WeChat Pay's answer is lost, the payout:

1. is stored as failed (state 3); the API answers `state` 3, "Read timed out", and the merchant is sent a notice with state 3 (`TransferOrderController.java`, lines 208 to 211);
2. is never asked about: `TransferOrderReissueTask` selects state 1 only (lines 52 to 54);
3. is paid again when the merchant, told it failed, retries with a new `mchOrderNo`: Jeepay makes a new `transferId`, which is the `partner_trade_no` WeChat Pay receives (`WxpayTransferService.java`, line 108), and WeChat Pay pays again.

Result: the payee is paid twice, and Jeepay shows one payout failed and one succeeded. See the run below. The same request without the fault gives one payout: state 1, then state 2 after the next minute's query.

The code's own comment at that place says what to do: "出现未明确的错误码时（SYSTEMERROR等），请务必用原商户订单号重试，或通过查询接口确认此次付款的结果。" (On an unclear error code such as SYSTEMERROR, retry with the original merchant order number, or confirm the result with the query; line 157.) WeChat Pay's documentation of this API (付款, updated 2025.11.05, https://pay.weixin.qq.com/doc/v2/merchant/4011989673 ) says: "当返回错误码为“SYSTEMERROR”时，请不要更换商户订单号，一定要使用原商户订单号重试（支持在2个自然日内重试），否则可能造成重复支付等资金风险。" (On SYSTEMERROR, do not change the merchant order number; retry with the original one, within 2 days, or there may be duplicate payments.) An answer that never came is less clear than `SYSTEMERROR`; Jeepay records it as failed, which leads the merchant to retry with a new order number: the duplicate payment the documentation warns of.

**What we are expecting**

A `WxPayException` without an error code (an I/O failure) is an unknown outcome: the payout goes to state 1 (`waiting()`, as for `SYSTEMERROR`), `TransferOrderReissueTask` asks `gettransferinfo`, and the merchant is not notified of a failure before the outcome is known.

**How to reproduce**

Our stand-in for WeChat Pay is a small HTTPS server of our own (Java; not included here, we can send it). The gateway's container resolves `api.mch.weixin.qq.com` to it, and the gateway's JVM trusts the test certificate authority that signed its certificate. It answers in WeChat Pay's V2 format (MD5-signed with the merchant key), implements `/mmpaymkttransfers/promotion/transfers` and `/mmpaymkttransfers/gettransferinfo`, keeps a ledger of the payouts it has made, and can be told to carry out one chosen request and answer it late.

1. Start the gateway as above, and configure a merchant, an app and the WeChat Pay interface (V2, with the merchant's API certificate), as an operator does in the console.
2. Tell the stand-in to carry out the next `/mmpaymkttransfers/promotion/transfers` of 2.21 CNY and to answer it only after 30 seconds.
3. As the merchant, call `POST /api/transferOrder` with `ifCode=wxpay`, `entryType=WX_CASH`, `amount=221`, `accountNo=<the payee's openid>`, and `transferDesc`, `notifyUrl`, `mchNo`, `appId`, `mchOrderNo`, `currency=cny`, `reqTime`, `version=1.0`, `signType=MD5`, `sign`.
4. Read the API answer, the notice at the merchant's `notifyUrl`, the payout's row in `t_transfer_order` (after the answer and after the next two minute tasks), and the stand-in's requests for its `partner_trade_no`.
5. As the merchant, pay the same payee 2.21 CNY again with a new `mchOrderNo`.

For Jeepay, all that matters is that WeChat Pay's answer arrives after WxJava stops waiting (20 seconds in our runs). We have the gateway's log, the database rows, and the stand-in's ledger and request log of this run, and can send them.

**Scope**

- We ran V2's `entPay`. V3's batch transfer (`transferBatches`, line 149) goes through the same `catch` (lines 155 to 168), and the 2025 transfer (`version202501Transfer`) has an identical `catch` (lines 239 to 252); we read those two paths and did not run them.
- The code is the same in the V3.2.9 release, on `master` (`e1ac908`) and on `dev` (`3395690`); none of the open branches changes these files (checked 2026-11-20).
- The same cause reaches WeChat Pay payments and refunds, and Alipay barcode payments and payouts; we are reporting those separately.

**Acceptance criteria**

- On a read timeout or a reset connection during `entPay`, the payout is in state 1, the API answer is not state 3, and no failure notice goes to the merchant.
- Within a minute `TransferOrderReissueTask` asks `gettransferinfo` about it; a payout WeChat Pay has made ends in state 2.
- The same holds for V3's batch transfer and for the 2025 transfer.
- An answer that does arrive from WeChat Pay with an error code is handled as today.
- A regression test: when `WxPayService` throws a `WxPayException` with no error code, caused by a `SocketTimeoutException`, the payout is in state 1.

**Possible fix** (not built or tested by us)

In this `catch`, treat a `WxPayException` that has no error code like `SYSTEMERROR`: return `ChannelRetMsg.waiting()`.

---

**运行记录 / From our run** (times are China Standard Time, as Jeepay logged them)

```text
POST /api/transferOrder   ifCode=wxpay  entryType=WX_CASH  amount=221  accountNo=o14openid-payee
Jeepay's answer, after 20.1 s:
    {"code": 0, "data": {"accountNo": "o14openid-payee", "amount": 221, "errMsg": "Read timed out",
     "state": 3, "transferId": "T2123925900263731202", ...}, "msg": "SUCCESS", ...}
Notice to the merchant's notifyUrl, 07:46:29:   state 3, amount 221

t_transfer_order: state 3, err_msg "Read timed out"; the same at 07:48:20, after two minute tasks

The stand-in's requests for partner_trade_no T2123925900263731202:
    07:46:09  /mmpaymkttransfers/promotion/transfers   (answered after 30 s; no /mmpaymkttransfers/gettransferinfo afterwards)

The merchant pays the same payee again with a new mchOrderNo:
    state 1; state 2 after the minute task of 07:47:00

The stand-in's payouts to o14openid-payee for these two payout orders:
    partner_trade_no T2123925900263731202   payment_no 10100000202611205003   2.21   SUCCESS   07:46:09
    partner_trade_no T2123925984443412482   payment_no 10100000202611205005   2.21   SUCCESS   07:46:30

Without the fault (2.20, just before): state 1, /mmpaymkttransfers/gettransferinfo at 07:47:00, state 2, one payout.
```

**代码 / Code** (`master` `e1ac908`; the same in V3.2.9 and on `dev`)

https://github.com/jeequan/jeepay/blob/e1ac9086d679826bab593f20431b7ecd292eb685/jeepay-payment/src/main/java/com/jeequan/jeepay/pay/channel/wxpay/WxpayTransferService.java#L155-L168

https://github.com/jeequan/jeepay/blob/e1ac9086d679826bab593f20431b7ecd292eb685/jeepay-payment/src/main/java/com/jeequan/jeepay/pay/ctrl/transfer/TransferOrderController.java#L207-L211

https://github.com/jeequan/jeepay/blob/e1ac9086d679826bab593f20431b7ecd292eb685/jeepay-payment/src/main/java/com/jeequan/jeepay/pay/task/TransferOrderReissueTask.java#L51-L54
