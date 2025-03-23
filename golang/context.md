# Go语言的Context包

## 基本概念

Context（上下文）是Go语言中用于跨API边界和进程间传递截止日期、取消信号和其他请求范围值的标准方式。Context包主要用于控制多个goroutine之间的协作，特别是处理请求的超时、取消以及传递关键请求范围的值。

### 为什么需要Context

在构建现代分布式系统和微服务架构时，请求通常需要经过多个服务或多个goroutine处理。在这个过程中常常需要：

1. **传递请求范围的值**：如请求ID、认证信息等
2. **传递取消信号**：如用户主动取消请求、超时等
3. **设置截止时间**：限制请求的最长处理时间
4. **协调多个goroutine**：确保它们在完成工作或被取消时能够正确清理资源

Context包正是为解决这些问题而设计的，它提供了一种优雅的方式来处理这些跨API边界和goroutine的信号传递。

## Context接口

Go的Context定义了以下接口：

```go
type Context interface {
    // Deadline返回完成工作的时间，如果没有设置截止时间则ok=false
    Deadline() (deadline time.Time, ok bool)
    
    // Done返回一个通道，当工作完成或被取消时关闭
    Done() <-chan struct{}
    
    // Err说明为什么工作被取消
    // 如果Done尚未关闭，Err返回nil
    // 如果Done已关闭，Err返回非nil错误：
    // - 如果是因为取消，返回Canceled
    // - 如果是因为超时，返回DeadlineExceeded
    Err() error
    
    // Value返回与键关联的值，如果没有，则返回nil
    Value(key interface{}) interface{}
}
```

## Context的创建

Context包提供了几种创建Context的函数：

### 1. Background和TODO

```go
// 返回一个非nil的空Context，用作整个程序的顶层Context
func Background() Context

// 与Background类似，当尚不清楚使用哪个Context时使用TODO
func TODO() Context
```

`Background` 和 `TODO` 是最基本的非nil Context，区别在于：

- `Background` 通常用作主函数、初始化和测试的顶层Context
- `TODO` 用于尚未确定具体Context应该使用哪种的情况，是一个占位符

### 2. WithCancel

```go
// 返回一个带有新Done通道的parent的副本
// 当调用返回的cancel函数或当关闭parent.Done通道时，返回Context的Done通道将被关闭
func WithCancel(parent Context) (ctx Context, cancel CancelFunc)
```

`WithCancel` 创建一个可以手动取消的Context：

```go
ctx, cancel := context.WithCancel(context.Background())
defer cancel() // 确保所有路径都会调用cancel

// 在某个条件下取消context
if someCondition {
    cancel()
}
```

### 3. WithDeadline和WithTimeout

```go
// 返回一个带有截止时间的parent的副本
// 当截止时间到期、调用返回的cancel函数或关闭parent.Done通道时，返回的Context的Done通道会关闭
func WithDeadline(parent Context, deadline time.Time) (Context, CancelFunc)

// 相当于WithDeadline(parent, time.Now().Add(timeout))
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)
```

`WithDeadline` 和 `WithTimeout` 创建带有时间限制的Context：

```go
// 设置截止时间
deadline := time.Now().Add(50 * time.Millisecond)
ctx, cancel := context.WithDeadline(context.Background(), deadline)
defer cancel()

// 或者设置超时时间
ctx, cancel := context.WithTimeout(context.Background(), 100 * time.Millisecond)
defer cancel()
```

### 4. WithValue

```go
// 返回一个带有键值对的parent副本
// 通过WithValue传递的值只适合请求范围内的数据，不应该用于传递可选参数
func WithValue(parent Context, key, val interface{}) Context
```

`WithValue` 创建带有键值对的Context：

```go
type contextKey string

ctx := context.WithValue(context.Background(), contextKey("user-id"), "12345")
// 稍后从context中获取值
userID := ctx.Value(contextKey("user-id")).(string)
```

## Context的使用模式

### 1. 传递请求范围的值

```go
func ProcessRequest(ctx context.Context, r *Request) {
    // 从context获取请求ID或创建新ID
    reqID, ok := ctx.Value(RequestIDKey).(string)
    if !ok {
        reqID = generateRequestID()
        ctx = context.WithValue(ctx, RequestIDKey, reqID)
    }
    
    // 处理请求，传递带有请求ID的context
    results, err := processStep1(ctx, r)
    // ...
}

func processStep1(ctx context.Context, r *Request) (Results, error) {
    // 可以从context获取请求ID
    reqID := ctx.Value(RequestIDKey).(string)
    log.Printf("Processing request %s in step1", reqID)
    // ...
}
```

### 2. 控制请求超时和取消

```go
func handleSearch(w http.ResponseWriter, r *http.Request) {
    // 为请求创建超时上下文
    ctx, cancel := context.WithTimeout(r.Context(), 100*time.Millisecond)
    defer cancel() // 无论search返回什么，都确保调用cancel
    
    results, err := search(ctx, r.FormValue("query"))
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(results)
}

func search(ctx context.Context, query string) (Results, error) {
    // 创建一个channel接收结果
    resultCh := make(chan Results)
    errCh := make(chan error)
    
    go func() {
        // 执行实际的搜索操作
        results, err := performSearch(query)
        if err != nil {
            errCh <- err
            return
        }
        resultCh <- results
    }()
    
    // 等待结果、错误或超时
    select {
    case results := <-resultCh:
        return results, nil
    case err := <-errCh:
        return nil, err
    case <-ctx.Done():
        return nil, ctx.Err() // 可能是DeadlineExceeded或Canceled
    }
}
```

### 3. 取消多个Goroutine

```go
func fetchAllData(ctx context.Context, urls []string) ([]Result, error) {
    // 为所有goroutine创建一个子上下文
    ctx, cancel := context.WithCancel(ctx)
    defer cancel()
    
    results := make([]Result, len(urls))
    errCh := make(chan error, 1)  // 只需要第一个错误
    done := make(chan struct{})
    
    // 启动goroutine获取每个URL
    for i, url := range urls {
        go func(i int, url string) {
            result, err := fetchData(ctx, url)
            if err != nil {
                select {
                case errCh <- err:
                    cancel() // 取消所有其他goroutine
                default:
                    // 已有错误被发送
                }
                return
            }
            results[i] = result
            
            // 检查是否所有goroutine都完成了
            if i == len(urls)-1 {
                close(done)
            }
        }(i, url)
    }
    
    // 等待完成或出错或父上下文取消
    select {
    case <-done:
        return results, nil
    case err := <-errCh:
        return nil, err
    case <-ctx.Done():
        return nil, ctx.Err()
    }
}

func fetchData(ctx context.Context, url string) (Result, error) {
    req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
    if err != nil {
        return Result{}, err
    }
    
    resp, err := http.DefaultClient.Do(req)
    if err != nil {
        return Result{}, err
    }
    defer resp.Body.Close()
    
    // ...处理响应
    return Result{/* ... */}, nil
}
```

## Context的最佳实践

### 1. 将Context作为第一个参数传递

按照Go的约定，Context应该作为函数的第一个参数传递：

```go
// 正确
func ProcessTask(ctx context.Context, task Task) error {
    // ...
}

// 不正确
func ProcessTask(task Task, ctx context.Context) error {
    // ...
}
```

### 2. 不要将Context存储在结构体中

Context应该通过函数调用链来传递，而不是存储在结构体中：

```go
// 错误做法
type Service struct {
    ctx context.Context
    // ...
}

// 正确做法
type Service struct {
    // ...没有context字段
}

func (s *Service) ProcessTask(ctx context.Context) {
    // 使用传入的context
}
```

例外情况：专门用于处理请求的结构体可以存储Context，例如带有Context的请求对象。

### 3. 总是检查ctx.Err()

在循环或长时间运行的操作中，应定期检查ctx.Err()：

```go
func ProcessLargeData(ctx context.Context, data []Item) error {
    for i, item := range data {
        // 定期检查上下文是否已取消
        if i%100 == 0 {
            select {
            case <-ctx.Done():
                return ctx.Err()
            default:
                // 继续处理
            }
        }
        
        // 处理数据项
        if err := processItem(ctx, item); err != nil {
            return err
        }
    }
    return nil
}
```

### 4. 不要传递nil Context

如果不确定要使用哪个Context，请使用context.TODO()：

```go
// 错误做法
func processRequest(ctx context.Context) {
    if userAuthenticated {
        process(ctx)
    } else {
        process(nil) // 不要传递nil
    }
}

// 正确做法
func processRequest(ctx context.Context) {
    if userAuthenticated {
        process(ctx)
    } else {
        process(context.TODO())
    }
}
```

### 5. 使用WithValue时使用自定义类型作为键

为了避免键冲突，应使用自定义类型而不是内置类型作为context.WithValue的键：

```go
// 定义自定义类型
type contextKey string

// 定义常量
const (
    userIDKey contextKey = "user-id"
    authTokenKey contextKey = "auth-token"
)

// 使用
ctx = context.WithValue(ctx, userIDKey, "12345")
```

### 6. 及时调用cancel函数

当创建带有取消功能的Context时，确保在所有路径上都调用cancel函数，通常使用defer：

```go
ctx, cancel := context.WithTimeout(parentCtx, 5*time.Second)
defer cancel() // 无论函数如何返回，都会调用cancel

// 处理请求...
```

### 7. 不要滥用Context.Value

Context.Value主要用于传递请求范围的数据，不应用于传递函数的可选参数：

```go
// 不推荐：使用context传递函数参数
func ProcessData(ctx context.Context) {
    // 从context获取选项
    concurrency := ctx.Value(concurrencyKey).(int)
    // ...
}

// 推荐：显式传递参数或使用选项模式
func ProcessData(ctx context.Context, concurrency int) {
    // ...
}

// 或者
type Options struct {
    Concurrency int
    // ...其他选项
}

func ProcessDataWithOptions(ctx context.Context, opts Options) {
    // ...
}
```

## Context的内部实现

理解Context的内部实现有助于更好地使用它。以下是简化的Context实现原理：

### 基础Context类型

```go
// 所有Context实现的基础
type emptyCtx int

func (emptyCtx) Deadline() (deadline time.Time, ok bool) {
    return
}

func (emptyCtx) Done() <-chan struct{} {
    return nil
}

func (emptyCtx) Err() error {
    return nil
}

func (emptyCtx) Value(key interface{}) interface{} {
    return nil
}
```

Background()和TODO()返回的是这种emptyCtx类型的值。

### cancelCtx

```go
type cancelCtx struct {
    Context

    mu       sync.Mutex            // 保护以下字段
    done     chan struct{}         // 在第一次取消时懒创建，然后关闭
    children map[canceler]struct{} // 子取消器集合
    err      error                 // 在done关闭后设置为非nil
}

func (c *cancelCtx) Done() <-chan struct{} {
    c.mu.Lock()
    if c.done == nil {
        c.done = make(chan struct{})
    }
    d := c.done
    c.mu.Unlock()
    return d
}

func (c *cancelCtx) Err() error {
    c.mu.Lock()
    err := c.err
    c.mu.Unlock()
    return err
}

func (c *cancelCtx) cancel(removeFromParent bool, err error) {
    // ...关闭done通道，设置错误，取消所有子context
}
```

### timerCtx

```go
type timerCtx struct {
    cancelCtx
    timer *time.Timer // 在超时时取消context
    deadline time.Time
}

func (c *timerCtx) Deadline() (deadline time.Time, ok bool) {
    return c.deadline, true
}

func (c *timerCtx) cancel(removeFromParent bool, err error) {
    c.cancelCtx.cancel(removeFromParent, err)
    c.timer.Stop()
}
```

### valueCtx

```go
type valueCtx struct {
    Context
    key, val interface{}
}

func (c *valueCtx) Value(key interface{}) interface{} {
    if c.key == key {
        return c.val
    }
    return c.Context.Value(key)
}
```

## Context的性能考虑

### 1. Context链的长度

每次调用WithValue/WithCancel/WithTimeout都会创建一个新的Context实例，形成一个链条。过长的链条可能影响Value查找的性能：

```go
ctx := context.Background()
ctx = context.WithValue(ctx, key1, val1)
ctx = context.WithValue(ctx, key2, val2)
// ...
ctx = context.WithValue(ctx, keyN, valN) // 查找keyM需要遍历整个链条
```

建议尽量保持Context链的合理长度。

### 2. 避免大对象作为Value

Context.Value不适合存储大型数据结构，应该存储轻量级的请求范围数据如ID、令牌等：

```go
// 不推荐：存储大型对象
ctx = context.WithValue(ctx, "data", largeDataStructure)

// 推荐：存储引用或ID
ctx = context.WithValue(ctx, "data-id", dataID)
```

### 3. 及时取消Context

不再需要的Context应该及时取消，以释放资源：

```go
ctx, cancel := context.WithTimeout(parent, timeout)
// 处理完成后立即取消，不要等到defer执行
cancel()
```

## Context与标准库的集成

Go标准库中许多包都支持Context：

### 1. net/http

```go
// 服务端：从请求获取Context
func handler(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    // ...使用ctx
}

// 客户端：创建带有Context的请求
req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
```

### 2. database/sql

```go
// 带有超时的数据库操作
ctx, cancel := context.WithTimeout(context.Background(), 1*time.Second)
defer cancel()

rows, err := db.QueryContext(ctx, "SELECT * FROM users WHERE id = ?", id)
```

### 3. os/exec

```go
// 带有超时的命令执行
ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
defer cancel()

cmd := exec.CommandContext(ctx, "sleep", "100")
err := cmd.Run() // 会在30秒后自动取消
```

## 实际应用示例

### 1. HTTP服务中的Context传播

```go
func handler(w http.ResponseWriter, r *http.Request) {
    // 从请求获取Context
    ctx := r.Context()
    
    // 添加请求ID
    requestID := uuid.New().String()
    ctx = context.WithValue(ctx, RequestIDKey, requestID)
    
    // 设置处理超时
    ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
    defer cancel()
    
    // 处理请求
    result, err := processRequest(ctx, r.FormValue("query"))
    if err != nil {
        if errors.Is(err, context.DeadlineExceeded) {
            http.Error(w, "Request timed out", http.StatusGatewayTimeout)
        } else if errors.Is(err, context.Canceled) {
            http.Error(w, "Request canceled", http.StatusBadRequest)
        } else {
            http.Error(w, err.Error(), http.StatusInternalServerError)
        }
        return
    }
    
    // 返回结果
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(result)
}

func processRequest(ctx context.Context, query string) (Result, error) {
    // 使用请求ID进行日志记录
    requestID := ctx.Value(RequestIDKey).(string)
    log.Printf("[%s] Processing request: %s", requestID, query)
    
    // 并行查询多个数据源
    resultCh := make(chan Result, 1)
    errCh := make(chan error, 1)
    
    go func() {
        result, err := fetchData(ctx, query)
        if err != nil {
            errCh <- err
            return
        }
        resultCh <- result
    }()
    
    // 等待结果或上下文取消
    select {
    case result := <-resultCh:
        return result, nil
    case err := <-errCh:
        return Result{}, err
    case <-ctx.Done():
        return Result{}, ctx.Err()
    }
}
```

### 2. 优雅取消长时间运行的操作

```go
func ProcessLargeFile(ctx context.Context, filename string) error {
    file, err := os.Open(filename)
    if err != nil {
        return err
    }
    defer file.Close()
    
    reader := bufio.NewReader(file)
    
    lineCount := 0
    for {
        // 定期检查是否应该取消
        if lineCount%1000 == 0 {
            select {
            case <-ctx.Done():
                return ctx.Err()
            default:
                // 继续处理
            }
        }
        
        line, err := reader.ReadString('\n')
        if err != nil {
            if err == io.EOF {
                break
            }
            return err
        }
        
        // 处理行
        if err := processLine(ctx, line); err != nil {
            return err
        }
        
        lineCount++
    }
    
    return nil
}

func processLine(ctx context.Context, line string) error {
    // 这里可能是一个耗时的操作
    select {
    case <-ctx.Done():
        return ctx.Err()
    case <-time.After(10 * time.Millisecond):
        // 模拟处理时间
        return nil
    }
}
```

### 3. 多层服务调用中的Context传播

```go
// API层
func handleAPI(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    
    // 从请求头提取跟踪ID，如果没有则创建新的
    traceID := r.Header.Get("X-Trace-ID")
    if traceID == "" {
        traceID = generateTraceID()
    }
    ctx = context.WithValue(ctx, TraceIDKey, traceID)
    
    // 设置超时
    ctx, cancel := context.WithTimeout(ctx, 2*time.Second)
    defer cancel()
    
    // 调用服务层
    result, err := serviceLayer.ProcessRequest(ctx, r.URL.Query())
    if err != nil {
        handleError(w, err, traceID)
        return
    }
    
    // 返回结果
    w.Header().Set("X-Trace-ID", traceID)
    json.NewEncoder(w).Encode(result)
}

// 服务层
type ServiceLayer struct {
    repo Repository
}

func (s *ServiceLayer) ProcessRequest(ctx context.Context, params url.Values) (Result, error) {
    // 提取跟踪ID
    traceID := ctx.Value(TraceIDKey).(string)
    log.Printf("[%s] Service processing request with params: %v", traceID, params)
    
    // 增加子标记
    spanID := generateSpanID()
    ctx = context.WithValue(ctx, SpanIDKey, spanID)
    
    // 调用仓库层
    data, err := s.repo.FetchData(ctx, params.Get("id"))
    if err != nil {
        return Result{}, fmt.Errorf("service error: %w", err)
    }
    
    // 处理数据
    return processData(data), nil
}

// 仓库层
type Repository struct {
    db *sql.DB
}

func (r *Repository) FetchData(ctx context.Context, id string) (Data, error) {
    // 提取跟踪和span ID
    traceID := ctx.Value(TraceIDKey).(string)
    spanID := ctx.Value(SpanIDKey).(string)
    log.Printf("[%s:%s] Repository fetching data for ID: %s", traceID, spanID, id)
    
    // 执行数据库查询
    row := r.db.QueryRowContext(ctx, "SELECT * FROM data WHERE id = ?", id)
    
    // 解析结果
    var data Data
    if err := row.Scan(&data.ID, &data.Value); err != nil {
        if errors.Is(err, context.DeadlineExceeded) {
            log.Printf("[%s:%s] Database query timed out", traceID, spanID)
        }
        return Data{}, fmt.Errorf("repository error: %w", err)
    }
    
    return data, nil
}
```

## 总结

Go的Context包提供了一个强大而优雅的解决方案，用于控制多个goroutine之间的协作以及跨API边界传递请求范围的值。Context的关键功能包括：

1. **请求范围值的传递**：通过WithValue创建携带键值对的Context
2. **取消信号的传递**：通过WithCancel创建可手动取消的Context
3. **截止时间/超时控制**：通过WithDeadline/WithTimeout限制操作时间
4. **多goroutine协调**：通过Done()通道和Err()方法实现协调

使用Context时，应遵循以下最佳实践：

1. 将Context作为第一个参数传递
2. 不要将Context存储在结构体中
3. 定期检查ctx.Err()
4. 不要传递nil Context
5. 使用自定义类型作为WithValue的键
6. 及时调用cancel函数
7. 不要滥用Context.Value传递可选参数

通过正确使用Context，可以构建更健壮、更可维护的并发程序，特别是在微服务架构和大型系统中，它能提供一致的请求追踪和控制机制。 