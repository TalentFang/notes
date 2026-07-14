# etcd 安装与核心配置详解

> 基于 maxs-ops 项目中 etcd 相关的 Ansible role 和 playbook 进行解读。

---

## 一、整体架构概览

项目中 etcd 相关的 Ansible role 共 6 个，覆盖了 etcd 集群的完整生命周期：

| Role | 职责 | 触发场景 |
|------|------|----------|
| `etcd` | 初始安装：证书签发、二进制分发、服务启动 | K8s 集群首次部署 |
| `etcd-add` | 动态扩容：将新节点加入已有集群 | 集群扩容 |
| `etcd-remove` | 动态缩容：从集群中移除节点 | 集群缩容 |
| `etcd-restore` | 数据恢复：从快照文件恢复数据 | 灾难恢复 |
| `etcd-refresh` | 证书刷新：重新签发证书并恢复数据 | 证书过期/轮换 |
| `etcd-conf-copy` | 证书分发：将证书复制到新节点 | 扩容前置 |

对应的 playbook 入口：

| Playbook | 说明 |
|----------|------|
| `playbooks/k8s/03.etcd.yml` | K8s 安装流程中的 etcd 安装（要求节点数 > 2） |
| `playbooks/etcd/etcd.yml` | 独立安装 etcd（无节点数限制） |
| `playbooks/add/add-etcd.yml` | 添加 etcd 节点（智能判断是否已安装） |
| `playbooks/add/add-etcd-tag.yml` | 通过 tag 触发 etcd 添加流程 |
| `playbooks/etcd/etcd-remove.yml` | 移除 etcd 节点 |
| `playbooks/backup/backup_etcd.yml` | 备份 etcd 快照 |
| `playbooks/restore/restore_etcd.yml` | 从快照恢复 |
| `playbooks/destroy/01.etcd.yml` | 销毁 etcd（停止服务 + 清理数据） |

---

## 二、目录结构与关键路径

```
/data/
├── comm/etcd/                          # etcd 工作根目录 (etcd_path)
│   ├── data/                           # 数据存储目录 (ETCD_DATA_DIR)
│   └── etcdback.db                     # 快照备份文件
├── asap/thirdSoft/                     # 安装包目录 (install_dir)
│   ├── etcd-v3.5.5-linux-{arch}.tar.gz # etcd 安装包
│   └── cfssl_1.6.0/                    # cfssl 证书工具
└── images/ingress/                     # 配置文件输出目录

/etc/etcd/cert/                         # 证书目录 (etcd_cert_dir)
├── ca.pem                              # CA 根证书
├── ca-key.pem                          # CA 私钥
├── ca-config.json                      # CA 签发配置
├── ca-csr.json                         # CA 证书签名请求
├── etcd.pem                            # etcd 服务证书
├── etcd-key.pem                        # etcd 服务私钥
├── etcd.csr                            # etcd 证书签名请求

/usr/local/bin/                         # 二进制目录 (bin_dir)
├── etcd                                # etcd server
├── etcdctl                             # etcd 客户端
├── etcdutl                             # etcd 管理工具
├── cfssl                               # 证书签发工具
├── cfssljson                           # cfssl JSON 输出
└── cfssl-certinfo                      # 证书信息查看
```

---

## 三、安装流程详解（etcd role）

`roles/etcd/tasks/main.yml` 的完整执行步骤：

### 3.1 目录准备

```yaml
- name: prepare some dirs
  file: name={{ item }} state=directory mode=0755
  with_items:
    - "{{ ETCD_DATA_DIR }}"    # /data/comm/etcd/data
    - "{{ install_dir }}"      # /data/asap/thirdSoft
    - "{{ etcd_cert_dir }}"    # /etc/etcd/cert
```

### 3.2 分发 cfssl 证书工具

将 `cfssl`、`cfssl-certinfo`、`cfssljson` 三个二进制文件从控制节点分发到所有 etcd 节点的 `/usr/local/bin/`。

### 3.3 证书签发流程

证书签发采用 **cfssl** 工具链，分为两步：

**第一步：签发 CA 根证书**（仅在第一个 etcd 节点执行）

```json
// ca-csr.json — CA 证书签名请求
{
  "CN": "etcd",
  "key": { "algo": "rsa", "size": 2048 },
  "names": [{ "C": "CN", "ST": "BeiJing", "L": "BeiJing", "O": "etcd", "OU": "system" }]
}
```

```json
// ca-config.json — CA 签发策略
{
  "signing": {
    "default": { "expiry": "876000h" },     // 默认 100 年过期
    "profiles": {
      "etcd": {
        "expiry": "876000h",                 // etcd profile 也是 100 年
        "usages": [
          "signing",                         // 允许签发子证书
          "key encipherment",                // RSA 密钥加密
          "server auth",                     // 服务端认证
          "client auth"                      // 客户端认证
        ]
      }
    }
  }
}
```

执行命令：
```bash
cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```

生成文件：`ca.pem`、`ca-key.pem`、`ca.csr`

**第二步：签发 etcd 服务证书**（仅在第一个 etcd 节点执行）

```json
// etcd-csr.json — etcd 证书签名请求（动态生成）
{
  "CN": "etcd",
  "hosts": [
    // 自动填充所有 etcd 节点的 IP
    {% for host in groups['etcd'] %}
    "{{ host }}",
    {% endfor %}
    "127.0.0.1"
  ],
  "key": { "algo": "rsa", "size": 2048 },
  "names": [{ "C": "CN", "ST": "BeiJing", "L": "BeiJing", "O": "etcd", "OU": "system" }]
}
```

执行命令：
```bash
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=etcd \
  etcd-csr.json | cfssljson -bare etcd
```

生成文件：`etcd.pem`、`etcd-key.pem`、`etcd.csr`

### 3.4 证书分发

1. 第一个节点通过 `fetch` 将证书拉取到控制机（如果控制机不在 etcd 组中）
2. 所有节点通过 `copy` 分发 `ca.pem`、`etcd.csr`、`etcd.pem`、`etcd-key.pem`

### 3.5 安装 etcd 二进制

- 解压 `etcd-v3.5.5-linux-{arch}.tar.gz`（arch 根据主机架构自动判断）
- 分发 `etcd`、`etcdctl`、`etcdutl` 到 `/usr/local/bin/`

### 3.6 创建 systemd 服务

服务文件模板 `etcd.service.j2` 生成到 `/etc/systemd/system/asap_etcd.service`，关键配置见下文第五节。

### 3.7 启动与健康检查

```
1. systemctl enable asap_etcd       # 开机自启
2. systemctl daemon-reload && restart # 重启服务
3. ionice -c2 -n0 -p $(pgrep etcd)  # 提升磁盘 IO 优先级（实时调度，最高优先级）
4. 轮询等待服务 active（最多 60 次，每次 10 秒）
5. 打印健康状态（endpoint health）
6. 打印 Leader 信息（endpoint status --cluster）
7. 验证所有节点 healthy（健康节点数 == etcd 组节点数）
8. 如果 k8s_master 不在 etcd 组中，fetch 证书到控制机
```

---

## 四、etcd 服务配置详解

### 4.1 systemd 服务参数

```ini
[Unit]
Description=Etcd Server
After=network.target network-online.target
Wants=network-online.target
RequiresMountsFor=/data

[Service]
Type=notify
WorkingDirectory=/data/comm/etcd
ExecStart=/usr/local/bin/etcd \
  --data-dir=/data/comm/etcd/data \
  --name=<hostname> \
  # ===== TLS 客户端通信 =====
  --cert-file=/etc/etcd/cert/etcd.pem \
  --key-file=/etc/etcd/cert/etcd-key.pem \
  --trusted-ca-file=/etc/etcd/cert/ca.pem \
  # ===== TLS 节点间通信 =====
  --peer-cert-file=/etc/etcd/cert/etcd.pem \
  --peer-key-file=/etc/etcd/cert/etcd-key.pem \
  --peer-trusted-ca-file=/etc/etcd/cert/ca.pem \
  # ===== 节点间通信 =====
  --initial-advertise-peer-urls=https://<IP>:2380 \
  --listen-peer-urls=https://<IP>:2380 \
  # ===== 客户端通信 =====
  --listen-client-urls=https://<IP>:2379,http://127.0.0.1:2379 \
  --advertise-client-urls=https://<IP>:2379 \
  # ===== 集群初始化 =====
  --initial-cluster-token=etcd-cluster-0 \
  --initial-cluster=<节点列表> \
  --initial-cluster-state=<new/existing> \
  # ===== 性能调优 =====
  --heartbeat-interval=500 \          # 心跳间隔 500ms
  --election-timeout=5000             # 选举超时 5000ms
Restart=on-failure
RestartSec=15
LimitNOFILE=65536
TimeoutStartSec=300s
IOSchedulingClass=0                   # I/O 调度类别：尽力而为
IOSchedulingPriority=0                # I/O 调度优先级：最高

[Install]
WantedBy=multi-user.target
```

### 4.2 各参数详解

#### 端口说明

| 端口 | 用途 | 协议 |
|------|------|------|
| 2379 | 客户端通信（etcdctl、应用访问） | HTTPS |
| 2380 | 节点间 peer 通信（集群内部同步） | HTTPS |

`listen-client-urls` 同时监听 HTTPS（外部访问）和 HTTP（localhost 本地访问），本地 HTTP 方便本机 etcdctl 操作无需证书。

#### TLS 双向认证

etcd 同时启用了**客户端 TLS** 和 **peer TLS**，两套使用相同的证书（`etcd.pem`）：
- 客户端通信：`--cert-file` + `--key-file` + `--trusted-ca-file`
- 节点间通信：`--peer-cert-file` + `--peer-key-file` + `--peer-trusted-ca-file`

#### 集群初始化参数

- `--initial-cluster-token=etcd-cluster-0`：集群标识 token，防止不同集群的数据混淆
- `--initial-cluster`：初始集群成员列表，格式 `hostname1=https://ip1:2380,hostname2=https://ip2:2380,...`
- `--initial-cluster-state`：
  - `new`：首次安装时使用
  - `existing`：扩容/恢复时使用（表示集群已存在）

#### 性能调优参数

- `--heartbeat-interval=500`：leader 向 follower 发送心跳的间隔（毫秒），默认 100ms，这里设为 500ms 减少网络开销
- `--election-timeout=5000`：follower 等待心跳超时后发起选举的时间（毫秒），默认 1000ms，这里设为 5000ms 适应大规模集群
- `ionice -c2 -n0`：设置 etcd 进程为 I/O 实时调度最高优先级，确保 etcd 在磁盘繁忙时仍能及时读写

---

## 五、集群变量生成逻辑

### 5.1 初始安装（roles/etcd/vars/main.yml）

```yaml
# ETCD_NODES: 用于 --initial-cluster 参数
# 格式: hostname1=https://ip1:2380,hostname2=https://ip2:2380
TMP_NODES: "{% for h in groups['etcd'] %}{{ hostvars[h]['hostname'] }}=https://{{ h }}:2380,{% endfor %}"
ETCD_NODES: "{{ TMP_NODES.rstrip(',') }}"

# CLUSTER_STATE: 首次安装为 new
CLUSTER_STATE: "new"

# ETCD_OUTER_NODES: 用于健康检查和状态查询
# 格式: https://ip1:2379,https://ip2:2379
TMP_OUTER_NODES: "{% for h in groups['etcd'] %}https://{{ h }}:2379,{% endfor %}"
ETCD_OUTER_NODES: "{{ TMP_OUTER_NODES.rstrip(',') }}"
```

### 5.2 扩容/恢复（roles/etcd-add/vars/main.yml）

```yaml
# ETCD_NODES 与初始安装相同
# CLUSTER_STATE 改为 existing（表示加入已有集群）
CLUSTER_STATE: "existing"
```

---

## 六、动态扩容流程（etcd-add role）

`roles/etcd-add/tasks/main.yml` 的执行步骤：

### 6.1 新节点安装

1. 准备目录
2. 从控制机分发证书（由 `etcd-conf-copy` role 预先完成）
3. 解压并分发 etcd 二进制
4. 生成 systemd 服务文件（`CLUSTER_STATE=existing`）
5. 启动服务并设置 IO 优先级

### 6.2 加入集群

```yaml
# 1. 排除新节点自身，获取其他节点 IP
set_fact: NODE_IPS="{% for host in groups['etcd'] %}{% if host == NODE_TO_ADD %}{% else %}{{ host }} {% endif %}{% endfor %}"

# 2. 遍历所有现有节点检查健康状态
shell: 'for ip in {{ NODE_IPS }};do
      ETCDCTL_API=3 etcdctl --endpoints=https://"$ip":2379 \
      --cacert=ca.pem --cert=etcd.pem --key=etcd-key.pem endpoint health;
    done'

# 3. 从健康检查结果中提取第一个健康节点
shell: 'echo -e "..." | grep "is healthy" | sed -n "1p" | cut -d: -f2 | cut -d/ -f3'

# 4. 在健康节点上执行 member add
shell: "ETCDCTL_API=3 etcdctl member add etcd-{{ NODE_TO_ADD }} \
      --peer-urls=https://{{ NODE_TO_ADD }}:2380"
```

### 6.3 智能判断（add-etcd.yml）

```yaml
# 先检查目标节点是否已安装 etcd
- name: 获取服务信息
  service_facts:

# 已安装：只更新证书并重启
- import_role: etcd-conf-copy
  when: "ansible_facts['services']['asap_etcd.service'] is defined"

# 未安装：创建用户 + 完整安装
- import_role: usradd-asap
  when: "ansible_facts['services']['asap_etcd.service'] is not defined"
- import_role: etcd-add
  when: "ansible_facts['services']['asap_etcd.service'] is not defined"
```

---

## 七、动态缩容流程（etcd-remove role）

`roles/etcd-remove/tasks/main.yml`：

```yaml
# 1. 获取当前节点在 etcd 集群中的 member ID
shell: "etcdctl member list -w table --cert ... --endpoints https://<leader>:2379
      | grep '<当前节点IP>' | awk '{print $2}'"

# 2. 调用 member remove 移除节点
shell: "etcdctl member remove <member_id> --cert ... --endpoints https://<leader>:2379"
```

注意：移除操作仅从集群元数据中删除节点，不会自动清理目标节点上的 etcd 数据和进程。

---

## 八、数据备份与恢复

### 8.1 备份（backup_etcd.yml）

```yaml
# 在第一个 etcd 节点执行快照
- shell: "etcdctl snapshot save /data/comm/etcd/etcdback.db"

# 拉取快照文件到控制机
- fetch:
    src: /data/comm/etcd/etcdback.db
    dest: /data/comm/etcd/
    flat: yes
```

### 8.2 恢复（etcd-restore role）

恢复脚本 `restore_etcd.sh.j2`：

```bash
#!/bin/bash
etcdctl snapshot restore /data/comm/etcd/etcdback.db \
  --name <hostname> \
  --initial-advertise-peer-urls "https://<IP>:2380" \
  --initial-cluster-token etcd-cluster-0 \
  --initial-cluster "<ETCD_NODES>" \
  --data-dir=/data/comm/etcd/data
```

恢复流程：
1. 停止 etcd 服务
2. 清空数据目录 `/data/comm/etcd/data`
3. 从控制机分发 `etcdback.db` 快照
4. 执行 `etcdctl snapshot restore` 恢复数据
5. 重启 etcd 服务并等待 active

---

## 九、证书刷新流程（etcd-refresh role）

`roles/etcd-refresh/tasks/main.yml` 是最复杂的操作，完整流程：

```
1. 备份当前数据 → etcdback.db
2. 停止 etcd 服务
3. 清空数据目录
4. 重新签发 CA 和 etcd 证书（全新 CA）
5. 分发新证书到所有节点
6. 更新 systemd 服务文件
7. 生成恢复脚本
8. 将快照从第一个节点拉取到控制机，再分发到其他节点
9. 在每个节点执行 snapshot restore（使用新证书的集群信息）
10. 启动服务并等待 active
```

注意：刷新证书会导致 CA 变更，因此需要完整的数据恢复流程来确保集群一致性。

---

## 十、销毁流程（destroy/01.etcd.yml）

```yaml
# 1. 停止并禁用服务
- service: name=asap_etcd state=stopped enabled=no

# 2. 清理所有文件
- file: name={{ item }} state=absent
  with_items:
    - /data/comm/etcd          # 数据目录
    - /etc/etcd/cert           # 证书目录
    - /etc/systemd/system/etcd.service  # 服务文件

# 3. 重载 systemd
- shell: systemctl daemon-reload

# 4. 从安装记录中删除 etcd 条目
- lineinfile:
    dest: /data/maxs-ops/inventory/install.txt
    regexp: "^playbooks/k8s/03.etcd.yml"
    state: absent
```

---

## 十一、关键设计要点总结

1. **全 TLS 加密**：客户端通信和节点间通信均使用 mTLS，证书由 cfssl 本地签发，有效期 100 年
2. **节点命名**：使用 Ansible inventory 中的 hostname（非 IP）作为 etcd 成员名，方便识别
3. **IO 优先级**：通过 `ionice` 将 etcd 设为实时 I/O 调度最高优先级，避免磁盘竞争导致集群抖动
4. **版本锁定**：etcd 版本锁定为 v3.5.5，架构通过 `architecture` 变量自动适配 amd64/arm64
5. **服务命名**：systemd 服务名为 `asap_etcd`（非默认的 `etcd`），避免与系统已有服务冲突
6. **扩容安全**：扩容前先检查目标节点是否已安装，已安装则只更新证书，避免重复部署
7. **健康验证**：安装/扩容后强制验证所有节点 healthy，不通过则标记失败
