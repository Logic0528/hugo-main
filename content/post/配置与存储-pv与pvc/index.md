+++
date = '2025-05-11T14:49:19+08:00'
draft = true
title = 'K8s-持久化存储-PV和PVC'
+++
### PV和PVC
PV和PVC是Kubernetes中用于管理持久化存储的两个重要概念。
#### PV（Persistent Volume）
- 是集群中的存储资源
- 由管理员事先创建和配置
- 独立与pod的生命周期
- 可以对接各种存储后端，如nfs、ceph、云存储等
- 定义了存储的容量、访问模式、存储类等信息  
  
#### PVC（Persistent Volume Claim）
- 是pod对存储资源的请求
- 由用户/开发者创建
- 类似于pod消耗Node资源，PVC消耗PV资源
- 声明需要的存储容量、访问模式、存储类等信息
- 不需要关心存储的具体实现细节
   
#### 工作流程
1. 管理员创建PV，定义存储资源的属性
```yaml
apiVersion: v1
kind: PersistentVolume #描述资源类型为PV类型
metadata:
  name: pv0001
spec:
  capacity: #容量配置
    storage: 5Gi
  volumeMode: Filesystem #存储类型为文件系统
  accessModes: #访问模式：ReadWriteOnce WeadWriteMany ReadOnlyMany
    - ReadWriteMany #可被单节点读写
  persistentVolumeReclaimPolicy: Retain #回收策略
  storageClassName: slow #创建PV的存储类名，需要与pvc的相同
  mountOptions: #加载配置
    - hard
    - nfsvers=4.1
  nfs: #连接到nfs
    path: /home/nfs/rw/pv-test #存储路径
    server: 192.168.189.130 #nfs服务地址
```    
2. 用户/开发者创建PVC，声明需要的存储资源
    ```yaml
    apiVersion: v1
    kind: PersistentVolumeClaim #资源类型为pvc
    metadata:
    name: nfs-pvc
    spec:
    accessModes:
        - ReadWriteMany
    volumeMode: Filesystem
    resources:
        requests:
        storage: 5Gi
    storageClassName: slow #名字要与对应的pv相同

    # selector: #使用选择器选择对应的pv
    ```
3. Kubernetes调度器根据PVC的要求，选择合适的PV进行绑定
   ```yaml
   apiVersion: v1
    kind: Pod
    metadata: 
    name: pvc-test-pod
    spec:
    containers:
        - image: nginx
        name: nginx-volume
        volumeMounts:
        - mountPath: /usr/share/nginx/html #挂载到容器的哪个目录
            name: test-volume #挂载哪个volume
    volumes:
    - name: test-volume
        persistentVolumeClaim:
        claimName: nfs-pvc
    tolerations:
    - key: "node-role.kubernetes.io/control-plane"
        operator: "Exists"
        effect: "NoSchedule"
   ```
   

