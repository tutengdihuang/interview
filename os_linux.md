## 计算机组成原理

- 虚拟内存和物理内存
- 什么是进程
- 操作系统层面的锁
    - 自旋锁
        - 当一个线程尝试去获取某一把锁的时候，如果这个锁此时已经被别人获取(占用)，那么此线程就无法获取到这把锁，该线程将会等待，间隔一段时间后会再次尝试获取。这种采用循环加锁 -> 等待的机制被称为自旋锁(spinlock)
        - 自旋锁避免了操作系统进程调度和线程切换，所以自旋锁通常适用在时间比较短的情况下
        - 自旋锁的目的是占着CPU资源不进行释放，等到获取锁立即进行处理
        - 适用场景： 对资源竞争不激烈，或者说持有锁的线程时间很短
    - mutex
        - 用于多线程编程中，防止两条线程同时对同一公共资源（也可以说是临界资源，就是能够被多个线程共享的数据）进行读写的机制
        - mutex的本质是一种变量

- 水平触发和边缘触发
- C/S模型中，应用层调用close后会发生什么
- 操作系统虚拟内存的作用是什么
- 操作系统中断是什么
- linux 系统 查看日志实时刷新
- linux查看内存
- linux查看应用程序端口 ps -ef | grep nginx
- 操作系统中线程和进程的对应关系，进程和线程的状态有哪些？
- 进程和线程不同状态直接的转移具体操作有哪些操作？具体说明一下，一个进程从运行态转入阻塞态有哪些情况
- 操作系统的虚拟地址和物理地址，虚拟地址到物理地址的转换过程，说一下页式管理的方式的转换过程
- 也表里具体存的是什么，能具体说明一下吗？
- io-wait是做什么的
- 题目：已知方法 curl  要求实现  MutilCurl 能够制定并发请求数量，并返回请求错误
- epoll 原理? select原理？ 什么水平触发 边缘触发？
- linux中io_wait在哪里用过
- tcp/ip协议中，第一次分手，server端回复fin ack后进入什么状态？对应在linux系统中的文件描述在哪里？
- 简述一下线程切换的完整过程
- 操作系统的线程模型有哪几种
- cache 如何实现回写
- 说说cache 流程，cache snoop 协议
- 你还用到了阿里云对象存储存储图片文件?那没有有用到它的其他一些功能?它是怎么收费的?图片加
  速用过吗?
#
- tcpdump
    - [refer](https://blog.csdn.net/qq_51574197/article/details/116171604?ops_request_misc=%257B%2522request%255Fid%2522%253A%2522167740263216782428647734%2522%252C%2522scm%2522%253A%252220140713.130102334..%2522%257D&request_id=167740263216782428647734&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~all~top_positive~default-1-116171604-null-null.142^v73^control,201^v4^add_ask,239^v2^insert_chatgpt&utm_term=tcpdump&spm=1018.2226.3001.4187)
#
- netstat
    - [refer](https://blog.csdn.net/qq_42014600/article/details/90372315?ops_request_misc=%257B%2522request%255Fid%2522%253A%2522167738459316800182794622%2522%252C%2522scm%2522%253A%252220140713.130102334..%2522%257D&request_id=167738459316800182794622&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~all~top_positive~default-1-90372315-null-null.142^v73^control,201^v4^add_ask,239^v2^insert_chatgpt&utm_term=netstat&spm=1018.2226.3001.4187)
#
- 如何查看进程的状态查看进程的网络状态
  - [refer](https://www.meijindong.com/posts/2679121756.html)
```bash
   ps -l   列出与本次登录有关的进程信息；
   ps -aux   查询内存中进程信息；
   ps -aux | grep ***   查询***进程的详细信息；
   top   查看内存中进程的动态信息；
   kill -9 pid   杀死进程。
```
#
- 为什么进程->线程->协程占用内存越来越小?

# 
- 协程会一直占用cpu资源吗
#
- 进程/线程 切换时的调度算法
#
- 线程切换的过程
#
- 写时复制?(操作系统)