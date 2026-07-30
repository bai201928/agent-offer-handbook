# Redis 8.8 零基础企业级基础课

## ——先看懂 Key、Value、TTL 和一条请求，再进入 30 道大厂面试题

> **版本基线**：Redis Open Source 8.8.0。  
> **项目主线**：NexusAgent——多租户 Agent + RAG 平台。  
> **学习目标**：从第一次使用 Redis，逐步走到缓存一致性、高可用、分布式锁、Stream、内存治理和 Agent Checkpoint。

---

# 一、为什么上一版初学者会读不懂

初学者第一次看到下面这句话：

```text
Redis 通过 aeProcessEvents、dict、SDS、listpack、PSYNC 和 COW 实现高性能与高可用。
```

通常无法建立画面，因为每个词都需要前置知识。

新版改成：

```text
先执行 SET 和 GET
  ↓
知道数据叫什么、存了什么、多久失效
  ↓
知道 Java 请求怎样到达 Redis
  ↓
知道数据为什么会过期、丢失、重复或占满内存
  ↓
最后再给机制和源码名称
```

学习规则：

1. 每次只新增一个概念；
2. 所有抽象术语先用订单或 Agent Run 演示；
3. 先讲“发生了什么”，再讲“内部怎样实现”；
4. 每道题最后给 2～3 分钟口述，不要求先背；
5. 低频、项目用不到的内容不单独设题。

---

# 二、从一个没有 Redis 的 Agent 请求开始

用户向 NexusAgent 提问：

```text
“请根据退款制度判断订单 O1001 是否可以退款。”
```

系统需要：

```text
读取会话
读取最近消息
读取知识库版本
查询向量库
调用模型
保存工具结果
更新限流计数
防止两个 Worker 重复执行
```

如果每一步都访问 MySQL：

```text
大量短小请求
  ↓
占用连接
  ↓
增加索引、锁和磁盘压力
```

Redis 适合保存**读取频繁、修改简单、允许设置生命周期的热状态**。

但订单、支付、审计等不可丢业务事实，仍应保存在 MySQL 或其他权威系统。

---

# 三、第一次认识 Key、Value 和 TTL

执行：

```redis
SET session:1001 "用户正在咨询退款" EX 1800
```

逐段解释：

```text
SET
  → 保存数据

session:1001
  → Key，数据的名字

"用户正在咨询退款"
  → Value，真正的数据

EX 1800
  → 1800 秒后过期
```

## 1. Key 是什么

**Key 是数据的唯一名字。**

类比储物柜编号：

```text
session:1001
order:O1001:status
agent:run:R9001:state
```

好的 Key 应让人看出：

```text
属于哪个业务
属于哪个租户
代表什么资源
业务 ID 是什么
是否带版本
```

## 2. Value 是什么

Value 是 Key 对应的内容：

```text
Key   = order:O1001:status
Value = PAID
```

Value 不只可以是文本，还可以是数字、JSON、二进制或 Redis 的集合结构。

## 3. TTL 是什么

TTL 是剩余有效时间。

```redis
TTL session:1001
```

返回 `1750`，表示还剩 1750 秒。

类比：

```text
Key   = 商品位置
Value = 商品
TTL   = 保质期
```

---

# 四、第一次认识 SET、GET、DEL 和 EXISTS

```redis
SET order:O1001:status PAID
GET order:O1001:status
EXISTS order:O1001:status
DEL order:O1001:status
```

含义：

| 命令 | 作用 |
|---|---|
| SET | 保存或覆盖 |
| GET | 读取 |
| EXISTS | 判断 Key 是否存在 |
| DEL | 立即删除 |

不要把“命令原子”误解为跨多条命令的整个 Java 业务也原子。

```text
GET
Java 计算
SET
```

这三步之间可能被其他请求插入。

---

# 五、什么是缓存命中和未命中

Java 读取：

```text
GET order:O1001:status
```

## 命中 Hit

Redis 中存在：

```text
返回 PAID
```

## 未命中 Miss

Redis 中不存在：

```text
返回空
  ↓
Java 查询 MySQL
  ↓
写回 Redis
```

这就是后面 Cache Aside 的基础。

---

# 六、Redis 为什么不能替代 MySQL

先记一句：

```text
MySQL 保存权威业务事实
Redis 保存热缓存和协调状态
```

| 数据 | 推荐事实源 |
|---|---|
| 用户、订单、支付 | MySQL |
| 文档正文 | MySQL / 对象存储 |
| 向量索引 | Milvus |
| 最近会话 | Redis |
| 工具结果短缓存 | Redis |
| 限流计数 | Redis |
| 任务租约 | Redis |
| 最终审计 | MySQL |
| 热 Checkpoint | Redis + 权威持久化 |

Redis 中 Key 不存在，可能是：

- 从未写入；
- TTL 到期；
- 被删除；
- 被淘汰；
- 故障切换丢失；
- 路由或网络问题。

因此不能把“Redis 没有”直接等同于“业务事实不存在”。

---

