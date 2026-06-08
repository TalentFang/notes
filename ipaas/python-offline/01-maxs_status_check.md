# maxs_status_check 模块详解

## 概述

`maxs_status_check` 是 ASAP/IPaaS 分布式监控系统的核心模块，负责对集群中的 20+ 组件进行健康状态检查。该模块通过 Ansible Inventory 发现主机，然后远程执行检查命令，验证各服务的运行状态。

## 目录结构

```
maxs_status_check/
├── maxs_status_check_tool.py    # 主入口文件
├── maxs_master_tool.py           # 主机信息工具
├── maxs_depdency_check.py        # 依赖检查工具
├── conf.ini                      # 配置文件
├── asap_monitor.service          # systemd 服务配置
└── common/                       # 30+ 组件检查器目录
```

## 核心文件

### 1. maxs_status_check_tool.py - 主入口

**命令行格式**:
```bash
python3 maxs_status_check_tool.py <maxs|rds> [base|all]
```

**参数说明**:
- `maxs`: 产品模式 - 完整中间件栈（Redis、MySQL、ES 等）
- `rds`: 产品模式 - 简化版进程检查
- `base`: 检查模式 - 基础组件（默认）
- `all`: 检查模式 - 完整检查

**执行流程**:

```
1. 参数解析与权限检查（必须 root）
2. 打印 ASCII 艺术 banner
3. hosts.get_hosts() 获取所有主机分组
4. 按编号顺序执行各项检查:
   [01]  CPU检查      → machine.check_cpu()
   [02]  内存检查     → machine.check_mem()
   [03]  磁盘检查      → machine.check_disk()
   [04]  集群时间      → machine.check_cluster_time() (需≥3节点)
   [05]  Redis        → redis.check()
   [06]  MySQL        → mysql.check()
   [07]  DM8          → dm.check()
   [08]  Vastbase     → vastbase.check()
   [09]  Zookeeper    → zookeeper.check()
   [10]  Kafka        → kafka.check()
   [11]  Flink        → flink.check()
   [12]  Process      → process.check()
   [13]  Web UI       → Web.check()
   [14]  K8S          → k8s.check()
   [15]  Hadoop       → hadoop.check()
   [16]  Spark        → spark.check()
   [17]  Livy         → livy.check()
   [18]  Kube-metrics → kube_metrics.check()
   [19]  ClickHouse   → clickhouse.check()
   [20]  Registry     → registry.check()
   [21]  Prometheus   → prometheus.check()
   [22]  PushGateway  → pushgateway.check()
   [23]  MinIO        → minio.check()
   [24]  ES           → ES.check()
   [25]  Etcd         → etcd.check() (需>1节点)
5. 业务自定义检查 (business_check.txt)
6. 汇总结果输出
```

### 2. common/ 目录 - 组件检查器

| 文件 | 功能 | 检查方式 |
|------|------|----------|
| **machine.py** | CPU/内存/磁盘/集群时间 | 远程执行 Python 脚本获取 psutil 数据 |
| **redis.py** | Redis 主从集群 | `redis-cli info Replication` |
| **mysql.py** | MySQL 单机和集群 | Python 子进程执行 `SHOW DATABASES` |
| **dm.py** | 达梦 DM8 数据库 | `disql SYSDBA/passwd@IP:5236` |
| **vastbase.py** | VastBase 数据库 | `vsql -h IP -p port -U user -W passwd` |
| **zookeeper.py** | ZK 集群角色 | `zkCli.sh -server IP:port ls /zookeeper` |
| **kafka.py** | Kafka 集群 | `kafka-broker-api-versions.sh` |
| **clickhouse.py** | ClickHouse 集群 | `clickhouse-client` SQL 查询 |
| **mongo.py** | MongoDB 副本集 | `mongo --eval 'rs.status()'` |
| **ES.py** | ElasticSearch 集群健康 | `curl -u elastic:passwd https://IP:9200/_cluster/health` |
| **k8s.py** | Kubernetes 节点/Pod/Service/Ingress | `kubectl get` 系列命令 |
| **flink.py** | Flink 集群 | HTTP 请求 TaskManager API |
| **hadoop.py** | Hadoop HDFS 集群 | `hdfs dfsadmin -report` + JMX |
| **spark.py** | Spark History Server | HTTP 请求 port 18080 |
| **livy.py** | Livy 批处理服务 | HTTP 请求 port 8998 |
| **etcd.py** | Etcd 集群健康 | `etcdctl endpoint health` |
| **prometheus.py** | Prometheus 监控 | `curl -i http://IP:9090/graph` |
| **pushgateway.py** | PushGateway | `curl -i http://IP:9091` |
| **kube_metrics.py** | Kube-metrics | `curl -i http://IP:32500/metrics` |
| **minio.py** | MinIO 对象存储 | `mc admin info minio-server` |
| **registry.py** | Docker 镜像仓库 | `curl -i http://IP:5000/v2/_catalog` |
| **nginx.py** | Nginx 服务 | 检查进程/配置 |
| **process.py** | 各类服务进程状态 | `systemctl status 服务名` |
| **Web.py** | MAXS Web UI | `requests.get('https://IP:8686')` |
| **crontab.py** | 定时任务 | `crontab -l` 检查 |
| **hosts.py** | 主机发现 | Ansible inventory 解析 |
| **call_remote.py** | 远程执行 | SSH 执行远程命令 |
| **readConfig.py** | 配置读取 | 读取 conf.ini |
| **ansible_tool.py** | Ansible 变量读取 | 读取 hosts 文件 :vars 段 |
| **writeLog.py** | 日志记录 | 按机器 UUID 生成日志文件 |

### 3. 主机发现机制 (hosts.py)

```python
def get_hosts():
    # 执行 ansible-inventory 命令获取主机列表
    cmd = 'ansible-inventory -i /data/maxs-ops/inventory/hosts --list'
    # 返回 JSON 格式的主机组信息
```

主机账号文件: `/data/maxs-ops/inventory/server.txt` (格式: `IP#端口#用户名`)

### 4. 远程执行机制 (call_remote.py)

```python
def call_remote_server(host, cmd):
    # 使用 RSA 私钥 /root/.ssh/id_rsa 连接
    client.connect(hostname=host, port=port, username=user, pkey=pkey)
    stdin, stdout, stderr = client.exec_command(cmd)
    return stdout.read().decode('utf-8')
```

### 5. 配置结构 (conf.ini)

```ini
[redis]          port=6379, 16379(集群端口)
[mysql]          port=3306, 集群端口13306
[vastbase]       port=15432, cluster_port=15432
[clickhouse]     port=9002
[zookeeper]      port=2181
[kafka]          port=9092
[flink]          url=http://DIST:9094/overview
[hadoop]         hdfs报告命令, 安全模式命令
[spark]          port=18080
[livy]           port=8998
[kube-metrics]   port=32500
[Maxsprocess]    定义各服务 systemctl 检查命令 (30+标签)
```

## 业务自定义检查

`/data/asap/check/business_check.txt` 格式:
```
check_name=command --args mode
```

## 关键设计模式

1. **远程执行**: 所有服务检查通过 SSH 远程执行
2. **配置驱动**: `conf.ini` 集中管理所有服务地址/端口/命令
3. **分组检查**: 按主机组并行检查该组内所有主机
4. **返回格式**: 统一打印 `[序号] Check X..........PASS/FAIL`
5. **错误收集**: 失败项加入 `error` 列表统一汇总

## VastBase 数据库支持

VastBase 是基于 PostgreSQL 的信创数据库，配置示例：

```ini
[vastbase]
port=15432
cluster_port=15432
user=root
passwd=maxs.PDG~2022
cmdPath=/home/vastbase/vasthome/bin/vsql
```

检查命令: `vsql -h IP -p port -U user -W passwd -d postgres -c "SELECT 1 FROM pg_database WHERE datname = 'SSA';" -t`