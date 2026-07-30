# Redis 8.8 面试题 Q25

## Q25. Redis Cluster 节点内存倾斜怎样发现和治理

**类型：项目深挖题｜难度：中等偏上｜重要性：★★★★☆**

### ① 什么是内存倾斜

Cluster 有三个 Master：

```text
A：20GB
B：8GB
C：7GB
```

即使 Slot 数量接近，A 仍明显更高。

### ② 常见原因

- 某些 Slot 中存在大 Key；
- 超级租户集中；
- Hash Tag 过粗；
- Key 数均衡但 Value 字节不均衡；
- TTL 差异；
- 热 Key 的本地结构增长；
- 模块或客户端缓冲差异；
- 迁槽未完成；
- 副本/复制缓冲状态不同。

### ③ 发现方法

不要只看 Slot 数。

按节点对比：

- used_memory；
- RSS；
- Key 数；
- 过期 Key；
- 客户端缓冲；
- QPS 与网络字节；
- Slot 数据字节；
- 大 Key 样本；
- 租户分布；
- 热 Key。

建立映射：

```text
tenant → key family → slot → node → bytes/QPS
```

### ④ 治理方法

#### 大 Key 拆分

一个 10GB Hash 不能靠把更多空 Slot 分给其他节点解决。

#### 调整 Hash Tag

从整个租户 Tag 改为 Run、订单或分桶 Tag。

#### 超级租户分桶

```text
tenant:T1:bucket:00
tenant:T1:bucket:01
...
```

但需要考虑多 Key 原子性边界。

#### 迁槽

把高字节 Slot 迁往空闲节点，不只是平均 Slot 数量。

#### 独立集群

极端超级租户或特殊状态可独立部署，避免影响共享集群。

### ⑤ 为什么扩容不一定自动解决

新增节点后，如果没有重新分配 Slot 和迁移数据，新节点仍然空闲。

即使迁槽完成，热 Key 仍然只能由一个节点处理，除非业务拆分或引入本地缓存等方案。

### ❌ 容易制造事故的写法

```java
// 看到 A 节点内存高，只新增一个节点，但不迁槽也不改 Key。
scaleOutClusterByOneNode();
```

### ✅ 企业级改进示例

```java
SkewReport report = analyzeByTenantKeyFamilySlot();
for (HotSlot slot : report.hotSlots()) {
    if (slot.causedByBigKey()) splitBigKey(slot);
    else migrateByBytes(slot, targetNode(slot));
}
```

### 🎙️ 2～3 分钟优秀回答

Cluster 内存倾斜是不同节点的数据字节、RSS 或负载明显不均。原因不只是 Slot 数不均，还可能是大 Key、超级租户、过粗 Hash Tag、Value 大小差异、TTL、客户端缓冲和热 Key。

排查时我会建立 tenant、Key family、Slot、Node、Bytes 和 QPS 的映射，按节点比较 used_memory、RSS、Key 数、缓冲、网络和大 Key 样本。治理包括拆大 Key、缩小 Hash Tag、给超级租户分桶、按字节而不是只按 Slot 数迁槽，必要时独立集群。

新增节点并不会自动让数据均衡，需要重新分配 Slot；而单个热 Key 即使扩容也仍在一个节点上，必须从业务模型、本地缓存或请求合并解决。

### 面试官可能继续追问

- 为什么 Slot 数量均衡但内存仍会不均？
- 给超级租户分桶会破坏哪些原子操作？

> **记忆句**：Cluster 均衡要看字节和 QPS，不只看 Slot 数。

---
