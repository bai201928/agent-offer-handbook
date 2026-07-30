# 第 2 课：Q11～Q20——消息为什么会丢、会重、会积压？

> 这一课不要把方案分开背。所有问题都沿同一条链路定位：业务数据库 → Producer → Broker → Consumer → 业务数据库 → Offset。

---

## Q11. `acks`、副本、ISR、`min.insync.replicas` 和幂等 Producer 是什么关系？

**题目定位**：必考题｜中等偏上｜频率 ⭐⭐⭐⭐⭐

### 先听懂：先区分“消息复制了几份”和“什么时候告诉 Producer 成功”

假设 Partition 0 有三个副本：

```text
Broker 1：Leader
Broker 2：Follower
Broker 3：Follower
```

此时涉及四个不同概念：

- **副本数**：总共计划保存几份；
- **ISR**：当前仍跟得上 Leader 的同步副本集合；
- **acks**：Producer 要等到什么程度才认为发送成功；
- **min.insync.replicas**：至少要有多少同步副本，Broker 才允许在 `acks=all` 下确认写入。

### 三种 ACK

#### `acks=0`

Producer 发出后不等待 Broker 确认。吞吐和延迟好，但网络失败、Broker 拒绝等情况难以及时发现。

#### `acks=1`

Leader 接收后确认。假设 Leader 已写入，但 Follower 尚未复制，Leader 立即宕机，最新数据可能丢失。

#### `acks=all`

Leader 等待当前 ISR 满足确认条件后再返回。它不是简单的“等待配置中的所有副本永远都写完”。真正的安全强度还取决于 `min.insync.replicas`。

### 例子

副本数为 3，设置：

```text
acks=all
min.insync.replicas=2
```

当 ISR 中还有 2 个副本时，可以写入；只剩 1 个时拒绝写入。这个设计是用暂时不可用换取更高的数据可靠性。

### 为什么还需要幂等 Producer？

Producer 发送后，Broker 其实写成功了，但 ACK 在网络中丢失。Producer 看到超时会重试，同一批消息可能被写两次。

Kafka 幂等 Producer 使用 Producer ID、Epoch 和序列号识别客户端重试批次，减少 Kafka 日志中的重复。现代 Kafka 在配置不冲突时默认启用幂等能力。

但它不能替代业务 `eventId`：应用重启后重新创建一条“新消息”，或者 MySQL 业务执行两次，Kafka 无法仅凭 Producer 序列号理解这是同一个业务事件。

### 2～3 分钟优秀回答

“副本数表示一个 Partition 计划保存多少份数据；ISR 是当前与 Leader 保持同步资格的副本集合；`acks` 决定 Producer 等待什么级别的 Broker 确认；`min.insync.replicas` 决定 `acks=all` 时至少需要多少同步副本。

例如副本数 3、`acks=all`、`min.insync.replicas=2`，只有至少两个同步副本可用时才确认写入；只剩一个时宁可拒绝写入，从而避免单副本继续承诺可靠性。

网络超时会造成发送结果未知：Broker 可能已经写入，但 Producer 没收到 ACK，重试可能产生重复。Kafka 幂等 Producer 通过 Producer ID 和序列号对客户端重试批次去重。不过它只解决 Kafka 写入边界内的部分重复，业务仍需要稳定 `eventId` 和消费端幂等。”

### 递进追问

1. 副本数为 3，ISR 只有 1，`acks=all` 一定会失败吗？
2. Producer 幂等能否保证 MySQL 只更新一次？

**记忆句：副本是份数，ISR 是合格队员，ACK 是确认门槛，eventId 是业务身份。**

---

## Q12. 如何保证消息尽量不丢失？

**题目定位**：场景必考题｜中等偏上｜频率 ⭐⭐⭐⭐⭐

### 先听懂：不能只回答 Producer、Broker、Consumer 三段

还要补上最前面的业务数据库。

```text
业务事实 → Producer → Broker → Consumer → 下游业务 → Offset
```

#### 第 1 段：业务事实到 Producer

订单数据库已经提交，但应用在调用 Kafka 前宕机，消息根本没有产生。

这个问题不是 `acks=all` 能解决的，因为消息还没到 Kafka。常用方案是 Outbox：业务表和待发送事件在一个 MySQL 本地事务中写入。

#### 第 2 段：Producer 到 Broker

- 检查发送回调和异常；
- 合理重试；
- 使用稳定 eventId；
- `acks=all`；
- 幂等 Producer。

#### 第 3 段：Broker 存储

- 多副本；
- 合理 `min.insync.replicas`；
- 禁止不安全 Leader 选举；
- 副本跨 Broker、跨机架；
- 监控 ISR、磁盘和副本不足。

#### 第 4 段：Consumer 到业务

先完成业务事务，再提交 Offset。这样更偏向 At-least-once：宁可重复，不轻易跳过。

#### 第 5 段：对账与人工恢复

即使配置完善，仍可能因为误删 Topic、保留期过短、错误重置 Offset、程序吞异常而造成业务缺口。核心链路需要对账和回放能力。

### 2～3 分钟优秀回答

“我会沿完整链路回答消息不丢，而不是只说 Kafka 配置。第一，业务数据库到 Producer 使用 Outbox，把订单状态和事件意图放在同一个本地事务中，避免数据库成功但消息没发；第二，Producer 使用稳定 eventId、发送回调、合理重试、`acks=all` 和幂等发送；第三，Broker 配置多副本、合理的 `min.insync.replicas`、跨故障域部署并禁止不安全选主；第四，Consumer 先提交业务事务，再提交 Offset，并通过唯一约束实现幂等。

另外我不会承诺绝对永不丢。错误保留策略、人工误操作和程序跳过异常仍可能造成静默丢失，所以核心系统还要监控 Outbox 超龄、ISR 缩小、Consumer Lag 和业务状态差异，并提供对账和回放工具。”

### 递进追问

1. Outbox Relay 发送成功，但还没标记 SENT 就宕机怎么办？
2. Consumer 捕获异常后仍提交 Offset 会发生什么？

**记忆句：不丢不是一个配置，而是业务意图、发送、存储、消费和对账五段闭环。**

---

## Q13. 为什么会重复消费？如何设计幂等？

**题目定位**：场景必考题｜中等偏上｜频率 ⭐⭐⭐⭐⭐

### 先听懂：重复不是 MQ 的罕见 Bug，而是可靠重试的正常结果

最典型窗口：

```text
1. Consumer 更新积分成功
2. Consumer 尚未提交 Offset
3. 应用突然宕机
4. 重启后再次读取同一消息
```

其他重复来源：

- Producer 超时后重试；
- Outbox 发送成功但状态未更新，再次发送；
- Rebalance 前后的进度窗口；
- 人工重置 Offset 回放；
- 消息从重试 Topic 返回主流程。

### 幂等是什么？

同一个业务操作执行一次和执行多次，最终结果相同。

错误示例：同一支付消息来两次，给用户增加两次积分。

正确目标：消息可重复到达，但积分流水只能创建一次。

### 四类常用幂等方案

#### 1. 插入型：数据库唯一约束

```sql
CREATE UNIQUE INDEX uk_points_event
ON points_ledger(event_id);
```

重复插入发生唯一键冲突，表示该事件已处理。

#### 2. 状态推进：条件更新或版本号

```sql
UPDATE orders
SET status = 'PAID', version = version + 1
WHERE order_id = ?
  AND status = 'UNPAID'
  AND version = ?;
```

#### 3. 金额操作：业务流水唯一键

每次扣款必须写唯一流水，再根据流水更新账户，不能只做 `balance = balance - 100`。

#### 4. 外部调用：operationId

调用支付、MCP 工具或第三方 API 时传稳定 operationId，并支持查询结果。否则本地去重成功但网络重试仍可能让外部执行两次。

### 为什么“先查 Redis，再处理”不够？

```java
if (!redis.hasKey(eventId)) {
    updateDatabase();
    redis.set(eventId, "done");
}
```

两台 Consumer 可能同时通过检查；数据库成功后 Redis 写失败也会产生不一致。Redis 可以加速，但关键事实应由数据库原子约束裁决。

### 2～3 分钟优秀回答

“重复消费通常来自 At-least-once 的失败窗口。例如消费者已经提交数据库事务，但在提交 Offset 前宕机，重启后会再次读取；Producer 超时重试、Outbox 重发和人工回放也会产生重复。

解决核心不是阻止所有重复，而是让业务幂等。插入业务优先使用数据库唯一索引；状态变更使用合法状态条件和版本号；金额操作使用业务流水唯一键；外部调用传稳定 operationId 并支持结果查询。

我通常在一个数据库事务中先插入 Inbox 或业务流水，再更新业务。唯一键冲突表示已经处理，直接返回成功。单纯‘先查 Redis 再写数据库’存在并发窗口、过期和双写问题，不能作为关键业务的最终裁决。”

### 递进追问

1. 分布式锁为什么不是幂等的首选？
2. 相同 eventId 携带了不同 Payload，应该怎么办？

**记忆句：允许消息重复，用稳定身份和数据库原子约束保证业务只生效一次。**

---

## Q14. 自动提交和手动提交 Offset 有什么区别？什么时候会丢、什么时候会重？

**题目定位**：高频题｜中等偏上｜频率 ⭐⭐⭐⭐⭐

### 先听懂：Offset 是进度，不是业务事务

消费一条消息至少有两件事：

```text
A. 业务数据库更新
B. 消费进度提交
```

两件事无法天然成为同一个原子事务。

### 先提交 Offset，再处理业务

```text
提交 Offset → 更新数据库
```

如果更新数据库时宕机，Kafka 认为进度已经前移，消息可能不会再次投递，形成业务丢失。

### 先处理业务，再提交 Offset

```text
更新数据库 → 提交 Offset
```

如果数据库成功后、Offset 提交前宕机，消息会再次投递，形成重复，但可用幂等处理。

### 自动提交为什么危险？

自动提交通常按时间周期推进 Consumer 已拉取到的位置，它不知道你的业务线程是否真的完成。如果 Poll 后把消息扔进线程池，然后自动提交，线程池中的任务还没执行就可能被视为已消费。

### 批量消费的特殊问题

一批 Offset 100～109，若 105 失败但 109 已提交，下次可能从 110 开始，105 被跳过。因此要设计整批重试、逐条幂等、失败记录或受控暂停。

### 2～3 分钟优秀回答

“Offset 表示 Consumer Group 的消费进度，它和业务数据库事务是两件事。先提交 Offset 再处理业务，处理失败时可能丢消息；先处理业务再提交 Offset，业务成功后进程崩溃会重复消费。因此关键业务通常关闭自动提交，采用先业务、后 Offset 的 At-least-once 路线，再通过幂等承受重复。

自动提交的风险是它只知道 Poll 的位置，不知道异步业务线程是否完成。尤其是 Poll 后立即扔进线程池，如果 Offset 已推进而线程池任务失败，就可能跳过消息。

批量消费还要避免中间一条失败却提交到批次末尾。我会让业务处理具备逐条幂等，并根据框架能力选择整批失败、记录失败项进入重试流程，或暂停该分区，保证进度不会越过未处理消息。”

### 递进追问

1. `commitSync` 和 `commitAsync` 如何选择？
2. 一批 500 条中第 100 条失败，如何避免后面 400 条全部阻塞？

**记忆句：Offset 是书签；先夹书签可能漏看，后夹书签可能重看。**

---

## Q15. At-most-once、At-least-once、Exactly-once 分别是什么？

**题目定位**：资深题｜中等偏上｜频率 ⭐⭐⭐⭐

### 先听懂：先看处理和进度的先后顺序

#### At-most-once：最多一次

倾向先推进进度，再处理业务。可能不重复，但失败时可能丢。

适合允许少量丢失、重复代价极高且可接受不完整的数据，例如部分非关键指标。

#### At-least-once：至少一次

先处理业务，再推进进度。尽量不丢，但可能重复。大多数核心业务选择这条路线，再用幂等保证业务效果。

#### Exactly-once：精确一次

必须先说明范围。

Kafka 幂等 Producer 可以避免特定发送重试在 Kafka 日志中产生重复；Kafka Transaction 可以把多个 Kafka 写入和消费 Offset 放进同一个 Kafka 事务，配合 `read_committed` 实现 Kafka-to-Kafka 处理边界内的一次语义。

但如果流程还写 MySQL、Milvus、调用 HTTP 支付或 MCP 工具，Kafka 事务不会自动把这些外部系统纳入原子提交。

### 更实用的说法：Exactly-once business effect

消息可以重复投递，但最终业务效果等同一次：

```text
稳定 eventId
+ 数据库唯一约束
+ 状态版本
+ Outbox / Inbox
+ 外部 operationId
```

### 2～3 分钟优秀回答

“At-most-once 一般先推进进度再处理，可能丢但尽量不重复；At-least-once 先处理再推进进度，尽量不丢但可能重复；Exactly-once 必须明确边界。

Kafka 的幂等 Producer 能处理特定客户端重试重复，Kafka Transaction 能原子提交 Kafka 内部的多分区写入和消费 Offset，消费者设置 `read_committed` 后只读取已提交事务。但它不能自动把 MySQL、Milvus、HTTP 或外部工具纳入同一个事务。

因此跨系统项目更常追求一次业务效果：允许消息重复投递，但通过 eventId、数据库唯一约束、状态版本、Outbox/Inbox 和外部 operationId，让最终业务状态与执行一次相同。回答这题最重要的是先画清事务边界，不能说开启幂等 Producer 就实现全链路 Exactly-once。”

### 递进追问

1. Kafka Transaction 能否同时原子提交 MySQL？
2. `read_committed` 会读到已回滚事务的消息吗？

**记忆句：先圈边界，再谈 Exactly-once；跨系统更关注一次业务效果。**

---

## Q16. Kafka 事务是什么？它适合解决什么问题？

**题目定位**：高频进阶题｜困难｜频率 ⭐⭐⭐⭐

### 先听懂：Kafka 事务主要解决 Kafka 内部“读一批、写一批、提交进度”

流处理场景：

```text
读取 input-topic 的消息
        ↓
计算结果
        ↓
写入 output-topic
        ↓
提交 input-topic 的 Offset
```

如果结果写成功但 Offset 没提交，重试会再次输出；如果 Offset 先提交但结果没写，输入被跳过。

Kafka Transaction 可以把：

- 写入一个或多个 Kafka Partition；
- 本次消费 Offset；

放在同一个 Kafka 事务中提交或回滚。

### 基本角色

Producer 配置稳定的 `transactional.id`，初始化事务，开始事务，发送记录和 Offset，最后 Commit 或 Abort。下游消费者使用 `isolation.level=read_committed`，只读取已经提交的事务消息。

### 它不是什么？

它不是一个通用分布式事务协调器。下面的 MySQL 写入不会自动和 Kafka 事务原子绑定：

```java
mysql.update(...);
kafkaTemplate.send(...);
```

若项目主问题是“MySQL 成功但 Kafka 没发”，通常仍优先考虑 Outbox。

### 2～3 分钟优秀回答

“Kafka 事务主要用于 Kafka 边界内的原子读写。例如消费者读取 input Topic，处理后写 output Topic，同时提交输入 Offset。事务可以让输出记录和 Offset 一起 Commit 或 Abort，配合下游 `read_committed`，避免结果写入与消费进度不一致。

Producer 通过稳定的 `transactional.id` 获得事务身份，执行 begin、send、sendOffsetsToTransaction 和 commit；异常时 abort。Kafka 还通过 Epoch 等机制隔离旧 Producer 实例。

但 Kafka 事务不会自动把 MySQL、Milvus 或 HTTP 调用纳入同一个原子事务。如果业务核心是数据库与 Kafka 双写，我通常使用 Outbox；如果是 Kafka-to-Kafka 流处理，则 Kafka Transaction 更合适。选方案前必须先说明事务边界。”

### 递进追问

1. 为什么 `transactional.id` 需要稳定且实例间不能冲突？
2. 消费者不配置 `read_committed` 会怎样？

**记忆句：Kafka 事务擅长 Kafka 内部读写，不是 MySQL 的万能事务。**

---

## Q17. MySQL 和 Kafka 如何保证最终一致？Outbox 是什么？

**题目定位**：项目深挖题｜困难｜频率 ⭐⭐⭐⭐⭐

### 先听懂：两次写入之间一定有宕机窗口

方案一：先写 MySQL，再发 Kafka。

```text
MySQL 成功
应用宕机
Kafka 没发送
```

方案二：先发 Kafka，再写 MySQL。

```text
Kafka 已可见
MySQL 回滚
下游处理了不存在的业务事实
```

### Outbox 的核心思路

把业务表和“我要发送这条消息”的记录写进同一个 MySQL 本地事务：

```text
一个事务
├── UPDATE orders SET status='PAID'
└── INSERT outbox_event(event_id, type, payload, status='NEW')
```

事务提交后，后台 Relay 扫描 Outbox，发送 Kafka，再标记为 SENT。

```text
业务请求
  ↓
MySQL：业务表 + Outbox
  ↓
Relay 重复扫描
  ↓
Kafka
```

### Relay 发送成功、标记 SENT 前宕机怎么办？

它会再次发送，所以 Outbox 保证的是不漏发，不保证绝不重复。Consumer 必须幂等。

### Polling Outbox 和 CDC

- 小中型项目：定时扫描、`SKIP LOCKED`、批量发送，容易理解；
- 大规模项目：通过数据库日志 CDC 捕获 Outbox 变化，降低轮询压力，但基础设施更复杂。

### 企业字段

```text
event_id
aggregate_type
aggregate_id
event_type
payload
status
retry_count
next_retry_at
created_at
sent_at
```

还要监控最老 NEW 事件年龄，而不仅是表里有多少行。

### 2～3 分钟优秀回答

“MySQL 与 Kafka 的核心问题是双写。先写数据库后发消息，中间宕机会漏发；先发消息后写数据库，会让下游看到最终回滚的业务。

Outbox 模式把业务修改和事件意图放在同一个 MySQL 本地事务中。例如订单改为 PAID 的同时插入一条 NEW 状态的 outbox_event。后台 Relay 持续扫描并发送 Kafka，成功后标记 SENT，失败则按退避重试。

Relay 在发送成功但标记前宕机时会重复发送，因此 Outbox 解决的是可靠发布，不消灭重复，消费端仍需 Inbox 或业务唯一约束。小项目可用轮询和 `SKIP LOCKED`，更大规模可用 CDC。运维上监控 Outbox 最老未发送年龄、失败次数和 Kafka 发送结果，并通过对账发现漏发。”

### 递进追问

1. 多个 Relay 实例如何避免同时抢同一行？
2. Outbox 表会不会无限增长，如何归档？

**记忆句：业务事实和发送意图同事务，Relay 可重发，Consumer 必须幂等。**

---

## Q18. Inbox 是什么？如何实现可靠的幂等消费？

**题目定位**：项目深挖题｜困难｜频率 ⭐⭐⭐⭐⭐

### 先听懂：Inbox 是消费端的“已处理事件登记表”

消息到达后，Consumer 在同一个数据库事务中：

1. 尝试插入 `inbox_event(event_id)`；
2. 插入成功，说明第一次处理；
3. 执行业务更新；
4. 事务一起提交；
5. 唯一键冲突，说明已处理，直接返回成功。

```sql
CREATE TABLE inbox_event (
    consumer_name VARCHAR(100) NOT NULL,
    event_id VARCHAR(100) NOT NULL,
    payload_hash VARCHAR(64) NOT NULL,
    processed_at DATETIME NOT NULL,
    PRIMARY KEY (consumer_name, event_id)
);
```

### 为什么 Inbox 和业务更新必须同事务？

如果先写 Inbox 成功，业务更新失败，重试时会被误判成已经处理；如果先更新业务再写 Inbox，中间宕机也会重复。

同一个本地事务使二者一起成功或一起回滚。

### 为什么保存 payload_hash？

防止上游错误地复用同一个 eventId，却发送了不同内容。遇到相同 ID、不同 Hash，不能静默跳过，应告警并隔离。

### Inbox 是否适合所有消息？

高吞吐日志型消息不一定要为每条都写 Inbox，可以通过天然幂等覆盖写、聚合窗口或 Kafka Transaction 处理。关键业务副作用才需要强幂等证据。

### 2～3 分钟优秀回答

“Inbox 是消费端保存已处理事件身份的表。消费者收到消息后，在同一个数据库事务中先尝试插入 `consumerName + eventId` 唯一记录，再更新业务。插入成功表示首次处理；唯一键冲突表示重复消息，直接返回成功。

Inbox 和业务更新必须在同一事务中，否则会出现 Inbox 已记录但业务失败，或业务成功但 Inbox 未记录的窗口。通常还保存 payload hash，发现相同 eventId 携带不同内容时进行告警，而不是误认为正常重复。

Inbox 适合订单、账户、Agent 工具副作用等关键业务；海量日志并不一定逐条落 Inbox，应根据业务幂等特性和成本选择。Offset 仍在业务事务成功后提交。”

### 递进追问

1. Inbox 表数据量很大如何归档？
2. 消费者需要调用外部 API 时，本地 Inbox 能否保证外部只执行一次？

**记忆句：Inbox 记录谁做过，业务事务决定是否真的做成。**

---

## Q19. 消费失败如何重试？重试 Topic、死信队列和毒消息分别是什么？

**题目定位**：线上场景题｜中等偏上｜频率 ⭐⭐⭐⭐⭐

### 先听懂：所有异常都立即重试，会把故障放大

把失败分三类：

#### 1. 短暂故障

数据库连接抖动、下游 503、网络超时。适合有限次数指数退避重试。

#### 2. 业务暂不可处理

依赖数据尚未到达、资源状态不满足。可延时重试，但要检查业务版本。

#### 3. 永久故障或毒消息

Schema 错误、字段缺失、代码 Bug、违反业务不变量。立即重试不会成功，反而持续占用线程和数据库。

### Kafka 常见重试架构

```text
main-topic
  ↓ 处理失败
retry-1m-topic
  ↓ 仍失败
retry-10m-topic
  ↓ 超过次数
DLQ
```

Retry Consumer 到期后再投回处理流程，或者直接执行同一处理器。

### DLQ 是什么？

死信队列保存超过重试次数或无法自动处理的消息。DLQ 不是垃圾桶，必须包含：

- 原 Topic、Partition、Offset；
- eventId 和 Key；
- 异常类型和堆栈摘要；
- 重试次数；
- 首次和最后失败时间；
- 负责人、回放按钮和审计记录。

### 顺序消息的特殊风险

同一订单的某条消息失败，如果跳过它继续处理后续消息，可能破坏状态顺序；如果一直阻塞，又会产生头阻塞。需要业务状态机、有限重试、隔离和人工裁决。

### 2～3 分钟优秀回答

“消费失败要先分类，不能所有异常都立即无限重试。网络超时和数据库抖动属于暂时故障，可以指数退避；依赖状态未就绪可以延时重试；Schema 错误、非法数据或代码 Bug 属于毒消息，应尽快隔离。

Kafka 项目常使用多级 Retry Topic，例如 1 分钟、10 分钟、1 小时，超过次数进入 DLQ。重试消息必须保留原 eventId，消费仍要幂等。DLQ 还要记录原 Topic、Partition、Offset、异常、重试次数和负责人，并提供修复后受控回放。

对于顺序消息，失败处理要更谨慎：跳过会乱序，持续重试会阻塞同 Key 后续消息。通常结合状态机、有限重试、单 Key 隔离和人工补偿，而不是简单吞异常后提交 Offset。”

### 递进追问

1. 为什么重试时不能生成新的 eventId？
2. DLQ 修复后如何安全回放，避免再次冲击系统？

**记忆句：暂时失败退避重试，永久失败隔离进 DLQ，重试身份不能变。**

---

## Q20. Kafka 积压几百万条怎么办？如何计算是否追得回来？

**题目定位**：线上必考题｜困难｜频率 ⭐⭐⭐⭐⭐

### 先听懂：积压就是生产速度长期大于消费速度

```text
生产速率 P
消费速率 C

P > C：Lag 增长
P < C：开始追赶
净追赶速度 = C - P
```

假设：

- 已积压 3,600,000 条；
- 当前生产 4,000 条/秒；
- 扩容后消费 10,000 条/秒。

净追赶速度：

```text
10,000 - 4,000 = 6,000 条/秒
```

理想清空时间：

```text
3,600,000 ÷ 6,000 = 600 秒，约 10 分钟
```

真实还要考虑失败重试、数据库瓶颈和分区倾斜。

### 线上处理步骤

#### 第一步：先止血

- 确认消费者是否全部报错；
- 必要时限流非核心生产者；
- 保护 MySQL、模型服务和第三方 API；
- 暂停会放大故障的无限重试。

#### 第二步：找瓶颈

检查：

- Consumer 是否发生异常或 Rebalance；
- 单条处理时间是否突然增加；
- MySQL 慢 SQL、锁等待、连接池；
- 外部接口超时；
- 某个 Key 是否造成 Partition 热点；
- 消息是否突然变大；
- GC、CPU、磁盘和网络。

#### 第三步：在并行上限内扩容

Consumer 数量超过 Partition 数不会继续提高传统消费组并行度。若 Partition 足够，可以扩实例；若不足，临时新建更多 Partition 的 Topic，通过搬运 Consumer 重新分发，再部署更多消费者。

#### 第四步：优化消费

- 批量读写；
- 合并数据库操作；
- 缓存重复查询；
- 将慢外部调用拆出；
- 控制单条超时；
- 对热点 Key 做业务分片，但不能破坏必须的顺序。

#### 第五步：恢复和复盘

清空 Lag 后还要降低临时资源、恢复原路由、对账、处理 DLQ，并修改容量模型和告警阈值。

### 不能只看消息条数

还要看：

- Consumer Lag；
- 最老消息年龄；
- 生产/消费速率；
- 每个 Partition 的 Lag；
- 预计恢复时间；
- 下游 SLO。

100 万条 1KB 消息和 100 万条 1MB 消息不是同一个问题。

### 2～3 分钟优秀回答

“处理百万级积压，我会先止血、再定位、再扩容，不会直接说增加消费者。首先确认 Consumer 是否异常，并保护 MySQL、模型服务和外部接口，必要时对非核心生产限流。然后看每个 Partition 的 Lag、最老消息年龄、生产和消费速率、单条处理时长、慢 SQL、连接池、外部超时和 Key 倾斜。

扩容前先看 Partition 数，因为传统消费组的有效并行实例通常不超过 Partition 数。如果分区足够就增加 Consumer；若分区不足且积压严重，可以建立更多分区的临时 Topic，用搬运程序快速重新分发，再部署更多消费者。长期则通过批量处理、合并数据库写、隔离慢调用和容量规划优化。

我还会计算净追赶速度：消费速率减生产速率。积压量除以净追赶速度得到理想恢复时间，并持续校验。清空后要对账、处理 DLQ、缩容临时资源和复盘根因。”

### 递进追问

1. Topic 有 4 个 Partition，已经有 4 个 Consumer，再加 20 台为什么没有效果？
2. 是否可以直接把 Offset 重置到最新位置？

**记忆句：先止血，再找瓶颈；扩容看分区；用净追赶速度算恢复时间。**
