# Redis 8.8 面试题 Q18

## Q18. 锁、Fencing Token 与幂等分别解决什么问题

**类型：资深高薪追问题｜难度：中等偏上｜重要性：★★★★★**

### ① 锁解决并发进入

锁的目标：

```text
尽量避免多个执行者同时进入关键区
```

它不能阻止一个已经失去锁的旧执行者继续运行。

### ② Fencing Token 解决旧执行者晚到

每次获得租约时领取一个单调递增编号：

```text
A 获得 token 41
锁过期
B 获得 token 42
```

下游资源记录已经接受的最大 Token。

A 的写晚到：

```text
41 < 42
```

被拒绝。

生活类比：

```text
旧排队号过期，柜台只接受更大的新号码
```

### ③ 幂等解决同一业务重复执行

同一个请求重复到达：

```text
requestId = E1001
```

无论执行一次还是多次，最终业务效果一致。

常用方案：

- 数据库唯一约束；
- 幂等表；
- 业务状态机；
- 版本条件更新；
- 外部 API Idempotency-Key；
- Inbox。

### ④ 三者怎样组合

Agent Tool 执行：

```text
锁
  → 减少同时执行

Fencing Token
  → 拒绝失去租约的旧 Worker 写状态

Idempotency Key
  → 防止外部 Tool 或业务结果重复
```

### ⑤ 一个完整时间线

```text
A 获得锁，token=41
A 调用 Tool，长时间卡住
锁过期
B 获得锁，token=42
B 完成并写状态
A 恢复，试图写结果
数据库发现 41 小于 42，拒绝
```

如果 Tool 调用本身已发生，还需要 Tool 侧用 requestId 去重。

### ⑥ 为什么只用分布式锁会出错

锁是有时间的租约，不是永久停止旧进程的物理开关。

JVM STW、网络分区、线程停顿都可能让旧持有者在租约失效后继续运行。

### ❌ 容易制造事故的写法

```java
// 认为只要拿过锁，后续写一定合法。
doLongWork();
repository.updateResult(runId, result); // 不校验租约代次
```

### ✅ 企业级改进示例

```java
int updated = repository.updateIfFenceIsNewer(
        runId, fencingToken, result);
if (updated == 0) {
    throw new StaleWorkerException();
}
// 外部调用同时携带稳定 requestId。
```

### 🎙️ 2～3 分钟优秀回答

锁、Fencing Token 和幂等解决不同层次的问题。锁用于减少多个执行者同时进入关键区；Fencing Token 是每次租约的单调递增编号，下游只接受更大的 Token，从而拒绝失去锁的旧执行者晚到写入；幂等保证同一个业务请求重复执行时，最终效果与一次相同。

例如 Worker A 获得 token 41 后卡住，锁过期；B 获得 token 42 并完成任务。A 恢复后写状态，数据库根据 token 拒绝 41。若外部 Tool 已被调用，则还要使用 requestId 作为幂等键，让 Tool 或本地 Inbox 去重。

所以项目中通常是锁减少并发、Fencing 防陈旧写、幂等控制重复副作用，三者不能互相替代。

### 面试官可能继续追问

- Fencing Token 必须由 Redis 生成吗？
- 数据库怎样原子拒绝较小 Token？

> **记忆句**：锁防同时进，Fencing 防旧人写，幂等防同一件事做两次。

---
