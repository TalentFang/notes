### kubeadm init 使用 kubelet 的具体步骤

1. **检查 kubelet 是否运行**：`kubeadm init` 执行前会检查 `kubelet` 是否已安装并处于运行状态。
    
2. **生成静态 Pod 清单**：`kubeadm` 根据配置生成 API Server、etcd、Controller Manager、Scheduler 等组件的静态 Pod 定义文件（YAML 格式），并写入 `/etc/kubernetes/manifests/` 目录。
    
3. **依赖 kubelet 启动静态 Pod**：`kubelet` 会持续监控 `/etc/kubernetes/manifests/` 目录，一旦发现新的静态 Pod 定义文件，就立即调用容器运行时（如 containerd）启动这些 Pod。
    
4. **等待控制平面就绪**：`kubeadm` 会轮询 API Server 的健康状态，确认控制平面组件已成功启动并运行。
    
5. **生成并配置 kubeconfig**：`kubeadm` 生成 `admin.conf` 等 kubeconfig 文件，供 `kubectl` 和后续组件使用，这些文件依赖 API Server 的地址和证书。

### 静态pod vs 动态pod
#### 1. 静态 Pod（Static Pod）

- **定义**：由 `kubelet` 直接管理，放置在 `/etc/kubernetes/manifests/` 目录下的 Pod。
    
- **网络模式**：静态 Pod 默认使用 **`hostNetwork: true`**，即**直接使用节点的网络命名空间**。
    
- **IP 来源**：它的 IP 就是**节点的主 IP**，没有经过 CNI 分配虚拟 IP。
    
- **例子**：`kube-apiserver-master-01` 的 IP 是 `192.168.111.190`，这正是节点 `master-01` 的 IP。
    

#### 2. 动态 Pod（普通 Pod）

- **定义**：通过 Deployment、StatefulSet 等控制器创建，由 API Server 调度到节点上的 Pod。
    
- **网络模式**：使用 **CNI 插件**分配独立的网络命名空间，每个 Pod 拥有一个独立的虚拟 IP。
    
- **IP 来源**：由 CNI 插件（如 Calico、Flannel）从 Pod 网络 CIDR 中分配，与节点 IP 不在同一个网段。
    
- **例子**：`coredns-9f86ffc8d-p5rhz` 的 IP 是 `113.122.184.99`，这是 Calico 分配的虚拟 IP。