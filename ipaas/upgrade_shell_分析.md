# AIXDR 升级系统工程分析

> 基于 develop 分支，分析日期：2026-06-04

## 一、工程概览

这是一个 **AIXDR（扩展检测与响应）平台的离线升级/回滚系统**，通过 Bash 脚本 + Ansible 实现对多节点集群或单机的自动化升级与回滚。

## 二、目录结构

```
upgrade_shell/
├── bin/                          # 核心引擎
│   ├── upgrade.sh                # 主入口：升级/回滚入口
│   ├── rollback.sh               # 回滚入口
│   ├── engine_check.sh           # 引擎任务状态检查
│   ├── tools/                    # 工具函数库
│   │   ├── logger.sh             # 日志输出（带颜色）
│   │   ├── ansible.sh            # Ansible 封装（远程执行、MySQL/ES/CK 操作）
│   │   ├── business.sh           # 业务服务停止/恢复、License 检查
│   │   ├── engine.sh             # 引擎服务停止/启动（maxs-ts、Flink）
│   │   ├── es.sh                 # Elasticsearch 操作
│   │   ├── script.sh             # 目录内脚本顺序执行器
│   │   ├── service.sh            # systemd 服务管理
│   │   └── version.sh            # 版本工具
│   ├── pre.d/                    # 前置检查脚本（按文件名排序执行）
│   │   ├── 001_remove_version_file.sh  # 清理旧版本文件
│   │   ├── 002_handle_history_issues.sh # 清理历史遗留检查项
│   │   ├── 010_check.sh          # 全量预检查（核心）
│   │   └── 015_unarchive.sh      # 解压 business.tar.gz
│   ├── upgrade.d/                # 核心升级脚本
│   │   └── 004_backup_and_upgrade.sh  # 备份 + 底座升级 + 业务升级
│   ├── post.d/                   # 后置脚本
│   │   ├── 002_platform.sh       # etcd备份、配置同步、SOAR重启
│   │   └── 003_engine.sh         # 引擎启动（清理旧任务、重启 maxs-ts）
│   ├── playbooks/                # Ansible Playbooks
│   └── build/                    # 构建脚本
├── business.d/                   # 各业务组件升级脚本
│   ├── maxs-pm/upgrade/          # 每个业务有独立的 upgrade/rollback 目录
│   ├── maxs-monitor/upgrade/
│   ├── maxs-rule/upgrade/ + rollback/
│   ├── maxs-ts/upgrade/ + rollback/
│   ├── ... (共 ~18 个业务组件)
│   └── aiagent/upgrade/ + rollback/
├── platform.d/                   # 底座（平台中间件）升级脚本
│   ├── maxs-ops/start.sh         # 运维工具平台
│   ├── python-offline/build.sh   # Python 离线包
│   └── third-soft/build.sh       # 第三方软件
└── .OLD_VERSION / .SUCCESS_NUMBER / .FAILED_BACK_NUMBER  # 状态标记文件
```

## 三、升级执行流程

### 入口命令

```bash
# 升级
sh upgrade.sh start [back_number]
# 回滚
sh upgrade.sh rollback [back_number]
```

`back_number` 是时间戳（如 `20260604153000`），用于标识本次升级，作为备份目录后缀。

### 升级流程（5个阶段）

```
┌─────────────────────────────────────────────────────┐
│  0. License 检查（checkLicense）                     │
│     → MySQL 查询 UPMS_LICENSE 表                     │
│     → Redis 查询 License 过期时间                     │
│     → 未激活或过期则拒绝升级                           │
├─────────────────────────────────────────────────────┤
│  1. 前置脚本（pre.d/，按文件名排序）                   │
│     001: 清理旧版本文件                               │
│     002: 清理历史遗留检查项                           │
│     010: 全量预检查 ← 核心                            │
│       → user_check: root 权限检查                     │
│       → version_check: 当前版本校验（支持 6.0.5 和 6.0.x）│
│       → os_check: 操作系统架构匹配                    │
│       → python_check: Python 环境健康检查              │
│       → disk_check: /data 使用率 <70%, 剩余 >100G     │
│       → mem_check: 内存使用率 <80%                    │
│       → cpu_check: CPU 使用率 <80%                    │
│       → io_check: IO 使用率 <70%（带重试）             │
│       → config_check: Docker registry/SSH/防火墙检查   │
│       → business_check: 各业务组件预检查脚本            │
│       → engine_stop: 停止 maxs-ts + Flink 任务        │
│       → business_stop_and_record: 停止所有K8S Pods    │
│         并记录副本数到 default_pod_replicas.txt        │
│     015: 解压 business.tar.gz                        │
├─────────────────────────────────────────────────────┤
│  2. 核心升级（upgrade.d/）                            │
│     004_backup_and_upgrade.sh:                       │
│       → MySQL 全库备份（backup.sql）                  │
│       → 第三方安装包备份                               │
│       → 组件配置全量备份（nginx/mysql/redis/k8s...）   │
│       → 底座包备份（maxs-ops）                        │
│       → 业务包备份（deploy 目录）                     │
│       → 更新 deploy 目录（ansible playbook）           │
│       → 底座升级（maxs-ops, python-offline, third-soft）│
│       → 业务升级（按固定顺序逐个组件）                  │
│         每个组件：                                     │
│           1. License 检查（NEED_ENABLES）              │
│           2. 执行 SQL（mysql.sql / es.sql / clickhouse.sql）│
│           3. 执行 upgrade_pre.sh（如果有）             │
│           4. 执行 start.sh（组件自定义升级逻辑）        │
│           5. 停旧 Pod → 重新部署（deploy.sh -d/-a）    │
│           6. 执行 upgrade_post.sh（如果有）            │
│       → 恢复物理机 systemd 服务                       │
├─────────────────────────────────────────────────────┤
│  3. 后置脚本（post.d/）                               │
│     002: etcd 备份、删除异常 Pod、更新检查项、         │
│          同步 maxs-ops 到备用机、flush ARP、           │
│          重启 SOAR 预编排                              │
│     003: 引擎启动（清理旧任务表、重启 maxs-ts）         │
├─────────────────────────────────────────────────────┤
│  4. 收尾                                             │
│     → 写入升级成功记录到 MySQL VERSION_HISTORY         │
│     → .FAILED_BACK_NUMBER → .SUCCESS_NUMBER           │
│     → 输出升级完成信息                                │
└─────────────────────────────────────────────────────┘
```

### 回滚流程

```
┌─────────────────────────────────────────────────────┐
│  0. 前置检查（与升级类似）                             │
│     → ip_check: IP 列表一致性检查                     │
│     → disk/mem/cpu/io_check                          │
├─────────────────────────────────────────────────────┤
│  1. 停止所有业务服务（business_stop）                  │
├─────────────────────────────────────────────────────┤
│  2. 恢复 MySQL（从 backup.sql 全库恢复）              │
├─────────────────────────────────────────────────────┤
│  3. 恢复 deploy 软链接（deploy_605 / deploy_607）     │
├─────────────────────────────────────────────────────┤
│  4. 底座回滚（rollback.d/ 或 rollback.sh）            │
│     → python-offline 重新部署                        │
│     → maxs-ops 从备份恢复                            │
├─────────────────────────────────────────────────────┤
│  5. 业务回滚（按固定顺序）                             │
│     → rollback_pre.sh（如果有）                       │
│     → rollback.sh（组件自定义回滚逻辑）               │
│     → SQL 回滚（ES/ClickHouse）                      │
│     → 停旧 Pod → 重新部署                            │
│     → rollback_post.sh（如果有）                     │
├─────────────────────────────────────────────────────┤
│  6. 版本号回写 MySQL                                  │
│  7. 恢复所有服务、重启 Pod、更新检查项                  │
└─────────────────────────────────────────────────────┘
```

## 四、关键设计模式

### 1. 脚本顺序执行器（`bin/tools/script.sh`）

```bash
run_scripts_in_dir_orderly() {
    find "$script_path" -name "*.sh" | sort | while read file; do
        sudo -E bash -ex "$file" "$back_number" "$cluster"
    done
}
```

通过文件名编号（`001_`, `010_`, `015_`）控制执行顺序，新增步骤只需添加新编号文件。

### 2. 集群/单机统一抽象

通过 `cluster` 变量（0=单机，1=集群）控制分支：
- 单机：本地直接执行命令
- 集群：通过 `exec_remote` + Ansible 在所有节点并行执行

### 3. 升级记录文件（`upgrade_record.txt`）

记录已升级的组件，回滚时用于判断哪些组件需要回滚。每回滚一个组件就从文件中删除对应行。

### 4. License 驱动的功能开关

- `NEED_ENABLES` 数组：决定哪些业务组件需要部署（如 BI、SOAR、数字人等）
- `COMP_ENABLES` 数组：记录组件启用状态
- `UPMS_LICENSE` 和 `UPMS_LICENSE_RELATION` 表：维护新旧 PID 映射关系

### 5. 状态标记文件

| 文件 | 用途 |
|------|------|
| `.OLD_VERSION` | 上次升级前的版本号 |
| `.FAILED_BACK_NUMBER` | 升级失败的备份编号（阻止重复升级） |
| `.SUCCESS_NUMBER` | 升级成功的备份编号（回滚入口） |
| `upgrade.pid` | 防止并发执行 |

### 6. 业务组件升级生命周期

每个业务组件（`business.d/<name>/upgrade/`）可包含：
- `upgrade_pre.sh` — 升级前自定义逻辑
- `start.sh` — 核心升级逻辑（替换配置、上传文件等）
- `upgrade_post.sh` — 升级后自定义逻辑
- `sql/mysql.sql` — MySQL DDL/DML
- `sql/es.sql` — Elasticsearch 索引操作
- `sql/clickhouse.sql` — ClickHouse DDL

### 7. 部署方式差异

| 组件类型 | 部署命令 | 说明 |
|----------|----------|------|
| 特殊组件（rule-engine, data-process-engine, maxs-ts 等） | `deploy.sh -c <name> -a` | 构建并部署 |
| 普通 Java 业务 | `deploy.sh -c <name> -d` | 仅部署（推送镜像到 K8S） |
| 静态服务（maxs-upgrade, bi-selenium 等） | 跳过 | 不需要重新打镜像 |
| 物理机服务（maxs-datareport 等） | `file_upload_exec_tool.sh` | 文件分发 + systemctl 重启 |

## 五、数据流总结

```
升级包(tar.gz)
  → 解压到 upgrade_shell/
  → pre.d 检查环境
  → 备份(MySQL + 配置 + 安装包)
  → 更新 /data/asap/deploy/
  → 升级底座(maxs-ops, python-offline, third-soft)
  → 升级业务(18+ 组件, 按固定顺序)
  → 后置(同步配置, 启动引擎)
  → 记录版本历史
```

整个系统是一个**离线、有状态、可回滚的自动化升级框架**，核心依赖 Ansible 进行远程编排，通过编号脚本实现可扩展的升级流水线。

## 六、支持的版本升级路径

- 6.0.5 → 6.0.8（支持跨版本，会额外执行 606/607 的 SQL）
- 6.0.7 → 6.0.8
- 其他版本需匹配 `SUPPORT_UPGRADE_VERSION` 变量
