# Jeepay: Double refund after a slow WeChat Pay refund is closed at two hours

Found via [Cosegan](https://github.com/cosegan) formal/mathematical proof based verification and analysis.

**Effects:** the customer is refunded twice after Jeepay closes a refund that WeChat Pay was still processing at two hours.

中文在前，English below.

**环境 / Environment**

- Jeepay `master` `e1ac9086d679826bab593f20431b7ecd292eb685` (2026-10-01; tag V3.2.9 plus 16 commits), built from source
- `jeepay-payment` on Java 17 (`eclipse-temurin:17-jre`) with Jeepay's own `conf/payment/application.yml`; MySQL 8.0 with `docs/sql/init.sql`; Redis 7; ActiveMQ classic 6.1.4 (`isys.mq.vender=activeMQ`)
- WeChat Pay: `weixin-java-pay` 4.7.2.B as Jeepay bundles it; API V2; normal merchant

**我们只在自己的测试环境中验证过**：Jeepay 网关未做任何修改，运行在 Docker 中；微信支付由我们自己写的一个模拟程序扮演（不是微信支付），包括它的退款结果通知。两个小时我们没有真的等：用一条 SQL 把这笔退款的 `expired_time` 改到一分钟之前，代替等待两小时；这是我们对 Jeepay 数据唯一的直接修改。我们没有在生产环境中见到这个问题，也不知道生产中有多少微信退款在两小时后仍在处理中。

**现象**

微信支付受理退款后才完成退款：申请退款接口的返回"仅代表业务的受理情况"，用银行卡支付的退款可能要几天（见下面引用的文档）。Jeepay 把这样的退款记为状态 1（退款中），每分钟查询一次。但 Jeepay 也给每笔退款设了两小时的过期时间（`RefundOrderController.java` 第 205 行："订单超时关闭时间 默认两个小时"），`RefundOrderExpiredTask` 每分钟把所有过期的、状态 0 或 1 的退款改为状态 4（退款任务关闭），只是一条 UPDATE，不询问渠道（`RefundOrderService.java` 第 130–140 行）。于是：

1. 申请两小时后，仍在微信支付处理中的退款被关闭（状态 4），`RefundOrderReissueTask` 不再查询它（只查状态 1，第 52 行）；
2. 微信支付完成退款并发送通知时，`updateIng2Success` 要求状态为 1（`RefundOrderService.java` 第 82–86 行），没有任何一行被更新，支付订单的 `refund_amount` 不增加，Jeepay 回复微信支付"update status error"（`ChannelRefundNoticeController.java` 第 122–125 行），商户也没有收到退款成功的通知；
3. 支付订单仍显示没有退款，"在途退款"的检查只看状态 1（`RefundOrderController.java` 第 102–104 行），所以 Jeepay 接受同一笔钱的第二次退款，微信支付再退一次。

结果：即使商户不再退款，Jeepay 也把微信支付已经完成的退款记为关闭；商户再退一次的话，顾客收到两笔退款，Jeepay 只记录了一笔。见下面的运行记录。

微信支付的文档（查询退款，更新时间 2025.03.21，https://pay.weixin.qq.com/doc/v2/merchant/4011935218 ）写道："退款有一定延时，用零钱支付的退款20分钟内到账，银行卡支付的退款3个工作日后重新查询退款状态。" 状态说明中有"PROCESSING—退款处理中"。

**预期结果**

渠道仍在处理的退款不会被定时任务在不询问渠道的情况下关闭：它保持状态 1，继续被查询，直到渠道给出成功或失败的结果（或者至少在关闭之前先查询渠道，只有渠道明确失败才关闭）。渠道对一笔退款的成功通知，即使 Jeepay 已把它关闭，也应当被记录下来。支付订单的可退金额要反映渠道实际完成的退款，同一笔钱不能被第二次退款。

**复现步骤**

我们的微信支付模拟程序是一个小型 HTTPS 服务（Java；此处不附，需要的话我们可以提供）。网关容器把 `api.mch.weixin.qq.com` 解析到它，网关的 JVM 信任为它签发证书的测试 CA。它按微信支付 V2 的格式应答（用商户密钥 MD5 签名），可以让退款保持 `PROCESSING`，可以在指定时刻完成退款并按文档格式向 `notify_url` 发送加密的 `req_info` 通知，并记录自己受理的每一笔付款和退款。

1. 按上面的环境启动网关，并像运营人员在后台那样配置商户、应用和微信支付参数（V2，商户 API 证书）。
2. 用 `WX_BAR` 支付 1.80 元并成功。
3. 让模拟程序受理退款但报告 `PROCESSING`；以商户身份调用 `POST /api/refund/refundOrder`，`refundAmount=80`。
4. 等一次分钟任务（Jeepay 查询 `/pay/refundquery`，得到 `PROCESSING`）。
5. 不等两小时：把这一笔退款的 `t_refund_order.expired_time` 改到一分钟之前。
6. 下一次分钟任务后查看这笔退款。
7. 让模拟程序完成这笔退款并发送通知；查看 Jeepay 的回复、`t_refund_order` 和 `t_pay_order`。
8. 以商户身份用新的 `mchRefundNo` 再退款 0.80 元；查看模拟程序的退款记录和 `t_pay_order`。

这次运行的网关日志、数据库记录以及模拟程序的账本和请求日志都在我们手里，需要的话可以提供。

**范围**

- 过期任务和退款通知的处理对所有渠道都一样；我们运行的是微信支付 V2。V3 同样把 `PROCESSING` 映射为 `WAITING`（`WxpayRefundService.java` 第 117–120 行、第 205–207 行），我们读了代码，没有运行。其他渠道的退款会不会超过两小时仍在处理中，我们没有看。
- V3.2.9 发布版、`master`（`e1ac908`）和 `dev`（`3395690`）上的相关代码完全相同；目前所有开着的分支都没有改动这些文件（2026-11-20 核对）。

**验收标准**

- 渠道查询结果为 `PROCESSING` 的状态 1 退款，不会被 `RefundOrderExpiredTask` 改为状态 4（或者：任务在关闭之前先查询渠道，只有渠道明确失败才关闭）。
- 渠道对 Jeepay 已关闭（或尚未完成）的退款发送成功通知时，退款被记为成功（状态 2），支付订单的 `refund_amount` 增加，商户收到通知。
- 渠道完成一笔退款后，同一笔钱的第二次退款被拒绝。
- 有回归测试：一笔过了 `expired_time`、查询结果为 `PROCESSING` 的状态 1 退款，过期任务运行后仍为状态 1。

**可能的修复**（我们没有实现或测试）

不要在没有渠道最终结果的情况下关闭状态 1 的退款：从 `updateOrderExpired` 的条件中去掉状态 1，或者关闭前先查询渠道；并且让 `updateIng2Success` 在渠道报告成功时也能更新状态 4 的退款。

---

**English**

**We verified this on our test setup only**: Jeepay's gateway, unchanged, in Docker, with WeChat Pay, including its refund notice, played by a stand-in program of our own (not WeChat Pay). We did not wait the two hours: one SQL statement moved this refund's `expired_time` one minute into the past instead; that is the only change we made to Jeepay's data directly. We have not seen this in production, and we do not know how many WeChat Pay refunds are still processing two hours after they were asked in production.

**What we are seeing**

WeChat Pay accepts a refund and completes it later: the refund API's answer "仅代表业务的受理情况" (only says the refund was accepted), and a refund of a bank-card payment can take days (documentation quoted below). Jeepay records such a refund in state 1 (退款中) and queries it every minute. But Jeepay also gives every refund an expiry two hours after it was asked (`RefundOrderController.java`, line 205: "订单超时关闭时间 默认两个小时"), and `RefundOrderExpiredTask` moves every expired refund in state 0 or 1 to state 4 (退款任务关闭) every minute, in one UPDATE, without asking the channel (`RefundOrderService.java`, lines 130 to 140). So:

1. two hours after it was asked, a refund WeChat Pay is still processing is closed (state 4), and `RefundOrderReissueTask` stops querying it (it selects state 1 only, line 52);
2. when WeChat Pay completes the refund and sends its notice, `updateIng2Success` requires state 1 (`RefundOrderService.java`, lines 82 to 86): no row changes, the payment's `refund_amount` is not increased, Jeepay answers WeChat Pay "update status error" (`ChannelRefundNoticeController.java`, lines 122 to 125), and the merchant gets no notice of the refund;
3. the payment still shows nothing refunded, and the "refund in progress" check counts state 1 only (`RefundOrderController.java`, lines 102 to 104), so Jeepay accepts a second refund of the same money, and WeChat Pay refunds again.

Result: even if the merchant does nothing more, Jeepay records as closed a refund that WeChat Pay made; if the merchant refunds again, the customer is refunded twice and Jeepay records one refund. See the run below.

WeChat Pay's documentation (查询退款, updated 2025.03.21, https://pay.weixin.qq.com/doc/v2/merchant/4011935218 ) says: "退款有一定延时，用零钱支付的退款20分钟内到账，银行卡支付的退款3个工作日后重新查询退款状态。" (Refunds take time: from the WeChat balance within 20 minutes; for a bank-card payment, query the refund again after 3 working days.) Its list of states includes "PROCESSING—退款处理中".

**What we are expecting**

A refund the channel is still processing is not closed by a timer without asking the channel: it stays in state 1 and keeps being queried until the channel reports success or failure (or, at the least, the channel is queried before the close, and only a definite failure closes it). A channel's success notice for a refund is recorded even if Jeepay had closed the refund. The payment's refundable amount reflects the refunds the channel made, so the same money cannot be refunded twice.

**How to reproduce**

Our stand-in for WeChat Pay is a small HTTPS server of our own (Java; not included here, we can send it). The gateway's container resolves `api.mch.weixin.qq.com` to it, and the gateway's JVM trusts the test certificate authority that signed its certificate. It answers in WeChat Pay's V2 format (MD5-signed with the merchant key), can keep a refund `PROCESSING`, can complete it when told and send the encrypted `req_info` notice to `notify_url` in the documented form, and keeps a ledger of the payments and refunds it has accepted.

1. Start the gateway as above, and configure a merchant, an app and the WeChat Pay interface (V2, with the merchant's API certificate), as an operator does in the console.
2. Pay 1.80 CNY with `WX_BAR`; it succeeds.
3. Tell the stand-in to accept refunds but report them `PROCESSING`; as the merchant, call `POST /api/refund/refundOrder` with `refundAmount=80`.
4. Wait for one minute task (Jeepay queries `/pay/refundquery` and gets `PROCESSING`).
5. Instead of waiting two hours, set this refund's `t_refund_order.expired_time` one minute in the past.
6. After the next minute task, read the refund.
7. Tell the stand-in to complete the refund and send its notice; read Jeepay's answer, `t_refund_order` and `t_pay_order`.
8. As the merchant, refund 0.80 CNY again with a new `mchRefundNo`; read the stand-in's refunds and `t_pay_order`.

We have the gateway's log, the database rows, and the stand-in's ledger and request log of this run, and can send them.

**Scope**

- The expiry task and the refund-notice handling are the same for every channel; we ran WeChat Pay V2. V3 also maps `PROCESSING` to `WAITING` (`WxpayRefundService.java`, lines 117 to 120 and 205 to 207); we read that and did not run it. We did not look at whether other channels' refunds can still be processing after two hours.
- The code is the same in the V3.2.9 release, on `master` (`e1ac908`) and on `dev` (`3395690`); none of the open branches changes these files (checked 2026-11-20).

**Acceptance criteria**

- A refund in state 1 whose channel query says `PROCESSING` is not moved to state 4 by `RefundOrderExpiredTask` (or: the task queries the channel before closing, and closes only on a definite failure).
- When the channel sends a success notice for a refund Jeepay has closed (or not yet finished), the refund is recorded as succeeded (state 2), the payment's `refund_amount` is increased, and the merchant is notified.
- After the channel has completed a refund, a second refund of the same money is refused.
- A regression test: a refund in state 1, past its `expired_time`, whose query says `PROCESSING`, is still in state 1 after the expiry task runs.

**Possible fix** (not built or tested by us)

Do not close refunds in state 1 without a final answer from the channel: drop state 1 from the condition in `updateOrderExpired`, or query the channel before closing; and let `updateIng2Success` also update a refund in state 4 when the channel reports success.

---

**运行记录 / From our run** (times are China Standard Time, as Jeepay logged them)

```text
A WX_BAR payment of 1.80, paid:   pay_order_id P2123924684364361730

07:41:20  POST /api/refund/refundOrder   refundAmount=80
          -> {"code": 0, "data": {"channelOrderNo": "5030000202611205015", "refundOrderId": "R2123924684708294658",
              "state": 1, ...}}   (the stand-in accepted the refund and reports it PROCESSING)
07:42:00  minute task: /pay/refundquery for R2123924684708294658 -> PROCESSING; the refund stays in state 1
          [instead of waiting two hours: this refund's expired_time set one minute in the past, 1 row changed]
07:43:00  minute task: the refund is in state 4; WeChat Pay was not asked
07:43:20  the stand-in completes the refund and sends its notice to /api/refund/notify/wxpay/R2123924684708294658
          Jeepay's answer: HTTP 200, "update status error"
          t_refund_order: still state 4;   t_pay_order: refund_amount 0, refund_state 0
          no request about R2123924684708294658 reached the stand-in after the close
07:43:25  the merchant refunds 80 again with a new mchRefundNo -> state 1; state 2 after the minute task of 07:44:00

The stand-in's refunds on this payment:
    out_refund_no R2123924684708294658   refund_id 5030000202611205015   0.80   SUCCESS (completed 07:43:20)
    out_refund_no R2123925208845299713   refund_id 5030000202611205019   0.80   SUCCESS
t_pay_order at the end:   refund_amount 80, refund_times 1
```

**代码 / Code** (`master` `e1ac908`; the same in V3.2.9 and on `dev`)

https://github.com/jeequan/jeepay/blob/e1ac9086d679826bab593f20431b7ecd292eb685/jeepay-payment/src/main/java/com/jeequan/jeepay/pay/ctrl/refund/RefundOrderController.java#L205

https://github.com/jeequan/jeepay/blob/e1ac9086d679826bab593f20431b7ecd292eb685/jeepay-service/src/main/java/com/jeequan/jeepay/service/impl/RefundOrderService.java#L129-L140

https://github.com/jeequan/jeepay/blob/e1ac9086d679826bab593f20431b7ecd292eb685/jeepay-service/src/main/java/com/jeequan/jeepay/service/impl/RefundOrderService.java#L81-L86

https://github.com/jeequan/jeepay/blob/e1ac9086d679826bab593f20431b7ecd292eb685/jeepay-payment/src/main/java/com/jeequan/jeepay/pay/ctrl/refund/ChannelRefundNoticeController.java#L118-L125
