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

## 访问 www.xxx.com 发生了什么

访问 `www.xxx.com` 的完整流程涉及网络通信、协议交互和系统处理等多个环节，以下是核心步骤的分阶段解析：


### **1. 域名解析（DNS 解析）**
- **本地缓存检查**：操作系统首先检查本地 `hosts` 文件或 DNS 缓存，查找 `www.xxx.com` 的 IP 地址。
- **递归解析**：
  - 向本地 DNS 服务器（如运营商提供的 DNS）发起查询。
  - 本地 DNS 服务器通过递归查询（根域名服务器 → 顶级域名服务器（`.com`） → 权威域名服务器（`xxx.com`））获取目标 IP。
- **结果缓存**：解析成功后，IP 地址被缓存（浏览器、本地 DNS 等），加速后续访问。


### **2. 建立连接（TCP 三次握手 + TLS 握手（HTTPS 场景））**
#### **TCP 连接（基于 HTTP/1.1 或 HTTP/2）**
- **三次握手**：客户端与服务器建立 TCP 连接（SYN → SYN+ACK → ACK）。
- **端口默认值**：HTTP（80）、HTTPS（443）。

#### **TLS 握手（HTTPS 场景）**
- **证书验证**：客户端验证服务器证书的合法性（CA 签名、域名匹配、有效期）。
- **密钥协商**：通过 DH 算法协商对称加密密钥，建立安全通道。
- **加密通信**：后续数据传输通过 AES 等算法加密。


### **3. 发送 HTTP 请求**
- **请求构造**：客户端发送 HTTP 请求（如 `GET / HTTP/1.1`），包含：
  - **请求行**：方法（GET/POST 等）、路径、协议版本。
  - **请求头**：`Host: www.xxx.com`、`User-Agent`、`Cookie` 等。
  - **请求体**（如 POST 数据）。
- **协议优化**：
  - HTTP/2：多路复用、头部压缩（HPACK）。
  - HTTP/3：基于 UDP 的 QUIC 协议，减少连接延迟。


### **4. 服务器端处理**
#### **负载均衡与流量分发**
- **CDN 缓存**：若域名配置了 CDN，请求先路由到最近的 CDN 节点，直接返回缓存的静态资源（如图片、CSS）。
- **负载均衡器**（如 Nginx、LVS）：将请求分发到后端应用服务器集群，实现高可用。

#### **应用服务器处理**
- **静态资源**：直接读取文件（如 HTML、JS），通过缓存策略（`Cache-Control`）减少重复计算。
- **动态内容**：
  - 调用业务逻辑（如 Java/Go 服务），查询数据库（如 MySQL、Redis）。
  - 生成响应数据（如 JSON、HTML）。
- **身份验证**：校验用户身份（JWT、Session），权限控制。

#### **响应构造**
- **响应行**：协议版本、状态码（如 200 OK、301 重定向、404 未找到）。
- **响应头**：`Content-Type`、`Set-Cookie`、`Cache-Control` 等。
- **响应体**：返回的内容（HTML 页面、JSON 数据等）。


### **5. 客户端处理响应**
- **解析响应**：浏览器解析 HTML，构建 DOM 树。
- **资源加载**：并行加载 CSS、JS、图片等（受 `max-concurrent-connections` 限制）。
- **执行脚本**：解析 JS（如 Vue/React 渲染），发起异步请求（AJAX/HTTP API）。
- **渲染页面**：呈现最终用户界面。


### **6. 连接管理**
- **TCP 四次挥手**：通信结束后关闭连接（或通过 `Keep-Alive` 保持长连接）。
- **HTTP/2 持久连接**：复用 TCP 连接，减少延迟。


### **扩展：高并发场景的优化（结合用户历史问题）**
- **CDN 加速**：静态资源分布式缓存，减少源站压力。
- **负载均衡**：多层负载（如 L4 层 SLB + L7 层 Nginx），流量分流。
- **缓存层**：Redis/Memcached 缓存热点数据，降低数据库负载。
- **异步处理**：消息队列（Kafka/RabbitMQ）削峰填谷，解耦服务。
- **分布式架构**：微服务拆分（参考用户之前的 Go-micro 问题），弹性扩展。


### **总结：全流程示意图**
```
客户端 → DNS 解析 → TCP/TLS 连接 → HTTP 请求 → 负载均衡 → 应用处理（缓存/数据库） → 响应 → 客户端渲染
       └─────────── 网络传输 ────────────┘       └────── 服务器处理 ──────┘
```

### **关键技术点**
- **可靠性**：TCP 保证传输有序，TLS 保证数据安全。
- **性能**：CDN、缓存、HTTP/2 多路复用、异步架构。
- **扩展性**：分布式系统、微服务、弹性伸缩（如 Kubernetes）。

通过以上流程，用户最终看到的是浏览器渲染的完整页面，背后涉及网络、协议、服务器架构等多层技术的协同工作。


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

你的内容已经符合 **最大三级标题（###）** 的要求，我做了一些格式优化，让它更清晰易读：  

---

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

## 并发安全
- cas  修改一块内存的值，值改变方式是a-b-a这个合理吗
    - 什么事ABA问题
        - 线程1，期望值为A，欲更新的值为B
        - 线程2，期望值为A，欲更新的值为B
        - 线程1抢先获得CPU时间片，而线程2因为其他原因阻塞
        - 线程1取值与期望的A值比较，发现相等然后将值更新为B
        - 线程3，期望值为B，欲更新的值为A，线程3取值与期望的值B比较，发现相等则将值更新为A
        - 线程2从阻塞中恢复，并且获得了CPU时间片，这时候线程2取值与期望的值A比较，发现相等则将值更新为B
        - 虽然线程2也完成了操作，但是线程2并不知道值已经经过了A->B->A的变化过程
    - 如何解决ABA问题
        - 在变量前面加上版本号，每次变量更新的时候变量的版本号都+1，即A->B->A就变成了1A->2B->3A

## mutex的原理
    - [refer](https://www.processon.com/view/link/6078e4416376891132d67bcf)

## 设计模式

- 说说常用的设计模式

## 切片
- https://www.processon.com/view/link/67dfe46a94f7ab2d0f7bfaa2?cid=67c57fc2ed8a6e36862f698d

## GC
- 1. go gc 为何是非分代的？
    - [refer](https://lingchao.xin/post/why-golang-garbage-collector-not-implement-generational-and-compact-gc.html)
- 2. go gc 为何是非紧缩的?
    - [refer](https://lingchao.xin/post/why-golang-garbage-collector-not-implement-generational-and-compact-gc.html)
    - [refer](https://www.jianshu.com/p/f1d62dcb0d76)
- 3. 并发三色标记扫描是什么？
- 4. go 如何实现的 并发三色标记扫描？
- 5. 强三色不变性和弱三色不变性的含义？
- 6. gc是为何需要写屏障？
- 7. 插入写屏障和删除写屏障的时机和区别？go中如何实现的·？
- 8. GC 的四个阶段？
- 9. 为何需要辅助标记和辅助清扫？
- 10. GC 4个阶段，STW发生在何时？
- 11. 描述下 gc 调步算法的实现？
- 12. 工作中gc debug的使用？
- 13. gc 清扫阶段 对象回收 和 内存单元回收的联系和差异？
- 14. 三色法为什么需要灰色
    - [refer](https://stackoverflow.com/questions/9285741/why-white-gray-black-in-gc#:~:text=2%20Answers&text=Gray%20means%20%22live%20but%20not,do%20a%20bit%20of%20marking.\)

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

### 实现题
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

## 面试题汇总
- 1.项目中用到的锁
- 2.介绍一下线程安全的共享内存方式
- 3.介绍一下goroutine
- 4.goroutine的自旋占用资源如何解决,gmp
- 5.介绍Linux系统信号
- 6.goroutine抢占时机,gc栈扫描
- 7.Gc触发时机
- 8.是否了解其他gc机制
- 10.Channel分配在堆上还是在栈上？哪些对象分配在堆上？哪些对象分配在栈上？
- 11.代码效率分析，考虑局部性原理
- 12.多核CPU下，cache如何保持一致，不冲突
- 13.uint类型溢出
- 14.聊聊rune类型
- 15.介绍一下channel，有缓冲和无缓冲的区别
- 16.channel是否线程安全
- 17.介绍一下Mutex的实现,是悲观锁还是乐观锁
- 18.Mutex几种模式?
- 19.Muxtez可以做自旋锁?
- 20.介绍一下RWMutex
- 21.介绍一下大对象和小对象，为什么小对象多了会造成gc压力？
- 22.介绍项目中遇到的oop情况
- 23.介绍项目中遇到的坑
- 24.如果指定指令执行的顺序
- 25.什么是写屏障、混合写屏障，如何实现？
- 26.gc的stw是怎么回事
- 27.协程之间是怎么调度的
- 28.简单聊聊内存逃逸
- 29.为什么sync.WaitGroup中Wait函数支持 WaitTimeout 功能
- 30.字符串转成byte数组，会发生内存拷贝吗？
- 31.http包的内存泄漏
- 32.Goroutine调度策略
- 33.对已经关闭的的chan进行读写，会怎么样？为什么？
- 34.实现阻塞读的并发安全Map
- 35.什么是goroutine leak？
- 36.data race问题怎么解决？能不能不加锁解决这个问题？
- grpc内部原理是什么
- time.Now有几次系统调用？如何优化
- 空struct{}是否使用过？会在什么情况下使用，举例说明一下
- 聊聊runtime
- 介绍下你平时都是怎么调试bug以及性能问题的?
- 通过通信来共享内存，而不是通过共享内存而通信，怎么理解这句话，如何处理共享变量？
- chan比mutex更轻么？还有更轻量的方法么？
- 什么时候用chan不如mutex效率高？
- 什么场景下会触发panic
  - 数组越界
  - 并发写map和未初始化map
  - 空指针调用
  - 过早关闭http响应体
  - 除数为0
  - 向已经关闭的chan发送信息
  - 重复关闭chan
  - 关闭未初始化的chan
  - sync.waitGroup计数为负数
- 什么事hash冲突，go中map如何解决hash冲突？
- 多个携程间的通信方法
- 除了mutex以外还有哪些方式安全读写共享变量
- 用过什么包管理工具
- golang的服务性能指标怎么查看，有哪些指标需要注意
- 协程池的意义是什么
- 8、Go 当中同步锁有什么特点？作用是什么
    - 当一个 Goroutine（协程）获得了 Mutex 后，其他 Goroutine（协程）就只能乖乖的等待，除非该 Goroutine 释放了该 Mutex。RWMutex 在读锁占用的情况下， 会阻止写，但不阻止读 RWMutex。 在写锁占用情况下，会阻止任何其他Goroutine（无论读和写）进来，整个锁相当于由该 Goroutine 独占
    同步锁的作用是保证资源在使用时的独有性，不会因为并发而导致数据错乱， 保证系统的稳定性。

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

## 并发控制
- CHANNEL 的RING BUFFER 实现 
- MUTEX 几种状态
- MUTEX 正常模式和饥饿模式
- MUTEX 允许自旋的条件
- RWMUTEX 实现 
- RWMUTEX 注意事项
- COND 是什么
- BROADCAST 和SIGNAL 区别
- COND 中WAIT 使用
- WAITGROUP 用法
- WAITGROUP 实现原理
- 什么是SYNC.ONCE
- 什么操作叫做原子操作
- 原子操作和锁的区别
- 什么是CAS
- SYNC.POOL 有什么用

## GMP调度
- GOROUTINE 定义
- 1.0 之前 GM 调度模型
- GMP 中WORK STEALING 机制
- GMP 中HAND OFF 机制
- 协作式的抢占式调度
- 基于信号的抢占式调度
- GMP 调度过程中存在哪些阻塞
- SYSMON 有什么作用 

## 垃圾回收
- 三色标记原理
- 插入写屏障
- 删除写屏障
- 写屏障 
- 混合写屏障
- GC 触发时机
- GO 语言中 GC 的流程是什么？
- GC 如何调优

## 微服务架构
- 您对微服务有何了解？
- 说说微服务架构的优势
- 微服务有哪些特点？
- 设计微服务的最佳实践是什么
- 微服务架构如何运作？
- 微服务架构的优缺点是什么
- 单片，SOA 和微服务架构有什么区别？
- 在使用微服务架构时，您面临哪些挑战
- 什么是领域驱动设计？
- 为什么需要域驱动设计（DDD）
- 什么是无所不在的语言？ 
- 什么是凝聚力？
- 什么是耦合？
- 什么是REST / RESTFUL 以及它的用途是什么
- 什么是不同类型的微服务测试？
- 如何设计一个熔断器

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
```

## 其他问题
- slice和数组的区别
- golang io.write的原理
  ![image](https://user-images.githubusercontent.com/31843331/153559726-7a20134f-4dbd-4100-bb24-21ff774a4f45.png)
- 账号系统怎么做认证的 session和cookie
- 线上qps多少
- 为什么用channel来控制协程数量，协程太多会timeout
- 分配在栈上和分配在堆上有什么区别，分配在栈上有什么好处
    - 参考：
    - 栈的内存管理简单，分配比堆上快
    - 栈的内存不需要回收，堆需要主动free
    - 栈的内存访问有更好的局部性，堆上的访问速度比栈上的速度要慢
- 怎么获取当前goroutine的数量，怎么获取当前goroutine的id
    - run.NumGoroutines()
    - goid 从runtime.stack上获取
- 线程间的通信方式一般有哪几种锁
- golang map[string]interface{}做形参能否传入，map[string]string
- 单核goroutine中死循环，怎么调度出来
- golang debug工具 性能分析
- 链表和数组的区别
- grpc为什么高效

## Map相关进阶
- map深拷贝浅拷贝
- slice和map的扩容机制，map扩容时读数据怎么处理的
- map实现及底层原理？(sixin)
    - [go 设计与实现](https://draveness.me/golang/docs/part2-foundation/ch03-datastructure/golang-hashmap/#%E6%89%A9%E5%AE%B9)
- 如何手动设计一个map
- 有一个写多读少的场景，怎么设计高性能map
- map 锁+map sync.map concurrentmap的区别
- sync map的原理
    - [refer1](https://blog.csdn.net/weixin_42663840/article/details/107958274)
    - [refer2](https://blog.csdn.net/u011957758/article/details/96633984?ops_request_misc=%257B%2522request%255Fid%2522%253A%2522164515668616781683951530%2522%252C%2522scm%2522%253A%252220140713.130102334..%2522%257D&request_id=164515668616781683951530&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~all~sobaiduend~default-1-96633984.pc_search_result_positive&utm_term=golang+syncmap&spm=1018.2226.3001.4187)
- Go 如何高效地拼接字符串?

## 其他资源
- [github他人收集](https://github.com/KeKe-Li/data-structures-questions/blob/master/src/chapter05/golang.01.md#Go%E4%B8%AD%E7%9A%84%E9%94%81%E5%A6%82%E4%BD%95%E5%AE%9E%E7%8E%B0)
## 无锁设计
- 怎么设计一个无锁的pool

## GMP进阶
- GMP什么时候回创建新的M，创建有数量限制吗
- 阻塞GM绑定之后就回去寻找新的M吗
- goroutine是什么，怎么执行
    - goroutine是比线程还轻量的执行单位，是用户层面的
    - 一个gourontine大约3kb左右
    - 上下文切换成本小
    - goroutine GMP模型，M：N模型
    - 如果可以聊聊goroutine的生老病死
- goroutine切换的原理
    - 网络io阻塞主动切换，cpu占用时间过长信号切换，锁，channel
- GO的GPM模型?P和M的数量怎么决定？如果在K8S容器部署，P和M又会有什么不同？
- GMP模型？全局队列没有g了，怎么办
    - 去其他p的g队列偷取
- goroutine的亲缘性怎么体现出来
- Golang中需要使用协程池吗？为什么？
- goroutine为啥不设置id
- 线程模型有哪些？为什么 Go Scheduler 需要实现 M:N 的方案？Go Scheduler 由哪些元素构成呢？

## 测试
- go项目如何左覆盖率测试
    - go test ./... -v -gcflags=-l -p 1 -coverprofile=coverage.out

## Runtime
- runtime.GOMAXPROCS(0)表示什么？为什么要这么用？

## Interface
- interface内部实现原理
- reflect的用途
- 指针实现接口和结构体实现接口有什么区别？
    - [refer](https://mp.weixin.qq.com/s/g-D_eVh-8JaIoRne09bJ3Q)

## 其他问题
- 当G发送阻塞时,G和M和P是如何变化的
- 什么是自旋,M为什么要自旋
- map使用时有什么要注意的

