# IPAAS 打包工具

## 概述

ipaas-packing 是一个**按需打包工具**，用于根据 CSV 配置文件自动化打包 IPAAS 产品（包含多种中间件和服务组件）。

### 功能特性

- 支持多种操作系统：CentOS、Euler、Kylin、UOS 等
- 支持多种 CPU 架构：x86、ARM
- 支持批量组件打包，包含依赖管理
- 支持 Ansible inventory hosts 文件自动生成
- 支持 ISO 和 build 两种打包模式

---

## 项目结构

```
ipaas-packing/
├── main.go              # 主入口程序
├── config.yaml          # 全局配置文件
├── go.mod / go.sum      # Go 依赖
├── hosts.py             # Ansible hosts 文件修改工具（Python）
│
├── config/              # 配置解析模块
│   └── main.go          # 配置结构体定义和解析
│
├── csv/                 # CSV 解析模块
│   ├── main.go          # CSV 入口
│   ├── filename.go      # 文件名解析
│   └── filerecord.go    # CSV 记录结构
│
├── packing/             # 打包核心模块
│   ├── main.go          # 打包入口
│   ├── prepare.go       # 目录准备
│   ├── build_ftp.go     # FTP 打包
│   └── template.go      # 模板生成
│
├── backup/              # 备份模块
│   └── main.go          # 备份入口
│
├── tools/               # 工具模块
│   ├── shell.go         # Shell 命令执行
│   └── log.go           # 日志工具
│
├── 503/605/606/607/608/ # 各版本的 CSV 配置目录
└── doc/                 # 文档目录
```

---

## 使用方法

### 基本语法

```bash
# 基本用法：指定 CSV 文件
go run main.go xxx.csv

# 带业务类型
go run main.go business xxx.csv
```

### CSV 文件命名规范

```
{产品名}_{操作系统}_{CPU架构}_{打包类型}_{版本}.csv

示例：
maxs_centos_x86_build_502.csv
maxs_euler_arm_build_608.csv
```

### CSV 文件格式

| 字段 | 说明 | 示例 |
|------|------|------|
| 组件名 | 组件标识 | mysql |
| 版本 | 可选，不填则用默认版本 | 5.7.36 |
| 部署模式 | 可选 | yarn |

CSV 行格式：
```
组件名,,                      # 无脚本，使用默认版本
组件名,"脚本内容"             # 带内联脚本
组件名,版本,部署模式          # 指定版本和模式
```

---

## 打包流程

### 流程图

```
┌─────────────────────────────────────────────────────────────────┐
│                         开始                                      │
└─────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────┐
│ 1. 命令行参数解析                                                 │
│    - 解析 CSV 文件名                                              │
│    - 识别操作类型（business / 空）                                │
└─────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────┐
│ 2. 解析 CSV 文件                                                  │
│    - 读取产品配置（名称/OS/CPU/版本/打包类型）                     │
│    - 读取组件列表（名称/版本/部署模式）                           │
└─────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────┐
│ 3. 解析 config.yaml                                              │
│    - 读取全局参数（远程机器、路径、hosts配置）                     │
│    - 匹配对应 OS 和 CPU 架构的配置                                │
│    - 替换 CSV 中的内联脚本（ipaas/buildScript/isoScript 等）      │
└─────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────┐
│ 4. 准备打包目录                                                   │
│    - 删除旧的 DEST 目录                                           │
│    - 创建新的 DEST 目录                                          │
└─────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
                    ┌─────────────┴─────────────┐
                    │                           │
                    ▼                           ▼
        ┌───────────────────┐       ┌───────────────────┐
        │   business 模式    │       │   普通模式        │
        └───────────────────┘       └───────────────────┘
                    │                           │
                    ▼                           ▼
        ┌───────────────────┐       ┌───────────────────┐
        │ 执行 IpaasScript   │       │ 执行 PreScript    │
        │ 复制 business 包   │       │ 复制 FTP 文件     │
        └───────────────────┘       │ 复制第三方组件    │
                    │              │ 执行 PostScript   │
                    │              └───────────────────┘
                    │                           │
                    │              ┌───────────┴───────────┐
                    │              ▼                       ▼
                    │    ┌─────────────────┐    ┌─────────────────┐
                    │    │ 执行 GitScript  │    │ 生成 Ansible    │
                    │    │ 拉取 GitLab 代码│    │ Hosts 模板      │
                    │    └─────────────────┘    └─────────────────┘
                    │              │                       │
                    │              └───────────┬───────────┘
                    │                          ▼
                    │              ┌─────────────────┐
                    │              │ 执行 BuildScript │
                    │              │ 构建最终产物     │
                    │              └─────────────────┘
                    │                          │
                    └──────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────┐
│ 5. ISO 打包（如需要）                                            │
│    - 执行 IsoScript 生成 ISO 镜像                                │
└─────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────┐
│ 6. 备份                                                          │
│    - 执行 BackupBuildScript                                      │
│    - 备份产物到指定路径                                          │
└─────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────┐
│                         结束                                      │
└─────────────────────────────────────────────────────────────────┘
```

### 详细步骤说明

#### 步骤 1: 命令行参数解析

```go
// 支持两种调用方式
// 1. ipaas-packing xxx.csv
// 2. ipaas-packing business xxx.csv

if len(os.Args) == 3 {
    operType = csvFile      // "business"
    csvFile = os.Args[2]     // 实际的 CSV 文件名
}
```

#### 步骤 2: 解析 CSV 文件

**产品信息解析**（从文件名提取）：
- `Name`: 产品名（如 maxs、linkone）
- `Os`: 操作系统（如 centos、euler、kylin）
- `Cpu`: CPU 架构（如 x86、arm）
- `Packing`: 打包类型（build / iso）
- `Version`: 版本号

**组件列表解析**（从文件内容提取）：
```csv
mysql,,           → Record{Name: "mysql"}
mongodb,5.4.6,    → Record{Name: "mongodb", Version: "5.4.6"}
kafka,,           → Record{Name: "kafka"}
```

#### 步骤 3: 解析 config.yaml

**全局参数**：
```yaml
param:
  - name: src          # 源文件目录
  - name: dest         # 目标打包目录
  - name: REMOTE_VM    # 远程打包机器 IP
  - name: ROOT_PASSWD  # 远程机器密码
  - name: HOSTS_FILE   # Ansible hosts 文件列表
  - name: HOSTS_SECTION # 组件类型列表
```

**OS 配置匹配**：
```yaml
os:
  - name: centos
    cpu: x86
    param:
      - name: BACK_UP_PATH
        value: /data/packing/backup/centos
    preScript: |       # 预处理脚本
      cp -rf $SRC/asap $DEST
    gitScript: |        # Git 拉取脚本
      sshpass -p $ROOT_PASSWD scp ...
    buildScript: |     # 构建脚本
      tar -zcvf asap.tar.gz ./asap/
```

#### 步骤 4: 准备打包目录

```go
func PreparePath(packingConfig config.PackingConfig, product csv.ProductInfo) error {
    rmPathCommand := "rm -rf " + packingConfig.Param["$DEST"]
    createPathCommand := "mkdir -p " + packingConfig.Param["$DEST"]
    tools.ExecShell(rmPathCommand, true)
    tools.ExecShell(createPathCommand, false)
    return nil
}
```

#### 步骤 5: 执行打包

**FTP 文件复制**：
1. 执行 `PreScript` - 复制基础文件（asap、images、install.sh 等）
2. 复制第三方组件（根据 CSV 组件列表）
3. 处理组件依赖关系
4. 执行 `PostScript` - 后处理

**GitLab 代码拉取**：
- 通过 SSH 从远程机器拉取 maxs-ops 仓库
- 拉取 python-offline 部署包

**模板生成**：
1. 根据 CSV 组件列表，删除不需要的 hosts 组
2. 替换版本号和部署模式
3. 使用 `hosts.py` 工具修改 Ansible inventory

**构建脚本**：
- 打包 thirdSoft.tar.gz
- 生成 asap.tar.gz
- 生成最终产物 ipaas_{OS}_{CPU}_1.0.tar.gz

#### 步骤 6: ISO 打包（如需要）

当 CSV 文件名包含 `iso` 时，执行 `IsoScript` 生成 ISO 镜像。

#### 步骤 7: 备份

执行 `BackupBuildScript` 和 `BackupIsoScript`，将产物复制到备份目录。

---

## 配置文件说明

### config.yaml 结构

```yaml
# 全局参数
param:
  - name: src              # 源文件基础路径
    value: /data/packing
  - name: dest             # 目标打包路径
    value: /data/packing/build
  - name: REMOTE_VM         # 远程打包机器
    value: 192.168.11.195
  - name: ROOT_PASSWD       # 远程机器密码
    value: maxs.PDG~2024

# 操作系统配置
os:
  - name: centos
    cpu: x86
    param:
      - name: BACK_UP_PATH
        value: /data/packing/backup/centos
      - name: BUILD_NAME
        value: build.5.0.1-CentOS-7.9
    preScript: |            # 构建前脚本
      cp -rf $SRC/asap $DEST
      cp -rf $SRC/images $DEST
    gitScript: |           # Git 拉取脚本
      sshpass -p $ROOT_PASSWD scp -r ...
    buildScript: |         # 构建脚本
      tar -zcvf thirdSoft.tar.gz ./thirdSoft/
    postScript: |           # 构建后脚本（可选）
    isoScript: |            # ISO 打包脚本（可选）
    third:                  # 第三方组件配置
      - name: mysql
        version: 5.7.36
        defaultVersion: 5.7.36
        filepath: rpms/mysql-5.7.36.tar.gz
        related: |          # 依赖的其他组件
          mysql
      - name: elasticsearch
        version: 7.16.3
        defaultVersion: 7.16.3
        filepath: rpms/elasticsearch-7.16.3.tar.gz
```

### 参数替换规则

在脚本中可以使用以下变量（大小写不敏感）：
- `$SRC`: 源目录（配置值 + 产品路径）
- `$DEST`: 目标目录（配置值 + 产品路径）
- `$PRODUCT`: 产品名称
- `$VERSION`: 产品版本
- `$OS`: 操作系统
- `$CPU`: CPU 架构
- `$PACKING`: 打包类型

---

## 组件版本说明

### 版本目录

| 目录 | 说明 |
|------|------|
| 503 | 旧版本（5.0.3） |
| 604 | 6.0.4 版本 |
| 605 | 6.0.5 版本 |
| 606 | 6.0.6 版本 |
| 607 | 6.0.7 版本 |
| 608 | 6.0.8 版本（最新） |

### 支持的操作系统

| 操作系统 | CPU | 说明 |
|----------|-----|------|
| centos | x86 | CentOS 7.9 |
| euler | x86/arm | 华为 EulerOS |
| kylin | x86/arm | 银河麒麟 |
| uos | x86/arm | 统信 UOS |
| redhat | x86 | RedHat EL8 |

---

## hosts.py 工具

Ansible inventory hosts 文件修改工具，支持三种操作：

```bash
# 删除整个组
python hosts.py delete {hosts文件} {组名}

# 修改版本号
python hosts.py version {hosts文件} {组名} {旧版本} {新版本}

# 添加部署模式
python hosts.py mode {hosts文件} {组名} {模式}
```

示例：
```bash
python hosts.py delete hosts flink
python hosts.py version hosts flink 1.16.0 1.13.0
python hosts.py mode hosts flink yarn
```

---

## 常用命令

### 本地打包测试

```bash
# 进入项目目录
cd /Users/fangkun/code/xdr/ipaas-packing

# 执行打包
go run main.go csv/maxs_centos_x86_build_502.csv

# 执行 business 模式打包
go run main.go business csv/maxs_centos_x86_build_502.csv
```

### 交叉编译

```bash
# Linux x86
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -o ipaas-packing main.go

# Linux ARM
CGO_ENABLED=0 GOOS=linux GOARCH=arm64 go build -o ipaas-packing main.go
```

---

## 注意事项

1. **远程机器配置**：确保 `config.yaml` 中的 `REMOTE_VM` 和 `REMOTE_USER`/`ROOT_PASSWD` 配置正确
2. **路径权限**：确保打包目录有读写权限
3. **SSH 信任**：首次运行需要确认 SSH 指纹，或使用 `StrictHostKeyChecking=no`
4. **组件依赖**：CSV 中组件的依赖关系在 `config.yaml` 的 `third[].related` 字段定义
5. **版本匹配**：CSV 中组件版本必须在 `config.yaml` 的 `third` 列表中存在