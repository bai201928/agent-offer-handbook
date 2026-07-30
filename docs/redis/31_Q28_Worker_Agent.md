# Redis 8.8 面试题 Q28

## Q28. 多个 Worker 并发更新 Agent 状态，怎样避免互相覆盖

**类型：高薪并发项目题｜难度：中等偏上｜重要性：★★★★★**

### ① 丢失更新是怎样发生的

状态当前版本 v7：

```text
Worker A 读到 v7，增加 tool-result
Worker B 读到 v7，更新 status
A 写回整个 JSON
B 写回整个 JSON
```

后写者覆盖前写者，某一部分变化消失。

### ② 为什么“Redis 命令原子”仍会丢

每个 GET 和 SET 独立原子，但整条：

```text
GET
Java 修改
SET
```

不是一个原子操作。

### ③ 方案一：拆字段并用原子命令

若两个 Worker 更新不同字段：

```text
HSET status
HSET tool-result-ref
```

可以减少整体覆盖，但仍需考虑状态机条件。

### ④ 方案二：WATCH/CAS

```text
WATCH key
读取 version=7
准备 version=8
MULTI
写入
EXEC
```

被别人修改后事务失败，应用重读并有界重试。

### ⑤ 方案三：Lua/Function 条件更新

脚本：

```text
读取当前 version
只有 expectedVersion 匹配
  执行更新
  version + 1
否则
  返回冲突
```

适合短小、有界更新。

### ⑥ 方案四：MySQL 乐观锁作为权威

```sql
UPDATE agent_run
SET status=?, version=version+1
WHERE run_id=? AND version=?
```

影响行数为 0 表示版本冲突。

Redis 作为热缓存跟随权威版本。

### ⑦ 方案五：单写者/事件串行化

同一个 Run 的事件按 Key 路由到同一 Kafka Partition 或 RocketMQ Message Group，由单个逻辑消费者顺序推进状态。

这减少并发冲突，但仍要处理重试和幂等。

### ⑧ 推荐组合

```text
消息按 runId 有序
+ 数据库状态机条件更新
+ Redis 热缓存
+ 租约/Fencing
+ 事件幂等
```

### ❌ 容易制造事故的写法

```java
RunState state = deserialize(redis.get(key));
state.setStatus("DONE");
redis.set(key, serialize(state)); // 覆盖其他 Worker 刚写入的字段
```

### ✅ 企业级改进示例

```java
String script = """
local current = tonumber(redis.call('HGET', KEYS[1], 'version'))
if current ~= tonumber(ARGV[1]) then return 0 end
redis.call('HSET', KEYS[1],
  'status', ARGV[2],
  'version', current + 1)
return 1
""";
long updated = redis.eval(script, keys, args);
if (updated == 0) throw new VersionConflictException();
```

### 🎙️ 2～3 分钟优秀回答

多个 Worker 并发更新时，即使单条 GET 和 SET 原子，GET、Java 修改、SET 的组合仍可能发生丢失更新。最简单的改进是按字段拆分并使用 HSET、INCR 等原子命令，但涉及状态迁移时仍需条件。

可以使用 WATCH 做乐观 CAS，或用短 Lua 根据 expectedVersion 原子比较并更新。关键状态我更倾向 MySQL 乐观锁作为权威：`WHERE version=?` 成功后版本加一，Redis 只缓存最新状态。另一种架构是让同一 runId 的事件进入同一消息分区，由单写者顺序推进。

企业级组合通常是消息顺序、数据库状态机、Redis 热缓存、Fencing 和事件幂等共同工作，而不是用一个分布式锁包住整个长流程。

### 面试官可能继续追问

- WATCH 高冲突时为什么可能导致活锁式重试？
- 消息有序是否意味着状态更新无需幂等？

> **记忆句**：单命令原子不等于读改写原子；状态更新必须带 expectedVersion。

---
