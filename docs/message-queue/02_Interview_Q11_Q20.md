# Kafka 4.3.x + RocketMQ 5.5 企业级消息队列教材：Q11～Q20

## Q11. `acks`、重试、幂等 Producer 如何一起工作？

**题目定位**：高频面试题｜中等偏上｜频率 ⭐⭐⭐⭐⭐

### 为什么现在问这道题？

这道题位于从“会使用 MQ”走向“能解释失败窗口”的关键位置。回答时必须依次覆盖：结论与边界、项目场景、底层机制、故障与恢复、监控验证、方案取舍。

### 2～3 分钟优秀回答

`acks=0` 不等待 Broker 确认；`acks=1` 只等待 Leader 接收，Leader 在副本同步前故障可能丢数据；`acks=all` 等待当前 ISR 满足确认条件，是最强的内建确认。它还要与 `min.insync.replicas` 配合，否则单副本 ISR 也可能确认成功。

网络超时会产生结果未知：Broker 可能已写入，但 Producer 没收到响应。自动重试可能导致重复，所以 Kafka 使用 Producer ID、Epoch 和 Sequence Number 对单个 Producer 会话中的重试批次去重。现代 Kafka Producer 在无冲突配置时默认开启幂等；要求 `acks=all`、重试开启、`max.in.flight` 不超过约束。

但 Kafka Producer 幂等只避免 Kafka 日志中的部分重复，不会去重应用重新创建的新消息，也不会保证 MySQL 或外部工具只执行一次。业务仍需稳定 `eventId`。配置建议不是死背参数，而是先确定允许的数据丢失、延迟和不可用窗口，再验证超时、Broker 切换和客户端重启下的结果。

### 可读但会制造事故的 Java 反例

```java
// 反例：发送超时后重新生成 eventId，Broker 若已写入就产生业务重复。
catch (TimeoutException e) {
    send(new Event(UUID.randomUUID().toString(), payload));
}
```

### 企业级改进代码

```java
// 改进：复用同一业务身份，并让 Kafka 客户端管理可重试发送。
Event event = new Event(stableEventId, payload);
producer.send(toRecord(event), (meta, ex) -> {
    outboxRepository.recordAttempt(stableEventId, meta, ex);
});
```

### 面试官递进追问

1. `acks=all` 是等待全部副本还是 ISR？
2. 幂等 Producer 跨进程重启的边界是什么？

### 复习记忆钩子

**ACK 管存储确认，序列号管客户端重试，eventId 管业务重复。**

---

## Q12. 如何保证消息不丢失？

**题目定位**：场景面试题｜中等偏上｜频率 ⭐⭐⭐⭐⭐

### 为什么现在问这道题？

这道题位于从“会使用 MQ”走向“能解释失败窗口”的关键位置。回答时必须依次覆盖：结论与边界、项目场景、底层机制、故障与恢复、监控验证、方案取舍。

### 2～3 分钟优秀回答

我会从四段回答，而不是只说 Producer、Broker、Consumer。第一，业务到 Producer：使用 MySQL Outbox，防止业务提交后进程在发送前崩溃；第二，Producer 到 Broker：稳定 eventId、`acks=all`、合理重试与超时；第三，Broker 存储：多副本、`min.insync.replicas`、禁止不安全选主、磁盘与跨机架部署；第四，Consumer 到业务：先提交业务，再提交 Offset，并通过 Inbox/唯一约束实现幂等。

即便四段都做了，也不应绝对承诺“永不丢”。磁盘多点损坏、错误删除、保留期过短、管理员重置 Offset、程序跳过异常记录都可能造成业务丢失。因此还需要对账：比较业务事实、Outbox、Kafka Offset、Inbox 和最终业务状态，发现静默缺口。

NexusAgent 中，任何已返回 `runId` 的请求必须有 MySQL `agent_run` 与 `outbox_event`。Relay 可重复发送，Worker 用 `eventId` 和 `version` 裁决。告警看 Outbox 超龄、Under-replicated、Lag 和状态对账差异。这样回答能说明“尽可能不丢”的工程闭环，而不是把某个 MQ 配置夸大成全局保证。

### 可读但会制造事故的 Java 反例

```java
// 反例：业务先提交，再直接发 Kafka；中间宕机会永久漏发。
orderRepository.markPaid(orderId);
producer.send(paymentEvent(orderId));
```

### 企业级改进代码

```java
// 改进：同一事务保存订单状态和 Outbox。
@Transactional
public void markPaid(String orderId) {
    orderRepository.markPaid(orderId);
    outboxRepository.insert(stableEventId(orderId), "PaymentConfirmed", orderId);
}
```

### 面试官递进追问

1. Outbox Relay 发送后、标记前崩溃怎么办？
2. 消费端先提交 Offset 有何风险？

### 复习记忆钩子

**不丢要跨四段：业务意图、发送、存储、消费，再用对账兜底。**

---

## Q13. 为什么会重复消费？如何设计业务幂等？

**题目定位**：场景面试题｜中等偏上｜频率 ⭐⭐⭐⭐⭐

### 为什么现在问这道题？

这道题位于从“会使用 MQ”走向“能解释失败窗口”的关键位置。回答时必须依次覆盖：结论与边界、项目场景、底层机制、故障与恢复、监控验证、方案取舍。

### 2～3 分钟优秀回答

重复是 At-least-once 系统的正常现象。常见原因包括：Producer 超时重试；Consumer 业务已提交但 Offset 未提交就崩溃；Rebalance 期间进度重复；手工回放；Relay 发送成功但 Outbox 状态未更新。目标不是幻想完全没有重复，而是让重复不会改变最终业务效果。

幂等策略按业务选择。插入型业务优先数据库唯一索引；状态推进使用版本号或合法状态机条件；金额扣减使用业务流水唯一键与事务；外部工具使用 `operationId` 并支持查询；Redis `SETNX` 只能做加速或短期门闩，不能在关键业务中替代数据库事实，因为过期、淘汰和数据库事务失败会造成偏差。

推荐消费者在一个本地事务中插入 Inbox、校验版本、修改业务。唯一冲突表示已经处理，直接返回成功。还要校验同一 eventId 的 Payload Hash，避免调用方错误复用幂等键却携带不同参数。面试中我会强调：幂等不是“查一下再写”，而是依靠原子唯一约束或条件更新裁决并发。

### 可读但会制造事故的 Java 反例

```java
// 反例：先查后写存在并发窗口，两台消费者都可能通过检查。
if (!redis.hasKey(eventId)) {
    accountRepository.debit(accountId, amount);
    redis.set(eventId, "done");
}
```

### 企业级改进代码

```java
// 改进：唯一约束与业务更新放在同一数据库事务。
@Transactional
public void consume(DebitEvent e) {
    if (!inboxRepository.tryInsert(e.eventId(), e.payloadHash())) return;
    accountRepository.debitWithLedger(e.accountId(), e.amount(), e.eventId());
}
```

### 面试官递进追问

1. 为什么分布式锁不是幂等首选？
2. 幂等键被复用但参数不同怎么办？

### 复习记忆钩子

**重复不可怕；稳定身份 + 原子裁决才是幂等。**

---

## Q14. At-most-once、At-least-once、Exactly-once 有什么边界？

**题目定位**：资深工程师题｜中等偏上｜频率 ⭐⭐⭐⭐

### 为什么现在问这道题？

这道题位于从“会使用 MQ”走向“能解释失败窗口”的关键位置。回答时必须依次覆盖：结论与边界、项目场景、底层机制、故障与恢复、监控验证、方案取舍。

### 2～3 分钟优秀回答

At-most-once 通常先提交进度再处理，可能丢但不重复；At-least-once 先处理再提交进度，可能重复但尽量不丢；Exactly-once 必须明确范围。Kafka 的幂等 Producer 可保证特定 Producer 会话的日志去重，Kafka Transaction 可原子提交多个 Kafka 分区的记录和消费 Offset，配合 `read_committed` 实现 Kafka-to-Kafka 处理的一次语义。

但当链路包含 MySQL、Milvus、HTTP 支付、邮件或 MCP 工具时，Kafka 事务无法自动把这些外部系统纳入一个原子提交。因此工程上更常说“Exactly-once business effect”：消息允许重复投递，但数据库唯一约束、状态版本、Outbox/Inbox 和外部 operationId 使最终业务效果等同一次。

回答这题要避免两个极端：一是说 Exactly-once 完全不可能，忽略 Kafka 边界内的事务能力；二是说开启 `enable.idempotence=true` 就实现全链路一次。正确方式是画出事务边界、列出崩溃点并说明恢复证据。

### 可读但会制造事故的 Java 反例

```java
// 反例：认为开启幂等 Producer 后，数据库写也天然 exactly-once。
props.put("enable.idempotence", true);
database.updateBalance(event);
producer.send(nextRecord(event));
```

### 企业级改进代码

```java
// 改进：数据库侧仍用业务流水唯一键；Kafka 事务只覆盖 Kafka 边界。
@Transactional
public void applyBusiness(Event e) {
    ledgerRepository.insertUnique(e.eventId(), e.amount());
    accountRepository.apply(e);
}
```

### 面试官递进追问

1. Kafka Transaction 能否原子提交 MySQL？
2. `read_committed` 有什么作用？

### 复习记忆钩子

**先圈定边界，再谈 Exactly-once；跨系统通常追求一次业务效果。**

---

## Q15. MySQL 与 Kafka 如何保证最终一致？Outbox 为什么更通用？

**题目定位**：项目深挖题｜困难｜频率 ⭐⭐⭐⭐⭐

### 为什么现在问这道题？

这道题位于从“会使用 MQ”走向“能解释失败窗口”的关键位置。回答时必须依次覆盖：结论与边界、项目场景、底层机制、故障与恢复、监控验证、方案取舍。

### 2～3 分钟优秀回答

核心问题是双写：先写 MySQL 后发 Kafka，中间崩溃会漏消息；先发 Kafka 后写 MySQL，会出现消息已见但业务回滚。Outbox 模式把业务表和事件意图写在同一个 MySQL 本地事务中，再由 Relay 异步发布。这样只要业务事实成功，事件意图就一定可恢复。

Relay 可以轮询、CDC 或专门连接器发布。发送成功但更新 Outbox 状态前崩溃，会产生重复发布，因此 eventId 必须稳定，消费者使用 Inbox/唯一约束幂等。还需要处理卡死、分片扫描、锁竞争、失败退避、保留归档和对账。

Kafka Transaction 适合 Kafka-to-Kafka 原子处理，但不能直接替代 MySQL Outbox。双写一致性要根据数据源和事务边界选方案。NexusAgent 中 `agent_run` 与 `outbox_event` 同事务，Relay 发布 `AgentRunRequested`，Worker 的 `inbox_record` 与 Run 状态更新同事务。任何 UNKNOWN 结果都通过 eventId 查询与对账，不生成新 Run。

### 可读但会制造事故的 Java 反例

```java
// 反例：本地事务提交后同步发 Kafka，中间故障无恢复证据。
@Transactional
public void createRun(Run run) {
    runRepository.insert(run);
}
producer.send(runRequested(run)); // 这里失败会漏发
```

### 企业级改进代码

```java
// 改进：业务事实与事件意图同事务。
@Transactional
public void createRun(Run run) {
    runRepository.insert(run);
    outboxRepository.insert(run.eventId(), "AgentRunRequested", run.toJson());
}
```

### 面试官递进追问

1. Outbox 表会不会成为瓶颈？
2. CDC Relay 与轮询 Relay 如何选？

### 复习记忆钩子

**一个本地事务保存事实与意图，异步发布允许重复，消费端幂等。**

---

## Q16. Kafka 消费成功与 Offset 提交顺序怎么设计？

**题目定位**：高频面试题｜中等偏上｜频率 ⭐⭐⭐⭐⭐

### 为什么现在问这道题？

这道题位于从“会使用 MQ”走向“能解释失败窗口”的关键位置。回答时必须依次覆盖：结论与边界、项目场景、底层机制、故障与恢复、监控验证、方案取舍。

### 2～3 分钟优秀回答

先提交 Offset 再处理业务会在进程崩溃时丢消息；先处理业务再提交 Offset 会在 ACK/提交前崩溃时重复。因此核心策略是 At-least-once + 幂等：业务事务成功后再提交相应 Partition 的下一 Offset。

批量消费最危险。若 100 条中第 50 条失败，却提交到第 100 条，失败记录被跳过；若完全不提交，前 49 条会重复。常见处理包括按 Partition 顺序处理并只提交连续成功水位；暂停失败分区，将失败记录发送到重试通道；或把业务工作转移到有完成跟踪的分区执行器。

`commitSync` 与 `commitAsync` 的选择也不是“同步一定好”。同步易理解但增加延迟；异步吞吐高却要处理回调乱序和最终同步提交。无论哪种，Offset 只是读取进度，业务成功证据仍应在数据库。滚动发布和 Rebalance 前还要尽力提交已完成水位。

### 可读但会制造事故的 Java 反例

```java
// 反例：拉到消息后立即提交，业务失败会造成逻辑丢失。
ConsumerRecords<String, Event> records = consumer.poll(Duration.ofSeconds(1));
consumer.commitSync();
records.forEach(this::process);
```

### 企业级改进代码

```java
// 改进：按分区处理，只提交连续成功的下一 Offset。
Map<TopicPartition, OffsetAndMetadata> offsets = processInPartitionOrder(records);
consumer.commitSync(offsets);
```

### 面试官递进追问

1. 异步提交回调为什么可能乱序？
2. 批次中间失败如何不阻塞所有分区？

### 复习记忆钩子

**业务先成功，Offset 后推进；只提交连续完成水位。**

---

## Q17. Kafka 如何设计重试 Topic、DLQ 和回放？

**题目定位**：项目深挖题｜中等偏上｜频率 ⭐⭐⭐⭐

### 为什么现在问这道题？

这道题位于从“会使用 MQ”走向“能解释失败窗口”的关键位置。回答时必须依次覆盖：结论与边界、项目场景、底层机制、故障与恢复、监控验证、方案取舍。

### 2～3 分钟优秀回答

Kafka 核心日志没有像业务 MQ 那样统一规定单条自动阶梯重试，因此应用常用重试 Topic：主消费失败后按错误类型写入 `retry-1m`、`retry-10m` 等通道，到期后重新进入处理；超过次数或遇到永久数据错误进入 DLQ。

关键不是 Topic 名，而是状态机。消息必须保留原 eventId、原 Topic/Partition/Offset、重试次数、下次时间、错误类型和 Payload Hash。瞬时错误才重试；权限拒绝、Schema 错误或非法状态不应无限重试。重试发布与当前 Offset 推进之间也要避免双写窗口，可使用 Kafka Transaction 覆盖 Kafka-to-Kafka，或使用数据库状态表与 Outbox。

DLQ 需要可运营：告警、Owner、修复工具、审批回放、隔离 Consumer Group、速率限制和对账。直接把死信再塞回主 Topic 可能形成故障风暴。回放前应修复根因并估算下游容量。

### 可读但会制造事故的 Java 反例

```java
// 反例：任何异常都原地无限重试，毒消息阻塞分区。
while (true) {
    try { process(record); break; }
    catch (Exception ignored) {}
}
```

### 企业级改进代码

```java
// 改进：分类、有界、保留原身份。
RetryDecision d = retryClassifier.classify(exception);
if (d.retryable() && record.attempt() < d.maxAttempts()) {
    retryPublisher.publish(record.nextAttempt(d.nextTime()));
} else {
    dlqPublisher.publish(record.withFailure(d.reason()));
}
```

### 面试官递进追问

1. 重试 Topic 如何保证不重复？
2. 为什么 DLQ 回放必须限速？

### 复习记忆钩子

**先分类，再有界重试；DLQ 要能运营、修复和审计。**

---

## Q18. Kafka 积压几百万条如何处理？

**题目定位**：场景面试题｜困难｜频率 ⭐⭐⭐⭐⭐

### 为什么现在问这道题？

这道题位于从“会使用 MQ”走向“能解释失败窗口”的关键位置。回答时必须依次覆盖：结论与边界、项目场景、底层机制、故障与恢复、监控验证、方案取舍。

### 2～3 分钟优秀回答

先止损和定位，不要立即加机器。第一步确认积压范围：哪个 Topic、Partition、Consumer Group，Lag 增长斜率和最老消息年龄是多少；第二步区分生产突增、消费者 Bug、依赖变慢、热点 Key、Rebalance 风暴、消息变大或 Broker/磁盘问题；第三步保护下游，避免扩容把 MySQL、Milvus 或第三方 API 冲垮。

修复根因后再提速。如果 Partition 尚有空闲，可以扩 Consumer；否则实例超过 Partition 数无效。还可优化批处理、减少单条远程调用、提高并发但保持分区顺序、暂时关闭非关键分支。若原 Topic 无法安全扩分区或积压处理逻辑需隔离，可建立受控的临时重分发 Topic，但这会改变顺序、增加重复和双写风险，必须保留 eventId 并对账。

用净追赶速率估算恢复时间，而不是说“扩十倍就十倍快”。若消费 15 万/分钟、生产仍有 10 万/分钟，净追赶只有 5 万/分钟。恢复后还要复盘容量、告警、压测和 Runbook；非关键日志若业务批准可跳过，但核心交易消息不能擅自重置 Offset。

### 可读但会制造事故的 Java 反例

```java
// 反例：看到 Lag 就把消费者从 10 台扩到 100 台，不看分区和下游。
k8s.scale("consumer", 100);
```

### 企业级改进代码

```java
// 改进：基于分区余量、净追赶率和下游预算计算扩容。
int usefulInstances = Math.min(topicPartitions, downstreamConcurrencyBudget);
scaleTo(usefulInstances);
estimateCatchUp(lag, consumeRateAfterScale, currentProduceRate);
```

### 面试官递进追问

1. 积压清空时间怎样计算？
2. 为什么扩分区可能破坏同 Key 顺序？

### 复习记忆钩子

**先定位增长原因，再算净追赶；扩容不能越过分区与下游上限。**

---

## Q19. 增加 Partition 就一定能解决积压吗？

**题目定位**：资深工程师题｜困难｜频率 ⭐⭐⭐⭐

### 为什么现在问这道题？

这道题位于从“会使用 MQ”走向“能解释失败窗口”的关键位置。回答时必须依次覆盖：结论与边界、项目场景、底层机制、故障与恢复、监控验证、方案取舍。

### 2～3 分钟优秀回答

不一定。增加 Partition 只提高潜在并行上限，不会自动让消费者逻辑变快，也不会自动把旧记录均匀迁移到新 Partition。若瓶颈在数据库锁、模型限流、单热点 Key、网络或反序列化，扩 Partition 反而放大压力。

更重要的是顺序和 Key 映射。默认哈希通常与 Partition 数相关，扩容后同一个 Key 的新消息可能路由到新 Partition，而旧消息仍在旧 Partition，跨时间顺序被打破。需要冻结写入迁移、版本化分区策略、创建新 Topic 或在业务层使用版本裁决。

Partition 过多还增加 Controller 元数据、Leader 切换、文件句柄、恢复和 Rebalance 成本。企业做法是从单 Partition 基准吞吐、平均消息大小、峰值倍率、目标恢复时间和热点系数反推，并预留增长空间。扩容前后必须压测 Produce/Fetch P99、ISR、Rebalance 和下游正确性。

### 可读但会制造事故的 Java 反例

```java
// 反例：线上直接把 12 个分区改成 120 个，仍认为 orderId 全历史有序。
admin.increasePartitions("order-event", 120);
```

### 企业级改进代码

```java
// 改进：迁移到新 Topic，并通过版本/双读阶段验证。
migration.createTopic("order-event-v2", 120);
migration.dualWriteWithStableEventId();
migration.verifyOffsetsAndOrderVersions();
migration.switchConsumers();
```

### 面试官递进追问

1. Kafka 增加 Partition 会重分布旧数据吗？
2. 如何避免热点 Key？

### 复习记忆钩子

**Partition 是并行上限，不是万能加速器；扩容先处理顺序与下游。**

---

## Q20. Kafka 如何做监控、容量规划和压测？

**题目定位**：资深工程师题｜困难｜频率 ⭐⭐⭐⭐

### 为什么现在问这道题？

这道题位于从“会使用 MQ”走向“能解释失败窗口”的关键位置。回答时必须依次覆盖：结论与边界、项目场景、底层机制、故障与恢复、监控验证、方案取舍。

### 2～3 分钟优秀回答

监控分三层。Broker 层看请求延迟、队列、网络、磁盘、GC、ISR 缩减、Under-replicated、Offline Partition 和 Controller 状态；Producer 看发送速率、批次大小、压缩率、重试和错误；Consumer 看 Lag、最老消息年龄、Poll、Commit、Rebalance 与处理时延。最终还要看业务完成 P99、失败率和对账差异。

容量至少包含峰值消息数、平均与 P99 字节、保留时间、副本因子、压缩比、流量增长、热点、故障冗余和追赶窗口。存储可粗略估算为 `每日写入字节 × 保留天数 × 副本因子 ÷ 压缩比`，再加索引、段文件、预留空间与重平衡余量。消费者容量由单条耗时和下游预算决定。

压测要覆盖稳定流、阶跃突发、大消息、热点 Key、消费者停机、Broker 故障、滚动升级和回放。不能只报峰值 TPS；要报告 P95/P99、错误率、Lag 增长与清空时间、ISR 变化和业务正确性。阈值来自 SLO 与处置时间，例如“最老消息年龄距离业务 SLA 还剩多少时间”通常比单纯 Lag 更有意义。

### 可读但会制造事故的 Java 反例

```java
// 反例：只用消息条数告警，不考虑消息年龄、大小和业务时限。
if (lag > 1_000_000) alert();
```

### 企业级改进代码

```java
// 改进：组合增长率、最老年龄和剩余处置时间。
if (oldestAge.compareTo(sla.minus(responseBudget)) > 0
        || lagGrowthRate > baselineGrowthRate * 3) {
    alertWithRunbook(topic, group, oldestAge, lagGrowthRate);
}
```

### 面试官递进追问

1. Lag 很大但为何可能不紧急？
2. 磁盘容量为什么要乘副本因子？

### 复习记忆钩子

**监控看原因、结果、正确性；容量看字节、保留、副本和追赶。**

---
