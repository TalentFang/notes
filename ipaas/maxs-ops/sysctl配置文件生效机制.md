# Sysctl 配置文件生效机制

## 一、参数生效机制

系统启动时，`systemd-sysctl.service` 按以下顺序加载所有配置，**后面的覆盖前面的**：

```
1. /usr/lib/sysctl.d/*.conf          ← systemd 默认参数
2. /usr/local/lib/sysctl.d/*.conf
3. /run/sysctl.d/*.conf
4. /etc/sysctl.d/*.conf              ← 自定义参数
5. /etc/sysctl.conf                  ← 最后加载，优先级最高
```

同一参数被多次写入时，**最后一次写入的值生效**。

手动执行时也可通过 `sysctl -p` 指定文件加载：

```bash
sysctl -p /etc/sysctl.conf                    # 加载指定文件
sysctl --system                                # 按顺序加载所有文件
```

## 二、配置文件及修改来源

### 1. /usr/lib/sysctl.d/*.conf（优先级最低）

由系统 rpm 包自带，maxs-ops 未做任何修改。

| 文件 | 来源包 | 写入的参数 |
|------|--------|-----------|
| `10-default-yama-scope.conf` | kernel/security 相关包 | `kernel.yama.ptrace_scope` |
| `50-coredump.conf` | systemd | `kernel.core_pattern`, `fs.suid_dumpable` |
| `50-default.conf` | systemd | `rp_filter`, `accept_source_route`, `promote_secondaries` 等 |
| `50-libkcapi-optmem_max.conf` | libkcapi | `net.core.optmem_max` |
| `50-lsyncd.conf` | lsyncd | `fs.inotify.max_user_watches` |
| `50-pid-max.conf` | systemd | `kernel.pid_max` |

### 2. /etc/sysctl.d/*.conf

| 文件 | 来源 | 修改者 |
|------|------|--------|
| `99-sysctl.conf` | systemd 包创建的符号链接 → `../sysctl.conf` | 不可独立修改，内容跟随 `/etc/sysctl.conf` |

`/etc/sysctl.d/99-sysctl.conf` 不是独立文件，是 systemd 包安装时创建的符号链接：

```
/etc/sysctl.d/99-sysctl.conf -> ../sysctl.conf
```

作用是让 `/etc/sysctl.conf` 通过 sysctl.d 机制被加载。

### 3. /etc/sysctl.conf（优先级最高）

由多个来源依次写入，最终值为多次叠加的结果。

#### 3.1 UOS security-tool（系统安装时执行一次）

安装包：`security-tool-2.0-1.88.uos25.18`

通过 `UnionTech-security.service`（Type=oneshot）在系统安装时执行一次，不会在启动时重新执行。

配置文件目录：`/etc/UnionTech_security/`

| 文件 | 用途 |
|------|------|
| `security.conf` | 基础安全规则（内核参数、SSH、文件权限等） |
| `usr-security.conf` | 用户自定义规则（覆盖基础规则） |
| `enhance-security.conf` | 增强安全规则 |


security-tool 写入的 sysctl 参数：

| 参数 | security.conf | usr-security.conf |
|------|:---:|:---:|
| `net.ipv4.conf.all.send_redirects` | `=0` | - |
| `net.ipv4.conf.default.send_redirects` | `=0` | - |
| `net.ipv4.conf.all.accept_source_route` | `=0` | - |
| `net.ipv4.conf.default.accept_source_route` | `=0` | - |
| `net.ipv4.conf.all.accept_redirects` | `=0` | - |
| `net.ipv4.conf.default.accept_redirects` | `=0` | - |
| `net.ipv4.conf.all.secure_redirects` | `=0` | - |
| `net.ipv4.conf.default.secure_redirects` | `=0` | - |
| `net.ipv4.icmp_echo_ignore_broadcasts` | `=1` | - |
| `net.ipv4.icmp_ignore_bogus_error_responses` | `=1` | - |
| `net.ipv4.conf.all.rp_filter` | `=1` | - |
| `net.ipv4.conf.default.rp_filter` | `=1` | - |
| `net.ipv4.tcp_syncookies` | `=1` | - |
| `kernel.dmesg_restrict` | `=1` | - |
| `net.ipv6.conf.all.accept_redirects` | `=0` | - |
| `net.ipv6.conf.default.accept_redirects` | `=0` | - |
| `net.core.busy_read` | `=100` | - |
| `net.ipv4.ip_forward` | `=0` | - |
| `kernel.sysrq` | - | `=0` |
| `kernel.panic` | - | `=3` |
| `fs.suid_dumpable` | - | `=0` |
| `net.ipv4.conf.all.log_martians` | - | `=1` |
| `net.ipv4.conf.default.log_martians` | - | `=1` |
| `net.ipv4.route.flush` | - | `=1` |
| `kernel.randomize_va_space` | - | `=2` |
| `net.ipv6.conf.all.forwarding` | - | `=0` |
| `kernel.kptr_restrict` | - | `=1` |

#### 3.2 maxs-ops Ansible 项目（UOS 上的执行情况）

**UOS 上 common.yml 的 sysctl 相关任务不会执行**，因为条件不匹配：

```yaml
# common.yml - 不执行（条件：CentOS/RedHat）
- name: 设置系统参数
  template:
    src: templates/95-sysctl.conf.j2
    dest: /etc/sysctl.d/95-sysctl.conf
  when: 'ansible_distribution in ["CentOS", "RedHat"]'   # UOS 不匹配
```

| Role | UOS 上是否执行 | 写入方式 | 写入的参数 |
|------|:-:|------|------|
| `roles/prepare/tasks/common.yml` | **不执行** | shell echo >> /etc/sysctl.conf | `bridge-nf-call-iptables`, `ip_forward=1`, `somaxconn`, `nf_conntrack_max`, `swappiness=0`, `max_map_count=655360`, `file-max=6553600` |
| `roles/prepare/tasks/kylin.yml` | **执行** | sed 替换 | `net.ipv4.ip_forward=0` → `=1` |
| `roles/vastbase/tasks/prepare.yml` | **执行** | sysctl 模块 | `fs.aio-max-nr`, `fs.file-max`, `netdev_max_backlog`, `rmem_default/max`, `wmem_default/max`, `somaxconn=4096`, `tcp_fin_timeout`, `tcp_retries1`, `tcp_syn_retries`, `overcommit_memory`, `ip_local_port_range`, `fs.nr_open`, `kernel.sem/shmall/shmmax/shmmni`, `vm.dirty_*`, `kernel.core_pattern` |
| `roles/usradd-asap/tasks/main.yml` | 视情况 | lineinfile | `vm.swappiness=0`, `vm.max_map_count=262144` |

#### 3.3 手动编辑

以下参数在 security-tool 和 Ansible 代码库中都找不到来源，可能是由运维人员手动编辑写入：

| 参数 | 值 |
|------|-----|
| `net.ipv4.tcp_keepalive_time` | `30` |
| `net.ipv4.tcp_keepalive_intvl` | `30` |
| `vm.min_free_kbytes` | `4607296` |
| `net.core.somaxconn` | `65535`（覆盖 vastbase 写入的 `4096`） |
| `kernel.shmall` | `1152921504606846720`（覆盖 vastbase 动态计算值） |
| `kernel.shmmax` | `18446744073709551615`（覆盖 vastbase 动态计算值） |

### 4. 覆盖关系总结

UOS 上各来源对 `/etc/sysctl.conf` 的写入按时间顺序叠加，后写入的覆盖先写入的：

```
security-tool（OS 安装）→ kylin.yml（Ansible）→ vastbase/prepare.yml（Ansible）→ 手动编辑
```

| 冲突参数 | security-tool | kylin.yml | vastbase | 手动编辑 | 最终生效 |
|---------|:---:|:---:|:---:|:---:|:---:|
| `net.ipv4.ip_forward` | `=0` | sed 替换 `=1` | - | - | `1`（kylin） |
| `vm.dirty_background_ratio` | `=30` | - | `=5` | - | `5`（vastbase） |
| `vm.dirty_ratio` | `=50` | - | `=10` | - | `10`（vastbase） |
| `net.core.somaxconn` | - | - | `=4096` | 手动改 `=65535` | `65535`（手动） |
| `kernel.shmall` | - | - | 动态计算 | 手动改 | 手动值（手动） |
| `kernel.shmmax` | - | - | 动态计算 | 手动改 | 手动值（手动） |

## 三、当前参数来源追溯（以 10.21.18.162 为例）

UOS 上的实际执行顺序：

```
① security-tool（OS 安装时）  →  写入 /etc/sysctl.conf
② common.yml shell echo       →  不执行（条件：CentOS/RedHat）
③ common.yml template         →  不执行（条件：CentOS/RedHat）
④ kylin.yml sed               →  执行（条件包含 UOS）→ 替换 ip_forward=0 为 1
⑤ vastbase/prepare.yml        →  执行 → sysctl 模块写入参数
```

| 参数 | security-tool | kylin.yml | vastbase/prepare | 手动编辑 | 最终值 |
|------|:---:|:---:|:---:|:---:|:---:|
| `net.ipv4.ip_forward` | `=0` | sed 替换为 `=1` | - | - | `1` |
| `net.ipv4.conf.all.send_redirects` | `=0` | - | - | - | `0` |
| `net.ipv4.conf.all.accept_source_route` | `=0` | - | - | - | `0` |
| `net.ipv4.conf.all.accept_redirects` | `=0` | - | - | - | `0` |
| `net.ipv4.conf.all.secure_redirects` | `=0` | - | - | - | `0` |
| `net.ipv4.icmp_echo_ignore_broadcasts` | `=1` | - | - | - | `1` |
| `net.ipv4.icmp_ignore_bogus_error_responses` | `=1` | - | - | - | `1` |
| `net.ipv4.conf.all.rp_filter` | `=1` | - | - | - | `1` |
| `net.ipv4.tcp_syncookies` | `=1` | - | - | - | `1` |
| `kernel.dmesg_restrict` | `=1` | - | - | - | `1` |
| `net.ipv6.conf.all.accept_redirects` | `=0` | - | - | - | `0` |
| `net.core.busy_read` | `=100` | - | - | - | `100` |
| `kernel.sysrq` | `=0` | - | - | - | `0` |
| `kernel.panic` | `=3` | - | - | - | `3` |
| `fs.suid_dumpable` | `=0` | - | - | - | `0` |
| `net.ipv4.conf.all.log_martians` | `=1` | - | - | - | `1` |
| `net.ipv4.route.flush` | `=1` | - | - | - | `1` |
| `kernel.randomize_va_space` | `=2` | - | - | - | `2` |
| `kernel.kptr_restrict` | `=1` | - | - | - | `1` |
| `vm.swappiness` | `=0` | - | - | - | `0` |
| `vm.max_map_count` | `=262144` | - | - | - | `262144` |
| `fs.file-max` | `=76724600` | - | `=76724600` | - | `76724600` |
| `net.core.netdev_max_backlog` | `=10000` | - | `=10000` | - | `10000` |
| `net.core.rmem_default` | - | - | `=262144` | - | `262144` |
| `net.core.rmem_max` | - | - | `=4194304` | - | `4194304` |
| `net.core.wmem_default` | - | - | `=262144` | - | `262144` |
| `net.core.wmem_max` | - | - | `=4194304` | - | `4194304` |
| `net.core.somaxconn` | - | - | `=4096` | 手动改 `=65535` | `65535` |
| `net.ipv4.tcp_fin_timeout` | - | - | `=60` | - | `60` |
| `net.ipv4.tcp_retries1` | - | - | `=5` | - | `5` |
| `net.ipv4.tcp_syn_retries` | - | - | `=5` | - | `5` |
| `vm.overcommit_memory` | - | - | `=0` | - | `0` |
| `net.ipv4.ip_local_port_range` | - | - | `=40000 65535` | - | `40000 65535` |
| `fs.nr_open` | - | - | `=20480000` | - | `20480000` |
| `kernel.shmmni` | - | - | `=4096` | - | `4096` |
| `kernel.sem` | - | - | 动态值 | - | `250 6400000 1000 25600` |
| `kernel.shmall` | - | - | 动态计算 | 手动改 | `1152921504606846720` |
| `kernel.shmmax` | - | - | 动态计算 | 手动改 | `18446744073709551615` |
| `vm.dirty_background_ratio` | `=30` | - | `=5` | - | `5` |
| `vm.dirty_ratio` | `=50` | - | `=10` | - | `10` |
| `vm.dirty_expire_centisecs` | - | - | `=500` | - | `500` |
| `vm.dirty_writeback_centisecs` | - | - | `=100` | - | `100` |
| `kernel.core_pattern` | - | - | vastbase 路径 | - | vastbase 路径 |
| `net.ipv4.tcp_keepalive_time` | - | - | - | 手动加 `=30` | `30` |
| `net.ipv4.tcp_keepalive_intvl` | - | - | - | 手动加 `=30` | `30` |
| `vm.min_free_kbytes` | - | - | - | 手动加 `=4607296` | `4607296` |
