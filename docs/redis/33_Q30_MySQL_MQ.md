# Redis 8.8 面试题 Q30

## Q30. 综合事故：会话消失、延迟尖刺、MySQL 回源和 MQ 积压怎样串起来

**类型：项目终极深挖题｜难度：中等偏上｜重要性：★★★★★**

### ① 事故背景

NexusAgent 把以下内容都放在一个 Redis Cluster：

- RAG 结果缓存；
- Session；
- 工具结果；
- 租约；
- 幂等记录；
- 热 Checkpoint。

配置：

```text
maxmemory 接近容器上限
allkeys-lru
没有单 Value 上限
所有租户 Key 使用 {tenantId}
客户端超时后重试三次
```

### ② 事故链第一段：数据模型失控

```text
工具结果不断增长
  ↓
单个 Run JSON 达到 20MB
  ↓
HGETALL/GET 大回复
  ↓
Redis 核心路径和网络变慢
  ↓
Lettuce 同连接命令排队
```

### ③ 第二段：内存达到上限

```text
大 Key + 碎片 + 缓冲 + COW
  ↓
达到 maxmemory
  ↓
allkeys-lru 开始淘汰
```

被淘汰的不只是普通缓存，还有 Session、租约和幂等记录。

表现：

- 用户会话消失；
- Worker 认为租约不存在，重复接管；
- 幂等 Key 消失，外部 Tool 重复调用；
- Checkpoint Miss。

### ④ 第三段：回源洪峰

缓存和 Session Miss 后：

```text
请求全部查询 MySQL/Milvus
  ↓
客户端与网关多层重试
  ↓
MySQL 连接池耗尽
  ↓
Milvus P99 上升
```

### ⑤ 第四段：消息积压

消费者处理每条消息都要访问 Redis 和 MySQL。

```text
下游变慢
  ↓
消费 TPS 下降
  ↓
Kafka/RocketMQ Lag 上升
  ↓
更多任务超时和重试
```

最终形成正反馈。

### ⑥ 现场止血顺序

1. 停止新增大结果或切对象存储；
2. 禁止大 Key 全量读取；
3. 降低消息消费者并发和重试；
4. 对回源设置 Semaphore 与租户预算；
5. 非核心请求降级；
6. 扩容 Redis 前确认是否需要迁槽；
7. 用 UNLINK 分批处理大对象；
8. 保护 MySQL；
9. 暂停可能重复副作用的任务；
10. 对幂等与租约状态进行权威对账。

### ⑦ 根因修复

#### 状态分池

```text
普通缓存集群：可淘汰
协调状态集群：noeviction
关键 Checkpoint：权威持久化 + Redis 热缓存
```

#### 数据模型

- Run 元数据小 Hash；
- 工具大结果对象存储；
- 每个对象字节和元素上限；
- 避免 HGETALL 大集合；
- 按最小原子单元设计 Hash Tag。

#### 流量保护

- 单层有界重试；
- 总 Deadline；
- Singleflight；
- 回源 Bulkhead；
- MQ 背压；
- 降级策略。

#### 正确性

- 数据库状态机；
- Fencing；
- 幂等键；
- Outbox/Inbox；
- Checkpoint 版本；
- 故障恢复演练。

### ⑧ 必须建立的监控

Redis：

- used_memory/RSS；
- evicted_keys/expired_keys；
- 大 Key 与回复字节；
- P99；
- 客户端缓冲；
- fork/COW；
- Slot/节点倾斜。

应用：

- Redis Pending；
- 回源 QPS；
- MySQL 连接池；
- Milvus P99；
- 重试次数；
- 幂等冲突；
- 租约丢失；
- Checkpoint 恢复。

MQ：

- 生产/消费 TPS；
- Lag；
- 重试与 DLQ；
- 最老消息年龄。

### ❌ 容易制造事故的写法

```java
// 所有状态都进一个可淘汰集群，异常时无限重试并直接回源。
catch (RedisException e) {
    return retry(3, () -> repository.loadEverything(runId));
}
```

### ✅ 企业级改进示例

```java
FailurePolicy policy = failurePolicy.forRequest(requestType);
return policy.execute(
        () -> cache.readBounded(key),
        () -> originBulkhead.execute(
                () -> singleflight.execute(key, () -> origin.load(id))),
        () -> degradedResult(requestType));
```

### 🎙️ 2～3 分钟优秀回答

这个事故不是一个 Redis 参数问题，而是一条跨组件正反馈链。首先大工具结果和整个 Run 被放入大 Key，产生大回复和客户端队头阻塞；随后数据集、碎片、缓冲和 COW 把内存推到 maxmemory，allkeys-lru 开始淘汰 Session、租约、幂等和 Checkpoint；大量 Miss 又压向 MySQL 与 Milvus，多层重试放大流量，消费者变慢后 Kafka/RocketMQ 继续积压。

止血时我先停止大结果写入和全量读取，限制回源并发、降低消费者并发、降级非核心请求、保护外部副作用，再分批清理大 Key。长期修复是状态分池：普通缓存可淘汰，协调状态独立 noeviction，关键 Checkpoint 有权威持久化；大结果进入对象存储，Key 有大小和 TTL 上限；同时使用总 Deadline、Singleflight、Bulkhead、幂等、Fencing 和 MQ 背压。

监控必须把 Redis 内存、回复字节、回源 QPS、MySQL 连接池、Milvus P99 和 MQ Lag 放在一条链上。

### 面试官可能继续追问

- 为什么简单扩容 Redis 可能无法解决 Hot Slot？
- 恢复后如何确认没有重复执行外部 Tool？

> **记忆句**：大 Key 拖延迟，内存淘汰关键状态，Miss 打爆事实源，消费变慢形成积压；治理必须跨组件闭环。

---

# 30 题复习与 70% 掌握验收

## 第一轮：能用自己的话解释概念

不要背源码名，先回答：

1. Redis 中 Key、Value、TTL、Hit 和 Miss 是什么？
2. String、Hash、Set、ZSet、Stream 分别解决什么？
3. 过期与淘汰有什么区别？
4. Master、Replica、Sentinel、Cluster 分别做什么？
5. 缓存为什么不能替代 MySQL 事实？
6. 锁、Fencing 与幂等为什么不是一回事？
7. 短期记忆、长期记忆与 Checkpoint 有什么区别？

## 第二轮：画出五条主链

### 请求链

```text
Java → Lettuce → 网络 → Redis → Key/对象 → 回复 → Java
```

### 缓存链

```text
Redis Miss → 有界回源 → 版本化回填
数据库提交 → 可靠失效 → TTL 兜底
```

### 高可用链

```text
持久化恢复
+ 复制冗余
+ Sentinel/Cluster 故障转移
+ 事实源对账
```

### 并发正确性链

```text
租约互斥
+ Fencing 拒绝旧写
+ 幂等防重复副作用
+ 版本条件更新
```

### 内存事故链

```text
Key 数/字节增长
→ 大回复与缓冲
→ RSS/COW
→ maxmemory
→ 淘汰或写失败
→ 回源和积压
```

## 第三轮：21 道 70% 主线题

能够独立完成下面 21 题，即达到本教材的基础验收目标：

```text
Q01 Redis 为什么快
Q02 类型、编码和 Key
Q03 大 Key/热 Key/Hot Slot
Q04 端到端延迟
Q05 Pipeline/事务/Lua
Q06 RDB/AOF
Q08 主从复制丢失窗口
Q09 Sentinel/Cluster
Q11 Cache Aside
Q12 旧值回填
Q13 穿透/击穿/雪崩
Q15 Redis 故障保护
Q16 原子计数与 TTL
Q17 分布式锁
Q18 Fencing 与幂等
Q20 Stream 与 MQ
Q21 过期与淘汰
Q22 淘汰策略
Q24 容量与 Lazy Free
Q27 Checkpoint
Q30 综合事故
```

## 2～3 分钟回答通用结构

```text
第一句：定义问题和边界
第二段：按时间线讲机制
第三段：结合项目方案
第四段：补失败窗口和监控
最后一句：给出取舍结论
```

不要只回答：

```text
“用锁”
“加 TTL”
“开主从”
“扩容”
“重试”
```

高分回答必须继续说明：

```text
锁过期怎么办？
TTL 是业务过期还是淘汰？
主从何时确认？
扩容是否受 Slot/热 Key 限制？
重试是否幂等、是否有总 Deadline？
```

## 最终记忆图

```text
Redis 企业级能力
=
数据模型有边界
+ 请求链可观测
+ 生命周期可治理
+ 高可用但不冒充强一致
+ 缓存与事实源可收敛
+ 并发靠版本、Fencing、幂等
+ 内存按数据、缓冲、RSS、COW 规划
+ Agent 状态可恢复、可审计
```
