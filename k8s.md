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
面试官答案：分组驱逐。

- 多master如何保持一致性。有什么风险点吗？
当时直接答的不会。





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
