# Redis 8.8 面试题 Q20

## Q20. Redis Stream 与 Kafka、RocketMQ 应怎样选择

**类型：消息与 Redis 交叉题｜难度：中等｜重要性：★★★★☆**

### ① 先理解普通 List 的不足

使用 List 做任务队列：

```text
生产者 LPUSH
消费者 BRPOP
```

可以工作，但当消费者拿走消息后崩溃：

```text
谁负责？
是否完成？
怎样重新投递？
```

需要应用自己补很多机制。

### ② Stream 增加了什么

Stream 是追加日志，可使用消费组。

核心概念：

- Stream Entry：一条消息；
- Consumer Group：共同处理的一组消费者；
- Consumer：具体实例；
- PEL：已投递但未 ACK 的消息；
- `XACK`：确认完成；
- `XAUTOCLAIM`：认领长时间未完成消息；
- Redis 8.8 `XNACK`：显式释放待处理消息。

### ③ Stream 的适用场景

- Redis 已是系统基础设施；
- 轻量、低到中等规模事件；
- 状态与事件希望在 Redis 内协同；
- 可以接受自己治理容量、重试和高可用；
- 简单内部任务。

### ④ 为什么企业核心异步链仍常选 Kafka/RocketMQ

专业 MQ 提供更成熟的：

- 大规模吞吐；
- 长期保留；
- 跨服务生态；
- 分区/队列扩展；
- 消费 Lag；
- 重试与死信；
- 跨集群治理；
- 运维工具。

Kafka 更适合高吞吐事件流、日志、数据管道；RocketMQ 更贴近 Java 业务消息、延时、事务和重试；RabbitMQ 适合灵活路由和传统消息代理场景。

### ⑤ Stream 也不是“消息绝不丢”

需要考虑：

- Redis 持久化；
- 主从复制窗口；
- Key 淘汰；
- `MAXLEN` 裁剪；
- PEL 积压；
- 消费者故障；
- ACK 时机；
- Cluster 与 Key 热点。

### ⑥ 项目选型

NexusAgent：

```text
本地轻量状态通知
  → Redis Stream 可选

跨服务核心任务链、审计事件
  → Kafka / RocketMQ

支付、外部副作用
  → 事实源 + Outbox + 专业 MQ + 幂等
```

### ❌ 容易制造事故的写法

```java
// 消费后立即 ACK，再执行外部 Tool。
List<Message> messages = stream.readGroup(...);
stream.ack(messages);
callExternalTool(messages); // 此处失败，消息已经确认
```

### ✅ 企业级改进示例

```java
for (Message message : stream.readGroup(...)) {
    try {
        processIdempotently(message);
        stream.ack(message.id()); // 业务成功后再确认
    } catch (RetryableException e) {
        // 保留在 PEL，稍后重试或主动 XNACK。
    }
}
```

### 🎙️ 2～3 分钟优秀回答

Redis Stream 是带消费组和待确认状态的追加日志。消费者读取后，未 ACK 的消息进入 PEL，可以通过 XACK 确认、XAUTOCLAIM 重新认领，Redis 8.8 还提供 XNACK 显式释放待处理消息。它比普通 List 更适合需要消费进度和重投的轻量任务。

但 Stream 不自动等于企业 MQ。持久化、复制窗口、Key 淘汰、MAXLEN 裁剪、PEL 积压和 Cluster 热点都需要治理。Kafka 更适合高吞吐事件流和数据管道，RocketMQ 更适合 Java 业务消息、延时和事务场景。

项目中我会把 Redis Stream 用于规模可控、与 Redis 状态紧密相关的内部事件；跨服务核心异步执行链使用 Kafka 或 RocketMQ，并配合 Outbox、幂等、重试和审计。

### 面试官可能继续追问

- PEL 中消息一直不 ACK 会怎样？
- Stream 的 MAXLEN 裁剪为什么可能影响未消费消息？

> **记忆句**：List 只有队列；Stream 多了消费组和签收；核心跨服务链仍优先专业 MQ。

---

# Redis 面试题 Q21～Q30 与复习验收

## 学习顺序

这一部分不再只问“命令怎么用”，而是进入生产环境真正容易出事故的四条链：

```text
生命周期链：过期 → 淘汰 → 内存不足
内存链：数据集 → RSS → 碎片 → 缓冲 → COW
并发链：锁 → 版本 → Fencing → 幂等
Agent 链：短期记忆 → Checkpoint → 恢复 → 语义缓存
```

每道题都要求回答边界、取舍和验证方法。
