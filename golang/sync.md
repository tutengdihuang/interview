# Go语言中的同步原语

## 基本概念

Go语言提供了丰富的同步原语，用于协调goroutine之间的执行和通信。这些同步工具位于标准库的`sync`和`sync/atomic`包中，提供了从基本的互斥锁到复杂的并发数据结构的各种功能。

虽然Go推荐通过通道（channel）进行通信和同步（"不要通过共享内存来通信，而是通过通信来共享内存"），但在某些场景下，使用同步原语可能更简单高效。

## Mutex（互斥锁）

互斥锁用于保护共享资源，确保同一时间只有一个goroutine可以访问该资源。

### 基本用法

```go
import "sync"

type Counter struct {
    mu    sync.Mutex
    value int
}

func (c *Counter) Increment() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.value++
}

func (c *Counter) Value() int {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.value
}
```

### 优缺点

**优点**：
- 简单直观
- 适用于保护短时间访问的共享资源

**缺点**：
- 可能导致死锁
- 持有锁时间过长会降低并发性能

### 最佳实践

1. **尽量缩小临界区**：
   ```go
   // 不好的写法
   mutex.Lock()
   // 大量计算
   // 访问共享资源
   mutex.Unlock()
   
   // 好的写法
   // 大量计算
   mutex.Lock()
   // 访问共享资源
   mutex.Unlock()
   ```

2. **使用defer确保解锁**：
   ```go
   mutex.Lock()
   defer mutex.Unlock()
   ```

3. **避免嵌套锁**，容易导致死锁：
   ```go
   // 危险：可能死锁
   func (c *Container) Method1() {
       c.mu.Lock()
       defer c.mu.Unlock()
       c.Method2()
   }
   
   func (c *Container) Method2() {
       c.mu.Lock() // 死锁！已经持有锁了
       defer c.mu.Unlock()
       // ...
   }
   ```

## RWMutex（读写互斥锁）

读写锁允许多个读操作并发执行，但写操作是互斥的。

### 基本用法

```go
type DataStore struct {
    rwmu  sync.RWMutex
    data  map[string]string
}

// 写操作需要写锁
func (ds *DataStore) Set(key, value string) {
    ds.rwmu.Lock()
    defer ds.rwmu.Unlock()
    ds.data[key] = value
}

// 读操作使用读锁
func (ds *DataStore) Get(key string) (string, bool) {
    ds.rwmu.RLock()
    defer ds.rwmu.RUnlock()
    value, ok := ds.data[key]
    return value, ok
}
```

### 适用场景

当共享资源的读操作远多于写操作时，读写锁比普通互斥锁更高效。

### 性能考虑

- 读写锁实现更复杂，有额外开销
- 只有在读操作明显多于写操作的场景才有优势
- 短时间操作可能普通互斥锁反而更快

## WaitGroup

WaitGroup用于等待一组goroutine完成执行。它维护一个计数器，当计数器归零时，等待的goroutine被释放。

### 基本用法

```go
func main() {
    var wg sync.WaitGroup
    
    for i := 0; i < 5; i++ {
        wg.Add(1) // 增加计数器
        go func(id int) {
            defer wg.Done() // 完成时减少计数器
            fmt.Printf("Worker %d starting\n", id)
            time.Sleep(time.Second)
            fmt.Printf("Worker %d done\n", id)
        }(i)
    }
    
    wg.Wait() // 阻塞直到计数器归零
    fmt.Println("All workers done")
}
```

### 注意事项

1. **Add必须在goroutine创建之前调用**：
   ```go
   // 正确
   wg.Add(1)
   go func() {
       defer wg.Done()
       // ...
   }()
   
   // 错误：可能在Add之前Done
   go func() {
       defer wg.Done()
       // ...
   }()
   wg.Add(1)
   ```

2. **不要复制WaitGroup**：应通过指针传递
   ```go
   // 正确
   func worker(wg *sync.WaitGroup) {
       defer wg.Done()
       // ...
   }
   
   // 错误：复制了WaitGroup
   func worker(wg sync.WaitGroup) {
       defer wg.Done()
       // ...
   }
   ```

3. **确保Add和Done数量相等**，否则可能永久阻塞或报错

## Once

Once确保某个函数只执行一次，即使在多个goroutine中调用。

### 基本用法

```go
var instance *Singleton
var once sync.Once

func GetInstance() *Singleton {
    once.Do(func() {
        instance = &Singleton{}
    })
    return instance
}
```

### 应用场景

1. **单例模式**：如上例所示
2. **一次性初始化**：
   ```go
   type Service struct {
       once   sync.Once
       config *Config
   }
   
   func (s *Service) Init() {
       s.once.Do(func() {
           s.config = loadConfig()
       })
   }
   ```

### 特性

- **线程安全**：可在多个goroutine中安全调用
- **只执行一次**：即使Do内部函数panic，也算执行过了
- **死锁安全**：即使回调中又调用了同一个Once的Do，也不会死锁（但函数只执行一次）

## Cond（条件变量）

条件变量用于goroutine等待特定条件满足。当条件满足时，一个或所有等待的goroutine被唤醒。

### 基本用法

```go
func main() {
    var mu sync.Mutex
    cond := sync.NewCond(&mu)
    
    queue := make([]int, 0, 10)
    
    // 消费者
    go func() {
        for {
            mu.Lock()
            for len(queue) == 0 {
                cond.Wait() // 释放锁并等待唤醒
            }
            item := queue[0]
            queue = queue[1:]
            fmt.Println("Consumed:", item)
            mu.Unlock()
            time.Sleep(time.Second) // 模拟处理时间
        }
    }()
    
    // 生产者
    for i := 0; i < 10; i++ {
        mu.Lock()
        queue = append(queue, i)
        fmt.Println("Produced:", i)
        cond.Signal() // 唤醒一个等待的goroutine
        mu.Unlock()
        time.Sleep(500 * time.Millisecond)
    }
    
    time.Sleep(10 * time.Second) // 让示例运行一段时间
}
```

### 方法说明

- **Wait()**：释放锁并等待唤醒，唤醒后重新获取锁
- **Signal()**：唤醒一个等待的goroutine
- **Broadcast()**：唤醒所有等待的goroutine

### 使用场景

1. **经典生产者-消费者模式**
2. **资源池**：等待资源可用
3. **工作队列**：等待任务到达

### 注意事项

1. **必须持有锁才能调用Wait、Signal和Broadcast**
2. **Wait会自动释放锁**，被唤醒后自动重新获取锁
3. **虚假唤醒**：Wait返回后应再次检查条件是否满足

## Pool

Pool用于存储和复用临时对象，减少内存分配和GC压力。

### 基本用法

```go
var bufferPool = sync.Pool{
    New: func() interface{} {
        return new(bytes.Buffer)
    },
}

func main() {
    // 从池中获取对象
    buf := bufferPool.Get().(*bytes.Buffer)
    buf.Reset() // 重置状态以便复用
    
    // 使用buf
    buf.WriteString("Hello, World!")
    fmt.Println(buf.String())
    
    // 使用完归还池
    bufferPool.Put(buf)
}
```

### 应用场景

1. **临时对象池**：如上例中的Buffer
2. **连接池**：数据库连接、HTTP客户端等
3. **工作池**：预分配worker对象

### 重要特性

- **自动清理**：GC可能在任何时候清空Pool
- **非确定性**：Get不保证返回最近放入的对象
- **无大小限制**：Pool不限制对象数量，需自行控制

## Map

sync.Map是一种并发安全的map，专为高并发读、低并发写的场景设计。

### 基本用法

```go
var m sync.Map

func main() {
    // 存储
    m.Store("key1", "value1")
    m.Store("key2", "value2")
    
    // 获取
    value, ok := m.Load("key1")
    if ok {
        fmt.Println("Found:", value)
    }
    
    // 删除
    m.Delete("key1")
    
    // 如果不存在则存储
    actualValue, loaded := m.LoadOrStore("key3", "value3")
    if !loaded {
        fmt.Println("Stored new value:", actualValue)
    }
    
    // 遍历
    m.Range(func(key, value interface{}) bool {
        fmt.Println(key, ":", value)
        return true // 继续遍历
    })
}
```

### 性能特点

- **适合读多写少**的场景
- **不需要初始化**，可直接使用
- **空间换时间**：内部使用两个map和原子操作
- 不适合有大量写操作的场景

### 使用建议

1. **不要用于所有场景**：只有在确实需要并发安全且读多写少时使用
2. **类型安全**：考虑封装提供类型安全的API

## 原子操作

原子操作提供了最基本的同步原语，确保在并发环境中安全修改单个值。

### 基本用法

```go
import (
    "fmt"
    "sync/atomic"
    "time"
)

func main() {
    var counter int64 = 0
    
    // 并发递增
    for i := 0; i < 100; i++ {
        go func() {
            atomic.AddInt64(&counter, 1)
        }()
    }
    
    time.Sleep(time.Second) // 等待goroutine完成
    fmt.Println("Final count:", atomic.LoadInt64(&counter))
    
    // 比较并交换
    var value int64 = 100
    swapped := atomic.CompareAndSwapInt64(&value, 100, 200)
    fmt.Println("Swapped:", swapped, "New value:", atomic.LoadInt64(&value))
}
```

### 支持的主要操作

- **Load**：原子读取
- **Store**：原子存储
- **Add**：原子加法
- **Swap**：原子交换
- **CompareAndSwap**：比较并交换

### 适用场景

1. **计数器**：如上例
2. **标志位**：设置和检查状态
3. **后备方案**：简单场景无需加锁

### Value类型

`atomic.Value`提供泛型原子存储：

```go
var config atomic.Value

// 存储任意类型
config.Store(Config{Timeout: 100})

// 读取并类型断言
currentConfig := config.Load().(Config)
```

适合读多写少的配置热更新场景。

## 同步原语的选择

选择哪种同步原语，取决于具体场景：

1. **通道（Channel）**：
   - 在goroutine间传递数据
   - 实现信号通知
   - CSP模式的通信

2. **互斥锁（Mutex/RWMutex）**：
   - 保护共享资源的简单访问
   - 需要精确控制临界区

3. **原子操作**：
   - 简单计数器或标志位
   - 性能关键场景
   - 最小粒度的同步

4. **WaitGroup**：
   - 等待一组goroutine完成
   - 并行工作的协调

5. **Cond**：
   - 等待条件满足
   - 生产者-消费者模式

6. **Once**：
   - 一次性初始化
   - 单例模式实现

7. **Pool**：
   - 对象复用
   - 减少GC压力

8. **Map**：
   - 并发读多写少的map
   - 无须显式锁的并发字典

## 常见并发模式

### 工作池模式

```go
func worker(id int, jobs <-chan int, results chan<- int, wg *sync.WaitGroup) {
    defer wg.Done()
    for job := range jobs {
        fmt.Printf("Worker %d processing job %d\n", id, job)
        time.Sleep(time.Second) // 模拟工作
        results <- job * 2      // 发送结果
    }
}

func main() {
    jobs := make(chan int, 100)
    results := make(chan int, 100)
    
    // 启动工作池
    var wg sync.WaitGroup
    for w := 1; w <= 3; w++ {
        wg.Add(1)
        go worker(w, jobs, results, &wg)
    }
    
    // 发送工作
    for j := 1; j <= 9; j++ {
        jobs <- j
    }
    close(jobs)
    
    // 等待所有工作完成
    go func() {
        wg.Wait()
        close(results)
    }()
    
    // 收集结果
    for result := range results {
        fmt.Println("Result:", result)
    }
}
```

### 并发控制：限制并发数

```go
func process(item int) {
    fmt.Println("Processing:", item)
    time.Sleep(time.Second)
}

func main() {
    items := make([]int, 100)
    for i := range items {
        items[i] = i
    }
    
    // 控制并发数
    concurrency := 5
    sem := make(chan struct{}, concurrency)
    
    var wg sync.WaitGroup
    for _, item := range items {
        wg.Add(1)
        sem <- struct{}{} // 获取信号量
        
        go func(item int) {
            defer func() {
                <-sem // 释放信号量
                wg.Done()
            }()
            process(item)
        }(item)
    }
    
    wg.Wait()
    fmt.Println("All items processed")
}
```

### 超时控制

```go
func longRunningTask() (string, error) {
    time.Sleep(2 * time.Second)
    return "Task result", nil
}

func main() {
    // 创建带超时的上下文
    ctx, cancel := context.WithTimeout(context.Background(), 1*time.Second)
    defer cancel()
    
    resultCh := make(chan string)
    errCh := make(chan error)
    
    go func() {
        result, err := longRunningTask()
        if err != nil {
            errCh <- err
            return
        }
        resultCh <- result
    }()
    
    select {
    case result := <-resultCh:
        fmt.Println("Success:", result)
    case err := <-errCh:
        fmt.Println("Error:", err)
    case <-ctx.Done():
        fmt.Println("Timeout:", ctx.Err())
    }
}
```

## 同步原语的性能考虑

### 性能比较

从高性能到低性能排序（一般情况下）：
1. 无锁设计 / CSP模式
2. 原子操作
3. 互斥锁/读写锁
4. 通道操作
5. 复杂同步原语（Pool, Map等）

### 避免常见陷阱

1. **细粒度锁 vs 粗粒度锁**：
   - 细粒度锁可提高并发度，但增加开销和死锁风险
   - 粗粒度锁实现简单，但可能造成瓶颈

2. **锁争用**：
   - 高并发下大量goroutine竞争同一把锁会导致性能下降
   - 考虑分片锁（Sharded Lock）减少争用

3. **GOMAXPROCS设置**：
   - 默认使用所有CPU核心
   - 不必过度设置，Go运行时会自动调度

4. **过度同步**：
   - 只在必要时使用同步原语
   - 优先使用无锁数据结构和不可变数据

## 同步原语的实际案例

### 配置热更新

```go
type Config struct {
    Timeout   int
    CacheTTL  int
    MaxConn   int
    UpdatedAt time.Time
}

var currentConfig atomic.Value

func loadConfig() Config {
    // 模拟从文件或远程加载配置
    return Config{
        Timeout:   1000,
        CacheTTL:  300,
        MaxConn:   100,
        UpdatedAt: time.Now(),
    }
}

func init() {
    // 初始加载
    config := loadConfig()
    currentConfig.Store(config)
    
    // 定期更新
    go func() {
        for {
            time.Sleep(time.Minute)
            newConfig := loadConfig()
            currentConfig.Store(newConfig)
            fmt.Println("Config updated:", newConfig.UpdatedAt)
        }
    }()
}

func getConfig() Config {
    return currentConfig.Load().(Config)
}

func handleRequest() {
    config := getConfig() // 每次请求获取最新配置
    // 使用配置...
}
```

### 并发安全的缓存

```go
type Cache struct {
    mu    sync.RWMutex
    items map[string]Item
}

type Item struct {
    Value      interface{}
    Expiration int64
}

func NewCache() *Cache {
    return &Cache{
        items: make(map[string]Item),
    }
}

func (c *Cache) Set(key string, value interface{}, duration time.Duration) {
    var expiration int64
    if duration > 0 {
        expiration = time.Now().Add(duration).UnixNano()
    }
    
    c.mu.Lock()
    defer c.mu.Unlock()
    
    c.items[key] = Item{
        Value:      value,
        Expiration: expiration,
    }
}

func (c *Cache) Get(key string) (interface{}, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    
    item, found := c.items[key]
    if !found {
        return nil, false
    }
    
    // 检查是否过期
    if item.Expiration > 0 && item.Expiration < time.Now().UnixNano() {
        return nil, false
    }
    
    return item.Value, true
}

func (c *Cache) Delete(key string) {
    c.mu.Lock()
    defer c.mu.Unlock()
    delete(c.items, key)
}

// 清理过期项
func (c *Cache) StartJanitor(interval time.Duration) {
    ticker := time.NewTicker(interval)
    go func() {
        for range ticker.C {
            c.mu.Lock()
            now := time.Now().UnixNano()
            for k, v := range c.items {
                if v.Expiration > 0 && v.Expiration < now {
                    delete(c.items, k)
                }
            }
            c.mu.Unlock()
        }
    }()
}
```

### 控制API请求速率

```go
type RateLimiter struct {
    mu       sync.Mutex
    last     time.Time
    rate     time.Duration
    allowance float64
    max      float64
}

func NewRateLimiter(rate int, per time.Duration) *RateLimiter {
    r := &RateLimiter{
        rate: per / time.Duration(rate),
        allowance: float64(rate),
        max: float64(rate),
        last: time.Now(),
    }
    return r
}

func (r *RateLimiter) Allow() bool {
    r.mu.Lock()
    defer r.mu.Unlock()
    
    now := time.Now()
    elapsed := now.Sub(r.last)
    r.last = now
    
    // 计算新的配额
    r.allowance += float64(elapsed) / float64(r.rate)
    if r.allowance > r.max {
        r.allowance = r.max
    }
    
    // 检查是否允许请求
    if r.allowance < 1.0 {
        return false
    }
    
    r.allowance -= 1.0
    return true
}

// 使用示例
func handleAPIRequest(limiter *RateLimiter) {
    if !limiter.Allow() {
        fmt.Println("Rate limit exceeded")
        return
    }
    
    // 处理API请求
    fmt.Println("Request processed")
}
```

## 常见问题和解决方案

### 1. 死锁问题

**症状**：程序永久挂起，无响应

**原因**：
- 循环等待锁
- 忘记解锁
- 条件变量等待但从不唤醒

**解决方案**：
- 保持一致的锁定顺序
- 使用超时机制
- 使用`defer mu.Unlock()`确保解锁
- 使用Go死锁检测器：`GODEBUG=schedtrace=1000,scheddetail=1`

### 2. 竞态条件

**症状**：不确定的行为，与时序相关的bug

**原因**：
- 未受保护的共享数据访问
- 读写竞争

**解决方案**：
- 使用race检测器：`go run -race` 或 `go test -race`
- 正确使用互斥锁或原子操作
- 遵循Go内存模型规则

### 3. 锁争用

**症状**：高并发下性能下降

**原因**：
- 大量goroutine争夺同一把锁

**解决方案**：
- 减少临界区大小
- 使用分片锁（Sharded Lock）
- 考虑无锁算法或数据结构
- 使用更细粒度的锁

### 4. 内存泄露

**症状**：内存使用持续增长

**原因**：
- goroutine泄露
- 同步原语未正确清理

**解决方案**：
- 使用context取消goroutine
- 确保WaitGroup正确使用
- 使用带缓冲通道防止goroutine阻塞
- 使用pprof检测泄露：`go tool pprof -alloc_space`

## 总结

Go的同步原语提供了丰富的并发编程工具，可以根据不同场景选择合适的同步机制：

1. **轻量级同步**：原子操作适用于简单计数器和标志位
2. **共享资源保护**：互斥锁和读写锁适用于临界区保护
3. **等待机制**：WaitGroup适用于等待goroutine完成
4. **条件协调**：Cond适用于等待条件满足
5. **对象复用**：Pool适用于减少GC压力
6. **并发容器**：Map提供无锁并发字典

在设计并发程序时，应遵循以下原则：

1. **优先使用CSP**：优先考虑通道（Channel）和无共享内存的设计
2. **最小化同步范围**：临界区应尽可能小
3. **避免嵌套锁**：防止死锁
4. **优先使用原子操作**：简单场景优先考虑原子操作而非互斥锁
5. **正确使用同步原语**：了解每种同步原语的适用场景和限制

最后，Go的并发不仅关乎性能，更关乎正确性。使用`go run -race`和`go test -race`检测竞态条件，使用分析工具如pprof诊断性能问题，确保并发程序的正确性和高效性。 