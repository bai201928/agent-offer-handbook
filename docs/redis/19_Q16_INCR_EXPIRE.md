# Redis 8.8 面试题 Q16

## Q16. 为什么 INCR 后再 EXPIRE 有竞态，怎样原子实现计数窗口

**类型：高频原子性题｜难度：中等｜重要性：★★★★★**

### ① 典型错误

限流代码：

```text
INCR key
如果第一次计数
  EXPIRE key 60
```

问题发生在两条命令之间。

### ② 故障时间线

```text
INCR 成功，计数变成 1
  ↓
应用进程崩溃
  ↓
EXPIRE 没执行
```

这个 Key 没有 TTL，可能永久存在。

另一种竞态：

```text
多个请求同时认为自己应该设置 TTL
```

虽然结果不一定总错，但逻辑不再是一个不可分割的窗口初始化。

### ③ 为什么 Pipeline 不能解决

Pipeline 只是一起发送：

```text
INCR
EXPIRE
```

Redis 仍把它们当两条独立命令。客户端在回复前崩溃不影响服务端执行，但如果第二条未送达或连接中断，仍可能只完成一半。

### ④ 正确方案

#### 方案 A：短 Lua/Function

在 Redis 内：

```text
执行 INCR
如果结果为 1
  设置 TTL
返回计数
```

整个脚本原子执行。

#### 方案 B：Redis 8.8 INCREX

Redis 8.8 提供 `INCREX`，把计数、边界和过期窗口组合到一个命令中，适合窗口计数限流。

使用前必须基于实际客户端与命令文档验证参数，不应只复制旧博客示例。

#### 方案 C：使用现成限流库

例如 Redisson 或网关限流，但仍要理解：

- 算法；
- Key；
- TTL；
- Cluster Slot；
- 故障策略。

### ⑤ 计数窗口还要考虑什么

- 时间窗口边界；
- 租户时区；
- 热 Key；
- Cluster 同槽；
- 内存上限；
- Redis 故障时开放还是拒绝；
- 返回剩余额度；
- 请求幂等性。

### ❌ 容易制造事故的写法

```java
long count = redis.incr(key);
if (count == 1) {
    redis.expire(key, 60); // 两条命令之间可以故障
}
```

### ✅ 企业级改进示例

```java
String script = """
local n = redis.call('INCR', KEYS[1])
if n == 1 then
  redis.call('EXPIRE', KEYS[1], ARGV[1])
end
return n
""";
long count = redis.eval(script, List.of(key), List.of("60"));
```

### 🎙️ 2～3 分钟优秀回答

`INCR` 后再 `EXPIRE` 是两条独立命令。如果 INCR 成功后应用崩溃或第二条命令未送达，计数 Key 可能没有 TTL，永久保留。Pipeline 只能减少 RTT，不能把两条命令变成一个原子操作。

传统做法是使用短 Lua 或 Function：在服务端执行 INCR，当结果为 1 时设置过期，然后返回计数。Redis 8.8 还提供 INCREX，把计数、范围和过期窗口组合进单命令，适合窗口计数场景，但要根据正式命令文档和客户端支持使用。

项目中还要考虑窗口边界、热 Key、Cluster Slot、限流状态的淘汰策略，以及 Redis 故障时 Fail-Open 还是 Fail-Closed。关键原则是“创建计数和设置生命周期必须成为同一个原子状态变化”。

### 面试官可能继续追问

- 为什么把两条命令放入 Pipeline 仍不是原子？
- 固定窗口在边界处为什么可能瞬间放过双倍请求？

> **记忆句**：计数创建和 TTL 初始化必须一起成功，不能靠两条普通命令拼运气。

---
