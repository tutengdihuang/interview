## Golang 基础

### init() 函数
- init() 函数是什么时候执行的？
    - 参考答案init() 函数是 Go 程序初始化的一部分。Go 程序初始化先于 main 函数，由 runtime 初始化每个导入的包，初始化顺序不是按照从上到下的导入顺序，而是按照解析的依赖关系，没有依赖的包最先初始化。
    - 每个包首先初始化包作用域的常量和变量（常量优先于变量），然后执行包的 init() 函数。同一个包，甚至是同一个源文件可以有多个 init() 函数。init() 函数没有入参和返回值，不能被其他函数调用，同一个包内多个 init() 函数的执行顺序不作保证。
    - 一句话总结： import –> const –> var –> init() –> main()

### Channel
- channel的应用场景
    - 如果面试官问的比较笼统可以从一下几个方面回答
        - 是什么
        - 有什么特性
        - 怎么用


- go channel使用需要注意的地方

  ![image](https://user-images.githubusercontent.com/31843331/153333030-3ca372b8-53c8-41db-ba86-ac89b9de636d.png)


## goroutine与线程的主要区别

### 1. 轻量级 vs 重量级
- **Goroutine**：由 Go 运行时调度，创建和销毁的开销非常低，通常一个 Go 程序可以同时运行数以万计的 goroutine。
- **线程**：由操作系统调度，创建和销毁的开销较大，数量通常受限于系统资源。

### 2. 调度机制
- **Goroutine**：Go 运行时（runtime）实现了自己的调度器，将大量的 goroutine 映射到较少的 OS 线程上。这种“多对少”的调度方式使得并发处理更高效。
- **线程**：直接由操作系统进行管理和调度，通常为“一对一”的关系，即每个线程对应一个调度实体。

### 3. 内存占用与栈管理
- **Goroutine**：初始栈非常小（通常只有几 KB），且可以根据需要动态扩展或收缩，因此内存利用率更高。
- **线程**：一般为每个线程分配较大的固定栈空间（通常为几 MB），内存开销较大。

### 4. 通信方式
- **Goroutine**：推荐使用基于 channel 的通信方式来传递数据，这种方式鼓励“不要通过共享内存来通信，而是通过通信来共享内存”，降低了数据竞争的风险。
- **线程**：通常依赖共享内存加锁（mutex、信号量等）来进行线程间的数据交换和同步，编写和维护代码时更容易出错，如死锁和竞态条件。

### 5. 使用场景
- **Goroutine**：非常适合高并发场景，如网络服务、并行计算等，能够轻松启动大量并发任务。
- **线程**：适用于需要操作系统级别控制的任务，如底层系统编程、涉及复杂进程间通信的应用等。

通过上述几点，可以清楚地看出 goroutine 是为 Go 语言并发设计的轻量级执行单元，而线程则是操作系统提供的相对重量级的并发机制。两者在创建成本、调度方式、内存管理和通信模型上均有较大区别，这也是 Go 语言在高并发场景下表现优异的关键原因。



## 如何主动关闭goroutine

在 Go 语言中，并没有直接提供一个 API 来“强制杀死”或“关闭”一个 goroutine。相反，关闭 goroutine 的最佳实践是设计 goroutine 时使其能在接收到退出信号时自行退出，这通常通过以下几种方式实现：

### 1. 使用 Channel 通知退出

利用一个专门的退出信号 channel，当需要关闭 goroutine 时，向这个 channel 发送信号或关闭它，然后在 goroutine 内部通过 `select` 语句监听这个退出信号。例如：

```go
done := make(chan struct{})

go func() {
    for {
        select {
        case <-done:
            // 收到退出信号，进行必要的清理工作后退出
            return
        default:
            // 执行正常任务
        }
    }
}()

// 当需要退出时：
close(done)
```

### 2. 使用 Context 进行控制

Go 的 `context` 包提供了上下文管理，通过 `context.WithCancel` 创建一个可取消的上下文，并将其传递给 goroutine。goroutine 定期检查上下文是否已取消，从而决定是否退出：

```go
ctx, cancel := context.WithCancel(context.Background())

go func(ctx context.Context) {
    for {
        select {
        case <-ctx.Done():
            // 收到取消信号，退出 goroutine
            return
        default:
            // 执行正常任务
        }
    }
}(ctx)

// 当需要关闭 goroutine 时：
cancel()
```

### 小结

- **无法强制杀死**：Go 没有提供直接关闭 goroutine 的方法。
- **设计退出机制**：应在 goroutine 内部设计退出逻辑，定期检查退出信号（通过 channel 或 context）。
- **优雅退出**：通过通知机制使 goroutine 能够完成必要的清理工作后退出，从而保持程序的健壮性和可维护性。

这种设计方式不仅符合 Go 的设计理念（鼓励通过通信来同步而非共享内存），也能有效避免资源泄漏和竞态条件。


## 为什么goroutine的调度更高效

goroutine 调度更高效主要源于以下几个方面：

1. **轻量级设计**  
   - **小栈空间**：goroutine 初始分配的栈非常小（通常仅几 KB），且可以按需动态扩展。这与传统线程每个预留固定且较大的栈空间（通常几 MB）形成鲜明对比，从而显著降低内存开销。citeturn0search0

2. **用户态调度（M:N 调度）**  
   - **Go 运行时调度器**：Go 使用自己的调度器在 M 个 OS 线程上调度 N 个 goroutine。由于切换发生在用户态，避免了内核级别的上下文切换开销，使得调度更为高效。citeturn0search0

3. **高效的调度算法**  
   - **工作窃取（Work-Stealing）机制**：调度器采用工作窃取算法来平衡各个线程间的任务分布，确保每个线程都能高效利用 CPU，从而进一步提升并发性能。citeturn0search0

4. **降低同步成本**  
   - **基于 channel 的通信**：goroutine 鼓励通过通信而不是共享内存来实现数据交换，这种设计减少了锁等同步机制带来的额外开销，同时也降低了死锁和竞态条件的风险。citeturn0search0

总的来说，goroutine 通过轻量级设计、用户态调度以及高效的调度算法，实现了高并发下的低资源消耗和快速切换，使得其在实际应用中能更高效地处理大规模并发任务。

# 并发控制


## 什么是死锁？

死锁是并发编程中一种常见的问题，指两个或多个 goroutine（或线程）互相等待对方释放资源或发送信号，从而导致所有相关 goroutine 都陷入永久等待状态。Go 运行时如果检测到所有 goroutine 都被阻塞，就会触发运行时错误（如 “fatal error: all goroutines are asleep - deadlock!”），从而使程序崩溃。citeturn0search0

## Go 什么情况会死锁？

在 Go 中，死锁通常出现在以下几种情况：

1. **无缓冲 channel 的错误使用**  
   - 在无缓冲 channel 中，发送操作必须有对应的接收操作才能完成。如果你在主 goroutine 中发送数据，但没有启动任何 goroutine 来接收数据，则发送操作会一直阻塞，最终导致死锁。  
   - 例如：
     ```go
     func main() {
         ch := make(chan int)
         // 没有对应的接收者，这里的发送操作会一直阻塞
         ch <- 1  
     }
     ```
   
2. **循环等待**  
   - 当多个 goroutine 形成循环依赖关系时（例如 A 等待 B 的结果，而 B 又等待 A 的信号），就会出现循环等待，导致死锁。

3. **错误的锁定顺序**  
   - 如果多个 goroutine 同时持有对方需要的锁（如使用 mutex），并且获取锁的顺序不当，也容易出现死锁。  

## go怎么避免死锁问题？

避免死锁需要在设计并发程序时就考虑好通信和同步机制，常用的预防措施包括：

1. **确保匹配的发送和接收**  
   - 在使用无缓冲 channel 时，保证每个发送操作都有对应的接收操作；在使用 buffered channel 时，确保缓冲区容量足够或有及时的消费者避免写入阻塞。  

2. **使用 Context 或超时机制**  
   - 利用 `select` 语句结合 `time.After` 或 `context`，可以为 channel 操作设置超时，避免因一直等待而导致死锁。例如：
     ```go
     select {
     case ch <- 1:
         // 正常发送数据
     case <-time.After(time.Second):
         // 超时处理逻辑
     }
     ```
   
3. **合理使用 WaitGroup 和信号量**  
   - 使用 `sync.WaitGroup` 可以协调多个 goroutine 的启动与退出，确保所有 goroutine 都能有序结束，从而减少因等待而引起的死锁风险。

4. **降低锁依赖**  
   - 尽量使用 channel 进行数据传递，而不是共享内存。如果必须使用锁，保持锁的粒度小且获取顺序一致，避免多个锁相互依赖导致循环等待。

5. **代码审查与测试**  
   - 对并发代码进行严格的审查和测试，利用工具或日志检查是否存在长时间阻塞的情况，有助于及时发现并解决潜在死锁问题。  

通过以上措施，可以在设计和实现并发程序时尽量避免死锁的发生，从而提高程序的稳定性和健壮性。citeturn0search0


## go sync包有哪些方法以及具体作用
下面是调整后的内容，所有标题均使用三级标题（###），保证最大标题级别不超过三级：

---

### Go sync 包主要组件及方法

Go 的 sync 包提供了一系列用于并发编程的同步原语，帮助开发者在多 goroutine 之间安全地共享数据，下面依次介绍各个组件及其主要方法：

---

### 1. Mutex（互斥锁）

- **Lock()**  
  锁定互斥锁。当一个 goroutine 调用 Lock() 后，其它 goroutine 尝试调用 Lock() 会被阻塞，直到该锁被解锁。  
- **Unlock()**  
  解锁互斥锁，允许等待该锁的 goroutine 继续执行。

**作用**：保护共享资源，确保同一时间只有一个 goroutine 访问临界区，从而防止数据竞争和不一致问题。  
citeturn0search0

---

### 2. RWMutex（读写互斥锁）

- **RLock()**  
  获取读锁。多个 goroutine 可以同时获取读锁，以实现并发读操作。  
- **RUnlock()**  
  释放读锁。  
- **Lock()**  
  获取写锁。写锁在获取时会阻塞其他读写操作，确保独占访问。  
- **Unlock()**  
  释放写锁。

**作用**：适用于读多写少的场景，通过允许并发读取提高性能，同时保证写操作时数据的一致性。  
citeturn0search0

---

### 3. WaitGroup（等待组）

- **Add(delta int)**  
  增加或减少等待计数器的值，用于设定需要等待完成的 goroutine 数量。  
- **Done()**  
  完成一个 goroutine 的任务，相当于调用 Add(-1)。  
- **Wait()**  
  阻塞当前 goroutine，直到等待组的计数器归零。

**作用**：用于等待一组并发任务全部完成，常用于在主 goroutine 中等待子 goroutine 执行完毕。  
citeturn0search0

---

### 4. Cond（条件变量）

- **Wait()**  
  在条件变量上等待，调用后 goroutine 会阻塞并释放相关联的锁，直到收到 Signal 或 Broadcast 通知后重新获取锁继续执行。  
- **Signal()**  
  唤醒一个等待该条件变量的 goroutine。  
- **Broadcast()**  
  唤醒所有等待该条件变量的 goroutine。

**作用**：用于复杂的同步场景中，当某个条件满足时通知等待的 goroutine，常见于生产者-消费者模式中协调状态变化。  
citeturn0search0

---

### 5. Once（只执行一次）

- **Do(func())**  
  确保传入的函数只被执行一次，无论 Do 被调用多少次。这常用于单例模式或一次性初始化操作。

**作用**：保证某段代码只执行一次，避免重复初始化或资源重复分配问题。  
citeturn0search0

---

### 6. Pool（对象池）

- **Get() interface{}**  
  从对象池中获取一个对象；如果池为空，则根据预设的 New 方法（如果有）创建一个新的对象。  
- **Put(x interface{})**  
  将对象放回池中，以便后续复用。

**作用**：减少对象频繁创建和销毁的开销，通过复用对象降低垃圾回收压力，适用于需要大量临时对象的场景。  
citeturn0search0

---

### 7. Map（并发安全的 map）

- **Load(key interface{}) (value interface{}, ok bool)**  
  从 Map 中获取指定 key 对应的 value；如果 key 不存在，则 ok 为 false。  
- **Store(key, value interface{})**  
  将 key-value 存入 Map 中。  
- **LoadOrStore(key, value interface{}) (actual interface{}, loaded bool)**  
  如果 key 存在，则返回现有的 value；如果不存在，则存入新的值并返回。  
- **Delete(key interface{})**  
  删除指定 key 对应的元素。  
- **Range(func(key, value interface{}) bool)**  
  遍历 Map 中的所有键值对，回调函数返回 false 则停止遍历。

**作用**：提供并发安全的 map 操作，避免在使用标准 map 时需要额外加锁，适合多 goroutine 同时访问的场景。  
citeturn0search0

---

### 总结

Go 的 sync 包为并发编程提供了多种同步原语，从互斥锁、读写锁到等待组、条件变量、只执行一次、对象池以及并发安全的 map，这些工具使得开发者能够更轻松、安全地管理并发访问，确保数据一致性和程序的正确性。合理选择和使用这些原语是编写高性能并发程序的关键。

## go context包的作用
### go context包的作用

Go 的 context 包用于在多个 goroutine 之间传递取消信号、截止时间和请求范围内的元数据，从而管理并协调并发操作。它解决了在分布式系统或并发任务中共享状态、传递取消信号和设置超时的问题。

### 主要功能

### 1. 控制取消和超时
- **创建可取消的 context**：通过 `context.WithCancel`、`context.WithTimeout` 或 `context.WithDeadline` 创建一个带有取消或超时功能的 context，允许在特定条件下自动终止相关操作。
- **及时释放资源**：当操作超时或主动取消时，相关的 goroutine 能够收到取消信号并优雅退出，从而避免资源泄露。

### 2. 传递请求范围的数据
- **共享元数据**：context 可用于传递请求相关的数据，如请求ID、认证信息、用户信息等，在整个调用链中共享这些信息便于日志记录、监控和错误追踪。
- **避免全局变量**：通过 context 传递数据，有助于保持代码整洁，避免使用全局变量或显式参数传递。

### 3. 协调并发任务
- **统一管理任务生命周期**：在复杂的并发操作中，context 能够将多个 goroutine 关联起来，当主任务取消时，所有子任务也能及时感知并退出。
- **提升系统健壮性**：通过统一的取消和超时控制，可以有效防止因单个任务阻塞或失控而影响整个系统的稳定性。

### 总结

Go 的 context 包是编写健壮并发程序的重要工具，通过传递取消信号、设置超时和共享请求数据，实现了对并发任务生命周期的统一管理，广泛应用于服务器和分布式系统开发中。

## 内存和系统
- 字节对齐和大小端序
    - [解答](https://www.yuque.com/docs/share/2f155ad2-4b48-415a-acf6-5ca11571d3db)

## golang gc 操作系统不真实释放内存怎么办

在 Go 语言中，垃圾回收（GC）机制会自动管理内存的分配和释放，但有时您可能会发现，即使 GC 已经回收了不再使用的内存，操作系统报告的内存占用（例如 RES 值）并未立即减少。这主要与 Go 运行时如何将未使用的内存归还给操作系统的策略有关。

### Go 的内存释放策略

Go 使用 `madvise` 系统调用来提示操作系统回收物理内存，主要有两种策略：

1. **MADV_DONTNEED**：立即将未使用的内存归还给操作系统。如果再次访问这些内存区域，会触发页面错误，需要重新分配物理页。使用此策略，程序的 RES 值会迅速下降。

2. **MADV_FREE**：标记内存为可回收，但不立即归还操作系统。当操作系统内存紧张时，才会回收这些内存。如果在此之前再次访问这些内存区域，不会触发页面错误。使用此策略，程序的 RES 值可能不会立即减少。

在 Go 1.11 及之前的版本，默认使用 `MADV_DONTNEED`；在 Go 1.12 至 Go 1.15 版本，默认使用 `MADV_FREE`；从 Go 1.16 开始，默认又切换回 `MADV_DONTNEED`。

### 手动释放内存

如果您希望更主动地将未使用的内存归还给操作系统，可以采取以下措施：

1. **设置环境变量**：通过设置环境变量 `GODEBUG=madvdontneed=1`，强制 Go 运行时使用 `MADV_DONTNEED` 策略。这将使内存更快地归还给操作系统。

   ```bash
   export GODEBUG=madvdontneed=1
   ```


2. **调用 `debug.FreeOSMemory()`**：手动调用此函数，提示 Go 运行时尝试将空闲内存归还给操作系统。

   ```go
   import "runtime/debug"

   func main() {
       // 业务逻辑
       debug.FreeOSMemory()
   }
   ```


需要注意的是，频繁调用 `debug.FreeOSMemory()` 可能会影响程序性能，应根据实际需求谨慎使用。

### 总结

Go 的内存释放策略会影响操作系统报告的内存占用情况。如果发现 GC 回收后内存未及时归还给操作系统，可通过设置环境变量或手动调用内存释放函数来优化。然而，这些操作可能带来额外的性能开销，建议在了解其影响的前提下使用。 


## context原理

在 Go 语言中，`context` 包用于在多个 goroutine 之间传递上下文信息，如取消信号、超时时间和请求范围内的数据。它的设计旨在简化并发操作的管理，确保在处理单个请求时，所有相关的 goroutine 能够协同工作，并在需要时及时退出。

**主要原理：**

1. **上下文传播：** `context` 在 goroutine 之间传递，使得各个子 goroutine 能够共享相同的上下文信息。这种机制确保了在处理一个请求的过程中，所有相关的 goroutine 都能访问到必要的元数据，如用户身份、认证令牌等。 citeturn0search1

2. **取消信号：** 通过 `context.WithCancel`、`context.WithTimeout` 或 `context.WithDeadline` 等函数，可以创建可取消的上下文。当上层操作需要取消时，相关的 goroutine 会收到取消信号，从而及时终止操作，释放资源。 citeturn0search1

3. **超时控制：** `context` 允许为操作设置超时时间或截止时间，确保长时间运行的操作不会无限制地占用资源。当超过设定的时间后，相关的操作会自动取消，防止系统资源被耗尽。 citeturn0search2

4. **数据传递：** 通过 `context.WithValue`，可以在上下文中存储键值对数据，供多个 goroutine 使用。这种方式避免了使用全局变量，确保数据在请求范围内传递，提升了代码的可维护性。 citeturn0search2

**实现机制：**

`context` 包定义了 `Context` 接口，包括 `Deadline`、`Done`、`Err` 和 `Value` 等方法。具体实现有四种类型：

- **`emptyCtx`：** 永不取消、无截止时间、无携带值的上下文，通常用于根上下文。

- **`cancelCtx`：** 可取消的上下文，包含一个 `done` 通道，用于通知取消事件。

- **`timerCtx`：** 带有超时或截止时间的上下文，内部维护一个定时器，到期时自动发送取消信号。

- **`valueCtx`：** 携带键值对数据的上下文，用于在 goroutine 间传递请求范围内的数据。

通过这些机制，`context` 在 Go 语言的并发编程中扮演了关键角色，确保了 goroutine 间的高效协作和资源管理。 


## single—flight实现原理

在 Go 语言的并发编程中，`singleflight` 是一个用于抑制重复函数调用的工具，确保针对相同的键（key），在同一时间内，函数只会被执行一次，多个请求共享同一结果。 citeturn0search5

**实现原理：**

- **核心结构：** `singleflight` 包的核心是 `Group` 结构体，它维护了一个映射（map），用于记录正在进行的请求。

- **请求合并：** 当多个 goroutine 同时对同一个键发起请求时，`singleflight` 会将这些请求合并为一个，只有第一个请求会真正执行函数，其余请求会等待第一个请求的结果。 citeturn0search5

- **同步机制：** `singleflight` 利用 `sync.Mutex` 和 `sync.WaitGroup` 来确保并发安全和同步执行。

**工作流程：**

1. **发起请求：** 当一个新的请求到来时，`singleflight` 会检查是否已有相同键的请求正在进行。

2. **合并请求：** 如果存在相同的请求，新的请求会被阻塞，等待第一个请求完成。

3. **执行函数：** 如果没有相同的请求，`singleflight` 会记录该请求，并执行对应的函数。

4. **返回结果：** 函数执行完成后，所有等待的请求都会收到相同的结果。

**应用场景：**

- **缓存击穿：** 在缓存失效时，防止大量并发请求直接打到数据库，`singleflight` 可以将这些请求合并为一个，减少数据库压力。 citeturn0search2

- **防止重复操作：** 在高并发环境下，避免对同一资源的重复操作，确保数据一致性。

通过 `singleflight`，开发者可以有效地控制并发请求，减少重复操作，提高系统的性能和稳定性。 

## 延迟队列
    - 参看zinx时间轮
    - 使用 Redis 的有序集合（Sorted Set）
    - 使用最小堆（Min-Heap

##  go-micro 的模块分为哪些
    - 组件
        - Go Micro
        - API
        - Sidecar
        - Web
        - Cli
        - Bot

## gRPC
- https://www.processon.com/view/link/6438b13924c38d10f2e15b83?cid=619467f5e401fd59f24bdd2d
### 基础概念
- grpc通信类型
    - 四种
    - 一元 流式，客户端和服务器分别凑合就可以，一共四种
## grpc为什么要使用http2.0当传输层协议？

gRPC 选择 HTTP/2 作为传输层协议，主要基于以下考虑：

1. **多路复用：** HTTP/2 支持在单一 TCP 连接上并发多个请求和响应，避免了传统 HTTP/1.x 中的队头阻塞问题，提高了传输效率。

2. **双向流：** HTTP/2 的双向流特性使得 gRPC 能够轻松实现客户端和服务器之间的双向通信，支持流式传输数据，满足复杂的通信需求。

3. **头部压缩：** HTTP/2 使用 HPACK 算法对头部信息进行压缩，减少了带宽消耗，提高了传输效率。

4. **标准化协议：** 采用公开标准的 HTTP/2 协议，使得 gRPC 更易于在不同平台和语言之间实现，增强了互操作性。

5. **安全性：** HTTP/2 天生支持 TLS，加密传输数据，确保通信安全。

综上所述，HTTP/2 的特性与 gRPC 的需求高度契合，使其成为 gRPC 的理想传输层协议。 

## grpc如何实现负载均衡

在 gRPC 中，负载均衡主要通过客户端实现，即由客户端根据特定策略选择合适的服务器实例进行请求。以下是实现 gRPC 负载均衡的常见方法：

### 1. 集成服务发现机制

客户端需要获取可用服务实例的列表，这通常通过服务发现机制实现。常用的服务发现工具包括 etcd、Consul 等。服务提供者在启动时将自身信息注册到服务发现组件，客户端则从中获取最新的服务实例列表。

### 2. 使用客户端负载均衡策略

gRPC 提供了多种客户端负载均衡策略，最常用的是轮询（Round Robin）策略。在此策略下，客户端按照顺序循环选择服务实例，以均衡负载。要使用轮询策略，需要在客户端连接时指定：

```go
import (
    "google.golang.org/grpc"
    "google.golang.org/grpc/balancer/roundrobin"
)

conn, err := grpc.Dial(
    "your_service_address",
    grpc.WithInsecure(),
    grpc.WithBalancerName(roundrobin.Name),
)
```


上述代码中，`grpc.WithBalancerName(roundrobin.Name)` 指定了使用轮询负载均衡策略。

### 3. 自定义负载均衡策略

如果内置的负载均衡策略不能满足需求，gRPC 允许开发者自定义策略。这需要实现 `balancer.Builder` 和 `balancer.Balancer` 接口，并在客户端连接时使用自定义的负载均衡器。

### 4. 外部负载均衡服务

除了客户端负载均衡，gRPC 也支持通过外部负载均衡器（如 Nginx、HAProxy）进行请求分发。在这种模式下，客户端将请求发送到负载均衡器，由其根据策略将请求转发到后端服务实例。

需要注意的是，选择何种负载均衡方式应根据具体的应用场景和需求进行权衡。 

##  两个客户端A,B A读取数据，B修改还未提交，A再次读取，A两次读取的消息是一致的吗？

在数据库系统中，事务的隔离级别决定了一个事务在多大程度上与其他事务隔离，以确保数据的一致性和完整性。常见的隔离级别包括：

1. **读未提交（Read Uncommitted）：** 事务可以读取到其他未提交事务的修改，可能导致脏读问题。

2. **读已提交（Read Committed）：** 事务只能读取到其他已提交事务的修改，避免了脏读，但可能出现不可重复读的问题。

3. **可重复读（Repeatable Read）：** 在同一个事务中，针对同一数据的多次读取结果是一致的，避免了脏读和不可重复读，但可能出现幻读。

4. **串行化（Serializable）：** 最高的隔离级别，所有事务顺序执行，避免了上述所有问题，但并发性能最低。

根据您的描述，客户端 A 读取数据，客户端 B 修改数据但未提交，然后 A 再次读取数据。在这种情况下，A 两次读取的数据是否一致，取决于数据库的隔离级别：

- **读未提交：** A 可能在第二次读取时看到 B 的未提交修改，导致两次读取结果不一致，这就是脏读现象。

- **读已提交及以上：** A 无法读取到 B 的未提交修改，因此两次读取结果一致。

因此，为避免脏读问题，建议将数据库的隔离级别设置为“读已提交”或更高。 


## grpc 版本字段增加，服务端客户端如何升级？先后顺序是什么

### gRPC 服务端与客户端版本升级策略总结


#### **一、核心原则**
1. **向后兼容优先**  
   - **字段添加**：新增字段时，避免修改现有字段的编号、类型或语义。
   - **字段废弃**：废弃字段需注释并保留编号，避免重用。
   - **可选字段**：使用 `optional` 或 `repeated` 字段，避免强制字段（`required`）。

2. **版本协商机制**  
   - 通过 **元数据（Metadata）** 传递版本号（如 `x - grpc - version`）。
   - 服务端根据版本号处理不同逻辑，支持新旧版本共存。


#### **二、升级步骤**
1. **服务端升级**  
   - **新增版本字段**：在 Protobuf 中添加新字段（如 `optional int32 version = 7;`）。
   - **兼容旧逻辑**：确保服务端能处理不含新版本字段的请求。
   - **发布新版本**：部署支持新字段的服务端，保持与旧客户端兼容。

2. **客户端升级**  
   - **添加新字段**：客户端代码中新增字段的序列化/反序列化逻辑。
   - **条件处理**：根据服务端返回的版本信息，决定是否使用新字段。
   - **逐步替换**：分批升级客户端，确保所有客户端支持新字段后，服务端可移除兼容逻辑。


#### **三、关键技术点**
1. **Protobuf 兼容性**  
   - **字段编号**：新增字段使用未使用的编号（如 `reserved` 保留范围）。
   - **WireFormat 特性**：未知字段自动忽略，旧版本服务端可透明处理新字段。

2. **元数据传递版本**  
   ```go
   // 客户端发送版本元数据
   md := metadata.New(map[string]string{"x-grpc-version": "2.0"})
   ctx := metadata.NewOutgoingContext(context.Background(), md)
   ```

3. **服务端多版本支持**  
   ```go
   func (s *Server) Handle(ctx context.Context, req *Request) (*Response, error) {
       md, _ := metadata.FromIncomingContext(ctx)
       version := md.Get("x-grpc-version")[0]
       
       if version == "2.0" {
           // 处理新版本逻辑
       } else {
           // 兼容旧版本逻辑
       }
       return &Response{}, nil
   }
   ```


#### **四、注意事项**
1. **避免强制升级**  
   - 服务端需长期支持旧版本，直到所有客户端完成升级。
   - 通过监控和日志追踪旧版本调用，逐步淘汰。

2. **版本回滚**  
   - 确保新版本服务端与旧客户端兼容，回滚时无需修改客户端。

3. **工具链支持**  
   - 使用 `protoc` 生成兼容新旧版本的代码。
   - 利用 gRPC 拦截器统一处理版本逻辑。


#### **五、示例流程**
1. **服务端**  
   - 新增字段 `version` 并发布 `v2.0` 服务端。
   - 处理请求时，根据 `version` 字段选择逻辑分支。

2. **客户端**  
   - 升级客户端代码，添加 `version` 字段。
   - 发送请求时携带 `version: 2.0`。
   - 服务端收到后按新版本逻辑处理。


#### **总结**
- **升级顺序**：优先升级服务端，确保兼容性；再逐步升级客户端。
- **核心策略**：通过元数据协商版本，Protobuf 字段设计保持向后兼容，服务端长期支持旧版本。
- **工具与监控**：利用拦截器、元数据和日志，实现平滑过渡与版本治理。

## 系统设计如何设计一个10亿访问量的系统


设计一个支持10亿级访问量的高并发系统，需要从架构分层、性能优化、分布式设计、弹性扩展、可靠性保障等多个维度综合考虑。以下是核心设计要点及技术方案：


### **一、分层架构设计**
1. **前端层（接入层）**  
   - **负载均衡**：四层（LVS）+ 七层（Nginx/Apache）负载均衡组合，支持百万级QPS。  
   - **CDN加速**：静态资源（图片、JS、CSS）通过CDN节点缓存，减少源站压力。  
   - **边缘计算**：边缘节点（如Cloudflare Workers）处理部分请求，降低延迟。  

2. **应用层（服务层）**  
   - **无状态服务**：业务逻辑模块化，避免本地状态（如内存缓存），支持水平扩展。  
   - **微服务拆分**：按业务领域拆分（用户、订单、支付等），独立部署，降低耦合。  
   - **服务网格（Service Mesh）**：Istio/Kuma管理服务间通信，实现流量治理（限流、熔断、重试）。  

3. **数据层**  
   - **缓存层**：Redis集群（分片+哨兵/Cluster）缓存热点数据，命中率>95%。  
   - **数据库**：  
     - 关系型数据库（MySQL）：分库分表（ShardingSphere/TDSQL），读写分离（主从复制）。  
     - NoSQL：MongoDB（文档存储）、Cassandra（宽列存储）处理海量非结构化数据。  
   - **搜索引擎**：Elasticsearch/OpenSearch支持亿级数据检索（如商品搜索）。  


### **二、性能优化与流量管理**
1. **缓存策略**  
   - 多级缓存：本地缓存（Caffeine/Guava）+ 分布式缓存（Redis）。  
   - 缓存失效：LRU策略，热点数据永不过期（定期刷新）。  
   - 防穿透：布隆过滤器（Bloom Filter）过滤无效Key。  

2. **异步与削峰填谷**  
   - 消息队列：Kafka/RocketMQ处理异步任务（如订单异步扣款、日志异步写入），削峰能力达10万TPS。  
   - 延迟队列：RabbitMQ/Redis实现延迟任务（如订单超时取消）。  

3. **限流与熔断**  
   - 限流：Sentinel（QPS限流）、漏桶算法（固定速率处理请求）。  
   - 熔断：Hystrix/Resilience4J，服务不可用时快速失败。  


### **三、分布式架构设计**
1. **服务发现与注册**  
   - Consul/Etcd实现服务注册中心，支持动态扩缩容。  

2. **分布式事务**  
   - 最终一致性：TCC（Try-Confirm-Cancel）、Saga模式（补偿事务）。  
   - 消息事务：RocketMQ事务消息保证最终一致性。  

3. **分布式ID**  
   - Snowflake算法（Twitter ID生成器）或美团Leaf，支持每秒百万级ID生成。  


### **四、弹性扩展与高可用**
1. **自动伸缩（Auto Scaling）**  
   - K8s集群：根据CPU/内存使用率自动扩缩容（HPA），分钟级完成实例部署。  
   - 无状态服务：Docker容器化，快速复制实例（如每个实例支持1万QPS，10万实例支持10亿QPS）。  

2. **异地多活（Multi-AZ）**  
   - 跨地域部署（如华北、华东、华南），流量就近接入，容灾备份。  
   - 数据同步：数据库CDC（Change Data Capture）实现多活数据异步同步。  


### **五、可靠性与监控**
1. **容灾备份**  
   - 数据库备份：全量备份（每日）+ 增量备份（Binlog，实时）。  
   - 异地灾备：数据定期同步到灾备中心（如AWS S3/Azure Blob）。  

2. **监控与日志**  
   - 监控：Prometheus+Grafana（CPU、内存、QPS、RT），APM工具（New Relic/Dynatrace）。  
   - 日志：ELK（Elasticsearch+Logstash+Kibana）聚合分析，链路追踪（Jaeger/SkyWalking）。  

3. **故障演练**  
   - Chaos Engineering：模拟服务宕机、网络分区，验证系统容错能力（如Netflix Chaos Monkey）。  


### **六、典型技术栈选型**
| 层级         | 技术方案                                                                 |
|--------------|--------------------------------------------------------------------------|
| 接入层       | Nginx+LVS+CDN（阿里云CDN/Cloudflare）                                   |
| 应用层       | Spring Cloud Alibaba/Kubernetes（微服务），Go/Python（高性能服务）      |
| 缓存层       | Redis Cluster（分片+哨兵），Caffeine（本地缓存）                         |
| 数据库       | MySQL（分库分表）+ MongoDB（海量数据）+ TiDB（HTAP场景）                |
| 消息队列     | Kafka（高吞吐）+ RabbitMQ（可靠消息）                                   |
| 服务治理     | Sentinel（限流）+ Dubbo（服务治理）+ Istio（服务网格）                  |
| 监控日志     | Prometheus+Grafana+ELK+SkyWalking                                       |


### **七、成本优化**
- **混合云部署**：核心服务（数据库）自建IDC，非核心（缓存、消息队列）使用云服务（AWS/Azure/阿里云）。  
- **Serverless**：部分低流量服务使用函数计算（如AWS Lambda/腾讯云SCF），按需付费。  
- **资源复用**：容器化共享资源（K8s Node池），避免物理机浪费。  


### **八、案例参考**
- **电商秒杀**：预加载库存到Redis，异步扣减库存，MQ削峰（如淘宝双11架构）。  
- **社交平台**：Feed流生成异步化（推拉结合模式），冷热数据分离（热数据Redis，冷数据HDFS）。  


### **总结设计步骤**
1. **流量评估**：计算峰值QPS（如10亿日活 × 0.1%峰值系数 ÷ 86400秒 ≈ 1157 QPS，需预留3倍冗余）。  
2. **分层设计**：从接入层到数据层逐层解耦，每一层独立扩展。  
3. **核心优化**：缓存命中率>90%，异步处理占比>80%，数据库QPS<1万（通过缓存和异步降低）。  
4. **容灾设计**：N+1备份，秒级故障切换，异地多活覆盖99.99%可用性。  

通过以上方案，系统可支撑10亿级访问量，同时保证高可用性（99.99%）、低延迟（P99<500ms）和成本可控。实际落地时需结合业务场景（如读多写少、实时性要求）调整技术选型，通过压测（如JMeter+流量染色）验证设计有效性。



## 微服务go-micro 的优缺点（微服务的优缺点）
    - 优点：逻辑清晰，简化部署，可扩展，灵活组合，技术异构，高可靠
    - 缺点：复杂度高，运维复杂，影响性能


## jwt原理
你的内容已经符合最多 **三级标题（###）** 的要求。如果你希望对格式做进一步优化，以下是一个确保所有标题最多为 **三级** 的版本：  

---

### **JWT（JSON Web Token）简介**  

JWT（JSON Web Token）是一种用于 **身份验证** 和 **信息安全传输** 的开放标准（RFC 7519）。它是一个紧凑的、自包含的令牌，通常用于在不同系统之间安全地传递信息。JWT 主要用于 **用户认证** 和 **授权**，并且广泛应用于微服务架构、单点登录（SSO）等场景。  

### **JWT 的结构**  
JWT 由 **三部分** 组成，每部分使用 `.` 号分隔：  
```plaintext
header.payload.signature
```  
示例：  
```plaintext
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VySWQiOjEyMywiZXhwIjoxNzAwMDAwMDAwfQ.J1K8QWzE5H6V5yhtLDXihJ4P2YhrjxC8QuXJf2LwD3s
```  
#### **1. Header（头部）**  
定义 JWT 的类型和签名算法（如 `HS256`）。  

#### **2. Payload（载荷）**  
包含用户信息（如 `userId`）、声明（Claims）以及过期时间（`exp`）。  

#### **3. Signature（签名）**  
使用 Header 指定的算法（如 HMAC SHA256）对前两部分签名，以确保数据未被篡改。  

### **JWT 的工作原理**  
1. **用户登录**：用户输入账号密码，服务器验证后生成 JWT，并返回给客户端。  
2. **客户端存储 JWT**：通常存储在 `localStorage` 或 `cookie` 中。  
3. **请求携带 JWT**：客户端在请求 API 时，将 JWT 放入 `Authorization` 头中：  
   ```plaintext
   Authorization: Bearer <JWT_TOKEN>
   ```
4. **服务器验证 JWT**：  
   - 服务器检查 JWT 是否有效（是否被篡改、是否过期）。  
   - 如果 JWT 有效，解析用户信息并处理请求。  
   - 如果 JWT 无效或过期，返回 `401 Unauthorized`。  

### **JWT 的优缺点**  
#### **优点**  
- **无状态（Stateless）**：不需要存储会话信息，适合分布式系统。  
- **安全性高**：JWT 通过签名保证数据完整性，防止篡改。  
- **支持跨平台**：基于 JSON 格式，适用于各种编程语言。  
- **高效**：结构紧凑，传输和解析速度快。  

#### **缺点**  
- **无法撤销**：JWT 生成后无法轻易撤销，需要额外的黑名单机制。  
- **占用空间**：比普通 Session ID 长，占用 HTTP 传输带宽。  
- **安全风险**：  
  - 如果存储在 `localStorage`，可能被 XSS 攻击窃取。  
  - 如果不加密 `payload`，敏感信息可能被窥探。  

### **适用场景**  
- **用户认证**（如 OAuth 2.0 授权）  
- **API 访问控制**（微服务之间的身份认证）  
- **单点登录（SSO）**  
- **无状态 Web 应用**  

### **总结**  
JWT 是一种轻量级、无状态的身份认证方案，适用于分布式系统。它通过签名保证数据完整性，但需要注意安全性，避免信息泄露和 XSS 攻击。  

---

这个版本已经保证所有标题的最大层级为 **三级（###）**，让内容更加清晰易读。这样符合 Markdown 规范，同时也更适合阅读和使用。


## 并发编程
- emao<-a<-b<-c, abc 任何一个出错，abc都退出，最后emo退出

##  channel关闭需要注意什么事情？


### **1. 只能由发送方关闭 channel**  
Go 语言的规范规定 **只有发送数据的一方** 应该关闭 `channel`，否则可能导致 `panic`。如果接收方尝试关闭 `channel`，会发生错误：  
```go
panic: close of closed channel
```  
#### ✅ **正确示例**  
```go
func sendData(ch chan int) {
    defer close(ch) // 只有发送方关闭 channel
    for i := 0; i < 5; i++ {
        ch <- i
    }
}

func main() {
    ch := make(chan int)
    go sendData(ch)

    for v := range ch { // 正确：range 能检测到 channel 关闭
        fmt.Println(v)
    }
}
```  
#### ❌ **错误示例**  
```go
func main() {
    ch := make(chan int)
    go func() {
        ch <- 10
    }()
    close(ch) // 错误：接收方不应该关闭 channel
}
```

---

### **2. 读取已关闭的 channel**  
- **关闭的 `channel` 仍然可以被读取**，但不会再有新数据，读取会返回 `channel` 类型的**零值**。  
- `range` 可以用来监听 `channel`，当 `channel` 关闭时会自动退出循环。  
```go
func main() {
    ch := make(chan int, 2)
    ch <- 1
    ch <- 2
    close(ch)

    fmt.Println(<-ch) // 1
    fmt.Println(<-ch) // 2
    fmt.Println(<-ch) // 0（int 的零值），不会 panic
}
```

---

### **3. 判断 `channel` 是否关闭**  
如果尝试从已关闭的 `channel` 读取数据，Go 语言会返回零值，但不会报错。因此，通常会使用 **`ok` 判断** 是否还可以读取：  
```go
func main() {
    ch := make(chan int, 2)
    ch <- 1
    close(ch)

    v, ok := <-ch
    fmt.Println(v, ok) // 1 true（channel 未关闭时读取）
    
    v, ok = <-ch
    fmt.Println(v, ok) // 0 false（channel 关闭且无数据）
}
```

---

### **4. 多个 Goroutine 访问 channel**  
如果多个 Goroutine 访问同一个 `channel`，需要小心 **并发问题**：  
- **多个发送者：**可以使用 `sync.WaitGroup` 确保所有发送者完成后再关闭 `channel`。  
- **多个接收者：**可以安全读取 `channel`，但必须确保 `channel` 关闭后不会再有数据发送。  

#### ✅ **正确示例（多个 Goroutine 发送数据，确保 `channel` 只关闭一次）**  
```go
func main() {
    ch := make(chan int)
    var wg sync.WaitGroup

    for i := 0; i < 3; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            ch <- id
        }(i)
    }

    go func() {
        wg.Wait()
        close(ch) // 确保 `channel` 只关闭一次
    }()

    for v := range ch {
        fmt.Println(v)
    }
}
```  
#### ❌ **错误示例（多个 Goroutine 可能导致 `channel` 关闭多次，引发 panic）**  
```go
func main() {
    ch := make(chan int)
    for i := 0; i < 3; i++ {
        go func() {
            ch <- 10
            close(ch) // 错误：可能多个 Goroutine 关闭同一个 `channel`
        }()
    }
}
```  
**修正方法：只有一个 Goroutine 负责关闭 `channel`。**

---

### **5. 不要向已关闭的 channel 发送数据**  
向已关闭的 `channel` 发送数据会触发 `panic`：  
```go
func main() {
    ch := make(chan int)
    close(ch)
    ch <- 10 // panic: send on closed channel
}
```

---

### **6. 使用 `select` 避免阻塞**  
在 `channel` 关闭后，向其继续发送数据会导致 `panic`，但接收数据不会，因此 `select` 可以用来安全处理：  
```go
func main() {
    ch := make(chan int, 1)
    done := make(chan struct{})

    go func() {
        for {
            select {
            case v, ok := <-ch:
                if !ok {
                    fmt.Println("Channel closed, exiting...")
                    return
                }
                fmt.Println("Received:", v)
            }
        }
    }()

    ch <- 1
    close(ch)

    <-done // 保持主协程运行
}
```

---

### **总结**
| 事项 | 说明 |
|------|------|
| **谁关闭 `channel`** | **发送方** 关闭 `channel`，接收方不能关闭 |
| **读取已关闭 `channel`** | 继续读取不会 panic，但会返回零值 |
| **判断 `channel` 是否关闭** | 使用 `<-ch, ok` 判断 |
| **多个 Goroutine 访问 `channel`** | 确保 `channel` 只被关闭一次 |
| **向已关闭 `channel` 发送数据** | 会导致 `panic` |
| **使用 `select` 监听 `channel`** | 防止阻塞、提高安全性 |

通过正确地管理 `channel` 的生命周期，可以避免 `panic` 和数据不一致的问题。



## 类型系统
- go中哪些是值类型，哪些是引用类型
    - 引用类型：指针，map，slice，channel，方法与函数
    - 值类型：int系列、float系列、bool、string、数组和结构体

- 值传递和引用传递的区别
    - golang中只有值传递
- 切片传递过去，如果被调用函数append()，原来的切片会不会变化
    - append会修改slice所使用的底层数组，如果数组的不需要扩容会影响原来的切片；如果扩容则会引用新的数组，不会影响原切片；

## Map相关
- 如何确认二个map是否相等
    - reflect.DeepEqual(c1, c2)，可以是map，slice，struct


## mutex的原理
    - [refer](https://www.processon.com/view/link/6078e4416376891132d67bcf)

## 设计模式

- 说说常用的设计模式

## 切片
- https://www.processon.com/view/link/67dfe46a94f7ab2d0f7bfaa2?cid=67c57fc2ed8a6e36862f698d


## 代码题

### 输出结果分析
```go
type query func(string) string

func exec(name string, vs ...query) string {
 ch := make(chan string)
 fn := func(i int) {
  ch <- vs[i](name)
 }
 for i, _ := range vs {
  go fn(i)
 }
 return <-ch
}

func main() {
 ret := exec("111", func(n string) string {
  return n + "func1"
 }, func(n string) string {
  return n + "func2"
 }, func(n string) string {
  return n + "func3"
 }, func(n string) string {
  return n + "func4"
 })
 fmt.Println(ret)
}
```
- 解答
输出 111funcn   n可能是1 2 3 4

## 实现题
- 给出方法定义
  - 实现 errgroup.Group
  - 实现 singleflight.Group

- [腾讯］使用golang实现一个端口监听的程序。

## 内存管理
- 内存对齐（from 不是山谷）
  - [链接答案](https://www.yuque.com/docs/share/2f155ad2-4b48-415a-acf6-5ca11571d3db)
  - [阅读2](https://mp.weixin.qq.com/s/H3399AYE1MjaDRSllhaPrw)

## Select机制
- 说说go语言的select机制？
  - (1)、select机制用来处理异步IO问题
  - (2)、select机制最大的一条限制就是每个case语句里必须是一个IO操作
  - (3)、golang在语言级别支持select关键字

## Channel
- channel阻塞和非阻塞内部实现
- Channel 的 ring buffer 实现
    - channel 中使用了 ring buffer（环形缓冲区) 来缓存写入的数据。ring buffer 有很多好处，而且非常适合用来实现 FIFO 式的固定长度队列。在 channel 中，ring buffer 的实现如下：
- Channel可以嵌套使用吗？即往channel里发送一个channel
    - 可以

## 语言特性
- 与其他语言相比，使用GO 有什么好处？
- GO 支持什么形式的类型转换？将整数转换为浮点数
- 什么是GOROUTINE？你如何停止它？
- 如何在运行时检查变量类型？
- GO 两个接口之间可以存在什么关系？
- GO 当中同步锁有什么特点？作用是什么
- GO 语言中CAP 函数可以作用于那些内容？
- GO CONVEY 是什么？一般用来做什么？
- GO 语言中 MAKE 的作用是什么？
- PRINTF(),SPRINTF(),FPRINTF() 都是格式化输出，有什么不同？
- GO 语言当中值传递和地址传递（引用传递）如何运用？有什么区别？举例说明
- GO 语言当中数组和切片在传递的时候的区别是什么？
- GO 语言是如何实现切片扩容的？
- DEFER 的作用和特点是什么？
- SLICE 的底层实现
- GOLANG SLICE 的扩容机制，有什么注意点？
- GOLANG 的参数传递、引用类型
- GOLANG MAP 如何扩容，查找
- golang map原理， map 怎么解决冲突
- map为什么要设计溢出桶
- map是线程安全的吗
- map并发写或同时读写为什么要panic，如果不panic会有什么问题，从map底层设计和结构说一下




## 代码输出题
下边的程序输出什么
```go
const (
x = iota
_
y
z = "zz"
k
p = iota
)

func main()  {
fmt.Println(x,y,z,k,p)
}

输出：


0 2 zz zz 5

```

## 其他问题

### slice和数组的区别

#### 数组与切片核心区别

| 特性 | 数组(Array) | 切片(Slice) |
| :--- | :--- | :--- |
| 长度 | 固定，声明时指定，不可变 | 动态，可自动扩容 |
| 类型 | `[N]T`，长度是类型一部分 | `[]T`，长度不属于类型 |
| 内存结构 | 连续内存，直接存元素 | 结构体：指针、len、cap |
| 传递方式 | 值传递，拷贝全部元素 | 传结构体，共享底层数组 |
| 初始化 | `var a [3]int` | `make([]int, 3, 5)` |
| 扩容 | 不能扩容 | `len == cap` 时自动扩容 |

#### 数组与切片本质

##### 数组：值类型
赋值、传参会完整拷贝，修改副本不影响原数组。
```go
a := [3]int{1,2,3}
b := a
b[0] = 0 // a 不变
```

##### 切片：底层数组视图
多个切片可共用底层数组，修改会互相影响。
```go
a := [3]int{1,2,3}
s1 := a[:]
s2 := s1
s2[0] = 0 // a、s1、s2 都变
```

#### 切片 len 与 cap
- `len`：当前元素个数
- `cap`：从起始到底层数组末尾长度
```go
s := make([]int, 2, 5)
s = append(s, 3,4,5) // len=cap=5
s = append(s, 6)     // 触发扩容
```

#### 零值
- 数组：所有元素为对应类型零值
- 切片：`nil`，`len=0`，`cap=0`

#### 使用场景
##### 数组使用场景
- 长度固定且为业务逻辑一部分
- 需要值语义、避免数据共享

##### 切片使用场景
- 动态长度、需要 append
- 函数传参更高效
- 需要操作数据片段

#### 示例代码
```go
// 数组不能 append
var arr [3]int = [3]int{1,2,3}

// 切片可以 append
var slice []int = []int{1,2,3}
slice = append(slice, 4)
```
###  golang io.write的原理
  ![image](https://user-images.githubusercontent.com/31843331/153559726-7a20134f-4dbd-4100-bb24-21ff774a4f45.png)
### 账号系统怎么做认证的 session和cookie
#### 账号系统认证核心逻辑
账号系统的认证本质是**验证用户身份合法性**，并在后续请求中持续识别用户，核心流程：
1. 用户提交账号密码 → 服务端验证通过后，生成**用户身份凭证**（如Session ID）；
2. 服务端存储凭证与用户信息的关联（如Redis存Session ID → 用户ID）；
3. 服务端将凭证返回客户端（通过Cookie/Header等）；
4. 客户端后续请求携带该凭证 → 服务端验证凭证有效性，确认用户身份。

#### Session 与 Cookie 基础概念
##### Cookie
客户端（浏览器）存储的**小型键值对数据**，由服务端通过响应头 `Set-Cookie` 下发，客户端后续请求会自动携带对应Cookie到服务端。
- 存储位置：客户端浏览器（可配置过期时间、域名、路径）；
- 大小限制：通常4KB左右；
- 核心作用：承载Session ID，或存储非敏感的少量用户信息（如记住登录状态）。

##### Session
服务端为每个登录用户创建的**身份会话**，本质是服务端存储的“用户状态数据”（如用户ID、登录时间、权限），通过Session ID关联用户。
- 存储位置：服务端（内存/Redis/Mysql等）；
- 无大小限制；
- 核心作用：安全存储用户登录状态，避免敏感信息暴露在客户端。

#### Session 与 Cookie 的核心区别
| 特性 | Cookie | Session |
| :--- | :--- | :--- |
| 存储位置 | 客户端浏览器 | 服务端 |
| 数据大小 | 有限制（约4KB） | 无限制（受服务端资源影响） |
| 安全性 | 较低（易被篡改/窃取） | 较高（仅Session ID暴露） |
| 服务器压力 | 无（存储在客户端） | 有（需占用服务端资源） |
| 有效期 | 可设置持久化（如7天） | 默认随会话结束失效（可配置） |
| 跨域支持 | 受同源策略限制 | 依赖Cookie传递ID，跨域需特殊处理 |

#### 账号系统中 Session+Cookie 认证流程
```mermaid
sequenceDiagram
    participant 客户端
    participant 服务端
    客户端->>服务端: 提交账号密码（登录请求）
    服务端->>服务端: 验证账号密码有效性
    服务端->>服务端: 生成唯一Session ID，存储Session（用户ID/权限）
    服务端->>客户端: 响应登录成功，通过Set-Cookie下发Session ID
    客户端->>服务端: 后续请求（自动携带Cookie中的Session ID）
    服务端->>服务端: 验证Session ID有效性，查询用户信息
    服务端->>客户端: 返回用户专属数据/操作结果
```

#### 关键注意事项
##### Cookie 安全配置
- `HttpOnly: true`：禁止JS读取Cookie，防止XSS攻击；
- `Secure: true`：仅在HTTPS协议下传输Cookie；
- `SameSite: Strict/Lax`：限制跨域请求携带Cookie，防止CSRF攻击；
- 设置合理的`Expires/Max-Age`：区分临时会话（关闭浏览器失效）和持久登录。

##### Session 优化
- 避免存储敏感信息（如密码），仅存用户ID、权限等核心数据；
- 用Redis替代内存存储Session，支持分布式部署（解决单点服务Session丢失问题）；
- 设置Session过期时间（如2小时无操作自动失效），定期清理无效Session。

#### 无Cookie场景的替代方案
若客户端不支持Cookie（如小程序/APP），可将Session ID放在：
- HTTP请求头（如`Authorization: Bearer {SessionID}`）；
- 请求参数（需加密，不推荐GET方式）；
- Token方案（如JWT，无需服务端存储Session，直接在Token中加密用户信息）。

### 总结
1. Cookie是客户端存储的“小凭证”，核心作用是传递Session ID，需做好安全配置；
2. Session是服务端存储的“用户会话”，安全但占用服务端资源，适合有状态认证；
3. 账号系统中通常结合使用：Cookie存Session ID，Session存用户状态，兼顾安全与易用性。
### 线上qps多少
### 为什么用channel来控制协程数量，协程太多会timeout
### 分配在栈上和分配在堆上有什么区别，分配在栈上有什么好处
    - 参考：
    - 栈的内存管理简单，分配比堆上快
    - 栈的内存不需要回收，堆需要主动free
    - 栈的内存访问有更好的局部性，堆上的访问速度比栈上的速度要慢
### 怎么获取当前goroutine的数量，怎么获取当前goroutine的id
    - run.NumGoroutines()
    - goid 从runtime.stack上获取
### 线程间的通信方式一般有哪几种锁

#### 线程间的通信方式

线程间通信主要有以下几种方式：

#### 共享内存

多个线程访问同一块内存区域，通过读写共享变量来交换信息，需配合锁机制防止数据竞争。

#### 锁（Lock / Mutex）

互斥锁保证同一时刻只有一个线程访问临界区，常见有：互斥锁（Mutex）、读写锁（RWMutex）、自旋锁（Spinlock）。

#### 信号量（Semaphore）

通过计数器控制多个线程对资源的访问数量，可用于限流或线程同步。

#### 条件变量（Condition Variable）

线程在某个条件不满足时主动挂起等待，由其他线程在条件满足时唤醒，常与互斥锁配合使用。

#### Channel（管道）

Go 语言推崇的方式，通过发送/接收消息传递数据，避免共享内存，天然线程安全。

#### 消息队列

线程将消息放入队列，其他线程从队列中取出处理，实现解耦和异步通信。

#### 信号（Signal）

操作系统层面的通知机制，一个线程向另一个线程发送信号来触发特定处理逻辑。

### golang map[string]interface{}做形参能否传入，map[string]string
#### 结论

**不能**直接传入，会编译报错。

#### 示例代码

```go
func foo(m map[string]interface{}) {}

func main() {
    m := map[string]string{"key": "value"}
    foo(m) // ❌ 编译错误：cannot use m (type map[string]string) as type map[string]interface{}
}
```

#### 原因

Go 是强类型语言，`map[string]string` 和 `map[string]interface{}` 是两个完全不同的类型，即使 `string` 实现了 `interface{}`，两种 map 类型之间也**不存在隐式转换**。

#### 解决方案

手动转换：

```go
m := map[string]string{"key": "value"}

converted := make(map[string]interface{}, len(m))
for k, v := range m {
    converted[k] = v
}
foo(converted) // ✅
```

使用泛型（Go 1.18+）：

```go
func foo[V any](m map[string]V) {}

func main() {
    m := map[string]string{"key": "value"}
    foo(m) // ✅
}
```

#### 总结

| 方式 | 是否可行 |
|------|--------|
| 直接传入 | ❌ 编译报错 |
| 手动转换后传入 | ✅ |
| 改用泛型形参 | ✅ |

### 单核goroutine中死循环，怎么调度出来

#### 结论

单核（`GOMAXPROCS=1`）下的纯死循环，**Go 1.14 之前无法被调度出来**，Go 1.14+ 通过异步抢占机制可以强制调度。

#### Go 1.14 之前的问题

Go 调度是**协作式**的，goroutine 只在以下时机让出 CPU：系统调用、channel 阻塞、`runtime.Gosched()`、函数调用。纯死循环没有任何调度点，其他 goroutine 永远无法执行。

```go
go func() {
    for {} // ❌ 无调度点，永远占用 CPU
}()
```

#### Go 1.14+ 异步抢占机制

引入基于 **SIGURG 信号**的异步抢占：`sysmon` 监控线程每 **10ms** 检测一次，若 goroutine 运行超时则发送信号，强制在安全点切换。

```go
go func() {
    for {} // ✅ 10ms 后被 sysmon 强制抢占调度出去
}()
```

#### 手动解决方案

```go
// 方式一：主动让出
go func() {
    for {
        runtime.Gosched() // 主动交出控制权
    }
}()

// 方式二：加入函数调用
go func() {
    for {
        doSomething() // 编译器自动插入调度检查点
    }
}()
```

#### 总结

| 场景 | 能否调度出来 |
|------|------------|
| Go 1.14 之前 + 纯死循环 | ❌ 无法调度 |
| Go 1.14+ + 纯死循环 | ✅ 异步抢占（10ms）|
| 任意版本 + `Gosched()` | ✅ 主动让出 |
| 任意版本 + 含函数调用 | ✅ 编译器插入检查点 |


### golang debug工具 性能分析
#### pprof —— 性能剖析核心工具

Go 内置性能分析工具，支持 CPU、内存、goroutine、阻塞等多维度分析。

```go
import _ "net/http/pprof"

func main() {
    go http.ListenAndServe(":6060", nil)
}
```

```bash
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30  # CPU
go tool pprof http://localhost:6060/debug/pprof/heap                # 内存
go tool pprof http://localhost:6060/debug/pprof/goroutine           # goroutine
go tool pprof -http=:8080 cpu.out                                   # Web UI
```

#### trace —— 执行追踪工具

分析 goroutine 调度、GC、系统调用时间线，定位延迟问题。

```bash
curl http://localhost:6060/debug/pprof/trace?seconds=5 > trace.out
go tool trace trace.out
```

#### benchmark —— 基准测试

```go
func BenchmarkFoo(b *testing.B) {
    for i := 0; i < b.N; i++ {
        Foo()
    }
}
```

```bash
go test -bench=. -benchmem
go test -bench=. -cpuprofile=cpu.out -memprofile=mem.out
```

#### dlv —— Delve 调试器

```bash
dlv debug main.go

(dlv) break main.main   # 设置断点
(dlv) continue          # 继续执行
(dlv) next              # 单步执行
(dlv) print varName     # 打印变量
(dlv) goroutines        # 查看所有 goroutine
```

#### go tool —— 内置分析工具集

```bash
go build -gcflags="-m" main.go    # 逃逸分析
go run -race main.go              # 数据竞争检测
go tool objdump -s "main.main" b  # 查看汇编
go build -gcflags="-m=2" main.go  # 编译优化详情
```

#### 总结

| 工具 | 用途 | 适用场景 |
|------|------|--------|
| `pprof` | CPU/内存/goroutine 分析 | 性能瓶颈定位 |
| `trace` | 调度时间线可视化 | 延迟/调度问题 |
| `benchmark` | 函数性能量化 | 代码优化对比 |
| `dlv` | 断点调试 | Bug 排查 |
| `-race` | 数据竞争检测 | 并发安全检查 |
| `-gcflags="-m"` | 逃逸分析 | 内存优化 |


### 链表和数组的区别

### grpc为什么高效

#### 基于 HTTP/2 协议

相比 HTTP/1.1，HTTP/2 带来多路复用、头部压缩、二进制分帧、服务端推送，单连接可并发处理多个请求，大幅降低延迟和带宽消耗。

```
HTTP/1.1：每个请求独占连接，队头阻塞严重
HTTP/2  ：单连接多路复用，多个 RPC 并发互不阻塞
```

#### 使用 Protobuf 序列化

默认使用 Protocol Buffers 二进制序列化，相比 JSON 体积小 3~10 倍，序列化速度快 5~10 倍。

```protobuf
message User {
    int64  id   = 1;  // 字段用编号代替字段名，极度压缩体积
    string name = 2;
}
```

#### 强类型 IDL 契约

通过 `.proto` 文件定义接口，自动生成客户端/服务端代码，避免运行时动态解析开销，编译期即可发现类型错误。

```protobuf
service UserService {
    rpc GetUser (GetUserRequest) returns (User);
    rpc ListUsers (ListRequest) returns (stream User); // 支持流式
}
```

#### 支持四种通信模式

比传统 REST 更灵活，支持流式传输，适合大数据量和实时场景。

| 模式 | 说明 |
|------|------|
| Unary | 普通一请求一响应 |
| Server Streaming | 服务端流式返回 |
| Client Streaming | 客户端流式发送 |
| Bidirectional Streaming | 双向流式通信 |

#### 连接复用与长连接

基于 HTTP/2 长连接，避免了 HTTP/1.1 频繁建立/销毁 TCP 连接的开销，连接建立成本几乎为零。

#### 总结

| 特性 | 优势 |
|------|------|
| HTTP/2 | 多路复用、头部压缩、二进制传输 |
| Protobuf | 体积小、序列化快 |
| 强类型 IDL | 编译期校验、无动态解析 |
| 流式支持 | 适合实时/大数据场景 |
| 长连接复用 | 减少连接建立开销 |

## Map相关进阶
###  map深拷贝浅拷贝
#### 结论

Go 中 map 赋值是**浅拷贝**，两个变量指向同一底层数据，修改一个会影响另一个。

#### 浅拷贝（默认行为）

```go
a := map[string]int{"x": 1, "y": 2}
b := a // ❌ 浅拷贝，b 和 a 指向同一底层数据

b["x"] = 999
fmt.Println(a["x"]) // 999，a 也被修改了
```

#### 深拷贝——value 为基本类型

```go
a := map[string]int{"x": 1, "y": 2}

b := make(map[string]int, len(a))
for k, v := range a {
    b[k] = v // ✅ 完全独立的副本
}

b["x"] = 999
fmt.Println(a["x"]) // 1，a 不受影响
```

#### 深拷贝——value 为引用类型

```go
a := map[string][]int{"nums": {1, 2, 3}}

// ❌ 只拷贝外层 map，内层 slice 仍是浅拷贝
b := make(map[string][]int)
for k, v := range a {
    b[k] = v
}
b["nums"][0] = 999
fmt.Println(a["nums"][0]) // 999，a 仍被影响

// ✅ 递归深拷贝内层数据
b := make(map[string][]int)
for k, v := range a {
    tmp := make([]int, len(v))
    copy(tmp, v)
    b[k] = tmp
}
```

#### 深拷贝——json 序列化（通用方案）

```go
func deepCopy(src map[string]interface{}) map[string]interface{} {
    data, _ := json.Marshal(src)
    dst := make(map[string]interface{})
    json.Unmarshal(data, &dst)
    return dst
}
```

#### 总结

| 方式 | 类型 | 是否独立 |
|------|------|--------|
| `b := a` | 浅拷贝 | ❌ 共享底层数据 |
| `for range`（value 基本类型） | 深拷贝 | ✅ |
| `for range`（value 引用类型） | 浅拷贝 | ❌ 内层仍共享 |
| 递归拷贝内层数据 | 深拷贝 | ✅ |
| json 序列化/反序列化 | 深拷贝 | ✅ 但性能较差 |

### slice和map的扩容机制，map扩容时读数据怎么处理的
### map实现及底层原理？(sixin)
    - [go 设计与实现](https://draveness.me/golang/docs/part2-foundation/ch03-datastructure/golang-hashmap/#%E6%89%A9%E5%AE%B9)
### 如何手动设计一个map

### 有一个写多读少的场景，怎么设计高性能map

#### 结论

写多读少场景下，核心矛盾是**写锁竞争激烈**，最优解是**分片锁 Map**，将锁粒度从全局降到分片级别。

#### 设计思路

```
一个大 Map（全局锁）
        ↓ 拆分
32 个小 Map（每片独立锁）

写 key="user_123"  → hash → 落到第 7  片 → 只锁第 7  片
写 key="order_456" → hash → 落到第 15 片 → 只锁第 15 片
→ 两个写操作完全并行，互不阻塞
```

#### 完整实现

```go
package main

import (
    "sync"
    "fmt"
)

const shardCount = 32 // 建议为 2 的幂次

type Shard struct {
    sync.RWMutex
    data map[string]interface{}
}

type ShardedMap [shardCount]*Shard

func NewShardedMap() ShardedMap {
    var m ShardedMap
    for i := 0; i < shardCount; i++ {
        m[i] = &Shard{data: make(map[string]interface{})}
    }
    return m
}

// hash 选片（位运算代替取模，更快）
func (m ShardedMap) getShard(key string) *Shard {
    hash := fnv32(key)
    return m[hash&(shardCount-1)]
}

// 写
func (m ShardedMap) Set(key string, val interface{}) {
    shard := m.getShard(key)
    shard.Lock()
    defer shard.Unlock()
    shard.data[key] = val
}

// 读
func (m ShardedMap) Get(key string) (interface{}, bool) {
    shard := m.getShard(key)
    shard.RLock()
    defer shard.RUnlock()
    return shard.data[key]
}

// 删除
func (m ShardedMap) Delete(key string) {
    shard := m.getShard(key)
    shard.Lock()
    defer shard.Unlock()
    delete(shard.data, key)
}

// FNV hash
func fnv32(key string) uint32 {
    h := uint32(2166136261)
    for i := 0; i < len(key); i++ {
        h = (h * 16777619) ^ uint32(key[i])
    }
    return h
}
```

#### 关键设计细节

**分片数取 2 的幂次：**
```go
hash & (shardCount - 1)  // ✅ 位运算，比取模快
hash % shardCount         // ❌ 除法，较慢
```

**为什么不用 sync.Map：**
```
sync.Map 内部维护 read map 和 dirty map
写多时 dirty → read 频繁提升（promotion），开销极大
写多场景性能甚至不如 map + 全局锁
```

**分片数如何选：**
```
推荐 = CPU 核数 × 4~8
生产环境通常设置 32 或 64
并发数 ≤ 分片数时，锁竞争趋近于零
```

#### 性能对比

| 方案 | 写并发 100 | 写并发 10000 |
|------|-----------|------------|
| `map` + 全局锁 | ⚠️ 串行阻塞 | ❌ 严重竞争 |
| `sync.Map` | ⚠️ dirty 提升开销 | ❌ 不适合写多 |
| 分片锁 32 片 | ✅ 竞争降低 32 倍 | ✅ 高吞吐 |

#### 总结

分片锁是写多读少场景的最优解，核心思想是**化整为零，分而治之**，32 个分片理论上将写锁竞争降低 32 倍。

### map 锁+map sync.map concurrentmap的区别
### sync map的原理
    - [refer1](https://blog.csdn.net/weixin_42663840/article/details/107958274)
    - [refer2](https://blog.csdn.net/u011957758/article/details/96633984?ops_request_misc=%257B%2522request%255Fid%2522%253A%2522164515668616781683951530%2522%252C%2522scm%2522%253A%252220140713.130102334..%2522%257D&request_id=164515668616781683951530&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~all~sobaiduend~default-1-96633984.pc_search_result_positive&utm_term=golang+syncmap&spm=1018.2226.3001.4187)
- Go 如何高效地拼接字符串?

## 其他资源
- [github他人收集](https://github.com/KeKe-Li/data-structures-questions/blob/master/src/chapter05/golang.01.md#Go%E4%B8%AD%E7%9A%84%E9%94%81%E5%A6%82%E4%BD%95%E5%AE%9E%E7%8E%B0)
## 无锁设计
- 怎么设计一个无锁的pool



## 测试
- go项目如何做覆盖率测试
#### 基本命令

```bash
# 运行测试并统计覆盖率
go test -cover ./...

# 输出覆盖率报告文件
go test -coverprofile=coverage.out ./...

# 查看覆盖率汇总
go tool cover -func=coverage.out
```

#### 可视化 HTML 报告

```bash
# 生成 HTML 文件
go tool cover -html=coverage.out -o coverage.html

# 直接打开浏览器
go tool cover -html=coverage.out
```

```
绿色 → 已覆盖代码
红色 → 未覆盖代码
灰色 → 不可执行代码
```

#### 覆盖率模式

```bash
# set（默认）：只标记是否执行过
go test -covermode=set -coverprofile=coverage.out ./...

# count：统计每行执行次数
go test -covermode=count -coverprofile=coverage.out ./...

# atomic：并发安全的 count（并发测试推荐）
go test -covermode=atomic -coverprofile=coverage.out ./...
```

#### 指定包与过滤

```bash
# 只测试指定包
go test -coverprofile=coverage.out ./internal/...

# 包含所有依赖包统计
go test -coverpkg=./... -coverprofile=coverage.out ./...

# 排除自动生成代码
go test -coverprofile=coverage.out ./... | grep -v "mock\|pb\|gen"
```

#### CI 设置覆盖率阈值

```bash
#!/bin/bash
go test -coverprofile=coverage.out ./...

COVERAGE=$(go tool cover -func=coverage.out | grep total | awk '{print $3}' | tr -d '%')
echo "Coverage: $COVERAGE%"

if (( $(echo "$COVERAGE < 80" | bc -l) )); then
    echo "❌ 覆盖率低于 80%"
    exit 1
fi
echo "✅ 覆盖率检查通过"
```

#### 集成 GitHub Actions

```yaml
name: Coverage
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-go@v4
        with:
          go-version: '1.21'
      - name: Run tests
        run: go test -coverprofile=coverage.out -covermode=atomic ./...
      - name: Upload to Codecov
        uses: codecov/codecov-action@v3
        with:
          files: ./coverage.out
```

#### 总结

| 命令 | 用途 |
|------|------|
| `go test -cover` | 快速查看覆盖率 |
| `-coverprofile=xxx.out` | 输出报告文件 |
| `cover -html` | 可视化 HTML |
| `cover -func` | 按函数查看覆盖率 |
| `-covermode=atomic` | 并发安全统计 |
| `-coverpkg=./...` | 包含所有依赖包 |

## Runtime
- runtime.GOMAXPROCS(0)表示什么？为什么要这么用？

## Interface
- interface内部实现原理
- reflect的用途
- 指针实现接口和结构体实现接口有什么区别？
    - [refer](https://mp.weixin.qq.com/s/g-D_eVh-8JaIoRne09bJ3Q)

## 其他问题
### 当G发送阻塞时,G和M和P是如何变化的

#### 当 G 发送阻塞时，G / M / P 的变化

#### 背景：GMP 模型简介

| 概念 | 含义 |
|------|------|
| **G** | Goroutine，Go 的轻量级协程 |
| **M** | Machine，OS 线程 |
| **P** | Processor，调度器上下文，持有本地运行队列 |

---

#### 阻塞的两种类型

#### 1. Channel 发送阻塞（用户态阻塞）

```
ch <- data   // 无接收方，G 阻塞
```

**变化过程：**

```
阻塞前：
  P → runq: [G2, G3...]
  M → P → G1(running)

阻塞时：
  G1 状态: running → waiting
  G1 从 M 上脱离，挂入 channel 的 sendq 队列
  M 不阻塞，继续向 P 要下一个 G

  P → runq: [G2, G3...]
  M → P → G2(running)   ← M/P 照常工作
  sendq: [G1]            ← G1 在此等待
```

**关键：M 和 P 不受影响，继续执行其他 G。**

---

#### 2. 系统调用阻塞（内核态阻塞）

```
syscall.Write(...)   // 进入内核，M 真正阻塞
```

**变化过程：**

```
阻塞前：
  M0 → P → G1(running)

进入 syscall：
  G1 状态: running → syscall
  P 与 M0 解绑！
  P 转移给 M1（新线程或从线程池取）
  M0 继续陪 G1 等待系统调用返回（M0 阻塞在内核）

  M1 → P → G2(running)  ← P 和 M1 照常工作
  M0 → G1              ← M0/G1 阻塞等待

syscall 返回：
  G1 尝试获取一个 P（优先原来的 P）
  ├─ 有空闲 P → G1 绑定 P，继续执行
  └─ 无空闲 P → G1 进入全局队列，M0 进入线程池休眠
```

---

#### 状态对比总结

| 场景 | G 状态 | M 状态 | P 状态 |
|------|--------|--------|--------|
| Channel 阻塞 | `waiting`，挂入 sendq | 继续运行其他 G | 不变，继续调度 |
| Syscall 阻塞 | `syscall` | 随 G 阻塞在内核 | **与 M 解绑**，转移给新 M |
| 阻塞恢复后 | `runnable` → `running` | 恢复或入池 | 重新绑定或等待 |

---

#### 核心设计哲学

> **P 是稀缺资源，绝不能被阻塞浪费。**

- **用户态阻塞**：G 自己挂起，M/P 毫无感知，代价极小。
- **内核态阻塞**：P 主动"逃离" M，交给别人继续干活，保证并发度不下降。

这就是 Go 能用少量 OS 线程支撑大量 Goroutine 的根本原因。


### 什么是自旋,M为什么要自旋
#### 什么是自旋

自旋（Spinning）是指 M 在**没有可运行 G 的情况下，不立即进入休眠**，而是空转循环、主动轮询，等待新的 G 出现。

```go
// 伪代码示意
for {
    if g := findRunnable(); g != nil {
        execute(g)
    }
    // 没找到 G，继续空转（自旋中）
}
```

---

#### 为什么 M 要自旋

#### 核心矛盾

| 方案 | 问题 |
|------|------|
| 没 G 就立刻休眠 | 新 G 来了需要唤醒 M，**唤醒有延迟**，影响响应速度 |
| 永远自旋不休眠 | **浪费 CPU**，空转消耗资源 |

自旋是两者之间的**折中策略**。

---

#### 自旋的本质收益

```
场景：G1 正在运行，即将产生新的 G2

  无自旋：M1 已休眠 → G2 入队 → 唤醒 M1 → M1 调度 G2
          ↑ 存在唤醒延迟（线程切换开销 ~μs 级）

  有自旋：M1 自旋中 → G2 入队 → M1 立刻发现 → 直接调度 G2
          ↑ 几乎零延迟
```

---

#### Go 运行时的自旋规则

**自旋 M 的数量上限：**

```
自旋 M 数量 ≤ GOMAXPROCS（即 P 的数量）
```

同时满足以下条件才允许自旋：
- 当前自旋的 M 数 × 2 < 正在运行的 P 数
- 没有其他自旋的 M 可以接手

**自旋检查的对象：**

```
① 本地队列 P.runq      → 有 G？直接拿
② 全局队列 sched.runq  → 有 G？批量取
③ 网络轮询器 netpoll   → 有就绪 G？取走
④ 其他 P 的队列        → Work Stealing 偷一半
```

**自旋超时后：**

```
M 仍未找到 G
  → 退出自旋状态
  → 调用 stopm() 进入休眠
  → 等待 wakep() 信号重新激活
```

---

#### 自旋 M 的生命周期

```
[有 G 可运行]
     ↓
  M 执行 G
     ↓
  G 执行完毕，本地队列为空
     ↓
  M 进入自旋 ──────── 找到新 G → 退出自旋，继续执行
     ↓（超时/无 G）
  M 退出自旋，进入休眠
     ↓
  新 G 入队，唤醒 M
     ↓
  M 重新竞争 P
```

---

#### 总结

> 自旋是 Go 调度器用**少量 CPU 空转换取低调度延迟**的权衡机制。

- **自旋 = 用 CPU 换时间**，适合 G 产生频繁的场景
- **限制自旋数量**，防止过多 M 空转浪费 CPU
- **自旋 + 抢占 + Work Stealing** 共同保障了 Go 高并发下的低延迟调度

### map使用时有什么要注意的

#### map 使用时有什么要注意的

---

#### 1. 使用前必须初始化

```go
// ❌ 错误：nil map 不能赋值，会 panic
var m map[string]int
m["key"] = 1  // panic: assignment to entry in nil map

// ✅ 正确：make 初始化
m := make(map[string]int)
m["key"] = 1

// ✅ 正确：字面量初始化
m := map[string]int{"key": 1}
```

---

#### 2. 读取不存在的 key 不会 panic

```go
m := make(map[string]int)

val := m["notExist"]       // 返回零值 0，不会报错
val, ok := m["notExist"]   // ok = false，推荐此写法判断是否存在

if val, ok := m["key"]; ok {
    // key 存在才处理
}
```

---

#### 3. 并发读写不安全

```go
// ❌ 多个 goroutine 同时读写同一个 map → fatal error: concurrent map read and map write

// ✅ 方案一：加互斥锁
var mu sync.RWMutex
mu.Lock()
m["key"] = 1
mu.Unlock()

// ✅ 方案二：使用 sync.Map（适合读多写少）
var sm sync.Map
sm.Store("key", 1)
val, ok := sm.Load("key")
```

---

#### 4. 不能对 map 的 value 取地址

```go
type User struct{ Age int }
m := map[string]User{"alice": {18}}

// ❌ 错误：map 内部会扩容，地址不稳定
p := &m["alice"]  // cannot take the address of m["alice"]

// ✅ 正确：取出来修改后再放回去
u := m["alice"]
u.Age = 20
m["alice"] = u
```

---

#### 5. 遍历顺序是随机的

```go
m := map[string]int{"a": 1, "b": 2, "c": 3}

// 每次遍历顺序不固定，Go 故意随机化
for k, v := range m {
    fmt.Println(k, v)
}

// ✅ 需要有序遍历：先提取 key 排序
keys := make([]string, 0, len(m))
for k := range m {
    keys = append(keys, k)
}
sort.Strings(keys)
for _, k := range keys {
    fmt.Println(k, m[k])
}
```

---

#### 6. map 只能用 == 可比较的类型作 key

```go
// ✅ 合法 key 类型：string、int、bool、指针、struct（字段均可比较）
m := map[string]int{}
m := map[int]string{}

// ❌ 非法 key 类型：slice、map、func（不可比较）
m := map[[]int]string{}   // invalid map key type []int
m := map[map[string]int]string{}  // invalid map key type
```

---

#### 7. 删除 key 用 delete，不会 panic

```go
m := map[string]int{"a": 1}

delete(m, "a")          // ✅ 正常删除
delete(m, "notExist")   // ✅ key 不存在也不会 panic
```

---

#### 8. map 是引用类型，赋值不会拷贝

```go
m1 := map[string]int{"a": 1}
m2 := m1  // m2 和 m1 指向同一底层数据

m2["a"] = 99
fmt.Println(m1["a"])  // 输出 99，m1 也被修改了

// ✅ 需要拷贝时，手动遍历复制
m3 := make(map[string]int)
for k, v := range m1 {
    m3[k] = v
}
```

---

#### 9. map 不会自动缩容

```go
m := make(map[int]int)
for i := 0; i < 100000; i++ {
    m[i] = i
}
for i := 0; i < 100000; i++ {
    delete(m, i)  // 删除所有 key，但内存不会释放
}

// ✅ 需要释放内存：重新赋值一个新 map
m = make(map[int]int)
```

---

#### 总结

| 注意点 | 关键结论 |
|--------|--------|
| 初始化 | 必须 `make` 或字面量，否则 panic |
| 并发安全 | 原生 map 不安全，用 `sync.Mutex` 或 `sync.Map` |
| 取地址 | value 不可取地址，需取出修改再放回 |
| 遍历顺序 | 随机，需有序须手动排序 |
| key 类型 | 必须可比较，slice/map/func 不可作 key |
| 引用类型 | 赋值共享底层数据，深拷贝需手动复制 |
| 内存释放 | 删除 key 不缩容，需重建 map 释放内存 |