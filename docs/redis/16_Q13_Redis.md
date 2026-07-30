# Redis 8.8 面试题 Q13

## Q13. 缓存穿透、击穿和雪崩有什么区别

**类型：缓存高频题｜难度：简单偏中等｜重要性：★★★★★**

### ① 穿透：查的东西本来就不存在

攻击者不断查询不存在的用户：

```text
Redis 没有
  ↓
MySQL 也没有
  ↓
下一次仍然全部回源
```

治理：

- 参数与权限校验；
- 空值短缓存；
- Bloom Filter；
- 限流；
- 黑名单和风控。

Bloom Filter 有假阳性，不能作为数据库事实。

### ② 击穿：一个热点 Key 突然过期

```text
热门模型配置过期
  ↓
1 万个请求同时 Miss
  ↓
1 万个请求同时查 MySQL
```

治理：

- JVM Singleflight；
- 短分布式互斥；
- 逻辑过期；
- Stale-While-Revalidate；
- 提前刷新；
- 热点预热。

锁只能选出少量重建者，仍要限制等待队列。

### ③ 雪崩：大量 Key 或整个 Redis 同时失效

原因可能是：

- 大量 Key 同时到期；
- Redis 集群故障；
- 网络隔离；
- 客户端超时配置错误；
- 发布导致缓存版本整体切换；
- 重试风暴。

治理：

- TTL 打散；
- 分批预热；
- 本地缓存；
- 回源预算；
- 舱壁；
- 熔断；
- 降级；
- 保护 MySQL/Milvus；
- Redis 故障演练。

### ④ 三者最重要的区别

```text
穿透 → 无效数据
击穿 → 单个热点
雪崩 → 大面积失效或整体故障
```

### ⑤ 为什么不能只背“布隆、锁、随机 TTL”

这些只是局部工具。

真正的企业级保护链：

```text
入口参数和权限
  ↓
租户限流
  ↓
JVM 请求合并
  ↓
Redis
  ↓
有界回源
  ↓
超时、熔断和降级
```

Redis 无法替 MySQL 决定它能承受多少并发，应用必须控制回源。

### ❌ 容易制造事故的写法

```java
if (redis.get(key) == null) {
    // 所有 Miss 都无限并发回源
    return repository.load(id);
}
```

### ✅ 企业级改进示例

```java
return singleflight.execute(key, () ->
    originSemaphore.withPermit(() -> {
        Value cached = redis.get(key);
        if (cached != null) return cached;
        return loadAndCacheWithSizeLimit(id);
    }));
```

### 🎙️ 2～3 分钟优秀回答

穿透是查询不存在的数据，Redis 和事实源都没有，导致每次都回源；击穿是单个高热 Key 到期，大量并发同时回源；雪崩是大量 Key 同时失效或 Redis 整体不可用，造成大面积回源洪峰。

穿透使用参数校验、短 TTL 空值、Bloom 和限流；击穿优先用 JVM Singleflight 合并同 Key 请求，跨实例再考虑短互斥、逻辑过期或 SWR；雪崩需要 TTL 打散、分批预热、回源预算、舱壁、熔断和降级。

我不会只背“布隆、锁、随机 TTL”。Bloom 有假阳性，锁不能增加数据库容量，随机 TTL 也解决不了 Redis 整体故障。真正的目标是让所有缓存 Miss 进入有上限的回源通道，保护 MySQL 和 Milvus。

### 面试官可能继续追问

- 空值缓存的 TTL 为什么通常较短？
- SWR 为什么不适合权限和库存？

> **记忆句**：穿透打无效请求，击穿打单热点，雪崩打整体回源容量。

---
