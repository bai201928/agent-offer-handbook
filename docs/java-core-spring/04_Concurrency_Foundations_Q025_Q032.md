# 第 4 章：并发第一步：先理解线程为什么会互相看不见

## 本章目标

从平台线程、线程状态和中断进入 JMM，再学习 volatile、synchronized、ReentrantLock、CAS 和 AQS。重点是建立因果链，不背过时的固定锁升级口诀。

## 前置知识

完成集合章节，理解共享对象、可变状态和原子复合操作。

## 本章学习路径

```text
线程与任务 → 状态与协作 → JMM 三问题 → volatile → 互斥锁 → CAS → AQS
```

> 本章不是按旧八股文件的出现顺序排列，而是按“先能运行，再解释原理，最后处理故障”的认知顺序组织。

---

### Q025：Java 线程、操作系统线程和虚拟线程是什么关系？

**题型**：核心面试题　｜　**频率**：★★★★★

**为什么现在学**：现代 Java 已不能只回答“一个 Java 线程对应一个 OS 线程”。

#### 1. 先用人话理解

平台线程通常与操作系统线程一一对应，创建和上下文切换成本较高；虚拟线程由 JDK 调度，大量虚拟线程复用较少平台载体线程，适合阻塞式 I/O 高并发。线程不是“同时执行”的保证，CPU 核数决定真正并行度。

#### 2. 它在 JVM / 框架里怎么运行

1. 平台线程由 OS 调度并拥有固定本地栈等资源
2. 虚拟线程作为 Java Thread 对象，由 JVM 挂载到载体线程执行
3. 遇到可挂起的阻塞 I/O 时，虚拟线程可卸载，载体线程执行其他任务
4. CPU 密集任务仍受核心数限制，虚拟线程不会提高单个计算速度

#### 3. 贯穿项目怎么用

Agent 请求要并发访问向量库、模型和工具，虚拟线程可保持同步代码风格；模型推理和大规模向量计算仍应受 CPU/GPU 资源池限制。

#### 4. 可读但容易出事故的写法

```java
for (int i=0;i<100_000;i++) new Thread(task).start(); // 平台线程耗尽
```

#### 5. 更稳妥的企业级写法

```java
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
  executor.submit(task);
}
```

#### 6. 2～3 分钟面试口述

> Java 平台线程通常一对一映射操作系统线程，创建数量受栈内存和内核调度成本限制。Java 21 正式引入虚拟线程，它仍是 Thread API，但由 JVM 调度，大量虚拟线程复用少量载体平台线程；阻塞 I/O 时可以挂起虚拟线程，把载体让给其他任务。它适合请求级、I/O 密集型高并发，不会让 CPU 密集计算变快，也不能消除数据库连接池、下游限流等资源上限。项目中我会用虚拟线程降低线程管理复杂度，同时保留外部资源隔离与背压。

#### 7. 面试官递进追问

- 虚拟线程为什么不应该再放进固定大小线程池？
- 并发和并行有什么区别？

**记忆钩子**：平台线程贵要复用，虚拟线程便宜按任务创建；CPU 仍只有那么多核。

---

### Q026：线程有哪些状态？interrupt、sleep、wait 应该怎样理解？

**题型**：高频基础题　｜　**频率**：★★★★★

**为什么现在学**：线程停止、超时和优雅关闭是工程常用能力，暴力 stop 已被弃用。

#### 1. 先用人话理解

Java 线程状态包括 NEW、RUNNABLE、BLOCKED、WAITING、TIMED_WAITING、TERMINATED。interrupt 不是强杀线程，而是发送协作式中断信号；阻塞方法可能抛 InterruptedException 并清除标志，业务代码应决定退出或恢复中断。

#### 2. 它在 JVM / 框架里怎么运行

1. start 使 NEW 线程进入可调度状态
2. 竞争 synchronized 监视器失败进入 BLOCKED
3. wait/join 等无超时等待进入 WAITING
4. sleep/带超时等待进入 TIMED_WAITING
5. interrupt 设置标志或唤醒可中断阻塞
6. 线程完成 run 后进入 TERMINATED

#### 3. 贯穿项目怎么用

Agent 任务取消时设置中断；工具调用循环定期检查中断；捕获 InterruptedException 后清理资源并恢复中断或退出。

#### 4. 可读但容易出事故的写法

```java
try { Thread.sleep(1000); }
catch (InterruptedException e) { /* 吞掉中断继续跑 */ }
```

#### 5. 更稳妥的企业级写法

```java
try { Thread.sleep(1000); }
catch (InterruptedException e) {
  Thread.currentThread().interrupt();
  return;
}
```

#### 6. 2～3 分钟面试口述

> Java 线程状态反映的是 JVM 视角：NEW 未启动，RUNNABLE 包含可运行和正在运行，BLOCKED 是等待进入 synchronized，WAITING 和 TIMED_WAITING 是等待通知或时间，结束后是 TERMINATED。interrupt 是协作式取消，不会直接杀死线程；对 sleep、wait、join 等阻塞操作会抛 InterruptedException，很多方法还会清除中断标志。因此捕获后不能无声吞掉，通常要退出任务，或恢复中断让上层感知。stop、suspend、resume 会破坏资源一致性，不应使用。

#### 7. 面试官递进追问

- BLOCKED 与 WAITING 的核心区别是什么？
- 为什么线程池关闭依赖中断协作？

**记忆钩子**：中断是通知，不是枪杀；收到后清理并退出或继续传递。

---

### Q027：JMM 的可见性、原子性、有序性和 happens-before 是什么？

**题型**：并发核心题　｜　**频率**：★★★★★

**为什么现在学**：并发问题不是“线程随机”，而是共享内存、缓存和重排序缺少约束。

#### 1. 先用人话理解

JMM 是 Java 对多线程读写共享变量的规范。可见性指一个线程的修改何时被另一个看到；原子性指操作是否不可分割；有序性指编译器和 CPU 重排后，其他线程能观察到什么顺序。happens-before 提供跨线程可见和顺序保证。

#### 2. 它在 JVM / 框架里怎么运行

1. 线程在寄存器、缓存和主内存之间读写共享数据
2. 编译器和 CPU 可在不破坏单线程语义下重排
3. 普通读写之间没有自动的跨线程时序保证
4. volatile 写先行发生于后续对同变量的读
5. 锁释放先行发生于后续获得同一锁
6. 线程 start、join 等也建立 happens-before

#### 3. 贯穿项目怎么用

模型路由配置热更新需要安全发布；任务状态从 RUNNING 改为 CANCELLED 需要让执行线程可靠看到。

#### 4. 可读但容易出事故的写法

```java
boolean stopped = false;
while (!stopped) { } // 可能永远看不到修改
```

#### 5. 更稳妥的企业级写法

```java
volatile boolean stopped = false;
while (!stopped) { Thread.onSpinWait(); }
```

#### 6. 2～3 分钟面试口述

> JMM 规定线程如何通过共享内存交互，核心问题是可见性、原子性和有序性。可见性保证修改能被其他线程观察，原子性保证复合操作不会被交叉，有序性约束重排序对其他线程的影响。happens-before 是判断并发正确性的规则，例如锁释放先于随后获得同一锁，volatile 写先于后续读，线程 start 前的操作对新线程可见。它不是说物理时间一定先后，而是保证观察结果。写并发代码时应通过这些规则建立明确的发布和同步关系。

#### 7. 面试官递进追问

- `i++` 为什么即使变量 volatile 仍不原子？
- final 字段的安全发布规则是什么？

**记忆钩子**：JMM 管三件事：看得到、拆不开、顺序对。

---

### Q028：volatile 能保证什么，不能保证什么？

**题型**：核心面试题　｜　**频率**：★★★★★

**为什么现在学**：volatile 常被误答成“轻量级锁”，会导致计数和状态机错误。

#### 1. 先用人话理解

volatile 保证对单个变量的可见性，并建立读写的有序关系；它不提供互斥，无法把读-改-写复合操作变成原子。适合状态标志、配置引用和单写多读场景，不适合竞争计数。

#### 2. 它在 JVM / 框架里怎么运行

1. volatile 写把之前操作按 JMM 规则发布
2. 后续 volatile 读能看到该写及之前可见状态
3. 编译器在读写周围加入相应屏障语义
4. `count++` 仍分为读取、计算、写回，线程可交叉

#### 3. 贯穿项目怎么用

热更新不可变 `ModelPolicy` 快照可用 volatile 引用；请求计数用 LongAdder 或 AtomicLong。

#### 4. 可读但容易出事故的写法

```java
volatile int count;
void inc() { count++; } // 仍会丢更新
```

#### 5. 更稳妥的企业级写法

```java
private final LongAdder count = new LongAdder();
void inc() { count.increment(); }
```

#### 6. 2～3 分钟面试口述

> volatile 主要提供两类保证：变量修改对其他线程可见，以及围绕该读写的有序性约束。它不加互斥，所以 `i++`、检查后更新等复合操作仍会产生竞态。适合停止标志、状态位、不可变配置快照引用等场景；计数、余额和多字段一致性需要原子类或锁。volatile 也不是“每次机械地从物理主内存读取”这么简单，准确说法是它在 JMM 中建立可见性和 happens-before 关系。

#### 7. 面试官递进追问

- 单例双重检查为什么需要 volatile？
- volatile 引用能否保证引用对象内部所有后续修改线程安全？

**记忆钩子**：volatile 让人看见，不让人排队。

---

### Q029：synchronized 的底层机制和现代锁状态应该怎样回答？

**题型**：核心高频题　｜　**频率**：★★★★★

**为什么现在学**：旧资料常背“偏向锁→轻量级锁→重量级锁”，但偏向锁自 JDK 15 默认禁用并已退出主线。

#### 1. 先用人话理解

synchronized 以对象监视器语义提供互斥、可重入和内存可见性。无竞争时 JVM 使用快速锁路径；竞争增加时可能膨胀为监视器并让线程阻塞。面试可说明历史偏向锁，但不能把它当 Java 21/25 的固定必经流程。

#### 2. 它在 JVM / 框架里怎么运行

1. 编译器为同步块生成 monitorenter/monitorexit 语义，或方法访问标志
2. 线程尝试获取对象监视器所有权
3. 无竞争时走优化后的轻量快速路径
4. 发生竞争、自旋失败或需要等待时可能进入监视器阻塞
5. 释放锁建立对后续获得同一锁线程的 happens-before

#### 3. 贯穿项目怎么用

更新同一 Agent 会话的关键状态可按会话锁串行，但不能把全租户操作锁在一个全局对象上。

#### 4. 可读但容易出事故的写法

```java
synchronized void updateAllTenants(Task t) { /* 全局大锁 */ }
```

#### 5. 更稳妥的企业级写法

```java
Object lock = lockBySession(t.sessionId());
synchronized (lock) { updateOneSession(t); }
```

#### 6. 2～3 分钟面试口述

> synchronized 基于对象监视器语义，提供互斥、可重入，并在释放与获取之间建立内存可见性。JVM 会对无竞争和短竞争走优化快速路径，竞争严重时可能膨胀为监视器并阻塞线程。旧版常背偏向、轻量、重量三级升级，但偏向锁从 JDK 15 已默认禁用，现代面试应把它作为历史优化而不是固定流程。工程上关键是锁对象和粒度：锁住全局 Service 会让无关租户串行，应按资源范围缩小临界区。

#### 7. 面试官递进追问

- 锁住 static 方法与实例方法分别锁什么？
- synchronized 为什么支持可重入？

**记忆钩子**：synchronized 锁监视器；现代重点是快路径、竞争膨胀和锁粒度。

---

### Q030：synchronized 和 ReentrantLock 应该怎样选？

**题型**：高频对比题　｜　**频率**：★★★★★

**为什么现在学**：不是简单的“一个性能高，一个方便”，现代 JVM 下性能差异不是首要依据。

#### 1. 先用人话理解

两者都提供互斥和可重入。synchronized 语法简单、自动释放、JVM 优化充分；ReentrantLock 提供可中断获取、超时、公平策略和多个 Condition，但必须 finally 解锁。

#### 2. 它在 JVM / 框架里怎么运行

1. synchronized 由 JVM 保证异常路径释放
2. ReentrantLock 基于 AQS 管理等待队列
3. `lockInterruptibly` 支持中断等待
4. `tryLock` 支持超时和失败降级
5. Condition 可建立多个等待集合

#### 3. 贯穿项目怎么用

高频短临界区优先 synchronized；需要超时抢锁、可取消或多个条件队列时用 ReentrantLock。

#### 4. 可读但容易出事故的写法

```java
lock.lock();
update();
lock.unlock(); // update 异常时永不解锁
```

#### 5. 更稳妥的企业级写法

```java
lock.lock();
try { update(); }
finally { lock.unlock(); }
```

#### 6. 2～3 分钟面试口述

> 选择 synchronized 还是 ReentrantLock，我先看能力而不是背性能结论。synchronized 代码简单、自动释放、JVM 优化成熟，普通互斥场景优先使用。ReentrantLock 基于 AQS，支持公平锁、可中断获取、tryLock 超时和多个 Condition，适合需要取消、限时等待或复杂协作的场景。它的风险是必须在 finally 中解锁。两者都能建立内存可见性，不能说 ReentrantLock 一定更快。

#### 7. 面试官递进追问

- 公平锁为什么通常吞吐量更低？
- Condition 与 Object.wait/notify 有什么关系？

**记忆钩子**：普通互斥用 synchronized；要超时、中断、多条件再用 Lock。

---

### Q031：CAS、原子类和 ABA 问题怎么理解？

**题型**：核心面试题　｜　**频率**：★★★★★

**为什么现在学**：CAS 是 Atomic 类、ConcurrentHashMap 和 AQS 的基础，但无锁不代表没有代价。

#### 1. 先用人话理解

CAS 比较内存位置当前值与期望值，相等才原子替换；失败由调用方重试。它避免线程挂起，但高竞争下会大量自旋，且只能直接保证一个原子状态。ABA 指值变回原样却丢失中间变化。

#### 2. 它在 JVM / 框架里怎么运行

1. 读取旧值
2. CPU 原子指令比较内存值与期望值
3. 相等则写入新值，不等返回失败
4. 调用方循环重试或转入其他路径
5. 版本号或标记可识别 ABA

#### 3. 贯穿项目怎么用

简单序号用 AtomicLong；高并发指标用 LongAdder；带版本状态用 AtomicStampedReference 或数据库版本列。

#### 4. 可读但容易出事故的写法

```java
AtomicInteger stock = new AtomicInteger(1);
stock.getAndDecrement(); // 可能减成负数，原子不等于业务正确
```

#### 5. 更稳妥的企业级写法

```java
stock.updateAndGet(v -> v > 0 ? v - 1 : v);
```

#### 6. 2～3 分钟面试口述

> CAS 是比较并交换：只有内存中的当前值等于期望值时才原子写入新值，否则失败重试。Atomic 类在此基础上提供线程安全更新。它避免阻塞切换，但高竞争会消耗 CPU，复杂多变量约束也不适合只靠 CAS。ABA 是值从 A 变 B 又回 A，单纯比较值看不出历史变化，可用版本号、StampedReference 或更高层状态机解决。无锁只是不使用互斥锁，不代表没有竞争和重试成本。

#### 7. 面试官递进追问

- AtomicLong 与 LongAdder 如何选？
- CAS 为什么可能产生饥饿？

**记忆钩子**：CAS 先比较再替换；失败不睡觉而重试，竞争高就很贵。

---

### Q032：AQS 是什么？它怎样支撑锁和同步器？

**题型**：资深工程师高薪题　｜　**频率**：★★★★☆

**为什么现在学**：理解 AQS 后，ReentrantLock、Semaphore、CountDownLatch 不再是互不相关的 API。

#### 1. 先用人话理解

AQS 是一个同步框架骨架：用一个 state 表示同步状态，用 FIFO 风格等待队列管理获取失败的线程，并提供独占/共享两种模式。子类只需定义如何尝试获取和释放。

#### 2. 它在 JVM / 框架里怎么运行

1. 线程调用 acquire，子类 `tryAcquire` 尝试修改 state
2. 失败线程包装为节点加入等待队列
3. 前驱释放后唤醒合适后继
4. 独占模式用于锁，共享模式允许多个线程同时通过
5. ConditionObject 维护条件等待队列，signal 后转回同步队列

#### 3. 贯穿项目怎么用

自定义租户并发闸门通常优先 Semaphore，而不是自己继承 AQS；只有标准同步器无法表达时才考虑自定义。

#### 4. 可读但容易出事故的写法

```java
class BusyLock { volatile boolean locked; } // 没有队列、取消和正确内存语义
```

#### 5. 更稳妥的企业级写法

```java
Semaphore permits = new Semaphore(20);
if (!permits.tryAcquire(100, MILLISECONDS)) throw new OverloadException();
try { callModel(); } finally { permits.release(); }
```

#### 6. 2～3 分钟面试口述

> AQS 即 AbstractQueuedSynchronizer，它把同步器共同机制抽出来：一个 state 表示资源状态，一个等待队列保存获取失败的线程，并处理入队、阻塞、唤醒、取消等复杂逻辑。具体同步器实现 tryAcquire、tryRelease 或共享版本。ReentrantLock 使用独占模式，Semaphore 和 CountDownLatch 使用共享模式。项目开发通常直接使用成熟同步器，不轻易自定义 AQS，因为取消、超时、公平和异常路径都很容易写错。

#### 7. 面试官递进追问

- AQS 队列与 Condition 队列有什么区别？
- state 为什么用 volatile？

**记忆钩子**：AQS 用 state 记资源，用队列排失败线程，子类定义获取释放规则。

---
