# Redis 8.8 面试题 Q27

## Q27. 一个可恢复的 Checkpoint 应包含什么

**类型：资深 Agent 项目深挖题｜难度：中等偏上｜重要性：★★★★★**

### ① Checkpoint 首先回答恢复问题

Worker 在步骤 4 后宕机。

恢复者需要知道：

- Run 是哪个；
- 当前状态；
- 已完成哪些步骤；
- 下一步是什么；
- 哪些外部 Tool 已调用；
- Tool 的 requestId；
- 结果存在哪里；
- 状态版本；
- 谁拥有租约；
- Checkpoint 的 schema 版本；
- 是否允许重放。

### ② 最小结构示例

```json
{
  "runId": "R1",
  "version": 18,
  "phase": "TOOL_COMPLETED",
  "completedSteps": ["plan", "retrieve", "tool:refund-check"],
  "nextStep": "compose-answer",
  "toolRequests": {
    "refund-check": {
      "requestId": "E1001",
      "status": "SUCCESS",
      "resultRef": "s3://.../result.json"
    }
  },
  "schemaVersion": 3,
  "updatedAt": "..."
}
```

### ③ 为什么大结果只存引用

Checkpoint 应小而稳定。工具返回 50MB 文件时：

```text
对象存储保存正文
Checkpoint 保存 resultRef、hash、size、status
```

### ④ 恢复过程

```text
读取权威 Checkpoint
  ↓
校验 schema 和版本
  ↓
获取新租约与 Fencing Token
  ↓
核对已完成副作用
  ↓
从 nextStep 继续
  ↓
条件更新新版本
```

### ⑤ Redis 热副本与权威 Checkpoint

可以：

```text
MySQL/对象存储 = 权威
Redis = 热缓存
```

Redis 丢失时从权威源恢复。

若 Redis 是唯一 Checkpoint，必须明确持久化、复制、淘汰和灾备边界，风险通常更高。

### ⑥ Schema 演进

旧版本 Checkpoint 可能由新代码读取。

需要：

- schemaVersion；
- 向后兼容；
- 迁移函数；
- 未知字段容忍；
- 灰度；
- 回滚；
- 旧任务恢复测试。

### ⑦ 验证

故障注入：

- 每一步前后杀进程；
- Tool 成功后、写 Checkpoint 前宕机；
- Redis 切换；
- 重复消息；
- 旧 Worker 恢复；
- Schema 升级中恢复。

### ❌ 容易制造事故的写法

```java
record Checkpoint(String runId, byte[] entireConversationAndAllFiles) {}
// 无版本、无步骤状态、无副作用 ID，也无法并发更新。
```

### ✅ 企业级改进示例

```java
record Checkpoint(
        String runId,
        long version,
        String phase,
        Set<String> completedSteps,
        String nextStep,
        Map<String, ToolCallState> toolCalls,
        int schemaVersion) {}
```

### 🎙️ 2～3 分钟优秀回答

可恢复 Checkpoint 需要记录 Run 身份、状态版本、已完成步骤、下一步、外部副作用的 requestId 与状态、结果引用、租约代次、schemaVersion 和更新时间。大工具结果不应直接塞入 Checkpoint，而是放对象存储，Checkpoint 保存引用、哈希和大小。

恢复时先读取权威 Checkpoint，校验 Schema 与版本，获取新的租约和 Fencing Token，核对哪些副作用已经成功，再从 nextStep 继续，并通过条件更新写入更高版本。Redis 可以保存热副本，但 MySQL 或对象存储通常承担权威事实。

我还会对每个步骤边界做进程终止、重复消息、Redis 切换和旧 Worker 恢复测试。Checkpoint 的价值不在于存得多，而在于能明确、安全、幂等地继续。

### 面试官可能继续追问

- Tool 成功但 Checkpoint 写失败时如何恢复？
- Checkpoint 为什么需要业务版本和 Schema 版本两种版本？

> **记忆句**：Checkpoint 不求存全，只求能判断已做什么、不能重做什么、下一步做什么。

---
