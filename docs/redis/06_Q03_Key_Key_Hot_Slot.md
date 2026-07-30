# Redis 8.8 面试题 Q03

## Q03. 大 Key、热 Key 与 Hot Slot 有什么区别，怎样治理

**类型：高频场景题｜难度：中等｜重要性：★★★★★**

### ① 三个词分别描述什么

#### 大 Key

一个 Key 对应的数据量很大：

```text
String 字节特别大
Hash/Set/ZSet/List 元素特别多
```

#### 热 Key

一个 Key 在短时间被访问特别频繁。

#### Hot Slot

Redis Cluster 中，很多数据或流量集中到少数 Slot 所在节点。

三者可能同时出现，也可能完全独立。

### ② 用 Agent 项目理解

```text
nexus:run:R1:all
```

保存了 20MB JSON，这是大 Key。

```text
nexus:model:config:gpt-main
```

每秒被读取 10 万次，这是热 Key。

所有大租户 Key 都使用：

```text
{tenant:T1}
```

它们全部落到同一个 Slot，这是 Hot Slot。

### ③ 大 Key 为什么危险

执行：

```redis
HGETALL huge-hash
```

Redis 需要：

```text
遍历大量字段
  ↓
构造大量回复
  ↓
放入客户端输出缓冲
  ↓
经过网络
  ↓
Java 创建大量对象并解码
```

危害包括：

- 核心执行路径占用时间长；
- 大回复导致队头阻塞；
- 复制、持久化、迁槽变重；
- 删除释放成本高；
- Java 堆和 GC 压力大。

### ④ 热 Key 为什么危险

即使 Value 很小，所有请求都打到一个 Key 对应的节点，也可能耗尽：

- 单节点 CPU；
- 网络带宽；
- 连接队列；
- 客户端 EventLoop。

治理方式：

- JVM 本地缓存；
- 请求合并；
- 热点副本或业务拆分；
- 限流；
- 预热；
- 允许陈旧时读副本。

### ⑤ Hot Slot 怎样产生

Cluster 根据 Key 计算 Slot。若 Hash Tag 设计过粗，很多 Key 被强制放在一起。

```text
每个节点拥有的 Slot 数量看起来差不多
≠ 每个节点承受的 QPS 和字节一样
```

因此需要监控每节点：

- QPS；
- 网络字节；
- 内存；
- CPU；
- Top Key；
- Slot 数据量。

### ⑥ 如何发现

不要在线执行 `KEYS *`。

可组合使用：

- `SCAN`/`HSCAN` 限速抽样；
- `MEMORY USAGE`；
- `redis-cli --bigkeys` 或 `--memkeys`；
- 命令统计；
- 网络和延迟指标；
- 客户端埋点；
- Cluster 节点对比。

没有一个固定阈值适合所有业务。10KB 对某些业务是大，1MB 对离线缓存也可能可接受，必须结合 P99 和吞吐压测。

### ⑦ 治理顺序

```text
先止血：禁止全量读、限流、隔离热点
  ↓
拆结构：元数据和大结果分离
  ↓
限制上限：字节、元素、回复
  ↓
迁移冷数据：对象存储/MySQL
  ↓
建立告警和容量测试
```

### ❌ 容易制造事故的写法

```java
Map<String, String> all = redis.hgetall("nexus:run:" + runId);
// 在线接口每次读取一个包含几万字段的大 Hash。
return objectMapper.writeValueAsString(all);
```

### ✅ 企业级改进示例

```java
List<String> fields = List.of("status", "owner", "version");
List<String> values = redis.hmget(runMetaKey, fields.toArray(String[]::new));
// 大工具结果按 toolName 独立 Key，接口按需读取。
```

### 🎙️ 2～3 分钟优秀回答

大 Key、热 Key 和 Hot Slot 是三个维度。大 Key 指单个 Value 字节或集合元素很多；热 Key 指访问频率集中；Hot Slot 指 Cluster 的数据或流量集中到少数 Slot 所在节点。

大 Key 会拖长遍历、回复构造、网络、复制、持久化和 Java 解码；热 Key 会形成单节点 CPU 与带宽热点；Hot Slot 常由过粗 Hash Tag 或超级租户引起。

我会用限速 SCAN、MEMORY USAGE、bigkeys/memkeys、客户端调用采样、回复字节和每节点 QPS 共同发现，而不是执行 KEYS。治理上，大 Key 要拆 Value、分页、字段白名单、限制对象大小并用 UNLINK；热 Key 用本地缓存、请求合并、限流和预热；Hot Slot 要调整 Hash Tag、拆超级租户和迁槽。

项目中我们把 Run 元数据保留为小 Hash，大工具结果独立存储并设置 TTL，避免一个 HGETALL 让续租命令排队。

### 面试官可能继续追问

- 为什么大 Key 会影响完全无关的短命令？
- 读副本是否可以彻底解决热 Key？

> **记忆句**：大 Key 看大小，热 Key 看频率，Hot Slot 看 Cluster 节点倾斜。

---
