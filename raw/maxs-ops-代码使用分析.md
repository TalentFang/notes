# maxs-ops 代码库使用情况全面分析

> 分析日期：2026-07-14

---

## 一、主流程入口总览

```
install.sh                  ← 全量安装
platform-ops.sh             ← 统一运维入口（install/add/del/start/stop/restart/status/destroy/modifyip...）
uninstall.sh                ← 全量卸载
single_upgrade_cluster.sh   ← 单机→集群升级
```

---

## 二、已弃用/未使用的代码

### 2.1 弃用脚本

| 脚本 | 说明 |
| --- | --- |
| `tools/upgrade_deploy.sh` | 业务微服务升级脚本，无任何外部调用方，功能已被 `deploy_scale.sh` + `deploy_upper.sh` 调用链完全覆盖 |

### 2.2 孤儿 Playbooks（19 个，完全无引用）

| # | playbook | 说明 |
| --- | --- | --- |
| 1 | `add/add-proxysql-tag.yml` | proxysql add 的 tag 变体，未被调用 |
| 2 | `add/add-psutil.yml` | 已被 `common/07.psutil.yml` 替代 |
| 3 | `add/add-python3.yml` | 未被任何地方调用 |
| 4 | `backup/backup_business.yml` | 业务备份，从未执行 |
| 5 | `business/02.business-service-sql.yml` | SQL 服务 playbook，未被编排调用 |
| 6 | `business/02.business-service-sql-ck.yml` | ClickHouse SQL 服务，未被编排调用 |
| 7 | `business/02.business-service-sql-dm8.yml` | DM8 SQL 服务，未被编排调用 |
| 8 | `business/02.business-service-sql-vastbase.yml` | Vastbase SQL 服务，未被编排调用 |
| 9 | `business/08.business-sh-scale-deploy.yml` | scale 部署 playbook，未被编排调用 |
| 10 | `common/01.check.yml` | 检查 playbook，未被引用 |
| 11 | `destroy/29.dm8.yml` | DM8 销毁 playbook，未被引用 |
| 12 | `etcd/etcd.yml` | 独立 etcd playbook，未被引用（走 `k8s/03.etcd.yml`） |
| 13 | `flink/flink-on-k8s.yml` | K8s 上 Flink playbook，未被引用 |
| 14 | `k8s/19.kube-createstorage.yml` | 存储创建 playbook，未被引用 |
| 15 | `logstash/01.logstash.yml` | Logstash playbook，未被引用（但 role 存在） |
| 16 | `restore/restore_business.yml` | 业务恢复 playbook，未被引用 |
| 17 | `upgrade/01.upgrade_rpms.yml` | RPM 升级，未被编排调用 |
| 18 | `upgrade/02.upgrade_docker.yml` | Docker 升级，未被编排调用 |
| 19 | `upgrade/03.upgrade_redis_data.yml` | Redis 数据升级，未被编排调用 |

### 2.3 孤儿 Roles（3 个，完全无引用）

| # | role | 说明 |
| --- | --- | --- |
| 1 | `cmake-kylin` | 无任何 playbook 或脚本引用 |
| 2 | `hadoop-del` | 无任何 playbook 或脚本引用 |
| 3 | `upgrade-openssh-kylin` | 仅在被注释的代码中出现 |

### 2.4 仅被注释引用的代码

| 文件/角色 | 说明 |
| --- | --- |
| `roles/install-java` | 仅在 `common/06.rpm-install.yml` 和 `add/add-common-pre.yml` 中被注释掉，未被实际执行 |
| `roles/transfer-openssh-kylin` | 仅在 `transfer/27.transfer-openssh.yml` 中注释出现 |

### 2.5 幻影引用（代码引用了不存在的文件）

| 代码引用的路径 | 实际存在的文件 | 位置 |
| --- | --- | --- |
| `playbooks/common/14.dependency_check.yml` | `14.dependency-check.yml`（下划线 vs 连字符） | `platform-ops.sh:929` |
| `playbooks/precheck/base.yml` | `precheck/base.yaml`（.yml vs .yaml） | `platform-ops.sh:610` |

---

## 三、主流程中使用的内容

### 3.1 核心 Playbooks（231 个活跃）

#### 预检

| playbook | 入口 |
| --- | --- |
| `precheck/base.yaml` | install.sh, platform-ops.sh |

#### 传输（30 个）

| playbook | 入口 |
| --- | --- |
| `transfer/01.asap.yml` | install.sh, add |
| `transfer/02.oper-backup.yaml` | install.sh, add |
| `transfer/04.k8s.yml` | install.sh, add |
| `transfer/05.hadoop.yml` | install.sh, add |
| `transfer/06.transfer-openssl-bash.yml` | install.sh, add |
| `transfer/07.transfer-mysql.yml` | install.sh, add |
| `transfer/08.transfer-redis.yml` | install.sh, add |
| `transfer/09.transfer-clickhouse.yml` | install.sh, add |
| `transfer/10.transfer-kernel.yml` | install.sh, add |
| `transfer/11.transfer-psutil.yml` | install.sh, add |
| `transfer/12.transfer-runtime.yml` | install.sh, add |
| `transfer/13.transfer-mongodb.yml` | install.sh, add |
| `transfer/14.transfer-es.yml` | install.sh, add |
| `transfer/15.transfer-spark.yml` | install.sh, add |
| `transfer/16.transfer-flink.yml` | install.sh, add |
| `transfer/17.transfer-prometheus.yml` | install.sh, add |
| `transfer/18.transfer-rpm.yml` | install.sh, add |
| `transfer/19.transfer-nginx.yml` | install.sh, add |
| `transfer/20.transfer-keepalived.yml` | install.sh, add |
| `transfer/21.transfer-minio.yml` | install.sh, add |
| `transfer/22.transfer-zookeeper.yml` | install.sh, add |
| `transfer/23.transfer-kafka.yml` | install.sh, add |
| `transfer/24.transfer-proxysql.yml` | install.sh, add |
| `transfer/25.transfer-livy.yml` | install.sh, add |
| `transfer/26.transfer-registry.yml` | install.sh, add |
| `transfer/27.transfer-openssh.yml` | install.sh, add |
| `transfer/28.transfer-kafka_exporter.yml` | install.sh, add |
| `transfer/29.transfer-nebula.yml` | install.sh, add |
| `transfer/30.transfer-dm8.yml` | install.sh, add |
| `transfer/31.transfer-vastbase.yml` | install.sh, add |
| `transfer/32.transfer-elasticsearch_exporter.yml` | install.sh, add |

#### 公共安装（23 个）

| playbook | 入口 |
| --- | --- |
| `common/00.chronyd.yml` | install.sh, add |
| `common/03.create_user_asap.yml` | install.sh, add |
| `common/04.decompression.yml` | install.sh |
| `common/05.sys_settings.yml` | install.sh |
| `common/06.rpm-install.yml` | install.sh |
| `common/06.remove_tar.yml` | install.sh |
| `common/07.psutil.yml` | install.sh |
| `common/08.firewalld.yml` | install.sh, add |
| `common/09.service_manage.yml` | install.sh, add |
| `common/10.upgade_openssl_bash.yml` | install.sh, add |
| `common/11.upgade_openssh.yml` | install.sh, add |
| `common/12.upgade_kernel.yml` | install.sh, add |
| `common/13.reboot.yml` | install.sh, add |
| `common/14.dependency-check.yml` | install.sh, add |
| `common/15.upgade_curl.yml` | install.sh, add |
| `common/16.ipv6.yml` | install.sh |
| `common/17.del_httpd.yml` | install.sh |
| `common/18.install_dependency.yml` | add（欧拉24.03） |
| `common/20.image-handle.yml` | install.sh |
| `common/30.common-param.yml` | install.sh, add, modifyip |
| `common/40.upgrade-ipaas-patch.yml` | install.sh |
| `common/41.log-collection.yml` | install.sh |
| `common/42.jq.yml` | install.sh |

#### 中间件安装（14 个）

| playbook | 组件 |
| --- | --- |
| `mysql/01.mysql.yml` | MySQL（MariaDB） |
| `dm8/01.dm8.yml` | DM8 |
| `vastbase/01.vastbase.yml` | VastBase |
| `mongodb/01.mongodb.yml` | MongoDB |
| `redis/redis.yml` | Redis |
| `clickhouse/clickhouse.yml` | ClickHouse |
| `es/es.yml` | ElasticSearch |
| `kafka/kafka.yml` | Kafka |
| `zookeeper/zookeeper.yml` | ZooKeeper |
| `hadoop/hadoop.yml` | Hadoop |
| `spark/spark.yml` | Spark |
| `livy/livy.yml` | Livy |
| `flink/flink.yml` | Flink |
| `nebula/nebula.yml` | Nebula |

#### K8s 相关（24 个）

| playbook | 作用 |
| --- | --- |
| `k8s/01.prepare.yml` | K8s 准备 |
| `k8s/02.runtime.yml` | Docker 运行时安装 |
| `k8s/03.etcd.yml` | etcd 集群部署 |
| `k8s/04.nginx.yml` | Nginx 负载均衡 |
| `k8s/05.keepalived.yml` | KeepAlived VIP |
| `k8s/06.kube-master.yml` | K8s Master 部署 |
| `k8s/07.kube-node.yml` | K8s Node 部署 |
| `k8s/08.calico.yml` | Calico 网络 |
| `k8s/09.configmap.yml` | K8s ConfigMap |
| `k8s/10.ingress-nginx.yml` | Ingress 控制器 |
| `k8s/11.coredns.yml` | CoreDNS |
| `k8s/12.kube-stat.yml` | kube-state-metrics |
| `k8s/13.cert99.yml` | 证书管理 |
| `k8s/14.minio.yml` | MinIO 部署 |
| `k8s/15.registry.yml` | Docker Registry |
| `k8s/16.kube-status-check.yml` | K8s 状态检查 |
| `k8s/17.k8s-add-lable.yml` | 添加标签 |
| `k8s/18.k8s-lable.yml` | K8s 标签管理 |
| `k8s/30.kube-cp.yml` | kubectl cp 工具 |
| `k8s/31.kube-exec.yml` | kubectl exec 工具 |
| `k8s/32.metrics-server.yml` | Metrics Server |
| `k8s/33.k9s.yml` | k9s 工具 |
| `k8s/34.add-network-policy.yml` | 网络策略 |
| `k8s/35.kube-reserved-config.yml` | 资源预留配置 |

#### 监控组件（5 个）

| playbook | 组件 |
| --- | --- |
| `prometheus/prometheus.yml` | Prometheus |
| `prometheus/pushgateway.yml` | Pushgateway |
| `kafka_exporter/kafka_exporter.yml` | Kafka Exporter |
| `keepalived_exporter/keepalived_exporter.yml` | Keepalived Exporter |
| `elasticsearch_exporter/elasticsearch_exporter.yml` | ES Exporter |

#### 业务部署（13 个活跃）

| playbook | 作用 |
| --- | --- |
| `business/00.business-pre.yml` | 业务部署前置 |
| `business/01.business.yml` | 业务部署主流程 |
| `business/03.business-sql-file.yml` | SQL 文件执行 |
| `business/03.business-sql-ck-file.yml` | ClickHouse SQL |
| `business/03.business-sql-dm8-file.yml` | DM8 SQL |
| `business/03.business-sql-vastbase-file.yml` | Vastbase SQL |
| `business/04.business-file-upload.yml` | 文件上传 |
| `business/05.business-sh-deploy.yml` | Shell 脚本执行 |
| `business/06.business-sh-pre-post-deploy.yml` | Pre/Post 脚本 |
| `business/07.business-file-upload-exec.yml` | 上传并执行 |
| `business/10.business-install-pre.yml` | 业务安装前置 |
| `business/11.file-scp-exec.yml` | SCP 执行 |
| `business/12.dispatch-flink-files.yml` | Flink 文件分发 |

#### 扩容（~50 个 `add/*.yml`）

| playbook | 组件 |
| --- | --- |
| `add/add-common-pre.yml` / `add/add-common-post.yml` | 通用扩容前置/后置 |
| `add/add-clickhouse.yml` / `add/add-clickhouse-tag.yml` | ClickHouse |
| `add/add-clickhousekeeper.yml` / `add/add-clickhousekeeper-tag.yml` | ClickHouse Keeper |
| `add/add-es.yml` / `add/add-es-tag.yml` | ES |
| `add/add-etcd.yml` / `add/add-etcd-tag.yml` | etcd |
| `add/add-flink.yml` / `add/add-flink-tag.yml` / `add/add-flink-monitor.yml` | Flink |
| `add/add-flume.yml` / `add/add-flume-tag.yml` | Flume |
| `add/add-hadoop.yml` / `add/add-hadoop-tag.yml` | Hadoop |
| `add/add-kafka.yml` / `add/add-kafka-tag.yml` / `add/add-kafka_exporter.yml` | Kafka |
| `add/add-k8s-conf-copy.yml` / `add/add-k8s-configmap.yml` / `add/add-k8s-shell.yml` | K8s 配置 |
| `add/add-kube-cert.yml` / `add/add-kube-master.yml` / `add/add-kube-node.yml` | K8s 节点 |
| `add/add-livy.yml` / `add/add-livy-tag.yml` | Livy |
| `add/add-minio.yml` / `add/add-minio-tag.yml` | MinIO |
| `add/add-mongodb.yml` | MongoDB |
| `add/add-mysql.yml` / `add/add-mysql-tag.yml` / `add/add-mysql-data.yml` | MySQL |
| `add/add-nebula.yml` / `add/add-nebula-tag.yml` | Nebula |
| `add/add-nginx.yml` / `add/add-nginx-tag.yml` | Nginx |
| `add/add-prometheus.yml` | Prometheus |
| `add/add-proxysql.yml` | ProxySQL |
| `add/add-pushgateway.yml` | Pushgateway |
| `add/add-redis.yml` / `add/add-redis-tag.yml` | Redis |
| `add/add-registry.yml` | Registry |
| `add/add-spark.yml` / `add/add-spark-tag.yml` | Spark |
| `add/add-vastbase.yml` | VastBase |
| `add/add-zookeeper.yml` / `add/add-zookeeper-tag.yml` | ZooKeeper |

#### 销毁（~33 个 `destroy/*.yml`）

| playbook | 组件 |
| --- | --- |
| `destroy/01.etcd.yml` | etcd |
| `destroy/02.docker.yml` / `destroy/02.k8s.yml` | Docker / K8s |
| `destroy/03.crontab.yml` | Crontab |
| `destroy/04.livy.yml` | Livy |
| `destroy/05.flink.yml` | Flink |
| `destroy/06.es.yml` | ES |
| `destroy/07.kafka.yml` | Kafka |
| `destroy/08.mongodb.yml` | MongoDB |
| `destroy/09.redis.yml` | Redis |
| `destroy/10.zookeeper.yml` | ZooKeeper |
| `destroy/11.nginx.yml` | Nginx |
| `destroy/12.proxysql.yml` | ProxySQL |
| `destroy/13.clickhouse.yml` / `destroy/13.clickhouse2.yml` | ClickHouse |
| `destroy/13.keepalived.yml` | KeepAlived |
| `destroy/14.mysql.yml` | MySQL |
| `destroy/15.hadoop.yml` | Hadoop |
| `destroy/16.spark.yml` | Spark |
| `destroy/17.minio.yml` / `destroy/17.registry.yml` | MinIO / Registry |
| `destroy/18.prometheus.yml` | Prometheus |
| `destroy/20.nebula.yml` | Nebula |
| `destroy/21.pushgeteway.yml` | Pushgateway |
| `destroy/22.kafka-exporter.yml` | Kafka Exporter |
| `destroy/23.business.yml` | 业务 |
| `destroy/24.reset_hosts.yml` | hosts 文件重置 |
| `destroy/25.files_and_vars.yml` | 文件清理 |
| `destroy/26.node_exporter.yml` | Node Exporter |
| `destroy/27.reset_firewalld.yml` | 防火墙重置 |
| `destroy/30.k9s.yml` | k9s |
| `destroy/31.keepalived_exporter.yml` | Keepalived Exporter |
| `destroy/32.vastbase.yml` | VastBase |
| `destroy/33.elasticsearch_exporter.yml` | ES Exporter |

#### 系统管理（12 个 `system/*.yml`）

| playbook | 作用 |
| --- | --- |
| `system/start.yml` / `system/stop.yml` | 启动/停止 |
| `system/restart.yml` | 重启 |
| `system/status.yml` | 状态查询 |
| `system/enable.yml` / `system/disable.yml` | 启用/禁用 |
| `system/template.yml` | 通用 systemd 模板 |
| `system/template_status.yml` / `system/template_status_when.yml` | 状态模板 |
| `system/template_enable.yml` / `system/template_disable.yml` | 启停模板 |
| `system/passwdmodify.yml` | 密码修改 |

#### 配置刷新（8 个 `config/*.yml`）

| playbook | 组件 |
| --- | --- |
| `config/01.mysql.yml` | MySQL |
| `config/02.redis.yml` | Redis |
| `config/03.es.yml` | ES |
| `config/04.zookeeper.yml` | ZooKeeper |
| `config/05.kafka.yml` | Kafka |
| `config/06.flink.yml` | Flink |
| `config/07.hadoop.yml` | Hadoop |
| `config/08.clickhouse.yml` | ClickHouse |

#### 其他

| playbook | 作用 |
| --- | --- |
| `authenticate/authenticate.yml` | SSH 免密 |
| `backup/backup_etcd.yml` | etcd 备份 |
| `restore/restore_etcd.yml` | etcd 恢复 |
| `oper-backup/oper-backup.yaml` | 操作机备份 |
| `nfs/nfs.yml` | NFS 部署 |
| `proxysql/proxysql.yml` | ProxySQL |
| `disable/disable-mongodb.yml` | MongoDB 禁用 |
| `disable/disable-vastbase.yml` | VastBase 禁用 |
| `etcd/etcd-remove.yml` | etcd 移除 |
| `single_upgrade_cluster/01~07.yml` | 单机升集群（7 个 playbook） |

### 3.2 活跃 Roles（190 个，按类型分）

#### 核心安装 Roles（~30 个）

`kafka`, `es`, `redis`, `clickhouse1`, `clickhouse2`, `clickhousekeeper`, `hadoop`, `spark`, `flink`, `mysql`, `mysql-arm`, `mongodb`, `zookeeper`, `nebula`, `vastbase`, `dm8`, `dm-python`, `nginx`, `bes`, `keepalived`, `minio`, `registry`, `livy`, `proxysql`, `nfs`, `flink-on-k8s`, `logstash` 等

#### K8s Roles（~20 个）

`calico`, `coredns`, `ingress-nginx`, `kube-master`, `kube-master-cert`, `kube-node`, `kube-stat`, `configmap`, `metrics-server`, `prepare`, `cert99`, `kube-cp`, `kube-create-class`, `kube-join-shell`, `kube-reserved-config`, `kube-status-check`, `add-network-policy`, `etcd`, `etcd-add`, `etcd-conf-copy`, `etcd-remove`, `etcd-restore`, `k9s`

#### 传输 Roles（~33 个）

`transfer-kafka`, `transfer-es`, `transfer-redis`, `transfer-mysql`, `transfer-clickhouse`, `transfer-hadoop`, `transfer-spark`, `transfer-flink`, `transfer-livy`, `transfer-mongodb`, `transfer-zookeeper`, `transfer-nebula`, `transfer-nginx`, `transfer-bes`, `transfer-keepalived`, `transfer-minio`, `transfer-registry`, `transfer-prometheus`, `transfer-kafka_exporter`, `transfer-elasticsearch_exporter`, `transfer-dm8`, `transfer-vastbase`, `transfer-proxysql`, `transfer-docker`, `transfer-openssl`, `transfer-openssh`, `transfer-bash`, `transfer-kernel`, `transfer-psutil`, `transfer-rpm`, `transfer-asap`, `transfer-backup`, `transfer-nfs`

#### 配置刷新 Roles（8 个）

`refresh-config-mysql`, `refresh-config-redis`, `refresh-config-es`, `refresh-config-zookeeper`, `refresh-config-kafka`, `refresh-config-flink`, `refresh-config-hadoop`, `refresh-config-clickhouse`

#### 扩容 Roles（~25 个）

`kafka-add`, `es-add`, `redis-add`, `hadoop-add`, `spark-add`, `flink-add`, `livy-add`, `mongodb-add`, `add-minio`, `add-vastbase`, `flink-monitor`, `etcd-add`, `mysql-data`, `mysql-data-upgrade`, `zookeeper-cluster-data`, `hadoop-cluster-data`, `minio-data`, `minio-single-bak`, `minio-single-rollback`, `es-upgade-cluster`, `vastbase-single-upgrade` 等

#### 公共 Roles（~23 个）

`rpm-install`, `psutil`, `python3`, `sys-settings`, `firewalld`, `decompression`, `move-tar`, `usradd-asap`, `usradd-dmdba`, `usradd-vastbase`, `upgrade-openssl`, `upgrade-openssh`, `upgrade-bash`, `curl`, `upgrade-kernel`, `upgrade-kernel2`, `upgrade-rpms`, `reboot`, `install-dependency`, `chrony`, `service-manage`, `jq`, `ipv6`, `delhttpd`, `image-handle`, `upgrade-ipaas-patch`, `check`, `dependency-check`, `log-collection`

#### 业务部署 Roles（~17 个）

`business-deploy`, `business-deploy-pre`, `business-install-pre`, `business-sh-deploy`, `business-sh-pre-pos-deploy`, `business-sh-scale-deploy`, `business-scale-copy`, `business-sql-file-deploy`, `business-sql-file-ck-deploy`, `business-sql-file-dm8-deploy`, `business-sql-file-vastbase-deploy`, `business-service-sql-deploy`（系列）, `business-file-upload`, `business-file-upload-exec`, `file-scp-exec`, `dispatch-flink-files`, `oper-backup`

#### 监控 Roles（~7 个）

`prometheus`, `node_exporter`, `kafka_exporter`, `pushgateway`, `elasticsearch_exporter`, `keepalived_exporter`, `k9s`

#### 工具型 Roles（13 个，仅被 `tools/*.sh` 引用）

`asap-log-purge`, `asap-monitor`, `change-ip`, `clear-hostname`, `device-id`, `docker-refresh`, `es-refresh`, `etcd-refresh`, `kube-master-refresh`, `mongodb-refresh`, `openssl`, `refresh-config-vastbase`, `reset-business`

### 3.3 工具脚本（39 个，全部活跃）

#### 入口编排

- `install.sh` — 全量安装
- `platform-ops.sh` — 统一运维入口
- `uninstall.sh` — 全量卸载
- `single_upgrade_cluster.sh` — 单机→集群升级

#### 核心工具

- `authenticate.sh` — SSH 免密
- `sh_pre_post_tool.sh` — Pre/Post 脚本执行
- `sh_destroy_tool.sh` — 销毁脚本执行
- `db_file_tool.sh` — SQL 文件执行
- `group_info_tool.sh` — 集群组信息查询
- `install_common.sh` — 通用节点安装
- `refresh_config.sh` — 组件配置刷新
- `kube_tool.sh` — kubectl 封装
- `create_master_mapping.sh` — Kafka master 映射
- `registry_tool.sh` — 镜像仓库管理

#### 辅助工具

- `ip_replace_record.sh` — IP 替换记录
- `modify_ansible_connection.sh` — ansible 连接修改
- `passwdmodify.sh` — 密码修改
- `cm_auth.sh` — CM 认证
- `cluster_status_tool.sh` — 集群状态检查
- `check_disk_type.sh` — 磁盘类型检测
- `iops_check.sh` — IOPS 检查
- `log_collection.sh` — 日志收集
- `log_collection_stat.sh` — 日志收集统计
- `print_os_info.sh` — OS 信息打印
- `recover_k8s_calico.sh` — Calico 恢复
- `k8s-add-lable.sh` — K8s 标签添加
- `update_inventory_host.sh` — Inventory 更新

#### 文件传输

- `file_scp_exec_tool.sh` — SCP + 执行
- `file_upload_exec_tool.sh` — 上传 + 执行
- `file_upload_tool.sh` — 文件上传
- `sh_exec_tool.sh` — Shell 执行
- `db_exec_directory_tool.sh` — 批量 SQL 执行

---

## 四、可以安全删除的代码清单

| 优先级 | 文件/目录 | 理由 |
| --- | --- | --- |
| 高 | `tools/upgrade_deploy.sh` | 无外部调用方，功能已被 `deploy_scale.sh` + `deploy_upper.sh` 替代 |
| 高 | `roles/cmake-kylin/` | 完全无引用 |
| 高 | `roles/hadoop-del/` | 完全无引用 |
| 高 | `roles/upgrade-openssh-kylin/` | 完全无引用（仅注释中出现） |
| 高 | `roles/transfer-openssh-kylin/` | 仅注释引用 |
| 高 | `playbooks/upgrade/01.upgrade_rpms.yml` | 未被编排调用 |
| 高 | `playbooks/upgrade/02.upgrade_docker.yml` | 未被编排调用 |
| 高 | `playbooks/upgrade/03.upgrade_redis_data.yml` | 未被编排调用 |
| 高 | `playbooks/logstash/01.logstash.yml` + `roles/logstash/` | 从未被调用 |
| 高 | `playbooks/etcd/etcd.yml` | 已被 `k8s/03.etcd.yml` 替代 |
| 高 | `playbooks/flink/flink-on-k8s.yml` + `roles/flink-on-k8s/` | 未被调用 |
| 中 | `playbooks/add/add-proxysql-tag.yml` | proxysql 扩容 tag 变体，未被调用 |
| 中 | `playbooks/add/add-psutil.yml` | 已被 `common/07.psutil.yml` 替代 |
| 中 | `playbooks/add/add-python3.yml` | 未被调用 |
| 中 | `playbooks/backup/backup_business.yml` | 未被调用 |
| 中 | `playbooks/restore/restore_business.yml` | 未被调用 |
| 中 | `playbooks/business/02.business-service-sql.yml` | SQL 服务 playbook，未被编排 |
| 中 | `playbooks/business/02.business-service-sql-ck.yml` | 同上 |
| 中 | `playbooks/business/02.business-service-sql-dm8.yml` | 同上 |
| 中 | `playbooks/business/02.business-service-sql-vastbase.yml` | 同上 |
| 中 | `playbooks/business/08.business-sh-scale-deploy.yml` | scale 部署，未被编排 |
| 中 | `playbooks/common/01.check.yml` | 检查 playbook，未被引用 |
| 中 | `playbooks/destroy/29.dm8.yml` | DM8 销毁，未被引用 |
| 中 | `playbooks/k8s/19.kube-createstorage.yml` | 存储创建，未被引用 |
| 低 | `roles/install-java/` | 所有引用均被注释，实际安装走 rpm-install |
| 修复 | `platform-ops.sh:929` | `14.dependency_check.yml` → `14.dependency-check.yml` |
| 修复 | `platform-ops.sh:610` | `base.yml` → `base.yaml` |

---

## 五、统计小结

| 类型 | 总数 | 活跃 | 弃用/孤儿 | 弃用率 |
| --- | --- | --- | --- | --- |
| Playbooks | 250 | 231 | 19 | 7.6% |
| Roles | 193 | 190（177+13） | 3 | 1.6% |
| 工具脚本 | 40 | 39 | 1 | 2.5% |

整体代码库利用率很高，弃用代码占比不大。主要有：
- **1 个弃用脚本**：`upgrade_deploy.sh`
- **3 个孤儿 role**，可安全删除
- **19 个未被调用的 playbook**（含 3 个 `upgrade/` 遗留、2 个 `logstash/flink-on-k8s` 未启用、4 个 `business/02` 未被编排等）
- **2 处文件名不匹配 bug**，需修复
- `roles/install-java` 引用全部被注释，是否仍需要需确认
