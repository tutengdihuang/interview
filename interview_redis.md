## redis zset 实现原理
    - 压缩链表，跳表
## redis的nvm

## Redis在项目中的使用场景
## redis常用的数据结构
## redis的zset实现
## Redis缓存淘汰策略
## Redis主从复制原理
## Redis怎么实现高可用

## redis缓存雪崩、缓存穿透、缓存击穿
    - [refer](https://www.processon.com/view/link/61e648387d9c0806a8b0cf29)

## redis string的编码方式
    - int 编码：保存long 型的64位有符号整数
    - embstr 编码：保存长度小于44字节的字符串
    - raw 编码：保存长度大于44字节的字符串
## redis的日志？主从同步用哪些？主从同步时候继续有数据写入怎么办？
    - AOF，RDB
    - RDB
    - replica_backoff_buffer(环形队列，可以被覆盖)
## redis的底层结构