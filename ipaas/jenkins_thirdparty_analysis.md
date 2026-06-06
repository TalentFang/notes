# Jenkins thirdSoft 维护机制分析

## 1. 构建流程架构

```
build_daily_xdr6.0.4 (MultiJob)
    │
    ├── maxs-dependencies        → Maven 依赖构建
    ├── maxs-pm-api-linkone     → PM API 构建
    ├── maxs-soar-api-linkone   → SOAR API 构建
    ├── maxs-asset-api-linkone  → Asset API 构建
    ├── maxs-intel-api-linkone  → Intel API 构建
    ├── maxs-commons-xdr         → 公共模块构建
    ├── maxs-monitor            → 监控模块构建
    ├── python-offline           → Python 离线包
    ├── maxs-ops                → 部署脚本 (只是 git clone)
    ├── ipaas-packing            → **打包配置 (关键)**
    └── ... (30+ 个子 job)
```

---

## 2. 关键 Jenkins Job 分析

| Job 名称 | 类型 | 功能 | 仓库 |
|---------|------|------|------|
| `build_daily_xdr6.0.4` | MultiJob | 主构建入口，编排所有子 job | - |
| `maxs-dependencies` | Maven | 依赖管理，生成 pom 并上传 Maven 仓库 | maxs-dependencies.git |
| `maxs-ops` | FreeStyle | 仅拉取部署脚本代码 | maxs-ops.git |
| `ipaas-packing` | FreeStyle | **打包配置**，定义安装介质内容 | ipaas-packing.git |
| `python-offline` | FreeStyle | Python 离线包构建 | - |

---

## 3. thirdSoft 来源分析

### 3.1 maxs-dependencies 日志分析

```
Started by upstream project "build_daily_xdr6.0.4" build number 433
Building in workspace /var/jenkins_home/workspace/maxs-dependencies

[INFO] Building maxs-dependencies 2.1
[INFO] --- maven-install-plugin:2.4:install (default-install) @ maxs-dependencies ---
[INFO] Installing pom.xml to /var/jenkins_home/repo/com/ais/framework/maxs-dependencies/2.1/

[INFO] Deployment in http://10.21.13.12:11005/repository/ais-repo/ (id=ais-repo,uniqueVersion=true)
```

**结论**：`maxs-dependencies` 只是 Maven 依赖管理，不涉及 thirdSoft 二进制文件。

### 3.2 ipaas-packing 分析

**仓库地址**: `https://gitlab.asiainfo-sec.com/maxs-ops/ipaas-packing`
**分支**: `release_pre`

**配置关键内容**:
```xml
<builders>
  <jenkins.plugins.publish__over__ssh.BapSshBuilderPlugin>
    <configName>192.168.11.197</configName>
    <execCommand>
      \cp -f /home/opt/jenkens/workspace/ipaas-packing/config.yaml /opt/packing_502/ipaas-packing/
      \cp -f /home/opt/jenkens/workspace/ipaas-packing/hosts.py /opt/packing_502/ipaas-packing/
      cd /opt/packing_502/ipaas-packing
      sed -i 's#/data/packing#/opt/packing_502#g' config.yaml
      ...
    </execCommand>
  </jenkins.plugins.publish__over__ssh.BapSshBuilderPlugin>
</builders>
```

**结论**：`ipaas-packing` 负责将打包配置传输到打包服务器，但不直接生成 thirdSoft.tar.gz。

---

## 4. thirdSoft 打包流程

```
┌─────────────────────────────────────────────────────────────────┐
│                    thirdSoft 维护流程                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. 构建阶段                                                     │
│     ipaas-packing job 执行                                        │
│         ↓                                                         │
│     config.yaml 中定义打包规则                                     │
│         ↓                                                         │
│     生成的安装介质包含 thirdSoft.tar.gz                             │
│         ↓                                                         │
│     上传到共享存储或目标机器                                        │
│                                                                  │
│  2. 部署阶段 (maxs-ops)                                           │
│     install.sh 的 transfer() 函数                                │
│         ↓                                                         │
│     通过 rsync/scp 传输到目标机器                                   │
│         ↓                                                         │
│     解压到 /data/asap/thirdSoft/                                  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5. elasticsearch_exporter 路径配置

根据 maxs-ops 本地代码分析：

| 文件                                           | tar_dir 配置                                                                 |
| -------------------------------------------- | -------------------------------------------------------------------------- |
| `roles/elasticsearch_exporter/vars/main.yml` | `/data/asap/thirdSoft/elasticsearch_exporter.tar.gz`                       |
| `roles/node_exporter/vars/main.yml`          | `/data/asap/thirdSoft/node_exporter-1.9.1.linux-{{ architecture }}.tar.gz` |
| `roles/clickhouse_exporter/vars/main.yml`    | `/data/asap/thirdSoft/clickhouse_exporter.tar.gz`                          |
| `roles/kafka_exporter/vars/main.yml`         | `/data/asap/kafka_exporter-1.6.0.linux-{{ architecture }}.tar.gz`          |

---

## 6. thirdSoft 路径配置

根据 `ipaas-packing/config.yaml` 第 2-5 行：

```yaml
param:
  - name: src
    value: /data/packing           # 源目录
  - name: dest
    value: /data/packing/build     # 目标目录
```

**关键路径**：
- **thirdSoft 源路径**: `/data/packing/thirdSoft/`
- **thirdSoft 目标路径**: `/data/packing/build/thirdSoft/`
- **打包服务器**: `10.21.10.197` (192.168.11.197)

---

## 7. 新增 elasticsearch_exporter.tar.gz 完整步骤

### 7.1 thirdSoft 传输方式

**是人工或脚本将 thirdSoft 传输到 Jenkins 服务器的。**

分析依据：
- `config.yaml` 的 `preScript` 中明确列出要复制的文件：
  ```bash
  cp $SRC/thirdSoft/bash-5.1.16.tar.gz $DEST/thirdSoft
  cp $SRC/thirdSoft/openssl-1.1.1q.tar.gz $DEST/thirdSoft
  ...
  ```
- 这些文件需要预先存在于 `/data/packing/thirdSoft/` 目录
- 传输方式可能是人工上传或专门的文件同步脚本

### 7.2 新增步骤

| 步骤 | 操作 | 说明 |
|------|------|------|
| **1** | **上传 elasticsearch_exporter.tar.gz** | 人工/脚本上传到 `/data/packing/thirdSoft/` |
| **2** | **修改 config.yaml** | 在 es 配置中添加 exporter 路径 |
| **3** | **提交代码** | 将修改推送到 `release_pre` 分支 |
| **4** | **触发构建** | Jenkins 执行 `ipaas-packing` job |

### 7.3 config.yaml 修改示例

**位置**: `config.yaml` 第 168-170 行

**修改前**：
```yaml
- name: es
  filepath: thirdSoft/elasticsearch-7.17.5-jdk.tar.gz
  sql: delete from CLUSTER_SERVICE_BASIC where ID=104;
```

**修改后**：
```yaml
- name: es
  filepath: |
    thirdSoft/elasticsearch-7.17.5-jdk.tar.gz
    thirdSoft/elasticsearch_exporter-1.7.0.linux-amd64.tar.gz
  sql: delete from CLUSTER_SERVICE_BASIC where ID=104;
```

### 7.4 thirdSoft 源目录内容示例

```
/data/packing/thirdSoft/
├── bash-5.1.16.tar.gz
├── openssl-1.1.1q.tar.gz
├── elasticsearch-7.17.5-jdk.tar.gz      ← ES 已存在
├── elasticsearch_exporter-1.7.0.linux-amd64.tar.gz   ← 需要新增
├── kafka_exporter-1.6.0.linux-amd64.tar.gz
├── node_exporter-1.1.2.linux-amd64.tar.gz
├── clickhouse/
├── kafka_2.12-3.4.0.tgz
└── ...
```

### 7.5 参考：kafka_exporter 的配置写法

```yaml
- name: kafka
  filepath: |
    thirdSoft/kafka_2.12-3.4.0.tgz
    thirdSoft/kafka_exporter-1.6.0.linux-amd64.tar.gz   ← 多行写法
  related: zookeeper
  sql: delete from CLUSTER_SERVICE_BASIC where ID=108;
```

---

## 8. transfer 阶段分析

`tools/install.sh` 的 `transfer()` 函数：
```bash
function transfer() {
  exec_ansible playbooks/transfer/01.asap.yml
  exec_ansible playbooks/transfer/17.transfer-prometheus.yml
  exec_ansible playbooks/transfer/28.transfer-kafka_exporter.yml
  # ... 共 31 个 transfer playbook
}
```

**关键发现**：`transfer-elasticsearch_exporter.yml` 不存在！

这说明 elasticsearch_exporter 的二进制传输可能通过以下方式之一：
1. 包含在 `asap.tar.gz` 或其他大包中
2. 直接从 Ansible role 的 `files/` 目录复制
3. 需要新增 `transfer-elasticsearch_exporter.yml`

---

## 9. transfer 阶段分析

`tools/install.sh` 的 `transfer()` 函数：
```bash
function transfer() {
  exec_ansible playbooks/transfer/01.asap.yml
  exec_ansible playbooks/transfer/17.transfer-prometheus.yml
  exec_ansible playbooks/transfer/28.transfer-kafka_exporter.yml
  # ... 共 31 个 transfer playbook
}
```

**关键发现**：`transfer-elasticsearch_exporter.yml` 不存在！

这说明 elasticsearch_exporter 的二进制传输可能通过以下方式之一：
1. 包含在 `asap.tar.gz` 或其他大包中
2. 直接从 Ansible role 的 `files/` 目录复制
3. 需要新增 `transfer-elasticsearch_exporter.yml`

---

## 10. 相关资源

- Jenkins: http://10.21.10.197:10110
- 主 Job: build_daily_xdr6.0.4
- 打包配置仓库: https://gitlab.asiainfo-sec.com/maxs-ops/ipaas-packing
- maxs-ops 仓库: https://gitlab.asiainfo-sec.com/maxs-ops/maxs-ops.git
- 打包服务器: 10.21.10.197
- 打包配置本地路径: /Users/fangkun/code/xdr/ipaas-packing/config.yaml

---

## 11. 附录：config.yaml 关键配置

### 11.1 路径配置 (第 1-13 行)
```yaml
param:
  - name: src
    value: /data/packing
  - name: dest
    value: /data/packing/build
  - name: REMOTE_VM
    value: 192.168.11.195
  - name: REMOTE_USER
    value: root
  - name: ROOT_PASSWD
    value: maxs.PDG~2024
```

### 11.2 preScript 中的 thirdSoft 复制 (第 75-85 行)
```bash
mkdir -p $DEST/thirdSoft
cp $SRC/thirdSoft/bash-5.1.16.tar.gz $DEST/thirdSoft
cp $SRC/thirdSoft/openssl-1.1.1q.tar.gz $DEST/thirdSoft
cp -rf $SRC/thirdSoft/paramiko $DEST/thirdSoft
cp -rf $SRC/thirdSoft/psutil $DEST/thirdSoft
cp -rf $SRC/thirdSoft/docker $DEST/thirdSoft
cp $SRC/thirdSoft/python3.tar.gz $DEST/thirdSoft
cp $SRC/thirdSoft/openssh.tar.gz $DEST/thirdSoft
cp $SRC/thirdSoft/simsun.ttf $DEST/thirdSoft
```

---

**分析时间**: 2026-05-19
**分析人**: Claude Code