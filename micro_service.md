- 如何划分微服务
    - [refer1](https://xie.infoq.cn/article/8b8cbe87fae37bc7b5f151812)

- 熔断需要注意什么

- 微服务宗旨是一个服务独占一个db，cache。 如果服务多了，一些公司。成千上万个服务，总不能弄成千上万个数据库
  - 考虑方向-业务隔离

- 服务降级如何实现的
- 服务处于高负载的情况下，探活和监控怎么能尽量不影响业务？
- gateway身份验证
- 有状态服务和无状态服务的区别
  - [refer](https://blog.csdn.net/universsky2015/article/details/105677992?ops_request_misc=%257B%2522request%255Fid%2522%253A%2522164604176216780271569545%2522%252C%2522scm%2522%253A%252220140713.130102334..%2522%257D&request_id=164604176216780271569545&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~all~sobaiduend~default-2-105677992.pc_search_result_positive&utm_term=%E6%9C%89%E7%8A%B6%E6%80%81%E5%92%8C%E6%97%A0%E7%8A%B6%E6%80%81&spm=1018.2226.3001.4187)

- 限流