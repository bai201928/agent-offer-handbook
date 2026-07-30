# 第 9 章：Spring Boot 到企业项目：请求怎样进入、落库、降级并被观测

## 本章目标

把自动装配、Starter、MVC、过滤链、MyBatis、微服务韧性和 Agent 项目串成一条端到端链。完成后可进行 70% 面试验收。

## 前置知识

完成 Spring 核心章节，知道容器、代理、事务和 Bean 生命周期。

## 本章学习路径

```text
启动入口 → 自动装配 → Web 请求链 → 数据访问 → 微服务韧性 → Agent 异步链 → 综合系统设计
```

> 本章不是按旧八股文件的出现顺序排列，而是按“先能运行，再解释原理，最后处理故障”的认知顺序组织。

---

### Q065：Spring Boot 启动和自动装配的现代流程是什么？

**题型**：Spring Boot 核心题　｜　**频率**：★★★★★

**为什么现在学**：旧八股只背 `spring.factories` 已不准确。现代 Boot 自动配置候选主要来自 `AutoConfiguration.imports`。

#### 1. 先用人话理解

`@SpringBootApplication` 组合配置、组件扫描和自动配置。Boot 根据 classpath、配置属性和已有 Bean 评估条件，满足条件才注册默认 Bean；用户自定义 Bean 通常让自动配置 back off。项目基线可用 Boot 3.5.x，同时了解 Boot 4.1 的 Jakarta/Servlet 与模块变化。

#### 2. 它在 JVM / 框架里怎么运行

1. SpringApplication 准备 Environment、监听器和上下文
2. 注册主配置与组件扫描得到业务 BeanDefinition
3. `@EnableAutoConfiguration` 导入自动配置候选
4. 现代候选列在 `META-INF/spring/...AutoConfiguration.imports`
5. `@ConditionalOnClass`、`OnMissingBean`、`OnProperty` 等评估
6. 创建匹配 Bean 并生成条件评估报告

#### 3. 贯穿项目怎么用

引入 Redis、JDBC、Web starter 后得到合理默认配置；自定义 DataSource 或客户端 Bean 时自动配置退让。

#### 4. 可读但容易出事故的写法

```java
// 旧式回答：所有自动配置都只从 spring.factories 读取
// 并且只要在 classpath 就无条件创建
```

#### 5. 更稳妥的企业级写法

```java
@AutoConfiguration
@ConditionalOnClass(ModelClient.class)
@ConditionalOnMissingBean(ModelClient.class)
class ModelAutoConfiguration { }
// 注册到 AutoConfiguration.imports
```

#### 6. 2～3 分钟面试口述

> Spring Boot 启动时由 SpringApplication 创建并刷新 ApplicationContext，`@SpringBootApplication` 同时提供配置、组件扫描和自动配置入口。自动配置不是无条件创建，而是根据 classpath、属性和现有 Bean 评估条件。现代 Boot 自定义自动配置类使用 `@AutoConfiguration`，候选写入 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`；`spring.factories` 是历史与其他扩展机制背景，不能作为当前唯一答案。用户声明自己的 Bean 后，OnMissingBean 配置会退让。

#### 7. 面试官递进追问

- 怎样查看某个自动配置为什么没生效？
- Boot 3 升 Boot 4 需要重点关注哪些兼容边界？

**记忆钩子**：Boot 看依赖和配置猜合理默认；现代候选进 AutoConfiguration.imports，用户自定义就退让。

---

### Q066：怎样设计一个真正可用的 Spring Boot Starter？

**题型**：资深工程师项目题　｜　**频率**：★★★★☆

**为什么现在学**：高薪岗位会追问你是否能把内部组件做成可复用平台能力，而不仅是会引入 starter。

#### 1. 先用人话理解

Starter 通常提供依赖集合、自动配置、配置属性、条件装配、扩展接口和测试支持。自动配置不应扫描用户包或抢占用户自定义 Bean；配置要有独立命名空间和 metadata。

#### 2. 它在 JVM / 框架里怎么运行

1. 定义公共 API 与核心实现模块
2. 定义 `@ConfigurationProperties` 并校验
3. 使用 `@AutoConfiguration` 与条件注解创建默认 Bean
4. 在 AutoConfiguration.imports 注册候选
5. 提供 `@ConditionalOnMissingBean` 扩展点
6. 用 ApplicationContextRunner 测试不同 classpath/属性组合
7. Starter 模块只聚合必要依赖，避免传递无关组件

#### 3. 贯穿项目怎么用

为 Agent 审计、模型客户端或租户上下文提供 `company-agent-spring-boot-starter`，业务只配置属性即可使用，也可覆盖接口实现。

#### 4. 可读但容易出事故的写法

```java
@Configuration
@ComponentScan("com.company.agent") // 扫描过宽，可能把内部和用户 Bean 全带进来
```

#### 5. 更稳妥的企业级写法

```java
@AutoConfiguration
@EnableConfigurationProperties(AgentProperties.class)
@ConditionalOnClass(AgentClient.class)
class AgentAutoConfiguration {
 @Bean @ConditionalOnMissingBean
 AgentClient agentClient(AgentProperties p) { return new DefaultAgentClient(p); }
}
```

#### 6. 2～3 分钟面试口述

> 一个成熟 Starter 不只是依赖集合。它应提供独立的自动配置模块、类型安全配置属性、条件装配、默认实现和用户覆盖扩展点，并用 AutoConfiguration.imports 注册。自动配置不能扫描用户业务包，也不应在用户已声明 Bean 时强行覆盖。配置前缀要属于公司命名空间，生成 metadata 提供 IDE 提示，并用 ApplicationContextRunner 覆盖缺少类、属性开关、用户覆盖等场景。Starter 的目标是低接入成本和可控退让。

#### 7. 面试官递进追问

- Starter 与 autoconfigure 模块为什么常拆开？
- 自动配置顺序怎样声明？

**记忆钩子**：Starter 给依赖入口，AutoConfiguration 给默认能力，条件和退让保证不抢用户控制权。

---

### Q067：Spring MVC 一次请求从网卡到 Controller 再到响应经历什么？

**题型**：Web 核心题　｜　**频率**：★★★★★

**为什么现在学**：请求链是过滤器、拦截器、参数绑定、异常处理和事务入口的共同地图。

#### 1. 先用人话理解

Servlet 容器接收请求，Filter 链先处理；DispatcherServlet 作为前端控制器找 HandlerMapping，HandlerAdapter 调用 Controller，参数解析器完成绑定，返回值处理器序列化响应；异常由 HandlerExceptionResolver/ControllerAdvice 处理。

#### 2. 它在 JVM / 框架里怎么运行

1. 容器解析 HTTP 并进入 FilterChain
2. DispatcherServlet 查 HandlerMapping 得到 HandlerExecutionChain
3. Interceptor preHandle 执行
4. HandlerAdapter 调用参数解析器并执行 Controller
5. 返回值处理器处理 JSON、视图或流
6. 异常解析器统一映射错误
7. Interceptor afterCompletion 与 Filter 返回链执行

#### 3. 贯穿项目怎么用

Trace、租户解析在 Filter；登录权限可在安全过滤链；业务授权可在方法层；Controller 只做协议转换和调用 Application Service。

#### 4. 可读但容易出事故的写法

```java
@RestController class AgentController {
 @PostMapping("/run") public Result run(HttpServletRequest r) {
   // 在 Controller 里解析租户、鉴权、事务、远程调用、落库全部完成
 }
}
```

#### 5. 更稳妥的企业级写法

```java
@PostMapping("/runs")
RunResponse create(@Valid @RequestBody CreateRunRequest request) {
 return appService.create(request);
}
```

#### 6. 2～3 分钟面试口述

> Spring MVC 中，Servlet 容器先让请求通过 Filter 链，DispatcherServlet 再通过 HandlerMapping 找到处理器及拦截器，通过 HandlerAdapter 完成参数解析并调用 Controller。返回结果由 ReturnValueHandler 转成 JSON、视图或流，异常交给 HandlerExceptionResolver 和 `@ControllerAdvice`。Interceptor 在 Controller 前后执行。工程上 Controller 应负责协议适配和校验，不把事务、远程调用和领域逻辑全部堆进去。

#### 7. 面试官递进追问

- HandlerMapping 与 HandlerAdapter 为什么分开？
- 参数校验异常在哪里被处理？

**记忆钩子**：Filter 先过门，Dispatcher 找人，Adapter 会调用，返回值和异常统一收尾。

---

### Q068：Filter、Interceptor、AOP 和 `@ControllerAdvice` 分别放什么逻辑？

**题型**：高频场景题　｜　**频率**：★★★★★

**为什么现在学**：四者都能“拦截”，但层级、可见信息和适用范围不同。

#### 1. 先用人话理解

Filter 属于 Servlet 层，可处理原始请求响应和安全链；Interceptor 属于 Spring MVC Handler 层，能感知 Controller；AOP 作用于 Spring Bean 方法，适合事务、业务指标和方法权限；ControllerAdvice 统一处理 Web 异常、绑定和响应增强。

#### 2. 它在 JVM / 框架里怎么运行

1. 先判断是否需要原始 HTTP/body/跨框架能力
2. 需要 Controller 映射信息时选择 Interceptor
3. 需要任意 Bean 方法切面时选择 AOP
4. 需要统一异常到 HTTP 响应时选择 Advice
5. 避免同一日志在四层重复记录并泄漏敏感数据

#### 3. 贯穿项目怎么用

Trace ID 和请求包装在 Filter；租户 Handler 校验在 Interceptor；工具调用审计在 AOP；领域异常映射在 Advice。

#### 4. 可读但容易出事故的写法

```java
// 用 AOP 强行读取和修改所有 HttpServletResponse，层级混乱
```

#### 5. 更稳妥的企业级写法

```java
// Filter: trace/headers
// Interceptor: handler/tenant
// AOP: service audit/transaction
// Advice: exception -> response
```

#### 6. 2～3 分钟面试口述

> Filter、Interceptor、AOP 和 ControllerAdvice 位于不同层。Filter 最接近 Servlet，适合 Trace、安全头、请求包装；Interceptor 了解 Spring MVC Handler，适合接口级上下文和轻量校验；AOP 代理 Spring Bean 方法，适合事务、审计和业务指标；ControllerAdvice 负责参数绑定异常和领域异常到 HTTP 响应的统一映射。选型关键是逻辑需要什么上下文以及作用范围，避免重复拦截和全量打印敏感 Prompt。

#### 7. 面试官递进追问

- Spring Security 为什么主要建立在 Filter 链？
- AOP 能否拦截 Controller？是否应该？

**记忆钩子**：HTTP 原始层用 Filter，MVC Handler 用 Interceptor，Bean 方法用 AOP，异常响应用 Advice。

---

### Q069：MyBatis 中 `#{}`、`${}`、Mapper 代理和执行链怎么理解？

**题型**：数据访问高频题　｜　**频率**：★★★★★

**为什么现在学**：SQL 注入是必问，资深回答还要说明 Mapper 为什么没有实现类却能执行。

#### 1. 先用人话理解

`#{}` 生成预编译占位符并通过 PreparedStatement 绑定值，安全且可复用；`${}` 是文本替换，只能用于经过白名单验证的表名、列名等 SQL 结构。Mapper 接口由 MyBatis 创建代理，根据方法定位 MappedStatement，再经过 Executor、StatementHandler、ParameterHandler 和 ResultSetHandler。

#### 2. 它在 JVM / 框架里怎么运行

1. Spring 扫描 Mapper 接口并注册 FactoryBean/代理
2. 调用 Mapper 方法进入代理
3. 根据 namespace + method 找 MappedStatement
4. Executor 选择缓存与执行策略
5. StatementHandler 创建并执行 SQL
6. ParameterHandler 绑定参数，ResultSetHandler 映射结果

#### 3. 贯穿项目怎么用

知识库检索的排序列不能直接接收用户 `${sort}`；必须映射到枚举白名单。大列表查询要分页，避免一次装满 JVM 堆。

#### 4. 可读但容易出事故的写法

```java
@Select("select * from chunk order by ${sort}")
List<Chunk> list(String sort);
```

#### 5. 更稳妥的企业级写法

```java
String column = switch (sort) {
 case CREATED_AT -> "created_at";
 case SCORE -> "score";
};
// 值参数始终使用 #{}，结构参数只使用受控白名单
```

#### 6. 2～3 分钟面试口述

> `#{}` 会生成 PreparedStatement 占位符并进行参数绑定，适合绝大多数值参数，能避免 SQL 注入；`${}` 是原样文本拼接，只能在表名、列名等无法参数化的结构位置，并且必须经过枚举白名单。Mapper 接口没有手写实现，是因为 MyBatis 为它生成代理，方法调用根据 namespace 和方法名找到 MappedStatement，再由 Executor、StatementHandler、ParameterHandler 和 ResultSetHandler 完成执行与映射。项目中还要关注分页、批量、N+1 和事务边界。

#### 7. 面试官递进追问

- MyBatis 一级缓存的作用域是什么？
- 批量 insert 为什么不能只在 Java for 循环中单条执行？

**记忆钩子**：`#{}` 绑值，`${}` 拼结构；Mapper 代理把方法映射到 MappedStatement。

---

### Q070：Spring Boot 和 Spring Cloud 分工是什么？服务发现、负载均衡、熔断怎样串起来？

**题型**：微服务高频题　｜　**频率**：★★★★☆

**为什么现在学**：用户项目使用微服务底座，面试不能只报组件名字。

#### 1. 先用人话理解

Spring Boot 解决单个应用的启动、配置和自动装配；Spring Cloud 提供分布式系统模式集成，如配置、发现、负载均衡、网关和熔断。一次调用先从注册/配置信息获得实例，客户端负载均衡选择目标，再由超时、重试、熔断和隔离保护。

#### 2. 它在 JVM / 框架里怎么运行

1. 服务实例注册并定期续约
2. 调用方获取健康实例列表
3. LoadBalancer 按策略选择实例
4. HTTP/RPC 客户端执行带连接/读取超时的调用
5. 瞬时失败按幂等与预算重试
6. 持续失败触发熔断，快速失败或降级
7. 指标恢复后半开探测

#### 3. 贯穿项目怎么用

模型路由服务不可用时，Agent 服务不能无限重试；应设置超时预算、有限重试、熔断、备用模型和明确错误。

#### 4. 可读但容易出事故的写法

```java
while (true) { callRemote(); } // 无限重试形成放大风暴
```

#### 5. 更稳妥的企业级写法

```java
Retry retry = Retry.of("model", limitedPolicy);
CircuitBreaker cb = CircuitBreaker.ofDefaults("model");
// 结合总体 deadline，不对非幂等操作盲重试
```

#### 6. 2～3 分钟面试口述

> Spring Boot 关注单个应用的可运行性和自动配置，Spring Cloud 关注分布式系统的通用模式。服务调用时先通过注册发现获得健康实例，客户端负载均衡选择目标，再由超时、重试、熔断、隔离和网关治理保护链路。重试必须受次数、总时间和幂等性约束，熔断用于持续失败时快速止损，不是替代修复。项目中还要避免所有服务使用同一个线程池和连接池形成级联故障。

#### 7. 面试官递进追问

- 服务降级与熔断有什么区别？
- 重试为什么可能把小故障放大成雪崩？

**记忆钩子**：Boot 管单应用，Cloud 管分布式协作；发现选实例，超时重试有限，持续失败就熔断。

---

### Q071：怎样为 Agent + RAG 请求设计并发、事务和上下文链路？

**题型**：项目深挖面试题　｜　**频率**：★★★★★

**为什么现在学**：这一题把 Java、集合、并发、JVM 和 Spring 串成真实项目经历。

#### 1. 先用人话理解

请求链应区分同步用户主链、可并行 I/O、数据库原子边界和异步事件。租户/Trace 上下文显式传递；检索和画像可并行；模型与工具受 Semaphore/连接池限制；本地事务只保存运行状态与 Outbox；提交后异步发送审计与后续任务。

#### 2. 它在 JVM / 框架里怎么运行

1. Filter 建立 Trace，Interceptor 校验租户
2. Controller 转成不可变 Command
3. Application Service 在短事务中创建 AgentRun 与 Outbox
4. 并行执行 RAG、画像和策略查询，统一 deadline
5. 模型/工具调用使用虚拟线程或隔离执行器，并受下游许可限制
6. 结果以幂等状态机更新
7. Outbox 发布 MQ，消费者用 Inbox/唯一约束幂等
8. 全链路记录队列、池、GC、下游与业务指标

#### 3. 贯穿项目怎么用

失败分类为可重试、永久失败和需要人工审批；取消向下传播中断，但状态恢复依赖数据库而不是仅内存 Future。

#### 4. 可读但容易出事故的写法

```java
@Transactional
public Result run(Request r) {
  var docs = rag.call();
  var answer = model.call(docs); // 长事务 + 远程 I/O
  repo.save(answer);
  return answer;
}
```

#### 5. 更稳妥的企业级写法

```java
@Transactional
RunId createRun(Command c) {
  RunId id = runRepo.insertPending(c);
  outbox.insert(new RunCreated(id));
  return id;
}
// 事务外执行远程步骤，状态机幂等更新
```

#### 6. 2～3 分钟面试口述

> 我会把 Agent 请求拆成协议入口、短本地事务、可并行远程步骤和可靠事件四部分。Filter/Interceptor 建立 Trace 与租户，Controller 转不可变命令；事务内只创建运行记录和 Outbox，不把模型调用放进长事务。RAG、画像等 I/O 用虚拟线程或隔离执行器并行，但受连接池和 Semaphore 限制，所有阶段共享 deadline。结果通过带版本的状态机幂等落库，Outbox 发布 MQ，消费者用唯一键或 Inbox 防重。监控同时覆盖线程/虚拟线程、连接池、队列、GC、下游和业务状态。

#### 7. 面试官递进追问

- 用户取消请求后，已经发出的远程调用怎样处理？
- 为什么仅靠 CompletableFuture 内存状态无法恢复？

**记忆钩子**：入口建上下文，事务只写真相，远程受限并发，状态机幂等，Outbox 保证可恢复。

---

### Q072：如果线上出现“接口变慢、线程暴涨、GC 频繁、事务偶发失效”，你怎样系统排查？

**题型**：资深工程师综合题｜70% 验收题　｜　**频率**：★★★★★

**为什么现在学**：这是本教材终点：不再按 Java、JVM、Spring 分科，而是按真实故障链回答。

#### 1. 先用人话理解

先建立时间线和影响范围，再从入口流量、执行器、锁、内存、GC、数据库事务、Spring 代理与下游依赖逐层验证。每个结论必须有指标、日志、线程栈、JFR、Trace 或复现实验证据。

#### 2. 它在 JVM / 框架里怎么运行

1. 确认版本发布、流量、租户和接口范围
2. 检查平台/虚拟线程数量、线程池队列、拒绝和连接池
3. 连续采集 JFR/线程转储，定位 CPU、锁和阻塞栈
4. 检查分配率、GC 暂停、堆/直接内存与缓存增长
5. 检查事务方法是否经过代理、自调用、异常吞噬和跨线程
6. 检查数据库慢 SQL、锁等待、Redis/MQ/模型延迟
7. 止血：限流、降级、回滚、扩容或关闭问题功能
8. 修复后用故障注入和长稳压测验证，并补告警与运行手册

#### 3. 贯穿项目怎么用

例如一个新 AOP 切面在事务内打印完整 Prompt，引发巨大字符串分配和 I/O；同时异步方法通过 this 调用绕过事务，可能同时表现为 GC 高和数据不一致。

#### 4. 可读但容易出事故的写法

```java
// 看到慢就加线程和堆，可能让数据库与 GC 更糟
maxPoolSize *= 10; Xmx *= 2;
```

#### 5. 更稳妥的企业级写法

```java
// 证据驱动：trace + JFR + GC log + pool/db metrics
// 先限流止血，再修复代理边界、对象分配和容量模型
```

#### 6. 2～3 分钟面试口述

> 综合排障我先做时间线和影响面，不直接猜 JVM 或 Spring。入口看流量和版本，执行层看线程、虚拟线程、池队列和连接池，JFR/线程栈看 CPU、锁与阻塞，GC 日志看分配率、暂停和老年代，事务检查是否经过代理、自调用、异常与跨线程边界，同时对齐数据库、Redis、MQ 和模型 API Trace。先用限流、降级、回滚或功能开关止血，再根据证据修复，并用相同负载、故障注入和长稳测试验证。高薪回答的关键是边界、证据和闭环，而不是罗列工具。

#### 7. 面试官递进追问

- 怎样证明事务失效而不是读到了缓存旧值？
- 修复后你会补哪些 SLI/SLO 和告警？

**记忆钩子**：先时间线，后分层证据；先止血，再修复，最后复现验证和防复发。

---
