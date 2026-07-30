# 第 8 章：Spring 核心：先看容器怎样把普通对象变成企业组件

## 本章目标

从 IOC/DI 开始，沿 BeanDefinition、生命周期、作用域、AOP、事务和循环依赖建立一条完整容器运行链。

## 前置知识

完成反射、注解、代理、类加载和并发章节。

## 本章学习路径

```text
配置元数据 → BeanDefinition → 实例化/注入 → 后处理器 → 代理 → Bean 可用 → 事务调用
```

> 本章不是按旧八股文件的出现顺序排列，而是按“先能运行，再解释原理，最后处理故障”的认知顺序组织。

---

### Q057：怎样用一句完整的话理解 Spring、IOC、DI 和 AOP？

**题型**：Spring 核心题　｜　**频率**：★★★★★

**为什么现在学**：这是 Spring 面试开场题，回答质量决定面试官是否继续深挖容器、代理和事务。

#### 1. 先用人话理解

Spring 的核心不是“少写 new”，而是用容器管理对象定义、创建、依赖、生命周期和扩展；DI 是容器把依赖交给对象的具体方式；AOP 则用代理把事务、日志等横切逻辑放到业务调用周围。

#### 2. 它在 JVM / 框架里怎么运行

1. 配置类、注解或 XML 被解析成 BeanDefinition
2. 容器根据定义创建 Bean 并解析依赖图
3. 构造器或 setter 接收依赖
4. BeanPostProcessor 在初始化前后扩展对象
5. 符合切点的 Bean 可被包装为代理
6. 业务调用经过代理拦截器链执行事务、指标等

#### 3. 贯穿项目怎么用

模型客户端、向量库和工具实现都交由 Spring 管理；切换供应商只替换 Bean；审计和事务由 AOP 统一处理。

#### 4. 可读但容易出事故的写法

```java
class AgentService {
 private final OpenAiClient client = new OpenAiClient(); // 写死创建和配置
}
```

#### 5. 更稳妥的企业级写法

```java
@Service
class AgentService {
 private final ModelClient client;
 AgentService(ModelClient client) { this.client = client; }
}
```

#### 6. 2～3 分钟面试口述

> Spring 可以理解为一个企业级对象容器和扩展平台。IOC 把对象创建、组装和生命周期控制权从业务代码交给容器；DI 是实现 IOC 的主要方式，对象通过构造器或 setter 声明依赖，容器负责提供；AOP 则用代理和拦截器链处理事务、日志、权限等横切逻辑。这样业务类只关注业务合同，基础设施可替换、可测试，并能在统一生命周期中被增强。Spring MVC、事务和 Spring Boot 自动配置都建立在这个容器基础上。

#### 7. 面试官递进追问

- IOC 与依赖倒置原则有什么联系和区别？
- Spring 是轻量级框架里的“轻量”指什么？

**记忆钩子**：IOC 管对象，DI 送依赖，AOP 围方法加通用能力。

---

### Q058：BeanDefinition、BeanFactory 和 ApplicationContext 分别是什么？

**题型**：Spring 底层高频题　｜　**频率**：★★★★★

**为什么现在学**：不了解内部对象模型，就无法真正解释 Spring 启动和 Bean 生命周期。

#### 1. 先用人话理解

BeanDefinition 是 Bean 的配方，不是 Bean 实例；BeanFactory 是创建和获取 Bean 的核心容器接口；ApplicationContext 在其基础上提供事件、资源、国际化、自动注册后处理器等企业能力。

#### 2. 它在 JVM / 框架里怎么运行

1. 扫描或配置解析器注册 BeanDefinition
2. BeanFactory 保存定义与单例对象
3. ApplicationContext refresh 准备环境和工厂
4. 注册 BeanFactoryPostProcessor 与 BeanPostProcessor
5. 实例化非懒加载单例
6. 发布上下文刷新完成事件

#### 3. 贯穿项目怎么用

自定义工具插件可通过 BeanDefinitionRegistryPostProcessor 注册定义，但普通业务不要在运行期随意改容器。

#### 4. 可读但容易出事故的写法

```java
@Component
class ToolRegistry {
  ToolRegistry(ApplicationContext c) { c.getBeanFactory(); /* 到处操纵容器 */ }
}
```

#### 5. 更稳妥的企业级写法

```java
@Configuration
class ToolConfig {
 @Bean ToolRegistry toolRegistry(List<Tool> tools) { return new ToolRegistry(tools); }
}
```

#### 6. 2～3 分钟面试口述

> BeanDefinition 是 Spring 对一个 Bean 的内部配方，记录类型、作用域、构造参数、依赖、初始化方法等；BeanFactory 根据配方创建和缓存 Bean；ApplicationContext 是更完整的容器，增加事件、资源、国际化以及自动发现后处理器等能力。启动时配置和扫描先变成 BeanDefinition，然后 refresh 组织后处理器、创建单例并完成扩展。业务代码应通过依赖注入获得组件，而不是到处拿 ApplicationContext 做服务定位。

#### 7. 面试官递进追问

- BeanFactoryPostProcessor 和 BeanPostProcessor 的处理对象有什么区别？
- `getBean` 一定返回原始对象吗？

**记忆钩子**：BeanDefinition 是配方，BeanFactory 是工厂，ApplicationContext 是完整企业容器。

---

### Q059：构造器、setter 和字段注入怎么选？

**题型**：高频工程题　｜　**频率**：★★★★★

**为什么现在学**：这不只是编码风格，影响不可变性、测试、循环依赖和 Bean 完整状态。

#### 1. 先用人话理解

必需依赖优先构造器注入：对象创建后立即完整、字段可 final、单元测试直接 new。setter 适合真正可选或可重新配置依赖。字段注入简短但隐藏依赖、难以脱离容器测试，也容易形成职责过多。

#### 2. 它在 JVM / 框架里怎么运行

1. 容器选择构造器并解析参数 Bean
2. 先完整创建依赖，再调用目标构造器
3. setter 注入在实例化后进行属性填充
4. 字段注入通过反射写字段
5. 构造器循环依赖无法靠提前暴露半成品解决

#### 3. 贯穿项目怎么用

AgentService 的 ModelClient、Repository 都是必需依赖，用构造器；可选指标监听器可通过列表或 ObjectProvider 注入。

#### 4. 可读但容易出事故的写法

```java
@Autowired private ModelClient client;
@Autowired private Repository repository;
```

#### 5. 更稳妥的企业级写法

```java
private final ModelClient client;
private final Repository repository;
AgentService(ModelClient client, Repository repository) {
 this.client = client; this.repository = repository;
}
```

#### 6. 2～3 分钟面试口述

> Spring 支持构造器、setter 和字段注入。必需依赖我优先构造器注入，因为依赖显式、字段可 final、对象创建后就是完整状态，也便于不启动 Spring 的单元测试。setter 更适合真正可选或需要重新配置的依赖。字段注入虽然代码短，但隐藏依赖、难测试，也会掩盖类职责过多。构造器参数太多通常是重构信号，而不是改回字段注入的理由。

#### 7. 面试官递进追问

- 单构造器为什么可以不写 `@Autowired`？
- ObjectProvider 适合解决什么问题？

**记忆钩子**：必需依赖进构造器，可选依赖用配置入口，字段注入少用。

---

### Q060：Bean 从定义到销毁经历哪些生命周期步骤？

**题型**：Spring 核心题　｜　**频率**：★★★★★

**为什么现在学**：生命周期是循环依赖、AOP、事务和自定义扩展点的公共主线。

#### 1. 先用人话理解

Spring 先实例化 Bean，再填充依赖，调用 Aware 回调和前置后处理器，执行初始化方法，随后后处理器可能返回代理对象。容器关闭时对受管理单例调用销毁回调。prototype 的销毁通常不由容器完整管理。

#### 2. 它在 JVM / 框架里怎么运行

1. 实例化：构造器或工厂方法创建对象
2. 属性填充：解析并注入依赖
3. 执行 BeanNameAware、BeanFactoryAware 等回调
4. BeanPostProcessor beforeInitialization
5. `@PostConstruct`、InitializingBean、initMethod
6. BeanPostProcessor afterInitialization，AOP 代理常在此阶段形成
7. 单例注册并对外使用
8. 关闭时执行 `@PreDestroy`、DisposableBean、destroyMethod

#### 3. 贯穿项目怎么用

模型客户端在初始化阶段校验配置，不应在构造器里发长时间远程请求；销毁阶段关闭 HTTP 连接和线程池。

#### 4. 可读但容易出事故的写法

```java
@Component
class Client {
 Client() { remoteHealthCheck(); } // 构造器阻塞启动且难重试
}
```

#### 5. 更稳妥的企业级写法

```java
@PostConstruct void validateConfig() { requireNonNull(apiKey); }
@PreDestroy void close() { httpClient.close(); }
```

#### 6. 2～3 分钟面试口述

> Bean 生命周期主线是：读取定义、实例化、依赖填充、Aware 回调、初始化前后处理、初始化方法、后处理器返回最终对象，最后容器关闭时销毁。AOP 代理通常由 BeanPostProcessor 在初始化后包装，因此容器里拿到的可能是代理而不是原对象。生命周期扩展要放对位置：构造器保持轻量，初始化做本地校验，应用就绪后再做需要完整上下文的任务，销毁时释放连接和线程。

#### 7. 面试官递进追问

- `@PostConstruct` 与 ApplicationRunner 适用场景有什么区别？
- 为什么 BeanPostProcessor 自身要较早创建？

**记忆钩子**：先造对象，再注依赖，再初始化和代理，最后关闭释放。

---

### Q061：Spring 单例 Bean 是不是线程安全？作用域怎么选？

**题型**：高频陷阱题　｜　**频率**：★★★★★

**为什么现在学**：Spring 默认单例只保证实例数量，不保证业务状态线程安全。

#### 1. 先用人话理解

singleton 表示每个 ApplicationContext/Bean 名通常一份对象；多个请求会并发调用同一实例。无状态 Service 通常安全；把请求字段、可变集合放在单例 Bean 中就会串数据。prototype 每次获取创建新实例，但注入单例时只在注入时解析一次，需代理或 Provider。

#### 2. 它在 JVM / 框架里怎么运行

1. 容器缓存单例 Bean 引用
2. Web 请求由多个线程并发调用该实例
3. 方法局部变量在各线程栈中隔离
4. 实例可变字段被所有请求共享
5. request/session 作用域由 Web 上下文代理管理

#### 3. 贯穿项目怎么用

AgentService 保持无状态；每次运行状态放 `AgentRunContext` 参数；租户配置缓存使用并发容器和不可变 Value。

#### 4. 可读但容易出事故的写法

```java
@Service class AgentService {
 private String currentTenant; // 并发请求互相覆盖
}
```

#### 5. 更稳妥的企业级写法

```java
@Service class AgentService {
 Result run(String tenantId, AgentRunContext context) { return execute(context); }
}
```

#### 6. 2～3 分钟面试口述

> Spring singleton 只表示容器中通常只有一个实例，不代表自动线程安全。Web 服务中多个线程会同时调用同一个 Service。方法局部变量属于线程栈，一般彼此隔离；实例字段则共享，如果保存当前用户、请求 DTO 或普通 HashMap 就会串数据。企业 Service 应尽量无状态，把请求状态通过参数或独立上下文传递；共享缓存用明确的并发和生命周期策略。prototype、request、session 等作用域要根据创建频率和上下文选择。

#### 7. 面试官递进追问

- 单例 Bean 注入 prototype Bean 会发生什么？
- request scope 代理如何让单例依赖请求对象？

**记忆钩子**：单例只管一份实例，不替你保护实例字段。

---

### Q062：Spring AOP 的 JDK 代理、CGLIB、拦截器链和 self-invocation 怎么理解？

**题型**：Spring 核心题　｜　**频率**：★★★★★

**为什么现在学**：事务失效、日志切面和权限代理都建立在这条调用链上。

#### 1. 先用人话理解

Spring AOP 是运行时代理。JDK 代理基于接口，CGLIB 生成目标类子类；Spring Boot 通常默认使用类代理配置，但核心判断仍要说明版本和配置。外部调用先进入代理，再按切点执行拦截器链；同类 `this.method()` 直接调用原对象，绕过代理。

#### 2. 它在 JVM / 框架里怎么运行

1. 容器找到 Advisor 与目标 Bean
2. 创建 JDK 或 CGLIB 代理
3. 调用方持有代理引用
4. 代理构建 MethodInterceptor 链
5. around/proceed 逐层进入目标方法再逐层返回
6. private/final 方法及自调用存在代理边界

#### 3. 贯穿项目怎么用

审计、指标和事务可使用 AOP；复杂业务流程不要藏在切面里。需要事务边界的方法放到独立协作 Bean。

#### 4. 可读但容易出事故的写法

```java
@Transactional public void outer() { this.save(); }
@Transactional public void save() { repo.insert(); } // 自调用不经过代理
```

#### 5. 更稳妥的企业级写法

```java
@Service class SaveService {
 @Transactional public void save() { repo.insert(); }
}
// outer 通过注入的 SaveService 调用
```

#### 6. 2～3 分钟面试口述

> Spring AOP 通过运行时代理实现。JDK 动态代理要求接口，CGLIB 通过生成子类代理目标类；具体选择受 Spring 与 Boot 配置影响。外部调用代理后，匹配的 Advisor 组成拦截器链，事务、日志等在目标方法前后执行。最常见边界是 self-invocation：同类内部 this 调用没有经过代理，所以事务或切面不会生效。private、final 等方法也受代理方式限制。工程上应把需要独立代理边界的方法拆到协作 Bean，而不是使用奇怪的自代理技巧。

#### 7. 面试官递进追问

- Spring AOP 与 AspectJ 的织入方式有什么区别？
- 多个切面的顺序怎样控制？

**记忆钩子**：外部进代理才有切面；this 直达原对象会绕路失败。

---

### Q063：Spring 声明式事务怎样工作？哪些情况会失效？

**题型**：Spring 必考题　｜　**频率**：★★★★★

**为什么现在学**：数据库一致性是 Java 后端核心，面试官会从 `@Transactional` 追问代理、异常、传播和长事务。

#### 1. 先用人话理解

`@Transactional` 被事务拦截器识别，方法进入前从事务管理器获取/加入事务，成功提交，符合回滚规则的异常则回滚。常见失效包括对象不受 Spring 管理、自调用、方法不可代理、异常被吞、回滚规则不匹配、跨线程和跨数据库误以为同一事务。

#### 2. 它在 JVM / 框架里怎么运行

1. 代理解析事务属性与传播行为
2. 根据数据源获取或加入事务
3. 绑定连接等资源到当前线程上下文
4. 执行业务方法
5. 根据返回/异常决定提交或回滚
6. 清理线程绑定资源

#### 3. 贯穿项目怎么用

创建 AgentRun 与 Outbox 事件应放同一数据库本地事务；模型网络调用不要包在事务里长时间占连接。

#### 4. 可读但容易出事故的写法

```java
@Transactional
public void create() {
 repo.insert();
 callModelFor30Seconds(); // 长事务占连接和锁
 try { outbox.insert(); } catch(Exception e) { log.warn("ignore"); }
}
```

#### 5. 更稳妥的企业级写法

```java
@Transactional
public void createRun(Command c) {
 repo.insert(c.run());
 outbox.insert(c.event());
}
// 事务提交后再异步调用远程模型
```

#### 6. 2～3 分钟面试口述

> Spring 声明式事务本质是 AOP 事务拦截器：进入方法前根据传播属性获取或加入事务，把连接资源绑定到当前线程，方法成功提交，满足回滚规则的异常则回滚。常见失效包括对象不是 Spring Bean、同类自调用绕过代理、异常被 catch 吞掉、默认回滚规则与异常类型不匹配、异步线程离开事务上下文，以及误以为一个本地事务能覆盖多个服务。事务内应只放必要数据库操作，避免远程调用形成长事务。

#### 7. 面试官递进追问

- REQUIRED 与 REQUIRES_NEW 的区别和风险是什么？
- 为什么跨线程后原事务通常不再存在？

**记忆钩子**：事务是代理围方法、资源绑线程；异常别吞，远程别塞长事务。

---

### Q064：Spring 循环依赖和三级缓存为什么存在？现代项目该不该依赖它？

**题型**：高频深挖题　｜　**频率**：★★★★★

**为什么现在学**：早期八股把“三级缓存解决一切”说得过度，当前 Spring Boot 默认还会阻止循环引用。

#### 1. 先用人话理解

构造器循环依赖无法创建任何一方，Spring 会报错。某些单例 setter/字段循环在允许循环引用时，可通过提前暴露对象工厂取得早期引用；三级缓存的关键价值是延迟决定返回原对象还是早期代理。prototype 和复杂代理边界不保证解决。

#### 2. 它在 JVM / 框架里怎么运行

1. A 实例化后尚未填充 B
2. 把能产生 A 早期引用的 ObjectFactory 放入三级缓存
3. 创建 B 时发现依赖 A，调用工厂得到 A 早期引用
4. 必要时 SmartInstantiationAwareBeanPostProcessor 提前生成代理
5. B 完成后注入 A，A 再完成初始化
6. 最终单例进入一级缓存，早期缓存清理

#### 3. 贯穿项目怎么用

循环依赖通常说明服务职责或方向错误。AgentPlanner 与 ToolExecutor 互相注入应提取协调器或事件接口，而不是打开配置硬撑。

#### 4. 可读但容易出事故的写法

```java
@Service class Planner { Planner(Executor e){} }
@Service class Executor { Executor(Planner p){} }
```

#### 5. 更稳妥的企业级写法

```java
@Service class AgentCoordinator {
 AgentCoordinator(Planner planner, ToolExecutor executor) {}
}
// planner 与 executor 不再互相依赖
```

#### 6. 2～3 分钟面试口述

> Spring 的三级缓存只在特定单例、非构造器循环依赖中通过提前暴露引用工作。一级缓存放完整单例，二级缓存放已产生的早期引用，三级缓存放 ObjectFactory，使 Spring 在真正需要时决定返回原对象还是提前创建 AOP 代理。构造器循环没有可暴露实例，因此无法解决；prototype 也不适用。现代 Spring Boot 默认不鼓励循环引用，架构上应通过拆分职责、事件或协调器消除，而不是把三级缓存当设计能力。

#### 7. 面试官递进追问

- 为什么只有二级缓存可能无法正确处理 AOP 早期引用？
- `@Lazy` 解决的是结构问题还是只推迟问题？

**记忆钩子**：三级缓存救特定创建顺序，不救错误架构；构造器环直接失败。

---
