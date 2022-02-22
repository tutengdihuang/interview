## redis zset 实现原理
    - 压缩链表，跳表
## redis的nvm

## Redis在项目中的使用场景
## redis常用的数据结构
## redis的zset实现
## Redis缓存淘汰策略
- [reference](https://www.processon.com/view/link/620b45476376897c8c7239d0)
## Redis主从复制原理
- [reference](https://www.processon.com/view/link/620b4875f346fb617416aed3)
## Redis怎么实现高可用
- [reference](https://www.processon.com/view/link/620b48f17d9c0807ec8cf49a)
## redis缓存雪崩、缓存穿透、缓存击穿
- [reference](https://www.processon.com/view/link/61e648387d9c0806a8b0cf29)

## redis string的编码方式
    - int 编码：保存long 型的64位有符号整数
    - embstr 编码：保存长度小于44字节的字符串
    - raw 编码：保存长度大于44字节的字符串
## redis的日志？主从同步用哪些？主从同步时候继续有数据写入怎么办？
    - AOF，RDB
    - RDB
    - replica_backoff_buffer(环形队列，可以被覆盖)
## redis的底层结构

## 2020-01-19 收录


- redis缓存雪崩、缓存穿透、缓存击穿？如何解决
    - 缓存穿透是指缓存和数据库中都没有的数据，而用户不断发起请求，如发起为id为“-1”的数据或id为特别大不存在的数据。这时的用户很可能是攻击者，攻击会导致数据库压力过大。

    - 接口层增加校验，如用户鉴权校验，id做基础校验，id<=0的直接拦截；
      从缓存取不到的数据，在数据库中也没有取到，这时也可以将key-value对写为key-null，缓存有效时间可以设置短点，如30秒（设置太长会导致正常情况也没法使用）。这样可以防止攻击用户反复用同一个id暴力攻击

    - 缓存击穿是指缓存中没有但数据库中有的数据（一般是缓存时间到期），这时由于并发用户特别多，同时读缓存没读到数据，又同时去数据库去取数据，引起数据库压力瞬间增大，造成过大压力

    - 设置热点数据永远不过期。
    - 加互斥锁

    - 缓存雪崩是指缓存中数据大批量到过期时间，而查询数据量巨大，引起数据库压力过大甚至down机。和缓存击穿不同的是，        缓存击穿指并发查同一条数据，缓存雪崩是不同数据都过期了，很多数据都查不到从而查数据库。
      缓存数据的过期时间设置随机，防止同一时间大量数据过期现象发生。
      如果缓存数据库是分布式部署，将热点数据均匀分布在不同搞得缓存数据库中。
      设置热点数据永远不过期。
- 那问你随机过期时间实现得不好就出现了大量key过期
- 关于分布式锁redis
    - 1. setnx + expire + del
    - 2. 如果expire时间过短，业务还没执行完锁失效了，那么别的请求可以共享资源了，该如何做？
    - 3. 如果expire时间过长，而业务执行完之后由于某种原因del lock失败，那么其他请求就获取不到锁了，该如何做？
    - 4. 如果业务还在执行，锁被别人del了，那么如何保证共享资源？
    - [参考答案](https://mp.weixin.qq.com/s/zwkK0YD6b94iwt_v36e-jw)
- redis里面有热点数据10w个。这时候一个程序员从数据库中捞了1000个新的数据返回，顶替了1000个热点数据（程序员用新的key塞入redis，导致redis中其他老的1000个key被删除）。用什么方式可以避免这样的情况发生？(字节面试题)
	
- redis 持久化有哪几种方式
- redis 集群有哪几种，redis集群怎么实现高可用
- redis怎么做高可用，机制
- 怎么保证redis和mysql的数据一致性
- redis的删除策略
- redis缓存穿透 如果用很多不存在的key攻击怎么办
    - 可以使用布隆过滤器，bitmap来实现
  
- redis sorted set score的范围是多少
    - (-2^53---- +2^53)

- redis数据结构, sort set 底层实现
    -     - [refer](https://processon.com/mindmap/60c09f127d9c087937196f50)
- redis 内存优化相关的设计
- 秒杀,促销设计
    - [refer](https://processon.com/mindmap/60f43a4c7d9c087bac5cd26f)

- redis如何保证lua脚本的一致性
    - 原子操作：Redis会将整个脚本作为一个整体执行，中间不会被其他进程或者进程的命令插入
    - [reference](https://segmentfault.com/a/1190000019676878)

- redis里面有热点数据10w个。这时候一个程序员从数据库中捞了1000个新的数据返回，顶替了1000个热点数据（程序员用新的key塞入redis，导致redis中其他老的1000个key被删除）。用什么方式可以避免这样的情况发生？
  - 参看答案： 利用lfu淘汰策略
  - 参考答案2： 冷热数据分离，redis机器分离

- redis 复制的原理 
- redis哨兵原理
- redis主动下线被动下线 
- redis复制延迟 
- redis的故障恢复
- redis cluster 集群中突然宕机了一台服务器会发生什么
- redis cluster 中为什么要用crc16算法 而不用哈希一致性算法？
- redis哨兵集群中有服务挂掉，哨兵会做哪些事？
- 热key的探测方案？
- redis 大量key过期，但是内存不释放？
  - 淘汰策略为懒淘汰，修改淘汰策略
- redis内存很多碎片，导致内存居高不下，如何解决？
- 为啥RedisCluster设计成16384个槽
  - [refer](https://zhuanlan.zhihu.com/p/99037321)


