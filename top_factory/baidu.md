## 001

- 1	数据库问题，给你10个数据库服务器，每个只能接500的qps，现在要实现4000qps，要怎么做？说用负载均衡，使用binlog保证10个服务器的数据一致性
- 2	 如果有有读有写，如何实现高并发，数据库读写分离
- 3	对于两个写库，两个请求向分别打到两个写库中，他们互相向对方同步，会不会出现不一致，
- 4	哈希的实现有哪几种，如何取hashcode，冲突检测几种方法
- 5	用过go，那么进程，协程，线程各自的优缺点
- 6	 算法题 z遍历二叉树，循环有序数组找指定值，
- 7	 1.事务是怎么实现的?(undo_log,MVCC)
- 8	mongodb和redis的区别
- 9	请你说说golang的CSP思想
- 10	go 内存逃逸分析（分析了栈帧，讲五种例子，描述堆栈优缺点，点头）
- 11	是否有逃逸分析过
- 12	defer recover 的问题
- 13	mysql 索引慢分析（线上开启slowlog，提取慢查询，然后仔细分析explain 中 tye字段以及extra字段，发生的具体场景及mysql是怎么做的

### 002
- 查看日志实时刷新 
- linux查看内存
- linux查看应用程序端口 ps -ef | grep nginx

- docker 为什么轻量
- docker run -v
- docker rm
- docker应用的linux技术 namespace和写时复制
- k8s

- git 如何查看文件
- git 工作区和暂存区的区别 head 是什么
- tcp udp的区别，三次握手和四次挥手，拥塞控制


- 进程线程和协程的区别
- 线程间的通信方式
- 一般有哪几种锁goroutine有哪几种锁
- 使用锁能实现同步吗
- golang slice和数组
- 在程序中能否对slice进行增删
- golang map[string]interface{}做形参能否传入，map[string]string
- 单核goroutine中死循环，怎么调度出来

- filebeat的问题
- 两段式提交
- sync.pool
- golang debug工具 性能分析
- map深拷贝浅拷贝
- redis的数据结构

- 大文件排序 文件切分+快排+堆排序
- 一个随机数生成器只有0和1 生成概率为p和1-p，要写一个随机数生成器，生成1和0的概率都为1/2
- 写程序反转链表 ac
- 链表和数组的区别

## 003
```go
账号系统怎么做认证的 session和cookie
线上qps多少
redis是自己搭建的还是？aws和华为云redis的集群
redis缓存穿透和缓存雪崩
对称加密和非对称加密
对称DES 3DES AES DESX Blowfish
非对称加密 RSA DSA ECC
为什么用channel来控制协程数量，协程太多会timeout

go1.14有了解吗 没有
为什么用go1.13  算是没答出来

tcp拥塞控制算法，答了慢开始，拥塞避免，快重传和快恢复，问有没有其他的算法
问 为什么需要拥塞避免，如果服务器性能超级好，带宽超级大，丢包就重传，还需要使用拥塞避免吗
两道算法题 连续数组的最大和 二叉搜索树第k大的节点
一个golang读写锁相关的题目，
操作系统虚拟内存的作用是什么
操作系统中断是什么
golang sync.map
golang内存分配
怎么实现线程安全的map
什么是并发安全，就是多线程下会产生预期的结果

分配在栈上和分配在堆上有什么区别，分配在栈上有什么好处
栈的内存管理简单，分配比堆上快
栈的内存不需要回收，堆需要主动free
栈的内存访问有更好的局部性，堆上的访问速度比栈上的速度要慢

cookie可以设置什么，http only ， domain ， key value，expire，
sync.map go 1.9 引入的

怎么获取当前goroutine的数量，怎么获取当前goroutine的id
run.NumGoroutines() 
goid 从runtime.stack上获取
```