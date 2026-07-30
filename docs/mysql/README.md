# MySQL 8.4.10 LTS 零基础企业级大厂面试教材

> 面向 MySQL 初学者、Java 后端、Agent/RAG 应用开发和 AI 平台岗位。  
> 版本基线：MySQL Community Server **8.4.10 LTS**。  
> 学习目标：从“能读懂一张表”逐步走到“能解释线上慢 SQL、并发事务、崩溃恢复、复制切换和分片项目”。

## 为什么重新生成

原始材料知识密度高，但存在三个学习问题：

1. 初学者还没建立表、行、主键和一次请求的画面，就进入 Handler、MVCC、Redo、GTID；
2. 同一章节混合基础概念、源码和项目结论，前置知识跳跃；
3. 难度标签不能反映真实追问深度，部分低频语法题占用主线时间。

新版采用：

```text
先具体
  一行数据、一次 SELECT、一次 UPDATE

再局部
  页、B+树、联合索引、回表

再并发
  事务、Read View、锁、死锁、条件更新

再故障
  Redo、Binlog、恢复、复制、切换

最后扩展
  分片、Agent 数据模型、P99、Schema 演进
```

## 文件结构

1. [零基础基础课：SQL、请求链与项目](00_Foundation_SQL_Request_and_Project.md)
2. [架构、连接、页与数据建模：Q01～Q06](01_Architecture_Data_Q01_Q06.md)
3. [索引、优化器与慢 SQL：Q07～Q13](02_Index_Optimizer_Q07_Q13.md)
4. [事务、MVCC、锁与 Spring：Q14～Q22](03_Transaction_MVCC_Lock_Q14_Q22.md)
5. [日志、恢复、复制与切换：Q23～Q29](04_Log_Recovery_Replication_Q23_Q29.md)
6. [分片、Agent 项目与生产治理：Q30～Q36](05_Sharding_Project_Q30_Q36.md)
7. [70% 验收、复习地图与错误校正](06_Review_70_Percent_and_Corrections.md)

## 每道题的固定结构

```text
① 为什么现在学
② 先建立具体画面
③ 从直觉进入标准概念
④ 深入原理与源码主链
⑤ 企业级边界与演进
⑥ 最小案例
⑦ 一句话小结
⑧ 2～3 分钟面试口述
⑨ 递进追问与记忆句
```

## 36 道题导航

| 编号 | 面试题 | 类型 | 实际难度 | 频率 | 所在章节 |
|---|---|---|---|---|---|
| Q01 | MySQL Server 层和 InnoDB 分别负责什么？ | 高频基础题 | 简单 | ★★★★★ | [01_Architecture_Data_Q01_Q06.md](01_Architecture_Data_Q01_Q06.md) |
| Q02 | 一条 SELECT 从 JDBC 到返回结果经历什么？ | 高频基础题 | 简单 | ★★★★★ | [01_Architecture_Data_Q01_Q06.md](01_Architecture_Data_Q01_Q06.md) |
| Q03 | 为什么数据库连接池不能设置得越大越好？ | 高频场景题 | 中等 | ★★★★★ | [01_Architecture_Data_Q01_Q06.md](01_Architecture_Data_Q01_Q06.md) |
| Q04 | InnoDB Buffer Pool 有什么作用？ | 高频基础题 | 简单 | ★★★★★ | [01_Architecture_Data_Q01_Q06.md](01_Architecture_Data_Q01_Q06.md) |
| Q05 | InnoDB 行格式和大字段怎样影响性能？ | 场景题 | 中等 | ★★★★☆ | [01_Architecture_Data_Q01_Q06.md](01_Architecture_Data_Q01_Q06.md) |
| Q06 | 企业级 Agent 平台的消息、Trace 和大 Prompt 应该怎样存储？ | 项目深挖题 | 中等偏上 | ★★★★☆ | [01_Architecture_Data_Q01_Q06.md](01_Architecture_Data_Q01_Q06.md) |
| Q07 | 为什么 InnoDB 索引使用 B+树，而不是二叉树或 Hash？ | 高频原理题 | 中等 | ★★★★★ | [02_Index_Optimizer_Q07_Q13.md](02_Index_Optimizer_Q07_Q13.md) |
| Q08 | 什么是聚簇索引、二级索引和回表？ | 高频原理题 | 中等 | ★★★★★ | [02_Index_Optimizer_Q07_Q13.md](02_Index_Optimizer_Q07_Q13.md) |
| Q09 | 联合索引的字段顺序应该怎样设计？ | 高频设计题 | 中等偏上 | ★★★★★ | [02_Index_Optimizer_Q07_Q13.md](02_Index_Optimizer_Q07_Q13.md) |
| Q10 | 什么是覆盖索引？覆盖列是不是越多越好？ | 高频优化题 | 中等 | ★★★★★ | [02_Index_Optimizer_Q07_Q13.md](02_Index_Optimizer_Q07_Q13.md) |
| Q11 | EXPLAIN 和 EXPLAIN ANALYZE 有什么区别？ | 高频工具题 | 中等 | ★★★★★ | [02_Index_Optimizer_Q07_Q13.md](02_Index_Optimizer_Q07_Q13.md) |
| Q12 | 已经有索引，优化器为什么仍可能选择全表扫描？ | 资深优化题 | 中等偏上 | ★★★★☆ | [02_Index_Optimizer_Q07_Q13.md](02_Index_Optimizer_Q07_Q13.md) |
| Q13 | 线上 Run 列表出现慢 SQL，你会怎样定位并治理？ | 项目深挖题 | 中等偏上 | ★★★★★ | [02_Index_Optimizer_Q07_Q13.md](02_Index_Optimizer_Q07_Q13.md) |
| Q14 | 如何真正理解事务的 ACID？ | 高频基础题 | 中等 | ★★★★★ | [03_Transaction_MVCC_Lock_Q14_Q22.md](03_Transaction_MVCC_Lock_Q14_Q22.md) |
| Q15 | MVCC 是什么？Undo 版本链怎样工作？ | 高频原理题 | 中等 | ★★★★★ | [03_Transaction_MVCC_Lock_Q14_Q22.md](03_Transaction_MVCC_Lock_Q14_Q22.md) |
| Q16 | READ COMMITTED 和 REPEATABLE READ 的主要区别是什么？ | 高频对比题 | 中等 | ★★★★★ | [03_Transaction_MVCC_Lock_Q14_Q22.md](03_Transaction_MVCC_Lock_Q14_Q22.md) |
| Q17 | 一致性读和当前读有什么区别？ | 高频并发题 | 中等 | ★★★★★ | [03_Transaction_MVCC_Lock_Q14_Q22.md](03_Transaction_MVCC_Lock_Q14_Q22.md) |
| Q18 | Record Lock、Gap Lock 和 Next-Key Lock 分别是什么？ | 高频锁题 | 中等偏上 | ★★★★★ | [03_Transaction_MVCC_Lock_Q14_Q22.md](03_Transaction_MVCC_Lock_Q14_Q22.md) |
| Q19 | MySQL 死锁是怎样产生和处理的？ | 高频场景题 | 中等偏上 | ★★★★★ | [03_Transaction_MVCC_Lock_Q14_Q22.md](03_Transaction_MVCC_Lock_Q14_Q22.md) |
| Q20 | 悲观锁和乐观锁应该怎样选择？ | 高频设计题 | 中等 | ★★★★★ | [03_Transaction_MVCC_Lock_Q14_Q22.md](03_Transaction_MVCC_Lock_Q14_Q22.md) |
| Q21 | 多个 Agent Worker 如何安全认领同一个 Run？ | 项目深挖题 | 中等偏上 | ★★★★★ | [03_Transaction_MVCC_Lock_Q14_Q22.md](03_Transaction_MVCC_Lock_Q14_Q22.md) |
| Q22 | Spring @Transactional 有哪些常见边界和失效问题？ | 高频框架题 | 中等 | ★★★★★ | [03_Transaction_MVCC_Lock_Q14_Q22.md](03_Transaction_MVCC_Lock_Q14_Q22.md) |
| Q23 | Redo Log、Undo Log 和 Binlog 有什么区别？ | 高频日志题 | 中等 | ★★★★★ | [04_Log_Recovery_Replication_Q23_Q29.md](04_Log_Recovery_Replication_Q23_Q29.md) |
| Q24 | MySQL 为什么需要 Redo 和 Binlog 的两阶段提交？ | 资深原理题 | 中等偏上 | ★★★★★ | [04_Log_Recovery_Replication_Q23_Q29.md](04_Log_Recovery_Replication_Q23_Q29.md) |
| Q25 | MySQL 崩溃恢复的大致流程是什么？ | 资深恢复题 | 中等偏上 | ★★★★☆ | [04_Log_Recovery_Replication_Q23_Q29.md](04_Log_Recovery_Replication_Q23_Q29.md) |
| Q26 | MySQL 主从复制的基本链路是什么？ | 高频架构题 | 中等 | ★★★★★ | [04_Log_Recovery_Replication_Q23_Q29.md](04_Log_Recovery_Replication_Q23_Q29.md) |
| Q27 | GTID 有什么作用？异步复制与半同步复制有什么差异？ | 资深高可用题 | 中等偏上 | ★★★★☆ | [04_Log_Recovery_Replication_Q23_Q29.md](04_Log_Recovery_Replication_Q23_Q29.md) |
| Q28 | 写主读从为什么会读不到刚写的数据？怎样解决？ | 高频一致性题 | 中等偏上 | ★★★★★ | [04_Log_Recovery_Replication_Q23_Q29.md](04_Log_Recovery_Replication_Q23_Q29.md) |
| Q29 | 主库故障后，怎样完成安全切换并保证业务可恢复？ | 项目深挖题 | 困难 | ★★★★☆ | [04_Log_Recovery_Replication_Q23_Q29.md](04_Log_Recovery_Replication_Q23_Q29.md) |
| Q30 | MySQL 什么时候需要分库分表？ | 高频架构题 | 中等 | ★★★★★ | [05_Sharding_Project_Q30_Q36.md](05_Sharding_Project_Q30_Q36.md) |
| Q31 | MySQL 分区表和分库分表有什么区别？ | 对比题 | 中等 | ★★★★☆ | [05_Sharding_Project_Q30_Q36.md](05_Sharding_Project_Q30_Q36.md) |
| Q32 | 分片键、路由表和全局 ID 应怎样设计？ | 资深架构题 | 中等偏上 | ★★★★☆ | [05_Sharding_Project_Q30_Q36.md](05_Sharding_Project_Q30_Q36.md) |
| Q33 | 跨分片事务、Join、排序和分页怎样处理？ | 资深系统设计题 | 困难 | ★★★★☆ | [05_Sharding_Project_Q30_Q36.md](05_Sharding_Project_Q30_Q36.md) |
| Q34 | 企业级 Agent/RAG 平台的 MySQL 表应该怎样设计？ | 项目深挖题 | 中等偏上 | ★★★★★ | [05_Sharding_Project_Q30_Q36.md](05_Sharding_Project_Q30_Q36.md) |
| Q35 | MySQL 接口 P99 突然升高，你会怎样定位？ | 项目终极深挖题 | 困难 | ★★★★★ | [05_Sharding_Project_Q30_Q36.md](05_Sharding_Project_Q30_Q36.md) |
| Q36 | 如何为 Agent 平台设计 MySQL 安全、容量和无损 Schema 演进？ | 资深治理题 | 困难 | ★★★★☆ | [05_Sharding_Project_Q30_Q36.md](05_Sharding_Project_Q30_Q36.md) |

## 难度标准

```text
简单
  能在 1～2 个核心概念内回答，追问较浅

中等
  需要讲清机制时间线和常见边界

中等偏上
  需要跨 SQL、存储、并发或应用层给出取舍

困难
  需要完整事故链、恢复方案、可观测性和验证闭环
```

难度以实际回答深度重新标注，不沿用原文件标签。

## 70% 验收

36 道题中，能够独立完成 25 道主线题，并能画出以下六张图，即达到基础验收：

```text
JDBC 到 InnoDB 请求图
B+树、二级索引和回表图
Read View 与锁范围图
Redo / Undo / Binlog 提交恢复图
主从复制与故障切换图
Agent 状态机、幂等与 Outbox 图
```

## 明确跳过

不作为主线题：

- 函数名清单；
- 纯 SQL 算法题；
- 冷门存储引擎；
- 页内位级实现和源码行号；
- 几乎不在项目使用的参数枚举；
- 只背定义、无法连接生产事故的题目。

## 版本与官方资料

- MySQL 8.4.10 Release Notes：<https://dev.mysql.com/doc/relnotes/mysql/8.4/en/news-8-4-10.html>
- MySQL 8.4 Reference Manual：<https://dev.mysql.com/doc/refman/8.4/en/>
- MySQL 8.4.10 Source Documentation：<https://dev.mysql.com/doc/dev/mysql-server/8.4.10/>

> 源码函数名用于建立定位地图，不要求初学者背行号。MySQL 本地事务也不能自动把 Redis、RocketMQ、Milvus 或外部 Tool 变成全局事务。
