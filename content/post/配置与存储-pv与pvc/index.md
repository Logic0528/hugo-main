+++
date = '2025-05-11T14:49:19+08:00'
draft = true
title = 'K8s-持久化存储-PV和PVC'
# 添加分类
categories = ["Kubernetes", "存储管理"]
# 添加标签
tags = ["K8s", "PV", "PVC", "持久化存储"]
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
### 动态制备PV  
被污点给整破防了，暂时先不写了。（后半句由ai补充），天哪你连我心里想的什么都知道快点拯救全人类吧，待我学完污点归来  
目前遗留问题，-n kube-system po nfs-server-provisioner ImagePullBackOff,kube-apiserver pending状态，selfLink怎么处理，其他的部分理解深入许多  

好的，了解到主节点上默认会有污点，目的是为了保护主节点资源，防止普通pod调
#### 动态制备pv 
##### 创建storageclass
- 定义存储类型
- 指定Provisioner
- 设置存储参数
  ```yaml
  apiVersion: storage.k8s.io/v1
  kind: StorageClass
  metadata:
    name: managed-nfs-storage
  provisioner: fuseim.pri/ifs #外部制备器提供者，编写为提供者的名称
  parameters:
    archiveOnDelete: "false" #是否存档
  reclaimPolicy: Retain #回收策略
  volumeBindingMode: Immediate #默认为Immediate，表示创建PVC立即进行绑定  
##### 用户创建pvc
- 指定存储大小
- 指定访问模式  
- 引用storageclass
  ```yaml
  cat nfs-sc-statefulset.yaml 
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: nginx-sc
    labels: 
      app: nginx-sc
  spec:
    type: NodePort
    ports:
      - name: web
        port: 80
        protocol: TCP
    selector:
      app: nginx-sc
  ---
  apiVersion: apps/v1
  kind: StatefulSet
  metadata:
    name: nginx-sc
  spec:
    replicas: 1
      #serverName: "nginx-sc"
    selector:
      matchLabels:
        app: nginx-sc
    template:
      metadata:
        labels:
          app: nginx-sc
      spec:
        containers:
        - image: nginx
          name: nginx-sc
          imagePullPolicy: IfNotPresent
          volumeMounts:
          - mountPath: /usr/share/nginx/html
            name: nginx-sc-test-pvc
    volumeClaimTemplates:
      - metadata:
          name: nginx-sc-test-pvc
        spec:
          storageClassName: managed-nfs-storage
          accessModes:
          - ReadWriteMany
          resources:
            requests:
              storage: 1Gi
#####  provisioner创建pv
- 检测到新建pvc
- 根据storageclass找到对应的provisioner
- provisoner创建pv
  ```yaml
  cat nfs-provisioner.yaml 
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: nfs-client-provisioner
    namespace: kube-system
    labels:
      app: nfs-client-provisioner
  spec: 
    replicas: 1
    strategy:
      type: Recreate
    selector:
      matchLabels:
        app: nfs-client-provisioner
    template:
      metadata:
        labels: 
          app: nfs-client-provisioner
      spec:
        
        serviceAccountName: nfs-client-provisioner
        containers:
          - name: nfs-client-provisioner
            image: quay.io/external_storage/nfs-client-provisioner:v4.0.0
            volumeMounts:
              - name: nfs-client-root
                mountPath: /persistentvolumes
            env:
              - name: PROVISIONER_NAME
                value: fuseim.pri/ifs
              - name: NFS_SERVER
                value: 192.168.189.130
              - name: NFS_PATH
                value: /home/data/nfs/rw    
        volumes:
          - name: nfs-client-root
            nfs:
              server: 192.168.189.130
              path: /home/data/nfs/rw
  
基本过程如上，nfs服务器地址写为master节点导致pod部署到master节点，有污点