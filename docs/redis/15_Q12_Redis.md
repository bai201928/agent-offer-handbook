# Redis 8.8 面试题 Q12

## Q12. 数据库更新并删除缓存后，为什么旧值仍可能回来

**类型：高频并发场景题｜难度：中等偏上｜重要性：★★★★★**

### ① 先看完整时间线

数据库当前版本是 v7，缓存为空。

```text
请求 A：Redis Miss
请求 A：从 MySQL 读到 v7
请求 A：开始做较慢的序列化

请求 B：把 MySQL 更新到 v8
请求 B：删除 Redis 缓存

请求 A：序列化完成
请求 A：把手里的 v7 写回 Redis
```

最终：

```text
MySQL = v8
Redis = v7
```

请求 B 明明删除过缓存，为什么仍失败？

因为请求 A 已经把旧数据读进 Java 内存。`DEL` 只能删除执行当时 Redis 中的 Key，不能删除 Java 中“正在路上的旧值”。

### ② TTL 能否解决

短 TTL 可以缩短错误持续时间，但不能阻止竞态发生。

### ③ 版本化 Key

知识库当前激活版本从 v7 切换到 v8：

```text
旧缓存：
retrieval:KB17:v7:queryHash

新缓存：
retrieval:KB17:v8:queryHash
```

迟到的 v7 只会写旧命名空间，不污染 v8 的读路径。

这是最容易理解、也常常最可靠的方式。

### ④ Value 携带版本

缓存值包含：

```json
{
  "version": 8,
  "data": "..."
}
```

写入时比较当前版本，只允许不低于已有版本的数据覆盖。

比较和写入必须在一个原子操作中完成，可以使用：

- 短 Lua/Function；
- WATCH；
- 适合的条件命令。

### ⑤ Tombstone 或最低有效版本

失效事件记录：

```text
KB17 当前最低有效版本 = 8
```

v7 请求回填前发现版本太旧，直接放弃。

### ⑥ RAG 项目中的缓存身份

语义检索结果不能只按问题文本做 Key，还应包含：

- tenant；
- knowledgeBaseId；
- activeVersion；
- ACL 版本；
- embedding 模型版本；
- reranker 版本；
- prompt/template 版本；
- query 归一化版本。

否则不是简单的“缓存旧”，而可能是权限或模型身份错误。

### ❌ 容易制造事故的写法

```java
String value = repository.load(kbId); // 可能读到旧版本
slowSerialize(value);
redis.set("retrieval:" + kbId + ":" + queryHash, value);
// 没有版本，迟到请求可以覆盖新缓存。
```

### ✅ 企业级改进示例

```java
long version = repository.loadActiveVersion(kbId);
String key = "retrieval:%s:v%d:%s".formatted(kbId, version, queryHash);
RetrievalResult result = repository.search(kbId, version, query);
redis.setex(key, ttlWithJitter(), serialize(result));
```

### 🎙️ 2～3 分钟优秀回答

删除缓存后旧值仍可能回来，是因为并发请求可能已经在删除前从数据库读到旧版本，并在删除后才完成回填。DEL 只能删除 Redis 中当时的 Key，无法感知 Java 内存里正在处理的旧结果。

最稳妥的方案通常是版本化 Key，例如知识库发布 v8 后，所有新请求使用包含 v8 的缓存身份，迟到的 v7 回填只写旧命名空间。也可以让 Value 携带版本，并通过 Lua、WATCH 或条件写原子判断；或者维护最低有效版本，拒绝旧回填。

在 RAG 项目中，缓存身份还要包含租户、知识库版本、ACL、Embedding、Reranker 和模板版本。短 TTL 只能缩短错误窗口，不能根治并发竞态。核心是让事实源版本拥有最终裁决权。

### 面试官可能继续追问

- 版本化 Key 会不会产生大量旧 Key？
- Value 版本比较为什么必须原子完成？

> **记忆句**：DEL 只能删 Redis 里的旧值，删不掉 Java 中已经出发的旧回填。

---
