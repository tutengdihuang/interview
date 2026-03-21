## 云原生

# Kubernetes 核心知识体系

> 大规模集群优化 · 网络 · 证书 · 监控 · 故障排查

---

## 一、集群架构与组件

### K8s Master 都有哪些组件，可以随意部署么？

#### 核心组件清单

| 组件 | 职责 |
|------|------|
| kube-apiserver | 集群唯一入口，所有组件通过它交互；支持水平扩展，建议多副本 + LB |
| etcd | 分布式 KV 存储，保存所有集群状态；建议独立 3/5 节点奇数部署 |
| kube-controller-manager | 运行各类控制器（Deployment、Node、Endpoint 等）；内置 leader election，同一时间只有一个主节点工作 |
| kube-scheduler | 负责 Pod 调度决策；支持多调度器，也内置 leader election |
| cloud-controller-manager | 与云厂商对接（LB、Route、Node 生命周期）；云环境必须，裸金属可省略 |

#### 能随意部署吗？

- **kube-apiserver**：可多副本，需共享同一 etcd 集群，前置 LB
- **etcd**：必须奇数节点（3/5/7），偶数无法获得 quorum；与 master 同机或独立均可，生产建议独立
- **kube-controller-manager / kube-scheduler**：同一时间只有一个 leader 真正工作，多副本部署是为了 HA 快速切换
- **禁止**跨 region 将单个 etcd 集群拉伸部署，延迟会导致频繁选主

---

### 一个 Pod 的完整创建流程

#### 调用链路

1. `kubectl apply` → kube-apiserver 鉴权 / 准入 / 持久化 etcd
2. kube-controller-manager（ReplicaSet Controller）检测到 Pod 缺少 → 创建 Pod 对象写入 apiserver
3. kube-scheduler watch 到 Unscheduled Pod → 执行过滤（Filter）+ 打分（Score）→ 绑定节点（Bind）
4. kubelet watch 到绑定到本节点的 Pod → 调用容器运行时（containerd/docker）创建 Pause 容器
5. 调用 CNI 插件（ADD 命令）为 Pod 配置网络（veth pair / IP / 路由）
6. 拉取业务镜像 → 启动 Init 容器 → 启动业务容器
7. kubelet 执行 livenessProbe / readinessProbe → 通过后 Endpoint 加入 Service

#### 关键细节

- **Pause 容器**：共享 Network Namespace，CNI 只需配置一次
- **CSI**：存储卷在容器启动前完成 attach/mount
- **Admission Webhook**：可在 apiserver 准入阶段注入 Sidecar（如 Istio）

---

## 二、网络体系

### CNI 实现原理与特定场景优化

#### CNI 工作流程

1. 容器运行时创建 netns → 调用 CNI 二进制（ADD / DEL / CHECK）
2. CNI 负责：创建 veth pair、分配 IP（IPAM）、设置路由、配置 iptables/eBPF 规则
3. 返回 JSON（IP、网关、路由）给运行时

#### 主流方案对比

| 方案 | 特点 |
|------|------|
| Flannel VXLAN | 覆盖网络，易部署，性能损耗 ~20%，适合测试/小规模 |
| Calico BGP | 纯三层路由，性能接近原生，支持 NetworkPolicy，大规模首选 |
| Cilium eBPF | 绕过 iptables，极低延迟，支持 L7 策略，Hubble 可观测性 |
| Macvlan/SR-IOV | 网络性能接近裸机，适合延迟敏感型工作负载 |

#### 特定场景优化

- **高吞吐**：使用 SR-IOV + DPDK，CNI 选 Multus 多网卡
- **安全隔离**：Cilium + NetworkPolicy，Calico GlobalNetworkPolicy
- **大规模节点（>500）**：Calico BGP RR 模式，避免全互联 BGP
- **混合云**：Calico over IPIP/VXLAN 打通跨云网络

---

### iptables Pod 流量的 Tables 具体指什么？

#### 五张表与 Pod 流量的关系

| 表 | 作用 |
|----|------|
| raw | 流量最先进入，conntrack bypass 在此设置，一般不涉及 Pod 转发 |
| mangle | 修改数据包头（TTL、MARK、DSCP）；Calico 利用此表打 MARK 标记流量来源 |
| nat | ★ 最核心：PREROUTING（DNAT，Service ClusterIP → PodIP）；POSTROUTING（SNAT/MASQUERADE，Pod 出口流量替换 NodeIP） |
| filter | 真正的允许/拒绝；NetworkPolicy 规则由此实现；默认链 FORWARD 决定节点间 Pod 互通 |
| security | SELinux 规则，K8s 通常不用 |

#### ClusterIP Service 完整 iptables 路径

```
OUTPUT/PREROUTING
  → KUBE-SERVICES
    → KUBE-SVC-xxx（负载均衡，按概率跳转）
      → KUBE-SEP-xxx（DNAT 到 PodIP:Port）

POSTROUTING
  → KUBE-POSTROUTING
    → MASQUERADE（跨节点时 SNAT 为 NodeIP）
```

#### iptables 性能问题与优化

- **规则量问题**：每个 Service Endpoint 生成 ~10 条规则，5000 个 Endpoint → 5 万条规则，线性匹配性能差
- **优化方案一**：迁移到 IPVS 模式（`kube-proxy --proxy-mode=ipvs`），哈希查找 O(1)
- **优化方案二（终极）**：Cilium eBPF，完全旁路 iptables

---

### 非 NodePort 和 LB 的 Service 如何暴露给外部？

#### 方案一：Ingress Controller

- 部署 nginx-ingress / Traefik / Istio Gateway
- Ingress Controller 自身通过 NodePort 或 HostNetwork 暴露
- 支持 TLS 终止、路径路由、认证等 L7 功能

#### 方案二：ExternalIPs

```yaml
spec:
  externalIPs:
    - 1.2.3.4
```

- 将集群节点真实 IP 绑定给 Service，kube-proxy 直接在该 IP 上建监听
- 适合裸金属且 IP 资源固定的场景

#### 方案三：MetalLB（裸金属 LB）

- **Layer2 模式**：ARP 广播宣告 VIP，故障转移秒级
- **BGP 模式**：与路由器建立 BGP 会话，宣告 Service IP 段，高可用且支持 ECMP

#### 方案四：HostNetwork + DaemonSet

- Pod 设置 `hostNetwork: true`，直接绑定节点端口
- 适合边缘节点或需要极低延迟的网关场景
- 缺点：端口冲突风险，无法在同一节点部署多副本

#### 方案五：Gateway API（新一代标准）

- 替代 Ingress 的标准化 API，支持 TCPRoute / GRPCRoute / HTTPRoute
- Cilium、Contour、Envoy Gateway 均已支持

---

### 网络问题排查

#### 排查层次

1. **Pod 内部**：exec 进容器，curl/ping 目标，确认网络栈是否正常
2. **Pod ↔ Pod（同节点）**：检查 veth pair、arp、路由表
3. **Pod ↔ Pod（跨节点）**：检查 overlay 封包（tcpdump on flannel.1/tunl0），节点路由
4. **Pod → Service**：检查 kube-proxy iptables/ipvs 规则，Endpoint 是否 Ready
5. **Pod → 外网**：检查 MASQUERADE 规则，节点 NAT 网关

#### 常用命令

```bash
# 容器内连通性测试
kubectl exec <pod> -- curl -v http://<svc>:<port>

# 查看 iptables nat 规则
iptables -t nat -L KUBE-SERVICES -n --line-numbers

# 查看 IPVS 规则
ipvsadm -Ln

# 抓包分析
tcpdump -i eth0 host <podIP> -nn

# 确认 Endpoint 是否存在
kubectl get ep <svc>

# 查看 Pod 网络命名空间路由
kubectl exec <pod> -- ip route
kubectl exec <pod> -- ip addr
```

---

## 三、证书体系

### K8s 内部都有哪些证书，是一样的吗？

#### 证书分类

| 证书 | 用途 |
|------|------|
| etcd CA & 证书 | etcd 集群内部 peer 通信 + apiserver 访问 etcd；独立 CA，与 k8s CA 分离 |
| k8s CA（cluster CA） | 签发集群内所有组件证书，核心 CA |
| apiserver 服务端证书 | 客户端（kubectl/kubelet）TLS 验证 apiserver 身份 |
| apiserver-kubelet 客户端证书 | apiserver 主动访问 kubelet（exec/logs）时使用 |
| controller-manager 客户端证书 | 访问 apiserver 的身份证书 |
| scheduler 客户端证书 | 访问 apiserver 的身份证书 |
| kubelet 客户端证书 | kubelet 向 apiserver 注册、上报状态；由 TLS Bootstrap 自动签发 |
| kubelet 服务端证书 | apiserver 访问 kubelet 时验证 kubelet 身份 |
| front-proxy CA & 证书 | API Aggregation（如 metrics-server）专用，与主 CA 分离 |
| ServiceAccount 密钥对 | 非 x509 证书，是 RSA/ECDSA 密钥对，用于签发/验证 ServiceAccount JWT Token |

#### 核心区别

- **etcd 与 k8s CA 分离**：防止集群内其他服务绕过鉴权直接访问 etcd
- **front-proxy CA 独立**：防止聚合层滥用主 CA 权限
- **kubelet 证书自动轮转**：通过 Bootstrap 机制签发，有效期 1 年，kube-controller-manager 自动续签
- **SA 密钥对特殊性**：apiserver 用私钥签 Token，kube-controller-manager 和 apiserver 共享公钥验证

#### 证书路径（kubeadm 默认）

```bash
/etc/kubernetes/pki/ca.{crt,key}                # k8s 集群 CA
/etc/kubernetes/pki/etcd/ca.{crt,key}           # etcd CA
/etc/kubernetes/pki/front-proxy-ca.{crt,key}    # 聚合层 CA
/etc/kubernetes/pki/apiserver.{crt,key}         # apiserver 服务端证书
/etc/kubernetes/pki/apiserver-kubelet-client.{crt,key}  # apiserver 访问 kubelet
/etc/kubernetes/pki/sa.{pub,key}                # ServiceAccount 密钥对
```

---

### Service Account 的使用

#### 工作原理

- Pod 内默认挂载 `/var/run/secrets/kubernetes.io/serviceaccount/token`（Projected Volume）
- Token 是 OIDC JWT，包含 namespace / sa-name，apiserver 用 `sa.pub` 验证签名
- Kubernetes 1.22+ 使用 **BoundServiceAccountToken**（含 aud/exp 字段），更安全，有过期时间

#### 最佳实践

- **最小权限原则**：为每个 workload 单独创建 SA，绑定专属 Role / ClusterRole
- **禁用默认挂载**：`automountServiceAccountToken: false`（对不需要访问 API 的 Pod）
- **External Secrets**：敏感凭据不放 Secret，用 external-secrets + Vault / AWS SSM 动态注入
- **IRSA / Workload Identity**：云环境用云厂商的 SA 与 IAM 绑定，避免静态 AK/SK

---

## 四、监控：Prometheus 内部组件与抓取机制

### Prometheus 的内部组件和抓取机制

#### 核心组件

| 组件 | 职责 |
|------|------|
| Prometheus Server | 核心进程，包含 TSDB 存储引擎、Scrape 引擎、PromQL 查询引擎、Rules 引擎、HTTP API |
| Alertmanager | 独立服务，接收 Prometheus 推送的告警，处理去重/分组/路由/静默/抑制，发送到钉钉/PD/Email 等 |
| Pushgateway | 接受短生命周期 Job 推送指标（批处理任务），Prometheus 定期从 PGW 拉取 |
| Exporter | 将第三方系统指标转为 Prometheus 格式暴露，如 node-exporter、mysqld-exporter、kube-state-metrics |
| ServiceMonitor / PodMonitor | Prometheus Operator CRD，声明式配置抓取目标，避免手写 scrape_configs |

#### 抓取（Scrape）机制详解

- **Pull 模型**：Prometheus 主动 HTTP GET `/metrics` 端点，默认 15s 间隔
- **服务发现**：`kubernetes_sd_config` 通过 apiserver 发现 Pod / Service / Node / Endpoint
- **relabeling**：抓取前通过 `relabel_configs` 过滤/重命名标签（`__address__` 决定抓取目标）
- **metric_relabel_configs**：抓取后过滤/删除不需要的指标，降低存储压力
- **写入 TSDB**：结果写入本地 chunks + index + WAL，默认保留 15d

#### 大规模 Prometheus 方案

| 方案 | 说明 |
|------|------|
| Thanos | Sidecar 旁挂 Prometheus，数据上传 S3/GCS，Querier 联邦查询，支持无限存储 |
| VictoriaMetrics | Remote Write 接收，高压缩比，单机性能强，适合替换 Prometheus 本地存储 |
| Cortex / Mimir | 多租户，水平扩展，Grafana 官方维护 |
| Prometheus Operator Sharding | 按 label hash 分片，每个分片抓不同目标集，适合超大集群 |

---

## 五、大规模集群优化

### 大规模集群下怎么优化 K8s 集群？

#### 控制平面优化

- **etcd**：独立 SSD 节点，调大 `--quota-backend-bytes`（默认 2G → 8G），开启 `--auto-compaction-retention`，3/5 节点部署
- **apiserver**：多副本 + LB，调大 `--max-requests-inflight`（默认 400）和 `--max-mutating-requests-inflight`（默认 200）
- **Watch 缓存**：`--watch-cache-sizes` 按资源类型调大，减少 etcd 直读压力
- **Consistent Read**：启用 APIServer Watch Cache，减少 etcd 线性读

#### 节点与调度优化

- **大节点**：调大 kubelet `--max-pods`（默认 110 → 250），配置 topology manager + CPU manager 做 NUMA 感知调度
- **节点心跳**：调整 `--node-status-update-frequency`（默认 10s → 30s）减少 apiserver 写压力
- **Descheduler**：周期性驱逐失衡 Pod，配合 topologySpreadConstraints 实现均匀分布
- **弹性伸缩**：Cluster Autoscaler 或 Karpenter 实现节点自动扩缩容

#### 数据平面优化

- **kube-proxy**：切换至 IPVS 模式，或使用 Cilium eBPF 完全替代
- **CNI 选型**：大规模推荐 Calico BGP 或 Cilium eBPF
- **镜像分发**：Dragonfly / Kraken P2P 分发，避免镜像仓库成瓶颈
- **日志采集**：Vector / Fluent Bit 替代 Fluentd，资源消耗降低 60%+

#### 资源管理

- **LimitRange**：为 namespace 设置默认 request/limit，防止资源抢占
- **ResourceQuota**：限制 namespace 总资源用量
- **PodDisruptionBudget**：保证滚动更新/节点维护时最小可用副本数
- **VPA**：自动推荐并调整 Pod 的 request/limit

---

### 大规模优化思路：K8s 多 Region 方案

#### 方案一：联邦集群（KubeFed / Admiralty）

- 每个 Region 独立 k8s 集群，通过 Federation 控制面统一分发资源
- 优点：Region 故障完全隔离
- 缺点：跨 Region Service 发现复杂，运维成本高

#### 方案二：Cluster API 统一管理

- 用 Cluster API 统一生命周期管理多个 Region 集群
- 各 Region 独立 etcd & 控制平面，全局流量由 Global LB（Anycast / GSLB）分发

#### 方案三：单集群多 Zone（中等规模推荐）

- 同 Region 多 AZ 拓扑感知调度，`topologySpreadConstraints` 保证 AZ 均匀分布
- 跨 Zone Service 流量：设置 `service.kubernetes.io/topology-mode: Auto`（本 Zone 优先路由）

#### 跨 Region 核心挑战

| 挑战 | 解决方案 |
|------|----------|
| 网络延迟 | 控制平面禁止跨 Region 拉伸（etcd RTT 要求 < 10ms） |
| 数据一致性 | 有状态服务使用跨 Region 复制型存储（TiDB、CockroachDB） |
| 服务发现 | Istio ServiceEntry 或 Submariner 实现跨集群 Service 互通 |
| 统一配置下发 | ArgoCD + ApplicationSet 多集群 GitOps 部署 |

---

## 六、Namespace、Cgroup 与隔离机制

### Namespace 和 Cgroup 的关系与作用

#### Linux Namespace（隔离视图）

| Namespace | 隔离内容 |
|-----------|----------|
| Network NS | 每个 Pod 独立网络栈（IP、路由、iptables） |
| PID NS | 容器内进程 PID 从 1 开始，看不到宿主机进程 |
| Mount NS | 独立文件系统视图，容器内 /proc /sys 独立挂载 |
| UTS NS | 独立 hostname / domainname |
| IPC NS | 独立 System V IPC / POSIX 消息队列 |
| User NS | UID/GID 映射，Rootless Container 的核心机制 |

#### Cgroup（资源限额）

- **CPU**：`cpu.shares` 对应 request（相对权重），`cpu.cfs_quota_us` 对应 limit（硬上限）
- **Memory**：`memory.limit_in_bytes` 对应 limit，超限触发 OOM Killer
- **K8s QoS 类**：
  - `Guaranteed`（request == limit）：OOM 最后被杀
  - `Burstable`（request < limit）：中等优先级
  - `BestEffort`（无 request/limit）：OOM 最先被杀
- **Cgroup v2**：统一层级，支持 `memory.high` 软限制（MemoryQoS），推荐 K8s 1.25+ 启用

#### 两者关系

```
Linux Namespace  →  隔离"能看到什么"（进程、网络、文件系统）
Linux Cgroup     →  限制"能用多少"（CPU、内存、IO、网络带宽）
两者共同构成容器隔离的基础，缺一不可
```

  

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

## 超哥解释：
https://jimmysong.io/kubernetes-handbook/concepts/pod-lifecycle.html
挂起（Pending）：Pod 已被 Kubernetes 系统接受，但有一个或者多个容器镜像尚未创建。等待时间包括调度 Pod 的时间和通过网络下载镜像的时间，这可能需要花点时间。


## 说下POD跨主机通信的过程
k8s1.14版本之后之后都走ipvs，通过四层转发。
首先 pod1 通过自己的以太网设备 eth0 把数据包发送到关联到 root 命名空间的 veth0 上，--》网桥查找转发表，发现找不到，则会把包转发到默认路由（root 命名空间的 eth0 设备）--》然后数据包经过 eth0 就离开了 Node1，被发送到网络。--》数据包到达 Node2 后，首先会被 root 命名空间的 eth0 设备发现--》然后通过网桥把数据路由到虚拟设备 veth1,最终数据表会被流转到与 veth1 配对的另外一端。

## APIserver 出现大量5XX，可能是出现了什么问题？
可能是etcd异常，https证书过期，主机安全端口被占用等等（没碰到过）

## K8S 集群节点出现NotReady 应该如何排查？
kubectle describe node xxx 查看异常原因，可能是node节点的kubelet进程或者kube proxy进程等基础组件异常导致，也有可能是主机的资源不足导致（网络，磁盘，内存，cpu）

## 你做过哪些基于K8S云原生的业务改造？
开放题，可以参考阿里和腾讯大厂的。

## 在节点上有200个工作中的容器的情况下，如何优雅下线。
# 云原生与 K8S 知识整理

---

### 滚动窗口（Tumbling Window）

滚动窗口是将数据流按**固定大小、不重叠**地切分成连续的窗口。

- 每个窗口之间**没有重叠**
- 每条数据**只属于一个窗口**
- 窗口"滚动"向前，前一个结束，下一个立即开始

```
数据流: 1 2 3 4 5 6 7 8 9
窗口大小: 3

窗口1: [1 2 3]
窗口2:          [4 5 6]
窗口3:                   [7 8 9]
```

**典型用途：** 每5分钟统计一次订单量、每小时汇总一次日志。

---

### 滑动窗口（Sliding Window）

滑动窗口也是**固定大小**，但窗口每次只向前移动一个**步长（step）**，窗口之间**可以重叠**。

- 有两个参数：**窗口大小（size）** 和 **滑动步长（step）**
- 每条数据可能**属于多个窗口**
- 能捕捉到更细粒度的变化趋势

```
数据流: 1 2 3 4 5 6 7 8
窗口大小: 4，步长: 2

窗口1: [1 2 3 4]
窗口2:     [3 4 5 6]
窗口3:         [5 6 7 8]
```

**典型用途：** 计算过去30分钟的移动平均值（每1分钟刷新一次）、实时检测异常波动。

---

### 滚动窗口 vs 滑动窗口

| 特性 | 滚动窗口 | 滑动窗口 |
|------|----------|----------|
| 重叠 | ❌ 无重叠 | ✅ 有重叠 |
| 数据归属 | 每条数据属于1个窗口 | 每条数据可属于多个窗口 |
| 计算频率 | 低（窗口结束才计算） | 高（每步都计算） |
| 敏感度 | 较低 | 较高，能捕捉细微变化 |
| 资源消耗 | 少 | 多 |

> 滚动窗口 = 切片，不重叠，像翻书一页一页翻；滑动窗口 = 扫描，有重叠，像放大镜在数据上滑过。

---

### HTTP 4xx —— 客户端错误

问题出在**请求方（客户端）**，是你发的请求有问题。

| 状态码 | 名称 | 含义 |
|--------|------|------|
| 400 | Bad Request | 请求格式错误、参数非法，服务器看不懂 |
| 401 | Unauthorized | 未认证，需要登录（没带 token 或 token 无效） |
| 403 | Forbidden | 已认证但无权限，服务器拒绝执行 |
| 404 | Not Found | 资源不存在 |
| 405 | Method Not Allowed | 请求方法不对（比如应该 POST 却用了 GET） |
| 409 | Conflict | 资源冲突（比如用户名已存在） |
| 422 | Unprocessable Entity | 格式对但语义错（常见于参数校验失败） |
| 429 | Too Many Requests | 请求频率超限，被限流了 |

---

### HTTP 5xx —— 服务端错误

问题出在**服务器**，请求本身没问题，但服务器处理时出了问题。

| 状态码 | 名称 | 含义 |
|--------|------|------|
| 500 | Internal Server Error | 服务器内部错误，通用兜底错误 |
| 501 | Not Implemented | 服务器不支持该功能 |
| 502 | Bad Gateway | 网关收到上游服务器的无效响应（常见于代理/转发层） |
| 503 | Service Unavailable | 服务不可用，通常是宕机或过载 |
| 504 | Gateway Timeout | 网关等待上游响应超时 |

**401 vs 403：**
- **401**：你是**谁**我不知道 → 去登录
- **403**：你是**谁**我知道，但你**没资格** → 权限不足

**502 vs 504：**
- **502**：上游返回了，但返回的内容**是错的**
- **504**：上游**没有返回**，等超时了

> 4xx 是你的锅，5xx 是服务器的锅。

---

### 基于 K8S 云原生的业务改造实践

#### 一、单体应用容器化改造

**背景：** 原有 Java 单体应用部署在物理机/虚拟机，发布流程繁琐，环境差异导致"本地能跑、线上报错"。

**改造内容：**
- 编写多阶段 `Dockerfile`，将构建与运行环境分离，镜像体积从 1.2G 压缩到 180MB
- 将配置从代码中剥离，改用 `ConfigMap` + `Secret` 注入
- 应用改造为无状态，Session 迁移到 Redis，本地缓存改为分布式缓存
- 建立私有镜像仓库（Harbor），规范镜像 tag 管理（禁用 latest）

**收益：** 发布时间从 30 分钟降到 3 分钟，环境一致性问题基本消除。

#### 二、微服务治理上 K8S

**背景：** 微服务原来用 Eureka + Zuul 做注册发现和网关，维护成本高。

**改造内容：**
- 服务注册发现从 Eureka 迁移到 K8S 原生 `Service` + `CoreDNS`
- 网关从 Zuul 替换为 `Ingress-Nginx`，路由规则用 annotation 管理
- 引入 `Istio` 做服务网格，实现流量灰度、熔断、链路追踪（对接 Jaeger）
- 用 `VirtualService` 实现按比例流量切分，支持金丝雀发布

**收益：** 去掉了 Eureka/Zuul 两套中间件，基础设施维护量减少约 40%。

#### 三、CI/CD 流水线云原生化

**改造内容：**
- 基于 `Jenkins` + `Kubernetes Plugin`，构建任务动态创建 Pod，用完即销毁
- 后期引入 `ArgoCD`，实现 GitOps：代码合并即触发部署，K8S 状态与 Git 仓库保持一致
- 流水线阶段：代码扫描（SonarQube）→ 单元测试 → 镜像构建 → 推送镜像 → 自动部署到测试环境 → 人工审批 → 发布生产

```
Git Push
   ↓
Jenkins Pipeline（K8S Pod 动态调度）
   ↓
镜像 Push 到 Harbor
   ↓
ArgoCD 检测到变更 → 自动同步到集群
```

**收益：** 构建资源按需使用，节省约 60% 构建机资源。

#### 四、弹性伸缩与资源优化

**改造内容：**
- 配置 `HPA`（水平自动扩缩容），根据 CPU/QPS 自动扩缩 Pod 数量
- 引入 `VPA`（垂直自动扩缩容），自动推荐 Request/Limit 合理值
- 使用 `Cluster Autoscaler`，在云上实现节点级别的弹性伸缩
- 针对大促场景，配置 `KEDA`，基于消息队列积压量触发扩容

**踩坑点：**
- HPA 与 VPA 不能同时作用于同一资源维度，需要合理分工
- 扩容冷却时间配置不当会导致频繁抖动

#### 五、有状态服务改造（数据库/中间件）

**改造内容：**
- MySQL 使用 `Operator`（如 Percona Operator）管理，实现自动化备份、故障切换
- Redis 集群通过 `Redis Operator` 部署，配置持久化 `PVC`
- 中间件的数据目录挂载 `StorageClass`（Ceph RBD），与 Pod 生命周期解耦

**原则：** 无状态服务完全跑在 K8S，有状态服务优先用云厂商托管服务（RDS/ElastiCache）。

#### 六、可观测性体系建设

| 层次 | 方案 |
|------|------|
| 指标监控 | Prometheus + Grafana，ServiceMonitor 自动发现 |
| 日志采集 | DaemonSet 部署 Filebeat，统一收到 Elasticsearch |
| 链路追踪 | SkyWalking Agent 以 Init Container 方式注入，业务代码零改动 |
| 告警 | AlertManager 对接钉钉/飞书机器人 |

#### 七、整体改造收益

| 指标 | 改造前 | 改造后 |
|------|--------|--------|
| 发布时长 | 30 min | 3 min |
| 故障恢复时间 | 人工介入 ~15min | 自愈 ~30s |
| 资源利用率 | ~20% | ~65% |
| 大促扩容 | 提前 2 天扩机器 | 分钟级自动扩容 |
| 环境交付 | 1~2 天 | 按需秒级创建 Namespace |

---

### K8S 节点优雅下线（200 个容器在运行）

核心思路：不能直接关机，要让业务流量先撤走，Pod 再迁移，最后才下线节点。

整体流程：**封锁 → 驱逐 → 下线**

#### 第一步：cordon 封锁节点

```bash
kubectl cordon <node-name>
```

- 将节点标记为 `SchedulingDisabled`
- **已有 Pod 不受影响**，继续运行
- 新 Pod 不会再调度到该节点

#### 第二步：drain 驱逐所有 Pod

```bash
kubectl drain <node-name> \
  --ignore-daemonsets \       # 忽略 DaemonSet 管理的 Pod
  --delete-emptydir-data \    # 允许删除使用 emptyDir 的 Pod
  --grace-period=60 \         # 给每个 Pod 60 秒优雅退出时间
  --timeout=600s              # 整体超时 10 分钟
```

#### 第三步：优雅退出的关键保障机制

**PreStop Hook —— 给容器"打招呼"**

```yaml
lifecycle:
  preStop:
    exec:
      command: ["/bin/sh", "-c", "sleep 10"]
```

**terminationGracePeriodSeconds —— 宽限期**

```yaml
spec:
  terminationGracePeriodSeconds: 60
```

完整退出流程：

```
发送 SIGTERM
     ↓
执行 preStop Hook
     ↓
等待进程自己退出（处理完存量请求）
     ↓
超过 terminationGracePeriodSeconds → 强制 SIGKILL
```

**PodDisruptionBudget（PDB）—— 保障最低可用副本数**

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: my-service-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: my-service
```

#### 第四步：处理特殊情况

**DaemonSet Pod 无法驱逐**

```bash
# drain 时加 --ignore-daemonsets
# DaemonSet Pod 随节点下线自动消失，不需要手动处理
```

**有状态服务（StatefulSet）**

```bash
# 确认 PVC 已经挂载到持久存储，与节点解耦
# StatefulSet 的 Pod 驱逐顺序是反向的（从最大序号开始）
```

**Pod 有 finalizer 卡着删不掉**

```bash
# 手动强制删除（最后手段）
kubectl delete pod <pod-name> --force --grace-period=0
```

#### 完整操作流程

```bash
# 1. 确认节点上的 Pod 情况
kubectl get pods --all-namespaces \
  --field-selector spec.nodeName=<node-name>

# 2. 封锁节点
kubectl cordon <node-name>

# 3. 检查 PDB 情况
kubectl get pdb --all-namespaces

# 4. 执行驱逐
kubectl drain <node-name> \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --grace-period=60 \
  --timeout=600s

# 5. 确认节点上 Pod 已清空
kubectl get pods --field-selector spec.nodeName=<node-name>

# 6. 节点下线/维护（此时可以安全关机）

# 7. 维护完成后恢复调度
kubectl uncordon <node-name>
```

#### 常见问题与注意点

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| drain 卡住不动 | PDB 限制驱逐 | 等待新 Pod 在其他节点就绪后自动继续 |
| Pod 一直 Terminating | finalizer 未释放 | 检查 finalizer，必要时手动清除 |
| 流量有短暂报错 | preStop 时间不够 | 增大 preStop sleep 时间，与 LB 摘流时间对齐 |
| 200 个 Pod 驱逐太慢 | 默认串行驱逐 | 配置 `--pod-eviction-timeout`，或分批操作 |
| 节点资源不够承载迁移的 Pod | 集群资源不足 | 提前扩容其他节点，或在低峰期操作 |

#### 核心原则总结

```
cordon（封锁）           →  不让新 Pod 进来
drain（驱逐）            →  把老 Pod 赶走
PDB                      →  保证驱逐过程中业务不中断
preStop                  →  给每个容器体面退出的机会
terminationGracePeriod   →  兜底强杀的最后期限
```

## 多master如何保持一致性。有什么风险点吗？


在 Kubernetes 或其他分布式系统中，多 Master 架构的核心挑战是：**多个节点同时拥有决策权，如何保证它们的状态始终一致？**

---

### 核心机制：Raft 共识算法

多 Master 一致性的基石是 **Raft 算法**（etcd 使用），它把集群中的节点分成三种角色：

- **Leader（领导者）**：唯一接受写请求的节点，负责把数据同步给其他节点
- **Follower（跟随者）**：被动接收 Leader 的数据，不主动发起写操作
- **Candidate（候选者）**：Leader 挂掉后，Follower 转变为 Candidate 参与选举

#### 写操作的完整流程

1. 客户端把写请求发给 Leader
2. Leader 把这条变更记录到本地日志（但还未提交）
3. Leader 把日志同步给所有 Follower
4. **超过半数节点（Quorum）确认收到后**，Leader 才正式提交这条变更
5. Leader 通知所有 Follower 也提交，最终返回客户端"写入成功"

这就是所谓的 **"多数派写入"**——只要超过一半的节点确认，这条数据就被认为是安全的。

---

### Leader 选举机制

Leader 并不是永久的，它通过**心跳 + 超时机制**来维持地位：

- Leader 每隔一小段时间向所有 Follower 发送心跳
- 如果某个 Follower 在超时时间内没收到心跳，它就认为 Leader 挂了，自动转为 Candidate
- Candidate 向其他节点请求投票，**获得超过半数投票者当选新 Leader**
- 每次选举会产生一个递增的 **任期号（Term）**，防止脑裂期间出现两个 Leader

---

### Kubernetes 多 Master 的具体实现

Kubernetes 的控制面由几个关键组件组成，它们各有不同的一致性策略：

#### etcd 集群（数据层）

所有集群状态都存在 etcd 里，etcd 本身就是一个 Raft 集群，通常部署 3 或 5 个节点。写操作只走 Leader，读操作可以走任意节点（但可能读到略旧的数据，需要开启 `linearizable read` 才能保证强一致）。

#### kube-apiserver（无状态，多活）

apiserver 本身不存状态，所有状态都在 etcd。因此多个 apiserver 可以**同时对外提供服务**，通过前面的负载均衡器分发请求。任何一个挂掉，其他的继续工作。

#### kube-controller-manager 和 kube-scheduler（主备选举）

这两个组件如果多个同时运行，会产生重复操作（比如同时调度同一个 Pod）。所以它们采用**主备模式**：

- 多个实例同时运行，但只有**抢到锁的那一个**（Active）真正工作
- 其他实例处于 Standby 状态，持续尝试抢锁
- 抢锁机制通过在 etcd 中创建一条 Lease（租约）记录实现，谁持有租约谁就是 Active

---

### 风险点

#### 脑裂（Split Brain）

网络分区时，集群可能被切割成两个互相联系不上的子集，各自认为对方挂了，各自选出一个 Leader，产生两个"权威"，数据出现分叉。

**Raft 的防护手段**：强制要求写操作必须得到多数派确认。如果一边只有少数节点，它永远无法完成写入，从而避免两边同时写出不同数据。

> 举例：3 个节点的 etcd，网络一切二变成 1+2 两组。只有 2 个节点那组能形成多数派继续工作；1 个节点那组无法完成任何写操作，自动降级。

#### 脑裂期间的读操作

少数派节点虽然无法写入，但可能仍对外提供**旧数据的读服务**，导致客户端读到过期数据。要避免这个问题，需要客户端只读 Leader，或开启线性一致性读。

#### etcd 磁盘 I/O 瓶颈

Raft 日志的每次提交都需要 `fsync` 到磁盘（确保宕机重启后数据不丢），这对磁盘延迟极为敏感。如果 etcd 节点磁盘较慢（比如 HDD 或云盘 IOPS 不足），Leader 心跳会超时，触发不必要的重新选举，集群出现周期性抖动。

**缓解方法**：etcd 节点务必使用 SSD，或独立挂载高性能数据盘。

#### 频繁选举期间的短暂不可用

Leader 宕机 → 发现超时 → 开始选举 → 新 Leader 产生，这个过程通常需要 **150ms ~ 300ms**（Raft 默认超时），期间所有写操作会阻塞。对写敏感的场景（如频繁创建 Pod）会有短暂中断感知。

#### 时钟偏移导致的 Lease 异常

kube-controller-manager 和 kube-scheduler 的主备切换依赖 Lease 租约，租约的过期判断依赖节点的本地时钟。如果节点之间时钟偏差过大，可能导致：

- 本该失效的 Active 节点仍认为自己持有锁
- 两个节点短暂同时处于 Active 状态

**缓解方法**：集群所有节点必须配置 NTP 时间同步，时钟偏差控制在 **500ms 以内**。

#### Quorum 节点数的选择风险

| 集群规模 | 可容忍宕机节点数 | 说明 |
|---------|--------------|------|
| 3 个节点 | 1 个 | 最常见配置，最小高可用 |
| 5 个节点 | 2 个 | 容错性更强，但写性能略低 |
| 2 个节点 | 0 个 | **等于单点**，任意一个宕机即不可写 |
| 4 个节点 | 1 个 | 和 3 个节点容错一样，多花一台机器 |

偶数节点是常见的陷阱：4 个节点和 3 个节点的容错能力相同，但增加了一台机器的成本和选举复杂度。**生产环境推荐 3 或 5 个 etcd 节点**。

---

### 一致性与可用性的取舍

多 Master 架构遵循 **CAP 理论**中的 CP 路线（一致性 + 分区容忍，牺牲部分可用性）：

- 网络分区时，少数派节点拒绝写入，**宁可不可用也不写脏数据**
- 这对数据库、配置系统是正确选择，但业务层需要做好重试逻辑来应对短暂不可用





# k8s的控制器有哪些  
Kubernetes（k8s）中的控制器是核心组件，用于管理和确保集群资源状态与用户期望状态一致。以下是一些常见的控制器：

### 工作负载管理类
- **Deployment**
    - **简介**：主要用于管理无状态应用，支持滚动更新、回滚和伸缩等操作。它通过控制 ReplicaSet 来管理 Pod 副本。
    - **应用场景**：适用于 Web 服务器、微服务等无状态应用的部署与更新。
- **ReplicaSet**
    - **简介**：确保指定数量的 Pod 副本在任何时刻都处于运行状态。它是 Deployment 的底层实现，Deployment 借助它间接管理 Pod。
    - **应用场景**：在需要精确控制 Pod 副本数量时发挥作用。
- **StatefulSet**
    - **简介**：用于管理有状态应用，为每个 Pod 提供稳定的网络标识和持久存储，保证 Pod 的有序部署、扩展和删除。
    - **应用场景**：适合数据库（如 MySQL、MongoDB）等有状态应用。
- **DaemonSet**
    - **简介**：保证集群中每个节点（或指定节点）上运行且仅运行一个 Pod 副本。
    - **应用场景**：常用于节点级服务，如日志收集器（Fluentd）、监控代理（Prometheus Node Exporter）。
- **Job**
    - **简介**：用于执行一次性任务，任务完成后 Pod 自动终止。
    - **应用场景**：适用于批处理作业，如数据导入、报表生成。
- **CronJob**
    - **简介**：定时执行的 Job，依据指定时间间隔运行任务。
    - **应用场景**：可用于定期任务，如每天凌晨的数据备份。

### 服务发现与负载均衡类
- **Service**
    - **简介**：为一组 Pod 提供统一的网络访问入口，实现服务发现和负载均衡。
    - **应用场景**：在微服务架构中，为前端应用提供后端服务的访问接口。
- **Ingress**
    - **简介**：用于管理集群外部到内部服务的 HTTP 和 HTTPS 路由，可实现基于域名、路径的流量分发。
    - **应用场景**：当需要对外暴露多个服务时，通过 Ingress 进行统一的路由管理。

### 配置管理类
- **ConfigMap**
    - **简介**：用于存储非敏感的配置数据，可将配置信息以键值对形式存储，并注入到 Pod 中。
    - **应用场景**：为应用程序提供配置文件、环境变量等。
- **Secret**
    - **简介**：用于存储敏感数据，如密码、令牌、密钥等，以加密形式存储和使用。
    - **应用场景**：在 Pod 中使用 Secret 来存储和访问数据库密码、API 密钥等。

### 节点管理类
- **Node Controller**
    - **简介**：负责监控节点的状态，处理节点的添加、删除和故障等情况。
    - **应用场景**：当节点出现故障时，Node Controller 会将该节点上的 Pod 迁移到其他节点。
- **PersistentVolume Controller**
    - **简介**：管理持久化存储卷（PersistentVolume，PV）和持久化存储卷声明（PersistentVolumeClaim，PVC），实现存储资源的分配和回收。
    - **应用场景**：为有持久化存储需求的应用（如数据库）分配和管理存储资源。

### 其他
- **Horizontal Pod Autoscaler（HPA）**
    - **简介**：根据 Pod 的 CPU、内存使用率或自定义指标自动调整 Pod 副本数量，实现应用的自动伸缩。
    - **应用场景**：在流量高峰时自动增加 Pod 副本，低谷时减少副本，提高资源利用率。
- **Namespace Controller**
    - **简介**：管理命名空间的生命周期，确保命名空间的创建、删除和清理操作正确执行。
    - **应用场景**：在多租户环境中，使用命名空间隔离不同用户或项目的资源。  


# 三、Operator 是什么？用 Operator 开发过什么软件？  
#### 1. Operator 定义  
**Operator** 是一种基于 Kubernetes 的 **自定义资源（CRD）+ 控制器（Controller）** 的模式，用于封装复杂应用的部署、配置和生命周期管理逻辑。它通过扩展 Kubernetes API，让用户可以像操作内置资源（如 Deployment）一样管理复杂应用（如数据库、中间件）。  

#### 2. 典型案例（基于 Operator 开发的软件）  
- **Prometheus Operator**： 自动化管理 Prometheus 监控集群，包括部署、配置和扩展。  
- **Cert-Manager**： 自动管理 Kubernetes 集群的 SSL/TLS 证书（如 Let’s Encrypt 证书）。  
- **etcd Operator**： 简化 etcd 分布式键值存储的部署和维护（如备份、故障恢复）。  
- **MySQL Operator**： 自动化管理 MySQL 数据库集群，支持主从复制、故障转移。  
- **Knative**： 基于 Kubernetes 的 Serverless 框架，通过 Operator 管理函数计算和事件驱动架构。  


# 四、工作中如何使用 Kubernetes？如何调度 Pod？  
#### 1. Kubernetes 使用场景（示例）  
- **应用部署**：通过 Deployment/StatefulSet 部署微服务、Web 应用、数据库等，定义镜像、资源配额、环境变量等。  
- **服务发现与负载均衡**：使用 Service 暴露应用，通过 Ingress 管理外部流量路由。  
- **弹性伸缩**：基于 CPU/内存指标或自定义指标（如队列长度）自动扩展 Pod 副本。  
- **配置管理**：通过 ConfigMap/Secret 注入配置文件或敏感信息，避免硬编码。  
- **监控与日志**：集成 Prometheus + Grafana 监控集群状态，通过 Fluentd/EFK 收集 Pod 日志。  

#### 2. Pod 调度的常用方法  
- **标签与选择器**：通过 `nodeSelector` 直接指定 Pod 调度到具有特定标签的节点（如 `disk=ssd`）。  
- **节点亲和性（Affinity）**：  
  - **软亲和性（Prefer）**：尽量调度到符合条件的节点（如优先选择内存大的节点）。  
  - **硬亲和性（Required）**：必须调度到符合条件的节点（如仅调度到 GPU 节点）。  
- **污点与容忍（Taints/Tolerations）**：  
  - 节点设置“污点”（如标记为专用节点），Pod 通过“容忍”声明允许调度到该节点。  
- **资源配额与限制**：通过 `requests`/`limits` 定义 Pod 的 CPU/内存需求，Kubernetes Scheduler 基于资源可用性调度。  
- **优先级与抢占**：为 Pod 定义优先级，低优先级 Pod 可被高优先级 Pod 抢占，确保关键应用优先调度。  


### 五、创建一个 Deployment 部署到集群时发生了什么？  
1. **用户提交请求**：  
   - 通过 `kubectl apply -f deployment.yaml` 或 API 调用，将 Deployment 资源定义提交到 API Server。  

2. **API Server 处理请求**：  
   - 验证 Deployment 定义是否符合规范，存储到 etcd 中，并通知相关组件（如控制器管理器）。  

3. **Deployment 控制器触发逻辑**：  
   - 根据 `spec.replicas` 创建对应的 **ReplicaSet**（若不存在），或更新现有 ReplicaSet 的副本数和 Pod 模板。  

4. **ReplicaSet 管理 Pod 副本**：  
   - ReplicaSet 确保集群中运行指定数量的 Pod（通过标签选择器匹配 Pod）。若当前副本数不足，创建新 Pod；若过多，删除多余 Pod。  

5. **Scheduler 调度 Pod**：  
   - Scheduler 接收未调度的 Pod，根据调度策略（如节点亲和性、资源可用性）选择合适的节点。  

6. **节点执行 Pod**：  
   - 目标节点的 kubelet 接收到调度结果，拉取 Pod 定义中的镜像（可能从私有仓库或 Docker Hub），启动容器。  

7. **状态反馈与更新**：  
   - Pod 启动后，通过 kubelet 向 API Server 报告状态（Running、Ready 等）。  
   - Deployment 控制器监控 ReplicaSet 和 Pod 的状态，更新 Deployment 的 `status` 字段（如可用副本数、更新进度）。  

8. **滚动更新（若配置）**：  
   - 若 Deployment 配置了滚动更新策略（默认策略），会逐步替换旧版本 Pod，确保服务不中断。  



# 当一个节点处于not-ready状态时，发生了什么 如何排查


您可能是指 Kubernetes 节点处于 **NotReady** 状态（拼写可能有误，正确状态为 `Ready` 或 `NotReady`）。当节点状态为 `NotReady` 时，表示节点无法正常参与集群工作，Pod 可能无法调度或运行在该节点上。以下是常见原因及排查方向：


### 一、核心原因分类
#### 1. **节点组件故障**
   - **kubelet 服务异常**：  
     kubelet 是节点上的核心代理，负责与 API Server 通信并管理 Pod。若 kubelet 进程崩溃、未启动或与 API Server 断开连接，节点会失去心跳，状态变为 `NotReady`。  
     - 检查节点上 kubelet 服务状态：  
       ```bash
       systemctl status kubelet  # 或 docker ps 查看 kubelet 容器状态
       ```
   - **容器运行时故障**：  
     如 Docker、containerd 等运行时服务异常，导致无法创建或运行容器。  
     - 检查运行时服务状态：  
       ```bash
       systemctl status docker  # 或 crictl info 查看容器运行时状态
       ```

#### 2. **网络通信问题**
   - **节点与 API Server 通信中断**：  
     节点无法连接到 Kubernetes API Server（如网络防火墙、IP 冲突、DNS 解析失败），导致 kubelet 无法上报状态。  
   - **节点间网络不通**：  
     节点无法与其他节点通信（如 Flannel、Calico 等网络插件配置错误），影响 Pod 网络功能。  
   - **CNI 插件异常**：  
     网络插件（如 CNI）未正确安装或运行，导致 Pod 无法分配 IP 地址，kubelet 报告节点网络不可用。  
     - 检查 CNI 插件日志：  
       ```bash
       journalctl -u kubelet | grep -i cni  # 或查看 /var/log/cni 目录日志
       ```

#### 3. **资源不足或配置问题**
   - **系统资源耗尽**：  
     节点的 CPU、内存、磁盘空间（`/var/lib/docker` 或 `/var/lib/kubelet` 分区）不足，或 PID 限制达到上限，导致 kubelet 无法正常工作。  
     - 查看节点资源使用情况：  
       ```bash
       kubectl describe node <节点名> | grep -i 'allocatable\|condition'
       df -h  # 检查磁盘空间
       ```
   - **内核参数或系统配置不匹配**：  
     如桥接网络配置未启用（`net.bridge.bridge-nf-call-iptables` 未设置为 1），或 SELinux 策略阻止容器运行。  
     - 验证内核参数：  
       ```bash
       sysctl net.bridge.bridge-nf-call-iptables  # 需为 1
       ```

#### 4. **节点被标记为不可调度（Cordon）**
   - 管理员手动执行 `kubectl cordon <节点名>` 将节点标记为不可调度，此时节点状态可能显示为 `Ready,SchedulingDisabled`，但严格来说不算 `NotReady`。若同时存在其他问题，可能变为 `NotReady`。

#### 5. **污点（Taint）未被容忍**
   - 节点被设置了 **污点**（如 `node.kubernetes.io/not-ready:NoSchedule`），且 Pod 未配置对应的 **容忍（Toleration）**，导致节点对 Pod 不可用（但节点自身状态可能仍为 `Ready`，需结合具体污点类型判断）。

#### 6. **节点硬件或基础设施故障**
   - 物理机/虚拟机故障（如磁盘损坏、网络接口故障）、云服务商底层问题（如 AWS EC2 实例故障）。


### 二、排查步骤
1. **查看节点详细状态**：  
   ```bash
   kubectl describe node <节点名>
   # 重点关注 Conditions 部分，特别是 "Ready" 状态的 Reason 和 Message
   ```
   - 若 `Reason` 为 `KubeletNotReady`：kubelet 未正常运行或通信失败。  
   - 若 `Reason` 为 `NetworkPluginNotReady`：CNI 插件未就绪（如 Pod 网络未配置）。  

2. **检查 kubelet 日志**：  
   ```bash
   journalctl -u kubelet -f  # 查看 kubelet 实时日志，寻找错误或警告信息
   ```

3. **验证容器运行时**：  
   ```bash
   crictl version  # 检查容器运行时版本是否与集群兼容
   docker info  # 若使用 Docker，检查 Docker 状态是否正常
   ```

4. **网络连通性测试**：  
   - 节点能否访问 API Server 的 IP 和端口（通常为 6443）。  
   - 节点间能否通过 Pod 网络 CIDR 互相通信（如 Flannel 的 VXLAN 隧道是否正常）。  

5. **资源可用性检查**：  
   - 确保节点有足够的 CPU、内存、磁盘空间，且 `kubelet` 配置的根目录（如 `/var/lib/kubelet`）未被占满。  


### 三、常见修复措施
- **重启节点组件**：重启 kubelet 或容器运行时服务。  
- **清理节点资源**：释放磁盘空间、杀死异常进程、调整资源配额。  
- **修复网络配置**：重新部署 CNI 插件、检查网络策略或防火墙设置。  
- **解除节点隔离**：若手动 cordon，执行 `kubectl uncordon <节点名>`（需先解决底层问题）。  
- **节点排水（Drain）**：若节点需要维护，先驱逐 Pod：  
  ```bash
  kubectl drain <节点名> --force --ignore-daemonsets
  ```


### 总结
节点 `NotReady` 状态通常是由组件故障、网络问题、资源不足或配置错误引起。通过 `kubectl describe node` 和日志分析定位具体原因，再针对性修复即可。若为云环境，还需结合云服务商的监控（如 EC2 实例状态、网络负载）进一步排查。

##
# ------------------------------------------------------------------------

# Kubernetes 核心知识体系

> 大规模集群优化 · 网络 · 证书 · 监控 · 故障排查

---

## 一、在大规模集群下，怎么优化 K8s 集群？

### 控制平面优化

| 组件 | 优化要点 |
|------|----------|
| etcd | 独立 SSD 节点；`--quota-backend-bytes` 调至 8G；开启 `--auto-compaction-retention`；奇数节点（3/5）部署 |
| apiserver | 多副本 + LB；调大 `--max-requests-inflight`（默认 400）和 `--max-mutating-requests-inflight`（默认 200）；启用 Watch Cache 减少 etcd 直读 |
| controller-manager | 调大 `--concurrent-*` 并发参数，如 `--concurrent-deployment-syncs` |
| scheduler | 启用 scheduling-framework 插件；开启 NodeName 快速路径 |

### 节点与调度优化

- **大节点**：调大 kubelet `--max-pods`（110 → 250），开启 CPU Manager + Topology Manager 做 NUMA 感知
- **节点心跳**：`--node-status-update-frequency` 从 10s 调至 30s，减少 apiserver 写压力
- **Descheduler**：周期性驱逐失衡 Pod，配合 `topologySpreadConstraints` 保证均匀分布
- **弹性伸缩**：Cluster Autoscaler 或 Karpenter 自动扩缩节点

### 数据平面优化

- **kube-proxy**：切换 IPVS 模式（O(1) 哈希查找替代线性 iptables），或用 Cilium eBPF 完全替代
- **CNI 选型**：大规模推荐 Calico BGP 或 Cilium eBPF
- **镜像分发**：Dragonfly / Kraken P2P，避免镜像仓库成瓶颈
- **日志采集**：Vector / Fluent Bit 替代 Fluentd，资源消耗降低 60%+

### 资源管理

| 机制 | 作用 |
|------|------|
| LimitRange | 为 namespace 设置默认 request/limit，防止资源抢占 |
| ResourceQuota | 限制 namespace 总资源用量 |
| PodDisruptionBudget | 保障滚动更新/节点维护时的最小可用副本 |
| VPA | 自动推荐并调整 Pod 的 request/limit |

---

## 二、非 NodePort 和 LB 的 Service 怎么暴露给外部集群？

### 方案一：Ingress Controller

- 部署 nginx-ingress / Traefik / Istio Gateway，Controller 自身通过 NodePort 或 HostNetwork 暴露
- 支持 TLS 终止、路径路由、认证等 L7 功能，是最常见的生产方案

### 方案二：ExternalIPs

```yaml
spec:
  externalIPs:
    - 1.2.3.4
```

- 将节点真实 IP 绑定给 Service，kube-proxy 直接在该 IP 上建监听
- 适合裸金属且 IP 资源固定的场景

### 方案三：MetalLB（裸金属 LB）

- **Layer2 模式**：ARP 广播宣告 VIP，故障转移秒级，但单节点承流
- **BGP 模式**：与路由器建立 BGP 会话宣告 Service IP 段，支持 ECMP 多路负载，高可用

### 方案四：HostNetwork + DaemonSet

- Pod 设置 `hostNetwork: true`，直接绑定节点端口
- 适合边缘网关或极低延迟场景
- 缺点：端口冲突风险，同节点无法部署多副本

### 方案五：Gateway API（新一代标准）

- 替代 Ingress 的标准化 API，支持 HTTPRoute / TCPRoute / GRPCRoute
- Cilium、Contour、Envoy Gateway 均已支持，逐步成为主流

---

## 三、K8s 内部都有哪些证书，是一样的么？

### 证书全览

| 证书 | 所属 CA | 用途 |
|------|---------|------|
| etcd peer 证书 | etcd CA | etcd 节点间互信通信 |
| etcd server 证书 | etcd CA | apiserver 访问 etcd |
| apiserver 服务端证书 | k8s CA | kubectl/kubelet 验证 apiserver 身份 |
| apiserver-kubelet-client 证书 | k8s CA | apiserver 主动访问 kubelet（exec/logs） |
| controller-manager 客户端证书 | k8s CA | controller-manager 访问 apiserver |
| scheduler 客户端证书 | k8s CA | scheduler 访问 apiserver |
| kubelet 客户端证书 | k8s CA | kubelet 注册节点、上报状态；TLS Bootstrap 自动签发 |
| kubelet 服务端证书 | k8s CA | apiserver 访问 kubelet 时验证其身份 |
| front-proxy 证书 | front-proxy CA | API Aggregation（metrics-server 等）专用 |
| ServiceAccount 密钥对 | 无（非 x509） | RSA/ECDSA 密钥对，签发/验证 SA JWT Token |

### 核心区别

- **etcd CA 与 k8s CA 分离**：防止集群内其他服务绕过鉴权直接访问 etcd
- **front-proxy CA 独立**：防止聚合层滥用主 CA 权限伪造请求
- **kubelet 证书自动轮转**：有效期 1 年，controller-manager 在到期前自动 approve CSR 续签
- **SA 密钥对特殊性**：不是证书，apiserver 用私钥签 Token，各组件用公钥验证

### 默认路径（kubeadm）

```bash
/etc/kubernetes/pki/ca.{crt,key}                       # k8s 集群 CA
/etc/kubernetes/pki/etcd/ca.{crt,key}                  # etcd CA
/etc/kubernetes/pki/front-proxy-ca.{crt,key}           # 聚合层 CA
/etc/kubernetes/pki/apiserver.{crt,key}                # apiserver 服务端
/etc/kubernetes/pki/apiserver-kubelet-client.{crt,key} # apiserver→kubelet
/etc/kubernetes/pki/sa.{pub,key}                       # SA 密钥对

# 证书运维
kubeadm certs check-expiration   # 查看到期时间
kubeadm certs renew all          # 手动续签全部
```

---

## 四、K8s Master 都有哪些组件，可以随意部署么？

### 核心组件职责

| 组件 | 职责 |
|------|------|
| kube-apiserver | 集群唯一入口，所有读写操作经过此处鉴权/准入/持久化 |
| etcd | 分布式 KV，存储全部集群状态，唯一有状态的核心组件 |
| kube-controller-manager | 运行 Deployment、ReplicaSet、Node、Endpoint 等内置控制器 |
| kube-scheduler | 监听 Unscheduled Pod，执行过滤 + 打分，完成节点绑定 |
| cloud-controller-manager | 对接云厂商（LB 创建、Route 同步、Node 生命周期），裸金属可省略 |

### 部署约束

| 组件 | HA 模式 | 关键约束 |
|------|---------|----------|
| apiserver | Active-Active | 无状态，可任意水平扩展，前置 LB 即可 |
| etcd | Raft 选主 | 必须奇数节点（3/5/7）；禁止跨 Region 拉伸（RTT < 10ms）；生产独立部署 |
| controller-manager | Leader Election | 多副本只有一个 leader 工作，其余热备；不可跨集群共享 |
| scheduler | Leader Election | 同上 |
| cloud-controller-manager | - | 与云 API 强依赖，裸金属不部署 |

---

## 五、Prometheus 的内部组件和抓取机制

### 核心组件

| 组件 | 职责 |
|------|------|
| Prometheus Server | 包含 Scrape 引擎、TSDB 存储、PromQL 查询引擎、Rules 引擎、HTTP API |
| Alertmanager | 独立服务，接收告警后做去重/分组/路由/静默/抑制，发送到钉钉/PagerDuty/Email |
| Pushgateway | 接受短生命周期 Job 主动推送指标，Prometheus 再从 PGW 定期拉取 |
| Exporter | 将第三方系统指标转为 `/metrics` 格式，如 node-exporter、kube-state-metrics |
| ServiceMonitor / PodMonitor | Prometheus Operator CRD，声明式配置抓取目标，替代手写 `scrape_configs` |

### 抓取（Scrape）机制

- **Pull 模型**：Prometheus 主动 HTTP GET `/metrics`，默认间隔 15s
- **服务发现**：`kubernetes_sd_config` 通过 apiserver 动态发现 Pod / Node / Service / Endpoint
- **relabel_configs**：抓取前过滤目标、重写标签，`__address__` 决定最终抓取地址
- **metric_relabel_configs**：抓取后丢弃不需要的指标，降低 TSDB 存储压力
- **写入 TSDB**：chunk 压缩存储 + WAL 防崩溃丢数，默认保留 15d

### 大规模存储扩展方案

| 方案 | 适用场景 |
|------|----------|
| Thanos Sidecar | 数据上传 S3/GCS，Querier 全局查询，无限存储 |
| VictoriaMetrics | Remote Write 接收，高压缩比，单机性能极强 |
| Cortex / Mimir | 多租户、水平扩展，Grafana 官方维护 |
| Prometheus Sharding | Operator 按 label hash 分片抓取，适合超大集群 |

---

## 六、大规模优化思路：K8s 多 Region 方案

### 方案一：联邦集群（KubeFed / Admiralty）

- 每个 Region 独立集群，Federation 控制面统一下发资源
- 优点：Region 故障完全隔离
- 缺点：跨集群 Service 发现复杂，运维成本高

### 方案二：Cluster API 统一管理

- 用 Cluster API 统一多 Region 集群的生命周期（创建/升级/销毁）
- 各 Region 独立控制平面，全局流量由 GSLB / Anycast 分发

### 方案三：单集群多 AZ（中等规模推荐）

- 同 Region 多可用区，用 `topologySpreadConstraints` 保证 Pod 跨 AZ 均匀分布
- 开启 `service.kubernetes.io/topology-mode: Auto` 实现 Service 流量本 Zone 优先

### 跨 Region 核心挑战

| 挑战 | 解决方案 |
|------|----------|
| 控制平面延迟 | 禁止跨 Region 拉伸 etcd，每 Region 独立控制平面 |
| 跨集群服务发现 | Submariner 或 Istio ServiceEntry 打通跨集群 Service |
| 有状态数据同步 | TiDB / CockroachDB 跨 Region 复制型数据库 |
| 统一配置下发 | ArgoCD + ApplicationSet 多集群 GitOps |

---

## 七、iptables Pod 流量的 Tables 具体指什么？

### 五张表的作用

| 表 | 处理时机 | 与 Pod 的关系 |
|----|----------|---------------|
| raw | 最先处理 | 设置 conntrack bypass（NOTRACK），Pod 流量一般不涉及 |
| mangle | 修改包头 | Calico 在此打 MARK 标记流量来源节点/Pod |
| **nat** | 地址转换 | ★ 最核心：PREROUTING 做 DNAT（ClusterIP→PodIP），POSTROUTING 做 SNAT/MASQUERADE |
| filter | 过滤放行 | NetworkPolicy 在此实现允许/拒绝；FORWARD 链控制跨节点 Pod 互通 |
| security | SELinux 标签 | K8s 通常不使用 |

### ClusterIP Service 完整转发路径

```
入方向（PREROUTING）：
  KUBE-SERVICES
    → KUBE-SVC-xxx（按概率负载均衡）
      → KUBE-SEP-xxx（DNAT: ClusterIP:Port → PodIP:Port）

出方向（POSTROUTING）：
  KUBE-POSTROUTING
    → MASQUERADE（跨节点时 SNAT 为 NodeIP，同节点不 SNAT）
```

### 性能瓶颈与优化

- **问题**：每个 Endpoint 生成 ~10 条规则，5000 个 Endpoint → 5 万条规则，链式线性匹配性能差
- **优化一**：切换 IPVS 模式（`--proxy-mode=ipvs`），哈希表 O(1) 查找
- **优化二（终极）**：Cilium eBPF，完全绕过 iptables，内核层直接转发

```bash
iptables -t nat -L KUBE-SERVICES -n --line-numbers   # 查看 nat 规则
iptables -t filter -L FORWARD -n                      # 查看转发规则
ipvsadm -Ln                                           # 查看 IPVS 规则
```

---

## 八、综合专题

### Pod 完整创建流程

1. `kubectl apply` → apiserver 鉴权 / 准入控制（Webhook）/ 写入 etcd
2. ReplicaSet Controller watch 到 Pod 缺少 → 创建 Pod 对象（状态 Pending）
3. kube-scheduler watch Unscheduled Pod → Filter（过滤）→ Score（打分）→ Bind（写绑定关系）
4. kubelet watch 到绑定本节点的 Pod → 调用容器运行时创建 **Pause 容器**（占位 Network NS）
5. 调用 **CNI 插件**（ADD 命令）配置网络（veth pair / IP / 路由 / iptables 规则）
6. CSI 完成存储 attach/mount → 拉取镜像 → 启动 Init 容器 → 启动业务容器
7. kubelet 执行 livenessProbe / readinessProbe → 就绪后 Endpoint Controller 将 PodIP 加入 Service

### CNI 实现原理与场景优化

#### 工作原理

- 容器运行时创建 netns → 按顺序调用 CNI 二进制（ADD / DEL / CHECK）
- CNI 执行：创建 veth pair → IPAM 分配 IP → 设置路由 → 下发 iptables/eBPF 规则
- 返回 JSON 结果（IP、Gateway、Route）给运行时

#### 特定场景优化

| 场景 | 方案 |
|------|------|
| 超大规模（>500 节点） | Calico BGP RR 模式，避免全互联 |
| 极低延迟 | SR-IOV + Multus 多网卡，绕过内核协议栈 |
| 安全隔离 | Cilium + NetworkPolicy L3/L4/L7 全层策略 |
| 混合云互通 | Calico over IPIP 或 Submariner 跨集群 |

### Master 组件功能与优化

| 组件 | 关键优化点 |
|------|-----------|
| apiserver | 多副本 + LB；调大 `--max-requests-inflight`；开启 Watch Cache |
| etcd | 独立 SSD；`--quota-backend-bytes` 调大；开启自动压缩；奇数节点 |
| controller-manager | 调大 `--concurrent-*` 并发参数 |
| scheduler | 启用 scheduling-framework 插件；开启 NodeName 快速路径 |

### 网络问题排查

```bash
# 容器内连通性
kubectl exec <pod> -- curl -v http://<svc>:<port>
kubectl exec <pod> -- ip route && ip addr

# 查看 Endpoint 是否就绪
kubectl get ep <svc-name>

# 检查 iptables nat 规则
iptables -t nat -L KUBE-SERVICES -n --line-numbers

# 检查 IPVS 规则
ipvsadm -Ln

# 抓包分析（跨节点 overlay）
tcpdump -i flannel.1 -nn host <podIP>
tcpdump -i eth0 -nn host <podIP>
```

#### 排查层次

1. **Pod 内部**：exec 进容器，curl/ping 目标，确认网络栈正常
2. **Pod ↔ Pod（同节点）**：检查 veth pair、arp、路由表
3. **Pod ↔ Pod（跨节点）**：检查 overlay 封包（tcpdump on flannel.1/tunl0），节点路由
4. **Pod → Service**：检查 kube-proxy iptables/ipvs 规则，Endpoint 是否 Ready
5. **Pod → 外网**：检查 MASQUERADE 规则，节点 NAT 网关

### ServiceAccount 使用

- Pod 默认挂载 `/var/run/secrets/kubernetes.io/serviceaccount/token`（Projected Volume）
- Token 是 OIDC JWT，含 namespace / sa-name / aud / exp，apiserver 用 `sa.pub` 验证
- K8s 1.22+ 启用 **BoundServiceAccountToken**，Token 有过期时间，更安全

#### 最佳实践

- 最小权限原则：每个 workload 独立 SA，绑定专属 Role / ClusterRole
- 不需要访问 API 的 Pod 设置 `automountServiceAccountToken: false`
- 云环境用 IRSA（AWS）/ Workload Identity（GCP）替代静态密钥

### Namespace 与 Cgroup

#### Linux Namespace（隔离视图）

| Namespace | 隔离内容 |
|-----------|----------|
| Network NS | 每个 Pod 独立网络栈（IP / 路由 / iptables） |
| PID NS | 容器内 PID 从 1 开始，看不到宿主机进程 |
| Mount NS | 独立文件系统，/proc /sys 独立挂载 |
| UTS NS | 独立 hostname |
| IPC NS | 独立消息队列/共享内存 |
| User NS | UID/GID 映射，Rootless 容器核心 |

#### Cgroup（资源限额）

- **CPU**：`cpu.shares` → request（相对权重），`cpu.cfs_quota_us` → limit（硬上限）
- **Memory**：`memory.limit_in_bytes` → limit，超限触发 OOM Killer
- **QoS 类别**（影响 OOM 被杀顺序）：
  - `Guaranteed`（request == limit）→ 最后被杀
  - `Burstable`（request < limit）→ 中等优先级
  - `BestEffort`（无 request/limit）→ 最先被杀
- **Cgroup v2**：统一层级，支持 `memory.high` 软限制（MemoryQoS），K8s 1.25+ 推荐启用

### 证书相关

```
k8s CA          → 签发 apiserver、kubelet、controller-manager、scheduler 证书
etcd CA         → 签发 etcd peer、etcd server、apiserver-etcd-client 证书（与 k8s CA 完全分离）
front-proxy CA  → 签发 API Aggregation 层证书（metrics-server 等扩展 API）
SA 密钥对       → 非证书，RSA 密钥对，私钥签 Token，公钥验证
```

- **kubelet 证书轮转**：TLS Bootstrap 自签，controller-manager 到期前自动 approve CSR 续签
- **手动续签**：`kubeadm certs renew all`
- **查看到期时间**：`kubeadm certs check-expiration`

# 当一个pod出现问题，如何进行排查

当一个 Pod 出现错误时，可按以下步骤进行排查：

### 1. 查看 Pod 基本信息与状态
使用 `kubectl get pods` 命令查看 Pod 的基本信息，尤其关注 `STATUS` 列。若状态异常，可进一步使用 `kubectl describe pod <pod-name>` 查看详细描述，其中包含了 Pod 的事件信息，能帮助你了解 Pod 的生命周期内发生了什么。
```bash
# 查看所有 Pod 的状态
kubectl get pods
# 查看特定 Pod 的详细信息
kubectl describe pod <pod-name>
```

### 2. 根据不同状态进行分析
- **Pending**：
    - **原因**：Pod 无法调度到节点上，可能是节点资源不足、节点选择器不匹配、污点和容忍度设置问题等。
    - **排查方法**：通过 `kubectl describe pod <pod-name>` 查看事件信息，若提示 `FailedScheduling`，会给出具体的调度失败原因。同时，使用 `kubectl describe nodes` 查看节点的资源使用情况。
- **ContainerCreating**：
    - **原因**：容器正在创建但未成功，可能是镜像拉取失败、容器运行时故障等。
    - **排查方法**：查看事件信息，若提示 `Failed to pull image`，则说明镜像拉取失败，可使用 `kubectl describe pod <pod-name>` 查看具体的镜像拉取错误信息。也可以通过 `docker pull <image-name>` 在节点上手动尝试拉取镜像，查看是否存在权限或网络问题。
- **Error** 或 **CrashLoopBackOff**：
    - **原因**：容器启动后异常退出，可能是应用程序本身的问题、配置错误、依赖项缺失等。
    - **排查方法**：使用 `kubectl logs <pod-name>` 查看容器的日志信息，这能帮助你了解应用程序在运行过程中出现的错误。若容器有多个，可使用 `kubectl logs <pod-name> -c <container-name>` 查看特定容器的日志。

### 3. 查看容器日志
使用 `kubectl logs` 命令查看容器的日志，这是排查应用程序问题的关键步骤。
```bash
# 查看 Pod 中默认容器的日志
kubectl logs <pod-name>
# 查看 Pod 中特定容器的日志
kubectl logs <pod-name> -c <container-name>
# 查看 Pod 中容器的过往日志（适用于容器已重启的情况）
kubectl logs <pod-name> -p -c <container-name>
```

### 4. 进入容器进行调试
若日志信息不足以定位问题，可使用 `kubectl exec` 命令进入容器内部进行调试。
```bash
# 进入 Pod 中默认容器的 shell
kubectl exec -it <pod-name> -- /bin/bash
# 进入 Pod 中特定容器的 shell
kubectl exec -it <pod-name> -c <container-name> -- /bin/bash
```
在容器内部，你可以检查文件系统、运行命令、查看进程状态等，以进一步排查问题。

### 5. 检查资源配置
检查 Pod 的资源请求和限制配置是否合理，是否存在资源不足或资源竞争的问题。可以通过 `kubectl describe pod <pod-name>` 查看资源配置信息。
```bash
# 查看 Pod 的资源配置
kubectl describe pod <pod-name> | grep -i 'resources'
```

### 6. 检查网络配置
若 Pod 存在网络问题，可使用 `kubectl exec` 命令在容器内部进行网络测试，如 `ping`、`curl` 等。
```bash
# 在 Pod 中测试网络连通性
kubectl exec -it <pod-name> -- ping <target-ip>
kubectl exec -it <pod-name> -- curl <target-url>
```
同时，检查 Service、Ingress 等网络资源的配置是否正确。

### 7. 检查配置文件
确保 Pod 的配置文件（如 YAML 文件）中没有语法错误或配置错误。可以使用 `kubectl create --dry-run=client -f <pod-yaml-file>` 命令进行配置文件的预检查。
```bash
# 预检查 Pod 配置文件
kubectl create --dry-run=client -f <pod-yaml-file>
```

### 8. 查看事件和监控信息
除了查看 Pod 的事件信息，还可以查看 Kubernetes 集群的事件信息，了解是否存在与 Pod 相关的其他异常事件。同时，结合监控工具（如 Prometheus、Grafana）查看 Pod 的性能指标，如 CPU 使用率、内存使用率等，以帮助定位问题。
```bash
# 查看集群的事件信息
kubectl get events
```

通过以上步骤，通常可以逐步定位到 Pod 出现错误的原因，并进行相应的修复。 



# StatefulSet的pod如何做服务发现； 考虑过 headless吗
下面为你提供一个使用无头服务实现 StatefulSet 的 Pod 服务发现的详细实例，涵盖 YAML 文件的创建、部署及验证步骤，同时包含 Python 客户端代码示例。

### 1. 无头服务的 YAML 文件
首先，创建一个无头服务，让 DNS 解析能直接返回后端 Pod 的 IP 地址。下面是对应的 YAML 文件 `headless-service.yaml`：
```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql-headless-service
  namespace: default
spec:
  clusterIP: None
  selector:
    app: mysql-statefulset
  ports:
    - protocol: TCP
      port: 3306
      targetPort: 3306
```
在这个 YAML 文件中：
- `clusterIP: None` 表明这是一个无头服务。
- `selector` 用于挑选标签为 `app: mysql-statefulset` 的 Pod。
- `ports` 定义了服务的端口为 3306。

### 2. StatefulSet 的 YAML 文件
接着，创建一个使用 MySQL 镜像的 StatefulSet。下面是对应的 YAML 文件 `mysql-statefulset.yaml`：
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql-statefulset
  namespace: default
spec:
  serviceName: mysql-headless-service
  replicas: 3
  selector:
    matchLabels:
      app: mysql-statefulset
  template:
    metadata:
      labels:
        app: mysql-statefulset
    spec:
      containers:
        - name: mysql
          image: mysql:8.0
          env:
            - name: MYSQL_ROOT_PASSWORD
              value: "password"
          ports:
            - containerPort: 3306
          volumeMounts:
            - name: mysql-persistent-storage
              mountPath: /var/lib/mysql
  volumeClaimTemplates:
    - metadata:
        name: mysql-persistent-storage
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 1Gi
```
在这个 YAML 文件中：
- `serviceName` 指定了 StatefulSet 所使用的无头服务名称。
- `replicas` 定义了要创建的 Pod 副本数量为 3。
- `volumeClaimTemplates` 为每个 Pod 创建了一个持久化存储卷。

### 3. 部署服务和 StatefulSet
使用以下命令来部署无头服务和 StatefulSet：
```bash
kubectl apply -f headless-service.yaml
kubectl apply -f mysql-statefulset.yaml
```

### 4. 验证服务发现
在 Pod 成功创建后，每个 Pod 都会有一个稳定的 DNS 名称，格式为 `<pod-name>.<service-name>.<namespace>.svc.cluster.local`。例如，第一个 Pod 的 DNS 名称为 `mysql-statefulset-0.mysql-headless-service.default.svc.cluster.local`。

你可以通过以下命令在一个临时 Pod 里验证 DNS 解析：
```bash
kubectl run -it --image=busybox:1.28 dns-test --restart=Never -- nslookup mysql-statefulset-0.mysql-headless-service.default.svc.cluster.local
```
这个命令会启动一个临时的 BusyBox Pod，然后使用 `nslookup` 命令解析指定 Pod 的 DNS 名称。

### 5. Python 客户端代码示例
以下是一个 Python 示例，展示如何通过 DNS 名称访问 StatefulSet 的 Pod：

    

在这个 Python 示例中，使用 `socket.gethostbyname` 函数通过 DNS 名称获取 Pod 的 IP 地址，然后使用 `mysql.connector` 库连接到 MySQL 数据库并执行 SQL 查询。

通过以上步骤，你就能完成一个完整的使用无头服务实现 StatefulSet 的 Pod 服务发现的实例。 


# 如何对pod进行滚动升级

在 Kubernetes（k8s）里，滚动升级是一种能在不中断服务的情况下逐步更新应用的重要策略。以下会综合前面提到的幂等性、有状态服务升级等内容，详细介绍在 k8s 中进行滚动升级的步骤、考虑因素以及实现方法。

### 滚动升级的前提条件
- **稳定的 k8s 集群**：确保 k8s 集群运行正常，各节点通信稳定，资源充足。
- **资源对象部署**：使用 Deployment、StatefulSet 等资源对象来部署应用。其中，Deployment 用于无状态应用，StatefulSet 用于有状态应用。

### 滚动升级的具体步骤

#### 1. 准备工作
- **检查应用状态**：使用 `kubectl get` 命令查看当前应用的状态，确保应用正常运行。
```bash
kubectl get deployments <deployment-name>
kubectl get statefulsets <statefulset-name>
```
- **备份数据（针对有状态服务）**：对于有状态服务，如数据库，在升级前进行数据备份，以防升级过程中出现问题导致数据丢失。可以使用数据库自带的备份工具，如 MySQL 的 `mysqldump`。

#### 2. 修改应用配置
可以通过编辑资源对象的 YAML 文件或者使用 `kubectl set` 命令来修改应用的配置，例如更新容器镜像版本。

##### 使用 `kubectl set` 命令
```bash
kubectl set image deployment/<deployment-name> <container-name>=<new-image>:<new-tag>
kubectl set image statefulset/<statefulset-name> <container-name>=<new-image>:<new-tag>
```
这里的 `<deployment-name>` 或 `<statefulset-name>` 是资源对象的名称，`<container-name>` 是容器的名称，`<new-image>` 是新的镜像名称，`<new-tag>` 是新的镜像标签。

##### 编辑 YAML 文件
使用 `kubectl edit` 命令编辑资源对象的 YAML 文件，将容器镜像更新为新版本。
```bash
kubectl edit deployment <deployment-name>
kubectl edit statefulset <statefulset-name>
```
在打开的编辑器中，找到容器镜像的配置部分，将其更新为新的镜像版本，然后保存退出。

#### 3. 触发滚动升级
修改完应用配置后，k8s 会自动触发滚动升级。可以使用 `kubectl rollout status` 命令查看升级状态。
```bash
kubectl rollout status deployment/<deployment-name>
kubectl rollout status statefulset/<statefulset-name>
```
此命令会实时显示升级的进度，直到升级完成。

#### 4. 监控升级过程
在升级过程中，使用以下命令监控资源对象的状态：
```bash
kubectl get deployments <deployment-name> -w
kubectl get statefulsets <statefulset-name> -w
```
`-w` 参数表示实时监控，当有状态变化时会实时显示。同时，还可以通过监控系统（如 Prometheus、Grafana）监控应用的性能指标，如 CPU 使用率、内存使用率、响应时间等。

#### 5. 处理升级异常
如果升级过程中出现问题，如部分 Pod 无法正常启动，可以使用 `kubectl describe` 和 `kubectl logs` 命令查看 Pod 的详细信息和日志，找出问题所在。
```bash
kubectl describe pod <pod-name>
kubectl logs <pod-name>
```
如果问题无法解决，可以使用 `kubectl rollout undo` 命令回滚到上一个稳定版本。
```bash
kubectl rollout undo deployment/<deployment-name>
kubectl rollout undo statefulset/<statefulset-name>
```

### 滚动升级时需要考虑的因素

#### 1. 幂等性
- **数据库操作**：在升级过程中，如果涉及数据库操作，要确保操作的幂等性。例如，使用唯一索引避免重复插入数据，使用条件判断进行更新操作。
- **HTTP 请求**：对于应用的 API 接口，要确保其具有幂等性。可以通过客户端生成唯一请求 ID，服务器端缓存处理结果的方式来实现。

#### 2. 服务上下游依赖
- **版本兼容性**：确保新版本的应用与上下游服务的接口兼容，避免因接口变更导致调用失败。
- **流量管理**：在升级过程中，要确保流量能够平滑地从旧版本的 Pod 切换到新版本的 Pod，可以使用服务网格（如 Istio）或 API 网关（如 Kong）来实现流量分割和路由。

#### 3. 数据一致性（针对有状态服务）
- **数据迁移**：如果升级过程中涉及数据库结构的变更，要提前规划好数据迁移方案，确保数据的一致性。可以采用增量迁移、双写等方式。
- **持久化存储**：使用持久卷声明（PVC）为有状态服务提供持久化存储，确保数据不会因为 Pod 的重启或升级而丢失。

### 示例：对一个 MySQL StatefulSet 进行滚动升级
```yaml
# mysql-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql-statefulset
spec:
  serviceName: mysql-service
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0  # 初始版本
        ports:
        - containerPort: 3306
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
  volumeClaimTemplates:
  - metadata:
      name: mysql-persistent-storage
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi
```
#### 升级步骤
1. 备份 MySQL 数据。
2. 使用 `kubectl set image` 命令更新 MySQL 镜像版本。
```bash
kubectl set image statefulset/mysql-statefulset mysql=mysql:8.1
```
3. 监控升级状态。
```bash
kubectl rollout status statefulset/mysql-statefulset
```
4. 检查 MySQL 服务是否正常运行，数据是否一致。

通过以上步骤和考虑因素，可以在 k8s 中安全、稳定地进行滚动升级。


user  userid


产品id 产品名称 价格

订单  userid  time  productid  quant


要求计算出上个月每一天的销售额


select p.price*o.quant, date  from p  joinon p.id=o.PID group by 
