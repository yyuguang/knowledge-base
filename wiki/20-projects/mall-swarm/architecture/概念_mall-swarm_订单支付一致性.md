---
type: "concept"
tags: ["ecommerce", "mall-swarm", "order", "payment", "consistency", "security"]
summary: "P0 订单一致性与支付安全复核：事务代理、库存并发、RabbitMQ 延迟取消、支付宝回调、前后台权限与 AI 工具闸门。"
sources:
  - "[[30-sources/repositories/mall-swarm/来源_mall-swarm_项目源码]]"
  - "/Users/yangyuguang/Documents/project_code/mall/mall-swarm/mall-portal/**"
  - "/Users/yangyuguang/Documents/project_code/mall/mall-swarm/mall-admin/**"
  - "/Users/yangyuguang/Documents/project_code/mall/mall-swarm/mall-gateway/**"
  - "/Users/yangyuguang/Documents/project_code/mall/mall-swarm/document/sql/mall.sql"
status: "evolving"
confidence: 0.9
created: "2026-07-13"
updated: "2026-07-13"
---

# 概念：mall-swarm 订单支付一致性

## 本轮范围、既有结论与证据规则

主题为 P0「订单一致性与支付安全」。复核上一轮的待验证结论：订单事务是否真实生效、库存是否条件扣减、RabbitMQ 与数据库是否原子、延迟取消/支付并发是否收敛、支付金额/归属/重复回调是否受控、前后台状态与权限是否受控、失败补偿和测试覆盖是否成立。

精确源码范围：`mall-portal` 的 `OmsPortalOrderService(.java|Impl.java)`、`OmsPortalOrderController`、`PortalOrderDao.xml`、`CancelOrderSender/Receiver`、`OrderTimeOutCancelTask`、`RabbitMqConfig`、`AlipayController/AlipayServiceImpl/AlipayClientConfig`、`MyBatisConfig`；`mall-admin` 的 `OmsOrderController/Service(.java|Impl.java)`、`OmsOrderDao.xml`、`OmsOrderOperateHistoryDao.xml`、`MyBatisConfig`；`mall-gateway` 的 `SaTokenConfig`、`application.yml`；`document/sql/mall.sql`；以及 portal/admin 测试。

证据等级：**A**=源码且自动化/运行结果验证；**B**=多处源码一致、尚未运行；**C**=配置、部署或外部环境待确认。本轮 `mvn -pl mall-portal -am test -DskipTests=false` 编译各模块后失败：两个 portal `@SpringBootTest` 在创建 Druid DataSource 时 MySQL `Connection refused`，同时 Nacos `localhost:9848` 不可达。因此以下业务结论均最高为 **B**，运行时 broker/数据库/支付平台行为为 **C**。

## 复核结果（代码事实，不等于运行时事实）

| 既有结论 / 验证项 | 复核结果 | 等级 | 精确证据 |
| --- | --- | --- | --- |
| “订单方法没有事务” | **纠正：不成立。** portal 与 admin 的订单 Service 接口方法标注 `@Transactional`，两个模块都 `@EnableTransactionManagement`；Controller 与 `CancelOrderReceiver` 都注入接口 Bean 后调用，属于代理可拦截的外部入口。`paySuccessByOrderSn` 在已开启的事务中同类调用 `paySuccess`，不会另开事务但仍处在外层事务。 | B | `mall-portal/.../service/OmsPortalOrderService.java:25-44,74-75`；`.../config/MyBatisConfig.java:10-13`；`OmsPortalOrderController.java:43-44,95-104`；`CancelOrderReceiver.java:16-20`；`mall-admin/.../service/OmsOrderService.java:22-56`；`.../config/MyBatisConfig.java:10-13` |
| 下单重复提交、库存锁定有幂等/条件更新 | **不具备。** 订单号仅 Redis `INCR`，DDL 未见 `oms_order.order_sn` 唯一索引；`hasStock` 读促销投影，随后 `lockStock` 读取 SKU 再写回 `lock_stock+qty`，无 `WHERE stock-lock_stock >= qty`、版本号或请求幂等键。 | B | `OmsPortalOrderServiceImpl.java:94-245,439-453,727-751`；`document/sql/mall.sql:444-489,1660-1672` |
| 支付成功、库存扣减和重复回调 | `notify` 做 RSA 验签且只接受 `TRADE_SUCCESS`；`paySuccessByOrderSn` 先按 `order_sn + status=0 + delete_status=0` 查询，因此同一成功回调在**串行**情形不再重复扣减。可是 `paySuccess(orderId)` 直接主键更新、无归属/状态谓词；库存 SQL 也无 `status/lock_stock/stock` 条件。两个并发回调都在读到状态 0 后可分别更新，最终扣减是否重复取决于实际事务隔离与交错，源码不能保证。 | B（串行保护）；B（并发缺口） | `AlipayServiceImpl.java:72-95`；`OmsPortalOrderServiceImpl.java:253-264,423-434`；`PortalOrderDao.xml:50-68`；`OmsPortalOrderController.java:48-54` |
| 下单事务与 MQ 原子性 | 下单的本地 DB 写操作由接口事务覆盖；`sendDelayMessageCancelOrder` 却在同一业务方法内经 `AmqpTemplate.convertAndSend` 直接发送。未见 outbox、事务同步 after-commit、publisher confirm/return callback 或消息唯一 ID。故 DB 回滚后消息已入 broker、或 DB 提交后消息发送失败，均无源码内可靠收敛机制。 | B | `OmsPortalOrderServiceImpl.java:230-245,328-334`；`CancelOrderSender.java:19-31`；`RabbitMqConfig.java:18-78` |
| 延迟取消可抗重复/失败，且与支付并发安全 | 取消消费者重投时，`cancelOrder` 的初次查询限定 `id + status=0 + delete_status=0`，**顺序重复消息**将 no-op；但取到订单后用主键更新而非条件更新，和支付的“先查后改”构成竞态。取消先提交会把订单关为 4；支付先提交会把订单置 1；两者可能各自读取 0 后分别补偿/扣库存，最终状态及库存/券/积分不由单条条件更新保证。消费者抛错会让容器的运行时 ACK/requeue/DLQ 行为取决于 Spring AMQP/Broker 配置，源码未配置 listener retry/DLQ。定时兜底类的 `@Component` 被注释。 | B（代码分支）；C（投递、ACK、重试） | `CancelOrderReceiver.java:13-21`；`OmsPortalOrderServiceImpl.java:297-325`；`RabbitMqConfig.java:36-78`；`OrderTimeOutCancelTask.java:12-29` |
| 支付金额、订单归属与重复回调受控 | **验签存在，关键业务绑定缺失。** `/alipay/pay`、`/webPay` 接收客户端 `outTradeNo/totalAmount/subject` 后原样给 SDK；发起前未见按订单号查询、当前会员归属、待付款状态或 `pay_amount` 比对。`/alipay/query` 同样无归属校验，且查询成功可推进订单。回调未比较 app_id、seller_id、total_amount、交易号或订单归属；仅验签和状态。 | B | `AlipayController.java:38-72`；`AliPayParam.java:12-27`；`AlipayServiceImpl.java:41-68,72-95,98-130,133-161` |
| 前台详情、取消、确认收货权限/状态机 | Gateway 对非白名单 `/mall-portal/**` 调 member login；`/alipay/**` 全白名单。前台 `list` 按当前会员过滤；`confirmReceiveOrder`、`deleteOrder` 校验订单归属并限制状态；`detail` 无归属校验；`cancelUserOrder` 调用的 `cancelOrder` 也不校验当前会员。`/order/cancelOrder` 甚至只是为任意 orderId 再发送延迟消息，不会立刻取消。 | B | `mall-gateway/.../SaTokenConfig.java:40-76`；`application.yml:56-79`；`OmsPortalOrderController.java:64-114`；`OmsPortalOrderServiceImpl.java:337-420` |
| 后台关闭、发货、历史的一致性 | admin 接口事务经接口代理可覆盖“订单写 + 历史写”。发货 SQL 限 `status=1`，但服务仍对**全部输入 ID**写状态 2 的历史，即更新行数为 0 也会留下“完成发货”记录。关闭仅限制未软删，不限制原状态；同样对所有输入 ID 写关闭历史，且不释放锁库存/退券/返积分。收货人、费用、备注接口可把调用方传入的 `status` 写进历史，未回读订单状态。 | B | `OmsOrderService.java:22-56`；`OmsOrderServiceImpl.java:42-152`；`OmsOrderDao.xml:36-68`；`OmsOrderOperateHistoryDao.xml:4-13` |
| 失败补偿 | portal 取消路径会释放 `lock_stock`、恢复券状态、返还 `useIntegration`；这些都在本地事务内。但 SQL 没有 `lock_stock >= qty` 条件，券更新/会员积分更新无版本或条件谓词，后台关闭未复用该补偿路径。购物车在下单事务内被软删，DB 回滚可回滚；Redis 订单号自增不可回滚。前台支付/取消/确认收货未写 `oms_order_operate_history`。 | B | `OmsPortalOrderServiceImpl.java:230-245,268-325`；`PortalOrderDao.xml:50-96`；`mall.sql:691-699` |

## 路径审查

| 路径 | 正常路径（代码事实） | 异常/并发路径（代码事实与待验证） |
| --- | --- | --- |
| 下单 | 登录会员 → 实时重算购物车/券/积分 → 读库存判断 → 锁库存 → 写订单/项 → 占券、扣积分、删购物车 → 发 TTL 消息。 | 重复请求会重复创建订单；两请求可越过同一次库存判断后交错锁定。DB 失败后的消息结果、MQ 失败后的待付款订单，均无可靠事件补偿（C）。 |
| 支付 | 用户经 `/alipay/pay|webPay` 获表单；Alipay `notify` 验签且 `TRADE_SUCCESS` 后，以待付款订单号推进 0→1，再扣真实库存/解锁库存。 | 未绑定发起参数到订单金额/会员；公开 `query` 可按外部订单号查询并推进；并发回调与取消均读 0 后写，最终状态/库存补偿无 CAS 保证。 |
| 延迟取消 | TTL 消息过期死信到 `mall.order.cancel`；消费者调用 `cancelOrder`，待付款订单转 4，并释放锁库存/券/积分。 | 重复消息的顺序重放通常 no-op；支付/取消并发不安全。消费者失败、broker 重启、TTL 到期准确性、ACK/requeue/DLQ 和多实例竞争都是 C。 |
| 后台履约 | 发货仅更新 `status=1` 订单为 2；前台本人且状态 2 才能确认收货至 3。 | 发货/关闭历史可与实际更新行不一致；后台关闭可以把已支付/发货/完成订单置 4，且不走支付后库存或营销补偿。 |

## 运行时事实与测试结论

**已运行命令：** `mvn -pl mall-portal -am test -DskipTests=false`。`mall-common`、`mall-mbg` 编译/测试阶段通过（无测试），`mall-portal` 两个测试均未加载应用上下文；根因是本地 MySQL 无法连接，Nacos 也不可达。它证明测试依赖真实基础设施，**不验证**任何订单状态、事务代理、RabbitMQ 或支付宝行为。

现有测试仅有 portal `contextLoads`、固定 ID 商品促销 DAO 查询，以及 admin 商品 DAO 测试；没有订单生成、失败回滚、支付回调验签/金额、MQ 消费、权限、幂等或并发测试。下一轮环境就绪后必须补充：

1. Testcontainers/隔离 MySQL + RabbitMQ 的下单成功、任一 DB 写失败回滚、发送失败和消费者重投测试；
2. 同一幂等键、同券、同 SKU 的并发下单，断言条件更新结果、库存非负和一笔有效订单；
3. 支付回调重复、支付与取消并发、支付后延迟消息到达、后台关闭各状态订单；
4. 伪造/错误金额/错误 app-seller/非本人订单号回调与支付发起；
5. Gateway 路由与直连服务两种方式的前后台鉴权、详情/取消越权测试。

## 对 AI 二开的影响（建议，不是现有能力）

AI 客服/订单助手绝不可拥有以下直连或自动执行权限：`/alipay/pay`、`/alipay/webPay`、`/alipay/notify`、`/alipay/query`、`/order/paySuccess`、`/order/cancelOrder`、后台 `/order/update/close`、`/order/update/delivery`、金额/地址修改，以及 Mapper/数据库/MQ 管理权限。模型不得构造支付金额、订单号、回调参数、库存/券/积分修正或状态值。

允许的最小工具仅应是“当前会话本人”的脱敏只读查单、状态解释和受控售后预填；现有 `detail` 无归属校验，修复前也不得封装为工具。取消待付款订单、确认收货和任何地址/购物车变更必须：服务端从会话派生 memberId、重新读取订单及状态、使用幂等键/条件更新、展示不可逆影响、取得一次性明确确认，并记录请求快照、确认人、订单版本、结果和审计 ID。支付必须跳转到受控支付页由用户主动完成；AI 不得代表用户点击、轮询后自动确认或模拟回调。

在新增 AI 写工具前，先将订单状态迁移封装为单一领域服务，采用 `UPDATE ... WHERE id=? AND status=? [AND version=?]`；库存/券/积分使用条件更新；订单创建采用客户幂等键和唯一约束；数据库提交后经 outbox/可靠事件发消息；消费者记录消息/业务幂等键并具备重试、死信、对账和告警。

## 相关页面与本轮增量

- [[20-projects/mall-swarm/architecture/概念_mall-swarm_mall-portal设计]]：修正事务结论并保留前台实现地图。
- [[20-projects/mall-swarm/architecture/概念_mall-swarm_mall-admin设计]]：后台订单接口、事务与审计记录边界。
- [[20-projects/mall-swarm/architecture/概念_mall-swarm_mall-gateway设计]]：支付白名单与入口鉴权。
- [[20-projects/mall-swarm/architecture/主题_mall-swarm_架构全景_综述]]：P0 风险优先级。
