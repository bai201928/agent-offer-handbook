# Redis 8.8 零基础企业级教材

这套教材面向：

- Redis 零基础或基础薄弱的学习者；
- Agent 应用开发工程师；
- Java 后端工程师；
- 希望应对互联网大厂高薪岗位 Redis 面试的同学。

## 教学原则

旧版虽然覆盖了源码、COW、PSYNC、Cluster、缓存一致性、锁和 Agent Memory，但开场术语密度过高。新版采用以下顺序：

```text
先看懂 SET/GET
  ↓
认识 Key、Value、TTL、Hit/Miss
  ↓
认识六种高频数据类型
  ↓
画出 Java → Redis 的请求链
  ↓
再学习慢、丢、重、过期、淘汰和内存
  ↓
最后进入源码、高可用和 Agent 项目深挖
```

每道题包含：

1. 面试官到底在问什么；
2. 一个具体业务时间线；
3. 一句人话解释；
4. 机制逐步运行；
5. 项目中的使用方式；
6. 容易制造事故的 Java 写法；
7. 企业级改进代码；
8. 2～3 分钟优秀回答；
9. 递进追问和记忆句。

## 文件顺序

1. [00_Redis_Foundation_and_Project.md](./00_Redis_Foundation_and_Project.md)  
   从 `SET session:1001 ... EX 1800` 开始，解释 Key、Value、TTL、命中、数据类型、连接、请求链、过期、持久化、主从、Sentinel 和 Cluster。

2. [01_Interview_Q01_Q10.md](./01_Interview_Q01_Q10.md)  
   Redis 为什么快、数据类型与编码、大 Key/热 Key、端到端延迟、Pipeline、RDB/AOF、fork/COW、复制、Sentinel 和 Cluster。

3. [02_Interview_Q11_Q20.md](./02_Interview_Q11_Q20.md)  
   Cache Aside、旧值回填、穿透/击穿/雪崩、两级缓存、Redis 故障保护、原子计数、分布式锁、Fencing、限流和 Stream。

4. [03_Interview_Q21_Q30_and_Review.md](./03_Interview_Q21_Q30_and_Review.md)  
   过期与淘汰、淘汰策略、RSS 与碎片、maxmemory、Lazy Free、Cluster 倾斜、Agent Memory、Checkpoint、并发状态、语义缓存和综合事故。

## 贯穿项目

NexusAgent 是一个多租户 Agent + RAG 平台：

```text
Agent Service
  ├─ Redis：会话、短期记忆、限流、租约、幂等、热缓存
  ├─ MySQL：状态机、版本、用户和审计事实
  ├─ Milvus：向量检索
  ├─ Kafka/RocketMQ：可靠异步链
  ├─ 对象存储：大工具结果和快照正文
  └─ LLM/Tool：外部副作用
```

教材不会把 Redis 描述成“万能中间件”，而是明确每一种保证的边界。

## 70% 掌握目标

完成教材后，应能独立回答至少 21 道主线题，并能画出：

```text
Java 请求链
缓存读写与失效链
复制与故障转移链
锁/Fencing/幂等链
内存上涨与回源积压链
Agent Checkpoint 恢复链
```

低频源码位级优化、项目不会使用的冷门命令和只适合背诵的争议题不单独设题。
