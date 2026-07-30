# 第 3 章：集合不是 API 清单，而是数据结构选型

## 本章目标

建立 List、Set、Map、Queue 的用途，再深入 ArrayList、HashMap 与并发集合。学完后能回答高频源码题，也能在项目中正确选型。

## 前置知识

知道对象、equals/hashCode、泛型和不可变 Key。

## 本章学习路径

```text
业务访问方式 → 选择接口 → 理解底层结构 → 分析复杂度 → 并发安全 → 项目验证
```

> 本章不是按旧八股文件的出现顺序排列，而是按“先能运行，再解释原理，最后处理故障”的认知顺序组织。

---

### Q017：List、Set、Map、Queue 应该怎样选？

**题型**：核心面试题　｜　**频率**：★★★★★

**为什么现在学**：集合框架首先是业务语义，不是背实现类。选错接口会让重复、顺序、查找和并发问题从设计阶段就埋下。

#### 1. 先用人话理解

List 表达有顺序、可重复的数据；Set 表达唯一性；Map 表达 Key 到 Value 的映射；Queue/Deque 表达排队、先进先出或双端操作。先根据业务动作选接口，再根据性能与并发选择实现。

#### 2. 它在 JVM / 框架里怎么运行

1. 判断是否需要键值映射、唯一性、顺序或队列语义
2. 估算主要操作是按下标读、按 Key 查、排序、头尾操作还是阻塞等待
3. 判断是否多线程共享
4. 再选择 ArrayList、HashSet、HashMap、TreeMap、BlockingQueue 等实现

#### 3. 贯穿项目怎么用

Agent 执行步骤用 List；已调用工具 ID 去重用 Set；模型客户端路由用 Map；后台待执行任务用 BlockingQueue。

#### 4. 可读但容易出事故的写法

```java
List<String> tools = new ArrayList<>();
if (!tools.contains(id)) tools.add(id); // 用 List 人工去重，O(n)
```

#### 5. 更稳妥的企业级写法

```java
Set<String> toolIds = new HashSet<>();
toolIds.add(id);
```

#### 6. 2～3 分钟面试口述

> 集合选型先看业务语义：需要有序且可重复用 List，需要唯一性用 Set，需要 Key 到 Value 映射用 Map，需要生产消费或排队用 Queue。然后再看访问模式和并发要求，例如随机访问优先 ArrayList，排序映射可用 TreeMap，多线程共享映射用 ConcurrentHashMap，生产消费协作用 BlockingQueue。项目中我不会因为熟悉 HashMap 就到处使用它，而会让接口直接表达业务约束。

#### 7. 面试官递进追问

- 为什么 Map 不属于 Collection 子接口？
- 有序具体可能指插入顺序、访问顺序还是排序顺序？

**记忆钩子**：先选业务语义，再选底层实现。

---

### Q018：ArrayList 的底层结构和扩容过程是什么？

**题型**：核心面试题　｜　**频率**：★★★★★

**为什么现在学**：ArrayList 是最常用集合，扩容、搬移和线程安全问题都从底层数组开始。

#### 1. 先用人话理解

ArrayList 用连续对象引用数组保存元素，`size` 表示实际元素数，数组长度表示容量。按下标访问是 O(1)；尾部追加摊销 O(1)；中间插入删除需要移动元素。容量不足时创建更大数组并复制。

#### 2. 它在 JVM / 框架里怎么运行

1. 首次添加时按实现策略创建初始数组
2. add 先检查 `size + 1` 是否超过容量
3. 不足时计算约 1.5 倍的新容量并处理最小所需容量
4. 使用数组复制迁移旧元素
5. 写入新元素并增加 size

#### 3. 贯穿项目怎么用

一次性加载 5 万条向量元数据时若已知数量，可预设容量减少多次复制；但更应考虑分页和流式处理，避免内存尖峰。

#### 4. 可读但容易出事故的写法

```java
List<Chunk> chunks = new ArrayList<>();
for (Chunk c : loadAll()) chunks.add(c); // 多次扩容且可能一次装满内存
```

#### 5. 更稳妥的企业级写法

```java
List<Chunk> chunks = new ArrayList<>(expectedSize);
// 更大数据仍应分页处理，而不是只调大容量
```

#### 6. 2～3 分钟面试口述

> ArrayList 底层是连续数组，size 与容量是两个概念。按索引读取可以直接定位，复杂度 O(1)；尾部追加在不扩容时也是 O(1)，但容量不足会新建更大数组并复制旧数据，所以是摊销 O(1)；中间插入删除需要搬移后续元素，通常是 O(n)。工程上如果能合理预估小批量数据可指定初始容量，但对大量数据应优先分页或流式处理，不能用预分配掩盖内存问题。

#### 7. 面试官递进追问

- 扩容为什么不是每次只加 1？
- `ensureCapacity` 什么时候有价值？

**记忆钩子**：ArrayList 是可扩容数组：读快，尾加快，中间改要搬家。

---

### Q019：ArrayList 和 LinkedList 应该怎么选？

**题型**：高频对比题　｜　**频率**：★★★★☆

**为什么现在学**：“LinkedList 插入删除一定更快”是典型错误答案。

#### 1. 先用人话理解

LinkedList 每个节点保存前后指针，已拿到节点时插入删除可 O(1)，但按下标找到节点仍需 O(n)。节点分散、额外对象多、CPU 缓存局部性差，因此绝大多数普通 List 场景 ArrayList 更快。双端队列通常优先 ArrayDeque。

#### 2. 它在 JVM / 框架里怎么运行

1. ArrayList 连续存储，随机访问与遍历局部性好
2. LinkedList 节点分散，按位置访问需要遍历
3. 中间插入的总成本包含定位节点与修改链接
4. 头尾队列需求可由 ArrayDeque 更高效表达

#### 3. 贯穿项目怎么用

Agent 对话消息通常追加和遍历，使用 ArrayList；工具调用待处理栈或双端队列使用 ArrayDeque，不默认选 LinkedList。

#### 4. 可读但容易出事故的写法

```java
List<Message> messages = new LinkedList<>();
Message m = messages.get(5000); // O(n)
```

#### 5. 更稳妥的企业级写法

```java
List<Message> messages = new ArrayList<>();
Deque<ToolCall> calls = new ArrayDeque<>();
```

#### 6. 2～3 分钟面试口述

> ArrayList 基于连续数组，随机访问和顺序遍历快，CPU 缓存友好；LinkedList 是双向链表，只有在已经持有目标节点时插入删除才是 O(1)，如果按下标操作，先定位节点仍是 O(n)，而且节点对象和指针带来更高内存成本。所以常规业务 List 默认优先 ArrayList。若需求是高效头尾入队出队，我通常选 ArrayDeque，而不是为了“链表插入快”机械选择 LinkedList。

#### 7. 面试官递进追问

- LinkedList 实现了哪些接口？
- ArrayDeque 为什么不能存 null？

**记忆钩子**：别只看改链接，先算找到节点的成本。

---

### Q020：HashMap 的 put 和 get 完整过程是什么？

**题型**：核心面试题　｜　**频率**：★★★★★

**为什么现在学**：这是集合面试的主线，后面的扩容、树化、线程安全和 ConcurrentHashMap 都建立在这里。

#### 1. 先用人话理解

HashMap 用数组桶定位大范围位置，冲突元素在桶内形成链表或红黑树。Key 的 hashCode 经扰动后与数组长度计算索引；put 找到同 Key 就覆盖，否则追加；get 按同样路径查找。

#### 2. 它在 JVM / 框架里怎么运行

1. 首次 put 触发 table 初始化
2. 计算 Key 哈希值并通过 `(n - 1) & hash` 定位桶
3. 空桶直接放入节点
4. 非空桶比较 hash、引用和 equals；相同 Key 覆盖 Value
5. 冲突严重时链表达到阈值且 table 足够大才树化
6. 插入后 size 超过 threshold 触发扩容

#### 3. 贯穿项目怎么用

租户模型配置使用不可变组合 Key；需要防止某个恶意或质量差的 hashCode 让大量 Key 堆在一个桶中。

#### 4. 可读但容易出事故的写法

```java
class Key { String tenant; public int hashCode(){ return 1; } } // 所有键冲突
```

#### 5. 更稳妥的企业级写法

```java
record ModelKey(String tenantId, String modelName) {}
Map<ModelKey, ModelConfig> configs = new HashMap<>();
```

#### 6. 2～3 分钟面试口述

> HashMap 的核心是数组加桶内冲突结构。put 时先计算 Key 的 hashCode 并做扰动，用 `(n-1)&hash` 定位桶；空桶直接插入，非空桶先比较 hash，再比较引用或 equals，相同 Key 更新 Value，不同 Key 加入链表或树。链表达到树化阈值且数组容量至少达到要求时才转红黑树，否则通常先扩容。get 走同样的定位和比较路径。它的平均复杂度接近 O(1)，但依赖合理哈希分布与稳定 Key。

#### 7. 面试官递进追问

- 为什么树化还要求数组容量至少 64？
- null Key 存在哪里？

**记忆钩子**：数组先定位，桶内再 equals；冲突多才树化。

---

### Q021：HashMap 为什么要求容量是 2 的幂？扩容和负载因子怎么权衡？

**题型**：资深工程师题　｜　**频率**：★★★★☆

**为什么现在学**：面试官不只想听默认值，而是要看你能否把位运算、分布和内存成本串起来。

#### 1. 先用人话理解

容量为 2 的幂时，`hash & (n-1)` 等价于取模但更快，也便于扩容到 2n 后判断元素留在原索引还是移动到 `oldIndex + oldCap`。负载因子 0.75 是时间与空间的折中，不是业务永远最佳值。

#### 2. 它在 JVM / 框架里怎么运行

1. threshold 通常等于 capacity × loadFactor
2. 超过 threshold 后容量翻倍
3. 新索引只取决于 hash 新增的那一位
4. 旧桶可拆成低位链和高位链，无需重新做完整取模
5. 容量太小冲突多，太大浪费数组空间

#### 3. 贯穿项目怎么用

本地小型缓存可预估容量；大型缓存不应靠无限 HashMap，而应使用 Caffeine/Redis 并设置上限和淘汰策略。

#### 4. 可读但容易出事故的写法

```java
Map<String,Object> cache = new HashMap<>(10_000_000); // 盲目预分配巨型数组
```

#### 5. 更稳妥的企业级写法

```java
int capacity = (int) Math.ceil(expected / 0.75);
Map<String,Object> map = new HashMap<>(capacity); // 仅适用于有界小缓存
```

#### 6. 2～3 分钟面试口述

> HashMap 容量使用 2 的幂，首先让索引可以用 `hash & (n-1)` 高效计算；其次容量翻倍时，元素只需根据 hash 新增位判断留在原桶还是移动到原索引加旧容量，扩容迁移更简单。默认负载因子 0.75 是冲突概率和数组空间的经验折中。工程上可根据有界数据量预估初始容量减少扩容，但不能把 HashMap 当无界缓存，否则最终是内存泄漏和 OOM。

#### 7. 面试官递进追问

- 初始容量参数为什么会被调整到 2 的幂？
- 负载因子调到 1 有什么代价？

**记忆钩子**：2 的幂让定位和扩容都简单；0.75 平衡空间与冲突。

---

### Q022：什么对象适合作为 HashMap Key？为什么常用 String？

**题型**：场景面试题　｜　**频率**：★★★★★

**为什么现在学**：这道题把不可变性、equals/hashCode 和哈希表正确性连接起来。

#### 1. 先用人话理解

Key 必须在存入后保持参与 equals/hashCode 的字段稳定，否则对象仍在旧桶里，但新 hash 指向别处，get 会找不到。String 不可变、实现稳定、可读性好，因此常用作 Key；组合业务 Key 可用不可变 record。

#### 2. 它在 JVM / 框架里怎么运行

1. put 时按当时 hash 定位桶
2. 修改 Key 字段后 hashCode 可能变化
3. Map 不会自动迁移该节点
4. 随后 get 使用新 hash 去另一个桶查询失败

#### 3. 贯穿项目怎么用

缓存 Key 应包含真正决定结果的租户、模型、版本等字段，并避免直接拼接易冲突字符串，可使用 record。

#### 4. 可读但容易出事故的写法

```java
class MutableKey { String tenant; }
MutableKey key = new MutableKey();
map.put(key, value);
key.tenant = "other"; // 可能再也取不到
```

#### 5. 更稳妥的企业级写法

```java
record CacheKey(String tenantId, String model, long version) {}
map.put(new CacheKey(t, m, v), value);
```

#### 6. 2～3 分钟面试口述

> 适合作为 HashMap Key 的对象要有稳定、正确且分布合理的 equals/hashCode，最重要的是存入后不能改变参与计算的字段。String 因为不可变、哈希实现稳定、语义清晰而常用作 Key。组合 Key 可以使用 record 或 final 不可变类。可变 Key 存入后若修改，节点仍在旧桶，但查询会按新 hash 去别的桶，导致逻辑上存在却无法找到。

#### 7. 面试官递进追问

- 字符串拼接组合 Key 有哪些隐患？
- hashCode 缓存为什么依赖不可变性？

**记忆钩子**：Key 入桶后不能改门牌。

---

### Q023：ConcurrentHashMap 在现代 JDK 中怎样保证并发安全？

**题型**：并发高频题　｜　**频率**：★★★★★

**为什么现在学**：这是集合与并发章节的桥梁，必须区别 JDK 7 分段锁和 JDK 8+ 当前实现。

#### 1. 先用人话理解

现代 ConcurrentHashMap 使用 volatile 可见性、CAS 和桶级 synchronized 配合。空桶插入常用 CAS；发生冲突时锁定桶头节点处理链表或树；扩容时多个线程可协助迁移。它不是给整个 Map 加一把大锁。

#### 2. 它在 JVM / 框架里怎么运行

1. table 与节点字段通过 volatile/内存语义保证可见性
2. 空桶用 CAS 尝试安装新节点
3. 非空桶对桶头 synchronized，缩小锁粒度
4. 计数使用分散计数思想降低热点
5. 扩容通过 forwarding node 标识并允许协助迁移

#### 3. 贯穿项目怎么用

模型客户端缓存和租户限流状态可用 ConcurrentHashMap，但复合操作仍需 `compute`、`merge` 或额外同步，不能把多个 get/put 当原子事务。

#### 4. 可读但容易出事故的写法

```java
if (!map.containsKey(key)) {
  map.put(key, createClient(key)); // 竞态，可能创建多次
}
```

#### 5. 更稳妥的企业级写法

```java
Client client = map.computeIfAbsent(key, this::createClient);
```

#### 6. 2～3 分钟面试口述

> JDK 8 之后 ConcurrentHashMap 不再使用 Segment 分段锁，而是组合 CAS、volatile 和桶级 synchronized。空桶插入可通过 CAS 完成；桶内已有元素时锁住桶头处理链表或红黑树；扩容时线程还能协助迁移。这样不同桶的操作可以并行。需要强调的是，单个方法线程安全不代表多个方法组合原子，例如 containsKey 再 put 仍有竞态，应该使用 computeIfAbsent、merge 等原子复合 API，或者建立更高层并发协议。

#### 7. 面试官递进追问

- `computeIfAbsent` 的映射函数为什么不应做长时间阻塞？
- ConcurrentHashMap 为什么不允许 null？

**记忆钩子**：空桶 CAS，冲突锁桶；单方法安全不等于组合安全。

---

### Q024：CopyOnWriteArrayList、BlockingQueue、ConcurrentLinkedQueue 应怎样选？

**题型**：项目场景题　｜　**频率**：★★★★☆

**为什么现在学**：并发集合不是“都线程安全所以随便选”，它们优化的读写模型完全不同。

#### 1. 先用人话理解

CopyOnWriteArrayList 写时复制整个数组，读无需加锁，适合元素少、读极多写极少；BlockingQueue 在空或满时阻塞，适合有背压的生产消费；ConcurrentLinkedQueue 是无界非阻塞队列，吞吐高但不会自动限制生产速度。

#### 2. 它在 JVM / 框架里怎么运行

1. 先判断读写比例与数据规模
2. 再判断是否需要容量上限和阻塞背压
3. Copy-on-write 写成本与数组大小成正比
4. 无界队列若消费跟不上会增长到内存耗尽
5. 阻塞队列可配合线程池形成明确容量边界

#### 3. 贯穿项目怎么用

插件监听器列表可用 CopyOnWriteArrayList；后台嵌入任务使用有界 BlockingQueue；高吞吐瞬时事件队列若无界必须配套外部限流和监控。

#### 4. 可读但容易出事故的写法

```java
Queue<Task> queue = new ConcurrentLinkedQueue<>();
// 生产无限快，消费慢时内存持续增长
```

#### 5. 更稳妥的企业级写法

```java
BlockingQueue<Task> queue = new ArrayBlockingQueue<>(10_000);
if (!queue.offer(task, 100, MILLISECONDS)) {
  throw new OverloadException();
}
```

#### 6. 2～3 分钟面试口述

> CopyOnWriteArrayList 通过写时复制换取无锁快照读，适合小规模、读多写极少的监听器或配置列表，不适合高频写。BlockingQueue 提供容量、阻塞和生产消费协调，适合线程池和后台任务背压。ConcurrentLinkedQueue 采用无锁链式结构，适合高并发非阻塞队列，但通常无界，消费不足时可能把压力转成 OOM。选型要看读写比例、容量边界和背压需求，而不只是“是否线程安全”。

#### 7. 面试官递进追问

- CopyOnWrite 迭代器看到的是实时数据吗？
- BlockingQueue 的 put、offer、add 有什么区别？

**记忆钩子**：读多写少用复制；要背压用有界阻塞；无界队列要防 OOM。

---
