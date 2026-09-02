# 新安装版本升级: Docker → Containerd + 组件版本更新

## Context

将 Ansible 安装代码从旧版本（Docker + K8s 1.23 + etcd 3.5.5 + Registry 2.8.1）更新为新版本。**新集群安装代码变更**，不涉及线上集群升级。

## 目标版本

| 组件 | 当前版本 | 目标版本 |
|------|---------|---------|
| 容器运行时 | Docker 20.10.x (tgz) | containerd (tgz) |
| Kubernetes | 1.23.0 | **1.36.2** |
| etcd (外部) | 3.5.5 | **3.6.x** |
| Registry | 2.8.1 | **3.1.1** |
| Calico | 3.24.5 | **3.29.x** |
| 自定义 kubeadm | 保留 | 保留 (需提供 1.36.2 版本) |

---

## 一、新增角色

### 1.1 `roles/containerd` — standalone containerd

文件结构:
```
roles/containerd/
  defaults/main.yml
  tasks/main.yml
  templates/config.toml.j2
  vars/main.yml
```

`defaults/main.yml`:
```yaml
install_dir: /data/asap/thirdSoft/containerd
SERVER_VIP: "{% if groups['inner_vip']|length > 0 %}{{ groups['inner_vip'][0] }}{% else %}{{ groups['k8s_master'][0] }}{% endif %}"
REGISTRY_ENDPOINT: "{% if groups['inner_vip']|length > 0 and groups['registry']|length > 1 %}{{ groups['inner_vip'][0] }}:5001{% else %}{{ groups['registry'][0] }}:5000{% endif %}"
```

`tasks/main.yml`:
1. 检查 containerd 是否已安装 (`systemctl is-active containerd`)
2. 创建目录: `/etc/containerd`, `install_dir`
3. 删除旧的二进制文件
4. 解压 `containerd-*.tgz` → `/usr/local/bin/`
5. 生成默认配置: `containerd config default > /etc/containerd/config.toml`
6. 通过 template 覆盖关键配置:
   - `SystemdCgroup = true`
   - `sandbox_image = "{{ REGISTRY_ENDPOINT }}/pause:<version>"`
   - registry mirrors 指向本地仓库 (非安全 HTTP)
7. 部署 systemd unit
8. `systemctl daemon-reload && enable containerd && restart containerd`
9. 轮询等待 containerd 运行
10. 设置 `ingress_add` fact（与 docker 角色保持一致）

`templates/config.toml.j2` 关键配置:
```toml
[plugins."io.containerd.grpc.v1.cri"]
  sandbox_image = "{{ REGISTRY_ENDPOINT }}/pause:<version>"

[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
  SystemdCgroup = true

[plugins."io.containerd.grpc.v1.cri".registry.mirrors]
  [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
    endpoint = ["http://{{ REGISTRY_ENDPOINT }}"]
  [plugins."io.containerd.grpc.v1.cri".registry.configs."{{ REGISTRY_ENDPOINT }}".tls]
    insecure_skip_verify = true
```

### 1.2 `roles/crictl` — 调试工具

```yaml
# tasks/main.yml
- 解压 crictl-*.tar.gz → /usr/local/bin/crictl
- 创建 /etc/crictl.yaml:
    runtime-endpoint: unix:///run/containerd/containerd.sock
    image-endpoint: unix:///run/containerd/containerd.sock
```

### 1.3 `roles/transfer-containerd` — 分发二进制

`defaults/main.yml`:
```yaml
install_dir: /data/asap/thirdSoft/containerd
```

`tasks/main.yml`:
1. 创建 `install_dir`
2. copy `containerd-*.tgz` + `crictl-*.tar.gz` 到目标节点
3. 校验文件存在

---

## 二、修改现有角色

### 2.1 `roles/kube-master/templates/kubeadm.conf.j2`

```diff
- criSocket: /var/run/dockershim.sock
+ criSocket: unix:///run/containerd/containerd.sock

- kubernetesVersion: 1.23.0
+ kubernetesVersion: 1.36.2

# K8s 1.27+ → kubeadm API 版本升级
- apiVersion: kubeadm.k8s.io/v1beta3  # (两处)
+ apiVersion: kubeadm.k8s.io/v1beta4  # (两处, 或 v1beta5)

# 内部 etcd 镜像标签 (外部 etcd 不触发此分支)
- imageTag: "3.5.1-0"
+ imageTag: "<K8s 1.36 对应的 etcd 镜像标签>"
```

### 2.2 `roles/kube-master-cert/templates/kubeadm_cert.conf.j2`

与 `kubeadm.conf.j2` 相同的变更。

### 2.3 `roles/kube-master/templates/kube_init.sh.j2`

```diff
# 两处 kubeadm init 命令都需要加 --cri-socket
- kubeadm init --config {{ ingress_install_dir }}/kubeadm.conf ...
+ kubeadm init --config {{ ingress_install_dir }}/kubeadm.conf --cri-socket=unix:///run/containerd/containerd.sock ...
```

### 2.4 `roles/etcd/tasks/main.yml`

```diff
# 版本变量化
- tar -xf etcd-v3.5.5-linux-{{ architecture }}.tar.gz
+ tar -xf etcd-v{{ ETCD_VERSION }}-linux-{{ architecture }}.tar.gz

- copy: src={{ install_dir }}/etcd-v3.5.5-linux-{{ architecture }}/{{ item }}
+ copy: src={{ install_dir }}/etcd-v{{ ETCD_VERSION }}-linux-{{ architecture }}/{{ item }}
```

`defaults/main.yml` 添加:
```yaml
ETCD_VERSION: "3.6.x"  # 具体版本号
```

### 2.5 `roles/etcd-add/tasks/main.yml`

与 `roles/etcd/tasks/main.yml` 相同的版本变量化。

### 2.6 `roles/etcd-refresh/tasks/main.yml`

与 `roles/etcd/tasks/main.yml` 相同的版本变量化。

### 2.7 `roles/registry/tasks/main.yml`

```diff
# 变量化版本号
- # 解压 registry_2.8.1_linux_{{ architecture }}.tar.gz
+ # 解压 registry_{{ REGISTRY_VERSION }}_linux_{{ architecture }}.tar.gz
```

`defaults/main.yml` 添加:
```yaml
REGISTRY_VERSION: "3.1.1"
```

### 2.8 `roles/registry/templates/config.yml.j2`

Registry v3.1.1 可能有配置格式变更。需对比后决定具体改动:
- 可能新增 `validation`、`compatibility` 相关配置
- S3 driver 参数基本不变
- 需实测确认

### 2.9 `roles/image-handle/templates/version.cnf.j2`

```diff
- calico/typha:v3.24.5#typha:v3.24.5
+ calico/typha:v3.29.x#typha:v3.29.x
- calico/kube-controllers:v3.24.5#kube-controllers:v3.24.5
+ calico/kube-controllers:v3.29.x#kube-controllers:v3.29.x
- calico/cni:v3.24.5#cni:v3.24.5
+ calico/cni:v3.29.x#cni:v3.29.x
- calico/node:v3.24.5#node:v3.24.5
+ calico/node:v3.29.x#node:v3.29.x

- kube-apiserver:v1.23.0
+ kube-apiserver:v1.36.2
- kube-scheduler:v1.23.0
+ kube-scheduler:v1.36.2
- kube-proxy:v1.23.0
+ kube-proxy:v1.36.2
- kube-controller-manager:v1.23.0
+ kube-controller-manager:v1.36.2

- etcd:3.5.1-0
+ etcd:<K8s 1.36 对应版本>

- coredns:v1.8.6
+ coredns:<K8s 1.36 对应版本>

- pause:3.6
+ pause:<K8s 1.36 对应版本>

# 其他组件版本确认后更新:
# ingress-nginx, metrics-server, kube-state-metrics
```

同样变更应用到 `version-arm.cnf.j2`。

### 2.10 `roles/image-handle/templates/image_handle.sh.j2`

```diff
# docker → ctr 命令
- docker load -i xxx.tar
+ ctr -n k8s.io image import xxx.tar

- docker tag src dst
+ ctr -n k8s.io image tag src dst

- docker push image
+ ctr -n k8s.io image push --plain-http image

- docker pull image
+ ctr -n k8s.io image pull --plain-http image
```

### 2.11 `roles/image-handle/templates/image_handle_bak.sh.j2`

与 `image_handle.sh.j2` 相同的 `docker` → `ctr -n k8s.io` 变更。

### 2.12 `tools/registry_tool.sh`

检测可用运行时 (docker / ctr) 并使用对应 CLI。

### 2.13 inventory 版本变量

在 `[all:vars]` 添加:
```ini
K8S_VERSION=1.36.2
ETCD_VERSION=3.6.x
REGISTRY_VERSION=3.1.1
CONTAINERD_VERSION=1.7.x  # 或对应 K8s 1.36 兼容版本
```

---

## 三、Playbook 变更

### 3.1 `playbooks/k8s/02.runtime.yml`

```diff
  - hosts:
    - k8s_master
    - k8s_node
    - oper_master
-   - aiagent
  roles:
-   - docker
+   - containerd
+   - crictl
```

### 3.2 `playbooks/transfer/12.transfer-runtime.yml`

```diff
- 检查 docker.service → 调用 transfer-docker
+ 检查 containerd.service → 调用 transfer-containerd
```

### 3.3 `playbooks/destroy/02.docker.yml` → 新增 `02.containerd.yml`

```yaml
- hosts: k8s, oper_master
  tasks:
    - 停止 containerd
    - 删除 /etc/containerd, /var/lib/containerd
    - 删除 /usr/local/bin/{containerd,containerd-shim,ctr,runc,crictl}
    - 删除 systemd unit
    - 更新 install.txt
```

### 3.4 `playbooks/destroy/02.k8s.yml`

```diff
- rm -rf /run/containerd/*
+ rm -rf /run/containerd/* /var/lib/containerd/*
```

### 3.5 其他 playbook

- `playbooks/add/add-kube-master.yml` — 检查是否引用 docker 角色
- `playbooks/add/add-kube-node.yml` — 同上
- `playbooks/upgrade/02.upgrade_docker.yml` — 不再适用，后续考虑新 upgrade playbook
- `playbooks/single_upgrade_cluster/` — 检查是否有相关引用

---

## 四、实施顺序

### 阶段一：运行时层
1. 创建 `roles/containerd`
2. 创建 `roles/crictl`
3. 创建 `roles/transfer-containerd`
4. 修改 `playbooks/k8s/02.runtime.yml`
5. 修改 `playbooks/transfer/12.transfer-runtime.yml`
6. 新增 `playbooks/destroy/02.containerd.yml`
7. 修改 `playbooks/destroy/02.k8s.yml`

### 阶段二：K8s 控制面
8. 修改 `roles/kube-master/templates/kubeadm.conf.j2`
9. 修改 `roles/kube-master-cert/templates/kubeadm_cert.conf.j2`
10. 修改 `roles/kube-master/templates/kube_init.sh.j2`

### 阶段三：组件版本 + 镜像
11. 修改 `roles/etcd` (版本变量化)
12. 修改 `roles/etcd-add`
13. 修改 `roles/etcd-refresh`
14. 修改 `roles/registry` (版本变量化 + config)
15. 修改 `roles/image-handle/templates/version.cnf.j2`
16. 修改 `roles/image-handle/templates/version-arm.cnf.j2`
17. 修改 `roles/image-handle/templates/image_handle.sh.j2`
18. 修改 `roles/image-handle/templates/image_handle_bak.sh.j2`
19. 修改 `tools/registry_tool.sh`

### 阶段四：清理 + 收尾
20. 更新 inventory 版本变量
21. 更新 `playbooks/add/` 相关 playbook
22. 旧文件标记过时（暂不删除: `roles/docker/`, `roles/transfer-docker/` 等）

---

## 五、文件变更总清单

### 新增 (~15 文件)
- `roles/containerd/defaults/main.yml`
- `roles/containerd/tasks/main.yml`
- `roles/containerd/templates/config.toml.j2`
- `roles/containerd/vars/main.yml`
- `roles/crictl/tasks/main.yml`
- `roles/crictl/templates/crictl.yaml.j2`
- `roles/transfer-containerd/defaults/main.yml`
- `roles/transfer-containerd/tasks/main.yml`
- `roles/transfer-containerd/vars/main.yml`
- `playbooks/destroy/02.containerd.yml`

### 修改 (~20+ 文件)
- `roles/kube-master/templates/kubeadm.conf.j2`
- `roles/kube-master/templates/kube_init.sh.j2`
- `roles/kube-master/defaults/main.yml`
- `roles/kube-master-cert/templates/kubeadm_cert.conf.j2`
- `roles/etcd/tasks/main.yml`
- `roles/etcd/defaults/main.yml`
- `roles/etcd-add/tasks/main.yml`
- `roles/etcd-refresh/tasks/main.yml`
- `roles/registry/tasks/main.yml`
- `roles/registry/defaults/main.yml`
- `roles/registry/templates/config.yml.j2`
- `roles/image-handle/templates/version.cnf.j2`
- `roles/image-handle/templates/version-arm.cnf.j2`
- `roles/image-handle/templates/image_handle.sh.j2`
- `roles/image-handle/templates/image_handle_bak.sh.j2`
- `tools/registry_tool.sh`
- `playbooks/k8s/02.runtime.yml`
- `playbooks/transfer/12.transfer-runtime.yml`
- `playbooks/destroy/02.k8s.yml`
- `inventory/hosts` (及所有变体)
- `playbooks/add/add-kube-master.yml`
- `playbooks/add/add-kube-node.yml`

### 过时保留 (不删除)
- `roles/docker/`
- `roles/docker-refresh/`
- `roles/transfer-docker/`
- `playbooks/upgrade/02.upgrade_docker.yml`
- `playbooks/destroy/02.docker.yml`

---

## 六、containerd 命名空间注意事项

- CRI 插件使用 `k8s.io` 命名空间
- `ctr` 默认使用 `default` 命名空间（**kubelet 不可见**）
- 所有镜像操作必须加 `-n k8s.io`
- `crictl` 自动使用正确的命名空间，推荐用于调试

---

## 七、待确认事项

以下在实施时需进一步确认:

1. **K8s 1.36.2 kubeadm API 版本**: `v1beta4` or `v1beta5`？
2. **内部 etcd 镜像标签**: K8s 1.36 内置 etcd 版本
3. **pause 镜像版本**: K8s 1.36 默认 pause 版本
4. **CoreDNS 版本**: K8s 1.36 默认 CoreDNS 版本
5. **Calico 3.29.x 具体版本号** 和镜像 tag
6. **Registry v3.1.1 config.yml 差异**: 与 v2 的兼容性
7. **自定义 kubeadm 二进制**: `/data/images/ingress/kubeadm` 需提供 1.36.2 版本
8. **ARM64 兼容性**: 所有组件 ARM64 二进制/镜像可用性
9. **containerd/K8s 版本兼容矩阵**: 确认 containerd 版本
