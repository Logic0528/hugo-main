+++
date = '2025-06-07T14:51:28+08:00'
draft = true
title = '面试'
+++

五月份进行了两次面试，面试不同于闭门造车问的许多问题需要进行规范的回答类似于八股文之类的，虽然我很不喜欢八股文代表的那种死板的精神，但不得不承认在对于基础知识的掌握有许多不足，在面试过程中很多概念熟悉了解但是语言组织不够流畅，对面试中没能回答上来或者回答不流畅或者以后面试可能会遇到的额问题进行整理。

#### k8s架构和基本组件及其功能
Kubernetes（k8s）是一个开源的容器编排平台，用于自动化部署、扩展和管理容器化应用。其架构采用主从（Master-Node）模型，核心组件分为控制平面（Control Plane）和工作节点（Worker Node）两部分。以下是详细架构和组件功能说明：

一、控制平面（Master Components）
控制平面负责集群的全局决策和调度，通常运行在独立的Master节点上。

API Server（kube-apiserver）

功能：集群的唯一入口，提供RESTful API，处理所有管理请求（如创建/删除Pod、部署应用等）。

特点：无状态设计，可水平扩展；通过认证、授权、准入控制保证安全性。

Scheduler（kube-scheduler）

功能：监听未调度的Pod，根据资源需求、节点负载、亲和性等策略，将Pod绑定到合适的工作节点。

策略：支持自定义调度算法（如优先选择低负载节点）。

Controller Manager（kube-controller-manager）

功能：运行一系列控制器，确保集群状态与预期一致。核心控制器包括：

Node Controller：监控节点状态（如宕机时重新调度Pod）。

Deployment Controller：管理副本数，实现滚动更新/回滚。

Service Controller：维护Service与Pod的映射关系。

etcd

功能：分布式键值存储数据库，保存集群所有配置数据和状态（如Pod、Service、Namespace等）。

特点：高可用性要求高，通常需部署奇数个节点（如3、5个）。

二、工作节点（Node Components）
每个工作节点负责运行容器化应用，包含以下组件：

kubelet

功能：节点上的“代理”，与API Server通信，管理本节点Pod的生命周期（创建/删除容器、监控资源等）。

关键职责：确保Pod中容器健康（通过探针检查），上报节点状态。

kube-proxy

功能：维护节点上的网络规则（如iptables/IPVS），实现Service的负载均衡和流量转发。

示例：将访问Service VIP的请求转发到后端Pod。

容器运行时（Container Runtime）

功能：负责运行容器（如Docker、containerd、CRI-O），支持OCI标准。

三、附加组件（Addons）
这些组件非必需，但为集群提供增强功能：

CNI插件（如Calico、Flannel）

功能：实现Pod间网络通信和跨节点网络策略。

CoreDNS

功能：为集群提供DNS解析服务（如Service名称到IP的映射）。

Ingress Controller（如Nginx Ingress）

功能：管理外部访问集群的HTTP/HTTPS路由规则。

Metrics Server

功能：收集资源使用指标（CPU/内存），供HPA（自动扩缩容）使用。

Dashboard

功能：提供Web UI界面管理集群。

#### jekins如何实现扩容
jenkins 的扩容主要通过 水平扩展（Horizontal Scaling） 和 垂直扩展（Vertical Scaling） 两种方式实现，具体取决于你的 Jenkins 架构（单机版或分布式）。以下是详细方案：

一、单机 Jenkins 扩容
适用于单节点 Jenkins，提升单个 Master 节点的处理能力。

1. 垂直扩展（Vertical Scaling）
方式：增加 Jenkins Master 节点的 CPU、内存、磁盘 等资源。

适用场景：任务量较小，但单个任务资源需求高（如大型构建任务）。

实现方法：

物理机：升级硬件配置。

虚拟机/云主机：调整实例规格（如 AWS EC2 从 t2.medium 升级到 t2.large）。

容器化部署：调整 Docker/Kubernetes 资源限制（resources.requests/limits）。

2. 优化 Jenkins 配置
调整 JVM 参数：修改 JENKINS_JAVA_OPTS，增加堆内存（如 -Xmx4g）。

bash
 在 Jenkins 启动脚本或 systemd 配置中设置
JAVA_OPTS="-Xmx4g -Xms2g"
清理旧数据：定期清理构建历史、日志、无用插件，减少磁盘占用。

二、分布式 Jenkins 扩容（水平扩展）
适用于高并发构建场景，通过 Master + Agent 架构 分散负载。

1. 添加 Jenkins Agent（工作节点）
原理：Master 负责调度任务，Agent 执行具体构建任务。

实现方式：

静态 Agent：手动或通过脚本添加固定节点（物理机/虚拟机）。

动态 Agent（弹性伸缩）：

Kubernetes Plugin：在 K8s 集群中自动创建/销毁 Pod 作为 Agent。

Docker Plugin：按需启动 Docker 容器作为 Agent。

云平台插件（AWS EC2、Azure VM）：根据负载自动扩缩容云实例。

2. Kubernetes 动态扩缩容（推荐方案）
步骤：

安装 Kubernetes Plugin：
Jenkins → 插件管理 → 安装 Kubernetes Plugin。

配置 Kubernetes Cloud：
Jenkins → 系统管理 → 节点管理 → 配置 Kubernetes 集群信息（API 地址、Namespace、认证等）。

定义 Pod 模板：
指定 Agent 的容器镜像、资源限制、卷挂载等（示例配置）：
```yaml
containers:
- name: jnlp
  image: jenkins/inbound-agent:latest
  resources:
    limits:
      cpu: "1"
      memory: "2Gi"
```
设置自动伸缩规则：

通过 Jenkins Pipeline 或任务配置指定 label（如 kubernetes）。

Kubernetes 根据任务队列长度自动创建/删除 Agent Pod。

效果：

无任务时：Agent Pod 数为 0，节省资源。

高并发时：自动创建多个 Agent Pod 并行执行任务。

三、高可用（HA）方案
若需进一步提升可用性，可结合以下策略：

Master 节点高可用：

使用 Jenkins 集群（多个 Master + 共享存储，如 NFS 或 S3）。

容器化部署时，通过 Kubernetes StatefulSet + PVC 持久化数据。

负载均衡：

通过 Nginx/HAProxy 代理多个 Jenkins Master。

备份恢复：

定期备份 JENKINS_HOME 目录（含配置和任务数据）。
#### 蓝绿发布，金丝雀发布，灰度发布
##### 一、金丝雀发布（Canary Release）
1. 核心原理
名称来源：类比矿工用金丝雀检测矿井瓦斯，先让小部分用户试用新版本，观察无问题后再逐步扩大范围。

实现方式：

将新版本（Canary）与旧版本同时在线，按比例（如 5%→20%→100%）将流量逐步切换到新版本。

通过监控（错误率、延迟等）判断新版本稳定性，发现问题立即回滚。

2. 典型场景
需要验证新功能在真实环境的表现。

避免全量发布导致的全局故障。

3. 实现工具
Kubernetes：通过 Ingress（如 Nginx Ingress 的 canary 注解）或 Service Mesh（如 Istio 的流量拆分）。

示例（Istio 配置）：

yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: my-app
spec:
  hosts:
    - my-app.example.com
  http:
    - route:
        - destination:
            host: my-app
            subset: v1  # 旧版本
          weight: 90    # 90%流量
        - destination:
            host: my-app
            subset: v2  # 新版本（金丝雀）
          weight: 10    # 10%流量
##### 二、蓝绿发布（Blue-Green Deployment）
1. 核心原理
颜色标识：

Blue：当前生产环境（旧版本）。

Green：新版本环境（完全独立部署）。

切换方式：在测试通过后，一次性将全部流量从 Blue 切换到 Green（通过负载均衡器或 DNS 切换）。

2. 典型场景
需要零停机时间的发布。

快速回滚（直接切回 Blue 环境）。

3. 优缺点
优点：发布和回滚极快，避免版本共存问题。

缺点：需双倍资源成本，数据库兼容性需提前处理。

4. 实现工具
Kubernetes：通过创建两套完整的 Deployment + Service，切换 selector 或使用 Ingress 路由。

云平台：AWS Elastic Beanstalk、Azure Deployment Slots。

##### 三、灰度发布（Gray Release）
1. 核心原理
广义概念：涵盖所有渐进式发布策略（包括金丝雀发布）。

狭义实现：按用户特征（如地理位置、用户ID、设备类型）逐步开放新版本，而非单纯按流量比例。

2. 典型场景
定向测试：仅对内部员工或特定用户群开放新功能。

A/B 测试：对比不同版本的用户行为数据。

3. 实现工具
前端：通过 Feature Flag（如 LaunchDarkly）控制功能可见性。

后端：Nginx/Apache 规则、Service Mesh（如 Istio 的基于 Header 的路由）。