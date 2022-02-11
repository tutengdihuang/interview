- 什么事精群效应？nginx是怎么解决的
    - 定义： 惊群效应就是当一个fd的事件被触发时，所有等待这个fd的线程或进程都被唤醒
    - Nginx的epoll工作流程
        - master进程先建好需要listen的socket后，然后再fork出多个woker进程，这样每个work进程都可以去accept这个socket
        - 当一个client连接到来时，所有accept的work进程都会受到通知，但只有一个进程可以accept成功，其它的则会accept失败，Nginx提供了一把共享锁accept_mutex来保证同一时刻只有一个work进程在accept连接，从而解决惊群问题
        - 当一个worker进程accept这个连接后，就开始读取请求，解析请求，处理请求，产生数据后，再返回给客户端，最后才断开连接，这样一个完成的请求就结束了

- 默认并发数是多少
    - 8，可以设置