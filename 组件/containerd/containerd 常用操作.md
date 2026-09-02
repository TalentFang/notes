# Containerd 常用操作

## 概述

Containerd 是行业标准的容器运行时，强调简单性、健壮性和可移植性。它管理容器的完整生命周期。与之交互的三套 CLI 工具：

| 工具 | 说明 | 适用场景 |
|------|------|----------|
| `ctr` | containerd 自带的调试客户端 | 底层操作、调试 |
| `crictl` | Kubernetes CRI 兼容客户端 | K8s 节点排错 |
| `nerdctl` | Docker 兼容 CLI（第三方） | 日常开发、兼容 Docker 习惯 |

---

## ctr —— 底层操作

### 镜像管理

```bash
# 拉取镜像
ctr image pull docker.io/library/nginx:latest
ctr image pull --platform linux/amd64 docker.io/library/alpine:latest

# 列出镜像
ctr image ls
ctr i ls -q                           # 仅显示名称

# 导出/导入镜像
ctr image export nginx.tar docker.io/library/nginx:latest
ctr image import nginx.tar

# 删除镜像
ctr image rm docker.io/library/nginx:latest

# 查看镜像详情
ctr image inspect docker.io/library/alpine:latest
```

### 容器管理

```bash
# 创建容器（不启动）
ctr container create docker.io/library/nginx:latest my-nginx

# 创建并启动
ctr run -d docker.io/library/nginx:latest my-nginx
ctr run --rm docker.io/library/alpine:latest test-sh sh

# 列出容器
ctr container ls
ctr c ls

# 查看容器详情
ctr container info my-nginx

# 启动/停止
ctr task start my-nginx                # 启动已有容器的任务
ctr task kill my-nginx                 # 停止任务
ctr task kill -s 9 my-nginx            # 强制终止

# 列出运行中的任务
ctr task ls

# 进入容器
ctr task exec -t --exec-id shell-1 my-nginx /bin/bash

# 暂停/恢复
ctr task pause my-nginx
ctr task resume my-nginx

# 删除容器
ctr container rm my-nginx
ctr c rm my-nginx
```

### 命名空间

```bash
# 列出命名空间
ctr ns ls

# 在指定命名空间操作
ctr -n k8s.io image ls                 # K8s 使用的默认命名空间
ctr -n k8s.io container ls

# 切换默认命名空间
export CONTAINERD_NAMESPACE=k8s.io
```

### 快照管理

```bash
ctr snapshot ls
ctr snapshot info <snapshot-key>
ctr snapshot tree <snapshot-key>       # 查看快照依赖树
```

---

## crictl —— Kubernetes 节点排错

### Pod 管理

```bash
# 列出 Pod
crictl pods
crictl pods -o wide
crictl pods --state Ready

# 查看 Pod 详情
crictl inspectp <pod-id>

# 查看 Pod 日志（导出到文件）
crictl logs -f <container-id>
```

### 容器管理

```bash
# 列出容器
crictl ps                            # 运行中的容器
crictl ps -a                         # 所有容器（含已退出）
crictl ps -a --state Exited

# 查看容器详情
crictl inspect <container-id>
crictl inspect <container-id> | jq .status.reason   # 退出原因

# 进入容器
crictl exec -it <container-id> /bin/bash
crictl exec -it <container-id> sh

# 查看日志
crictl logs <container-id>
crictl logs -f --tail=100 <container-id>
crictl logs --since=5m <container-id>

# 停止/删除
crictl stop <container-id>
crictl rm <container-id>
```

### 镜像管理

```bash
crictl images
crictl image ls
crictl pull docker.io/library/alpine:latest
crictl rmi <image-id>
crictl image info <image-id>
```

### 运行时信息

```bash
crictl info                          # containerd 运行时信息
crictl version                       # CRI 版本
crictl stats <container-id>          # 容器资源使用
```

### Pod 日志

```bash
# 查看 Pod 所有容器日志
crictl logs <container-id>

# 持续跟踪
crictl logs -f <container-id>

# 按时间过滤（crictl v1.27+）
crictl logs --since=10m <container-id>
crictl logs --timestamp <container-id>
```

---

## nerdctl —— Docker 兼容 CLI

### 镜像管理

```bash
# 拉取/推送
nerdctl pull nginx:latest
nerdctl pull --platform amd64 alpine:latest

# 构建镜像
nerdctl build -t my-app:v1 .

# 列出镜像
nerdctl images
nerdctl image ls

# 导出/导入 & 标签
nerdctl tag nginx:latest my-registry/nginx:latest
nerdctl save -o nginx.tar nginx:latest
nerdctl load -i nginx.tar

# 删除
nerdctl rmi nginx:latest

# 查看镜像历史
nerdctl history nginx:latest
```

### 容器生命周期

```bash
# 运行
nerdctl run -d --name web -p 8080:80 nginx:latest
nerdctl run -it --rm alpine:latest sh
nerdctl run -d --restart=always --name app -e KEY=value my-app:v1

# 列出
nerdctl ps
nerdctl ps -a

# 启停
nerdctl stop web
nerdctl start web
nerdctl restart web

# 进入
nerdctl exec -it web /bin/bash

# 日志
nerdctl logs -f web
nerdctl logs --tail=50 web

# 删除
nerdctl rm web
nerdctl rm -f web                    # 强制删除运行中的容器
```

### Compose

```bash
nerdctl compose up -d
nerdctl compose down
nerdctl compose ps
nerdctl compose logs -f
```

### 网络和卷

```bash
nerdctl network ls
nerdctl network create mynet
nerdctl network inspect mynet

nerdctl volume ls
nerdctl volume create myvol
nerdctl volume inspect myvol
```

---

## 服务操作

### systemd 管理

```bash
systemctl status containerd
systemctl restart containerd
systemctl stop containerd

# 开机自启
systemctl enable containerd
systemctl disable containerd

# 查看日志
journalctl -u containerd -f
journalctl -u containerd --since "10 min ago"
journalctl -u containerd -n 100
```

---

## 配置管理

### 配置文件位置

```bash
# 默认配置
/etc/containerd/config.toml

# 生成默认配置
containerd config default > /etc/containerd/config.toml

# 查看当前运行的配置
containerd config dump
```

### 常用配置项

```toml
# /etc/containerd/config.toml

# 镜像加速（registry mirror）
[plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
  endpoint = ["https://mirror.example.com"]

# 私有仓库认证
[plugins."io.containerd.grpc.v1.cri".registry.configs."registry.example.com".auth]
  auth = "base64-encoded-credentials"

# pause 镜像
[plugins."io.containerd.grpc.v1.cri"]
  sandbox_image = "registry.k8s.io/pause:3.9"

# Cgroup 驱动
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
  SystemdCgroup = true
```

### 代理设置

```bash
# systemd 代理（创建 override）
mkdir -p /etc/systemd/system/containerd.service.d
cat > /etc/systemd/system/containerd.service.d/http-proxy.conf << 'EOF'
[Service]
Environment="HTTP_PROXY=http://proxy:8080"
Environment="HTTPS_PROXY=http://proxy:8080"
Environment="NO_PROXY=localhost,127.0.0.1,.local"
EOF

systemctl daemon-reload
systemctl restart containerd
```

---

## 日志与排错

### 容器日志位置

```bash
# containerd 自身日志
/var/log/containerd/

# 容器/Pod 日志（K8s 环境）
/var/log/pods/
/var/log/containers/

# CRI 日志等级
# 在 config.toml 中配置
[debug]
  level = "debug"
```

### 常见排错命令

```bash
# 查看 containerd 统计
ctr metric

# 使用 ctr 查看 K8s 命名空间的资源
ctr -n k8s.io task ls
ctr -n k8s.io container ls

# 检查运行时
crictl info | jq .config.containerd.runtimes

# 查看磁盘使用
du -sh /var/lib/containerd/

# 清理未使用的资源
crictl rmi --prune               # 清理未使用的镜像
ctr -n k8s.io snapshot ls | grep -v KEY | wc -l  # 快照数量
```

### containerd 存储路径

```bash
# 默认数据目录
/var/lib/containerd/

# 结构
/var/lib/containerd/
├── io.containerd.content.v1.content/   # 镜像层（blobs）
├── io.containerd.snapshotter.v1.overlayfs/  # 快照（容器层）
├── io.containerd.metadata.v1.bolt/     # 元数据数据库
└── tmpmounts/                          # 临时挂载
```

### 磁盘清理

```bash
# 清理所有未使用的镜像
crictl rmi --prune

# 删除已退出的容器
crictl rm $(crictl ps -a --state Exited -q)

# 清理快照（需停服）
systemctl stop containerd
rm -rf /var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/*
systemctl start containerd
```

---

## 小结：命令对照

| 操作 | ctr | crictl | nerdctl |
|------|-----|--------|---------|
| 拉取镜像 | `ctr i pull` | `crictl pull` | `nerdctl pull` |
| 列出镜像 | `ctr i ls` | `crictl images` | `nerdctl images` |
| 运行容器 | `ctr run` | — | `nerdctl run` |
| 列出运行容器 | `ctr task ls` | `crictl ps` | `nerdctl ps` |
| 执行命令 | `ctr task exec` | `crictl exec` | `nerdctl exec` |
| 查看日志 | — | `crictl logs` | `nerdctl logs` |
| 停止容器 | `ctr task kill` | `crictl stop` | `nerdctl stop` |
| 删除容器 | `ctr c rm` | `crictl rm` | `nerdctl rm` |
| 删除镜像 | `ctr i rm` | `crictl rmi` | `nerdctl rmi` |
