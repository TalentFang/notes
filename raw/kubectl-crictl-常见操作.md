# kubectl & crictl 常见操作

## kubectl

kubectl 是 Kubernetes 的 CLI 工具，用于与集群 API Server 交互。

### 集群信息

```bash
kubectl cluster-info                        # 集群信息
kubectl get nodes                           # 列出所有节点
kubectl get ns                              # 列出所有命名空间
kubectl api-resources                       # 列出所有 API 资源类型
kubectl explain pod.spec.containers         # 查看资源字段说明
```

### Pod 操作

```bash
kubectl get pods                            # 默认命名空间
kubectl get pods -A                         # 所有命名空间
kubectl get pods -n <ns>                    # 指定命名空间
kubectl get pods -o wide                    # 显示更多信息（IP、节点）
kubectl get pods -o yaml                    # 以 YAML 输出

kubectl describe pod <name> -n <ns>         # Pod 详情（含 Events）

kubectl logs <pod> -n <ns>                  # 查看日志
kubectl logs <pod> -c <container> -n <ns>   # 指定容器日志
kubectl logs -f <pod> -n <ns>               # 实时跟踪（follow）
kubectl logs --tail=100 <pod> -n <ns>       # 最近 100 行
kubectl logs --since=1h <pod> -n <ns>       # 最近 1 小时

kubectl exec -it <pod> -n <ns> -- sh        # 进入容器
kubectl exec <pod> -n <ns> -- <cmd>         # 执行单条命令

kubectl port-forward <pod> 8080:80 -n <ns>  # 端口转发
kubectl cp <pod>:/path/file ./local -n <ns> # 从 Pod 拷贝文件
```

### 工作负载

```bash
kubectl get deploy,sts,ds -n <ns>           # 列出 Deploy/StatefulSet/DaemonSet
kubectl scale deploy/<name> --replicas=3 -n <ns>   # 扩缩容
kubectl rollout restart deploy/<name> -n <ns>      # 滚动重启
kubectl rollout status deploy/<name> -n <ns>       # 查看滚动状态
kubectl rollout undo deploy/<name> -n <ns>         # 回滚

kubectl edit deploy/<name> -n <ns>          # 在线编辑
kubectl patch deploy/<name> -n <ns> -p '{"spec":{"replicas":2}}'  # 打补丁
```

### 标签与选择器

```bash
kubectl get pods -l app=nginx               # 按标签筛选
kubectl label node <name> role=worker       # 添加标签
kubectl taint node <name> key=value:NoSchedule  # 添加污点
```

### 排查

```bash
kubectl get events -n <ns> --sort-by=.lastTimestamp  # 排序事件
kubectl top pods -n <ns>                    # 资源用量（需要 metrics-server）
kubectl top nodes
kubectl describe node <name>                # 节点详情（资源、Conditions）

kubectl auth can-i create pods --as=user    # 检查权限
```

---

## crictl

crictl 是 CRI（Container Runtime Interface）的 CLI 工具，直接与容器运行时（containerd、CRI-O）交互，不经过 kubelet。适合排查容器层面的问题。

### 基本信息

```bash
crictl info                                 # 运行时信息
crictl version                              # 版本
crictl stats                                # 容器资源统计
```

### 镜像管理

```bash
crictl images                               # 列出所有镜像
crictl images | grep <keyword>              # 过滤镜像
crictl rmi <image-id>                       # 删除镜像
crictl pull <image>                         # 拉取镜像
crictl imagefsinfo <id>                     # 镜像文件系统信息
```

### Pod（PodSandbox）操作

```bash
crictl pods                                 # 列出所有 Pod Sandbox
crictl pods --name <pod-name>               # 按名称过滤
crictl inspectp <pod-id>                    # Pod Sandbox 详情
crictl pod-stats                            # Pod 资源使用
```

### 容器操作

```bash
crictl ps                                   # 列出运行中的容器
crictl ps -a                                # 所有容器（含已停止）
crictl ps -a --name <container-name>        # 按名称过滤
crictl ps --label app=nginx                 # 按标签过滤

crictl inspect <container-id>               # 容器详情（State、Mounts 等）
crictl logs <container-id>                  # 查看容器日志
crictl logs -f <container-id>               # 实时跟踪
crictl logs --tail=100 <container-id>       # 最近 100 行

crictl exec -it <container-id> sh           # 进入容器
crictl exec <container-id> <cmd>            # 执行命令

crictl stats <container-id>                 # 容器资源使用
```

### 生命周期管理

```bash
crictl start <container-id>                 # 启动已创建的容器
crictl stop <container-id>                  # 停止容器
crictl rm <container-id>                    # 删除容器
crictl stopp <pod-id>                       # 停止 Pod Sandbox
crictl rmp <pod-id>                         # 删除 Pod Sandbox
crictl update --cpuset-cpus 0-3 <cid>       # 更新容器 CPU 绑核
```

### 排查常用组合

```bash
# 查 Pod 对应的容器
POD_ID=$(crictl pods --name <pod> -q)
CONTAINER_ID=$(crictl ps --pod $POD_ID -q)
crictl logs $CONTAINER_ID

# 批量清理已退出的容器
crictl rm $(crictl ps -a -q --state Exited)

# 批量清理未使用的镜像
crictl rmi --prune
```

---

## kubectl vs crictl 对应关系

| 操作 | kubectl | crictl |
|------|---------|--------|
| 列出 Pod | `kubectl get pods` | `crictl pods` |
| 列出容器 | ❌ 不支持 | `crictl ps` |
| 容器日志 | `kubectl logs <pod>` | `crictl logs <container-id>` |
| 进入容器 | `kubectl exec -it <pod> -- sh` | `crictl exec -it <cid> sh` |
| 资源统计 | `kubectl top pods` | `crictl stats` |
| Pod 详情 | `kubectl describe pod` | `crictl inspectp <pod-id>` |

> key 区别：kubectl 走 API Server → kubelet → CRI；crictl 直接对接 CRI，即使 kubelet 异常也能用。

## crictl 常用排查场景

- **Pod 一直 ContainerCreating**：`crictl pods` 找到 Pod Sandbox 是否存在，`crictl ps -a` 看容器是否创建成功
- **容器反复重启**：`crictl logs <cid>` 直接看容器标准输出，不受 kubelet 限速
- **镜像拉取失败**：`crictl pull <image>` 直接测试运行时拉取镜像
- **CPU/内存异常**：`crictl stats` 实时看容器资源使用，比 metrics-server 更及时
