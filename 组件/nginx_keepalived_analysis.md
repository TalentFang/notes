# Nginx 和 Keepalived 部署架构分析

## 一、部署流程

### 1. Nginx / BES 部署

**部署入口**: `playbooks/k8s/04.nginx.yml`

**负载均衡器选择**: 通过 `LOAD_BALANCE_TYPE` 变量控制
```ini
# inventory/hosts
[all:vars]
LOAD_BALANCE_TYPE=nginx  # 或 bes
```

**部署流程**:
1. **文件传输阶段** (`playbooks/transfer/19.transfer-nginx.yml`)
   - 根据 `LOAD_BALANCE_TYPE` 选择调用 `transfer-nginx` 或 `transfer-bes` role
   - 从 `/data/asap/thirdSoft/` 解压到 `/data/comm/`
   - 仅在服务未安装时执行（通过 `service_facts` 检测）

| 特性 | Nginx | BES (BWS) |
|------|-------|-----------|
| 服务名 | `asap_nginx` | `asap_bes` |
| 运行目录 | `/data/comm/nginx` | `/data/comm/bes` |
| 配置文件 | `nginx.conf` | `bws.conf` |
| 启动命令 | `sbin/nginx` | `bin/bws.sh` |
| Role | `nginx` | `bes` |

**部署流程**:
1. **文件传输阶段** (`playbooks/transfer/19.transfer-nginx.yml`)
   - 调用 `transfer-nginx` role
   - 从 `/data/asap/thirdSoft/nginx.tar.gz` 解压到 `/data/comm/`
   - 仅在服务未安装时执行（通过 `service_facts` 检测）

2. **安装配置阶段** (`roles/nginx/tasks/main.yml`)
   - 创建 `/data/asap/keys` 目录
   - 分发 libmodsecurity 库（条件：非麒麟/openEuler/UOS 系统）
   - 分发 SSL 证书（ssa.crt, ssa.key.unsecure）
   - 复制业务配置 `nginx_conf.txt`
   - 渲染配置文件：
     - 多节点模式 (`groups['nginx']|length > 1`) → `nginx.conf.j2` (369行)
     - 单节点模式 (`groups['nginx']|length == 1`) → `nginx_single.conf` (31行)
   - 部署 systemd unit 文件
   - 配置 logrotate
   - 启用并启动 `asap_nginx` 服务

**关键路径**:
- 安装目录: `/data/asap/thirdSoft`
- 运行目录: `/data/comm/nginx`
- 配置目录: `/data/comm/nginx_conf`

### 2. Keepalived 部署

**部署入口**: `playbooks/k8s/05.keepalived.yml`

**部署流程**:
1. **文件传输阶段** (`playbooks/transfer/20.transfer-keepalived.yml`)
   - 调用 `transfer-keepalived` role
   - 创建 `/etc/keepalived` 和安装目录
   - 分发 keepalived RPM 包
   - 仅在服务未安装时执行

2. **安装配置阶段** (`roles/keepalived/tasks/main.yml`)
   - 自动检测网络接口 (`LB_IF`, `LB_IF_OUTER`)
   - 安装 keepalived RPM 包（带重试机制）
   - 处理 openEuler/RHEL 的软链接兼容性
   - 渲染统一配置 `keepalived.conf.j2`
   - 部署健康检查脚本（check_nginx.sh, check_bes.sh, check_flume.sh, check_lsyncd.sh）
   - 部署通知脚本（notify_master.sh, notify_backup.sh, notify_fault.sh）
   - 部署 VIP 监控脚本（vip_watcher.sh + systemd timer）
   - 启用并启动 keepalived 服务

**Keepalived 特性**:
- 支持 3 个 VRRP 实例：VI_1 (nginx VIP), VI_2 (flume VIP), VI_3 (lsyncd VIP)
- 支持 unicast 模式（`PARAM_KEEPALIVED_UNICAST=1`）
- VIP 监控：每 5 分钟检查 VIP 是否存在，丢失则重启 keepalived
- 健康检查：根据 `LOAD_BALANCE_TYPE` 选择检查 nginx 或 bes

## 二、升级流程（当前状态）

**现状**: **没有专门的 nginx/keepalived 升级 playbook**

**现有升级 playbook** (`playbooks/upgrade/`):
- `01.upgrade_rpms.yml` - RPM 包升级
- `02.upgrade_docker.yml` - Docker 升级
- `03.upgrade_redis_data.yml` - Redis 数据升级

**当前升级方式**: 通过 transfer + add 流程实现
1. 传输新版本文件
2. 重新运行角色任务

## 三、扩缩容流程

### 1. 扩容 (Scale-out)

**入口**: `playbooks/add/add-nginx.yml`

**流程**:
```
节点类型判断
├── 已安装节点 (asap_nginx/asap_bes.service 已存在)
│   ├── 重新运行 add_nginx-tag
│   ├── 重新运行 add-flume-tag
│   ├── 刷新 keepalived_exporter 配置
│   └── 启用 nginx/bes
└── 新节点 (asap_nginx/asap_bes.service 不存在)
    ├── 调用 usradd-asap (创建用户)
    ├── 调用 nginx/bes role
    ├── 调用 keepalived role
    └── 调用 keepalived_exporter role
```

### 2. 缩容 (Scale-in)

**入口**: `playbooks/destroy/11.nginx.yml` 和 `playbooks/destroy/13.keepalived.yml`

**Nginx/BES 缩容流程**:
1. 停止 asap_nginx 或 asap_bes 服务（取决于当前负载均衡器类型）
2. 删除配置文件、systemd unit、logrotate 配置
3. 从 install.txt 中删除安装记录

**Keepalived 缩容流程**:
1. 停止 vip-watcher.timer 和 keepalived 服务
2. 卸载所有 keepalived RPM 包
3. 删除 systemd unit 和配置目录
4. 从 install.txt 中删除安装记录

## 四、更新 Nginx / BES 版本需要的改动

如果要更新 nginx 或 bes 版本，需要以下改动：

### 1. 文件层面

**Nginx 更新**:

| 文件 | 改动类型 | 说明 |
|------|----------|------|
| `/data/asap/thirdSoft/nginx.tar.gz` | **替换** | 上传新版本 nginx 二进制包 |
| `roles/transfer-nginx/tasks/main.yml` | **可能修改** | 如果解压路径或方式变化 |
| `roles/nginx/templates/nginx.conf.j2` | **可能修改** | 如果新版本有配置语法变化 |
| `roles/nginx/templates/nginx.service.j2` | **可能修改** | 如果启动命令或参数变化 |
| `roles/nginx/defaults/main.yml` | **可能修改** | 如果安装路径变化 |

**BES 更新**:

| 文件 | 改动类型 | 说明 |
|------|----------|------|
| `/data/asap/thirdSoft/bes.tar.gz` | **替换** | 上传新版本 bes 二进制包 |
| `roles/transfer-bes/tasks/main.yml` | **可能修改** | 如果解压路径或方式变化 |
| `roles/bes/templates/bws.conf.j2` | **可能修改** | 如果新版本有配置语法变化 |
| `roles/bes/templates/bes.service.j2` | **可能修改** | 如果启动命令或参数变化 |
| `roles/bes/defaults/main.yml` | **可能修改** | 如果安装路径变化 |

### 2. 流程层面

**方案 A: 滚动升级（推荐）**

Nginx 升级:
```bash
# 1. 传输新版本
ansible-playbook -l nginx playbooks/transfer/19.transfer-nginx.yml

# 2. 重新部署（带 restart_nginx 标签）
ansible-playbook -l nginx -t restart_nginx playbooks/k8s/04.nginx.yml
```

BES 升级:
```bash
# 1. 传输新版本
ansible-playbook -l nginx playbooks/transfer/19.transfer-nginx.yml

# 2. 重新部署（带 restart_nginx 标签）
ansible-playbook -l nginx -t restart_nginx playbooks/k8s/04.nginx.yml
```

**方案 B: 创建专用升级 playbook**
创建 `playbooks/upgrade/04.upgrade_nginx.yml`:
```yaml
- hosts: nginx
  tasks:
    - name: 获取服务信息
      service_facts:

    - name: 备份当前配置
      copy:
        src: "{{ nginx_dir }}/conf/nginx.conf"
        dest: "{{ nginx_dir }}/conf/nginx.conf.bak.{{ ansible_date_time.iso8601_basic }}"
        remote_src: yes

    - name: 停止 nginx 服务
      systemd:
        name: "{{ 'asap_nginx' if (LOAD_BALANCE_TYPE is undefined or LOAD_BALANCE_TYPE == 'nginx') else 'asap_bes' }}"
        state: stopped

    - name: 传输新版本
      import_role:
        name: "{{ 'transfer-nginx' if (LOAD_BALANCE_TYPE is undefined or LOAD_BALANCE_TYPE == 'nginx') else 'transfer-bes' }}"

    - name: 重新部署 nginx/bes
      import_role:
        name: "{{ 'nginx' if (LOAD_BALANCE_TYPE is undefined or LOAD_BALANCE_TYPE == 'nginx') else 'bes' }}"

    - name: 验证服务启动
      shell: "systemctl is-active {{ 'asap_nginx' if (LOAD_BALANCE_TYPE is undefined or LOAD_BALANCE_TYPE == 'nginx') else 'asap_bes' }}"
      register: svc_status
      until: '"active" in svc_status.stdout'
      retries: 10
      delay: 3
```

### 3. 需要特别注意的点

1. **配置兼容性**: 新版本 nginx/bes 的配置语法可能有变化，需要验证模板是否兼容
2. **模块兼容性**: libmodsecurity 库可能需要同步更新
3. **SSL 证书**: 确保证书格式与新版本兼容
4. **Keepalived 联动**: nginx/bes 停止期间 keepalived 会触发 VIP 漂移，需要考虑切换时机
5. **业务影响**: 建议在低峰期进行，或使用蓝绿部署方式
6. **负载均衡器切换**: 如果需要从 nginx 切换到 bes，需要修改 `LOAD_BALANCE_TYPE` 变量并重新部署

### 4. 验证步骤

**Nginx 升级后验证**:
- [ ] `curl http://localhost:80` - 本地访问正常
- [ ] `curl https://<VIP>:8686` - VIP 访问正常
- [ ] `systemctl status asap_nginx` - 服务状态正常
- [ ] 检查 `/data/comm/nginx/logs/error.log` - 无错误日志
- [ ] Keepalived VIP 状态正常（无异常漂移）

**BES 升级后验证**:
- [ ] `curl http://localhost:80` - 本地访问正常
- [ ] `curl https://<VIP>:8686` - VIP 访问正常
- [ ] `systemctl status asap_bes` - 服务状态正常
- [ ] 检查 `/data/comm/bes/logs/error.log` - 无错误日志
- [ ] Keepalived VIP 状态正常（无异常漂移）

## 五、总结

| 维度 | Nginx / BES | Keepalived |
|------|-------------|------------|
| **部署方式** | transfer + role | transfer + role |
| **升级方式** | 无专用 playbook | 无专用 playbook |
| **扩容方式** | `add-nginx.yml` | 随 nginx 一起部署 |
| **缩容方式** | `destroy/11.nginx.yml` | `destroy/13.keepalived.yml` |
| **单机/集群** | 通过 `groups['nginx']\|length` 判断 | 始终以集群模式部署 |
| **负载均衡类型** | 通过 `LOAD_BALANCE_TYPE` 选择 | N/A |

**建议**: 为 nginx 和 keepalived 创建专门的 upgrade playbook，参考现有 `04.upgrade_elasticsearch_exporter.yml` 的模式（备份 → 停止 → 传输 → 重启 → 验证）。

## 六、文件清单

### Nginx 相关文件

```
roles/nginx/
├── tasks/main.yml              # 主任务文件
├── defaults/main.yml           # 默认变量
├── nginx.yml                   # 独立入口 playbook
├── cleannginx.yml              # 清理 playbook
└── templates/
    ├── nginx.conf.j2           # 多节点配置模板 (369行)
    ├── nginx_single.conf       # 单节点配置模板 (31行)
    ├── nginx.service.j2        # systemd unit 文件
    └── nginx.j2                # logrotate 配置

roles/transfer-nginx/
├── tasks/main.yml              # 解压 nginx.tar.gz
└── defaults/main.yml           # 默认变量

playbooks/
├── k8s/04.nginx.yml            # 安装入口 (nginx 或 bes)
├── transfer/19.transfer-nginx.yml  # 文件传输
├── destroy/11.nginx.yml        # 销毁 playbook
└── add/add-nginx.yml           # 扩容 playbook
```

### BES (BWS) 相关文件

```
roles/bes/
├── tasks/main.yml              # 主任务文件
├── defaults/main.yml           # 默认变量
└── templates/
    ├── bws.conf.j2             # 多节点配置模板
    ├── bws_single.conf         # 单节点配置模板
    ├── bes.service.j2          # systemd unit 文件
    └── bes.j2                  # logrotate 配置

roles/transfer-bes/
├── tasks/main.yml              # 解压 bes 安装包
└── defaults/main.yml           # 默认变量

playbooks/
├── k8s/04.nginx.yml            # 安装入口 (nginx 或 bes)
├── transfer/19.transfer-nginx.yml  # 文件传输 (包含 bes)
├── destroy/11.nginx.yml        # 销毁 playbook
└── add/add-nginx.yml           # 扩容 playbook
```

### Keepalived 相关文件

```
roles/keepalived/
├── tasks/main.yml              # 主任务文件 (209行)
├── defaults/main.yml           # 默认变量
├── keepalived.yml              # 独立入口 playbook
├── cleankeepalived.yml         # 清理 playbook
└── templates/
    ├── keepalived.conf.j2      # 统一配置模板 (186行)
    ├── keepalived-master.conf.j2  # 旧版主节点配置 (已弃用)
    ├── keepalived-backup.conf.j2  # 旧版备节点配置 (已弃用)
    ├── check_nginx.sh.j2       # nginx 健康检查脚本
    ├── check_bes.sh.j2         # bes 健康检查脚本
    ├── check_flume.sh.j2       # flume 健康检查脚本
    ├── check_lsyncd.sh.j2      # lsyncd 健康检查脚本
    ├── notify_master.sh.j2     # 主节点通知脚本
    ├── notify_backup.sh.j2     # 备节点通知脚本
    ├── notify_fault.sh.j2      # 故障通知脚本
    ├── vip_watcher.sh.j2       # VIP 监控脚本 (87行)
    ├── vip-watcher.service.j2  # VIP 监控 systemd service
    ├── vip-watcher.timer.j2    # VIP 监控 systemd timer
    ├── add_lb.sh.j2            # LB 添加脚本
    └── clear_lb.sh.j2          # LB 清除脚本

roles/transfer-keepalived/
├── tasks/main.yml              # 分发 RPM 包
└── defaults/main.yml           # 默认变量

playbooks/
├── k8s/05.keepalived.yml       # 安装入口
├── transfer/20.transfer-keepalived.yml  # 文件传输
├── destroy/13.keepalived.yml   # 销毁 playbook
└── add/add-nginx.yml           # 扩容入口 (同时处理 keepalived)
```
