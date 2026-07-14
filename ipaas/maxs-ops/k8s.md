# k8s-master 安装与核心配置详解

> 基于 maxs-ops 项目中 kube-master 相关的 Ansible role 和 playbook 进行解读。

---

## 一、整体架构概览

k8s-master 的安装涉及 3 个核心 role 和多个辅助 role：

| Role | 职责 | 触发场景 |
|------|------|----------|
| `kube-master` | 控制平面初始化：RPM 安装、kubeadm init、master join、kubeconfig 配置 | K8s 首次安装 |
| `kube-master-cert` | 证书刷新：重新签发 apiserver 证书并更新所有 master | 证书过期/轮换 |
| `kube-master-refresh` | IP 变更：更新所有配置中的 IP、重新初始化集群、节点重新加入 | 集群 IP 迁移 |

辅助 role：

| Role | 职责 |
|------|------|
| `cert99` | 90 天证书自动续期 |
| `kube-reserved-config` | kubelet 资源预留配置 |
| `kube-stat` | 集群状态检查 |
| `kube-status-check` | 详细状态诊断 |
| `kube-cp` | 集群控制平面操作（kubectl cp） |

Playbook 执行顺序：

```
01.prepare.yml          # 系统准备
02.runtime.yml          # 容器运行时（containerd/docker）
03.etcd.yml             # etcd 集群
04.nginx.yml            # Nginx 负载均衡（master HA）
05.keepalived.yml       # Keepalived VIP 漂移
06.kube-master.yml      # ★ k8s 控制平面初始化
07.kube-node.yml        # Worker 节点加入
08.calico.yml           # CNI 网络插件
...
13.cert99.yml           # 证书续期
```

---

## 二、目录结构与关键路径

```
/data/
├── images/                                 # install_dir
│   └── ingress/                            # ingress_install_dir — 核心配置目录
│       ├── kubeadm.conf                    # kubeadm 初始化配置
│       ├── kubeadm_cert.conf               # 证书刷新用配置
│       ├── token.txt                       # kubeadm init 输出（含 join 命令）
│       ├── master.sh                       # master join 命令
│       ├── node.sh                         # node join 命令
│       ├── calico_ip6.yaml                 # Calico CNI
│       ├── coredns.yaml                    # CoreDNS
│       ├── deploy.yaml                     # Ingress Nginx
│       ├── init-k8s.yaml / init-k8s-one.yaml  # ConfigMap
│       └── metrics-server.yaml             # Metrics Server
├── rpms/k8s/                               # k8s_install_dir — RPM 安装包
│   └── *.rpm                               # kubelet, kubeadm, kubectl 等
└── asap/thirdSoft/                         # 第三方工具

/etc/kubernetes/                            # K8s 核心配置目录
├── admin.conf                              # 管理员 kubeconfig
├── pki/                                    # PKI 证书目录
│   ├── ca.crt / ca.key                     # K8s CA
│   ├── apiserver.crt / apiserver.key       # API Server 证书
│   ├── apiserver-kubelet-client.crt/key    # API Server → kubelet 客户端证书
│   ├── front-proxy-ca.crt/key              # 前端代理 CA
│   ├── sa.key / sa.pub                     # ServiceAccount 签名密钥
│   └── etcd/                               # etcd 证书（本地模式）
├── pki_bak/                                # 证书备份（cert refresh 时）
└── manifests/                              # 静态 Pod 清单

/var/lib/kubelet/                           # kubelet 数据目录
/var/lib/kubelet-back/                      # kubelet 数据备份（refresh 时）
/root/.kube/config                          # root 用户 kubeconfig
/home/asap/.kube/config                     # asap 用户 kubeconfig
```

---

## 三、核心配置文件详解：kubeadm.conf.j2

这是 k8s 控制平面的**核心配置文件**，由 `kube-master` role 生成到 `/data/images/ingress/kubeadm.conf`。

### 3.1 InitConfiguration 部分

```yaml
apiVersion: kubeadm.k8s.io/v1beta3
kind: InitConfiguration
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: abcdef.0123456789abcdef      # bootstrap token（固定值，24 小时过期）
  ttl: 24h0m0s
  usages:
  - signing                             # 允许签署证书
  - authentication                      # 允许身份认证
localAPIEndpoint:
  advertiseAddress: {{ inventory_hostname }}   # 本机 IP
  bindPort: 6443
nodeRegistration:
  criSocket: /var/run/dockershim.sock   # CRI socket（使用 dockershim）
  name: {{ hostname }}                  # 节点名称（使用 hostname 变量）
  imagePullPolicy: Never                # 不拉取镜像（使用离线镜像）
  taints:
  - effect: PreferNoSchedule            # 软污点：尽量不调度到 master
    key: node-role.kubernetes.io/master
```

**关键设计点：**
- `imagePullPolicy: Never`：所有镜像预装到本地 registry，不从公网拉取
- `PreferNoSchedule`（软污点）而非 `NoSchedule`（硬污点）：master 允许但不鼓励运行业务 Pod
- bootstrap token 使用固定值 `abcdef.0123456789abcdef`，简化多 master 部署

### 3.2 ClusterConfiguration 部分

```yaml
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
kubernetesVersion: 1.23.0
clusterName: kubernetes
certificatesDir: /etc/kubernetes/pki
imageRepository: {{ REGISTRY_IP }}       # 镜像仓库地址
```

#### API Server 配置

```yaml
apiServer:
  certSANs:                             # 证书 SAN（Subject Alternative Names）
  - "{{ SERVER_VIP }}"                  # VIP 地址
  {% for host in groups['k8s_master'] %}
  - "{{ host }}"                        # 每个 master 的 IP
  - "{{ hostname }}"                    # 每个 master 的 hostname
  {% endfor %}
  timeoutForControlPlane: 4m0s          # 控制平面超时 4 分钟
```

**SERVER_VIP 逻辑：**
```yaml
SERVER_VIP: "{% if groups['vip']|length > 0 %}
               {{ groups['vip'][0] }}
             {% else %}
               {{ groups['k8s_master'][0] }}
             {% endif %}"
```
- 有 VIP 组：使用 Keepalived VIP
- 无 VIP 组：使用第一个 master 节点 IP

#### Control Plane Endpoint

```yaml
{% if groups['vip']|length > 0 %}
controlPlaneEndpoint: "{{ SERVER_VIP }}:16443"   # 有 VIP 时用 16443（nginx 代理端口）
{% else %}
controlPlaneEndpoint: "{{ SERVER_VIP }}:6443"    # 无 VIP 时用 6443（直连）
{% endif %}
```

**HA 架构说明：**
- 有 VIP 时：`VIP:16443 → Nginx(6443) → kube-apiserver(6443)`
  - Nginx 做 4 层负载均衡，监听 16443，转发到各 master 的 6443
  - Keepalived 提供 VIP 漂移
- 无 VIP 时：直接连接第一个 master 的 6443

#### etcd 配置

```yaml
etcd:
{% if groups['inner_vip']|length > 0 %}
  external:                              # 外部 etcd 模式
    endpoints:
    {% for host in groups['etcd'] %}
    - https://{{ host }}:2379
    {% endfor %}
    caFile: {{ etcd_cert_dir }}/ca.pem
    certFile: {{ etcd_cert_dir }}/etcd.pem
    keyFile: {{ etcd_cert_dir }}/etcd-key.pem
{% else %}
  local:                                 # 本地 etcd 模式
    imageTag: "3.5.1-0"
    dataDir: /var/lib/etcd
{% endif %}
```

**两种 etcd 模式：**
- `external`：etcd 独立部署在专用节点上（有 inner_vip 时）
- `local`：etcd 以静态 Pod 方式运行在 master 节点上

#### 网络配置

```yaml
networking:
  dnsDomain: cluster.local
  podSubnet: {{ CLUSTER_IPV4_IPV6 }}       # Pod 网段（支持 IPv4/IPv6 双栈）
  serviceSubnet: {{ SERVICE_IPV4_IPV6 }}   # Service 网段
```

---

## 四、安装流程详解（kube-master role）

`roles/kube-master/tasks/main.yml` 的完整执行步骤：

### 4.1 前置准备

```
1. 确保 /etc/resolv.conf 存在
2. 如果 k8s_master[0] 不在 etcd 组中：从操作机同步 etcd 证书到当前节点
3. 创建目录：install_dir, ingress_install_dir, k8s_install_dir, /etc/kubernetes/pki
```

### 4.2 安装 K8s 组件

```yaml
# 安装 RPM 包（kubelet, kubeadm, kubectl 等）
- shell: "rpm -Uvh {{ k8s_install_dir }}/*.rpm --nodeps --force"
  retries: 3, delay: 10                  # 最多重试 3 次

# 启用并启动 kubelet
- shell: systemctl enable kubelet
- shell: systemctl daemon-reload && systemctl restart kubelet
```

### 4.3 第一个 Master 初始化（kubeadm init）

```
1. 生成 kubeadm.conf 配置文件（仅第一个 master）
2. 备份原有 kubeadm 和 pki 目录
3. 替换 kubeadm 二进制为新版
4. 执行 kube_init.sh 脚本
```

**kube_init.sh 脚本逻辑：**
```bash
#!/bin/bash
# 首次尝试
kubeadm init --config kubeadm.conf --ignore-preflight-errors=SystemVerification &> token.txt
if [ $? -eq 0 ]; then exit 0; fi

# 失败后重试最多 3 次（每次 reset 后重新 init）
for num in `seq 1 3`; do
    kubeadm reset -f
    kubeadm init --config kubeadm.conf --ignore-preflight-errors=SystemVerification &> token.txt
    if [ $? -eq 0 ]; then exit 0; fi
    sleep 2
done

# 4 次都失败则退出
exit 1
```

### 4.4 等待与验证

```
1. 等待 /etc/kubernetes/admin.conf 生成（超时 300s）
2. 等待 6443 端口可用（超时 180s）
3. 等待 kube-apiserver Pod Ready（最多 60 次，每次 2s）
```

### 4.5 提取 Join Token

```yaml
# 从 token.txt 中提取 master join 命令
- shell: "grep -B2 '\-\-control-plane' token.txt > master.sh"

# 从 token.txt 中提取 node join 命令
- shell: "grep -A1 'kubeadm join' token.txt | tail -2 > node.sh"

# 拉取到操作机
- fetch: src=master.sh dest=master.sh flat=yes
- fetch: src=node.sh dest=node.sh flat=yes
```

### 4.6 分发 PKI 证书

从第一个 master 拉取并分发到所有节点：
```
ca.crt / ca.key              # K8s CA
sa.key / sa.pub              # ServiceAccount 签名密钥
front-proxy-ca.crt/key       # 前端代理 CA
admin.conf                   # 管理员 kubeconfig
```

### 4.7 其他 Master 加入集群（带重试）

```yaml
# 执行 master join（最多重试 4 次，每次失败 reset + 重新分发证书 + 重试）
- block:
  - shell: "sh master.sh"
  rescue:
    - block:
      - shell: "kubeadm reset -f"
      - copy: 分发 pki 证书
      - shell: "sh master.sh"          # 第 2 次尝试
      rescue:
        - block:
          - shell: "kubeadm reset -f"
          - copy: 分发 pki 证书
          - shell: "sh master.sh"      # 第 3 次尝试
          rescue:
            - block:
              - shell: "kubeadm reset -f"
              - copy: 分发 pki 证书
              - shell: "sh master.sh"  # 第 4 次尝试
```

### 4.8 配置收尾

```
1. 设置 KUBECONFIG 环境变量
2. 创建 /root/.kube/config（权限 asap:asap）
3. 创建 /home/asap/.kube/config（asap 用户）
4. 移除 master 污点（允许 master 调度 Pod）
5. 轮询等待 kubelet active
6. 删除 master.sh（安全清理 join token）
7. 配置 kubelet 日志轮转
```

---

## 五、变量详解

### 5.1 defaults/main.yml 核心变量

```yaml
# 安装路径
install_dir: /data/images
k8s_install_dir: /data/rpms/k8s
ingress_install_dir: /data/images/ingress

# VIP 逻辑：有 vip 组用 vip，否则用第一个 master
SERVER_VIP: "{% if groups['vip']|length > 0 %}
               {{ groups['vip'][0] }}
             {% else %}
               {{ groups['k8s_master'][0] }}
             {% endif %}"

# etcd 集群节点列表
TMP_NODES: "{% for h in groups['etcd'] %}
              {{ hostvars[h]['hostname'] }}=https://{{ h }}:2380,
            {% endfor %}"
ETCD_NODES: "{{ TMP_NODES.rstrip(',') }}"

# 镜像仓库地址（有 inner_vip 且多 registry 时用 vip:5001，否则用第一个 registry:5000）
REGISTRY_IP: "{% if groups['inner_vip']|length > 0 and groups['registry']|length > 1 %}
                {{ groups['inner_vip'][0] }}:5001/prod
              {% else %}
                {{ groups['registry'][0] }}:5000/prod
              {% endif %}"

# master join 过滤关键字
master_filter: \-\-control-plane

# kubelet 日志配置
kubelet_log_dir: /var/log/kubelet
freq: daily       # 轮转频率
rot: 7            # 保留份数
```

---

## 六、证书刷新流程（kube-master-cert role）

当 API Server 证书过期或需要轮换时使用。

### 6.1 流程概览

```
1. 生成 kubeadm_cert.conf（与 kubeadm.conf 类似，但包含 kube-reserved 配置）
2. 第一个 master：备份旧 apiserver 证书 → 重新生成 apiserver 证书
3. 其他 master：备份 pki 目录 → 从第一个 master 复制新证书 → 重新生成 apiserver 证书
4. 所有非新增 master：重新生成 apiserver-kubelet-client 和 front-proxy-client 证书
5. 杀死 kube-apiserver 容器（docker kill）触发自动重启
```

### 6.2 kubeadm_cert.conf 与 kubeadm.conf 的区别

`kubeadm_cert.conf` 额外包含 kubelet 资源预留配置：

```yaml
nodeRegistration:
  kubeletExtraArgs:
    kube-reserved: "cpu=200m,memory=512Mi,ephemeral-storage=2Gi"
    system-reserved: "cpu=300m,memory=1Gi,ephemeral-storage=4Gi"
    eviction-hard: "memory.available<300Mi,nodefs.available<10%,nodefs.inodesFree<5%,imagefs.available<10%"
    eviction-soft: "memory.available<1Gi,nodefs.available<15%,imagefs.available<15%"
    eviction-soft-grace-period: "memory.available=2m,nodefs.available=3m,imagefs.available=3m"
```

### 6.3 信创系统特殊处理

```yaml
# 仅在信创系统上重新生成 apiserver 证书
- shell: "kubeadm init phase certs apiserver --config ..."
  when: "ansible_distribution in ['Kylin Linux Advanced Server', 'openEuler', 'UOS Server 25']"
```

信创系统（麒麟、openEuler、UOS）上需要额外重新生成 apiserver 证书，可能是因为这些系统的 OpenSSL/TLS 库行为差异。

---

## 七、IP 变更流程（kube-master-refresh role）

这是最复杂的流程，用于集群整体 IP 迁移（如 VIP 切换、网段变更）。

### 7.1 单 master 模式（k8s_master 数量 == 1）

直接执行 `ip_modify.sh` 脚本，完整重建集群：

```bash
re_init() {
    # 1. 重置集群
    kubeadm reset -f
    rm -rf /root/.kube/config /data/asap/.kube/config /home/asap/.kube/config

    # 2. 用更新后的 kubeadm.conf 重新初始化
    kubeadm init --config /data/images/ingress/kubeadm.conf

    # 3. 配置 kubeconfig
    cp /etc/kubernetes/admin.conf /root/.kube/config
    cp /etc/kubernetes/admin.conf /data/asap/.kube/config
    cp /etc/kubernetes/admin.conf /home/asap/.kube/config

    # 4. 去除 master 污点
    kubectl taint node --all node-role.kubernetes.io/master-

    # 5. 部署网络插件和基础组件
    kubectl apply -f calico_ip6.yaml
    kubectl apply -f init-k8s.yaml

    # 6. 重启 kubelet
    systemctl restart kubelet

    # 7. 部署 Ingress 和业务
    kubectl apply -f deploy.yaml
    kubectl apply -f coredns.yaml
    # ... 部署所有业务服务
}
```

### 7.2 多 master 模式（k8s_master 数量 > 1）

分三个阶段执行：

**阶段一：配置更新（第一个 master 执行）**

```
1. 关闭防火墙
2. 执行 refresh_business_yaml.sh — 替换所有业务 YAML 中的旧 IP 为新 VIP
3. 执行 refresh_docker_images.sh — 替换所有 Docker 镜像 tag 中的旧 IP 为新 VIP
```

**阶段二：第一个 Master 重建**

```
1. 停止 kubelet
2. 执行 backup_k8s.sh — 备份 /etc/kubernetes → /etc/kubernetes-back，/var/lib/kubelet → /var/lib/kubelet-back
3. 恢复 pki 证书目录，删除 apiserver 和 etcd peer 证书（需要重新生成）
4. 强制杀死 kube-scheduler、kube-controller-manager、kube-apiserver 进程
5. 从 etcd 节点拉取并分发 etcd 证书
6. 执行 kubeadm init 重新初始化
7. 提取 join 命令
8. 删除所有其他 master 和 node 节点
9. 重启 Calico 网络插件
10. 生成 certificate key（供其他 master join）
11. 分发 join 脚本到所有节点
```

**阶段三：其他节点重新加入**

```
其他 master：
1. kubeadm reset -f
2. 执行 join 脚本（带 --control-plane --certificate-key）

其他 node：
1. kubeadm reset -f
2. 执行 join 脚本（不带 control-plane 参数）
```

**阶段四：收尾**

```
1. 部署 coredns、metrics-server
2. 去除 master 污点
3. 部署 Ingress
4. 配置 kubeconfig（root + asap 用户）
5. 重启 kube-state-metrics
6. 开启防火墙
```

### 7.3 IP 替换脚本详解

**refresh_business_yaml.sh** — 替换业务 YAML 中的镜像地址：
```bash
new_ip="{{ SERVER_VIP }}:"
# 遍历 /data/asap/deploy/ 下所有服务
for servicename in $(ls /data/asap/deploy/); do
    oyaml=${servicepath}/${servicename}/deployment/${servicename}.k8s.yaml
    sed -i "s/[0-9]\+\.[0-9]\+\.[0-9]\+\.[0-9]\+:/$new_ip/g" "$oyaml"
done
# 同样处理 /data/asap/deploy-*/ 额外部署目录
```

**refresh_docker_images.sh** — 替换本地 Docker 镜像 tag：
```bash
new_ip="{{ SERVER_VIP }}:"
for image in $(docker images --format "{{.Repository}}:{{.Tag}}"); do
    new_image=$(echo "$image" | sed "s/[0-9]\+\.[0-9]\+\.[0-9]\+\.[0-9]\+:/$new_ip/g")
    docker image tag "$image" "$new_image"
done
```

**refresh_k8s_conf.sh** — 替换 kubeadm 配置中的 IP：
```bash
# 读取旧 IP 列表文件，逐一替换为新 IP
while read line; do
    sed -i "s/$line/${new_ip}/g" kubeadm_cert.conf
done < k8s_master_ip_back
# 同样处理 /data/images/ingress/ 下所有配置文件
```

---

## 八、Kubeconfig 配置流程

所有 master 节点执行的统一 kubeconfig 配置：

```yaml
# 1. 设置环境变量
- shell: echo "export KUBECONFIG=/etc/kubernetes/admin.conf" >> ~/.bash_profile

# 2. 设置 admin.conf 权限（所有用户可读）
- shell: chmod 666 /etc/kubernetes/admin.conf

# 3. root 用户 kubeconfig
- file: name=/root/.kube state=directory owner=asap group=asap
- copy: src=/etc/kubernetes/admin.conf dest=/root/.kube/config

# 4. asap 用户 kubeconfig
- shell: |
    mkdir -p /home/asap/.kube/
    cp /root/.kube/config /home/asap/.kube/
    chown -R asap:asap /home/asap
    echo "export KUBECONFIG=/home/asap/.kube/config" >> ~/.bash_profile
    echo "source <(kubectl completion bash)" >> ~/.bashrc
```

---

## 九、Master 污点处理

```yaml
# 安装时设置软污点（kubeadm.conf 中）
taints:
- effect: PreferNoSchedule
  key: node-role.kubernetes.io/master

# 安装完成后去除污点（允许 master 调度 Pod）
- shell: kubectl taint node --all node-role.kubernetes.io/master-
```

项目选择 **去除 master 污点**，允许在 master 节点上调度业务 Pod。这在小规模集群中常见，可以充分利用 master 节点的资源。

---

## 十、日志管理

### 10.1 kubelet 日志过滤（rsyslog）

```conf
# /etc/rsyslog.d/ignore-kubelet.conf
if $programname == 'kubelet' then {
    /var/log/kubelet/kubelet.log
    stop
}
```

将 kubelet 日志从系统日志中分离，写入独立文件。

### 10.2 kubelet 日志轮转（logrotate）

```conf
# /etc/logrotate.d/kubelet
/var/log/kubelet/*.log {
    daily                  # 每天轮转
    missingok
    rotate 7               # 保留 7 份
    compress
    delaycompress
    notifempty
    create 0640 root root
    sharedscripts
    postrotate
        systemctl restart rsyslog >/dev/null 2>&1 || true
    endscript
}
```

---

## 十一、关键设计要点总结

1. **HA 架构**：Nginx（4 层负载）+ Keepalived（VIP 漂移），`VIP:16443 → Nginx:16443 → apiserver:6443`
2. **离线部署**：`imagePullPolicy: Never` + 本地 registry（`REGISTRY_IP`），所有镜像预装
3. **重试机制**：kubeadm init 最多重试 4 次（reset + 重试），master join 最多重试 4 次
4. **双 etcd 模式**：有 inner_vip 时用外部 etcd，否则用本地静态 Pod etcd
5. **etcd 证书同步**：如果 k8s_master[0] 不在 etcd 组中，需要额外同步 etcd 证书
6. **双用户 kubeconfig**：同时配置 root 和 asap 用户的 kubeconfig 和命令补全
7. **去污点策略**：去除 master 污点，允许 master 调度业务 Pod
8. **日志分离**：kubelet 日志独立存放 + logrotate 轮转，避免污染系统日志
9. **信创适配**：证书刷新时对麒麟/openEuler/UOS 做特殊处理
10. **IP 迁移**：完整的 IP 变更流程，包括配置替换、镜像重 tag、集群重建、节点重新加入
