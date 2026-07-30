# 第二部分：从一行数据到一次请求——架构、连接、页与数据建模

本章先回答“数据放在哪里、请求怎么进去、为什么会慢”。读完后，再进入索引才不会只背 B+树名词。

本章学习顺序：

```text
Q01 MySQL Server 层和 InnoDB 分别负责什么？
  ↓
Q02 一条 SELECT 从 JDBC 到返回结果经历什么？
  ↓
Q03 为什么数据库连接池不能设置得越大越好？
  ↓
Q04 InnoDB Buffer Pool 有什么作用？
  ↓
Q05 InnoDB 行格式和大字段怎样影响性能？
  ↓
Q06 企业级 Agent 平台的消息、Trace 和大 Prompt 应该怎样存储？
```

---

## Q01. MySQL Server 层和 InnoDB 分别负责什么？

> **题型**：高频基础题　｜　**实际难度**：简单　｜　**面试频率**：★★★★★　｜　**重要性**：★★★★★

### ① 为什么现在学习这题

先把 MySQL 拆成 Server 与 InnoDB 两层，后面的索引、事务、日志才有位置可放。

### ② 先建立一个具体画面

把 MySQL 想成餐厅：Server 层负责接单、看菜单并安排做法，InnoDB 负责真正从仓库取食材、加工和保存库存记录。

### ③ 从直觉进入标准概念

MySQL 不是一个不可拆分的黑盒，最重要的基础模型是：

```text
客户端 / JDBC
  ↓
MySQL Server 层
  ├── 连接、认证、权限
  ├── SQL 解析
  ├── 优化器
  ├── 执行器
  └── Binlog
  ↓ Handler 接口
InnoDB
  ├── Buffer Pool 与数据页
  ├── B+树索引
  ├── 事务、MVCC、锁
  ├── Redo / Undo
  └── 崩溃恢复
```

可直接背诵：

- Server 层负责“理解 SQL、选择执行计划、组织执行”；
- InnoDB 负责“数据怎样存、怎样查、怎样并发修改、崩溃后怎样恢复”；
- Binlog 属于 Server 层，Redo/Undo 属于 InnoDB；
- Handler 是执行器访问不同存储引擎的统一接口。

### ④ 深入原理与源码主链

源码主链可以简化为：

```text
handle_connection
  → do_command
  → dispatch_command
  → dispatch_sql_command
  → mysql_parse
  → Sql_cmd_dml::execute
  → 优化器与迭代器执行
  → handler::ha_*
  → ha_innobase
  → row_search_mvcc / InnoDB 写入路径
```

`THD` 保存一次客户端会话的事务、权限、诊断和执行状态；执行器不会直接操作磁盘文件，而是通过 `handler` 调用 InnoDB。InnoDB 再访问 Buffer Pool、B+树和事务系统。

这条边界能帮助排障：语法、权限和计划问题多在 Server 层；页、锁、Redo、Undo 和 Buffer Pool 问题多在 InnoDB。

### ⑤ 企业级边界与演进

企业项目不能看到“数据库慢”就统一扩容。应先判断：

- 连接获取慢：应用连接池或数据库并发入口；
- 优化阶段慢：复杂 SQL、统计信息或计划；
- InnoDB 读慢：页未命中、索引访问量大；
- 写慢：锁、Redo、刷盘或复制；
- 返回慢：大结果集和客户端读取。

MySQL 的本地事务保证不能外推到 Redis、RocketMQ、Milvus 或外部工具。

### ⑥ 最小案例

```sql
SELECT id, status
FROM agent_run
WHERE tenant_id = 1001
ORDER BY created_at DESC
LIMIT 20;
```

状态变化：

```text
JDBC 发送 SQL
→ Server 解析并选择 idx_run_tenant_status_time
→ InnoDB 从索引页取记录
→ Server 组装结果集
→ JDBC 读取返回
```

### ⑦ 本题小结

> Server 层决定“怎样执行”，InnoDB 决定“怎样存取、并发和恢复”。

### ⑧ 2～3 分钟面试口述

#### 口述回答

**(0:00–0:30) 先给结论**

“MySQL 可以分为 Server 层和存储引擎层。Server 层负责连接、认证、SQL 解析、优化、执行框架和 Binlog；InnoDB 负责数据页、B+树索引、Buffer Pool、事务、锁以及 Redo、Undo 和崩溃恢复。”

**(0:30–1:10) 解释调用关系**

“一条 SQL 先进入 Server 层，优化器生成执行计划，执行器再通过 Handler 接口向 InnoDB 请求记录。InnoDB 返回行以后，Server 层继续做表达式计算、排序、聚合或结果发送。”

**(1:10–1:50) 结合项目**

“在 NexusAgent 中，Run 列表变慢时，我不会直接说是 InnoDB 慢，而会先区分连接池等待、优化器选路、InnoDB 读页、锁等待和结果集传输。不同阶段需要不同证据。”

**(1:50–2:20) 边界与总结**

“Binlog 属于 Server 层，Redo 和 Undo 属于 InnoDB。把这些层混在一起，会导致把加索引、扩连接池和调刷盘参数当成同一种优化。”

### ⑨ 面试官可能继续追问

- Handler 接口为什么重要？
- 这个机制的失败窗口在哪里？你会用什么指标、SQL 或压测证明方案有效？

> **记忆句**：Server 负责理解和组织，InnoDB 负责存取、并发与恢复。

---

## Q02. 一条 SELECT 从 JDBC 到返回结果经历什么？

> **题型**：高频基础题　｜　**实际难度**：简单　｜　**面试频率**：★★★★★　｜　**重要性**：★★★★★

### ① 为什么现在学习这题

学会把一次慢请求拆成连接、解析、优化、存储和返回，而不是笼统地说“数据库慢”。

### ② 先建立一个具体画面

用户请求 Run 列表时，SQL 不是瞬移到磁盘：它要先拿连接、过网络、被解析和优化，再由 InnoDB 找页，最后把结果传回 Java。

### ③ 从直觉进入标准概念

一条查询至少经历九步：

```text
1. 应用从连接池取得连接
2. JDBC 编码 SQL 和参数
3. MySQL 接收协议包
4. 认证、权限和会话检查
5. 解析 SQL，建立语法树
6. 优化器选择访问路径与 Join 顺序
7. 执行器按迭代器拉取记录
8. InnoDB 访问索引页和数据页
9. Server 编码结果，JDBC 读取并映射对象
```

必须区分：

- SQL 执行时间；
- 连接池等待；
- 网络传输；
- 大结果集读取和 Java 对象创建。

所以接口 P99 不等于 `EXPLAIN` 中某个节点的耗时。

### ④ 深入原理与源码主链

查询执行的高频源码心智链：

```text
do_command
→ dispatch_command(COM_QUERY)
→ dispatch_sql_command
→ mysql_parse
→ Sql_cmd_dml::execute
→ Query_expression / Query_block::optimize
→ JOIN::optimize
→ 访问路径迭代器 Read()
→ handler::ha_index_read_map / ha_rnd_next
→ ha_innobase
→ row_search_mvcc
```

MySQL 8.x 执行计划最终形成一组迭代器。父迭代器不断调用子迭代器的 `Read()` 拉取行；Nested Loop、Filter、Sort、Aggregate 都可理解为不同迭代器。

InnoDB 读取时，先在 Buffer Pool 找页；缺页才从表空间读取。通过二级索引取到主键后，可能再次访问聚簇索引。

### ⑤ 企业级边界与演进

生产排障需要端到端 Trace，至少记录：

```text
pool_wait_ms
mysql_execute_ms
rows_examined
rows_sent
result_bytes
SQL digest
transaction_id / trace_id
```

大结果集会长时间占连接，即使数据库很快，也可能让后续请求在 HikariCP 等待。

### ⑥ 最小案例

```sql
SELECT id, status, created_at
FROM agent_run
WHERE tenant_id = ?
ORDER BY created_at DESC, id DESC
LIMIT 50;
```

输出不仅是 50 行，还包含执行计划、扫描行、网络字节和连接占用时间。列表接口应显式投影，不能无条件 `SELECT *`。

### ⑦ 本题小结

> 一条 SELECT 是连接、解析、优化、页访问和结果传输的完整流水线。

### ⑧ 2～3 分钟面试口述

#### 口述回答

**(0:00–0:30) 概括链路**

“Java 先从连接池取得连接，通过 MySQL 协议发送 SQL；Server 层完成解析、权限检查、优化和执行；执行器通过 Handler 调用 InnoDB；InnoDB 读取页和索引，结果再经 Server、网络和 JDBC 返回。”

**(0:30–1:10) 说明为什么要分段**

“接口慢可能慢在连接池等待、优化器选路、页读取、锁等待、排序、结果传输或 Java 反序列化。数据库内部只是一部分，所以不能只看一条 Slow Log 就下结论。”

**(1:10–1:50) 结合项目**

“NexusAgent 的列表接口曾经使用 SELECT *，把大 Prompt 和 Trace 一起取回。SQL 本身没有明显锁等待，但连接持有时间和返回字节很大，最终连 Worker 写状态也开始排队。”

**(1:50–2:20) 排查方法**

“我会给一次请求记录 connection_wait、query_time、rows_examined、rows_sent 和 response_bytes，先判断慢在哪一段，再决定是改 SQL、改索引、拆大字段还是调整连接预算。”

### ⑨ 面试官可能继续追问

- 如何判断问题位于 Server 层还是 InnoDB？
- 这个机制的失败窗口在哪里？你会用什么指标、SQL 或压测证明方案有效？

> **记忆句**：慢请求要分段，不要把 JDBC 到 Java 返回都叫 SQL 时间。

---

## Q03. 为什么数据库连接池不能设置得越大越好？

> **题型**：高频场景题　｜　**实际难度**：中等　｜　**面试频率**：★★★★★　｜　**重要性**：★★★★★

### ① 为什么现在学习这题

连接池是数据库入口的背压阀；理解它才能解释为什么扩容应用后数据库反而更慢。

### ② 先建立一个具体画面

20 个 Pod 每个开 100 个连接，看似更强，实际可能让 2000 条 SQL 同时挤进一个只能稳定处理 150 个活跃查询的数据库。

### ③ 从直觉进入标准概念

连接池解决的是“复用昂贵连接”和“限制应用并发进入数据库”，不是凭空增加数据库吞吐。

连接过少：请求在应用侧排队，CPU 和磁盘可能未被利用。

连接过多：

- MySQL 活跃线程和会话状态增多；
- Buffer、排序、临时表内存按并发放大；
- 锁竞争和上下文切换增加；
- 慢 SQL 同时进入，把排队从应用转移到数据库；
- 故障时重试洪峰更严重。

### ④ 深入原理与源码主链

每个连接对应一个 `THD` 会话上下文，并可能持有事务、MDL、行锁、排序缓冲和结果状态。

```text
并发量 ≈ 吞吐 × 平均响应时间
```

当存储或锁已经饱和，继续增加并发只会提高排队时间。

### ⑤ 企业级边界与演进

- API、RocketMQ Worker、报表任务使用独立连接预算；
- 设置连接获取超时，拒绝无限等待；
- 事务外执行模型、HTTP 和 MCP 调用；
- 结合 `Threads_running`、连接等待、锁等待和磁盘利用率压测；
- 总连接数按所有 Pod 相加。

### ⑥ 最小案例

20 个 Pod、每个池 50，理论上限为 1000。若数据库稳定活跃并发只有 120，多出的连接只会让故障同时压入更多 SQL。

### ⑦ 本题小结

> 连接池既是复用器，也是数据库入口的背压阀。

### ⑧ 2～3 分钟面试口述

#### 口述回答

“连接池的作用是复用连接并控制进入数据库的并发，不是连接越多吞吐越高。真正瓶颈仍然是 MySQL 的 CPU、I/O、锁、日志和 Buffer Pool。连接过多会让更多 SQL 同时竞争资源，增加上下文切换、锁等待和缓存抖动。NexusAgent 会把在线 API、异步 Worker 和批量归档分开限额。连接池大小应通过混合读写压测决定，同时观察 active、pending、获取连接耗时、Threads_running、CPU、锁等待和提交延迟。”

### ⑨ 面试官可能继续追问

- 连接池大小如何通过压测确定？
- 连接获取超时和 SQL 超时应该怎样配合？

> **记忆句**：连接池是闸门，不是数据库算力放大器。

---

## Q04. InnoDB Buffer Pool 有什么作用？

> **题型**：高频基础题　｜　**实际难度**：简单　｜　**面试频率**：★★★★★　｜　**重要性**：★★★★★

### ① 为什么现在学习这题

先知道行和索引页通常从哪里读，才能理解命中率、脏页、刷盘和容量。

### ② 先建立一个具体画面

查询一行时，InnoDB 通常先检查 Buffer Pool 中是否已经有对应 16KB 页；没有才从表空间读入。

### ③ 从直觉进入标准概念

Buffer Pool 主要保存数据页、索引页、脏页和管理信息。查询命中时不需要从数据文件读取；更新通常先修改内存页并产生 Redo，后台再刷脏页。它不是 SQL 结果缓存，也不缓存 Java 对象。

### ④ 深入原理与源码主链

```text
Page Hash：按表空间和页号定位缓存页
Free List：空闲页
LRU List：冷热页
Flush List：脏页
```

读取主链：

```text
row_search_mvcc
→ btr_cur_search_to_nth_level
→ buf_page_get
→ 命中或读取页
```

### ⑤ 企业级边界与演进

命中率高仍可能因为锁等待、宽扫描、日志刷盘或大结果集而慢。还要关注脏页、工作集、扫描污染、容器内存和 Buffer Pool 外的内存。

### ⑥ 最小案例

`agent_run` 保存窄元数据，`agent_message` 保存正文，列表只访问热点索引页，详情才加载消息正文页。

### ⑦ 本题小结

> Buffer Pool 缓存页，Redo 保证改过的页可恢复。

### ⑧ 2～3 分钟面试口述

#### 口述回答

“Buffer Pool 是 InnoDB 缓存数据页和索引页的核心内存区域。读取未命中时才从磁盘加载；更新后页成为脏页并生成 Redo，提交不必同步把所有随机数据页写回。它同时影响读性能和写入平滑性，但索引错误、锁等待、日志刷盘和大结果集不能靠简单增大 Buffer Pool 解决。”

### ⑨ 面试官可能继续追问

- Buffer Pool 命中率高为什么仍可能慢？
- 中点插入式 LRU 解决什么问题？

> **记忆句**：Buffer Pool 缓存页，Redo 保证改过的页可恢复。

---

## Q05. InnoDB 行格式和大字段怎样影响性能？

> **题型**：场景题　｜　**实际难度**：中等　｜　**面试频率**：★★★★☆　｜　**重要性**：★★★★★

### ① 为什么现在学习这题

表设计不是只看能否存下；行宽会影响页密度、回表、网络和 Java 堆。

### ② 先建立一个具体画面

列表接口只要 id、status，却因为 `SELECT *` 把 50 条大 Prompt 一起返回，扫描不多但连接和网络被占满。

### ③ 从直觉进入标准概念

InnoDB 以页组织记录。行越宽，每页记录越少，Buffer Pool 有效容量下降，扫描、排序、网络和 Java 映射字节增加。大 `VARCHAR`、`TEXT`、`BLOB` 可能使用页外存储。

### ④ 深入原理与源码主链

大变长列放不下时，聚簇记录保存外部页引用；访问正文需读取额外页。更新大 JSON/TEXT 也可能产生更多 Undo、Redo 和复制字节。

### ⑤ 企业级边界与演进

- 热元数据与大正文纵向拆表；
- 列表显式投影；
- 大附件与完整 Trace 放对象存储；
- 限制单消息、单 Prompt 和单 JSON 大小；
- 监控结果字节和平均行长。

### ⑥ 最小案例

```text
agent_run：状态和版本
agent_message：消息正文
trace_blob：对象地址、摘要和大小
```

### ⑦ 本题小结

> 大字段治理的核心是冷热分离和按需投影。

### ⑧ 2～3 分钟面试口述

#### 口述回答

“InnoDB 以页组织记录。行越宽，同一页容纳的记录越少；大 VARCHAR、TEXT 或 BLOB 还可能使用页外存储。宽行降低 Buffer Pool 有效容量，即使 WHERE 使用索引，SELECT * 仍可能因为回表和大字段传输而慢。我会把 agent_run 热元数据与 agent_message 正文拆开，列表显式投影，特别大的 Trace 和附件放对象存储。”

### ⑨ 面试官可能继续追问

- 页外存储是否意味着 TEXT 完全不占聚簇记录？
- 怎样确定大字段拆表的收益？

> **记忆句**：宽行降低页密度，大字段必须按需读取。

---

## Q06. 企业级 Agent 平台的消息、Trace 和大 Prompt 应该怎样存储？

> **题型**：项目深挖题　｜　**实际难度**：中等偏上　｜　**面试频率**：★★★★☆　｜　**重要性**：★★★★★

### ① 为什么现在学习这题

把前面数据类型和冷热访问模式落到 Agent 项目，建立真实建模能力。

### ② 先建立一个具体画面

Run 元数据、消息正文、Trace 和附件访问频率不同，不应全部塞进一张宽表。

### ③ 从直觉进入标准概念

```text
业务事实：Run 状态、审批、版本、operationId → MySQL
在线消息：按 run_id + seq_no 查询 → MySQL
超大对象：附件、完整 Trace、原始响应 → 对象存储
向量索引：可重建 Embedding → Milvus
热缓存：可重建摘要和结果 → Redis
```

MySQL 必须保留对象 URI、摘要、大小、版本和上传状态。

### ④ 深入原理与源码主链

推荐状态机：

```text
PENDING_UPLOAD
→ 对象上传并校验 digest
→ MySQL 事务更新 AVAILABLE
→ 同事务写 Outbox
```

消息表通过 `(tenant_id, run_id, seq_no)` 唯一索引保证局部顺序和去重。

### ⑤ 企业级边界与演进

制定单条正文和单 Run 上限、在线保留与归档周期、租户隔离、加密、PII 脱敏、对象生命周期、孤儿清理和恢复演练。

### ⑥ 最小案例

```sql
UNIQUE KEY uk_message_run_seq(tenant_id, run_id, seq_no);
UNIQUE KEY uk_tool_operation(tenant_id, operation_id);
```

大 Prompt 放对象存储，MySQL 保存 `object_uri + sha256 + size + schema_version`。

### ⑦ 本题小结

> MySQL 保存不可丢状态和引用，大对象与可重建索引进入专用系统。

### ⑧ 2～3 分钟面试口述

#### 口述回答

“我先按访问频率和恢复价值分层：Run 元数据属于在线热行；消息按 run_id 和序号存独立表；大 Trace、附件和长期归档放对象存储，MySQL 保存摘要、状态和对象地址。关键业务状态与对象引用在本地事务中提交，异步归档通过 Outbox 触发并用 event_id 幂等。我还会限制单条消息大小，监控行宽、rows_sent、结果字节和对象下载失败率。”

### ⑨ 面试官可能继续追问

- JSON 字段什么时候应拆成关系列？
- 对象上传成功但数据库更新超时时怎样对账？

> **记忆句**：热元数据进窄表，大正文按序分页，大附件进对象存储。
