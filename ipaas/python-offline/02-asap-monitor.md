# asap-monitor 模块详解

## 概述

`asap-monitor` 是系统监控和 Flink 集群健康监控组件，包含两个主要功能：
1. **系统指标采集**: 通过 `perfstats-to-syslog.py` 采集 CPU、内存、磁盘、网络、RAID 等系统指标
2. **Flink 监控**: 通过 `flink_monitor.sh` 和 `checkTaskmanager.py` 监控 Flink 集群状态并在故障时自动重启服务

## 目录结构

```
asap-monitor/
├── perfstats-to-syslog.py       # 主监控脚本
├── perfstats-to-syslog.cfg      # 配置文件
├── perfstats-to-syslog.cfg.template  # 配置模板
├── checkTaskmanager.py          # TaskManager 检查脚本
├── flink_monitor.sh             # Flink 监控脚本
├── asap_monitor.service         # systemd 服务配置
└── yum/                         # 离线 yum 源配置
```

## 核心文件

### 1. perfstats-to-syslog.py - 系统指标采集

#### 用途
作为监控代理（monitoring-agent），采集并上报系统性能指标到 syslog 和数据库。

#### 核心类结构

| 类名 | 功能 |
|------|------|
| `DeltaMeter` | 计算指标增量，用于网络流量等累积值 |
| `SimpleMeter` | 简单类型转换器，用于 CPU 百分比等瞬时值 |
| `TaskThread` | 线程调度器，定时执行任务 |
| `AReporter` | 基础 reporter 类，处理指标注册、数据收集、处理、上报 |
| `SystemReporter` | 系统指标采集器，继承自 AReporter |
| `ContextFilter` | 日志过滤器，添加 hostname 字段 |

#### 采集的指标类型

- **CPU**: `psutil.cpu_percent()` 获取 CPU 使用率
- **内存**: 虚拟内存和交换内存的总量、可用、已用、百分比等
- **磁盘**: 每个挂载路径的总容量、已用、可用、使用率
- **网络**: 每个网络接口的发送/接收字节数、数据包数、错误数、丢包数
- **磁盘 IO**: 每个磁盘的读写次数、读写字节数、读写时间、忙碌时间
- **RAID**: 通过 MegaCLI 命令获取 RAID 状态

#### 数据上报

- 写入本地日志文件 `/data/asap/logs/asap_monitor.log`（滚动日志，单文件最大 10MB，保留 7 个备份）
- 插入数据库表：`SSA.MONITOR_MEM`, `SSA.MONITOR_DISK`, `SSA.MONITOR_IO_DISK`, `SSA.MONITOR_NETWORK`, `SSA.MONITOR_RAID`
- 支持 MySQL、DM（达梦）和 VastBase 数据库

#### 工作流程

```
main()
  → 读取配置文件 perfstats-to-syslog.cfg
  → 创建 SystemReporter 实例
  → 启动 reporter 线程（daemon 模式）
  → reporter.prestart() → collect() + register()
  → 定时任务：每 interval 秒执行 task()
    → collect() 采集数据
    → process() 处理数据
    → insert_mem/disk/io_disk/network/raid() 写入数据库
```

### 2. checkTaskmanager.py - TaskManager 检查

#### 用途
检查当前主机是否在 Flink TaskManager 列表中。

#### 工作逻辑

```
checkTaskManager(taskmanagerUrl)
  → 访问 TaskManager API 获取所有 TaskManager 列表
  → 比对当前主机名是否在列表中
  → 如果在：exit(0) 表示正常
  → 如果不在：
      → 遍历 masters 配置中的所有 JobManager
      → 尝试从每个 JobManager 获取 TaskManager 列表
      → 如果数量不匹配，记录日志
      → exit(1) 表示异常
```

### 3. flink_monitor.sh - Flink 健康监控

#### 用途
监控 Flink 集群健康状态，在检测到故障时自动重启相关服务。

#### 主要逻辑

**单机模式**（master 和 worker 都为 1 个）:
- 从 flink-conf.yaml 获取 REST API 端口
- 调用 `http://DIST:{port}/overview` API 获取集群概况
- 检查 taskmanager 数量：
  - API 返回 0 且本地无 taskmanager → 重启 flink
  - API 返回 0 但本地有 taskmanager → 杀死本地 taskmanager，重启 flink taskmanager

**集群模式**（多于 1 个 master 或 worker）:
- 从配置获取 Prometheus Gateway 主机
- 调用 `http://{promgateway_host}:9094/taskmanagers` 获取 TaskManager 列表
- 执行 `checkTaskmanager.py` 检查当前主机是否在列表中
- 如果不在，检查是否有 OOM 错误：
  - 20 分钟内发生过 OOM → 重启 flink taskmanager
  - 超过 20 分钟 → 累计次数，达到 60 次后重启
- 检查当前主机是否为 JobManager：
  - 如果是，且本地无 JobManager → 重启 flink jobmanager
- 检查本地 TaskManager 数量是否正常（应为 1）：
  - 不为 1 → 杀死多余的 taskmanager，重启 flink taskmanager

#### 关键配置路径
- Flink 目录: `/data/comm/flink-1.16.0`
- 配置文件: `${flink_dir}/conf/flink-conf.yaml`
- Masters 配置: `${flink_dir}/conf/masters`
- Workers 配置: `${flink_dir}/conf/workers`
- 日志文件: `/data/comm/flink-1.16.0/log/flink-asap-taskexecutor-*-{hostname}.log`

#### 计数器机制
在 `/data/comm/flink-1.16.0/tmp/numfile` 中记录检查次数，达到 60 次时强制重启。

## systemd 服务配置

```ini
[Unit]
Description=22th:perfstats
RequiresMountsFor=/data

[Service]
EnvironmentFile=/etc/service.env
Type=forking
ExecStart=/bin/sh -c 'source /etc/profile && python3 $DEPLOY_HOME/asap-monitor/perfstats-to-syslog.py&'
ExecStop=/bin/kill -s TERM $MAINPID
Restart=always              # 总是重启
RestartSec=60               # 重启间隔 60 秒
StartLimitBurst=60          # 最多重启 60 次
StartLimitInterval=36000    # 36000 秒内
TimeoutSec=720

[Install]
WantedBy=multi-user.target
```

## 配置说明 (perfstats-to-syslog.cfg)

```ini
[syslog]
host = DIST              # syslog 服务器地址
port = 10050             # syslog 端口
pollingInSec = 300       # 采集间隔（5 分钟）

[db]
name = SSA               # 数据库名
host = DB0               # 数据库地址
port = 3306              # 数据库端口
user = root              # 用户名
pwd = maxs.PDG~2022      # 密码
type = vastbase          # 数据库类型（mysql/dm/vastbase）

[general]
paths = root:/data,root:/,root:/home  # 要监控的磁盘路径
```

## 数据库支持

| db_type | 数据库 | 连接方式 |
|---------|--------|----------|
| `mysql` | MySQL | `pymysql.connect()` |
| `dm` | DM8（达梦） | `dmPython.connect()` |
| `vastbase` | VastBase | `psycopg2.connect()` |
| 其他 | 默认 VastBase | `psycopg2.connect()` |

## 架构图

```
┌─────────────────────────────────────────────────────────────┐
│               asap_monitor.service (systemd)                │
│                                                             │
│   ┌─────────────────────────────────────────────────────┐   │
│   │    perfstats-to-syslog.py (监控代理)                 │   │
│   │                                                     │   │
│   │   SystemReporter (采集系统指标)                      │   │
│   │   ├── CPU, Memory, Disk, Network, DiskIO, RAID      │   │
│   │   └── 上报到 DB (MySQL/DM/VastBase) + 日志文件       │   │
│   └─────────────────────────────────────────────────────┘   │
│                           ▲                                  │
│                           │ 定时调用                         │
│                           ▼                                  │
│   ┌─────────────────────────────────────────────────────┐   │
│   │           flink_monitor.sh (Flink 健康监控)          │   │
│   │                                                     │   │
│   │   ├── 单机模式: 检查 JobManager/TaskManager 数量     │   │
│   │   ├── 集群模式: 调用 checkTaskmanager.py            │   │
│   │   │          └── 检查当前主机是否在 TaskManager 列表  │   │
│   │   │          └── 检查 OOM 日志                        │   │
│   │   └── 异常时重启 asap_flink / asap_flink_taskmanager  │   │
│   └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

## 数据表结构

监控数据写入以下 SSA 库表：
- `MONITOR_MEM` - 内存指标
- `MONITOR_DISK` - 磁盘使用指标
- `MONITOR_IO_DISK` - 磁盘 IO 指标
- `MONITOR_NETWORK` - 网络流量指标
- `MONITOR_RAID` - RAID 状态指标