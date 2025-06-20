+++
date = '2025-06-07T14:51:28+08:00'
draft = true
title = '面试'
+++

五月份进行了两次面试，面试不同于闭门造车,问的许多问题需要进行规范的回答类似于八股文之类的，虽然我很不喜欢八股文代表的那种死板的精神，但不得不承认在对于基础知识的掌握有许多不足，在面试过程中很多概念熟悉了解但是语言组织不够流畅，对面试中没能回答上来或者回答不流畅或者以后面试可能会遇到的额问题进行整理。
屡战屡败，屡败屡战，面试中不会的问题记下来解决也是一种收获

## 1.k8s架构和基本组件
k8s是一个开源的容器编排平台，用于自动化部署，扩展和管理容器化应用。其架构采用主从架构（master-node），核心组件分为控制平面control plane和工作节点worker node。  

控制平面组件：  
kube-apiserver:API网关和集群控制中心，处理所有管理请求如创建/删除pod，部署应用等
etcd: 分布式键值存储，保存集群配置,如pod,service,namespace  
kube-scheduler: 负责pod调度到合适的节点  
kube-controller-manager: 运行控制器进程，包括node controller,deployment controller,service controller
  
节点组件：  
kubelet: 管理节点上的pod  
kube-proxy: 实现网络规则和负载均衡，维护节点上的网络配置  
container runtime: 负责运行容器（如docker，containerd）
  
附加组件： 
coreDNS：集群DNS服务，提供域名解析功能，将服务名解析为IP地址。
IngressContorller：负责外部流量访问，支持负载均衡和路由规则。HTTP路由
NetworkPluginL: calico，flannel等

## 2.pod,deployment,service
pod： 是k8s中最小的调度单位，封装一个或多个容器，共享网络和存储
  
deployment： 基于ReplicaSet的更高层抽象，声明式管理Pod的期望状态，如副本数，镜像版本，更新策略
作用： 确保pod副本数始终等于期望值，自动重启/创建失败的pod，支持滚动更新，回滚版本  

service：抽象的网络层，为一组pod提供固定访问入口 ip+端口 
作用：
内部服务发现：同意集群内其他pod可通过service域名访问
外部暴露： 通过nodeport等类型将服务暴露到集群外
负载均衡： 将流量分发到后端多个pod副本

deployment是管理pod的控制器，service是抽象网络层，三者通过标签匹配。

## 3.nginx与负载均衡
nginx是一款高性能的web服务器，反向代理服务器，负载均衡器
web服务器： 处理静态资源的请求，直接返回文件内容，性能远超传统服务器  
反向代理服务器： 将客户端请求转发到后端多个应用服务器如tomcat，隐藏真实服务器地址  
负载均衡器： 分发客户端请求到多个后端服务器，避免单点过载

负载均衡： 将网络流量均发到多个计算资源如服务器，容器，避免单一节点过载，提高系统的可用性，性能和容错能力

nginx负载均衡方式：
轮询；默认策略，按顺序分发请求
weight: 根据权重分配
ip_hash： 基于客户端IP哈希
等....深入了解后补充

负载均衡的优点

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
```yaml
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
 ```
