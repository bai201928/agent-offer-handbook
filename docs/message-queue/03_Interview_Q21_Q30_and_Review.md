# Kafka 4.3.x + RocketMQ 5.5 企业级消息队列教材：Q21～Q30 与复习验收

## Q21. RocketMQ 的 Topic、MessageQueue、CommitLog、ConsumeQueue 是什么？

**题目定位**：核心面试题｜中等｜频率 ⭐⭐⭐⭐⭐

### 为什么现在问这道题？

这道题位于从“会使用 MQ”走向“能解释失败窗口”的关键位置。回答时必须依次覆盖：结论与边界、项目场景、底层机制、故障与恢复、监控验证、方案取舍。

### 2～3 分钟优秀回答

RocketMQ 的 Topic 是逻辑分类，MessageQueue 是消息存储和传输的最小逻辑队列单位。Broker 接收消息后，正文主要顺序追加到统一 CommitLog；后台 Dispatch 根据物理位置构建 ConsumeQueue 等逻辑索引，消费者先定位逻辑队列，再回到 CommitLog 读取正文。带 Key 时还可构建检索索引。

因此面试中不能简单说“RocketMQ Queue 就是 Kafka Partition，完全一样”。二者在并行和局部顺序上有相似作用，但存储模型不同：Kafka 每个 Topic-Partition 是独立日志；RocketMQ 使用统一 CommitLog + 逻辑消费索引，更适合解释大量 Topic 下的写入组织。

NameServer 主要维护 Broker 上报的路由，不保存消息；Producer 查询并缓存路由后选择队列发送；Consumer Group 保存或推进消费状态。RocketMQ 5.x Topic 还区分 NORMAL、FIFO、TRANSACTION、DELAY 等消息类型，设计时应先根据语义建 Topic。

### 可读但会制造事故的 Java 反例

```java
// 反例：把 NameServer 当成消息转发代理。
nameserverClient.send(message); // 概念错误
```

### 企业级改进代码

```java
// 改进：客户端从 NameServer 获取路由，再向 Broker/Proxy 发送。
Route route = routeService.lookup(topic);
MessageQueue queue = selector.select(route.writableQueues(), shardingKey);
producer.send(queue, message);
```

### 面试官递进追问

1. ConsumeQueue 是否保存完整消息正文？
2. NameServer 在发送热路径中做什么？

### 复习记忆钩子

**正文进 CommitLog，消费看 ConsumeQueue，NameServer 管路由。**

---

## Q22. 项目为什么选择 RocketMQ？

**题目定位**：高频面试题｜中等｜频率 ⭐⭐⭐⭐

### 为什么现在问这道题？

这道题位于从“会使用 MQ”走向“能解释失败窗口”的关键位置。回答时必须依次覆盖：结论与边界、项目场景、底层机制、故障与恢复、监控验证、方案取舍。

### 2～3 分钟优秀回答

选择 RocketMQ 不应只说“Java 开发、阿里开源”。更有说服力的理由是业务语义：如果系统大量使用 FIFO 消息组、原生延时消息、事务消息、消费重试与 DLQ，而且团队是 Java 微服务栈，RocketMQ 能减少自建业务消息能力的成本。5.5.0 还需要结合实际客户端兼容性、运维成熟度和组织基础设施评估。

NexusAgent 若以业务命令为主，例如订单超时关闭、Agent 审批到期、同一 Run 严格状态推进、本地事务后可靠发消息，RocketMQ 很合适。若核心是海量事件流、日志、CDC、流计算、多组回放和数据平台生态，则 Kafka 优先。

选型时我会做 PoC：相同消息大小、可靠性配置、副本拓扑和消费逻辑下，测吞吐、P99、故障恢复、积压追赶、运维工具、跨机房与团队能力。最后给出可回退方案，而不是用绝对化营销数字决定。

### 可读但会制造事故的 Java 反例

```java
// 反例：仅凭开发语言选 MQ。
if (teamUsesJava) return "RocketMQ";
```

### 企业级改进代码

```java
// 改进：用需求矩阵和验证证据选型。
MqDecision d = evaluate(
    needsFifo, needsDelay, needsLocalTransactionBridge,
    replayWindow, streamEcosystem, opsCapability, p99Slo);
return poc.verify(d);
```

### 面试官递进追问

1. RocketMQ 事务消息是否保证下游事务？
2. Kafka 何时仍是更优选择？

### 复习记忆钩子

**选型看业务语义、生态、运维和实测，不看单一标签。**

---

## Q23. RocketMQ 5.x 如何保证 FIFO 顺序？

**题目定位**：高频面试题｜中等偏上｜频率 ⭐⭐⭐⭐⭐

### 为什么现在问这道题？

这道题位于从“会使用 MQ”走向“能解释失败窗口”的关键位置。回答时必须依次覆盖：结论与边界、项目场景、底层机制、故障与恢复、监控验证、方案取舍。

### 2～3 分钟优秀回答

RocketMQ 5.x 的现代回答应以 Message Group 为中心。Producer 为需要有序的消息设置相同消息组，Broker 按消息组维护 FIFO 存储与投递语义；不同消息组可以并行。比如同一 `orderId` 或 `runId` 使用相同 Message Group，而不同订单独立消费。

完整顺序需要生产顺序、存储顺序和消费顺序。若多个 Producer 并发发送同一消息组，发送调用本身没有建立顺序，Broker 无法猜测业务先后；PushConsumer 监听器内再异步分发也会破坏完成顺序。顺序消息失败时，后续消息通常要等待，毒消息可能造成头阻塞，因此必须限制重试并提供人工处置。

旧版常讲 `MessageQueueSelector` 把同一订单发到同一队列，这仍有助于理解 4.x Remoting 客户端，但面向 5.x 应明确消息组是业务 FIFO 边界。消费者仍需要版本号和幂等，防止重复与迟到事件造成状态倒退。

### 可读但会制造事故的 Java 反例

```java
// 反例：同一消息组在监听器里再次并发分发。
listener = message -> {
    pool.submit(() -> process(message));
    return ConsumeResult.SUCCESS;
};
```

### 企业级改进代码

```java
// 改进：监听器同步完成业务，并使用稳定 Message Group。
Message msg = messageBuilder
    .setTopic("order-fifo")
    .setMessageGroup(orderId)
    .setBody(payload)
    .build();
producer.send(msg);
```

### 面试官递进追问

1. FIFO 消息失败为什么会阻塞后续消息？
2. 多个 Producer 如何建立发送先后？

### 复习记忆钩子

**同组有序、组间并行；监听器同步完成，版本防状态倒退。**

---

## Q24. RocketMQ 5.x 延时消息如何工作？

**题目定位**：高频面试题｜中等偏上｜频率 ⭐⭐⭐⭐

### 为什么现在问这道题？

这道题位于从“会使用 MQ”走向“能解释失败窗口”的关键位置。回答时必须依次覆盖：结论与边界、项目场景、底层机制、故障与恢复、监控验证、方案取舍。

### 2～3 分钟优秀回答

RocketMQ 5.x 延时消息用毫秒时间戳表示预期投递时间。消息到 Broker 后先进入基于时间的存储/调度体系，在到期前对普通消费者不可见；到期后转为 Ready，进入正常存储与投递流程。它适合订单超时、审批到期、任务重试和定时触发。

面试中不要只背旧版固定延时等级和 `ScheduleMessageService`。现代 5.x 文档强调时间戳、Delay 类型 Topic、时间范围和投递精度约束。延时到期并不等于业务准时完成：Broker 故障、积压和消费者变慢都会产生实际延迟，因此它是“最早可见时间”附近的可靠触发，不是硬实时调度。

项目中延时消息必须携带业务版本。比如 30 分钟后关闭订单，消费者到期后先检查订单仍是 UNPAID 且版本匹配；若已经支付则幂等跳过。不能因为旧的超时消息晚到就关闭已支付订单。

### 可读但会制造事故的 Java 反例

```java
// 反例：延时到期后无条件关闭订单。
public void onTimeout(String orderId) {
    orderRepository.close(orderId);
}
```

### 企业级改进代码

```java
// 改进：延时只是触发器，数据库状态机才做最终裁决。
public void onTimeout(OrderTimeoutEvent e) {
    orderRepository.closeIfUnpaidAndVersion(
        e.orderId(), e.expectedVersion());
}
```

### 面试官递进追问

1. 延时消息能否保证毫秒级准时执行？
2. 为什么必须检查业务状态？

### 复习记忆钩子

**延时消息负责到期触发，数据库状态机负责是否执行。**

---

## Q25. RocketMQ 事务消息如何实现？

**题目定位**：高频面试题｜困难｜频率 ⭐⭐⭐⭐⭐

### 为什么现在问这道题？

这道题位于从“会使用 MQ”走向“能解释失败窗口”的关键位置。回答时必须依次覆盖：结论与边界、项目场景、底层机制、故障与恢复、监控验证、方案取舍。

### 2～3 分钟优秀回答

RocketMQ 事务消息解决 Producer 本地事务与消息可投递之间的一致性。流程是：Producer 先发送半消息，Broker 保存并标记不可投递；Producer 执行本地事务；成功发送 Commit，失败发送 Rollback；若二次确认丢失或结果未知，Broker 调用 Transaction Checker 回查本地事务状态。

关键边界有三点。第一，它保证的是本地核心事务与消息最终可见的一致，不是同步强一致；第二，它不保证下游消费业务成功，下游仍需重试、幂等和 DLQ；第三，事务回查必须读取可靠本地事务记录，不能靠内存变量猜测。

例如订单服务在本地事务表保存 `transaction_id` 与订单状态。Checker 根据这张表返回 Commit、Rollback 或 Unknown。若本地事务已经提交，即使第一次 Commit ACK 丢失，回查仍能恢复。对于 Kafka 主线项目，我会比较 Outbox：Outbox 更通用、可审计且不绑定 MQ 事务协议；RocketMQ 事务消息开发体验更直接，但同样要设计下游一致性。

### 可读但会制造事故的 Java 反例

```java
// 反例：回查根据 JVM 内存判断，重启后状态丢失。
return localCache.get(transactionId) ? COMMIT : ROLLBACK;
```

### 企业级改进代码

```java
// 改进：回查读取本地事务耐久事实。
TransactionRecord tx = txRepository.find(transactionId);
if (tx == null) return TransactionResolution.UNKNOWN;
return tx.committed()
    ? TransactionResolution.COMMIT
    : TransactionResolution.ROLLBACK;
```

### 面试官递进追问

1. 半消息对消费者是否可见？
2. A 成功、B 消费持续失败怎么办？

### 复习记忆钩子

**半消息先占位，本地事务给结论，回查读耐久事实；下游仍要幂等。**

---

## Q26. Kafka 与 RocketMQ 如何选型？确认与 Broker 架构有何差异？

**题目定位**：高频对比题｜困难｜频率 ⭐⭐⭐⭐⭐

### 为什么现在问这道题？

这道题位于从“会使用 MQ”走向“能解释失败窗口”的关键位置。回答时必须依次覆盖：结论与边界、项目场景、底层机制、故障与恢复、监控验证、方案取舍。

### 2～3 分钟优秀回答

Kafka 更像分布式事件日志和流平台，优势是高吞吐、长时间回放、多 Consumer Group、Connect/Streams 生态和数据管道；RocketMQ 更偏业务消息中间件，FIFO、延时、事务消息、重试/DLQ 与 Java 业务开发体验更完整。不能用固定 TPS 或“Kafka 会丢、RocketMQ 不丢”做结论，两者可靠性都取决于配置和部署。

存储上，Kafka 每个 Topic-Partition 是独立日志段；RocketMQ 正文统一写 CommitLog，ConsumeQueue 提供逻辑消费索引。协调上，Kafka 4.x 使用 KRaft Controller；RocketMQ 使用 NameServer 提供路由，HA 还涉及 Broker/Controller 等部署模式。确认上，Kafka Producer 通过 `acks` 与 ISR 控制写确认，消费者提交 Offset；RocketMQ 还显式提供消费结果、重试和 DLQ 等业务化机制，刷盘与复制是独立维度。

NexusAgent 默认 Kafka：事件回放、文档流水线和多组订阅更重要。若订单超时、事务消息和 FIFO 业务命令占主导，则选择 RocketMQ。最终必须用同一可靠性目标、消息模型和硬件做 PoC，并考虑组织已有平台和运维成本。

### 可读但会制造事故的 Java 反例

```java
// 反例：用宣传数字和语言偏好做最终选型。
return throughputRequired ? "Kafka" : "RocketMQ";
```

### 企业级改进代码

```java
// 改进：按权重评分并通过故障 PoC 验证。
Decision decision = score(
    replayAndStreamEcosystem,
    fifoDelayTransactionNeeds,
    reliabilitySlo,
    multiTenantIsolation,
    opsAndTeamSkill);
return faultInjectionPoc.verify(decision);
```

### 面试官递进追问

1. Kafka 是否支持事务？
2. RocketMQ 同步刷盘是否等于同步复制？

### 复习记忆钩子

**Kafka 强事件流与回放，RocketMQ 强业务消息语义；可靠性靠配置与验证。**

---

## Q27. RabbitMQ 只需掌握哪些内容？什么时候选它？

**题目定位**：选型题｜中等｜频率 ⭐⭐⭐

### 为什么现在问这道题？

这道题位于从“会使用 MQ”走向“能解释失败窗口”的关键位置。回答时必须依次覆盖：结论与边界、项目场景、底层机制、故障与恢复、监控验证、方案取舍。

### 2～3 分钟优秀回答

RabbitMQ 只需掌握交换机、队列、绑定、路由键、确认、持久化和死信。它基于 AMQP 路由模型，Direct、Topic、Fanout 等 Exchange 能灵活把消息路由到不同 Queue，适合中小规模业务解耦、复杂路由和已有 RabbitMQ 运维体系。

与 Kafka 的日志模型不同，RabbitMQ 更像消息代理；与 RocketMQ 相比，它的路由表达非常灵活，但在海量可回放事件流、分区日志和大数据生态上不是首选。不能机械背“微秒延迟”或“百万队列”，要在真实消息大小、确认、持久化、镜像/Quorum Queue 和消费者行为下压测。

本教材不把 RabbitMQ 作为主攻。面试时能说明：如果团队已有成熟 RabbitMQ、吞吐中等、需要 AMQP 灵活路由和任务队列，我会选择它；如果是数据平台事件流选 Kafka，如果是 Java 业务消息、FIFO/延时/事务消息选 RocketMQ。

### 可读但会制造事故的 Java 反例

```java
// 反例：生产者直接写一个无持久化临时队列，重启即丢。
channel.queueDeclare("task", false, false, true, null);
```

### 企业级改进代码

```java
// 改进：关键队列使用耐久配置，并结合 publisher confirm。
channel.queueDeclare("task", true, false, false, arguments);
channel.confirmSelect();
```

### 面试官递进追问

1. Exchange 与 Queue 是什么关系？
2. RabbitMQ 何时不适合替代 Kafka？

### 复习记忆钩子

**RabbitMQ 记路由与任务队列；主线仍是 Kafka，业务消息再看 RocketMQ。**

---

## Q28. 消息队列与观察者、发布订阅模式是什么关系？

**题目定位**：核心面试题｜中等｜频率 ⭐⭐⭐⭐

### 为什么现在问这道题？

这道题位于从“会使用 MQ”走向“能解释失败窗口”的关键位置。回答时必须依次覆盖：结论与边界、项目场景、底层机制、故障与恢复、监控验证、方案取舍。

### 2～3 分钟优秀回答

观察者模式通常是进程内对象关系：Subject 持有或调用 Observer，通知往往同步，生命周期在同一应用内。发布订阅通过 Broker 或事件总线把发布者和订阅者解耦，双方不直接知道彼此，可以跨进程、持久化、重试和独立扩缩容。

Kafka、RocketMQ 和 RabbitMQ 都支持发布订阅，但具体消费语义不完全相同。Consumer Group 使同一订阅内多个实例分摊消息，不同 Group 独立获得消息流；这比“发布者循环通知所有观察者”复杂得多。MQ 还涉及消息保留、确认、重试、顺序和高可用，因此不能只用设计模式解释其工程能力。

项目中，领域服务发布 `AgentRunCompleted`，审计、通知和评估分别使用不同 Group 订阅，生产者不依赖这些服务。契约仍通过 Schema、版本和治理中心管理，所以发布订阅是降低运行依赖，不是消除所有耦合。

### 可读但会制造事故的 Java 反例

```java
// 反例：订单服务直接持有所有下游客户端，新增订阅者必须改代码。
observers.forEach(o -> o.onPaymentConfirmed(order));
```

### 企业级改进代码

```java
// 改进：发布稳定领域事件，下游使用独立 Consumer Group。
producer.send(new ProducerRecord<>(
    "payment.confirmed", order.id(), eventSerializer.serialize(event)));
```

### 面试官递进追问

1. 发布订阅是否完全无耦合？
2. 同组和不同组的消息语义有何区别？

### 复习记忆钩子

**观察者多为进程内直接通知，发布订阅靠 Broker 跨进程解耦。**

---

## Q29. 让你从零设计一个 MQ，如何回答？

**题目定位**：资深工程师系统设计题｜困难｜频率 ⭐⭐⭐⭐

### 为什么现在问这道题？

这道题位于从“会使用 MQ”走向“能解释失败窗口”的关键位置。回答时必须依次覆盖：结论与边界、项目场景、底层机制、故障与恢复、监控验证、方案取舍。

### 2～3 分钟优秀回答

我会先明确目标：消息大小、吞吐、延迟、保留、顺序、投递语义、租户数、RPO/RTO 和跨机房需求。然后分八层设计。

第一，API 与协议：Producer Send、Consumer Fetch/Ack、Admin，定义批量、压缩、超时和错误码。第二，元数据：Topic、Partition、Replica、Consumer Group 与权限，由一致性控制面管理。第三，存储：追加日志、段文件、索引、校验、Page Cache、刷盘和清理。第四，副本：Leader/Follower、同步集合、选主和机架隔离。第五，消费：拉取、Offset、组协调、分区分配和重平衡。第六，语义：重试、幂等 Producer、事务、顺序与 DLQ。第七，伸缩：分区迁移、限流、配额和热点治理。第八，运维：监控、审计、备份、故障演练和升级兼容。

我还会主动说明无法同时最大化吞吐、强一致、低延迟和高可用。初版可先实现单 Broker 追加日志和 Offset，再逐步增加分区、副本与协调面。系统设计回答的加分点不是背 Kafka，而是明确状态保存在哪里、哪个 ACK 证明什么、每个崩溃点如何恢复以及怎样验收。

### 可读但会制造事故的 Java 反例

```java
// 反例：Broker 收到消息只放内存队列，返回成功后重启即丢。
memoryQueue.add(message);
return Ack.SUCCESS;
```

### 企业级改进代码

```java
// 改进伪代码：追加日志、校验、满足持久化/副本策略后再确认。
LogPosition pos = commitLog.append(batch);
flushPolicy.await(pos);
replicationPolicy.awaitRequiredReplicas(pos);
return Ack.success(pos);
```

### 面试官递进追问

1. 控制面与数据面如何拆分？
2. 怎样设计幂等 Producer 序列号？

### 复习记忆钩子

**先给 SLO，再画协议、存储、副本、消费、语义、伸缩和运维。**

---

## Q30. 深挖 NexusAgent：如何设计一条可恢复异步执行链？

**题目定位**：项目深挖题｜困难｜频率 ⭐⭐⭐⭐⭐

### 为什么现在问这道题？

这道题位于从“会使用 MQ”走向“能解释失败窗口”的关键位置。回答时必须依次覆盖：结论与边界、项目场景、底层机制、故障与恢复、监控验证、方案取舍。

### 2～3 分钟优秀回答

NexusAgent 的目标是不让网关超时、Pod 重启、Broker 切换或外部工具结果未知造成任务丢失和重复副作用。入口先用 `idempotencyKey` 创建唯一 `runId`，在一个 MySQL 事务写 `agent_run` 与 `outbox_event`。Relay 使用同一 eventId 发布 `agent.run.command`，Key 为 runId。

Kafka 使用多副本和可靠 Producer 配置。Agent Worker 收到后，在事务中插入 Inbox，并用 `runId + version` 抢占合法状态；重复事件返回已有结果，旧版本拒绝推进。调用外部 MCP 工具时使用稳定 `operationId`，超时先查询工具侧状态，不能换新 ID 重试。业务提交后再提交 Offset。

失败处理分层：瞬时错误进入有界重试；永久错误进入终态和 DLQ；消费者停机通过 Lag 和最老年龄告警；Outbox 超龄由 Reconciler 重发；状态不一致按 runId/eventId/operationId 对账。监控把 Trace 与业务状态关联，但 Trace 不是业务真相。顺序只保证同一 runId，避免全局单分区。

面试时我会声明这是工程设计，不冒充生产经历；给出测试矩阵：发送 ACK 前杀进程、业务提交后 Offset 前杀进程、Broker Leader 切换、工具超时、重平衡和 DLQ 回放，并用“无漏 Run、无重复副作用、状态不倒退、SLO 内恢复”作为验收标准。

### 可读但会制造事故的 Java 反例

```java
// 反例：异常就创建新 Run 和新 operationId 重试，产生重复工具副作用。
catch (Exception e) {
    executeTool(UUID.randomUUID().toString(), request);
}
```

### 企业级改进代码

```java
// 改进：复用稳定身份，结果未知先查询。
ToolResult result = toolClient.execute(operationId, request);
if (result.isUnknown()) {
    result = toolClient.query(operationId);
}
toolOperationRepository.record(operationId, result);
```

### 面试官递进追问

1. 业务已提交但 Offset 未提交会怎样？
2. 如何证明回放没有重复调用工具？

### 复习记忆钩子

**Run、event、operation 三个身份稳定；UNKNOWN 先查询，恢复靠耐久事实。**

---

# 第四篇：学习验收与复习路线

## 1. 70% 掌握标准

完成教材后，至少做到：

- 不看答案，能在 2～3 分钟回答 Q01～Q13；
- 能画出 Producer → Broker → Partition → Consumer Group → Offset；
- 能解释 ACK、刷盘、复制、Offset、Retention 五者不是一回事；
- 能用 Outbox/Inbox 回答不丢与重复；
- 能用净追赶速率回答消息积压；
- 能说清 Kafka Transaction 与 RocketMQ 事务消息的边界；
- 能根据事件流和业务消息需求做 Kafka/RocketMQ 选型；
- 能用 NexusAgent 讲出至少三个崩溃点和恢复证据。

## 2. 三轮复习

### 第一轮：概念与主链

只学 Q01～Q10，要求能解释所有首次出现术语。

### 第二轮：可靠性与场景

学习 Q11～Q20，手画四张图：

1. 消息不丢四段图；
2. 幂等消费事务图；
3. Outbox/Inbox 图；
4. 积压定位与恢复图。

### 第三轮：对比与深挖

学习 Q21～Q30，录音练习 2～3 分钟回答。每次回答必须包含一个边界和一个取舍，避免只背结论。

## 3. 面试回答统一模板

```text
1. 先给结论，并声明保证边界；
2. 放到一个真实可信的项目场景；
3. 解释 Producer/Broker/Consumer 或事务状态变化；
4. 说明至少两个故障窗口；
5. 给出幂等、重试、对账与监控；
6. 最后说明代价、替代方案和验证方式。
```

## 4. 官方事实基线

- [Apache Kafka 4.3.x Documentation](https://kafka.apache.org/43/)
- [Apache Kafka 4.3 Release Announcement](https://kafka.apache.org/blog/2026/05/22/apache-kafka-4.3.0-release-announcement/)
- [Apache Kafka Producer Configuration](https://kafka.apache.org/43/generated/producer_config.html)
- [Apache Kafka Consumer Rebalance Protocol](https://kafka.apache.org/43/operations/consumer-rebalance-protocol/)
- [Apache Kafka Transaction Protocol](https://kafka.apache.org/43/operations/transaction-protocol/)
- [Apache RocketMQ 5.5.0 Release Notes](https://rocketmq.apache.org/release-notes/)
- [Apache RocketMQ 5.0 Concepts](https://rocketmq.apache.org/docs/introduction/02concepts/)
- [Apache RocketMQ FIFO Message](https://rocketmq.apache.org/docs/featureBehavior/03fifomessage/)
- [Apache RocketMQ Delay Message](https://rocketmq.apache.org/docs/featureBehavior/02delaymessage/)
- [Apache RocketMQ Transaction Message](https://rocketmq.apache.org/docs/featureBehavior/04transactionmessage/)
- [Apache RocketMQ Consumption Retry](https://rocketmq.apache.org/docs/featureBehavior/10consumerretrypolicy/)

> 教材中的版本事实应在后续升级时重新核对。禁止把旧版 ZooKeeper、固定延时等级或早期客户端行为直接套到新版本。
