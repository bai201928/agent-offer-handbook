# 第四部分：并发时谁能读、谁能写——事务、MVCC、锁与 Spring 边界

本章先建立版本可见性，再学习当前读、锁范围、死锁和 Worker 原子认领。

```text
Q14 ACID → Q15 MVCC → Q16 RC/RR → Q17 一致性读/当前读
→ Q18 锁范围 → Q19 死锁 → Q20 悲观/乐观锁
→ Q21 Worker 认领 → Q22 Spring 事务边界
```

---

## Q14. 如何真正理解事务的 ACID？

> **题型**：高频基础题｜**实际难度**：中等｜**频率**：★★★★★

### ① 为什么现在学
事务是 MVCC、锁和日志的总框架，但数据库不会自动理解业务规则。

### ② 具体画面
转账事务能保证两条 SQL 一起提交，但“同一 Run 只能被一个 Worker 领取”仍需条件更新或唯一约束。

### ③ 标准概念
- 原子性：本地事务整体提交或回滚；
- 一致性：数据库约束与业务状态机共同保持合法状态；
- 隔离性：MVCC 和锁控制并发影响；
- 持久性：提交结果可由日志恢复。

### ④ 深入原理

```text
Undo → 回滚与历史版本
MVCC/锁 → 隔离
Redo/WAL → 崩溃恢复
Binlog → 复制、CDC、时间点恢复
```

### ⑤ 企业级边界
外部 HTTP、LLM、Milvus、RocketMQ 和 MCP Tool 不在 MySQL 本地事务中。事务内只做状态、约束和 Outbox；提交后调用外部系统；回写时再用 version/CAS 裁决。

### ⑥ 最小案例

```sql
UPDATE agent_run
SET status='RUNNING',owner=?,state_version=state_version+1
WHERE tenant_id=? AND id=?
  AND status='PENDING' AND state_version=?;
```

`affected_rows=1` 才获得执行权。

### ⑦ 小结
> ACID 提供本地事务机制，业务不变量必须显式建模。

### ⑧ 2～3 分钟口述
“原子性保证本地事务整体提交或回滚；隔离性控制并发可见性和锁；持久性依赖 Redo 与恢复；一致性是数据库机制、约束与业务规则共同完成。MySQL 不会自动知道一个 Run 只能执行一次，也不能把 Redis、MQ、Milvus 和外部 Tool 纳入本地事务。NexusAgent 在事务内用条件 UPDATE 认领 Run 并写 Outbox，事务外调用 Tool，再用 operation_id 唯一约束和 version 防止重复与旧写覆盖。”

### ⑨ 追问
- ACID 中的一致性由谁定义？
- 为什么长外部调用不能放在事务中？

> **记忆句**：事务保证数据库边界，业务不变量要显式表达。

---

## Q15. MVCC 是什么？Undo 版本链怎样工作？

> **题型**：高频原理题｜**实际难度**：中等｜**频率**：★★★★★

### ① 为什么现在学
用 Undo 版本链解释非阻塞读取，建立 MVCC 第一张图。

### ② 具体画面
事务 A 更新记录后，事务 B 可沿 Undo 链找到符合自身 Read View 的旧版本。

### ③ 标准概念
记录可理解为带有最后修改事务 ID 和指向 Undo 的回滚指针。当前版本不可见时，InnoDB 沿 Undo 构造更早版本，直到满足 Read View。

### ④ 深入原理

```text
row_search_mvcc
→ 判断当前记录可见性
→ row_vers_build_for_consistent_read
→ 沿 DB_ROLL_PTR 读取 Undo
→ 构造可见历史记录
```

Undo 同时用于回滚和一致性读。长事务长期持有 Read View，会使历史版本难以清理。

### ⑤ 企业级边界
普通 SELECT 可读快照，不代表最新；UPDATE、DELETE 和锁定读仍需当前版本和锁；副本延迟与 MVCC 是两种不同的旧数据。

### ⑥ 最小案例
Worker A 快照看到 PENDING，但 B 已提交 RUNNING；A 不能直接执行工具，必须用条件 UPDATE 由当前事实裁决。

### ⑦ 小结
> MVCC = Read View + Undo 版本链。

### ⑧ 2～3 分钟口述
“MVCC 是多版本并发控制。InnoDB 为记录保存事务信息，并通过 Undo 保留旧版本。一致性读用 Read View 判断当前版本是否可见，不可见时沿 Undo 链构造历史版本。它让普通查询和写入在很多场景并发，但不是无锁，也不保证读到最新。Run 详情可读快照，Worker 认领必须使用条件 UPDATE。长事务会让旧版本长期不能清理，增加 Undo 和 Purge 压力。”

### ⑨ 追问
- Read View 中的活跃事务列表怎样参与可见性判断？
- 长事务如何影响 Purge？

> **记忆句**：快照由 Read View 裁决，旧版本来自 Undo。

---

## Q16. READ COMMITTED 和 REPEATABLE READ 的主要区别是什么？

> **题型**：高频对比题｜**实际难度**：中等｜**频率**：★★★★★

### ① 为什么现在学
隔离级别的核心差异在 Read View 建立时机。

### ② 具体画面
RC 每条一致性读通常建立新视图；RR 在事务内通常复用第一次快照读的视图。

### ③ 标准概念
RC 后一次 SELECT 可看到其他事务刚提交的数据；RR 的快照读在同一事务中更稳定。锁定读、UPDATE、DELETE 读取当前版本，不等同于快照读。

### ④ 深入原理
RC 语句级更新 Read View，RR 事务级复用。RR 的范围当前读还可能通过 Next-Key Lock 控制幻影插入。

### ⑤ 企业级边界
高并发状态更新更重要的是短事务、索引和 CAS；稳定报表快照可评估 RR；希望减少部分 Gap Lock 并允许每语句看新提交，可评估 RC。

### ⑥ 最小案例
同一事务两次普通 SELECT：另一事务中间提交更新，RC 通常第二次看到新值，RR 通常仍看旧快照；`FOR UPDATE` 则读取当前可锁定版本。

### ⑦ 小结
> RC 通常每语句新视图，RR 通常事务内复用视图。

### ⑧ 2～3 分钟口述
“在 InnoDB 一致性读中，RC 通常每条语句建立新的 Read View，RR 通常在事务第一次快照读时建立并复用，因此 RC 可能出现不可重复读，RR 的快照更稳定。但锁定读和写操作仍读取当前版本。隔离级别不是越高越好，要结合读一致性、锁范围和并发量选择；任务状态更新仍应使用条件 UPDATE，而不是只依赖隔离级别。”

### ⑨ 追问
- RC 与 RR 对 Gap Lock 有什么影响？
- 当前读为什么不复用普通快照？

> **记忆句**：RC 每次读更新视图，RR 事务内保持视图。

---

## Q17. 一致性读和当前读有什么区别？

> **题型**：高频并发题｜**实际难度**：中等｜**频率**：★★★★★

### ① 为什么现在学
先分清普通 SELECT 与锁定读，才能解释“为什么查到了却不能直接改”。

### ② 具体画面
普通 SELECT 可能读历史版本；`SELECT ... FOR UPDATE` 必须读取当前可锁定记录。

### ③ 标准概念
一致性读通常指普通 SELECT，根据 Read View 读取可见版本；当前读包括 `FOR UPDATE`、`FOR SHARE`、UPDATE、DELETE，读取用于修改/锁定的当前版本并按访问路径加锁。

### ④ 深入原理

```text
consistent read → 可见性判断 → 必要时构造 Undo 版本
current read    → 读取当前记录 → 添加记录/范围锁
```

没有合适索引时，当前读可能扫描并锁住大量记录。

### ⑤ 企业级边界
任务认领优先单条条件 UPDATE；只有需要读取多个字段做复杂决策时，才在短事务中使用 `FOR UPDATE`。不能持锁等待模型或外部 API。

### ⑥ 最小案例

```text
错误：SELECT PENDING → 调工具 → UPDATE
正确：UPDATE ... WHERE status=PENDING → COMMIT → 调工具
```

### ⑦ 小结
> 一致性读负责看快照，当前读负责并发写入裁决。

### ⑧ 2～3 分钟口述
“普通 SELECT 通常是一致性读，根据 Read View 读可见版本；UPDATE、DELETE 和 SELECT FOR UPDATE 属于当前读，读取用于修改或锁定的当前记录。普通 SELECT 看到 PENDING，不代表稍后仍有修改资格。Run 认领优先使用一条条件 UPDATE 并检查 affected_rows；只有复杂短临界区才使用 FOR UPDATE。快照读解决读取并发，当前读和条件更新解决修改裁决。”

### ⑨ 追问
- 当前读的锁范围由什么决定？
- 为什么缺索引会放大锁范围？

> **记忆句**：一致性读看版本，当前读看最新并参与加锁。

---

## Q18. Record Lock、Gap Lock 和 Next-Key Lock 分别是什么？

> **题型**：高频锁题｜**实际难度**：中等偏上｜**频率**：★★★★★

### ① 为什么现在学
理解锁住记录、间隙还是二者，才能解释幻读防护和锁范围。

### ② 具体画面
查询 id 在 10 到 20 之间时，InnoDB 可能不仅锁已有记录，还锁住记录之间的插入间隙。

### ③ 标准概念
- Record Lock：锁索引中的具体记录；
- Gap Lock：锁记录之间的间隙，主要阻止插入；
- Next-Key Lock：记录锁与前方间隙锁的组合。

### ④ 深入原理

```text
WHERE 条件
→ 优化器选择索引
→ InnoDB 扫描索引区间
→ 对访问记录/间隙加锁
```

锁的是索引记录和范围。缺少索引可能让 UPDATE 扫描并锁住大量聚簇记录。

### ⑤ 企业级边界
锁定读和 UPDATE 必须有匹配索引；事务短小且统一顺序；使用 `performance_schema.data_locks` 和 `data_lock_waits` 观察；`SKIP LOCKED` 仍需幂等和状态检查。

### ⑥ 最小案例

```sql
SELECT id FROM task
WHERE status='PENDING'
ORDER BY id LIMIT 1
FOR UPDATE SKIP LOCKED;
```

需要 `(status,id)` 索引。

### ⑦ 小结
> InnoDB 锁的是索引记录与范围，执行计划决定真实锁范围。

### ⑧ 2～3 分钟口述
“Record Lock 锁索引记录；Gap Lock 锁记录间隙，主要阻止插入；Next-Key Lock 是记录锁与间隙锁组合。RR 下范围当前读可能用 Next-Key Lock 防止范围内出现新记录。所谓行锁依赖实际索引路径，如果 Worker 按 status 查询但缺少匹配索引，就可能扫描并锁住大量 Run。分析锁必须看条件、索引、隔离级别和实际计划。”

### ⑨ 追问
- 唯一索引完整等值查询何时只需记录锁？
- InnoDB 是否会自动把大量行锁升级为表锁？

> **记忆句**：锁记录、锁间隙或二者，范围取决于索引和条件。

---

## Q19. MySQL 死锁是怎样产生和处理的？

> **题型**：高频场景题｜**实际难度**：中等偏上｜**频率**：★★★★★

### ① 为什么现在学
死锁是锁顺序形成的等待环，不是随机玄学。

### ② 具体画面
事务 A 先锁 Run 再锁 Tool，事务 B 反过来，双方各等对方释放。

### ③ 标准概念

```text
T1 持有 A，等待 B
T2 持有 B，等待 A
```

InnoDB 检测到等待环后会回滚一个事务。应用必须将死锁视为可重试的本地事务错误。

### ④ 深入原理
死锁与锁等待超时不同：死锁有等待环；超时可能只是长时间等待。可用 `SHOW ENGINE INNODB STATUS` 与 Performance Schema 查看锁关系。

### ⑤ 企业级边界
统一加锁顺序、缩短事务、补索引、批量更新按主键排序；重试整个事务，次数有限并带退避；重试不得重复外部 Tool 副作用。

### ⑥ 最小案例
T1 更新 run1→run2，T2 更新 run2→run1；改为都按 ID 升序，并把外部调用移出事务。

### ⑦ 小结
> 死锁是正常可恢复错误，但高频死锁必须治理根因。

### ⑧ 2～3 分钟口述
“死锁是多个事务形成循环等待。InnoDB 默认检测等待图并回滚一个事务。常见根因是更新顺序不一致、范围锁过大、缺索引和事务过长。应用要捕获死锁并有限重试整个本地事务，同时统一锁顺序、缩短事务、补索引。NexusAgent 统一先更新 agent_run 再更新 tool_operation，远程 Tool 不放在事务内，确保重试不会重复外部副作用。”

### ⑨ 追问
- 如何用 Performance Schema 查看当前锁等待？
- 死锁重试为什么必须重新读取数据？

> **记忆句**：死锁是等待环，统一顺序、缩短事务并允许重试。

---

## Q20. 悲观锁和乐观锁应该怎样选择？

> **题型**：高频设计题｜**实际难度**：中等｜**频率**：★★★★★

### ① 为什么现在学
核心取舍是冲突概率、持锁时间和失败重试成本。

### ② 具体画面
高冲突库存先锁后改更直接；大多数时候无冲突的 Run 状态使用 version 条件更新吞吐更高。

### ③ 标准概念
悲观锁先锁再改，例如 `FOR UPDATE`；乐观锁不提前长期占锁，更新时校验旧状态/version，失败后重新读取。

### ④ 深入原理
乐观锁不是完全无锁，最终 UPDATE 仍取得记录锁；它只是把冲突裁决推迟到写入时，缩短持锁时间。

### ⑤ 企业级边界
简单状态机优先 CAS；多行复杂不变量可用短事务锁定读；跨外部系统不能依赖数据库锁，需 operationId、Outbox 和幂等。

### ⑥ 最小案例

```sql
UPDATE agent_run
SET status='SUCCEEDED',state_version=state_version+1
WHERE id=? AND status='RUNNING' AND state_version=?;
```

### ⑦ 小结
> 悲观锁用等待换所有权，乐观锁用版本冲突换短事务。

### ⑧ 2～3 分钟口述
“悲观锁先锁当前记录再修改；乐观锁在 UPDATE 条件中比较旧状态或 version。冲突频繁、临界区短且必须读取复杂状态时可用悲观锁；冲突相对少、希望降低锁等待时用版本 CAS。Run 认领使用单条条件 UPDATE；批量挑选任务可在短事务中使用 FOR UPDATE SKIP LOCKED。两者都不能保护事务外的外部 Tool。”

### ⑨ 追问
- 乐观锁失败后怎样重试？
- CAS 失败率过高说明什么？

> **记忆句**：高冲突悲观裁决，低冲突版本更新。

---

## Q21. 多个 Agent Worker 如何安全认领同一个 Run？

> **题型**：项目深挖题｜**实际难度**：中等偏上｜**频率**：★★★★★

### ① 为什么现在学
把条件更新、版本和幂等用于真实 Worker 抢占。

### ② 具体画面
两个 Worker 同时看到 PENDING，必须让数据库用一条 UPDATE 决定唯一胜者。

### ③ 标准方案

```sql
UPDATE agent_run
SET status='RUNNING',owner=?,lease_until=?,state_version=state_version+1
WHERE tenant_id=? AND id=?
  AND status='PENDING' AND state_version=?;
```

只有 affected_rows=1 获得执行权。

### ④ 深入原理
并发事务会在同一聚簇记录上串行裁决。后来的事务获得锁后重新检查 WHERE，发现状态已变，更新 0 行。

### ⑤ 企业级边界
需要 lease_until、owner/version、超时接管、事务提交后再 ACK、Tool operationId 幂等、完成回写再次校验 version，以及 Reconciler 扫描过期 RUNNING。

### ⑥ 故障时间线

```text
W1 CAS 成功 → COMMIT → 调工具
W2 CAS 0 行 → 幂等退出
W1 超时 → W3 用更高 version 接管
W1 迟到回写 → UPDATE 0 行，被拒绝
```

### ⑦ 小结
> 认领和回写都要由数据库条件更新裁决。

### ⑧ 2～3 分钟口述
“先 SELECT 再 UPDATE 存在竞态。认领时我使用带 tenant、status 和 state_version 的单条条件 UPDATE，affected_rows=1 才成功，并在同一短事务写 Run Event/Outbox 后提交。远程 Tool 在事务外执行，使用 operation_id 唯一约束；完成时再次校验 owner、status 和 version，防止旧 Worker 覆盖新状态。还要有 lease 超时接管、重复消息幂等和过期任务对账。”

### ⑨ 追问
- 认领成功后 Worker 崩溃怎么办？
- 为什么 Lease 不能阻止暂停的旧 Worker？

> **记忆句**：不要先查再改，让条件 UPDATE 在数据库内裁决。

---

## Q22. Spring @Transactional 有哪些常见边界和失效问题？

> **题型**：高频框架题｜**实际难度**：中等｜**频率**：★★★★★

### ① 为什么现在学
数据库事务边界还受 Spring 代理、异常和线程影响。

### ② 具体画面
同类内部调用绕过代理，方法上的 `@Transactional` 可能没有真正开启事务。

### ③ 常见问题
- 自调用绕过代理；
- 对象不由 Spring 管理；
- 异常被吞掉或回滚规则不匹配；
- 异步线程不继承事务上下文；
- 多数据源事务管理器选择错误；
- 外部 HTTP/LLM 调用放在长事务；
- 消息 ACK 与数据库提交顺序错误。

### ④ 深入原理

```text
代理进入 → 获取 Connection → 绑定线程上下文
→ 执行业务方法 → commit/rollback → 归还连接
```

换线程后原事务上下文通常不存在。传播行为描述本地事务加入、挂起或新建，不等于跨系统事务。

### ⑤ 企业级边界
事务入口放在明确 Service；事务内只做数据库临界区；明确 rollbackFor；外部调用在提交后；通过 Outbox 发布事件；用集成测试验证代理、自调用和异常路径。

### ⑥ 最小案例

```text
@Transactional claimAndCreateOutbox()
COMMIT
→ 调 MCP Tool
→ @Transactional finishRunWithVersion()
```

### ⑦ 小结
> Spring 事务是代理管理的线程内本地数据库边界。

### ⑧ 2～3 分钟口述
“@Transactional 通常依赖 Spring AOP 代理，在代理入口开启本地数据库事务。自调用、非 Spring 对象、异常被吞、异步线程切换、多数据源和传播设置都可能让实际边界与预期不同。它不会自动覆盖 Redis、MQ、HTTP 和外部 Tool。NexusAgent 把认领和完成设计成独立短事务，远程调用放在事务外，通过 Outbox 连接数据库事实与消息发布，并用集成测试验证真实提交和回滚。”

### ⑨ 追问
- 为什么自调用可能失效？
- 数据库提交后消息发送失败怎么办？

> **记忆句**：事务注解依赖代理，边界、异常与传播都要验证。
