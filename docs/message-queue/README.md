# 消息队列零基础到大厂面试教材

> **主线技术**：Kafka 4.3.x 先学，RocketMQ 5.5 再对照，RabbitMQ 只保留选型所需内容。  
> **适合人群**：第一次系统学习消息队列的同学，以及准备互联网大厂 Agent 应用开发、Java 后端、AI 平台岗位的同学。  
> **学习目标**：不是背 30 道答案，而是先看懂一条消息如何产生、保存、被消费和失败，再独立回答至少 70% 的高频与项目深挖题。

## 为什么要按这个顺序学习？

很多教材一开始就讲 ISR、ACK、Exactly-once、Outbox。问题是：读者连 Partition 和 Offset 在哪里都没有画面，只能把术语背下来。

这套教材改成四步：

1. **先看懂一条消息**：谁发送、发到哪里、谁保存、谁消费；
2. **再认识 Kafka 的零件**：Broker、Topic、Partition、Offset、Key、Consumer Group、副本；
3. **再解释四类事故**：为什么会丢、会重、会乱、会积压；
4. **最后进入企业项目**：Outbox、Inbox、重试、DLQ、事务消息、监控、容量与系统设计。

## 阅读顺序

### 第 0 课：先把所有术语变成具体画面

[00｜从一条订单消息开始认识 Kafka 和 RocketMQ](./00_Kafka_RocketMQ_Foundation_and_Project.md)

完成后，你应该能够不看答案解释：

- Producer、Broker、Consumer 分别是谁；
- Topic 和 Partition 为什么不是同一个东西；
- Offset 为什么既像消息编号，又像消费书签；
- Consumer Group 为什么可以扩容，但实例数不能无限增加；
- Kafka 的副本、Leader、Follower、ISR 分别解决什么问题；
- RocketMQ 的 MessageQueue、CommitLog、ConsumeQueue、Message Group、半事务消息是什么。

### 第 1 课：基础模型与高频原理题

[01｜Q01～Q10：使用场景、Kafka 模型、消费组、顺序与性能](./01_Interview_Q01_Q10.md)

这些题解决“我知道 MQ 是什么，也能解释为什么这样设计”。

### 第 2 课：可靠性与线上故障题

[02｜Q11～Q20：ACK、副本、丢失、重复、幂等、事务、重试与积压](./02_Interview_Q11_Q20.md)

这些题解决“消息出问题时，我能定位是哪一段失败，并给出恢复方案”。

### 第 3 课：RocketMQ、选型与企业级项目深挖

[03｜Q21～Q30：RocketMQ 5.5、技术选型、Agent 项目与 MQ 系统设计](./03_Interview_Q21_Q30_and_Review.md)

这些题解决“我能把原理落到真实系统，并接受资深面试官连续追问”。

## 每道题的固定学习结构

每题不直接让你背答案，而按下面顺序展开：

1. **先听懂问题在问什么**；
2. **用一个具体业务走一遍**；
3. **再补充 Kafka / RocketMQ 机制**；
4. **指出最常见的错误理解**；
5. **给出 2～3 分钟优秀回答**；
6. **补充项目追问和记忆句**。

## 学习验收

完成整套教材后，至少做到：

- 能在纸上画出 `Producer → Broker → Topic → Partition → Consumer Group → Consumer → 数据库`；
- 能从生产端、Broker、消费端和业务数据库四段分析消息丢失；
- 能解释为什么 At-least-once 会重复，以及数据库唯一约束为什么比“先查 Redis 再处理”更可靠；
- 能根据分区数、消费者数、单条处理时间估算消费能力；
- 能处理百万级积压，而不是只回答“增加消费者”；
- 能说明 Kafka 与 RocketMQ 各自适合什么项目，并明确 RabbitMQ 只在路由灵活、传统业务消息等场景作为补充。

> 教材中的吞吐量和项目数字只用于教学推演。面试时必须替换为自己的压测、实习或项目数据，不要虚构生产经历。
