+++
date = '2025-07-02T15:47:16+08:00'
draft = true
title = '在k8s集群上部署mysql'
+++
## 1.创建存储配置PV与PVC
mysql-pvc.yaml
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  # 如果是AWS/GKE等云环境，可以指定对应的存储类
  # storageClassName: gp2
```
mysql-pv.yaml
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv  # PV名称，需唯一
spec:
  capacity:
    storage: 10Gi  # 存储容量，需与PVC请求一致
  accessModes:
    - ReadWriteOnce  # 访问模式，需与PVC匹配
  persistentVolumeReclaimPolicy: Retain  # 回收策略（Retain/Delete/Recycle）
  hostPath:
    path: /data/mysql  # 节点上的实际存储路径（需提前创建）
```
  
## 2.创建配置文件configmap
```yaml
# mysql-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-config
data:
  my.cnf: |
    [mysqld]
    character-set-server=utf8mb4
    collation-server=utf8mb4_unicode_ci
    default-storage-engine=InnoDB
    max_connections=500
    [client]
    default-character-set=utf8mb4
```
## 3.创建secret存储密码
```bash
kubectl create secret generic mysql-secret \
  --from-literal=root-password="your-root-password" \
  --from-literal=password="your-user-password"
```

## 4.创建deployment部署MySQL实例
```yaml
# mysql-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        ports:
        - containerPort: 3306
          name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: root-password
        - name: MYSQL_DATABASE
          value: mydatabase
        - name: MYSQL_USER
          value: myuser
        - name: MYSQL_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: password
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
        - name: config-volume
          mountPath: /etc/mysql/my.cnf
          subPath: my.cnf
      volumes:
      - name: mysql-persistent-storage
        persistentVolumeClaim:
          claimName: mysql-pvc
      - name: config-volume
        configMap:
          name: mysql-config
```
## 5.创建service
```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  ports:
    - port: 3306
  selector:
    app: mysql
  clusterIP: None  # 使用无头服务，如果需要直接访问Pod
  # 如果需要从集群外部访问，可以使用NodePort或LoadBalancer类型
  # type: NodePort
  # ports:
  # - port: 3306
  #   nodePort: 30006
```
## 6.部署命令
```bash
kubectl apply -f mysql-pvc.yaml
kubectl apply -f mysql-config.yaml
kubectl apply -f mysql-deployment.yaml
kubectl apply -f mysql-service.yaml
```

## 7.在本机上部署和在docker上部署和在k8s集群上部署有什么不同呢
在本机上部署无疑环境不隔离，缺乏扩展性，在docker上部署和在K8s集群上部署区别体现出k8s为何被称为容器编排管理器，k8s通过yaml配置文件定义deployment，service，pvc等资源，由k8s集群自动调度到节点，完全声明式管理。支持通过deployment的replicas字段一键扩容多实例，service负载均衡实现流量分发，扩展能力极强，pv实现数据持久化

## 补充
MySQL作为有状态应用理应使用statefulset而不是deployment，扩展为多节点集群，主从复制MySQL cluster时必须使用statefulset，以确保数据一致性和服务稳定性，此处单节点测试