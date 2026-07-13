

> 本文档基于对话历史整理，涵盖 Kubernetes 核心概念、网络模型、组件关系及常见运维操作。

---

## 1. Kubernetes 基础架构

### 1.1 节点角色

| 角色 | 职责 | 运行组件 |
|------|------|----------|
| **Master / Control Plane** | 集群管理、调度、决策 | API Server、etcd、Scheduler、Controller Manager |
| **Worker / Node** | 运行用户应用程序 Pod | kubelet、kube-proxy、容器运行时 |

> **本质**：Master = 大脑（决策），Worker = 手脚（执行）。

### 1.2 节点组件

| 组件 | 所在节点 | 作用 |
|------|----------|------|
| **kubelet** | 所有节点 | 节点代理，管理 Pod 生命周期、健康检查、状态上报 |
| **kube-proxy** | 所有节点 | 维护网络规则，实现 Service 的负载均衡 |
| **容器运行时** | 所有节点 | 实际运行容器（containerd、CRI-O 等） |

---

## 2. Master 节点详解

### 2.1 Master 默认不允许调度普通 Pod

- **原因**：通过污点（Taint）隔离，防止业务 Pod 抢占控制平面资源。
- **查看污点**：

```bash
kubectl describe node <master-node> | grep Taints
# 输出: node-role.kubernetes.io/control-plane:NoSchedule
```

- **允许调度（移除污点）**：

```bash
kubectl taint nodes <master-node> node-role.kubernetes.io/control-plane:NoSchedule-
```

- **恢复禁止调度**：

```bash
kubectl taint nodes <master-node> node-role.kubernetes.io/control-plane:NoSchedule
```

### 2.2 控制平面组件运行方式

控制平面组件（API Server、etcd、Controller Manager、Scheduler）以**静态 Pod** 形式运行：

- 由 kubelet 直接管理，不受 API Server 调度
- 配置文件位于 `/etc/kubernetes/manifests/`
- 使用 `hostNetwork: true`，直接绑定节点 IP

---

## 3. Pod 详解

### 3.1 什么是 Pod

- Kubernetes 中最小的部署单元
- 包含一个或多个容器，共享网络命名空间和存储卷

### 3.2 静态 Pod vs 动态 Pod

| 特性 | 静态 Pod | 动态 Pod（普通 Pod） |
|------|----------|---------------------|
| **管理方式** | kubelet 直接管理 | API Server + 控制器（Deployment 等） |
| **网络模式** | 默认 `hostNetwork: true` | CNI 分配的虚拟网络 |
| **IP 来源** | 节点的主 IP | CNI 分配的虚拟 IP（如 Calico、Flannel） |
| **典型用途** | 控制平面组件（API Server、etcd） | 业务应用（nginx、coredns 等） |

---

## 4. Kubernetes 网络模型

### 4.1 CNI（Container Network Interface）

- **定义**：容器网络接口标准，用于为 Pod 配置网络和分配 IP。
- **部署方式**：以 **DaemonSet** 形式在每个节点上运行。
- **调用方式**：**本地调用**（kubelet → 容器运行时 → CNI 插件）。
- **常见插件**：Calico、Flannel、Cilium、Weave。

> **IP 分配时机**：Pod 创建时，由 CNI 插件从 Pod 网络 CIDR 中分配。

### 4.2 kube-proxy

- **角色**：Kubernetes 内部的 L4 虚拟路由器/负载均衡器。
- **本质**：网络规则配置员，通过修改宿主机内核的 `iptables` 或 `IPVS` 规则实现流量转发。
- **部署方式**：DaemonSet，每个节点一个 Pod。
- **工作模式**：
  - `iptables`：默认，适合中小规模集群
  - `IPVS`：高性能，适合大规模集群
  - `userspace`：已基本淘汰

### 4.3 三种网络组件对比

| 组件 | 部署方式 | 调用方式 | 作用 |
|------|----------|----------|------|
| **CNI 插件** | DaemonSet（每节点一个） | 本地调用 | 为 Pod 分配 IP，配置网络 |
| **kube-proxy** | DaemonSet（每节点一个） | 监听 API Server，维护本地规则 | 实现 Service L4 负载均衡 |
| **Ingress Controller** | Deployment + Service | 监听 API Server | 实现 L7（HTTP/HTTPS）路由 |

---

## 5. Service 详解

### 5.1 Service 的本质

Service 是一个**四层负载均衡器的抽象**，是一个在 etcd 中存储的**资源配置对象**。

- 为动态变化的 Pod IP 提供**固定的访问入口（ClusterIP）**
- 本身不运行进程，由 kube-proxy 实现其功能

### 5.2 Service 类型

| 类型 | 说明 | 适用场景 |
|------|------|----------|
| **ClusterIP** | 集群内部虚拟 IP | 内部服务相互调用 |
| **NodePort** | 在每个节点开放静态端口 | 测试/简单外部访问 |
| **LoadBalancer** | 云厂商负载均衡器 | 生产环境对外服务 |
| **ExternalName** | DNS 映射到外部服务 | 访问外部服务 |

### 5.3 访问流程

```
客户端 Pod → Service ClusterIP → kube-proxy 维护的 iptables/IPVS 规则 → 后端 Pod
```

---

## 6. kubeadm 部署

### 6.1 kubeadm 是什么

Kubernetes 官方提供的**集群部署工具**，用于快速初始化、加入节点和升级集群。

### 6.2 kubeadm init 与 kubelet 的关系

`kubeadm init` **依赖 kubelet** 来启动控制平面的静态 Pod：

1. `kubeadm` 生成静态 Pod 清单到 `/etc/kubernetes/manifests/`
2. `kubelet` 监控该目录，启动控制平面组件
3. `kubeadm` 等待 API Server 就绪

> 没有 kubelet，kubeadm init 无法完成。

### 6.3 高可用集群（2 Master + 1 Worker）部署要点

- 需要**负载均衡器**（如 HAProxy）为多个 Master 提供统一入口
- 第一个 Master：`kubeadm init --upload-certs`
- 第二个 Master：`kubeadm join --control-plane`
- Worker 节点：`kubeadm join`（不带 `--control-plane`）

---

## 7. 常见运维命令

### 7.1 查看节点

```bash
kubectl get nodes -o wide
```

### 7.2 查看 Pod 及其所在节点

```bash
kubectl get pods -A -o wide
```

### 7.3 查看特定节点上的 Pod

```bash
kubectl get pods -A --field-selector spec.nodeName=<node-name>
```

### 7.4 查看节点污点

```bash
kubectl describe node <node-name> | grep Taints
```

### 7.5 查看 kube-system 命名空间组件

```bash
kubectl get pods -n kube-system -o wide
```

### 7.6 查看 kubelet 日志（在节点本地）

```bash
journalctl -u kubelet -f
```

---

## 8. 关键概念速查

| 概念 | 一句话总结 |
|------|-----------|
| **Pod** | Kubernetes 最小部署单元，一个或多个容器 |
| **Service** | 为动态 Pod 提供固定访问入口的抽象 |
| **kube-proxy** | 维护网络规则，实现 Service L4 负载均衡 |
| **CNI** | 为 Pod 分配 IP 和配置网络的插件标准 |
| **kubelet** | 每个节点的代理，管理 Pod 生命周期 |
| **静态 Pod** | kubelet 直接管理的 Pod，用于控制平面组件 |
| **污点（Taint）** | 阻止 Pod 调度到特定节点的机制 |

---

> **维护提示**：本文档基于 2026-06-17 的对话整理，Kubernetes 版本持续演进，建议结合官方文档使用。