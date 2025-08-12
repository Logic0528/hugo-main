+++

draft = true

title = 'kubernetes秋招面试题准备'

date = '2025-08-12T15:00:19+08:00'

+++



## 一、k8s组件篇

`Master节点:`

kube-apiserver (API服务器)   

etcd (分布式存储) 

kube-scheduler (调度器)  

kube-controller-manager (控制器)

`Worker节点：`

kubelet (节点代理)  

kube-proxy (网络代理)    

Container Runtime (容器运行时)

### 1、kube-apiserver

1. kube-apiserver的核心作用，为什么说它是k8s的中枢

   - kube-apiserver是k8s所有组件的统一接口，负责接收，验证和处理所有客户端请求（如kubectl命令，其他组件通信），并将集群状态持久化到etcd

   - 之所以称为中枢，是因为所有组件均通过apiserver交互，而非直接通信，它是集群数据的唯一入口

2. kube-apiserver如何处理客户端请求（如kubectl命令）？是否支持高可用部署？如何实现 多实例 + 负载均衡？

   - 接收请求-->依次经过认证(Authentication,验证请求者身份，token，证书)-->授权(Authorization,验证请求者权限，如RBAC规则)-->准入控制(Admission Control,修改或拒绝请求，如资源限制检查)-->验证通过后，将数据写入etcd,并返回结果

   - 支持高可用部署：通过多实例 + 负载均衡实现（如HAProxy），多个apiserver实例共享etcd集群，负载均衡器分发请求，避免单点故障

3. kube-apiserver如何保证API调用的安全性？

   - 认证：支持客户端证书，用户名密码等方式，验证请求者身份

   - 授权：通过RBAC（基于角色的访问控制）等机制，检查请求者是否有权限执行操作；

4. API Server的API版本有哪些？不同版本的区别是什么？如何应对高并发请求？

   - API版本：分为alpha(测试版，可能删除)，beta(测试版，功能基本稳定)，stable(稳定版，如v1,apps/v1)，不同版本对应资源的功能成熟度。

   - 高并发优化：通过缓存如etcd缓存，减少重复查询，支持请求限流防止过载，可水平扩展apiserver实例分担压力

### 2、etcd

1. etcd在k8s中存储什么数据？为什么选择etcd作为存储后端？

   - 存储k8s集群的所有核心机制，包括Pod,Service,Deployment等资源的定义和状态，以及集群配置（如RBAC规则）

   - 选择etcd原因：分布式一致性，基于Raft协议，保持数据多副本一致，强一致性读写，高可用（集群部署），支持事务和监听机制（适合实时同步集群状态）

2. etcd是单节点还是分布式的？如何部署etcd集群保证高可用？

   - etcd是分布式存储，需部署集群

   - Raft协议作用：通过领导者选举，保证集群有一个主节点Leader处理写请求，从节点同步数据，若Leader故障，重新选举新Leader，确保数据不丢失且一致

3. 如何备份和恢复etcd数据？数据增长过快怎么办？

   - 备份：使用etcdctl snapshot save <备份文件>需指定etcd地址和证书
   - 恢复：etcdctl snapshot restore <备份文件>需指定新的集群名称和数据目录。
   - 数据增长过快：启用自动压缩，定期处理过期数据，使用SSD提升读写性能

4. etcd的读写性能对k8s集群有什么影响？如何优化？

   etcd性能直接影响集群响应速度，如Pod创建延迟

   优化方式：存储介质使用SSD，集群规模控制在3-5节点，减少Raft协议开销

### 3、kube-scheduler

1. kube-scheduler的核心作用是什么？它要解决的核心问题是什么？
   - 核心作用：为新创建的Pod选择最合适的Node节点
   - 解决的核心问题：在满足Pod资源需求和调度策略的前提下，优化集群资源使用率
2. 详细描述Pod的调度过程，过滤阶段和打分阶段分别做什么。
   - 过滤：从所有Node中排除不满足Pod要求的节点，输出可行节点列表
   - 打分：对可行节点按优先级评分，得分最高的节点被选中
   - 绑定：将Pod与选中的Node绑定，通过apiserver更新Pod的spec.nodeName字段
3. 调度策略
   - 节点亲和性nodeAffinity：Pod通过规则指定偏好或必须部署的Node
   - Pod亲和性podAffinity：Pod偏好与特定Pod部署在同一拓扑域（如数据库Pod在同一机房）
   - 污点taint与容忍toleration：node通过污点排斥Pod，Pod通过容忍突破排斥

### 4、kube-controller-manager

1. kube-controller-manager作用是什么？它包含哪些核心控制器？
   - 作用是通过控制器监测集群状态，确保实际状态与期望状态一致
   - 核心控制器包括DeploymentController,ReplicaSetController,NodeController,JobController,ServiceController
2. 控制器的调谐循环是如何工作的？
   - 监测：通过apiserver监听目标资源的状态变化，如Deployment的副本数
   - 对比：将实际状态与期望状态对比
   - 调谐：执行操作使实际状态接近期望状态
3. 常见控制器的作用
   1. DeploymentController: 管理Replicaset,通过控制ReplicaSet的数量和版本，保证Deployment的期望副本数和滚动更新
   2. NodeController: 监测Node健康状态，若Node失联

### 5、kubelet

1. kubelet核心作用是什么？它与Master节点的哪个组件直接交互
   - 核心作用是在Node上运行Pod，并保证Pod按期望状态运行，处理Master下发到本节点的任务
   - 直接与kube-apiserver交互，定期从apiserver获取Node上的Pod定义，上报Pod和Node状态
2. kubelet如何保证Pod按照期望状态运行？
   - 监听：通过apiserver的Watch机制，实时获取分配到当前Node的pod信息
   - 创建：根据Pod定义，通过容器运行时创建容器
   - 监控：持续检查Pod和容器状态，若状态异常，尝试重启
   - 上报：将Pod状态通过apiserver写入etcd
3. kubelet如何执行Pod的存活探针和就绪探针？探针失败会触发什么操作？
   - 存活探针livenessprobe: 检测容器是否存活，失败则触发容器重启
   - 就绪探针readnessprobe: 检测容器是否就绪，失败则将Pod从Service的端点列表中移除
   - kubelet定期执行探针，配置periodseconds,根据结果执行对应操作
4. 什么是静态Pod?kubelet如何管理静态Pod?
   - 静态Pod是由kubelet直接管理的Pod,不通过apiserver,配置文件通常放在/etc/kubernetes/manifests
   - kubelet定期扫描目录，若配置文件更新，自动重建Pod,静态Pod在apiserver中可见，但无法通过kubectl删除
   - 用途：部署Master组件，如apiserver,etcd，确保组件随Node启动而启动

###  7、Container Runnertime

## 二、简述题

### 1.**简述Kubernetes创建一个Pod的主要流程**

Kubernetes中创建一个Pod涉及多个组件之间联动，主要流程如下：

- 客户端提交Pod的配置信息（可以是yaml文件定义的信息）到kube-apiserver。
- Apiserver收到指令后，通知给controller-manager创建一个资源对象。
- Controller-manager通过api-server将Pod的配置信息存储到etcd数据中心中。
- Kube-scheduler检测到Pod信息会开始调度预选，会先过滤掉不符合Pod资源配置要求的节点，然后开始调度调优，主要是挑选出更适合运行Pod的节点，`然后将Pod的资源配置单发送到Node节点上的kubelet组件上。`
- Kubelet根据scheduler发来的资源配置单运行Pod，运行成功后，将Pod的运行信息返回给scheduler，scheduler将返回的Pod运行状况的信息存储到etcd数据中心。

### 2.**简述Kubernetes自动扩容机制**

Kubernetes使用Horizontal Pod Autoscaler（HPA）的控制器实现基于CPU使用率进行自动Pod扩缩容的功能。HPA控制器周期性地监测目标Pod的资源性能指标，并与HPA资源对象中的扩缩容条件进行对比，在满足条件时对Pod副本数量进行调整。

Kubernetes中的某个Metrics Server（Heapster或自定义Metrics Server）持续采集所有Pod副本的指标数据。HPA控制器通过Metrics Server的API（Heapster的API或聚合API）获取这些数据，基于用户定义的扩缩容规则进行计算，得到目标Pod副本数量。

当目标Pod副本数量与当前副本数量不同时，**HPA控制器就向Pod的副本控制器（Deployment、RC或ReplicaSet）发起scale操作，调整Pod的副本数量，完成扩缩容操作**

### 3.**简述Kubernetes RC的机制**

Replication Controller用来管理Pod的副本，保证集群中存在指定数量的Pod副本。当定义了RC并提交至Kubernetes集群中之后，Master节点上的Controller Manager组件获悉，并同时巡检系统中当前存活的目标Pod，并确保目标Pod实例的数量刚好等于此RC的期望值，若存在过多的Pod副本在运行，系统会停止一些Pod，反之则自动创建一些Pod

### 4.**简述Kubernetes Replica Set和Replication Controller之间有什么区别**

Replica Set和Replication Controller类似，都是确保在任何给定时间运行指定数量的Pod副本。不同之处在于RS使用**基于集合的选择器**，而Replication Controller使用<u>基于权限的选择器</u>。

### 5.**简述Kubernetes中什么是静态Pod**

静态Pod是由kubelet进行管理的仅存在于特定Node的Pod上，他们不能通过API Server进行管理，无法与ReplicationController、Deployment或者DaemonSet进行关联，并且kubelet无法对他们进行健康检查。静态Pod总是由kubelet进行创建，并且总是在kubelet所在的Node上运行。

### 6.**简述Kubernetes中Pod的健康检查方式**

对Pod的健康检查可以通过两类探针来检查：LivenessProbe和ReadinessProbe。

- LivenessProbe探针：用于判断容器是否存活（running状态），如果LivenessProbe探针探测到容器不健康，则kubelet将杀掉该容器，并根据容器的重启策略做相应处理。若一个容器不包含LivenessProbe探针，kubelet认为该容器的LivenessProbe探针返回值用于是“Success”。
- ReadineeProbe探针：用于判断容器是否启动完成（ready状态）。如果ReadinessProbe探针探测到失败，则Pod的状态将被修改。Endpoint Controller将从Service的Endpoint中删除包含该容器所在Pod的Eenpoint。
- startupProbe探针：启动检查机制，应用一些启动缓慢的业务，避免业务长时间启动而被上面两类探针kill掉

### 7.**简述Kubernetes各模块如何与API Server通信**

Kubernetes API Server作为集群的核心，负责集群各功能模块之间的通信。集群内的各个功能模块通过API Server将信息存入etcd，当需要获取和操作这些数据时，则通过API Server提供的REST接口（用GET、LIST或WATCH方法）来实现，从而实现各模块之间的信息交互。

如kubelet进程与API Server的交互：每个Node上的kubelet每隔一个时间周期，就会调用一次API Server的REST接口报告自身状态，API Server在接收到这些信息后，会将节点状态信息更新到etcd中。

如kube-controller-manager进程与API Server的交互：kube-controller-manager中的Node Controller模块通过API Server提供的Watch接口实时监控Node的信息，并做相应处理。

如kube-scheduler进程与API Server的交互：Scheduler通过API Server的Watch接口监听到新建Pod副本的信息后，会检索所有符合该Pod要求的Node列表，开始执行Pod调度逻辑，在调度成功后将Pod绑定到目标节点上。

**简述Kubernetes kubelet监控Worker节点资源是使用什么组件来实现的**

kubelet使用cAdvisor对worker节点资源进行监控。在Kubernetes系统中，cAdvisor已被默认集成到kubelet组件内，当kubelet服务启动时，它会自动启动cAdvisor服务，然后cAdvisor会实时采集所在节点的性能指标及在节点上运行的容器的性能指标。

## 三、网络篇

### 1.**简述Kubernetes网络模型**

Kubernetes网络模型中每个Pod都拥有一个独立的IP地址，并假定所有Pod都在一个可以直接连通的、扁平的网络空间中。所以不管它们是否运行在同一个Node（宿主机）中，都要求它们可以直接通过对方的IP进行访问。设计这个原则的原因是，用户不需要额外考虑如何建立Pod之间的连接，也不需要考虑如何将容器端口映射到主机端口等问题。

同时为每个Pod都设置一个IP地址的模型使得同一个Pod内的不同容器会共享同一个网络命名空间，也就是同一个Linux网络协议栈。这就意味着同一个Pod内的容器可以通过localhost来连接对方的端口。

在Kubernetes的集群里，IP是以Pod为单位进行分配的。一个Pod内部的所有容器共享一个网络堆栈（相当于一个网络命名空间，它们的IP地址、网络设备、配置等都是共享的）。

### 2.**简述Kubernetes CNI模型**

CNI提供了一种应用容器的插件化网络解决方案，定义对容器网络进行操作和配置的规范，通过插件的形式对CNI接口进行实现。CNI仅关注在创建容器时分配网络资源，和在销毁容器时删除网络资源。在CNI模型中只涉及两个概念：容器和网络。

- 容器（Container）：是拥有独立Linux网络命名空间的环境，例如使用Docker或rkt创建的容器。容器需要拥有自己的Linux网络命名空间，这是加入网络的必要条件。
- 网络（Network）：表示可以互连的一组实体，这些实体拥有各自独立、唯一的IP地址，可以是容器、物理机或者其他网络设备（比如路由器）等。

对容器网络的设置和操作都通过插件（Plugin）进行具体实现，CNI插件包括两种类型：CNI Plugin和IPAM（IP Address Management）Plugin。CNI Plugin负责为容器配置网络资源，IPAM Plugin负责对容器的IP地址进行分配和管理。IPAM Plugin作为CNI Plugin的一部分，与CNI Plugin协同工作。

### 3.**简述Kubernetes网络策略**

为实现细粒度的容器间网络访问隔离策略，Kubernetes引入Network Policy。

Network Policy的主要功能是对Pod间的网络通信进行限制和准入控制，设置允许访问或禁止访问的客户端Pod列表。Network Policy定义网络策略，配合策略控制器（Policy Controller）进行策略的实现。

### 4.**简述Kubernetes网络策略原理**

Network Policy的工作原理主要为：policy controller需要实现一个API Listener，监听用户设置的Network Policy定义，并将网络访问规则通过各Node的Agent进行实际设置（Agent则需要通过CNI网络插件实现）。

### 5.**简述Kubernetes中flannel的作用**

Flannel可以用于Kubernetes底层网络的实现，主要作用有：

- 它能协助Kubernetes，给每一个Node上的Docker容器都分配互相不冲突的IP地址。
- 它能在这些IP地址之间建立一个覆盖网络（Overlay Network），通过这个覆盖网络，将数据包原封不动地传递到目标容器内。

### 6.**简述Kubernetes Calico网络组件实现原理**

Calico是一个基于BGP的纯三层的网络方案，与OpenStack、Kubernetes、AWS、GCE等云平台都能够良好地集成。

Calico在每个计算节点都利用Linux Kernel实现了一个高效的vRouter来负责数据转发。每个vRouter都通过BGP协议把在本节点上运行的容器的路由信息向整个Calico网络广播，并自动设置到达其他节点的路由转发规则。

Calico保证所有容器之间的数据流量都是通过IP路由的方式完成互联互通的。Calico节点组网时可以直接利用数据中心的网络结构（L2或者L3），不需要额外的NAT、隧道或者Overlay Network，没有额外的封包解包，能够节约CPU运算，提高网络效率。

### 7.**简述kube-proxy的作用**

kube-proxy运行在所有节点上，它监听apiserver中service和endpoint的变化情况，创建路由规则以提供服务IP和负载均衡功能。简单理解此进程是Service的透明代理兼负载均衡器，其核心功能是将到某个Service的访问请求转发到后端的多个Pod实例上。

### 8.**简述Kubernetes Service类型**

通过创建Service，可以为一组具有相同功能的容器应用提供一个统一的入口地址，并且将请求负载分发到后端的各个容器应用上。其主要类型有：

- ClusterIP：虚拟的服务IP地址，该地址用于Kubernetes集群内部的Pod访问，在Node上kube-proxy通过设置的iptables规则进行转发；
- NodePort：使用宿主机的端口，使能够访问各Node的外部客户端通过Node的IP地址和端口号就能访问服务；
- LoadBalancer：使用外接负载均衡器完成到服务的负载分发，需要在spec.status.loadBalancer字段指定外部负载均衡器的IP地址，通常用于公有云。

### 9.**简述Kubernetes Service分发后端的策略**

Service负载分发的策略有：RoundRobin和SessionAffinity

- RoundRobin：默认为轮询模式，即轮询将请求转发到后端的各个Pod上。
- SessionAffinity：基于客户端IP地址进行会话保持的模式，即第1次将某个客户端发起的请求转发到后端的某个Pod上，之后从相同的客户端发起的请求都将被转发到后端相同的Pod上。

### 10.**简述Kubernetes外部如何访问集群内的服务**

对于Kubernetes，集群外的客户端默认情况，无法通过Pod的IP地址或者Service的虚拟IP地址：虚拟端口号进行访问。通常可以通过以下方式进行访问Kubernetes集群内的服务：

- 映射Pod到物理机：将Pod端口号映射到宿主机，即在Pod中采用hostPort方式，以使客户端应用能够通过物理机访问容器应用。
- 映射Service到物理机：将Service端口号映射到宿主机，即在Service中采用nodePort方式，以使客户端应用能够通过物理机访问容器应用。
- 映射Sercie到LoadBalancer：通过设置LoadBalancer映射到云服务商提供的LoadBalancer地址。这种用法仅用于在公有云服务提供商的云平台上设置Service的场景。

### 11.**简述Kubernetes ingress**

Kubernetes的Ingress资源对象，用于将不同URL的访问请求转发到后端不同的Service，以实现HTTP层的业务路由机制。

Kubernetes使用了Ingress策略和Ingress Controller，两者结合并实现了一个完整的Ingress负载均衡器。使用Ingress进行负载分发时，Ingress Controller基于Ingress规则将客户端请求直接转发到Service对应的后端Endpoint（Pod）上，从而跳过kube-proxy的转发功能，kube-proxy不再起作用，全过程为：ingress controller + ingress 规则 ----> services。

同时当Ingress Controller提供的是对外服务，则实际上实现的是边缘路由器的功能。