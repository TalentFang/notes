# ClickHouse 学习文档

## 目录

1. [概述](#1-概述)
2. [核心组件介绍](#2-核心组件介绍)
3. [核心概念与原理](#3-核心概念与原理)
4. [单机部署](#4-单机部署)
5. [集群部署](#5-集群部署)
6. [常用操作与维护](#6-常用操作与维护)

---

## 1. 概述

ClickHouse 是俄罗斯搜索公司 Yandex 开源的 OLAP 数据库，专为在线分析处理（OLAP）设计。

### 核心特性

| 特性 | 说明 |
|------|------|
| 列式存储 | 仅读取查询需要的列，大幅减少 IO |
| 向量化执行 | 使用 SIMD 指令加速计算 |
| 数据压缩 | 高效压缩算法，减少存储空间 |
| 并行处理 | 分布式处理，充分利用多核 |
| SQL 支持 | 兼容标准 SQL，易于使用 |
| 实时写入 | 支持高并发实时数据写入 |
| 分片副本 | 支持分布式部署和数据冗余 |

### 适用场景

- 用户行为分析
- 日志分析
- 实时报表
- 物联网数据处理
- 点击流分析

---

## 2. 核心组件介绍

### 2.1 ClickHouse 架构图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              ClickHouse 集群                                 │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                        Coordinator Node                                │  │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐       │  │
│  │  │  Query Pipeline │  │  Interpreter    │  │  Optimizer      │       │  │
│  │  └─────────────────┘  └─────────────────┘  └─────────────────┘       │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                    │                                         │
│              ┌─────────────────────┼─────────────────────┐                  │
│              ▼                     ▼                     ▼                  │
│  ┌───────────────────┐  ┌───────────────────┐  ┌───────────────────┐       │
│  │   Shard 1        │  │   Shard 2        │  │   Shard 3        │       │
│  │  ┌─────────────┐ │  │  ┌─────────────┐ │  │  ┌─────────────┐ │       │
│  │  │ Replica 1-1 │ │  │  │ Replica 2-1 │ │  │  │ Replica 3-1 │ │       │
│  │  └─────────────┘ │  │  └─────────────┘ │  │  └─────────────┘ │       │
│  │  ┌─────────────┐ │  │  ┌─────────────┐ │  │  ┌─────────────┐ │       │
│  │  │ Replica 1-2 │ │  │  │ Replica 2-2 │ │  │  │ Replica 3-2 │ │       │
│  │  └─────────────┘ │  │  └─────────────┘ │  │  └─────────────┘ │       │
│  └───────────────────┘  └───────────────────┘  └───────────────────┘       │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 核心组件

#### ClickHouse Server

主服务进程，负责处理查询和数据存储。

主要功能：
- 接收客户端请求
- 解析和执行 SQL
- 管理数据和索引
- 与其他节点通信

#### Keeper (ZooKeeper)

分布式协调服务，用于管理集群元数据。

- 存储表结构信息
- 管理复制状态
- 协调分布式 DDL
- 处理故障恢复

#### ClickHouse Client

命令行客户端工具，用于连接服务器执行查询。

#### clickhouse-benchmark

性能压测工具，用于基准测试。

#### clickhouse-copier

数据迁移工具，用于在集群间迁移数据。

### 2.3 内部组件

```
┌─────────────────────────────────────────────────────────────────┐
│                       ClickHouse Server                         │
│                                                                 │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐            │
│  │ Interpreter │  │  Optimizer  │  │   Planner   │            │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘            │
│         │                │                │                    │
│  ┌──────┴────────────────┴────────────────┴──────┐            │
│  │                 Pipeline                       │            │
│  │  ┌───────┐  ┌───────┐  ┌───────┐  ┌───────┐  │            │
│  │  │ Read  │→ │ Filter │→ │ Aggregate│→ │Limit │  │            │
│  │  └───────┘  └───────┘  └───────┘  └───────┘  │            │
│  └─────────────────────────────────────────────┘            │
│                          │                                     │
│  ┌───────────────────────┴────────────────────────┐           │
│  │                  MergeTree Engine              │           │
│  │  ┌───────────┐ ┌───────────┐ ┌───────────┐   │           │
│  │  │  Parts    │ │  Marks    │ │ Index     │   │           │
│  │  └───────────┘ └───────────┘ └───────────┘   │           │
│  └─────────────────────────────────────────────┘            │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3. 核心概念与原理

### 3.1 MergeTree 表引擎

MergeTree 是 ClickHouse 最核心的表引擎，支持大规模数据存储和快速查询。

#### 数据结构

```
MergeTree Table
├── Part 1 (2026-05-01)
│   ├── 20260501_20260501_1_1_0.zip
│   │   ├── data.bin     (列数据)
│   │   ├── data.mrk2    (标记文件)
│   │   └── primary.idx   (主键索引)
│   ├── ...
│   └── 20260501_20260501_5_5_0.zip
├── Part 2 (2026-05-02)
│   └── ...
└── Part 3 (2026-05-03)
    └── ...
```

#### 核心机制

| 机制 | 说明 |
|------|------|
| 分区 (Partition) | 数据按分区键物理隔离，支持快速分区裁剪 |
| 排序键 (Sort Key) | 数据按主键排序，决定索引结构 |
| 标记文件 (Marks) | 行号与数据块位置的映射，加速数据读取 |
| 主键索引 (Primary Key Index) | 稀疏索引，快速定位数据块 |
| TTL | 数据生命周期管理 |

#### 合并过程

```
Writes:  Part_1_1_1  Part_1_2_2  Part_1_3_3
            │           │           │
            └───────────┴───────────┘
                        ▼
                   Background Merge
                        │
                        ▼
                Merged Part_1_1_3
```

### 3.2 数据存储原理

#### 列式存储

```
传统行存储:                    ClickHouse 列存储:
┌────────────────┐            ┌─────────┐ ┌─────────┐ ┌─────────┐
│ id │ name │ age│            │ id (1,2)│ │ name    │ │ age     │
│────┼───────┼────│            └─────────┘ └─────────┘ └─────────┘
│ 1  │ Alice │ 25 │                              ↓           ↓
│ 2  │ Bob   │ 30 │                           读取列      读取列
│ 3  │ Carol │ 28 │
└────────────────┘
```

#### 压缩算法

ClickHouse 支持多种压缩算法：

| 压缩算法 | 适用场景 | 压缩率 |
|----------|----------|--------|
| LZ4 | 通用场景，快速解压 | ~1.7x |
| ZSTD | 高压缩比 | ~2-3x |
| Delta | 有序递增数据 | 高 |
| Gorilla | 时间序列数据 | 极高 |

### 3.3 查询执行原理

#### 解析流程

```
SQL Query
    │
    ▼
┌─────────────┐
│  Parser     │  解析 SQL，生成 AST
└──────┬──────┘
       │
       ▼
┌─────────────┐
│ Interpreter │  分析 AST，验证语义
└──────┬──────┘
       │
       ▼
┌─────────────┐
│ Optimizer   │  优化查询计划
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  Planner    │  生成执行 Pipeline
└──────┬──────┘
       │
       ▼
   Execution
```

#### 向量化执行

ClickHouse 使用向量化执行引擎，将操作应用于数据向量而非单行：

```
传统执行:                    向量化执行:
for row in rows:        →    batch = rows[0:1024]
    result.append(op(row))      result = vector_op(batch)
```

### 3.4 复制与故障恢复

#### 副本复制流程

```
写入流程:
┌──────────┐    Write     ┌──────────┐
│  Leader  │ ───────────→ │ Follower │
│  Replica │              │  Replica │
└──────────┘              └──────────┘
       │                         │
       │    日志同步              │
       ▼                         ▼
    ┌──────────────────────────────┐
    │          ZooKeeper           │
    │      (协调复制状态)           │
    └──────────────────────────────┘
```

#### 故障检测与恢复

1. **健康检查**：ClickHouse 定期检查副本健康状态
2. **故障标记**：失败节点被标记为不可用
3. **自动恢复**：故障节点恢复后自动同步缺失数据
4. **副本重新平衡**：自动重新分配数据

### 3.5 分片 (Sharding)

数据分片策略：

| 分片方式 | 说明 |
|----------|------|
| Hash 分片 | 按 key 哈希值分片，数据均匀分布 |
| 范围分片 | 按时间或范围分片，适合时序数据 |
| 手动分片 | 自定义分片规则 |

### 3.6 核心概念速查表

| 概念 | 说明 |
|------|------|
| Database | 数据库，表的集合 |
| Table | 表，由引擎管理的数据结构 |
| Part | 数据片段，MergeTree 的物理存储单位 |
| Partition | 分区，逻辑数据划分 |
| Shard | 分片，数据子集 |
| Replica | 副本，数据冗余 |
| Merge | 后台合并任务 |
| Mutation | 数据修改操作（ALTER UPDATE/DELETE） |
| TTL | 数据生命周期 |
| Skein | 合并树的任务调度单位 |
| Query Log | 查询日志 |

---

## 4. 单机部署

### 4.1 系统要求

| 组件 | 最低要求 | 推荐配置 |
|------|----------|----------|
| CPU | 4 核 | 32+ 核 |
| 内存 | 8 GB | 64+ GB |
| 磁盘 | 100 GB SSD | 1+ TB NVMe SSD |
| 网络 | 1 Gbps | 10 Gbps |

### 4.2 RPM 安装（CentOS/RHEL）

```bash
# 添加官方仓库
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://packages.clickhouse.com/rpm/clickhouse.repo

# 安装 ClickHouse
sudo yum install -y clickhouse-server clickhouse-client

# 启动服务
sudo systemctl start clickhouse-server
sudo systemctl enable clickhouse-server

# 检查状态
sudo systemctl status clickhouse-server

# 连接客户端
clickhouse-client
```

### 4.3 DEB 安装（Debian/Ubuntu）

```bash
# 添加仓库
sudo apt-get install -y apt-transport-https ca-certificates dirmngr
sudo apt-key adv --keyserver keyserver.ubuntu.com --recv 8919F6BD2B48D754
echo "deb https://packages.clickhouse.com/deb stable main" | sudo tee /etc/apt/sources.list.d/clickhouse.list

# 安装
sudo apt-get update
sudo apt-install -y clickhouse-server clickhouse-client

# 启动
sudo systemctl start clickhouse-server
sudo systemctl enable clickhouse-server
```

### 4.4 Docker 部署

```bash
# 拉取镜像
docker pull clickhouse/clickhouse-server:latest

# 启动容器
docker run -d \
  --name clickhouse \
  -p 8123:8123 \
  -p 9000:9000 \
  -e CLICKHOUSE_DB=test \
  -e CLICKHOUSE_USER=admin \
  -e CLICKHOUSE_PASSWORD=password \
  clickhouse/clickhouse-server:latest

# 连接到客户端
docker exec -it clickhouse clickhouse-client
```

### 4.5 配置修改

```bash
# 主配置文件
/etc/clickhouse-server/config.xml

# 用户配置
/etc/clickhouse-server/users.xml

# 日志配置
/etc/clickhouse-server/config.xml 中 <logger> 部分

# 常用配置项
```

| 配置项 | 说明 | 默认值 |
|--------|------|--------|
| listen_host | 监听地址 | 127.0.0.1 |
| tcp_port | TCP 端口 | 9000 |
| http_port | HTTP 端口 | 8123 |
| max_connections | 最大连接数 | 4096 |
| max_memory_usage | 最大内存使用 | 10GB |
| path | 数据存储路径 | /var/lib/clickhouse |
| log_path | 日志路径 | /var/log/clickhouse |

### 4.6 基本操作

```sql
-- 创建数据库
CREATE DATABASE IF NOT EXISTS mydb;

-- 创建表
CREATE TABLE mydb.my_table (
    id UInt64,
    name String,
    created_at DateTime
) ENGINE = MergeTree()
ORDER BY id
PARTITION BY toYYYYMM(created_at)
TTL created_at + INTERVAL 3 MONTH;

-- 插入数据
INSERT INTO mydb.my_table VALUES (1, 'Alice', now());
INSERT INTO mydb.my_table SELECT * FROM external_table;

-- 查询数据
SELECT * FROM mydb.my_table WHERE id > 100;

-- 查看系统表
SELECT * FROM system.tables WHERE database = 'mydb';
SELECT * FROM system.part WHERE database = 'mydb';
SELECT * FROM system.metrics;
```

### 4.7 数据导入

```bash
# 从 CSV 导入
clickhouse-client --query "INSERT INTO mydb.my_table FORMAT CSV" < data.csv

# 从 JSON 导入
clickhouse-client --query "INSERT INTO mydb.my_table FORMAT JSONEachRow" < data.json

# 从文件导入
cat data.csv | clickhouse-client --query "INSERT INTO mydb.my_table FORMAT CSV"
```

---

## 5. 集群部署

### 5.1 集群架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                         ClickHouse 集群                              │
│                                                                      │
│  ┌─────────────┐          ┌─────────────┐          ┌─────────────┐  │
│  │  Shard 1   │          │  Shard 2   │          │  Shard 3   │  │
│  │ ┌────┐┌────┐│          │ ┌────┐┌────┐│          │ ┌────┐┌────┐│  │
│  │ │ R1 ││ R2 ││          │ │ R1 ││ R2 ││          │ │ R1 ││ R2 ││  │
│  │ └────┘└────┘│          │ └────┘└────┘│          │ └────┘└────┘│  │
│  └──────┬──────┘          └──────┬──────┘          └──────┬──────┘  │
│         │                        │                        │        │
│         └────────────────────────┼────────────────────────┘        │
│                                  │                                 │
│                         ┌────────┴────────┐                       │
│                         │   ZooKeeper     │                       │
│                         │     Cluster      │                       │
│                         └─────────────────┘                       │
└─────────────────────────────────────────────────────────────────────┘
```

### 5.2 ZooKeeper 集群部署

```bash
# ZooKeeper 配置文件
cat > /opt/zookeeper/conf/zoo.cfg << EOF
tickTime=2000
dataDir=/var/lib/zookeeper
clientPort=2181
initLimit=10
syncLimit=5
server.1=zk1:2888:3888
server.2=zk2:2888:3888
server.3=zk3:2888:3888
EOF

# 设置 myid
echo "1" > /var/lib/zookeeper/myid  # 对应 server.1

# 启动 ZooKeeper
docker run -d --name zk1 -e ZOOKEEPER_SERVER_ID=1 \
  -v /opt/zookeeper/conf:/conf \
  -p 2181:2181 zookeeper:3.8
```

### 5.3 ClickHouse 集群配置

```xml
<!-- /etc/clickhouse-server/config.d/cluster.xml -->
<clickhouse>
    <remote_servers>
        <my_cluster>
            <!-- 分片配置 -->
            <shard>
                <internal_replication>true</internal_replication>
                <replica>
                    <host>ch1</host>
                    <port>9000</port>
                    <user>default</user>
                    <password></password>
                </replica>
                <replica>
                    <host>ch1-replica2</host>
                    <port>9000</port>
                    <user>default</user>
                    <password></password>
                </replica>
            </shard>
            <shard>
                <internal_replication>true</internal_replication>
                <replica>
                    <host>ch2</host>
                    <port>9000</port>
                    <user>default</user>
                    <password></password>
                </replica>
            </shard>
            <shard>
                <internal_replication>true</internal_replication>
                <replica>
                    <host>ch3</host>
                    <port>9000</port>
                    <user>default</user>
                    <password></password>
                </replica>
            </shard>
        </my_cluster>
    </remote_servers>
</clickhouse>
```

### 5.4 创建分布式表

```sql
-- 在每个分片上创建本地表
CREATE TABLE mydb.local_table (
    id UInt64,
    name String,
    created_at DateTime
) ENGINE = MergeTree()
ORDER BY id
PARTITION BY toYYYYMM(created_at);

-- 创建分布式表
CREATE TABLE mydb.distributed_table AS mydb.local_table
ENGINE = Distributed(my_cluster, mydb, local_table, rand());

-- 写入分布式表（数据会自动分片）
INSERT INTO mydb.distributed_table VALUES (1, 'Alice', now());

-- 从分布式表查询（自动合并结果）
SELECT * FROM mydb.distributed_table WHERE id > 100;
```

### 5.5 集群部署步骤

```bash
# 1. 安装 ClickHouse（所有节点）
sudo yum install -y clickhouse-server clickhouse-client

# 2. 配置集群
cat > /etc/clickhouse-server/config.d/cluster.xml << 'EOF'
<clickhouse>
    <remote_servers>
        <production_cluster>
            <shard>
                <internal_replication>true</internal_replication>
                <replica>
                    <host>ch-node1</host>
                    <port>9000</port>
                </replica>
            </shard>
            <shard>
                <internal_replication>true</internal_replication>
                <replica>
                    <host>ch-node2</host>
                    <port>9000</port>
                </replica>
            </shard>
        </production_cluster>
    </remote_servers>
</clickhouse>
EOF

# 3. 配置 ZooKeeper
cat > /etc/clickhouse-server/config.d/zookeeper.xml << 'EOF'
<clickhouse>
    <zookeeper>
        <node>
            <host>zk1</host>
            <port>2181</port>
        </node>
        <node>
            <host>zk2</host>
            <port>2181</port>
        </node>
        <node>
            <host>zk3</host>
            <port>2181</port>
        </node>
    </zookeeper>
</clickhouse>
EOF

# 4. 重启服务
sudo systemctl restart clickhouse-server

# 5. 验证集群状态
clickhouse-client --query "SELECT * FROM system.clusters"
```

### 5.6 副本配置

```sql
-- 创建带副本的表
CREATE TABLE mydb.replicated_table (
    id UInt64,
    name String,
    created_at DateTime
) ENGINE = ReplicatedMergeTree(
    '/clickhouse/tables/{shard}/replicated_table',
    '{replica}'
)
ORDER BY id
PARTITION BY toYYYYMM(created_at);
```

参数说明：
- `/clickhouse/tables/{shard}/replicated_table` - ZooKeeper 路径
- `{replica}` - 副本名称，通常为 hostname

---

## 6. 常用操作与维护

### 6.1 性能优化

#### 分区设计

```sql
-- 按月分区示例
CREATE TABLE mydb.events (
    event_id UInt64,
    event_time DateTime,
    payload String
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(event_time)
ORDER BY (event_id, event_time);

-- TTL 示例：3个月后删除
CREATE TABLE mydb.events_ttl (
    event_id UInt64,
    event_time DateTime,
    payload String
) ENGINE = MergeTree()
ORDER BY event_id
TTL event_time + INTERVAL 3 MONTH;
```

#### 物化视图加速查询

```sql
-- 创建物化视图
CREATE MATERIALIZED VIEW mydb.agg_hourly
ENGINE = SummingMergeTree()
ORDER BY (hour, metric)
AS SELECT
    toStartOfHour(event_time) AS hour,
    count() AS cnt,
    sum(value) AS total_value
FROM mydb.events
GROUP BY hour;
```

### 6.2 监控

```sql
-- 查看 Query Log
SELECT * FROM system.query_log ORDER BY event_time DESC LIMIT 10;

-- 查看 Merge 日志
SELECT * FROM system.merge_log ORDER BY event_time DESC LIMIT 10;

-- 查看指标
SELECT * FROM system.metrics;
SELECT * FROM system.events;

-- 查看资源使用
SELECT * FROM system.metrics WHERE metric LIKE '%Memory%';

-- 查看副本状态
SELECT * FROM system.replicas WHERE database = 'mydb';

-- 查看分片信息
SELECT * FROM system.clusters;
SELECT * FROM system.shards;
```

### 6.3 备份与恢复

```bash
# 备份数据目录
sudo tar -czf clickhouse_backup_$(date +%Y%m%d).tar.gz /var/lib/clickhouse/data/

# 恢复
sudo tar -xzf clickhouse_backup_20260501.tar.gz -C /var/lib/clickhouse/

# 使用 clickhouse-backup 工具（推荐）
clickhouse-backup create backup_name
clickhouse-backup restore backup_name
```

### 6.4 常见问题排查

```bash
# 查看错误日志
sudo tail -f /var/log/clickhouse-server/clickhouse-server.log

# 检查端口占用
netstat -tlnp | grep clickhouse

# 检查磁盘空间
df -h

# 查看内存使用
clickhouse-client --query "SELECT * FROM system.metrics WHERE metric LIKE '%Memory%'"

# 检查 ZooKeeper 连接
clickhouse-client --query "SELECT * FROM system.zookeeper WHERE path='/'"
```

### 6.5 常用 SQL 函数

```sql
-- 时间函数
SELECT now(), today(), yesterday();
SELECT toStartOfHour(now()), toStartOfDay(now());
SELECT dateDiff('day', date1, date2);

-- 聚合函数
SELECT count(), sum(column), avg(column), max(column), min(column);
SELECT uniqExact(user_id) FROM table;

-- 数组函数
SELECT arrayJoin([1,2,3]);
SELECT arrayFilter(x -> x > 0, [-1, 0, 1, 2]);

-- 字符串函数
SELECT upper('hello'), lower('WORLD');
SELECT extractAll('abc123def456', '[0-9]+');
```

---

## 附录：配置参考

### config.xml 核心配置

```xml
<clickhouse>
    <!-- 网络配置 -->
    <listen_host>::</listen_host>
    <http_port>8123</http_port>
    <tcp_port>9000</tcp_port>

    <!-- 路径配置 -->
    <path>/var/lib/clickhouse/</path>
    <temp_path>/var/lib/clickhouse/tmp/</temp_path>

    <!-- 日志配置 -->
    <logger>
        <level>information</level>
        <log>/var/log/clickhouse-server/clickhouse-server.log</log>
        <errorlog>/var/log/clickhouse-server/clickhouse-server.err.log</errorlog>
    </logger>

    <!-- 资源限制 -->
    <max_memory_usage>16GB</max_memory_usage>
    <max_connections>4096</max_connections>
    <keepAliveTimeout>3000</keepAliveTimeout>

    <!-- 压缩配置 -->
    <compression>
        <case>
            <min_part_size>1000000000</min_part_size>
            <min_part_ratio>0.01</min_part_ratio>
            <method>zstd</method>
        </case>
    </compression>
</clickhouse>
```

---

## 参考链接

- [ClickHouse 官方文档](https://clickhouse.com/docs/zh/)
- [ClickHouse GitHub](https://github.com/ClickHouse/ClickHouse)
- [ClickHouse 社区](https://github.com/ClickHouse/ClickHouse/discussions)