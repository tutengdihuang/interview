## 并发控制
### CHANNEL 的RING BUFFER 实现 
#### Channel 的 Ring Buffer 实现

---

#### 1. Channel 的底层结构 hchan

```go
type hchan struct {
    qcount   uint           // 当前队列中的元素个数
    dataqsiz uint           // 环形队列的容量（make 时指定）
    buf      unsafe.Pointer // 指向环形缓冲区的指针
    elemsize uint16         // 每个元素的大小
    closed   uint32         // 是否已关闭
    elemtype *_type         // 元素类型

    sendx uint  // 发送索引（写指针）
    recvx uint  // 接收索引（读指针）

    recvq waitq // 阻塞的接收者队列（sudog 链表）
    sendq waitq // 阻塞的发送者队列（sudog 链表）

    lock mutex  // 互斥锁，保护所有字段
}
```

---

#### 2. Ring Buffer 的内存布局

```
make(chan int, 6)  →  创建容量为 6 的环形缓冲区

buf 指向的内存：
┌─────┬─────┬─────┬─────┬─────┬─────┐
│  0  │  1  │  2  │  3  │  4  │  5  │
└─────┴─────┴─────┴─────┴─────┴─────┘
   ↑                          
 recvx=0                      
 sendx=0                      
 qcount=0                     

发送 3 个元素后（A, B, C）：
┌─────┬─────┬─────┬─────┬─────┬─────┐
│  A  │  B  │  C  │     │     │     │
└─────┴─────┴─────┴─────┴─────┴─────┘
   ↑              ↑
 recvx=0        sendx=3
 qcount=3
```

---

#### 3. 发送数据（写指针 sendx 移动）

```go
ch <- val
```

```
写入流程：

① 加锁
② 将 val 拷贝到 buf[sendx] 位置
③ sendx = (sendx + 1) % dataqsiz   ← 取模实现环形
④ qcount++
⑤ 解锁

示意：
┌─────┬─────┬─────┬─────┬─────┬─────┐
│  A  │  B  │  C  │  D  │     │     │
└─────┴─────┴─────┴─────┴─────┴─────┘
   ↑                   ↑
 recvx=0             sendx=4
 qcount=4
```

---

#### 4. 接收数据（读指针 recvx 移动）

```go
val := <-ch
```

```
读取流程：

① 加锁
② 从 buf[recvx] 位置拷贝数据到 val
③ 清空 buf[recvx]（GC 友好）
④ recvx = (recvx + 1) % dataqsiz   ← 取模实现环形
⑤ qcount--
⑥ 解锁

接收 A、B 后：
┌─────┬─────┬─────┬─────┬─────┬─────┐
│     │     │  C  │  D  │     │     │
└─────┴─────┴─────┴─────┴─────┴─────┘
               ↑              ↑
            recvx=2         sendx=4
            qcount=2
```

---

#### 5. 环形特性：指针绕回

```
容量为 6，当 sendx 到达末尾后自动绕回头部：

继续发送 E、F、G：
┌─────┬─────┬─────┬─────┬─────┬─────┐
│  G  │     │  C  │  D  │  E  │  F  │
└─────┴─────┴─────┴─────┴─────┴─────┘
   ↑    ↑      ↑
sendx=1       recvx=2
qcount=5

sendx 从 5 → (5+1)%6 = 0，绕回到头部写入 G
读取顺序依然是 C → D → E → F → G（FIFO 保证）
```

---

#### 6. 缓冲区满/空时的行为

```
缓冲区判断条件：

  空：qcount == 0
  满：qcount == dataqsiz

┌────────────┬────────────────────────────────────────┐
│   状态      │              行为                       │
├────────────┼────────────────────────────────────────┤
│ 发送时缓冲满 │ G 挂入 sendq，M/P 调度其他 G，等待唤醒  │
│ 接收时缓冲空 │ G 挂入 recvq，M/P 调度其他 G，等待唤醒  │
│ 缓冲未满    │ 数据直接写入 buf，发送方不阻塞            │
│ 缓冲非空    │ 数据直接从 buf 读取，接收方不阻塞         │
└────────────┴────────────────────────────────────────┘
```

---

#### 7. 发送方唤醒接收方的优化路径

```
存在阻塞的接收者（recvq 非空）时，跳过 Ring Buffer 直接交付：

正常路径：sender → buf → receiver（两次拷贝）

优化路径：sender → receiver（一次拷贝，零 buf 经过）
  ↓
从 recvq 取出等待的 G
直接将数据拷贝到该 G 的栈上
唤醒该 G，放入 P 的运行队列
```

---

#### 总结

| 字段 | 作用 |
|------|------|
| `buf` | 环形缓冲区内存起始地址 |
| `sendx` | 下一次写入的位置，满时阻塞 |
| `recvx` | 下一次读取的位置，空时阻塞 |
| `qcount` | 当前元素数，判断空/满 |
| `dataqsiz` | 缓冲区容量，取模实现环形 |

> Ring Buffer 的本质：**用 `(index + 1) % cap` 的取模运算，让线性数组首尾相连，实现高效的 FIFO 无锁化读写。**


### MUTEX 几种状态
#### Mutex 的几种状态

---

#### 1. Mutex 的底层结构

```go
type Mutex struct {
    state int32  // 状态字段（多个标志位压缩在一个 int32 中）
    sema  uint32 // 信号量，用于阻塞/唤醒 goroutine
}
```

`state` 字段的位布局：

```
 int32 (32 bit)
┌─────────────────────────────┬──────────┬─────────┬────────┐
│     waiters count（29位）    │ starving │ woken   │ locked │
│         等待者数量            │  饥饿位  │  唤醒位  │ 锁定位  │
└─────────────────────────────┴──────────┴─────────┴────────┘
  bit[31:3]                     bit[2]     bit[1]    bit[0]
```

---

#### 2. 三个核心状态位

```go
const (
    mutexLocked      = 1 << 0  // 0001  锁是否被持有
    mutexWoken       = 1 << 1  // 0010  是否有被唤醒的 G
    mutexStarving    = 1 << 2  // 0100  是否处于饥饿模式
)
```

| 状态位 | 值 | 含义 |
|--------|----|------|
| `mutexLocked` | `1` | 锁已被某个 G 持有 |
| `mutexWoken` | `2` | 有 G 被唤醒，正在尝试获取锁 |
| `mutexStarving` | `4` | 锁处于饥饿模式 |

---

#### 3. Locked 状态（锁定）

```
mutexLocked = 1

state: ...0001

含义：锁已被持有，其他 G 无法直接获取

变化：
  Lock()   → state |= mutexLocked    （加锁）
  Unlock() → state &^= mutexLocked   （解锁）

┌─────────┐   Lock()    ┌─────────┐
│ unlocked│ ──────────→ │ locked  │
└─────────┘             └─────────┘
```

---

#### 4. Woken 状态（唤醒）

```
mutexWoken = 2

state: ...0010

含义：已有一个自旋的 G 被唤醒，Unlock 时不必再额外唤醒新的 G，避免多余唤醒

触发场景：
  正常模式下，G 从 sema 队列被唤醒后
  设置 mutexWoken 告知解锁方"已有人在路上了"

┌──────────────────────────────────────────┐
│  Unlock 判断 woken 位                     │
│  woken = 1 → 不唤醒新 G，减少无效竞争     │
│  woken = 0 → 从等待队列唤醒一个 G         │
└──────────────────────────────────────────┘
```

---

#### 5. Starving 状态（饥饿）

```
mutexStarving = 4

state: ...0100

触发条件：某个 G 等待锁超过 1ms，进入饥饿模式

正常模式 vs 饥饿模式对比：

┌──────────────┬───────────────────────────────────────────┐
│   正常模式    │ 新来的 G 可自旋抢锁，吞吐高但可能饿死老 G  │
├──────────────┼───────────────────────────────────────────┤
│   饥饿模式    │ 锁直接交给等待队列队头的 G，严格 FIFO      │
│              │ 新来的 G 不允许自旋，直接入队               │
└──────────────┴───────────────────────────────────────────┘

退出饥饿模式的条件（满足任一）：
  ① 当前 G 等待时间 < 1ms
  ② 当前 G 是等待队列中最后一个
```

---

#### 6. 状态组合与流转

```
初始状态：state = 0（未锁定、未唤醒、非饥饿）

正常加锁流程：
  state = 0000 → Lock() → state = 0001（Locked）

有竞争时：
  state = 0001 → 新 G 自旋等待 → state = 0011（Locked + Woken）

等待超时触发饥饿：
  state = 0011 → 等待 > 1ms → state = 0101（Locked + Starving）

解锁并唤醒：
  state = 0101 → Unlock() → 直接移交给队头 G

完整状态流转图：

  [0000] 未加锁
     ↓ Lock()
  [0001] 已加锁
     ↓ 有竞争+自旋
  [0011] 已加锁 + 有G被唤醒
     ↓ 等待超过1ms
  [0111] 已加锁 + 唤醒 + 饥饿
     ↓ Unlock()
  [0100] 饥饿模式下移交锁
     ↓ 队列清空/等待<1ms
  [0000] 回到初始状态
```

---

#### 7. waiters 等待者计数

```go
// 等待者数量存储在 state 高 29 位
waiters := state >> mutexWaiterShift  // mutexWaiterShift = 3

每次 G 进入阻塞：state += 1 << 3  （waiters++）
每次 G 被唤醒：  state -= 1 << 3  （waiters--）

作用：
  Unlock 时判断是否有等待者需要唤醒
  饥饿模式判断是否是最后一个等待者
```

---

#### 总结

| 状态 | 位 | 触发条件 | 作用 |
|------|----|---------|------|
| `Locked` | bit0 | `Lock()` 成功 | 标记锁被持有 |
| `Woken` | bit1 | G 被从 sema 唤醒 | 避免 Unlock 重复唤醒 |
| `Starving` | bit2 | 等待超过 1ms | 切换 FIFO 公平模式，防止饿死 |
| `waiters` | bit3~31 | 每次 G 阻塞入队 | 记录等待人数，决定是否唤醒 |

> Mutex 的精髓：**用一个 int32 的不同位，同时表达锁状态、唤醒信号、饥饿模式和等待人数，兼顾高吞吐（自旋）与公平性（饥饿模式）。**


### MUTEX 正常模式和饥饿模式
#### Mutex 正常模式和饥饿模式

---

#### 1. 两种模式的核心区别

```
┌──────────────┬──────────────────────────┬──────────────────────────┐
│              │        正常模式            │        饥饿模式            │
├──────────────┼──────────────────────────┼──────────────────────────┤
│ 锁的分配方式  │ 新 G 和等待队列竞争        │ 直接交给队头 G             │
│ 新来的 G     │ 允许自旋抢锁               │ 直接排队，禁止自旋          │
│ 公平性       │ 弱（新 G 更容易抢到）       │ 强（严格 FIFO）            │
│ 吞吐量       │ 高                        │ 低                        │
│ 延迟         │ 可能很高（老 G 饿死）       │ 可预期（有序排队）          │
└──────────────┴──────────────────────────┴──────────────────────────┘
```

---

#### 2. 正常模式（Normal Mode）

```
默认模式，追求高吞吐

流程：
  ① G2 尝试获取锁，锁已被 G1 持有
  ② G2 先自旋（spin）一段时间，期望 G1 很快释放
  ③ 自旋失败 → G2 调用 sema 阻塞，进入等待队列尾部
  ④ G1 释放锁 → 唤醒队头的 G2
  ⑤ G2 被唤醒后，需要和新来的 G3、G4... 重新竞争锁

竞争示意：
  等待队列：[G2, G5, G6]（已等待一段时间）
  新来的：   G3, G4（刚到，CPU 上热乎的）

  Unlock() → G2 被唤醒，但 G3/G4 正在自旋
           → G3 或 G4 极可能先抢到锁   ← G2 又输了
           → G2 重新入队尾部等待
```

**为什么新 G 更容易抢到：**

```
新来的 G 正在 CPU 上运行（热状态）
被唤醒的 G 需要重新调度上 CPU（冷状态）

热 G 自旋抢锁 >> 冷 G 竞争，天然不公平
```

---

#### 3. 饥饿模式（Starvation Mode）

```
当某个 G 等待锁超过 1ms，触发饥饿模式

触发：
  G2 在等待队列中等待时间 > 1ms
  → 下次获得锁时，将 mutexStarving 置为 1
  → 进入饥饿模式

饥饿模式下的规则：
  ① 锁的所有权直接从 Unlock 的 G 移交给队头等待的 G
  ② 新来的 G 不允许自旋
  ③ 新来的 G 直接进入等待队列尾部
  ④ 严格按照 FIFO 顺序分配锁

移交示意：
  等待队列：[G2, G5, G6]
  新来的：   G3, G4

  Unlock() → 锁直接给 G2（队头）
           → G3、G4 乖乖排到队尾
           → 队列变为 [G5, G6, G3, G4]
```

---

#### 4. 模式切换时机

```
正常模式 → 饥饿模式：
  条件：某个 G 的等待时间超过 1ms
  操作：state |= mutexStarving

  [正常模式]
      ↓ 某 G 等待 > 1ms
  [饥饿模式]


饥饿模式 → 正常模式：
  满足以下任意一个条件即退出

  条件①：当前获得锁的 G 等待时间 < 1ms（说明队列消化很快）
  条件②：当前 G 是等待队列中最后一个（队列已清空）
  操作：state &^= mutexStarving

  [饥饿模式]
      ↓ 等待时间<1ms 或 队列最后一个
  [正常模式]
```

---

#### 5. 自旋的条件（正常模式下）

```go
// 满足以下全部条件才允许自旋
func canSpin() bool {
    return  runtime_canSpin(iter)       // 自旋次数未超限（最多4次）
         && gomaxprocs > 1              // 多核环境
         && 有空闲的 P                   // 有其他 P 可以运行 G
         && 本地队列为空                  // 当前 P 没有其他待运行的 G
}

// 饥饿模式下：直接跳过自旋，进入阻塞
if starving {
    // 不自旋，直接排队
}
```

---

#### 6. Unlock 在两种模式下的行为

```
正常模式下的 Unlock：

  ① 清除 mutexLocked 位
  ② 检查是否有等待者
  ③ 有等待者 → 通过 sema 唤醒队头 G
  ④ 被唤醒的 G 和新来的 G 公平竞争（实际上不公平）


饥饿模式下的 Unlock：

  ① 不清除 mutexLocked 位（锁还没真正释放）
  ② 直接通过 sema 唤醒队头 G
  ③ 将锁的所有权直接移交给该 G
  ④ 新来的 G 见到 mutexStarving=1，直接入队不抢锁
```

---

#### 7. 完整流程对比

```
正常模式完整流程：

  G 来了
    ↓
  尝试 CAS 加锁 → 成功 → 执行临界区
    ↓ 失败
  自旋等待（最多4次）
    ↓ 还是失败
  进入 sema 阻塞，加入等待队列尾部
    ↓ 被唤醒
  和新 G 竞争锁 → 可能再次失败 → 重新入队尾
    ↓ 等待 > 1ms
  下次唤醒时切换为饥饿模式


饥饿模式完整流程：

  G 来了
    ↓
  发现 mutexStarving = 1
    ↓
  直接进入 sema 阻塞，加入等待队列尾部（不自旋不抢锁）
    ↓ 轮到自己（FIFO）
  直接获得锁，无需竞争
    ↓
  检查退出条件 → 满足 → 切回正常模式
```

---

#### 总结

| 对比项 | 正常模式 | 饥饿模式 |
|--------|---------|---------|
| 触发条件 | 默认 | 等待时间 > 1ms |
| 新 G 行为 | 自旋抢锁 | 直接入队尾 |
| 锁分配 | 竞争（新 G 占优） | 严格 FIFO |
| 吞吐量 | 高 | 低 |
| 延迟 | 不可控 | 可控 |
| 退出条件 | — | 等待 < 1ms 或队列清空 |

> Mutex 两种模式的本质是**吞吐与公平的动态平衡**：平时用正常模式榨干 CPU 性能，一旦发现有 G 被饿死，立刻切换饥饿模式保障公平，两种模式相互兜底。

### MUTEX 允许自旋的条件
#### Mutex 允许自旋的条件

---

#### 1. 自旋的本质

```
自旋（Spin）：G 在获取锁失败后，不立即阻塞挂起
             而是空转 CPU 执行 pause 指令，反复尝试获取锁

目的：避免频繁的 goroutine 挂起/唤醒带来的上下文切换开销
代价：空转浪费 CPU
```

---

#### 2. 四个允许自旋的条件（全部满足）

```go
// runtime/proc.go 源码逻辑
func sync_runtime_canSpin(i int) bool {
    return i < active_spin          // 条件① 自旋次数 < 4
        && ncpu > 1                 // 条件② 多核 CPU
        && gomaxprocs > sched.npidle // 条件③ 有正在工作的 P
        && !osyield                 // 条件④ 本地队列为空
}
```

---

#### 3. 条件① 自旋次数不超过 4 次

```
const active_spin = 4  // 最多自旋 4 次

每次自旋执行 30 个 pause 指令（procyield(30)）

iter=0 → 自旋第1次（30个pause）
iter=1 → 自旋第2次（30个pause）
iter=2 → 自旋第3次（30个pause）
iter=3 → 自旋第4次（30个pause）
iter=4 → 超过限制，停止自旋，进入 sema 阻塞

原因：
  自旋是短暂等待的优化手段
  超过 4 次说明锁被长期持有，继续自旋只是浪费 CPU
  此时应老实排队阻塞
```

---

#### 4. 条件② 必须是多核 CPU

```
ncpu > 1

单核情况：
  ┌─────────────────────────────────────┐
  │  CPU Core 0                         │
  │  G1 持有锁（在运行）                  │
  │  G2 自旋等锁（也想运行）              │
  └─────────────────────────────────────┘

  单核下 G2 自旋 = G1 无法运行 = 锁永远不会释放
  → 死锁，毫无意义

多核情况：
  ┌──────────────┐    ┌──────────────┐
  │  CPU Core 0  │    │  CPU Core 1  │
  │  G1 持有锁   │    │  G2 自旋等锁  │
  └──────────────┘    └──────────────┘

  G1 和 G2 并行运行，G2 自旋期间 G1 可以释放锁
  → 自旋有意义
```

---

#### 5. 条件③ 当前有忙碌的 P（非全部空闲）

```
gomaxprocs > sched.npidle

含义：至少有一个 P 正在运行 G（不是所有 P 都空闲）

反例（所有 P 都空闲）：
  所有 P 都没有可运行的 G
  说明整个程序处于低负载/空转状态
  此时自旋毫无意义，应立即休眠释放 CPU

正例（有 P 在忙）：
  其他 P 正在运行 G，系统有活跃任务
  持有锁的 G 正在某个 P 上运行
  自旋等待它释放锁是合理的
```

---

#### 6. 条件④ 当前 P 的本地运行队列为空

```
p.runqhead == p.runqtail  // 本地队列为空

本地队列非空时：
  ┌─────────────────────────────────────┐
  │  P.runq: [G3, G4, G5]              │
  │  G2 正在自旋等锁                    │
  └─────────────────────────────────────┘

  G2 占着 M 自旋，G3/G4/G5 全部饿死
  → 浪费严重，不如让 G2 阻塞，让 M 去跑 G3

本地队列为空时：
  ┌─────────────────────────────────────┐
  │  P.runq: []（空）                   │
  │  G2 正在自旋等锁                    │
  └─────────────────────────────────────┘

  M 反正也没事干，自旋等锁不浪费任何资源
  → 自旋合理
```

---

#### 7. 饥饿模式下直接禁止自旋

```go
// 进入 Lock 时，发现饥饿模式直接跳过自旋
if atomic.LoadInt32(&m.state) & mutexStarving != 0 {
    // 饥饿模式：禁止自旋，直接入队阻塞
    break
}

原因：
  饥饿模式的核心是保障老 G 优先获锁
  允许新 G 自旋会破坏 FIFO 公平性
  → 新来的 G 一律乖乖排队
```

---

#### 8. 自旋的完整决策流程

```
G 尝试获取锁，失败
      ↓
  是饥饿模式？
  ├─ 是 → 直接阻塞入队，禁止自旋
  └─ 否 ↓
  满足4个自旋条件？（次数/多核/有忙P/队列空）
  ├─ 否 → 直接阻塞入队
  └─ 是 ↓
  执行 procyield(30)，空转30个pause
      ↓
  重新检查锁状态
  ├─ 锁释放了 → CAS 抢锁，成功则退出
  └─ 没释放   → iter++，回到条件判断
      ↓
  iter >= 4（自旋4次仍未获锁）
      ↓
  进入 sema 阻塞，加入等待队列
```

---

#### 总结

| 条件 | 具体要求 | 违反后果 |
|------|---------|---------|
| 自旋次数 | `iter < 4` | 超过4次无意义，改为阻塞 |
| 多核 CPU | `ncpu > 1` | 单核自旋=死锁，直接禁止 |
| 有忙碌的 P | `gomaxprocs > npidle` | 无活跃任务，自旋浪费 CPU |
| 本地队列为空 | `runq == 空` | 队列有 G 等待，自旋剥夺其执行权 |
| 非饥饿模式 | `mutexStarving == 0` | 饥饿模式禁止自旋，保障公平 |

> 自旋的本质是**用极短的 CPU 空转，换取避免线程切换的收益**，四个条件共同确保自旋只在"值得"的时候发生。

### RWMUTEX 实现 
#### RWMutex 实现

---

#### 1. RWMutex 的底层结构

```go
type RWMutex struct {
    w           Mutex   // 复用 Mutex，用于写锁互斥
    writerSem   uint32  // 写等待者的信号量（等待读全部完成）
    readerSem   uint32  // 读等待者的信号量（等待写锁释放）
    readerCount int32   // 当前活跃的读者数量（可为负数）
    readerWait  int32   // 写锁等待期间，还剩多少读者未释放
}

const rwmutexMaxReaders = 1 << 30  // 最大并发读者数（约10亿）
```

---

#### 2. 核心设计：readerCount 的双重含义

```
readerCount 是整个 RWMutex 的核心字段

正常状态（无写锁）：
  readerCount >= 0
  代表当前正在读的 G 数量

写锁介入后：
  readerCount -= rwmutexMaxReaders  →  变为负数
  负数 = 写锁标志 + 剩余读者数

  读 G 看到 readerCount < 0 → 感知到有写锁在等 → 阻塞

示意：
  readerCount =  3         → 3个G正在读，无写锁
  readerCount = -rwmutexMaxReaders + 3  → 有写锁等待，还有3个读者
```

---

#### 3. RLock 读加锁

```go
func (rw *RWMutex) RLock() {
    // readerCount + 1
    if atomic.AddInt32(&rw.readerCount, 1) < 0 {
        // readerCount 为负 → 当前有写锁在等待
        // 阻塞，等待写锁释放
        runtime_SemacquireMutex(&rw.readerSem, false, 0)
    }
    // readerCount >= 0 → 无写锁，直接获得读锁
}
```

```
流程示意：

无写锁时：
  G1 RLock → readerCount: 0 → 1  （直接获得，不阻塞）
  G2 RLock → readerCount: 1 → 2  （直接获得，不阻塞）
  G3 RLock → readerCount: 2 → 3  （直接获得，不阻塞）

有写锁等待时：
  readerCount 已被写锁减去 rwmutexMaxReaders（变负）
  G4 RLock → readerCount: -N → -(N-1)，仍 < 0
           → 阻塞在 readerSem 上
```

---

#### 4. RUnlock 读解锁

```go
func (rw *RWMutex) RUnlock() {
    // readerCount - 1
    if r := atomic.AddInt32(&rw.readerCount, -1); r < 0 {
        // r < 0 说明有写锁在等待
        rw.rUnlockSlow(r)
    }
}

func (rw *RWMutex) rUnlockSlow(r int32) {
    // readerWait - 1
    if atomic.AddInt32(&rw.readerWait, -1) == 0 {
        // 最后一个读者释放 → 唤醒写锁等待的 G
        runtime_Semrelease(&rw.writerSem, false, 1)
    }
}
```

```
流程示意：

写锁等待中，3个读者陆续释放：
  G1 RUnlock → readerWait: 3 → 2，不唤醒
  G2 RUnlock → readerWait: 2 → 1，不唤醒
  G3 RUnlock → readerWait: 1 → 0，唤醒写 G ← 最后一个读者负责唤醒
```

---

#### 5. Lock 写加锁

```go
func (rw *RWMutex) Lock() {
    // ① 先抢 Mutex，阻止其他写者
    rw.w.Lock()

    // ② readerCount 减去大数，变为负数，阻止新读者进入
    r := atomic.AddInt32(&rw.readerCount, -rwmutexMaxReaders) + rwmutexMaxReaders

    // ③ r > 0 说明还有活跃的读者，需要等待
    if r != 0 && atomic.AddInt32(&rw.readerWait, r) != 0 {
        // 阻塞，等待所有读者释放（由最后一个 RUnlock 唤醒）
        runtime_SemacquireMutex(&rw.writerSem, false, 0)
    }
    // 所有读者已释放，写锁获取成功
}
```

```
写加锁流程：

Step1: 抢 Mutex（挡住其他写者）
  w.Lock() → 只有一个写者能进入

Step2: 标记写锁（挡住新读者）
  readerCount -= rwmutexMaxReaders
  readerCount 变负 → 新 RLock 会阻塞

Step3: 等待存量读者退出
  记录当前活跃读者数到 readerWait
  阻塞在 writerSem，等最后一个读者唤醒自己

Step4: 获得写锁，进入临界区
```

---

#### 6. Unlock 写解锁

```go
func (rw *RWMutex) Unlock() {
    // ① readerCount 加回大数，恢复正数，允许读者进入
    r := atomic.AddInt32(&rw.readerCount, rwmutexMaxReaders)

    // ② 唤醒所有阻塞的读者
    for i := 0; i < int(r); i++ {
        runtime_Semrelease(&rw.readerSem, false, 0)
    }

    // ③ 释放 Mutex，允许其他写者竞争
    rw.w.Unlock()
}
```

```
写解锁流程：

Step1: 恢复 readerCount（允许新读者）
  readerCount += rwmutexMaxReaders → 变回正数

Step2: 批量唤醒所有等待的读者
  有 r 个读者阻塞在 readerSem
  全部唤醒，并发读恢复

Step3: 释放 Mutex
  下一个写者可以开始竞争
```

---

#### 7. 完整并发场景演示

```
初始：readerCount=0，无任何锁

① G1、G2、G3 并发 RLock：
  readerCount = 3
  三者同时读，互不阻塞          ← 读读并行 ✅

② G4 发起 Lock（写锁）：
  w.Lock() 成功
  readerCount = 3 - rwmutexMaxReaders（变负）
  readerWait = 3
  G4 阻塞在 writerSem

③ G5 发起 RLock：
  readerCount < 0 → 阻塞在 readerSem   ← 写锁挡住新读者 ✅

④ G1、G2、G3 依次 RUnlock：
  readerWait: 3 → 2 → 1 → 0
  最后一个（G3）唤醒 G4

⑤ G4 获得写锁，进入临界区：
  独占执行                      ← 写写互斥 ✅，读写互斥 ✅

⑥ G4 Unlock：
  readerCount 恢复正数
  唤醒 G5（及其他等待读者）
  释放 w.Mutex

⑦ G5 获得读锁，继续读          ← 写后读恢复 ✅
```

---

#### 8. 读写优先级与写饥饿问题

```
RWMutex 是写优先设计：

写锁等待期间：
  新来的读者 → readerCount < 0 → 直接阻塞
  不允许读者插队到写锁前面

但存在写饥饿风险：
  ┌──────────────────────────────────────────┐
  │ G_read 不断 RLock/RUnlock               │
  │ readerCount 始终 > 0                    │
  │ G_write 的 Lock 一直等不到 readerWait=0  │
  └──────────────────────────────────────────┘

Go 的缓解策略：
  写锁一旦调用 Lock()，后续新读者全部阻塞
  存量读者读完后必然 readerWait → 0
  → 写锁最终一定能获得，不会真正饿死
```

---

#### 总结

| 操作 | readerCount 变化 | 关键行为 |
|------|----------------|---------|
| `RLock` | +1 | 若 < 0 则阻塞（有写锁） |
| `RUnlock` | -1 | 若触发 readerWait=0 则唤醒写锁 |
| `Lock` | -rwmutexMaxReaders | 变负阻断新读者，等存量读者清零 |
| `Unlock` | +rwmutexMaxReaders | 变正放行读者，批量唤醒，释放 Mutex |

> RWMutex 的精髓：**用 readerCount 的正负号作为写锁标志，一个字段同时承担计数与信号两种职责，实现读读并行、读写互斥、写写互斥的高效分离。**

### RWMUTEX 注意事项  
#### RWMutex 注意事项

---

#### 1. 不可复制

```go
// ❌ 错误：复制 RWMutex 会连同内部状态一起复制
type Cache struct {
    mu sync.RWMutex
    data map[string]string
}

c1 := Cache{}
c2 := c1        // 复制了 mu 的 readerCount/state 等内部状态
c2.mu.RLock()   // 行为未定义，可能死锁

// ✅ 正确：通过指针传递
c2 := &c1
// 或结构体字段始终用指针接收者操作
func (c *Cache) Get(k string) string {
    c.mu.RLock()
    defer c.mu.RUnlock()
    return c.data[k]
}
```

```
go vet 能检测到 RWMutex 的复制行为
建议始终用指针接收者操作含有 RWMutex 的结构体
```

---

#### 2. Lock 和 RLock 不可重入

```go
// ❌ 写锁重入 → 死锁
func foo() {
    rw.Lock()
    bar()       // bar 内部再次 Lock() → 永久阻塞
    rw.Unlock()
}

func bar() {
    rw.Lock()   // 死锁！写锁不可重入
    defer rw.Unlock()
}

// ❌ 读锁重入（有写锁等待时）→ 死锁
func foo() {
    rw.RLock()
    bar()
    rw.RUnlock()
}

func bar() {
    rw.RLock()   // 若此时有写锁在等待
                 // 写锁阻断新读者 → bar 阻塞
                 // foo 的 RUnlock 永远不会执行 → 死锁
    defer rw.RUnlock()
}
```

```
Go 的 sync.RWMutex 没有重入机制
任何形式的嵌套加锁都可能导致死锁
```

---

#### 3. RLock/RUnlock 必须成对调用

```go
// ❌ 错误：RUnlock 多调用一次
rw.RLock()
rw.RUnlock()
rw.RUnlock()  // panic: sync: RUnlock of unlocked RWMutex

// ❌ 错误：加锁后忘记解锁（写锁永远占用）
func update() {
    rw.Lock()
    if err != nil {
        return    // 忘记 Unlock，后续所有操作全部阻塞
    }
    rw.Unlock()
}

// ✅ 正确：用 defer 保证成对
func update() {
    rw.Lock()
    defer rw.Unlock()
    // 无论如何返回都会解锁
}
```

---

#### 4. 持锁期间不能调用会阻塞的操作

```go
// ❌ 错误：持有写锁期间执行耗时操作
func save() {
    rw.Lock()
    defer rw.Unlock()

    http.Get("https://example.com")  // 网络 IO，可能阻塞数秒
    db.Query("SELECT ...")           // 数据库查询，不可控耗时
    time.Sleep(time.Second)          // 主动休眠

    // 期间所有读写操作全部阻塞！
}

// ✅ 正确：锁内只做最小操作，IO 放锁外
func save() {
    // 锁外执行耗时操作
    result, err := http.Get("https://example.com")

    // 锁内只做数据写入
    rw.Lock()
    defer rw.Unlock()
    cache["key"] = result
}
```

---

#### 5. 写锁等待期间新读锁会阻塞（写优先陷阱）

```go
// 场景：高频读场景下，偶尔一次写锁可能造成读请求积压

rw.RLock() → rw.RLock() → rw.RLock() ...  // 读请求持续不断

rw.Lock()  // 写锁等待
           // 此后所有新 RLock 全部阻塞
           // 等待已有读者释放

// 存量读者释放 → 写锁获得 → 执行 → 释放
// 积压的读请求才能批量恢复

// ⚠️ 注意：写锁等待时间越长，读请求积压越多
// 锁粒度要尽量小，减少写锁持有时间
```

---

#### 6. 不要用 RWMutex 保护简单原子操作

```go
// ❌ 过度使用：简单计数器用 RWMutex 反而更慢
var rw sync.RWMutex
var count int

func increment() {
    rw.Lock()
    count++
    rw.Unlock()
}

// ✅ 正确：简单数值用 atomic，开销更小
var count int64

func increment() {
    atomic.AddInt64(&count, 1)
}

// RWMutex 适合场景：
//   读多写少 + 临界区操作复杂（map、slice、结构体）
// atomic 适合场景：
//   简单数值的读写（int、bool、pointer）
```

---

#### 7. 避免锁的粒度过粗

```go
// ❌ 粒度过粗：整个函数加一把大锁
func (c *Cache) BatchGet(keys []string) []string {
    c.mu.RLock()
    defer c.mu.RUnlock()

    result := make([]string, len(keys))
    for i, k := range keys {
        result[i] = c.data[k]
        time.Sleep(time.Millisecond) // 模拟计算，全程持锁
    }
    return result
}

// ✅ 粒度适中：只在访问共享数据时加锁
func (c *Cache) BatchGet(keys []string) []string {
    result := make([]string, len(keys))
    for i, k := range keys {
        c.mu.RLock()
        result[i] = c.data[k]   // 只锁数据读取
        c.mu.RUnlock()
        // 计算逻辑放锁外
    }
    return result
}
```

---

#### 8. RWMutex 与 Mutex 的选型

```
读写次数决定选型：

读多写少（读:写 > 10:1）
  → 用 RWMutex
  → 读并发提升明显

读写均衡 或 写多读少
  → 用 Mutex
  → RWMutex 额外维护 readerCount/readerWait 反而更慢

数据量极小（简单 int/bool）
  → 用 atomic
  → 无锁操作，性能最优

对比：
┌──────────────┬────────────┬──────────────────────────┐
│   类型        │  适用场景   │         特点              │
├──────────────┼────────────┼──────────────────────────┤
│ sync.Mutex   │ 读写均衡    │ 简单，无额外开销            │
│ sync.RWMutex │ 读多写少    │ 读并发高，写有等待开销       │
│ sync/atomic  │ 简单数值    │ 无锁，性能最优              │
└──────────────┴────────────┴──────────────────────────┘
```

---

#### 总结

| 注意点 | 风险 | 解决方案 |
|--------|------|---------|
| 不可复制 | 状态异常/死锁 | 指针传递，`go vet` 检测 |
| 不可重入 | 死锁 | 避免嵌套加锁，拆分函数 |
| 必须成对 | panic/永久阻塞 | 始终使用 `defer` 解锁 |
| 持锁勿阻塞 | 所有并发请求积压 | 锁内只做内存操作 |
| 写优先陷阱 | 写锁期间读请求积压 | 缩短写锁持有时间 |
| 选型不当 | 性能下降 | 读多写少才用 RWMutex |

> RWMutex 的本质是**用复杂度换并发度**，只有在读多写少的场景下收益才能覆盖其维护成本，否则一把 Mutex 更简单可靠。

### COND 是什么
###  BROADCAST 和SIGNAL 区别
### COND 中WAIT 使用场景
### WAITGROUP 用法

#### WaitGroup 用法

---

#### 1. 基本用法

```go
var wg sync.WaitGroup

for i := 0; i < 3; i++ {
    wg.Add(1)               // 启动前 +1
    go func(id int) {
        defer wg.Done()     // 结束时 -1
        fmt.Println("G", id, "done")
    }(i)
}

wg.Wait()                   // 阻塞，直到 counter == 0
fmt.Println("all done")
```

```
输出（顺序不固定）：
G 0 done
G 2 done
G 1 done
all done
```

---

#### 2. Add 必须在 go 之前调用

```go
// ❌ 错误：Add 在 goroutine 内部调用
for i := 0; i < 3; i++ {
    go func(id int) {
        wg.Add(1)           // 可能在 Wait() 之后才执行
        defer wg.Done()
    }(i)
}
wg.Wait()                   // 可能提前返回！

// ✅ 正确：Add 在启动 goroutine 之前调用
for i := 0; i < 3; i++ {
    wg.Add(1)               // 先 +1，再启动
    go func(id int) {
        defer wg.Done()
    }(i)
}
wg.Wait()
```

---

#### 3. 配合 defer 使用 Done

```go
// ❌ 不安全：函数中途 panic 或 return，Done 不会执行
func worker(wg *sync.WaitGroup) {
    // ...
    if err != nil {
        return              // 忘记 Done，counter 永远不归零
    }
    wg.Done()
}

// ✅ 安全：defer 保证任何情况下都执行 Done
func worker(wg *sync.WaitGroup) {
    defer wg.Done()         // 第一行就 defer，万无一失
    // ...
    if err != nil {
        return              // defer 保证 Done 执行
    }
}
```

---

#### 4. 通过指针传递 WaitGroup

```go
// ❌ 错误：值传递会复制 WaitGroup 的内部状态
func worker(wg sync.WaitGroup) {  // 值传递
    defer wg.Done()               // 操作的是副本，主 G 感知不到
}

// ✅ 正确：指针传递
func worker(wg *sync.WaitGroup) { // 指针传递
    defer wg.Done()               // 操作同一个 WaitGroup
}

func main() {
    var wg sync.WaitGroup
    wg.Add(1)
    go worker(&wg)                // 传指针
    wg.Wait()
}
```

---

#### 5. 批量启动任务

```go
func main() {
    var wg sync.WaitGroup
    tasks := []string{"task1", "task2", "task3", "task4", "task5"}

    wg.Add(len(tasks))            // 一次性 Add 总数
    for _, task := range tasks {
        go func(t string) {
            defer wg.Done()
            process(t)
        }(task)                   // 注意：传参避免闭包捕获问题
    }

    wg.Wait()
    fmt.Println("all tasks done")
}
```

---

#### 6. 配合 errgroup 收集错误

```go
import "golang.org/x/sync/errgroup"

// 原生 WaitGroup 无法收集错误
// ✅ 使用 errgroup 代替
func main() {
    var eg errgroup.Group

    for i := 0; i < 3; i++ {
        id := i
        eg.Go(func() error {      // 内部自动 Add(1) 和 Done()
            if id == 2 {
                return fmt.Errorf("task %d failed", id)
            }
            return nil
        })
    }

    if err := eg.Wait(); err != nil {  // 返回第一个错误
        fmt.Println("error:", err)
    }
}
```

---

#### 7. 控制并发数量（配合 channel）

```go
// WaitGroup 本身不限制并发数
// 配合 buffered channel 实现并发限制

func main() {
    var wg sync.WaitGroup
    limit := make(chan struct{}, 3)  // 最多同时 3 个 goroutine

    for i := 0; i < 10; i++ {
        wg.Add(1)
        limit <- struct{}{}         // 占一个槽位，满了就阻塞
        go func(id int) {
            defer wg.Done()
            defer func() { <-limit }() // 释放槽位
            process(id)
        }(i)
    }

    wg.Wait()
    fmt.Println("all done")
}
```

```
同时运行的 goroutine 数量：
  ┌───┬───┬───┐
  │G1 │G2 │G3 │  ← 最多3个并发
  └───┴───┴───┘
  G4~G10 等待槽位释放
```

---

#### 8. 嵌套使用 WaitGroup

```go
// 父任务等待子任务，子任务内部再等待孙任务
func main() {
    var parentWg sync.WaitGroup

    for i := 0; i < 3; i++ {
        parentWg.Add(1)
        go func(id int) {
            defer parentWg.Done()

            var childWg sync.WaitGroup  // 每个父任务有自己的 childWg
            for j := 0; j < 3; j++ {
                childWg.Add(1)
                go func(cid int) {
                    defer childWg.Done()
                    fmt.Printf("parent %d child %d done\n", id, cid)
                }(j)
            }
            childWg.Wait()              // 等待子任务完成
            fmt.Printf("parent %d done\n", id)
        }(i)
    }

    parentWg.Wait()
    fmt.Println("all done")
}
```

---

#### 9. 超时控制（配合 context）

```go
// WaitGroup 本身不支持超时
// ✅ 配合 context + channel 实现超时等待

func waitWithTimeout(wg *sync.WaitGroup, timeout time.Duration) bool {
    done := make(chan struct{})

    go func() {
        wg.Wait()
        close(done)             // 所有任务完成，关闭 channel
    }()

    select {
    case <-done:
        return true             // 正常完成
    case <-time.After(timeout):
        return false            // 超时
    }
}

func main() {
    var wg sync.WaitGroup
    wg.Add(2)
    go func() { defer wg.Done(); time.Sleep(time.Second) }()
    go func() { defer wg.Done(); time.Sleep(time.Second) }()

    if !waitWithTimeout(&wg, 3*time.Second) {
        fmt.Println("timeout!")
    } else {
        fmt.Println("done!")
    }
}
```

---

#### 总结

| 场景 | 要点 |
|------|------|
| 基本用法 | `Add` 在 `go` 前，`Done` 用 `defer`，`Wait` 阻塞汇聚 |
| 传递方式 | 始终传指针，禁止值传递和复制 |
| 错误收集 | 原生不支持，改用 `errgroup` |
| 并发限制 | 配合 `buffered channel` 控制并发数 |
| 超时控制 | 配合 `channel + select` 实现超时等待 |
| 嵌套使用 | 每层用独立的 WaitGroup，职责清晰 |

> WaitGroup 的本质是**计数器 + 信号量的组合**，`Add/Done` 维护计数，`Wait` 监听归零信号，用法虽简单，但 Add 时机、指针传递、defer 配合缺一不可。
### WAITGROUP 实现原理 

#### WaitGroup 实现原理

---

#### 1. WaitGroup 底层结构

```go
type WaitGroup struct {
    noCopy noCopy    // 禁止复制的标记（go vet 检测用）
    state1 [3]uint32 // 核心状态：counter + waiter + sema
}
```

`state1` 的内存布局：

```
64位系统（8字节对齐）：
┌─────────────────────┬─────────────────────┬──────────┐
│     counter（32位）  │     waiter（32位）   │ sema（32位）│
│   Add 计数器         │   Wait 等待者数量    │  信号量   │
└─────────────────────┴─────────────────────┴──────────┘
  state1[0]高32位        state1[0]低32位       state1[1]

32位系统（4字节对齐）：
┌──────────┬─────────────────────┬─────────────────────┐
│ sema（32位）│    counter（32位）  │    waiter（32位）   │
│  信号量   │   Add 计数器         │   Wait 等待者数量   │
└──────────┴─────────────────────┴──────────┴──────────┘
  state1[0]    state1[1]高32位      state1[1]低32位
```

```go
// 根据对齐情况取 statep 和 semap
func (wg *WaitGroup) state() (statep *uint64, semap *uint32) {
    if uintptr(unsafe.Pointer(&wg.state1))%8 == 0 {
        // 64 位对齐
        return (*uint64)(unsafe.Pointer(&wg.state1)), &wg.state1[2]
    }
    // 32 位对齐
    return (*uint64)(unsafe.Pointer(&wg.state1[1])), &wg.state1[0]
}
```

---

#### 2. Add 的实现

```go
func (wg *WaitGroup) Add(delta int) {
    statep, semap := wg.state()

    // counter += delta（操作高32位）
    state := atomic.AddUint64(statep, uint64(delta)<<32)

    v := int32(state >> 32)  // 取 counter
    w := uint32(state)       // 取 waiter

    // counter 不能为负
    if v < 0 {
        panic("sync: negative WaitGroup counter")
    }

    // counter > 0 或 没有等待者 → 直接返回
    if v > 0 || w == 0 {
        return
    }

    // counter == 0 且有等待者 → 唤醒所有 Wait 的 G
    *statep = 0  // 清空 counter 和 waiter
    for ; w != 0; w-- {
        runtime_Semrelease(semap, false, 0)
    }
}
```

```
Add(1) 流程：

  state 高32位 counter +1
  ┌──────────────┬──────────┐
  │ counter: 0→1 │ waiter:0 │
  └──────────────┴──────────┘
  counter > 0 → 直接返回，无需唤醒

Add(-1) 即 Done() 流程（counter减到0时）：

  state 高32位 counter -1
  ┌──────────────┬──────────┐
  │ counter: 1→0 │ waiter:2 │
  └──────────────┴──────────┘
  counter == 0 且 waiter > 0
  → 清空 state
  → 循环唤醒 2 个等待的 G
```

---

#### 3. Done 的实现

```go
func (wg *WaitGroup) Done() {
    wg.Add(-1)  // 本质就是 Add(-1)
}
```

```
每次 goroutine 完成任务调用 Done()：

  wg.Add(1)  → counter = 1
  wg.Add(1)  → counter = 2
  wg.Add(1)  → counter = 3

  wg.Done()  → counter = 2
  wg.Done()  → counter = 1
  wg.Done()  → counter = 0 → 唤醒所有 Wait 的 G
```

---

#### 4. Wait 的实现

```go
func (wg *WaitGroup) Wait() {
    statep, semap := wg.state()

    for {
        state := atomic.LoadUint64(statep)
        v := int32(state >> 32)  // counter
        w := uint32(state)       // waiter

        // counter 已经为 0，无需等待
        if v == 0 {
            return
        }

        // CAS 将 waiter + 1，登记自己为等待者
        if atomic.CompareAndSwapUint64(statep, state, state+1) {
            // 阻塞，等待 Add(负数) 将 counter 清零时唤醒
            runtime_Semacquire(semap)
            // 被唤醒后 statep 已被清零
            if *statep != 0 {
                panic("sync: WaitGroup is reused before previous Wait has returned")
            }
            return
        }
        // CAS 失败（并发修改），重试
    }
}
```

```
Wait() 流程：

  ① 读取当前 state
  ② counter == 0 → 直接返回（任务已全部完成）
  ③ counter > 0  → CAS 将 waiter+1，登记自己
  ④ 阻塞在 semap，挂起等待
  ⑤ 被 Add(counter归零) 唤醒后返回
```

---

#### 5. 完整执行时序

```
主 G                G1               G2               G3
  │                  │                │                │
wg.Add(3)            │                │                │
  │ counter=3        │                │                │
  │                  │                │                │
wg.Wait()            │                │                │
  │ waiter=1         │                │                │
  │ 阻塞             │                │                │
  │                  │                │                │
  │               wg.Done()           │                │
  │               counter=2           │                │
  │               不唤醒              │                │
  │                  │             wg.Done()           │
  │                  │             counter=1           │
  │                  │             不唤醒              │
  │                  │                │            wg.Done()
  │                  │                │            counter=0
  │                  │                │            waiter=1>0
  │                  │                │            唤醒主G ←─┐
  │◄─────────────────────────────────────────────────────────┘
wg.Wait() 返回
  │ 继续执行
```

---

#### 6. state 字段的原子操作设计

```
为什么 counter 和 waiter 压缩在一个 uint64 里？

分开存储的问题：
  修改 counter 和 waiter 需要两次原子操作
  两次操作之间可能被其他 G 插入，产生竞态

合并存储的优势：
  一次 atomic.AddUint64 同时更新
  一次 atomic.CAS 同时判断 + 修改
  天然保证 counter 和 waiter 的一致性

  ┌──────────────────────────────────────────┐
  │  uint64                                  │
  │  高32位：counter  低32位：waiter          │
  │                                          │
  │  Add(delta): atomic.AddUint64(uint64(delta)<<32) │
  │  只改高32位，低32位不受影响               │
  └──────────────────────────────────────────┘
```

---

#### 7. 使用中的常见陷阱

```go
// ❌ 陷阱①：在 goroutine 内部调用 Add
go func() {
    wg.Add(1)   // 可能在 Wait() 之后才执行，导致 Wait 提前返回
    defer wg.Done()
    // ...
}()
wg.Wait()

// ✅ 正确：启动 goroutine 前调用 Add
wg.Add(1)
go func() {
    defer wg.Done()
    // ...
}()
wg.Wait()


// ❌ 陷阱②：Wait 返回前重用 WaitGroup
wg.Add(1)
go func() {
    defer wg.Done()
}()
wg.Wait()
wg.Add(1)  // ⚠️ 上一轮 Wait 的 G 可能还未完全退出，state 未清零

// ✅ 正确：确保上一轮完全结束再复用
wg.Wait()
time.Sleep(...)  // 或通过设计保证复用安全


// ❌ 陷阱③：counter 被减为负数
wg.Add(1)
wg.Done()
wg.Done()  // panic: sync: negative WaitGroup counter
```

---

#### 总结

| 字段 | 位置 | 作用 |
|------|------|------|
| `counter` | state 高32位 | 记录未完成的任务数 |
| `waiter` | state 低32位 | 记录阻塞在 Wait 的 G 数量 |
| `sema` | state1[2] | 信号量，用于唤醒 Wait 的 G |

| 方法 | 核心操作 | 触发唤醒条件 |
|------|---------|------------|
| `Add(n)` | counter += n | counter==0 且 waiter>0 |
| `Done()` | counter -= 1 | 同上 |
| `Wait()` | waiter += 1，阻塞 | 被 Add 归零时唤醒 |

> WaitGroup 的精髓：**将 counter 和 waiter 压缩进一个 uint64，用一次原子操作保证两者的强一致性，以最小的内存开销实现了多 goroutine 的汇聚同步。**

#### 1. WaitGroup 底层结构

```go
type WaitGroup struct {
    noCopy noCopy    // 禁止复制的标记（go vet 检测用）
    state1 [3]uint32 // 核心状态：counter + waiter + sema
}
```

`state1` 的内存布局：

```
64位系统（8字节对齐）：
┌─────────────────────┬─────────────────────┬──────────┐
│     counter（32位）  │     waiter（32位）   │ sema（32位）│
│   Add 计数器         │   Wait 等待者数量    │  信号量   │
└─────────────────────┴─────────────────────┴──────────┘
  state1[0]高32位        state1[0]低32位       state1[1]

32位系统（4字节对齐）：
┌──────────┬─────────────────────┬─────────────────────┐
│ sema（32位）│    counter（32位）  │    waiter（32位）   │
│  信号量   │   Add 计数器         │   Wait 等待者数量   │
└──────────┴─────────────────────┴──────────┴──────────┘
  state1[0]    state1[1]高32位      state1[1]低32位
```

```go
// 根据对齐情况取 statep 和 semap
func (wg *WaitGroup) state() (statep *uint64, semap *uint32) {
    if uintptr(unsafe.Pointer(&wg.state1))%8 == 0 {
        // 64 位对齐
        return (*uint64)(unsafe.Pointer(&wg.state1)), &wg.state1[2]
    }
    // 32 位对齐
    return (*uint64)(unsafe.Pointer(&wg.state1[1])), &wg.state1[0]
}
```

---

#### 2. Add 的实现

```go
func (wg *WaitGroup) Add(delta int) {
    statep, semap := wg.state()

    // counter += delta（操作高32位）
    state := atomic.AddUint64(statep, uint64(delta)<<32)

    v := int32(state >> 32)  // 取 counter
    w := uint32(state)       // 取 waiter

    // counter 不能为负
    if v < 0 {
        panic("sync: negative WaitGroup counter")
    }

    // counter > 0 或 没有等待者 → 直接返回
    if v > 0 || w == 0 {
        return
    }

    // counter == 0 且有等待者 → 唤醒所有 Wait 的 G
    *statep = 0  // 清空 counter 和 waiter
    for ; w != 0; w-- {
        runtime_Semrelease(semap, false, 0)
    }
}
```

```
Add(1) 流程：

  state 高32位 counter +1
  ┌──────────────┬──────────┐
  │ counter: 0→1 │ waiter:0 │
  └──────────────┴──────────┘
  counter > 0 → 直接返回，无需唤醒

Add(-1) 即 Done() 流程（counter减到0时）：

  state 高32位 counter -1
  ┌──────────────┬──────────┐
  │ counter: 1→0 │ waiter:2 │
  └──────────────┴──────────┘
  counter == 0 且 waiter > 0
  → 清空 state
  → 循环唤醒 2 个等待的 G
```

---

#### 3. Done 的实现

```go
func (wg *WaitGroup) Done() {
    wg.Add(-1)  // 本质就是 Add(-1)
}
```

```
每次 goroutine 完成任务调用 Done()：

  wg.Add(1)  → counter = 1
  wg.Add(1)  → counter = 2
  wg.Add(1)  → counter = 3

  wg.Done()  → counter = 2
  wg.Done()  → counter = 1
  wg.Done()  → counter = 0 → 唤醒所有 Wait 的 G
```

---

#### 4. Wait 的实现

```go
func (wg *WaitGroup) Wait() {
    statep, semap := wg.state()

    for {
        state := atomic.LoadUint64(statep)
        v := int32(state >> 32)  // counter
        w := uint32(state)       // waiter

        // counter 已经为 0，无需等待
        if v == 0 {
            return
        }

        // CAS 将 waiter + 1，登记自己为等待者
        if atomic.CompareAndSwapUint64(statep, state, state+1) {
            // 阻塞，等待 Add(负数) 将 counter 清零时唤醒
            runtime_Semacquire(semap)
            // 被唤醒后 statep 已被清零
            if *statep != 0 {
                panic("sync: WaitGroup is reused before previous Wait has returned")
            }
            return
        }
        // CAS 失败（并发修改），重试
    }
}
```

```
Wait() 流程：

  ① 读取当前 state
  ② counter == 0 → 直接返回（任务已全部完成）
  ③ counter > 0  → CAS 将 waiter+1，登记自己
  ④ 阻塞在 semap，挂起等待
  ⑤ 被 Add(counter归零) 唤醒后返回
```

---

#### 5. 完整执行时序

```
主 G                G1               G2               G3
  │                  │                │                │
wg.Add(3)            │                │                │
  │ counter=3        │                │                │
  │                  │                │                │
wg.Wait()            │                │                │
  │ waiter=1         │                │                │
  │ 阻塞             │                │                │
  │                  │                │                │
  │               wg.Done()           │                │
  │               counter=2           │                │
  │               不唤醒              │                │
  │                  │             wg.Done()           │
  │                  │             counter=1           │
  │                  │             不唤醒              │
  │                  │                │            wg.Done()
  │                  │                │            counter=0
  │                  │                │            waiter=1>0
  │                  │                │            唤醒主G ←─┐
  │◄─────────────────────────────────────────────────────────┘
wg.Wait() 返回
  │ 继续执行
```

---

#### 6. state 字段的原子操作设计

```
为什么 counter 和 waiter 压缩在一个 uint64 里？

分开存储的问题：
  修改 counter 和 waiter 需要两次原子操作
  两次操作之间可能被其他 G 插入，产生竞态

合并存储的优势：
  一次 atomic.AddUint64 同时更新
  一次 atomic.CAS 同时判断 + 修改
  天然保证 counter 和 waiter 的一致性

  ┌──────────────────────────────────────────┐
  │  uint64                                  │
  │  高32位：counter  低32位：waiter          │
  │                                          │
  │  Add(delta): atomic.AddUint64(uint64(delta)<<32) │
  │  只改高32位，低32位不受影响               │
  └──────────────────────────────────────────┘
```

---

#### 7. 使用中的常见陷阱

```go
// ❌ 陷阱①：在 goroutine 内部调用 Add
go func() {
    wg.Add(1)   // 可能在 Wait() 之后才执行，导致 Wait 提前返回
    defer wg.Done()
    // ...
}()
wg.Wait()

// ✅ 正确：启动 goroutine 前调用 Add
wg.Add(1)
go func() {
    defer wg.Done()
    // ...
}()
wg.Wait()


// ❌ 陷阱②：Wait 返回前重用 WaitGroup
wg.Add(1)
go func() {
    defer wg.Done()
}()
wg.Wait()
wg.Add(1)  // ⚠️ 上一轮 Wait 的 G 可能还未完全退出，state 未清零

// ✅ 正确：确保上一轮完全结束再复用
wg.Wait()
time.Sleep(...)  // 或通过设计保证复用安全


// ❌ 陷阱③：counter 被减为负数
wg.Add(1)
wg.Done()
wg.Done()  // panic: sync: negative WaitGroup counter
```

---

#### 总结

| 字段 | 位置 | 作用 |
|------|------|------|
| `counter` | state 高32位 | 记录未完成的任务数 |
| `waiter` | state 低32位 | 记录阻塞在 Wait 的 G 数量 |
| `sema` | state1[2] | 信号量，用于唤醒 Wait 的 G |

| 方法 | 核心操作 | 触发唤醒条件 |
|------|---------|------------|
| `Add(n)` | counter += n | counter==0 且 waiter>0 |
| `Done()` | counter -= 1 | 同上 |
| `Wait()` | waiter += 1，阻塞 | 被 Add 归零时唤醒 |

> WaitGroup 的精髓：**将 counter 和 waiter 压缩进一个 uint64，用一次原子操作保证两者的强一致性，以最小的内存开销实现了多 goroutine 的汇聚同步。**

### 什么是SYNC.ONCE
好问题！在较新版本的 Go（1.19+）中，`sync.Once` 的结构已经更新，`done` 的类型从 `uint32` 改为了 `atomic.Uint32`：

```go
// Go 1.19+ 实际源码
type Once struct {
    done atomic.Uint32  // 不再是裸 uint32，而是 atomic.Uint32 类型
    m    Mutex
}
```

对应的 `Do` 实现也随之更新：

```go
func (o *Once) Do(f func()) {
    if o.done.Load() == 0 {  // 用 .Load() 方法替代 atomic.LoadUint32
        o.doSlow(f)
    }
}

func (o *Once) doSlow(f func()) {
    o.m.Lock()
    defer o.m.Unlock()
    if o.done.Load() == 0 {
        defer o.done.Store(1)  // 用 .Store() 方法替代 atomic.StoreUint32
        f()
    }
}
```

两个版本的对比：

| | Go 1.19 之前 | Go 1.19+ |
|--|--|--|
| done 类型 | `uint32` | `atomic.Uint32` |
| 读操作 | `atomic.LoadUint32(&o.done)` | `o.done.Load()` |
| 写操作 | `atomic.StoreUint32(&o.done, 1)` | `o.done.Store(1)` |
| 核心逻辑 | 完全一致 | 完全一致 |

本质逻辑没有变化，只是将原子操作从**函数调用**风格改为了**方法调用**风格，代码更简洁，语义更清晰。感谢指正！
### 什么操作叫做原子操作
#### 什么是原子操作

---

#### 1. 原子操作的定义

```
原子操作（Atomic Operation）是指一个操作在执行过程中
不会被任何其他操作中断，要么完全执行，要么完全不执行

核心特性：
  不可分割 → 执行过程中不会被打断
  无中间状态 → 其他 G 看不到操作的中间过程
  并发安全 → 无需加锁即可保证数据一致性
```

---

#### 2. 为什么需要原子操作

```go
// 看似简单的 i++ 实际上是三条指令
var i int64 = 0

i++
// 底层汇编：
// LOAD  → 从内存读取 i 的值到寄存器   step1
// ADD   → 寄存器值 +1                 step2
// STORE → 将结果写回内存              step3

// 并发问题：
// G1: LOAD i=0
// G2: LOAD i=0   ← G2 在 G1 STORE 之前读取
// G1: ADD  → 1
// G2: ADD  → 1
// G1: STORE i=1
// G2: STORE i=1  ← 期望 i=2，实际 i=1，数据丢失！
```

```
原子操作将 LOAD + ADD + STORE 合并为一条不可分割的 CPU 指令
其他 G 只能看到操作前或操作后的值，永远看不到中间状态
```

---

#### 3. Go 中的原子操作包

```go
import "sync/atomic"

// 支持的类型：
// int32、int64、uint32、uint64、uintptr、unsafe.Pointer
// Go 1.19+ 新增泛型封装：atomic.Int32、atomic.Int64 等
```

---

#### 4. 五类原子操作

##### Load 原子读

```go
var val int64 = 100

// 普通读（非原子）：可能读到其他 G 写到一半的中间值
v := val

// 原子读：保证读到的是完整的值
v := atomic.LoadInt64(&val)

// 1.19+ 写法
var val atomic.Int64
v := val.Load()
```

---

##### Store 原子写

```go
var val int64

// 普通写：写入过程可能被其他 G 看到一半
val = 100

// 原子写：写入要么完成，要么未开始，不存在中间态
atomic.StoreInt64(&val, 100)

// 1.19+ 写法
var val atomic.Int64
val.Store(100)
```

---

##### Add 原子加减

```go
var counter int64

// 并发不安全
counter++

// 原子加（返回新值）
newVal := atomic.AddInt64(&counter, 1)   // +1
newVal := atomic.AddInt64(&counter, -1)  // -1
newVal := atomic.AddInt64(&counter, 10)  // +10

// 1.19+ 写法
var counter atomic.Int64
counter.Add(1)
```

---

##### CAS 比较并交换（Compare And Swap）

```go
// CAS 是原子操作中最重要的原语
// 含义：如果当前值 == old，则将值改为 new，返回是否成功

var val int64 = 100

// 只有 val == 100 时，才将 val 改为 200
swapped := atomic.CompareAndSwapInt64(&val, 100, 200)
// swapped = true,  val = 200（交换成功）

swapped = atomic.CompareAndSwapInt64(&val, 100, 300)
// swapped = false, val = 200（val 已经是 200，不等于 100，失败）

// 1.19+ 写法
var val atomic.Int64
val.Store(100)
ok := val.CompareAndSwap(100, 200)
```

```
CAS 底层原理：

  CPU 提供 CMPXCHG 指令，一条指令完成比较+交换
  操作系统级别保证原子性，无法被中断

  CAS 是实现无锁数据结构的基础
  sync.Mutex、sync.Once、channel 底层都用到了 CAS
```

---

##### Swap 原子交换

```go
var val int64 = 100

// 将 val 设为 200，返回旧值
old := atomic.SwapInt64(&val, 200)
// old = 100, val = 200

// 1.19+ 写法
var val atomic.Int64
val.Store(100)
old := val.Swap(200)
```

---

#### 5. Go 1.19+ 泛型封装类型

```go
// 新版本提供了面向对象风格的原子类型，更安全易用
var (
    i   atomic.Int32
    j   atomic.Int64
    u   atomic.Uint32
    b   atomic.Bool
    p   atomic.Pointer[MyStruct]  // 泛型指针
    v   atomic.Value              // 任意类型
)

// atomic.Bool 示例
var flag atomic.Bool
flag.Store(true)
fmt.Println(flag.Load())   // true
flag.Swap(false)
flag.CompareAndSwap(false, true)

// atomic.Value 存任意类型（读写类型必须一致）
var v atomic.Value
v.Store(map[string]int{"a": 1})
m := v.Load().(map[string]int)
```

---

#### 6. 原子操作 vs 互斥锁

```
性能对比（操作耗时，越小越好）：

  atomic.AddInt64    ~  2ns   ← 硬件级别，极快
  sync.Mutex Lock    ~ 15ns   ← 需要内核调用
  sync.RWMutex RLock ~ 25ns   ← 维护读写状态

适用场景对比：

┌──────────────┬──────────────┬──────────────────────────┐
│              │  atomic       │  Mutex                   │
├──────────────┼──────────────┼──────────────────────────┤
│ 操作复杂度   │ 单个变量      │ 任意复杂操作              │
│ 数据结构     │ 简单值类型    │ map、slice、struct        │
│ 性能         │ 极高          │ 较高                     │
│ 使用难度     │ 简单          │ 简单                     │
│ 典型场景     │ 计数器/标志位 │ 保护复杂数据结构          │
└──────────────┴──────────────┴──────────────────────────┘
```

---

#### 7. 典型使用场景

```go
// ① 并发计数器
var count atomic.Int64

for i := 0; i < 1000; i++ {
    go func() {
        count.Add(1)
    }()
}


// ② 全局开关/标志位
var started atomic.Bool

func Start() {
    if started.CompareAndSwap(false, true) {
        // 只有一个 G 能进入，其他 G CAS 失败
        doStart()
    }
}


// ③ 无锁读多写少的配置热更新
var config atomic.Value

// 写（低频）
func UpdateConfig(cfg *Config) {
    config.Store(cfg)
}

// 读（高频，无锁）
func GetConfig() *Config {
    return config.Load().(*Config)
}


// ④ 自旋锁实现
type SpinLock struct {
    state atomic.Int32
}

func (s *SpinLock) Lock() {
    for !s.state.CompareAndSwap(0, 1) {
        runtime.Gosched()  // 让出 CPU，避免空转
    }
}

func (s *SpinLock) Unlock() {
    s.state.Store(0)
}
```

---

#### 总结

| 操作 | 方法 | 作用 |
|------|------|------|
| 原子读 | `Load` | 读取完整值，不会读到中间态 |
| 原子写 | `Store` | 写入完整值，不会写到一半 |
| 原子加减 | `Add` | 加减操作不可分割 |
| 比较交换 | `CAS` | 条件更新，无锁编程基础 |
| 原子交换 | `Swap` | 设置新值并返回旧值 |

> 原子操作的本质：**利用 CPU 提供的 CMPXCHG 等硬件指令，在不加锁的情况下保证单个变量操作的不可分割性**，是介于普通变量和互斥锁之间的轻量级并发利器。

### 原子操作和锁的区别

#### 原子操作和锁的区别

---

#### 1. 本质区别

```
原子操作：
  依赖 CPU 硬件指令（CMPXCHG、LOCK XADD 等）
  在指令级别保证操作不可分割
  用户态完成，不涉及内核

互斥锁：
  依赖操作系统的线程调度机制
  通过阻塞/唤醒 goroutine 保证互斥
  竞争时需要陷入内核态
```

```
硬件层面示意：

原子操作：
  CPU ──LOCK指令──→ 总线锁定 → 完成操作 → 释放总线
  整个过程在 CPU 层面保证，纳秒级

互斥锁：
  G1 加锁 → 进入临界区 → 解锁 → 唤醒 G2
  涉及 goroutine 挂起/唤醒，微秒级
```

---

#### 2. 操作粒度区别

```go
// 原子操作：只能操作单个简单变量
var count atomic.Int64
count.Add(1)           // ✅ 单个 int64

var flag atomic.Bool
flag.Store(true)       // ✅ 单个 bool

// ❌ 无法原子操作多个变量
// 以下操作不是原子的，存在竞态
count.Add(1)
flag.Store(true)       // 两步之间可能被其他 G 插入


// 互斥锁：可以保护任意复杂的操作
var mu sync.Mutex
mu.Lock()
count++                // ✅ 可以同时操作多个变量
flag = true            // ✅ 整个临界区是原子的
slice = append(slice, 1)
mu.Unlock()
```

---

#### 3. 保护数据类型区别

```
原子操作支持的类型：
  int32、int64、uint32、uint64
  uintptr、unsafe.Pointer
  atomic.Bool、atomic.Value（任意类型，但读写类型需一致）

互斥锁保护的类型：
  任意类型：map、slice、struct、interface
  任意组合：多个变量的联动修改
  任意逻辑：函数调用、条件判断、循环

┌──────────────────────┬────────────┬──────────┐
│      数据类型         │   atomic   │  Mutex   │
├──────────────────────┼────────────┼──────────┤
│ int / uint 系列       │     ✅     │    ✅    │
│ bool                 │     ✅     │    ✅    │
│ pointer              │     ✅     │    ✅    │
│ map                  │     ❌     │    ✅    │
│ slice                │     ❌     │    ✅    │
│ struct（多字段联动）   │     ❌     │    ✅    │
│ 复杂业务逻辑          │     ❌     │    ✅    │
└──────────────────────┴────────────┴──────────┘
```

---

#### 4. 性能区别

```
基准测试对比（单次操作耗时）：

  atomic.AddInt64        ~  2ns   █
  sync.Mutex Lock/Unlock ~ 15ns   ████████
  sync.RWMutex Lock      ~ 25ns   █████████████
  sync.RWMutex RLock     ~ 10ns   █████

无竞争场景：
  atomic  ≈ 普通变量读写（硬件直接支持）
  Mutex   需要 CAS 操作 state 字段，有额外开销

有竞争场景：
  atomic  CAS 失败 → 自旋重试，CPU 空转
  Mutex   竞争失败 → goroutine 挂起，不占 CPU

高并发竞争场景：
  atomic  大量 CAS 失败 → 自旋风暴 → 性能急剧下降
  Mutex   goroutine 有序排队 → 性能平稳下降
```

---

#### 5. 实现机制区别

```
原子操作底层：

  以 AddInt64 为例，Go 编译后对应汇编：
  LOCK XADDQ AX, (BX)

  LOCK  前缀：锁定内存总线，阻止其他 CPU 访问
  XADDQ 指令：交换并相加，一条指令完成

  ┌──────┐  LOCK  ┌──────────┐
  │ CPU0 │───────→│ 内存总线  │← CPU1 被阻塞
  └──────┘        └──────────┘
  硬件保证，不涉及 OS 调度


互斥锁底层：

  Lock() → CAS 尝试获取
         → 失败 → 自旋
         → 自旋失败 → runtime_SemacquireMutex
                    → goroutine 挂起
                    → OS 线程调度介入

  ┌────┐  竞争  ┌──────────────┐  阻塞  ┌──────┐
  │ G1 │───────→│ Mutex state  │───────→│ G2   │
  └────┘        └──────────────┘        │ 挂起  │
                                        └──────┘
  涉及 goroutine 生命周期管理
```

---

#### 6. 编程模型区别

```go
// 原子操作：无锁编程，逻辑分散
var (
    count   atomic.Int64
    ready   atomic.Bool
    version atomic.Int32
)

func update() {
    count.Add(1)
    version.Add(1)
    ready.Store(true)
    // ⚠️ 三个操作之间没有整体原子性
    // 其他 G 可能看到 count 更新了但 ready 还是 false
}


// 互斥锁：有锁编程，逻辑集中
type State struct {
    mu      sync.Mutex
    count   int64
    ready   bool
    version int32
}

func (s *State) update() {
    s.mu.Lock()
    defer s.mu.Unlock()
    s.count++
    s.version++
    s.ready = true
    // ✅ 三个操作整体原子，其他 G 要么全看到，要么全看不到
}
```

---

#### 7. 死锁风险区别

```go
// 原子操作：不会死锁
var count atomic.Int64
count.Add(1)  // 无论如何不会死锁，没有锁的概念


// 互斥锁：使用不当会死锁
var mu sync.Mutex

// ❌ 重入死锁
mu.Lock()
mu.Lock()   // 永久阻塞！Go Mutex 不可重入

// ❌ 循环等待死锁
// G1: 持有 mu1，等待 mu2
// G2: 持有 mu2，等待 mu1
```

---

#### 8. 适用场景对比

```
原子操作适合：

  ① 简单计数器
     var reqCount atomic.Int64
     reqCount.Add(1)  // 请求计数，高频操作

  ② 状态标志位
     var shutdown atomic.Bool
     shutdown.Store(true)  // 优雅退出开关

  ③ 无锁配置热更新
     var cfg atomic.Value
     cfg.Store(newConfig)  // 读多写少的配置

  ④ 自旋锁/无锁队列等底层数据结构


互斥锁适合：

  ① 保护 map
     mu.Lock(); m["key"] = val; mu.Unlock()

  ② 保护多字段结构体的联动修改
     mu.Lock(); s.count++; s.updated = true; mu.Unlock()

  ③ 保护包含 IO 的临界区
     mu.Lock(); write to file; mu.Unlock()

  ④ 读多写少用 RWMutex 优化
     mu.RLock(); read data; mu.RUnlock()
```

---

#### 9. 完整对比总结

```
┌──────────────┬───────────────────────┬───────────────────────┐
│   对比维度    │       原子操作         │       互斥锁           │
├──────────────┼───────────────────────┼───────────────────────┤
│ 实现层次      │ CPU 硬件指令           │ OS 线程调度            │
│ 操作粒度      │ 单个变量               │ 任意复杂操作            │
│ 数据类型      │ 简单值类型             │ 任意类型               │
│ 性能          │ 极高（纳秒级）          │ 较高（微秒级）          │
│ 高竞争表现    │ 自旋风暴，性能下降      │ 有序排队，性能平稳      │
│ 死锁风险      │ 无                    │ 使用不当会死锁          │
│ 整体原子性    │ 不支持多变量            │ 支持任意多变量          │
│ 使用复杂度    │ 简单                  │ 简单                   │
│ 适用场景      │ 计数器/标志位/热更新    │ 复杂数据结构保护        │
└──────────────┴───────────────────────┴───────────────────────┘
```

> 核心结论：**原子操作是 CPU 硬件级的点操作，快但只能管一个变量；互斥锁是 OS 调度级的面操作，稍慢但能管一片逻辑。两者不是替代关系，而是不同粒度下的并发利器，按场景配合使用才是正道。**


### 什么是CAS
#### 什么是 CAS

---

#### 1. CAS 的定义

```
CAS（Compare And Swap，比较并交换）
是一种硬件级别的原子操作，由 CPU 指令直接支持

语义：
  如果内存中的值 == 期望值（old）
  则将内存中的值替换为新值（new）
  返回是否替换成功

伪代码：
  func CAS(addr *int, old, new int) bool {
      if *addr == old {
          *addr = new
          return true   // 交换成功
      }
      return false      // 交换失败，值已被其他 G 修改
  }
  // 以上过程是一条不可分割的 CPU 指令
```

---

#### 2. CAS 的硬件实现

```
x86 架构对应指令：CMPXCHG（Compare and Exchange）

汇编层面：
  LOCK CMPXCHG [addr], newVal

  LOCK    → 锁定内存总线，其他 CPU 无法访问该地址
  CMPXCHG → 比较 + 交换，一条指令完成，不可中断

执行过程：
  ┌─────────────────────────────────────────┐
  │  CPU 执行 CMPXCHG                        │
  │                                         │
  │  ① 读取 addr 的当前值                    │
  │  ② 与 old 比较                           │
  │  ③ 相等 → 写入 new，返回 true            │
  │     不等 → 不写入，返回 false            │
  │                                         │
  │  以上三步在硬件层面不可分割               │
  └─────────────────────────────────────────┘
```

---

#### 3. Go 中的 CAS 操作

```go
import "sync/atomic"

var val int64 = 100

// 旧版写法
swapped := atomic.CompareAndSwapInt64(&val, 100, 200)
// val == 100 → 替换为 200，swapped = true
// val != 100 → 不替换，  swapped = false

// 1.19+ 新版写法
var val atomic.Int64
val.Store(100)
ok := val.CompareAndSwap(100, 200)


// 支持的 CAS 类型
atomic.CompareAndSwapInt32
atomic.CompareAndSwapInt64
atomic.CompareAndSwapUint32
atomic.CompareAndSwapUint64
atomic.CompareAndSwapPointer
```

---

#### 4. CAS 的执行流程

```
场景：G1 和 G2 同时对 val=100 执行 CAS(100, 200)

时间线：
  val = 100

  G1: 读取 val=100，准备 CAS(100→200)
  G2: 读取 val=100，准备 CAS(100→200)

  G1: LOCK CMPXCHG 执行
      val==100 ✅ → val=200，返回 true   ← G1 成功

  G2: LOCK CMPXCHG 执行
      val==200 ❌ → 不修改，返回 false   ← G2 失败

  结果：val=200，只有 G1 成功，G2 感知到竞争失败

核心：LOCK 保证 G1 操作期间 G2 无法介入
```

---

#### 5. CAS 配合自旋实现无锁更新

```go
// CAS 单次失败不代表放弃，配合循环重试实现无锁更新
var count atomic.Int64

func increment() {
    for {
        old := count.Load()          // 读取当前值
        new := old + 1               // 计算新值
        if count.CompareAndSwap(old, new) {
            break                    // CAS 成功，退出循环
        }
        // CAS 失败 → 说明其他 G 已修改 → 重新读取重试
    }
}

// 等价于（内部实现）
atomic.AddInt64(&count, 1)  // Add 底层就是 CAS 自旋
```

```
自旋 CAS 时序：

  G1: Load=0 → CAS(0,1) ✅ → count=1，退出
  G2: Load=0 → CAS(0,1) ❌ → 重试
      Load=1 → CAS(1,2) ✅ → count=2，退出
  G3: Load=0 → CAS(0,1) ❌ → 重试
      Load=0 → CAS(0,1) ❌ → 重试
      Load=2 → CAS(2,3) ✅ → count=3，退出
```

---

#### 6. CAS 在 Go 运行时中的应用

```go
// ① sync.Mutex 加锁
func (m *Mutex) Lock() {
    // 快速路径：CAS 尝试直接获锁
    if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
        return  // 无竞争，直接获锁成功
    }
    // 慢速路径：自旋 + 阻塞
    m.lockSlow()
}


// ② sync.Once 保证只执行一次
func (o *Once) doSlow(f func()) {
    o.m.Lock()
    defer o.m.Unlock()
    if o.done.Load() == 0 {
        defer o.done.Store(1)
        f()
    }
}


// ③ goroutine 状态切换
// G 的状态从 _Grunnable → _Grunning
// 底层通过 CAS 保证状态切换的原子性
atomic.CompareAndSwapUint32(&gp.atomicstatus, _Grunnable, _Grunning)


// ④ channel 操作
// sendx / recvx 索引更新
// 关闭 channel 时 closed 字段的更新
```

---

#### 7. ABA 问题

```
CAS 的经典陷阱：ABA 问题

场景：
  val = A

  G1: 读取 val = A，准备 CAS(A → C)
      （G1 被调度器暂停）

  G2: CAS(A → B) 成功，val = B
  G3: CAS(B → A) 成功，val = A   ← 值被改回 A

  G1: 恢复执行，CAS(A → C)
      val == A ✅ → 成功！

  问题：G1 认为 val 没有变化
        但实际上 val 经历了 A → B → A 的变化
        G1 的 CAS 基于"过期"的认知成功了
```

```
ABA 问题的影响：

  简单计数器：无影响（只关心最终值）

  链表/队列等数据结构：有影响
    节点 A 被删除后又重新插入
    G1 认为队头没变，实际上队列结构已改变
    可能导致数据丢失或结构破坏
```

```go
// ✅ 解决方案：版本号（Stamped）

type StampedValue struct {
    val     int64
    version int64  // 每次修改版本号 +1
}

// CAS 时同时比较值和版本号
// A → B → A 时版本号已经变化
// version: 0 → 1 → 2
// G1 持有的 (A, version=0) 与当前 (A, version=2) 不匹配
// CAS 失败，ABA 问题被规避

// Java 中有 AtomicStampedReference
// Go 中需自行实现或使用 unsafe.Pointer + 版本号
```

---

#### 8. CAS 的优缺点

```
优点：

  ① 无锁：不需要 OS 调度介入，性能极高
  ② 无死锁：没有锁就没有死锁风险
  ③ 非阻塞：失败直接返回，不挂起 goroutine
  ④ 乐观并发：适合冲突少的场景


缺点：

  ① ABA 问题：需要版本号等额外机制解决
  ② 自旋开销：高竞争下 CAS 频繁失败，CPU 空转严重
  ③ 单变量限制：只能保证单个变量的原子性
  ④ 只适合简单操作：复杂逻辑仍需互斥锁
```

---

#### 9. CAS vs 互斥锁 选型

```
低竞争场景（推荐 CAS）：
  操作简单（单个变量）
  临界区极短
  冲突概率低

  atomic.AddInt64(&counter, 1)  ← 纳秒级

高竞争场景（推荐 Mutex）：
  大量 goroutine 同时竞争
  CAS 失败率高，自旋风暴反而更慢
  操作复杂，需要保护多个变量

  mu.Lock(); 复杂操作; mu.Unlock()  ← goroutine 有序排队

经验法则：
  单个数值的并发读写   → atomic（CAS）
  多个变量的联动修改   → Mutex
  读多写少的复杂结构   → RWMutex
```

---

#### 总结

| 特性 | 说明 |
|------|------|
| 定义 | 比较内存值与期望值，相等则替换为新值 |
| 硬件支持 | x86 的 `LOCK CMPXCHG` 指令，一条指令不可分割 |
| 返回值 | bool，表示是否替换成功 |
| 失败处理 | 配合自旋循环重试，实现无锁更新 |
| 经典问题 | ABA 问题，用版本号解决 |
| 适用场景 | 低竞争、单变量、简单操作 |

> CAS 的本质：**用"乐观锁"的思路，假设大多数时候没有竞争，先尝试直接修改，失败了再重试，以此避免了锁带来的上下文切换开销，是无锁编程的基石。**

### SYNC.POOL 有什么用
#### sync.Pool 有什么用

---

#### 1. sync.Pool 的定义

```
sync.Pool 是 Go 标准库提供的对象复用池

核心作用：
  缓存临时对象，减少重复创建和销毁
  降低 GC 压力，提升高频分配场景的性能

本质：
  用空间换时间
  将用完的对象放回池子，下次直接取用
  避免频繁触发 GC
```

---

#### 2. 没有 Pool 时的问题

```go
// 高并发场景下，每次请求都创建新对象
func handleRequest() {
    buf := make([]byte, 4096)  // 每次分配 4KB
    // 使用 buf 处理请求...
    // 函数结束，buf 变成垃圾，等待 GC 回收
}

// 问题：
// 每秒 10000 个请求
// 每次分配 4KB
// 每秒产生 40MB 垃圾
// GC 频繁触发 → STW 停顿 → 延迟飙升
```

```
内存分配压力示意：

  请求1 → 分配 buf → 用完 → 等待GC
  请求2 → 分配 buf → 用完 → 等待GC
  请求3 → 分配 buf → 用完 → 等待GC
  ...
  GC 触发 → 全部回收 → 延迟暴增

  使用 Pool 后：
  请求1 → 从池取 buf → 用完 → 放回池
  请求2 → 从池取 buf → 用完 → 放回池
  请求3 → 从池取 buf → 用完 → 放回池
  ...
  GC 压力大幅降低
```

---

#### 3. 底层结构

```go
type Pool struct {
    noCopy  noCopy         // 禁止复制
    local   unsafe.Pointer // 每个 P 的本地池数组（poolLocal 数组）
    localSize uintptr      // local 数组的大小

    victim     unsafe.Pointer // 上一轮 GC 前的 local（二级缓存）
    victimSize uintptr        // victim 大小

    New func() any         // 池为空时，自动创建新对象的函数
}

// 每个 P 独享一个 poolLocal，避免竞争
type poolLocal struct {
    poolLocalInternal
    pad [128 - unsafe.Sizeof(poolLocalInternal{})%128]byte // 缓存行对齐
}

type poolLocalInternal struct {
    private any       // 只属于当前 P，存一个对象，存取无需加锁
    shared  poolChain // 当前 P 和其他 P 共享，存多个对象，需要加锁
}
```

```
Pool 的三级存储结构：

  ┌─────────────────────────────────────────────┐
  │  P0.private  →  单个对象，无锁，最快          │  一级
  ├─────────────────────────────────────────────┤
  │  P0.shared   →  双端队列，本P推/弹，他P偷     │  二级
  │  P1.shared                                  │
  │  P2.shared   ...                            │
  ├─────────────────────────────────────────────┤
  │  victim      →  上轮 GC 残留，即将被清除      │  三级
  ├─────────────────────────────────────────────┤
  │  New()       →  三级都没有，调用 New 创建     │  兜底
  └─────────────────────────────────────────────┘
```

---

#### 4. Get 的执行流程

```go
func (p *Pool) Get() any {
    // 绑定当前 P，禁止抢占
    l, pid := p.pin()

    // ① 取 private（无锁，最快）
    x := l.private
    l.private = nil

    if x == nil {
        // ② 从本地 shared 队列头部弹出
        x, _ = l.shared.popHead()

        if x == nil {
            // ③ 从其他 P 的 shared 队列尾部偷取（Work Stealing）
            x = p.getSlow(pid)
        }
    }
    runtime_procUnpin()

    // ④ 三级都没有，调用 New 创建新对象
    if x == nil && p.New != nil {
        x = p.New()
    }
    return x
}
```

```
Get 查找顺序：

  private（当前P私有）
      ↓ 没有
  shared 队列头部（当前P）
      ↓ 没有
  其他P的 shared 队列尾部（偷取）
      ↓ 没有
  victim（上轮GC残留）
      ↓ 没有
  New() 创建新对象
```

---

#### 5. Put 的执行流程

```go
func (p *Pool) Put(x any) {
    if x == nil {
        return  // 不放入 nil
    }

    l, _ := p.pin()  // 绑定当前 P

    // ① 优先放入 private（无锁）
    if l.private == nil {
        l.private = x
    } else {
        // ② private 已有值，放入 shared 队列头部
        l.shared.pushHead(x)
    }
    runtime_procUnpin()
}
```

---

#### 6. GC 时 Pool 的清理机制

```
Pool 与 GC 的关系：

  每次 GC 触发时：
  ① 当前 local  → 移动到 victim（降级为二级缓存）
  ② 上轮 victim → 直接清空（对象被 GC 回收）
  ③ 新的 local  → 重新初始化为空

时间线：
  GC第1次：local → victim，victim 清空
  GC第2次：新local → victim，旧victim（原local）清空

  对象在 Pool 中最多存活两轮 GC

Pool 不适合存放需要长期保留的对象！
```

---

#### 7. 基本用法

```go
// 创建 Pool，提供 New 函数
var bufPool = sync.Pool{
    New: func() any {
        return make([]byte, 4096)  // 池为空时自动创建
    },
}

func handleRequest(data []byte) {
    // 从池中取出 buf
    buf := bufPool.Get().([]byte)
    defer bufPool.Put(buf)  // 用完放回池中

    // 使用 buf 处理数据
    copy(buf, data)
    process(buf[:len(data)])
}
```

---

#### 8. 实际场景：bytes.Buffer 复用

```go
var bufferPool = sync.Pool{
    New: func() any {
        return &bytes.Buffer{}
    },
}

func formatJSON(v any) ([]byte, error) {
    buf := bufferPool.Get().(*bytes.Buffer)
    defer func() {
        buf.Reset()           // ✅ 放回前必须重置，清空上次内容
        bufferPool.Put(buf)
    }()

    if err := json.NewEncoder(buf).Encode(v); err != nil {
        return nil, err
    }
    return append([]byte{}, buf.Bytes()...), nil  // 拷贝出来再返回
}
```

---

#### 9. 实际场景：标准库中的应用

```go
// fmt 包大量使用 sync.Pool
// src/fmt/print.go
var ppFree = sync.Pool{
    New: func() any { return new(pp) },
}

func Fprintf(w io.Writer, format string, a ...any) (n int, err error) {
    p := ppFree.Get().(*pp)  // 从池取
    p.init()
    // ... 格式化逻辑
    ppFree.Put(p)            // 用完放回
    return
}

// encoding/json、net/http、go/token 等标准库
// 都大量使用 sync.Pool 复用临时对象
```

---

#### 10. 使用注意事项

```go
// ① 放回前必须重置对象状态
buf := pool.Get().(*bytes.Buffer)
// ❌ 不重置直接放回，下次取出有脏数据
pool.Put(buf)

// ✅ 重置后放回
buf.Reset()
pool.Put(buf)


// ② 不能存放带状态的长期对象
// Pool 两轮 GC 后对象被回收
// ❌ 不适合：数据库连接、文件句柄（用 连接池 代替）
// ✅ 适合：临时 buffer、临时 struct、编解码器


// ③ Get 返回值可能是 nil（未设置 New 时）
obj := pool.Get()
if obj == nil {
    obj = createNew()  // 需要自行处理 nil 情况
}


// ④ 不可复制
pool1 := sync.Pool{New: func() any { return &Obj{} }}
pool2 := pool1  // ❌ 复制 Pool 行为未定义，go vet 会报错


// ⑤ Pool 中的对象大小尽量一致
// 大小不一致导致内存碎片，复用效果差
// ✅ 每种大小单独一个 Pool
var (
    smallPool = sync.Pool{New: func() any { return make([]byte, 1024) }}
    largePool = sync.Pool{New: func() any { return make([]byte, 65536) }}
)
```

---

#### 11. 性能对比

```
基准测试（每次分配 4KB buffer，10000次并发）：

  不用 Pool：
    分配次数：10000 次
    GC 次数：频繁
    耗时：~500ns/op
    内存分配：4KB/op

  使用 Pool：
    分配次数：P 的数量次（约 8 次）
    GC 次数：极少
    耗时：~50ns/op   ← 提升 10x
    内存分配：0B/op  ← 几乎零分配
```

---

#### 总结

| 特性 | 说明 |
|------|------|
| 核心作用 | 复用临时对象，降低 GC 压力 |
| 存储结构 | private + shared + victim 三级缓存 |
| GC 行为 | 每轮 GC 降级，两轮后清空 |
| 适用对象 | 临时 buffer、编解码器等短生命周期对象 |
| 不适用对象 | 数据库连接等需要长期持有的资源 |
| 使用要点 | 放回前重置状态，不可复制，处理 nil |

> sync.Pool 的本质：**以 P 为粒度的分层对象缓存，用 private 消除锁竞争，用 victim 缓冲 GC 抖动，将频繁分配的临时对象变成可复用的资源，是 Go 高性能服务的标配优化手段。**

## 并发安全
### cas  修改一块内存的值，值改变方式是a-b-a这个合理吗
    #### 什么事ABA问题
        - 线程1，期望值为A，欲更新的值为B
        - 线程2，期望值为A，欲更新的值为B
        - 线程1抢先获得CPU时间片，而线程2因为其他原因阻塞
        - 线程1取值与期望的A值比较，发现相等然后将值更新为B
        - 线程3，期望值为B，欲更新的值为A，线程3取值与期望的值B比较，发现相等则将值更新为A
        - 线程2从阻塞中恢复，并且获得了CPU时间片，这时候线程2取值与期望的值A比较，发现相等则将值更新为B
        - 虽然线程2也完成了操作，但是线程2并不知道值已经经过了A->B->A的变化过程
    - 如何解决ABA问题
        - 在变量前面加上版本号，每次变量更新的时候变量的版本号都+1，即A->B->A就变成了1A->2B->3A
