# Redis 8.8 零基础企业级面试教材

本教材面向 Redis 零基础或基础薄弱的 Agent 应用开发、Java 后端、RAG 与 AI 平台岗位候选人。

## 为什么重写

旧版覆盖了事件循环、内部编码、COW、PSYNC、Cluster、缓存一致性和 Agent Checkpoint，但起点过高。新版先让学习者看懂：

```text
SET / GET
  ↓
Key、Value、TTL、Hit、Miss
  ↓
String、Hash、List、Set、ZSet、Stream
  ↓
Java → Lettuce → Redis 的请求链
  ↓
过期、淘汰、持久化、复制与故障转移
  ↓
缓存一致性、锁、内存治理与 Agent 恢复
```

每道题固定包含：具体场景、生活直觉、机制时间线、事故型 Java、企业级改进、2～3 分钟优秀回答、递进追问与记忆句。

## 基础课

1. [Key、Value、TTL、Hit 与 Miss](./00_Overview_Key_Value_TTL.md)
2. [六种高频数据类型](./01_Data_Types.md)
3. [Java 请求链、连接、过期与高可用第一张图](./02_Request_Connection_and_HA.md)
4. [NexusAgent 项目地图与学习验收](./03_Project_Map_and_Review.md)

## 30 道面试题

1. [Q01. Redis 为什么快？“因为数据在内存”为什么不够](./04_Q01_Redis.md)
2. [Q02. 数据类型、内部编码与 Key 设计有什么关系](./05_Q02_Key.md)
3. [Q03. 大 Key、热 Key 与 Hot Slot 有什么区别，怎样治理](./06_Q03_Key_Key_Hot_Slot.md)
4. [Q04. 一条 Redis 命令的端到端延迟由哪些部分组成](./07_Q04_Redis.md)
5. [Q05. Pipeline、MULTI/EXEC、WATCH 与 Lua/Function 怎样选择](./08_Q05_Pipeline_MULTI_EXEC_WATCH_Lua_Function.md)
6. [Q06. RDB、AOF 与混合持久化怎样选择](./09_Q06_RDB_AOF.md)
7. [Q07. fork 与 COW 为什么会造成内存和延迟峰值](./10_Q07_fork_COW.md)
8. [Q08. 有主从复制，为什么主节点故障后仍可能丢数据](./11_Q08_Redis.md)
9. [Q09. Sentinel 与 Cluster 有什么区别，项目中怎样选择](./12_Q09_Sentinel_Cluster.md)
10. [Q10. Slot、Hash Tag、MOVED 与 ASK 分别是什么](./13_Q10_Slot_Hash_Tag_MOVED_ASK.md)
11. [Q11. Cache Aside 的正确读写流程是什么](./14_Q11_Cache_Aside.md)
12. [Q12. 数据库更新并删除缓存后，为什么旧值仍可能回来](./15_Q12_Redis.md)
13. [Q13. 缓存穿透、击穿和雪崩有什么区别](./16_Q13_Redis.md)
14. [Q14. 本地缓存、Redis 与 SWR 怎样组成两级缓存](./17_Q14_Redis_SWR.md)
15. [Q15. Redis 故障时怎样保护 MySQL、Milvus 和消息消费链](./18_Q15_Redis_MySQL_Milvus.md)
16. [Q16. 为什么 INCR 后再 EXPIRE 有竞态，怎样原子实现计数窗口](./19_Q16_INCR_EXPIRE.md)
17. [Q17. 一个正确的 Redis 分布式锁至少需要哪些机制](./20_Q17_Redis.md)
18. [Q18. 锁、Fencing Token 与幂等分别解决什么问题](./21_Q18_Fencing_Token.md)
19. [Q19. 固定窗口、滑动窗口、令牌桶怎样选择](./22_Q19_Redis.md)
20. [Q20. Redis Stream 与 Kafka、RocketMQ 应怎样选择](./23_Q20_Redis_Stream_Kafka_RocketMQ.md)
21. [Q21. 过期删除与内存淘汰有什么区别](./24_Q21_Redis.md)
22. [Q22. maxmemory-policy 应该怎样选择](./25_Q22_maxmemory_policy.md)
23. [Q23. used_memory、RSS、内存碎片和客户端缓冲怎样理解](./26_Q23_used_memory_RSS.md)
24. [Q24. maxmemory、Lazy Free 与容量规划应该怎样做](./27_Q24_maxmemory_Lazy_Free.md)
25. [Q25. Redis Cluster 节点内存倾斜怎样发现和治理](./28_Q25_Redis_Cluster.md)
26. [Q26. Agent 的短期记忆、长期记忆和 Checkpoint 有什么区别](./29_Q26_Agent_Checkpoint.md)
27. [Q27. 一个可恢复的 Checkpoint 应包含什么](./30_Q27_Checkpoint.md)
28. [Q28. 多个 Worker 并发更新 Agent 状态，怎样避免互相覆盖](./31_Q28_Worker_Agent.md)
29. [Q29. Agent 语义缓存为什么不能只看向量相似度](./32_Q29_Agent.md)
30. [Q30. 综合事故：会话消失、延迟尖刺、MySQL 回源和 MQ 积压怎样串起来](./33_Q30_MySQL_MQ.md)

## 学习阶段

```text
Q01～Q05：正确使用 Redis
Q06～Q10：恢复、复制与高可用
Q11～Q15：缓存一致性与故障保护
Q16～Q20：原子性、锁、限流与消息
Q21～Q25：生命周期、内存与 Cluster 治理
Q26～Q30：Agent Memory、Checkpoint 与综合事故
```

## 70% 验收目标

完成教材后，应能独立回答至少 21 道主线题，并能画出：

- Java 请求链；
- Cache Aside 与版本化回填链；
- 复制与故障转移链；
- 锁、Fencing 与幂等链；
- 内存上涨、淘汰、回源与 MQ 积压链；
- Agent Checkpoint 恢复链。

低频源码位级优化、项目不会使用的冷门命令和纯争议题不单独设题。
