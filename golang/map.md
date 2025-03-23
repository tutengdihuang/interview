# Map的实现与应用

## 基本概念

Map是Go语言中的关联数组，也称为哈希表。它将键（key）与值（value）相关联，提供了高效的查找、插入和删除操作，是一种重要的数据结构。

Go语言中map的声明和使用：

```go
// 声明map
var m map[KeyType]ValueType

// 初始化map
m = make(map[KeyType]ValueType)

// 声明并初始化
m := make(map[KeyType]ValueType)

// 声明并初始化带初始容量的map
m := make(map[KeyType]ValueType, initialCapacity)

// 字面量初始化
m := map[KeyType]ValueType{
    key1: value1,
    key2: value2,
}
```

## 内部实现

### 数据结构

Go语言中的map是使用哈希表实现的，其底层数据结构在运行时定义（`src/runtime/map.go`）：

```go
// A header for a Go map.
type hmap struct {
    count     int    // map中的元素个数
    flags     uint8  // 状态标志
    B         uint8  // 桶（bucket）数量的对数 log_2
    noverflow uint16 // 溢出桶的近似数量
    hash0     uint32 // 哈希种子

    buckets    unsafe.Pointer // 桶数组的指针，数组大小为2^B
    oldbuckets unsafe.Pointer // 扩容时的旧桶数组
    nevacuate  uintptr        // 扩容时已迁移的桶数量

    extra *mapextra // 可选字段
}

// 桶结构
type bmap struct {
    tophash [bucketCnt]uint8 // 存储键的哈希值的高8位
    // 以下字段在编译期间添加
    // keys     [bucketCnt]keytype
    // values   [bucketCnt]valuetype
    // overflow *bmap
}
```

### 工作原理

1. **哈希计算**：
   - 对key使用哈希函数计算哈希值
   - 低位哈希值用于选择桶
   - 高位哈希值（tophash）存储在桶中，用于加速查找

2. **存储机制**：
   - 每个桶最多存储8个键值对
   - 当某个桶满了，会创建一个溢出桶，形成链表
   - 键值对按顺序紧凑存储在桶中

3. **查找过程**：
   - 计算key的哈希值，确定桶的位置
   - 在桶中比较tophash和key，找到匹配的键值对
   - 如果当前桶未找到，继续查找溢出桶

4. **扩容机制**：
   - 装载因子 > 6.5或溢出桶太多时触发扩容
   - 当元素数量/桶数量 > 6.5时，桶数量翻倍（增量扩容）
   - 当溢出桶过多但装载因子低时，不增加桶数量，只进行数据迁移和重新组织（等量扩容）
   - 扩容是渐进式的，每次map操作会迁移少量数据，而不是一次性迁移所有数据

## Map的特性

### 无序性

Map的迭代顺序是不确定的，即使多次迭代同一个map，每次迭代的顺序也可能不同。这是为了防止程序员依赖特定的迭代顺序。

```go
m := map[string]int{"a": 1, "b": 2, "c": 3}
// 多次迭代可能会产生不同的顺序
for k, v := range m {
    fmt.Println(k, v)
}
```

如果需要有序迭代，可以：
1. 将键存储在一个切片中
2. 对切片进行排序
3. 按照排序后的切片访问map

```go
m := map[string]int{"a": 1, "b": 2, "c": 3}
keys := make([]string, 0, len(m))
for k := range m {
    keys = append(keys, k)
}
sort.Strings(keys)
for _, k := range keys {
    fmt.Println(k, m[k])
}
```

### 并发不安全

标准的map类型不是并发安全的，多个goroutine并发读写map可能会导致panic或数据不一致。

```go
m := make(map[string]int)

// 并发写入可能导致panic
go func() {
    for i := 0; i < 1000; i++ {
        m[fmt.Sprintf("key%d", i)] = i
    }
}()

go func() {
    for i := 0; i < 1000; i++ {
        m[fmt.Sprintf("key%d", i)] = i * 10
    }
}()
```

解决方案：
1. 使用互斥锁保护map

```go
var (
    m   = make(map[string]int)
    mux sync.Mutex
)

func writeMap(key string, value int) {
    mux.Lock()
    defer mux.Unlock()
    m[key] = value
}

func readMap(key string) (int, bool) {
    mux.Lock()
    defer mux.Unlock()
    v, ok := m[key]
    return v, ok
}
```

2. 使用sync.Map（Go 1.9+）

```go
var m sync.Map

// 写入
m.Store("key", value)

// 读取
value, ok := m.Load("key")

// 删除
m.Delete("key")

// 如果不存在则写入
actual, loaded := m.LoadOrStore("key", value)

// 遍历
m.Range(func(key, value interface{}) bool {
    // 处理key, value
    return true // 返回true继续遍历，返回false停止遍历
})
```

sync.Map适用于以下场景：
- 只读或只写场景
- 读多写少场景
- 键值固定的场景

## 常见操作

### 基本操作

```go
// 创建map
m := make(map[string]int)

// 插入或更新
m["key"] = 42

// 获取值
v := m["key"]

// 获取值并检查键是否存在
v, ok := m["key"]
if ok {
    // 键存在
} else {
    // 键不存在
}

// 删除键值对
delete(m, "key")

// 获取map长度
length := len(m)

// 判断map是否为空
isEmpty := len(m) == 0
```

### 复合值类型

map的值可以是任何类型，包括结构体、切片、map等：

```go
// 值为结构体
type Person struct {
    Name string
    Age  int
}
people := make(map[string]Person)
people["alice"] = Person{Name: "Alice", Age: 30}

// 值为切片
groups := make(map[string][]string)
groups["fruits"] = []string{"apple", "banana", "orange"}

// 值为map
nested := make(map[string]map[string]int)
nested["outer"] = make(map[string]int)
nested["outer"]["inner"] = 42
```

### 清空map

Go语言没有内置的清空map的方法，可以：

1. 重新分配一个新map（推荐）

```go
m = make(map[KeyType]ValueType)
```

2. 遍历并删除所有键（不推荐，效率低）

```go
for k := range m {
    delete(m, k)
}
```

## 性能优化

### 预分配容量

当你知道map的大致容量时，在创建时指定初始容量可以减少扩容次数，提高性能：

```go
// 预分配容量为100的map
m := make(map[string]int, 100)
```

### 避免不必要的键查找

如果需要多次访问同一个键，可以先将值取出来，避免重复查找：

```go
// 优化前
for i := 0; i < 1000; i++ {
    m["key"] += i
}

// 优化后
if v, ok := m["key"]; ok {
    for i := 0; i < 1000; i++ {
        v += i
    }
    m["key"] = v
}
```

### 使用指针减少内存消耗

对于大型结构体，可以存储指针而不是值，减少内存消耗并提高性能：

```go
// 存储值
peopleByValue := make(map[string]LargeStruct)
peopleByValue["key"] = largeStruct

// 存储指针
peopleByPointer := make(map[string]*LargeStruct)
peopleByPointer["key"] = &largeStruct
```

## 实际应用示例

### 计数器/频率统计

```go
func wordCount(text string) map[string]int {
    words := strings.Fields(text)
    counts := make(map[string]int, len(words))
    for _, word := range words {
        counts[word]++
    }
    return counts
}
```

### 缓存实现

```go
type Cache struct {
    storage map[string]interface{}
    mutex   sync.RWMutex
    timeout time.Duration
    lastAccessed map[string]time.Time
}

func NewCache(timeout time.Duration) *Cache {
    cache := &Cache{
        storage:     make(map[string]interface{}),
        lastAccessed: make(map[string]time.Time),
        timeout:     timeout,
    }
    
    // 启动自动清理过期项的goroutine
    go cache.janitor()
    
    return cache
}

func (c *Cache) Get(key string) (interface{}, bool) {
    c.mutex.RLock()
    defer c.mutex.RUnlock()
    
    val, exists := c.storage[key]
    if exists {
        c.lastAccessed[key] = time.Now()
    }
    return val, exists
}

func (c *Cache) Set(key string, value interface{}) {
    c.mutex.Lock()
    defer c.mutex.Unlock()
    
    c.storage[key] = value
    c.lastAccessed[key] = time.Now()
}

func (c *Cache) janitor() {
    ticker := time.NewTicker(c.timeout / 2)
    defer ticker.Stop()
    
    for range ticker.C {
        c.deleteExpired()
    }
}

func (c *Cache) deleteExpired() {
    now := time.Now()
    c.mutex.Lock()
    defer c.mutex.Unlock()
    
    for key, lastAccessed := range c.lastAccessed {
        if now.Sub(lastAccessed) > c.timeout {
            delete(c.storage, key)
            delete(c.lastAccessed, key)
        }
    }
}
```

### 去重

```go
func removeDuplicates(items []string) []string {
    seen := make(map[string]struct{})
    result := make([]string, 0, len(items))
    
    for _, item := range items {
        if _, ok := seen[item]; !ok {
            seen[item] = struct{}{}
            result = append(result, item)
        }
    }
    
    return result
}
```

### 集合操作

```go
type Set map[string]struct{}

func NewSet() Set {
    return make(map[string]struct{})
}

func (s Set) Add(item string) {
    s[item] = struct{}{}
}

func (s Set) Remove(item string) {
    delete(s, item)
}

func (s Set) Contains(item string) bool {
    _, exists := s[item]
    return exists
}

func (s Set) Size() int {
    return len(s)
}

func (s Set) Union(other Set) Set {
    result := NewSet()
    for item := range s {
        result.Add(item)
    }
    for item := range other {
        result.Add(item)
    }
    return result
}

func (s Set) Intersection(other Set) Set {
    result := NewSet()
    for item := range s {
        if other.Contains(item) {
            result.Add(item)
        }
    }
    return result
}

func (s Set) Difference(other Set) Set {
    result := NewSet()
    for item := range s {
        if !other.Contains(item) {
            result.Add(item)
        }
    }
    return result
}
```

## 常见问题与陷阱

### nil map 的行为

未初始化的map（nil map）可以进行读取操作，但不能写入：

```go
var m map[string]int  // nil map
v := m["key"]         // 返回零值，不会panic
v, ok := m["key"]     // ok = false，不会panic
fmt.Println(len(m))   // 输出0，不会panic

m["key"] = 42         // panic: assignment to entry in nil map
```

因此，使用map前务必确保它已被初始化：

```go
m := make(map[string]int) // 初始化map
```

### 并发写入导致的问题

前面提到，map不是并发安全的。并发写入可能导致：
1. 数据竞争，导致不可预期的行为
2. 运行时panic：`concurrent map writes`

### 删除不存在的键

删除不存在的键是安全的，不会产生错误：

```go
m := make(map[string]int)
delete(m, "nonexistent") // 不会panic，也不会有任何负面影响
```

### 使用浮点数作为键

由于浮点数的精度问题，使用浮点数作为键可能导致意外行为：

```go
m := make(map[float64]string)
m[0.1] = "hello"
fmt.Println(m[0.1])                               // hello
fmt.Println(m[0.1+0.2-0.2])                       // 可能为空字符串（取决于精度）
fmt.Println(m[math.Round((0.1+0.2-0.2)*1e6)/1e6]) // 更安全的方式
```

为避免问题，可以：
1. 避免使用浮点数作为键
2. 对浮点数进行处理（如取固定小数位）后再作为键
3. 使用字符串表示浮点数（如"0.1"）作为键

### 访问不存在的键

访问不存在的键会返回该类型的零值，这可能导致判断错误：

```go
m := map[string]int{"a": 0, "b": 1}
v := m["c"] // v = 0，但键"c"并不存在

// 正确的做法是使用"comma ok"惯用法
v, ok := m["c"] // v = 0, ok = false
if ok {
    // 键存在
} else {
    // 键不存在
}
```

## 与其他数据结构的比较

### Map vs Slice

| 特性     | Map                 | Slice               |
|----------|---------------------|---------------------|
| 存储方式 | 键值对              | 顺序列表            |
| 查找效率 | O(1)平均，O(n)最坏  | O(n)                |
| 内存使用 | 较高，有额外开销    | 较低，紧凑          |
| 有序性   | 无序                | 有序                |
| 适用场景 | 键值查找、去重      | 顺序访问、排序数据  |

### Map vs Struct

| 特性     | Map                    | Struct                |
|----------|------------------------|----------------------|
| 字段访问 | 运行时（字符串键）     | 编译时（固定字段名）  |
| 类型安全 | 运行时检查             | 编译时检查           |
| 性能     | 较慢（有哈希查找）     | 较快（直接内存访问） |
| 灵活性   | 高（可动态添加字段）   | 低（固定字段）       |
| 适用场景 | 动态字段、JSON处理     | 固定字段的数据结构   |

### Map vs sync.Map

| 特性     | Map                     | sync.Map                 |
|----------|-------------------------|--------------------------|
| 并发安全 | 否                      | 是                       |
| 性能     | 单线程更快              | 并发场景更好            |
| API      | 简单直观                | 方法调用                 |
| 类型安全 | 是（编译时）            | 否（使用interface{}）    |
| 适用场景 | 单线程、需要类型安全    | 并发读写、读多写少场景   |
``` 