
## GMP调度
### GOROUTINE 定义
### 1.0 之前 GM 调度模型
### GMP 中WORK STEALING 机制
### GMP 中HAND OFF 机制
### 协作式的抢占式调度
### 基于信号的抢占式调度
### GMP 调度过程中存在哪些阻塞
### SYSMON 有什么作用 

## GMP进阶
### GMP什么时候回创建新的M，创建有数量限制吗
### 阻塞GM绑定之后就回去寻找新的M吗
### goroutine是什么，怎么执行
    - goroutine是比线程还轻量的执行单位，是用户层面的
    - 一个gourontine大约3kb左右
    - 上下文切换成本小
    - goroutine GMP模型，M：N模型
    - 如果可以聊聊goroutine的生老病死
### goroutine切换的原理
    - 网络io阻塞主动切换，cpu占用时间过长信号切换，锁，channel
### GO的GPM模型?P和M的数量怎么决定？如果在K8S容器部署，P和M又会有什么不同？
### GMP模型？全局队列没有g了，怎么办
    - 去其他p的g队列偷取
### goroutine的亲缘性怎么体现出来
### Golang中需要使用协程池吗？为什么？
### goroutine为啥不设置id
### 线程模型有哪些？为什么 Go Scheduler 需要实现 M:N 的方案？Go Scheduler 由哪些元素构成呢？
