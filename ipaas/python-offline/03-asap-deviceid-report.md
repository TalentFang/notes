# asap-deviceid-report 模块详解

## 概述

`asap-deviceid-report` 是设备 ID 上报服务，负责采集服务器硬件信息生成唯一设备 ID，并上报到 Redis 许可证映射表中，用于软件授权验证。

## 目录结构

```
asap-deviceid-report/
├── deviceid_report.py            # 核心逻辑
├── deviceid_report.cfg          # 配置文件
└── deviceid_report.cfg.template # 配置模板
```

## 工作流程

```
1. 获取设备ID
   ↓
2. 查询数据库许可证
   ↓
3. 检查设备ID重复（防盗用）
   ↓
4. 上报Redis
   ↓
5. 清理过期数据
```

## 核心功能

### 1. 获取设备 ID

根据 `product_name` 配置调用不同的硬件标识获取方法：

| 产品类型 | 获取方式 |
|----------|----------|
| `AUTO/AUTOEX` | 综合多种方式（SMBIOS 序列号、DMI ID、MAC 地址等） |
| `DELL/BAODE` | 使用 `dmidecode -s system-serial-number` |
| `LIHUA/SOFT/EXUSB` | 获取第一个非虚拟网卡 MAC，通过 CRC32+Base36 算法生成 7 位设备 ID |
| `XINGHAN` | 读取 `board_serial` |
| 其他 | 自定义产品直接返回 `'1234567'` |

### 2. 查询数据库许可证

连接 MySQL/DM8/VastBase 从 `UPMS_LICENSE` 表获取该设备 ID 关联的许可证数据（Base64 编码）。

### 3. 检查设备 ID 重复（防盗用）

从 Redis 检查是否存在相同设备 ID 但不同 IP 的情况（可能被盗用）。

### 4. 上报 Redis

将设备信息写入 Redis Hash:
```python
{
  "hostIp": ...,        # 主机 IP
  "hostName": ...,      # 主机名
  "deviceId": ...,      # 设备 ID
  "licenseUUID": ...,   # 许可证 UUID
  "licMap": {...},      # 许可证映射
  "isLicense": bool,    # 是否有许可证
  "hostOs": ...,        # 操作系统
  "cpuArchitecture": ...,  # CPU 架构
  "cpuModel": ...       # CPU 型号
}
```

- Redis Key 过期时间: 62 秒
- 循环重试: 3 次

### 5. 清理过期数据

删除 300 秒内无心跳的其他设备 ID 记录。

## 配置说明 (deviceid_report.cfg)

```ini
[db]
type = vastbase          # 数据库类型 (mysql/dm/vastbase)
host = DB0               # 数据库地址
port = 15432            # VastBase 端口
user = root              # 用户名
pwd = maxs.PDG~2022      # 密码
name = UPMS              # 数据库名

[redis]
type = standalone        # Redis 模式 (cluster/sentinel/standalone)
host = REDIS             # Redis 地址
port = 6379              # Redis 端口
pwd = !QAZxsw2#EDC(0Ol1) # Redis 密码
db = 0

[product]
name = AUTOEX            # 产品名称
license_host = 127.0.0.1 # 许可证服务器 IP
```

## 数据库支持

| db_type | 数据库 | 连接方式 |
|---------|--------|----------|
| `mysql` | MySQL | `pymysql.connect()` |
| `dm` | DM8（达梦） | `dmPython.connect()` |
| `vastbase` | VastBase | `psycopg2.connect()` |
| 其他 | 默认 VastBase | `psycopg2.connect()` |

## Redis 数据结构

**Key**: `REDIS_HOST`

**Hash field**: `device_id`

**过期时间**: 62 秒（每次上报刷新）

## 使用场景

该模块主要用于：
1. **软件授权验证**: 确保软件只在授权的设备上运行
2. **设备追踪**: 追踪软件部署的设备信息
3. **许可证防盗**: 检测相同设备 ID 在不同 IP 的异常情况