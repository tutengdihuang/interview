
## GMP调度
### GOROUTINE 定义

#### 什么是 Goroutine

Goroutine 是 Go 运行时管理的**用户态轻量级线程**，由 Go 运行时调度，而非操作系统内核调度。

```go
go func() {
    fmt.Println("我是一个 goroutine")
}()
```

#### 与线程的核心区别

| 对比维度 | 线程（Thread） | Goroutine |
|---------|--------------|-----------|
| 创建成本 | 约 1MB 栈内存 | 初始 2~8KB 栈内存 |
| 调度方式 | 内核态调度（切换代价高） | 用户态调度（切换代价极低） |
| 切换开销 | 微秒级（需陷入内核） | 纳秒级（用户态完成） |
| 数量上限 | 受系统资源限制，通常数千 | 可轻松创建百万个 |
| 栈大小 | 固定（通常 8MB） | 动态伸缩（最大 1GB） |

#### 核心特性

**动态栈**：Goroutine 初始栈只有 2~8KB，按需自动扩容和收缩，不会预分配大量内存。

**用户态调度**：由 Go 运行时的调度器（Scheduler）在用户态完成调度，无需陷入内核，切换成本极低。

**与线程的关系**：多个 Goroutine 复用少量的 OS 线程（M），通过 GMP 模型实现 M:N 调度。

```
1000 个 Goroutine 可能只对应 8 个 OS 线程
运行时负责在线程上不断切换 Goroutine
对开发者透明
```

---

### 1.0 之前 GM 调度模型

#### GM 模型结构

Go 1.0 之前只有 G（Goroutine）和 M（Machine/OS线程）两个角色，没有 P：

```
G1  G2  G3  G4  G5  G6      ← 所有 Goroutine
        │
        ▼
  全局 Goroutine 队列（加全局锁）
        │
   ┌────┴────┐
   ▼         ▼
   M1        M2              ← OS 线程直接从全局队列取 G
```

#### GM 模型的三大缺陷

**缺陷一：全局队列的单锁竞争**

所有 M 共享同一个全局 Goroutine 队列，每次取 G 都需要加锁。线程越多，锁竞争越激烈，性能随核数增加反而下降。

```go
// 伪代码：每次调度都要抢这把锁
globalLock.Lock()
g := globalQueue.pop()
globalLock.Unlock()
```

**缺陷二：M 持有内存缓存导致浪费**

每个 M 都持有一份内存缓存（mcache），即使 M 阻塞在系统调用中，这份内存缓存也无法被其他 M 使用，造成大量内存浪费。

**缺陷三：系统调用导致频繁线程切换**

G 发起系统调用时，M 会阻塞，运行时需要新建或唤醒另一个 M 来继续执行其他 G。系统调用频繁时，线程创建和切换的开销非常大。

#### GM 模型的根本问题

```
GM 模型的瓶颈：
  多个 M 争抢全局队列的锁
        ↓
  扩展性极差，无法充分利用多核
```

Go 1.1 引入 P（Processor）角色，将全局队列拆分为每个 P 独立的本地队列，彻底解决了锁竞争问题，形成现在的 GMP 模型。

---

### GMP 中 Work Stealing 机制

#### 背景

GMP 模型中每个 P 都有自己的本地运行队列（Local Run Queue，最多 256 个 G）。当某个 P 的本地队列为空，对应的 M 无事可做时，如果直接休眠会造成 CPU 浪费——此时就需要 Work Stealing。

#### Work Stealing 的执行流程

```
P 的本地队列为空
        │
        ▼
① 先尝试从全局队列取 G
        │ 没有？
        ▼
② 随机选一个其他 P，偷取其本地队列后半部分的 G
        │ 还没有？
        ▼
③ 检查 netpoll（网络轮询器）是否有就绪的 G
        │ 还没有？
        ▼
④ M 休眠，P 与 M 解绑，进入空闲列表
```

#### 偷取数量

每次偷取目标 P 本地队列中**一半**的 G：

```go
// 伪代码
stolen := len(target.runq) / 2
```

偷一半而不是全部，是为了保留局部性——让原 P 的 G 尽量在同一个 M/CPU 上运行，减少缓存失效。

#### Work Stealing 的价值

```
没有 Work Stealing：
  P0 有 100 个 G 堆积    P1 空闲休眠
  → CPU 利用率 50%，P1 的核浪费

有 Work Stealing：
  P1 发现自己空闲 → 偷 P0 的 50 个 G → 两个核都在工作
  → CPU 利用率接近 100%
```

Work Stealing 使 GMP 模型能够**自动均衡负载**，无需开发者手动控制 Goroutine 分配。

---

### GMP 中 Hand Off 机制

#### 背景

当一个 G 发起**系统调用**（如文件 IO、网络阻塞读写）时，对应的 M 会陷入内核阻塞，无法继续执行其他 G。如果 P 一直等待这个 M 恢复，会造成 CPU 浪费。Hand Off（交接）机制解决这个问题。

#### Hand Off 的执行流程

```
G0 发起阻塞系统调用
        │
        ▼
M0 陷入内核阻塞（与 P0 解绑）
        │
        ▼
P0 寻找空闲的 M1（或新建一个 M）
        │
        ▼
P0 与 M1 绑定，继续执行本地队列中其他的 G
        │
        ▼
（M0 系统调用返回后）
M0 尝试获取空闲 P
  ├─ 有空闲 P → M0 绑定该 P，G0 继续执行
  └─ 没有空闲 P → G0 放入全局队列，M0 休眠
```

#### 图示

```
系统调用前：               系统调用发生后（Hand Off）：

P0 — M0 — G0(运行中)       P0 — M1 — G1(继续运行)
          G1                         G2
          G2               M0 — G0(阻塞中，等系统调用返回)
```

#### Hand Off 的价值

没有 Hand Off，一个 G 的系统调用会导致整个 P 的队列停止执行。有了 Hand Off，系统调用对其他 Goroutine 完全透明，CPU 利用率始终保持高位。

---

### 协作式的抢占式调度

#### 什么是协作式抢占

早期 Go（1.13 及之前）的抢占依赖 Goroutine **主动让出**CPU，而不是强制剥夺。运行时在特定的**安全点**插入检查代码，Goroutine 执行到这些位置时检查是否需要让出。

#### 触发时机

协作式抢占的检查点主要是**函数调用入口**：

```go
// 编译器在每个函数入口自动插入类似以下的检查
func someFunc() {
    // 自动插入的抢占检查
    if goroutine.stackguard0 == stackPreempt {
        runtime.morestack() // 触发调度
    }
    // 实际业务代码...
}
```

`sysmon` 监控线程发现某个 G 运行超过 **10ms**，会将其 `stackguard0` 设为 `stackPreempt`，等该 G 下次发生函数调用时自然让出。

#### 协作式抢占的致命缺陷

```go
// 这段代码会导致整个程序卡死（Go 1.13 及之前）
func main() {
    go func() {
        for {
            // 纯计算循环，没有函数调用
            // 永远不会触发抢占检查点
            // 导致其他 Goroutine 无法被调度
        }
    }()
    time.Sleep(time.Second) // 永远无法执行到这里
}
```

没有函数调用的纯计算循环（或汇编代码）会**永久占用 M**，其他 G 得不到调度机会，整个程序陷入假死。

---

### 基于信号的抢占式调度

#### 背景

Go 1.14 引入基于信号的**真正抢占式调度**，彻底解决协作式抢占无法处理纯计算循环的问题。

#### 实现原理

利用操作系统的 `SIGURG` 信号，强制中断正在运行的 Goroutine：

```
sysmon 发现 G 运行超过 10ms
        │
        ▼
向 G 所在的 M 发送 SIGURG 信号
        │
        ▼
M 的信号处理函数 sighandler 被触发
        │
        ▼
在当前 G 的栈上注入一个 asyncPreempt 函数调用
        │
        ▼
G 被强制暂停，保存现场（寄存器、PC 指针等）
        │
        ▼
调度器选择下一个 G 运行
        │
        ▼
被抢占的 G 后续恢复执行时，从保存的现场继续
```

#### 与协作式的对比

| 对比维度 | 协作式抢占 | 基于信号的抢占 |
|---------|-----------|--------------|
| Go 版本 | 1.13 及之前 | 1.14+ |
| 触发方式 | Goroutine 主动让出 | 信号强制中断 |
| 检查点 | 函数调用入口 | 任意位置 |
| 纯计算循环 | 无法抢占（致命缺陷） | 可以抢占 |
| 实现复杂度 | 低 | 高（需保存完整现场） |

#### 为什么选择 SIGURG

Go 选择 `SIGURG`（紧急 IO 信号）而非 `SIGUSR1/SIGUSR2`，是因为：
- `SIGURG` 在绝大多数程序中不被使用，不会干扰业务逻辑
- CGO 代码也不太可能注册 `SIGURG` 处理函数，冲突风险最低

---

### GMP 调度过程中存在哪些阻塞

#### 一、系统调用阻塞

最常见的阻塞类型，触发 Hand Off 机制：

```
文件 IO（read/write）
网络 IO（在非 netpoll 路径上）
syscall.Syscall 调用
CGO 调用
```

M 陷入内核，P 与 M 解绑，寻找新 M 继续执行队列中的其他 G。

#### 二、Channel 阻塞

```go
ch := make(chan int)

// 发送阻塞：没有接收方时
ch <- 1   // G 进入 channel 的 sendq 队列，让出 M

// 接收阻塞：没有发送方时
<-ch      // G 进入 channel 的 recvq 队列，让出 M
```

G 被放入 channel 的等待队列，M 被释放去执行其他 G，不会浪费线程资源。

#### 三、sync 原语阻塞

```go
var mu sync.Mutex
mu.Lock()   // 抢不到锁时，G 进入锁的等待队列，让出 M

var wg sync.WaitGroup
wg.Wait()   // 未完成时，G 阻塞等待

cond.Wait() // sync.Cond 等待条件
```

#### 四、time.Sleep 阻塞

```go
time.Sleep(time.Second)
// G 被放入计时器堆，让出 M
// 到期后由 sysmon 或 netpoll 重新唤醒放入运行队列
```

Sleep 期间 M 完全不阻塞，会去执行其他 G。

#### 五、网络 IO 阻塞（netpoll）

Go 对网络 IO 做了特殊处理，底层使用 epoll/kqueue/IOCP：

```
G 发起网络 IO（如 conn.Read()）
        │
        ▼
数据未就绪 → G 注册到 netpoll，让出 M
        │
        ▼
M 继续执行其他 G
        │
        ▼
数据就绪 → netpoll 唤醒 G，放回运行队列
```

网络 IO 阻塞**不会阻塞 M**，这是 Go 网络性能高的核心原因之一。

#### 六、GC STW 阻塞

GC 的 Mark Setup 和 Mark Termination 阶段会短暂暂停所有 G：

```
所有 G 在最近的安全点停下
等待 STW 完成（通常 < 1ms）
恢复执行
```

#### 阻塞类型总览

| 阻塞类型 | M 是否阻塞 | 处理机制 |
|---------|-----------|---------|
| 系统调用 | 是 | Hand Off，P 转移到新 M |
| Channel | 否 | G 进等待队列，M 执行其他 G |
| sync 锁 | 否 | G 进锁等待队列，M 执行其他 G |
| time.Sleep | 否 | G 进计时器堆，M 执行其他 G |
| 网络 IO | 否 | G 注册 netpoll，M 执行其他 G |
| GC STW | 是（极短） | 所有 G 在安全点暂停 |

---

### SYSMON 有什么作用

#### 什么是 sysmon

`sysmon`（System Monitor）是 Go 运行时的**后台监控线程**，独立于 GMP 体系之外，**不需要绑定 P** 就能运行，是运行时的"守护进程"。

```
GMP 体系：  G - P - M（需要 P 才能运行）

sysmon：    独立 M，无需 P，优先级最高，永不休眠
```

#### sysmon 的六大职责

**职责一：抢占长时间运行的 Goroutine**

每隔 **10ms** 检查一次，发现运行超过 10ms 的 G：
- 1.13 及之前：设置抢占标志，等待 G 下次函数调用时让出
- 1.14 及之后：发送 `SIGURG` 信号，强制抢占

**职责二：处理长时间阻塞在系统调用的 M**

```
M 陷入系统调用超过 20μs
        │
        ▼
sysmon 检测到该 M 超时
        │
        ▼
触发 Hand Off：将 P 从阻塞的 M 上抢走
将 P 交给其他 M，保证 CPU 不空闲
```

**职责三：定时触发 GC**

距离上次 GC 超过 **2 分钟**，强制触发一次 GC，防止长时间不回收垃圾。

**职责四：netpoll 网络轮询**

定期调用 `netpoll`，将已经就绪的网络 IO 对应的 G 重新放入运行队列：

```
sysmon 每隔 10ms 调用 netpoll(0)
检查是否有网络事件就绪
将就绪的 G 注入全局运行队列
```

**职责五：归还内存给操作系统（scavenge）**

定期检查空闲的 mspan，将长时间未使用的物理内存通过 `madvise(MADV_DONTNEED)` 归还给操作系统，降低程序的 RSS（常驻内存）。

**职责六：计时器管理**

检查到期的 `time.Sleep` / `time.Timer`，将对应的 G 从计时器堆中取出，放回运行队列等待调度。

#### sysmon 的工作频率

sysmon 并非固定频率运行，而是**自适应休眠**：

```
初始休眠 20μs
若无事发生 → 逐步增加休眠时间，最长 10ms
若有任务（抢占、GC、netpoll）→ 立即缩短休眠间隔
```

这样在程序空闲时 sysmon 几乎不消耗 CPU，在繁忙时能快速响应。

#### sysmon 职责总览

| 职责 | 触发条件 | 作用 |
|------|---------|------|
| 抢占调度 | G 运行 > 10ms | 防止单个 G 长期占用 M |
| 系统调用超时 | M 阻塞 > 20μs | Hand Off，保证 P 不空闲 |
| 强制 GC | 距上次 GC > 2min | 防止垃圾长期堆积 |
| netpoll 轮询 | 每 10ms | 唤醒就绪的网络 IO goroutine |
| 内存归还 | 空闲内存超时 | 降低进程 RSS |
| 计时器检查 | 定时器到期 | 唤醒 Sleep/Timer 的 goroutine |

## GMP进阶
### goroutine是什么，怎么执行
    - goroutine是比线程还轻量的执行单位，是用户层面的
    - 一个gourontine大约3kb左右
    - 上下文切换成本小
    - goroutine GMP模型，M：N模型
    - 如果可以聊聊goroutine的生老病死
### goroutine切换的原理
    - 网络io阻塞主动切换，cpu占用时间过长信号切换，锁，channel

### GMP模型？全局队列没有g了，怎么办
    - 去其他p的g队列偷取
### GMP 什么时候会创建新的 M，创建有数量限制吗

#### 触发创建新 M 的时机

Go 运行时在以下四种情况下会创建新的 M：

**情况一：有可运行的 G 但没有空闲 M**

```
全局或本地队列中有 G 等待运行
所有 M 都在忙碌或阻塞
没有空闲 M 可以复用
        ↓
运行时创建新的 M
```

**情况二：M 陷入系统调用，P 需要新 M 接管**

Hand Off 机制触发时，P 从阻塞的 M 上解绑，寻找空闲 M：

```
空闲 M 列表（sched.midle）中有空闲 M？
  └─ 有 → 唤醒空闲 M，与 P 绑定
  └─ 无 → 创建新 M
```

**情况三：新建 Goroutine 时**

`go func()` 创建新 G 后，运行时检查是否有空闲 P 和 M，如果 P 有 G 可运行但没有 M，则创建新 M。

**情况四：CGO 调用**

CGO 调用会占用一个 M，且该 M 在 CGO 期间不能被复用，运行时可能需要额外创建 M 来维持 Go 代码的运行。

#### 数量限制

有限制，默认上限是 **10000 个 M**：

```go
// src/runtime/proc.go
var (
    sched.maxmcount int32 = 10000  // M 的最大数量
)
```

超过上限时程序会抛出异常并崩溃：

```
runtime: program exceeds 10000-thread limit
fatal error: thread exhaustion
```

可以通过 `debug.SetMaxThreads(n)` 修改上限：

```go
import "runtime/debug"
debug.SetMaxThreads(20000) // 调高上限（谨慎使用）
```

#### M 的生命周期

M 并非用完即销毁，而是会被复用：

```
M 完成工作 → 放入空闲列表（sched.midle）→ 等待被再次唤醒
长时间无工作 → 运行时回收（但不保证立即销毁）
```

实际运行中活跃的 M 数量通常等于 `GOMAXPROCS`，远小于 10000 的上限。

---

### 阻塞的 G 和 M 绑定之后就会去寻找新的 M 吗

#### 准确描述：不是 G 和 M 绑定，而是 P 和 M 绑定

阻塞时的主角是 **P**，不是 G。流程如下：

```
G 发起阻塞操作（系统调用）
        │
        ▼
M 陷入阻塞（M 与 G 仍绑定，等待内核返回）
        │
        ▼
P 从 M 上解绑（P 不等 M）
        │
        ▼
P 寻找新的 M 来继续执行本地队列中其他的 G：
  ├─ sched.midle 有空闲 M → 唤醒空闲 M
  └─ 没有空闲 M → 创建新 M
        │
        ▼
P 与新 M 绑定，继续运行
```

#### 系统调用返回后

阻塞的 M 系统调用返回时，M 上的 G 需要继续执行，此时：

```
系统调用返回
        │
        ▼
M 尝试获取一个空闲 P：
  ├─ 有空闲 P → M 与 P 绑定，G 继续运行
  └─ 没有空闲 P → G 放入全局运行队列，M 进入空闲列表休眠
```

#### 非系统调用的阻塞（Channel、锁等）

这类阻塞不会触发寻找新 M 的逻辑：

```
G 因 channel/lock 阻塞
        │
        ▼
G 让出 M（G 进入等待队列）
M 和 P 仍然绑定，直接从本地队列取下一个 G 运行
（M 完全不阻塞，无需寻找新 M）
```

#### 总结

| 阻塞类型 | P 的行为 | M 的行为 | 是否寻找新 M |
|---------|---------|---------|------------|
| 系统调用 | 与 M 解绑，寻找新 M | 随 G 阻塞在内核 | 是 |
| Channel/锁 | 不解绑，继续运行 | 不阻塞，切换下一个 G | 否 |
| netpoll | 不解绑，继续运行 | 不阻塞，G 注册到 netpoll | 否 |

---

### Go 的 GMP 模型？P 和 M 的数量怎么决定？如果在 K8S 容器部署，P 和 M 又会有什么不同？

#### GMP 模型简介

```
G（Goroutine）：用户态轻量级线程，执行具体任务
M（Machine）  ：OS 线程，真正在 CPU 上执行代码
P（Processor）：调度上下文，持有本地运行队列，连接 G 和 M

关系：
  P 持有一组 G（本地队列）
  M 必须绑定一个 P 才能执行 G
  一个 P 同时只绑定一个 M
```

```
本地队列          本地队列
[G G G G]        [G G G G]
    │                 │
    P                 P
    │                 │
    M                 M
   (OS线程)          (OS线程)

         全局队列 [G G G G G]
```

#### P 的数量

P 的数量由 `GOMAXPROCS` 决定，**默认等于 CPU 核数**：

```go
// 自动设置为 CPU 核数
runtime.GOMAXPROCS(0)          // 0 表示查询当前值
runtime.GOMAXPROCS(4)          // 手动设置为 4
os.Setenv("GOMAXPROCS", "4")   // 环境变量方式
```

P 的数量决定了**最大并行度**，即最多有多少个 G 同时在不同 CPU 核上运行。

#### M 的数量

M 的数量是动态的，由运行时按需创建：

```
活跃 M 数量 ≈ GOMAXPROCS（正常情况）
阻塞 M 数量 = 正在执行系统调用的 G 数量
M 上限      = 10000（默认）
```

#### K8S 容器部署的问题

在 K8S 中，容器通常使用 CPU **Limit** 限制资源，例如：

```yaml
resources:
  requests:
    cpu: "1"
  limits:
    cpu: "2"    # 最多使用 2 个 CPU
```

但节点实际可能有 **64 个核**，`runtime.NumCPU()` 读取的是宿主机的核数：

```
宿主机：64 核
容器 CPU Limit：2 核

GOMAXPROCS 默认值 = runtime.NumCPU() = 64
实际可用 CPU = 2

结果：
  64 个 P 争抢 2 个 CPU 配额
  大量 P 得不到 CPU 时间
  调度开销激增，goroutine 延迟上升
  频繁的上下文切换反而降低性能
```

#### K8S 下的正确做法

使用 `uber-go/automaxprocs` 自动感知容器的 CPU 配额：

```go
import _ "go.uber.org/automaxprocs"

// 程序启动时自动执行：
// 读取 cgroup CPU quota
// 将 GOMAXPROCS 设置为容器实际可用的 CPU 数
// 输出日志：maxprocs: Updating GOMAXPROCS=2: determined from CPU quota
```

或手动设置：

```go
// 容器内读取实际 CPU 配额
func getContainerCPU() int {
    // 读取 /sys/fs/cgroup/cpu/cpu.cfs_quota_us
    // 除以 /sys/fs/cgroup/cpu/cpu.cfs_period_us
    // 得到实际 CPU 核数
}
runtime.GOMAXPROCS(getContainerCPU())
```

#### P 和 M 在容器中的差异总结

| 场景 | P 数量 | M 活跃数 | 问题 |
|------|--------|---------|------|
| 物理机（64核） | 64 | ~64 | 无问题 |
| 容器 Limit=2，未修复 | 64 | ~64 | 严重调度开销 |
| 容器 Limit=2，已修复 | 2 | ~2 | 正常 |

---

### Goroutine 的亲缘性怎么体现出来

#### 什么是 Goroutine 亲缘性

亲缘性（Affinity）指 Goroutine 倾向于在**同一个 P/M（CPU）上持续运行**，而不是每次调度都随机分配到不同的 CPU 核上，目的是充分利用 CPU 缓存（L1/L2 Cache）。

#### 体现一：G 优先在创建它的 P 上运行

```go
// 在 P0 上运行的 G 执行了 go func()
go someFunc()

// 新建的 G 优先放入当前 P0 的本地队列 runnext 槽位
// 而不是全局队列或其他 P 的队列
```

`runnext` 是每个 P 的特殊槽位，**下次调度优先取这个槽位的 G**，保证父子 Goroutine 在同一个 P 上运行，共享 CPU 缓存中的热数据。

#### 体现二：G 从系统调用返回后优先回原来的 P

```
G 系统调用返回
        │
        ▼
优先尝试获取之前绑定的 P（oldP）
  ├─ oldP 空闲 → 直接绑定，G 继续在同一个 P 上运行
  └─ oldP 忙碌 → 才去抢其他空闲 P
```

#### 体现三：Work Stealing 只偷一半

Work Stealing 每次只偷目标 P 本地队列的**一半**，而不是全部。保留另一半在原 P 上运行，维持局部性，避免相关联的 G 被打散到不同 CPU 上。

#### 体现四：本地队列优先于全局队列

```
调度顺序：
① runnext（最高优先级，当前 P 上次放入的 G）
② 本地队列（当前 P 的 runq）
③ 全局队列（每 61 次调度检查一次）
④ Work Stealing（其他 P 的队列）
```

全局队列的低优先级确保 G 尽量在同一个 P 上流转。

#### 亲缘性的局限

Go 目前**没有提供 API 让开发者手动绑定 Goroutine 到特定 CPU**。如果业务上确实需要强亲缘性（如 NUMA 架构优化），需要通过 CGO 调用 `pthread_setaffinity_np` 或使用第三方库。

---

### Golang 中需要使用协程池吗？为什么？

#### 结论：大多数场景不需要

Go 的 Goroutine 本身已经很轻量，协程池能解决的问题运行时大多已经处理好了。

#### Goroutine 的实际开销

```
创建一个 Goroutine：
  内存：初始栈 2~8KB
  时间：约 2~5μs（用户态，无系统调用）

对比线程池的必要性：
  线程创建：约 1MB 内存 + 数十μs（需陷入内核）
  → 线程必须池化复用

Goroutine 创建：2KB + 2μs
  → 大多数场景直接 go func() 就够了
```

#### 需要协程池的场景

**场景一：需要严格限制并发数量**

```go
// 不限制并发：100 万个请求同时到来，创建 100 万个 goroutine
// 虽然每个只有 2KB，但 100 万 × 2KB = 2GB，OOM 风险
for _, req := range requests {
    go handle(req) // 危险
}

// 用 channel 做信号量限制并发数
sem := make(chan struct{}, 100) // 最多 100 个并发
for _, req := range requests {
    sem <- struct{}{}
    go func(r Request) {
        defer func() { <-sem }()
        handle(r)
    }(req)
}
```

**场景二：任务创建极其频繁，追求极致性能**

每秒百万次以上的任务创建，协程池可以节省反复创建/销毁 goroutine 的开销（虽然很小）。使用 `ants` 等库：

```go
import "github.com/panjf2000/ants/v2"

pool, _ := ants.NewPool(100)
defer pool.Release()

pool.Submit(func() {
    handle(req)
})
```

#### 不需要协程池的场景

```
普通 HTTP 服务（每个请求一个 goroutine）       → 不需要
数据库查询、RPC 调用                           → 不需要
后台定时任务                                   → 不需要
消息消费（已有消费者数量限制）                  → 不需要
```

#### 两种限制并发的方式对比

| 方式 | 适用场景 | 代码复杂度 |
|------|---------|-----------|
| channel 信号量 | 简单限流，无需复用 goroutine | 低 |
| ants 协程池 | 极高频任务，需要复用 | 中 |
| 直接 go func() | 并发数可控的常规场景 | 最低 |

#### 一句话总结

> Go 的 Goroutine 足够轻量，**限制并发数**才是真正的需求，协程池只是实现手段之一，channel 信号量往往更简单直接。

---

### Goroutine 为什么不设置 ID

#### Go 官方的明确态度

Go 官方**故意**不提供 Goroutine ID，这是一个设计决策而非遗漏。

#### 原因一：防止滥用 Thread Local Storage（TLS）模式

其他语言（Java、Python）中，线程 ID 通常配合 Thread Local Storage 使用，将状态绑定到特定线程：

```java
// Java 中常见模式
ThreadLocal<User> currentUser = new ThreadLocal<>();
currentUser.set(user); // 绑定到当前线程
// 后续代码通过 threadId 取出
```

如果 Go 提供 Goroutine ID，开发者很自然地会写出类似代码：

```go
// 如果有 goroutineID，开发者可能会这样写
var goroutineLocalStorage = map[int64]interface{}{}
goroutineLocalStorage[getGoroutineID()] = someState
```

**这在 Go 中是危险的**，因为 Goroutine 会被运行时迁移到不同的 M（OS线程）上，"线程本地"存储的概念在 Go 里根本不成立。

#### 原因二：破坏 Goroutine 的轻量性设计

Goroutine ID 需要额外的存储和管理开销，与 Go "让 Goroutine 尽可能轻量"的目标相悖。

#### 原因三：鼓励正确的并发模式

Go 的并发哲学是：

> **通过 channel 通信共享数据，而不是通过共享数据进行通信。**

Goroutine ID 会诱导开发者用"找到特定 goroutine，给它发消息"的方式编程，而不是通过 channel 解耦。

#### 实际需要 ID 的场景怎么办

如果确实需要追踪（如日志关联），正确做法是用 `context` 传递：

```go
// 正确：通过 context 传递请求 ID
func handleRequest(ctx context.Context, req Request) {
    requestID := ctx.Value("requestID")
    log.Printf("[%s] handling request", requestID)

    // 传递给子 goroutine
    go func() {
        subTask(ctx) // ctx 携带 requestID，不依赖 goroutine ID
    }()
}
```

#### 如果强行获取 Goroutine ID

技术上可以通过解析 `runtime.Stack()` 的输出获取，但官方明确反对：

```go
// 可以但不推荐，性能差，且官方不保证格式稳定
func getGoroutineID() int64 {
    var buf [64]byte
    n := runtime.Stack(buf[:], false)
    // 解析 "goroutine 18 [running]:" 中的数字
    ...
}
```

---

### 线程模型有哪些？为什么 Go Scheduler 需要实现 M:N 的方案？Go Scheduler 由哪些元素构成？

#### 三种线程模型

**模型一：N:1（多用户线程 : 1 内核线程）**

```
用户线程  G1 G2 G3 G4 G5
               │
           用户态调度器
               │
           内核线程 M1
               │
              CPU
```

| 优点 | 缺点 |
|------|------|
| 线程切换在用户态完成，极快 | 无法利用多核（永远只有 1 个内核线程） |
| 无需系统调用 | 一个线程阻塞，所有线程阻塞 |

**模型二：1:1（1 用户线程 : 1 内核线程）**

```
G1 → M1 → CPU核1
G2 → M2 → CPU核2
G3 → M3 → CPU核3
```

| 优点 | 缺点 |
|------|------|
| 充分利用多核 | 线程创建/切换需陷入内核，开销大 |
| 一个阻塞不影响其他 | 线程数受系统资源限制，无法创建大量线程 |

这是 Java 线程、Pthread 采用的模型。

**模型三：M:N（M 用户线程 : N 内核线程）**

```
G1 G2 G3 G4 G5 G6 G7 G8   （M 个用户线程）
         用户态调度器
      M1    M2    M3        （N 个内核线程）
      │     │     │
    CPU1  CPU2  CPU3
```

| 优点 | 缺点 |
|------|------|
| 充分利用多核 | 实现复杂 |
| 用户态调度，切换快 | 用户态和内核态调度需协调 |
| 可创建大量用户线程 | |

#### 为什么 Go 选择 M:N

```
N:1 的问题：无法多核并行 → Go 需要多核
1:1 的问题：线程开销大 → Go 需要百万级 Goroutine
M:N 的方案：两者兼得
  ✓ 少量 OS 线程（M≈CPU核数）充分利用多核
  ✓ 大量 Goroutine 在用户态调度，创建/切换开销极低
  ✓ 系统调用阻塞时 Hand Off，不影响其他 Goroutine
```

#### Go Scheduler 的构成元素

**核心角色：G、M、P**

```
G（Goroutine）
  ├─ 栈空间（初始 2~8KB，动态伸缩）
  ├─ 状态（running/runnable/waiting/dead）
  ├─ 程序计数器（PC）
  └─ goroutine 函数及参数

M（Machine / OS Thread）
  ├─ 绑定的 P（当前执行上下文）
  ├─ 当前运行的 G
  ├─ g0（调度用特殊 goroutine，使用系统栈）
  └─ 线程本地存储（TLS）

P（Processor）
  ├─ 本地运行队列 runq（最多 256 个 G）
  ├─ runnext（下次优先运行的 G）
  ├─ mcache（内存分配缓存）
  └─ 状态（idle/running/syscall/dead）
```

**全局数据结构：schedt**

```go
// src/runtime/runtime2.go
type schedt struct {
    lock mutex

    midle        muintptr  // 空闲 M 链表
    nmidle       int32     // 空闲 M 数量
    maxmcount    int32     // M 上限（默认 10000）

    pidle        puintptr  // 空闲 P 链表
    npidle       uint32    // 空闲 P 数量

    runq         gQueue    // 全局运行队列
    runqsize     int32     // 全局队列大小

    gcwaiting    uint32    // GC 是否在等待运行
}
```

**特殊 Goroutine：g0**

每个 M 都有一个特殊的 `g0`，使用操作系统原生栈（而非 Go 动态栈），专门用于执行调度逻辑：

```
G 需要被调度时：
  M 切换到 g0 的栈上
  g0 执行 schedule() 函数，选择下一个 G
  g0 切换到新 G 的栈上，执行业务代码
```

**网络轮询器：netpoll**

```
封装了 epoll（Linux）/ kqueue（macOS）/ IOCP（Windows）
非阻塞处理网络 IO
就绪的 G 由 sysmon 或调度器周期性注入运行队列
```

#### 完整调度元素图

```
┌─────────────────────────────────────────┐
│              schedt（全局调度器）          │
│  全局队列 [G G G G]   空闲P列表   空闲M列表│
└────────────────┬────────────────────────┘
                 │
    ┌────────────┼────────────┐
    ▼            ▼            ▼
┌───────┐   ┌───────┐   ┌───────┐
│   P   │   │   P   │   │   P   │
│[G G G]│   │[G G G]│   │[G G G]│  ← 本地队列
└───┬───┘   └───┬───┘   └───┬───┘
    │            │            │
    M            M            M      ← OS 线程
    │            │            │
  CPU核         CPU核        CPU核

              sysmon（独立 M，无需 P）
              netpoll（网络事件驱动）
```
