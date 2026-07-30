# Redis 8.8 面试题 Q05

## Q05. Pipeline、MULTI/EXEC、WATCH 与 Lua/Function 怎样选择

**类型：高频机制题｜难度：中等｜重要性：★★★★★**

### ① 四种机制解决的不是同一个问题

| 机制 | 主要目的 |
|---|---|
| Pipeline | 减少多条命令的网络往返 |
| MULTI/EXEC | 让一组命令连续执行 |
| WATCH | 在提交前检查数据是否被别人修改 |
| Lua/Function | 在 Redis 内原子完成读、判断、写 |

### ② Pipeline 不是事务

普通方式执行 100 次 GET：

```text
发 1 条 → 等回复
发 1 条 → 等回复
...
```

Pipeline：

```text
连续发 100 条
  ↓
Redis 依次执行
  ↓
连续返回结果
```

它减少 RTT，但中间仍可能穿插其他客户端命令，也不保证“全部成功或全部失败”。

### ③ MULTI/EXEC 是什么

```redis
MULTI
SET a 1
INCR b
EXEC
```

命令先进入队列，`EXEC` 时连续执行。

但 Redis 事务不像 MySQL：

- 没有传统隔离级别；
- 运行时错误不会自动把前面命令回滚；
- 不应把它理解成跨 MySQL 与 Redis 的分布式事务。

### ④ WATCH 是乐观检查

```text
WATCH balance
读取 balance
计算新值
MULTI
写入
EXEC
```

如果 `balance` 在提交前被其他客户端修改，`EXEC` 放弃，应用需要重试。

适合低冲突 CAS，不适合高冲突热点无限重试。

### ⑤ Lua/Function 是服务端原子逻辑

例如：

```text
读取锁的 owner
只有 owner 匹配才删除
```

读、判断、删必须不可被其他命令插入，可以用短 Lua。

代价是脚本执行期间占用核心命令路径，所以必须：

- Key 数有界；
- 循环有界；
- 集合大小有界；
- 返回字节有界。

### ⑥ 选择顺序

```text
一个原生命令能完成
  → 优先原生命令

只想减少 RTT
  → Pipeline

简单连续执行
  → MULTI/EXEC

低冲突条件更新
  → WATCH

需要原子读判断写
  → 短 Lua/Function
```

### ❌ 容易制造事故的写法

```java
redis.setAutoFlushCommands(false);
for (String key : oneMillionKeys) {
    redis.get(key); // 无上限地积累请求和回复
}
redis.flushCommands();
```

### ✅ 企业级改进示例

```java
for (List<String> batch : partition(keys, 200)) {
    try (StatefulRedisConnection<String, String> c = dedicatedConnection()) {
        c.setAutoFlushCommands(false);
        List<RedisFuture<String>> futures = new ArrayList<>();
        for (String key : batch) {
            futures.add(c.async().get(key));
        }
        c.flushCommands();
        awaitWithDeadline(futures);
    }
}
```

### 🎙️ 2～3 分钟优秀回答

Pipeline、事务和 Lua 的目标不同。Pipeline 只是把多条命令批量发送，减少网络 RTT，不提供多命令原子性。MULTI/EXEC 会先排队，在 EXEC 时连续执行，但没有关系数据库式的运行时自动回滚。WATCH 是乐观锁，监视的 Key 在提交前变化时事务放弃。Lua 或 Function 可以在服务端原子完成读、判断和写。

选择时我优先看能否用单个原生命令，例如 SET NX PX、INCR、ZPOPMIN。批量独立命令用有限批次 Pipeline；简单事务用 MULTI/EXEC；低冲突 CAS 用 WATCH；复杂条件更新用短小 Lua。

风险上，Pipeline 批次过大会占用客户端和服务端缓冲；WATCH 高冲突会反复失败；长 Lua 会阻塞其他命令；Cluster 多 Key 还要求同 Slot。核心是先明确目标是减少 RTT 还是保证原子性。

### 面试官可能继续追问

- 为什么 Redis 事务出现运行时错误后不会全部回滚？
- Lua 脚本在 Cluster 中为什么要求多 Key 同 Slot？

> **记忆句**：少 RTT 用 Pipeline；连续执行用事务；条件原子读写用短脚本。

---
