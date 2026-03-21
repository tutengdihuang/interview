## etcd 怎么实现分布式锁

### 核心机制：Lease + Put + Watch

**1. 创建 Lease（租约）**

```go
lease, _ := client.Grant(ctx, 10) // TTL = 10秒
```

Lease 相当于一个"计时器"，持有锁的节点必须持续续约，否则锁自动释放，避免死锁。

**2. Put with Lease（原子写入）**

```go
client.Put(ctx, "/lock/my-service", nodeID, clientv3.WithLease(lease.ID))
```

**3. 用 Prefix + Revision 排队**

etcd 分布式锁的生产实现（`concurrency` 包）不是简单抢占，而是排队：

```
所有竞争者写入同一前缀：
/lock/my-service/0000000001  ← 最小 revision，获得锁
/lock/my-service/0000000002  ← Watch 前一个 key
/lock/my-service/0000000003  ← Watch 前一个 key
```

每个竞争者只 Watch 自己前一个 key，前一个 key 删除时，自己获得锁。

**4. 释放锁**

```go
client.Revoke(ctx, lease.ID) // 撤销 Lease，key 自动删除
```

---

## etcd 的原理

### 整体架构

```
Client
  │
  ▼
gRPC API 层
  │
  ▼
Raft 共识层  ←── 保证多节点数据一致
  │
  ▼
WAL（预写日志）
  │
  ▼
boltdb（持久化 KV 存储）
```

### 数据模型：MVCC

etcd 每次写入不覆盖旧值，而是追加新版本：

```
key: /foo
  revision=1 → "bar"
  revision=5 → "baz"   ← 当前值
  revision=9 → "qux"
```

好处：Watch 可以从任意历史版本开始监听，不会错过事件。

### Watch 机制

```
Client ──Watch("/foo")──► etcd
                            │
其他客户端写入 /foo          │
                            ▼
                      推送事件给 Watcher（基于 revision 有序推送）
```

Watch 是长连接 + 服务端推送，不是轮询。

### Lease（租约）

- 每个 Lease 有 TTL
- 客户端需周期性 `KeepAlive`
- Lease 过期 → 关联的所有 key 自动删除

---

## CAP 理论

### 三个核心概念

| 字母 | 含义 | 解释 |
|---|---|---|
| C | Consistency（一致性） | 所有节点同一时刻看到相同的数据 |
| A | Availability（可用性） | 每个请求都能收到响应（不保证最新） |
| P | Partition Tolerance（分区容忍） | 网络分区时系统仍能运行 |

### 核心结论

**P 在分布式系统中必须满足**（网络故障是常态），所以实际只能在 C 和 A 之间取舍：

```
网络分区发生时：

CP 系统（如 etcd、ZooKeeper）：
  拒绝服务 / 返回错误  ← 保证一致性，牺牲可用性

AP 系统（如 Cassandra、DynamoDB）：
  返回可能过时的数据  ← 保证可用性，牺牲一致性
```

### etcd 属于 CP

节点数 < quorum 时，etcd 拒绝写入，保证不出现脑裂。

---

## Raft 原理

### 三种角色

```
Leader   ← 唯一处理写请求，定期发心跳
Follower ← 被动接收日志，响应心跳
Candidate← 选举中的临时状态
```

### Leader 选举

```
1. Follower 超时未收到心跳 → 转为 Candidate
2. Candidate 自增 Term，向所有节点发 RequestVote
3. 获得多数票（quorum）→ 成为新 Leader
4. 其余节点收到新 Leader 心跳 → 退回 Follower
```

**Term（任期）** 是逻辑时钟，每次选举 +1，用于识别过期消息。

### 日志复制

```
Client
  │ 写请求
  ▼
Leader  ──AppendEntries──►  Follower1  ✅
        ──AppendEntries──►  Follower2  ✅
        ──AppendEntries──►  Follower3  ❌（网络问题）

多数节点确认 → Leader Commit → 返回客户端成功
```

**只要多数节点（n/2 + 1）写入成功，即可提交**，少数节点落后不影响整体。

### 日志一致性保证

- Leader 永不覆盖自己的日志，只追加
- Follower 日志与 Leader 冲突时，强制以 Leader 为准
- 已 Commit 的日志永远不会丢失

### Raft vs Paxos

| | Raft | Paxos |
|---|---|---|
| 可理解性 | 高，角色清晰 | 低，论文晦涩 |
| Leader | 强 Leader | 无固定 Leader |
| 工程实现 | etcd、TiKV | Chubby |