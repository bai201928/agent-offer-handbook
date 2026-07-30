# 企业级消息队列教材导航

本模块以 **Kafka 4.3.x 为主线、RocketMQ 5.5 为第二主线、RabbitMQ 为选型补充**，面向 Agent 应用开发、Java 后端、AI 平台和数据基础设施岗位。

## 阅读顺序

1. [基础概念与企业项目设计](./00_Kafka_RocketMQ_Foundation_and_Project.md)
2. [Q01～Q10：基础模型、消费组、顺序、性能与高可用](./01_Interview_Q01_Q10.md)
3. [Q11～Q20：可靠性、幂等、事务边界、重试、积压与容量](./02_Interview_Q11_Q20.md)
4. [Q21～Q30：RocketMQ 5.5、选型、MQ 系统设计与项目深挖](./03_Interview_Q21_Q30_and_Review.md)

## 学习结果

完成三轮复习后，应能够：

- 解释 Broker、Topic、Partition、Offset、Replica、ISR、Consumer Group、MessageQueue、CommitLog、ConsumeQueue、Message Group 和半事务消息；
- 从业务意图、Producer、Broker、Consumer 四段分析消息丢失；
- 使用 Outbox、Inbox、唯一约束、版本号和 operationId 设计端到端幂等；
- 处理顺序、重试、DLQ、回放和百万级积压；
- 根据事件流与业务消息需求完成 Kafka、RocketMQ、RabbitMQ 选型；
- 用 2～3 分钟结构化回答 30 道大厂高频与深挖问题。
