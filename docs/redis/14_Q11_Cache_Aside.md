# Redis 8.8 面试题 Q11

## Q11. Cache Aside 的正确读写流程是什么

**类型：缓存必考题｜难度：简单｜重要性：★★★★★**

### ① 什么是 Cache Aside

Cache Aside 可以理解为：

```text
应用自己决定什么时候查缓存、什么时候查数据库、什么时候让缓存失效
```

Redis 不会自动替 MySQL 维护缓存。

### ② 读流程一步一步走

用户查询商品或知识库配置：

```text
1. Java 先 GET Redis
2. 命中 → 直接返回
3. 未命中 → 查询 MySQL
4. MySQL 有数据 → 写回 Redis，并设置 TTL
5. 返回结果
```

### ③ 写流程为什么通常是“更新数据库，再删除缓存”

```text
1. 更新 MySQL
2. MySQL 事务成功提交
3. 删除 Redis 缓存
4. 下一次读请求从 MySQL 重建缓存
```

为什么不是先删缓存再更新数据库？

```text
先删缓存
  ↓
另一个读请求发现 Miss
  ↓
读到数据库旧值
  ↓
回填旧缓存
  ↓
数据库才更新成功
```

为什么通常不直接同时更新缓存？

- 缓存可能由多表聚合；
- 可能漏掉某些缓存 Key；
- 并发写可能互相覆盖；
- 删除后按事实源重建更简单；
- Redis 本来就不是最终事实。

### ④ 完整读流程还要补什么

- 空值短缓存；
- TTL 随机抖动；
- 单次回填大小限制；
- 热点请求合并；
- 回源并发上限；
- 版本；
- Redis 故障时降级。

### ⑤ 删除缓存失败怎么办

MySQL 已成功，但 Redis `DEL` 超时：

```text
不能回滚已经提交的数据库事务
```

可通过：

- Outbox；
- CDC；
- 可靠消息；
- 重试表；
- 定时对账；
- TTL 兜底。

### ⑥ Redis 原子性边界

每个 `GET`、`SET`、`DEL` 可以独立原子执行。

但下面整条链不是 Redis 原子事务：

```text
GET Redis
  → 查 MySQL
  → SET Redis
```

这就是为什么后面还会出现并发旧值回填。

### ❌ 容易制造事故的写法

```java
@Transactional
public void updateProduct(Product p) {
    redis.del("product:" + p.id()); // 先删除
    repository.update(p);           // 事务尚未提交
}
```

### ✅ 企业级改进示例

```java
@Transactional
public void updateProduct(Product p) {
    repository.update(p);
    outboxRepository.save(CacheInvalidation.of("product:v3:" + p.id()));
}
// Outbox 消费者在事务提交后重试删除，TTL 仍作为兜底。
```

### 🎙️ 2～3 分钟优秀回答

Cache Aside 的读流程是先查 Redis，命中直接返回，未命中再查 MySQL 或 Milvus，校验结果大小和版本后回填并设置 TTL。写流程通常是先更新数据库，事务提交后删除缓存，而不是先删缓存，也不是盲目同时更新所有缓存。

先删缓存可能让并发读请求在数据库更新前读到旧值并回填。直接更新缓存则容易遗漏聚合字段、缓存版本和其他派生 Key。删除后由下一次读取根据事实源重建，逻辑更简单。

数据库成功但删除失败时，我会使用 Outbox、CDC 或可靠消息异步重试失效，并以 TTL 作为最终兜底。完整方案还包括空值缓存、TTL 抖动、回填限流、版本和对象大小限制。Cache Aside 提供的是可控最终一致，不是 MySQL 与 Redis 的分布式事务。

### 面试官可能继续追问

- 为什么延迟双删不能从根本上保证一致性？
- 数据库事务提交后怎样可靠触发缓存删除？

> **记忆句**：读：缓存→事实源→回填；写：事实源提交→失效缓存→可靠重试。

---
