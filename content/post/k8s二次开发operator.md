+++
date = '2025-09-28T18:00:00+08:00'
draft = 'true'
title = 'k8s二次开发operator'
+++

#### 名词概念

1. client-go: kubernetes官方提供的go语言的客户端库，go应用使用该库可以访问kubernetes的APIServer，这样我们就可以通过编程对kubernetes资源进行增删改查操作
2. CRD：Custom Resource Definition，自定义资源定义，允许用户自定义新的资源类型，像内置资源一样使用和管理，用于扩展kubernetes API，实现自定义对象的声明式管理
3. Operator: 基于CRD实现，通过编程方式管理应用的生命周期，监听CRD对象的变化，自动执行相应操作，通过控制器保证应用处于预期状态
4. Operator SDK：开发工具包，帮助开发者快速构建kubernetes operator，基于controller-runtime库，是对cilent-go的再封装,写控制器更容易
5. Etcd-Operator：通过k8s API观察etcd集群的当前状态，分析当前状态与期望状态的差别，调用etcd集群管理API或k8s API消除这些差别

#### 环境配置:go,operator-sdk,wsl+dockerdesktop+k8s

![img](https://iqeubg8au73.feishu.cn/space/api/box/stream/download/asynccode/?code=MzhiM2E5MWFkOGJkOTU2OWI5NDNiZmQzMTMyMDUwNzZfNmxhNkRrWkRQS1EzRUhNVkJRek1HUzU5a0lYMlZYUFRfVG9rZW46QVVDNWJKREVLb3k0UjJ4VGRrcmNJaWN1bkJlXzE3NTkwNTIzNjM6MTc1OTA1NTk2M19WNA)

```Bash
#安装 Go
wget https://dl.google.com/go/go1.21.6.linux-amd64.tar.gz
sudo tar -C /usr/local -xzf go1.21.6.linux-amd64.tar.gz
vim ~/.bashrc

#在文件末尾添加以下内容（复制粘贴，避免拼写错误）export GOROOT=/usr/local/go
export GOROOT=/usr/local/go
export GOPATH=$HOME/go
export PATH=$PATH:$GOROOT/bin:$GOPATH/bin

source ~/.bashrc
#克隆仓库
git clone https://github.com/operator-framework/operator-sdk.git
cd operator-sdk

#检出指定版本
git checkout v1.32.0
#临时禁用当前用户的Git证书验证（仅当前终端生效）
export GIT_SSL_NO_VERIFY=1
go mod tidy
#编译并安装
make install
```

## 开发operator

1. 初始化项目,生成基础的项目结构 go.mod, main.go, Makefile 等

```Bash
mkdir my-operator
cd my-opereator
operator-sdk init --domain example.com --repo github.com/Logic0528/operator-1
```

2. 创建API和Controller

```Bash
operator-sdk create api --group cache --version v1 --kind Memcached
```

结果会多两个文件：

### `api/v1/memcached_types.go` → CRD 定义

1. MemcachedSpec----描述用户想要什么。CRD的spec部分，用户期望的状态，这里默认生成了一个Foo示例字段

```Bash
type MemcachedSpec struct {
        // INSERT ADDITIONAL SPEC FIELDS - desired state of cluster
        // Important: Run "make" to regenerate code after modifying this file

        // Foo is an example field of Memcached. Edit memcached_types.go to remove/update
        Foo string `json:"foo,omitempty"`
}
```

比如我们要做一个Memcached Operator可以写成：

```Bash
type MemcachedSpec struct {
    Size int32 `json:"size"`
}
```

表示用户在CR中可以写：

```Bash
spec:
  size: 3
```

2. MemcachedStatus----描述系统实际情况

```Bash
type MemcachedStatus struct {
        // INSERT ADDITIONAL STATUS FIELD - define observed state of cluster
        // Important: Run "make" to regenerate code after modifying this file
}
```

Status对应CRD的status部分，实际的运行状态，Operator控制器会更新这里，比如：

```Bash
type MemcachedStatus struct {
    Nodes []string `json:"nodes"`
}
```

那么用户能看到：

```Bash
status:
  nodes:
    - pod1
    - pod2
    - pod3
```

3. Memcached----定义单个CR

```Bash
type Memcached struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    Spec   MemcachedSpec   `json:"spec,omitempty"`
    Status MemcachedStatus `json:"status,omitempty"`
}
```

这是定义单个对象的完整结构

- TypeMeta:K8s资源的元数据，apiversion,kind等
- ObjectMeta:对象元数据，名字namespace，labels等
- Spec:期望状态
- Status：实际状态

翻译成yaml就是：

```Bash
apiVersion: cache.example.com/v1
kind: Memcached
metadata:
  name: example-memcached
spec:
  size: 3
status:
  nodes:
    - pod1
    - pod2
    - pod3
```

4. MemcachedList----定义对象列表

```Bash
type MemcachedList struct {
    metav1.TypeMeta `json:",inline"`
    metav1.ListMeta `json:"metadata,omitempty"`
    Items           []Memcached `json:"items"`
}
```

这是kubectl get memcached 返回的列表结构，就像kubectl get pods返回多个pod

5. init()

```Bash
func init() {
    SchemeBuilder.Register(&Memcached{}, &MemcachedList{})
}
```

这是把Memcached和MemcachedList注册到k8s API里，让Operator认识这个新的资源

### `controllers/memcached_controller.go` → 控制器逻辑

1. 导入&Reconciler定义

```Bash
type MemcachedReconciler struct {
        client.Client
        Scheme *runtime.Scheme
}
```

这是我们的控制器对象：

- client.Client可以调用Get,List,Create,Update,Delete去操作Pod/Deployment/Service等
- Scheme用来告诉Controller认识哪些资源类型，比如CRD、Deployment、Service

1. RBAC权限声明

```Bash
//+kubebuilder:rbac:groups=cache.example.com,resources=memcacheds,verbs=get;list;watch;create;update;patch;delete
//+kubebuilder:rbac:groups=cache.example.com,resources=memcacheds/status,verbs=get;update;patch
//+kubebuilder:rbac:groups=cache.example.com,resources=memcacheds/finalizers,verbs=update
```

这些注解会在 make manifests时生成RBAC权限，告诉kubernetes这个Operator有权进行哪些操作

2. **核心逻辑Reconcile**

```Bash
func (r *MemcachedReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    _ = log.FromContext(ctx)

    // TODO(user): your logic here

    return ctrl.Result{}, nil
}
```

这是**最关键的函数**，每次用户操作CR时都会触发

- req ctrl.Request 告诉你是哪一个Memcached对象发生了变化
  - 里面有namespace和name

 当前是空实现，知识返回nil，什么也不会做

3. SetupWithManager

```Bash
func (r *MemcachedReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&cachev1.Memcached{}).
        Complete(r)
}
```

这段告诉Manager：

- 我要这个控制器监听Memcached对象的变化
- 只要Memcached发生变化，就调用Reconcile

**好了,你已经学会导数的概念了，现在来写一下这道高考导数压轴题吧！**

## 实现一个最小可运行的operator

功能：

- 用户在CR里写：spec.size: N
- Operator就创建/更新一个Deployment，Deployment里运行N个Memcached Pod

### memcached_controller.go

```Bash
package controllers

import (
    "context"
    "fmt"

    appsv1 "k8s.io/api/apps/v1"
    corev1 "k8s.io/api/core/v1"
    "k8s.io/apimachinery/pkg/api/errors"
    "k8s.io/apimachinery/pkg/api/resource"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/types"
    "k8s.io/apimachinery/pkg/util/intstr"

    "k8s.io/apimachinery/pkg/runtime"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/client"
    "sigs.k8s.io/controller-runtime/pkg/log"

    cachev1 "github.com/Logic0528/operator-1/api/v1"
)

type MemcachedReconciler struct {
    client.Client
    Scheme *runtime.Scheme
}

// RBAC 权限
//+kubebuilder:rbac:groups=cache.example.com,resources=memcacheds,verbs=get;list;watch;create;update;patch;delete
//+kubebuilder:rbac:groups=cache.example.com,resources=memcacheds/status,verbs=get;update;patch
//+kubebuilder:rbac:groups=cache.example.com,resources=memcacheds/finalizers,verbs=update
//+kubebuilder:rbac:groups=apps,resources=deployments,verbs=get;list;watch;create;update;patch;delete
//+kubebuilder:rbac:groups=core,resources=pods,verbs=get;list

func (r *MemcachedReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    logger := log.FromContext(ctx)

    // 1. 取出 CR 对象
    var memcached cachev1.Memcached
    if err := r.Get(ctx, req.NamespacedName, &memcached); err != nil {
        if errors.IsNotFound(err) {
            // 对象被删除了，不需要处理
            return ctrl.Result{}, nil
        }
        return ctrl.Result{}, err
    }

    // 2. 检查 Deployment 是否存在
    dep := &appsv1.Deployment{}
    depName := fmt.Sprintf("%s-deployment", memcached.Name)
    err := r.Get(ctx, types.NamespacedName{Name: depName, Namespace: memcached.Namespace}, dep)

    if errors.IsNotFound(err) {
        // 3. 不存在就创建
        dep = r.deploymentForMemcached(&memcached)
        logger.Info("Creating Deployment", "namespace", dep.Namespace, "name", dep.Name)
        if err := r.Create(ctx, dep); err != nil {
            return ctrl.Result{}, err
        }
    } else if err == nil {
        // 4. 存在就对比 replicas 是否一致
        size := int32(memcached.Spec.Size)
        if *dep.Spec.Replicas != size {
            dep.Spec.Replicas = &size
            logger.Info("Updating Deployment replicas", "namespace", dep.Namespace, "name", dep.Name, "replicas", size)
            if err := r.Update(ctx, dep); err != nil {
                return ctrl.Result{}, err
            }
        }
    } else {
        return ctrl.Result{}, err
    }

    return ctrl.Result{}, nil
}

// 根据 CR 生成 Deployment
func (r *MemcachedReconciler) deploymentForMemcached(m *cachev1.Memcached) *appsv1.Deployment {
    replicas := int32(m.Spec.Size)
    labels := map[string]string{"app": "memcached"}

    dep := &appsv1.Deployment{
        ObjectMeta: metav1.ObjectMeta{
            Name:      fmt.Sprintf("%s-deployment", m.Name),
            Namespace: m.Namespace,
        },
        Spec: appsv1.DeploymentSpec{
            Replicas: &replicas,
            Selector: &metav1.LabelSelector{
                MatchLabels: labels,
            },
            Template: corev1.PodTemplateSpec{
                ObjectMeta: metav1.ObjectMeta{Labels: labels},
                Spec: corev1.PodSpec{
                    Containers: []corev1.Container{{
                        Name:  "memcached",
                        Image: "memcached:1.6.6", // 固定一个简单镜像
                        Ports: []corev1.ContainerPort{{
                            ContainerPort: 11211,
                            Name:          "memcached",
                        }},
                        Resources: corev1.ResourceRequirements{
                            Requests: corev1.ResourceList{
                                corev1.ResourceCPU:    resource.MustParse("100m"),
                                corev1.ResourceMemory: resource.MustParse("128Mi"),
                            },
                        },
                        LivenessProbe: &corev1.Probe{
                            Handler: corev1.Handler{
                                TCPSocket: &corev1.TCPSocketAction{
                                    Port: intstr.FromInt(11211),
                                },
                            },
                            InitialDelaySeconds: 30,
                            PeriodSeconds:       10,
                        },
                    }},
                },
            },
        },
    }

    // 设置 OwnerReference，这样 CR 删除时 Deployment 也会删掉
    ctrl.SetControllerReference(m, dep, r.Scheme)
    return dep
}

func (r *MemcachedReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&cachev1.Memcached{}).
        Owns(&appsv1.Deployment{}). // 监听 Deployment 变化
        Complete(r)
}
```

下一步

1. 生成RBAC和CRD

```Bash
make manifests
```

1. 部署Operator

```Bash
make install
make run
```

1. 创建一个CR示例

```Bash
#cache_v1_memcached.yaml

apiVersion: cache.example.com/v1
kind: Memcached
metadata:
  name: example-memcached
spec:
  size: 3
kubectl apply -f config/samples/cache_v1_memcached.yaml
```

1. 查看效果

```Bash
kubectl get deployments
kubectl get pods
```

### 关键文件解析

- **my-operator/config/crd/bases/cache.example.com_memcacheds.yaml**

执行”make manifests“后生成的CRD定义文件，用来告诉kubernetes API server有一种新的资源类型，比如memcached。定义了这个资源的API Group，版本，Kind，Plural，Spec，Status等

- **config/samples/cache_v1_memcached.yaml**

CRD的一个实例，在集群里面真的创建的一个对象
