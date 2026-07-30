# 十三、NexusAgent 中 Redis 的位置

```text
用户请求
  ↓
Agent Service
  ├─ Redis：会话、短期记忆、限流、租约、幂等、热缓存
  ├─ MySQL：状态机、用户、审计、版本事实
  ├─ Milvus：向量检索
  ├─ Kafka/RocketMQ：可靠异步事件
  ├─ 对象存储：大工具结果、快照正文
  └─ LLM/Tool：外部副作用
```

## Key 示例

```text
nexus:session:{T1:U1}
nexus:run:{T1:R1}:meta
nexus:run:{T1:R1}:tool-result:search
nexus:rate:{T1}:2026073015
nexus:lease:{T1:R1}
nexus:idempotency:{T1:E1}
nexus:checkpoint:{T1:R1}:v7
```

花括号是 Cluster Hash Tag。只让真正需要多 Key 原子操作的最小单元同槽。

---

# 十四、状态分级

| 级别 | 示例 | Redis 丢失后 |
|---|---|---|
| L0 可重建缓存 | 检索、模型结果 | 回源 |
| L1 临时状态 | Session | 回源或重新建立 |
| L2 协调状态 | 租约、限流、幂等 | 影响正确性，要隔离 |
| L3 热关键状态 | Checkpoint | 权威存储 + Redis 热副本 |
| L4 业务事实 | 订单、支付、审计 | 不只存 Redis |

---

# 十五、进入面试题前的自测

不用专业术语，回答：

1. Key、Value、TTL 分别是什么？
2. Hit 和 Miss 是什么？
3. Redis 为什么不能替代 MySQL？
4. String 与 Hash 的区别？
5. Set 与 ZSet 的区别？
6. List 与 Stream 的区别？
7. 一条 GET 怎样从 Java 到 Redis？
8. 过期、删除和淘汰的区别？
9. RDB、AOF、Replica、Sentinel、Cluster 分别做什么？
10. 哪些 Agent 数据适合 Redis，哪些必须有权威事实源？

不能回答时先复习本文件，不要直接背 2～3 分钟答案。

---

# 十六、30 题学习地图

```text
Q01～Q05：正确使用 Redis
速度、类型、Key、大 Key、端到端延迟、原子工具

Q06～Q10：恢复与高可用
RDB/AOF、fork/COW、复制、Sentinel、Cluster

Q11～Q15：缓存一致性与故障保护
Cache Aside、旧值回填、穿透击穿雪崩、两级缓存、回源保护

Q16～Q20：并发正确性与消息
原子计数、锁、Fencing、限流、Stream

Q21～Q25：生命周期与内存策略
过期、淘汰、内存视角、Lazy Free、Cluster 倾斜

Q26～Q30：Agent 项目深挖
记忆、Checkpoint、并发状态、语义缓存、综合事故
```

---

# 概念速查表

| 概念 | 一句人话 |
|---|---|
| Redis Server | 保存并处理数据的进程 |
| Key | 数据的唯一名字 |
| Value | 真正内容 |
| TTL | 剩余有效时间 |
| Hit | Redis 找到了 |
| Miss | Redis 没找到 |
| String | 整体值 |
| Hash | 多字段对象 |
| List | 简单顺序列表 |
| Set | 不重复集合 |
| ZSet | 带分数排序 |
| Stream | 带消费进度的日志 |
| RDB | 快照 |
| AOF | 写流水 |
| Replica | 副本 |
| Sentinel | 自动故障巡检与切换 |
| Cluster | 多节点分片 |
| Eviction | 内存不足时淘汰 |
