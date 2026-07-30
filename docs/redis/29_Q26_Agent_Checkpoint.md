# Redis 8.8 面试题 Q26

## Q26. Agent 的短期记忆、长期记忆和 Checkpoint 有什么区别

**类型：Agent 项目核心题｜难度：中等｜重要性：★★★★★**

### ① 短期记忆

服务于当前会话或当前 Run。

例如：

- 最近 N 轮对话；
- 当前工具调用结果；
- 当前计划；
- 临时检索结果；
- 当前 Token 预算。

特点：

- 高频读写；
- 生命周期短；
- 可设置 TTL；
- 通常可以压缩或重建。

Redis 很适合保存热短期记忆，但要限制大小。

### ② 长期记忆

跨会话长期保留，并需要检索、审计、版本和权限。

例如：

- 用户长期偏好；
- 已确认事实；
- 企业知识；
- 历史经验摘要；
- 可追溯记忆来源。

通常以 MySQL、对象存储、Milvus 等作为事实与检索系统，Redis 只缓存热点索引或结果。

### ③ Checkpoint

Checkpoint 是 Agent 工作流在某个安全点的可恢复状态。

它不是“所有日志的备份”，而是回答：

```text
系统重启后，从哪里继续？
哪些步骤已经完成？
哪些副作用不能重复？
下一步应该执行什么？
```

### ④ 三者关系

```text
短期记忆 → 帮助当前推理
长期记忆 → 跨会话保留知识
Checkpoint → 帮助工作流恢复
```

同一段文本可能同时被用于短期上下文和长期记忆，但生命周期、事实源和恢复语义不同。

### ⑤ 为什么不能把所有内容塞进一个 Redis JSON

- 大 Key；
- 每次改一个字段要整体覆盖；
- 并发更新丢失；
- TTL 难分；
- 敏感数据难分级；
- Checkpoint 与普通缓存一起被淘汰；
- 恢复时无法判断副作用状态。

### ⑥ 推荐分层

```text
Redis：
  热 Session、最近消息、当前租约、热 Checkpoint、短期缓存

MySQL：
  Run 状态机、步骤事实、幂等、审计、Checkpoint 索引

对象存储：
  大工具结果、完整上下文快照

Milvus：
  长期语义记忆向量
```

### ❌ 容易制造事故的写法

```java
// 一个 Key 同时保存最近对话、长期记忆、工具结果、审计和 Checkpoint。
redis.set("agent:" + runId, serialize(allState));
```

### ✅ 企业级改进示例

```java
redis.hset(runMetaKey, smallRunMeta);
redis.setex(shortMemoryKey, shortTtl, compactConversation);
checkpointRepository.save(authoritativeCheckpoint);
objectStore.put(largeToolResult);
```

### 🎙️ 2～3 分钟优秀回答

短期记忆服务当前会话和 Run，例如最近对话、当前计划和临时工具结果，特点是高频、短生命周期和可压缩，适合 Redis 热存储。长期记忆跨会话保存，需要来源、权限、版本和检索，通常由 MySQL、对象存储和 Milvus 作为事实与检索系统，Redis 只做热点缓存。

Checkpoint 是工作流的可恢复状态，记录已完成步骤、副作用、版本和下一步，不等于完整日志或全部上下文。它的目标是进程故障后安全继续，而不是仅提高模型回答质量。

项目中我会把三者分层，不把整个 Agent 状态塞入一个巨大 Redis JSON。Redis 保存热状态，MySQL 保存状态机与审计，大结果放对象存储，长期语义记忆进入向量库。

### 面试官可能继续追问

- 短期记忆过期后怎样从事实源恢复？
- 长期记忆为什么不能只按向量相似度写入？

> **记忆句**：短期记忆帮当前思考，长期记忆跨会话检索，Checkpoint 让流程安全续跑。

---
