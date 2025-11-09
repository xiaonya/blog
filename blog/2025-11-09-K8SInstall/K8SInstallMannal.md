---
slug: 2025-11-09-K8SInstall
title: K8S 安装详细步骤记录
authors: xiaonya
tags: ["K8S","containerd","calico","kubeadm","getting-started","tutorial"]
---

记录初学K8S时的安装步骤以及踩到的坑，供自己和他人参考。
<!-- truncate -->

# K8S 安装记录
## 目标
k8s集群，使用containerd作为容器运行时，calico作为网络插件，coredns作为dns服务，etcd作为集群存储。
## 环境
Fedora 42 \
Linux 6.16.8-200.fc42.x86_64
## 基本思路
1. 准备环境
2. 安装containerd
3. 配置ctr配置以及镜像源
4. 安装kubeadm、kubelet、kubectl
5. 初始化master节点
6. 安装网络插件calico
7. 加入worker节点
8. 检查集群状态

## 详细步骤
### 1. 准备环境
- 关闭swap，开启内核ip转发
```bash
# 1.关闭 swap
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
# Fedora系统还需要关闭zram swap，其他系统不清楚
# 参考 FedoraDisableSWAP.md


# 2.加载内核模块
sudo modprobe overlay
sudo modprobe br_netfilter

# 3.设置内核参数
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system
```
- 关闭防火墙和selinux
```bash
sudo systemctl disable --now firewalld
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=disabled/' /etc/selinux/config
```
### 2. 安装containerd运行时
```bash
sudo dnf install -y containerd
sudo systemctl enable --now containerd
```
### 3. 配置containerd
- 生成默认配置文件
```bash
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
```
- 修改配置文件/etc/containerd/config.toml
- 不能直接在文件尾部添加内容，需要找到对应位置进行修改，确认不存在才能直接添加
```bash
# 1. 使用SystemdCgroup
# 参考https://github.com/containerd/containerd/blob/main/docs/cri/config.md#cgroup-driver
version = 3
[plugins.'io.containerd.cri.v1.runtime'.containerd.runtimes.runc.options]
  SystemdCgroup = true
  
# 2. 配置阿里云镜像源
# 参考1. https://github.com/containerd/containerd/blob/main/docs/cri/registry.md
# 参考2. https://bychannel.github.io/containerd%E9%85%8D%E7%BD%AE%E9%95%9C%E5%83%8F%E5%8A%A0%E9%80%9F/
# 参考3. https://help.aliyun.com/zh/acr/user-guide/accelerate-the-pulls-of-docker-official-images
# 需要自己确定containerd的版本号，接下来以containerd 2.x为例
# 在cri配置文件中添加如下内容，指定镜像加速地址文件夹
[plugins."io.containerd.cri.v1.images".registry]
   config_path = "/etc/containerd/certs.d"
# 创建文件夹
sudo mkdir -p /etc/containerd/certs.d/docker.io
# 创建加速地址文件,该文件地址暂未找到官方文档说明
# 建议直接根据参考2创建

# 重启containerd
sudo systemctl restart containerd

```

### 4. 安装kubeadm、kubelet、kubectl
- 添加kubernetes源
```bash
# 参考https://mirrors.ustc.edu.cn/help/kubernetes.html#yum
# 执行如下命令（注意修改为自己需要的版本号）：
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.ustc.edu.cn/kubernetes/core:/stable:/v1.34/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.34/rpm/repodata/repomd.xml.key
EOF
# 如果需要使用 CRI-O，执行以下命令：
cat <<EOF | tee /etc/yum.repos.d/cri-o.repo
[cri-o]
name=CRI-O
baseurl=https://mirrors.ustc.edu.cn/kubernetes/addons:/cri-o:/stable:/v1.32/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.32/rpm/repodata/repomd.xml.key
EOF
```
- 安装kubeadm、kubelet、kubectl
```bash
sudo dnf install -y kubelet kubeadm kubectl
sudo systemctl enable --now kubelet
```


### 5. 初始化master节点
> 可以提前使用kubeadm config images list命令查看需要拉取的镜像，然后使用ctr命令提前拉取
> 一般来说指定镜像源后应该不会卡了，拉取的相关说明需要参考第6步的说明。
```bash
sudo kubeadm init --image-repository registry.aliyuncs.com/google_containers
# 此处可能会卡在以下这种情况
# [kubelet-check] Waiting for a healthy kubelet at http://127.0.0.1:10248/healthz. This can take up to 4m0s
# 这种情况往往是kubectl没有启动成功,无法验证安装是否成功，可以使用下面的命令查看kubelet日志,定位问题
sudo journalctl -xeu kubelet -n 50
# 修好问题后执行以下命令重试
sudo kubeadm reset
sudo kubeadm init --image-repository registry.aliyuncs.com/google_containers
```
- 初始化流程结束后配置kubectl
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

- 集群信息获取
```bash
# 查看安装的k8s版本
kubectl version
# 查看集群信息
kubectl cluster-info
# 查看各种组件版本
kubectl get api-versions
# 查看组件状态
kubectl get componentstatuses
```
- 检查系统pod状态
```bash
# 查看所有命名空间的pod状态
kubectl get pods -A
# 查看指定命名空间的pod状态
kubectl get pods -n kube-system
# 如果看到有些pod状态非Running，可以使用下面的命令查错
kubectl describe pod <pod名称> -n <命名空间>
```
- 将所有节点标记为可调度
```bash
# 处理以下警告使用
# Warning  FailedScheduling  41s (x2 over 6m5s)  default-scheduler  0/1 nodes are available: 1 node(s) had untolerated taint {node.kubernetes.io/not-ready: }. no new claims to deallocate, preemption: 0/1 nodes are available: 1 Preemption is not helpful for scheduling.
kubectl taint nodes --all node-role.kubernetes.io/master-
```
>在未安装网络插件前，pods和coredns的状态可能不是Ready和Running，这是正常的，将在下一步安装网络插件后解决。

### 6. 安装网络插件calico
```bash
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
# 接着使用第5步的检查系统pod状态指令，确认网络插件正常工作，且node状态为Ready，才算是集群正常工作
```
>此处新手常遇到的问题是镜像未生效，containerd的操作方式与docker有所区别，接下来介绍如何使用ctr命令查看和拉取镜像。

需要注意的几点：
- 在ctr中存在命名空间的区分，默认使用的命名空间是default，而**k8s使用的命名空间是k8s.io**，所以想要查看k8s的镜像需要指定-n参数
- ctr命令不会自动使用镜像源，需要指定–hosts-dir=/etc/containerd/certs.d
- 想要自动使用镜像源，可以使用crictl命令拉取
- 如果要确定此命令是否真的使用了镜像加速，可以增加–debug=true参数
- ctr命令存在使用image的简写i的使用方式
```bash
# 查看所有镜像
sudo ctr -n <镜像地址> images list
# 拉取镜像
sudo ctr -n k8s.io images pull <镜像地址>
# 例如拉取coredns的镜像
ctr i pull --hosts-dir=/etc/containerd/certs.d docker.io/calico/cni:v3.25.0
ctr i pull --hosts-dir=/etc/containerd/certs.d registry.k8s.io/sig-storage/csi-provisioner:v3.5.0

# 查看k8s.io命名空间下的所有镜像
sudo ctr -n k8s.io i ls

# 使用crictl拉取镜像
sudo crictl pull <镜像地址>
```
### 7. 加入worker节点
正常完成前面的1-4步后，master节点已经可以正常工作了，接下来需要将worker节点加入集群，woker节点的1-4步操作与master节点相同，完成后执行以下操作：
- 在master节点上获取新的token以及加入命令
```bash
kubeadm token create --print-join-command
```
- 在worker节点上执行获取到的命令
```bash
sudo kubeadm join <master节点IP>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash值>
# 例kubeadm join 192.168.0.62:6443 --token 8vm7ed.o0c8hd1sni050ujg --discovery-token-ca-cert-hash sha256:913eb08165815807a28e5b11b3d5c9f16541d4dc9cf36079e1d2ceed54690832
```
- token相关命令
```bash
# 查看token
kubeadm token list
# 创建新的token
kubeadm token create
# 删除token
kubeadm token delete <token>
```

### 8. 检查集群状态
```bash
kubectl get nodes
kubectl get pods -A
```
- 如果需要删除集群，使用以下命令
```bash
sudo kubeadm reset
sudo rm -rf $HOME/.kube
sudo systemctl disable --now kubelet
sudo dnf remove -y kubelet kubeadm kubectl
sudo dnf remove -y containerd
sudo systemctl disable --now containerd
```
