# Keepalived

## 一、原理

### 1. VRRP 协议

Keepalived 的核心是基于 **VRRP（Virtual Router Redundancy Protocol，虚拟路由冗余协议）** 实现 IP 高可用。

```
┌─────────────────┐        ┌─────────────────┐
│    Node A        │        │    Node B        │
│  (MASTER)        │        │  (BACKUP)        │
│  priority=120    │        │  priority=100    │
│                  │        │                  │
│  VIP: 10.0.0.100 │        │  VIP: (idle)     │
└────────┬─────────┘        └────────┬─────────┘
         │                           │
         └───────────┬───────────────┘
                     │
             ┌───────┴───────┐
             │   Switch/LAN   │
             └───────┬───────┘
                     │
               ┌─────┴─────┐
               │  Clients   │  ← 始终访问 VIP 10.0.0.100
               └───────────┘
```

**工作原理**：

1. 多台机器组成一个虚拟路由器组，共享一个 VIP
2. 组内通过**优先级（priority）**选举一台作为 MASTER
3. MASTER 周期性地发送 **VRRP 通告（Advertisement）** 给 BACKUP，间隔默认 1 秒
4. BACKUP 在 `master_adv_interval * 3 + skew_time` 时间内没收到通告，就认为 MASTER 挂了，自己升级为 MASTER 并接管 VIP
5. MASTER 恢复后会发送优先级更高的通告，通过 `nopreempt` 控制是否立即夺回

### 2. 核心机制

**优先级选举**：
- 优先级范围 0-255，MASTER 的 priority 最高
- priority=255 时，节点认为自己拥有 VIP 所属的物理网卡，强制成为 MASTER
- 健康检查脚本通过 `weight` 动态调整优先级

**健康检查（vrrp_script）**：
```
每 interval 秒执行一次脚本
  ├── 返回 0（成功）→ weight 加到 priority
  └── 返回非0（失败）→ weight 从 priority 减去

连续失败 fall 次 → 触发切换
连续成功 rise 次 → 恢复正常
```

**VRRP 通信方式**：
| 方式 | 原理 | 适用场景 |
|------|------|----------|
| 组播（默认） | 发到 `224.0.0.18`，同一 L2 域内所有节点可见 | 物理机房、支持组播的网络 |
| 单播（unicast） | 明确指定 peer IP 列表，逐一点对点发送 | 云环境、不支持组播的网络 |

## 二、配置详解

### 1. 全局配置（global_defs）

```
global_defs {
    router_id hostname           # 唯一标识本节点，建议用主机名
    vrrp_skip_check_adv_addr    # 不检查通告中的源地址（单播时建议开启）
    vrrp_strict                 # 严格模式，开启后会因 iptables 等原因导致启动失败
}
```

### 2. VRRP 实例（vrrp_instance）

```
vrrp_instance VI_1 {
    state MASTER              # 初始状态：MASTER 或 BACKUP
    interface eth0            # VIP 绑定的物理网卡
    virtual_router_id 51      # 虚拟路由器 ID，同一组内必须一致（0-255）
    priority 120              # 优先级，越大越优先
    advert_int 1              # VRRP 通告间隔（秒）
    nopreempt                 # 禁止抢占：MASTER 恢复后不自动夺回 VIP

    authentication {
        auth_type PASS         # PASS 或 AH
        auth_pass your_pass    # 密码，同一组内必须一致（最多8字符）
    }

    virtual_ipaddress {
        10.0.0.100/24 dev eth0           # VIP + 子网掩码
        10.0.0.101/24 dev eth0 label eth0:1  # 使用 label 创建别名
    }

    track_script {
        check_nginx             # 关联健康检查脚本
    }

    notify_master "/path/to/script.sh"   # 切换为 MASTER 时执行
    notify_backup "/path/to/script.sh"   # 切换为 BACKUP 时执行
    notify_fault  "/path/to/script.sh"   # 故障时执行
    notify_stop   "/path/to/script.sh"   # keepalived 停止时执行
}
```

### 3. 健康检查脚本（vrrp_script）

```
vrrp_script check_nginx {
    script "/etc/keepalived/check_nginx.sh"   # 检查脚本路径
    interval 3           # 检查间隔（秒）
    weight -60           # 失败时扣减权重，0表示失败直接切换状态
    fall 3               # 连续失败多少次触发故障
    rise 3               # 连续成功多少次恢复
    timeout 5            # 脚本超时时间（秒）
    user root            # 脚本执行用户
}
```

**weight 策略对比**：

| weight 设置 | 失败时行为 |
|-------------|-----------|
| `weight -60` | priority 降低 60 分，降到低于 BACKUP 则切换 |
| `weight 0` | 不调整优先级，直接触发 FAULT 状态（立即切换） |
| `weight -5` | 渐进式降权，适合需要累积多次失败才切换的场景 |

### 4. 单播配置（unicast）

```
vrrp_instance VI_1 {
    ...
    unicast_src_ip 192.168.1.10           # 本机源 IP

    unicast_peer {
        192.168.1.11                       # 对端 IP 列表
        192.168.1.12
    }
}
```

### 5. 完整示例：Nginx 高可用

```
global_defs {
    router_id nginx-lb-01
}

vrrp_script check_nginx {
    script "killall -0 nginx"       # 仅检查进程，不依赖 systemd
    interval 3
    weight -60
    fall 3
    rise 3
}

vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 100
    priority 120
    nopreempt
    advert_int 1

    authentication {
        auth_type PASS
        auth_pass 12345678
    }

    virtual_ipaddress {
        10.0.0.100/24 dev eth0
    }

    track_script {
        check_nginx
    }

    notify_master "/etc/keepalived/notify_master.sh"
    notify_backup "/etc/keepalived/notify_backup.sh"
}
```

## 三、常见用法

### 1. 基本操作

```bash
# 启动/停止
systemctl start keepalived
systemctl stop keepalived

# 查看状态
systemctl status keepalived
journalctl -u keepalived -f

# 检查 VIP 是否绑定
ip addr show eth0 | grep <VIP>

# 查看 VRRP 通告日志
tcpdump -i eth0 -nn 'proto 112'
```

### 2. 脑裂预防

**方案 A：nopreempt + 随机优先级**

```yaml
# 仅 MASTER 设置 nopreempt，BACKUP 使用随机优先级防止多备节点同时竞选
# MASTER:
state MASTER
priority 120
nopreempt

# BACKUP:
state BACKUP
priority {{ 109 | random(61, 1) }}   # 61~169 随机值
```

**方案 B：VIP 看门狗**

独立于 keepalived 的监控进程，检测到 MASTER 标记文件存在但 VIP 丢失时，重启 keepalived：

```bash
if [ -f "/var/run/keepalived_master" ]; then
    if ! ip addr show eth0 | grep -q "$VIP"; then
        systemctl restart keepalived
    fi
fi
```

### 3. 基于状态的自动化

notify 脚本接收 4 个参数：`$1=TYPE(GROUP/INSTANCE)`, `$2=NAME(VI_1)`, `$3=TYPE_MASTER/BACKUP/FAULT`, `$4=PRIORITY`

```bash
#!/bin/bash
# notify_master.sh
TYPE=$1
NAME=$2
STATE=$3

case $STATE in
    "MASTER")
        touch /var/run/keepalived_master
        # 启动本节点需要独占的服务
        ;;
    "BACKUP")
        rm -f /var/run/keepalived_master
        # 停止本节点不需要的服务
        ;;
    "FAULT")
        rm -f /var/run/keepalived_master
        logger -t keepalived "Fault detected, stopping keepalived"
        systemctl stop keepalived
        ;;
esac
```

## 四、关键配置参数速查

| 参数 | 含义 | 默认值 | 建议值 |
|------|------|--------|--------|
| `advert_int` | VRRP 通告间隔（秒） | 1 | 1 |
| `fall` | 连续失败次数触发故障 | 3 | 3 |
| `rise` | 连续成功次数恢复 | 3 | 2-3 |
| `weight` | 健康检查权重调整 | — | MASTER: -60, BACKUP: -5 |
| `nopreempt` | 禁止抢占 | 关闭 | MASTER 上开启 |
| `virtual_router_id` | 虚拟路由器 ID | — | 同组一致，不同组不冲突 |

## 五、故障排查

```bash
# 1. 检查 keepalived 进程
ps aux | grep keepalived

# 2. 查看配置语法错误
keepalived -t -f /etc/keepalived/keepalived.conf

# 3. 前台运行调试（打印日志到终端）
keepalived -n -D -f /etc/keepalived/keepalived.conf

# 4. 检查是否收到 VRRP 包
tcpdump -i eth0 vrrp -nn

# 5. 常见启动失败原因
# - vrrp_strict 开启 + iptables 未放行 → 关闭 strict 或添加 iptables 规则
# - virtual_router_id 冲突 → 同一 L2 域内 router_id 必须唯一
# - interface 名称错误 → ip link show 确认网卡名
# - auth_pass 不一致 → 同组内密码必须完全一致
# - SELinux 拦截 → setenforce 0 或配置策略
```
