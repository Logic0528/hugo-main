+++
date = '2025-06-30T14:59:31+08:00'
draft = true
title = 'K8s集群初始化'
categories = ['k8s']
tags = ['k8s']
+++

自己动手，丰衣足食，经过多次的搭建k8s集群现在可以说是基本明白了，记录一下过程不用每次都再搜索别人的教程了

### 准备工作
```bash
#关闭防火墙
systemctl stop firewalld
systemctl disable firewalld

#关闭swap分区
swapoff -a
永久关闭需修改/etc/fstab
sudo vim /etc/fstab

#关闭selinux
setenforce 0
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
```

```bash
# 添加配置文件并加载ipvs模块（kube-proxy使用ipvs模式）
cat <<EOF | sudo tee /etc/modules-load.d/ipvs.conf
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
nf_conntrack
EOF

# 执行命令加载模块
systemctl restart systemd-modules-load.service
modprobe -- ip_vs ip_vs_rr ip_vs_wrr ip_vs_sh nf_conntrack


# 添加内核参数配置文件，参数在重新启动后保持不变
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
EOF

# 执行命令使参数生效
sysctl --system
```

### 安装前检查
```bash
# 执行命令检查selinux状态，输出应为SELinux status：disabled
sestatus

# 检查swap是否已经关闭，输出应为空
cat /proc/swaps

# 检查ipvs模块是否已经加载
lsmod |grep ip_vs

# 检查防火墙是否已经关闭
systemctl status firewalld

# 检查内核参数是否已经设置，值应该为 1
cat /proc/sys/net/ipv4/ip_forward
```

### 安装docker,容器运行时
Ubuntu系统：
```bash
# step 1: 安装必要的一些系统工具
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg

# step 2: 信任 Docker 的 GPG 公钥
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Step 3: 写入软件源信息
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://mirrors.aliyun.com/docker-ce/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
 
# Step 4: 安装Docker
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# 安装指定版本的Docker-CE:
# Step 1: 查找Docker-CE的版本:
# apt-cache madison docker-ce
#   docker-ce | 17.03.1~ce-0~ubuntu-xenial | https://mirrors.aliyun.com/docker-ce/linux/ubuntu xenial/stable amd64 Packages
#   docker-ce | 17.03.0~ce-0~ubuntu-xenial | https://mirrors.aliyun.com/docker-ce/linux/ubuntu xenial/stable amd64 Packages
# Step 2: 安装指定版本的Docker-CE: (VERSION例如上面的17.03.1~ce-0~ubuntu-xenial)
# sudo apt-get -y install docker-ce=[VERSION]
```

centos系统：
```bash
# step 1: 安装必要的一些系统工具
sudo yum install -y yum-utils

# Step 2: 添加软件源信息
yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

# Step 3: 安装Docker
sudo yum install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Step 4: 开启Docker服务
sudo service docker start

# 注意：
# 官方软件源默认启用了最新的软件，您可以通过编辑软件源的方式获取各个版本的软件包。例如官方并没有将测试版本的软件源置为可用，您可以通过以下方式开启。同理可以开启各种测试版本等。
# vim /etc/yum.repos.d/docker-ce.repo
#   将[docker-ce-test]下方的enabled=0修改为enabled=1
#
# 安装指定版本的Docker-CE:
# Step 1: 查找Docker-CE的版本:
# yum list docker-ce.x86_64 --showduplicates | sort -r
#   Loading mirror speeds from cached hostfile
#   Loaded plugins: branch, fastestmirror, langpacks
#   docker-ce.x86_64            17.03.1.ce-1.el7.centos            docker-ce-stable
#   docker-ce.x86_64            17.03.1.ce-1.el7.centos            @docker-ce-stable
#   docker-ce.x86_64            17.03.0.ce-1.el7.centos            docker-ce-stable
#   Available Packages
# Step2: 安装指定版本的Docker-CE: (VERSION例如上面的17.03.0.ce.1-1.el7.centos)
# sudo yum -y install docker-ce-[VERSION]
```
接下来这一点是之前几次失败的原因
```bash
# 生成默认配置
containerd config default | sudo tee /etc/containerd/config.toml
```
配置文件
```bash
# 修改/etc/containerd/config.toml配置
disabled_plugins = []
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
   SystemdCgroup = true # 这里改为true

[plugins."io.containerd.grpc.v1.cri"]
  # 这里修改镜像地址
  sandbox_image = "crpi-u3f530xi0zgtvmy0.cn-beijing.personal.cr.aliyuncs.com/image-infra/pause:3.10"
```
注意plugins."io.containerd.grpc.v1.cri"里面有个systemd_cgroup=true/false,之前出错就是因为有这个，和下面的SystemdCgroup冲突了，有的话注释掉
```bash

# 设置开机启动
systemctl enable containerd

# 启动containerd
systemctl start containerd

# 检查状态
systemctl status containerd
```

### 安装kubelet,kubeadm,kubectl
```bash
# 配置kubernetes 源
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.33/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.33/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF


# 安装kubelet，kubeadm，kubectl
yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes


# 启动kubelet，启动完成查看kubelet的状态可能是异常的，不用惊慌
systemctl enable --now kubelet
```

### 集群初始化
```bash
sudo kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --apiserver-advertise-address=192.168.189.100 \
  --control-plane-endpoint=192.168.189.100 \
  --upload-certs
```
### 安装flannel网络插件
```bash
wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```
补充一点，关于权限问题，在master节点上执行下面的命令，配置kubectl的权限
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```