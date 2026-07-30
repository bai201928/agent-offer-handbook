# Redis 8.8 面试题 Q04

## Q04. 一条 Redis 命令的端到端延迟由哪些部分组成

**类型：高频排障题｜难度：中等偏上｜重要性：★★★★★**

### ① 为什么 Slow Log 很短，接口仍可能很慢

Java 接口耗时 120ms，Redis Slow Log 只有 2ms，并不矛盾。

因为一次调用还包括：

```text
Java 线程等待
+ 客户端连接/命令队列
+ 命令编码
+ 请求网络
+ Redis 事件循环等待
+ 命令执行
+ 回复构造和输出缓冲
+ 返回网络
+ Java 解码与回调
```

Slow Log 主要观察 Redis 真正执行命令的阶段，不是整条链路。

### ② 一条命令分段理解

```text
A. Java 业务线程准备命令
B. Lettuce 把命令交给 Netty EventLoop
C. 命令可能在客户端 Pending 队列等待
D. TCP 把请求送到 Redis
E. Redis 读取和解析
F. 命令等待前面的工作完成
G. 执行 GET/HGET/ZRANGE
H. 生成回复并写入 Output Buffer
I. 网络返回
J. Lettuce 解码并完成 Future
K. 业务线程继续
```

### ③ 常见“服务端不慢但接口慢”的原因

- 前面有一个大回复，占住同一连接；
- 网络重传或跨可用区；
- 客户端 EventLoop 被阻塞；
- 连接池等待；
- Java 反序列化慢；
- GC；
- Output Buffer 堆积；
- 命令还没真正到达 Redis；
- 总 Deadline 被多层重试消耗。

### ④ 工具分别能看什么

| 工具 | 主要观察 |
|---|---|
| SLOWLOG | 命令执行时间 |
| LATENCY | fork、AOF、过期等延迟事件 |
| commandstats | 命令次数与累计耗时 |
| INFO clients | 连接与缓冲 |
| Java Trace | 整体链路和客户端排队 |
| 网络指标 | RTT、重传、带宽 |
| JVM 指标 | EventLoop、GC、线程池 |

### ⑤ 排查顺序

```text
先看端到端 Trace
  ↓
确认时间花在客户端、网络还是 Redis
  ↓
看回复字节和 Pending
  ↓
再看 Slow Log、Latency、CPU
  ↓
定位大 Key、长脚本、fork 或网络问题
```

盲目增加超时时间只会把故障更晚暴露。

### ❌ 容易制造事故的写法

```java
try {
    return redis.get(key);
} catch (RedisCommandTimeoutException e) {
    // 不定位，直接把超时从 100ms 改成 5s，并重试三次。
    return retryThreeTimes(() -> redis.get(key));
}
```

### ✅ 企业级改进示例

```java
try (var span = tracer.spanBuilder("redis.get").startSpan()) {
    span.setAttribute("redis.key_class", "run-meta");
    span.setAttribute("redis.expected_max_bytes", 4096);
    return redis.get(key);
} finally {
    // 同时采集客户端 pending、RTT、回复字节和服务端指标。
}
```

### 🎙️ 2～3 分钟优秀回答

Redis 接口延迟必须拆成端到端阶段。除了服务端命令执行，还包括 Java 调度、Lettuce 命令队列、网络、Redis 事件循环等待、回复缓冲、返回网络、Java 解码和 GC。因此接口 P99 很高而 Slow Log 很短是完全可能的。

排查时我先看 Trace，确认连接等待、客户端 Pending 和总 Deadline；再看网络 RTT、重传和带宽；服务端看 SLOWLOG、LATENCY、commandstats、CPU、Output Buffer；最后检查大 Key、大回复、长 Lua 和 JVM EventLoop。

项目中曾有一个 HGETALL 返回十几 MB，Redis 命令执行不算特别慢，但大回复占住共享连接，导致后面的租约续期 Future 延迟完成。拆 Key、限制字段、隔离关键连接后恢复。

所以我的原则是先分段定位，不能看到 Redis Timeout 就只加连接池或超时时间。

### 面试官可能继续追问

- Lettuce 为什么通常不需要为每个线程创建连接？
- 同一连接上的大回复如何造成队头阻塞？

> **记忆句**：Slow Log 只是一段；端到端延迟要从 Java 一直量到 Java。

---
