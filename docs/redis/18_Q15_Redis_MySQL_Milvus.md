# Redis 8.8 面试题 Q15

## Q15. Redis 故障时怎样保护 MySQL、Milvus 和消息消费链

**类型：资深项目深挖题｜难度：中等偏上｜重要性：★★★★★**

### ① 为什么 Redis 故障会扩大成全系统事故

正常：

```text
90% 请求命中 Redis
10% 请求回源
```

Redis 故障后：

```text
100% 请求回源
```

如果客户端还自动重试三次：

```text
原始流量 × 多层重试
```

MySQL、Milvus、线程池和连接池会迅速耗尽，RocketMQ/Kafka 消费也会因为下游变慢而积压。

### ② 第一原则：失败要快

Redis 不是所有请求都值得等待几秒。

- 设置短连接/命令超时；
- 使用统一总 Deadline；
- 避免 SDK、业务、网关每层独立重试；
- 非幂等操作不盲目重试；
- 超时后快速进入降级路径。

### ③ 第二原则：回源必须有预算

使用：

- Semaphore；
- Bulkhead；
- 租户级限流；
- 热点 Singleflight；
- 优先级队列；
- 只允许核心请求回源。

例如 MySQL 只允许额外承受 300 QPS，就不能在 Redis 故障时放入 5000 QPS。

### ④ 第三原则：数据分级降级

| 数据 | Redis 故障时 |
|---|---|
| 普通结果缓存 | 回源但受限 |
| 推荐与统计 | 返回旧值或默认值 |
| Session | 尝试事实源或要求重新登录 |
| 权限 | 不确定时拒绝或查权威源 |
| 限流状态 | 根据安全目标 Fail-Open/Fail-Closed |
| 租约 | 暂停接管，避免双执行 |
| 幂等记录 | 不确定时不执行外部副作用 |

### ⑤ 第四原则：消息消费者要背压

消费者处理消息时依赖 Redis。

Redis 故障后不应无限拉取：

```text
降低并发
暂停部分分区/队列
延长重试
保护 MySQL
让 MQ 作为暂存缓冲
```

### ⑥ 第五原则：演练完整事故链

演练：

```text
Redis 超时
→ 客户端重试
→ MySQL 连接池
→ Milvus 延迟
→ MQ Lag
→ API P99
```

不能只验证“Redis 能否自动切换”，还要验证下游是否被保护。

### ❌ 容易制造事故的写法

```java
try {
    return redis.get(key);
} catch (Exception e) {
    // 每个请求都直接查数据库，无并发上限。
    return repository.load(id);
}
```

### ✅ 企业级改进示例

```java
try {
    return redis.get(key);
} catch (RedisException e) {
    if (!policy.allowOriginFallback(requestType)) {
        return degradedResponse();
    }
    return originBulkhead.execute(() ->
            singleflight.execute(key, () -> repository.load(id)));
}
```

### 🎙️ 2～3 分钟优秀回答

Redis 故障最大的风险不是少一个缓存，而是原本被缓存吸收的流量瞬间压向 MySQL、Milvus 和消息消费者，再叠加多层重试形成放大。

我的处理分五层：第一，Redis 使用短超时和统一总 Deadline，失败要快；第二，回源必须通过 Semaphore、舱壁和租户预算，有明确 QPS 上限；第三，按状态分级降级，例如推荐可返回旧值，权限不确定时查权威源或拒绝，租约和幂等不确定时不能贸然执行副作用；第四，MQ 消费者降低并发或暂停拉取，让消息系统承担缓冲；第五，做包含 MySQL 连接池、Milvus、MQ Lag 和 API P99 的故障演练。

核心不是让所有请求在 Redis 故障时继续成功，而是让系统以可控方式保住关键业务。

### 面试官可能继续追问

- 限流状态在 Redis 故障时应该 Fail-Open 还是 Fail-Closed？
- 消息消费者怎样暂停分区而不丢失进度？

> **记忆句**：Redis 故障先失败快，再限回源，再分级降级，最后让 MQ 背压。

---
