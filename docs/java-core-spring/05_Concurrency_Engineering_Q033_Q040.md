# 第 5 章：并发第二步：把线程能力变成可控的生产系统

## 本章目标

学习 ThreadLocal、线程池、CompletableFuture、虚拟线程、并发容器、死锁与协调器。重点是容量、取消、上下文传播和故障恢复。

## 前置知识

完成第 4 章，理解 JMM、锁、CAS 与 AQS。

## 本章学习路径

```text
上下文隔离 → 线程复用 → 任务编排 → 虚拟线程 → 背压与共享状态 → 死锁 → 协调工具
```

> 本章不是按旧八股文件的出现顺序排列，而是按“先能运行，再解释原理，最后处理故障”的认知顺序组织。

---

### Q033：ThreadLocal 的底层原理、内存泄漏和使用边界是什么？

**题型**：核心面试题　｜　**频率**：★★★★★

**为什么现在学**：租户、Trace、事务和安全上下文经常依赖 ThreadLocal；在线程池和虚拟线程中使用错误会串数据或泄漏。

#### 1. 先用人话理解

ThreadLocal 不是一个全局 Map，而是每个 Thread 内部有自己的 ThreadLocalMap；ThreadLocal 对象作为弱引用 Key，业务值是强引用。Key 被回收后 Value 可能仍留在线程里，线程池线程长期不结束就形成泄漏。必须在 finally 中 remove。

#### 2. 它在 JVM / 框架里怎么运行

1. 调用 set 时访问当前 Thread 的 ThreadLocalMap
2. 以当前 ThreadLocal 实例作为 Key 保存 Value
3. get 只访问当前线程自己的 Map
4. Key 为弱引用，失去外部引用后可能变 null
5. 清理依赖后续访问或显式 remove，长期空闲线程可能一直持有 Value

#### 3. 贯穿项目怎么用

Web 请求进入时放入 tenantId/traceId，完成后清理；异步任务不能假设自动继承上下文，应显式捕获和恢复。

#### 4. 可读但容易出事故的写法

```java
TENANT.set(tenantId);
service.run(); // 异常时未 remove，线程复用后串租户
```

#### 5. 更稳妥的企业级写法

```java
TENANT.set(tenantId);
try { service.run(); }
finally { TENANT.remove(); }
```

#### 6. 2～3 分钟面试口述

> ThreadLocal 为每个线程保存独立变量，底层数据实际位于 Thread 的 ThreadLocalMap 中。Map 的 Key 对 ThreadLocal 是弱引用，但 Value 是强引用；如果 ThreadLocal 对象失去引用，Key 可能被 GC，Value 却仍被长生命周期线程持有，所以在线程池里必须 finally remove。它适合请求上下文，不适合隐藏业务参数。跨线程、CompletableFuture 和虚拟线程场景要显式传播，不能依赖当前线程恰好不变。

#### 7. 面试官递进追问

- InheritableThreadLocal 在线程池中为什么不可靠？
- ScopedValue 与 ThreadLocal 的设计差异是什么？

**记忆钩子**：值跟线程走，线程会复用；set 后必须 finally remove。

---

### Q034：ThreadPoolExecutor 七个核心参数和任务提交流程是什么？

**题型**：并发核心题　｜　**频率**：★★★★★

**为什么现在学**：线程池是大厂 Java 面试必考，也是项目最容易因无界队列或错误拒绝策略出事故的地方。

#### 1. 先用人话理解

线程池用有限线程复用执行任务。提交时不是先扩到 maximumPoolSize：先创建核心线程；核心满后入队；队列满才创建非核心线程；达到最大线程数且队列仍满才拒绝。

#### 2. 它在 JVM / 框架里怎么运行

1. 任务数小于 corePoolSize：创建核心线程
2. 核心线程已满：尝试进入 workQueue
3. 队列已满且线程数小于 maximumPoolSize：创建非核心线程
4. 线程数达到最大且队列满：执行 RejectedExecutionHandler
5. 非核心空闲超过 keepAliveTime 后回收；allowCoreThreadTimeOut 可影响核心线程

#### 3. 贯穿项目怎么用

Embedding 批处理、MQ 消费回调和模型调用要用隔离线程池，避免一个下游拖垮所有任务。

#### 4. 可读但容易出事故的写法

```java
ExecutorService pool = Executors.newFixedThreadPool(100); // 常配无界队列，积压转成内存
```

#### 5. 更稳妥的企业级写法

```java
ThreadPoolExecutor pool = new ThreadPoolExecutor(
  16, 32, 60, SECONDS,
  new ArrayBlockingQueue<>(1000),
  namedFactory("embedding-"),
  new ThreadPoolExecutor.CallerRunsPolicy());
```

#### 6. 2～3 分钟面试口述

> ThreadPoolExecutor 的关键参数包括核心线程数、最大线程数、空闲存活时间、时间单位、工作队列、线程工厂和拒绝策略。任务提交顺序是：先补核心线程，核心满后入队，队列满后才扩到最大线程数，最后拒绝。因此最大线程数在无界队列下通常不会生效。工程上我会使用有界队列、命名线程工厂、明确拒绝策略，并按依赖隔离线程池，同时监控活跃线程、队列深度、拒绝数和任务耗时。

#### 7. 面试官递进追问

- 为什么不建议直接使用 Executors 工厂方法？
- CallerRunsPolicy 既是降速又有什么风险？

**记忆钩子**：先核心，后排队；队满再扩容；都满才拒绝。

---

### Q035：线程池大小、队列和拒绝策略怎样做容量设计？

**题型**：项目深挖题　｜　**频率**：★★★★★

**为什么现在学**：背 `CPU核数+1` 不能解决真实系统，面试官会追问外部连接池、响应时间和突发流量。

#### 1. 先用人话理解

CPU 密集任务线程数接近可用核心数；I/O 密集可更多，但受连接池、下游 QPS、内存和延迟目标约束。队列不是越大越好，大队列让请求等到超时才执行；拒绝策略必须与业务降级相连。

#### 2. 它在 JVM / 框架里怎么运行

1. 测量单任务 CPU 时间、等待时间和目标吞吐
2. 确定下游并发上限，如 JDBC 连接、模型限流和 HTTP 连接池
3. 根据 Little's Law 估算系统中的并发任务数
4. 给队列设置可解释的等待预算
5. 拒绝时返回过载、降级、落 MQ 或调用方执行，不能静默丢弃
6. 压测阶梯流量并观察 p95/p99、队列和拒绝

#### 3. 贯穿项目怎么用

模型 A 限制 50 并发，就算线程池开 500 也只会制造排队和 429；应使用 Semaphore + 有界执行器。

#### 4. 可读但容易出事故的写法

```java
new ThreadPoolExecutor(200, 1000, 60, SECONDS,
new LinkedBlockingQueue<>(), factory, new DiscardPolicy());
```

#### 5. 更稳妥的企业级写法

```java
Semaphore modelPermits = new Semaphore(50);
// 线程池容量、连接池和模型并发限制统一规划
```

#### 6. 2～3 分钟面试口述

> 线程池参数不能只套公式。CPU 密集任务线程数通常接近核心数；I/O 密集任务可以更多，但最终受数据库连接池、HTTP 连接池、下游并发限制和内存约束。队列容量应对应可接受等待时间，而不是越大越安全；过大只会把过载隐藏为高延迟。拒绝策略要有业务语义，例如快速失败、调用方降速、落可靠队列或降级，不能静默丢任务。最终必须通过真实比例压测和运行指标校准。

#### 7. 面试官递进追问

- 怎样用 Little's Law 解释并发数？
- 线程池和数据库连接池谁应该更大？

**记忆钩子**：线程数看资源，队列看等待预算，拒绝要有业务语义。

---

### Q036：CompletableFuture 怎样正确编排异步任务和异常？

**题型**：高频场景题　｜　**频率**：★★★★☆

**为什么现在学**：Agent 天然需要并行检索、模型和工具调用，错误用法会阻塞公共池、丢异常或制造线程切换。

#### 1. 先用人话理解

CompletableFuture 表达异步结果及依赖关系。`thenApply` 转换结果，`thenCompose` 串联另一个异步阶段，`thenCombine` 合并并行结果；`allOf` 只表示全部结束，不直接收集结果。应传入专用 Executor 并统一超时、取消和异常映射。

#### 2. 它在 JVM / 框架里怎么运行

1. 提交独立任务到明确执行器
2. 使用 combine/allOf 建立依赖图
3. 为远程阶段设置超时
4. 用 handle/exceptionally/whenComplete 区分恢复和观察
5. 最终在系统边界 join，并把 CompletionException 解包成领域异常

#### 3. 贯穿项目怎么用

并行获取向量检索和用户画像，再组合成 Prompt；任一关键任务失败则取消其他阶段或使用明确降级。

#### 4. 可读但容易出事故的写法

```java
CompletableFuture.supplyAsync(this::callModel); // 偷用 commonPool
// 没有保存 Future，也没有观察异常
```

#### 5. 更稳妥的企业级写法

```java
CompletableFuture<Result> f = CompletableFuture
 .supplyAsync(this::retrieve, ioExecutor)
 .orTimeout(800, MILLISECONDS)
 .thenCombine(profileFuture, this::buildResult);
```

#### 6. 2～3 分钟面试口述

> CompletableFuture 用来描述异步结果和依赖关系。thenApply 是同步转换，thenCompose 用于扁平化串行异步调用，thenCombine 合并两个并行结果，allOf 等待多个任务。项目中我会显式指定隔离执行器，不把阻塞 I/O 随意放进 commonPool；每个远程阶段设置超时，统一处理 CompletionException，并决定降级、重试或取消。异步不是把代码包进 supplyAsync 就结束，还要管理线程池、上下文、异常和生命周期。

#### 7. 面试官递进追问

- thenApply 与 thenCompose 的区别是什么？
- 任务超时后底层操作一定停止了吗？

**记忆钩子**：异步要管四件事：执行器、依赖、超时、异常。

---

### Q037：虚拟线程适合什么场景？Java 24/25 后还要担心 pinning 吗？

**题型**：现代高薪题　｜　**频率**：★★★★☆

**为什么现在学**：虚拟线程是当前 Java 面试热点，但不能回答成“替代所有线程池”。

#### 1. 先用人话理解

虚拟线程适合大量相互独立、阻塞 I/O 为主的短任务，采用每任务一线程。它不扩大数据库连接、模型配额或 CPU。Java 24 的 JEP 491 消除了 synchronized 阻塞导致的绝大多数载体线程 pinning，但 native/foreign 调用等边界仍需观测。

#### 2. 它在 JVM / 框架里怎么运行

1. 每个请求或工具调用创建虚拟线程
2. 执行 Java 可挂起 I/O 时虚拟线程卸载
3. 载体线程执行其他虚拟线程
4. 外部资源仍需 Semaphore/连接池限流
5. CPU 密集阶段依旧用有限并行度
6. 通过 JFR、线程转储和指标观察阻塞与资源

#### 3. 贯穿项目怎么用

RAG 并行 HTTP/JDBC 请求可使用虚拟线程；批量 Embedding 的 CPU/本地模型计算仍用受限执行器。

#### 4. 可读但容易出事故的写法

```java
try (var e = Executors.newVirtualThreadPerTaskExecutor()) {
  for (;;) e.submit(this::cpuHeavyLoop); // 无限提交 CPU 任务
}
```

#### 5. 更稳妥的企业级写法

```java
Semaphore downstream = new Semaphore(100);
Thread.startVirtualThread(() -> {
  downstream.acquire();
  try { callRemote(); } finally { downstream.release(); }
});
```

#### 6. 2～3 分钟面试口述

> 虚拟线程适合高并发阻塞 I/O，让同步代码保持易读，并减少平台线程资源成本。它不是性能魔法：CPU 密集任务仍受核心数限制，数据库连接池和下游限流也不会扩大，所以必须保留容量控制。Java 24 的 JEP 491 已让 synchronized 中阻塞的虚拟线程在绝大多数情况下可以释放载体线程，旧版“见 synchronized 就 pin”答案已过时，但 native 或 foreign 调用等情况仍要通过 JFR 和压测验证。虚拟线程通常按任务创建，不再池化虚拟线程本身。

#### 7. 面试官递进追问

- 虚拟线程与异步非阻塞框架怎样选择？
- 为什么 ThreadLocal 在百万虚拟线程下要谨慎？

**记忆钩子**：虚拟线程省线程，不省下游资源；I/O 多用，CPU 多仍限并行。

---

### Q038：如何减少共享可变状态，而不是到处加锁？

**题型**：资深工程师题　｜　**频率**：★★★★☆

**为什么现在学**：高薪面试关注设计能力：最好的并发 Bug 往往是通过状态分区和不可变设计消失的。

#### 1. 先用人话理解

锁是在共享可变状态已经存在后的补救。更优先的办法是不可变对象、线程封闭、按 Key 分片、消息传递、原子数据库更新和并发容器的原子 API。

#### 2. 它在 JVM / 框架里怎么运行

1. 识别真正共享的数据
2. 把请求局部状态留在线程栈或上下文对象中
3. 配置用不可变快照原子替换
4. 按 tenantId/sessionId 分区，避免全局热点
5. 跨服务用消息和数据库约束建立状态机
6. 只有必须共享的最小临界区再加锁

#### 3. 贯穿项目怎么用

每个 AgentRun 使用自己的不可变输入和步骤列表；全局仅共享只读配置和有界客户端缓存；会话更新按 sessionId 串行。

#### 4. 可读但容易出事故的写法

```java
synchronized(agentService) { runAnyTenant(request); }
```

#### 5. 更稳妥的企业级写法

```java
AgentRun run = AgentRun.start(request); // 每请求独立状态
sessionExecutor.submit(request.sessionId(), () -> update(run));
```

#### 6. 2～3 分钟面试口述

> 并发安全不应从选择哪把锁开始，而应先减少共享可变状态。我会让请求状态线程封闭，配置使用不可变快照，按租户或会话分片热点，用消息和数据库状态机传递跨服务变化，并使用 ConcurrentHashMap 的 compute 等原子 API。只有剩下无法消除的最小临界区才加锁。这样不仅吞吐更高，也降低死锁、锁顺序和测试复杂度。

#### 7. 面试官递进追问

- Actor/按 Key 串行模型有什么优缺点？
- 不可变对象会不会带来过多复制？

**记忆钩子**：先消共享，再缩临界区，最后才选锁。

---

### Q039：死锁四个条件是什么？线上怎样定位和预防？

**题型**：核心面试题　｜　**频率**：★★★★★

**为什么现在学**：死锁是经典题，但高质量回答必须包含 JVM 工具、数据库锁和预防策略。

#### 1. 先用人话理解

死锁发生在多个执行单元循环等待对方持有的资源。四个必要条件是互斥、占有且等待、不可剥夺、循环等待。破坏任意条件可避免死锁，工程上最常用统一锁顺序、超时获取和减少嵌套锁。

#### 2. 它在 JVM / 框架里怎么运行

1. 线程 A 持有锁 1 等锁 2
2. 线程 B 持有锁 2 等锁 1
3. 双方无法主动剥夺对方资源
4. 形成等待环
5. 使用 `jstack`、`jcmd Thread.print` 或 JFR 查线程与锁
6. 数据库死锁查看引擎日志和事务信息

#### 3. 贯穿项目怎么用

同时更新两个会话或账户时按 ID 排序后加锁；远程调用绝不能放在本地锁内长时间等待。

#### 4. 可读但容易出事故的写法

```java
synchronized (lockA) {
  synchronized (lockB) { update(); }
} // 另一处顺序相反
```

#### 5. 更稳妥的企业级写法

```java
Object first = idA.compareTo(idB) < 0 ? lockA : lockB;
Object second = first == lockA ? lockB : lockA;
synchronized(first) { synchronized(second) { update(); } }
```

#### 6. 2～3 分钟面试口述

> 死锁需要四个条件：资源互斥、线程占有资源同时等待、资源不可剥夺、等待关系形成环。线上首先用 jstack、jcmd Thread.print 或 JFR 看线程栈和锁拥有关系；数据库死锁则看数据库事务与锁日志。预防通常是统一锁顺序、缩小临界区、避免锁内远程调用、使用 tryLock 超时以及降低同时持有多个资源的机会。处理不能只靠重启，还要修复锁协议并增加检测和告警。

#### 7. 面试官递进追问

- 活锁和饥饿与死锁有什么区别？
- 数据库死锁为什么可能被自动回滚一个事务？

**记忆钩子**：四条件成环才死锁；统一顺序、限时等待、锁内不远程。

---

### Q040：CountDownLatch、Semaphore、CyclicBarrier 和 Phaser 怎样选？

**题型**：场景面试题　｜　**频率**：★★★★☆

**为什么现在学**：这些工具都“让线程等”，但等待模型完全不同。

#### 1. 先用人话理解

CountDownLatch 是一次性倒计时，一个或多个等待者等若干任务完成；Semaphore 控制同时进入资源的许可数；CyclicBarrier 让固定数量线程在屏障会合并可复用；Phaser 支持动态注册和多阶段协作。

#### 2. 它在 JVM / 框架里怎么运行

1. 判断是等待完成、限制并发还是阶段会合
2. Latch 计数只能向零减少，不能重置
3. Semaphore acquire/release 管许可，必须 finally 归还
4. Barrier 所有参与者到齐后一起继续
5. Phaser 可动态增减参与者并推进 phase

#### 3. 贯穿项目怎么用

启动时并行加载多个索引后用 CountDownLatch 汇总；模型并发限制用 Semaphore；多阶段批处理可用 Phaser。

#### 4. 可读但容易出事故的写法

```java
CountDownLatch latch = new CountDownLatch(3);
executor.submit(() -> { load(); }); // 忘记 countDown，永久等待
```

#### 5. 更稳妥的企业级写法

```java
executor.submit(() -> {
  try { load(); }
  finally { latch.countDown(); }
});
```

#### 6. 2～3 分钟面试口述

> CountDownLatch 适合一次性等待多个任务完成，计数只能减到零；Semaphore 适合限制并发资源，例如最多 50 个模型调用；CyclicBarrier 让固定数量参与者在某个阶段会合后一起继续，并可重复使用；Phaser 更灵活，支持动态参与者和多阶段。选择时先问业务是“等完成”“限并发”还是“多方会合”。所有许可和计数更新都要放在 finally 等可靠路径中，避免永久阻塞。

#### 7. 面试官递进追问

- Semaphore 公平和非公平模式有什么差异？
- CompletableFuture allOf 能否替代 CountDownLatch？

**记忆钩子**：等完成用 Latch，限并发用 Semaphore，固定会合用 Barrier，多阶段用 Phaser。

---
