# Redis 8.8 面试题 Q23

## Q23. used_memory、RSS、内存碎片和客户端缓冲怎样理解

**类型：资深内存排障题｜难度：中等偏上｜重要性：★★★★★**

### ① used_memory 是什么

大致表示 Redis 分配器用于数据结构和内部对象的内存。

可以先把它理解为：

```text
Redis 认为自己已经分配并使用的主要内存
```

### ② RSS 是什么

RSS 是操作系统看到 Redis 进程实际驻留在物理内存中的页面。

```text
used_memory = Redis 逻辑使用视角
RSS         = 操作系统进程驻留视角
```

RSS 可能高于 used_memory。

### ③ 为什么 RSS 会高

- allocator 尚未把空闲页归还操作系统；
- 内存碎片；
- COW；
- 大量客户端缓冲；
- 模块内存；
- fork 子进程；
- 内存映射；
- 大页与操作系统行为。

### ④ 什么是内存碎片

假设仓库腾出很多小空位，但新来的大箱子放不进去。

Redis 已经释放部分对象，不代表操作系统能立即收回连续内存页。

`mem_fragmentation_ratio` 可作观察，但不能只看一个比值下结论：

- 数据量小时比例容易失真；
- RSS 还包含缓冲和其他区域；
- 应结合绝对字节与变化趋势。

### ⑤ 客户端缓冲为什么重要

每个客户端可能有：

- Query Buffer：还未处理的请求；
- Output Buffer：已生成但尚未发送完的回复。

慢消费者、Pub/Sub、复制连接或大回复会使缓冲增长。

例如 Java 客户端网络很慢：

```text
Redis 已构造 100MB 回复
  ↓
网络发不出去
  ↓
Output Buffer 占用内存
```

### ⑥ 排查内存上涨的顺序

```text
1. 数据集是否增长
2. Key/Value 大小与数量
3. TTL 和过期清理
4. 客户端缓冲
5. 复制/AOF 缓冲
6. RSS 与碎片
7. fork/COW
8. 模块和容器
```

不要一看到 RSS 高就立即执行危险的全量扫描或重启。

### ❌ 容易制造事故的写法

```java
if (rss > usedMemory * 1.5) {
    restartRedis(); // 只看比例就重启，可能掩盖大回复和慢客户端
}
```

### ✅ 企业级改进示例

```java
MemoryDiagnosis diagnosis = inspect(
        keyspaceBytes(),
        clientOutputBuffers(),
        replicationBuffers(),
        latestCowBytes(),
        rssBytes(),
        allocatorFragmentation());
alertWithBreakdown(diagnosis);
```

### 🎙️ 2～3 分钟优秀回答

used_memory 主要是 Redis 分配器和数据结构视角下的内存，RSS 是操作系统看到进程实际驻留的物理内存。RSS 高于 used_memory 可能来自 allocator 未归还页面、碎片、客户端和复制缓冲、fork/COW、模块或操作系统行为。

内存碎片可以理解为释放后留下很多不易复用或无法立即归还的空隙。mem_fragmentation_ratio 只能结合绝对字节和趋势判断，数据量小时比例可能失真。

排查时我先确认 Key 数、Value 字节和 TTL，再看客户端 Query/Output Buffer、复制/AOF 缓冲、RSS、COW 和容器 Working Set。曾见过慢消费者让大回复长期堆在 Output Buffer，重启只能暂时缓解，根因是无界回复与客户端背压。

### 面试官可能继续追问

- used_memory 降低后 RSS 为什么可能不立即降低？
- 输出缓冲达到限制时客户端会怎样？

> **记忆句**：used_memory 是 Redis 账本，RSS 是操作系统账单，中间差额要拆碎片、缓冲和 COW。

---
