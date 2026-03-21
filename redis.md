## redis zset 实现原理
    - 压缩链表，跳表
## redis的nvm
- https://www.processon.com/view/link/61e5084a7d9c0806a8ab9829
## Redis在项目中的使用场景
- https://blog.csdn.net/agonie201218/article/details/123640871
## redis常用的数据结构
- string, hash , set zset, list
#### Redis 五种数据类型底层结构

---

#### 底层编码总览

| 类型 | 小数据编码 | 大数据编码 | 升级不可逆 |
|---|---|---|---|
| String | int / embstr | raw (SDS) | ✓ |
| Hash | listpack | hashtable | ✓ |
| List | listpack | quicklist | ✓ |
| Set | intset / listpack | hashtable | ✓ |
| ZSet | listpack | skiplist + hashtable | ✓ |

> 编码**只升不降**——一旦升级到大数据结构，即使删到只剩一个元素也不会退回 listpack，这是 Redis 的设计取舍。

---

#### String

Redis 的 String 底层是自研的 **SDS（Simple Dynamic String）**，而非 C 语言原生字符串。

**SDS 结构**

```
[ len | alloc | flags | buf[] ]
```

- `len`：已使用字节数，O(1) 获取长度
- `alloc`：总分配字节数
- `flags`：类型标识
- `buf[]`：实际字节数据（二进制安全，可含 `\0`）

SDS 相比 C 字符串的优势：O(1) 获取长度、二进制安全、空间预分配减少 realloc 次数。

**三种编码**

*int*

- 适用：值是整数且在 long 范围内（≤ 20 位）
- 直接将整数存在 `redisObject.ptr` 指针字段里，不额外分配内存

*embstr*

- 适用：短字符串，≤ 44 字节
- `redisObject` 和 SDS 在**一块连续内存**里，只需一次 `malloc` / `free`
- 只读，修改时会先转换为 raw

*raw（SDS）*

- 适用：长字符串，> 44 字节
- `redisObject` 和 SDS **分开分配**，两次 `malloc`
- 支持原地追加、空间预分配

---

#### Hash

**listpack（紧凑列表）**

- 触发条件：字段数 ≤ 128 且每个 value ≤ 64 字节
- key 和 value 交替存储在**一块连续内存**，无指针开销，内存极省
- 查找是 O(N) 线性扫描，数据量小时影响可忽略

内存布局：

```
[ total-bytes | last-offset | num-elements | entry | entry | ... | end ]
```

每个 entry 包含前一节点长度（用于反向遍历）、编码类型和实际数据。

**hashtable**

- 触发条件：字段数 > 128 或任意 value > 64 字节，自动升级
- 链式哈希表，平均 O(1) 读写
- 使用**渐进式 rehash**：扩容时维护新旧两张表，每次操作时迁移少量槽位，避免一次性阻塞

---

#### List

**listpack**

- 触发条件：元素数 ≤ 128 且每个元素 ≤ 64 字节（Redis 7.0+）
- 连续内存块存储，节省指针开销

**quicklist**

- 触发条件：超出 listpack 阈值后升级
- 本质是**双向链表 + 每个节点是一个 listpack**
- 兼顾了链表的 O(1) 头尾增删效率和 listpack 的内存紧凑性
- 可通过 `list-max-listpack-size` 配置每个节点的最大大小

结构示意：

```
head <-> [listpack] <-> [listpack] <-> [listpack] <-> tail
```

---

#### Set

Set 有三种编码，按条件依次选择。

**listpack**

- 触发条件：所有元素为字符串且元素数 ≤ 128（Redis 7.2+ 新增）
- 内存紧凑，小集合优先使用

**intset（整数集合）**

- 触发条件：所有元素均为整数 且 元素数 ≤ 512
- 有序整数数组，支持**二分查找**，内存极紧凑
- 支持 int16、int32、int64 三种编码，自动按需升级

结构：

```
[ encoding | length | contents[] ]
```

**hashtable**

- 触发条件：含非整数元素，或元素数超出阈值
- 升级后退化为普通哈希表，O(1) 判重 / 查找

---

#### ZSet（Sorted Set）

ZSet 是五种类型中结构最复杂的，大数据场景下**同时维护两个结构**。

**listpack**

- 触发条件：元素数 ≤ 128 且每个 member ≤ 64 字节
- member 和 score 成对交替存储，按 score 有序排列
- 查找 O(N)，数据量小时可接受

**skiplist + hashtable（双结构）**

超出 listpack 阈值后升级，两个结构**共享同一份 member 字符串对象**（引用计数），内存不翻倍。

*跳表（skiplist）*

- 多层有序链表，每层跨度不同
- 范围查询（`ZRANGE`、`ZRANGEBYSCORE`）：O(log N)
- 随机层数生成，平均每个节点约 1.33 个指针

```
L3: head ─────────────────────────────► node_D ► tail
L2: head ──────────► node_B ──────────► node_D ► tail
L1: head ► node_A ► node_B ► node_C ► node_D ► tail
```

*哈希表（hashtable）*

- member → score 的映射表
- 精确查找（`ZSCORE`、`ZRANK`）：O(1)
- 与跳表互补，两者配合覆盖所有查询场景

---

#### 阈值配置参考

以下参数均可在 `redis.conf` 中调整：

| 配置项 | 默认值 | 说明 |
|---|---|---|
| `hash-max-listpack-entries` | 128 | Hash 使用 listpack 的最大字段数 |
| `hash-max-listpack-value` | 64 | Hash 使用 listpack 的最大值字节数 |
| `list-max-listpack-size` | 128 | List 每个 quicklist 节点最大元素数 |
| `set-max-intset-entries` | 512 | Set 使用 intset 的最大元素数 |
| `set-max-listpack-entries` | 128 | Set 使用 listpack 的最大元素数（7.2+）|
| `zset-max-listpack-entries` | 128 | ZSet 使用 listpack 的最大元素数 |
| `zset-max-listpack-value` | 64 | ZSet 使用 listpack 的最大 member 字节数 |

## redis的zset实现
## Redis缓存淘汰策略
- [reference](https://www.processon.com/view/link/620b45476376897c8c7239d0)
## Redis主从复制原理
- [reference](https://www.processon.com/view/link/620b4875f346fb617416aed3)
## Redis怎么实现高可用
- [reference](https://www.processon.com/view/link/620b48f17d9c0807ec8cf49a)
## redis缓存雪崩、缓存穿透、缓存击穿
- [reference](https://www.processon.com/view/link/61e648387d9c0806a8b0cf29)

## redis string的编码方式
    - int 编码：保存long 型的64位有符号整数
    - embstr 编码：保存长度小于44字节的字符串
    - raw 编码：保存长度大于44字节的字符串
## redis的日志？主从同步用哪些？主从同步时候继续有数据写入怎么办？
    - AOF，RDB
    - RDB
    - replica_backoff_buffer(环形队列，可以被覆盖)
## redis的底层结构

- redis缓存雪崩、缓存穿透、缓存击穿？如何解决
    - 缓存穿透是指缓存和数据库中都没有的数据，而用户不断发起请求，如发起为id为“-1”的数据或id为特别大不存在的数据。这时的用户很可能是攻击者，攻击会导致数据库压力过大。

      - 接口层增加校验，如用户鉴权校验，id做基础校验，id<=0的直接拦截；
        从缓存取不到的数据，在数据库中也没有取到，这时也可以将key-value对写为key-null，缓存有效时间可以设置短点，如30秒（设置太长会导致正常情况也没法使用）。这样可以防止攻击用户反复用同一个id暴力攻击

    - 缓存击穿是指缓存中没有但数据库中有的数据（一般是缓存时间到期），这时由于并发用户特别多，同时读缓存没读到数据，又同时去数据库去取数据，引起数据库压力瞬间增大，造成过大压力

      - 设置热点数据永远不过期。
      - 加互斥锁

    - 缓存雪崩是指缓存中数据大批量到过期时间，而查询数据量巨大，引起数据库压力过大甚至down机。和缓存击穿不同的是，        缓存击穿指并发查同一条数据，缓存雪崩是不同数据都过期了，很多数据都查不到从而查数据库。
      缓存数据的过期时间设置随机，防止同一时间大量数据过期现象发生。
      如果缓存数据库是分布式部署，将热点数据均匀分布在不同搞得缓存数据库中。
      设置热点数据永远不过期。
- 那问你随机过期时间实现得不好就出现了大量key过期
- 关于分布式锁redis
    - 1. setnx + expire + del
    - 2. 如果expire时间过短，业务还没执行完锁失效了，那么别的请求可以共享资源了，该如何做？
    - 3. 如果expire时间过长，而业务执行完之后由于某种原因del lock失败，那么其他请求就获取不到锁了，该如何做？
    - 4. 如果业务还在执行，锁被别人del了，那么如何保证共享资源？
    - [参考答案](https://mp.weixin.qq.com/s/zwkK0YD6b94iwt_v36e-jw)
- redis里面有热点数据10w个。这时候一个程序员从数据库中捞了1000个新的数据返回，顶替了1000个热点数据（程序员用新的key塞入redis，导致redis中其他老的1000个key被删除）。用什么方式可以避免这样的情况发生？(字节面试题)
	
- redis 持久化有哪几种方式
- redis 集群有哪几种，redis集群怎么实现高可用
- redis怎么做高可用，机制
- 怎么保证redis和mysql的数据一致性
- redis的删除策略
- redis缓存穿透 如果用很多不存在的key攻击怎么办
    - 可以使用布隆过滤器，bitmap来实现
  
- redis sorted set score的范围是多少
    - (-2^53---- +2^53)

- redis数据结构, sort set 底层实现
    - [refer](https://processon.com/mindmap/60c09f127d9c087937196f50)
- redis 内存优化相关的设计
- 秒杀,促销设计
    - [refer](https://processon.com/mindmap/60f43a4c7d9c087bac5cd26f)

- redis如何保证lua脚本的一致性
    - 原子操作：Redis会将整个脚本作为一个整体执行，中间不会被其他进程或者进程的命令插入
    - [reference](https://segmentfault.com/a/1190000019676878)

### redis里面有热点数据10w个。这时候一个程序员从数据库中捞了1000个新的数据返回，顶替了1000个热点数据（程序员用新的key塞入redis，导致redis中其他老的1000个key被删除）。用什么方式可以避免这样的情况发生？
  - 参看答案： 利用lfu淘汰策略
  - 参考答案2： 冷热数据分离，redis机器分离

#### Redis 五种数据类型底层结构

---

#### 底层编码总览

| 类型 | 小数据编码 | 大数据编码 | 升级不可逆 |
|---|---|---|---|
| String | int / embstr | raw (SDS) | ✓ |
| Hash | listpack | hashtable | ✓ |
| List | listpack | quicklist | ✓ |
| Set | intset / listpack | hashtable | ✓ |
| ZSet | listpack | skiplist + hashtable | ✓ |

> 编码**只升不降**——一旦升级到大数据结构，即使删到只剩一个元素也不会退回 listpack，这是 Redis 的设计取舍。

---

#### String

Redis 的 String 底层是自研的 **SDS（Simple Dynamic String）**，而非 C 语言原生字符串。

**SDS 结构**

```
[ len | alloc | flags | buf[] ]
```

- `len`：已使用字节数，O(1) 获取长度
- `alloc`：总分配字节数
- `flags`：类型标识
- `buf[]`：实际字节数据（二进制安全，可含 `\0`）

SDS 相比 C 字符串的优势：O(1) 获取长度、二进制安全、空间预分配减少 realloc 次数。

**三种编码**

*int*

- 适用：值是整数且在 long 范围内（≤ 20 位）
- 直接将整数存在 `redisObject.ptr` 指针字段里，不额外分配内存

*embstr*

- 适用：短字符串，≤ 44 字节
- `redisObject` 和 SDS 在**一块连续内存**里，只需一次 `malloc` / `free`
- 只读，修改时会先转换为 raw

*raw（SDS）*

- 适用：长字符串，> 44 字节
- `redisObject` 和 SDS **分开分配**，两次 `malloc`
- 支持原地追加、空间预分配

---

#### Hash

**listpack（紧凑列表）**

- 触发条件：字段数 ≤ 128 且每个 value ≤ 64 字节
- key 和 value 交替存储在**一块连续内存**，无指针开销，内存极省
- 查找是 O(N) 线性扫描，数据量小时影响可忽略

内存布局：

```
[ total-bytes | last-offset | num-elements | entry | entry | ... | end ]
```

每个 entry 包含前一节点长度（用于反向遍历）、编码类型和实际数据。

**hashtable**

- 触发条件：字段数 > 128 或任意 value > 64 字节，自动升级
- 链式哈希表，平均 O(1) 读写
- 使用**渐进式 rehash**：扩容时维护新旧两张表，每次操作时迁移少量槽位，避免一次性阻塞

---

#### List

**listpack**

- 触发条件：元素数 ≤ 128 且每个元素 ≤ 64 字节（Redis 7.0+）
- 连续内存块存储，节省指针开销

**quicklist**

- 触发条件：超出 listpack 阈值后升级
- 本质是**双向链表 + 每个节点是一个 listpack**
- 兼顾了链表的 O(1) 头尾增删效率和 listpack 的内存紧凑性
- 可通过 `list-max-listpack-size` 配置每个节点的最大大小

结构示意：

```
head <-> [listpack] <-> [listpack] <-> [listpack] <-> tail
```

---

#### Set

Set 有三种编码，按条件依次选择。

**listpack**

- 触发条件：所有元素为字符串且元素数 ≤ 128（Redis 7.2+ 新增）
- 内存紧凑，小集合优先使用

**intset（整数集合）**

- 触发条件：所有元素均为整数 且 元素数 ≤ 512
- 有序整数数组，支持**二分查找**，内存极紧凑
- 支持 int16、int32、int64 三种编码，自动按需升级

结构：

```
[ encoding | length | contents[] ]
```

**hashtable**

- 触发条件：含非整数元素，或元素数超出阈值
- 升级后退化为普通哈希表，O(1) 判重 / 查找

---

#### ZSet（Sorted Set）

ZSet 是五种类型中结构最复杂的，大数据场景下**同时维护两个结构**。

**listpack**

- 触发条件：元素数 ≤ 128 且每个 member ≤ 64 字节
- member 和 score 成对交替存储，按 score 有序排列
- 查找 O(N)，数据量小时可接受

**skiplist + hashtable（双结构）**

超出 listpack 阈值后升级，两个结构**共享同一份 member 字符串对象**（引用计数），内存不翻倍。

*跳表（skiplist）*

- 多层有序链表，每层跨度不同
- 范围查询（`ZRANGE`、`ZRANGEBYSCORE`）：O(log N)
- 随机层数生成，平均每个节点约 1.33 个指针

```
L3: head ─────────────────────────────► node_D ► tail
L2: head ──────────► node_B ──────────► node_D ► tail
L1: head ► node_A ► node_B ► node_C ► node_D ► tail
```

*哈希表（hashtable）*

- member → score 的映射表
- 精确查找（`ZSCORE`、`ZRANK`）：O(1)
- 与跳表互补，两者配合覆盖所有查询场景

---

#### 阈值配置参考

以下参数均可在 `redis.conf` 中调整：

| 配置项 | 默认值 | 说明 |
|---|---|---|
| `hash-max-listpack-entries` | 128 | Hash 使用 listpack 的最大字段数 |
| `hash-max-listpack-value` | 64 | Hash 使用 listpack 的最大值字节数 |
| `list-max-listpack-size` | 128 | List 每个 quicklist 节点最大元素数 |
| `set-max-intset-entries` | 512 | Set 使用 intset 的最大元素数 |
| `set-max-listpack-entries` | 128 | Set 使用 listpack 的最大元素数（7.2+）|
| `zset-max-listpack-entries` | 128 | ZSet 使用 listpack 的最大元素数 |
| `zset-max-listpack-value` | 64 | ZSet 使用 listpack 的最大 member 字节数 |

#### Redis 大量 Key 过期但内存不释放

---

#### 过期键的删除机制

Redis 不会在 key 到期的瞬间立即回收内存，而是采用两种策略组合：

**惰性删除**

key 过期后不主动清理，等到下次**访问该 key 时**才检查是否过期，过期则删除并释放内存。好处是 CPU 开销极小，缺点是长期不访问的 key 会一直占用内存。

**定期删除**

Redis 默认每 100ms 执行一次过期扫描（由 `hz` 参数控制，默认 `hz=10`），每次从设置了过期时间的 key 中**随机抽取 20 个**检查，删除其中已过期的，若过期比例超过 25% 则继续抽查，直到比例低于 25% 或超过时间限制（默认 25ms）为止。

这意味着如果过期 key 数量极大，定期扫描**来不及全部清理**，大量 key 会在内存中滞留。

---

#### 内存不释放的根本原因

即使过期 key 被 Redis 逻辑删除了，内存也不一定立即归还给操作系统，原因在于 Redis 的内存分配器行为。

**内存分配器的内存池机制**

Redis 默认使用 **jemalloc** 作为内存分配器。jemalloc 维护自己的内存池，从操作系统申请大块内存后切分使用，key 被删除后内存归还给 jemalloc 的内存池，**而不是立刻还给操作系统**。

从操作系统视角（`top` / `free`）看，Redis 进程占用的 RSS（Resident Set Size）并不会立刻下降，但 Redis 内部的 `used_memory` 已经减少了。

**如何区分两者**

```bash
INFO memory

used_memory:         # Redis 实际使用内存（逻辑值）
used_memory_rss:     # 操作系统分配给 Redis 的物理内存（RSS）
mem_fragmentation_ratio: # RSS / used_memory，> 1.5 说明碎片严重
```

若 `used_memory` 已下降但 `used_memory_rss` 仍然很高，说明内存被 jemalloc 持有，尚未归还 OS，**这是正常现象**，不是内存泄漏。

---

#### 内存碎片加剧问题

大量 key 集中过期后，内存中会留下大量不连续的空洞，产生**内存碎片**：

- 新写入的 key 大小与空洞大小不匹配，jemalloc 不得不申请新内存
- `mem_fragmentation_ratio` 持续走高（> 1.5 需关注，> 2.0 需处理）
- 即使 `used_memory` 不高，RSS 依然居高不下

**碎片整理（Redis 4.0+）**

```bash
# 查看碎片率
INFO memory | grep mem_fragmentation_ratio

# 开启自动碎片整理
CONFIG SET activedefrag yes
CONFIG SET active-defrag-ignore-bytes 100mb   # 碎片超过 100MB 才整理
CONFIG SET active-defrag-threshold-lower 10   # 碎片率超过 10% 开始整理
CONFIG SET active-defrag-threshold-upper 100  # 碎片率超过 100% 全力整理
```

activedefrag 在后台增量整理，对性能影响较小，**推荐生产环境开启**。

---

#### 大量 Key 集中过期的问题

除了内存不释放，集中过期还会引发另一个问题：定期删除任务占用过多 CPU，导致 Redis 响应延迟上升（毫秒级卡顿）。

**根本原因**

定期删除每轮最多执行 25ms，若过期 key 过多，扫描循环持续触发，主线程被占用，正常命令处理延迟。

**避免方案：过期时间加随机抖动**

```python
# 不推荐：所有 key 同一时刻过期
redis.setex(key, 3600, value)

# 推荐：加随机抖动，打散过期时间
import random
ttl = 3600 + random.randint(0, 600)
redis.setex(key, ttl, value)
```

---

#### 排查与处理步骤总结

| 步骤 | 命令 / 操作 | 目的 |
|---|---|---|
| 1. 确认逻辑内存 | `INFO memory` 看 `used_memory` | 判断 Redis 是否真的占用多 |
| 2. 确认 RSS | `INFO memory` 看 `used_memory_rss` | 判断是否只是未归还 OS |
| 3. 查碎片率 | `mem_fragmentation_ratio` | 是否需要碎片整理 |
| 4. 开启 activedefrag | `CONFIG SET activedefrag yes` | 自动整理碎片 |
| 5. 调高 hz | `CONFIG SET hz 20` | 加快定期删除频率（适度） |
| 6. 检查 maxmemory 策略 | `CONFIG GET maxmemory-policy` | 确认淘汰策略是否合理 |
| 7. 新 key 加随机 TTL | 业务代码修改 | 避免下次集中过期 |
  - 淘汰策略为懒淘汰，修改淘汰策略


### redis内存很多碎片，导致内存居高不下，如何解决？
#### Redis 内存碎片导致内存居高不下的解决方案

---

#### 先确认是否真的是碎片问题

处理前先用 `INFO memory` 确认碎片率，避免误判：

```bash
127.0.0.1:6379> INFO memory

used_memory:          524288000   # Redis 逻辑使用内存（~500MB）
used_memory_rss:      1073741824  # OS 分配给 Redis 的物理内存（~1GB）
mem_fragmentation_ratio: 2.05     # RSS / used_memory，重点关注这个
```

**判断标准**

| 碎片率范围 | 状态 | 建议 |
|---|---|---|
| 1.0 ~ 1.5 | 正常 | 无需处理 |
| 1.5 ~ 2.0 | 偏高 | 观察 + 考虑开启 activedefrag |
| > 2.0 | 严重 | 需要主动处理 |
| < 1.0 | 内存换页 | 说明内存不足，Redis 数据被换到 swap |

---

#### 方案一：开启自动碎片整理（推荐，Redis 4.0+）

Redis 4.0 引入了 **activedefrag**（主动碎片整理），在后台增量搬移内存中的碎片，对性能影响可控。

**开启方式**

```bash
# 临时生效（重启失效）
CONFIG SET activedefrag yes

# 永久生效，写入 redis.conf
activedefrag yes
```

**关键参数调优**

```bash
# 碎片字节数超过 100MB 才开始整理（避免小碎片频繁触发）
CONFIG SET active-defrag-ignore-bytes 100mb

# 碎片率超过 10% 开始低强度整理
CONFIG SET active-defrag-threshold-lower 10

# 碎片率超过 100% 全力整理（CPU 占用上限）
CONFIG SET active-defrag-threshold-upper 100

# 整理过程中 CPU 使用率下限（低碎片时）
CONFIG SET active-defrag-cycle-min 1

# 整理过程中 CPU 使用率上限（高碎片时）
CONFIG SET active-defrag-cycle-max 25
```

**工作原理**

activedefrag 通过 jemalloc 的 API 识别碎片化的内存页，将存活对象复制到新的连续内存块，释放旧页归还给 OS。整个过程在主线程的空闲时间内**增量执行**，不会造成明显卡顿。

---

#### 方案二：重启 Redis 实例（彻底但有代价）

重启是最彻底的碎片清理方式，进程退出后 OS 直接回收全部内存，重启后从 RDB/AOF 加载数据，内存布局重新紧凑。

**操作步骤（主从架构下滚动重启）**

```bash
# 1. 先重启从节点，确认数据同步完整后再重启主节点
# 2. 重启前保存最新快照
redis-cli BGSAVE

# 3. 等待 BGSAVE 完成
redis-cli LASTSAVE   # 返回时间戳，前后对比确认已更新

# 4. 重启进程
systemctl restart redis
```

**注意事项**

- 单机无从节点时重启会有短暂不可用，需要业务侧做好降级
- 数据量大时 RDB 加载时间较长，需评估恢复时间（RTO）
- Cluster 模式下逐节点滚动重启，不影响整体可用性

---

#### 方案三：调整内存分配器为 libc（不推荐常规使用）

jemalloc 比 glibc 的 malloc 更擅长减少碎片，通常不建议换回 libc。但若 jemalloc 本身版本较旧，可以考虑升级 Redis 版本（Redis 自带的 jemalloc 版本会随版本迭代更新）。

```bash
# 查看当前使用的分配器
INFO memory | grep mem_allocator
# mem_allocator:jemalloc-5.3.0
```

升级 Redis 版本通常比换分配器更安全，新版 jemalloc 碎片整理能力更强。

---

#### 方案四：从源头减少碎片产生

碎片产生的根本原因是**频繁的内存分配和释放**，且新旧对象大小差异大。从业务层控制可以长期降低碎片率：

**避免大量小 key 频繁更新**

```bash
# 不推荐：每次更新一个小字段产生大量碎片
HSET user:1001 age 18
HSET user:1001 age 19
HSET user:1001 age 20

# 推荐：批量更新减少操作频次
HMSET user:1001 age 20 name "Alice" city "Beijing"
```

**控制 value 大小的一致性**

value 大小变化越剧烈，碎片越严重。同类数据尽量保持相近的大小，或者使用固定长度的序列化格式（如 protobuf）代替 JSON。

**大量 key 过期时间加随机抖动**

集中过期会造成内存空洞，打散过期时间可降低碎片产生速率：

```python
import random
ttl = 3600 + random.randint(0, 300)
redis.setex(key, ttl, value)
```

---

#### 处理流程总结

```
发现内存居高不下
       │
       ▼
INFO memory 查 mem_fragmentation_ratio
       │
  碎片率 > 1.5？
       │
      是 ──► 开启 activedefrag ──► 观察碎片率是否下降
       │                                    │
       │                              仍不下降
       │                                    │
       │                              ▼
       │                       评估是否可以重启
       │                       （主从滚动重启）
       │
      否 ──► 排查其他原因
             （内存泄漏 / maxmemory 未设置 / 数据量真的增长）
```

**优先级建议**：生产环境优先开启 `activedefrag`，观察 1～2 小时碎片率变化；若效果不明显且业务允许，再考虑滚动重启。

### 为啥RedisCluster设计成16384个槽
  - [refer](https://zhuanlan.zhihu.com/p/99037321)
### 100w数据，redis怎么取出来合适

#### 100 万数据从 Redis 取出的合适方案

---

#### 核心原则：绝对不能用 KEYS

```bash
# 高危操作，严禁在生产使用
KEYS *
KEYS user:*
```

`KEYS` 是 O(N) 全量扫描，执行期间**阻塞主线程**，100 万 key 的情况下耗时可达数百毫秒甚至数秒，期间所有请求全部排队等待，直接导致服务不可用。

---

#### 方案一：SCAN 游标迭代（通用首选）

`SCAN` 采用游标分批扫描，每次只处理少量 key，**不阻塞主线程**。

**基本用法**

```bash
# 第一次调用，游标传 0
SCAN 0 MATCH user:* COUNT 100

# 返回值：[下一个游标, [key列表]]
# 1) "68096"
# 2) 1) "user:1023"
#    2) "user:5091"
#    ...

# 用返回的游标继续扫描，直到游标返回 0 表示完成
SCAN 68096 MATCH user:* COUNT 100
```

**注意事项**

- `COUNT` 是**提示值**，不是精确返回数量，实际每次返回数量由 Redis 内部决定
- 结果可能有**重复**，业务侧需做去重处理
- 扫描过程中新增或删除的 key **不保证被扫到**，这是 SCAN 的设计取舍
- 不保证每次返回恰好 COUNT 个，小数据集时可能一次返回全部

**Python 示例（完整迭代）**

```python
import redis

r = redis.Redis()
cursor = 0
all_keys = []

while True:
    cursor, keys = r.scan(cursor, match="user:*", count=100)
    all_keys.extend(keys)
    if cursor == 0:
        break

# 去重
all_keys = list(set(all_keys))
print(f"共扫描到 {len(all_keys)} 个 key")
```

---

#### 方案二：SCAN + PIPELINE 批量取值（高吞吐）

光拿到 key 列表还不够，如果逐个 `GET` 会产生 100 万次网络往返，需要用 **Pipeline** 合并请求。

```python
import redis

r = redis.Redis()
cursor = 0
result = {}

while True:
    cursor, keys = r.scan(cursor, match="user:*", count=500)
    
    if keys:
        # 批量取值，一次 pipeline 发送，减少网络往返
        pipe = r.pipeline()
        for key in keys:
            pipe.get(key)
        values = pipe.execute()
        
        for key, value in zip(keys, values):
            if value is not None:
                result[key] = value
    
    if cursor == 0:
        break

print(f"共取出 {len(result)} 条数据")
```

**Pipeline 的效果**

| 方式 | 100 万次请求网络开销 | 说明 |
|---|---|---|
| 逐条 GET | ~100 万次 RTT | 极慢，不可用 |
| Pipeline（每批 500） | ~2000 次 RTT | 提升约 500 倍 |
| Pipeline（每批 1000） | ~1000 次 RTT | 进一步提升 |

每批大小建议 **500～1000**，过大会导致单次响应包过大，占用内存和网络带宽。

---

#### 方案三：数据结构决定取法

如果 100 万数据是存在同一个集合类型中，而非散落的独立 key，取法完全不同。

**Hash（大 Hash 拆分取）**

```bash
# 不能用 HGETALL（100万字段一次返回，内存爆炸）
HGETALL big_hash   # 危险

# 用 HSCAN 分批取
HSCAN big_hash 0 COUNT 500
```

**Set**

```bash
# 禁止
SMEMBERS big_set   # 危险

# 用 SSCAN
SSCAN big_set 0 COUNT 500
```

**ZSet**

```bash
# 按 score 范围分批取，天然支持分页
ZRANGEBYSCORE leaderboard -inf +inf LIMIT 0 500
ZRANGEBYSCORE leaderboard -inf +inf LIMIT 500 500
# ...

# 或用 ZSCAN
ZSCAN big_zset 0 COUNT 500
```

**List**

```bash
# 用 LRANGE 分页，O(S+N)，S 是偏移量，深度分页性能下降
LRANGE mylist 0 499
LRANGE mylist 500 999
# ...
```

---

#### 方案四：数据导出场景用 RDB / AOF 解析

如果目标是**全量数据导出**（如迁移、备份分析），不要走 Redis 协议，直接解析持久化文件效率更高，完全不影响线上服务。

```bash
# 触发 RDB 快照
redis-cli BGSAVE

# 用 rdb-tools 解析 RDB 文件
pip install rdbtools
rdb --command json /var/lib/redis/dump.rdb > data.json

# 或用 redis-rdb-cli（更快，支持过滤）
rct -f dump -s /var/lib/redis/dump.rdb -o ./output -t string
```

适合场景：数据迁移、离线分析、全量备份，**不适合实时业务读取**。

---

#### 方案五：Cluster 模式下的特殊处理

Cluster 模式下 key 分散在多个节点，单节点 SCAN 只能扫本节点的 key，需要对每个主节点分别执行 SCAN。

```python
from rediscluster import RedisCluster

startup_nodes = [{"host": "127.0.0.1", "port": "7000"}]
rc = RedisCluster(startup_nodes=startup_nodes)

all_keys = []
# RedisCluster 的 scan_iter 自动对所有主节点执行 SCAN
for key in rc.scan_iter(match="user:*", count=500):
    all_keys.append(key)
```

或者手动获取所有主节点地址，逐节点并发执行 SCAN，最后合并结果。

---

#### 方案对比与选型

| 方案 | 适用场景 | 是否阻塞 | 复杂度 |
|---|---|---|---|
| SCAN + PIPELINE | 通用，线上实时取 | 否 | 低 |
| HSCAN / SSCAN / ZSCAN | 大集合类型分批取 | 否 | 低 |
| LRANGE 分页 | List 顺序读取 | 否 | 低 |
| RDB 文件解析 | 全量导出 / 迁移 | 否（离线） | 中 |
| KEYS（禁止） | — | **是** | — |

**生产环境核心原则**：所有涉及大量 key 的操作，一律走 `SCAN` 系列命令，配合 `PIPELINE` 批量取值，单批控制在 500～1000 条，避免任何一次性全量命令。

### 网关集群发生脑裂了，你的hash环会受到什么影响，怎么解决（也许不是redis相关问题，redis也有脑裂的情况）
  - 参看答案： 网关集群发生脑裂了，你的hash环会受到影响，因为脑裂后，有一半的节点会认为自己是主节点，而另一半会认为自己是从节点，这就会导致hash环被分成两半，一半的请求会被路由到从节点，另一半会被路由到主节点，这就会导致数据不一致的问题。
  - 参考答案： 可以通过配置哨兵集群来解决这个问题，当哨兵集群检测到主节点挂掉后，会自动选举一个从节点作为新的主节点，同时会更新hash环，确保所有节点都能正常路由请求。
### redis哈希和集合有啥关系

#### Redis 哈希（Hash）和集合（Set）的关系

---

#### 表面上看：两者完全不同

| 对比点 | Hash | Set |
|---|---|---|
| 存储内容 | field-value 键值对 | 无序唯一元素 |
| 典型命令 | `HSET` `HGET` `HMGET` | `SADD` `SMEMBERS` `SISMEMBER` |
| 使用场景 | 对象属性存储 | 去重、标签、关系 |

从业务语义上两者没有直接关联，但在**底层实现**上有非常深的关系。

---

#### 本质关系：共用同一套底层编码

**小数据阶段：两者都用 listpack**

```
Hash（小）:  [ field1 | value1 | field2 | value2 | ... ]  ← listpack
Set（小）:   [ elem1  | elem2  | elem3  | ...          ]  ← listpack
```

两者在数据量小时都退化为同一块连续内存结构，差别只是 Hash 里两个相邻 entry 是 field-value 对，Set 里每个 entry 是独立元素。

**大数据阶段：都升级为 hashtable**

```
Hash（大）:  hashtable，key=field，value=value
Set（大）:   hashtable，key=elem， value=NULL
```

Set 的 hashtable 只用了 key，value 一律为 NULL。**Set 本质上就是一个 value 全为空的 Hash**，这不是巧合，是 Redis 的刻意设计。

---

#### 从源码角度看两者的关系

Redis 内部 `redisObject` 结构相同，编码类型（encoding）决定底层实现：

```c
// Hash 的编码演进
OBJ_ENCODING_LISTPACK  →  OBJ_ENCODING_HT

// Set 的编码演进
OBJ_ENCODING_LISTPACK  →  OBJ_ENCODING_INTSET  →  OBJ_ENCODING_HT
```

升级为 `OBJ_ENCODING_HT` 后，Hash 和 Set **共用同一个 dict（字典）结构**，区别仅在于：

```c
// Hash 存储
dictAdd(dict, field, value);   // field → value

// Set 存储
dictAdd(dict, elem, NULL);     // elem  → NULL（value 位置为空）
```

---

#### Set 比 Hash 多一种编码：intset

Set 独有的 intset 是 Hash 没有的，这是两者底层的唯一差异点：

```
Set 全为整数时：  intset（有序整数数组，二分查找）
Hash 无论如何：   不会使用 intset
```

intset 是 Set 专属的内存优化，利用整数的有序性做二分查找，比 listpack 更省内存、查找更快。

---

#### 编码升级触发条件对比

**Hash 升级为 hashtable**

```
字段数    > hash-max-listpack-entries（默认 128）
          或
任意 value > hash-max-listpack-value（默认 64 字节）
```

**Set 升级为 hashtable**

```
含非整数元素时：listpack 元素数 > set-max-listpack-entries（默认 128）
全为整数时：   intset 元素数   > set-max-intset-entries（默认 512）
```

---

#### 一个有趣的实验

可以用这个实验直观感受两者的底层同源性：

```bash
# 创建一个小 Hash
HSET myhash a 1 b 2 c 3
OBJECT ENCODING myhash   # → listpack

# 创建一个小 Set（非整数）
SADD myset a b c
OBJECT ENCODING myset    # → listpack

# 两者编码完全相同！

# 扩大数据量触发升级
# Hash 升级
HSET bighash $(python3 -c "print(' '.join([f'f{i} v{i}' for i in range(200)]))")
OBJECT ENCODING bighash  # → hashtable

# Set 升级（非整数）
# 添加 200 个字符串元素后
OBJECT ENCODING bigset   # → hashtable

# Set 全整数时
SADD intset 1 2 3 4 5
OBJECT ENCODING intset   # → intset  ← Set 独有
```

---

#### 关系总结

```
             小数据              大数据
Hash:    listpack          →    hashtable（field→value）
                                              ↑
Set:     listpack/intset   →    hashtable（elem→NULL）
                                    共用同一个 dict 结构
```

两者的关系可以用一句话概括：**Set 是 value 全为 NULL 的特殊 Hash**，小数据共用 listpack，大数据共用 hashtable，Set 额外拥有针对纯整数场景优化的 intset 编码。理解这层关系，有助于在选型时做出更合适的决策——需要存 field-value 对用 Hash，只需要存唯一值用 Set，底层开销本质相同。

### redis/etcd使用ap还是cp

#### Redis / etcd 使用 AP 还是 CP

---

#### 先回顾 CAP 定理

分布式系统中三者不可兼得，只能选两个：

| 字母 | 含义 | 说明 |
|---|---|---|
| C | Consistency（一致性） | 所有节点在同一时刻看到的数据相同 |
| A | Availability（可用性） | 每个请求都能收到响应（不保证最新数据） |
| P | Partition tolerance（分区容错） | 网络分区时系统仍能继续运行 |

网络分区（P）在分布式系统中是**必须容忍的客观存在**，所以实际选择只在 CP 和 AP 之间。

---

#### etcd：CP 系统

etcd 是**强一致性优先**，明确选择 CP。

**实现基础：Raft 共识算法**

```
Leader
  │
  ├─► Follower 1  ─┐
  ├─► Follower 2  ─┼─ 超过半数节点写入成功才返回客户端
  └─► Follower 3  ─┘
```

- 所有写操作必须经过 Leader，Leader 将日志复制给 Follower
- 超过半数节点（quorum）确认后才返回写入成功
- 任何时刻读 Leader 数据**保证是最新的**（线性一致性）

**发生网络分区时的行为**

```
[Leader + 少数节点]    |    [多数节点]
                       ↑
                    网络分区

少数节点分区：Leader 无法获得 quorum → 拒绝写入 → 牺牲可用性
多数节点分区：重新选举 Leader → 恢复服务
```

少数派分区宁可拒绝服务，也不写入可能不一致的数据，**C 优先于 A**。

**适用场景**

etcd 正是因为 CP 特性，才适合做：服务注册发现、分布式锁、配置中心、Kubernetes 的元数据存储。这些场景要求数据绝对可信，不能读到旧数据。

---

#### Redis：整体偏 AP，但要分场景

Redis 没有一个统一的 CAP 定性，**不同部署模式下取舍不同**。

**单机 Redis：CAP 不适用**

CAP 讨论的是分布式场景，单机不存在网络分区问题，不在 CAP 讨论范围内。

**主从 + 哨兵模式：AP**

Redis 主从复制是**异步的**：

```
Client → Master（写入成功，立即返回）
              │
              └─► Slave（异步复制，可能有延迟）
```

- 主节点写入成功即返回，不等从节点确认 → **优先可用性**
- 主节点宕机、哨兵切换期间（15～30 秒）可能丢失部分数据 → **不保证强一致性**
- 脑裂场景下新旧主节点同时写入，数据会冲突 → **一致性进一步削弱**

*脑裂问题及缓解*

```bash
# 限制主节点：至少有 N 个从节点且复制延迟不超过 M 秒才接受写入
# 牺牲部分可用性换取一定的一致性保障
min-replicas-to-write 1
min-replicas-max-lag 10
```

**Redis Cluster：AP 为主，局部 CP**

```
[Slot 0-5460]   [Slot 5461-10922]   [Slot 10923-16383]
  Master A  ←→    Master B      ←→    Master C
  Slave  A'        Slave  B'           Slave  C'
```

- 同样是异步复制，分区时优先保证可用性 → **整体 AP**
- 但分区节点超过半数宕机时，该分片拒绝服务 → **局部 CP 行为**
- 跨槽事务不支持，一致性保证更弱

---

#### 两者对比

| 对比点 | Redis（主从/Cluster） | etcd |
|---|---|---|
| CAP 定位 | **AP**（可用性优先） | **CP**（一致性优先） |
| 复制方式 | 异步复制（主从）<br>异步复制（Cluster） | 同步复制 |
| 场景 | 一般场景（读写分离、缓存等） | 服务注册发现、配置中心等 |
| 数据一致性 | 最终一致性（主从）<br>局部 CP（Cluster） | 强一致性 |
| 分区容忍 | 必须容忍（P） | 必须容忍（P） |

---

#### 总结

- **Redis** 更偏 AP，整体可用性优先，局部 CP 行为（分区时拒绝服务）
- **etcd** 更偏 CP，一致性优先，分区时拒绝写入
- 选择时需根据业务场景**是否需要强一致性**来判断

---

### redis可以作为消息队列使用吗？如何防止消息丢失
    - 广播订阅模式：基于Redis的 Pub/Sub 机制，一旦有客户端往某个key里面publish一个消息，所有subscribe的客户端都会触发事件
    - 集群订阅模式：基于Redis List双向+ 原子性 + BRPOP
    - 如何避免消息丢失
        - 写入时候要求启用事务处理，保证写一定成功
        - redis配置成任何变更一定实时持久化，比如存储端是磁盘的话，每次变更马上同步写入磁盘，才算完成
        - 消费端也要实现事务方式，处理完成后，再回来真实删除消息
        - 多线程或者多端同时并发处理，可以通过锁的方式来规避
        - [refer](https://www.yisu.com/zixun/117203.html)

#
- Redis 如何实现的分布式锁？ setnx 、 redisson ，为什么要用 redisson ? watchdog "如果让你基于 setnx 实现一个 watchdog 怎么做？怎么做 watch 呢？

#
- 怎么样基于redis存储文件的
- redis的RDB瘦身你是怎么做的，key是不过期吗？

#
- sortset数据量特别大的时候,有什么问题?

#
- sortset数据插入时,时间复杂度是多少?
    - O(LogN)

#
- zset 为啥不用红黑树,二叉树,为什么用跳跃表?基于什么原因?

#
- redis key的过期方式

#
- lru的特点

#
- redis是怎么记录lru的

# redis zset为什么用跳表，不用avl，红黑树？
```md
从内存占用上来比较，跳表比平衡树更灵活一些。平衡树每个节点包含 2 个指针（分别指向左右子树），而跳表每个节点包含的指针数目平均为 1/(1-p)，具体取决于参数 p 的大小。如果像 Redis里的实现一样，取 p=1/4，那么平均每个节点包含 1.33 个指针，比平衡树更有优势。
在做范围查找的时候，跳表比平衡树操作要简单。在平衡树上，我们找到指定范围的小值之后，还需要以中序遍历的顺序继续寻找其它不超过大值的节点。如果不对平衡树进行一定的改造，这里的中序遍历并不容易实现。而在跳表上进行范围查找就非常简单，只需要在找到小值之后，对第 1 层链表进行若干步的遍历就可以实现。
从算法实现难度上来比较，跳表比平衡树要简单得多。平衡树的插入和删除操作可能引发子树的调整，逻辑复杂，而跳表的插入和删除只需要修改相邻节点的指针，操作简单又快速
```
- https://www.cnblogs.com/jajian/p/16801106.html