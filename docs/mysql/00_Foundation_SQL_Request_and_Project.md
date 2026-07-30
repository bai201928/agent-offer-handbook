# 第一部分：零基础先建立 MySQL 的第一张图

> 本部分不要求你先懂 B+树、MVCC、Redo、Binlog。目标只有一个：先看懂“一行数据怎样被写进去，又怎样被查出来”。

---

## 1. 数据库、表、行、列到底是什么

假设 NexusAgent 要保存一次任务：

```text
Run 10001
租户：tenant-7
状态：PENDING
创建时间：2026-07-30 16:00
```

最简单的表可以想成一张有规则的 Excel：

| id | tenant_id | status | created_at |
|---:|---:|---|---|
| 10001 | 7 | PENDING | 2026-07-30 16:00:00 |

对应关系：

```text
数据库 Database
  = 存放多张业务表的空间

表 Table
  = 一类业务对象的集合，例如 agent_run

行 Row
  = 一个具体对象，例如 Run 10001

列 Column
  = 对象的一项属性，例如 status

主键 Primary Key
  = 每一行不可重复的身份证

唯一约束 Unique Key
  = 某组业务字段不能重复
```

创建表：

```sql
CREATE TABLE agent_run (
    id            BIGINT NOT NULL,
    tenant_id     BIGINT NOT NULL,
    status        VARCHAR(32) NOT NULL,
    state_version BIGINT NOT NULL DEFAULT 0,
    created_at    DATETIME(6) NOT NULL,
    updated_at    DATETIME(6) NOT NULL,
    PRIMARY KEY (id)
) ENGINE=InnoDB;
```

插入一行：

```sql
INSERT INTO agent_run (
    id, tenant_id, status, created_at, updated_at
) VALUES (
    10001, 7, 'PENDING', NOW(6), NOW(6)
);
```

查询：

```sql
SELECT id, status
FROM agent_run
WHERE tenant_id = 7
  AND id = 10001;
```

更新：

```sql
UPDATE agent_run
SET status = 'RUNNING',
    state_version = state_version + 1,
    updated_at = NOW(6)
WHERE tenant_id = 7
  AND id = 10001
  AND status = 'PENDING';
```

删除：

```sql
DELETE FROM agent_run
WHERE tenant_id = 7
  AND id = 10001;
```

这四类操作就是最基础的 CRUD：

```text
Create → INSERT
Read   → SELECT
Update → UPDATE
Delete → DELETE
```

---

## 2. 主键、唯一约束和普通索引不是一回事

### 2.1 主键

```sql
PRIMARY KEY (id)
```

作用：

- 唯一标识一行；
- 不能为 NULL；
- InnoDB 通常按主键组织整行数据。

### 2.2 唯一约束

消息重试时，同一个工具操作不能重复落库：

```sql
UNIQUE KEY uk_tool_operation (tenant_id, operation_id)
```

即使两个线程同时插入，数据库也只允许一个成功。**先查询是否存在，再决定插入**无法抵抗并发竞态；最终裁决应由唯一约束完成。

### 2.3 普通索引

```sql
KEY idx_run_tenant_status_time
    (tenant_id, status, created_at DESC, id DESC)
```

普通索引主要帮助数据库更快找到目标范围，但不保证业务值唯一。

记忆：

```text
主键：这一行是谁
唯一键：哪些业务身份不能重复
普通索引：从哪里更快找到数据
```

---

## 3. SQL 的基本阅读顺序

一条常见列表查询：

```sql
SELECT id, status, created_at
FROM agent_run
WHERE tenant_id = 7
  AND status = 'RUNNING'
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

先按人话读：

```text
从 agent_run 表
找到 tenant_id=7 且 status=RUNNING 的行
按 created_at、id 从新到旧排序
只取前 20 条
最后返回 id、status、created_at
```

SQL 的书写顺序和逻辑处理顺序不同。初学阶段记住这条主线即可：

```text
FROM / JOIN
  ↓
WHERE
  ↓
GROUP BY
  ↓
HAVING
  ↓
SELECT
  ↓
ORDER BY
  ↓
LIMIT
```

这能解释为什么：

- `WHERE` 不能直接使用同层 `SELECT` 别名；
- `HAVING` 常用于聚合结果过滤；
- `LIMIT` 在排序之后截取结果。

---

## 4. JOIN 先理解“把两张表按关系拼起来”

Run 表：

| id | tenant_id | agent_id | status |
|---:|---:|---:|---|
| 10001 | 7 | 501 | RUNNING |

Agent 表：

| id | tenant_id | name |
|---:|---:|---|
| 501 | 7 | CustomerSupportAgent |

查询 Run 与 Agent 名称：

```sql
SELECT r.id, r.status, a.name
FROM agent_run r
JOIN agent a
  ON a.tenant_id = r.tenant_id
 AND a.id = r.agent_id
WHERE r.tenant_id = 7;
```

最常见的两种：

```text
INNER JOIN
  只返回左右两边都匹配的行

LEFT JOIN
  左表全部保留，右表没有匹配时补 NULL
```

大厂面试不会只看你能否背 JOIN 类型，更关心：

- Join 条件是否有索引；
- 是否遗漏 tenant_id 导致跨租户串数据；
- 驱动表实际有多少行；
- Nested Loop 会循环多少次；
- 返回的数据是否过大。

---

## 5. 字段类型先从业务正确性选择

| 数据 | 常用类型 | 初学者理解 |
|---|---|---|
| 主键、数量 | BIGINT / INT | 整数 |
| 金额 | DECIMAL | 精确十进制，不用 DOUBLE 存钱 |
| 状态、短名称 | VARCHAR | 长度可变的短字符串 |
| 大正文 | TEXT / MEDIUMTEXT | 应按需读取，避免列表 SELECT * |
| 时间 | DATETIME(6) | 保存业务时间 |
| 灵活扩展属性 | JSON | 适合低频扩展，不替代核心列 |
| 固定摘要 | BINARY / VARBINARY | 可比字符串更紧凑 |

几个常见校正：

```text
VARCHAR(100) 中的 100 是字符上限，不是固定 100 字节。
INT 的存储范围不由旧式显示宽度决定。
TEXT 不是无限大，不同 TEXT 类型有不同字节上限。
NULL、空字符串和 0 是三种不同状态。
```

---

## 6. 一次 Java 请求的第一张图

```text
Controller
  ↓
Service
  ↓ 从 HikariCP 取得 Connection
JDBC / MySQL Driver
  ↓ 网络协议
MySQL Server
  ├─ 认证与权限
  ├─ 解析 SQL
  ├─ 优化器选执行计划
  └─ 执行器
  ↓ Handler
InnoDB
  ├─ Buffer Pool
  ├─ B+树与数据页
  ├─ MVCC 与锁
  └─ Redo / Undo
  ↓
结果集返回 JDBC
  ↓
Java 映射对象
```

因此接口慢可能慢在：

```text
等待连接
网络
SQL 解析与优化
索引扫描
回表
锁等待
磁盘读取
排序与临时表
大结果集传输
Java 对象创建和 GC
```

后面的教材会逐段拆开，而不是把所有问题都叫作“SQL 慢”。

---

## 7. 贯穿全书的 NexusAgent 项目

### 7.1 四个不变量

```text
1. tenant-7 的数据不能被 tenant-8 查询或更新。
2. 一个 PENDING Run 同一时刻只能有一个 Worker 认领成功。
3. 消息重试不能让同一个 Tool Operation 重复产生外部副作用。
4. 接口返回成功后，关键状态能够恢复、审计和对账。
```

### 7.2 核心表

```sql
CREATE TABLE agent_run (
    id              BIGINT NOT NULL,
    tenant_id       BIGINT NOT NULL,
    agent_id        BIGINT NOT NULL,
    status          VARCHAR(32) NOT NULL,
    owner           VARCHAR(64) NULL,
    state_version   BIGINT NOT NULL DEFAULT 0,
    created_at      DATETIME(6) NOT NULL,
    updated_at      DATETIME(6) NOT NULL,
    PRIMARY KEY (id),
    KEY idx_run_tenant_status_time
        (tenant_id, status, created_at DESC, id DESC)
) ENGINE=InnoDB;

CREATE TABLE agent_message (
    id              BIGINT NOT NULL,
    tenant_id       BIGINT NOT NULL,
    run_id          BIGINT NOT NULL,
    seq_no          INT NOT NULL,
    role            VARCHAR(16) NOT NULL,
    content         MEDIUMTEXT NOT NULL,
    created_at      DATETIME(6) NOT NULL,
    PRIMARY KEY (id),
    UNIQUE KEY uk_message_run_seq
        (tenant_id, run_id, seq_no),
    KEY idx_message_run
        (tenant_id, run_id, seq_no)
) ENGINE=InnoDB;

CREATE TABLE tool_operation (
    id              BIGINT NOT NULL,
    tenant_id       BIGINT NOT NULL,
    operation_id    VARCHAR(96) NOT NULL,
    run_id          BIGINT NOT NULL,
    tool_name       VARCHAR(64) NOT NULL,
    status          VARCHAR(32) NOT NULL,
    result_digest   VARCHAR(128) NULL,
    created_at      DATETIME(6) NOT NULL,
    updated_at      DATETIME(6) NOT NULL,
    PRIMARY KEY (id),
    UNIQUE KEY uk_tool_operation
        (tenant_id, operation_id)
) ENGINE=InnoDB;

CREATE TABLE outbox_event (
    id              BIGINT NOT NULL,
    tenant_id       BIGINT NOT NULL,
    event_id        VARCHAR(96) NOT NULL,
    aggregate_type  VARCHAR(32) NOT NULL,
    aggregate_id    BIGINT NOT NULL,
    event_type      VARCHAR(64) NOT NULL,
    payload         JSON NOT NULL,
    published       TINYINT NOT NULL DEFAULT 0,
    created_at      DATETIME(6) NOT NULL,
    PRIMARY KEY (id),
    UNIQUE KEY uk_outbox_event (tenant_id, event_id),
    KEY idx_outbox_publish (published, created_at, id)
) ENGINE=InnoDB;
```

### 7.3 一次 Run 的业务时间线

```text
用户提交任务
  ↓
插入 agent_run=PENDING
  ↓
Worker 条件 UPDATE 认领
  ↓
agent_run=RUNNING，state_version+1
  ↓
事务外调用模型、Milvus、MCP Tool
  ↓
tool_operation 用 operation_id 幂等
  ↓
事务内更新 Run 终态并写 outbox_event
  ↓
发布 RocketMQ
  ↓
消费者继续执行后续流程
```

学习每个 MySQL 机制时，都要问：

```text
它在这条链的哪个位置？
它保护了什么？
它没有保护什么？
失败后怎样恢复和验证？
```
