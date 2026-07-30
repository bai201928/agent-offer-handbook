# 第 3 课：Q21～Q30——RocketMQ 5.5、选型与企业级项目深挖

> Kafka 已经建立了消息地图。现在学习 RocketMQ 时不要重新死背一套名词，而要先找相似位置，再理解它为什么采用不同设计。

---

## Q21. RocketMQ 的 NameServer、Broker、Topic、MessageQueue、CommitLog、ConsumeQueue 分别是什么？

**题目定位**：核心题｜中等｜频率 ⭐⭐⭐⭐⭐

### 先听懂：先用“总账和目录”理解 RocketMQ

```text
NameServer      = 路由通讯录
Broker          = 保存和投递消息的仓库
Topic           = 业务分类
MessageQueue    = Topic 内的有序分片
CommitLog       = 保存消息正文的统一大日志
ConsumeQueue    = 帮消费者查找正文位置的轻量目录
```

### 一条消息怎么走？

1. Broker 把自己的 Topic 和队列路由上报给 NameServer；
2. Producer 查询并缓存路由；
3. Producer 直接向目标 Broker 或 5.x 架构中的 Proxy 发送；
4. Broker 把消息正文追加到 CommitLog；
5. 后台根据消息物理位置构建 ConsumeQueue；
6. Consumer 按 Topic 和 MessageQueue 的消费进度读取 ConsumeQueue；
7. 再根据索引位置读取 CommitLog 中的正文。

### 为什么不能说 MessageQueue 就是 Kafka Partition，完全一样？

它们都提供分片、并行、队列内顺序和 Offset，所以在业务作用上相似。

但底层存储组织不同：

- Kafka 每个 Topic-Partition 本身是一条独立日志；
- RocketMQ 多个 Topic 的正文主要统一写入 CommitLog，再通过 ConsumeQueue 构建逻辑消费索引。

### RocketMQ 5.x Topic 还多了什么？

Topic 会区分 NORMAL、FIFO、DELAY、TRANSACTION 等消息类型。建 Topic 时要先明确业务语义，不能把不同类型随意混在同一 Topic。

### 2～3 分钟优秀回答

“NameServer 是 RocketMQ 的轻量路由注册与发现组件，Broker 向它上报 Topic 和队列信息，客户端获取路由后直接连接 Broker 或 Proxy；NameServer 不保存业务消息正文。

Topic 是逻辑业务分类，MessageQueue 是 Topic 内的有序分片和消费单位。Broker 收到消息后，正文主要顺序追加到 CommitLog，再构建 ConsumeQueue 这类逻辑索引。Consumer 先根据 Topic、Queue 和消费 Offset 定位 ConsumeQueue，再回到 CommitLog 读取完整消息。

MessageQueue 和 Kafka Partition 在并行、Offset 和局部顺序上相似，但底层存储模型不同，不能说完全一样。RocketMQ 5.x 还区分 NORMAL、FIFO、DELAY 和 TRANSACTION Topic，设计时要先根据消息语义创建对应类型。”

### 递进追问

1. ConsumeQueue 是否保存完整消息正文？
2. NameServer 挂掉一台，已有客户端是否立刻无法发送？

**记忆句：路由问 NameServer，正文进 CommitLog，消费查 ConsumeQueue。**

---

## Q22. RocketMQ 如何保证消息可靠？Queue 数和 Consumer 数有什么关系？

**题目定位**：高频题｜中等偏上｜频率 ⭐⭐⭐⭐⭐

### 先听懂：同样沿“发送、存储、消费”三段看

### 1. Producer 发送阶段

Producer 要处理发送结果和异常。同步发送能直接获得结果；超时重试仍可能产生结果未知和重复，因此消息要有稳定业务 ID。

### 2. Broker 存储阶段

RocketMQ 常讨论两个独立维度：

#### 刷盘

- 异步刷盘：先写入内存映射区域，后台刷磁盘；
- 同步刷盘：等待达到磁盘持久化条件后确认。

#### 复制

- 异步复制：主节点确认后再同步从节点；
- 同步复制：等待副本达到要求后确认。

同步刷盘不等于同步复制，两者不能混为一个开关。

### 3. Consumer 阶段

Consumer 业务成功后 ACK；失败或超时会进入重试流程，超过次数进入 DLQ。下游业务仍然必须幂等。

### Queue 和 Consumer 的并行关系

在基于队列分配的传统集群消费中，一个 MessageQueue 同一时刻由消费组中的一个实例负责。Topic 只有 4 个 Queue，而已经有 4 个有效 Consumer 时，再增加 20 台机器通常不能继续提升队列并行度。

RocketMQ 5.x PushConsumer 还引入更细粒度的消息级负载均衡体验，但面试回答仍应先说明：**最终吞吐受 Queue、消息组、客户端模式和下游处理能力共同限制，不能只堆 Consumer。**

### 积压时怎么办？

- Queue 数足够：扩 Consumer；
- Queue 数不足：增加队列或建立临时多队列 Topic，用搬运程序重新分发；
- 下游是瓶颈：优化批量、数据库、外部调用；
- 非关键历史消息确实可放弃：必须经过业务审批和审计后再重置消费位点，不能开发人员自行跳过。

### 2～3 分钟优秀回答

“RocketMQ 可靠性也要分发送、存储和消费三段。Producer 需要检查同步或异步发送结果，超时重试复用同一业务 ID；Broker 端要区分刷盘和复制，同步刷盘表示本机持久化确认，同步复制表示副本也达到要求，两者不是同一件事；Consumer 业务成功后确认，失败进入重试，超过次数进入 DLQ，同时业务要幂等。

积压扩容前先看 Queue 数和消费模式。传统集群消费中，一个 MessageQueue 同一时刻由组内一个实例负责，所以 4 个 Queue 配 4 个活跃 Consumer 后，再加机器通常没有效果。Queue 不足时可增加队列，或建立临时多队列 Topic 重新分发积压消息。长期还要优化数据库、批量和外部接口，不能只扩机器。”

### 递进追问

1. 同步刷盘打开后，主节点宕机是否一定不丢？
2. 为什么消费失败重试仍需要幂等？

**记忆句：刷盘管本机，复制管副本，ACK 管消费，扩容先看 Queue。**

---

## Q23. RocketMQ 5.x 如何保证 FIFO 顺序消息？

**题目定位**：必考题｜中等偏上｜频率 ⭐⭐⭐⭐⭐

### 先听懂：FIFO 不是整个 Topic 只有一条队伍

RocketMQ 5.x 使用 Message Group 表达局部顺序。

```text
Message Group = O1001
创建订单 → 支付订单 → 发货订单

Message Group = O2002
创建订单 → 取消订单
```

同一个 Message Group 内按 FIFO 处理；不同组可以并行。

### 为什么选择 Message Group，而不是全局顺序？

如果整个 Topic 全局顺序，只能像一个收费窗口，任何消息慢都会挡住全部业务。按订单分组后，每个订单内部有序，订单之间并行，吞吐更高。

### 完整顺序的三个条件

#### 1. 生产顺序

两个 Producer 同时发送同一个 Message Group，但业务没有建立谁先谁后，Broker 无法猜出正确顺序。上游必须先建立确定的事件版本或串行发送关系。

#### 2. 存储与投递顺序

Producer 设置相同 Message Group，并使用 FIFO Topic，RocketMQ 维护组内顺序语义。

#### 3. 消费完成顺序

Consumer 收到后如果立即扔进普通线程池：

```java
pool.submit(() -> process(message));
return SUCCESS;
```

线程调度可能让发货先于支付完成。组内处理必须保持串行完成，或使用按 Key 串行的执行器。

### 失败为什么会头阻塞？

如果“支付”处理失败，后面的“发货”不能越过它，否则顺序失去意义。因此毒消息可能阻塞同组后续消息。要限制重试、告警、隔离单个 Message Group 并提供人工处理。

### 2～3 分钟优秀回答

“RocketMQ 5.x 的 FIFO 应以 Message Group 为核心。需要有序的消息设置相同消息组，例如同一 `orderId` 或 `runId`；同组消息按 FIFO 处理，不同消息组可以并行，从而实现局部顺序而不是牺牲整个 Topic 的吞吐。

完整顺序需要生产、存储投递和消费三个阶段都成立。上游必须按正确顺序发送；Topic 要使用 FIFO 类型并设置稳定 Message Group；Consumer 不能收到后再用无序线程池并发完成同组消息。

顺序消息失败时后续同组消息通常需要等待，因此毒消息会造成头阻塞。项目中要用有限重试、状态版本、幂等、单组隔离和人工处置。旧版常讲 `MessageQueueSelector` 把同订单路由到同一 Queue，但面向 5.x 我会优先说明 Message Group。”

### 递进追问

1. 不同 Message Group 之间是否有顺序保证？
2. 同组某条消息永久失败，怎样减少对其他订单的影响？

**记忆句：同组有序，组间并行；生产、投递、消费三段都不能乱。**

---

## Q24. RocketMQ 5.x 延时消息如何工作？

**题目定位**：高频题｜中等｜频率 ⭐⭐⭐⭐

### 先听懂：延时消息不是让线程 `sleep` 30 分钟

订单创建后 30 分钟未支付要关闭。如果应用自己创建定时线程：

- 应用重启后任务可能丢；
- 多实例重复执行；
- 百万定时任务占大量内存；
- 难以迁移和恢复。

RocketMQ 延时消息把触发时间交给 Broker 持久化管理。

### 5.x 的现代机制

Producer 设置毫秒级投递时间戳，消息发送到 DELAY Topic。到期前处于 Timing 状态，对普通 Consumer 不可见；达到时间后转为 Ready，进入正常投递。

```java
long deliverAt = System.currentTimeMillis() + 30 * 60_000L;
Message message = builder
        .setTopic("order-timeout")
        .setDeliveryTimestamp(deliverAt)
        .setKeys(orderId)
        .setBody(payload)
        .build();
```

### 到期是否保证这一毫秒立即执行？

不保证硬实时。到期表示可以投递，Broker 调度、故障、网络和 Consumer 积压都会增加延迟。

### 为什么 Consumer 还要查订单状态？

用户可能在第 10 分钟已经支付。延时消息在第 30 分钟到达后必须执行条件更新：

```sql
UPDATE orders
SET status='CLOSED'
WHERE order_id=? AND status='UNPAID';
```

延时消息是触发器，数据库才是业务真相。

### 大量消息同一时刻到期有什么问题？

例如百万优惠券都在 00:00 到期，会形成定时洪峰。应打散时间、提前容量评估，并保护下游。

### 2～3 分钟优秀回答

“RocketMQ 5.x 延时消息通过投递时间戳实现。Producer 把消息发送到 DELAY 类型 Topic，并设置毫秒级 Delivery Timestamp；消息到期前保存在时间相关的存储和调度体系中，对普通 Consumer 不可见，到期后变为 Ready 并进入正常投递。

它适合订单超时、审批提醒和延迟重试，比应用线程 `sleep` 更容易持久化、扩容和故障恢复。但它不是硬实时调度，到期只表示最早可以投递，Broker、网络和 Consumer 积压都会带来延迟。

消费时必须重新检查业务状态。例如订单已经支付，旧的超时消息应幂等跳过。大量消息设置同一到期时间还会形成定时洪峰，需要打散和容量保护。”

### 递进追问

1. 延时消息重复到达会不会重复关闭订单？
2. 为什么不能把数据库状态直接相信消息里的旧值？

**记忆句：Broker 负责到点触发，数据库状态机负责是否执行。**

---

## Q25. RocketMQ 事务消息如何实现？半事务消息是什么？

**题目定位**：高薪深挖题｜困难｜频率 ⭐⭐⭐⭐⭐

### 先听懂：它解决“本地事务成功，但消息没发”的问题

订单服务需要：

```text
MySQL：订单从 UNPAID 改成 PAID
RocketMQ：发布 PaymentConfirmed
```

直接执行两次写入，中间一定有失败窗口。

### 完整流程

```text
1. Producer 发送半事务消息
2. Broker 保存并标记暂不可投递
3. Producer 执行本地事务
4. 成功：发送 Commit
5. 失败：发送 Rollback
6. 结果未知或二次确认丢失：Broker 回查 Producer
7. Producer 查询本地事务记录，返回 Commit / Rollback / Unknown
```

### 半事务消息是什么？

已经到达 Broker，但还没有最终确认，消费者暂时看不到。它像已经进入仓库但贴着“待审核”标签的包裹。

### 回查为什么必须查数据库？

错误方式：

```java
Map<String, Boolean> localResult = new ConcurrentHashMap<>();
```

Producer 重启后内存清空，无法判断事务结果。

正确方式是通过 `transactionId` 查询订单表或事务状态表。

### 它保证什么？

保证本地核心事务与消息最终是否可投递的一致性。

### 它不保证什么？

不保证下游积分服务一定成功，不保证跨服务强一致。Consumer 仍需重试、幂等和 DLQ。

### 与 Outbox 怎么选？

- RocketMQ 事务消息：协议内直接支持半消息和回查，业务消息体验直观；
- Outbox：不绑定特定 MQ，可审计、可查询，适合 Kafka 主线和多中间件系统；
- 两者都不能替代下游幂等与对账。

### 2～3 分钟优秀回答

“RocketMQ 事务消息解决 Producer 本地事务和消息可投递之间的一致性。Producer 先发送半事务消息，Broker 保存后暂不向 Consumer 投递；随后 Producer 执行本地事务，成功发送 Commit，失败发送 Rollback。如果二次确认因网络或重启丢失，Broker 会回查 Producer，Producer 根据数据库中的事务记录返回最终状态。

半消息已经在 Broker，但处于不可投递状态。事务回查必须查询可靠持久化事实，不能依赖 JVM 内存。

它保证的是本地事务与消息最终可见的一致，不保证下游消费事务成功，所以 Consumer 仍需幂等、重试、DLQ 和对账。Kafka 主线项目我通常使用 Outbox；RocketMQ 业务系统可使用事务消息，但会根据审计、运维和中间件绑定成本选择。”

### 递进追问

1. 回查时本地事务仍在执行，应该返回什么？
2. 半消息 Commit 后，积分服务永久失败怎么办？

**记忆句：先发不可见半消息，本地事务后确认，结果未知靠数据库回查。**

---

## Q26. Kafka、RocketMQ、RabbitMQ 如何选型？

**题目定位**：选型必考题｜中等偏上｜频率 ⭐⭐⭐⭐⭐

### 先听懂：不要按“哪个更强”选，要按业务问题选

| 需求 | Kafka | RocketMQ | RabbitMQ |
| --- | --- | --- | --- |
| 海量事件流、日志、CDC | 首选 | 可以 | 通常不是首选 |
| 历史回放、流计算生态 | 强 | 支持回溯，生态不同 | 不是核心优势 |
| 业务 FIFO、延时、事务消息 | 需业务设计或生态补充 | 原生体验强 | 有路由和确认能力，事务语义不同 |
| Java 微服务业务消息 | 适合 | 很适合 | 适合传统业务 |
| Exchange 与复杂路由 | 不是核心模型 | Tag/过滤等 | 核心优势 |
| 超高吞吐与数据平台 | 强 | 强，但定位更偏业务消息 | 一般 |

### 为什么本教材 Kafka 首选？

NexusAgent 项目包含：

- 文档入库事件；
- Agent Run 状态流；
- 工具审计；
- 评估结果；
- CDC 和离线分析；
- 多消费组回放和重建投影。

这些更贴近持久事件流和数据平台生态，所以 Kafka 为主线。

### RocketMQ 何时更合适？

- 大量原生延时任务；
- 同订单或同 Run 的 FIFO 消息组；
- 需要半事务消息衔接本地事务；
- 团队是 Java 微服务，已有成熟 RocketMQ 平台和运维经验；
- 希望内建消费重试与 DLQ 体验。

### RabbitMQ 几句话掌握

RabbitMQ 以 Exchange、Queue、Binding、Routing Key 为核心，路由模型灵活，适合传统业务通知、工作队列和复杂路由。面对本项目的大规模事件回放和流处理，它不是第一选择，但不能简单说它“性能差所以淘汰”。

### 选型必须做 PoC

相同条件测试：

- 消息大小；
- 副本和可靠性配置；
- 生产/消费批量；
- P99 延迟；
- Broker 故障恢复；
- 积压追赶；
- 跨机房；
- 运维、权限、监控和团队能力。

### 2～3 分钟优秀回答

“选 MQ 我不会只比较宣传 TPS，而是看业务语义、生态、可靠性、运维和团队能力。Kafka 擅长高吞吐持久事件流、历史回放、CDC、流处理和数据平台生态；RocketMQ 更偏复杂业务消息，原生 FIFO Message Group、延时消息、事务消息、重试和 DLQ 体验较好；RabbitMQ 的 Exchange 和 Routing Key 路由灵活，适合传统业务通知和工作队列。

NexusAgent 主项目需要文档事件、Run 状态流、审计、评估、多消费组回放和数据分析，因此 Kafka 优先。若项目核心是订单超时、同业务实体严格顺序和本地事务后可靠发消息，而且公司已有 RocketMQ 平台，我会选 RocketMQ。

最终要在相同消息大小、可靠配置和拓扑下 PoC，测试吞吐、P99、故障恢复、积压追赶、跨机房和运维能力，而不是用绝对化表格决定。”

### 递进追问

1. Kafka 支持事务，为什么业务事务场景仍可能选择 RocketMQ？
2. 公司已有成熟 Kafka 平台，但某业务需要延时消息，是否一定再建 RocketMQ？

**记忆句：事件流与生态看 Kafka，复杂业务语义看 RocketMQ，灵活路由看 RabbitMQ。**

---

## Q27. Kafka 和 RocketMQ 的 Broker 存储架构有什么区别？

**题目定位**：资深原理题｜困难｜频率 ⭐⭐⭐⭐

### 先听懂：一本本独立日志，和一本总账加目录

### Kafka

每个 Topic-Partition 是独立日志目录和 Segment 文件：

```text
order-event-0/
order-event-1/
payment-event-0/
```

Consumer 直接按 Partition 和 Offset 读取对应日志。

优点：Partition 模型直接、天然适合分区级复制、回放和并行流处理。

代价：Topic 和 Partition 极多时，文件、索引、元数据、Page Cache 和故障恢复成本上升。

### RocketMQ

不同 Topic 消息正文主要统一顺序追加到 CommitLog，再构建各 Topic/Queue 的 ConsumeQueue：

```text
CommitLog：所有正文的总账
ConsumeQueue：不同 Topic/Queue 的目录
```

优点：写入路径集中，统一 CommitLog 保持顺序追加；ConsumeQueue 较轻量。

代价：读取要通过逻辑索引定位正文，调度和存储组件组织方式不同。

### 协调与路由

- Kafka 4.x 使用 KRaft Controller 管理元数据和 Partition Leader；
- RocketMQ 使用 NameServer 提供路由发现，具体高可用还与 Broker 主从、副本和 Controller/DLedger 部署方式有关。

### 不要说的绝对结论

- “Kafka Topic 多了一定随机写、一定性能崩溃”；
- “RocketMQ 可以无限 Topic”；
- “RocketMQ Master 挂了永远无缝切换”；
- “Kafka 只适合日志，RocketMQ 只适合订单”。

真实表现取决于版本、硬件、分区/队列规模、消息大小和部署方式。

### 2～3 分钟优秀回答

“Kafka 和 RocketMQ 最核心的存储差异是日志组织方式。Kafka 每个 Topic-Partition 是独立追加日志，由 Segment 和索引组成，复制、顺序和消费进度都以 Partition 为中心。RocketMQ 则把不同 Topic 的消息正文主要统一追加到 CommitLog，再为不同 Topic 和 MessageQueue 构建 ConsumeQueue 逻辑索引，消费者根据索引定位正文。

因此 Kafka 的 Partition 模型非常直接，适合分区级复制、回放和流处理；但分区数量极大时，文件、元数据和恢复成本需要控制。RocketMQ 的统一 CommitLog 让写入路径集中，ConsumeQueue 作为轻量目录，但它不是没有元数据和队列成本。

协调方面，Kafka 4.x 使用 KRaft Controller 管理元数据和 Leader；RocketMQ 使用 NameServer 做路由发现，高可用还要结合具体 Broker 副本架构。选型应通过相同条件压测，避免绝对化结论。”

### 递进追问

1. Kafka Segment 为什么要滚动切分？
2. RocketMQ ConsumeQueue 损坏后是否能根据 CommitLog 重建？

**记忆句：Kafka 是分区独立日志，RocketMQ 是统一正文总账加消费目录。**

---

## Q28. 企业级 Agent / RAG 项目如何设计消息队列？

**题目定位**：项目深挖题｜困难｜频率 ⭐⭐⭐⭐⭐

### 先听懂：先画业务状态，再建 Topic

NexusAgent 是多租户 Agent 与知识库平台，核心异步链路：

```text
用户提交文档
  ↓
解析与切分
  ↓
Embedding
  ↓
写入 Milvus
  ↓
索引版本完成
```

另一条链路：

```text
用户提交 Agent Run
  ↓
规划
  ↓
工具调用
  ↓
人工审批
  ↓
恢复执行
  ↓
评估和审计
```

### Topic 设计

```text
rag-document-command    Key=documentId
rag-document-event      Key=documentId
agent-run-command       Key=runId
agent-run-event         Key=runId
tool-audit-event        Key=operationId
evaluation-result       Key=evaluationRunId
```

Topic 不按每个微服务机械创建，而按以下边界评审：

- 数据语义；
- 保留时间；
- 权限与敏感级别；
- 消息大小；
- 顺序实体；
- 吞吐与消费组数量。

### 消息信封

```json
{
  "eventId": "E1001",
  "eventType": "DocumentImportRequested",
  "schemaVersion": 2,
  "tenantId": "T1",
  "aggregateId": "D1001",
  "aggregateVersion": 3,
  "traceId": "TR1001",
  "occurredAt": "2026-07-30T14:00:00+08:00",
  "payload": {}
}
```

### 可靠性闭环

```text
API MySQL 事务
├── 写 document_version / agent_run
└── 写 outbox_event
        ↓
Outbox Relay → Kafka
        ↓
Consumer：Inbox + 业务状态版本
        ↓
重试 Topic → DLQ → 修复后回放
```

### 外部工具副作用

Agent 调用发邮件、创建工单或扣费工具时，必须使用稳定 operationId。Consumer 本地 Inbox 只能保证本地事务，不自动保证第三方没有执行两次。

### 监控指标

- Producer 发送失败率和超时；
- Outbox 最老未发送年龄；
- 每 Partition Lag 和最老消息年龄；
- Consumer 成功率、处理 P95/P99；
- Retry 和 DLQ 增长；
- 单租户流量、Key 倾斜；
- Agent Run 卡在某状态的数量；
- MySQL、Milvus、模型服务和工具服务的依赖指标。

### 2～3 分钟优秀回答

“在 Agent/RAG 项目中，我先定义状态机和恢复边界，再设计 Topic。文档入库使用 `documentId` 作为 Key，Agent Run 使用 `runId`，保证同一实体事件局部有序，不同实体并行。Topic 按业务语义、保留、权限和吞吐划分，例如 document-command、document-event、run-command、run-event 和 tool-audit，而不是每个微服务一个 Topic。

API 在同一 MySQL 事务中写业务状态和 Outbox，Relay 可重复发布 Kafka；Consumer 在本地事务中使用 Inbox、eventId 和 aggregateVersion 做幂等和防状态倒退，成功后提交 Offset。暂时故障进入多级 Retry Topic，永久故障进入 DLQ，修复后受控回放。

Agent 外部工具调用还要传稳定 operationId，因为本地 Inbox 不能保证第三方只执行一次。监控不仅看 Lag，还看最老消息年龄、Outbox 超龄、重试率、单租户倾斜、Run 状态停留时间和下游依赖容量。”

### 递进追问

1. Embedding 完成但 Milvus 写入超时，如何判断是否重试？
2. 同一文档重新上传新版本，旧的延迟消息到达怎么办？

**记忆句：状态机定真相，Key 定顺序，Outbox 保发布，Inbox 保幂等，operationId 保外部副作用。**

---

## Q29. 让你从零设计一个消息队列，你会怎么设计？

**题目定位**：系统设计题｜困难｜频率 ⭐⭐⭐⭐

### 先听懂：面试官不要求你一小时重造 Kafka

他看你能否把一个复杂系统拆成模块，并说明取舍。

### 第一步：先定目标

必须先问：

- 点对点工作队列还是发布订阅？
- 目标吞吐和 P99？
- 消息最大多大？
- 是否持久化和回放？
- 顺序范围是什么？
- 至少一次还是最多一次？
- 保留多久？
- 单机房还是跨机房？

没有这些约束，任何设计都只是空谈。

### 第二步：最小架构

```text
Producer SDK
    ↓ RPC
Broker 集群
    ↓ 追加日志
Topic / Partition
    ↓ Fetch
Consumer SDK / Consumer Group
```

还需要控制面：

```text
Controller / Metadata Service
├── Broker 注册与心跳
├── Topic 和 Partition 元数据
├── Leader 选举
└── Consumer Group 协调
```

### 第三步：存储

- Append-only Log；
- Segment 滚动文件；
- 稀疏 Offset 索引；
- 时间索引；
- Page Cache；
- 保留和清理；
- 校验和防止损坏。

不建议把海量消息正文直接逐条存普通关系数据库，因为随机索引、事务和行存储开销通常不适合日志型吞吐。

### 第四步：高可用

- Partition 多副本；
- Leader/Follower；
- 同步资格；
- 自动选主；
- 跨故障域放置；
- ACK 门槛；
- 元数据控制器自身也要高可用。

### 第五步：消费模型

- Consumer Group 分配；
- Offset 存储；
- Pull 和长轮询；
- Rebalance；
- 手动确认；
- 重试、DLQ 和回放。

### 第六步：企业能力

- ACL、TLS、多租户配额；
- Schema 兼容；
- 指标、日志、消息轨迹；
- 限流和背压；
- 管理台和审计；
- 扩分区、迁移、滚动升级；
- 容灾和故障演练。

### 2～3 分钟优秀回答

“我会先明确消息模型、吞吐、延迟、可靠语义、顺序范围、保留期和容灾要求，再设计，而不是直接说 Producer、Broker、Consumer。

数据面采用 Topic 和 Partition，把消息追加到 Segment 日志，通过 Offset 和稀疏索引读取；Producer 批量发送，Consumer 使用 Pull 和长轮询。控制面负责 Broker 注册、元数据、Partition Leader、Consumer Group 和 Rebalance。

高可用通过多副本、Leader/Follower、ACK 门槛、自动选主和跨故障域部署实现；消费端保存 Offset，提供手动确认、重试、DLQ 和回放。最后补齐 ACL、TLS、配额、Schema、监控、消息轨迹、限流、扩容迁移和容灾演练。

取舍上，更多副本和更强 ACK 提高可靠性但增加延迟与不可用概率；更多 Partition 增加并行但提高元数据和恢复成本；全局顺序需要单分区，会牺牲吞吐。”

### 递进追问

1. 如何处理一条超大消息？
2. 扩容 Broker 后，已有 Partition 如何迁移并保持服务？

**记忆句：先定语义，再画数据面和控制面，最后补可靠性与运维。**

---

## Q30. 项目深挖：Agent 异步执行链故障后如何恢复？

**题目定位**：高薪项目题｜困难｜频率 ⭐⭐⭐⭐⭐

### 场景

用户提交 Run `R1001`，系统调用搜索工具、生成结果并写入审计。故障发生：

```text
1. Worker 已调用外部搜索工具成功
2. 写 MySQL 结果时发生网络超时
3. Worker 没提交 Offset
4. 消息再次投递
```

问题：怎样避免外部工具被再次调用并产生双重副作用？

### 第一步：把 Run 拆成可恢复步骤

```text
CREATED
PLANNING
TOOL_PENDING
TOOL_RUNNING
TOOL_SUCCEEDED
GENERATING
COMPLETED
FAILED
```

MySQL 保存当前状态和版本，Kafka 负责推动状态，不把唯一事实只放在消息里。

### 第二步：每个外部操作有稳定 operationId

```text
operationId = hash(runId + stepId + toolName + businessVersion)
```

第一次调用前，在数据库保存工具操作记录：

```text
operation_id
run_id
step_id
request_hash
status=PENDING/RUNNING/SUCCEEDED/FAILED
external_result_ref
```

外部工具支持幂等键时，传 operationId；重试时复用同一个 ID。

### 第三步：处理结果未知

网络超时不等于工具失败。恢复逻辑：

1. 查询本地工具操作表；
2. 若 SUCCEEDED，复用结果，不再次调用；
3. 若外部系统支持查询，用 operationId 查询；
4. 无法查询且副作用高风险，进入人工确认；
5. 只有确认未执行后才再次调用。

### 第四步：消息消费幂等

Consumer 使用 `eventId + aggregateVersion`：

- 同 eventId 重复：跳过；
- 旧版本：拒绝状态倒退；
- 新版本且状态合法：推进状态；
- 相同 eventId 不同 payload hash：隔离告警。

### 第五步：恢复与补偿

- Retry Topic 处理暂时故障；
- DLQ 保存永久失败；
- 定时扫描长时间停留在 TOOL_RUNNING 的 Run；
- 对账 Kafka 事件、Run 状态、工具操作表和审计记录；
- 提供人工 Resume、Skip、Compensate，但所有操作写审计。

### 第六步：回答时不要虚构

可以把它作为项目设计和故障演练方案，但没有真实上线数据时要说“我们通过故障注入验证”，不要声称处理过真实千万级事故。

### 2～3 分钟优秀回答

“Agent 异步链路的难点是外部工具调用和本地状态不能天然原子提交。我的做法是把 Run 设计成持久化状态机，每个步骤有版本，每次外部副作用生成稳定 operationId，并在调用前写工具操作表。

如果工具调用成功但本地写入超时，消息会重投。恢复时不会直接再次调用，而是先查工具操作表；若已成功则复用结果，若外部支持按 operationId 查询则查询真实结果，结果未知且副作用高风险时进入人工确认。重试始终复用同一个 operationId。

消息侧使用 eventId、payload hash 和 aggregateVersion 做幂等与防状态倒退；暂时故障进入 Retry Topic，永久失败进入 DLQ；后台扫描长时间停滞的 Run，并对账 Kafka、MySQL、工具记录和审计。这样 MQ 提供可重投，数据库提供业务真相，operationId 保证外部副作用可恢复。”

### 递进追问

1. 第三方工具不支持幂等键也不支持查询，怎么办？
2. 人工点击 Resume 两次如何防止重复恢复？

**记忆句：消息可以重投，状态必须可恢复，外部副作用必须有稳定身份。**

---

# 三轮学习与 70% 验收方案

## 第一轮：只求听懂，不背答案

阅读第 0 课和每题的“先听懂”。完成以下任务：

1. 画出 Kafka 消息链路；
2. 用订单例子解释每个术语；
3. 能指出一条消息在哪些地方可能失败。

## 第二轮：遮住答案口述

每题按五句话组织：

```text
1. 先给结论
2. 用具体项目例子
3. 解释机制
4. 说明失败边界
5. 给出监控或取舍
```

不会时只看关键词，不整段背诵。

## 第三轮：故障推演

对每个方案追问：

- 这里宕机会怎样？
- 谁负责重试？
- 重试会不会重复？
- 数据库如何裁决？
- 如何监控并证明恢复成功？

## 30 题掌握标准

### 必须独立回答的 21 题

Q01、Q02、Q03、Q04、Q05、Q06、Q08、Q09、Q10、Q11、Q12、Q13、Q14、Q15、Q17、Q19、Q20、Q21、Q23、Q25、Q26。

掌握这 21 题即达到 70%。

### 冲击高薪岗位再掌握

Q16 Kafka 事务、Q18 Inbox、Q22 RocketMQ 可靠性、Q24 延时消息、Q27 存储架构、Q28 Agent 项目、Q29 设计 MQ、Q30 故障恢复。

## 最终速记图

```text
基础：Producer → Broker → Topic → Partition/Queue → Consumer Group → Offset
可靠：Outbox → ACK/副本 → Inbox/幂等 → Offset
异常：Retry → DLQ → 回放 → 对账
顺序：Key / Message Group → 同分片 → 串行完成 → 版本裁决
积压：生产速率、消费速率、分区并行、净追赶速度
选型：Kafka 事件流，RocketMQ 业务语义，RabbitMQ 灵活路由
```
