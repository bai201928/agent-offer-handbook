# Redis 8.8 面试题 Q09

## Q09. Sentinel 与 Cluster 有什么区别，项目中怎样选择

**类型：高频选型题｜难度：简单偏中等｜重要性：★★★★★**

### ① 先问项目需要解决什么

```text
问题 A：单个 Master 容量够，但故障时希望自动切换
问题 B：单个 Master 的容量或吞吐已经不够
```

A 更接近 Sentinel，B 才需要 Cluster。

### ② Sentinel 是什么

一套典型结构：

```text
Master
├─ Replica 1
└─ Replica 2

Sentinel 1 / 2 / 3 持续监控
```

Sentinel 负责：

- 监控；
- 判断故障；
- 多个 Sentinel 共同确认；
- 选择一个 Replica 提升；
- 重新配置其他副本；
- 告诉客户端新 Master。

它不把数据分到多个 Master。

### ③ Cluster 是什么

Cluster 把 16384 个 Slot 分给多个 Master：

```text
Master A 负责一部分 Slot
Master B 负责一部分 Slot
Master C 负责一部分 Slot
每个 Master 再有 Replica
```

它同时解决：

- 水平分片；
- 每个分片的故障转移。

### ④ 为什么不能认为 Cluster 一定更好

Cluster 增加：

- 多 Key 跨 Slot 限制；
- Hash Tag；
- MOVED/ASK；
- 迁槽；
- Hot Slot；
- 客户端拓扑；
- 多节点容量治理。

若单节点容量与 QPS 足够，Sentinel 更简单。

### ⑤ 选择表

| 约束 | 更适合 |
|---|---|
| 数据能放单 Master | Sentinel |
| 只需要自动故障转移 | Sentinel |
| 单节点容量不够 | Cluster |
| 吞吐需要多 Master | Cluster |
| 大量多 Key 原子操作 | Sentinel 更简单 |
| 团队无法管理分片复杂性 | 先避免 Cluster |

### ⑥ 两者共同边界

Sentinel 和 Cluster 都基于复制与故障检测：

- 都可能有复制延迟；
- 都不能替代事实源；
- 都需要故障演练；
- 都需要客户端正确发现拓扑变化。

### ❌ 容易制造事故的写法

```java
// 因为“Cluster 更高级”，数据只有 2GB 也直接上 12 节点 Cluster，
// 业务却大量使用跨 Key Lua，最后被跨 Slot 限制反复改造。
```

### ✅ 企业级改进示例

```java
// 先做容量和吞吐评估：单主足够时使用 Sentinel。
// 达到单节点瓶颈且数据可分片后，再设计 Cluster Key 与 Hash Tag。
```

### 🎙️ 2～3 分钟优秀回答

Sentinel 与 Cluster 解决的问题不同。Sentinel 建立在一套主从之上，负责监控和自动故障转移，不负责分片；Cluster 把 Key 分布到 16384 个 Slot，由多个 Master 承担数据和流量，并为每个分片提供副本与故障转移。

如果单 Master 的容量和 QPS 足够，只需要高可用，我倾向 Sentinel，因为架构、多 Key 操作和运维更简单。只有单节点容量或吞吐不够时才引入 Cluster。

Cluster 会增加 Hash Tag、跨 Slot 限制、MOVED/ASK、迁槽和 Hot Slot 等复杂度。两者仍基于异步复制，不能承诺零丢失。因此选型依据是容量、吞吐、原子操作范围和团队能力，而不是简单认为 Cluster 更高级。

### 面试官可能继续追问

- Sentinel 如何从主观下线走到客观下线？
- Cluster 中某个 Master 故障后由谁接管 Slot？

> **记忆句**：Sentinel = 单分片自动切换；Cluster = 多分片 + 每分片高可用。

---
