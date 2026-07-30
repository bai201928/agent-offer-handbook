# 九、一条 Java GET 怎样到达 Redis

Java：

```java
String value = commands.get("agent:run:R1:status");
```

完整画面：

```text
Java 业务代码
  ↓
Lettuce 把 GET 编码成网络字节
  ↓
TCP 连接
  ↓
Redis 读取并解析 GET
  ↓
在内存索引中查找 Key
  ↓
取出 Value
  ↓
构造回复
  ↓
网络返回
  ↓
Lettuce 解码
  ↓
Java 得到 RUNNING
```

以后遇到：

```text
事件循环
RESP
processCommand
call
dict
addReply
```

只需把它们放回这条链：

```text
收请求 → 解析 → 执行 → 找数据 → 回回复
```

---

# 十、连接是什么

Java 通过 TCP 连接与 Redis 通信。

类比电话线：

```text
Java 服务 ←连接→ Redis Server
```

连接不是越多越好。过多连接增加：

- 文件描述符；
- 服务端客户端状态；
- 缓冲；
- 调度；
- 网络压力。

Lettuce 的普通连接通常可线程安全复用；阻塞命令、Pub/Sub 和关键续租链路可按用途隔离。

---

# 十一、过期、删除和淘汰不要混淆

```text
过期：TTL 到了，业务保质期结束
删除：应用执行 DEL
淘汰：达到 maxmemory，为新数据腾空间
```

锁、幂等、限流和 Checkpoint 被淘汰，可能改变业务正确性。

所以：

```text
普通缓存
  → 可使用淘汰策略

协调状态
  → 独立容量池，常考虑 noeviction

关键事实
  → 不只放 Redis
```

---

# 十二、单机恢复与高可用第一张图

## RDB

```text
定期拍数据快照
```

## AOF

```text
记录写操作流水
```

## Replica

```text
Master 把写复制给备用节点
```

默认存在复制延迟，“有副本”不等于绝对不丢。

## Sentinel

```text
监控一套主从
  ↓
主节点故障
  ↓
选择新主并通知客户端
```

不负责数据分片。

## Cluster

```text
Key → Slot → 多个 Master
```

解决单节点容量与吞吐不足，但增加：

- Hash Tag；
- MOVED/ASK；
- 跨 Slot；
- 迁槽；
- Hot Slot。

---

