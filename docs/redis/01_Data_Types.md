# 七、六种高频数据类型

## 1. String：整体值或计数器

```redis
SET agent:run:R1:status RUNNING
INCR tenant:T1:request-count
```

适合：

- 状态；
- Token；
- 受控大小 JSON；
- 计数器；
- 小型工具结果。

错误做法：

```text
把完整对话、所有工具结果和日志塞进一个几十 MB JSON
```

## 2. Hash：可以按字段修改的对象

```redis
HSET agent:run:R1 status RUNNING owner worker-3 version 7
HGET agent:run:R1 status
HMGET agent:run:R1 status owner version
```

类比：

```text
String = 整张表单打包
Hash   = 每个字段可以单独读写
```

适合字段数量受控的小对象。

## 3. List：有顺序的一列数据

```redis
LPUSH recent:messages "你好"
LRANGE recent:messages 0 9
```

适合：

- 最近 N 条记录；
- 简单双端队列；
- 有边界历史。

需要 ACK、消费组和失败重投时，应学习 Stream 或专业 MQ。

## 4. Set：不重复的成员集合

```redis
SADD completed:tasks task-1
SISMEMBER completed:tasks task-1
```

适合：

- 去重；
- 标签；
- 成员判断；
- 已完成集合。

## 5. ZSet：成员带分数并排序

```redis
ZADD retry:tasks 1753873200 task-1
ZRANGE retry:tasks 0 -1 WITHSCORES
```

适合：

- 排行榜；
- 延迟任务；
- 优先级；
- 滑动窗口。

## 6. Stream：带消费进度的追加日志

```redis
XADD agent:events * runId R1 type TOOL_FINISHED
```

支持：

- 消费组；
- PEL；
- ACK；
- 重新认领；
- Redis 8.8 的 XNACK。

类比：

```text
List   = 普通包裹队列
Stream = 有负责人、签收和未签收记录的配送系统
```

---

# 八、数据类型选择表

| 需求 | 优先类型 |
|---|---|
| 整体值、计数 | String |
| 按字段读写 | Hash |
| 简单顺序列表 | List |
| 唯一成员 | Set |
| 分数排序 | ZSet |
| 消费组与 ACK | Stream |

选择顺序：

```text
访问模式
  ↓
数据类型
  ↓
元素和字节上限
  ↓
TTL
  ↓
内部编码
```

不是先背 listpack、quicklist 和跳表，再强行套业务。

---

