
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
#### 多核 CPU 下 Cache 如何保持一致

---

#### 1. 问题的根源

```
多核 CPU 每个核心都有自己的私有缓存（L1/L2）

┌─────────────────────────────────────────────┐
│  Core0        Core1        Core2        Core3│
│  L1 Cache     L1 Cache     L1 Cache     L1  │
│  L2 Cache     L2 Cache     L2 Cache     L2  │
├─────────────────────────────────────────────┤
│              L3 Cache（共享）                │
├─────────────────────────────────────────────┤
│              主内存（Memory）                │
└─────────────────────────────────────────────┘

问题：
  Core0 读取变量 x=1，缓存到自己的 L1
  Core1 也读取变量 x=1，缓存到自己的 L1
  Core0 将 x 改为 2，写入自己的 L1
  Core1 的 L1 中 x 仍然是 1  ← 数据不一致！
```

---

#### 2. 缓存一致性协议总览

```
解决方案：缓存一致性协议（Cache Coherence Protocol）

核心思想：
  所有 CPU 核心对同一内存地址的缓存状态
  必须遵守统一的协议规则
  通过状态机 + 总线通信保证一致性

主流协议：
  MSI   → 最基础的三态协议
  MESI  → 最广泛使用（Intel x86）
  MOESI → AMD 扩展版本
  MESIF → Intel 进一步扩展版本
```

---

#### 3. MESI 协议：四种缓存状态

```
MESI 是目前最主流的缓存一致性协议
每个缓存行（Cache Line，64字节）有四种状态：

┌──────────┬────────────────────────────────────────────┐
│  状态     │                含义                        │
├──────────┼────────────────────────────────────────────┤
│ Modified │ 已修改，只有我有，与内存不一致，我负责写回   │
│ Exclusive│ 独占，只有我有，与内存一致，未被修改         │
│ Shared   │ 共享，多个核心都有，与内存一致，只读         │
│ Invalid  │ 无效，该缓存行已失效，不可使用               │
└──────────┴────────────────────────────────────────────┘

状态首字母组合 = MESI
```

---

#### 4. MESI 状态流转

```
初始状态：所有核心缓存均为 Invalid

场景：Core0 和 Core1 操作同一变量 x

① Core0 读取 x：
  Core0: Invalid → Exclusive（只有我有，内存一致）
  ┌────────┐      ┌────────┐
  │ Core0  │      │ Core1  │
  │ E: x=1 │      │ I:     │
  └────────┘      └────────┘

② Core1 也读取 x：
  总线通知 → Core0: Exclusive → Shared
  Core1: Invalid → Shared
  ┌────────┐      ┌────────┐
  │ Core0  │      │ Core1  │
  │ S: x=1 │      │ S: x=1 │  ← 多核共享，只读
  └────────┘      └────────┘

③ Core0 修改 x=2：
  Core0 发出 Invalidate 广播
  Core1: Shared → Invalid（缓存失效）
  Core0: Shared → Modified
  ┌────────┐      ┌────────┐
  │ Core0  │      │ Core1  │
  │ M: x=2 │      │ I:     │  ← Core1 缓存失效
  └────────┘      └────────┘

④ Core1 再次读取 x：
  Core1 发出读请求
  Core0: Modified → Shared，将 x=2 写回内存/直传
  Core1: Invalid  → Shared
  ┌────────┐      ┌────────┐
  │ Core0  │      │ Core1  │
  │ S: x=2 │      │ S: x=2 │  ← 数据同步
  └────────┘      └────────┘
```

---

#### 5. 总线监听机制（Bus Snooping）

```
MESI 的通信基础：总线监听（Snooping）

每个 CPU 核心持续监听总线上的所有读写请求
根据请求类型更新自己的缓存状态

总线消息类型：

┌──────────────────┬────────────────────────────────────┐
│   消息类型        │              含义                   │
├──────────────────┼────────────────────────────────────┤
│ Read             │ 请求读取某地址的数据                  │
│ Read Response    │ 返回读取的数据                       │
│ Invalidate       │ 通知其他核心将该缓存行置为 Invalid    │
│ Invalidate Ack   │ 确认已失效                          │
│ Read Invalidate  │ 读取并独占（准备修改）               │
│ Writeback        │ 将 Modified 数据写回内存             │
└──────────────────┴────────────────────────────────────┘

流程示意：
  Core0 写操作
    ↓ 发出 Invalidate(addr) 广播
  Core1、Core2、Core3 收到广播
    ↓ 各自将该地址缓存行置为 Invalid
    ↓ 回复 Invalidate Ack
  Core0 收到所有 Ack
    ↓ 缓存行变为 Modified，执行写入
```

---

#### 6. Store Buffer 和 Invalidate Queue

```
问题：等待 Invalidate Ack 会让 CPU 停顿（性能损耗）

解决方案：引入异步缓冲区

Store Buffer（写缓冲）：
  Core0 写操作不等 Ack
  先将数据写入 Store Buffer
  异步等待 Ack，收到后再提交到缓存
  CPU 可以继续执行后续指令

  Core0写 → Store Buffer → （异步）等Ack → 写入L1 Cache

Invalidate Queue（失效队列）：
  其他核心收到 Invalidate 请求
  不立即处理，先放入 Invalidate Queue 回复 Ack
  异步处理队列中的失效操作

  Core1收到Invalidate → 放入队列 → 立即回Ack
                      → 异步处理队列 → L1 Cache置Invalid

代价：
  引入了可见性问题！
  Core0 写入 Store Buffer 但未提交
  Core1 处理 Invalidate Queue 前仍读旧值
  这就是 CPU 层面的内存乱序根源
```

---

#### 7. 内存屏障（Memory Barrier）

```
Store Buffer 和 Invalidate Queue 导致的乱序
需要内存屏障（Memory Barrier）来解决

写屏障（Store Barrier / sfence）：
  强制刷新 Store Buffer
  确保屏障前的写操作对所有核心可见
  之后的写操作不会被重排到屏障前

读屏障（Load Barrier / lfence）：
  强制处理 Invalidate Queue
  确保读到最新数据
  之后的读操作不会被重排到屏障前

全屏障（Full Barrier / mfence）：
  同时包含读写屏障

┌──────────────────────────────────────────────┐
│  Core0                    Core1              │
│  x = 1                    while flag == 0 {} │
│  sfence  ← 写屏障          lfence  ← 读屏障   │
│  flag = 1                 print(x)           │
│                                              │
│  写屏障确保 x=1 先于 flag=1 对外可见          │
│  读屏障确保读到最新的 flag 和 x               │
└──────────────────────────────────────────────┘
```

---

#### 8. Go 中的内存模型与 happens-before

```go
// Go 内存模型基于 happens-before 原则
// 保证操作的可见性和顺序性

// ① Channel 通信（内置内存屏障）
ch := make(chan int, 1)

go func() {
    x = 1        // A
    ch <- 1      // B：发送，写屏障
}()

<-ch             // C：接收，读屏障
fmt.Println(x)   // D：保证看到 x=1

// A happens-before B happens-before C happens-before D


// ② sync.Mutex（内置内存屏障）
var mu sync.Mutex
var x int

go func() {
    mu.Lock()
    x = 1        // 写入 x
    mu.Unlock()  // 释放锁，写屏障
}()

mu.Lock()        // 加锁，读屏障
fmt.Println(x)   // 保证看到 x=1
mu.Unlock()


// ③ atomic 操作（内置屏障）
var flag atomic.Int32
var x int

go func() {
    x = 1
    flag.Store(1)   // 原子写，含写屏障
}()

for flag.Load() == 0 {}  // 原子读，含读屏障
fmt.Println(x)           // 保证看到 x=1
```

---

#### 9. 伪共享（False Sharing）问题

```
伪共享：两个不相关的变量恰好在同一缓存行（64字节）

┌──────────────────────────────────────────────┐
│              Cache Line（64字节）              │
│  [  变量A（Core0使用）  |  变量B（Core1使用）  ]│
└──────────────────────────────────────────────┘

Core0 修改 A → 整个缓存行 Invalidate → Core1 的 B 也失效
Core1 修改 B → 整个缓存行 Invalidate → Core0 的 A 也失效
两个核心反复争抢同一缓存行 → 性能急剧下降

// ❌ 伪共享
type BadCounter struct {
    a int64  // Core0 频繁修改
    b int64  // Core1 频繁修改
    // a 和 b 在同一缓存行！
}

// ✅ 缓存行填充（Padding）消除伪共享
type GoodCounter struct {
    a   int64
    _   [56]byte  // 填充到 64 字节
    b   int64
    _   [56]byte  // 填充到 64 字节
}

// Go 标准库中的实际案例
// sync.Pool 的 poolLocal 结构体：
type poolLocal struct {
    poolLocalInternal
    pad [128 - unsafe.Sizeof(poolLocalInternal{})%128]byte
    // 用 padding 避免伪共享
}
```

---

#### 10. 完整一致性保障链路

```
硬件层：
  MESI 协议 + 总线监听
  保证缓存行级别的一致性
      ↓
硬件优化层：
  Store Buffer + Invalidate Queue
  提升性能，但引入可见性问题
      ↓
指令层：
  内存屏障指令（mfence/sfence/lfence）
  强制刷新缓冲区，恢复可见性
      ↓
语言层（Go）：
  atomic 原子操作（含屏障）
  Channel 通信（含屏障）
  sync.Mutex（含屏障）
      ↓
开发者：
  遵循 Go 内存模型
  通过上述同步原语保证并发正确性
  避免伪共享优化热点性能
```

---

#### 总结

| 机制 | 解决的问题 | 层次 |
|------|---------|------|
| MESI 协议 | 多核缓存数据不一致 | 硬件 |
| 总线监听 | 感知其他核心的读写操作 | 硬件 |
| Store Buffer | 减少写操作等待延迟 | 硬件优化 |
| Invalidate Queue | 减少失效确认延迟 | 硬件优化 |
| 内存屏障 | 防止 CPU 指令重排，强制可见 | 指令 |
| atomic/Channel/Mutex | 对开发者屏蔽底层细节 | Go 运行时 |
| 缓存行填充 | 消除伪共享 | 代码优化 |

> 多核缓存一致性的本质：**硬件通过 MESI 协议保证缓存行状态一致，Store Buffer 和 Invalidate Queue 提升性能但引入乱序，内存屏障强制恢复顺序，Go 的 atomic、Channel、Mutex 在语言层封装了这些底层机制，让开发者无需直接操心 CPU 细节。**

### 13.uint类型溢出

#### uint 类型溢出

---

#### 1. 什么是整数溢出

```
整数溢出：
  整数运算结果超出该类型能表示的范围
  多余的高位被截断，得到一个"意外"的值

Go 的整数溢出：
  编译期不报错
  运行时不 panic
  默默发生，结果"回绕"（Wrap Around）
```

---

#### 2. uint 各类型的取值范围

```
┌───────────┬──────┬────────────────────┬────────────────────────┐
│   类型     │ 位数 │       最小值        │         最大值          │
├───────────┼──────┼────────────────────┼────────────────────────┤
│ uint8     │  8   │         0          │          255           │
│ uint16    │  16  │         0          │         65535          │
│ uint32    │  32  │         0          │      4294967295        │
│ uint64    │  64  │         0          │ 18446744073709551615   │
│ uint      │ 32/64│         0          │  平台相关               │
└───────────┴──────┴────────────────────┴────────────────────────┘

uint 最大值的二进制全为 1：
  uint8  → 1111 1111 = 255
  uint16 → 1111...1111 = 65535
```

---

#### 3. 上溢：超过最大值回绕到 0

```go
// uint8 最大值 255，再加 1
var a uint8 = 255
a++
fmt.Println(a)  // 输出：0  ← 回绕到最小值

// 原因（二进制）：
//  1111 1111  (255)
// +0000 0001  (+1)
// ──────────
// 1 0000 0000  → 第9位溢出被截断
//   0000 0000  = 0
```

```
上溢示意：

  0 ──────────────────────────── 255
  │                               │
  └───────────────────────────────┘
       超过 255 → 回绕到 0 →  继续增加
```

---

#### 4. 下溢：低于 0 回绕到最大值

```go
// uint8 最小值 0，再减 1
var b uint8 = 0
b--
fmt.Println(b)  // 输出：255  ← 回绕到最大值

// 原因（二进制补码回绕）：
//  0000 0000  (0)
// -0000 0001  (-1)
// ──────────
//  1111 1111  = 255

var c uint8 = 3
c -= 5
fmt.Println(c)   // 输出：254
// 3 - 5 = -2 → 无符号回绕 → 256 - 2 = 254
```

---

#### 5. 溢出的回绕公式

```
上溢：result = (真实值) mod (MaxValue + 1)
下溢：result = (真实值 + MaxValue + 1) mod (MaxValue + 1)

uint8 示例：
  255 + 1  = 256  mod 256 = 0
  255 + 2  = 257  mod 256 = 1
  255 + 10 = 265  mod 256 = 9

  0   - 1  = -1   → -1 + 256 = 255
  0   - 2  = -2   → -2 + 256 = 254
  3   - 5  = -2   → -2 + 256 = 254
```

---

#### 6. 有符号 int 的溢出

```go
// int8 范围：-128 ~ 127
var i int8 = 127
i++
fmt.Println(i)  // 输出：-128  ← 正数溢出变负数

// 二进制：
//  0111 1111  (127)
// +0000 0001
// ──────────
//  1000 0000  = -128（符号位变为1）

var j int8 = -128
j--
fmt.Println(j)  // 输出：127  ← 负数溢出变正数

// 有符号溢出是未定义行为（UB）
// Go 中不会 panic，但结果不可依赖
```

---

#### 7. 溢出引发的经典 Bug

```go
// Bug①：循环条件永远为真
var i uint8 = 0
for i >= 0 {         // ❌ uint 永远 >= 0，死循环！
    fmt.Println(i)
    i++
}

// Bug②：长度计算溢出
var a, b uint32 = 1, 4294967295  // b = MaxUint32
sum := a + b
fmt.Println(sum)  // 输出：0，不是 4294967296！

// Bug③：切片索引计算溢出
var idx uint = 0
idx -= 1            // 下溢 → idx = MaxUint
s := make([]int, 10)
_ = s[idx]          // panic: index out of range

// Bug④：安全漏洞（整数溢出绕过检查）
func allocate(size uint32) []byte {
    if size > 1024 {
        panic("too large")
    }
    // 攻击者传入 size = 0
    // 后续计算 size - 1 = MaxUint32 → 绕过检查
    return make([]byte, size)
}
```

---

#### 8. 编译期常量溢出 vs 运行期溢出

```go
// 编译期常量溢出 → 编译报错
const x uint8 = 256        // ❌ 编译错误：constant 256 overflows uint8
const y uint8 = 255 + 1    // ❌ 编译错误

// 运行期变量溢出 → 静默回绕，不报错不 panic
var a uint8 = 255
a = a + 1                  // ✅ 编译通过，运行时 a = 0

var b uint8 = 200
var c uint8 = 100
d := b + c                 // ✅ 编译通过，运行时 d = 44（300 mod 256）
```

---

#### 9. 如何检测溢出

```go
// 方法①：加法溢出检测
func addUint8Safe(a, b uint8) (uint8, bool) {
    result := a + b
    if result < a {         // 溢出后 result 一定小于 a
        return 0, false     // 溢出
    }
    return result, true
}

// 方法②：乘法溢出检测
func mulUint64Safe(a, b uint64) (uint64, bool) {
    if a == 0 || b == 0 {
        return 0, true
    }
    result := a * b
    if result/a != b {      // 溢出后除法无法还原
        return 0, false
    }
    return result, true
}

// 方法③：使用 math/bits 包（Go 1.9+）
import "math/bits"

// Add 返回结果和进位 carry
result, carry := bits.Add64(a, b, 0)
if carry != 0 {
    // 溢出
}

// Mul 返回高位和低位
hi, lo := bits.Mul64(a, b)
if hi != 0 {
    // 溢出（结果超过 64 位）
}
```

---

#### 10. 升级类型避免溢出

```go
// ❌ 直接用 uint8 计算，可能溢出
var a, b uint8 = 200, 100
result := a + b             // 溢出！结果 44

// ✅ 升级为更大类型计算，再判断
func addSafe(a, b uint8) (uint8, error) {
    r := int(a) + int(b)    // 升级为 int 计算
    if r > math.MaxUint8 {
        return 0, errors.New("overflow")
    }
    return uint8(r), nil
}

// ✅ 检查边界再计算
func safeSub(a, b uint) (uint, error) {
    if b > a {
        return 0, errors.New("uint underflow")
    }
    return a - b, nil
}
```

---

#### 11. math 包提供的边界常量

```go
import "math"

// 各类型最大最小值常量
math.MaxUint8   = 255
math.MaxUint16  = 65535
math.MaxUint32  = 4294967295
math.MaxUint64  = 18446744073709551615

math.MaxInt8    = 127
math.MinInt8    = -128
math.MaxInt16   = 32767
math.MinInt16   = -32768
math.MaxInt32   = 2147483647
math.MinInt32   = -2147483648
math.MaxInt64   = 9223372036854775807
math.MinInt64   = -9223372036854775808

// 使用示例：溢出前检查
func safeAdd(a, b uint64) (uint64, error) {
    if a > math.MaxUint64 - b {
        return 0, errors.New("overflow")
    }
    return a + b, nil
}
```

---

#### 总结

| 类型 | 上溢行为 | 下溢行为 | 编译期溢出 |
|------|---------|---------|----------|
| uint8 | 255+1=0 | 0-1=255 | 编译报错 |
| uint16 | 65535+1=0 | 0-1=65535 | 编译报错 |
| uint32 | MaxUint32+1=0 | 0-1=MaxUint32 | 编译报错 |
| uint64 | MaxUint64+1=0 | 0-1=MaxUint64 | 编译报错 |
| int8 | 127+1=-128 | -128-1=127 | 编译报错 |

| 防御手段 | 适用场景 |
|---------|---------|
| 升级类型计算 | 简单加减乘除 |
| 计算前边界检查 | 减法防下溢 |
| `math/bits` 包 | 高性能溢出检测 |
| 结果反向验证 | 乘法溢出检测 |

> uint 溢出的本质：**无符号整数在二进制层面做模运算（mod 2^N），超出范围的高位直接截断，Go 运行时不检测不 panic，静默回绕是 bug 和安全漏洞的温床，关键计算必须主动做边界检查。**

### 14.聊聊rune类型
#### 聊聊 rune 类型

---

#### 1. rune 的本质

```go
// rune 是 int32 的别名，本质上就是一个 32 位有符号整数
type rune = int32

// 用于表示一个 Unicode 码点（Code Point）
// 取值范围：0 ~ 0x10FFFF（1114111）

// 以下两种写法完全等价
var r1 rune   = '中'
var r2 int32  = '中'
fmt.Println(r1 == r2)  // true
```

```
Unicode 码点：
  每个字符在 Unicode 标准中都有唯一编号
  'A'  → U+0041 → 65
  '中' → U+4E2D → 20013
  '😀' → U+1F600 → 128512

rune 就是存储这个编号的容器
```

---

#### 2. rune vs byte vs string

```go
// byte = uint8，存储 ASCII 字符或 UTF-8 的单个字节
// rune = int32，存储完整的 Unicode 字符

var b byte = 'A'      // 'A' = 65，ASCII 范围内，1个字节
var r rune = '中'     // '中' = 20013，需要3个字节（UTF-8编码）

// string 底层是 UTF-8 编码的字节序列
s := "Hello中文"

fmt.Println(len(s))           // 11（字节数：5 + 3 + 3）
fmt.Println(len([]rune(s)))   // 7（字符数：5 + 2）
```

```
"中" 的存储方式对比：

  byte 视角（UTF-8编码）：
  ┌──────┬──────┬──────┐
  │ 0xE4 │ 0xB8 │ 0xAD │  → 3个字节
  └──────┴──────┴──────┘

  rune 视角（Unicode码点）：
  ┌──────────────────────┐
  │       0x4E2D         │  → 1个rune（int32）
  └──────────────────────┘
```

---

#### 3. UTF-8 编码规则

```
Go 的 string 使用 UTF-8 编码存储
不同字符占用不同字节数：

┌──────────────────────┬──────────┬──────────────────────────┐
│     Unicode 范围      │ 字节数   │       编码格式             │
├──────────────────────┼──────────┼──────────────────────────┤
│ U+0000  ~ U+007F     │   1字节  │ 0xxxxxxx                 │
│ U+0080  ~ U+07FF     │   2字节  │ 110xxxxx 10xxxxxx        │
│ U+0800  ~ U+FFFF     │   3字节  │ 1110xxxx 10xxxxxx 10xxxx │
│ U+10000 ~ U+10FFFF   │   4字节  │ 11110xxx 10xxxxxx ...    │
└──────────────────────┴──────────┴──────────────────────────┘

示例：
  'A'  → U+0041 → 0x41               → 1字节：01000001
  '©'  → U+00A9 → 0xC2 0xA9          → 2字节
  '中' → U+4E2D → 0xE4 0xB8 0xAD     → 3字节
  '😀' → U+1F600→ 0xF0 0x9F 0x98 0x80→ 4字节
```

---

#### 4. 字符串遍历：byte 遍历 vs rune 遍历

```go
s := "Hello中"

// ① 按 byte 遍历（下标遍历）
// 得到的是字节索引和字节值
for i := 0; i < len(s); i++ {
    fmt.Printf("byte[%d] = %x\n", i, s[i])
}
// byte[0] = 48  (H)
// byte[1] = 65  (e)
// ...
// byte[5] = e4  ('中'的第1个字节)
// byte[6] = b8  ('中'的第2个字节)
// byte[7] = ad  ('中'的第3个字节)


// ② 按 rune 遍历（range 遍历）
// 得到的是字符的字节起始位置和完整 Unicode 码点
for idx, r := range s {
    fmt.Printf("rune[%d] = %c (%d)\n", idx, r, r)
}
// rune[0] = H (72)
// rune[1] = e (101)
// rune[2] = l (108)
// rune[3] = l (108)
// rune[4] = o (111)
// rune[5] = 中 (20013)  ← 索引跳到5，跨越了3个字节
```

```
range 遍历自动处理 UTF-8 解码：

  s = "H e l l o 中"
       0 1 2 3 4 5 6 7  ← 字节索引
                 ↑
              idx=5，rune='中'，占3字节
              下一个字符从 idx=8 开始
```

---

#### 5. string 与 []rune 的转换

```go
s := "Hello中文世界"

// string → []rune（按字符切割）
runes := []rune(s)
fmt.Println(len(runes))     // 9（字符数）
fmt.Println(string(runes[5]))  // 中

// []rune → string
r := []rune{72, 101, 108, 108, 111, 20013}
fmt.Println(string(r))      // Hello中

// 字符串反转（正确做法：转 []rune 再反转）
func reverseString(s string) string {
    runes := []rune(s)
    for i, j := 0, len(runes)-1; i < j; i, j = i+1, j-1 {
        runes[i], runes[j] = runes[j], runes[i]
    }
    return string(runes)
}

fmt.Println(reverseString("Hello中文"))  // 文中olleH ✅

// ❌ 错误做法：按 byte 反转
func wrongReverse(s string) string {
    b := []byte(s)
    for i, j := 0, len(b)-1; i < j; i, j = i+1, j-1 {
        b[i], b[j] = b[j], b[i]  // 多字节字符被打乱
    }
    return string(b)  // 乱码！
}
```

---

#### 6. rune 字面量

```go
// 单引号表示 rune 字面量
var r rune

r = 'A'       // ASCII 字符
r = '中'      // 中文字符
r = '😀'      // emoji
r = '\n'      // 转义字符（换行）
r = '\t'      // 转义字符（制表符）
r = '\u4e2d'  // Unicode 转义（'中'）
r = '\U0001F600'  // Unicode 转义（'😀'，大写U支持4字节）
r = '\x41'    // 十六进制（'A'）

fmt.Printf("%c %d %U\n", r, r, r)
// 输出格式：
// %c → 字符本身
// %d → 十进制码点值
// %U → Unicode 格式（如 U+4E2D）
```

---

#### 7. string 与 rune 的常用操作

```go
import (
    "strings"
    "unicode"
    "unicode/utf8"
)

s := "Hello中文World"

// ① 获取字符数（非字节数）
charCount := utf8.RuneCountInString(s)
fmt.Println(charCount)  // 12

// ② 获取第 N 个字符
func getNthChar(s string, n int) rune {
    for i, r := range s {
        if i == n { return r }
    }
    return -1
}

// ③ 判断字符类型
r := '中'
fmt.Println(unicode.IsLetter(r))   // true  是字母
fmt.Println(unicode.IsDigit(r))    // false 不是数字
fmt.Println(unicode.IsSpace(r))    // false 不是空白

r2 := 'A'
fmt.Println(unicode.IsUpper(r2))   // true
fmt.Println(unicode.ToLower(r2))   // 97 ('a')

// ④ 字符串截取（按字符而非字节）
func subString(s string, start, end int) string {
    runes := []rune(s)
    return string(runes[start:end])
}
fmt.Println(subString("Hello中文", 0, 7))  // Hello中文

// ⑤ 统计特定字符
count := strings.Count(s, "中")  // 1

// ⑥ UTF-8 编码字节数
size := utf8.RuneLen('中')  // 3
```

---

#### 8. 常见陷阱

```go
// 陷阱①：用下标访问 string 得到 byte 而非 rune
s := "中文"
fmt.Println(s[0])          // 228（0xE4，'中'的第一个字节，非字符）
fmt.Println(string(s[0]))  // 乱码！

// ✅ 正确：转为 []rune 再下标访问
r := []rune(s)
fmt.Println(string(r[0]))  // 中


// 陷阱②：len() 返回字节数而非字符数
s := "中文"
fmt.Println(len(s))           // 6（字节数）
fmt.Println(len([]rune(s)))   // 2（字符数）


// 陷阱③：string 切片按字节切，可能切断多字节字符
s := "中文"
fmt.Println(s[0:3])   // "中"（恰好是3字节，正确）
fmt.Println(s[0:2])   // 乱码！切断了'中'的UTF-8编码


// 陷阱④：rune 转 string 是按码点转，不是数字转字符串
r := rune(65)
fmt.Println(string(r))         // "A"（码点65对应'A'）
fmt.Println(strconv.Itoa(65))  // "65"（数字转字符串）


// 陷阱⑤：无效 UTF-8 序列
s := string([]byte{0xff, 0xfe})  // 无效 UTF-8
for _, r := range s {
    fmt.Printf("%c %d\n", r, r)
    // 输出 U+FFFD（替换字符），码点65533
}
```

---

#### 9. unicode/utf8 常用函数

```go
import "unicode/utf8"

s := "Hello中"

// 字符数
utf8.RuneCountInString(s)     // 6

// 是否是合法 UTF-8
utf8.ValidString(s)           // true

// 解码第一个 rune
r, size := utf8.DecodeRuneInString(s)
// r = 'H', size = 1（字节数）

// 从末尾解码最后一个 rune
r, size = utf8.DecodeLastRuneInString(s)
// r = '中', size = 3

// 单个 rune 的 UTF-8 编码字节数
utf8.RuneLen('A')    // 1
utf8.RuneLen('中')   // 3
utf8.RuneLen('😀')  // 4

// 编码 rune 到 []byte
buf := make([]byte, 4)
n := utf8.EncodeRune(buf, '中')
// buf = [0xe4, 0xb8, 0xad, 0x00], n = 3
```

---

#### 总结

| 对比项 | byte（uint8） | rune（int32） |
|--------|-------------|-------------|
| 本质 | UTF-8 的单个字节 | Unicode 码点 |
| 大小 | 1 字节 | 4 字节 |
| 表示范围 | 0~255 | 0~0x10FFFF |
| 适用字符 | ASCII | 所有 Unicode 字符 |
| 遍历方式 | `s[i]` 下标 | `range` 遍历 |
| 字符串长度 | `len(s)` 字节数 | `len([]rune(s))` 字符数 |

> rune 的本质：**int32 的别名，承载 Unicode 码点，是 Go 处理多语言文本的基本单元。string 是 UTF-8 字节序列，byte 是字节视角，rune 是字符视角，三者配合才能正确处理中文、emoji 等多字节字符。**


### 15.介绍一下channel，有缓冲和无缓冲的区别
### 16.channel是否线程安全
### 17.介绍一下Mutex的实现,是悲观锁还是乐观锁
### 18.Mutex几种模式?
### 19.Muxte可以做自旋锁?
### 20.介绍一下RWMutex
### 21.介绍一下大对象和小对象，为什么小对象多了会造成gc压力？

#### 介绍大对象和小对象，为什么小对象多了会造成 GC 压力

---

#### 1. Go 内存分配器的对象分类

```
Go 运行时按对象大小分为三类：

┌──────────────┬───────────────┬────────────────────────────┐
│   类型        │    大小范围    │         分配方式            │
├──────────────┼───────────────┼────────────────────────────┤
│ 微小对象      │  < 16 bytes   │ mcache 中的 tiny 分配器     │
│ 小对象        │ 16B ~ 32KB   │ mcache → mcentral → mheap  │
│ 大对象        │  > 32KB      │ 直接从 mheap 分配            │
└──────────────┴───────────────┴────────────────────────────┘
```

---

#### 2. Go 内存分配器的层次结构

```
┌─────────────────────────────────────────────────────┐
│                    应用程序                           │
└─────────────────────────────────────────────────────┘
              ↓ 分配请求
┌─────────────────────────────────────────────────────┐
│   mcache（每个 P 私有，无锁）                         │
│   缓存各种规格的 mspan，直接响应小对象分配              │
└─────────────────────────────────────────────────────┘
              ↓ mcache 不够用
┌─────────────────────────────────────────────────────┐
│   mcentral（全局，按 size class 分组，有锁）           │
│   管理所有 mspan，为 mcache 补充 span                 │
└─────────────────────────────────────────────────────┘
              ↓ mcentral 不够用
┌─────────────────────────────────────────────────────┐
│   mheap（全局堆，管理所有内存页）                      │
│   向 OS 申请内存，为 mcentral 提供 span               │
└─────────────────────────────────────────────────────┘
              ↓ mheap 不够用
┌─────────────────────────────────────────────────────┐
│   操作系统（mmap/sbrk）                              │
└─────────────────────────────────────────────────────┘
```

---

#### 3. Size Class：小对象的规格化分配

```
Go 将小对象按 size class 分为 67 种规格
每种规格对应一类 mspan

常见 size class：
┌────────┬───────────┬──────────────────────────┐
│ class  │  大小      │      典型对象             │
├────────┼───────────┼──────────────────────────┤
│   1    │   8B      │ 小整数指针                │
│   2    │  16B      │ 2个指针                   │
│   3    │  24B      │ 小 struct                 │
│   4    │  32B      │ 4个指针                   │
│  ...   │  ...      │  ...                     │
│  20    │  256B     │ 小 map entry              │
│  ...   │  ...      │  ...                     │
│  66    │  32KB     │ 最大小对象                 │
└────────┴───────────┴──────────────────────────┘

分配规则：
  申请 18B → 向上取整到 24B（class 3）
  申请 100B → 向上取整到 112B
  避免了碎片，提升了分配效率
```

---

#### 4. 小对象的分配流程

```
申请一个 32B 的小对象：

Step1: 查找 mcache 中对应 size class 的 mspan
  ┌──────────────────────────────────┐
  │  mcache                          │
  │  [class1][class2]...[class32]... │
  │                  ↑               │
  │           找到 32B 的 span       │
  └──────────────────────────────────┘
  span 中有空闲槽位 → 直接分配，返回指针（无锁，极快）

Step2: mcache 中该 span 满了 → 从 mcentral 获取新 span
  mcentral 维护 busy/free 两个 span 链表
  取一个有空闲槽的 span 给 mcache（需加锁）

Step3: mcentral 也没有 → 从 mheap 申请新内存页
  mheap 切割内存页，创建新 span（需加锁）

Step4: mheap 也不够 → 向 OS 申请内存（mmap）
```

---

#### 5. 大对象的分配流程

```
申请一个 1MB 的大对象：

  直接跳过 mcache 和 mcentral
  从 mheap 的 free treap（空闲树）中找合适的内存页
  ↓ 找不到
  向 OS 申请新内存（mmap）

大对象的特点：
  ① 每次分配都经过 mheap，有全局锁竞争
  ② 分配的内存页数多，占用虚拟地址空间大
  ③ GC 扫描时一次扫描大块内存
  ④ 生命周期通常较长，长期占用堆空间
```

---

#### 6. 为什么小对象多了会造成 GC 压力

```
原因①：触发 GC 更频繁

  Go GC 触发条件：堆内存 >= 上次GC后 × (1 + GOGC/100)
  
  大量小对象 → 堆内存快速增长 → 频繁触发 GC
  
  ┌───────────────────────────────────────────┐
  │  上次 GC 后堆 = 100MB，GOGC=100           │
  │  触发阈值 = 200MB                          │
  │                                           │
  │  每秒创建 100万个 64B 小对象               │
  │  每秒新增 = 100万 × 64B = 64MB            │
  │  ~1.5秒后触发 GC → GC 极其频繁！           │
  └───────────────────────────────────────────┘


原因②：GC 扫描工作量大

  GC 需要扫描所有堆上的对象，标记存活对象
  对象数量越多，扫描工作量越大

  100个 1MB 对象  → 100次扫描
  100万个 100B 对象 → 100万次扫描！
  
  总内存相同，但小对象数量多
  → 指针追踪次数更多
  → GC 标记阶段耗时更长
  → STW 和并发标记时间更长


原因③：内存碎片化

  大量小对象分散在各个 mspan 中
  部分 mspan 只有少量对象存活
  整个 span 无法归还 OS（有存活对象）
  造成内存碎片，实际内存利用率低

  ┌────────────────────────────────────────┐
  │  mspan（8KB，128个64B槽）              │
  │  ████░░░████░░░░░░████░░░████░░░░░░   │
  │  存活  死亡  存活   死亡  存活  死亡    │
  │                                        │
  │  只要有一个存活对象，整个 span 不能回收 │
  └────────────────────────────────────────┘


原因④：写屏障开销增大

  GC 并发标记阶段需要写屏障（Write Barrier）
  每次修改指针都触发写屏障检查
  小对象多 → 指针操作多 → 写屏障触发多
  → 影响程序正常执行速度（mutator 效率下降）


原因⑤：mcache 压力增大

  大量不同 size class 的小对象
  导致 mcache 中多个 span 被占满
  频繁向 mcentral 申请新 span（有锁竞争）
  → 分配延迟增加
```

---

#### 7. 大对象 vs 小对象 GC 压力对比

```
相同总内存（1GB）的场景：

方案A：1000个 1MB 大对象
  GC 扫描次数：1000次
  触发 GC 频率：较低（对象创建慢）
  内存碎片：少
  GC 耗时：较短

方案B：1000万个 100B 小对象
  GC 扫描次数：1000万次  ← 相差10000倍！
  触发 GC 频率：极高（对象创建快）
  内存碎片：多
  GC 耗时：极长

  ┌─────────────────────────────────────────┐
  │  GC 压力：方案B >> 方案A                 │
  │  相同内存，小对象数量多 → GC 压力更大     │
  └─────────────────────────────────────────┘
```

---

#### 8. 减少小对象 GC 压力的优化方案

```go
// ① sync.Pool 复用小对象（最有效）
var pool = sync.Pool{
    New: func() any {
        return &MyStruct{}
    },
}

func process() {
    obj := pool.Get().(*MyStruct)  // 从池取，不触发分配
    defer func() {
        obj.Reset()
        pool.Put(obj)              // 用完放回，不触发GC
    }()
    // 使用 obj...
}


// ② 预分配批量内存，避免逐个分配
// ❌ 每次分配一个
for i := 0; i < 10000; i++ {
    obj := &SmallStruct{}  // 10000次分配
    process(obj)
}

// ✅ 批量分配，切片复用
batch := make([]SmallStruct, 10000)  // 1次分配
for i := range batch {
    process(&batch[i])
}


// ③ 使用值类型代替指针类型（减少堆分配）
// ❌ 指针类型：逃逸到堆
type Node struct {
    Val  int
    Next *Node  // 指针，Next 指向的对象在堆上
}

// ✅ 值类型数组：减少指针，降低 GC 扫描
type NodePool struct {
    nodes [1024]Node  // 连续内存，GC 扫描更高效
    head  int
}


// ④ 减少临时对象产生
// ❌ 频繁创建临时 []byte
func handler(r *http.Request) {
    body, _ := io.ReadAll(r.Body)  // 每次新建
    process(body)
}

// ✅ 复用 buffer
var bufPool = sync.Pool{
    New: func() any { return new(bytes.Buffer) },
}
func handler(r *http.Request) {
    buf := bufPool.Get().(*bytes.Buffer)
    defer func() { buf.Reset(); bufPool.Put(buf) }()
    buf.ReadFrom(r.Body)
    process(buf.Bytes())
}


// ⑤ 调整 GOGC 减少 GC 频率（以内存换CPU）
// 小对象多时适当增大 GOGC
debug.SetGCPercent(200)  // 堆增长到3倍再触发GC
```

---

#### 9. 对象分配的黄金原则

```
原则①：能用栈就不用堆
  避免不必要的逃逸，减少堆分配

原则②：能复用就不新建
  sync.Pool 复用临时对象

原则③：能批量就不零散
  批量分配代替逐个 new

原则④：能值类型就不用指针
  减少指针数量，降低 GC 扫描负担

原则⑤：能预分配就不动态扩容
  make([]T, 0, expectedSize)
  预分配容量，避免多次扩容产生临时对象
```

---

#### 总结

| 对比项 | 小对象（< 32KB） | 大对象（> 32KB） |
|--------|---------------|---------------|
| 分配路径 | mcache → mcentral → mheap | 直接 mheap |
| 分配速度 | 极快（mcache 无锁） | 较慢（mheap 有锁） |
| GC 扫描 | 数量多，扫描次数多 | 数量少，扫描次数少 |
| 内存碎片 | 容易产生碎片 | 碎片较少 |
| GC 触发 | 快速堆增长，触发频繁 | 增长较慢，触发较少 |
| 优化手段 | sync.Pool、批量分配、值类型 | 复用、预分配 |

> 小对象造成 GC 压力的本质：**数量多导致扫描工作量呈线性增长，分配快导致堆增长速度快从而频繁触发 GC，碎片化导致内存利用率低，三者叠加使 GC 的标记、清扫、压缩开销远超大对象场景。**

### 22.介绍项目中遇到的oop情况
### 23.介绍项目中遇到的坑
### 24.如果指定指令执行的顺序
### 26.gc的stw是怎么回事
#### Go 的 GC STW 是怎么回事

#### Go GC 的整体设计思路

Go 使用**三色标记 + 混合写屏障**的并发 GC，核心目标是把 STW 时间压缩到**亚毫秒级**。大部分 GC 工作与用户 goroutine **并发**执行，只有极少数阶段需要 STW。

---

#### Go GC 的四个阶段

```
Mark Setup      →    Marking      →    Mark Termination    →    Sweeping
(STW, 极短)        (并发执行)          (STW, 极短)              (并发执行)
```

1. **Mark Setup（STW ✅）**
   - 开启**写屏障（Write Barrier）**
   - 必须等所有 goroutine 都到达安全点才能开启
   - STW 时间极短，微秒级

2. **Marking（并发，无 STW）**
   - GC goroutine 与用户 goroutine **同时运行**
   - 从 GC Roots 出发，三色标记遍历所有对象
   - 写屏障保证并发标记的正确性（新分配/修改的对象不会漏标）
   - 用户 goroutine 也会被征用来做标记工作（**辅助标记 Assist**）

3. **Mark Termination（STW ✅）**
   - 关闭写屏障
   - 处理并发标记阶段残留的少量任务
   - 同样是微秒级，是 STW 最主要的阶段

4. **Sweeping（并发，无 STW）**
   - 并发清扫未被标记的对象，回收内存
   - 与用户代码同时运行，不阻塞

---

#### 为什么 Mark Setup 必须 STW？

写屏障需要**原子地**开启，必须保证开启瞬间所有 goroutine 都感知到这一变化。如果某个 goroutine 在写屏障开启前修改了引用关系，就可能导致对象**漏标**被错误回收。

---

#### Go 如何通知 goroutine 进入 STW？

```
方式一：协作式（Go 1.14 之前）
  goroutine 在函数调用时主动检查是否需要暂停

方式二：异步抢占（Go 1.14+）
  runtime 向 goroutine 发送 SIGURG 信号
  goroutine 在信号处理时强制暂停
  → 解决了长时间运行的 goroutine 导致 STW 延迟的问题
```

---

#### Go STW 的实际耗时

| Go 版本 | 典型 STW 时间 |
|---|---|
| Go 1.4 及以前 | 数百毫秒 |
| Go 1.5 | 引入并发 GC，降至 10ms 以内 |
| Go 1.8 | 引入混合写屏障，降至 < 1ms |
| Go 1.14+ | 异步抢占，通常 < 100μs |

---

#### 混合写屏障的作用

并发标记期间，用户 goroutine 可能修改引用关系，写屏障负责"拦截"这些修改：

```go
// 伪代码：写屏障的逻辑
func writePointer(slot *ptr, new ptr) {
    shade(*slot)  // 将旧值标灰（防止丢失）
    shade(new)    // 将新值标灰（防止漏标）
    *slot = new
}
```

这样即使并发执行，也能保证所有存活对象都被正确标记。

---

#### 一句话总结

> Go 的 GC 通过**并发标记 + 混合写屏障**，将 STW 压缩到只有两个极短的时间点（开启/关闭写屏障），现代 Go 程序的 STW 通常在 **100 微秒以内**，对大多数业务几乎无感知。
### 27.协程之间是怎么调度的


### 28.简单聊聊内存逃逸 

#### 常见内存逃逸场景

**1. 返回局部变量的指针**

```go
func newUser() *User {
    u := User{Name: "Tom"} // u 逃逸到堆
    return &u
}
```

函数返回后栈帧销毁，但调用方仍持有 `&u`，所以 `u` 必须逃逸到堆。

---

**2. 变量被赋值给接口类型**

```go
var i interface{} = 42  // 42 逃逸到堆
fmt.Println(i)          // 同理，参数为 interface{} 也会逃逸
```

接口是 `(type, pointer)` 结构，编译期无法确定具体类型，统一逃逸到堆。

---

**3. 闭包引用外部变量**

```go
func counter() func() int {
    n := 0  // n 逃逸到堆
    return func() int {
        n++
        return n
    }
}
```

闭包生命周期比外层函数长，`n` 必须在堆上持续存活。

---

**4. 编译期大小不确定的变量**

```go
func makeSlice(n int) []int {
    s := make([]int, n)  // n 运行时才确定，逃逸到堆
    return s
}
```

---

**5. 变量过大超过栈限制**

```go
func big() {
    arr := [1 << 20]byte{}  // 1MB，超出栈大小，逃逸到堆
    _ = arr
}
```

---

**6. 发送指针到 channel**

```go
func send(ch chan *int) {
    n := 42     // n 逃逸到堆
    ch <- &n    // 接收方生命周期不确定
}
```

---

**7. slice/map 存储指针**

```go
func store() {
    m := make(map[string]*int)
    n := 1       // n 逃逸到堆
    m["a"] = &n  // map 的生命周期不确定
}
```

---

#### 如何查看逃逸

```bash
go build -gcflags='-m' main.go
```

```
./main.go:4:2: u escapes to heap
./main.go:10:3: n escapes to heap
```

---

#### 一句话总结

> 核心规律：**只要一个变量的生命周期超出了当前函数栈帧，或者编译器无法在编译期确定其大小/去向，它就会逃逸到堆。**


### 29.为什么sync.WaitGroup中Wait函数支持 WaitTimeout 功能
### 30.字符串转成byte数组，会发生内存拷贝吗？
#### 字符串转 []byte，会发生内存拷贝吗？

**通常情况下：会发生拷贝。** 但 Go 编译器在特定场景下会做优化，**零拷贝**处理。

---

#### 为什么默认会拷贝？

Go 中 `string` 和 `[]byte` 的底层结构不同：

```go
// string 底层结构
type StringHeader struct {
    Data uintptr  // 指向只读内存
    Len  int
}

// []byte 底层结构
type SliceHeader struct {
    Data uintptr  // 指向可读写内存
    Len  int
    Cap  int
}
```

- `string` 是**不可变**的，数据存在只读内存区
- `[]byte` 是**可变**的，需要可读写内存

两者语义不同，直接共享内存会破坏 string 的不可变性，所以**必须拷贝一份新内存**给 `[]byte`。

---

#### 编译器优化：这些场景零拷贝

Go 编译器识别出 `[]byte` **只是临时使用、不会被修改**时，会跳过拷贝：

**场景一：用于 map 查询**
```go
m := map[string]int{"hello": 1}
b := []byte("hello")
_ = m[string(b)]  // 零拷贝，编译器优化
```

**场景二：用于字符串比较**
```go
b := []byte("hello")
if string(b) == "hello" {  // 零拷贝
}
```

**场景三：for range 遍历**
```go
b := []byte("hello")
for i, c := range string(b) {  // 零拷贝
    _ = i, c
}
```

---

#### 强制零拷贝的"黑科技"写法

通过 `unsafe` 包直接复用底层内存指针，绕过拷贝：

```go
import "unsafe"

func stringToBytes(s string) []byte {
    sh := (*reflect.StringHeader)(unsafe.Pointer(&s))
    bh := reflect.SliceHeader{
        Data: sh.Data,
        Len:  sh.Len,
        Cap:  sh.Len,
    }
    return *(*[]byte)(unsafe.Pointer(&bh))
}
```

⚠️ **注意**：此方法返回的 `[]byte` 指向只读内存，如果对其修改会导致 **panic 或未定义行为**，仅适合只读场景，生产慎用。

Go 1.20+ 官方提供了更安全的方式：

```go
import "unsafe"

// Go 1.20+ 推荐
b := unsafe.Slice(unsafe.StringData(s), len(s))  // 零拷贝，但仍不可修改
```

---

#### 对比总结

| 场景 | 是否拷贝 | 说明 |
|---|---|---|
| 普通 `[]byte(str)` 转换 | ✅ 拷贝 | 默认行为，安全 |
| map 查询 / 字符串比较 | ❌ 零拷贝 | 编译器自动优化 |
| `unsafe` 强转 | ❌ 零拷贝 | 手动操作，不可修改 |
| Go 1.20 `unsafe.Slice` | ❌ 零拷贝 | 官方方式，仍不可修改 |

---

#### 一句话总结

> 字符串转 `[]byte` **默认会发生内存拷贝**，因为 string 不可变而 []byte 可变；但编译器在临时使用的特定场景会自动优化为零拷贝，极致性能场景可用 `unsafe` 手动实现零拷贝，但转换后**不能修改**该 []byte。


### 31.http包的内存泄漏
### 32.Goroutine调度策略

#### Goroutine 调度策略

---

#### 先理解 GMP 模型

Go 调度器基于 **GMP** 三个核心角色：

```
G（Goroutine）  → 用户级轻量线程，有自己的栈
M（Machine）    → 操作系统线程，真正执行代码
P（Processor）  → 逻辑处理器，持有本地任务队列，连接 G 和 M
```

```
P1 [本地队列: G2 G3 G4]          全局队列: [G7 G8 G9]
   └── M1 正在执行 G1

P2 [本地队列: G5 G6]
   └── M2 正在执行 G5
```

P 的数量默认等于 CPU 核心数（`GOMAXPROCS`），决定了最大并行度。

---

#### 核心调度策略

**1. 本地队列优先**

每个 P 持有一个**本地运行队列（LRQ）**，容量 256。G 优先从当前 P 的本地队列取，减少锁竞争。

```
G 被创建 → 优先放入当前 P 的本地队列
         → 本地队列满了 → 放入全局队列
```

---

**2. Work Stealing（工作窃取）**

某个 P 的本地队列为空时，不会闲着，而是去"偷"其他 P 的任务：

```
P2 本地队列空了
    ↓
从其他 P 的队列尾部偷取一半 G
    ↓
继续执行，避免 M 空转
```

```go
// 调度优先级伪代码
func schedule() {
    // 1. 每隔 61 次调度，从全局队列取（防止饥饿）
    if gp == nil && schedtick%61 == 0 {
        gp = globrunqget()
    }
    // 2. 从本地队列取
    if gp == nil {
        gp = runqget(p)
    }
    // 3. 本地没有，去偷
    if gp == nil {
        gp = stealWork()
    }
}
```

---

**3. 抢占式调度**

早期 Go 是协作式调度，goroutine 需要主动让出。Go 1.14 引入**异步抢占**：

```
协作式（Go 1.14 之前）              异步抢占（Go 1.14+）
──────────────────────              ─────────────────────
goroutine 在函数调用时              sysmon 线程监控所有 P
主动检查抢占标志                    发现某 G 运行超过 10ms
主动让出 CPU                        发送 SIGURG 信号强制抢占
                                    解决了死循环 goroutine 霸占 M 的问题
```

---

**4. sysmon 后台监控线程**

Go runtime 有一个**不需要 P 就能运行**的监控线程 sysmon，负责：

```
① 抢占运行超过 10ms 的 G
② 检测陷入系统调用过久的 M，解绑其 P 给其他 M 使用
③ 执行 netpoll，将网络就绪的 G 放回队列
④ 触发定时 GC
```

---

**5. 系统调用时的调度处理**

```
G 发起系统调用（阻塞）
    ↓
M 陷入内核，无法执行其他 G
    ↓
P 与该 M 解绑
    ↓
P 寻找空闲 M 或新建 M 继续执行队列中的其他 G
    ↓
系统调用返回 → 原 M 尝试重新绑定 P
             → 无空闲 P → G 放入全局队列，M 休眠
```

---

#### 调度时机汇总

| 触发场景 | 说明 |
|---|---|
| `runtime.Gosched()` | 主动让出 CPU |
| 系统调用 | M 阻塞，P 转移 |
| channel 阻塞 | G 挂起，调度其他 G |
| 函数调用（栈增长检查点） | 协作式抢占检查 |
| 运行超过 10ms | sysmon 异步抢占 |
| `time.Sleep` | G 挂起进入计时器队列 |

---

#### 一句话总结

> Go 调度器基于 GMP 模型，通过**本地队列优先、Work Stealing、异步抢占、sysmon 监控**四大策略，在用户态实现了高效的 M:N 调度，以极低的上下文切换开销支撑起百万级 goroutine 的并发。

### 33.对已经关闭的的chan进行读写，会怎么样？为什么？
### 34.实现阻塞读的并发安全Map
### 35.什么是goroutine leak？
### 36.data race问题怎么解决？能不能不加锁解决这个问题？
### grpc内部原理是什么？
### time.Now有几次系统调用？如何优化

#### Go `time` 包原理

---

#### 两种时钟来源

Go 的 `time.Now()` 内部同时读取**两种时钟**：

```
Wall Clock（墙上时钟）        Monotonic Clock（单调时钟）
──────────────────────        ──────────────────────────
反映现实世界时间               只增不减，从进程启动开始计数
会被 NTP 校准跳变              不受系统时间调整影响
用于显示时间                   用于计算时间差
```

```go
t := time.Now()
// 内部结构同时包含两个时钟读数：
// t.wall → 挂钟时间（含 wallToInternal 偏移）
// t.ext  → 单调时钟读数（nanoseconds since process start）
```

---

#### `time.Time` 的底层结构

```go
type Time struct {
    wall uint64   // 高1位标记是否有单调时钟
                  // 高33位存秒数
                  // 低30位存纳秒数
    ext  int64    // 有单调时钟时：存 monotonic 纳秒数
                  // 无单调时钟时：存完整秒数（扩展范围）
    loc  *Location // 时区信息
}
```

---

#### Wall Clock vs Monotonic 的使用场景

```go
// 计算耗时 → 用 Monotonic（自动）
start := time.Now()
time.Sleep(time.Second)
elapsed := time.Since(start) // 内部用 ext 字段做差，不受时钟跳变影响

// 比较绝对时间 → 用 Wall Clock
t1 := time.Now()
t2 := t1.Round(0) // Round(0) 会剥离 Monotonic 读数，只保留 Wall Clock
```

---

#### 系统调用链路

```
time.Now()
    ↓
runtime.walltime()  +  runtime.nanotime()
    ↓                        ↓
vDSO: clock_gettime      vDSO: clock_gettime
(CLOCK_REALTIME)         (CLOCK_MONOTONIC)
    ↓
直接读内核映射的用户态内存页
（无需陷入内核，零系统调用）
```

---

#### Timer 定时器原理

`time.NewTimer` / `time.After` 底层由 runtime 的**四叉堆（timerHeap）** 管理：

```
runtime 维护一个最小堆
    ↓
堆顶 = 最近要触发的 timer
    ↓
sysmon 或 P 在调度循环中检查堆顶
    ↓
到期 → 将回调 G 放入运行队列
```

```go
// 每个 P 持有自己的 timer 堆，减少锁竞争（Go 1.14+）
type p struct {
    timers     []*timer  // 四叉堆
    timerMask  uint32
}
```

定时器触发流程：

```
time.NewTimer(d)
    ↓
创建 timer，加入当前 P 的堆
    ↓
调度循环 → checkTimers() → 堆顶到期
    ↓
执行 timer.f()，向 channel 发送时间值
    ↓
阻塞在 <-timer.C 的 goroutine 被唤醒
```

---

#### Ticker 与 Timer 的区别

```go
// Timer：触发一次
t := time.NewTimer(time.Second)
<-t.C  // 1秒后触发，之后 channel 不再有数据

// Ticker：周期触发
tk := time.NewTicker(time.Second)
for range tk.C {  // 每秒触发一次
    // ...
}
```

底层区别：

```
Timer  → period = 0，触发后不重新入堆
Ticker → period > 0，触发后以 period 为间隔重新入堆
```

---

#### 常见陷阱

**陷阱一：Timer 未触发时要手动 Stop**

```go
t := time.NewTimer(time.Second)
// 提前不需要了
if !t.Stop() {
    <-t.C  // 必须排空 channel，否则下次 Reset 行为异常
}
```

**陷阱二：time.After 的内存泄漏**

```go
// ❌ 危险：每次循环都创建新 timer，旧 timer 在超时前不会被 GC
for {
    select {
    case <-time.After(time.Minute):  // 持续泄漏
    }
}

// ✅ 正确：复用 timer
timer := time.NewTimer(time.Minute)
defer timer.Stop()
for {
    timer.Reset(time.Minute)
    select {
    case <-timer.C:
    }
}
```

**陷阱三：time.Sleep 精度问题**

```go
// Sleep 的实际时间 >= 指定时间，受调度器影响
// 高精度场景不应依赖 Sleep 做计时
time.Sleep(time.Millisecond) // 实际可能睡 1~3ms
```

---

#### 时区处理原理

```go
time.LoadLocation("Asia/Shanghai")
    ↓
读取 /usr/share/zoneinfo/Asia/Shanghai（tzdata 文件）
    ↓
解析 POSIX 时区规则，缓存到 Location 结构体
    ↓
time.Time.In(loc) 做偏移计算（仅影响显示，不改变内部存储）
```

内部存储**始终是 UTC**，时区只影响格式化输出。

---

#### 一句话总结

> `time` 包核心是**双时钟设计**（Wall + Monotonic），通过 vDSO 零系统调用读时间，Timer/Ticker 由每个 P 的**四叉堆**管理调度，时区只影响显示不影响存储，使用时注意 Timer 泄漏和 Stop/Reset 的正确姿势。

### 空struct{}是否使用过？会在什么情况下使用，举例说明一下

这个问题我们刚刚详细讲过，核心用法总结如下：

**零内存特性** — `unsafe.Sizeof(struct{}{})` 输出 `0`，所有实例共享同一地址 `runtime.zerobase`。

**六大使用场景：**

1. `chan struct{}` — 事件通知，只传信号不传数据
2. `map[T]struct{}` — 实现 Set 集合
3. `make(chan struct{}, n)` — 控制并发的信号量
4. 无状态方法接收者 — 工具类/策略类对象
5. `context.WithValue` 的 key — 防止包间 key 冲突
6. `close(quit)` 广播退出信号 — Worker 退出控制

核心原则就一句话：**只需要"有或没有"的语义时，用 `struct{}`；需要传递具体值时，才用其他类型。**

往上翻可以看到每个场景的完整代码示例 👆

### 聊聊runtime

#### Go Runtime 是什么

Go Runtime 是**随每个 Go 程序一起编译进二进制的运行时库**，不依赖外部 VM，程序启动时自动初始化。

```
你的 Go 代码
    +
Go Runtime（调度器 + GC + 内存管理 + 系统调用封装...）
    ↓
编译为单一可执行文件
```

---

#### Runtime 的核心模块

---

**1. Goroutine 调度器（GMP）**

```
G（Goroutine）  用户级协程，初始栈 2KB，可动态增长
M（Machine）    OS 线程，真正执行代码
P（Processor）  逻辑处理器，持有本地任务队列

默认 P 数量 = CPU 核心数（GOMAXPROCS）
```

调度器负责把海量 G 映射到少量 M 上运行，实现 **M:N 调度**，核心策略是本地队列 + Work Stealing + 异步抢占（详见 GMP 篇）。

---

**2. 内存管理**

Go Runtime 自己管理内存，不直接用 `malloc`：

```
大对象（>32KB）  → 直接从堆分配
小对象（<32KB）  → mcache → mcentral → mheap 多级分配

mcache   每个 P 私有，无锁分配，极快
mcentral 全局，按大小分类的空闲链表
mheap    全局堆，向 OS 申请大块内存（Arena）
```

```
分配流程：
G 申请内存
  → 先查当前 P 的 mcache（无锁，极快）
  → mcache 不够 → 向 mcentral 申请
  → mcentral 不够 → 向 mheap 申请
  → mheap 不够 → 向 OS 申请
```

---

**3. 垃圾回收（GC）**

三色标记 + 混合写屏障 + 并发清扫，STW 压缩到微秒级（详见 GC 篇）。

Runtime 通过 `GOGC` 控制 GC 触发时机：

```go
// GOGC 默认值 100，表示堆增长 100% 时触发 GC
// GOGC=200 → 堆增长 200% 才触发，GC 更少但内存占用更高
// GOGC=off → 关闭 GC
os.Setenv("GOGC", "200")

// Go 1.19+ 可以设置堆内存上限
runtime/debug.SetMemoryLimit(512 << 20) // 最多用 512MB
```

---

**4. 栈管理**

Go 的栈是**动态增长**的，不像 OS 线程有固定栈大小：

```
goroutine 初始栈：2KB
    ↓
栈空间不够时（函数调用检测到）
    ↓
分配新栈（2倍扩容）
    ↓
复制所有数据到新栈（Go 1.3+ 连续栈）
    ↓
更新所有指针指向新地址
```

这就是为什么 Go 能轻松支持百万 goroutine — 初始栈极小，按需增长。

---

**5. 系统调用封装**

Runtime 封装了所有系统调用，在 syscall 前后做调度处理：

```
G 发起阻塞系统调用
    ↓
Runtime 将 M 与 P 解绑
    ↓
P 找其他 M 继续执行队列中的 G
    ↓
syscall 返回 → M 尝试重新绑定 P
```

网络 I/O 则通过 **netpoller**（封装 epoll/kqueue）实现非阻塞：

```
网络读写 → 注册到 epoll
         → G 挂起（不占 M）
         → 数据就绪 → sysmon 唤醒 G → 重新调度
```

---

**6. sysmon 监控线程**

Runtime 启动一个**不需要 P** 的后台线程，负责：

```
① 抢占运行超过 10ms 的 G（发 SIGURG）
② 检测长时间阻塞在 syscall 的 M，解绑其 P
③ 执行 netpoll，唤醒网络就绪的 G
④ 触发定时 GC
⑤ 归还长期空闲的内存给 OS（scavenger）
```

---

**7. 程序启动流程**

```
操作系统加载二进制
    ↓
runtime.rt0_go（汇编引导）
    ↓
初始化 g0（主 goroutine 的调度栈）
    ↓
runtime.schedinit()
  → 初始化 P 列表
  → 初始化内存分配器
  → 初始化 GC
    ↓
runtime.newproc(main.main)  创建主 goroutine
    ↓
runtime.mstart()  启动调度循环
    ↓
执行 main.main
```

---

**8. 常用 Runtime API**

```go
runtime.GOMAXPROCS(n)      // 设置 P 的数量
runtime.NumGoroutine()     // 当前 goroutine 数量
runtime.Gosched()          // 主动让出 CPU
runtime.GC()               // 手动触发 GC
runtime.GOOS               // 当前操作系统
runtime.GOARCH             // 当前 CPU 架构
runtime.NumCPU()           // CPU 核心数
runtime.Stack(buf, false)  // 获取当前 goroutine 栈信息
runtime.Caller(1)          // 获取调用者信息（文件/行号）
```

---

#### 各模块关系图

```
┌─────────────────────────────────────────┐
│              你的 Go 程序                 │
├─────────────────────────────────────────┤
│  调度器(GMP) │  GC  │  内存分配器          │
├─────────────────────────────────────────┤
│      netpoller    │    sysmon            │
├─────────────────────────────────────────┤
│           系统调用封装层                   │
├─────────────────────────────────────────┤
│              操作系统                     │
└─────────────────────────────────────────┘
```

---

#### 一句话总结

> Go Runtime 是内嵌在每个 Go 程序中的"微型操作系统"，核心是 **GMP 调度器、三色 GC、多级内存分配器**三大支柱，配合 netpoller 和 sysmon，让 Go 程序在用户态高效管理百万级并发，同时对开发者完全透明。

### 讲解一下 g0 m0

#### g0 和 m0 是什么

---

#### m0 — 主线程

`m0` 是 Go 程序启动时创建的**第一个 OS 线程**，是唯一一个在 runtime 中静态声明的 M：

```go
// runtime/proc.go
var (
    m0 m        // 静态声明，不需要在堆上分配
    g0 g        // 与 m0 绑定的 g0
)
```

普通 M 是动态创建的，m0 是**程序启动时由操作系统直接创建**的主线程，整个进程生命周期内唯一。

---

#### g0 — 调度协程

每个 M 都有一个自己的 `g0`，不只是 m0 才有：

```
m0 → 绑定 → g0（主线程的调度栈）
m1 → 绑定 → g0（m1 的调度栈）
m2 → 绑定 → g0（m2 的调度栈）
```

**g0 不是普通 goroutine**，它的特殊之处：

```
普通 G                      g0
──────────────────          ────────────────────────
用户代码运行的地方            调度代码运行的地方
初始栈 2KB，动态增长          栈固定且更大（8KB 或系统栈）
有生命周期，可被回收           与 M 共存亡，永不回收
受调度器调度                  调度器本身就跑在 g0 上
```

---

#### g0 的核心作用

**g0 专门用来执行 runtime 自身的代码**，包括：

```
① 调度循环 schedule()
② goroutine 的创建 newproc()
③ GC 相关工作
④ 栈扩容/收缩
⑤ 系统调用的前后处理
```

**为什么要单独用 g0 跑这些？**

```
用户 G 的栈很小（初始 2KB）
runtime 的某些操作（如栈扩容本身）不能在被扩容的栈上运行
→ 必须切换到一个独立的、固定大小的栈上执行
→ 这个栈就是 g0 的栈
```

---

#### 启动流程中 m0/g0 的初始化

```
操作系统创建进程
    ↓
runtime/asm_amd64.s: rt0_go（汇编入口）
    ↓
① 初始化 g0 的栈边界（stacklo / stackhi）
    ↓
② 将 g0 地址写入 TLS（线程本地存储）
   让 getg() 能拿到当前 g
    ↓
③ 初始化 m0，将 m0.g0 = &g0 相互绑定
    ↓
④ runtime.schedinit() 在 g0 上执行
   → 初始化 P 列表
   → 初始化内存分配器
    ↓
⑤ runtime.newproc() 创建 main goroutine（第一个普通 G）
    ↓
⑥ runtime.mstart() 启动调度循环
   → m0 开始在 g0 上跑 schedule()
   → 取出 main goroutine 开始执行
```

---

#### g0 与普通 G 的切换过程

M 在 g0 和普通 G 之间不断切换：

```
M 上的执行流：

g0(schedule)                    找到可运行的 G
    ↓
execute(gp)                     切换到用户 G 的栈
    ↓
[用户代码运行]                   goroutine 正常执行
    ↓
触发调度（阻塞/抢占/主动让出）
    ↓
mcall(gopark / gosched_m)       从用户栈切回 g0 栈
    ↓
g0(schedule)                    重新调度下一个 G
```

关键函数：
```go
// 用户 G → g0：通过 mcall
// mcall 切换到 g0 的栈，然后调用传入的函数
func mcall(fn func(*g))

// g0 → 用户 G：通过 gogo
// gogo 恢复用户 G 的寄存器和栈指针
func gogo(buf *gobuf)
```

---

#### 用一张图表示

```
┌─────────────────────────────────────┐
│               M0                    │
│                                     │
│  ┌──────────┐      ┌─────────────┐  │
│  │    g0    │ ←──→ │  用户 G     │  │
│  │(调度栈)  │      │(用户代码栈)  │  │
│  │          │      │             │  │
│  │schedule()│      │ func main() │  │
│  │newproc() │      │ user code.. │  │
│  │gcWork()  │      │             │  │
│  └──────────┘      └─────────────┘  │
│    mcall() ↕ gogo()                 │
└─────────────────────────────────────┘
         ↕ TLS 寄存器
    getg() 获取当前运行的 g
```

---

#### TLS 的作用

Runtime 通过 **TLS（Thread Local Storage）** 让每个 M 随时知道当前在哪个 G 上运行：

```go
// 伪代码
func getg() *g {
    return *(*g)(getThreadLocalStorage()) // 读 TLS 寄存器
}

// 调度器随时可以调用 getg() 知道：
// - 当前是 g0？→ 正在执行 runtime 代码
// - 当前是普通 G？→ 正在执行用户代码
```

---

#### 一句话总结

> `m0` 是进程的**主 OS 线程**，静态初始化无需堆分配；`g0` 是每个 M 专属的**调度协程**，runtime 的核心逻辑（调度、GC、栈管理）都运行在 g0 的栈上，用户代码和 runtime 代码通过 `mcall/gogo` 在 g0 和普通 G 之间来回切换。

### 介绍下你平时都是怎么调试bug以及性能问题的?

#### 调试 Bug

---

**1. 日志定位（最常用）**

```go
// 结构化日志，带上关键上下文
log.Printf("order failed: orderID=%s userID=%d err=%v", orderID, userID, err)

// 生产环境推荐 zap / zerolog，性能更好
zap.L().Error("order failed",
    zap.String("orderID", orderID),
    zap.Int64("userID", userID),
    zap.Error(err),
)
```

关键原则：**错误要带上下文往上抛，不要只打 err.Error()**

```go
// ❌ 信息丢失
return err

// ✅ 保留调用链
return fmt.Errorf("createOrder: %w", err)
```

---

**2. Delve 断点调试**

```bash
# 安装
go install github.com/go-delve/delve/cmd/dlv@latest

# 调试本地程序
dlv debug main.go

# attach 到运行中的进程
dlv attach <pid>

# 常用命令
(dlv) break main.go:42      # 打断点
(dlv) continue              # 运行到断点
(dlv) next                  # 下一行
(dlv) step                  # 进入函数
(dlv) print variableName    # 打印变量
(dlv) goroutines            # 查看所有 goroutine
(dlv) goroutine 18 bt       # 查看指定 goroutine 的调用栈
```

VSCode / GoLand 内置了 Delve，可以直接图形化断点调试。

---

**3. 单元测试 + 表驱动验证**

```go
func TestCreateOrder(t *testing.T) {
    cases := []struct {
        name    string
        input   Order
        wantErr bool
    }{
        {"正常下单", Order{Amount: 100}, false},
        {"金额为负", Order{Amount: -1}, true},
    }

    for _, c := range cases {
        t.Run(c.name, func(t *testing.T) {
            err := CreateOrder(c.input)
            if (err != nil) != c.wantErr {
                t.Errorf("got err=%v, wantErr=%v", err, c.wantErr)
            }
        })
    }
}
```

---

**4. race detector 检测数据竞争**

```bash
go test -race ./...
go run -race main.go
```

输出示例：
```
WARNING: DATA RACE
Write at 0x00c0000b4010 by goroutine 7:
  main.add()  main.go:12

Previous read at 0x00c0000b4010 by goroutine 6:
  main.read() main.go:8
```

---

**5. 线上 panic 排查**

```go
// recover 捕获 panic 并打印完整堆栈
defer func() {
    if r := recover(); r != nil {
        buf := make([]byte, 4096)
        n := runtime.Stack(buf, false)
        log.Errorf("panic: %v\n%s", r, buf[:n])
    }
}()
```

---

#### 性能问题排查

---

**1. pprof — 性能剖析（最核心）**

```go
import _ "net/http/pprof"

func main() {
    go http.ListenAndServe(":6060", nil) // 开启 pprof 端口
    // ...
}
```

采集并分析：

```bash
# CPU 耗时分析（采集 30 秒）
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30

# 内存分析
go tool pprof http://localhost:6060/debug/pprof/heap

# goroutine 泄漏排查
go tool pprof http://localhost:6060/debug/pprof/goroutine

# 进入交互界面后
(pprof) top10       # 耗时 top10 函数
(pprof) list func   # 查看某函数的逐行耗时
(pprof) web         # 生成火焰图（需要 graphviz）
```

火焰图示例含义：
```
横轴 = 函数占用 CPU 时间比例（越宽越耗时）
纵轴 = 调用栈深度
找最宽的"平顶山" = 性能瓶颈所在
```

---

**2. Benchmark 基准测试**

```go
func BenchmarkCreateOrder(b *testing.B) {
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        CreateOrder(Order{Amount: 100})
    }
}
```

```bash
# 运行 benchmark
go test -bench=. -benchmem ./...

# 输出
BenchmarkCreateOrder-8   1000000   1023 ns/op   256 B/op   4 allocs/op
#                        次数       每次耗时      每次内存    每次分配次数
```

---

**3. trace — 调度延迟分析**

```bash
# 采集 trace
curl -o trace.out http://localhost:6060/debug/pprof/trace?seconds=5

# 可视化分析
go tool trace trace.out
```

能看到：
```
→ goroutine 调度延迟
→ GC 停顿时间点
→ 系统调用阻塞情况
→ P 的利用率
```

---

**4. 内存逃逸分析**

```bash
go build -gcflags='-m' ./...
# 找出不该逃逸到堆的变量，减少 GC 压力
```

---

**5. 线上监控（prometheus + grafana）**

```go
// 埋点关键指标
var orderDuration = prometheus.NewHistogramVec(
    prometheus.HistogramOpts{
        Name:    "order_duration_seconds",
        Buckets: prometheus.DefBuckets,
    },
    []string{"status"},
)

// 记录耗时
start := time.Now()
err := CreateOrder(order)
orderDuration.WithLabelValues(statusLabel(err)).
    Observe(time.Since(start).Seconds())
```

---

#### 排查思路总结

```
Bug 排查优先级：
日志上下文 → race detector → 单测复现 → Delve 断点 → panic 堆栈

性能排查优先级：
Benchmark 确认问题 → pprof CPU/内存 → 逃逸分析 → trace 调度
→ 线上监控告警
```

---

#### 一句话总结

> Bug 靠**结构化日志 + race detector + Delve**；性能靠 **pprof 火焰图 + Benchmark + trace**，线上靠 **prometheus 监控提前发现**，核心原则是**用数据说话，不靠猜**。

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
#### chan 比 mutex 更轻吗？

**不一定，大多数场景下 mutex 比 chan 更轻。**

---

#### 底层结构对比

```go
// sync.Mutex 底层结构
type Mutex struct {
    state int32  // 4 bytes
    sema  uint32 // 4 bytes
}
// 总计：8 bytes

// chan 底层结构（hchan）
type hchan struct {
    qcount   uint          // 队列中元素数量
    dataqsiz uint          // 环形队列容量
    buf      unsafe.Pointer // 环形缓冲区
    elemsize uint16
    closed   uint32
    elemtype *_type
    sendx    uint          // 发送索引
    recvx    uint          // 接收索引
    recvq    waitq         // 等待接收的 goroutine 队列
    sendq    waitq         // 等待发送的 goroutine 队列
    lock     Mutex         // 内部也用了 mutex！
}
// 总计：~100 bytes，内部还包含一个 Mutex
```

**chan 内部本身就包含一个 Mutex**，还有额外的队列和缓冲区开销。

---

#### Benchmark 对比

```go
// 简单计数器场景
BenchmarkMutex-8      100000000    11 ns/op
BenchmarkChannel-8     30000000    45 ns/op
BenchmarkAtomic-8     300000000     4 ns/op
```

性能排序：**atomic > mutex > channel**

---

#### 什么时候该用哪个？

```
需要传递数据 / 异步通信          → chan
保护共享资源 / 临界区             → Mutex
简单计数 / 状态标记               → atomic
读多写少                          → RWMutex
```

---

#### 更轻量的方案

---

**1. atomic — 最轻量（无锁操作）**

```go
import "sync/atomic"

// 计数器
var count int64
atomic.AddInt64(&count, 1)       // 原子加
atomic.LoadInt64(&count)         // 原子读
atomic.StoreInt64(&count, 0)     // 原子写
atomic.CompareAndSwapInt64(&count, old, new) // CAS

// Go 1.19+ 泛型版本更易用
var flag atomic.Bool
flag.Store(true)
flag.Load()

var val atomic.Int64
val.Add(1)
```

原理：直接使用 CPU 的 **CAS 指令**（LOCK XCHG），不涉及 goroutine 挂起，是最轻量的同步手段。

---

**2. RWMutex — 读多写少场景**

```go
var mu sync.RWMutex
var cache map[string]string

// 读操作：允许多个 goroutine 并发读
func Get(key string) string {
    mu.RLock()
    defer mu.RUnlock()
    return cache[key]
}

// 写操作：独占
func Set(key, val string) {
    mu.Lock()
    defer mu.Unlock()
    cache[key] = val
}
```

```
读:写 = 9:1 时，RWMutex 比 Mutex 快 ~3-5 倍
```

---

**3. sync.Map — 特定场景的并发 Map**

```go
var m sync.Map

m.Store("key", "value")

val, ok := m.Load("key")

m.LoadOrStore("key", "default")

m.Range(func(k, v interface{}) bool {
    fmt.Println(k, v)
    return true // 返回 false 停止遍历
})
```

适合场景：
```
✅ key 基本只写一次，读多次（如路由表、配置）
✅ 多个 goroutine 操作不相交的 key 集合
❌ 频繁写入同一批 key → 用 Mutex + map 更好
```

---

**4. Once — 只执行一次**

```go
var (
    once     sync.Once
    instance *Config
)

func GetConfig() *Config {
    once.Do(func() {
        instance = loadConfig() // 只执行一次，天然并发安全
    })
    return instance
}
```

比手写 double-check 加锁更简洁、更安全。

---

**5. Pool — 减少内存分配**

```go
var bufPool = sync.Pool{
    New: func() interface{} {
        return make([]byte, 4096)
    },
}

func handle(data []byte) {
    buf := bufPool.Get().([]byte)
    defer bufPool.Put(buf) // 用完归还，下次复用

    // 使用 buf 处理数据...
}
```

减少堆分配，降低 GC 压力，高并发场景收益明显。

---

#### 各方案横向对比

| 方案 | 耗时 | 适用场景 | 复杂度 |
|---|---|---|---|
| `atomic` | ~4ns | 计数、状态标记 | 低 |
| `sync.Mutex` | ~11ns | 通用临界区保护 | 低 |
| `sync.RWMutex` | ~4ns(读) | 读多写少 | 低 |
| `sync.Map` | ~15ns | 特定 map 场景 | 低 |
| `chan` | ~45ns | 通信、异步传值 | 中 |
| `sync.Pool` | ~8ns | 对象复用 | 中 |

---

#### 选型口诀

```
传递数据用 chan
保护数据用 Mutex
只读多用 RWMutex
简单标记用 atomic
对象复用用 Pool
执行一次用 Once
```

---

#### 一句话总结

> chan **不比 mutex 轻**，其内部就封装了 mutex 加队列；更轻量的选择是 **atomic（最轻）→ Mutex → RWMutex → chan**，根据场景选型，核心原则是 **用通信传递数据，用 atomic/mutex 保护状态**。

### 多个携程间的通信方法

#### 多个 Goroutine 间的通信方法

---

#### 1. channel — 最地道的 Go 方式

**单向数据传递：**

```go
ch := make(chan int, 1)

go func() {
    ch <- 42 // 发送
}()

val := <-ch // 接收
```

**一对多广播（close）：**

```go
done := make(chan struct{})

for i := 0; i < 3; i++ {
    go func(id int) {
        <-done // 阻塞等待
        fmt.Printf("goroutine %d 收到信号\n", id)
    }(i)
}

close(done) // 同时唤醒所有等待者
```

**多路复用（select）：**

```go
func worker(ch1, ch2 <-chan int, quit <-chan struct{}) {
    for {
        select {
        case v := <-ch1:
            fmt.Println("ch1:", v)
        case v := <-ch2:
            fmt.Println("ch2:", v)
        case <-quit:
            fmt.Println("退出")
            return
        }
    }
}
```

---

#### 2. 共享内存 + sync 保护

**Mutex 保护共享变量：**

```go
type SafeCounter struct {
    mu    sync.Mutex
    count int
}

func (c *SafeCounter) Add() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.count++
}

func (c *SafeCounter) Get() int {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.count
}
```

**RWMutex 读多写少：**

```go
type SafeCache struct {
    mu   sync.RWMutex
    data map[string]string
}

func (c *SafeCache) Get(key string) string {
    c.mu.RLock()         // 多个 goroutine 可并发读
    defer c.mu.RUnlock()
    return c.data[key]
}

func (c *SafeCache) Set(key, val string) {
    c.mu.Lock()          // 写时独占
    defer c.mu.Unlock()
    c.data[key] = val
}
```

---

#### 3. atomic — 轻量级状态共享

```go
var (
    counter int64
    running atomic.Bool
)

// 多个 goroutine 安全地操作
go func() { atomic.AddInt64(&counter, 1) }()
go func() { atomic.AddInt64(&counter, 1) }()

// 状态标记
running.Store(true)
if running.Load() {
    fmt.Println("运行中")
}
```

适合**简单计数、状态标记**，不适合复杂数据结构。

---

#### 4. sync.Map — 并发安全的 Map

```go
var m sync.Map

// 多个 goroutine 并发写不同 key
go func() { m.Store("a", 1) }()
go func() { m.Store("b", 2) }()

// 读取
val, ok := m.Load("a")

// 不存在则写入，存在则返回已有值
actual, loaded := m.LoadOrStore("c", 3)
fmt.Println(actual, loaded)
```

---

#### 5. context — 跨 goroutine 取消/超时传递

```go
ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
defer cancel()

go func(ctx context.Context) {
    select {
    case <-time.After(5 * time.Second):
        fmt.Println("工作完成")
    case <-ctx.Done():
        fmt.Println("被取消:", ctx.Err()) // context deadline exceeded
    }
}(ctx)
```

**一个 context 可以同时取消多个 goroutine：**

```go
ctx, cancel := context.WithCancel(context.Background())

for i := 0; i < 5; i++ {
    go func(id int, ctx context.Context) {
        <-ctx.Done()
        fmt.Printf("goroutine %d 退出\n", id)
    }(i, ctx)
}

cancel() // 同时通知所有 goroutine 退出
```

---

#### 6. WaitGroup — 等待多个 goroutine 完成

```go
var wg sync.WaitGroup
results := make([]int, 5)

for i := 0; i < 5; i++ {
    wg.Add(1)
    go func(id int) {
        defer wg.Done()
        results[id] = id * id // 每个 goroutine 写不同下标，无竞争
    }(i)
}

wg.Wait()
fmt.Println(results) // [0 1 4 9 16]
```

---

#### 7. errgroup — WaitGroup 的增强版（带错误收集）

```go
import "golang.org/x/sync/errgroup"

g, ctx := errgroup.WithContext(context.Background())

for i := 0; i < 3; i++ {
    id := i
    g.Go(func() error {
        if id == 2 {
            return fmt.Errorf("task %d failed", id)
        }
        return nil
    })
}

if err := g.Wait(); err != nil {
    fmt.Println("有任务失败:", err)
}
```

---

#### 各方案对比

| 方案 | 适用场景 | 方向 |
|---|---|---|
| `channel` | 数据传递、事件通知、流水线 | 单向/双向 |
| `Mutex/RWMutex` | 保护共享资源 | 共享内存 |
| `atomic` | 计数器、状态标记 | 共享内存 |
| `sync.Map` | 并发读写 map | 共享内存 |
| `context` | 超时/取消信号传播 | 单向广播 |
| `WaitGroup` | 等待一批任务完成 | 同步屏障 |
| `errgroup` | 等待任务 + 收集错误 | 同步屏障 |

---

#### 选型决策树

```
需要传递数据？
  ├── 是 → channel
  └── 否 → 只是同步状态？
              ├── 简单计数/标记 → atomic
              ├── 保护复杂结构 → Mutex / RWMutex
              ├── 等待完成     → WaitGroup / errgroup
              └── 传播取消     → context
```

---

#### 一句话总结

> Go 提供了 **channel（通信）、sync 族（共享内存保护）、context（取消传播）** 三类通信手段，优先用 channel 传递数据，用 Mutex/atomic 保护状态，用 context 传播取消信号。

### 除了mutex以外还有哪些方式安全读写共享变量

#### 除了 Mutex，还有哪些方式安全读写共享变量

---

#### 1. atomic — 原子操作

```go
var count int64

// 原子加减
atomic.AddInt64(&count, 1)
atomic.AddInt64(&count, -1)

// 原子读写
val := atomic.LoadInt64(&count)
atomic.StoreInt64(&count, 100)

// CAS（Compare And Swap）乐观锁
old, new := int64(1), int64(2)
swapped := atomic.CompareAndSwapInt64(&count, old, new)
```

Go 1.19+ 泛型封装更易用：

```go
var flag atomic.Bool
var num  atomic.Int64
var ptr  atomic.Pointer[Config]

flag.Store(true)
num.Add(1)
ptr.Store(&Config{})
```

**原理**：直接映射 CPU 的 `LOCK` 指令，无需 goroutine 挂起，是最轻量的同步方式。

---

#### 2. channel — 串行化访问

把共享变量的所有读写操作**集中在一个 goroutine** 里，外部通过 channel 通信：

```go
type Cache struct {
    get chan getReq
    set chan setReq
}

type getReq struct {
    key  string
    resp chan string
}

type setReq struct {
    key, val string
}

func NewCache() *Cache {
    c := &Cache{
        get: make(chan getReq),
        set: make(chan setReq),
    }
    go c.run() // 只有这一个 goroutine 访问 data
    return c
}

func (c *Cache) run() {
    data := make(map[string]string)
    for {
        select {
        case req := <-c.get:
            req.resp <- data[req.key]
        case req := <-c.set:
            data[req.key] = req.val
        }
    }
}

func (c *Cache) Get(key string) string {
    resp := make(chan string, 1)
    c.get <- getReq{key, resp}
    return <-resp
}

func (c *Cache) Set(key, val string) {
    c.set <- setReq{key, val}
}
```

**原理**：共享变量只被单一 goroutine 持有，天然无竞争，这正是 Go 的核心哲学：
> *Don't communicate by sharing memory, share memory by communicating.*

---

#### 3. sync.Map — 并发安全 Map

```go
var m sync.Map

// 写
m.Store("key", "value")

// 读
val, ok := m.Load("key")

// 不存在则写，存在则返回旧值
actual, loaded := m.LoadOrStore("key", "new")

// 删除
m.Delete("key")

// 遍历
m.Range(func(k, v any) bool {
    fmt.Println(k, v)
    return true
})
```

内部原理：

```
读操作 → 优先读 readOnly map（无锁，原子操作）
写操作 → 操作 dirty map（加锁）
miss 次数过多 → dirty 提升为 readOnly
```

适合**读多写少、key 稳定**的场景，频繁写用 `Mutex + map` 性能更好。

---

#### 4. sync.RWMutex — 读写分离锁

```go
var mu sync.RWMutex
var data = make(map[string]int)

// 并发读，不互斥
func Read(key string) int {
    mu.RLock()
    defer mu.RUnlock()
    return data[key]
}

// 写时独占，阻塞所有读
func Write(key string, val int) {
    mu.Lock()
    defer mu.Unlock()
    data[key] = val
}
```

```
读:写 = 9:1 场景下，比 Mutex 快 3~5 倍
```

---

#### 5. Copy-On-Write — 写时复制

读操作直接读，写操作复制一份新数据修改后原子替换，适合**写极少、读极多**的场景：

```go
type Config struct {
    settings map[string]string
}

var cfg atomic.Pointer[Config]

func init() {
    cfg.Store(&Config{settings: map[string]string{}})
}

// 读：直接读，无锁
func GetSetting(key string) string {
    return cfg.Load().settings[key]
}

// 写：复制 + 原子替换
func UpdateSetting(key, val string) {
    for {
        old := cfg.Load()

        // 复制一份新的
        newSettings := make(map[string]string, len(old.settings))
        for k, v := range old.settings {
            newSettings[k] = v
        }
        newSettings[key] = val

        newCfg := &Config{settings: newSettings}

        // CAS 原子替换
        if cfg.CompareAndSwap(old, newCfg) {
            return
        }
        // 替换失败说明有并发写，重试
    }
}
```

---

#### 6. sync.Once — 只写一次

```go
var (
    once     sync.Once
    instance *DB
)

func GetDB() *DB {
    once.Do(func() {
        instance = connectDB() // 全局只执行一次，并发安全
    })
    return instance
}
```

适合**单例初始化**，比 double-check 加锁更简洁安全。

---

#### 各方案对比

| 方案 | 性能 | 适用场景 |
|---|---|---|
| `atomic` | ⭐⭐⭐⭐⭐ | 计数器、简单状态标记 |
| `sync.RWMutex` | ⭐⭐⭐⭐ | 读多写少的复杂结构 |
| `sync.Map` | ⭐⭐⭐ | 读多写少、key 稳定的 map |
| `channel 串行化` | ⭐⭐⭐ | 复杂状态机、符合 Go 哲学 |
| Copy-On-Write | ⭐⭐⭐⭐ | 写极少、读极多的配置类数据 |
| `sync.Once` | ⭐⭐⭐⭐⭐ | 只初始化一次的单例 |

---

#### 一句话总结

> 除 Mutex 外，**atomic 最轻量**适合简单状态，**channel 串行化**最符合 Go 哲学，**RWMutex** 适合读多写少，**Copy-On-Write** 适合配置热更新，根据读写比例和数据复杂度选择合适方案。
### 用过什么包管理工具

#### Go 包管理工具

---

#### 发展历史

```
GOPATH 时代                dep 时代              Go Modules 时代
（Go 1.11 之前）           （官方实验性）          （Go 1.11+，1.16 默认开启）
所有代码放 $GOPATH/src     第一个半官方方案         go mod，目前标准方案
无版本概念                  有 vendor 和锁文件      语义化版本，完整依赖管理
```

---

#### 现在的标准：Go Modules

**初始化项目：**

```bash
go mod init github.com/yourname/yourproject
```

生成两个文件：

```
go.mod   → 声明模块名和依赖
go.sum   → 所有依赖的哈希校验，保证安全可复现
```

**go.mod 示例：**

```
module github.com/yourname/yourproject

go 1.21

require (
    github.com/gin-gonic/gin v1.9.1
    go.uber.org/zap v1.26.0
    gorm.io/gorm v1.25.4
)

require (
    // indirect 间接依赖
    golang.org/x/net v0.17.0 // indirect
)
```

---

#### 常用命令

```bash
# 添加依赖（自动更新 go.mod）
go get github.com/gin-gonic/gin@v1.9.1

# 升级到最新版本
go get github.com/gin-gonic/gin@latest

# 整理依赖（删除无用，补全缺失）
go mod tidy

# 下载所有依赖到本地缓存
go mod download

# 查看依赖树
go mod graph

# 将依赖复制到项目 vendor 目录（离线构建用）
go mod vendor

# 检查依赖是否有安全问题
go mod verify
```

---

#### 版本号规范（语义化版本）

```
v1.2.3
│ │ └─ Patch：bug 修复，向后兼容
│ └─── Minor：新功能，向后兼容
└───── Major：破坏性变更，需改 import 路径

// v2+ 版本需要修改 import 路径
import "github.com/foo/bar/v2"
```

版本选择机制（MVS）：

```
A 依赖 C v1.2
B 依赖 C v1.3
→ Go 自动选择 C v1.3（最小满足版本原则）
```

---

#### replace — 本地开发调试神器

```
// go.mod
replace (
    // 用本地路径替换远程包（开发联调）
    github.com/yourname/common => ../common

    // 替换某个特定版本（紧急修复）
    github.com/foo/bar v1.2.0 => github.com/myfork/bar v1.2.0-fix
)
```

---

#### 私有仓库配置

```bash
# 设置私有仓库，跳过 proxy 和 checksum 校验
go env -w GONOSUMDB="gitlab.company.com/*"
go env -w GOFLAGS="-mod=mod"

# 配置私有仓库地址
go env -w GOPRIVATE="gitlab.company.com/*"
```

---

#### 国内加速配置

```bash
# 设置代理（七牛云，推荐）
go env -w GOPROXY="https://goproxy.cn,direct"

# 或阿里云
go env -w GOPROXY="https://mirrors.aliyun.com/goproxy/,direct"
```

---

#### vendor 模式 vs 普通模式

```bash
# vendor 模式：依赖复制到项目内，适合 CI/离线环境
go mod vendor
go build -mod=vendor ./...

# 普通模式：依赖缓存在 $GOPATH/pkg/mod，多项目共享
go build ./...
```

```
vendor 优点：                    vendor 缺点：
✅ 离线可构建                    ❌ 仓库体积大
✅ 不依赖外网和 proxy            ❌ 依赖更新麻烦
✅ 构建速度快                    ❌ 代码审查噪声多
```

---

#### 工具对比

| 工具 | 时代 | 现状 |
|---|---|---|
| GOPATH | Go 早期 | 已淘汰 |
| `govendor` | 1.5~1.10 | 已淘汰 |
| `dep` | 1.9~1.11 | 已淘汰 |
| **Go Modules** | 1.11+ | ✅ 当前标准 |

---

#### 一句话总结

> Go 目前唯一标准是 **Go Modules**，核心是 `go.mod` 声明依赖 + `go.sum` 保证安全，日常用 `go get` 加依赖、`go mod tidy` 整理、`GOPROXY` 加速，私有仓库配 `GOPRIVATE` 跳过校验。

### golang的服务性能指标怎么查看，有哪些指标需要注意

#### Go 服务性能指标查看与关注点

---

#### 一、内置 pprof — 最核心的工具

```go
import _ "net/http/pprof"

func main() {
    go http.ListenAndServe(":6060", nil)
}
```

访问地址：

```bash
# 网页总览
http://localhost:6060/debug/pprof/

# CPU 耗时（采集 30 秒）
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30

# 堆内存
go tool pprof http://localhost:6060/debug/pprof/heap

# goroutine 数量和栈
go tool pprof http://localhost:6060/debug/pprof/goroutine

# 阻塞分析
go tool pprof http://localhost:6060/debug/pprof/block

# 锁竞争
go tool pprof http://localhost:6060/debug/pprof/mutex
```

交互命令：

```bash
(pprof) top10        # 耗时 top10 函数
(pprof) list func    # 某函数逐行耗时
(pprof) web          # 生成火焰图（需要 graphviz）
```

---

#### 二、runtime/metrics — 运行时指标

```go
import "runtime/metrics"

func collectMetrics() {
    samples := []metrics.Sample{
        {Name: "/memory/classes/heap/objects:bytes"},  // 堆对象内存
        {Name: "/gc/cycles/total:gc-cycles"},          // GC 总次数
        {Name: "/sched/goroutines:goroutines"},        // goroutine 数量
        {Name: "/cpu/classes/gc/total:cpu-seconds"},  // GC CPU 消耗
    }

    metrics.Read(samples)
    for _, s := range samples {
        fmt.Printf("%s = %v\n", s.Name, s.Value)
    }
}
```

---

#### 三、expvar — 轻量指标暴露

```go
import "expvar"

var (
    reqCount  = expvar.NewInt("request_total")
    errCount  = expvar.NewInt("error_total")
    cacheHit  = expvar.NewInt("cache_hit")
)

func handler(w http.ResponseWriter, r *http.Request) {
    reqCount.Add(1)
    // ...
}

// 访问 http://localhost:6060/debug/vars 查看所有指标
```

---

#### 四、Prometheus + Grafana — 生产环境标配

```go
import (
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promhttp"
)

var (
    // 请求耗时分布
    httpDuration = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "http_request_duration_seconds",
            Buckets: prometheus.DefBuckets,
        },
        []string{"path", "status"},
    )

    // 当前 goroutine 数量
    goroutineGauge = prometheus.NewGauge(prometheus.GaugeOpts{
        Name: "go_goroutines_current",
        Help: "Current goroutine count",
    })
)

func init() {
    prometheus.MustRegister(httpDuration, goroutineGauge)
}

// 暴露指标接口
http.Handle("/metrics", promhttp.Handler())
```

---

#### 五、关键性能指标清单

---

**① 内存指标**

```go
var ms runtime.MemStats
runtime.ReadMemStats(&ms)

ms.HeapAlloc    // 当前堆内存使用量       ⚠️ 持续增长 → 内存泄漏
ms.HeapSys      // 向 OS 申请的堆内存总量
ms.HeapObjects  // 堆上存活对象数量        ⚠️ 过多 → GC 压力大
ms.StackInuse   // 栈内存使用量
ms.Sys          // 进程总内存占用
```

---

**② GC 指标**

```go
ms.NumGC        // GC 总次数              ⚠️ 过于频繁 → 调大 GOGC
ms.PauseTotalNs // GC 总暂停时间
ms.PauseNs      // 最近 256 次 GC 暂停时间
ms.GCCPUFraction // GC 占 CPU 时间比例   ⚠️ > 5% 需要优化
ms.LastGC       // 上次 GC 时间戳
```

---

**③ Goroutine 指标**

```go
runtime.NumGoroutine() // 当前 goroutine 数量
                       // ⚠️ 持续增长不回落 → goroutine 泄漏
```

排查泄漏：

```bash
# 抓取两次快照对比
curl http://localhost:6060/debug/pprof/goroutine?debug=1
```

---

**④ 请求指标（业务层）**

```
QPS        每秒请求量           监控突增/突降
延迟       P50 / P95 / P99     ⚠️ P99 飙高 → 有慢请求
错误率     5xx / 超时比例       ⚠️ > 0.1% 需告警
```

---

**⑤ 系统资源指标**

```
CPU 使用率     ⚠️ 持续 > 80% → 扩容或优化热点
内存使用率     ⚠️ 持续增长 → 内存泄漏
FD 数量        ⚠️ 接近 ulimit → 连接未释放
线程数量       ⚠️ 过多 → 频繁阻塞系统调用
```

---

#### 六、指标采集完整示例

```go
func recordRuntimeMetrics() {
    ticker := time.NewTicker(10 * time.Second)
    for range ticker.C {
        var ms runtime.MemStats
        runtime.ReadMemStats(&ms)

        // 上报到 prometheus 或打印
        fmt.Printf(
            "goroutines=%d heap_alloc=%dMB heap_obj=%d gc_num=%d gc_cpu=%.2f%%\n",
            runtime.NumGoroutine(),
            ms.HeapAlloc/1024/1024,
            ms.HeapObjects,
            ms.NumGC,
            ms.GCCPUFraction*100,
        )
    }
}
```

---

#### 七、各指标异常对应排查方向

| 异常现象 | 可能原因 | 排查工具 |
|---|---|---|
| 内存持续增长 | 内存泄漏 / goroutine 泄漏 | heap pprof |
| CPU 飙高 | 热点函数 / GC 频繁 | CPU pprof 火焰图 |
| P99 延迟高 | GC STW / 锁竞争 / 慢查询 | trace / mutex pprof |
| Goroutine 暴涨 | goroutine 泄漏 | goroutine pprof |
| GC 频繁 | 对象分配过多 / GOGC 过小 | heap pprof + 逃逸分析 |
| 线程数暴涨 | 大量阻塞系统调用 | trace 工具 |

---

#### 一句话总结

> 开发调试用 **pprof 火焰图**定位热点，生产环境用 **Prometheus + Grafana** 监控四大黄金指标（QPS、延迟、错误率、饱和度），重点关注 **goroutine 数量、堆内存趋势、GC 频率和 P99 延迟**，任何持续单调增长的指标都是告警信号。

### 协程池的意义是什么
### 8、Go 当中同步锁有什么特点？作用是什么
    - 当一个 Goroutine（协程）获得了 Mutex 后，其他 Goroutine（协程）就只能乖乖的等待，除非该 Goroutine 释放了该 Mutex。RWMutex 在读锁占用的情况下， 会阻止写，但不阻止读 RWMutex。 在写锁占用情况下，会阻止任何其他Goroutine（无论读和写）进来，整个锁相当于由该 Goroutine 独占
    同步锁的作用是保证资源在使用时的独有性，不会因为并发而导致数据错乱， 保证系统的稳定性。
