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
- 如何主动关闭goroutine

- goroutine和线程有什么区别

- 为什么goroutine的调度更高效
    - 用户态避免上下文切换（避免用户态到内核态的切换）

    - 创建与销毁的开销更小，通过goroutine runtime

    - 内存消耗更少，大约一个groutine的大小为3kb

- goroutine是什么，怎么执行
    - goroutine是比线程还轻量的执行单位，是用户层面的
    - 一个gourontine大约3kb左右
    - 上下文切换成本小
    - goroutine GMP模型，M：N模型
    - 如果可以聊聊goroutine的生老病死
- GO的GPM模型?P和M的数量怎么决定？如果在K8S容器部署，P和M又会有什么不同？

- 什么是死锁？go什么情况会死锁？怎么避免死锁问题？
- go sync包有哪些方法以及具体作用
- go context包的作用
- 字节对齐和大小端序
    - [解答](https://www.yuque.com/docs/share/2f155ad2-4b48-415a-acf6-5ca11571d3db)
- golang gc 操作系统不真实释放内存怎么办
- map实现及底层原理？(sixin)
    - [go 设计与实现](https://draveness.me/golang/docs/part2-foundation/ch03-datastructure/golang-hashmap/#%E6%89%A9%E5%AE%B9)
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
- grpc通信类型
    - 四种
    - 一元 流式，客户端和服务器分别凑合就可以，一共四种


- promethus和elk的区别
- 架构的知识
    - 可扩展
    - 可伸缩

- 如何手动设计一个map

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
    - 制毒channel不需要关闭
- 索引如何优化？
    - 查看数据库相关

- go中哪些是值类型，哪些是引用类型
    - 引用类型：指针，map，slice，channel，方法与函数
    - 值类型：int系列、float系列、bool、string、数组和结构体
- sync map的原理
    - [refer](https://blog.csdn.net/weixin_42663840/article/details/107958274)
- 值传递和引用传递的区别
    - golang中只有值传递
- 切片传递过去，如果被调用函数append()，原来的切片会不会变化
    - 不会，已经实验过
- map 锁+map sync.map concurrentmap的区别

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
- GMP模型？全局队列没有g了，怎么办
    - 去其他p的g队列偷取
- 说说常用的设计模式

- slice的扩容规则（sixin）
- gc 原理？三色法？混合写屏障？
- 垃圾回收怎么检测阻塞


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











