# Redis 8.8 面试题 Q22

## Q22. maxmemory-policy 应该怎样选择

**类型：高频配置题｜难度：中等｜重要性：★★★★★**

### ① 先理解策略在回答什么

当 Redis 已达到内存上限，新写入来了：

```text
拒绝新写
还是删掉某些旧数据？
删哪些？
```

### ② noeviction

不主动淘汰，超过上限后多数新增写返回错误。

适合：

- 数据不能被静默移走；
- 锁、幂等、协调状态；
- 上层能够处理写失败；
- 容量与告警严格。

它不是“生产一定不能用”，关键是业务能否承受写失败，以及数据是否允许淘汰。

### ③ allkeys-lru / volatile-lru

LRU 倾向淘汰最近较少使用的 Key。

- `allkeys`：所有 Key 参与；
- `volatile`：只有设置 TTL 的 Key 参与。

Redis 使用近似 LRU，不维护全量精确顺序，以降低成本。

### ④ allkeys-lfu / volatile-lfu

LFU 根据访问频率衰减估计，倾向保留长期高频热点。

适合热点相对稳定的缓存。

### ⑤ random 与 ttl

- random：随机选择；
- volatile-ttl：倾向淘汰 TTL 更短的 Key。

通常需要明确业务理由，不是默认首选。

### ⑥ 怎样选择

```text
纯缓存、可重建、热点稳定
  → allkeys-lfu 可作为候选

纯缓存、最近访问更重要
  → allkeys-lru

只有带 TTL 的缓存允许淘汰
  → volatile 系列，但需防止无 TTL Key 占满空间

正确性状态不允许静默消失
  → 独立实例 + noeviction + 写失败处理
```

### ⑦ 为什么“所有 Key 都设置 TTL + volatile”仍可能危险

- 有 Key 漏设 TTL；
- 无 TTL Key 逐渐占满内存；
- 可选淘汰候选不足；
- 新写返回 OOM。

必须监控 TTL 覆盖率和无 TTL Key。

### ❌ 容易制造事故的写法

```java
# 因为“LFU 最智能”，所有实例无条件使用：
maxmemory-policy allkeys-lfu
# 结果导致锁和幂等记录也会被淘汰。
```

### ✅ 企业级改进示例

```java
# cache cluster
maxmemory-policy allkeys-lfu

# coordination cluster
maxmemory-policy noeviction
# 应用必须处理 OOM 写失败并触发告警/降级。
```

### 🎙️ 2～3 分钟优秀回答

maxmemory-policy 决定达到内存上限后是拒绝写，还是淘汰哪些 Key。noeviction 不会静默删除数据，但会让新增写失败，适合不允许被淘汰的协调状态，前提是容量、告警和写失败处理完整。

普通可重建缓存可以选择 allkeys-lru 或 allkeys-lfu。LRU 更强调最近访问，LFU 更适合长期热点；volatile 系列只淘汰带 TTL 的 Key，但如果存在大量无 TTL Key，可能没有足够候选并最终 OOM。

我不会为整个系统选一个统一策略，而是按状态分池：结果缓存用可淘汰策略，锁、幂等和关键 Checkpoint 使用独立实例与 noeviction，并由 MySQL 或审计记录兜底。策略选择必须通过真实访问分布和回源成本压测验证。

### 面试官可能继续追问

- 近似 LRU 为什么不维护精确链表？
- volatile-lru 在没有足够 TTL Key 时会发生什么？

> **记忆句**：策略不是按 Redis 选，而是按数据能否被静默移走来选。

---
