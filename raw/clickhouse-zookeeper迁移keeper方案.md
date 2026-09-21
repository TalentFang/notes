# ClickHouse + ZooKeeper 迁移 ClickHouse Keeper 方案(官方工具)

## 背景与目标

- 当前:ck server + ZooKeeper(`KEEPER_TYPE=ZOOKEEPER`),zk 3 节点 `/data/comm/zookeeper-3.7.2`(服务 `asap_zookeeper`,端口 2181)
- 目标:ck server + ClickHouse Keeper(`KEEPER_TYPE=KEEPER`),keeper 3 节点(服务 `clickhouse-keeper`,tcp 9181 / raft 9234)
- 迁移工具:**`clickhouse-keeper-converter`**(ClickHouse 22.4+ 自带,官方唯一支持方式;或全量二进制 `clickhouse keeper-converter`)
- 关键前提:ZooKeeper 与 Keeper 存储格式不兼容、共识协议不同,**无法在线/混合迁移**,必须停 zk → 转换 → 起 keeper;要求 ZooKeeper 3.4+

### 目录约定(本仓库)

| 组件 | 路径 |
|------|------|
| zk snapshot | `/data/comm/zookeeper-3.7.2/data/version-2/` |
| zk transaction log | `/data/comm/zookeeper-3.7.2/logs/version-2/` |
| keeper snapshot 目录 | `/data/comm/clickhousekeeper/coordination/snapshots/` |
| keeper log 目录 | `/data/comm/clickhousekeeper/coordination/logs/` |

---

## 一、迁移前验证(zk 模式完好)

```bash
# 1. 工具可用性
clickhouse-keeper-converter --version           # 或 clickhouse keeper-converter --version

# 2. 服务(每节点)
systemctl is-active clickhouse-server asap_zookeeper      # 期望都 active

# 3. zk 集群角色与版本
/data/comm/zookeeper-3.7.2/bin/zkServer.sh status         # 1 leader + 2 follower
echo srvr | nc 127.0.0.1 2181                             # 确认 ZK 版本 >= 3.4
```

```sql
-- 4. ck 集群拓扑与副本同步
clickhouse-client --host <节点> --port 9002 --user root --password 'maxs.PDG~2022'
SELECT * FROM system.clusters WHERE cluster='cluster_ck';   -- 3 分片拓扑正确
SELECT database, table, is_leader, is_readonly, absolute_delay
FROM system.replicas;                                       -- 全部 is_readonly=0、delay=0
```

## 二、迁移前数据 / 元数据检查(留基线)

```sql
-- 数据:Replicated 表清单、parts 总量
SELECT database, count() AS tables FROM system.tables
  WHERE engine LIKE 'Replicated%' GROUP BY database;

SELECT * FROM system.zookeeper WHERE path='/';

-- 元数据:每张 Replicated 表对应的 zk path(迁移后逐条核对)
SELECT database, table, zookeeper_path FROM system.replicas where database='default';
```

```bash
# 导出 zk 中 ck 元数据树(留档 + 用于核对)
/data/comm/zookeeper-3.7.2/bin/zkCli.sh -server 127.0.0.1:2181 ls -R /clickhouse > /tmp/ck_znode_before.txt
```

## 三、迁移步骤(官方 converter 流程)

### 阶段 1 — 部署 keeper(先不启动)

1. inventory 增加 `clickhousekeeper` 组(可复用 3 台 zk 节点或独立 3 台),设 `KEEPER_TYPE=KEEPER`。
2. 准备好 keeper 目录与配置,但**先不启动**(避免在无快照时选出空 leader):
   ```bash
   # 只准备目录/二进制/配置,不执行 systemctl start
   ansible-playbook -l clickhousekeeper playbooks/clickhouse/clickhouse.yml --tags add_clickhouse --skip-tags add_clickhouse_start
   ```
   > 若 keeper 已空跑过,阶段 5 放快照前必须 `systemctl stop clickhouse-keeper` 并清空 `coordination/logs`、`coordination/snapshots`。

### 阶段 2 — 停写入 + 停后台任务(所有 ck 节点)

```sql
SYSTEM STOP MERGES;
SYSTEM STOP FETCHES;
SYSTEM STOP REPLICATED SENDS;
SYSTEM STOP DISTRIBUTED SENDS;
```


```

# 停止
clickhouse-client --user=default --password=maxs.PDG~2022 --query="SYSTEM STOP MERGES ON CLUSTER cluster_ck;SYSTEM STOP FETCHES ON CLUSTER cluster_ck;SYSTEM STOP REPLICATED SENDS ON CLUSTER cluster_ck;SYSTEM STOP DISTRIBUTED SENDS ON CLUSTER cluster_ck;"

# 验证
SELECT count() FROM system.merges WHERE is_currently_executing = ;
SELECT count() FROM system.replication_queue;

# 恢复
clickhouse-client --user=default --password=maxs.PDG~2022 --query="SYSTEM START MERGES ON CLUSTER cluster_ck;SYSTEM START FETCHES ON CLUSTER cluster_ck;SYSTEM START REPLICATED SENDS ON CLUSTER cluster_ck;SYSTEM START DISTRIBUTED SENDS ON CLUSTER cluster_ck;"
```

### 阶段 3 — 停 zk + 强制刷新一致快照

```bash
# 1. 记录 leader
/data/comm/zookeeper-3.7.2/bin/zkServer.sh status    # 找 Mode: leader 的节点

# 2. 停所有 zk 节点
ansible zookeeper -m service -a "name=asap_zookeeper state=stopped"

# 3. 推荐:单独重启一次 leader 再停,强制把最新状态刷成一致快照到磁盘
#    (在 leader 上执行 start 再 stop)
```

### 阶段 4 — 在 leader 上运行 converter

在 **zk leader** 节点执行,转换生成单个 Keeper 快照:

```bash
clickhouse-keeper-converter \
  --zookeeper-logs-dir /data/comm/zookeeper-3.7.2/logs/version-2 \
  --zookeeper-snapshots-dir /data/comm/zookeeper-3.7.2/data/version-2 \
  --output-dir /tmp/keeper_snapshot

ls /tmp/keeper_snapshot/          # 生成 snapshot_<N>.bin
```

### 阶段 5 — 分发快照到所有 keeper 节点(启动前)

**必须在任何 keeper 节点启动前**把快照复制到位,否则该节点可能选自己为空 leader:

```bash
# 从 leader 分发到每个 keeper 节点的 snapshot 目录
for ip in <keeper1> <keeper2> <keeper3>; do
  rsync -a /tmp/keeper_snapshot/snapshot_*.bin \
    ${ip}:/data/comm/clickhousekeeper/coordination/snapshots/
done

# 确保 logs 目录为空(新 raft 日志从快照开始)
for ip in <keeper1> <keeper2> <keeper3>; do
  ssh ${ip} 'chown -R clickhouse:clickhouse /data/comm/clickhousekeeper/coordination'
done
```

### 阶段 6 — 启动 keeper 集群

```bash
ansible clickhousekeeper -m service -a "name=clickhouse-keeper state=started enabled=yes"
# 验证 raft:3 节点,1 leader + 2 follower
echo stat | nc <keeper_ip> 9181
```

### 阶段 7 — 切换 ck 配置 + 重启

1. 确认 inventory 里 `KEEPER_TYPE=KEEPER`、`clickhousekeeper` 组已就位。
2. 重刷配置并重启 ck(metrika.xml 自动把 zookeeper-servers 指向 keeper 9181):
   ```bash
   ansible-playbook playbooks/clickhouse/clickhouse.yml --tags refresh_config
   ```

### 阶段 8 — 恢复后台任务 + 停旧 zookeeper(观察稳定后再做)

```sql
SYSTEM START MERGES;
SYSTEM START FETCHES;
SYSTEM START REPLICATED SENDS;
SYSTEM START DISTRIBUTED SENDS;
```

```bash
ansible zookeeper -m service -a "name=asap_zookeeper state=stopped enabled=no"
```

## 四、迁移后验证

```sql
-- 1. 协调服务已切到 keeper(9181)
SELECT * FROM system.zookeeper WHERE path='/';

-- 2. 副本元数据完好、同步正常
SELECT database, table, is_readonly, absolute_delay, zookeeper_path FROM system.replicas;
-- 期望:is_readonly=0、delay=0、zookeeper_path 不变

-- 3. 分布式 DDL 实测(证明 keeper 协调生效)
CREATE DATABASE ck_mig_check ON CLUSTER cluster_ck;
SELECT hostName(), name FROM system.databases WHERE name='ck_mig_check';  -- 3 节点都出现
DROP DATABASE ck_mig_check ON CLUSTER cluster_ck;

-- 4. 数据抽样校验:迁移前后 parts / 行数对比(与阶段二基线一致)
SELECT count() FROM system.parts WHERE active;
```

```bash
# 用 keeper 客户端核对 /clickhouse/tables 与迁移前基线一致
clickhouse-keeper-client --host <keeper_ip> --port 9181 ls -R /clickhouse/tables
```

## 五、迁移后调优(官方推荐,写入 keeper_config.xml 的 coordination_settings)

| 设置 | 默认 | 推荐 |
|------|------|------|
| `max_requests_batch_size` | 100 | 10000 |
| `force_sync` | true | false |
| `compress_logs` | false | true(本仓库模板已设 true) |

## 六、回滚方案

zk 数据原样保留,若迁移失败:改回 `KEEPER_TYPE=ZOOKEEPER` → 重刷 metrika.xml(`--tags refresh_config`)→ 重启 ck,即回到原 zk 协调。

## 风险提示

- 核心风险在**阶段 4~6**:快照必须在 keeper 启动前分发到位;转换后到切换前的任何新写入都会丢失,故必须先停写入、停后台任务。
- converter 只支持**一对一**转换(单 zk 集群 → 单 keeper 快照),多 zk 集群合并需改源码。
- ACL:zk 全加密或全未加密可直接转换(ACL 保留);部分加密需先 `setAcl -R` 清 ACL 再转换。本仓库未启用 zk ACL 加密,可直接转换。
- Distributed DDL 队列、RBAC 等仅存于 zk 的元数据,依赖 converter 一并迁移,转换后需重点核对 `/clickhouse/task_queue/ddl`。
