+++
date = '2025-05-15T15:52:46+08:00'
draft = true
title = 'K8s-污点与容忍'
tags = ['K8s','污点'，'容忍']
categories = ['K8s','污点'，'容忍']
+++
### 污点（Taint）
#### effect:
NoSchedule：不允许调度，已经存在的Pod不受影响  

PreferNoSchedule：尽量不调度，已经存在的Pod不受影响  

NoExecute：不允许调度，并且驱逐已经存在的Pod  

### 容忍（Toleration）
#### operator:
Equal：需要key和value完全匹配  

Exists：只需要key匹配，不需要value匹配 
#### value:
污点的值 

示例代码：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-tolerations
spec:
  tolerations:
  - key: "key1"           # 污点的键
    operator: "Equal"     # 操作符：Equal或Exists
    value: "value1"       # 污点的值
    effect: "NoSchedule"  # 效果
    tolerationSeconds: 3600  # 容忍时间（仅用于NoExecute）
```
###  亲和力（Affinity）
#### NodeAffinity
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-node-affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:  # 硬性要求
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/os
            operator: In
            values:
            - linux
      preferredDuringSchedulingIgnoredDuringExecution:  # 软性偏好
      - weight: 1
        preference:
          matchExpressions:
          - key: disk-type
            operator: In
            values:
            - ssd
```
#### PodAffinity
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-pod-affinity
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - web-store
        topologyKey: kubernetes.io/hostname
```
#### PodAntiAffinity(节点反亲和力)
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-pod-antiaffinity
spec:
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: app
              operator: In
              values:
              - cache
          topologyKey: kubernetes.io/hostname
```
没什么好说的，类似于标签，慢工出细活~