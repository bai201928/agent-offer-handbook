# Redis 8.8 面试题 Q29

## Q29. Agent 语义缓存为什么不能只看向量相似度

**类型：Agent/RAG 高薪项目题｜难度：中等偏上｜重要性：★★★★★**

### ① 什么是语义缓存

普通缓存按完全相同 Key 命中。

语义缓存希望：

```text
“怎么申请退款”
“退款流程是什么”
```

问题表达不同但语义相近，可以复用结果。

### ② 只看向量相似度有什么风险

两句话相似，不代表答案可以安全复用。

还要考虑：

- 租户；
- 用户权限；
- 知识库；
- 知识库版本；
- 时间；
- 地区；
- 模型；
- Prompt；
- Tool 状态；
- 会话上下文；
- 输出格式；
- 风险等级。

例子：

```text
员工 A 有财务权限
员工 B 没有
```

问题完全相同，也不能共享包含敏感数据的答案。

### ③ 缓存身份分两层

第一层：硬过滤身份。

```text
tenant + ACL + KB version + model + template + tool policy
```

只有这些一致，才进入候选集合。

第二层：语义相似度。

```text
embedding similarity ≥ threshold
```

再通过 reranker、关键词或业务规则校验。

### ④ Value 应保存什么

- 原问题；
- 规范化问题；
- 答案；
- 来源文档 ID；
- 文档版本；
- 模型和 Prompt 版本；
- ACL 摘要；
- 创建时间；
- 过期时间；
- 风险标签；
- 置信度。

### ⑤ TTL 与失效

政策类知识更新后，旧答案需要按版本失效。

高风险回答 TTL 更短，或完全不使用语义缓存。

### ⑥ Redis 与向量库怎样配合

Redis 可保存：

- 热候选；
- 结果对象；
- 相似度索引的热点层；
- 命中统计。

大规模向量召回通常由 Milvus/Redis Search 等向量能力承担。无论使用哪种组件，都必须先做租户和 ACL 隔离。

### ⑦ 如何评估

- 命中率；
- 正确复用率；
- 错误复用率；
- 权限泄漏率必须为零；
- 节省 Token/延迟；
- 不同阈值下的 Precision；
- 知识版本切换恢复时间；
- 人工抽检。

### ❌ 容易制造事故的写法

```java
// 只要余弦相似度 > 0.85 就跨租户返回历史答案。
if (cosine(queryVector, cached.vector()) > 0.85) {
    return cached.answer();
}
```

### ✅ 企业级改进示例

```java
SemanticIdentity identity = new SemanticIdentity(
        tenantId, aclVersion, kbVersion, modelVersion, promptVersion);
List<Candidate> candidates = cache.searchWithin(identity, queryVector);
return candidates.stream()
        .filter(c -> c.similarity() >= threshold)
        .filter(policyValidator::safeToReuse)
        .findFirst();
```

### 🎙️ 2～3 分钟优秀回答

语义缓存不能只看向量相似度，因为表达相似不代表业务身份和权限相同。安全命中应先做硬过滤，包括 tenant、ACL、knowledge base version、模型、Prompt、Tool policy 和输出要求，只有这些一致后才比较语义相似度，并可增加 reranker 与业务规则。

缓存 Value 应保存来源文档、版本、模型和模板版本、ACL 摘要、时间、风险标签与置信度。知识发布新版本时通过版本化 Key 自然切换，高风险、权限和强时效问题可以禁用语义缓存或缩短 TTL。

评估不能只看命中率，还要看正确复用率、错误复用率、权限泄漏、Token 节省和版本切换效果。项目目标是安全地复用答案，而不是尽可能多命中。

### 面试官可能继续追问

- 相似度阈值应该怎样离线评估？
- 知识库更新时如何快速失效语义缓存？

> **记忆句**：先匹配身份和权限，再匹配语义；相似只是候选，不是授权。

---
