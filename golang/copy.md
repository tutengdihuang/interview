# Go语言 copy() 函数详解

## 1. 基本概念
- `copy()` 是 Go 的内置函数
- 语法：`copy(dst, src)` 返回复制的元素个数
- 只复制切片的第一层元素（浅拷贝）
- 不会自动扩容目标切片

## 2. 复制行为
### 2.1 基础类型切片
```go
srcInts := []int{1, 2, 3}
dstInts := make([]int, len(srcInts))
copy(dstInts, srcInts) // 完整复制值
```

### 2.2 引用类型切片
```go
type Person struct {
    Friends *[]string
}
srcPeople := []Person{{Friends: &[]string{"Tom"}}}
dstPeople := make([]Person, 1)
copy(dstPeople, srcPeople) // 只复制指针
```

## 3. 重要注意事项
### 3.1 长度和容量
```go
dst := make([]int, 2, 4)    // 长度2，容量4
src := []int{1, 2, 3, 4}    // 长度4
n := copy(dst, src)         // n == 2，只复制2个元素
```

### 3.2 内存安全
```go
// 错误示例
var dst []int
copy(dst, []int{1,2,3})     // 复制0个元素，dst仍为nil

// 正确示例
dst := make([]int, 3)
copy(dst, []int{1,2,3})     // 正确复制
```

## 4. 常见应用场景
### 4.1 切片克隆
```go
clone := make([]int, len(original))
copy(clone, original)
```

### 4.2 切片截取
```go
src := []int{1,2,3,4,5}
dst := make([]int, 3)
copy(dst, src[1:4])  // 复制部分元素
```

## 5. 性能考虑
- `copy` 是内置函数，性能较好
- 对于小切片，直接循环赋值可能更快
- 大切片使用 `copy` 更有优势

## 6. 并发安全
- `copy` 本身是并发安全的
- 但需要注意源和目标切片的并发访问

```go
// 需要适当的同步机制
var mu sync.Mutex
mu.Lock()
copy(dst, src)
mu.Unlock()
```

## 7. 最佳实践
### 7.1 预先分配目标切片
```go
dst := make([]T, len(src))
copy(dst, src)
```

### 7.2 检查返回值
```go
if n := copy(dst, src); n != len(src) {
    // 处理复制不完整的情况
}
```

### 7.3 深拷贝替代方案
```go
// 可以使用序列化/反序列化
// 或实现自定义的深拷贝方法
// 或使用反射
```

## 8. 常见错误
1. 未初始化目标切片
2. 忽略目标切片长度限制
3. 假设是深拷贝
4. 未考虑并发安全性

## 9. 特殊用例
1. 在环形缓冲区中使用
2. 在字节流处理中使用
3. 在消息队列实现中使用

## 总结
- copy 是浅拷贝
- 只复制第一层元素
- 需要注意目标切片的长度
- 在并发环境中需要适当的同步机制
- 对于需要深拷贝的场景，需要使用其他方案
