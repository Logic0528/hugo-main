+++

date = '2025-08-07T17:28:19+08:00'

draft = true

title = '基于 K8s 的微服务项目容器化部署与高可用实践'

+++

#### 项目亮点:

1. **云原生架构**：全服务容器化部署，通过 K8s 实现资源动态调度、故障自愈，符合云原生 “弹性、可扩展” 特性
2. **自动化流程**：CI/CD 流水线将部署周期从小时级缩短至分钟级，减少人工操作错误
3. **全链路监控**：覆盖节点硬件、服务性能、业务指标，可快速定位故障与性能瓶颈
4. ^_^

### 一.服务器角色分配

|   服务器IP   |    角色    | 硬件建议 |                说明                |
| :----------: | :--------: | :------: | :--------------------------------: |
| 10.11.27.171 | Master节点 | 2 核 4G  |            集群控制节点            |
| 10.11.27.172 | Worker节点 | 4 核 8G  |     运行应用容器，Gitlab服务器     |
| 10.11.27.173 | Worker节点 | 4 核 8G  |         运行网关，认证服务         |
| 10.11.27.174 | Worker节点 | 4 核 8G  |         运行网关，权限服务         |
| 10.11.27.175 | Worker节点 | 4 核 8G  | 运行业务服务，前端，Harbor镜像仓库 |

### 二.前期准备工作

在五台服务器上安装docker,k8s,calico等,并在171服务器上初始化k8s集群，详细步骤见另一篇博客，这里不再赘述。

### 三.配置集群基础功能

部署Metrics Server(提供资源监控)
Metrics Server 用于收集Pod/节点的CPU，内存使用数据，是HPA的依赖

```bash
# 下载适配k8s 1.33的配置文件
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.6.4/components.yaml

# 修改配置（解决本地集群证书问题）
kubectl edit deployment metrics-server -n kube-system
# 在spec.template.spec.containers.args中添加：
# - --kubelet-insecure-tls

# 验证Metrics Server状态
kubectl get pods -n kube-system | grep metrics-server

# 查看资源使用情况
kubectl top nodes
kubectl top pods
```

### 四.部署存储解决方案（持久化数据）

微服务需要持久化存储，本地集群推荐使用NFS

1. 选择10.11.27.175作为NFS服务器
   ```bash
   # 安装NFS服务
   yum install -y nfs-utils rpcbind
   
   # 创建共享目录
   mkdir -p /data/nfs/k8s
   chmod 777 /data/nfs/k8s
   
   # 配置NFS共享（允许集群所有节点访问）
   cat >> /etc/exports << EOF
   /data/nfs/k8s 10.11.27.0/24(rw,no_root_squash,sync)
   EOF
   
   # 启动NFS服务
   systemctl start rpcbind
   systemctl start nfs-server
   systemctl enable rpcbind
   systemctl enable nfs-server
   
   # 验证共享
   exportfs -r  # 刷新配置
   exportfs  # 应显示共享目录和允许的网段
   ```

2. 在所有节点安装NFS客户端
   ```bash
   yum install -y nfs-utils
   
   # 验证NFS服务器连接，在任意节点执行即可
   showmount -e 10.11.27.175
   # 预期输出：Export list for 10.11.27.175: /data/nfs/k8s 10.11.27.0/24
   ```

3. 在k8s中部署NFS存储类(StorageClass)
   ```bash
   vim nfs-storageclass.yaml
   
   apiVersion: v1
   kind: ServiceAccount
   metadata:
     name: nfs-provisioner
     namespace: kube-system
   ---
   apiVersion: rbac.authorization.k8s.io/v1
   kind: ClusterRole
   metadata:
     name: nfs-provisioner-runner
   rules:
     - apiGroups: [""]
       resources: ["persistentvolumes"]
       verbs: ["get", "list", "watch", "create", "delete"]
     - apiGroups: [""]
       resources: ["persistentvolumeclaims"]
       verbs: ["get", "list", "watch", "update"]
     - apiGroups: ["storage.k8s.io"]
       resources: ["storageclasses"]
       verbs: ["get", "list", "watch"]
     - apiGroups: [""]
       resources: ["events"]
       verbs: ["create", "update", "patch"]
   ---
   apiVersion: rbac.authorization.k8s.io/v1
   kind: ClusterRoleBinding
   metadata:
     name: run-nfs-provisioner
   subjects:
     - kind: ServiceAccount
       name: nfs-provisioner
       namespace: kube-system
   roleRef:
     kind: ClusterRole
     name: nfs-provisioner-runner
     apiGroup: rbac.authorization.k8s.io
   ---
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: nfs-provisioner
     namespace: kube-system
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: nfs-provisioner
     template:
       metadata:
         labels:
           app: nfs-provisioner
       spec:
         serviceAccountName: nfs-provisioner
         containers:
         - name: nfs-provisioner
           image: registry.cn-hangzhou.aliyuncs.com/lfy_k8s_images/nfs-subdir-external-provisioner:v4.0.2
           volumeMounts:
           - name: nfs-client-root
             mountPath: /persistentvolumes
           env:
           - name: PROVISIONER_NAME
             value: nfs-provisioner  # 存储类名称，后续会引用
           - name: NFS_SERVER
             value: 10.11.27.175    # NFS服务器IP
           - name: NFS_PATH
             value: /data/nfs/k8s   # NFS共享目录
         volumes:
         - name: nfs-client-root
           nfs:
             server: 10.11.27.175
             path: /data/nfs/k8s
   ---
   apiVersion: storage.k8s.io/v1
   kind: StorageClass
   metadata:
     name: nfs-storageclass  # 存储类名称，创建PVC时需指定
   provisioner: nfs-provisioner  # 与Deployment中的PROVISIONER_NAME一致
   parameters:
     archiveOnDelete: "false"  # 删除PVC时自动删除数据
   ```

4. 部署存储类
   ```bash
   kubectl apply -f nfs-storageclass.yaml
   
   # 验证存储类部署
   kubectl get pods -n kube-system | grep nfs-provisioner 
   kubectl get sc 
   ```

### 五.部署私有镜像仓库

在10.11.27.175上部署Harbor作为私有仓库存储pig项目镜像

1. 安装Docker Compose
   ```bash
   # 在175节点执行
   curl -L "https://github.com/docker/compose/releases/download/v2.20.3/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
   chmod +x /usr/local/bin/docker-compose
   docker-compose --version  
   ```

2. 部署Harbor
   ```bash
   # 在175节点下载Harbor
   wget https://github.com/goharbor/harbor/releases/download/v2.8.4/harbor-offline-installer-v2.8.4.tgz
   tar -zxvf harbor-offline-installer-v2.8.4.tgz
   cd harbor
   
   # 复制配置文件并修改
   cp harbor.yml.tmpl harbor.yml
   vi harbor.yml 
   hostname: 10.11.27.175（改为175的IP）
   # 注释掉HTTPS相关配置（测试环境无需HTTPS）
   # 可选：修改默认密码（harbor_admin_password）
   
   # 安装Harbor
   ./install.sh
   
   # 启动Harbor
   docker-compose up -d
   ```

3. 配置信任私有仓库,需在所有节点执行

   ```bash
   cat > /etc/docker/daemon.json << EOF
   {
     "insecure-registries": ["10.11.27.175:80"]
   }
   EOF
   
   # 重启Docker和containerd
   systemctl restart docker
   systemctl restart containerd
   
   # 登录Harbor（默认账号admin，密码见harbor.yml配置）
   docker login 10.11.27.175:80
   ```

### 六.构建pig项目镜像并推送至Harbor

1. 安装构建依赖
   ```bash
   # 175节点上执行
   yum install -y git maven
   ```

2. 克隆代码并修改配置
   ```bash
   git clone https://github.com/pig-mesh/pig.git
   cd pig
   
   # 修改配置文件，适配k8s环境：
   # 1. 所有服务的Nacos地址改为k8s内部服务名（后续会创建nacos-service）
   find . -name "application.yml" -exec sed -i 's/server-addr: .*/server-addr: nacos-service:8848/g' {} \;
   
   # 2. 数据库地址改为k8s内部服务名（后续会创建mysql-service）
   find . -name "application.yml" -exec sed -i 's/url: jdbc:mysql:\/\/.*/url: jdbc:mysql:\/\/mysql-service:3306\/pig?useUnicode=true&characterEncoding=utf8&serverTimezone=Asia\/Shanghai/g' {} \;
   
   # 3. Redis地址改为k8s内部服务名（后续会创建redis-service）
   find . -name "application.yml" -exec sed -i 's/host: .*/host: redis-service/g' {} \;
   ```

3. 构建Docker镜像并推送
   ```bash
   # 构建基础镜像（项目根目录下）
   mvn clean package -DskipTests
   
   # 构建并推送Nacos镜像（pig-register）
   cd pig-register
   docker build -t 10.11.27.175:80/pig/pig-register:latest .
   docker push 10.11.27.175:80/pig/pig-register:latest
   
   # 构建并推送网关镜像（pig-gateway）
   cd ../pig-gateway
   docker build -t 10.11.27.175:80/pig/pig-gateway:latest .
   docker push 10.11.27.175:80/pig/pig-gateway:latest
   
   # 构建并推送认证服务（pig-auth）
   cd ../pig-auth
   docker build -t 10.11.27.175:80/pig/pig-auth:latest .
   docker push 10.11.27.175:80/pig/pig-auth:latest
   
   # 构建并推送权限服务（pig-upms-biz）
   cd ../pig-upms/pig-upms-biz
   docker build -t 10.11.27.175:80/pig/pig-upms-biz:latest .
   docker push 10.11.27.175:80/pig/pig-upms-biz:latest
   
   # 构建并推送监控服务（pig-monitor）
   cd /root/pig/pig-visual/pig-monitor
   docker build -t 10.11.27.175:80/pig/pig-monitor:latest .
   docker push 10.11.27.175:80/pig/pig-monitor:latest
   
   # 构建并推送代码生成服务（pig-codegen）
   cd /root/pig/pig-visual/pig-codegen
   docker build -t 10.11.27.175:80/pig/pig-codegen:latest .
   docker push 10.11.27.175:80/pig/pig-codegen:latest
   
   # 构建并推送定时任务服务（pig-quartz）
   cd /root/pig/pig-visual/pig-quartz
   docker build -t 10.11.27.175:80/pig/pig-quartz:latest .
   docker push 10.11.27.175:80/pig/pig-quartz:latest
   ```

4. 验证镜像推送结果
   浏览器访问http://10.11.27.175:80,在pig项目下能看到推送的镜像

### 七.部署pig项目依赖服务

1. 部署MySQL，使用存储类持久化数据
   ```bash
   vim mysql-deploy.yaml
   
   apiVersion: v1
   kind: PersistentVolumeClaim
   metadata:
     name: mysql-pvc
   spec:
     accessModes:
     - ReadWriteOnce
     storageClassName: nfs-storageclass 
     resources:
       requests:
         storage: 10Gi  
   ---
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: mysql
   spec:
     replicas: 1
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
           image: mysql:8.0
           ports:
           - containerPort: 3306
           env:
           - name: MYSQL_ROOT_PASSWORD
             value: "root"  # 数据库密码，需与pig项目配置一致
           - name: MYSQL_DATABASE
             value: "pig"   # 初始化数据库名
           volumeMounts:
           - name: mysql-data
             mountPath: /var/lib/mysql
           - name: init-sql
             mountPath: /docker-entrypoint-initdb.d  # 初始化SQL目录
         volumes:
         - name: mysql-data
           persistentVolumeClaim:
             claimName: mysql-pvc
         - name: init-sql
           hostPath:
             path: /root/pig/db  # 指向节点上pig项目的db目录（需确保该目录存在）
   ---
   apiVersion: v1
   kind: Service
   metadata:
     name: mysql-service
   spec:
     selector:
       app: mysql
     ports:
     - port: 3306
       targetPort: 3306
     clusterIP: None  # 无头服务，适合数据库
   ```

   部署MySQL
   ```bash
   # 确保pig项目的db目录在部署节点存在
   kubectl apply -f mysql-deploy.yaml
   
   # 验证MySQL状态
   kubectl get pods | grep mysql
   ```

2. 部署redis
   ```bash
   vim redis-deploy.yaml
   
   apiVersion: v1
   kind: PersistentVolumeClaim
   metadata:
     name: redis-pvc
   spec:
     accessModes:
     - ReadWriteOnce
     storageClassName: nfs-storageclass
     resources:
       requests:
         storage: 5Gi
   ---
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: redis
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: redis
     template:
       metadata:
         labels:
           app: redis
       spec:
         containers:
         - name: redis
           image: redis:7.0
           ports:
           - containerPort: 6379
           command: ["redis-server", "--requirepass", "redis"]  # 密码与pig配置一致
           volumeMounts:
           - name: redis-data
             mountPath: /data
         volumes:
         - name: redis-data
           persistentVolumeClaim:
             claimName: redis-pvc
   ---
   apiVersion: v1
   kind: Service
   metadata:
     name: redis-service
   spec:
     selector:
       app: redis
     ports:
     - port: 6379
       targetPort: 6379
   ```

   部署redis
   ```bash
   kubectl apply -f redis-deploy.yaml
   kubectl get pods | grep redis
   ```

3. 部署Nacos集群
   ```bash
   vim nacos-deploy.yaml
   
   apiVersion: v1
   kind: PersistentVolumeClaim
   metadata:
     name: nacos-pvc-0
   spec:
     accessModes:
     - ReadWriteOnce
     storageClassName: nfs-storageclass
     resources:
       requests:
         storage: 5Gi
   ---
   apiVersion: v1
   kind: PersistentVolumeClaim
   metadata:
     name: nacos-pvc-1
   spec:
     accessModes:
     - ReadWriteOnce
     storageClassName: nfs-storageclass
     resources:
       requests:
         storage: 5Gi
   ---
   apiVersion: v1
   kind: Service
   metadata:
     name: nacos-service
   spec:
     ports:
     - port: 8848
       name: server
     clusterIP: None
     selector:
       app: nacos
   ---
   apiVersion: apps/v1
   kind: StatefulSet
   metadata:
     name: nacos
   spec:
     serviceName: nacos-service
     replicas: 2
     selector:
       matchLabels:
         app: nacos
     template:
       metadata:
         labels:
           app: nacos
       spec:
         containers:
         - name: nacos
           image: 10.11.27.175:80/pig/pig-register:latest  # 私有仓库镜像
           ports:
           - containerPort: 8848
           env:
           - name: MODE
             value: "cluster"
           - name: NACOS_SERVERS
             value: "nacos-0.nacos-service.default.svc.cluster.local:8848,nacos-1.nacos-service.default.svc.cluster.local:8848"
           - name: SPRING_DATASOURCE_PLATFORM
             value: "mysql"
           - name: MYSQL_SERVICE_HOST
             value: "mysql-service"
           - name: MYSQL_SERVICE_DB_NAME
             value: "pig_config"
           - name: MYSQL_SERVICE_USER
             value: "root"
           - name: MYSQL_SERVICE_PASSWORD
             value: "root"
           volumeMounts:
           - name: nacos-data
             mountPath: /home/nacos/data
     volumeClaimTemplates:
     - metadata:
         name: nacos-data
       spec:
         accessModes: [ "ReadWriteOnce" ]
         storageClassName: "nfs-storageclass"
         resources:
           requests:
             storage: 5Gi
   ```

   部署nacos
   ```bash
   kubectl apply -f nacos-deploy.yaml
   kubectl get pods | grep nacos  # 2个实例均为Running
   ```

### 八.部署pig核心微服务

1. 部署网关 pig-gateway
   ```bash
   vim gateway-deploy.yaml
   
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: pig-gateway
   spec:
     replicas: 2  # 2个实例实现负载均衡
     selector:
       matchLabels:
         app: pig-gateway
     template:
       metadata:
         labels:
           app: pig-gateway
         annotation:
           prometheus.io/scrape: "true"  # 允许Prometheus采集
           prometheus.io/path: "/actuator/prometheus"  # 指标暴露路径
           prometheus.io/port: "9999"  # 指标暴露端口（网关的端口）
       spec:
         containers:
         - name: pig-gateway
           image: 10.11.27.175:80/pig/pig-gateway:latest
           ports:
           - containerPort: 9999
           livenessProbe:  # 健康检查
             httpGet:
               path: /actuator/health
               port: 9999
             initialDelaySeconds: 60
             periodSeconds: 10
   ---
   apiVersion: v1
   kind: Service
   metadata:
     name: pig-gateway-service
   spec:
     selector:
       app: pig-gateway
     ports:
     - port: 80
       targetPort: 9999
   ```
   
   部署网关
   
   ```bash
   kubectl apply -f gateway-deploy.yaml
   kubectl get pods | grep pig-gateway
   ```
   
2. 部署认证服务 pig-auth

   ```bash
   vim auth-deploy.yaml
   
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: pig-auth
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: pig-auth
     template:
       metadata:
         labels:
           app: pig-auth
         annotations:
           prometheus.io/scrape: "true"
           prometheus.io/path: "/actuator/prometheus"
           prometheus.io/port: "3000"  # 认证服务的端口
       spec:
         containers:
         - name: pig-auth
           image: 10.11.27.175:80/pig/pig-auth:latest
           ports:
           - containerPort: 3000
   ---
   apiVersion: v1
   kind: Service
   metadata:
     name: pig-auth-service
   spec:
     selector:
       app: pig-auth
     ports:
     - port: 3000
       targetPort: 3000
   ```

   部署认证服务

   ```bash
   kubectl apply -f auth-deploy.yaml
   kubectl get pods | grep pig-auth 
   ```

3. 部署权限服务 pig-upms-biz

   ```bash
   vim upms-deploy.yaml
   
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: pig-upms-biz
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: pig-upms-biz
     template:
       metadata:
         labels:
           app: pig-upms-biz
         annotations:
           prometheus.io/scrape: "true"
           prometheus.io/path: "/actuator/prometheus"
           prometheus.io/port: "4000"  # 权限服务的端口
       spec:
         containers:
         - name: pig-upms-biz
           image: 10.11.27.175:80/pig/pig-upms-biz:latest
           ports:
           - containerPort: 4000
           env:
           - name: SPRING_PROFILES_ACTIVE
             value: "prod"
   ---
   apiVersion: v1
   kind: Service
   metadata:
     name: pig-upms-biz-service
   spec:
     selector:
       app: pig-upms-biz
     ports:
     - port: 4000
       targetPort: 4000
   ```

   部署权限服务

   ```bash
   kubectl apply -f upms-deploy.yaml
   kubectl get pods | grep pig-upms-biz  
   ```

4. 部署监控服务 pig-monitor

   ```bash
   vim monitor-deploy.yaml
   
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: pig-monitor
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: pig-monitor
     template:
       metadata:
         labels:
           app: pig-monitor
         annotations:
           prometheus.io/scrape: "true"
           prometheus.io/path: "/actuator/prometheus"
           prometheus.io/port: "5001"  # 权限服务的端口
       spec:
         containers:
         - name: pig-monitor
           image: 10.11.27.175:80/pig/pig-monitor:latest
           ports:
           - containerPort: 5001
   ---
   apiVersion: v1
   kind: Service
   metadata:
     name: pig-monitor-service
   spec:
     selector:
       app: pig-monitor
     ports:
     - port: 5001
       targetPort: 5001
   ```

   部署监控服务

   ```bash
   kubectl apply -f monitor-deploy.yaml
   ```

5. 部署代码生成服务 pig-codegen

   ```bash
   vim codegen-deploy.yaml
   
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: pig-codegen
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: pig-codegen
     template:
       metadata:
         labels:
           app: pig-codegen
         annotations:
           prometheus.io/scrape: "true"
           prometheus.io/path: "/actuator/prometheus"
           prometheus.io/port: "5003"  # 权限服务的端口
       spec:
         containers:
         - name: pig-codegen
           image: 10.11.27.175:80/pig/pig-codegen:latest
           ports:
           - containerPort: 5003
   ---
   apiVersion: v1
   kind: Service
   metadata:
     name: pig-codegen-service
   spec:
     selector:
       app: pig-codegen
     ports:
     - port: 5003
       targetPort: 5003
   ```

   部署代码生成服务

   ```bash
   kubectl apply -f codegen-deploy.yaml
   ```

6. 部署定时任务服务 pig-quartz

   ```bash
   vim quartz-deploy.yaml
   
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: pig-quartz
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: pig-quartz
     template:
       metadata:
         labels:
           app: pig-quartz
         annotations:
           prometheus.io/scrape: "true"
           prometheus.io/path: "/actuator/prometheus"
           prometheus.io/port: "5006"  # 权限服务的端口
       spec:
         containers:
         - name: pig-quartz
           image: 10.11.27.175:80/pig/pig-quartz:latest
           ports:
           - containerPort: 5006
   ---
   apiVersion: v1
   kind: Service
   metadata:
     name: pig-quartz-service
   spec:
     selector:
       app: pig-quartz
     ports:
     - port: 5006
       targetPort: 5006
   ```
   
   部署

   ```bash
   kubectl apply -f quartz-deploy.yaml
   ```
   
   

### 九.部署Ingress控制器(外部访问入口)

部署Ingress-NGINX,实现外部流量路由到网关服务

1. 部署Ingress控制器

   ```bash
   kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.9.4/deploy/static/provider/baremetal/deploy.yaml
   
   # 验证部署
   kubectl get pods -n ingress-nginx 
   ```

2. 配置Ingress路由规则

   ```bash
   vim pig-ingress.yaml
   
   apiVersion: networking.k8s.io/v1
   kind: Ingress
   metadata:
     name: pig-ingress
     annotations:
       nginx.ingress.kubernetes.io/rewrite-target: /
   spec:
     ingressClassName: nginx
     rules:
     - host: pig.local  # 本地测试域名
       http:
         paths:
         - path: /
           pathType: Prefix
           backend:
             service:
               name: pig-gateway-service
               port:
                 number: 80
   ```

   部署Ingress

   ```bash
   kubectl apply -f pig-ingress.yaml
   
   # 查看Ingress状态
   kubectl get ingress 
   ```

### 十、验证应用访问

1. 查看Ingress控制器暴露的端口

   ```bash
   kubectl get svc -n ingress-nginx
   ```

2. 配置本地hosts
   ```bash
   10.11.27.172 pig.local  # 指向任意worker节点IP
   ```

3. 访问应用

   浏览器访问`http://pig.local:30080/admin`（使用上一步的 NodePort），预期能看到`pig`项目的登录页面（默认账号`admin`，密码`123456`）

### 十一、部署前端

1. 构建并推送前端镜像

   ```bash
   # 克隆前端代码（在175节点）
   git clone https://gitee.com/log4j/pig-ui.git
   cd pig-ui
   
   # 安装依赖并打包（需Node.js环境）
   yum install -y nodejs npm
   npm install
   npm run build  # 生成dist目录
   
   # 构建Docker镜像
   cat > Dockerfile << EOF
   FROM nginx:alpine
   COPY dist/ /usr/share/nginx/html/
   COPY nginx.conf /etc/nginx/conf.d/default.conf  # 可选：自定义Nginx配置
   EOF
   
   # 自定义Nginx配置（解决跨域问题）
   cat > nginx.conf << EOF
   server {
       listen 80;
       root /usr/share/nginx/html;
       index index.html;
   
       # 转发API请求到网关
       location /api/ {
           proxy_pass http://pig-gateway-service:80/;
       }
   }
   EOF
   
   # 构建并推送镜像
   docker build -t 10.11.27.175:80/pig/pig-ui:latest .
   docker push 10.11.27.175:80/pig/pig-ui:latest
   ```

2. 在k8s中部署前端

   ```bash
   vim ui-deploy.yaml
   
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: pig-ui
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: pig-ui
     template:
       metadata:
         labels:
           app: pig-ui
       spec:
         containers:
         - name: pig-ui
           image: 10.11.27.175:80/pig/pig-ui:latest
           ports:
           - containerPort: 80
   ---
   apiVersion: v1
   kind: Service
   metadata:
     name: pig-ui-service
   spec:
     selector:
       app: pig-ui
     ports:
     - port: 80
       targetPort: 80
   ```

   部署

   ```bash
   kubectl apply -f ui-deploy.yaml
   ```

3. 配置Ingress路由前端

   ```bash
   # 修改pig-ingress.yaml,添加前端路由：
   
   apiVersion: networking.k8s.io/v1
   kind: Ingress
   metadata:
     name: pig-ingress
     annotations:
       nginx.ingress.kubernetes.io/rewrite-target: /
   spec:
     ingressClassName: nginx
     rules:
     - host: pig.local
       http:
         paths:
         - path: /admin  # 前端路径
           pathType: Prefix
           backend:
             service:
               name: pig-ui-service
               port:
                 number: 80
         - path: /  # API路径
           pathType: Prefix
           backend:
             service:
               name: pig-gateway-service
               port:
                 number: 80
   ```

   更新Ingress

   ```bash
   kubectl apply -f pig-ingress.yaml
   ```

   访问验证：浏览器打开`http://pig.local:30080/admin`（30080 为 Ingress 的 NodePort）

### 十二、配置HPA自动扩缩容

1. 创建HPA配置

   ```bash
   vim gate-hpa.yaml
   
   apiVersion: autoscaling/v2
   kind: HorizontalPodAutoscaler
   metadata:
     name: pig-gateway-hpa
   spec:
     scaleTargetRef:
       apiVersion: apps/v1
       kind: Deployment
       name: pig-gateway  # 关联网关的Deployment
     minReplicas: 2  # 最小实例数
     maxReplicas: 5  # 最大实例数
     metrics:
     - type: Resource
       resource:
         name: cpu
         target:
           type: Utilization
           averageUtilization: 70  # CPU使用率超过70%时扩容
     - type: Resource
       resource:
         name: memory
         target:
           type: Utilization
           averageUtilization: 80  # 内存使用率超过80%时扩容
   ```

   部署HPA

   ```bash
   kubectl apply -f gateway-hpa.yaml
   
   # 验证HPA
   kubectl get hpa
   # 输出应显示当前实例数、目标CPU/内存使用率
   ```

### 十三、Kubernetes原生部署监控系统 Prometheus + Grafana

1. 通过 K8s 部署时，需要定义以下资源：

   - Namespace: 独立命令空间，隔离监控组件
   - ConfigMap: 存储Prometheus配置，如监控目标，采集规则
   - Deployment: 部署Grafana(无状态应用)
   - StatufalSet: 部署Prometheus(有状态应用，需要持久化存储指标数据)
   - Service: 暴露Prometheus和Grafana的访问入口
   - PersistentVolumeClaim: 为Prometheus提供持久化存储

2. 创建命名空间
   ```bash
   vim monitoring-namespace.yaml
   
   apiVersion: v1
   kind: Namespace
   metadata:
   	name: monitoring
   	
   kubectl apply -f monitoring-namespace.yaml
   ```

3. 部署Prometheus

   - 创建Prometheus ConfigMap配置文件

     ```bash
     # prometheus-config.yaml
     apiVersion: v1
     kind: ConfigMap
     metadata:
       name: prometheus-config
       namespace: monitoring
     data:
       prometheus.yml: |
         global:
           scrape_interval: 15s  # 默认采集间隔
     
         # 监控目标配置
         scrape_configs:
           # 1. 监控 K8s 节点（通过 node-exporter）
           - job_name: 'node-exporter'
             kubernetes_sd_configs:
               - role: node  # 自动发现所有节点
             relabel_configs:
               - source_labels: [__address__]
                 action: replace
                 regex: '(.*):10250'
                 replacement: '${1}:9100'  # 指向 node-exporter 的 9100 端口
               - action: labelmap
                 regex: __meta_kubernetes_node_label_(.+)
     
           # 2. 监控 K8s 集群组件（如 apiserver）
           - job_name: 'kubernetes-apiservers'
             kubernetes_sd_configs:
               - role: endpoints
             scheme: https
             tls_config:
               ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
             bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
             relabel_configs:
               - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
                 action: keep
                 regex: default;kubernetes;https
     
           # 3. 监控所有 Pod（自动发现）
           - job_name: 'kubernetes-pods'
             kubernetes_sd_configs:
               - role: pod
             relabel_configs:
               - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
                 action: keep
                 regex: true  # 只采集带 annotation: prometheus.io/scrape: "true" 的 Pod
     ```

   - 部署node-expoter,通过DaemonSet在所有节点部署node-expoter,采集节点硬件指标
     ```bash
     # node-exporter.yaml
     apiVersion: apps/v1
     kind: DaemonSet
     metadata:
       name: node-exporter
       namespace: monitoring
     spec:
       selector:
         matchLabels:
           app: node-exporter
       template:
         metadata:
           labels:
             app: node-exporter
         spec:
           containers:
           - name: node-exporter
             image: prom/node-exporter:v1.5.0
             ports:
             - containerPort: 9100
               hostPort: 9100  # 暴露到节点端口，便于 Prometheus 访问
             volumeMounts:
             - name: proc
               mountPath: /host/proc
               readOnly: true
             - name: sys
               mountPath: /host/sys
               readOnly: true
           volumes:
           - name: proc
             hostPath:
               path: /proc
           - name: sys
             hostPath:
               path: /sys
     ---
     # 暴露 node-exporter 服务（可选，用于调试）
     apiVersion: v1
     kind: Service
     metadata:
       name: node-exporter
       namespace: monitoring
     spec:
       selector:
         app: node-exporter
       ports:
       - port: 9100
         targetPort: 9100
       type: ClusterIP
     ```

   - 部署Prometheus StatefulSet+ PVC

     ```bash
     # prometheus-deploy.yaml
     apiVersion: v1
     kind: PersistentVolumeClaim
     metadata:
       name: prometheus-data
       namespace: monitoring
     spec:
       accessModes:
         - ReadWriteOnce
       resources:
         requests:
           storage: 10Gi  # 按需调整存储大小
       storageClassName: 
     ---
     apiVersion: apps/v1
     kind: StatefulSet
     metadata:
       name: prometheus
       namespace: monitoring
     spec:
       serviceName: prometheus
       replicas: 1
       selector:
         matchLabels:
           app: prometheus
       template:
         metadata:
           labels:
             app: prometheus
         spec:
           containers:
           - name: prometheus
             image: prom/prometheus:v2.45.0
             args:
               - "--config.file=/etc/prometheus/prometheus.yml"
               - "--storage.tsdb.path=/prometheus"
               - "--web.enable-lifecycle"  # 支持热重载配置
             ports:
             - containerPort: 9090
             volumeMounts:
             - name: config-volume
               mountPath: /etc/prometheus
             - name: prometheus-data
               mountPath: /prometheus
           volumes:
           - name: config-volume
             configMap:
               name: prometheus-config
       volumeClaimTemplates:
       - metadata:
           name: prometheus-data
         spec:
           accessModes: [ "ReadWriteOnce" ]
           resources:
             requests:
               storage: 10Gi
           storageClassName: nfs-storageclass
     ---
     # 暴露 Prometheus 服务
     apiVersion: v1
     kind: Service
     metadata:
       name: prometheus
       namespace: monitoring
     spec:
       selector:
         app: prometheus
       ports:
       - port: 9090
         targetPort: 9090
       type: NodePort  # 暴露到节点端口，便于外部访问
     ```

   - 在`pom.xml`文件中添加Prometheus指标暴露依赖

     ```bash
     <dependency>
         <groupId>org.springframework.boot</groupId>
         <artifactId>spring-boot-starter-actuator</artifactId>
     </dependency>
     <dependency>
         <groupId>io.micrometer</groupId>
         <artifactId>micrometer-registry-prometheus</artifactId>
     </dependency>
     ```

   - 在 `application.yml`中开启端点

     ```bash
     management:
       endpoints:
         web:
           exposure:
             include: prometheus,health  # 暴露 prometheus 端点
       metrics:
         export:
           prometheus:
             enabled: true
     ```

     

   - 应用配置文件与验证状态

     ```bash
     kubectl apply -f prometheus-config.yaml
     kubectl apply -f node-exporter.yaml
     kubectl apply -f prometheus-deploy.yaml
     
     kubectl get pods -n monitoring | grep prometheus  
     kubectl get svc -n monitoring prometheus  # 查看 NodePort
     ```

4. 部署Grafana

   - 部署Grafana Deployment + Service

     ```bash
     # grafana-deploy.yaml
     apiVersion: apps/v1
     kind: Deployment
     metadata:
       name: grafana
       namespace: monitoring
     spec:
       replicas: 1
       selector:
         matchLabels:
           app: grafana
       template:
         metadata:
           labels:
             app: grafana
         spec:
           containers:
           - name: grafana
             image: grafana/grafana:9.5.2
             ports:
             - containerPort: 3000
             env:
             - name: GF_SECURITY_ADMIN_PASSWORD
               value: "grafana123"  # 初始密码
             volumeMounts:
             - name: grafana-data
               mountPath: /var/lib/grafana
           volumes:
           - name: grafana-data
             emptyDir: {}  # 临时存储，生产环境建议用 PVC 持久化
     ---
     # 暴露 Grafana 服务
     apiVersion: v1
     kind: Service
     metadata:
       name: grafana
       namespace: monitoring
     spec:
       selector:
         app: grafana
       ports:
       - port: 3000
         targetPort: 3000
       type: NodePort  # 暴露到节点端口
     ```

   - 应用配置文件并验证

     ```bash
     kubectl apply -f grafana-deploy.yaml
     
     kubectl get pods -n monitoring | grep grafana  # 状态应为 Running
     kubectl get svc -n monitoring grafana  # 查看 NodePort
     ```

     

### 十四、数据备份

1. 创建备份脚本backup.sh
   ```bash
   #!/bin/bash
   BACKUP_DIR="/data/backup"
   TIMESTAMP=$(date +%Y%m%d_%H%M%S)
   NFS_DATA_DIR="/data/nfs/k8s"  # NFS共享目录
   
   # 创建备份目录
   mkdir -p $BACKUP_DIR
   
   # 压缩备份
   tar -zcvf $BACKUP_DIR/nfs-backup-$TIMESTAMP.tar.gz $NFS_DATA_DIR
   
   # 保留最近30天的备份
   find $BACKUP_DIR -name "nfs-backup-*.tar.gz" -mtime +30 -delete
   ```

2. 配置定时任务

   ```bash
   chmod +x backup.sh
   
   crontab -e
   # 添加以下内容：
   0 2 * * * /root/backup.sh
   ```

### 十五、搭建CI/CD流水线

1. 部署GitLab服务器

   ```bash
   # 创建数据目录（持久化GitLab数据）
   mkdir -p /data/gitlab/{config,logs,data}
   
   # 启动GitLab
   docker run -d \
     --hostname 10.11.27.172 \
     --publish 80:80 --publish 443:443 \
     --name gitlab \
     --restart always \
     --volume /data/gitlab/config:/etc/gitlab \
     --volume /data/gitlab/logs:/var/log/gitlab \
     --volume /data/gitlab/data:/var/opt/gitlab \
     gitlab/gitlab-ce:16.0.0-ce.0
   ```

2. 初始化Gitlab

   - 访问http://10.11.27.172(首次需等待较长时间)

   - 获取初始密码
     ```bash
     docker exec -it gitlab grep 'Password:' /etc/gitlab/initial_root_password
     ```

   - 登录账号：root，输入密码后修改为自定义密码

   - 创建项目

3. 部署Gitlab Runner

   ```bash
   # 启动Runner
   docker run -d \
     --name gitlab-runner \
     --restart always \
     --volume /srv/gitlab-runner/config:/etc/gitlab-runner \
     --volume /var/run/docker.sock:/var/run/docker.sock \  # 允许操作Docker
     gitlab/gitlab-runner:v16.0.0
   ```

4. 注册Runner到GitLab

   - 在GitLab项目页面获取注册信息
     进入项目->Settings->CI/CD->Runners->Expand,记下URL和Registration token

   - 执行注册命令
     ```bash
     docker exec -it gitlab-runner gitlab-runner register
     ```

   - Gitlab instance URL: http://10.11.27.172
   - Registration tokne: 上述令牌
   - Tags: pig,k8s,172
   - Executor: docker
   - Default Docker image: maven:3.8-openjdk-11

5. 配置Runner权限

   Runner需要访问k8s集群，推送镜像到175节点的Harbor

   ```bash
   # 1. 从k8s Master节点（171）复制kubeconfig到172节点
   # 在171节点执行：
   scp /root/.kube/config root@10.11.27.172:/root/.kube/config
   
   # 2. 在172节点创建Maven缓存目录
   mkdir -p /data/maven/repository
   
   # 3. 编辑Runner配置
   vi /srv/gitlab-runner/config/config.toml
   
   # 在[[runners]]下的[runners.docker]中添加volumes配置：
   [[runners]]
     ...
     [runners.docker]
       ...
       volumes = [
         "/var/run/docker.sock:/var/run/docker.sock",
         "/root/.kube/config:/root/.kube/config",  # 挂载k8s配置
         "/data/maven/repository:/root/.m2/repository"  # 缓存Maven依赖
       ]
   
   # 4. 重启Runner
   docker restart gitlab-runner
   
   docker login 10.11.27.175:80
   ```

### 十六、编写CI/CD流水线配置文件

```bash
# 定义流水线阶段（按顺序执行）
stages:
  - build       # 编译代码
  - test        # 单元测试
  - build-image # 构建Docker镜像
  - push-image  # 推送镜像到Harbor（175节点）
  - deploy      # 部署到k8s集群

# 全局变量
variables:
  HARBOR_URL: "10.11.27.175:80"  # Harbor在175节点
  HARBOR_PROJECT: "pig"          # Harbor中的项目名
  K8S_NAMESPACE: "default"       # 部署到k8s的default命名空间
  MAVEN_OPTS: "-Dmaven.repo.local=/root/.m2/repository"  # 复用Maven缓存

# ${CI_COMMIT_SHORT_SHA} 是 GitLab CI/CD 内置的环境变量，用于表示当前代码提交的短哈希值（Short Commit SHA），是 Git 提交记录的唯一标识符

# 阶段1：编译代码
build:
  stage: build
  tags: [pig, k8s, 172]  # 匹配172节点的Runner
  image: maven:3.8-openjdk-11  # 用Maven镜像编译
  script:
    - echo "编译代码..."
    - mvn clean package -DskipTests  # 跳过测试（单独阶段执行）
  artifacts:
    paths:
      - "**/target/*.jar"  # 保存编译产物
    expire_in: 1 hour      # 1小时后自动清理

# 阶段2：单元测试
test:
  stage: test
  tags: [pig, k8s, 172]
  image: maven:3.8-openjdk-11
  script:
    - echo "运行单元测试..."
    - mvn test
  artifacts:
    paths:
      - "**/target/surefire-reports/"  # 保存测试报告

# 阶段3-5：网关服务（pig-gateway）的构建、推送、部署
build-gateway-image:
  stage: build-image
  tags: [pig, k8s, 172]
  image: docker:20.10  # 用Docker镜像构建
  dependencies:
    - build  # 依赖编译阶段的产物
  script:
    - echo "构建网关镜像..."
    - cd pig-gateway
    - docker build -t $HARBOR_URL/$HARBOR_PROJECT/pig-gateway:${CI_COMMIT_SHORT_SHA} .

push-gateway-image:
  stage: push-image
  tags: [pig, k8s, 172]
  image: docker:20.10
  script:
    - echo "登录Harbor..."
    - docker login -u admin -p $HARBOR_PASSWORD $HARBOR_URL  # 密码从GitLab变量获取
    - echo "推送网关镜像..."
    - docker push $HARBOR_URL/$HARBOR_PROJECT/pig-gateway:${CI_COMMIT_SHORT_SHA}

deploy-gateway:
  stage: deploy
  tags: [pig, k8s, 172]
  image: bitnami/kubectl:1.28  # 带kubectl的镜像
  script:
    - echo "更新k8s网关部署..."
    - kubectl set image deployment/pig-gateway pig-gateway=$HARBOR_URL/$HARBOR_PROJECT/pig-gateway:${CI_COMMIT_SHORT_SHA} -n $K8S_NAMESPACE
    - kubectl rollout status deployment/pig-gateway -n $K8S_NAMESPACE  # 等待部署完成

# 阶段3-5：认证服务（pig-auth）的构建、推送、部署
build-auth-image:
  stage: build-image
  tags: [pig, k8s, 172]
  image: docker:20.10
  dependencies:
    - build
  script:
    - echo "构建认证服务镜像..."
    - cd pig-auth
    - docker build -t $HARBOR_URL/$HARBOR_PROJECT/pig-auth:${CI_COMMIT_SHORT_SHA} .

push-auth-image:
  stage: push-image
  tags: [pig, k8s, 172]
  image: docker:20.10
  script:
    - docker login -u admin -p $HARBOR_PASSWORD $HARBOR_URL
    - docker push $HARBOR_URL/$HARBOR_PROJECT/pig-auth:${CI_COMMIT_SHORT_SHA}

deploy-auth:
  stage: deploy
  tags: [pig, k8s, 172]
  image: bitnami/kubectl:1.28
  script:
    - echo "更新k8s认证服务部署..."
    - kubectl set image deployment/pig-auth pig-auth=$HARBOR_URL/$HARBOR_PROJECT/pig-auth:${CI_COMMIT_SHORT_SHA} -n $K8S_NAMESPACE
    - kubectl rollout status deployment/pig-auth -n $K8S_NAMESPACE
# 阶段3-5：权限服务（pig-upms-biz）的构建、推送、部署
build-upms-biz-image:
  stage: build-image
  tags: [pig, k8s, 172]
  image: docker:20.10
  dependencies:
    - build
  script:
    - echo "构建权限服务镜像..."
    - cd pig-upms/pig-upms-biz  
    - docker build -t $HARBOR_URL/$HARBOR_PROJECT/pig-upms-biz:${CI_COMMIT_SHORT_SHA} .
    
push-upms-biz-image:
  stage: push-image
  tags: [pig, k8s, 172]
  image: docker:20.10
  script:
    - docker login -u admin -p $HARBOR_PASSWORD $HARBOR_URL
    - echo "推送权限服务镜像..."
    - docker push $HARBOR_URL/$HARBOR_PROJECT/pig-upms-biz:${CI_COMMIT_SHORT_SHA}

deploy-upms-biz:
  stage: deploy
  tags: [pig, k8s, 172]
  image: bitnami/kubectl:1.28
  script:
    - echo "更新k8s权限服务部署..."
    - kubectl set image deployment/pig-upms-biz pig-upms-biz=$HARBOR_URL/$HARBOR_PROJECT/pig-upms-biz:${CI_COMMIT_SHORT_SHA} -n $K8S_NAMESPACE
    - kubectl rollout status deployment/pig-upms-biz -n $K8S_NAMESPACE

# 阶段3-5：监控服务（pig-monitor）的构建、推送、部署
build-monitor-image:
  stage: build-image
  tags: [pig, k8s, 172]
  image: docker:20.10
  dependencies:
    - build
  script:
    - echo "构建监控服务镜像..."
    - cd pig-visual/pig-monitor  # 路径：visual子模块下
    - docker build -t $HARBOR_URL/$HARBOR_PROJECT/pig-monitor:${CI_COMMIT_SHORT_SHA} .

push-monitor-image:
  stage: push-image
  tags: [pig, k8s, 172]
  image: docker:20.10
  script:
    - docker login -u admin -p $HARBOR_PASSWORD $HARBOR_URL
    - echo "推送监控服务镜像..."
    - docker push $HARBOR_URL/$HARBOR_PROJECT/pig-monitor:${CI_COMMIT_SHORT_SHA}

deploy-monitor:
  stage: deploy
  tags: [pig, k8s, 172]
  image: bitnami/kubectl:1.28
  script:
    - echo "更新k8s监控服务部署..."
    - kubectl set image deployment/pig-monitor pig-monitor=$HARBOR_URL/$HARBOR_PROJECT/pig-monitor:${CI_COMMIT_SHORT_SHA} -n $K8S_NAMESPACE
    - kubectl rollout status deployment/pig-monitor -n $K8S_NAMESPACE

# 阶段3-5：代码生成服务（pig-codegen）的构建、推送、部署
build-codegen-image:
  stage: build-image
  tags: [pig, k8s, 172]
  image: docker:20.10
  dependencies:
    - build
  script:
    - echo "构建代码生成服务镜像..."
    - cd pig-visual/pig-codegen
    - docker build -t $HARBOR_URL/$HARBOR_PROJECT/pig-codegen:${CI_COMMIT_SHORT_SHA} .

push-codegen-image:
  stage: push-image
  tags: [pig, k8s, 172]
  image: docker:20.10
  script:
    - docker login -u admin -p $HARBOR_PASSWORD $HARBOR_URL
    - echo "推送代码生成服务镜像..."
    - docker push $HARBOR_URL/$HARBOR_PROJECT/pig-codegen:${CI_COMMIT_SHORT_SHA}

deploy-codegen:
  stage: deploy
  tags: [pig, k8s, 172]
  image: bitnami/kubectl:1.28
  script:
    - echo "更新k8s代码生成服务部署..."
    - kubectl set image deployment/pig-codegen pig-codegen=$HARBOR_URL/$HARBOR_PROJECT/pig-codegen:${CI_COMMIT_SHORT_SHA} -n $K8S_NAMESPACE
    - kubectl rollout status deployment/pig-codegen -n $K8S_NAMESPACE

# 阶段3-5：定时任务服务（pig-quartz）的构建、推送、部署
build-quartz-image:
  stage: build-image
  tags: [pig, k8s, 172]
  image: docker:20.10
  dependencies:
    - build
  script:
    - echo "构建定时任务服务镜像..."
    - cd pig-visual/pig-quartz
    - docker build -t $HARBOR_URL/$HARBOR_PROJECT/pig-quartz:${CI_COMMIT_SHORT_SHA} .

push-quartz-image:
  stage: push-image
  tags: [pig, k8s, 172]
  image: docker:20.10
  script:
    - docker login -u admin -p $HARBOR_PASSWORD $HARBOR_URL
    - echo "推送定时任务服务镜像..."
    - docker push $HARBOR_URL/$HARBOR_PROJECT/pig-quartz:${CI_COMMIT_SHORT_SHA}

deploy-quartz:
  stage: deploy
  tags: [pig, k8s, 172]
  image: bitnami/kubectl:1.28
  script:
    - echo "更新k8s定时任务服务部署..."
    - kubectl set image deployment/pig-quartz pig-quartz=$HARBOR_URL/$HARBOR_PROJECT/pig-quartz:${CI_COMMIT_SHORT_SHA} -n $K8S_NAMESPACE
    - kubectl rollout status deployment/pig-quartz -n $K8S_NAMESPACE
```

### 十七、配置Gitlab保密变量

避免密码明文暴露的同时，也方便引用，如上面的$HARBOR_PASSWORD

1. 进入项目->Settings->CI/CD->Variables->Add variable
2. 添加变量：
   - 名称：HARBOR_PASSWORD
   - value：HARBOR12345
   - 勾选 "Mask variable"

### 十八、测试CI/CD流水线

1. 将配置文件提交到GitLab仓库

   ```bash
   # 在本地项目目录执行
   git add .gitlab-ci.yml
   git commit -m "add ci/cd pipeline (gitlab on 172)"
   git push origin main
   ```

2. 查看流水线执行状态





