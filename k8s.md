## 云原生

- 在大规模集群下，怎么优化k8s集群
- 非nodeport和lb的svc怎么暴露给外部集群
- k8s内部都有哪些证书，是一样的么
- k8s master都有哪些组件，可以随意部署么
- prometheus的内部组件和抓取机制
- 大规模优化思路k8s多region方案
- iptables pod 流量的tables具体指什么
- 一个pod创建流程，cni实现原理和某些特定场景优化，master组件功能和优化，网络问题排查，service account等使用，namespace和cgroup，证书相关

  

- 1.怎么让K8S集群内资源使用量更平均
  答：目前碰到过资源不均衡的情况主要是node节点宕机后，迁移的pod不会自动调度会node节点，这种情况可以手工delete 或者scale 副本伸缩一次，pod会被scheduler自动调度会node.


- 2.如何修改scheduler的调度策略。
通过修改deployment的spec下的节点亲和pod.spec.affinity和pod亲和反亲和 podAffinity/podAntiAffinity 影响调度器的调度策略。

- 3.Deployment和SatefulSet 的根本区别在哪里
有状态和无状态的区别
无状态：
deployment认为所有的pod都是一样的,可以随意扩容和缩容
不用考虑顺序的要求,不用考虑在哪个node节点上运行

有状态：
实例之间有差别，每个实例都有自己的独特性，元数据不同，例如etcd, zookeeper
可以实现有序，优雅的部署和扩展、删除和终止(例如: mysql 主从关系，先启动主，再启动从)

- POD创建过程中，controller和scheduler 各起到了什么作用，两者的联系是什么？
控制器：定义了pod的部署方式如多少副本，调度器：将未分配节点的pod,通过调度算法计算，自动分配到对应的pod.
scheduler 先检测到pod未分配节点的pod ，然后执行分配，controller 通过watch apiservice的对象变化，指挥在对应的node创建pod.（用词可能不准确）。


- kube-proxy 在ISO 7层中的那一层
传输层

StatefulSet 的滚动升级的过程是什么样的，现在我们希望只升级 StatefulSet 中的任意个节点进行测试, 可以怎么做?
有序部署，有序扩展，即Pod是有顺序的，在部署或者扩展的时候要依据定义的顺序依次依次进行（即从0到N-1，在下一个Pod运行之前所有之前的Pod必须都是Running和Ready状态），基于init containers来实现。
我们的做法是更新主机或者仓库的镜像，但是不更新标签，手工重启pod.


- Kubernetes 的所有资源约定了版本号, 为什么要这么做?
google同时多个小团队开发，加快版本迭代速度。

- pod中penging状态，代表了什么？
scheduler正在给pod指定node节点，如果长期如此可能节点资源不足，或者不满足nodeselector节点选择器和 affinity亲和性导致。

超哥解释：
https://jimmysong.io/kubernetes-handbook/concepts/pod-lifecycle.html
挂起（Pending）：Pod 已被 Kubernetes 系统接受，但有一个或者多个容器镜像尚未创建。等待时间包括调度 Pod 的时间和通过网络下载镜像的时间，这可能需要花点时间。


- 说下POD跨主机通信的过程
k8s1.14版本之后之后都走ipvs，通过四层转发。
首先 pod1 通过自己的以太网设备 eth0 把数据包发送到关联到 root 命名空间的 veth0 上，--》网桥查找转发表，发现找不到，则会把包转发到默认路由（root 命名空间的 eth0 设备）--》然后数据包经过 eth0 就离开了 Node1，被发送到网络。--》数据包到达 Node2 后，首先会被 root 命名空间的 eth0 设备发现--》然后通过网桥把数据路由到虚拟设备 veth1,最终数据表会被流转到与 veth1 配对的另外一端。

- APIserver 出现大量5XX，可能是出现了什么问题？
可能是etcd异常，https证书过期，主机安全端口被占用等等（没碰到过）

- K8S 集群节点出现NotReady 应该如何排查？
kubectle describe node xxx 查看异常原因，可能是node节点的kubelet进程或者kube proxy进程等基础组件异常导致，也有可能是主机的资源不足导致（网络，磁盘，内存，cpu）

- 你做过哪些基于K8S云原生的业务改造？
开放题，可以参考阿里和腾讯大厂的。

- 在节点上有200个工作中的容器的情况下，如何优雅下线。
答可以打好taint和toleration  加入那几个node 实现驱逐下线。
面试官答案：分组驱逐，实现方案我没想好。

- 多master如何保持一致性。有什么风险点吗？
当时直接答的不会。