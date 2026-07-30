# Redis 8.8 面试题 Q17

## Q17. 一个正确的 Redis 分布式锁至少需要哪些机制

**类型：分布式锁必考题｜难度：中等｜重要性：★★★★★**

### ① 锁要解决什么

多个 Worker 同时尝试执行同一个 Agent Run：

```text
Worker A
Worker B
```

希望同一时刻只有一个进入关键区。

### ② 加锁

常见单实例形式：

```redis
SET lock:run:R1 random-owner-token NX PX 30000
```

含义：

- `NX`：Key 不存在时才成功；
- `PX 30000`：30 秒自动过期；
- Value：当前持有者的随机唯一 Token。

### ③ 为什么必须设置过期

Worker 拿锁后宕机，如果没有 TTL，锁会永久存在。

### ④ 为什么 Value 不能只写固定字符串

错误：

```text
Value = locked
```

锁过期后 B 获得新锁，A 恢复并执行 `DEL`，可能把 B 的锁删除。

必须让每次加锁都有唯一 owner token。

### ⑤ 解锁为什么必须比较 owner

正确逻辑：

```text
读取锁 Value
如果等于自己的 Token
  删除
否则
  不做任何事
```

读、判断、删必须原子完成，使用 Lua/Function。

### ⑥ 锁过期但任务还没完成怎么办

两种基本策略：

- 任务时间有严格上限，TTL 留足；
- 看门狗续租。

续租也必须比较 owner，且应用长时间 Stop-The-World、网络分区或 EventLoop 阻塞时仍可能失去锁。

### ⑦ 锁能否保证外部副作用绝不重复

不能。

```text
A 获得锁
A 调用外部支付/Tool，已经成功
A 在写结果前锁过期
B 获得锁并再次调用
```

因此还需要：

- 业务幂等键；
- 状态机；
- Fencing Token；
- 外部系统去重；
- 对账和补偿。

### ⑧ Cluster 注意

锁相关 Key 与需要原子更新的状态 Key 若进入同一个 Lua，必须同 Slot。

### ❌ 容易制造事故的写法

```java
if (redis.setnx(lockKey, "locked")) {
    try {
        doExternalToolCall();
    } finally {
        redis.del(lockKey); // 可能删除别人的新锁
    }
}
```

### ✅ 企业级改进示例

```java
String token = UUID.randomUUID().toString();
boolean locked = redis.set(lockKey, token, SetArgs.Builder.nx().px(30_000));
try {
    if (!locked) throw new BusyException();
    doIdempotentWork(requestId);
} finally {
    compareAndDelete(lockKey, token); // Lua 原子比较后删除
}
```

### 🎙️ 2～3 分钟优秀回答

正确的 Redis 分布式锁至少包含五点：第一，使用 `SET key uniqueToken NX PX ttl` 原子加锁；第二，必须设置 TTL，避免持有者宕机后永久死锁；第三，每次锁 Value 使用唯一 owner token；第四，解锁通过 Lua 原子比较 token 后删除，不能直接 DEL；第五，长任务需要有边界续租或任务时长上限。

但 Redis 锁只提供一段互斥租约，不等于业务副作用绝对一次。应用暂停、网络分区或锁过期后，旧持有者仍可能继续运行。因此关键写入还需要 Fencing Token，外部调用需要幂等键和状态机。

项目中我把锁、幂等记录和业务事实分开设计，不会用一个锁替代所有并发与一致性问题。

### 面试官可能继续追问

- 锁自动续期仍有哪些失败窗口？
- 为什么 Redis 锁不能替代数据库唯一约束？

> **记忆句**：加锁要 NX + TTL + 唯一主人；解锁要比较主人；业务还要幂等与栅栏。

---
