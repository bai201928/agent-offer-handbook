# 第五章：企业级 Agent/RAG 平台项目深挖

## Q34. 如何从零设计一个企业级多租户 Agent/RAG 平台？

**题型**：项目系统设计题  
**实际难度**：困难  
**学习层级**：70% 必会题

### 业务目标

平台需要接收短任务和长任务，检索企业知识，调用模型和 MCP 工具，并支持多租户、权限、审计、恢复、成本和容灾。

### 最小架构

```text
企业用户
  → Gateway / Agent API
  → Run Service / MySQL 事实 + Outbox
  → RocketMQ
  → Runtime Worker / LangGraph4j
  → RAG / Milvus
  → Model Gateway
  → MCP Tool Gateway
  → RunEvent / SSE
  → Reconciler
```

### 数据所有权

| 状态 | 主要位置 |
|---|---|
| Run、权限、版本、审批、ToolOperation、Outbox | MySQL 事实 |
| 热点、限流、租约、可重建热 Checkpoint | Redis |
| 调度和事件传播 | RocketMQ |
| 带知识与权限版本的检索索引 | Milvus |
| 模型输出 | 候选结果，不拥有业务最终写权 |

### 2～3 分钟面试口述回答

我会先定义平台用户、任务是否长时间、是否调用高风险工具，以及一次 Run 什么时候才算成功。控制面管理租户、Agent、Prompt、模型、工具、Policy、知识库和图版本；数据面承载在线 API、Run、检索、模型和工具执行。MySQL 保存 Run、权限、版本、Outbox 和 ToolOperation 等事实，Redis 保存可失效热点和协调状态，Milvus 保存带知识和权限版本的派生索引，RocketMQ 负责至少一次触发。

新 Run 在入口固定 Manifest 和绝对 deadline，事务提交 Run 与 Outbox 后返回 runId。Worker 按状态机执行并保存 Checkpoint，工具调用使用稳定 operationId、审批和 Fencing，SSE 从持久 RunEvent 恢复。多租户隔离贯穿身份、数据库、Key、索引、队列、连接和成本。最后用版本不混用、跨租户哨兵、重复副作用为零、故障恢复和 RPO/RTO 演练证明系统成功，而不是只看模型能否返回文本。

### 递进追问

- 控制面故障为什么不应改变在途 Run？
- 哪些状态必须放 MySQL，哪些可以只放 Redis？

---

## Q35. Agent 平台为什么要区分控制面、数据面和领域边界？

**题型**：项目架构深挖题  
**实际难度**：困难  
**学习层级**：进阶加分题

### 事故画面

Worker 每个 Step 都读取最新 Prompt、Tool 和 Policy；滚动升级过程中，同一个 Run 混用了新旧模型、知识和图版本，结果无法重放。

### 设计原则

1. 控制面发布不可变配置和版本，不直接执行在线任务。
2. 数据面按 Run 固定 Manifest 执行。
3. 领域边界按业务规则和事实所有权划分，不按 Redis、MQ、Controller 等技术层拆分。
4. 早期优先模块化单体；只有独立扩容、合规、隔离和团队自治收益明确时才拆服务。
5. 发布采用版本兼容、灰度、回滚和旧版本保留窗口。

### 2～3 分钟面试口述回答

控制面决定允许使用哪些租户、策略和版本，数据面负责执行一次具体 Run。新 Run 在入口解析不可变 Manifest，其中包含 Prompt、模型、工具、Policy、知识和图版本；在途 Run 始终按这个快照执行，因此控制面短暂故障或滚动发布不会让语义漂移。

领域边界按业务规则和数据所有权划分，例如 Run、Knowledge、ToolOperation 和 Billing 各自拥有事实，不按 Redis、MQ 或 Controller 拆服务。早期可以采用模块化单体保持事务和调试简单，只有独立扩容、故障隔离、合规或团队自治收益超过分布式成本时才拆。这样控制面配置可以灰度回滚，数据面任务可以稳定重放和审计。

### 递进追问

- 旧 Manifest 和旧模型版本什么时候可以删除？
- 控制面与数据面能否共用数据库？

---

## Q36. 一次 Agent Run 的状态机、版本快照、Checkpoint 和 SSE 怎样设计？

**题型**：项目可靠性深挖题  
**实际难度**：困难  
**学习层级**：进阶加分题

### 状态机

```text
QUEUED
→ RUNNING
→ WAITING_APPROVAL / UNKNOWN
→ SUCCEEDED / FAILED / CANCELED
```

### 核心机制

- Manifest 固定 Agent、Prompt、Model、Tool、Policy、Knowledge 和 Graph 版本。
- `stateVersion` 防止普通并发覆盖。
- `generation/fencing` 阻止旧 Worker 接管后继续写。
- Checkpoint 保存已完成 Step、下一安全位置、operationId 和必要上下文。
- RunEvent 使用单调 sequence 持久化，SSE 按 `Last-Event-ID` 续传。

### 2～3 分钟面试口述回答

一次 Run 需要明确状态机和中间态，例如 QUEUED、RUNNING、WAITING_APPROVAL、UNKNOWN 和终态。创建时固定 Manifest，所有状态更新带 stateVersion 和 generation，旧 Worker 条件更新失败后必须停止。Checkpoint 保存已完成步骤、下一安全节点、工具 operationId 和必要上下文，允许新 Worker 从安全位置恢复，而不是从头再次调用外部工具。

进度事件持久化为 RunEvent 并使用单调 sequence，SSE 只是实时投影；客户端断线后通过 Last-Event-ID 补发。恢复时新 Worker 领取更高 generation，重读 Run、Checkpoint、Outbox 和 ToolOperation，只重放幂等安全步骤。Checkpoint 解决恢复，RunEvent 解决进度和审计，两者都不能只存在 JVM 内存。

### 递进追问

- Checkpoint 应该保存完整上下文还是摘要？
- RunEvent 与状态更新怎样保证一致顺序？

---

## Q37. RAG 文档入库和在线检索怎样同时保证版本、权限、引用和原子发布？

**题型**：项目 RAG 深挖题  
**实际难度**：困难  
**学习层级**：进阶加分题

### 入库链

```text
创建 knowledgeVersion
→ 文档解析与切片
→ Embedding
→ staging Milvus 索引
→ 数量/维度/权限/召回校验
→ 原子切换 activeVersion
→ 保留旧版本回滚窗口
```

### 在线读链

```text
tenantId + knowledgeVersion + aclVersion
→ 向量检索与 Metadata 过滤
→ 权威 Policy 二次裁决
→ 生成答案并保存 documentId/chunkId/sourceVersion 引用
```

### 2～3 分钟面试口述回答

RAG 入库不能边构建边被线上使用。我会为每次发布创建不可变 knowledgeVersion，解析、切片、Embedding 和 Milvus 索引都写 staging 版本。构建完成后校验文档数、Chunk 数、向量维度、权限标签和抽样召回，通过后在事实库原子切换 activeVersion，旧版本保留回滚窗口。

在线 Run 固定 knowledgeVersion 和 aclVersion，检索请求携带 tenantId 和权限过滤，返回候选后仍由权威 Policy 再裁决，防止索引延迟造成越权。答案引用保存 documentId、chunkId、来源版本和摘要，便于审计和重放。权限撤销通过版本提升和可靠事件更新缓存与索引；版本落后时宁可拒绝或回源，也不能返回可能越权的数据。

### 递进追问

- Milvus 蓝绿切换失败如何回滚？
- 文档删除后怎样证明所有派生副本已失效？

---

## Q38. MCP Tool Gateway 怎样实现零信任、幂等、审批和未知结果恢复？

**题型**：项目工具安全深挖题  
**实际难度**：困难  
**学习层级**：进阶加分题

### 安全边界

1. 模型输出只视为候选 ToolCall，不能直接获得凭据和任意网络访问。
2. Gateway 校验认证主体、tenant、Tool 白名单、参数 Schema、Policy 和风险等级。
3. 高风险操作进入 `WAITING_APPROVAL`。
4. 使用短期最小权限凭据、网络 allowlist 和资源级授权。
5. 每次操作使用稳定 `operationId` 并持久化 ToolOperation。
6. 超时进入 `UNKNOWN`，使用相同 operationId 查询、对账或转人工。

### 2～3 分钟面试口述回答

MCP Tool Gateway 的核心是把模型与真实权限隔离。模型输出只是候选调用，不能直接获得密钥或任意网络访问。Gateway 根据认证主体、租户、工具白名单、参数 Schema、Policy、风险等级和审批进行裁决，高风险操作进入 WAITING_APPROVAL。

每次工具调用使用稳定 operationId，先持久化 ToolOperation，再使用短期最小权限凭据和网络 allowlist 执行。超时不能直接当失败并换 ID 重试，因为远端可能已经完成；状态应进入 UNKNOWN，使用相同 operationId 查询远端、对账或人工裁决。所有结果记录摘要、版本和审计事件。Prompt Injection 即使影响模型文本，也不能绕过模型外部的授权和网络边界。

### 递进追问

- 外部工具不支持幂等查询怎么办？
- 审批完成后怎样阻止旧请求继续执行？

---

## Q39. 多租户 Agent 平台怎样治理 Noisy Neighbor、资源隔离和成本归因？

**题型**：项目高薪题  
**实际难度**：困难  
**学习层级**：进阶加分题

### 隔离维度

- 身份：tenantId 只能来自认证上下文。
- 数据：进入 MySQL 唯一键、Redis Key、Milvus 过滤、MQ Header 和审计。
- 容量：租户 QPS、并发、Token、存储、队列和工具预算。
- 工作负载：在线、批量、检索、Tool、Reconciler 使用独立队列和资源池。
- 成本：模型 Token、Embedding、检索、工具、存储、消息和人工审批。

### 2～3 分钟面试口述回答

多租户隔离要贯穿身份、数据和资源。tenantId 必须来自认证上下文，进入 MySQL 唯一键、Redis Key、Milvus 过滤、消息 Header 和审计，不能相信模型或请求体自报。资源上按租户等级限制 QPS、并发、Token、存储、队列和工具调用；在线 Run、批量导入和 Reconciler 使用不同队列、线程池和连接预算。

超级租户可以独立分片或使用专属资源，防止 Noisy Neighbor。成本按模型 Token、Embedding、检索、工具、存储、消息和人工审批归因，但优化不能只看便宜，还要结合业务成功率、质量和尾延迟。指标按租户等级聚合，具体 tenantId 放 Trace 和日志，避免指标高基数。

### 递进追问

- 如何证明所有查询都带租户过滤？
- 共享资源如何平滑迁移到专属资源？

---

## Q40. 综合系统设计：十倍流量、模型和 Milvus 抖动、恶意请求与区域故障同时发生怎么办？

**题型**：终极系统设计题  
**实际难度**：困难  
**学习层级**：进阶加分题

### 复合事故

```text
活动流量十倍
+ 热点租户长 Run
+ 模型限流
+ Milvus P99 上升
+ 恶意文档诱导 MCP 访问内网
+ 主区域网络分区
```

### 处置顺序

1. **止血**：冻结发布，租户/全局限流，暂停批量与高风险工具，保护身份和 MySQL 事实。
2. **隔离**：在线/批量/Reconciler 分舱；模型和 Milvus 使用有界并发、deadline、单层重试和熔断。
3. **降级**：缓存只读、减小检索范围、备用模型标记质量等级；UNKNOWN 不伪装成功。
4. **安全**：MCP Policy、审批、短期凭据、参数校验和网络 allowlist。
5. **容灾**：新 generation 单写区域接管，旧区域写被 Fencing 拒绝。
6. **恢复**：身份与 MySQL → Outbox/MQ → Runtime → Redis/Milvus 派生状态 → 分阶段放流。
7. **验收**：对账 Run、权限、索引和工具副作用，验证 P99、重复副作用、跨租户哨兵和 RPO/RTO。

### 2～3 分钟面试口述回答

这是一个复合事故，我会先止血而不是盲目扩容。冻结发布，按租户和全局限流，暂停批量导入和高风险工具，保护身份和 MySQL 事实库。在线、批量和 Reconciler 使用独立舱壁；模型和 Milvus 设置有界并发、总 deadline、单层重试和熔断。可接受的请求使用缓存或降低检索规模，备用模型必须记录版本和质量降级，外部工具超时保持 UNKNOWN。

恶意请求由 MCP Gateway 的 Policy、参数校验、短期凭据和网络 allowlist 拦截。区域分区时只允许新 generation 的单写区域推进，旧区域写被 Fencing 拒绝。灾备恢复顺序是身份与事实库、Outbox/消息、运行时，再重建 Redis 和 Milvus，最后分阶段放流。恢复后对账 Run、权限、索引和外部副作用，并用业务成功率、P99、重复副作用、跨租户哨兵和 RPO/RTO 证明系统真正恢复。

### 递进追问

- 灾备切换时未消费消息怎样处理？
- 怎样设计演练证明这不是纸面架构？
