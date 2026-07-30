# 系统设计、可靠性、容量与 Agent 平台面试教材

本教材面向系统设计零基础学习者，并以互联网大厂 Java 后端、Agent 应用开发、RAG/AI 平台高薪岗位为目标。

## 学习顺序

1. [零基础课：系统设计六步法](./00_Foundation_and_Interview_Method.md)
2. [第一章：答题基础与通用架构 Q01～Q05](./01_Method_Capacity_LoadBalance_Q01_Q05.md)
3. [第二章：常问独立系统设计 Q06～Q17](./02_Common_System_Design_Q06_Q17.md)
4. [第三章：幂等、可靠性与分布式一致性 Q18～Q27](./03_Reliability_Consistency_Q18_Q27.md)
5. [第四章：容量、压测、稳定性与事故 Q28～Q33](./04_Capacity_Performance_Incident_Q28_Q33.md)
6. [第五章：企业级 Agent/RAG 项目深挖 Q34～Q40](./05_Agent_RAG_Project_DeepDive_Q34_Q40.md)
7. [复习地图、28 道 70% 必会题与错误校正](./06_Review_70_Percent_and_Corrections.md)

## 题目分布

| 范围 | 内容 |
|---|---|
| Q01～Q05 | 系统设计方法、容量、状态、同步异步、负载均衡 |
| Q06～Q17 | 秒杀、RPC、短链、点赞、ID、延迟取消、十倍流量、Redis、锁、微服务 |
| Q18～Q27 | 幂等、事实源、ACK、Outbox/Inbox、缓存一致性、事务、灾备 |
| Q28～Q33 | 容量单位、压测、背压、HPA、可观测性、事故处理 |
| Q34～Q40 | 企业 Agent/RAG 平台、Run、RAG、MCP、多租户与复合事故 |

## 教材验收目标

学完后应能：

- 独立回答至少 28 道主线题；
- 20 分钟内完成一道普通系统设计；
- 画出正常链、失败链和恢复链；
- 解释缓存、MQ、分片、副本各自的收益与代价；
- 在项目追问中讲清事实源、幂等、版本、容量、观测和灾备；
- 不把“用了某组件”当成正确性证明。
