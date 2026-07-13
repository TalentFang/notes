# Kubernetes 学习文档

## 目录

1. [Kubernetes 概述](#1-kubernetes-概述)
2. [核心组件介绍](#2-核心组件介绍)
3. [组件架构图](#3-组件架构图)
4. [部署方式](#4-部署方式)
5. [部署实践 - Minikube](#5-部署实践---minikube)
6. [部署实践 - K3s](#6-部署实践---k3s)
7. [部署实践 - Kubeadm 高可用集群](#7-部署实践---kubeadm-高可用集群)

---

## 1. Kubernetes 概述

Kubernetes（简称 K8s）是一个开源的容器编排平台，用于自动化容器化应用的部署、扩缩容和管理。

### 核心特性

| 特性 | 说明 |
|------|------|
| 自我修复 | 自动重启失败的容器，替换和重新调度不可死的容器 |
| 水平扩缩容 | 根据 CPU 等资源使用率自动扩缩容应用 |
| 服务发现与负载均衡 | 为容器提供统一的访问入口，自动分配 IP |
| 自动化 rollout/rollback | 平滑升级或回滚应用版本 |
| 配置与密钥管理 | 管理配置数据和敏感信息 |
| 存储编排 | 自动挂载存储系统 |

---

## 2. 核心组件介绍

### 2.1 控制平面组件 (Control Plane)

#### kube-apiserver

Kubernetes 控制平面的前端，提供 REST API 接口，是所有组件交互的中心。

- 暴露 Kubernetes API
- 处理所有 RESTful 请求
- 是唯一与 etcd 通信的组件
- 支持水平扩展

#### etcd

轻量级、分布式的键值存储，用于保存整个集群的状态数据。

- 存储集群配置数据和状态
- 使用 Raft 一致性算法保证数据一致性
- 建议生产环境使用奇数节点（1/3/5/7）
- 所有 API 对象存储在 `/registry` 路径下

#### kube-controller-manager

运行控制器进程的组件，负责维护集群状态。

| 控制器 | 功能 |
|--------|------|
| Node Controller | 监测节点状态，响应节点故障 |
| Replication Controller | 维护 Pod 副本数 |
| Endpoints Controller | 管理服务 endpoints |
| Service Account Controller | 创建默认 Service Account |
| Cloud Controller Manager | 与云服务商集成 |

#### kube-scheduler

负责 Pod 的调度，将新创建的 Pod 调度到合适的节点。

调度决策考虑因素：
- 资源需求（CPU/内存）
- 亲和性/反亲和性规则
- 拓扑限制（区域/可用区）
- 污点和容忍

### 2.2 Node 组件

#### kubelet

运行在每个节点上的代理，负责管理容器生命周期。

- 向 API Server 注册节点
- 监控容器健康状态
- 启动/停止容器
- 汇报节点资源情况

#### kube-proxy

运行在每个节点上的网络代理，维护网络规则。

- 允许 Pod 到 Pod 的通信
- 实现 Service 负载均衡
- 支持 iptables 或 IPVS 模式

#### container runtime

容器运行时，负责运行容器。

支持的运行时：
- containerd
- CRI-O
- Docker Engine (通过 dockershim，已弃用)

### 2.3 附加组件

| 组件 | 功能 |
|------|------|
| kube-dns/CoreDNS | 集群 DNS 服务 |
| Ingress Controller | HTTP/HTTPS 路由 |
| Metrics Server | 资源监控指标 |
| Dashboard | Web UI 管理界面 |
| Prometheus | 监控和告警 |

---

## 3. 组件架构图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              Control Plane                                  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐ │
│  │ kube-       │  │   etcd      │  │  kube-      │  │   kube-scheduler    │ │
│  │ apiserver   │──│             │  │ controller- │  │                     │ │
│  │             │  │             │  │ manager     │  │                     │ │
│  └──────┬──────┘  └──────┬──────┘  └─────────────┘  └─────────────────────┘ │
│         │                │                                                      │
│         └────────────────┼──────────────────────────────────────              │
│                          │                                                      │
└──────────────────────────┼──────────────────────────────────────────────────┘
                           │ API
┌──────────────────────────┼──────────────────────────────────────────────────┐
│                          │                                                      │
│     ┌────────────────────┴────────────────────┐                               │
│     │              Node 1                     │                               │
│     │  ┌──────────┐  ┌──────────┐  ┌───────┐  │                               │
│     │  │ kubelet  │  │ kube-    │  │ Pods  │  │                               │
│     │  │          │  │ proxy    │  │       │  │                               │
│     │  └──────────┘  └──────────┘  │ ┌───┐ │  │                               │
│     │                              │ │   │ │  │                               │
│     │                              │ │   │ │  │                               │
│     │                              └───┴─┘ │  │                               │
│     └────────────────┬───────────────────┘                               │
│                    │                                                       │
│     ┌──────────────┴───────────────────┐                                 │
│     │              Node 2              │                                 │
│     │  ┌──────────┐  ┌──────────┐  ┌───────┐  │                        │
│     │  │ kubelet  │  │ kube-    │  │ Pods  │  │                        │
│     │  │          │  │ proxy    │  │       │  │                        │
│     │  └──────────┘  └──────────┘  │ ┌───┐ │  │                        │
│     │                              │ │   │ │  │                        │
│     │                              │ │   │ │  │                        │
│     │                              └───┴─┘ │  │                        │
│     └────────────────┬───────────────────┘                               │
│                    │                                                       │
│     ┌──────────────┴───────────────────┐                                 │
│     │              Node N              │                                 │
│     │  ┌──────────┐  ┌──────────┐  ┌───────┐  │                        │
│     │  │ kubelet  │  │ kube-    │  │ Pods  │  │                        │
│     │  │          │  │ proxy    │  │       │  │                        │
│     │  └──────────┘  └──────────┘  │ ┌───┐ │  │                        │
│     │                              │ │   │ │  │                        │
│     │                              │ │   │ │  │                        │
│     │                              └───┴─┘ │  │                        │
│     └────────────────┬───────────────────┘                               │
└─────────────────────┼───────────────────────────────────────────────────┘
                      │
              ┌───────┴───────┐
              │  Container    │
              │  Runtime      │
              │ (containerd)  │
              └───────────────┘
```

### Kubernetes 核心对象关系图

```
┌─────────────────────────────────────────────────────────────────┐
│                         Kubernetes 集群                         │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                     Control Plane                         │  │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────────┐  ┌─────────┐  │  │
│  │  │ API     │  │ etcd    │  │ Controller  │  │ Sched- │  │  │
│  │  │ Server  │  │         │  │ Manager     │  │ uler   │  │  │
│  │  └────┬────┘  └────┬────┘  └─────────────┘  └────┬────┘  │  │
│  └───────┼────────────┼─────────────────────────────┼────────┘  │
│          │            │                             │           │
└──────────┼────────────┼─────────────────────────────┼───────────┘
           │            │                             │
           ▼            ▼                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                          Nodes                                  │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐  │
│  │     Node 1      │  │     Node 2      │  │     Node N      │  │
│  │  ┌───────────┐  │  │  ┌───────────┐  │  │  ┌───────────┐  │  │
│  │  │  kubelet  │  │  │  │  kubelet  │  │  │  │  kubelet  │  │  │
│  │  └─────┬─────┘  │  │  └─────┬─────┘  │  │  └─────┬─────┘  │  │
│  │        │        │  │        │        │  │        │        │  │
│  │  ┌─────┴─────┐  │  │  ┌─────┴─────┐  │  │  ┌─────┴─────┐  │  │
│  │  │ kube-proxy│  │  │  │ kube-proxy│  │  │  │ kube-proxy│  │  │
│  │  └─────┬─────┘  │  │  └─────┬─────┘  │  │  └─────┬─────┘  │  │
│  │        │        │  │        │        │  │        │        │  │
│  │  ┌─────┴─────┐  │  │  ┌─────┴─────┐  │  │  ┌─────┴─────┐  │  │
│  │  │ container │  │  │  │ container │  │  │  │ container │  │  │
│  │  │ runtime   │  │  │  │ runtime   │  │  │  │ runtime   │  │  │
│  │  └───────────┘  │  │  └───────────┘  │  │  └───────────┘  │  │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4. 部署方式

### 4.1 部署方式对比

| 方式 | 适用场景 | 复杂度 | 管理难度 |
|------|----------|--------|----------|
| Minikube | 本地开发/学习 | 低 | 简单 |
| Kind | 本地测试/CI | 低 | 简单 |
| K3s | 边缘/IoT/轻量级 | 中 | 中等 |
| Kubeadm | 生产环境 | 高 | 复杂 |
| Rancher | 企业级/多集群 | 中 | 简单 |
| eks/aks/gke | 云环境 | 低 | 简单 |

### 4.2 各部署方式说明

#### Minikube

单节点本地 Kubernetes 环境，适合学习和开发。

```bash
# 系统要求
# - 2 CPU+
# - 2GB 内存
# - 20GB 磁盘
# - 支持虚拟化（VirtualBox/Hyper-V/KVM）
```

#### Kind (Kubernetes in Docker)

使用 Docker 容器运行 Kubernetes 集群，适合 CI/CD 和本地测试。

#### K3s

轻量级 Kubernetes，适合资源受限环境（边缘、IoT）。

- 二进制文件 < 100MB
- 低内存占用 (~512MB)
- 支持 SQLite 替代 etcd（单节点）

#### Kubeadm

官方推荐的集群创建工具，适合生产环境。

#### 云服务商托管

| 服务商 | 产品 |
|--------|------|
| AWS | EKS |
| Azure | AKS |
| Google Cloud | GKE |
| 阿里云 | ACK |
| 腾讯云 | TKE |

---

## 5. 部署实践 - Minikube

### 5.1 环境准备

```bash
# 检查虚拟化支持 (Linux)
grep -E '(vmx|svm)' /proc/cpuinfo

# 安装 kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# 安装 Docker (如未安装)
sudo apt-get update
sudo apt-get install -y docker.io

# 安装 conntrack（依赖）
sudo apt-get install -y conntrack
```

### 5.2 安装 Minikube

```bash
# 下载 minikube
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo mv minikube-linux-amd64 /usr/local/bin/minikube
sudo chmod +x /usr/local/bin/minikube

# 验证安装
minikube version
```

### 5.3 启动集群

```bash
# 使用 docker 驱动启动
minikube start --driver=docker

# 指定资源
minikube start --driver=docker --cpus=4 --memory=8g

# 查看状态
minikube status

# 查看 kubectl 配置
kubectl config current-context
```

### 5.4 常用操作

```bash
# 部署应用
kubectl create deployment nginx --image=nginx
kubectl get deployments

# 暴露服务
kubectl expose deployment nginx --port=80 --type=NodePort

# 查看 pods
kubectl get pods

# 访问服务（自动打开浏览器）
minikube service nginx

# 查看集群信息
kubectl cluster-info

# 进入 minikube 环境
minikube ssh

# 停止集群
minikube stop

# 删除集群
minikube delete

# 查看控制台
minikube dashboard
```

### 5.5 部署示例

```yaml
# nginx-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
  type: NodePort
```

```bash
# 部署
kubectl apply -f nginx-deployment.yaml

# 查看部署
kubectl get deploy,pods,services

# 查看 pod 日志
kubectl logs -l app=nginx

# 扩缩容
kubectl scale deployment nginx-deployment --replicas=5

# 删除
kubectl delete -f nginx-deployment.yaml
```

---

## 6. 部署实践 - K3s

K3s 是轻量级 Kubernetes，适合学习、边缘计算和资源受限环境。

### 6.1 单节点部署

```bash
# 安装 K3s（自动包含 kubectl）
curl -sfL https://get.k3s.io | sh -

# 检查服务状态
sudo systemctl status k3s

# 查看节点
kubectl get nodes

# 查看集群信息
kubectl cluster-info
```

### 6.2 查看 kubeconfig

```bash
# 默认配置位置
cat /etc/rancher/k3s/k3s.yaml

# 复制到用户目录
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config
export KUBECONFIG=~/.kube/config

# 测试
kubectl get nodes
```

### 6.3 多节点集群部署

```bash
# 服务器节点（控制平面）
curl -sfL https://get.k3s.io | K3S_TOKEN=<TOKEN> sh -

# 获取 token
cat /var/lib/rancher/k3s/server/node-token

# 代理节点加入
curl -sfL https://get.k3s.io | K3S_URL=https://<SERVER_IP>:6443 K3S_TOKEN=<TOKEN> sh -
```

### 6.4 使用 SQLite（单节点可选）

```bash
# 默认使用 etcd，指定 SQLite
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="server --cluster-init" sh -
```

### 6.5 卸载 K3s

```bash
# 在服务器上
/usr/local/bin/k3s-uninstall.sh

# 在代理上
/usr/local/bin/k3s-agent-uninstall.sh
```

---

## 7. 部署实践 - Kubeadm 高可用集群

### 7.1 环境准备

#### 系统要求

| 组件 | 要求 |
|------|------|
| CPU | 2核+ |
| 内存 | 2GB+ |
| 磁盘 | 20GB+ |
| 系统 | Ubuntu 16.04+ / CentOS 7+ |

#### 所有节点执行

```bash
# 设置主机名
hostnamectl set-hostname k8s-master-1

# 添加 hosts
cat >> /etc/hosts << EOF
192.168.1.101 k8s-master-1
192.168.1.102 k8s-master-2
192.168.1.103 k8s-node-1
192.168.1.104 k8s-node-2
EOF

# 关闭防火墙
systemctl stop firewalld
systemctl disable firewalld

# 关闭 SELinux
setenforce 0
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config

# 关闭 swap
swapoff -a
sed -i '/ swap / s/^\(.*\)$/#\1/' /etc/fstab

# 加载桥接模块
cat > /etc/modules-load.d/k8s.conf << EOF
br_netfilter
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
nf_conntrack
EOF

modprobe br_netfilter
modprobe ip_vs

# 设置网桥参数
cat > /etc/sysctl.d/k8s.conf << EOF
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF

sysctl -p /etc/sysctl.d/k8s.conf

# 安装 containerd
apt-get update
apt-get install -y containerd

# 配置 containerd
mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml

# 重启 containerd
systemctl restart containerd
systemctl enable containerd

# 安装 kubeadm, kubelet, kubectl
apt-get install -y apt-transport-https curl
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-archive-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' > /etc/apt/sources.list.d/kubernetes.list
apt-get update
apt-get install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl
```

### 7.2 初始化控制平面（第一个 master）

```bash
# 创建高可用配置文件
cat > kubeadm-config.yaml << EOF
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
clusterName: kubernetes
kubernetesVersion: v1.28.0
controlPlaneEndpoint: "192.168.1.100:6443"
networking:
  podSubnet: "10.244.0.0/16"
  serviceSubnet: "10.96.0.0/12"
apiServer:
  certSANs:
  - 192.168.1.101
  - 192.168.1.102
  - 192.168.1.100
  - 127.0.0.1
  localAPIEndpoint:
    advertiseAddress: 192.168.1.101
    bindPort: 6443
etcd:
  local:
    serverCertSANs:
    - k8s-master-1
    - 192.168.1.101
    peerCertSANs:
    - k8s-master-1
    - 192.168.1.101
EOF

# 初始化（第一个 master）
kubeadm init --config=kubeadm-config.yaml --upload-certs

# 保存输出信息（后续加入节点需要）
# 记录以下内容：
# - kubeadm join 命令（控制平面）
# - kubeadm join 命令（工作节点）
# - 证书加密密钥

# 配置 kubectl
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```

### 7.3 加入其他控制平面节点

```bash
# 在其他 master 节点执行（使用初始化时的命令）
kubeadm join 192.168.1.100:6443 \
  --control-plane \
  --certificate-key <certificate-key> \
  --discovery-token <discovery-token> \
  --apiserver-advertise-address=192.168.1.102
```

### 7.4 加入工作节点

```bash
# 在工作节点执行
kubeadm join 192.168.1.100:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

### 7.5 安装网络插件（Calico）

```bash
# 下载 Calico manifest
curl -LO https://docs.projectcalico.org/v3.26/manifests/calico.yaml

# 修改 pod 子网（如需要）
# - name: CALICO_IPV4POOL_CIDR
#   value: "10.244.0.0/16"

# 应用
kubectl apply -f calico.yaml

# 检查状态
kubectl get pods -n kube-system
kubectl get nodes
```

### 7.6 常用故障排查

```bash
# 查看 kubelet 日志
journalctl -u kubelet -f

# 重置集群
kubeadm reset
rm -rf /etc/kubernetes/manifests/ /var/lib/etcd /var/lib/kubelet

# 检查证书
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text -noout

# 检查 etcd 状态
crictl --endpoint unix:///var/run/containerd/containerd.sock \
  ps | grep etcd
```

---

## 附录：常用命令速查

```bash
# 集群信息
kubectl cluster-info
kubectl get nodes
kubectl describe node <node-name>

# Pod 操作
kubectl get pods
kubectl get pods -o wide
kubectl describe pod <pod-name>
kubectl logs <pod-name>
kubectl exec -it <pod-name> -- /bin/bash

# Deployment 操作
kubectl get deployments
kubectl create -f <file.yaml>
kubectl apply -f <file.yaml>
kubectl delete -f <file.yaml>
kubectl rollout status deployment/<name>
kubectl rollout undo deployment/<name>

# Service 操作
kubectl get services
kubectl expose deployment <name> --port=80 --type=NodePort
kubectl describe service <name>

# 调试
kubectl get events --sort-by='.lastTimestamp'
kubectl top nodes
kubectl top pods

# 上下文
kubectl config get-contexts
kubectl config use-context <context-name>
```

---

## 参考链接

- [Kubernetes 官方文档](https://kubernetes.io/zh/docs/)
- [Kubernetes GitHub](https://github.com/kubernetes/kubernetes)
- [K3s 官方文档](https://docs.k3s.io/)
- [Minikube 官方文档](https://minikube.sigs.k8s.io/docs/)
- [Kubeadm 官方文档](https://kubernetes.io/zh/docs/reference/setup-tools/kubeadm/)