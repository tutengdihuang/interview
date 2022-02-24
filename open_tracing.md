- 链路追踪的原理？
- 链路追踪基本定义
```go
有哪些span
Operation name：操作名称 （也可以称作 Span name）。
Start timestamp：起始时间。
Finish timestamp：结束时间。
Span tag：一组键值对构成的 Span 标签集合。键值对中，键必须为 String，值可以是字符串、布尔或者数字类型。
Span log：一组 Span 的日志集合。每次 Log 操作包含一个键值对和一个时间戳。键值对中，键必须为 String，值可以是任意类型。
SpanContext: pan 上下文对象。每个 SpanContext 包含以下状态：
要实现任何一个 OpenTracing，都需要依赖一个独特的 Span 去跨进程边界传输当前调用链的状态（例如：Trace 和 Span 的 ID）。
Baggage Items 是 Trace 的随行数据，是一个键值对集合，存在于 Trace 中，也需要跨进程边界传输。
References（Span 间关系）：相关的零个或者多个 Span（Span 间通过 SpanContext 建立这种关系）

```