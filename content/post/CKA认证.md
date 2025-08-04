+++ date = '2025-07-29T10:28:19+08:00' draft = true title = 'CKA认证'+++
## CKA认证



### 一.PVC

- 注意：解题前需连接到对应的主机。

题目： mariadb namespace 中的MariaDB Deployment 被误删除。请恢复该deployment并确保数据持久性。

请按照以下步骤:

1. 如下规格在mariadb namespace 中创建名为 mariadb 的PVC：
   	- 访问模式为ReadWriteOnce(依题目)
   	- 存储为250Mi
2. 集群中现有一个PersistentVolume,您必须使用现有的PV。
3. 编辑位于~/mariadb-deployment.yaml 的 MariaDB Deployment 文件，以使用上一步中创建的PVC。
4. 将更新的 Deploment 文件应用到集群
5. 确保MariaDB Deployment正在运行且稳定

#### 搭建模拟环境

正式考试中不需要搭建模拟环境，只需按照下面的解题步骤解题即可。在自己电脑上可以搭建个模拟环境模拟考试，同时起到练习的作用

1.创建本地供应商

​	动态制备 PV ,我们需要一个provisioner,一个 storageclass,在local-path-storage命名空间下。

​	**Provisioner** 是一个核心组件，负责动态创建和管理持久化存储，Provisioner允许在Pod请求存储时**自动创建**所需的存储卷，无需管理员预先创建，这依赖于StorageClass资源的定义

```bash
kubectl create ns local-path-storage
```

local-path-provisioner.yaml

```bash

```

local-path-storage.yaml

```bash
	
```
2.创建namespace

```bash
kubectl create ns mariadb
```

3.mariadb-pv.yaml

```bash
apiVersion: v1
kind: PersistentVolume
metadata:
	name: mariadb-pv
spec:
	capacity:
		storage:250Mi
    accessModes:
    	- ReadWriteOnce
    persistentVolumeReclaimPolicy: Retain
    storageClasName: local-path
    hostPath:
    	path: /mnt/data/mariadb
```

```bash
kubectl apply -f mariadb-pv.yaml
kubectl get pv
```

4.mariadb-deployment.yaml

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
	name: mariadb
	namespace: mariadb
spec:
	replicas: 1
	selector:
		matchLabels:
		app: mariadb
    template:
    	metadata:
    		labels:
    			app: mariadb
        spec:
        	containers:
        	- name: mariadb
        	  image: mariadb:10.5
        	  imagePullPolicy: IfNotPresent
        	  ports:
        	  - containerPort: 3306
        	  env:
        	  - name: MYSQL_ROOT_PASSWORD
        	  	value: "rootpassword"
```

#### 解题

解题即是在考试环境下需要做的操作

1. 切换到指定节点

2. 检查pv,以及pv用到的storageclass
   ```bash
   kubectl get pv
   kubectl get sc
   ```

3. 创建pvc
   ```bash
   apiVersion: v1
   kind: PersistentVolumeClaim
   metadata:
   	name: mariadb   ##按题目要求修改
   	namespace: mariadb   ##按题目要求修改
   spec:
   	storageClassName: local-path   ##第二步查到的sc
   	accessModes:
   		- ReadWriteOnce   ##按题目要求修改
       resources:
       	requests:
       		storage: 250Mi   ##按题目要求修改
   ```

   ```bash
   kubectl apply -f pvc.yaml
   kubectl get pvc -n mariadb
   ```

4. 配置Deployment关联PVC
   ```bash
   vim ~/mariadb-deployment.yaml
   ```

   增加**volumes**和**volumeMounts**部分

   ```bash
   volumes：
   	- name:mariadb-data
   	persistentVolumeClaim:
   		claimName: "mariadb"
   volumesMounts:
   	- name: mariadb-data
   	mountPath: /var/lib/mysql   ##按题目要求修改
   ```

   完整的deployment.yaml文件如下：

   ```bash
   apiVersion: apps/v1 
   kind: Deployment 
   metadata: 
     name: mariadb 
     namespace: mariadb   # 指定 namespace 
   spec: 
     replicas: 1 
     selector: 
       matchLabels: 
         app: mariadb 
     template: 
       metadata: 
         labels: 
           app: mariadb 
       spec: 
         containers: 
         - name: mariadb 
           image: mariadb:10.5   # 以考试为准，无需自己指定 
           imagePullPolicy: IfNotPresent 
           ports: 
           - containerPort: 3306 
           env: 
           - name: MYSQL_ROOT_PASSWORD 
             value: "rootpassword"       # 可按题目或需求设置
             
           volumeMounts: 
           - name: mariadb-da 
             mountPath: /var/lib/mysql   # 数据挂载路径 
           volumes:
           - name: mariadb-data 
             persistentVolumeClaim: 
               claimName: "mariadb" 
   ```
   
5. 检查Deployment和Pod运行状态
   ```bash
   kubectl apply -f ~/mariadb-deployment.yaml
   kubectl get deployment -n mariadb
   ```

   Pod状态应为Running

6. exit退出当前节点

### 二.四层代理Service

题目：

1. 重新配置 spline namespace 中现有的 front-end Deployment, 以公开现有容器nginx的端口80/TCP。
2. 创建一个名为 front-end-svc 的新Service，以公开容器端口 80/TCP
3. 配置新的Service，以通过NodePort公开各个Pod

#### 搭建模拟环境

1. 创建namespace
   ```bash
   kubectl create ns spline
   ```

2. front-end Deployment
   ```bash
   vim 2-svc-deployment.yaml
   
   apiVersion: v1
   kind: Deployment
   metadata:
   	name：front-end
   	namespace: spline
   spec:
   	replicas: 1
   	selector:
   		matchLabels:
   			app: front-end
       template:
       	metadata:
       		labels:
       			app: front-end
           spec:
           	containers:
           		- name: nginx
           		  image: nginx:1.25
           		  imagePullPolicy: IfNotPresent
   ```

3. 
   ```bash
   kubectl apply -f 2-svc-deploy.yaml
   ```

#### 解题

1. 连接指定节点

2. 修改 front-end deployment 配置，添加端口配置

     在name: nginx下方新增端口配置

      ```bash
      kubectl -n spline edit deployment front-end
      
      ports:
      - containerPort: 80
        protocol: TCP
      ```

      注意：

      - 正确缩进，确保ports在name: nginx同层级下
      - 执行 kubectl -n spline edit deployment front-end 之后若提示如下，说明修改成功:deploymnet.apps/front-end edited

3. 暴露 Deployment，创建 NodePort 类型的Service，执行命令：
   ```bash
   kubectl -n spline expose deployment front-end --type=NodePort -port=80 --target-port=80 --name=front-end-svc
   ```

   看到service/front-end-svc exposed,表示创建service成功
   **参数说明**：

   - --type=NodePort : 依题目要求
   - --port=80 ：Service监听端口
   - --target-port : 容器内应用端口
   - --name=front-end-svc : Service名称

4. 查看Service
   ```bash
   kubectl -n spline get svc front-end-svc -o wide
   
   #测试访问，用查到的cluster-ip
   curl 10.96.101.208:80
   ```

5. exit退出当前节点

### 三.七层代理资源Ingress

题目：创建新的 Ingress 资源，具体如下：
名称： echo-hello
Namespace:  sound

使用 Service 端口 8080 在 http://example.org/echo-hello 上公开 echoserver-service 的 Service

可以使用以下命令检查 echoserver-service Service 的可用性，该命令应返回 hello world

#### 搭建模拟环境

1. 安装Ingress-nginx-controller
   ```bash
   #ingress-deploy.yaml
   
   
   
   kubectl apply -f ingress-deploy.yaml
   ```

2. 创建命名空间
   ```bash
   kubectl create ns sound
   ```

3. 创建pod

   ```bash
   vim 3-ingress-deploy-pod.yaml
   
   apiVersion: v1
   kind: Deployment
   metadata:
   	name: echoserver
   	namespace: sound
   spec: 
   	repliacs: 1
   	selector: 
   		matchLabels:
   			app: echoserver
       template:
       	metadata:
       		labels:
       			app: echoserver
           spec:
           	containers:
           	- name: echoserver
           	  image: docker.io/library/tomcat:8.5.34-jre8-alpine
           	  imagePullPolicy: IfNotPresent
           	  ports:
           	  - containerPort:8080
   ```

   ```bash
   kubectl apply -f 3-ingress-deploy-pod.yaml
   ```

4. 创建service
   ```bash
   vim 3-ingress-nsvc.yaml
   ```

   ```bash
   apiVersion: v1
   kind: Service
   metadata:
   	name: echoserver-service
   	namespace: sound
   spec:
   	selector:
   		app: echoserver
       ports:
       	- protocol: TCP
       	  port: 8080
       	  targetPort: 8080
   ```

   ```bash
   kubectl apply -f 3-ingress-svc.yaml
   kubectl get svc -n sound
   
   curl ip:8080
   #返回网页内容，说明可以通过svc代理pod
   ```

#### 解题

1. 连接到指定节点

2. 查询 IngressClass ,确认 class 名字
   ```bash
   kubectl get ingressclasses.networking.k8s.io
   #kubectl get ingclass  # 等效于原命令，更简洁
   
   #查询结果如下
   NAME      CONTROLLER
   nginx     k8s.io/ingress-nginx
   ```

3. 编写 ingress.yaml
   ```bash
   vim ingress.yaml
   ```

   ```bash
   apiVersion: v1
   kind: Ingress
   metadata:
     name: echo-hello
     namespace: sound  
     annotations:
       nginx.ingress.kubernetes.io/rewrite-target: /  # 将请求路径重写为根路径
   spec:
     ingressClassName: nginx  # 指定使用nginx类型的Ingress控制器
     rules:
     - host: example.org  # 匹配的域名
       http:  # HTTP路由规则
         paths:
         - path: /echo-hello  # 匹配的路径
           pathType: Prefix  # 路径匹配类型为前缀匹配
           backend:
             service:
               name: echoserver-service  # 转发到的服务名称
               port:
                 number: 8080  # 转发到的服务端口
   ```
   
   
   
4. 创建 ingress 资源
   ```bash
   kubectl apply -f ingress.yaml
   kubectl get ingress -n sound
   
   #验证Ingress配置
   curl http://example.org/echo-hello
   
   exit
   ```

### 四.网络策略NetworkPolicy

题目：从提供的YAML文件中选择并应用适当的NetworkPolicy。确保所选的NetworkPolicy不会过于宽松，同时允许运行在fronted和backend namespace 中的 fronted 和 backend Deployment之间的通信。

首先，分析fronted 和 backend Deployment,了解其通信需求，以便确定需要应用的NetworkPolicy.

接着，检查位于 ~/netpol 文件夹中的 NetworkPolicy YAML实例。

最后，应用一个NetworkPolicy,以启用 frontend 和 backend Deployment 之间的通信，但不要使其过于宽松。

#### 搭建模拟环境

1. 创建命名空间
   ```bash
   kubectl create ns frontend
   kubectl create ns backend
   ```

2. 在 fronted 和 backend 下部署pod
   ```bash
   vim 4-fronted.aml
   ```

   ```bash
   apiVersion: apps/v1
   kind: Deployment
   metadata:
   	name: fronted-app
   	namespace: frontend
   spec:
   	replicas: 1
   	selector:
   		matchLabels:
   			app: frontend
       template:
       	metadata:
       		labels:
       			app: frontend
           spec:
           	containers:
           	- name: nginx
           	  image: nginx:1.25
           	  imagePullPolicy: IfNotPresent
           	  ports:
           	  - containerPort:80
   ```

   ```bash
   kubectl apply -f 4-frontend.yaml
   ```

   ```bash
   vim 4-backend.yaml
   ```

   ```bash
   apiVersion: apps/v1
   kind: Deployment
   metadata:
   	name: backend-app
   	namespace: backend
   spec:
   	replicas: 1
   	selector:
   		matchLabels:
   			app: backend
           template:
           	metadata:
           		labels:
           			app: backend
               spec: 
               	containers:
               	- name: nginx
               	  image: nginx:1.25
               	  imagePullPolicy: IfNotPresent
               	  ports:
               	  - containerPort: 80
   ```

   ```bash
   kubectl apply -f 4-backend.yaml
   ```

3. 创建backend默认拒绝策略
   ```bash
   vim networkpolicy.yaml
   ```

   ```bash
   apiVersion: networking.k8s.io/v1
   kind： NetworkPolicy
   metadata:
   	name: default-deny-all
   spec:
   	podSelector: {}
   	policyTypes:
   	 - Ingress
   ```

#### 解题

1. 连接到指定节点

2. 检查fronted和backend两个namespace的标签
   ```bash
   kubectl get ns frontend backend --show-labls
   ```

3. 检查frontend和backend两个namespace下所有pod的标签
   ```bash
   kubectl -n frontend get pod --show-labels
   kubectl -n backend get pod --show-labels
   ```

4. 查看默认拒绝策略
   检查backend namespace中是否有默认拒绝的Network Policy

   ```bash
   kubectl get networkpolicy -n backend
   ```

5. 查看题目给定的NetworkPolicy示例，并选择最合适的策略
   ```bash
   cat ~/netpol/netpol1.yaml
   cat ~/netpol/netpol2.yaml
   cat ~/netpol/netpol3.yaml
   
   ## 根据题目要求选择合适的应用
   kubectl apply -f ***
   ```

### 五.配置管理中心configmap

题目：

名为nginx-static-test的NGINX Deployment 正在nginx-static-test namespace 中运行，它通过名为 nginx-static-config 的ConfigMap进行配置。

更新nginx-static-config 这个configmap 以仅允许 TLSv1.3连接

注意：您可以根据需要重新创建，启动或扩展资源

您可以使用以下命令测试更改：
```bash
curl -k --tls-max 1.2 https://web.k8snginx.local
```

#### 搭建模拟环境

1. 创建命名空间
   ```bash
   kubectl create ns nginx-static-test
   ```

2. 资源准备：ConfigMap
   ```bash
   apiVersion: v1
   kind: ConfigMap
   metadata:
   	name: nginx-static-config
   	namespace: nginx-static-test
   data:
   	nginx.conf:{
   		worker_processes 1;
   		
   		envnts{
   			worker_connectionss 1024;
   		}
   		
   		http{
   		# SSL配置加在这里，http块内，server块外
   			ssl_protocols TLSv1.2;
   			ssl_ciphers 'ECHDE-RSA-***'
   			ssl_perfer_srever_ciphers on;
   		}
   		
   		server{
   			listen 80;
   			server_name web.k8snginx.local;
   			
   			localtion/{
   				root /usr/share/nginx/html;
   				index index.html index.htm;
   			}
   		}
   }
   
   kubectl apply -f nginx-static-config.yaml
   ```
   
   资源准备：Deployment
   ```bash
   apiVersion: apps/v1
   kind: Deployment
   metadata:
   	name: nginx-static-test
   	namespace: nginx-static-test
   spec:
   	replicas: 1
   	selector:
   		matchlabels:
   			app: nginx-static-test
       template:
       	metadata:
       		labels:
       			app: nginx-static-test
           spec:
           	containers:
           	- name: nginx
           	  image: nginx:1.25
           	  volumeMounts:
           	  - name: nginx-config
           	    mountPath: /etc/nginx/nginx.conf
           	    subPath: nginx.conf
                 volumes:
                 - name: nginx-config
                   configMap:
                   	name: nginx-static-config
   ```
   
   ```bash
   kubectl apply -f nginx-static-deployment.yaml
   
   kubectl get pods -n nginx-static-test
   ```

#### 解题

1. 连接到指定节点

2. 导出现有的nginx-static-config ConfigMap
   ```bash
   kubectl -n nginx-static-test get configmap nginx-static-config -o yaml > nginx-static-config.yaml
   ```

3. 修改ConfigMap的ssl_protocols配置
   ```bash
   vim nginx-static-config.yaml
   
   data:
   	nginx.conf:
   		#其他配置...
   		ssl_protocols TLSv1.3;  # 按题目要求修改
   ```

4. 先删除旧的ConfigMap,再创建新的configmap
   ```bash
   kubectl delete configmap nginx-static-test -n nginx-static-test
   
   kubectl apply -f nginx-static-test -n nginx-static-test
   ```

5. 重启nginx-static-test Deployment
   ```bash
   kubectl -n nginx-static-test rollout restart deployment nginx-static-test
   
   kubectl get pods -n nginx-static-test
   ```

6. 测试TLSv1.2连接，按照目前修改，访问是失败的说明配置正确

   ```bash
   curl -k --tls-max 1.2 https://web.k8snginx.local
   
   exit
   ```

### 六.存储类storageclass

题目：

首先，为名为rancher.io/local-path的现有制备器，创建一个名为test-local-path的新Storageclass

将卷绑定模式设置为 WaitForFirstConsumer

接下来，将test-local-path Storageclass设置为默认的Storageclass

请勿修改任何现有的Deployment和PVC

#### 解题

1. 连接至指定节点

2. 创建正确的Storageclass Yaml文件
   ```bash
   vim sc.yaml
   
   apiVeison: storage.k8s.io/v1
   kind: Storageclass
   metadata:
   	name: test-local-path
   	annotations:
   		storageclass.kubernetes.io/is-default-class:"true"
   provisioner: rancher.io/local-path  # 制备器的名字
   volumeBindingMode: WaitForFirstConsumer  # 卷绑定模式依题目要求 
   
    kubectl apply -f sc.yaml 
   ```
   
3. 验证test-local-path是否是默认的Storageclass
   ```bash
   kubectl get storageclass
   
   # 输出如下
   NAME　                PROVISIONER            DEFAULT
   test-local-path(default)  rancher.io/local-path  true
   
   # 提示是true说明是对的
   ```

4. exit退出当前节点

### 七.HPA（Pod水平自动扩缩容）

题目：

在 autoscale namespace 中创建一个名为 nginx-server 的新 HorizontalPodAutoscaler(HPA),此HPA必须定位到 autoscale namespace 中名为 nginx-server的现有Deployment

将HPA设置为每个Pod的CPU使用率旨在50%。将其配置为至少有一个Pod，且不超过4个Pod。此外，将缩小稳定窗口设置为30秒

#### 搭建模拟环境



#### 解题

1. 连接至指定节点

2. 创建hpa
   ```bash
   kubectl autoscale deployment nginx-server --cpu-percent=50 --min=1 --max=4 -n autoscale
   ```

3. 修改HPA的稳定窗口为30s
   ```bash
   kubectl edit horizontalpodautoscaler nginx-server -n autoscale
   
   spec:
   	maxReplicas: 4
   	minReplicas: 1
   	behavior:  # 新增字段
   		scaleDown:
   			stabilizationWindowsSeconds: 30
   ```

4. exit退出当前节点

### 八.Pod优先级PriorityClass

题目：

为用户工作负载创建一个名为 high-priority 的新 PriorityClass,其值比用户定义的现有最高优先级类值小一。修改在priority namespace 中运行的现有 busybox-logger Deployment，以使用 high-priority

确保 busybox-logger Deployment 在设置了新优先级类后成功部署

#### 搭建模拟环境

1. 创建namespace
   ```bash
   kubectl create ns priority
   ```

2. 创建PriorityClass资源
   ```bash
   vim priority-init.yaml
   
   apiVersion: scheduling.k8s.io/v1
   kind: PriorityClass
   metadata:
   	name: max-priority
   	value: 1000000000
   	globalDefault: false
   	
   kubectl apply -f priority-init.yaml
   ```

3. 创建busybox-logger这个deployment资源
   ```bash
   vim busybox-logger.yaml
   
   apiVersion: apps/v1
   kind: Deployment
   metadata:
   	name: busybox-logger
   	namespace: priority
   spec:
   	replicas: 1
   	selector:
   		matchLabels:
   			app: busybox-logger
       template:
       	metadata:
       		labels:
       			app: busybox-logger
           spec:
           	containers:
           	- name: busybox
           	  image: busybox:1.28
           	  imagePollPolicy: IfNotPresent
           	  command: ["/bin/sh","-c","while true;do echo 'Logging...';sleep 5;done"]
           	  
   kubectl apply -f busybox-logger.yaml
   kubectl get pods -n priority
   ```

#### 解题

1. 连接到指定节点

2. 查看当前集群存在的PriorityClass
   ```bash
   kubectl get priorityclass
   ```

3. 创建新的PriorityClass
   ```bash
   vim priority.yaml
   
   apiVersion: scheduling.k8s.io/v1
   kind: PriorityClass
   metadata:
   	name: high-priority
   value: 999999999
   globalDefault: false
   
   kubectl apply -f priority.yaml
   ```

4. 验证PriorityClass是否创建成功
   ```bash
   kubectl get priorityclass
   ```

5. 在线修改Deployment的priorityClassName
   ```bash
   kubectl edit deployment busybox-logger -n priority
   
   # 找到spec.template.spec部分，通常在dnsPolicy:ClusterFirst上方
   # 添加以下内容
   priorityClassName: high-priority
   ```

6. exit退出当前节点

### 九.resource配置 cpu 和 内存请求和限制

题目：

您管理一个 WordPress 应用程序。由于资源的请求过高，导致某些pod无法启动

relative-fawn namesapce 中的 WordPress 应用程序包含：
	具有 3 个副本的 WordPress Deployment

按照如下方式调整所有 pod 的资源请求
	- 将节点资源平均分配给这 3 个 Pod 
	- 为每个 Pod 分配公平的 CPU 和内存份额
	- 添加足够的开销以保持节点稳定

请确保，对容器和初始化容器使用完全相同的请求，您无需更改任何资源限制

在更新资源请求时，可以暂时将 WordPress Deployment 缩放为 0 个副本可能会有所帮助

更新后，请确认：

	- WordPress 保持 3 个副本
	- 所有 Pod 都在运行并准备就绪

#### 搭建模拟环境

1. 创建命名空间
   ```bash
   kubectl create namespace relative-fawn
   ```

2. 创建 WordPress Deployment
   ```bash
   vim wordpress-deployment.yaml
   
   apiVersion: apps/v1
   kind: Deployment
   metadata:
   	name: wordpress
   	namespace: relative-fawn
   spec:
   	replicas: 3
   	selector:
   		matchLabels:
   			app: wordpress
       template:
       	metadata:
       		labels:
       			app: wordpress
           spec:
           	initContainers:
           	- name: init-mysql
           	  image: busybox:1.28
           	  imagePullPolicy: IfNotPresent
           	  command: ['sh','-c','sleep 6']
           	  resources:
           	  	limits:
           	  		cpu: 1000m
           	  		memory: 500Mi
               containers:
               - name: wordpress
                 imagePullPolicy: IfNotPresnet
                 ports:
                 - containerPort: 80
                 resources:
                 	limits:
                 		cpu: 1000m
                 		memory: 500Mi
                 		
   kubectl apply -f wordpress -deployment.yaml
   
   kubectl get pods -n relative-fawn
   ```

#### 解题

1. 连接到指定节点

2. 将WordPress Deployment 副本缩放为0
   ```bash
   # 必须先缩为 0 再改 requests,否则新 Pod 起不来
   kubectl -n relative scale deployment wordpress --replicas=0
   
   # 检查副本数
   kubectl -n  relative-fawn get deployment wordpress
   
   # 如果ready为0，说明缩放成功
   ```

3. 检查 Node 资源使用情况
   ```bash
   kubectl get nodes
   
   kubectl describe node node-name
   ```

4. 修改 Deployment 资源
   ```bash
   kubectl -n relative-fawn edit deployment wordpress
   
   # 将 2 个 container 的 requests 部分改为：
   resources:
   	requests:
   		cpu: 100m
   		memory: 200Mi
   ```

5. 将 wordpress 副本数恢复为 3
   ```bash
   kubectl -n relative-fawn scale deployment wordpress --replicas=3
   ```

6. 检查pod副本数
   ```bash
   kubectl -n relative-fawn get pod
   ```

7. exit退出当前节点

### 十.自定义资源CRD

题目：

验证已经部署到集群的 cert-manager 应用程序。

使用 kubectl 将 cert-manager 命名空间所有自定义的资源（CRD）的列表，保存到~/resources.yaml

使用 kubectl,提取定制资源 Certificate 的 subject 规范字段的文档，并将其保存到~/subject.yaml

#### 解题

1. 连接至指定节点

2. 检查 cert-manager 资源
   ```bash
   kubectl get pods -n cert-manager
   ```

3. 获取 cert-manager 的 CRD 资源列表
   ```bash
   kubectl get crd | grep cert-manager
   
   # 保存结果到文件
   kubectl get crd | grep cert-manager > ~resource.yaml
   ```

4. 获取 Certificate 的 subject 规范字段
   ```bash
   kubectl explain certificate.spec.subject
   
   # 保存结果到文件
   kubectl explain certificate.spec.subject > ~/subject.yaml
   ```

5. exit退出当前节点

### 十一.Gateway网关

题目：

将现有 Web 应用程序从 Ingress 迁移到 Gateway API ，您必须维护 HTTPS 访问权限。

注意：集群中安装了一个名为 nginx 的 Gatewayclass。

首先，创建一个名为 web-local-gatewayn的 Gateway，主机名为 gateway.web.k8s.local ,并保持现有名为 web 的 Ingress 资源的现有 TLS和侦听器配置。

接下来，创建一个名为 web-route 的 HTTPRoute, 主机名为gateway.web.k8s.local ,并保持现有名为 web 的 Ingress 资源的现有路由规则。

您可以使用以下命令测试 Gateway API 配置：

curl -Lk https://gateway.web.k8s.local:31443

最后，删除名为 web 的现有 Ingress 资源。

#### 搭建模拟环境

1. 创建 deployment 和 service
   ```bash
   apiVersion: apps/v1
   kind: Deployment
   metadata:
   	name: web
   spec:
   	replicas: 1
   	selector:
   		matchLabels:
   			app: web
       template:
       	metadata:
       		labels:
       			app: web
           spec:
           	containers:
           	- name: web
           	  image: nginx:1.25
           	  imagePullPolicy: IfNotPresent
           	  ports:
           	  - containerPort: 80
   ---
   apiVersion: v1
   kind: Service
   metadata:
   	name: web
   spec:
   	selector:
   		app: web
       ports:
       - port: 80
         targetPort: 80
         
   kubectl apply -f web-deploy.yaml
   kubectl get pods
   ```

2. 准备TLS Secret
   ```bash
   # 用openssl生成key和crt
   openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
   -subj "/CN=gateway.web.k8s.local"\
   -keyout tls.key -out tls.crt
   
   # 创建sercret
   kubectl create secret tls web-cert --cert=tls.crt --key=tls.key
   kubectl get secert web-cert -oyaml
   
   # 创建ingress,后面要迁移到gateway
   vim web-ingress.yaml
   
   apiVersion: networking.k8s.io/v1
   kind: Ingress
   metadata:
     name: web
   spec:
     ingressClassName: nginx
     tls:
     - hosts:
       - gateway.web.k8s.local
       secretName: web-cert
     rules:
   **- host: gateway.web.k8s.local**
       http:
         paths:
         - path: /
           pathType: Prefix
           backend:
             service:
               name: web
               port:
                 number: 80
                 
   kubectl apply -f web-ingress.yaml
   kubectl get ingress
   ```

#### 解题

1. 连接至指定节点

2. 检查现有的 ingress ,提取重要信息
   ```bash
   kubectl get ingress web -o yaml
   ```

3. 创建gateway
   ```bash
   vim gateway.yaml
   
   apiVersion: gateway.networking.k8s.io/v1
   kind: Gateway
   metadata:
   	name: web-local-gateway
   spec： 
   	gatewayClassName: nginx  # 依题目修改
   	listeners:
   	- name: https
   	protocol: HTTPS  # 看上面ingress是什么
   	port: 443
   	hostname: gateway.web.k8s.local
   	tls:
   		mode: Terminate
   		certificateRefs:
   		- name: web-cert  # 填写ingress里的tls secretname
   		
   kubectl apply -f gateway.yaml
   
   kubectl get gateway
   ```

4. 创建 httproute
   ```bash
   vim httproute.yaml
   
   apiVersion: gateway.networking.k8s.io/v1
   kind: HTTPRoute
   metadata:
   	name: web-route
   spec:
   	parentRefs:
   	- name: web-local-gateway
   	hostnames:
   	- "gateway.web.k8s.local"
   	rules:
   	- matches:
   		- path:
   			type: PathPrefix
   			value: /  # ingress里path路径
           backendRefs:
           - name: web  # ingress里service name
             port: 80   # ingress里service port
             
   kubectl apply -f httproute.yaml
   kubectl get httproute
   ```

5. 测试gateway api 配置
   ```bash
   curl -Lk https://gateway.web.k8s.local:31443
   ```

6. 删除web ingress
   ```bash
   kubectl delete ingress web
   ```

7. exit退出当前节点

### 十二.sidecar边车容器

题目：

为了将传统应用程序集成到K8s的日志架构（如kubectl logs）中，通常的做法是添加一个用于日志流式传输的sidecar容器。

更新现有的sync-leverager Deployment,将使用busybox:stable镜像，且名为sidecar的并置容器，添加到现有的pod,新的并置容器必须运行以下命令：
/bin/sh -c "tail -n+1 -f /var/log/sync-leverager.log"

使用挂载在 /var/log 的Volume，使日志文件sync-leverager.log可供并置容器使用

除了添加所需的卷挂载之外，请勿修改现有容器的规范

#### 搭建模拟环境

1. 创建deployment资源
   ```bash
   vim sidecar-deploy.yaml
   
   apiVersion: apps/v1
   kind: Deployment
   metadata:
   	name: sync-leverager
   	labels:
   		app: sync-leverager
   spec:
   	replicas: 1
   	selector:
   		matchLabels:
   			app: sync-leverager
       template:
       	metadata:
       		labels:
       			app: sync-leverager
           spec:
           	containers:
           	- name: sync-leverager
           	  image: nginx:1.25
           	  imagePullPolicy: IfNotPresent
           	  command: ["/bin/sh", "-c", "while true;do echo \"$(date) INFO log line\" >> /var/log/sync-leverager.log;sleep 5;done"]
           	  
   kubectl apply -f sidecar-deploy.yaml
   ```

#### 解题

1. 连接到指定节点

2. 导出yaml文件
   ```bash
   kubectl get deployment sync-leverager -o yaml > sidecar.yaml
   ```

3. 编辑yaml文件
   ```bash
   vim sidecar.yaml
   
   # 新增内容如下
   spec.template.spec:
   	volumes:
   	- name: varlog
   	  emptyDir: {}
       containers:
       	...
       - command:
       		...
         volumeMounts:
         - name: varlog
           mountPath: /var/log
       - name: sidecar
         image: busybox:stable
         imagePullPolicy: IfNotPresent
         args: ["/bin/sh","-c","tail -n+1 -f /var/log/sync-leverager.log"]
         volumeMounts:
         - name: varlog
           mountPath: /var/log
   ```

   ```bash
   kubectl apply -f sidecar.yaml
   ```

4. 检查deployment,pod,sidecar容器日志
   ```bash
   kubectl get deployment sync-leverager
   
   kubectl get pod | grep sync-leverager
   
   kubectl logs sync-leverager-** -c sidecar
   ```

5. exit退出当前节点

### 十三.calico网络插件

题目：

文档地址  
Flannel Manifest：
  https://github.com/flannel-io/flannel/releases/download/v0.26.1/kube-flannel.yml   

Calico Manifest：
  https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/tigera-operator.yaml   

Context：  集群的 CNI 未通过安全审核，已被移除。您必须安装一个可以实施网络策略的新 CNI。

Task:

安装并设置满足以下要求的容器网络接口（CNI）：

选择并安装以下CNI选项之一：

1. Flannel版本 0.26.1
2. Calico版本 3.27.0

选择的CNI必须：

1. 让Pod相互通信
2. 支持Network Policy实施
3. 从清单文件安装（请勿使用helm）

#### 解题

1. 连接到指定节点

2. 下载calico tigera-operator.yaml 文件并创建（Flannel不支持NetworkPolicy,只能选Calico）
   ```bash
   wget https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/tigera-operator.yaml
   
   kubectl create -f tigera-operater.yaml  # 必须用create，apply可能导致CRD冲突
   ```

3. 查看 PodCIDR
   ```bash
   kubectl cluster-info dump | grep -i cluster-cidr
   
   # 假设输出为： 
   "clusterCIDR": "10.244.0.0/16"
   # 记住这个地址：10.244.0.0/16 
   ```

4. 下载 Calico 自定义资源配置 custom-resources.yaml
   ```bash
   wget 
   https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/custom
   resources.yaml 
   # 备注：这个网址考试没给，可以在tigera-operator.yaml这个下载地址基础上记个单词custom-resources.yaml 即可。
   ```

5. 编辑 custom-resources.yaml 文件，修改 cidr 字段 
   ```bash
   vim custom-resources.yaml 
   
   spec: 
   	calicoNetwork: 
   		ipPools: 
   		- blockSize: 26 
   		  cidr: 10.244.0.0/16   # 按上步骤实际查询的pod网段写 
   		  encapsulation: VXLANCrossSubnet 
   	 	  natOutgoing: Enabled 
   	      nodeSelector: all()
   ```

6. 创建 Calico 自定义资源
   ```bash
   kubectl create -f custom-resources.yaml
   ```

7. 检查Calico组件运行状态
   ```bash
   kubectl -n calico-system get pod
   ```

8. exit退出当前节点

### 十四.argocd

### 十五.etcd修复

题目：

kubeadm 配置的集群已迁移到新机器，他需要更改配置才能成功运行。



修复在机器迁移过程中损坏的单节点集群

首先，确定损坏的集群组件，并调查导致其损坏的原因
接下来，修复所有损坏的集群组件的配置。
最后，确保集群运行正常，确保每个节点和 Pod 都处于Ready状态

注意： 

- 已停用的集群使用外部 etcd 服务器
- 确保重新启动所有必要的服务和组件，以使更改生效。否则可能导致分数降低。

#### 解题

1. 连接到指定节点

2. 修复kube-apiserver配置（etcd地址）
   ```bash
   # 编辑kube-apiserver的静态Pod清单文件
   vim /etc/kubernetes/manifests/kube-apiserver.yaml
   
   # 找到--etcd-server参数
   # 改为：
   --etcd-server=https://127.0.0.1:2379  #指向本地etcd服务
   ```

3. 重启kubelet服务
   ```bash
   systemctl daemon-reload
   systemctl restart kubelet
   ```

4. 检查节点和Pod状态（此时可能还是有问题）
   ```bash
   kubectl get nodes
   
   kubectl get pod -n kube-system
   ```

5. 修复 kube-scheduler-master01 配置
   ```bash
   vim /etc/kubernetes/manifests/kube-scheduler.yaml
   
   # 找到resources 配置
   resources:
   	requests:
   	cpu: 100m
   ```

6. 等待kubelet自动重启kube-scheduler,一两分钟后再次检查集群状态
   ```bash
   kubectl get nodes
   
   kubectl get pod -n kube-system
   ```

7. exit退出当前节点

### 十六.cri-dockerd容器运行时

题目：

Context：您的任务是为 Kubernetes 准备一个 Linux 系统。 Docker 已被安装，但您需要为 kubeadm  配置它。 

完成以下任务，为 Kubernetes 准备系统：   

设置 cri-dockerd ：  

1.安装 Debian 软件包 ~/cri-dockerd_0.3.6.3-0.ubuntu-jammy_amd64.deb Debian 软件包 使用 dpkg 安装。  

2.启用并启动 cri-docker 服务   

配置以下系统参数:  

net.bridge.bridge-nf-call-iptables 设 置 为 1  
net.ipv6.conf.all.forwarding 设 置 为 1  
net.ipv4.ip_forward 设 置 为 1  
net.netfilter.nf_conntrack_max 设置为 131072   

确保这些系统参数在系统重启后仍然存在，并应用于正在运行的系统。

#### 解题

1. 连接到指定节点

2. 安装cri-dockerd并启动
   ```bash
   sudo dpkg -i ~/cri-dockerd_0.3.6.3-0ubuntu-jammy_amd64.deb
   
   sudo systemctl enable cri-docker
   sudo systemctl start cri-docker
   ```

3. 配置系统参数
   ```bash
   sudo modprobe br_netfilter
   
   sudo vim /etc/sysctl.conf
   
   #添加以下内容到文件最后
   net.bridge.bridge-nf-call-iptables = 1 
   net.ipv6.conf.all.forwarding = 1 
   net.ipv4.ip_forward = 1 
   net.netfilter.nf_conntrack_max = 131072 
   ```

4. 让配置生效
   ```bash
   sudo sysctl -p
   ```

5. exit退出当前节点


