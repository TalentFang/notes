# asap-log-purge 模块详解

## 概述

`asap-log-purge` 是日志清理服务，当磁盘使用率超过阈值时，自动清理各服务日志文件，防止磁盘空间耗尽。

## 目录结构

```
asap-log-purge/
├── log_purge.py            # 核心逻辑
├── log_purge.cfg           # 配置文件
└── log_purge.cfg.template # 配置模板
```

## 工作流程

```
1. 从数据库读取 DISK_PERCENT 阈值
   ↓
2. 获取当前 /data 磁盘使用率
   ↓
3. 若超过阈值，查询日志路径列表
   ↓
4. 根据服务类型采用不同清理策略
   ↓
5. 清空日志文件
```

## 核心功能

### 1. 阈值检测

- 从数据库 `SSA.SYS_CFG` 表读取 `LOG_DELETE` 配置获取 `DISK_PERCENT` 阈值
- 获取当前 `/data` 磁盘使用率
- 若超过阈值，才执行清理操作

### 2. 获取日志路径列表

查询 `SSA.MONITOR_PROCESS_CMD` 表获取各服务的日志路径列表。

### 3. 清理策略

根据不同服务类型采用不同清理策略：

| 服务类型 | 清理方式 |
|----------|----------|
| **spark/flink/livy/hadoop/kafka** | 清空目录内所有日志文件 |
| **elasticsearch** | 清空主日志文件 + 所有 `gc.log*` 文件 |
| **其他** | 清空单个日志文件 |

### 4. 安全清空

使用 `sudo cat /dev/null > file` 命令安全清空而非删除文件，保持文件句柄和权限不变。

## 配置说明 (log_purge.cfg)

```ini
[db]
type = vastbase          # 数据库类型 (mysql/dm/vastbase/pgsql)
host = DB0               # 数据库地址
port = 15432            # VastBase 端口
user = root              # 用户名
pwd = maxs.PDG~2022      # 密码
name = SSA               # 数据库名
```

## 数据库支持

| db_type | 数据库 | 连接方式 |
|---------|--------|----------|
| `mysql` | MySQL | `pymysql.connect()` |
| `dm` | DM8（达梦） | `dmPython.connect()` |
| `vastbase` | VastBase | `psycopg2.connect()` |
| `postgresql` | PostgreSQL | `psycopg2.connect()` |
| 其他 | 默认 VastBase | `psycopg2.connect()` |

## 安全特性

1. **条件执行**: 仅在磁盘使用率超过阈值时才清理
2. **文件清空**: 使用 `cat /dev/null` 而非 `rm`，保持文件权限
3. **服务区分**: 不同服务采用不同的清理策略

## 使用场景

该模块主要用于：
1. **磁盘空间管理**: 防止日志文件占用过多磁盘空间
2. **自动化运维**: 无需人工干预，自动清理
3. **分级清理**: 针对不同服务采用适合的清理方式