# Redis 8.8 面试题 Q08

## Q08. 有主从复制，为什么主节点故障后仍可能丢数据

**类型：高频高可用题｜难度：中等｜重要性：★★★★★**

### ① 先建立时间线

```text
客户端向 Master 写入 X
  ↓
Master 在本地执行成功
  ↓
Master 回复客户端“成功”
  ↓
X 还没来得及复制到 Replica
  ↓
Master 突然故障
  ↓
旧 Replica 被提升
```

新 Master 中没有 X，于是客户端曾经收到成功的写丢失。

### ② 默认复制为什么有这个窗口

Redis 复制通常是异步的。主节点不会为每一次普通写都等待所有副本完全持久化后才回复。

这样性能更高，但存在确认边界。

### ③ replid、offset、backlog 和 PSYNC 是什么

先用录像理解：

```text
replid  = 这段录像属于哪一场
offset  = 已经播放到第几个字节
backlog = Master 暂存的一小段历史录像
PSYNC   = Replica 告诉 Master：我缺哪一段
```

短暂断线后，如果缺失部分仍在 backlog，可以只补这段，叫部分同步。

如果历史已经不在，或身份不匹配，就需要全量同步：

```text
先发快照
再补快照后的增量命令
```

### ④ 怎样缩小风险

- 配置最小健康副本数量和最大延迟；
- 使用 `WAIT` 等待一定副本确认；
- 合理配置 backlog；
- 跨机架/可用区部署；
- 关键事实保存在 MySQL；
- 切换后按版本对账；
- 对读己之写请求走 Master 或版本校验。

这些只能缩小风险，不能把 Redis 变成共识式强一致数据库。

### ⑤ 从副本读取也会有旧值

主从复制存在延迟。

```text
刚写 Master
立即读 Replica
```

可能读不到最新值。

对权限、支付、刚更新配置等要求“读己之写”的场景，应避免盲目读副本。

### ❌ 容易制造事故的写法

```java
// 刚写 Master 后立刻强制从 Replica 读取权限版本。
redisMaster.set(permissionKey, "v8");
return redisReplica.get(permissionKey); // 可能仍是 v7
```

### ✅ 企业级改进示例

```java
redisMaster.set(permissionKey, "v8");
// 在读己之写窗口内读 Master；普通可陈旧查询才允许读 Replica。
return redisMaster.get(permissionKey);
```

### 🎙️ 2～3 分钟优秀回答

Redis 主从默认通常是异步复制。Master 执行写并回复客户端时，这条写可能还在网络或 Replica 的处理队列中。如果 Master 在同步完成前故障，提升旧 Replica 后，就可能缺少已经确认给客户端的数据。

复制恢复依靠 replid、offset、replication backlog 和 PSYNC。短暂断线且缺失区间仍在 backlog 中可部分同步，否则需要快照加增量的全量同步。

生产中可以通过最小副本配置、最大复制延迟、WAIT、合理 backlog 和跨故障域部署降低风险，但这些不等于强一致。关键业务事实仍应由 MySQL 保存，故障切换后按业务版本对账。从 Replica 读取也要接受复制延迟，对读己之写场景应走 Master 或使用版本判断。

### 面试官可能继续追问

- WAIT 能否提供绝对强一致？
- 部分同步为什么需要 backlog？

> **记忆句**：副本提高可用性，但异步复制存在“已回复、未复制”的窗口。

---
