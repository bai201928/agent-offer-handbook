# Java 核心、集合、并发、JVM 与 Spring：初学者到大厂高薪岗位教材

这套教材由五份早期八股材料重新划分、去重和合并而来。它不是按“Java 基础 / 集合 / 并发 / JVM / Spring”五个文件机械拼接，而是按一个 Java 请求真正运行的前置依赖组织。

## 阅读顺序

1. [重构说明与学习地图](00_Reading_Map_and_Question_Migration.md)
2. [第 1 章：Java 运行与类型 Q001–Q008](01_Java_Runtime_and_Types_Q001_Q008.md)
3. [第 2 章：对象、异常、泛型与反射 Q009–Q016](02_Objects_Exceptions_Generics_Reflection_Q009_Q016.md)
4. [第 3 章：集合与数据结构 Q017–Q024](03_Collections_and_Data_Structures_Q017_Q024.md)
5. [第 4 章：并发基础 Q025–Q032](04_Concurrency_Foundations_Q025_Q032.md)
6. [第 5 章：并发工程 Q033–Q040](05_Concurrency_Engineering_Q033_Q040.md)
7. [第 6 章：JVM 内存、对象与类加载 Q041–Q048](06_JVM_Memory_Object_ClassLoading_Q041_Q048.md)
8. [第 7 章：GC 与故障排查 Q049–Q056](07_GC_and_Troubleshooting_Q049_Q056.md)
9. [第 8 章：Spring 核心 Q057–Q064](08_Spring_Core_Q057_Q064.md)
10. [第 9 章：Spring Boot、Web、数据与项目 Q065–Q072](09_Spring_Boot_Web_Data_Project_Q065_Q072.md)
11. [70% 掌握验收与复习路线](10_Review_and_70_Percent_Acceptance.md)
12. [官方版本与事实来源](99_Official_Sources.md)

## 贯穿项目

多租户 Agent + RAG Java 平台：

```text
HTTP → Spring MVC → 租户/Trace
     → Agent Application Service
     → RAG / 用户画像 / 模型 / MCP 工具
     → MySQL / Redis / MQ
     → JVM、线程池、虚拟线程、GC 与可观测性
```

## 教学原则

- 每次只引入一个新概念；
- 先用人话和运行画面解释，再给术语；
- 每道题都有事故写法、改进写法和 2～3 分钟口述；
- 不把历史实现当成现代固定答案；
- 不虚构生产经历，项目案例只用于学习和面试表达；
- 72 道题中 51 道为 70% 主线，其余用于资深岗位深挖。

## 技术基线

- 项目示例：Java 21 LTS、Spring Boot 3.5.x；
- 现代补充：Java 25 LTS、Spring Boot 4.1；
- JVM：现代 G1 / ZGC、JFR、统一 GC 日志；
- 并发：虚拟线程、JEP 491 后的 synchronized 行为；
- Spring Boot 自动配置：`AutoConfiguration.imports`。
