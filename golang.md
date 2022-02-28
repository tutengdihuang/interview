## golang
- init() 函数是什么时候执行的？
    - 参考答案init() 函数是 Go 程序初始化的一部分。Go 程序初始化先于 main 函数，由 runtime 初始化每个导入的包，初始化顺序不是按照从上到下的导入顺序，而是按照解析的依赖关系，没有依赖的包最先初始化。

    - 每个包首先初始化包作用域的常量和变量（常量优先于变量），然后执行包的 init() 函数。同一个包，甚至是同一个源文件可以有多个 init() 函数。init() 函数没有入参和返回值，不能被其他函数调用，同一个包内多个 init() 函数的执行顺序不作保证。

    - 一句话总结： import –> const –> var –> init() –> main()
- channel的应用场景
    - 如果面试官问的比较笼统可以从一下几个方面回答
        - 是什么
        - 有什么特性
        - 怎么用
- go channel使用需要注意的地方
  ![image](https://user-images.githubusercontent.com/31843331/153333030-3ca372b8-53c8-41db-ba86-ac89b9de636d.png)

- 如何主动关闭goroutine

- goroutine和线程有什么区别

- 为什么goroutine的调度更高效
    - 用户态避免上下文切换（避免用户态到内核态的切换）

    - 创建与销毁的开销更小，通过goroutine runtime

    - 内存消耗更少，大约一个groutine的大小为3kb


- 什么是死锁？go什么情况会死锁？怎么避免死锁问题？
- go sync包有哪些方法以及具体作用
- go context包的作用
- 字节对齐和大小端序
    - [解答](https://www.yuque.com/docs/share/2f155ad2-4b48-415a-acf6-5ca11571d3db)
- golang gc 操作系统不真实释放内存怎么办
- context原理
- single—flight实现原理
- 延迟队列
    - 参看zinx时间轮
- go-micro 的模块分为哪些
    - 组件
        - Go Micro
        - API
        - Sidecar
        - Web
        - Cli
        - Bot
## grpc
- grpc通信类型
    - 四种
    - 一元 流式，客户端和服务器分别凑合就可以，一共四种
- grpc为什么要使用http2.0当传输层协议？
- grpc如何实现负载均衡



- 两个客户端A,B A读取数据，B修改还未提交，A再次读取，A两次读取的消息是一致的吗？
- grpc 版本字段增加，服务端客户端如何升级？先后顺序是什么
    - 先升级服务端，后升级客户端
    - 如果字段修改- 保持将修改参数改为增加一个新的字段，先升级服务器，再升级客户端
        - 然后等客户端升级
        - 然后服务端删除老字段
        - 然后服务器在舍掉该字段
- 如何设计一个10亿访问量的系统

- gin和beego的区别
    - 对mvc的支持
        - beego支持完整的mvc
        - gin不支持完整的mvc
    - 对路由的支持
        - Beego 支持正则路由，支持restful Controller路由
        - Gin不支持正则路由
    - 适用场景
        - 在业务更加复杂的项目，适用beego
        - 在需要快速开发的项目，适用beego
    - Gin在性能方面较beego更好
        - 当某个接口性能遭到较大的挑战，考虑用Gin重写
        - 如果项目的规模不大，业务相对简单，适用Gin

- go-micro 的优缺点（微服务的优缺点）
    - 优点：逻辑清晰，简化部署，可扩展，灵活组合，技术异构，高可靠
    - 缺点：复杂度高，运维复杂，影响性能
- go-micro存在的意义
- grpc 加密
    - jwt

- emao<-a<-b<-c, abc 任何一个出错，abc都退出，最后emo退出
- channel关闭需要注意什么事情？
    - 生产者关闭
    - 多个写入者时，需要在外城使用一个WaitGroup，等所有写入者完成之后再关闭.
    - 只读channel不需要关闭
- 索引如何优化？
    - 查看数据库相关

- go中哪些是值类型，哪些是引用类型
    - 引用类型：指针，map，slice，channel，方法与函数
    - 值类型：int系列、float系列、bool、string、数组和结构体

- 值传递和引用传递的区别
    - golang中只有值传递
- 切片传递过去，如果被调用函数append()，原来的切片会不会变化
    - append会修改slice所使用的底层数组，如果数组的不需要扩容会影响原来的切片；如果扩容则会引用新的数组，不会影响原切片；


- 如何确认二个map是否相等
    - reflect.DeepEqual(c1, c2)，可以是map，slice，struct
- cas  修改一块内存的值，值改变方式是a-b-a这个合理吗
    - 什么事ABA问题
        - 线程1，期望值为A，欲更新的值为B
        - 线程2，期望值为A，欲更新的值为B
        - 线程1抢先获得CPU时间片，而线程2因为其他原因阻塞
        - 线程1取值与期望的A值比较，发现相等然后将值更新为B
        - 线程3，期望值为B，欲更新的值为A，线程3取值与期望的值B比较，发现相等则将值更新为A
        - 线程2从阻塞中恢复，并且获得了CPU时间片，这时候线程2取值与期望的值A比较，发现相等则将值更新为B
        - 虽然线程2也完成了操作，但是线程2并不知道值已经经过了A->B->A的变化过程
    -
    - 如何解决ABA问题
        - 在变量前面加上版本号，每次变量更新的时候变量的版本号都+1，即A->B->A就变成了1A->2B->3A
- mutex的原理
    - [refer](https://www.processon.com/view/link/6078e4416376891132d67bcf)

- 说说常用的设计模式

- slice的扩容规则（sixin）

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
- 7. 插入写屏障和删除写屏障的时机和区别？go中如何实现的？
- 8. GC 的四个阶段？
- 9. 为何需要辅助标记和辅助清扫？
- 10. GC 4个阶段，STW发生在何时？
- 11. 描述下 gc 调步算法的实现？
- 12. 工作中gc debug的使用？
- 13. gc 清扫阶段 对象回收 和 内存单元回收的联系和差异？
## 说出打印结果（探探）
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

- 给出方法定义
  - 实现 errgroup.Group
  - 实现 singleflight.Group

- [腾讯］使用golang实现一个端口监听的程序。

- 内存对齐（from 不是山谷）
  - [链接答案](https://www.yuque.com/docs/share/2f155ad2-4b48-415a-acf6-5ca11571d3db)
  - [阅读2](https://mp.weixin.qq.com/s/H3399AYE1MjaDRSllhaPrw)

- 说说go语言的select机制？
  - (1)、select机制用来处理异步IO问题
  - (2)、select机制最大的一条限制就是每个case语句里必须是一个IO操作
  - (3)、golang在语言级别支持select关键字
  
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

- 29.为什么sync.WaitGroup中Wait函数支持 WaitTimeout 功能.

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
- golang的服务性能指标怎么查看，有哪些指标需要注
- 协程池的意义是什么
- 8、Go 当中同步锁有什么特点？作用是什么
    - 当一个 Goroutine（协程）获得了 Mutex 后，其他 Goroutine（协程）就只能乖乖的等待，除非该 Goroutine 释放了该 Mutex。RWMutex 在读锁占用的情况下， 会阻止写，但不阻止读 RWMutex。 在写锁占用情况下，会阻止任何其他Goroutine（无论读和写）进来，整个锁相当于由该 Goroutine 独占
    同步锁的作用是保证资源在使用时的独有性，不会因为并发而导致数据错乱， 保证系统的稳定性。
    
- 29、Channel 的 ring buffer 实现
    -channel 中使用了 ring buffer（环形缓冲区) 来缓存写入的数据。ring buffer 有很多好处，而且非常适合用来实现 FIFO 式的固定长度队列。在 channel 中，ring buffer 的实现如下：
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
- GO 语言当中数组和切片在传递的时候的区别是什么？ 14]
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
- CHANNEL 的RING BUFFER 实现 
- MUTEX 几种状
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
- GOROUTINE 定义
- 1.0 之前 GM 调度模型
- GMP 中WORK STEALING 机制
- GMP 中HAND OFF 机制
- 协作式的抢占式调度
- 基于信号的抢占式调度
- GMP 调度过程中存在哪些阻塞
- SYSMON 有什么作用 
- 三色标记原理
- 插入写屏障
- 删除写屏障
- 写屏障 
- 混合写屏障
- GC 触发时机
- GO 语言中 GC 的流程是什么？
- GC 如何调优
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
- 下边的程序输出什么

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

## map + sync map
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
- Go 如何高效地拼接字符串 ?

- [github他人收集](https://github.com/KeKe-Li/data-structures-questions/blob/master/src/chapter05/golang.01.md#Go%E4%B8%AD%E7%9A%84%E9%94%81%E5%A6%82%E4%BD%95%E5%AE%9E%E7%8E%B0)
- 怎么设计一个无锁的pool
## GMP
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