
## 面试题汇总
### 1.项目中用到的锁
### 2.介绍一下线程安全的共享内存方式
### 3.介绍一下goroutine
### 4.goroutine的自旋占用资源如何解决,gmp
### 5.介绍Linux系统信号

#### 介绍 Linux 系统信号

---

#### 1. 什么是系统信号

```
信号（Signal）是 Linux 提供的一种异步通知机制
用于通知进程发生了某种事件

本质：
  内核 → 进程 的软件中断
  进程收到信号后中断当前执行，转去处理信号
  处理完毕后恢复原来的执行流程

类比：
  信号 ≈ 硬件中断的软件模拟
  就像手机来电打断你正在做的事情
  接完电话再继续之前的工作
```

---

#### 2. 信号的来源

```
┌──────────────────────────────────────────────────┐
│               信号来源                            │
├──────────────┬───────────────────────────────────┤
│  硬件异常     │ 除零错误 → SIGFPE                  │
│              │ 非法内存访问 → SIGSEGV              │
│              │ 非法指令 → SIGILL                   │
├──────────────┼───────────────────────────────────┤
│  用户操作     │ Ctrl+C → SIGINT                    │
│              │ Ctrl+Z → SIGTSTP                   │
│              │ Ctrl+\ → SIGQUIT                   │
├──────────────┼───────────────────────────────────┤
│  内核发送     │ 子进程退出 → SIGCHLD               │
│              │ 定时器到期 → SIGALRM                │
│              │ 进程超出资源限制 → SIGXCPU           │
├──────────────┼───────────────────────────────────┤
│  进程发送     │ kill() 系统调用                     │
│              │ raise() 给自己发信号                 │
│              │ kill 命令行工具                      │
└──────────────┴───────────────────────────────────┘
```

---

#### 3. 常见信号一览

```
标准信号（1~31）：

┌────────┬──────────┬────────────┬──────────────────────────────┐
│ 信号值  │  名称    │  默认行为   │           触发场景            │
├────────┼──────────┼────────────┼──────────────────────────────┤
│   1    │ SIGHUP   │ 终止进程   │ 终端断开、进程组首进程退出      │
│   2    │ SIGINT   │ 终止进程   │ Ctrl+C 中断                   │
│   3    │ SIGQUIT  │ 终止+Core  │ Ctrl+\ 退出并生成 core dump   │
│   4    │ SIGILL   │ 终止+Core  │ 执行非法 CPU 指令              │
│   5    │ SIGTRAP  │ 终止+Core  │ 断点/调试陷阱                  │
│   6    │ SIGABRT  │ 终止+Core  │ abort() 函数调用               │
│   7    │ SIGBUS   │ 终止+Core  │ 内存总线错误，地址未对齐        │
│   8    │ SIGFPE   │ 终止+Core  │ 浮点/整数除零错误              │
│   9    │ SIGKILL  │ 终止进程   │ 强制杀死，不可捕获不可忽略      │
│  10    │ SIGUSR1  │ 终止进程   │ 用户自定义信号1                │
│  11    │ SIGSEGV  │ 终止+Core  │ 非法内存访问（段错误）          │
│  12    │ SIGUSR2  │ 终止进程   │ 用户自定义信号2                │
│  13    │ SIGPIPE  │ 终止进程   │ 向无读端的管道写入             │
│  14    │ SIGALRM  │ 终止进程   │ alarm() 定时器超时             │
│  15    │ SIGTERM  │ 终止进程   │ 默认终止信号（可捕获）          │
│  17    │ SIGCHLD  │ 忽略       │ 子进程状态变化                 │
│  18    │ SIGCONT  │ 继续执行   │ 恢复被停止的进程               │
│  19    │ SIGSTOP  │ 暂停进程   │ 强制暂停，不可捕获不可忽略      │
│  20    │ SIGTSTP  │ 暂停进程   │ Ctrl+Z 终端暂停                │
└────────┴──────────┴────────────┴──────────────────────────────┘
```

---

#### 4. 信号的三种默认行为

```
Term（终止）：
  进程收到信号后直接退出
  例：SIGINT、SIGTERM、SIGHUP

Core（终止 + Core Dump）：
  进程退出并生成 core 文件（内存快照）
  用于 gdb 等调试工具事后分析崩溃原因
  例：SIGSEGV、SIGFPE、SIGQUIT

Stop（暂停）：
  进程挂起，等待 SIGCONT 恢复
  例：SIGSTOP、SIGTSTP

Cont（继续）：
  恢复被 Stop 的进程
  例：SIGCONT

Ign（忽略）：
  信号被丢弃，进程无感知
  例：SIGCHLD、SIGURG
```

---

#### 5. 信号的处理方式

```go
// 进程对信号有三种处理选择：

// ① 默认处理（SIG_DFL）
// 执行内核定义的默认行为（终止/忽略/暂停）

// ② 忽略信号（SIG_IGN）
signal.Ignore(syscall.SIGPIPE)  // 忽略 SIGPIPE

// ③ 自定义处理函数（Signal Handler）
sigs := make(chan os.Signal, 1)
signal.Notify(sigs, syscall.SIGINT, syscall.SIGTERM)
go func() {
    sig := <-sigs
    fmt.Println("收到信号:", sig)
    // 执行清理逻辑...
}()

// ⚠️ 注意：SIGKILL 和 SIGSTOP 不可被捕获、忽略、自定义
// 这两个信号由内核强制执行，任何程序都无法阻止
```

---

#### 6. 不可捕获的信号

```
SIGKILL（9）和 SIGSTOP（19） 是两个特殊信号：

SIGKILL：
  立即强制终止进程
  进程无法捕获、无法忽略、无法阻塞
  内核直接回收资源
  使用场景：进程无响应时强制清理

  kill -9 <pid>   ← 发送 SIGKILL

SIGSTOP：
  立即强制暂停进程
  进程无法捕获、无法忽略、无法阻塞
  等待 SIGCONT 恢复

对比 SIGTERM（15）：
  可以被捕获 → 程序可以做清理再退出
  是优雅退出的标准方式
  kill <pid>  ← 默认发送 SIGTERM

优雅退出流程：
  发送 SIGTERM → 进程捕获 → 执行清理 → 主动退出
       ↓ 超时仍未退出
  发送 SIGKILL → 强制终止
```

---

#### 7. 信号发送方式

```bash
# 命令行发送信号
kill -<信号> <pid>
kill -15 1234      # 发送 SIGTERM
kill -9  1234      # 发送 SIGKILL
kill -s SIGTERM 1234

# 发送信号给进程组
kill -15 -1234     # 负号表示进程组 ID

# 查看所有信号
kill -l

# 快捷键
Ctrl+C  → SIGINT  （中断）
Ctrl+Z  → SIGTSTP （暂停）
Ctrl+\  → SIGQUIT （退出+Core）
```

```c
// C 语言发送信号
#include <signal.h>

kill(pid, SIGTERM);   // 向指定进程发送信号
raise(SIGUSR1);       // 向自己发送信号
killpg(pgrp, SIGINT); // 向进程组发送信号
```

---

#### 8. Go 中的信号处理

```go
package main

import (
    "fmt"
    "os"
    "os/signal"
    "syscall"
    "time"
)

func main() {
    sigs := make(chan os.Signal, 1)

    // 注册要监听的信号
    signal.Notify(sigs,
        syscall.SIGINT,   // Ctrl+C
        syscall.SIGTERM,  // kill
        syscall.SIGHUP,   // 终端断开
        syscall.SIGUSR1,  // 用户自定义
    )

    go func() {
        for sig := range sigs {
            switch sig {
            case syscall.SIGINT, syscall.SIGTERM:
                fmt.Println("收到退出信号，开始优雅退出...")
                cleanup()
                os.Exit(0)

            case syscall.SIGHUP:
                fmt.Println("收到 SIGHUP，重新加载配置...")
                reloadConfig()

            case syscall.SIGUSR1:
                fmt.Println("收到 SIGUSR1，自定义处理...")
            }
        }
    }()

    // 模拟主程序运行
    for {
        fmt.Println("running...")
        time.Sleep(time.Second)
    }
}

func cleanup() { fmt.Println("清理资源...") }
func reloadConfig() { fmt.Println("重载配置...") }
```

---

#### 9. 信号与 Go 运行时的关系

```
Go 运行时内部使用信号实现多种机制：

SIGPROF  → pprof 性能分析采样
SIGSEGV  → 捕获非法内存访问，转为 Go panic
SIGBUS   → 捕获内存对齐错误，转为 Go panic
SIGFPE   → 捕获除零错误，转为 Go panic
SIGPIPE  → 向已关闭连接写入时的处理

Go 抢占调度（1.14+）：
  使用 SIGURG 信号实现异步抢占
  向运行中的 G 发送 SIGURG
  强制 G 在安全点让出 CPU
  解决了早期版本 G 无法被抢占的问题

⚠️ 注意：
  Go 运行时已注册了部分信号的处理
  业务代码监听这些信号需谨慎
  避免与运行时的信号处理冲突
```

---

#### 10. 实际应用场景

```
① 优雅退出（最常见）
  监听 SIGTERM / SIGINT
  收到信号后：停止接收新请求 → 等待处理中请求完成 → 关闭资源 → 退出

② 配置热更新
  监听 SIGHUP
  收到信号后重新读取配置文件，无需重启服务

③ 日志切割（logrotate）
  监听 SIGUSR1
  收到信号后关闭旧日志文件，打开新日志文件

④ 动态调试
  监听 SIGUSR2
  收到信号后输出当前 goroutine 堆栈 / 性能数据

⑤ 进程守护
  监听 SIGCHLD
  父进程感知子进程退出，及时回收资源避免僵尸进程
```

---

#### 总结

| 信号 | 值 | 可捕获 | 典型用途 |
|------|----|--------|---------|
| SIGINT | 2 | ✅ | Ctrl+C 中断程序 |
| SIGTERM | 15 | ✅ | 优雅退出标准信号 |
| SIGKILL | 9 | ❌ | 强制终止，不可阻止 |
| SIGSTOP | 19 | ❌ | 强制暂停，不可阻止 |
| SIGHUP | 1 | ✅ | 配置热更新 |
| SIGSEGV | 11 | ✅ | 段错误，内存非法访问 |
| SIGUSR1/2 | 10/12 | ✅ | 用户自定义逻辑 |
| SIGURG | 23 | ✅ | Go 运行时抢占调度 |

> 信号的本质：**内核提供的异步事件通知机制，是进程间通信最轻量的方式，优雅退出、热更新、调试采样等生产级特性都依赖信号机制实现。**


### 6.goroutine抢占时机,gc栈扫描
对，协作式抢占的**表格内容**丢失了，下面是完整版：

```markdown
#### Go goroutine 抢占时机 & GC 栈扫描

##### 一、Goroutine 抢占时机
Go 的调度器使用协作式 + 异步抢占两种机制（Go 1.14+ 引入异步抢占）。

###### 1. 协作式抢占（主动让出）

| 时机 | 说明 |
|------|------|
| `runtime.Gosched()` | 显式让出 CPU |
| 函数调用序言（function prologue） | 编译器在函数入口插入 `morestack` 检查，判断是否需要扩栈或抢占 |
| channel 阻塞 | goroutine 写/读 channel 无法继续时 |
| 系统调用 | 进入 syscall 时 M 与 P 分离 |
| `time.Sleep` | 挂起当前 goroutine |
| GC STW | 所有 goroutine 到达安全点后停止 |

\```go
// 编译器在函数入口插入类似如下的检查
TEXT runtime·morestack(SB),NOSPLIT,$0
    // 检查 stackguard0 是否被设置为 stackPreempt
    // 如果是，则触发抢占
\```

###### 2. 异步抢占（Go 1.14+）
* `sysmon` 后台线程每 10ms 扫描一次所有 P
* 若某 goroutine 运行超过 10ms，向对应的 M 发送 `SIGURG` 信号
* 信号处理函数将 goroutine 标记为可抢占，在下一个安全点挂起

\```
sysmon goroutine
    └── retake()
         └── preemptone()
              └── 发送 SIGURG → asyncPreempt()
                   └── mcall(gopreempt_m)
                        └── goroutine 进入 _Grunnable 状态
\```

##### 二、GC 栈扫描
GC 需要扫描每个 goroutine 的栈，找出所有存活的指针（根对象）。

###### 1. 为什么要扫描栈？
\```
堆对象 ← 栈上指针（局部变量、函数参数）
         ↑
         GC 必须找到这些根，才能追踪完整的对象引用图
\```

###### 2. 扫描流程
\```
STW（Stop The World）
    │
    ├── 所有 goroutine 到达安全点（safe point）
    │       └── 通过抢占机制保证
    │
    └── 并发标记阶段
            │
            ├── 每个 goroutine 被 suspend
            │
            ├── scanstack(gp)
            │       ├── 遍历栈帧（stack frames）
            │       ├── 查询 stackmap（指针位图）
            │       └── 将指针加入 GC 工作队列（灰色对象）
            │
            └── goroutine 恢复运行（STW 结束后）
\```

###### 3. 安全点（Safe Point）
栈扫描必须在安全点进行，此时栈上的指针是精确已知的。
* 编译器为每个函数生成 `stackmap`（指针位图），记录每个栈帧中哪些位置是指针
* 只有在函数调用边界处，stackmap 才是有效的
* 异步抢占后，goroutine 停在信号处理函数中，运行时能找到对应的 stackmap

\```go
// 伪代码：编译器生成的指针位图
funcdata _FUNCDATA_LocalsPointerMaps, gclocals·xxxx
// 告诉 GC：栈帧偏移 0x8 处是指针，偏移 0x10 不是指针...
\```

###### 4. 栈收缩与扫描的配合
\```
GC 扫描期间栈不能移动
    └── 扫描前：stackLock 锁定
    └── 扫描后：解锁，允许 goroutine 扩/缩栈
\```

##### 三、完整协作流程
\```
sysmon 检测到 goroutine 运行 >10ms
    │
    ▼
设置 g.stackguard0 = stackPreempt
发送 SIGURG 信号
    │
    ▼
goroutine 在安全点被挂起（_Grunnable）
    │
    ▼
GC 调用 scanstack(gp)
    ├── 读取 stackmap
    ├── 扫描每帧指针
    └── 标记为灰色/黑色
    │
    ▼
goroutine 恢复运行
\```

##### 四、关键数据结构
\```go
type g struct {
    stack        stack      // [lo, hi] 栈内存范围
    stackguard0  uintptr    // 抢占标志位，= stackPreempt 表示被请求抢占
    stackguard1  uintptr    // cgo 使用
    sched        gobuf      // 保存 PC/SP 等寄存器，用于恢复执行
    atomicstatus uint32     // _Gidle/_Grunnable/_Grunning/_Gsyscall/_Gwaiting
}
\```

> 抢占让 goroutine 停在安全点，栈扫描利用编译器生成的 `stackmap` 精确找到栈上所有指针，两者配合让 Go GC 做到并发、精确、低停顿。
```

### 7.Gc触发时机

#### GC 触发时机

---

#### 1. GC 触发的三种方式总览

```
Go GC 的触发分为三大类：

  ① 自动触发  →  运行时根据内存增长自动判断
  ② 手动触发  →  调用 runtime.GC() 强制触发
  ③ 定时触发  →  距离上次 GC 超过 2 分钟强制触发
```

---

#### 2. 自动触发：堆内存增长比例（最核心）

```
触发条件：
  当前堆内存大小 >= 上次 GC 后堆内存大小 × (1 + GCPercent/100)

默认 GCPercent = 100（即 GOGC=100）

示意：
  上次 GC 后堆内存 = 4MB
  GOGC = 100
  触发阈值 = 4MB × (1 + 100/100) = 8MB
  当堆增长到 8MB → 自动触发 GC

  GC 完成后堆内存 = 3MB
  下次触发阈值 = 3MB × 2 = 6MB
```

```
GOGC 的影响：

  GOGC=100（默认）：堆翻倍时触发，平衡内存与CPU
  GOGC=200       ：堆增长3倍触发，GC更少，内存占用更高
  GOGC=50        ：堆增长1.5倍触发，GC更频繁，内存占用更低
  GOGC=off       ：关闭自动GC，只能手动触发

  ┌──────────┬──────────────┬──────────────┐
  │  GOGC    │  GC 频率     │  内存占用    │
  ├──────────┼──────────────┼──────────────┤
  │  小       │  频繁        │  低          │
  │  大       │  稀少        │  高          │
  │  off     │  不自动触发  │  最高        │
  └──────────┴──────────────┴──────────────┘
```

---

#### 3. Go 1.19+ 新增：GOMEMLIMIT 软内存上限

```
Go 1.19 引入 GOMEMLIMIT，提供更精细的触发控制

触发条件：
  当进程内存使用接近 GOMEMLIMIT 时
  即使 GOGC 阈值未达到，也会提前触发 GC

作用：
  防止程序因内存使用过高被 OOM Killer 杀死
  在内存紧张时更激进地触发 GC

设置方式：
  GOMEMLIMIT=1GiB   // 环境变量
  runtime/debug.SetMemoryLimit(1 << 30)  // 代码设置

GOGC 与 GOMEMLIMIT 的关系：
  ┌──────────────────────────────────────────────┐
  │  GOGC        → 基于堆增长比例触发（相对阈值）  │
  │  GOMEMLIMIT  → 基于绝对内存上限触发（绝对阈值）│
  │  两者同时生效，哪个先达到就先触发              │
  └──────────────────────────────────────────────┘
```

---

#### 4. 手动触发：runtime.GC()

```go
// 开发者主动调用，立即触发一次完整 GC
runtime.GC()

// 典型使用场景：

// ① 测试/基准测试前清理垃圾，避免干扰结果
func BenchmarkXxx(b *testing.B) {
    runtime.GC()  // 先 GC，让堆处于干净状态
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        // ...
    }
}

// ② 执行完一段大内存操作后，主动释放
func processBigData() {
    data := loadHugeData()  // 加载大量数据
    process(data)
    data = nil
    runtime.GC()            // 主动回收，避免内存长期占用
}

// ③ 内存敏感服务在空闲期主动触发
func idleCleanup() {
    ticker := time.NewTicker(10 * time.Minute)
    for range ticker.C {
        runtime.GC()
    }
}
```

---

#### 5. 定时触发：2 分钟兜底机制

```
触发条件：
  距离上次 GC 超过 2 分钟，强制触发

目的：
  防止程序长期处于低分配状态
  内存中积累的垃圾长期得不到回收

场景示意：
  程序启动后内存分配很少
  GOGC 阈值长时间未达到
  2 分钟定时器到期 → 触发 GC → 清理陈旧对象

源码位置：
  runtime/proc.go → sysmon 后台监控线程
  每隔一段时间检查是否超过 2 分钟未 GC
  forcegcperiod = 2 * 60 * 1e9（纳秒）
```

---

#### 6. sysmon 的触发检测

```
sysmon 是 Go 运行时的后台监控线程
独立于 GMP 模型，不需要 P，直接绑定 M 运行

sysmon 负责的 GC 相关工作：

  ① 定时检查是否触发 GC
     每次循环检查距上次 GC 是否超过 2 分钟

  ② 触发 GC 的方式
     设置 gcTrigger，唤醒 forcegc goroutine
     forcegc goroutine 调用 gcStart() 开始 GC

  sysmon 循环间隔：
    初始 20μs
    最长 10ms（自适应，空闲时逐渐增大间隔）
```

---

#### 7. GC 触发的完整判断逻辑

```go
// runtime/mgc.go
type gcTrigger struct {
    kind gcTriggerKind
    now  int64   // 当前时间（定时触发用）
    n    uint32  // 当前 GC 轮次
}

type gcTriggerKind int
const (
    gcTriggerHeap   gcTriggerKind = iota  // 堆内存增长触发
    gcTriggerTime                          // 定时触发
    gcTriggerCycle                         // 手动触发（指定轮次）
)

// 触发条件判断
func (t gcTrigger) test() bool {
    switch t.kind {
    case gcTriggerHeap:
        // 堆大小 >= 触发阈值
        return memstats.heap_live >= memstats.gc_trigger

    case gcTriggerTime:
        // 距上次 GC 超过 2 分钟
        return t.now - mheap_.gcMark >= forcegcperiod

    case gcTriggerCycle:
        // 当前轮次 < 目标轮次（手动触发时设置）
        return int32(t.n-work.cycles) > 0
    }
    return true
}
```

---

#### 8. 触发时机总结对比

```
┌────────────────┬───────────────────────────┬──────────────────────────┐
│   触发类型      │         触发条件           │        典型场景           │
├────────────────┼───────────────────────────┼──────────────────────────┤
│ 堆增长触发      │ 堆内存 >= 上次GC后 × 2    │ 正常业务运行，最常见       │
│ (gcTriggerHeap)│ （GOGC=100 时）            │                          │
├────────────────┼───────────────────────────┼──────────────────────────┤
│ 内存上限触发    │ 内存接近 GOMEMLIMIT        │ 内存敏感场景，防 OOM      │
│ (1.19+)        │                           │                          │
├────────────────┼───────────────────────────┼──────────────────────────┤
│ 定时触发        │ 距上次 GC 超过 2 分钟      │ 低分配场景，兜底清理       │
│ (gcTriggerTime)│                           │                          │
├────────────────┼───────────────────────────┼──────────────────────────┤
│ 手动触发        │ 调用 runtime.GC()         │ 测试/大内存释放/空闲清理   │
│ (gcTriggerCycle)│                          │                          │
└────────────────┴───────────────────────────┴──────────────────────────┘
```

---

#### 9. 调优建议

```go
// ① 调整 GOGC（适合内存宽裕、追求低延迟的服务）
os.Setenv("GOGC", "200")           // 减少 GC 频率
runtime/debug.SetGCPercent(200)    // 代码方式设置


// ② 设置 GOMEMLIMIT（适合容器化部署，防止 OOM）
// 容器内存限制 512MB，设置软上限 400MB
runtime/debug.SetMemoryLimit(400 << 20)


// ③ 关闭自动 GC + 手动控制（适合延迟极敏感场景）
runtime/debug.SetGCPercent(-1)  // 关闭自动 GC
// 在业务低峰期手动触发
go func() {
    ticker := time.NewTicker(time.Minute)
    for range ticker.C {
        runtime.GC()
    }
}()


// ④ 使用 sync.Pool 减少分配，降低 GC 触发频率
var pool = sync.Pool{
    New: func() any { return make([]byte, 4096) },
}
```

---

#### 总结

| 触发方式 | 触发条件 | 控制参数 |
|---------|---------|---------|
| 堆增长触发 | 堆内存达到上次 GC 后的 `1+GOGC/100` 倍 | `GOGC` / `SetGCPercent` |
| 内存上限触发 | 内存接近 GOMEMLIMIT | `GOMEMLIMIT` / `SetMemoryLimit` |
| 定时触发 | 距上次 GC 超过 2 分钟 | 不可配置，固定 2 分钟 |
| 手动触发 | 调用 `runtime.GC()` | 业务代码主动控制 |

> GC 触发的本质：**以堆内存翻倍为核心触发条件，辅以 2 分钟定时兜底和内存上限保护，在吞吐量（少触发）与内存占用（多触发）之间动态平衡，通过 GOGC 和 GOMEMLIMIT 两个旋钮交给开发者灵活调控。**


### 8.是否了解其他gc机制
### 10.Channel分配在堆上还是在栈上？哪些对象分配在堆上？哪些对象分配在栈上？

#### Channel 分配在堆上还是栈上？哪些对象分配在堆上？哪些对象分配在栈上？

---

#### 1. Channel 分配在哪里

```go
// make(chan T) 创建的 channel 底层是 hchan 结构体
// 通过逃逸分析决定分配位置，但绝大多数情况在堆上

ch := make(chan int)      // hchan 分配在堆上
ch := make(chan int, 10)  // hchan + ring buffer 都在堆上

// 原因：
// ① channel 通常在多个 goroutine 之间共享
// ② goroutine 的栈是私有的，无法跨 G 共享
// ③ 编译器逃逸分析检测到 channel 被多个 G 引用 → 逃逸到堆
```

```
channel 内存布局（堆上）：

  ┌─────────────────────────────────┐
  │           hchan                 │  ← 堆上分配
  │  qcount   dataqsiz  buf         │
  │  sendx    recvx     closed      │
  │  recvq    sendq     lock        │
  ├─────────────────────────────────┤
  │         ring buffer             │  ← 堆上连续分配
  │  [elem0][elem1][elem2]...       │
  └─────────────────────────────────┘

  ch 变量本身（指针）→ 栈上
  ch 指向的 hchan  → 堆上
```

---

#### 2. 栈和堆的核心区别

```
栈（Stack）：
  每个 goroutine 私有
  由编译器自动管理，函数返回自动释放
  分配/释放极快（移动栈指针即可）
  大小有限（Go 初始 2KB，最大 1GB）
  GC 不负责管理栈内存

堆（Heap）：
  所有 goroutine 共享
  由 GC 负责回收
  分配相对较慢
  大小受物理内存限制
  对象生命周期由引用决定

  ┌──────────┐  ┌──────────┐  ┌──────────┐
  │  G1 栈   │  │  G2 栈   │  │  G3 栈   │  各自独立
  └──────────┘  └──────────┘  └──────────┘
  ┌─────────────────────────────────────┐
  │              堆                     │  所有 G 共享
  └─────────────────────────────────────┘
```

---

#### 3. 逃逸分析（核心决策机制）

```
Go 编译器通过逃逸分析（Escape Analysis）
在编译期决定对象分配在栈上还是堆上

核心原则：
  能分配在栈上就分配在栈上（性能更好）
  必须分配在堆上才分配在堆上（生命周期更长）

查看逃逸分析结果：
  go build -gcflags="-m" main.go
  go build -gcflags="-m -m" main.go  // 更详细
```

```
逃逸分析示例：

// 未逃逸（栈上）
func noEscape() int {
    x := 42        // x 不逃逸，分配在栈上
    return x       // 返回值，不是指针
}

// 已逃逸（堆上）
func escaped() *int {
    x := 42        // x 逃逸到堆！
    return &x      // 返回指针，函数返回后仍需存活
}

编译器输出：
  ./main.go:8:2: moved to heap: x   ← 逃逸提示
```

---

#### 4. 分配在栈上的对象

```go
// ① 函数内的局部变量（未逃逸）
func foo() {
    x := 10           // 栈上
    y := 3.14         // 栈上
    z := "hello"      // 字符串头部在栈上，底层数据在静态区
    arr := [3]int{}   // 固定大小数组，栈上
}


// ② 值类型的临时变量
func calc(a, b int) int {
    sum := a + b      // 栈上
    return sum
}


// ③ 函数参数和返回值
func add(a, b int) int {  // a、b 在栈上
    return a + b          // 返回值在栈上
}


// ④ 小对象且未被外部引用
func process() {
    p := Point{X: 1, Y: 2}  // 未逃逸 → 栈上
    fmt.Println(p.X + p.Y)
}


// ⑤ 固定大小且较小的数组
func arrayOnStack() {
    arr := [10]int{1, 2, 3}  // 小数组，栈上
    _ = arr
}
```

---

#### 5. 分配在堆上的对象

```go
// ① 返回局部变量的指针
func newInt() *int {
    x := 42
    return &x   // x 逃逸到堆！函数返回后 x 仍需存活
}


// ② make 创建的引用类型
ch := make(chan int)        // 堆上
sl := make([]int, 100)     // 底层数组在堆上
m  := make(map[string]int) // 堆上


// ③ new 创建的对象
p := new(int)    // 堆上（大多数情况）
p := new(Point)  // 堆上


// ④ 接口类型存储具体值
var i interface{} = 42   // 42 逃逸到堆（装箱）
var w io.Writer = os.Stdout


// ⑤ 闭包捕获的变量
func counter() func() int {
    count := 0          // count 逃逸到堆！被闭包捕获
    return func() int {
        count++
        return count
    }
}


// ⑥ 大对象（超过栈大小限制）
func bigObj() {
    arr := [1 << 20]int{} // 8MB 数组，超过栈限制 → 堆上
    _ = arr
}


// ⑦ 动态大小的 slice
func dynamicSlice(n int) {
    s := make([]int, n)  // n 运行时才知道大小 → 堆上
    _ = s
}


// ⑧ goroutine 共享的变量
var shared *int

func setShared() {
    x := 42
    shared = &x   // x 被全局变量引用 → 逃逸到堆
}
```

---

#### 6. 逃逸的六大场景

```
场景① 指针逃逸
  函数返回局部变量的指针
  func f() *int { x := 1; return &x }  → x 逃逸

场景② 接口逃逸
  将具体类型赋值给接口
  var i interface{} = someStruct{}  → someStruct 逃逸
  fmt.Println(x)  → x 作为 interface{} 传入，逃逸

场景③ 闭包逃逸
  闭包引用了外部函数的变量
  被引用的变量逃逸到堆

场景④ 动态大小逃逸
  make([]int, n)，n 为变量时逃逸
  编译期无法确定大小，无法在栈上分配

场景⑤ 大对象逃逸
  对象过大，超出栈空间
  直接分配到堆上

场景⑥ 全局变量引用逃逸
  局部变量被全局变量或长生命周期变量引用
  必须逃逸到堆以保证生命周期
```

---

#### 7. 典型案例分析

```go
package main

import "fmt"

// 案例①：未逃逸
func stackAlloc() int {
    x := 100     // 不逃逸，栈上
    return x     // 返回值，非指针
}

// 案例②：逃逸
func heapAlloc() *int {
    x := 100     // 逃逸，堆上
    return &x
}

// 案例③：fmt.Println 导致逃逸
func printEscape() {
    x := 100
    fmt.Println(x)  // x 作为 interface{} 传入 → 逃逸！
}

// 案例④：slice 底层数组逃逸
func sliceEscape() []int {
    s := make([]int, 10)  // 底层数组逃逸
    return s              // slice 被返回，底层数组必须存活
}

// 案例⑤：闭包逃逸
func closureEscape() func() {
    msg := "hello"        // msg 逃逸，被闭包捕获
    return func() {
        fmt.Println(msg)
    }
}
```

```bash
# 查看逃逸分析结果
go build -gcflags="-m" main.go

# 输出：
# ./main.go:13:2: moved to heap: x        ← 案例②
# ./main.go:19:14: x escapes to heap      ← 案例③
# ./main.go:25:15: make([]int, 10) escapes to heap  ← 案例④
# ./main.go:31:2: moved to heap: msg      ← 案例⑤
```

---

#### 8. 栈 vs 堆 性能对比

```
栈分配：
  操作：移动栈指针（SP -= size）
  耗时：~1ns
  GC：不参与，函数返回自动回收
  并发：无竞争（每个 G 私有）

堆分配：
  操作：从 mheap 找合适的内存块
  耗时：~100ns（比栈慢约100倍）
  GC：需要 GC 扫描和回收
  并发：需要加锁或 mcache 隔离

结论：
  栈分配比堆分配快 ~100 倍
  减少逃逸 = 减少堆分配 = 减少 GC 压力 = 提升性能
```

---

#### 总结

| 分配位置 | 典型对象 | 特点 |
|---------|---------|------|
| 栈上 | 局部变量、函数参数、小数组、未逃逸对象 | 自动回收、极快、G 私有 |
| 堆上 | 逃逸对象、make/new、接口装箱、闭包捕获变量、大对象 | GC 回收、较慢、G 共享 |

| 对象 | 分配位置 | 原因 |
|------|---------|------|
| `make(chan T)` | 堆上 | 跨 G 共享，必须逃逸 |
| `make([]T, n)` | 堆上 | 底层数组需跨作用域存活 |
| `make(map)` | 堆上 | 引用类型，跨 G 共享 |
| 返回的指针变量 | 堆上 | 函数返回后仍需存活 |
| 未被引用的局部变量 | 栈上 | 函数返回后无需存活 |
| 固定小数组 | 栈上 | 大小固定，生命周期明确 |

> 核心结论：**Go 编译器通过逃逸分析自动决策，channel 因跨 goroutine 共享几乎必然分配在堆上；能留在栈上的对象尽量留在栈上，减少堆分配是 Go 性能优化的重要手段。**


### 11.代码效率分析，考虑局部性原理
### 12.多核CPU下，cache如何保持一致，不冲突
### 13.uint类型溢出
### 14.聊聊rune类型
### 15.介绍一下channel，有缓冲和无缓冲的区别
### 16.channel是否线程安全
### 17.介绍一下Mutex的实现,是悲观锁还是乐观锁
### 18.Mutex几种模式?
### 19.Muxtez可以做自旋锁?
### 20.介绍一下RWMutex
### 21.介绍一下大对象和小对象，为什么小对象多了会造成gc压力？
### 22.介绍项目中遇到的oop情况
### 23.介绍项目中遇到的坑
### 24.如果指定指令执行的顺序
### 25.什么是写屏障、混合写屏障，如何实现？
### 26.gc的stw是怎么回事
### 27.协程之间是怎么调度的
### 28.简单聊聊内存逃逸 
### 29.为什么sync.WaitGroup中Wait函数支持 WaitTimeout 功能
### 30.字符串转成byte数组，会发生内存拷贝吗？
### 31.http包的内存泄漏
### 32.Goroutine调度策略
### 33.对已经关闭的的chan进行读写，会怎么样？为什么？
### 34.实现阻塞读的并发安全Map
### 35.什么是goroutine leak？
### 36.data race问题怎么解决？能不能不加锁解决这个问题？
### grpc内部原理是什么？
### time.Now有几次系统调用？如何优化
### 空struct{}是否使用过？会在什么情况下使用，举例说明一下
### 聊聊runtime
### 介绍下你平时都是怎么调试bug以及性能问题的?
### 通过通信来共享内存，而不是通过共享内存而通信，怎么理解这句话，如何处理共享变量？
### chan比mutex更轻么？还有更轻量的方法么？
### 什么时候用chan不如mutex效率高？
### 什么场景下会触发panic
  - 数组越界
  - 并发写map和未初始化map
  - 空指针调用
  - 过早关闭http响应体
  - 除数为0
  - 向已经关闭的chan发送信息
  - 重复关闭chan
  - 关闭未初始化的chan
  - sync.waitGroup计数为负数
### 什么事hash冲突，go中map如何解决hash冲突？
### 多个携程间的通信方法
### 除了mutex以外还有哪些方式安全读写共享变量
### 用过什么包管理工具
### golang的服务性能指标怎么查看，有哪些指标需要注意
### 协程池的意义是什么
### 8、Go 当中同步锁有什么特点？作用是什么
    - 当一个 Goroutine（协程）获得了 Mutex 后，其他 Goroutine（协程）就只能乖乖的等待，除非该 Goroutine 释放了该 Mutex。RWMutex 在读锁占用的情况下， 会阻止写，但不阻止读 RWMutex。 在写锁占用情况下，会阻止任何其他Goroutine（无论读和写）进来，整个锁相当于由该 Goroutine 独占
    同步锁的作用是保证资源在使用时的独有性，不会因为并发而导致数据错乱， 保证系统的稳定性。
