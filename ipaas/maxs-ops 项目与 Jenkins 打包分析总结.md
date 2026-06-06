# maxs-ops 项目与 Jenkins 打包分析总结

---

## 1. 项目概述

**maxs-ops**: 基于 Ansible 的高可用集群自动化部署工具
- Kubernetes v1.23 + Docker v20.10.7 + Calico
- 支持 CentOS 7.9, Kylin, UOS 等操作系统
- 182 个 Ansible roles，38 个 playbook 目录

---

## 2. Jenkins 架构分析

### 2.1 关键 Jenkins Job

| Job | 类型 | 功能 |
|-----|------|------|
| `build_daily_xdr6.0.4` | MultiJob | 主构建入口（不包含 ipaas-packing） |
| `maxs-dependencies` | Maven | 依赖管理，上传到 Maven 仓库 |
| `maxs-ops` | FreeStyle | 仅 git clone 部署脚本 |
| `ipaas-packing` | FreeStyle | **打包配置（已废弃1-2年）** |
| `build_uosv25_x86` | MultiJob | 当前使用的打包 job |

### 2.2 build_daily_xdr6.0.4 触发的子 Job (32个)

```
maxs-dependencies, maxs-pm-api-linkone, maxs-soar-api-linkone,
maxs-asset-api-linkone, maxs-intel-api-linkone, maxs-commons-xdr,
maxs-monitor, python-offline, lkone-maxs-pm, lkone-maxs-intel,
lkone-maxs-rv, maxs-rule-engine_lo, maxs-rule-lo6.x,
data-process-engine-xdr, lkone-frontend, lkone-ssa-web,
lkone-vis-web, lkone-maxs-web, lkone-maxs-soar, lkone-maxs-bi-web,
maxs-ts-alone, maxs-upgrade, maxs-asset6.0.4, lkone-maxs-pboc-report,
lkone-maxs-bi, lkone-maxs-bi-engine, lkone-maxs-dm,
lkone-maxs-datareport, data-access-xdr, maxs-ops, linkone-maxs-szr
```

**注意**: `ipaas-packing` 不在触发列表中！

---

## 3. ipaas-packing 打包机制

### 3.1 仓库信息
- 地址: `https://gitlab.asiainfo-sec.com/maxs-ops/ipaas-packing`
- 分支: `release_pre`
- 本地路径: `/Users/fangkun/code/xdr/ipaas-packing`

### 3.2 config.yaml 结构

```yaml
param:
  - src: /data/packing         # 源目录
  - dest: /data/packing/build  # 目标目录

os:
  - name: centos
    cpu: x86
    preScript:    # 预执行：复制文件到 thirdSoft/
    gitScript:    # Git 脚本：从远程拉取 maxs-ops 等
    buildScript:  # 构建脚本：打包 thirdSoft.tar.gz
    isoScript:    # ISO 脚本：生成 ISO
    third:        # 第三方组件配置（数据库配置）
```

### 3.3 buildScript 核心打包逻辑

```bash
cd $DEST
tar -zcvf thirdSoft.tar.gz ./thirdSoft/
tar -zcvf asap.tar.gz ./asap/
tar -zcvf ipaas_$OS_$CPU_1.0.tar.gz \
    ./asap.tar.gz ./images/ ./rpms/ \
    ./thirdSoft.tar.gz ./maxs-ops ./install.sh
```

### 3.4 thirdSoft 文件复制 (preScript)

```bash
mkdir -p $DEST/thirdSoft
cp $SRC/thirdSoft/bash-5.1.16.tar.gz $DEST/thirdSoft
cp $SRC/thirdSoft/elasticsearch-7.17.5-jdk.tar.gz $DEST/thirdSoft
# ... 其他文件
```

### 3.5 third 组件配置示例

```yaml
third:
  - name: kafka
    filepath: |
      thirdSoft/kafka_2.12-3.4.0.tgz
      thirdSoft/kafka_exporter-1.6.0.linux-amd64.tar.gz
    sql: delete from CLUSTER_SERVICE_BASIC where ID=108;
```

### 3.6 SQL 字段说明

- `sql` 字段是**预置语句**，用于产品安装界面执行
- **不由 ipaas-packing 执行**，而是界面安装时使用
- ID 分配：mysql(101), mongodb(102), redis(103), es(104), clickhouse(105), etc.

---

## 4. 当前打包流程 (build_uosv25_x86)

### 4.1 执行命令

```bash
./ipaas-packing ./608/maxs_uosv25_x86_iso_608.csv
```

### 4.2 CSV 文件格式

```csv
vastbase,,
nginx,,
keepalived,,
minio,,
zookeeper,,
redis,,
kafka,,
es,,
registry,,
etcd,,
k8s,,
pushgateway,,
prometheus,,
clickhouse,,
business,...
isoScript,...
```

第一列是组件名称，匹配 config.yaml 中的 third 配置。

---

## 5. Elasticsearch Exporter 分析

### 5.1 问题现象

```bash
# 错误信息
connection refused  # localhost:9200 没监听
Empty reply         # 外网 IP 但要求 HTTPS
HTTP Request failed with code 401  # 需要认证
```

### 5.2 ES 配置确认

```yaml
xpack.security.http.ssl.enabled: true    # 强制 HTTPS
xpack.security.enabled: true            # 启用安全认证
```

### 5.3 Exporter 版本问题

- 版本: `elasticsearch_exporter-1.7.0.linux-amd64`
- **不支持** `--es.username` 和 `--es.password` 参数！
- 支持的参数：
  - `--es.uri` - ES 地址
  - `--es.ssl-skip-verify` - 跳过 SSL 验证
  - `--es.ca` - CA 证书
  - `--es.client-private-key` - 客户端私钥
  - `--es.client-cert` - 客户端证书

### 5.4 文件位置

- 可执行文件: `/root/elasticsearch_exporter-1.7.0.linux-amd64/elasticsearch_exporter`
- 实际路径: `/root/elasticsearch_exporter-1.7.0.linux-amd64/`

### 5.5 认证问题

- 用户 `elastic` 测试失败：`unable to authenticate user [elastic]`
- **需要获取正确的用户名密码或关闭 ES 安全认证**

---

## 6. 新增 elasticsearch_exporter.tar.gz 的步骤

### 6.1 方式：添加到 es 组件

修改 `config.yaml` 第 183-185 行：

```yaml
- name: es
  filepath: |
    thirdSoft/elasticsearch-7.17.5-jdk.tar.gz
    thirdSoft/elasticsearch_exporter-1.7.0.linux-amd64.tar.gz
  sql: delete from CLUSTER_SERVICE_BASIC where ID=104;
```

### 6.2 前提条件

1. 将 `elasticsearch_exporter-1.7.0.linux-amd64.tar.gz` 放入 `/data/packing/thirdSoft/`
2. 提交代码到 ipaas-packing 仓库
3. 触发打包构建

### 6.3 transfer 问题

`maxs-ops/roles/` 中**没有** `transfer-elasticsearch_exporter.yml`
exporter 的传输可能通过其他方式（如包含在 asap.tar.gz 中）

---

## 7. 关键路径汇总

| 用途 | 路径 |
|------|------|
| 打包源目录 | `/data/packing/thirdSoft/` |
| 打包目标目录 | `/data/packing/build/` |
| 打包服务器 | `10.21.10.197` |
| ES 服务器 | `192.168.11.205` |
| ES 监听地址 | `192.168.11.205:9200` (HTTPS + 认证) |

---

## 8. 待解决问题

1. **ES 认证方式**：需要确认正确的用户名密码或关闭安全认证
2. **Exporter 版本**：当前版本不支持用户名密码参数
3. **打包触发**：`ipaas-packing` 已废弃，当前用哪个 job 打包？
4. **文件传输**：`transfer-elasticsearch_exporter.yml` 不存在

---

## 9. 相关文档

- 分析文档: `/Users/fangkun/code/notes/jenkins_thirdparty_analysis.md`
- ipaas-packing 代码: `/Users/fangkun/code/xdr/ipaas-packing`