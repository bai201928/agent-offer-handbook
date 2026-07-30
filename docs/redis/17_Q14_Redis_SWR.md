# Redis 8.8 面试题 Q14

## Q14. 本地缓存、Redis 与 SWR 怎样组成两级缓存

**类型：企业场景题｜难度：中等｜重要性：★★★★☆**

### ① 两级缓存是什么

```text
请求
  ↓
JVM 本地 Caffeine
  ↓ Miss
Redis
  ↓ Miss
MySQL / Milvus
```

L1 本地缓存速度最快，不经过网络；L2 Redis 可在多个应用实例间共享。

### ② 为什么需要本地缓存

一个模型配置 Key 每秒被所有实例读取几万次，即使 Redis 很快，也会形成热 Key。

把极热、体积小、允许短暂陈旧的数据放到本地缓存，可减少 Redis QPS 和网络延迟。

### ③ 两级缓存为什么更难一致

实例 A、B、C 各自有一份 L1。

数据库更新后：

```text
A 收到失效通知
B 收到失效通知
C 正好断线，没收到
```

C 仍可能返回旧值。

因此正确性不能只依赖“通知一定送达”。

### ④ 企业级组合

- L1 TTL 比 Redis 更短；
- Value 携带版本；
- 更新后发送失效事件；
- 事件丢失时由 TTL 和版本兜底；
- 强一致链路绕过 L1；
- 超级热 Key 可预热；
- 本地缓存设置最大容量。

### ⑤ Pub/Sub、Stream 和 MQ 的差异

Redis Pub/Sub 实时但不保留离线历史，断线时消息可能丢。

可靠失效事件可以使用：

- Redis Stream；
- Kafka；
- RocketMQ；
- CDC。

即使使用可靠消息，读路径仍应检查版本，因为事件存在传播延迟。

### ⑥ 什么是 SWR

SWR 是 Stale-While-Revalidate。

缓存逻辑过期后：

```text
先返回一份仍可接受的旧值
  ↓
只让少量后台任务刷新
```

适合：

- 商品详情；
- RAG 摘要；
- 模型展示配置；
- 非关键统计。

不适合：

- 权限；
- 库存；
- 支付；
- 风控结果；
- 刚更新后必须立即可见的数据。

### ❌ 容易制造事故的写法

```java
@Cacheable("permissions")
public Permission loadPermission(String userId) {
    // 本地缓存 24 小时，无版本、无失效可靠性。
    return repository.loadPermission(userId);
}
```

### ✅ 企业级改进示例

```java
public Permission loadPermission(String userId, long requiredVersion) {
    Versioned<Permission> local = l1.getIfPresent(userId);
    if (local != null && local.version() >= requiredVersion) {
        return local.value();
    }
    return loadFromRedisOrOrigin(userId, requiredVersion);
}
```

### 🎙️ 2～3 分钟优秀回答

两级缓存通常是 JVM Caffeine 作为 L1、Redis 作为 L2，最后回源 MySQL 或 Milvus。本地缓存可以降低 Redis 热 Key 压力和网络延迟，但每个实例都有独立副本，因此一致性更复杂。

我会让 L1 TTL 短于 Redis，Value 携带业务版本，并通过 Stream、Kafka、RocketMQ 或 CDC 发送失效事件。通知延迟或丢失时仍由版本检查和 TTL 兜底，强一致链路绕过旧本地值。

SWR 适合允许短时间陈旧的数据：逻辑过期后先返回旧值，只让少量后台任务刷新，避免热点同步回源。但权限、库存、支付和风控不能返回旧值。两级缓存的目标不是绝对同步，而是明确最大陈旧窗口和失败后的恢复路径。

### 面试官可能继续追问

- 为什么 Pub/Sub 不适合作为唯一失效通道？
- 逻辑过期与物理 TTL 应怎样配合？

> **记忆句**：L1 降热度，L2 做共享；通知加速失效，版本和 TTL 保证兜底。

---
