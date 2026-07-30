# Redis 8.8 面试题 Q10

## Q10. Slot、Hash Tag、MOVED 与 ASK 分别是什么

**类型：Cluster 高频题｜难度：中等偏上｜重要性：★★★★☆**

### ① Slot 为什么存在

Cluster 需要决定每个 Key 由哪个 Master 保存。

它不是直接把每一个 Key 写进中央路由表，而是先把 Key 映射到 16384 个 Slot，再把 Slot 分给节点。

```text
Key
  ↓ 计算
Slot
  ↓ 路由
Master
```

### ② Hash Tag 是什么

默认使用整个 Key 计算 Slot。

如果 Key 中有有效花括号：

```text
order:{O1001}:meta
order:{O1001}:items
```

只使用 `{O1001}` 中的内容计算，因此两个 Key 进入同一个 Slot。

为什么需要同 Slot：

- 多 Key 命令；
- 事务；
- Lua/Function；
- 一些需要原子更新的业务。

### ③ MOVED 是永久路由提示

客户端把请求发错节点，节点返回：

```text
这个 Slot 的稳定负责人是另一个节点
```

客户端应更新 Slot→节点缓存，以后直接去正确节点。

### ④ ASK 是迁槽期间的临时提示

Slot 正在从源节点迁往目标节点。

节点告诉客户端：

```text
这一次先去目标节点，并发送 ASKING
但不要把长期路由立刻永久改掉
```

### ⑤ 为什么 Hash Tag 不能太大

错误：

```text
nexus:{tenant:T1}:run:R1
nexus:{tenant:T1}:run:R2
nexus:{tenant:T1}:所有其他 Key
```

整个租户全部进入一个 Slot。

正确方向：

```text
nexus:run:{T1:R1}:meta
nexus:run:{T1:R1}:checkpoint
```

只有同一个 Run 需要原子操作的 Key 同槽。

### ⑥ 客户端应怎样处理

使用 Cluster-aware Lettuce：

- 维护拓扑；
- 处理重定向；
- 定期刷新；
- 对连接异常有边界重试；
- 不在业务代码中硬编码节点地址。

### ⑦ Slot 均衡不等于负载均衡

两个节点拥有相同 Slot 数，但某个 Slot 中有超级热 Key，负载仍然不均。

必须结合 QPS、字节、内存和热 Key 观察。

### ❌ 容易制造事故的写法

```java
String key = "nexus:{tenant:" + tenantId + "}:run:" + runId;
// 一个超级租户的全部 Key 都固定到同一 Slot。
```

### ✅ 企业级改进示例

```java
String tag = tenantId + ":" + runId;
String meta = "nexus:run:{" + tag + "}:meta";
String checkpoint = "nexus:run:{" + tag + "}:checkpoint";
// 只把同一 Run 中确实需要原子操作的 Key 放在一起。
```

### 🎙️ 2～3 分钟优秀回答

Redis Cluster 先通过 Key 计算 16384 个 Slot，再把 Slot 分配给 Master。Hash Tag 允许只使用花括号中的内容计算 Slot，让相关 Key 同槽，以支持多 Key 命令、事务或 Lua。

MOVED 表示这个 Slot 的稳定归属在另一个节点，客户端应更新长期拓扑；ASK 常出现在迁槽期间，只要求本次请求临时去目标节点并发送 ASKING，不能直接当成永久路由。

项目中我只让一个订单、Run 等最小原子单元共享 Hash Tag，不会把整个大租户放进同一 Tag，否则会形成 Hot Slot。客户端使用 Cluster-aware Driver 自动处理拓扑和重定向，同时监控每节点 QPS、字节和内存，因为 Slot 数量均衡不代表实际负载均衡。

### 面试官可能继续追问

- ASKING 命令解决什么？
- 迁槽期间源节点和目标节点怎样判断 Key 在哪里？

> **记忆句**：Key 先到 Slot，再到节点；MOVED 是长期地址，ASK 是迁移中的临时地址。

---

# Redis 面试题 Q11～Q20

## 学习顺序

这一部分从“Redis 自己是否可用”进入“Redis 与 MySQL、Milvus、Java 并发怎样共同保证业务可控”。

```text
先学正确读写缓存
  ↓
再看并发怎样产生旧值
  ↓
再保护回源和热点
  ↓
最后学习原子计数与分布式锁
```

核心原则：

> Redis 的单条命令可以原子执行，但 Java、Redis、MySQL、Milvus 和外部 Tool 之间不会自动形成一个全局事务。
