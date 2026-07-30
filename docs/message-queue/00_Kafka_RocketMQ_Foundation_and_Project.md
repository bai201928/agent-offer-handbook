# 第 0 课：从一条订单消息开始认识 Kafka 和 RocketMQ

> 这一课不背面试答案。目标只有一个：让你脑中真正出现一条消息从产生到被处理的完整画面。后面所有可靠性、幂等、顺序、积压和事务问题，都是这条链路某个位置出了问题。

---

## 一、先不要管 Kafka：什么是“发消息”？

假设用户支付订单 `O1001` 后，系统还要完成三件事：

1. 增加积分；
2. 发送短信；
3. 生成审计记录。

最直接的写法是订单服务依次调用三个系统：

```text
订单服务
  ├── 调用积分服务
  ├── 调用短信服务
  └── 调用审计服务
```

问题是，只要短信服务卡住，支付接口就可能一直等待。以后再增加推荐系统，订单服务还要修改代码。

于是我们把“支付成功”写成一份通知：

```json
{
  "eventId": "E9001",
  "orderId": "O1001",
  "eventType": "PAYMENT_CONFIRMED",
  "occurredAt": "2026-07-30T14:00:00+08:00"
}
```

订单服务只负责把这份通知交给一个独立系统。积分、短信、审计服务自己去取。

这个独立系统就是消息队列中间件。

### 用快递理解

- 订单服务：寄件人；
- 消息：包裹；
- Kafka / RocketMQ：快递中转中心；
- 积分、短信、审计服务：不同收件人；
- 消息保存：包裹进入仓库；
- 消息确认：系统记录包裹已经被签收或处理。

### 它和 Java `Queue` 有什么区别？

Java `Queue` 通常只存在一个进程的内存里。应用一重启，内存任务可能就没了；别的服务也不能直接可靠地读取它。

Kafka 和 RocketMQ 是独立部署的分布式系统，能跨进程、跨机器保存和传输消息，并支持持久化、副本、消费进度、失败重试和集群扩容。

---

# 二、一条 Kafka 消息的完整路线

先记住这张最小地图：

```text
订单服务 Producer
        │ 发送 PaymentConfirmed
        ▼
Kafka 集群中的 Broker
        │ 保存到 payment-event Topic
        ▼
Topic 中的某个 Partition
        │ 消息位置是 Offset 128
        ▼
积分 Consumer Group
        │ 某个 Consumer 实例读取
        ▼
更新积分数据库
        │ 业务成功后提交消费进度
        ▼
下次从 Offset 129 继续
```

接下来每次只认识一个新词。

---

## 三、Producer：谁在发送消息？

**一句人话：Producer 就是消息的发送者。**

在订单案例中，订单服务里的 Kafka 客户端就是 Producer。它把 Java 对象序列化成字节，再通过网络发送给 Kafka Broker。

```java
ProducerRecord<String, String> record = new ProducerRecord<>(
        "payment-event",     // Topic
        "O1001",             // Key
        paymentJson           // 消息正文
);
producer.send(record);
```

现在不需要理解 Topic 和 Key，先知道 Producer 在发送时至少要回答两个问题：

1. 这条消息属于什么业务分类？
2. 这条消息最终进入哪个并行队列？

### 常见误区

`send()` 方法返回，不一定代表下游业务已经完成。它最多说明 Kafka 客户端完成了某个发送阶段；真正的确认范围要看 ACK 配置和回调结果。

---

## 四、Broker：消息发到哪台机器？

**一句人话：Broker 是 Kafka 集群中的服务器节点。**

可以把一个 Broker 理解为一个大型仓库节点。一个 Kafka 集群通常由多台 Broker 组成：

```text
Kafka 集群
├── Broker 1
├── Broker 2
└── Broker 3
```

Broker 主要负责：

- 接收 Producer 发来的消息；
- 把消息追加到磁盘日志；
- 把数据复制给其他 Broker 上的副本；
- 接收 Consumer 的拉取请求；
- 返回 Topic、Partition 等元数据信息。

Kafka 4.x 使用 KRaft 管理集群元数据和控制器选举，不再依赖 ZooKeeper。初学阶段只需知道：**Broker 负责数据，Controller 负责管理集群中的重要元数据和 Leader 变化。**

### 生活类比

- Broker：某个城市的快递仓库；
- Kafka 集群：多个城市仓库组成的物流网络；
- Controller：负责仓库和路线调度的控制中心。

---

## 五、Topic：消息属于哪个业务文件夹？

**一句人话：Topic 是消息的逻辑分类。**

电商系统可以这样划分：

```text
order-event       订单状态事件
payment-event     支付事件
inventory-event   库存事件
```

Producer 发送时指定 Topic，Consumer 订阅时也指定 Topic。

### Topic 本身是不是一个队列？

不要把 Topic 想成一根单独的队列。为了并行处理，一个 Topic 还会继续拆成多个 Partition。

```text
payment-event Topic
├── Partition 0
├── Partition 1
└── Partition 2
```

所以：

- Topic 负责“业务分类”；
- Partition 才是 Kafka 真正保存和并行处理消息的单位。

### 常见误区

Topic 不是按“每个 Java 类”创建，也不是微服务有多少个就机械创建多少个。企业项目还要考虑消息语义、保留时间、权限、吞吐量和数据敏感级别。

---

## 六、Partition：为什么一个 Topic 要拆成多个抽屉？

**一句人话：Partition 是 Topic 的物理分片，也是 Kafka 的并行和局部顺序边界。**

把 Topic 想成一个文件柜，Partition 是里面的抽屉：

```text
payment-event
├── Partition 0：消息 A、D、G...
├── Partition 1：消息 B、E、H...
└── Partition 2：消息 C、F、I...
```

每个 Partition 本质上是一条只追加的日志。新消息写到末尾，不会插到中间。

### 为什么需要多个 Partition？

假设一个 Consumer 每秒只能处理 1,000 条消息：

- 只有 1 个 Partition，通常只能让一个消费组成员负责它，并行能力有限；
- 有 6 个 Partition，可以分给最多 6 个活跃 Consumer 实例并行处理；
- 多个 Partition 还可以分布在不同 Broker 上，分散存储和网络压力。

### Partition 带来的代价

Kafka 只能天然保证**同一个 Partition 内的写入顺序**。Partition 0 和 Partition 1 之间没有全局先后关系。

因此 Partition 同时决定三件事：

1. 数据放在哪里；
2. 最多可以有多少并行消费者；
3. 顺序能保证到什么范围。

---

## 七、Offset：消息编号和消费书签

**一句人话：Offset 是消息在某个 Partition 中的位置编号。**

```text
Partition 0
Offset 0  创建订单
Offset 1  支付订单
Offset 2  发货订单
```

每个 Partition 都从自己的 Offset 0 开始。Partition 0 的 Offset 10 和 Partition 1 的 Offset 10 是两条不同消息。

### Offset 有两个容易混淆的含义

#### 1. 消息自己的 Offset

Broker 把消息追加到 Partition 后，为它分配位置。例如：

```text
payment-event / Partition 2 / Offset 128
```

这表示消息在 `payment-event` 的 2 号分区第 128 个位置附近。

#### 2. Consumer Group 提交的 Offset

消费者处理完消息后，会记录“这个消费组下次从哪里继续”。

假设已成功处理 Offset 128，提交的通常是下一个位置 129：

```text
积分消费组 committed offset = 129
```

消费者重启后，从 129 附近继续读取。

### 为什么 Offset 会导致丢失或重复？

把消费过程拆成两步：

```text
步骤 A：更新积分数据库
步骤 B：提交 Offset
```

- 先做 B 再做 A：进度已经前移，但数据库更新失败，消息可能被跳过；
- 先做 A 再做 B：数据库成功后进程崩溃，Offset 没提交，重启后会再次读到同一消息。

因此主流企业系统通常选择第二种：允许重复投递，再让业务处理具备幂等性。

### 重要纠正

提交 Offset 不会立即删除 Kafka 中的消息。Kafka 按保留时间或磁盘策略清理日志，因此可以重置 Offset 后重新消费历史消息。

---

## 八、Key：如何让同一订单进入同一 Partition？

**一句人话：Key 是 Producer 用来决定消息路由的重要业务标识。**

订单 `O1001` 会产生三条事件：

```text
ORDER_CREATED
PAYMENT_CONFIRMED
ORDER_SHIPPED
```

如果三条消息随机进入不同 Partition，就无法依靠 Kafka 保证它们的顺序。

因此把 `orderId` 作为 Key：

```java
new ProducerRecord<>("order-event", orderId, payload);
```

Kafka 默认分区策略会根据 Key 计算目标 Partition。在分区数量不变等前提下，相同 Key 通常进入同一 Partition。

```text
Key=O1001 ──► Partition 1
Key=O2002 ──► Partition 0
Key=O3003 ──► Partition 2
```

这样同一个订单局部有序，不同订单仍可以并行。

### 为什么不能只用 tenantId 做 Key？

如果一个大租户产生了 60% 的消息，而所有消息都使用 tenantId 作为 Key，就可能全部压到一个 Partition，形成热点。

更合理的 Key 通常是：

- 电商：`orderId`；
- Agent：`runId`；
- 文档入库：`documentId`；
- 账户流水：`accountId`，但还要评估大账户热点。

### 扩分区的顺序风险

当 Partition 数量改变后，Key 的哈希取模结果可能改变。同一 Key 的老消息在旧分区，新消息进入新分区，因此不能无条件承诺“扩分区后仍保持全历史顺序”。

---

## 九、Consumer：谁在读取消息？

**一句人话：Consumer 是消息的处理者。**

积分服务里的 Kafka 客户端订阅 `payment-event`，不断向 Broker 请求新消息：

```text
Consumer：有没有新消息？
Broker：有，这是 Partition 1 的 Offset 128～137。
Consumer：我处理完后再提交进度。
```

Kafka Consumer 使用拉取模型。消费者主动 Fetch，Broker 不会不顾消费者能力一直强行推送。

### 拉取有什么好处？

- 快消费者可以多拉、批量处理；
- 慢消费者可以控制节奏；
- 可以重置 Offset，回放历史消息；
- Broker 不需要为每个消费者保存复杂的逐条推送状态。

没有消息时，Consumer 可以使用长轮询等待，不是毫无间隔地空转请求。

---

## 十、Consumer Group：多台消费者如何一起干活？

**一句人话：Consumer Group 是一组共同分摊同一份工作的消费者。**

积分服务部署了三台实例，它们使用相同 `group.id=points-service`：

```text
payment-event：6 个 Partition

points-service 消费组
├── Consumer A：Partition 0、1
├── Consumer B：Partition 2、3
└── Consumer C：Partition 4、5
```

同一个消费组内，一个 Partition 在同一时刻只交给一个组成员处理。一个 Consumer 可以负责多个 Partition。

### 为什么 6 个 Partition 部署 10 个 Consumer 不能获得 10 倍能力？

因为只有 6 个 Partition 可以分配，最多 6 个实例有活干，剩余实例空闲。

```text
并行消费者上限 ≈ Partition 数量
```

但不同消费组可以各自读取同一个 Topic：

```text
payment-event
├── points-service  增加积分
├── sms-service     发送短信
└── audit-service   写审计日志
```

每个消费组都有自己独立的 Offset，不会互相抢走消息。

### 什么是 Rebalance？

当 Consumer 增加、减少、崩溃或订阅关系变化时，Kafka 需要重新分配 Partition，这叫 Rebalance。

Rebalance 期间可能暂停消费；处理时间过长、频繁重启或配置不当会导致反复重平衡。因此消费逻辑不能无限阻塞。

---

## 十一、副本：一台 Broker 坏了，消息怎么办？

**一句人话：Replica 是 Partition 在其他 Broker 上的备份。**

假设 Partition 0 有三个副本：

```text
Partition 0
├── Broker 1：Leader
├── Broker 2：Follower
└── Broker 3：Follower
```

### Leader 做什么？

Producer 和 Consumer 通常与该 Partition 的 Leader 交互。Leader 负责接收写入和提供读取。

### Follower 做什么？

Follower 从 Leader 拉取并复制消息，平时作为备份。Leader 故障后，合格 Follower 可以成为新 Leader。

### ISR 是什么？

ISR 可以先理解成“当前仍然跟得上 Leader、具备同步资格的副本集合”。

```text
ISR = {Broker 1, Broker 2, Broker 3}
```

如果 Broker 3 长时间落后，它会暂时离开 ISR：

```text
ISR = {Broker 1, Broker 2}
```

### 为什么副本数为 3 不等于任何情况下都不丢？

可靠性还取决于：

- Producer 等待什么级别的 ACK；
- `min.insync.replicas` 最少要求多少同步副本；
- 是否允许从严重落后的副本中选 Leader；
- 副本是否跨机器、跨机架部署；
- 是否有人误删 Topic 或错误重置消费进度。

副本是可靠性的基础，不是“绝对不丢”的魔法开关。

---

## 十二、ACK：Producer 收到“成功”到底代表什么？

**一句人话：ACK 是 Broker 对本次发送请求的确认。**

Kafka Producer 常见配置：

- `acks=0`：不等待 Broker 确认，快，但最容易在异常时丢失；
- `acks=1`：Leader 接收后确认，Leader 尚未同步给其他副本就故障时存在风险；
- `acks=all`：等待当前同步副本满足确认要求，可靠性最高，但延迟和不可用概率会增加。

`acks=all` 还要和 `min.insync.replicas` 配合。例如副本数 3、最少同步副本 2：当只剩 1 个同步副本时，宁可拒绝写入，也不降低到单副本确认。

### ACK 不代表什么？

Broker ACK 只说明 Kafka 存储阶段达到配置要求，不代表：

- 积分已经增加；
- 短信已经发送；
- MySQL 本地事务一定和消息同时成功；
- 整条业务链路实现 Exactly-once。

面试时一定要问清“这个成功是哪一段的成功”。

---

## 十三、Kafka 为什么不强调“每条消息同步刷盘”？

RocketMQ 面试中经常讨论同步刷盘和异步刷盘。Kafka 的可靠性主线更强调顺序日志、操作系统 Page Cache、多副本复制和 ACK 条件。

### Page Cache 是什么？

应用写文件时，数据通常先进入操作系统管理的内存页，再由操作系统批量刷到磁盘。这样可以减少频繁磁盘操作。

Kafka 利用顺序写、批量、Page Cache 和副本机制获得高吞吐。企业可靠配置通常依靠多副本确认，而不是要求每条消息都等待单机物理磁盘完成强制刷盘。

### 这是否意味着 Kafka 不可靠？

不是。可靠性来自整体组合：

```text
幂等 Producer
+ acks=all
+ 合理 min.insync.replicas
+ 多副本跨故障域
+ 禁止不安全选主
+ 业务 Outbox / 幂等消费 / 对账
```

只背“写 Page Cache 所以会丢”是不完整的。

---

# 十四、把 Kafka 的概念重新串成一次真实运行

订单 `O1001` 支付成功，完整过程如下：

1. 订单服务在数据库把订单改成 `PAID`；
2. Producer 创建 `PaymentConfirmed` 消息；
3. 指定 Topic 为 `payment-event`；
4. 使用 `orderId=O1001` 作为 Key；
5. Key 被路由到 Partition 1；
6. Partition 1 的 Leader 位于 Broker 2；
7. Leader 追加消息，并让 Follower 同步；
8. 达到 ACK 条件后，Producer 收到成功；
9. 积分消费组中的 Consumer A 负责 Partition 1；
10. Consumer A 拉取到 Offset 128；
11. Consumer A 更新积分数据库；
12. 成功后提交 Offset 129；
13. 短信消费组和审计消费组也分别读取同一条消息。

```text
订单服务
  │ Producer：Topic=payment-event, Key=O1001
  ▼
Broker 2 / Partition 1 Leader / Offset 128
  │ 复制到 Follower
  ▼
points-service Consumer Group
  │ 更新积分成功
  ▼
Committed Offset = 129
```

后面的面试题，本质上都在改动其中某一步：

- 第 1 步成功、第 2 步没执行：数据库和消息双写不一致；
- 第 8 步 ACK 丢失：Producer 不知道是否写入，可能重试；
- 第 11 步成功、第 12 步前宕机：重复消费；
- 同一订单进入不同 Partition：顺序失效；
- 生产速度大于第 11 步处理速度：消息积压。

---

# 十五、理解 RocketMQ：先和 Kafka 对照，再看不同存储方式

学完 Kafka 后，再看 RocketMQ 会简单很多。

## 1. NameServer：路由通讯录

**一句人话：NameServer 主要告诉客户端 Topic 的队列和 Broker 在哪里。**

Broker 定期向 NameServer 上报路由。Producer 查询路由并缓存，然后直接向 Broker 或 5.x 架构中的 Proxy 发送消息。

NameServer 不是保存业务消息正文的仓库，也不是每条消息都必须经过的中转站。

## 2. RocketMQ Broker：真正保存消息

Broker 接收 Producer 消息、保存数据、构建消费索引并向 Consumer 投递。

## 3. Topic：业务分类，但 5.x 还区分消息类型

RocketMQ 5.x 常见 Topic 类型：

- NORMAL：普通消息；
- FIFO：顺序消息；
- DELAY：延时消息；
- TRANSACTION：事务消息。

这和 Kafka Topic 只表达逻辑数据流的使用体验有所不同。

## 4. MessageQueue：类似 Kafka Partition 的并行队列

RocketMQ 一个 Topic 由多个 MessageQueue 组成。它们同样提供：

- 水平分片；
- 并行消费；
- 队列内顺序；
- Offset 定位。

但不要说 MessageQueue 和 Kafka Partition 在底层完全相同。

## 5. CommitLog：消息正文统一写入的大日志

RocketMQ Broker 收到不同 Topic 的消息后，正文主要顺序追加到 CommitLog：

```text
CommitLog
Offset 1000：order-topic 消息
Offset 1200：payment-topic 消息
Offset 1450：agent-topic 消息
```

它像一个统一的大仓库流水账。

## 6. ConsumeQueue：帮助消费者查找正文的目录

如果所有消息都混在 CommitLog，消费者怎么快速找到自己 Topic 和 Queue 的消息？

RocketMQ 会构建 ConsumeQueue，它保存的是较轻量的逻辑索引，例如正文在 CommitLog 的位置和长度。

```text
payment-topic / Queue 1 / ConsumeQueue
├── 指向 CommitLog Offset 1200
├── 指向 CommitLog Offset 1880
└── 指向 CommitLog Offset 2450
```

消费者先沿 ConsumeQueue 找位置，再读取 CommitLog 中的消息正文。

### 一句话记忆

**正文进 CommitLog，消费查 ConsumeQueue，路由问 NameServer。**

---

## 十六、RocketMQ 的 Queue、Offset 和 Consumer Group

RocketMQ Queue 和 Kafka Partition 都可以理解为 Topic 内的有序分片。

```text
payment-topic
├── MessageQueue 0：Offset 0、1、2...
├── MessageQueue 1：Offset 0、1、2...
└── MessageQueue 2：Offset 0、1、2...
```

消费者也通过消费进度知道下一次从哪里继续。不同 Consumer Group 可以独立消费同一 Topic。

区别主要体现在客户端 API、负载均衡、确认、重试和底层存储组织上，而不是“一个有 Offset、另一个没有”。

---

## 十七、RocketMQ 5.x FIFO：Message Group 是什么？

**一句人话：Message Group 表示哪些消息必须按先后顺序处理。**

同一个订单的消息设置同一个 Message Group：

```text
Message Group = O1001
1. ORDER_CREATED
2. PAYMENT_CONFIRMED
3. ORDER_SHIPPED
```

RocketMQ 保证同一 Message Group 内的 FIFO；不同 Message Group 可以并行。

```text
O1001 内部有序
O2002 内部有序
O1001 与 O2002 之间不要求全局有序
```

旧版客户端常用 `MessageQueueSelector` 把相同订单路由到同一 Queue。理解它有帮助，但面向 RocketMQ 5.x，面试回答应优先说明 Message Group 是业务顺序边界。

### 顺序的三个条件

1. Producer 本身要按正确顺序发送；
2. Broker 要按组保存和投递；
3. Consumer 不能把同一组消息再次无序并发处理。

如果 Consumer 收到后立刻扔进普通线程池，仍可能先完成“发货”，后完成“支付”。

---

## 十八、同步刷盘和同步复制不是一回事

RocketMQ 常见两个独立问题：

### 1. 刷盘

- 异步刷盘：数据写入内存映射区域后较快返回，后台批量刷磁盘；
- 同步刷盘：等待数据达到磁盘持久化条件后再返回，可靠性更高但延迟更大。

### 2. 主从复制

- 异步复制：主节点确认后再复制到从节点；
- 同步复制：等待从节点达到要求后再确认。

所以企业面试不能只说“开同步刷盘就绝对不丢”。还要问：

- 是否有副本；
- 复制是否同步；
- 主节点故障后谁接管；
- 生产者重试是否会重复；
- 消费端业务是否幂等。

---

## 十九、半事务消息：先放一个不可见的包裹

**一句人话：半事务消息已经到 Broker，但消费者暂时看不到。**

订单服务要同时完成：

1. MySQL 订单状态变成 `PAID`；
2. RocketMQ 中出现 `PaymentConfirmed`。

如果直接执行两次操作，中间宕机会出现一个成功、一个失败。

RocketMQ 事务消息流程：

```text
1. Producer 发送半事务消息
2. Broker 保存，消息暂不可消费
3. Producer 执行本地 MySQL 事务
4. 本地事务成功：Commit
5. 本地事务失败：Rollback
6. Broker 长时间不知道结果：回查 Producer
```

### 回查根据什么判断？

必须查询可靠的数据库事务记录或订单状态，不能查 JVM 内存变量，因为 Producer 重启后内存会丢失。

### 事务消息没有解决什么？

它主要保证本地事务与消息是否可投递之间的最终一致。它不会自动保证积分 Consumer 一定成功。下游仍需幂等、重试、DLQ 和人工补偿。

---

## 二十、延时消息：到时间后才变得可消费

RocketMQ 5.x 延时消息使用投递时间戳。消息先进入时间相关存储，达到时间后变为可投递。

场景：

- 订单 30 分钟未支付则关闭；
- Agent 人工审批 2 小时未处理则提醒；
- 失败任务 1 分钟后重试。

### 延时到期是否等于业务准时完成？

不是。到期只表示消息可以进入正常投递流程。如果 Broker 或 Consumer 正在积压，实际处理仍会延迟。

### 为什么到期后还要查数据库状态？

订单可能已经支付。正确做法是：

```sql
UPDATE orders
SET status = 'CLOSED'
WHERE order_id = 'O1001'
  AND status = 'UNPAID';
```

延时消息只负责触发，数据库状态机负责最终裁决。

---

# 二十一、Kafka 与 RocketMQ 概念对照表

| 你要解决的问题 | Kafka | RocketMQ 5.x | 初学者记忆 |
| --- | --- | --- | --- |
| 谁保存消息 | Broker | Broker | 仓库节点 |
| 业务分类 | Topic | Topic | 文件夹 |
| 并行分片 | Partition | MessageQueue | 文件柜抽屉 |
| 消息位置 | Offset | Offset | 排队编号 |
| 消费进度 | Committed Offset | Consumer Offset | 书签 |
| 多实例分摊 | Consumer Group | Consumer Group | 一组工人分工 |
| 局部顺序 | 相同 Key 进入同一 Partition | 相同 Message Group FIFO | 同订单进同通道 |
| 数据备份 | Partition Replica | 主从/副本机制 | 仓库备份 |
| 业务延时 | 通常需延时 Topic、调度或生态方案 | 原生 DELAY Topic | 定时包裹 |
| 本地事务衔接 | 常用 Outbox；Kafka 事务主要覆盖 Kafka 边界 | 半事务消息 + 回查 | 先暂存，事务后确认 |
| 存储组织 | 每个 Topic-Partition 独立日志 | CommitLog 正文 + ConsumeQueue 索引 | 多本日志 vs 总账+目录 |

RabbitMQ 在这套教材中只需记住：它以 Exchange、Queue、Routing Key 为核心，路由能力直观，适合传统业务消息和复杂路由；但本教材主项目更重视高吞吐事件流、回放和分区扩展，因此 Kafka 为首选。

---

# 二十二、把概念放进 NexusAgent 企业项目

现在再进入 Agent 项目，术语就不会悬空。

用户提交一个 Agent Run：

```text
runId = R1001
```

系统需要完成规划、工具调用、审计、评估和状态投影。

推荐 Topic：

```text
agent-run-command    Key=runId
agent-run-event      Key=runId
tool-audit-event     Key=operationId
rag-document-event   Key=documentId
evaluation-result    Key=evaluationRunId
```

推荐 Consumer Group：

```text
agent-worker
run-projector
audit-sink
evaluation-aggregator
```

一次 Run 的事件使用 `runId` 作为 Key，尽量进入同一 Partition，保证该 Run 的局部顺序；不同 Run 分散并行。

### 为什么不能请求一到就直接发 Kafka？

假设：

1. MySQL 创建 Run 成功；
2. 应用还没来得及发 Kafka 就宕机。

前端已经拿到 `runId`，但 Worker 永远不知道有任务。

后面教材会引入 Outbox：在同一个 MySQL 事务中同时保存 Run 和待发送事件，再由后台 Relay 重复发布。此处先记住问题，不急着背方案。

---

# 二十三、学完本课必须能画出的总图

```text
业务系统
  │ 创建消息：eventId + key + payload
  ▼
Producer
  │ 指定 Topic，根据 Key 选择 Partition
  ▼
Kafka Broker
  │ Leader 追加日志，Follower 复制
  ▼
Topic / Partition / Offset
  │ Consumer Group 分配 Partition
  ▼
Consumer
  │ 更新 MySQL / Milvus / 外部工具
  ▼
业务成功后提交 Offset
```

RocketMQ 对应：

```text
Producer
  │ 查询 NameServer 路由
  ▼
Broker / Topic / MessageQueue
  │ 正文写 CommitLog，索引写 ConsumeQueue
  ▼
Consumer Group / Consumer Offset
  │ FIFO 用 Message Group
  │ 延时用 Delivery Timestamp
  │ 事务用 Half Message + Commit/Rollback + Check
  ▼
业务数据库
```

---

# 二十四、基础概念自测

不看前文回答：

1. Topic 和 Partition 的区别是什么？
2. Offset 为什么不能作为全局消息 ID？
3. 6 个 Partition 配 10 个同组 Consumer，为什么只有 6 个有活干？
4. 相同订单如何尽量进入同一 Kafka Partition？
5. Broker ACK 为什么不等于积分已经增加？
6. Kafka 提交 Offset 后，消息为什么还能回放？
7. Leader 和 Follower 分别做什么？ISR 又是什么？
8. RocketMQ 的 CommitLog 和 ConsumeQueue 分别保存什么？
9. Message Group 为什么只能保证局部顺序？
10. 半事务消息为什么对消费者暂时不可见？

能用自己的话回答 7 道以上，再进入 Q01～Q10。不能回答时，不要背术语，重新沿订单 `O1001` 的完整路线走一遍。
