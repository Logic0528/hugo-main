+++
date = '2025-10-20T18:00:00+08:00'
draft = 'true'
title = 'Operator开发-2-定时扩缩容容器'
+++

引言：上一篇我们讲了operator开发的环境配置与初步框架搭建，介绍了一些名词概念和关键代码含义，这一篇简单实现一个定时扩缩容容器的operator，同时补充一些概念知识与代码库作用



### 一、外部库与函数

- ```go
  import (
  	"context"
  	"fmt"
  	"time"
  
  	v1 "k8s.io/api/apps/v1"
      ...
  	ctrl "sigs.k8s.io/controller-runtime"
  
  	apiv1alpha1 "github.com/Logic0528/operator-1/api/v1alpha1"
  )
  ```

  v1与ctrl是给两个常用的外部包起的别名

  - v1 "k8s.io/api/apps/v1"：这个包来自Kubernetes官方API库，定义Deployment，StatefulSet，DaemonSet,RepliacaSet这些资源对象

    ```go
    deploy := &v1.Deployment{}
    ```

  - ctrl "sigs.k8s.io/controller-runtime":这个包来自controller-runtime框架，是Operator开发的核心库之一。提供控制器运行框架，用于编写、注册、运行Operator Controller

  - apiv1alpha1 "github.com/Logic0528/operator-1/api/v1alpha1"：这是我的自定义GO包，存放我定义的CRD类型，一般路径如下：

    ```go
    your-operator/
    └─ api/
       └─ v1alpha1/
           ├─ scaler_types.go   <-- 这里定义了 Scaler struct
           └─ groupversion_info.go
    ```

    代码中apiv1alpha1.Scaler{}就是从我仓库里的api/v1alpha1包导入的Scaler类型

- ```go
  type ScalerReconciler struct {
  	client.Client
  	Scheme *runtime.Scheme
  }
  ```

  Kubernetes Operator 控制器结构体（Reconciler）定义，在Operator模型中，每种CRD都对应一个Reconciler

  - client.Client:封装了 Kubernetes API 的客户端，用来在 Reconcile 中读写资源对象。它可以让我们直接操作集群中的资源，就像**kubectl**一样。

    ```go
    r.Client.Get(...) //获取对象
    
    r.Client.Create(...)//创建对象
    ```

  - Scheme *runtime.Scheme:定义了各种API类型和他们对应的Go Struct类型之间的映射关系，没太搞懂

- ```go
  func (r *ScalerReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
  	_ = log.FromContext(ctx)
  ```

  控制器的核心循环函数 Reconcile Loop,它每次被触发时，会同步一次集群中某个对象的实际状态和期望状态

  - r *ScalerReconciler：接收者，定义的控制器实例，也就是上面的type ScalerReconciler struct，它由client和scheme字段，可以操作集群

  - ctx context.Context：上下文对象，用于控制超时，取消请求，携带日志

  - req ctrl.request：请求，当前要处理的对象

  - ctrl.Request,error：Reconcile的返回值，用于告诉控制器下一步要不要再调度

    ```go
    ctrl.Result{}, nil	//正常结束，不需要立即重新执行
    ctrl.Result{Requeue: true}, nil	//立即再次执行一次（适合状态还没就绪的情况）
    ctrl.Result{RequeueAfter: 10 * time.Second}, nil	//10 秒后再次执行一次
    ctrl.Result{}, err	//出错了，控制器会自动重试
    ```

### 二、定时扩缩容容器operator

下面逐行代码分析如何实现一个定时扩缩容容器的operator

- ```go
  scaler := &apiv1alpha1.Scaler{}
  ```

  这里创建了一个空的Scaler结构体实现，Scaler是自定义的CRD，&表示取地址，生成一个指针，用于client.Get填充数据

  

- ```go
  err := r.Get(ctx, req.NamespacedName, scaler)
  if err != nil {
  		//如果没有发现这个scaler实例，我们就用client.IgnoreNotFound(err)来忽略错误，让进程不中断
  		return ctrl.Result{}, client.IgnoreNotFound(err)
  	}
  ```

  r.Get(ctx,req.NamespaceName,scaler)用controller-runtime提供的Client.Get方法去K8s集群获取这个资源实例，**req.NamespaceName提供了namespace+name，告诉API Server要获取哪个具体对象**，获取成功后，scaler指针里就有完整的CR对象内容，包括spec，status等字段。

  如果报错，client.IgnoreNotFound(err)会忽略资源不存在的错误，保证Reconcile循环不被打断

- ```go
  startTime := scaler.Spec.Start
  endTime := scaler.Spec.End
  replicas := scaler.Spec.Replicas
  
  currentHour := time.Now().Local().Hour()
  log.Info(fmt.Sprintf("currentTime: %d", currentHour))
  ```

  从Scaler CR中读取配置，在/deploy-scaler/api/v1alpha1/scaler_types.go文件下

- ```go
  if currentHour >= startTime && currentHour <= endTime {
  		//从scaler实例遍历deployments
  		for _, deploy := range scaler.Spec.Deployments {
  			//创建一个新的deployment实例
  			deployment := &v1.Deployment{}
  			err := r.Get(ctx, types.NamespacedName{
  				Name:      deploy.Name,
  				Namespace: deploy.Namespace,
  			}, deployment)
  			if err != nil {
  				return ctrl.Result{}, err
  			}
  
  			//判断当前k8s集群中的deployment副本数是否等于scaler中指定的副本数
  			if deployment.Spec.Replicas != &replicas {
  				deployment.Spec.Replicas = &replicas
  				err := r.Update(ctx, deployment)
  				if err != nil {
  					return ctrl.Result{}, err
  				}
  			}
  		}
  	}
  ```

  遍历scaler.Spec.Deployments,这是我们想管理的Deployment列表，r.Get获取每个Deployment对象，判断Deployment的Spec.Replicas是否等于你期望的replicas，如果不相等，就修改Deployment并调用r.update更新到Kubernetes

完整Reconcile函数代码如下

```go
func (r *ScalerReconciler) Reconcile(ctx cont-ext.Context, req ctrl.Request) (ctrl.Result, error) {
	_ = log.FromContext(ctx)

	// TODO(user): your logic here
	log := logger.WithValues("Request.Namespace", req.Namespace, "Request.Name", req.Name)
	log.Info("Reconcile called")

	//创建一个scaler实例
	scaler := &apiv1alpha1.Scaler{}
	err := r.Get(ctx, req.NamespacedName, scaler)
	if err != nil {
		//如果没有发现这个scaler实例，我们就用client.IgnoreNotFound(err)来忽略错误，让进程不中断
		return ctrl.Result{}, client.IgnoreNotFound(err)
	}

	startTime := scaler.Spec.Start
	endTime := scaler.Spec.End
	replicas := scaler.Spec.Replicas

	currentHour := time.Now().Local().Hour()
	log.Info(fmt.Sprintf("currentTime: %d", currentHour))

	//从startTime开始endTime为止
	if currentHour >= startTime && currentHour <= endTime {
		//从scaler实例遍历deployments
		for _, deploy := range scaler.Spec.Deployments {
			//创建一个新的deployment实例
			deployment := &v1.Deployment{}
			err := r.Get(ctx, types.NamespacedName{
				Name:      deploy.Name,
				Namespace: deploy.Namespace,
			}, deployment)
			if err != nil {
				return ctrl.Result{}, err
			}

			//判断当前k8s集群中的deployment副本数是否等于scaler中指定的副本数
			if deployment.Spec.Replicas != &replicas {
				deployment.Spec.Replicas = &replicas
				err := r.Update(ctx, deployment)
				if err != nil {
					return ctrl.Result{}, err
				}
			}
		}
	}
	return ctrl.Result{RequeueAfter: time.Duration(10 * time.Second)}, nil
}
```

补充：两个相似文件的关系与区别

```go
/deploy-scaler/config/crd/bases/api.example.com_scalers.yaml
/deploy-scaler/config/samples/api_v1alpha1_scaler.yaml
```

- /deploy-scaler/config/crd/bases/api.example.com_scalers.yaml：自定义资源定义，让Kubernetes认识Scaler这种新类型
- /deploy-scaler/config/samples/api_v1alpha1_scaler.yaml：自定义资源实例，用来创建一个具体的Scaler对象，触发控制器逻辑

### 三、部署并运行operator（本地运行）

1. 安装CRD，让Kubernets知道什么是Scaler

   ```go
   kubectl apply -f config/crd/bases/
   ```

   验证是否成功：

   ```go
   kubectl get crd | grep scaler
   ```

   ![image-20251020161129513](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\image-20251020161129513.png)

2. 在项目根目录执行

   ```go
   make run
   ```

   这会在本地启动 controller-runtime（`main.go`），连接到当前 kubeconfig 的集群,保持这个终端 **开着别关**，它是 Operator 的“脑子”。

3. 准备被控制的Deployments

   ```go
   kubectl create deployment abc --image=nginx
   kubectl create deployment def --image=nginx
   ```

   ![image-20251020161320372](https://raw.githubusercontent.com/Logic0528/MyImages/main/images/image-20251020161320372.png)

4. 创建Scaler自定义资源

   ```go
   kubectl apply -f config/samples/api_v1alpha1_scaler.yaml
   ```

   ![image-20251020161438669](https://raw.githubusercontent.com/Logic0528/MyImages/main/images/image-20251020161438669.png)

可以看到副本数成功扩容到了三，回到刚才打开的那个窗口看下日志

![image-20251020161743958](https://raw.githubusercontent.com/Logic0528/MyImages/main/images/image-20251020161743958.png)


成功！

